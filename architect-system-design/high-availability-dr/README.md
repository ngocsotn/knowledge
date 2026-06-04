# High Availability and Disaster Recovery (HA/DR)

Comprehensive system design study guide for architecting resilient, fault-tolerant, and geographically distributed architectures that guarantee zero-downtime operations.

---

## 1. Core Metrics: RTO and RPO

When designing Disaster Recovery (DR) strategies, two business metrics define your technical requirements:

```
[Normal Operation] ──(Disaster Event)──> [Downtime & Repair] ──> [Fully Restored]
        |                                        |
        |<─── RPO (Max tolerable data loss) ────>|<─── RTO (Max tolerable downtime) ──>|
```

### A. RTO (Recovery Time Objective)
The maximum tolerable duration of downtime after a disaster before service must be restored.
- *Question*: "How quickly do we need to get back online?"
- *Example*: An RTO of 5 minutes means the system must failover and resume serving traffic within 5 minutes of a failure.

### B. RPO (Recovery Point Objective)
The maximum tolerable age of data that can be lost due to a disaster.
- *Question*: "How much data can we afford to lose?"
- *Example*: An RPO of 1 hour means database backups must be taken at least every hour; if a database crashes, we may lose up to 1 hour of writes. An RPO of 0 requires synchronous multi-region database replication.

---

## 2. Multi-Region Deployment Topologies

To survive the outage of an entire cloud region (e.g., AWS us-east-1), applications must be distributed across multiple geographical zones.

| Topology | Mechanics | Pros | Cons | Cost |
| :--- | :--- | :--- | :--- | :--- |
| **Active-Passive (Warm Standby / Pilot Light)** | Traffic is routed to Region A (Primary). Region B is idle or running minimal resources. Database replicates asynchronously. | Simple, no write conflicts. | High RTO during failover; risk of DNS replication delay. | Medium |
| **Active-Active (Multi-Region)** | Both regions actively serve user traffic concurrently. DNS (Route53, Cloudflare) distributes traffic globally based on latency. | **Near-Zero RTO/RPO**; maximum scalability and speed. | **Extreme write-conflict complexity**; data replication lag. | High |

---

## 3. The Split-Brain Problem

In a distributed Active-Active database or cluster, **Split-Brain** occurs when a network partition cuts off communication between nodes (e.g., Region A and Region B), but both regions continue running independently.

### The Danger:
- Region A thinks Region B is dead and accepts writes.
- Region B thinks Region A is dead and accepts writes.
- When the network partition heals, the two regions have conflicting updates for the same data, leading to severe data corruption.

### Solutions & Mitigation Strategies:
1. **Consensus Protocols (Raft, Paxos)**: Databases use a quorum (majority vote). If a cluster has 3 nodes and a network partition splits them into a group of 1 and 2, the group of 2 has the majority ($>50\%$) and remains writeable. The isolated node of 1 cannot achieve quorum and automatically switches to read-only.
2. **Conflict-Free Replicated Data Types (CRDTs)**: Special data structures (like Grow-Only counters or LWW-Element-Set) designed to merge conflicting writes mathematically without needing locks (eventual convergence).
3. **Fencing Tokens**: A monotonic increasing counter used to invalidate outdated/orphaned master nodes trying to write to storage.

---

## 4. Chaos Engineering

**Chaos Engineering** is the discipline of experimenting on a system in production to build confidence in the system's capability to withstand turbulent conditions.
- **Goal**: Proactively identify hidden architectural flaws (e.g., cascading retry storms, misconfigured timeouts, circular dependencies) before they cause actual customer outages.
- **Netflix Chaos Monkey**: Automatically terminates random microservice instances in production to verify that the application has self-healing capabilities.

---

## 5. Hard Interview Questions & Deep Answers

### Q1: How do you design an Active-Active multi-region architecture for a global E-commerce checkout system?
**Answer**:
Designing Active-Active for checkout is highly complex because of stock management (avoiding double-selling the same physical product) and transactional consistency.
1. **DNS Layer**: Use Anycast DNS or Latency-based Routing (e.g., AWS Route53) to route users to the nearest regional gateway (e.g., US-West vs. EU-Central).
2. **Database Strategy (ScyllaDB / Cassandra vs. Spanner)**:
   - Use a **NewSQL database** with synchronous multi-region replication (like Google Cloud Spanner) if strong consistency is mandatory. Spanner uses Atomic Clocks and TrueTime API to coordinate global serializable transactions.
   - Alternatively, use **Database Sharding with Location Affinity**: Route US users' orders to US-West shards and EU users' to EU shards. Since users rarely shop across regions concurrently, we isolate write operations, avoiding global conflicts.
3. **Inventory Management**: Use asynchronous reserving patterns. Assign a specific portion of the inventory to each region (e.g., 500 items in US, 500 in EU). If a region runs out of local inventory, it queries other regions asynchronously for transfers.

### Q2: What is "Cascading Failure," and how do you design systems to prevent it during a regional failover?
**Answer**:
- **Cascading Failure**: A failure in one component triggers failures in other components in a domino effect, eventually bringing down the entire platform.
- **Failover Scenario**: If Region A fails, DNS immediately reroutes all Region A traffic to Region B. If Region B is only provisioned to handle its normal local capacity, the sudden influx of 100% additional traffic will instantly overwhelm Region B's CPU, database connection pool, and thread limits, causing Region B to crash as well (complete blackout).
- **Prevention Designs**:
  1. **Autoscaling Pre-warming**: Ensure secondary regions maintain enough baseline headroom or can scale up instantly under massive load using predictive scheduling.
  2. **Circuit Breakers**: Gracefully degrade non-critical features (e.g., hide product reviews or recommendations) to free up CPU cycles for core checkout paths.
  3. **Rate Limiting & Shedding**: Implement **Load Shedding** at the API Gateway. When queue latency exceeds a threshold, instantly reject a percentage of non-critical requests with `503 Service Unavailable` to protect database stability.

### Q3: Explain how "Split-Brain" affects write operations in multi-region Kafka or Redis clusters, and how you recover from it.
**Answer**:
- **Redis (Active-Passive Sentinel / Cluster)**: If Sentinel nodes lose contact with the primary Redis master in Region A due to network partition, they will promote a replica in Region B to master. If some clients are still connected to Region A, they will keep writing to it (Dual Master).
- **Recovery & Data Loss**: Redis replication is asynchronous. Once the partition heals, Sentinel demotes the old master in Region A to replica. The old master clears its entire database and replicates from the new master in Region B. **All writes accepted by Region A during the partition are permanently lost**.
- **Mitigation**: Configure `min-replicas-to-write 1` and `min-replicas-max-lag 10`. If the master loses contact with replicas, it immediately rejects incoming writes, minimizing data loss during partitions.
- **Kafka**: Kafka uses **ZooKeeper** or **Raft (KRaft)** to elect leaders. A partition cannot elect a new leader without a majority. For writes, configuring `acks=all` ensures a write is only successful if it is written to the minimum in-sync replicas (`min.insync.replicas`), preventing split-brain writes.
