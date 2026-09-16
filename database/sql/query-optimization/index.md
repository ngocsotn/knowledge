# SQL Query Optimization & Tuning

Guide covering EXPLAIN analysis, query execution plans, query planner, and identifying hardware bottlenecks.

The primary diagnostic tool for slow relational database queries is the `EXPLAIN` (or `EXPLAIN ANALYZE`) command.

* **EXPLAIN:** Shows the query optimizer's estimated execution plan, cost estimates, and indexing choices without actually running the query.
* **EXPLAIN ANALYZE:** Executes the query, returning the actual execution time, disk page reads, and loop counts alongside the initial estimates.

### High-Risk Execution Plan Markers (What to Fix)
1. **Seq Scan (Sequential/Full Table Scan):**
   * *Meaning:* The engine is reading the entire table page-by-page from disk.
   * *Fix:* Add an index to the filtered columns in the `WHERE` or `JOIN` clauses.
2. **Filter / Hash Join on Unindexed Fields:**
   * *Meaning:* The engine is dynamically building a hash table in memory to resolve a join because no foreign key index exists.
   * *Fix:* Create indexes on foreign key relationship columns.
3. **Filesort / Disk Sort:**
   * *Meaning:* The database lacks a pre-sorted index to handle an `ORDER BY` statement, forcing it to sort records in slow disk space.
   * *Fix:* Create a composite index that covers the filtering column and the sorting column.

---

## 2. Advanced Indexing Strategies

### A. Index Types
- **B-Tree (Balanced Tree)**: The default index type. Excellent for range queries (`>`, `<`, `BETWEEN`) and equality lookups. Keeps data sorted and balanced.
- **Hash Index**: Limited to equality lookups (`=`). Faster than B-Tree for pure exact-match checks but does not support range scans or sorting.
- **GIN (Generalized Inverted Index)**: Indexes composite values where a single row can have multiple keys (e.g., arrays, JSONB, text lexemes). Maps elements back to document rows.
- **GiST (Generalized Search Tree)**: Used for geometric shapes, IP address ranges, and full-text search. Great for "contains" or "intersects" conditions.

### B. Partial Indexes
A **partial index** is built over a subset of rows defined by a conditional filter:
```sql
CREATE INDEX idx_unresolved_tasks ON tasks (created_at) 
WHERE status = 'pending';
```
- **Performance/Storage Advantage**: The index remains extremely small because it only indexes rows matching the filter condition. Lookups on pending tasks are extremely fast, and database memory is conserved.

### C. Covering Indexes (Index-Only Scan)
A covering index allows the database to resolve an entire query completely within the index pages without having to load any pages from the heap (the main table space).
```sql
CREATE INDEX idx_users_email_include_name ON users (email) 
INCLUDE (name);
```
- **How it works**: The index is sorted by `email`. The payload `name` is stored at the leaf nodes but is not part of the sorted tree structure. If you run `SELECT name FROM users WHERE email = 'test@example.com'`, the database runs an **Index-Only Scan**, completely skipping heap disk I/O.

---

## 3. Transactional Locks & Concurrency Control

Relational databases use locking mechanisms to maintain ACID isolation guarantees, but incorrect locking leads to extreme latency and contention.

### A. Lock Levels
- **Row-Level Locks**: Shared (`FOR SHARE`) or Exclusive (`FOR UPDATE`). Restricts access to a specific row.
- **Table-Level Locks**: Prevents schema or massive data changes on an entire table.
- **Lock Escalation**: When a database runs out of memory for millions of individual row locks, it may automatically convert them into a single table lock, completely blocking concurrency.

### B. Pessimistic vs. Optimistic Locking
- **Pessimistic Locking**: Prevents concurrent updates by locking the row immediately upon reading:
  ```sql
  -- Blocks other transactions trying to select/update this row until this transaction commits
  SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;
  ```
  - *Best for*: High-contention environments where conflicts are highly likely.
