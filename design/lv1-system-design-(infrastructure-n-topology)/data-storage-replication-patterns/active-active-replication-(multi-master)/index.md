# Active-Active (Multi-Master) Replication

All nodes in the cluster act as master nodes, simultaneously accepting read and write requests from clients.
- **Pros:** Highest write throughput; local write latency optimization.
- **Cons:** Extremely complex synchronization; risk of write conflict anomalies. Requires conflict resolution engines like Conflict-Free Replicated Data Types (CRDTs) or Last-Write-Wins (LWW).

## Interview Questions & Answers

### Q1: What is the Split-Brain problem in Active-Active replication?
- **Answer:** Split-brain occurs when a network partition cuts a cluster in half, isolating masters. If both isolated sections continue accepting writes independently, their states will diverge. When the partition heals, reconciling the competing, conflicting records becomes mathematically difficult without losing data, risking ledger or state corruption.
