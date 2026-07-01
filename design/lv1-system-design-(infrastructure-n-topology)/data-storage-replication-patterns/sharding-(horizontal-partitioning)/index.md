# Database Sharding (Horizontal Partitioning)

Comprehensive guide covering database sharding, horizontal vs. vertical partitioning, routing topologies, data distribution algorithms, and the core engineering challenges of sharding.

---

## 1. What is Sharding?

**Sharding** is a database architecture pattern where a single logical database is partitioned horizontally across multiple physical database instances (shards). Each individual physical database is completely independent and contains a distinct subset of the overall data.

### Vertical vs. Horizontal Partitioning
- **Vertical Partitioning (Normalization-like separation):**
  - *Mechanism:* Splitting tables by **columns**.
  - *Example:* Moving large, rarely accessed text columns (e.g., `user_bio`) or sensitive fields (e.g., `credit_card`) into a separate physical table, while keeping core fields (`user_id`, `email`) in the main table.
- **Horizontal Partitioning (Sharding):**
  - *Mechanism:* Splitting tables by **rows**.
  - *Example:* Row indices `1` to `1,000,000` are sent to Shard A, while indices `1,000,001` to `2,000,000` are sent to Shard B. The schema remains identical across all shards, but the data rows are disjoint.

---

## 2. Sharding Routing Topologies

To read or write data, the application must resolve which specific physical shard holds the target row based on a **Shard Key**. There are two primary routing topologies:

### A. Routing Proxy Layer (e.g., Vitess, PgBouncer, ProxySQL)
The application talks to a single, unified database endpoint proxy. The proxy parses the SQL query, extracts the shard key, routes the query to the correct physical shard, aggregates results, and returns them to the client.

```
                  ┌──────────────────────┐
                  │  Client Application  │
                  └──────────┬───────────┘
                             │ (Query: WHERE user_id = 99)
                             ▼
                  ┌──────────────────────┐
                  │    Routing Proxy     │ (Parses SQL, extracts key)
                  └─────┬──────────┬─────┘
           ┌────────────┘          └────────────┐
           ▼                                    ▼
┌──────────────────────┐             ┌──────────────────────┐
│  Shard A (IDs 1-50)  │             │ Shard B (IDs 51-100) │ (user_id 99 here)
└──────────────────────┘             └──────────────────────┘
```

- **Pros:** Keeps application code extremely clean. The app behaves as if it is talking to a single database.
- **Cons:** Introduces an extra network hop and a potential proxy bottleneck.

### B. Smart Client (Application-Level Routing)
The application client library directly maintains the map of shard ranges and physical database IP addresses. It performs the shard key calculation in-process and opens a TCP socket directly to the target database shard.
- **Pros:** Maximum performance (zero proxy overhead or network hops).
- **Cons:** Extremely complex application configuration. Upgrading client libraries across multiple services to sync new shards is error-prone.

---

## 3. Core Sharding Algorithms

1. **Range-Based Sharding:**
   - *Mechanism:* Rows are split based on ranges of the shard key (e.g., `user_id` 1–10k -> Shard 1, 10k–20k -> Shard 2).
   - *Cons:* Extremely prone to write hot-spots. If new users are registered sequentially, all fresh write traffic hits the highest shard range, leaving other shards idle.
2. **Hash-Based Sharding:**
   - *Mechanism:* Apply a cryptographic hash (like MurmurHash3) to the shard key and modulo the result by the number of active shards: `ShardID = Hash(key) % N`.
   - *Pros:* Excellent, uniform data distribution across all shards.
   - *Cons:* Changing the number of shards (re-sharding) is a nightmare—it invalidates the modulo math, requiring moving ~90% of the entire database rows to different shards.
3. **Consistent Hashing:**
   - *Mechanism:* Objects and physical servers are mapped onto a 360-degree virtual ring. Objects are routed to the nearest physical server clockwise on the ring.
   - *Pros:* Scaling or shrinking the database cluster only requires moving a tiny fraction of the data (typically `1/N` of the overall database).

---

## 4. Key Engineering Challenges of Sharding

