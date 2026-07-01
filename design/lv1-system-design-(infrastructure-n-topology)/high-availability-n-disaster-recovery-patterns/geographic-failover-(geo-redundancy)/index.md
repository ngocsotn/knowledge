# Geographic Failover & Redundancy

To survive the outage of an entire cloud region (e.g., AWS us-east-1), applications must be distributed across multiple geographical zones.

| Topology | Mechanics | Pros | Cons | Cost |
| :--- | :--- | :--- | :--- | :--- |
| **Active-Passive (Warm Standby / Pilot Light)** | Traffic is routed to Region A (Primary). Region B is idle or running minimal resources. Database replicates asynchronously. | Simple, no write conflicts. | High RTO during failover; risk of DNS replication delay. | Medium |
| **Active-Active (Multi-Region)** | Both regions actively serve user traffic concurrently. DNS (Route53, Cloudflare) distributes traffic globally based on latency. | **Near-Zero RTO/RPO**; maximum scalability and speed. | **Extreme write-conflict complexity**; data replication lag. | High |

---

## Interview Questions & Answers

### Q1: What is the difference between RTO and RPO in disaster recovery?
- **Answer:**
  - **RTO (Recovery Time Objective):** The maximum tolerable **Duration** of system downtime before service is restored (e.g., system must recover within 30 minutes).
  - **RPO (Recovery Point Objective):** The maximum tolerable **Data Loss** measured in time (e.g., if a database fails, restoring from a 4-hour-old backup is acceptable, meaning RPO is 4 hours).
