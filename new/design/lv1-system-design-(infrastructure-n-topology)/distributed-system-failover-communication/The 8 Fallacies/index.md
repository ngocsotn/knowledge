# The 8 Fallacies of Distributed Computing

The 8 Fallacies represent false assumptions developers frequently make when designing distributed software systems:
1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

## Interview Questions & Answers

### Q1: Why is assuming 'latency is zero' a dangerous design mistake in microservices?
- **Answer:** N+1 network call queries. If a developer assumes latency is zero, they might design microservices to talk to each other sequentially in long blocking loops (e.g., retrieving a list of 50 items and executing a separate network call to verify details for each item). At scale, this accumulates milliseconds of network packet latency, leading to slow response times and cascading thread starvation across services.
