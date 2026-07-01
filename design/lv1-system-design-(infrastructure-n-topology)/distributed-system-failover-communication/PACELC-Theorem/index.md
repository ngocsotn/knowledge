# PACELC Theorem

The CAP theorem only applies when a network partition is actively occurring. In normal operations (no partitions), systems still face trade-offs between **Latency (L)** and **Consistency (C)**.

The **PACELC** theorem addresses this:
* If there is a **P**artition, how does the system trade off **A**vailability vs **C**onsistency?
* **E**lse (normal operations), how does the system trade off **L**atency vs **C**onsistency?

### PACELC Classification Examples
* **MongoDB (PC/EC):** Under partitions, chooses Consistency (C). Under normal operations, also chooses Consistency (C), adding write latency to replicate data.
* **Cassandra (PA/EL):** Under partitions, chooses Availability (A). Under normal operations, chooses Latency (L), returning data from the nearest node instantly while replicating in the background.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: Why is a CA (Consistent + Available) system impossible in distributed databases?
* **Answer:** A "CA" system assumes a perfect network that never experiences packet drops, delays, or partitions. In reality, network failures are physically inevitable. If a network partition occurs and Node A cannot talk to Node B, and a write request hits Node A, the system is forced to make a choice: either accept the write on Node A (violating consistency, since Node B will be out-of-sync, choosing AP), or reject the write (violating availability, choosing CP). A distributed system must support Partition Tolerance, leaving only the choice between CP and AP.

### Q2: What is "Eventual Consistency" and how is it resolved in AP databases?
* **Answer:** Eventual Consistency is a consistency model where replicas are allowed to drift and contain conflicting data temporarily during network partitions, but are guaranteed to eventually reconcile and become identical once communication resumes and no new writes are made. AP databases (like Cassandra) resolve write conflicts during reconciliation using algorithms like **LWW (Last-Write-Wins)** based on timestamps, or **CRDTs (Conflict-Free Replicated Data Types)** which merge state mathematically without conflicts.

### Q3: Explain PACELC theorem using Cassandra and PostgreSQL as examples.
* **Answer:** The PACELC theorem expands CAP by analyzing normal, partition-free operations.
  * **PostgreSQL (Clustered - PC/EC):** If a **P**artition occurs, it chooses **C**onsistency over Availability (PC). **E**lse, in normal operation, it chooses **C**onsistency over Latency (EC), forcing writers to wait for replica logs to sync before returning success.
  * **Cassandra (PA/EL):** If a **P**artition occurs, it chooses **A**vailability (PA). **E**lse, in normal operation, it chooses low **L**atency (EL), performing fast local reads/writes and updating other nodes asynchronously.

## Interview Questions & Answers

### Q1: What is a PACELC-compliant real-world database example?
- **Answer:** **Apache Cassandra** is highly configurable. Under a partition (P), you can prioritize Availability (A) over Consistency (C). Else (E), you can choose low Latency (L) over Consistency (C) by configuring local write/read consistency to `ONE` or `QUORUM` to bypass synchronous cross-datacenter page replication.
