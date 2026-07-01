# Command Query Responsibility Segregation (CQRS)

**CQRS** is a design pattern that segregates the data models and database engines used to write data (Commands) from those used to read data (Queries).

In a traditional application:
* The same domain model and database are used for both reads and writes.
* *The Problem:* As applications scale, optimization goals clash. Writes need normalization to prevent anomalies, while reads need complex indices, denormalization, or search caches (like Elasticsearch) to scale.

```
                      ┌───────────┐
                      │  Client   │
                      └─────┬─────┘
            ┌───────────────┴───────────────┐
            ▼ (Write Path)                  ▼ (Read Path)
      ┌───────────┐                   ┌───────────┐
      │ Commands  │ (State changes)   │  Queries  │ (Reads only)
      └─────┬─────┘                   └─────┬─────┘
            ▼                               ▼
      ┌───────────┐   Async Sync      ┌───────────┐
      │ Write DB  ├──────────────────►│  Read DB  │ (e.g., Elasticsearch, Read Cache)
      └───────────┘  (Events/PubSub)  └───────────┘
```

* **Command Path (Writes):** Processes business logic, performs input validation, modifies state, and persists data. Return values are typically empty or contain strictly status validation responses (no entity payloads).
* **Query Path (Reads):** Fetches read-only data. Bypasses domain validation, optimized strictly for low-latency retrieval.

---

## 2. Event Sourcing (The Natural Companion)

**Event Sourcing** is an architectural pattern where we do **not** store the current state of an entity. Instead, we store **the entire historical sequence of events (state changes)** that occurred to that entity in an append-only log (the **Event Store**).

* **How to derive State:** To get the current state of an entity (e.g., a user's account balance), we pull all its events from disk and **replay** them sequentially from start to finish.
* **Why it fits CQRS:** Since replaying events on every read request is highly expensive, we use **CQRS**. The write path appends events to the Event Store. An event handler reads these events asynchronously, builds a denormalized "Read View" of the current state, and saves it in a fast Read DB (like Redis or MongoDB) for query retrieval.

---

## Interview Questions & Answers

### Q1: What are the main benefits and trade-offs of using CQRS?
- **Answer:**
  - **Benefits:** Maximizes scalability by allowing read and write databases to scale independently; read models can be highly denormalized for sub-millisecond query lookups.
  - **Trade-offs:** High architectural complexity; introduces **Eventual Consistency** because syncing the read database from the write log takes time.