- **Optimistic Locking**: Assumes conflicts are rare. Uses a `version` column to check for changes before committing:
  ```sql
  -- Application reads version = 5
  -- Application attempts write:
  UPDATE accounts SET balance = 150, version = 6 
  WHERE id = 1 AND version = 5;
  ```
  - *Best for*: Read-heavy, low-contention environments. If version check fails, the application retries the transaction.

### C. Deadlocks
A **deadlock** occurs when transaction A holds a lock on row 1 and requests a lock on row 2, while transaction B holds a lock on row 2 and requests a lock on row 1. Both are permanently blocked.
- **Prevention Rules**:
  1. **Consistent Lock Ordering**: Always lock rows/resources in the exact same sequence across all application code threads.
  2. **Short-Lived Transactions**: Minimize the time locks are held by keeping transactions small.
  3. **Low Isolation Levels**: Use the lowest acceptable isolation level (e.g., Read Committed over Serializable) where possible.

---

## 4. Connection Pooling Tuning Parameters

Establishing a TCP connection to a database server is highly expensive: it requires a 3-way handshake, TLS exchange, and backend authentication/process initialization.

- **Without Connection Pooling:** The application opens a new connection on every incoming API request and closes it immediately after. Under load, this exhausts OS file descriptors and overwhelms database CPU.
- **With Connection Pooling (e.g., PgBouncer, HikariCP, pgxpool):** The application maintains a warm pool of active, open database connections.

### Key Tuning Knobs:
1. **MaxOpenConns (Max Connections)**: The absolute cap of active connections. Setting this too high causes thread starvation, high RAM consumption, and excessive CPU context switching on the database server. A database server with 8 cores typically performs best with a small, tightly bounded pool (e.g., 20-50 connections).
2. **MaxIdleConns (Max Idle Connections)**: The number of unused connections kept alive in the pool. Set this equal to or slightly lower than MaxOpenConns to avoid repeated connection opening/closing churn during bursty traffic.
3. **ConnMaxLifetime (Max Lifetime)**: The maximum age of a connection. Periodically recycling connections prevents memory leaks and ensures dead connections are pruned.
4. **ConnMaxIdleTime (Idle Timeout)**: Closes connections that have been idle for too long, freeing up server resources.

---

## 5. High-Impact Performance Patterns

### 1. N+1 Query Problem (Common ORM Pitfall)
* *The Bug:* A query retrieves a list of $N$ parent records, and the application iterates through them, running an individual SQL query for each record to fetch its child relation (resulting in $1 + N$ SQL queries).
* *The Fix:* Use eager loading or join queries to fetch all parents and children in a single, batched operation:
  ```sql
  -- Efficient Join
  SELECT posts.*, comments.* FROM posts LEFT JOIN comments ON posts.id = comments.post_id;
  ```

### 2. Database Normalization vs. Denormalization Trade-Offs

While Third Normal Form (3NF) reduces redundancy, eliminates modification anomalies, and ensures data integrity, highly normalized schemas require complex, multi-table joins. At high read scale, performing these joins repeatedly on every request degrades database performance and spikes CPU usage. 

To bypass join overhead, systems deploy selective, deliberate **Denormalization**—intentionally storing redundant or pre-computed data to trade write speed, storage space, and write integrity for ultra-fast, single-table reads.

#### A. Common Denormalization Scenarios
1. **Caching Aggregates:** Storing a count or sum directly on the parent row (e.g., `total_comments` on a `posts` table) rather than running `SELECT COUNT(*)` on every page load.
2. **Pre-joining (Redundant Column Storage):** Copying a frequently read field from a lookup table (e.g., storing the `username` directly in the `comments` table alongside the `user_id`) to display comment feeds in a single SQL select without joining the massive `users` table.
3. **Historical Snapshots:** Duplicating current price and address details at the moment of checkout onto the `order_items` table (e.g., `checkout_price`) so the record remains frozen, even if the product's catalog price changes in the future.

