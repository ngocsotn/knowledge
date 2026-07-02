# SQL Database Sharding & Replication Topologies

Comprehensive study guide covering database replication formats, scaling topologies (Active-Passive, Active-Active, Consensus), database sharding (horizontal partitioning), consistent hashing, and split-brain mitigation.

---

## 1. Replication Topologies & Propagation Mechanics

Replication copies the same data across multiple physical database nodes. This guarantees **High Availability** (survives node failure) and **Read Scalability** (delegating reads to secondary nodes).

```
                 Active-Passive (Read Scaling)
            ┌───────────────┐
            │ Master (Write)├──────────────┐
            └───────┬───────┘              │ Asynchronous
                    │                      ▼ WAL Shipping
                    │ Synchronous   ┌───────────────┐
                    ▼               │  Slave (Read) │
            ┌───────────────┐       └───────────────┘
            │  Slave (Read) │
            └───────────────┘

                 Consensus-Based (Majority Quorum)
                  ┌────────────────────────┐
                  │      Client Write      │
                  └───────────┬────────────┘
                              │ (Quorum write: 2 of 3 nodes)
                    ┌─────────┼─────────┐
                    ▼         ▼         ▼
                ┌───────┐ ┌───────┐ ┌───────┐
                │Leader │ │Follower││Follower│ (Paxos/Raft)
                └───────┘ └───────┘ └───────┘
```

### A. Active-Passive (Master-Slave) Replication
* **Writes**: Handled strictly by the designated **Primary (Master)** node.
* **Reads**: Delegated across one or more **Standby (Slave/Replica)** nodes.
* **Synchronization Styles**:
  - **Synchronous**: The Master writes to WAL, ships the log to the slave, waits for the slave to confirm writing to its disk, and *then* returns success to the client.
    - *Pros*: Zero data loss if the master crashes.
    - *Cons*: Write latency equals the slowest network round-trip; writes fail if a single slave goes down.
  - **Asynchronous**: The Master returns success to the client immediately after writing to its local WAL, shipping logs to slaves in the background.
    - *Pros*: Extremely low write latency; resilient to slave outages.
    - *Cons*: Risk of data loss (Replication Lag) if the master crashes before shipping logs.
  - **Semi-Synchronous**: The Master waits for at least **one** slave to acknowledge receipt of the WAL record before committing the transaction.

### B. Active-Active (Multi-Master) Replication
* **Writes/Reads**: Every node in the cluster can accept both reads and writes.
* **Challenge**: Highly vulnerable to write conflicts and sync delays. Conflict resolution requires **Last-Write-Wins (LWW)** (vulnerable to clock drift), **Conflict-Free Replicated Data Types (CRDTs)**, or complex application-level merge rules.

### C. Consensus-Based Replication (Paxos/Raft)
Used by modern cloud databases (e.g., CockroachDB, Spanner). Writes must be acknowledged by a **Majority Quorum** ($>50\%$) of nodes before committing, eliminating single-point-of-failure sync vulnerabilities.

### D. Replication Log Formats
* **Statement-Based Replication (SBR)**: Ships raw SQL query strings.
  - *Cons*: Fails on non-deterministic queries (e.g., `WHERE created_at = NOW()`).
* **Row-Based Replication (RBR)**: Ships the actual byte-level changes of rows.
  - *Cons*: Generates massive WAL traffic (e.g., a bulk update of 1 million rows generates 1 million separate row-change records).
* **Logical WAL Shipping**: Ships raw storage engine Write-Ahead Log bytes directly.

---

## 2. Database Sharding (Horizontal Partitioning)

When a dataset grows too large to fit on a single physical disk or server, it must be **Sharded**—split horizontally across multiple independent database nodes.

```
                         Sharding Topologies
     ┌──────────────────────────────────────────────────────────┐
     │                      Incoming Write                      │
     └────────────────────────────┬─────────────────────────────┘
                                  │
                  Calculate Sharding Key (e.g., hash(user_id))
                                  ├─────────────────────────────┐
                        Range-Based                     Hash-Based
                   (e.g., ID 0 - 100000)             (e.g., hash % N)
                          ┌───────┴───────┐              ┌──────┴──────┐
                          ▼               ▼              ▼             ▼
                      ┌───────┐       ┌───────┐      ┌───────┐     ┌───────┐
                      │Shard 1│       │Shard 2│      │Shard 1│     │Shard 2│
                      └───────┘       └───────┘      └───────┘     └───────┘
```

