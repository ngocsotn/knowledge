# Database Advanced Patterns

Comprehensive study guide for distributed systems, transaction management, data consistency, and integration patterns in modern microservice architectures.

---

## 1. The Fall of 2PC (Two-Phase Commit) & XA

In a distributed microservice architecture, a single business process often spans multiple services, each owning its private database. Ensuring atomic, global transactions across these databases is extremely challenging.

### Why 2PC/XA Fails at Scale:
- **How 2PC Works**: A coordinator node asks all participant databases to "prepare" to commit. Once all participants vote "yes," the coordinator sends a "commit" message to all of them.
- **Why it scales poorly**:
  1. **Blocking Protocol**: During the prepare phase, participants lock database rows. If the coordinator crashes or the network partitions, those locks stay held indefinitely, blocking other concurrent transactions and causing system-wide cascading thread pool exhaustion.
  2. **High Latency**: Requires multiple network round-trips over the network, making it highly vulnerable to packet loss, latency spikes, and coordinator failures.
  3. **Single Point of Failure**: The coordinator is a critical bottleneck and single point of failure.

---

## 2. Saga Pattern (Distributed Transactions)

The **Saga Pattern** manages distributed transactions by breaking down a global transaction into a series of local, independent transactions. Each local transaction updates its own database and triggers the next step in the saga via message/event streaming.

### Orchestration vs. Choreography Sagas

```
Choreography (Event-Driven)            Orchestrator-Driven
[Order Svc] --(Created)--> [Payment]   [Order Svc] <---> [Saga Orchestrator]
[Payment] --(Paid)-------> [Stock]                       |              |
                                                    [Payment]        [Stock]
```

- **Choreography (Decentralized)**:
  - Participants listen to events from message queues and trigger their own operations.
  - *Pros*: Simple to start, loosely coupled, no single point of failure.
  - *Cons*: High cognitive overhead; hard to trace, debug, or understand the global state of a saga as the number of steps grows.
- **Orchestration (Centralized)**:
  - A dedicated "Orchestrator" service coordinates all steps. It explicitly commands each participant service to perform its action and handles the sequence.
  - *Pros*: Centralized visibility, easier state-machine tracking, simple to design and test complex workflows.
  - *Cons*: High coupling risk (orchestrator easily accumulates too much business logic); orchestrator failure blocks sagas.

### Handling Failures: Compensating Transactions
Since sagas cannot rely on database-level rollbacks (local transactions are committed immediately to keep locks short), failures must be handled using **Compensating Transactions**.
- If Step 3 (Reserve Stock) fails, the saga triggers reverse operations in reverse order: Step 2 compensation (Refund Payment) followed by Step 1 compensation (Cancel Order).
- Compensating transactions **must be idempotent** because they might be retried multiple times due to network or service crashes.

---

## 3. CQRS & Event Sourcing

### A. CQRS (Command Query Responsibility Segregation)
Separates read operations (Queries) from write operations (Commands) at the application and database level.
- **Write Database**: Highly optimized for transactional consistency, normalization, and quick updates (e.g., PostgreSQL).
- **Read Database**: Highly optimized for search, indexing, and complex read queries (e.g., Elasticsearch, Redis).
- **Sync**: Writes asynchronously sync to the read model via message queues or Change Data Capture (CDC).

### B. Event Sourcing
Instead of storing the *current state* of an entity, Event Sourcing stores the **entire history of changes** as a sequential sequence of immutable events in an **Event Store**.
- **State Reconstruction**: To find the current state of an account, the system reads all historical events (e.g., `AccountCreated`, `Deposited`, `Withdrawn`) from the start and replays them.
- **Snapshots**: To avoid replaying millions of events, the system periodically saves state snapshots (e.g., state at event 1000). Replay starts from the latest snapshot.

---

## 4. The Dual-Write Problem & Transactional Outbox

### The Dual-Write Problem
- **Problem**: When a service needs to update its local database AND publish an event to a message broker (Kafka/RabbitMQ) in a single request.
  ```go
  // Insecure Pseudo-code
  db.Save(user)
  kafka.Publish("user-created", user) // If database succeeds but Kafka fails -> Out of Sync!
  ```
  Whether you write to the DB first or publish to Kafka first, a network timeout or crash between the two actions will leave your system in an inconsistent state.

