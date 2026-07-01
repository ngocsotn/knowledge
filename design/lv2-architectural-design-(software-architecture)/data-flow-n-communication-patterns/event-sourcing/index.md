# Event Sourcing

**Event Sourcing** is an architectural pattern where we do **not** store the current state of an entity. Instead, we store **the entire historical sequence of events (state changes)** that occurred to that entity in an append-only log (the **Event Store**).

* **How to derive State:** To get the current state of an entity (e.g., a user's account balance), we pull all its events from disk and **replay** them sequentially from start to finish.
* **Why it fits CQRS:** Since replaying events on every read request is highly expensive, we use **CQRS**. The write path appends events to the Event Store. An event handler reads these events asynchronously, builds a denormalized "Read View" of the current state, and saves it in a fast Read DB (like Redis or MongoDB) for query retrieval.

---

## Interview Questions & Answers

### Q1: How do you handle state lookup in Event Sourcing without replaying millions of events?
- **Answer:** Snapshots. The system periodically (e.g., every 100 events) serializes and saves the aggregate's current state as a physical snapshot in a fast database. To restore state, the repository loads the latest snapshot and replays only the subsequent events generated after that snapshot's offset, reducing restoration time from $O(N)$ to $O(1)$.