### A. Sharding Topologies
1. **Range-Based Sharding**: Maps keys to shards based on discrete value ranges (e.g., Shard A stores User IDs 1–100,000; Shard B stores 100,001–200,000).
  - *Cons*: Leads to severe write hotspots on the newest shard if IDs are auto-incrementing.
2. **Hash-Based Sharding**: Applies a cryptographic hash function to the sharding key and computes the target node modulo the total number of shards ($N$):
  $$\text{Shard ID} = \text{Hash}(\text{Key}) \pmod N$$
  - *Cons*: Adding or removing a shard ($N \to N+1$) invalidates the entire modulo ring, forcing the migration of up to $90\%$ of overall database rows.
3. **Directory-Based Sharding**: A centralized lookup service maps individual sharding keys to physical node locations. Introduces a query routing hop and a single point of failure (SPOF).

### B. Consistent Hashing
To prevent massive data reshuffling when scaling shards up or down, databases utilize **Consistent Hashing**:

```
                       Consistent Hashing Ring
                            Node A (0)
                           ┌─────────┐
                        .-'           '-.
                     .-'                 '-.
        Node C (240)│                       │ Node B (120)
                    │     Key (hash=150)    │
                     '-.   *Clockwise*   .-'
                        '-.           .-'
                           '-.     .-'
                              '---'
```

* **The Hash Ring**: Nodes and keys are hashed onto a circular space (e.g., $0$ to $2^{32}-1$).
* **Routing**: A key's hash is mapped to the ring, and the write routes to the first physical node encountered walking **clockwise**.
* **Scaling**: When adding a new node, only a fraction of keys ($\frac{1}{N+1}$) must be migrated.
* **Virtual Nodes (VNodes)**: To prevent statistical load imbalance (hotspots), each physical server is mapped to multiple logical points on the ring (e.g., 200 vnodes per server), ensuring even data distribution.

---

## 3. Split-Brain & Consensus Fencing

During a network partition, isolated database partitions can lose connection to each other but remain accessible to different clients.

### A. The Split-Brain Catastrophe
If an Active-Passive cluster partitions, Standby nodes in Partition B may believe the Primary node in Partition A is dead and elect a new Primary. If both Primaries accept writes, the database state diverges, causing irreversible data corruption.

```
       [ Client A ]                             [ Client B ]
            │                                         │
            ▼ (Write)                                 ▼ (Write)
     ┌─────────────┐                           ┌─────────────┐
     │  Primary A  │    ◄─── NETWORK ───►      │  Primary B  │
     │ (Acme Corp) │         PARTITION         │ (Acme Corp) │
     └─────────────┘                           └─────────────┘
  *Diverged data, inconsistent accounts, checksum mismatch on rejoin*
```

### B. Mitigation Controls
1. **Majority Quorum Math**: To proceed with a leader election or confirm any write, a partition **MUST** contain a strict majority of the total cluster nodes:
  $$\text{Quorum} \ge \left\lfloor \frac{N}{2} \right\rfloor + 1$$
  This mathematical rule guarantees that only a single partition can ever act as the active cluster (since a network partition can only ever contain at most one majority segment).
2. **Epoch Fencing (Generation Numbers)**: Every leader election increments a global `epoch` counter. If a former, partitioned Primary attempts to write to downstream nodes, those nodes reject the write because the incoming write bears an obsolete `epoch` token.
3. **STONITH (Shoot The Other Node In The Head)**: A hard fencing protocol where surviving nodes utilize a physical power management interface (IPMI/PDU) to physically cut power to the partitioned master before promoting a new leader.

---

## 4. Highly Technical Interview Q&As

### Q1: How do you design and scale database sharding for a global high-volume e-commerce platform?
- **Answer**:
  1. **Selecting the Sharding Key**: The sharding key is the most critical decision. For an e-commerce platform, `user_id` or `account_id` is ideal for high cardinality and even hash distribution. Never shard on low-cardinality fields (e.g., `country_code`) or timestamps which create severe hotspots.
  2. **Consistent Hashing with Virtual Nodes**: Route writes using a consistent hashing algorithm. Map physical shards to 100–200 virtual nodes (VNodes) on the hash ring. This statistically distributes user accounts evenly, maintaining a deviation of $<5\%$, and allows seamless scale-out without bulk data redistribution.
  3. **Mitigating Joint/Cross-Shard Limitations**:
     - *Localize Transactions*: Ensure all frequently joined tables (e.g., `users`, `orders`, `order_items`) share the identical sharding key (`user_id`). This ensures all related rows sit physically on the same shard, allowing fast, localized ACID transactions and joins.
     - *Handle Cross-Shard Queries*: For queries spanning multiple shards (e.g., global sales reports), implement an API routing layer (like Vitess/PgBouncer) that queries shards in parallel and aggregates results in memory, or offload these queries entirely to a replicated data lake (e.g., BigQuery/ClickHouse) via Change Data Capture (CDC).

