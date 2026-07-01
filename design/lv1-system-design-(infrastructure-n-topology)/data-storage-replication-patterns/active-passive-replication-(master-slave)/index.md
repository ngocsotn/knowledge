# Active-Passive (Master-Slave) Replication

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

## Interview Questions & Answers

### Q1: What is Replication Lag, and how do you protect against "Read-Your-Own-Writes" inconsistency?
- **Answer:** Replication Lag occurs because replica nodes receive master changes asynchronously over the network. If a user writes to master and immediately reads from a lagging replica, they see stale data, assuming their write was lost.
- **Mitigation:** **Read-Your-Own-Writes Gating:** Configure the application to route reads to the **Master** node for a short window (e.g., 5 seconds) immediately following any write operation executed by that specific user, ensuring they always see their own updates.
