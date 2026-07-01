# Gossip Protocol

Gossip Protocol is a decentralized, peer-to-peer communication pattern where nodes in a distributed system periodically share metadata state with a few randomly selected peer nodes.
- **The Mechanism:** Information spreads exponentially across the cluster like a virus (epidemic routing).
- **Scale:** Decoupled; no single point of failure; excellent for tracking cluster membership, node failures, and schema versions in large systems (used heavily in Redis Cluster and Apache Cassandra).

## Interview Questions & Answers

### Q1: Why is Gossip Protocol highly scalable compared to centralized coordination?
- **Answer:** Centralized coordinators (like ZooKeeper or a master node) suffer from network bottlenecks as the cluster grows because every node must communicate with the master. Gossip Protocol is completely peer-to-peer. Each node communicates with only a tiny, fixed number of random peers, keeping network and memory overhead flat ($O(1)$ per node) regardless of total cluster size.
