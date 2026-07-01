# SQL Database Sharding & Replication Topologies

Comprehensive guide on horizontal sharding, partition routing, split-brain, and consensus replication.

* **Vertical Scaling (Scale-up):**
  * *Definition:* Adding more power (CPU, RAM, SSDs) to a single database server.
  * *Limits:* Finite hardware capacity limits, high cost curves, and single point of failure (SPOF) risks.
* **Horizontal Scaling (Scale-out):**
  * *Definition:* Spreading the database load across multiple cheap servers (nodes) in a cluster.
  * *Mechanism:* Leverages Replication (for reads) and Sharding (for writes).

---

## Replication Formats & Data Propagation Mechanics
- **Statement-Based Replication (SBR):** Replicates raw SQL statements. (Slightly smaller logs, but vulnerable to non-deterministic functions like `NOW()`).
- **Row-Based Replication (RBR):** Replicates actual byte-level changes. (Safe and consistent, but generates massive log files).

## Split-Brain and Majority Quorum
During a network partition, isolated nodes may attempt to promote their own Primary node. To prevent split-brain data corruption, clusters utilize **Majority Quorum Consensus** (e.g., Raft/Paxos), requiring nodes to achieve more than 50% approval before electing a leader or confirming writes.
