# Peer-to-Peer (P2P) Architecture

P2P is a decentralized distributed architecture where all nodes (peers) share equal privileges, acting simultaneously as both clients (requesters) and servers (providers).
- **The Mechanism:** Nodes communicate directly without central coordinators, sharing bandwidth, processing power, and storage (e.g., BitTorrent, blockchain networks, Gossip P2P).

## Interview Questions & Answers

### Q1: How do P2P networks locate resources without a central directory?
- **Answer:** Distributed Hash Tables (DHT). Peer networks construct a logical keyspace across nodes. Each peer is responsible for storing a range of key-value pairs (resource location coordinates), allowing $O(\log N)$ lookup routing across peer nodes.
