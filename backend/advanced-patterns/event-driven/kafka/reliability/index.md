# Apache Kafka: Reliability & Data Guarantees

Under the hood guide covering ISR, Replication, and Exactly Once Semantics.

## Reliability Patterns

### 1. ISR (In-Sync Replicas)
The subset of follow partitions that are fully caught up and synchronized with the partition leader's latest write logs.

### 2. Replication Factor
The number of physical broker instances storing identical partition logs. A replication factor of 3 ensures two broker failures can occur without data loss.

### 3. Exactly Once Semantics (EOS)
A combined guarantee of **idempotent producers** (deduplicated sequence numbers) and **transactional coordinators** (atomic multi-partition writes) ensuring messages are processed exactly once.

## Interview Questions & Answers

### Q1: What is the difference between acks=0, acks=1, and acks=all in Kafka?
- **Answer:**
  - **`acks=0`:** The producer considers the write successful as soon as it sends the network packet. Zero durability guarantees; highest throughput.
  - **`acks=1`:** The producer waits for confirmation from the partition Leader broker only. If the leader crashes before replicas sync, data is lost.
  - **`acks=all` (or `-1`):** The producer waits for confirmation from the leader AND all In-Sync Replicas (ISR). Highest durability; slower write latency.
