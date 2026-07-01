# Heartbeat & Ping Patterns

Heartbeat-Ping patterns are failure detection mechanisms used to monitor node health and cluster connectivity in distributed networks.
- **Heartbeat:** A node periodically sends a "still alive" message to a master node or coordinate group.
- **Ping:** A monitoring node explicitly sends a query to peer nodes and waits for an echo response.

## Interview Questions & Answers

### Q1: What is the risk of a short Heartbeat interval in high-scale clusters?
- **Answer:** Network storms and false positives. If the interval is too short, the master is flooded with heartbeat packets, wasting bandwidth and CPU. If network spikes occur, the master might miss a single heartbeat and falsely assume a healthy node has crashed, triggering expensive, unnecessary node eviction and partition rebalancing. Mitigate this by utilizing sliding-window failure detectors (like Cassandra's Phi Accrual Failure Detector).