### Q2: What is the database "Split-Brain" scenario, and how do you prevent it in a replicated cluster?
- **Answer**: Split-brain occurs during a network partition when a single database cluster splits into two isolated segments, and each segment acts as if the other is dead. In an Active-Passive cluster, the secondary partition may promote a standby node to primary. If both partitions accept concurrent writes, the database states diverge, causing irreconcilable data corruption upon reconnection.
- **Prevention**:
  1. **Majority Quorum**: Force elections and writes to require an absolute majority of total nodes: $\lfloor \frac{N}{2} \rfloor + 1$. Since only one partition can contain the majority, the minority partition automatically disables writes and enters read-only fail-closed mode.
  2. **Epoch Fencing**: Every leader promotion increments an epoch counter. Downstream storage nodes reject any write carrying an obsolete epoch number from a partitioned leader.
  3. **STONITH**: Use physical power-distribution units (PDUs) to forcefully power down the partitioned master node before promoting a standby to primary.

### Q3: Compare Synchronous, Asynchronous, and Semi-Synchronous replication in terms of write latency, data durability, and split-brain risk.
- **Answer**:
  - **Synchronous**:
    - *Latency*: High. Write latency is bounded by the slowest network round-trip from the primary to the replica.
    - *Durability*: Excellent. Zero data loss on primary crash since the replica is guaranteed to have a physical copy of the WAL.
    - *Split-Brain Risk*: None, but vulnerable to write-starvation (writes halt completely if a replica goes offline).
  - **Asynchronous**:
    - *Latency*: Extremely Low. Bounded only by the primary's local disk WAL write.
    - *Durability*: Risk of data loss. If the primary crashes, transactions written within the "replication lag" window are lost.
    - *Split-Brain Risk*: High. If failover is triggered too early, a replica promoted to primary may have missing records, causing state divergence.
  - **Semi-Synchronous**:
    - *Latency*: Moderate. The primary commits immediately after receiving confirmation that **at least one** replica has appended the transaction to its relay log.
    - *Durability*: High. Safeguards against data loss for single-node crashes since a backup copy is guaranteed to exist.
    - *Split-Brain Risk*: Low, providing an optimal balance of throughput and safety for high-volume enterprise systems.

### Q4: Explain Consistent Hashing ring mechanics and how virtual nodes prevent statistical load imbalance (hotspots).
- **Answer**:
  - **Ring Mechanics**: Consistent Hashing maps both physical database shards and data keys onto a circular hash space ($0$ to $2^{32}-1$). A key's target shard is determined by finding its hash position on the ring and walking clockwise to the first physical shard hash position. When a shard is added or removed, only keys residing between that shard and its predecessor are migrated, minimizing data rebalancing.
  - **Virtual Nodes (VNodes)**: If you only map physical shards directly (e.g., 3 physical servers), random hash distribution will lead to unequal segment sizes on the ring, causing one server to shoulder up to $70\%$ of the data load (statistical hotspot). To prevent this, map each physical server to multiple **Virtual Nodes** (e.g., Server A is mapped to 200 separate, scattered hash keys across the ring: `server_a-1`, `server_a-2`, etc.). This breaks the ring into hundreds of tiny interleaved segments, ensuring statistically uniform data distribution across all physical nodes (load imbalance deviation $<5\%$).

### Q5: How do you achieve read-after-write consistency in an asynchronously replicated Active-Passive database cluster?
- **Answer**: Under asynchronous replication, a client writing to the master and immediately reading from a replica may see stale data due to replication lag. To prevent this, apply three application-level patterns:
  1. **User-Context Routing**: Force any read request modifying or reading a user's *own* profile to route exclusively to the Master database. All other general reads (e.g., browsing other profiles) are routed to replicas.
  2. **Temporal Stickiness**: Following a write, store a temporary key in the user's browser cookie or Redis session (e.g., `just_updated: true`) with a TTL slightly larger than the maximum replication lag (e.g., 5 seconds). During this TTL, the routing layer forces all reads from this user to hit the Master.
  3. **LSN-Based Routing (Pinning)**: When the Master commits a write, it returns the committed transaction's Log Sequence Number (LSN) to the client. When the client reads, the API router includes this LSN. The router checks if the candidate replica's current `page_LSN` is equal to or greater than the transaction LSN. If yes, it queries the replica; if no (the replica is lagging), it falls back to the Master.

