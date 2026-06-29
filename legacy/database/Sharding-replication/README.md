# Database Scaling: Replication & Sharding

Comprehensive interview study guide covering database scalability patterns, high-availability replication, and horizontal sharding.

---

## 1. Vertical vs. Horizontal Scaling

* **Vertical Scaling (Scale-up):**
  * *Definition:* Adding more power (CPU, RAM, SSDs) to a single database server.
  * *Limits:* Finite hardware capacity limits, high cost curves, and single point of failure (SPOF) risks.
* **Horizontal Scaling (Scale-out):**
  * *Definition:* Spreading the database load across multiple cheap servers (nodes) in a cluster.
  * *Mechanism:* Leverages Replication (for reads) and Sharding (for writes).

---

## 2. Database Replication (High Availability & Read Scale)

Replication copies the same database schema and records across multiple nodes.

```
                       ┌───────────────┐
                       │ Primary (Db)  │ (Writes only)
                       └───────┬───────┘
                               │ (Sync/Async replication)
                     ┌─────────┴─────────┐
                     ▼                   ▼
             ┌───────────────┐   ┌───────────────┐
             │ Secondary 1   │   │ Secondary 2   │ (Reads only)
             └───────────────┘   └───────────────┘
```

### Replication Strategies
1. **Synchronous Replication:**
   * *Mechanism:* The primary node waits for confirmation from replicas that the write has been written to their disks before returning success to the client.
   * *Trade-off:* High write latency, but guarantees zero data loss if the primary crashes.
2. **Asynchronous Replication:**
   * *Mechanism:* The primary commits writes immediately and replicates data to secondaries in the background.
   * *Trade-off:* Zero impact on write latency, but risk of **replication lag** and data loss if the primary crashes before replicas sync.

### Replication Formats & Data Propagation Mechanics
Data replication operates under different logical and physical formats to propagate changes down the log pipeline. Understanding these formats requires analyzing the Write-Ahead Log (WAL) internals.

#### 1. Under the Hood: The Write-Ahead Log (WAL) & LSN Tracking
All ACID-compliant databases use a **Write-Ahead Log (WAL)** (called Redo Log in MySQL/Oracle) to guarantee durability and atomicity.

* **The Write-Ahead Constraint:** The database engine *must* flush modified transactions to the WAL on physical disk before it is legally allowed to modify the actual data files (heap tables or B+Tree pages). This is because sequential disk writes to the WAL are orders of magnitude faster than random-access database page updates.
* **Log Sequence Number (LSN):** Every single record appended to the WAL is assigned a unique, monotonically increasing 64-bit integer called the **Log Sequence Number (LSN)**. LSN represents the exact byte offset in the log stream.
* **State Synchronization:** Replicas track their replication progress using LSNs. In every poll, the replica requests the primary to send logs starting from its local `Last_Applied_LSN`. If a network interruption occurs, the primary uses the LSN gap to determine exactly which WAL segments must be retransmitted, preventing full database sync scans.

```
Primary WAL Disk File
┌─────────────────┬─────────────────┬─────────────────┐
│ LSN: 1000       │ LSN: 1080       │ LSN: 1160       │ (Inflight LSN: 1240)
└────────┬────────┴────────┬────────┴────────┬────────┘
         │                 │                 │
         │ (Shipped)       │ (Shipped)       │ (Waiting in Sender Buffer)
         ▼                 ▼                 ▼
Replica Relay Log
┌─────────────────┬─────────────────┐
│ LSN: 1000       │ LSN: 1080       │ (Replica is lag-bound. Last Applied: LSN 1080)
└─────────────────┴─────────────────┘
```

#### 2. Detailed Replication Formats

##### A. Statement-Based Replication (SBR)
* **Mechanics:** The primary logs and transmits the raw SQL statements (e.g., `UPDATE accounts SET balance = balance * 1.05 WHERE active = true`).
* **Pros:** Minimal network payload. Logging a single 100-character SQL query requires less than a kilobyte of traffic even if it updates 10 million database rows.
* **Cons:** **Non-Deterministic Anomalies.** If the SQL query contains non-deterministic functions (e.g., `LIMIT 1` without explicit `ORDER BY`, `NOW()`, `RAND()`, `UUID()`), the statement will yield divergent database states when run independently on the primary and the replica.