#### B. The Cost: Modification Anomalies (The Bugs of Denormalization)
When you store redundant data, you face three primary database anomalies:
* **Update Anomaly:** If a user changes their `username`, and that username is stored redundantly across millions of `comments` rows, failure to update *every single occurrence* leaves the database in an inconsistent state.
* **Insert Anomaly:** You cannot store a piece of information without creating a parent record (e.g., if you store student and course info in one table, you cannot store a course's details until at least one student registers for it).
* **Delete Anomaly:** Deleting a row deletes unrelated data that is tightly coupled in the same record (e.g., deleting the last student registered for a class accidentally deletes the entire course description).

#### C. Enterprise Mitigation Patterns (How to Keep Denormalized Data Consistent)
* **Materialized Views (RDBMS Native):** Pre-compute and store the results of complex join queries physically on disk. Periodically refresh the view (`REFRESH MATERIALIZED VIEW` in Postgres) or use database triggers to update it incrementally.
* **Database Triggers:** Write native triggers that automatically update the redundant values inside a single database transaction. For example, inserting a comment triggers:
  ```sql
  UPDATE posts SET total_comments = total_comments + 1 WHERE id = NEW.post_id;
  ```
* **Application-Level Double Writes:** In the codebase, wrap both writes in a single local SQL transaction block.
* **CQRS (Read-Query Segregation):** Keep the write model fully normalized, and stream updates asynchronously via CDC (Change Data Capture) or Kafka to a denormalized read database (like Elasticsearch or Redis).

---

## 6. Popular Interview Questions & High-Impact Answers

### Q1: What is the N+1 query problem, and how do you diagnose and fix it?
* **Answer:** The N+1 query problem occurs when an application executes one query to fetch parent records (1 query) and then executes an additional query for *each* returned parent row to fetch its child relation ($N$ queries), generating massive database network overhead. Diagnose this by analyzing database query logs (or APM tracing) and observing repeating, identical queries with changing IDs. Fix it by utilizing **Eager Loading** (fetching relations in a single, batched `IN` query) or writing an explicit `JOIN` in a single SQL operation.

### Q2: What is the difference between EXPLAIN and EXPLAIN ANALYZE?
* **Answer:** `EXPLAIN` provides the static query execution plan generated by the database optimizer, using cached table statistics to estimate cost, rows, and width without executing the SQL. `EXPLAIN ANALYZE` actually runs the query on the database engine, returning the actual execution times, CPU loops, and memory page counts. Use `EXPLAIN ANALYZE` for precise testing, but use caution with destructive commands (like `UPDATE` or `DELETE`) as they will commit unless executed in a transaction that is rolled back.

### Q3: Why is Connection Pooling critical for database scalability? How do you size the pool?
* **Answer:** Establishing a database connection requires a complete network handshake, authentication verification, and server thread/process allocation, which takes milliseconds of latency and high CPU overhead. A connection pool keeps a fixed set of open connections active. It allows application threads to check out connections instantly and return them immediately, eliminating connection establishment latency and capping the maximum simultaneous connection load on the database engine, avoiding crash thresholds.
* **Pool Sizing Rule**: Sizing should be surprisingly small. Formula from Postgres developers:
  $$\text{Connections} = ((\text{Core Count} \times 2) + \text{Spindle Count})$$
  Adding too many connections causes high disk I/O wait times and CPU thread context-switching overhead, degrading global database throughput.

### Q4: [SQL Denormalization Struggle] When you selectively denormalize a database to improve read performance, how do you handle update synchronization at scale, and what are the major pitfalls of each approach?
* **Answer:** 
  You can synchronize denormalized data using three primary patterns:
  1. **Database Triggers (Synchronous):** Write database-level triggers that update redundant columns on every insert or update.
     * *Pitfall:* Increases write latency, creates lock contention on hot parent rows (e.g., updating a post counter), and hides business logic from application code, making debugging hard.
  2. **Application Transaction-Level Writes (Dual Write):** Wrap both writes (normalized write + denormalized update) in a single application transaction.
     * *Pitfall:* Highly fragile. If another developer writes a new feature that updates the database without writing to the redundant field, the database immediately drifts.
  3. **Event-Driven Outbox / CDC (Asynchronous):** Update the normalized table, write an outbox event, and let an async worker or CDC connector (Debezium) update the denormalized read models asynchronously.
     * *Pitfall:* Introduces **eventual consistency**. The application must tolerate a delay where a user updates their profile name but sees their old name on existing comments for a few seconds. This is the standard pattern for high-scale enterprise SaaS systems.

### Q5: An API normally responds in 100 ms, but suddenly takes 5–10 seconds while database CPU reaches 95%. What do you investigate first, and how do you fix it?

* **Answer:** Treat the database as the leading bottleneck, then prove which workload consumes its CPU before changing indexes or adding hardware. Compare a healthy time window with the incident window: request rate, query latency, rows examined, lock waits, connection-pool wait, cache hit rate, and database CPU.

  ```mermaid
  flowchart TD
      A[API latency rises to 5-10 seconds] --> B[Confirm database CPU and wait metrics]
      B --> C[Inspect slow query logs and top query fingerprints]
      C --> D{What changed?}
      D -->|Plan or index regression| E[EXPLAIN ANALYZE and fix index or query]
      D -->|Query count spike| F[Find N+1 or sudden traffic]
      D -->|Sessions waiting| G[Inspect locks and long transactions]
      D -->|Pool wait| H[Fix pool sizing, leaks, or slow queries]
      E --> I[Deploy safely and measure]
      F --> I
      G --> I
      H --> I
  ```

  Investigate in this order:

  1. **Slow query logs and database activity.** Find top queries by total time, mean time, calls, rows read, and temporary files. Group normalized query fingerprints instead of inspecting only raw SQL. Example: one `SELECT ... WHERE user_id = ?` taking 4 seconds and running 20,000 times can dominate CPU even when each request looks small.
  2. **Query execution plans.** Run `EXPLAIN` first, then carefully run `EXPLAIN ANALYZE` in a safe environment or read-only transaction. Compare estimated rows with actual rows. A plan that changed from an index scan to a sequential scan can indicate a missing index, stale statistics, data growth, or changed parameter selectivity.
  3. **Missing or inefficient indexes.** Check filters, joins, sort keys, and composite-index column order. Example:

     ```sql
     CREATE INDEX CONCURRENTLY idx_orders_user_status_created
     ON orders (user_id, status, created_at DESC);
     ```

     Add an index only when the workload and plan justify it. Indexes also increase write cost, storage, vacuum work, and database CPU.
  4. **N+1 queries.** Inspect APM traces and query counts per API request. One endpoint loading 100 orders with one query per order creates 101 queries instead of one batched query. Replace the loop with eager loading, `JOIN`, or `WHERE id IN (...)`.
  5. **Connection pool exhaustion.** Check active connections, idle connections, pool wait time, leaked connections, and transaction duration. A full pool can make API requests wait even when application CPU is normal. Do not fix this by blindly increasing pool size; too many database sessions can increase contention and CPU.
  6. **Lock contention and long transactions.** Inspect blocked sessions, lock holders, deadlocks, idle transactions, and recent migrations or bulk writes. A transaction holding locks for minutes can make many queries wait and create a latency cascade.
  7. **Sudden query-volume increase.** Check traffic, retries, scheduled jobs, deployments, feature flags, bot traffic, and missing pagination. A retry storm can multiply database work while API servers still look healthy.

  Fix based on evidence:

  - Kill or pause clearly runaway jobs only after confirming ownership and impact.
  - Roll back a deployment or query-plan regression when incident timing matches.
  - Add or correct indexes, then verify write and storage cost.
  - Fix N+1 queries, add pagination, and batch reads.
  - Refresh statistics when estimates are stale; rebuild indexes only when evidence supports corruption or severe bloat.
  - Shorten transactions and resolve lock ordering or long-running writes.
  - Bound retries, add backoff, and prevent retry storms.
  - Tune connection pools to database capacity, not API instance count alone.
  - Cache stable read results only after fixing inefficient database work; caching should not hide correctness or freshness bugs.

  Do not start by adding application servers: normal API CPU and memory with database CPU at 95% points to a database workload, plan, lock, or connection problem. Do not add an index blindly, because an index can fail to match the query, increase write cost, or leave the true N+1 or lock issue unresolved. After each change, compare p50/p95/p99 latency, database CPU, query count, plan, lock waits, error rate, and correctness.