### The Transactional Outbox Pattern
Guarantees **at-least-once delivery** of messages without using distributed transactions.

```
[User Request] ──> [DB Transaction: Save User + Save Outbox Row]
                               |
                       (Polls DB or reads WAL)
                               v
                       [Outbox Publisher] ──> [Kafka / RabbitMQ]
```

1. **Atomic Write**: In a single database transaction, the application writes the user record to the `users` table AND inserts a corresponding message record into an `outbox` table in the same database.
2. **Asynchronous Relaying**: A separate, lightweight background worker (the Outbox Publisher) reads unsent messages from the `outbox` table, publishes them to Kafka, and marks them as sent in the DB.
3. **Optimized Relaying (CDC)**: Instead of polling the database frequently, the publisher reads the database **Write-Ahead Log (WAL)** (e.g., PostgreSQL Logical Replication via Debezium) to intercept writes to the outbox table in near real-time, eliminating poll latency and DB overhead.

---

## 5. Hard Interview Questions & Deep Answers

### Q1: How do you handle eventual consistency and reading stale data in a CQRS/Event Sourced system?
**Answer**:
Eventual consistency is a natural consequence of separating write models from read models. The latency between writing to the write-DB and projecting to the read-DB can be milliseconds to seconds, leading to a "read-your-own-write" consistency issue (user updates profile but still sees old data upon redirection).
- **Mitigation Patterns**:
  1. **Write-Through Caching / Optimistic UI Update**: The frontend immediately updates its UI state with the client's payload, assuming success, before the read model is synced.
  2. **Session / Client-Side Version Tracking**: Each write returns the latest state version number (or a transaction sequence ID). When fetching, the client includes this version. The read-model gateway checks if its local version is $\ge$ client version; if not, it polls with a short backoff or reads directly from the write-DB as a fallback.
  3. **Direct Read on Critical Paths**: Critical operational flows (e.g., authentication, checkout) bypass read model databases entirely and query the normalized write database directly to guarantee strong consistency.

### Q2: How do you guarantee message deduplication and handle duplicate messages in the downstream consumers?
**Answer**:
The Transactional Outbox pattern guarantees **at-least-once** delivery. Network failures, broker crashes, or worker restarts can cause identical messages to be sent or consumed multiple times. Downstream consumers must handle these duplicates to guarantee **exactly-once processing** via **Idempotent Consumers**:
1. **Unique Message Identifier**: Each message must carry a globally unique ID (e.g., `message_id: uuid`).
2. **Idempotence Store**: The consumer stores processed message IDs in a fast, transaction-safe database (or Redis with lock).
3. **Consumer Steps**:
   - Check if `message_id` exists in the idempotence database inside a transaction.
   - If yes, skip processing (it is a duplicate) and acknowledge the broker.
   - If no, process the message payload, update the local business tables, insert the `message_id` into the idempotence store, and commit all changes in a **single atomic transaction**.

### Q3: What is "Event Drift" in Event Sourcing, and how do you handle schema upgrades of historical events?
**Answer**:
- **Event Drift**: Occurs when the structure (schema) of business events changes over time (e.g., splitting a field, changing field names), but historical events stored in the immutable Event Store cannot be modified.
- **Handling Schema Upgrades**:
  1. **Upcasting (Best Practice)**: Build an intermediate translation layer (Upcasters) in the application. When historical events are loaded from the Event Store, they are passed through a chain of Upcasters that transform old schemas (e.g., v1) into the current schema (e.g., v3) in-memory before the event reaches the domain aggregate.
  2. **Lazy Migration (Copy and Replace)**: A background migration script reads old events, maps them to the new schema format, and writes them to a new event stream, leaving a pointer linking the old stream to the new one.
  3. **Metadata Versioning**: Include an explicit version tag in the event envelope metadata. Resolvers use factory patterns to parse JSON payloads into different class versions based on the metadata tag.
