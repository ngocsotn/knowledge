# CQRS & Event Sourcing

Comprehensive interview study guide covering Command Query Responsibility Segregation (CQRS), Read/Write segregation, and Event Sourcing patterns.

---

## 1. Meaning of CQRS

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

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What are the main benefits and trade-offs of using CQRS?
* **Answer:**
  * **Benefits:** segregating models allows you to optimize read and write paths independently (e.g., scaling write instances separately from read replicas). It also allows you to use different database engines (e.g., PostgreSQL for relational writes, and Elasticsearch for autocomplete searches), maximizing read performance.
  * **Trade-offs:** Introduces massive complexity. It requires managing separate code paths, data stores, and replication systems. Because read replicas sync asynchronously, the system suffers from **eventual consistency**, meaning clients might read stale data immediately after a write.

### Q2: How is data synchronized between the Write Database and Read Database in a CQRS architecture?
* **Answer:** Data sync is handled asynchronously using **Event-Driven Pub/Sub** or **Change Data Capture (CDC)**:
  1. When a Command modifies the Write DB, the application publishes an event (e.g., `OrderPaid`) to a message broker (like Kafka).
  2. An asynchronous **Event Handler** service consumes this event.
  3. The handler performs any required transformations, computes denormalized values, and writes the updated state directly to the Read Database, bringing it up-to-sync.

### Q3: What is a "Snapshot" in Event Sourcing, and why is it necessary?
* **Answer:** In Event Sourcing, deriving current state requires replaying every historical event of an entity. If an entity (e.g., a high-frequency trading account) has millions of event logs over several years, replaying them on startup is extremely slow, adding severe latency. A **Snapshot** is a checkpoint saving the exact state of an entity at a specific event index (e.g., state at event #10,000). To load the current state, the system loads the latest snapshot instantly and replays *only* the few events that occurred after the snapshot, optimizing startup performance.
