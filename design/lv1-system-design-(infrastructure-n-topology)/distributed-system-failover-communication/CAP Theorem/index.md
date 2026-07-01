# CAP Theorem

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

## Interview Questions & Answers

### Q1: Why is "Partition Tolerance" non-optional in distributed systems?
- **Answer:** Hardware reality. Physical networks will inevitably experience packet dropouts, switch failures, or cut fibers, isolating nodes. Because network partitions are guaranteed to occur eventually, you cannot choose "CA" (Consistency + Availability without Partition Tolerance). You must configure your system to choose either **Consistency** (CP - fail-closed) or **Availability** (AP - return stale data) when a partition occurs.
