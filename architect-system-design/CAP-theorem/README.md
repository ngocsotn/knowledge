# The CAP & PACELC Theorems

Comprehensive interview study guide covering the CAP Theorem, PACELC extensions, and real-world distributed consistency trade-offs.

---

## 1. The CAP Theorem

The CAP Theorem states that any distributed data store can guarantee at most two of three core attributes simultaneously:

```
                       Consistency (C)
                       /             \
                      /               \
                     /    Partition    \
                    /    Tolerance (P)  \
                   /                     \
        Availability (A) ───────────── (None)
```

1. **Consistency (C):** Every read receives the most recent write or an error. (Single, unified view of data across all nodes).
2. **Availability (A):** Every non-failing node returns a non-error response for every request (even if the data is stale or out-of-sync).
3. **Partition Tolerance (P):** The system continues to operate despite an arbitrary number of messages being dropped or delayed by the network between nodes.

---

## 2. Partition Tolerance is Not Optional

In a real-world network, network partitions (dropped packets, hardware faults, cable failures) are inevitable. Therefore, **you cannot choose "CA" (Consistency + Availability without Partition Tolerance) in a distributed system.**

When a network partition (P) occurs, a distributed system must choose between:

### 1. CP (Consistency under Partition)
* **Behavior:** The system refuses to accept writes or reads on nodes that cannot communicate with the primary database, returning errors to preserve absolute data consistency.
* **Usage:** Financial ledgers, double-entry bookkeeping, medical databases (e.g., PostgreSQL clusters, MongoDB).

### 2. AP (Availability under Partition)
* **Behavior:** Every node continues to accept reads and writes, returning whatever data it holds, even if it is stale. Replicas will reconcile differences once the partition heals.
* **Usage:** Social media feeds, comment sections, shopping carts, DNS lookups (e.g., Cassandra, DynamoDB).

---

## 3. The PACELC Theorem (The Real-World Extension)

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
