# Redis Scaling: Cluster & Sentinel

Detailed guide for high availability (Sentinel) and horizontal scalability (Native Gossip Cluster).

## Redis Sentinel (High Availability)
Sentinel handles monitoring, notification, and automatic master-to-replica failover for single-master Redis topologies.
- **Role:** If the master fails, Sentinel coordinates majority consensus to promote the healthiest replica, instructing client drivers to reconnect to the new master.

## Redis Cluster (Horizontal Scalability)
Redis Cluster provides multi-master horizontal write scaling and partition sharding without a proxy layer.
- **Hash Slots:** Reads/writes are partitioned across **16,384 logical Hash Slots** distributed across master nodes.
- **CRC16 Hashing:** Keys are mapped to hash slots using the algorithm:
  $$\text{Slot} = \text{CRC16}(\text{key}) \pmod{16384}$$
- **Hash Tags:** Substrings wrapped in curly braces (e.g., `{user_101}:profile`) override standard hashing, forcing keys with matching tags to map to the exact same hash slot, enabling safe multi-key operations.

## Interview Questions & Answers

### Q1: What is the difference between Redis Sentinel and Redis Cluster?
- **Answer:** **Redis Sentinel** is designed for **High Availability (HA)**. It coordinates failover for a single master with read replicas, but does not split data (no sharding)—all nodes have full copies of the dataset. **Redis Cluster** is designed for **Horizontal Scaling**. It shards data across multiple master nodes using hash slots, splitting both read and write workloads across the cluster.
