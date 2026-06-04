# Database Transactions & ACID Isolation

Comprehensive study guide covering database transactions, ACID properties, isolation levels, and concurrency anomalies.

---

## 1. Meaning & Core Concepts

A database transaction is a logical unit of processing that contains one or more database operations (reads and writes). The primary goal is to ensure database integrity under concurrent access and system failures.

---

## 2. ACID Isolation Levels

The **I** in ACID (Isolation) guarantees how concurrent transactions see changes made by one another. The SQL standard defines four isolation levels, which trade off consistency against concurrency performance.

```
Read Uncommitted      Read Committed      Repeatable Read       Serializable
┌────────────────┐   ┌────────────────┐   ┌────────────────┐   ┌────────────────┐
│ Lowest safety. │   │ Blocks reading │   │ Reads same raw │   │ Strict locks.  │
│ Reads uncom-   │   │ uncommitted    │   │ multiple times │   │ Sequential execution│
│ mitted data.   │   │ writes.        │   │ safely.        │   │ representation.│
└────────────────┘   └────────────────┘   └────────────────┘   └────────────────┘
```

### Concurrency Anomalies (The Bugs we Defend Against)
1. **Dirty Read:** Transaction A reads data modified by Transaction B before Transaction B commits. If B rolls back, A operates on invalid, phantom data.
2. **Non-Repeatable Read:** Transaction A reads a row. Transaction B updates that row and commits. Transaction A re-reads the row and sees different values.
3. **Phantom Read:** Transaction A executes a range query (e.g., count active users). Transaction B inserts a new row matching that range and commits. Transaction A re-runs the range query and sees new "phantom" rows.

---

## 3. Isolation Levels vs. Anomalies Table

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Mechanism / Overhead |
| :--- | :---: | :---: | :---: | :--- |
| **Read Uncommitted** | Allowed | Allowed | Allowed | No locks. Extremely fast but unsafe. |
| **Read Committed** | **Prevented** | Allowed | Allowed | Reads only committed data. (PostgreSQL/SQL Server default). |
| **Repeatable Read** | **Prevented** | **Prevented** | Allowed / Prevented | Reads a snapshot frozen at transaction start. (MySQL default; PostgreSQL prevents Phantoms here too). |
| **Serializable** | **Prevented** | **Prevented** | **Prevented** | Full serialization. Strict locking or optimistic verification. Slowest. |

---

## 4. Isolation Enforcement: Locking & MVCC

Modern databases use two primary concurrency architectures:

### 1. Two-Phase Locking (2PL - Pessimistic)
* **Mechanism:** Transactions acquire shared (read) or exclusive (write) locks on resources and hold them until execution ends.
* **Trade-off:** High latency. Readers block writers, and writers block readers, leading to frequent deadlocks.

### 2. Multi-Version Concurrency Control (MVCC - Optimistic)
* **Mechanism:** Standard in PostgreSQL, MySQL (InnoDB), and Oracle. Writers do not lock out readers. Instead, when data is modified, the database creates a new, timestamped version of the row on disk.
* **Why it is fast:** Readers read a historical, immutable snapshot of the data, completely avoiding locks on writers. "Readers never block writers, and writers never block readers."

---

## 5. Multi-Version Concurrency Control (MVCC) Under the Hood

### A. Row Version Metadata (PostgreSQL xmin/xmax vs. MySQL Undo Logs)

Relational engines implement MVCC differently, each with major architectural trade-offs:

#### 1. PostgreSQL (Append-Only Tuples)
* Every physical table row (tuple) contains hidden metadata columns:
  * `xmin`: The transaction ID (TxID) that inserted/created this version of the row.
  * `xmax`: The transaction ID that deleted or updated (which is a delete + insert) this version.
* **Update Process**: PostgreSQL does not overwrite the row. It writes a completely new tuple to disk. It sets the `xmax` of the old tuple to the current TxID and sets the `xmin` of the new tuple to the current TxID.
* **The Drawback (Vacuuming)**: Since updated/deleted tuples are left on disk (known as **Dead Tuples**), PostgreSQL requires a background process called **VACUUM** (or Autovacuum) to scan tables, clean up unneeded old tuple versions, and reclaim disk space. High update volume can trigger "Vacuum bloat" and degrade performance.