* **Cross-Shard Joins:** Joining tables across different physical servers is virtually impossible to do performantly. Complex queries must either be de-normalized (replicating read-only data across all shards) or resolved via slow in-memory application-level merges.
* **Distributed Transactions (Atomic Commits):** Performing atomic updates across two different shards (e.g., transferring funds from User A on Shard 1 to User B on Shard 2) requires heavy, high-latency coordinator protocols like **Two-Phase Commit (2PC)** or Saga Orchestrations.
* **Re-Sharding Operations:** As the system grows, shards will inevitably fill up and must be split. Moving terabytes of database rows live in production without downtime (using tools like Change Data Capture - CDC) is exceptionally difficult.

---

## 5. High-Impact Interview Questions & Answers

### Q1: What makes a "good" Shard Key versus a "bad" Shard Key? Give examples.
* **Answer:**
  - **Good Shard Key (High Cardinality, Even Write Distribution):** A key like `user_id` or `tenant_id` is an excellent shard key. It has millions of unique values (high cardinality), ensuring data is split finely across many servers. It also results in a highly uniform distribution of reads and writes when hashed.
  - **Bad Shard Key (Low Cardinality or Sequentially Ordered):**
    - `country_code` is a bad shard key because it has low cardinality (fewer than 200 countries). A massive country (like the US) will create a giant, bloated shard that dwarfs a smaller country shard, causing massive physical data skew.
    - `created_at` timestamp is a bad shard key because it is sequentially ordered. All new user signups or orders occur in the current minute, meaning 100% of all write queries will hit the newest shard, creating an expensive database hot-spot while older shards sit idle.

### Q2: How does Consistent Hashing minimize the data-migration footprint when adding a new physical shard to a cluster?
* **Answer:** Under standard modular sharding (`Hash(key) % N`), adding a single database shard changes `N` to `N + 1`. This modifies the hash modulo result for almost every single key in the database, forcing a massive, cluster-wide migration of nearly 90% of the overall data.
- **Consistent Hashing Solution:** It maps both keys and physical servers onto a circular 360-degree hash ring. Keys are routed clockwise to the nearest physical server node. When a new physical server is inserted onto the ring, it only intercepts keys located between itself and its counter-clockwise neighbor. All other keys on the rest of the ring are completely unaffected. Only a small fraction of the data (approximately `TotalData / (N + 1)`) needs to be migrated, minimizing network overhead and keeping database performance stable during scaling.

### Q3: How do you perform a transaction that spans across two separate physical shards without using high-latency 2PC (Two-Phase Commit)?
* **Answer:** In high-throughput architectures, Two-Phase Commit is avoided because it locks database rows across multiple servers until the slowest network node responds, easily causing system deadlocks. The primary alternative is the **Saga Pattern**:
  1. **Split into Local Transactions:** Break the cross-shard transaction into a sequence of isolated, local transactions on each individual shard.
  2. **Asynchronous Orchestration:** Shard 1 executes its local transaction (e.g., debiting $100 from User A) and publishes a success event (`MoneyDebitedEvent`) to a message queue.
  3. **Event-Driven Continuation:** Shard 2 consumes this event and executes its local transaction (e.g., crediting $100 to User B).
  4. **Compensating Transactions (Rollbacks):** If Shard 2 fails (e.g., User B's account is suspended), it publishes a failure event (`CreditFailedEvent`). Shard 1 consumes the failure event and triggers a **compensating transaction** (e.g., refunding $100 back to User A) to return the distributed system back to a consistent state.

### Q4: What is the "Fan-Out" query problem in sharding, and how do you optimize against it?
* **Answer:**
  - **The Problem:** If a client executes a query that does not include the shard key in the `WHERE` clause (e.g., `SELECT * FROM users WHERE status = 'active'`), the router or proxy has no idea which physical shard contains the matching rows. It is forced to **fan-out** the query to **every single active shard** in the cluster, aggregate the results, and sort them in memory. This is highly inefficient and negates the scalability advantages of sharding.
  - **The Optimizations:**
    - **Enforce Shard Keys in Queries:** Always structure API and repository queries to include the shard key (e.g., fetching user data using `user_id` instead of a generic email search).
    - **Maintain a Secondary Index Look-Up Table:** Maintain a fast key-value store (like Redis) or a lookup table on a single metadata server that maps non-shard-key fields to the correct shard key (e.g., mapping `email -> user_id`). The app queries the lookup table first, retrieves the `user_id` shard key, and then makes a highly targeted query directly to the correct shard.
    - **Materialized Views:** Duplicate (de-normalize) critical fields into other tables that are sharded by a different key, allowing direct reads without cross-shard routing.