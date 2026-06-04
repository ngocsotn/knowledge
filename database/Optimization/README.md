# Database Query Optimization & Performance Tuning

Comprehensive interview study guide covering database tuning, execution plan analysis, advanced indexing strategies, locking mechanisms, connection pooling, and scaling tactics.

---

## 1. Analyzing Performance with EXPLAIN

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

### 2. Database Normalization Trade-Offs
* While 3NF (Third Normal Form) reduces redundancy and ensures data integrity, highly normalized schemas require complex, multi-table joins that degrade read performance under heavy load. Selective, deliberate **denormalization** can bypass join overhead.

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