##### B. Row-Based Replication (RBR)
* **Mechanics:** The primary completely ignores the executing SQL statements. Instead, it logs the exact byte-level changes applied to individual table rows (e.g., "Row 542 changed `balance` from 100.00 to 105.00").
* **Pros:** Complete deterministic safety. Replicas simply overwrite bytes directly, entirely bypassing non-deterministic SQL execution paths. It is the default in modern MySQL.
* **Cons:** High log volume. Modifying 1 million rows generates 1 million separate row-event records inside the binary log, easily exhausting network throughput and triggering write amplification on replicas.

##### C. Mixed-Based Replication (MBR)
* **Mechanics:** A hybrid engine (like MySQL's `MIXED` mode) defaults to fast Statement-Based Replication for standard queries. However, if the query parser detects a non-deterministic function (`NOW()`, trigger actions, auto-increment side-effects), it automatically upgrades *only* that specific transaction to Row-Based Replication to preserve absolute data safety.

#### 3. Physical vs. Logical Replication (PostgreSQL Internals)
In PostgreSQL, database replication is split into two completely different paradigms:

* **Physical Replication (Streaming WAL):**
  - **Mechanics:** Ships raw byte-for-byte binary changes of disk blocks (write-ahead log segments) from the primary to the secondary. Replicas run in continuous Recovery Mode, applying these block changes directly.
  - **Trade-off:** Fast and low-overhead, but highly restrictive. Replicas must run the **exact same major OS version, CPU architecture, and PostgreSQL binary version** as the primary. The entire database cluster is copied—you cannot replicate a single database table or schema selectively.
* **Logical Replication:**
  - **Mechanics:** An internal worker thread (Logical Decoder) parses raw binary WAL files on the fly and converts them back into high-level, database-agnostic logical replication events (e.g., "Insert into Table X values Y"). These events are streamed using a Pub/Sub model.
  - **Trade-off:** High CPU overhead to decode logs, but provides extreme flexibility. Enables replication between **different database engines or PostgreSQL major versions** (crucial for zero-downtime upgrades). Allows selective table replication and custom data transformations during transit.

```
Physical vs Logical Replication Flow:

[Writes] ──► [Primary WAL] 
                  │
                  ├─────► (Stream Raw Bytes) ─────► [Replica Heap Disk] (Physical)
                  │
                  └─────► [Logical Decoder] ──► (DML Events) ──► [Target Table] (Logical)
```

```
[Client Write] ──► [Primary DB] ──► [Writes to WAL Buffer] ──► [Flush to Primary WAL Disk]
                                                                      │
                                                           (Log Sender Thread)
                                                                      │
                                                           (Encrypted TCP Network)
                                                                      │
                                                                      ▼
[Client Read]  ◄── [Replica DB] ◄── [Apply Thread] ◄── [Relay Log Disk] ◄── (Log Receiver Thread)
```

---

## 3. Consensus Algorithms & High Availability Failovers (Raft vs. Paxos & Split-Brain)

When a primary database crashes, the cluster must execute a failover—promoting a replica to become the new primary to maintain write availability. Doing this safely is incredibly complex.

### 1. The Split-Brain Disaster & Fencing Tokens

If a network partition cuts off communication between database nodes, but leaves both partitions connected to separate client bases:
1. Replicas in Partition B assume the primary in Partition A is dead and promote replica B to be the new primary.
2. The system now has **two active primaries** accepting conflicting writes simultaneously.
3. Once the partition heals, reconciling divergent data logs creates severe data corruption and transactional loss.

```
       Partitions Isolated by Network Split
       [Clients Group A]              [Clients Group B]
              │                              │
              ▼                              ▼
        [Primary Node A]    ✖   ✖   ✖  [Secondary Node B]
       (Can't reach B)     Network     (Assumes A is dead,
                            Split       promotes itself!)
```

#### The Mitigation: Fencing Tokens (Epoch / Generation Numbers)
Even with consensus, an old primary node might undergo a long Stop-the-World garbage collection (GC) pause, lose its lease, get demoted, and wake up thinking it is still the authoritative leader. This is called a **Zombie Leader**.

To prevent a zombie leader from writing stale data or sending invalid API commands:
* **Generational Epochs:** Every time a new primary election occurs, the cluster increments a global, monotonically increasing integer called the **Epoch Number** (or Term in Raft).
* **The Fencing Lock:** Downstream shared services (like network storage, or shared caching systems) are configured to register the current Epoch. When a client performs a write, it attaches its current fencing token:
  $$\text{Write Request} = \{\text{Data}, \text{Epoch: } 12\}$$
* **Validation:** If the shared storage has already accepted a write from Epoch 13, it will instantly reject any incoming write from the zombie leader holding Epoch 12, protecting transactional history.

---

### 2. Built-in Consensus Protocols: Raft vs. Paxos

Modern high-availability database engines do not rely on fragile external monitoring scripts. They use mathematical consensus protocols to ensure that all database replicas agree on a single sequence of state transitions.

```
┌────────────────────────────────────────────────────────┐
│ Comparison of Database Consensus Protocols             │
├─────────────────┬───────────────────┬──────────────────┤
│ Attribute       │ Raft              │ Paxos (Multi)    │
├─────────────────┼───────────────────┼──────────────────┤
│ Complexity      │ Low (Easier)      │ High (Very Hard) │
│ State Model     │ Single Leader     │ Symmetric Peer   │
│ Concurrency     │ Sequential Pipeline│ Out-of-Order OK │
│ Popular Systems │ Consul, Etcd      │ Spanner, Cassandra│
└─────────────────┴───────────────────┴──────────────────┘
```

#### A. The Raft Consensus Protocol
Raft decomposes consensus into three independent sub-problems:

##### 1. Leader Election
* Nodes can be in one of three states: **Leader**, **Follower**, or **Candidate**.
* Followers expect periodic `AppendEntries` heartbeats from the leader. If a follower detects a heartbeat timeout, it increments its election **Term** (Epoch) and transitions to the Candidate state.
* The Candidate votes for itself and requests votes from other nodes. To prevent split votes where multiple followers candidates split the ballot equally, Raft enforces **Randomized Election Timeouts** (e.g., each node waits a random period between 150ms and 300ms before triggering an election).
* To win the election, a Candidate must receive a strict majority of votes from the entire cluster.

##### 2. Log Replication
* All writes are routed to the elected Leader.
* The Leader appends the write to its local log and sends an `AppendEntries` RPC to all followers.
* The followers append the write to their local WAL and return success to the leader.
* Once the Leader receives write confirmations from a **strict majority of nodes (Quorum)**, it considers the log entry **Committed**. The leader then applies the entry to its local state machine (database engine) and returns success to the client, piggybacking the commit confirmation to the followers in subsequent heartbeats.

##### 3. Safety (Leader Completeness Property)
Raft guarantees that if a log entry is committed in a given term, that entry will be present in the logs of the leaders for all higher-terms. To enforce this, a follower will **reject a candidate's vote request** if the candidate's log is less up-to-date than the follower's own log (based on LSN/Index and Term).

---

#### B. The Paxos Consensus Protocol
Paxos is a symmetric, peer-to-peer consensus model. Unlike Raft, Paxos does not mathematically require a single permanent leader; nodes can propose values concurrently.

##### Multi-Paxos Phases:
1. **Phase 1a (Prepare):** A Proposer selects a unique, increasing proposal number `N` and broadcasts a `Prepare(N)` request to a majority of Acceptors.
2. **Phase 1b (Promise):** If an Acceptor receives `Prepare(N)` with `N` greater than any preparation number it has seen, it promises never to accept a proposal numbered less than `N`. It returns the highest-numbered proposal it has already accepted.
3. **Phase 2a (Accept):** If the Proposer receives promises from a majority of Acceptors, it sends an `Accept(N, Value)` request to those Acceptors.
4. **Phase 2b (Accepted):** If an Acceptor receives `Accept(N, Value)`, it accepts the proposal unless it has already promised to ignore it. It registers the value and broadcasts the acceptance to all Learners (the database replicas).

* **Why Spanner/CockroachDB use Multi-Paxos/Multi-Raft:** These modern distributed databases divide their keyspace into small segments (ranges). Instead of running a single consensus ring for the entire database cluster, they run thousands of independent **Paxos/Raft groups at the individual range level (Range Partitions)**, allowing ultra-high parallel throughput.

---

### 3. Comparison with Legacy Orchestrators (Patroni / Redis Sentinel)
It is crucial to distinguish between databases with **built-in consensus** vs. legacy databases requiring **external orchestration failover**:

* **Built-In Consensus (Raft/Paxos):** Systems like etcd, CockroachDB, or Cassandra handle replication, leader elections, and node failures directly in their native binary. There are no external helper scripts.
* **External Orchestration (PostgreSQL Patroni / Redis Sentinel):** Relational databases like PostgreSQL were written before distributed systems became standard. They do not know about other nodes.
  - To support HA, developers run a helper tool (e.g., **Patroni**) alongside the database.
  - Patroni relies on a separate, external Distributed Consensus Store (DCS) like **etcd** to maintain a leader key lease.
  - If Patroni on the primary node loses its connection to etcd, Patroni shuts down PostgreSQL immediately, and the remaining Patroni nodes promote a replica to primary.

---

### 4. CAP Theorem Trade-offs: CP vs. AP Failovers

When a network partition occurs, the system's architects must choose between **Consistency (CP)** or **Availability (AP)**:

* **CP (Consistency / Partition Tolerance) - e.g., Raft/Paxos Databases:**
  - The partition containing the minority of nodes instantly detects it cannot achieve quorum. It **rejects all incoming writes**, prioritizing data correctness and preventing split-brain.
  - *Trade-off:* High data integrity, but write availability is lost for clients connected to the minority partition.
* **AP (Availability / Partition Tolerance) - e.g., Cassandra / DynamoDB:**
  - Nodes in both partitions continue accepting reads and writes. The databases diverge.
  - *Trade-off:* High write availability, but requires complex conflict resolution schemes when the partition heals (e.g., **CRDTs (Conflict-Free Replicated Data Types)**, **Vector Clocks**, or **Last-Write-Wins (LWW)** which discards data based on NTP timestamps).

---

## 4. Sharding Routing Architectures

When a database is sharded across $K$ physical instances, routing client requests to the correct target node requires an explicit coordinator tier:

```
        Client-Side Routing                    Proxy-Based Routing                   Coordinator (Gossip) Routing
   ┌───────────────────────────┐         ┌───────────────────────────┐         ┌──────────────────────────┐
   │ Client (Direct mapping in │         │ Client (Sends to proxy)   │         │ Client (To any random    │
   │ config or smart library)  │         └─────────────┬─────────────┘         │ node in cluster)         │
   └──────┬─────────────┬──────┘                       │                       └─────────────┬────────────┘
          │             │                              ▼                                     │
          ▼             ▼                        [Router Proxy]                              ▼
     [Shard A]     [Shard B]               (Vitess, PgBouncer, Envoy)                  [Target Node]
                                                  /          \                         (Gossip ring routes
                                                 ▼            ▼                         internally)
                                             [Shard A]    [Shard B]
```

### 1. Client-Side Routing (Smart Client)
* **Mechanism:** The application code or driver (e.g., Cassandra smart client) contains the sharding directory and hashing logic. It hashes the sharding key locally and establishes a direct TCP connection to the correct target shard database.
* **Pros:** Maximum performance; eliminates network hops and proxy latency.
* **Cons:** Thin clients become complex; database topology updates require redeploying or dynamically updating configuration states in all application instances.

### 2. Proxy-Based Routing
* **Mechanism:** Applications connect to a pool of thin, high-performance middleware proxies (e.g., YouTube's **Vitess** for MySQL, **PgBouncer** / **Citus** for PostgreSQL). The proxy parses incoming SQL statements, extracts the sharding key, hashes it, and forwards the query to the correct shard.
* **Pros:** Complete abstraction. Application code remains clean, treating the proxy as a single massive monolithic database instance. Schema changes and topology scaling are managed centrally inside the proxy.
* **Cons:** Adds an extra network hop (Client $\to$ Proxy $\to$ Shard Database) and introduces proxy compute/memory overhead.

### 3. Coordinator / Gossip-Driven Routing
* **Mechanism:** Clients can connect to *any* random database node in the cluster. Every node participates in a Peer-to-Peer **Gossip Protocol** to share cluster maps. If a node receives a query for a key it doesn't own, it proxies the request internally to the correct neighbor node and returns the result (e.g., Apache Cassandra, DynamoDB).
* **Pros:** High fault tolerance; simple client architecture.
* **Cons:** High internal cluster network traffic due to continuous inter-node gossiping and internal query forwarding.

---

## 5. Database Sharding (Write Scale)

Sharding is the process of breaking a single massive table into smaller, disjoint subsets (shards) and distributing them across independent server nodes.

### Sharding Strategies
1. **Range-Based Sharding:**
   * *Mechanism:* Shards are split by value boundaries (e.g., User IDs 1-1M on Shard A, 1M-2M on Shard B).
   * *Risk:* Hotspots (e.g., if new users are highly active, Shard B receives all write traffic).
2. **Hash-Based Sharding:**
   * *Mechanism:* Applies a hash function to a sharding key (e.g., `hash(user_id) % num_shards`) to determine the target node.
   * *Pros:* Evenly distributes traffic across shards, avoiding hotspots.
   * *Cons:* Resharding (adding new shards) is expensive as it alters the modulo calculation, requiring massive data movement.
3. **Consistent Hashing:**
   * *Mechanism:* Uses a logical hashing ring where both nodes and data keys are mapped.
   * *Pros:* Adding or removing database nodes only requires moving a tiny fraction of keys, resolving the standard hash-sharding scale issue.

---

## 6. Advanced Sharding Challenges

### A. The Resharding & Rebalancing Nightmare
As a sharded database grows, you will eventually deplete disk/CPU capacity on your existing shards and must add new physical nodes to the cluster.
1. **The Modulo Sharding Trap**: If your cluster uses simple modulo hashing (`hash(user_id) % K`), adding a single new database server (scaling from $K$ to $K+1$) changes the base of the modulo calculation. This forces **almost 90% of all existing keys to change their destination shard**, triggering massive, cluster-wide network and disk IO thrashing as rows migrate between servers, frequently taking down the production system.
2. **The Consistent Hashing Solution**: By mapping servers and keys to a logical ring, adding a new node $S_{new}$ only intercepts keys belonging to its clockwise neighbor. To make the distribution perfectly even, **Virtual Nodes (vnodes)** are used: each physical server represents hundreds of scattered virtual points on the ring. Adding a server redistributes only $1/N$ of the total dataset, enabling smooth, zero-downtime background rebalancing.

```
       Consistent Hashing Ring (0 - 2^32)
       
                [Node A (vnode 1)]
               /                  \
   [Node C]───*                    *───[Node B]
             /                      \
            *                        *
             \                      /
              *────────────────────*
                 [Node A (vnode 2)]
```

### B. Scatter-Gather Queries (The Latency Multiplier)
* **Single-Shard Query**: Querying by the sharding key (e.g., `WHERE tenant_id = 'netflix'`) allows the router to target a single physical node immediately. Time complexity is isolated.
* **Scatter-Gather (Non-Sharded Query)**: If you query a column that is *not* part of the sharding key (e.g., `WHERE email = 'alex@example.com'`), the database router does not know which shard owns the row.
  1. The router is forced to **broadcast (scatter)** the query to **all $K$ shards** in parallel.
  2. Each shard runs the query, consumes local CPU/RAM index blocks, and returns matching rows.
  3. The router collects (gathers) all results, dedupes/merges them, and returns them to the application.
* **The Tail Latency Problem**: The latency of a scatter-gather query is determined by the **slowest responding shard in the entire cluster (the p99.9 bottleneck)**. If one shard is undergoing a GC pause or background backup, the entire user query halts, cascading into severe tail-latency spikes.

---

## 7. Popular Interview Questions & High-Impact Answers

### Q1: What is "Replication Lag" and what anomalies can it cause in an application?
* **Answer:** Replication lag is the delay between a write committing on the primary node and that write being applied to a secondary read replica. This can cause **Read-Your-Own-Writes anomalies**: if a user updates their profile (writing to primary) and immediately refreshes the page (reading from a lagging replica), they will see their old profile state, leading them to think the action failed. To prevent this, route critical user-owned reads to the primary node for a short period after a write.

### Q2: What is a Sharding Key, and how do you choose a good one?
* **Answer:** A sharding key is the column used to determine how data is distributed across shards. A good sharding key must:
  1. Have high cardinality (large number of unique values) to prevent data skew.
  2. Produce an even distribution of reads and writes to avoid hotspotting (e.g., avoiding keys like timestamps or incremental IDs).
  3. Align with common query patterns to ensure most queries target a single shard instead of performing highly expensive "scatter-gather" queries across all nodes.

### Q3: What are the drawbacks/complexities of sharding a database?
* **Answer:** Sharding introduces massive engineering trade-offs:
  1. **Joins are lost:** Joining tables across different physical servers (cross-shard joins) is extremely slow or unsupported.
  2. **Referential Integrity is lost:** Enforcing unique constraints or foreign keys across physical servers is impossible without expensive distributed locks.
  3. **Operational Complexity:** Backups, index management, schema migrations, and rebalancing database nodes become significantly harder to orchestrate.

### Q4: What is a "Split-Brain" scenario in database replication, and how does Quorum solve it?
* **Answer:** Split-Brain is a catastrophic failure state during a network partition where two nodes simultaneously assume they are the single, authoritative primary. Both accept conflicting write traffic, permanently corrupting the transaction log history. **Quorum** mitigates this by enforcing that a node can only act as primary if it can communicate with a strict majority of nodes ($\lfloor N/2 \rfloor + 1$). During a partition, the isolated primary detects it has lost majority connection, steps down to read-only, and only the partition holding the majority of nodes is allowed to elect a new primary, guaranteeing single-leader safety.

### Q5: Compare Client-Side vs. Proxy-Based database sharding routing. When would you prefer one over the other?
* **Answer:**
  - **Client-Side Routing:** The application contains sharding map logic. Highly performant ($O(1)$ lookup and direct connection to shard), but couples the client code directly to database topology, making updates difficult across massive service pools. Prefer for ultra-low latency requirements and stable, slowly changing database clusters.
  - **Proxy-Based Routing:** Clients connect to a centralized proxy (e.g., Vitess/PgBouncer) which abstracts the sharding topology. Simplifies client development (application treats database as one single massive monolith), but introduces an extra network hop and proxy processing overhead. Prefer for massive, dynamic, rapidly scaling microservice fleets where database operational abstraction is critical.

### Q6: [Struggle Question] How do you execute high-performance searches on columns that are not part of the sharding key, avoiding the performance degradation of Scatter-Gather queries?
* **Answer:** 
  1. **Secondary Index Tables (Mapping Tables)**: Maintain a separate, highly indexable lookup table (or Redis cache) that maps the non-sharded attribute to the sharding key (e.g., mapping `email ──► tenant_id`). The client performs a sub-millisecond $O(1)$ lookup to find the `tenant_id`, then runs a single-shard query targeting that specific database node.
  2. **Dual-Key Sharding (Replication)**: Write the same record to two separate indexes or clusters sharded differently. For example, store user data on Cluster A sharded by `tenant_id` (for operational tenant flows) and replicate/sync it to Cluster B sharded by `user_id` (for user-specific authentication flows).
  3. **External Search Engine Indexing**: Stream database writes via Change Data Capture (CDC) into an external search cluster like Elasticsearch or Opensearch. Direct all complex search queries, multi-column filters, and text lookups to Elasticsearch, keeping the relational database isolated to direct single-shard OLTP writes.

