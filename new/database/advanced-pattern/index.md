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

---

## 6. Multi-Tenant Enterprise Database Patterns

When building Multi-Tenant SaaS systems, you must choose how to isolate tenant data at the storage tier:

| Pattern | Isolation Level | Cost Efficiency | Operational Complexity | Core Risk |
| :--- | :---: | :---: | :---: | :--- |
| **Database per Tenant** (Physical) | **Highest** | **Lowest** (Idle DBs cost $) | Medium-High (Many DBs to update) | Slow provisioning; resource waste. |
| **Schema per Tenant** (Logical) | **Medium** | **Medium** | **Highest** (Schema migrations lag) | Database connection pool exhaustion. |
| **Shared Schema** (Row-level) | Low-Medium | **Highest** | **Lowest** (Single schema update) | Developer error leaking tenant data. |

### A. Deep Dive: Row-Level Isolation (Shared Database & Shared Schema)
The most common and cost-effective approach. Every table contains a `tenant_id` column.
* **The Vulnerability**: A developer forgets to include `WHERE tenant_id = ?` in a complex query, resulting in a cross-tenant data leak.
* **The Solution (PostgreSQL Row-Level Security - RLS)**:
  Enforce isolation directly at the database engine level so queries are automatically scoped:
  ```sql
  -- 1. Enable RLS on the table
  ALTER TABLE transactions ENABLE ROW LEVEL SECURITY;
  
  -- 2. Create an isolation policy
  CREATE POLICY tenant_isolation_policy ON transactions
    FOR ALL
    USING (tenant_id = current_setting('app.current_tenant_id'));
  ```
  * In the application connection pool, every transaction must first execute:
    `SET LOCAL app.current_tenant_id = 'tenant_abc';`
  * The engine then transparently injects the `tenant_id` filter into *all* queries, completely eliminating the developer leakage risk.

---

## 7. Distributed Locking

In a microservice environment, local language locks (like Go's `sync.Mutex` or Java's `synchronized`) are useless because multiple application instances run on separate physical servers with disjoint virtual memory.

### A. Redis-Based Distributed Locks

To prevent concurrent race conditions across clusters, use a shared coordinator like Redis:

#### 1. Single-Instance Redis Lock
* **Acquiring Lock**: Execute an atomic `SET` with `NX` (Not Exists) and `PX` (Expiration Time):
  ```redis
  SET order_lock_123 "unique_request_uuid" NX PX 30000
  ```
  * `NX` guarantees mutual exclusion.
  * `PX 30000` guarantees the lock auto-expires in 30 seconds if the owner node crashes (preventing permanent deadlocks).
* **Releasing Lock**: You must *only* release the lock if the value matches your `unique_request_uuid` (preventing Node A from deleting a lock that has expired and been acquired by Node B). This requires a **Lua Script** executed atomically in Redis:
  ```lua
  if redis.call("get", KEYS[1]) == ARGV[1] then
      return redis.call("del", KEYS[1])
  else
      return 0
  end
  ```

#### 2. Redlock Algorithm (Multi-Instance Distributed Consensus)
If the single Redis master node crashes, a secondary replica promoted via failover might not have received the lock key yet (as replication is async), allowing two nodes to acquire the same lock.
* **The Redlock Steps**:
  1. The client gets the current timestamp.
  2. It attempts to acquire the lock sequentially across $N$ independent Redis master nodes (typically 5) using the same key and random value. It uses a very small timeout for each node (e.g., 5-10ms) to avoid blocking if a node is down.
  3. The client computes the elapsed time. The lock is successfully acquired **only if**:
     * The client successfully locked a strict **majority of nodes** ($N/2 + 1$, i.e., 3 out of 5).
     * The total elapsed time to acquire the locks is **less than the lock validity time**.
  4. If acquired, the lock's actual active time is the initial validity time minus the elapsed acquisition time.
  5. If the client fails to acquire the majority, it must immediately send a broadcast unlock script to **all nodes** to clean up partial locks.

---

## 8. Hard Interview Questions & Deep Answers (Extended)

### Q4: Explain the famous critique of Redis Redlock by Martin Kleppmann. Why does Redlock fail under severe asynchronous network and system conditions?
**Answer**:
Martin Kleppmann proved that Redlock is unsafe for distributed correctness because it relies on the physical system clock, which can drift, jump, or freeze in real-world systems. He outlined two main failure paths:
1. **GC Pause (Stop-The-World)**:
   * Client A acquires locks on Redis nodes 1, 2, and 3 (majority).
   * Client A immediately enters a long garbage collection (JVM/Go STW) pause.
   * While Client A is paused, the locks expire on the Redis nodes.
   * Client B acquires the locks on nodes 1, 2, 3 (since they expired).
   * Client A wakes up from its GC pause, assumes its lock is still valid, and writes to the database—violating mutual exclusion!
2. **Clock Drift**:
   * Redis Node 3's system clock drifts forward rapidly.
   * Client A acquires locks on Nodes 1 and 2. Node 3's clock jumps forward, immediately expiring the lock on Node 3.
   * Client B can now lock Nodes 3, 4, and 5, resulting in both Client A and Client B holding the lock simultaneously.
* **Conclusion**: For **performance locks** where occasional duplicates are acceptable, Redlock is fine. For **correctness/safety locks** where double-locking means data corruption, you must use a consensus-backed coordinator like **ZooKeeper** or **etcd** which uses sequentially incremented numbers called **Fencing Tokens** (or raft term indexes) to reject stale writes at the storage layer.

### Q5: In a multi-tenant SaaS application, how do you handle asymmetric JWT verification when different tenants insist on using their own identity providers (IdPs like Okta, Azure AD, Ping Identity)?
**Answer**:
To verify asymmetric JWTs signed by tenant-specific IdPs without compiling static public keys into your application code:
1. **Dynamic Issuer Parsing**: When a request arrives, parse the unverified JWT header to extract the `"iss"` (Issuer) claim or inspect the hostname (e.g., `tenant-a.saas.com`).
2. **Key Provider Routing (JWKS Lookup)**: Maintain a tenant routing metadata table mapping `tenant_id` to their IdP's JWKS (JSON Web Key Set) URL (e.g., `https://tenant-a.okta.com/oauth2/v1/keys`).
3. **Dynamic JWKS Client**: Use a JWKS caching client with a circuit breaker. It fetches the JWKS from the routed tenant-specific URL, caches the public keys in memory indexed by the token's key ID (`kid`), and uses the matching public key to verify the token signature.
4. **Tenant Metadata Caching**: Cache the metadata lookup in Redis to avoid hitting PostgreSQL on every request gateway. This keeps verification extremely fast and fully decoupled across hundreds of corporate enterprise tenants.

