# Publisher-Subscriber (Pub/Sub) Pattern

The Publisher-Subscriber pattern is an asynchronous messaging paradigm where publishers broadcast events to a message broker without knowing who (or how many) subscribers exist.
- **Decoupling:** Decouples systems across time, space, and technology boundaries, boosting cluster resilience and write throughput.

## Interview Questions & Answers

### Q1: How do you handle message deduplication in a Pub/Sub consumer?
- **Answer:** Implement **Idempotent Consumers**. Ensure every message payload carries a unique transaction ID (`idempotency_key`). When the consumer receives a message, it verifies if the key exists in a fast transactional store (like Redis/PostgreSQL). If present, it discards the message as a duplicate; if not, it executes the write and commits the key.
