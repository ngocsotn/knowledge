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
Data replication operates under different physical formats to propagate changes down the log pipeline:

#### 1. Statement-Based Replication (SBR)
* **How it works:** The primary logs and sends the exact SQL queries executed (e.g., `UPDATE users SET points = points + 10 WHERE signup_date > '2026-01-01'`).
* **Pros:** Highly compact log size; uses minimal network bandwidth.
* **Cons:** Fragile to non-deterministic functions. Statements utilizing functions like `NOW()`, `RAND()`, or relying on specific auto-increment order can yield completely different data states when run on replicas.

#### 2. Row-Based Replication (RBR) / Write-Ahead Log (WAL) Shipping
* **How it works:** The primary writes the exact byte-level disk block changes (row modifications) directly from its **Write-Ahead Log (WAL)** and ships these logs to the replicas.
* **Pros:** 100% deterministic and safe. Replicas accurately mirror exact bytes, completely bypassing non-deterministic function bugs.
* **Cons:** High log volume and network usage (e.g., a query modifying 1 million rows generates 1 million separate row-change records in the WAL instead of one short SQL statement).

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

## 3. High Availability Failovers: Split-Brain & Quorum

When a primary database crashes, a replica must be promoted to the new primary to maintain write availability.

### The Split-Brain Disaster
If a network partition cuts off communication between the primary and replica nodes, but leaves both partitions connected to separate client bases:
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

### The Mitigation: Quorum (Majority Rule Consensus)
Modern stateful databases (e.g., MongoDB, CockroachDB, PostgreSQL with Patroni, Elasticsearch) enforce **Quorum-based Consensus** (Raft or Paxos algorithms) to eliminate split-brain:
* **The Formula:** A primary can only accept writes if it remains in communication with a strict majority of nodes in the cluster:
  $$\text{Quorum Node Count} \ge \lfloor N/2 \rfloor + 1$$
* **Under a Partition:**
  - If a 3-node cluster splits into $\{A\}$ and $\{B, C\}$:
    - Node $A$ is isolated (1 node out of 3 is $< 50\%$). It detects it has lost quorum, steps down immediately, and shifts to read-only mode.
    - Nodes $B$ and $C$ can communicate (2 nodes out of 3 is $> 50\%$, satisfying quorum). They safely hold an election, promote Node $B$ to primary, and continue accepting writes.
    - Result: Only one write-capable primary exists at any time, preventing data corruption.

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

## 6. Popular Interview Questions & High-Impact Answers

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