#### 2. MySQL InnoDB (In-Place Updates with Undo Logs)
* InnoDB updates rows **in-place** to avoid physical table bloat.
* **Update Process**: It writes the new row directly inside the table page, but writes the old version of the row to a separate sequential buffer called the **Undo Log** (or Rollback Segment).
* Each row header stores a `roll_ptr` pointer pointing to its previous version in the Undo Log.
* **The Benefit**: Table files do not suffer from tuple bloat; dead versions are stored sequentially and purged automatically when no active transactions need them.

### B. Read Views and Visibility Rules
When an MVCC transaction begins, the database generates a **Read View (Active Transaction List Snapshot)**.
* Any changes made by transactions that committed *before* our Read View was created are **visible**.
* Any changes made by transactions that are still active or started *after* our Read View was created are **invisible**.
* During execution, the engine traverses the version chain (using `xmin/xmax` or the Undo Log `roll_ptr`) until it finds the first version that is visible to our Read View.

---

## 6. SQL Row-Level Locking (Pessimistic Locking)

When optimistic MVCC is insufficient to prevent race conditions, databases use explicit lock mechanisms:

### A. Lock Types and Modes
1. **Shared Locks (S - Read Lock)**:
   * Multiple transactions can hold S-locks on the same row simultaneously.
   * Prevents other transactions from writing to the row.
   * *SQL Syntax*: `SELECT ... FOR SHARE` (PostgreSQL) or `SELECT ... LOCK IN SHARE MODE` (MySQL).
2. **Exclusive Locks (X - Write Lock)**:
   * Only one transaction can hold an X-lock on a row.
   * Prevents other transactions from reading or writing.
   * *SQL Syntax*: `SELECT ... FOR UPDATE`.
3. **Intent Locks (IS / IX - Table Level)**:
   * Before acquiring a row-level S or X lock, a transaction must acquire an **Intent Shared (IS)** or **Intent Exclusive (IX)** lock on the *table*.
   * This allows the database to check if a table-level lock can be granted without scanning millions of individual rows to see if any are locked.

### B. Row Lock Contention & Escalation
* **Contention**: Under high write volume, many threads waiting for the same row-level exclusive lock (e.g., updating a hot merchant account balance) will queue up. This depletes the application connection pool, spiking latency.
* **Lock Escalation**: In some databases (like SQL Server), if a query acquires too many individual row locks (usually > 5,000), the engine automatically escalates them into a single table-level lock to save memory—completely blocking all concurrent writes. (PostgreSQL and MySQL InnoDB do *not* escalate locks; they can hold millions of row locks without memory overhead).

---

## 7. Advanced Database Race Conditions (Anomalies)

Even under Repeatable Read isolation, databases are vulnerable to two sophisticated race conditions:

### A. Lost Update Anomaly

```
Tx 1 (Balance: 100)                      Tx 2 (Balance: 100)
 │                                        │
 ├─── Read Balance (100)                  │
 │                                        ├─── Read Balance (100)
 ├─── Add 50 locally (150)                │
 ├─── Commit Write (150)                  │
 │                                        ├─── Add 20 locally (120)
 │                                        └─── Commit Write (120)  <─── Overwrites Tx 1!
```

* **The Problem**: Two transactions concurrently read the same record, perform separate calculations, and write back updates. The second transaction's write silently overwrites the first transaction's work, losing the balance update.
* **The Solution**:
  1. **Pessimistic Locking**: Use `SELECT balance FROM accounts WHERE id = 1 FOR UPDATE` to block Tx 2 until Tx 1 commits.
  2. **Optimistic Concurrency Control (OCC)**: Add a `version` column.
     `UPDATE accounts SET balance = 150, version = version + 1 WHERE id = 1 AND version = current_version;`
     If another transaction updated the row first, the version check fails, and we retry.

### B. Write Skew Anomaly (The Silent Serializable Killer)
Write Skew is a highly technical anomaly that occurs under **Repeatable Read** but is prevented under **Serializable**.

* **The Invariant**: A medical clinic requires at least one doctor to be active on call.
* **The Starting State**: Both Doctor A and Doctor B are currently active (On Call count = 2).

```
Tx 1 (Doctor A)                          Tx 2 (Doctor B)
 │                                        │
 ├─── SELECT COUNT(*) On Call (Gets 2)     │
 │    (Count >= 2, safe to check off)     ├─── SELECT COUNT(*) On Call (Gets 2)
 │                                        │    (Count >= 2, safe to check off)
 ├─── UPDATE Doc A SET status = 'Offline'  │
 ├─── Commit                              ├─── UPDATE Doc B SET status = 'Offline'
 │                                        └─── Commit
```

