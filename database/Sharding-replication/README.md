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

---

## 3. Database Sharding (Write Scale)

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

## 4. Popular Interview Questions & High-Impact Answers

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