* **The Disaster**: Both transactions commit successfully because they updated *different, non-overlapping rows* (Tx 1 updated Doc A, Tx 2 updated Doc B). However, the invariant is violated: **zero** doctors are now on call.
* **The Solution**:
  1. **Pessimistic Locking**: Force row-level locking on the shared condition during query: `SELECT * FROM doctors WHERE status = 'On Call' FOR UPDATE`.
  2. **Serializable Isolation**: Force the engine to monitor read sets and abort one of the transactions on validation failure.

---

## 8. Popular Interview Questions & High-Impact Answers (Extended)

### Q4: Explain the difference between PostgreSQL and MySQL InnoDB in how they implement MVCC. What are the operational implications of each?
* **Answer:** PostgreSQL implements MVCC using **Append-Only Tuples**: updating a row creates a completely new physical row on disk and marks the old one as dead. This requires active **VACUUMing** to reclaim space, leading to table bloat and high disk write overhead if not tuned. MySQL InnoDB uses **In-Place Updates with Undo Logs**: the table row is overwritten directly, and the historical version is saved sequentially to a shared Undo Log file. InnoDB avoids table bloat, but long-running transactions can cause the Undo Log space to expand dramatically as the database must preserve history chains.

### Q5: What is "Write Skew", and why does the standard "Repeatable Read" isolation level fail to prevent it?
* **Answer:** Write Skew is an anomaly where two concurrent transactions read the same data, verify a shared business invariant, update *different, non-overlapping rows*, and commit. Because they mutate different records, row-level locks do not conflict, and under Repeatable Read, both transactions can commit successfully—violating the business invariant. Repeatable Read fails to prevent Write Skew because it only guarantees that *existing* rows read by a single transaction will not change; it does not coordinate locks or serialize across independent rows based on shared query constraints.

### Q6: [Struggle Question] Under high-throughput systems, how do you prevent "Hotspot Lock Contention" on a single database row (e.g., a globally shared balance or stock ledger)?
* **Answer:** 
  1. **Row Sharding / Distributed Counters**: Instead of one row storing `balance = 1000000`, split the ledger into $N$ separate rows (shards), e.g., `balance_shard_1`, `balance_shard_2`, etc. Transactions randomly update one of the shards, reducing lock contention by a factor of $N$. The true balance is computed using a fast `SUM(balance)` range scan.
  2. **Async Batching (Write-Behind)**: Queue updates in an in-memory buffer (like Redis or Kafka). Periodically merge/aggregate the increments (e.g., 10,000 increments of +1) and run a single consolidated database update: `UPDATE counter SET value = value + 10000`.
  3. **Queue-Based Serializing**: Route all writes to the hot row through a single-threaded queue worker, converting concurrent parallel lock struggles into an orderly, non-blocking sequential processing stream.


---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between Non-Repeatable Read and Phantom Read?
* **Answer:** A **Non-Repeatable Read** applies to a **single row**: a concurrent transaction updates or deletes that exact row, causing subsequent reads to see changed attributes. A **Phantom Read** applies to a **range query**: a concurrent transaction inserts new rows matching the filter range, causing subsequent range queries to return extra, newly created records.

### Q2: How does Multi-Version Concurrency Control (MVCC) solve the reader-writer blocking problem?
* **Answer:** Under traditional locking, a write transaction blocks all read transactions on the target row to prevent dirty reads. **MVCC** solves this by keeping multiple versions of a row concurrently. When a write occurs, a new version is created while the old version is preserved. Concurrent readers are directed to read the old, consistent version matching their transaction start timestamp, enabling concurrent reads and writes without lock-based blocking.

### Q3: What is a Database Deadlock, and how does the engine resolve it?
* **Answer:** A deadlock occurs when two or more transactions hold locks on resources the other transaction needs to proceed, creating a circular dependency block (e.g., Tx 1 locks Row A and waits for Row B; Tx 2 locks Row B and waits for Row A). Relational engines resolve this by running active **Deadlock Detection** background threads that trace lock-dependency cycles. When a cycle is detected, the engine forcibly aborts one of the transactions (the "deadlock victim"), rolling back its changes and releasing its locks so the surviving transaction can proceed.
