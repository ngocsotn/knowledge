# SQL Concurrency, Locking & MVCC

Comprehensive study guide covering database locks, lock compatibility, transaction isolation levels, anomalies, Multi-Version Concurrency Control (MVCC) in PostgreSQL vs. InnoDB, and distributed locking.

---

## 1. Locking Primitives & Compatibility Matrix

To maintain data integrity under concurrent access, database engines utilize locks at different granularities (Row, Page, Table) and modes (Shared, Exclusive, Intent).

### Lock Modes
* **Shared (S) Lock**: Acquired for read operations (`SELECT`). Multiple transactions can hold S locks on the same resource simultaneously.
* **Exclusive (X) Lock**: Acquired for write operations (`INSERT`/`UPDATE`/`DELETE`). Only one transaction can hold an X lock, blocking all other reads and writes.
* **Intent Locks (IS / IX)**: Acquired at the table level before obtaining row-level S or X locks. They signal that a transaction holds or intends to acquire locks within that table, preventing table-wide lock modifications.

### Lock Compatibility Matrix

```
  Requested Mode │ Shared (S) │ Exclusive (X) │ Intent Shared (IS) │ Intent Exclusive (IX)
  ───────────────┼────────────┼───────────────┼────────────────────┼──────────────────────
  Shared (S)     │    Compatible   │   BLOCKED     │     Compatible     │   BLOCKED
  Exclusive (X)  │    BLOCKED     │   BLOCKED     │     BLOCKED        │   BLOCKED
  Intent S (IS)  │    Compatible   │   BLOCKED     │     Compatible     │   Compatible
  Intent X (IX)  │    BLOCKED     │   BLOCKED     │     Compatible     │   Compatible
```

### Optimistic vs. Pessimistic Locking
* **Pessimistic Locking (`SELECT ... FOR UPDATE`)**: Explicitly locks target rows at the database level. Blocks concurrent writes until the transaction completes.
  - *Best Fit*: High-contention workloads where transaction collisions are highly frequent.
* **Optimistic Locking (Application-Level Version Check)**: Does not lock database rows. Instead, updates assert that the record's version has not changed:
  ```sql
  UPDATE orders SET status = 'shipped', version = 3 WHERE id = 101 AND version = 2;
  ```
  - *Best Fit*: Low-contention workloads; avoids locking overhead.

---

## 2. Transaction Isolation Levels & Anomalies

The SQL-92 standard defines four transaction isolation levels, which trade off performance (concurrency) for strict consistency.

### Concurrency Anomalies
1. **Dirty Read**: Transaction A reads data modified by Transaction B before B commits. If B rolls back, A's read is invalid.
2. **Non-Repeatable Read (Fuzzy Read)**: Transaction A reads a row. Transaction B updates that same row and commits. Transaction A re-reads the row and gets different values.
3. **Phantom Read**: Transaction A executes a range query (e.g., count orders $> \$100$). Transaction B inserts a **new** row within that range and commits. Transaction A re-runs the range query and sees "phantom" rows.
4. **Write Skew**: A serialization anomaly where two concurrent transactions read overlapping data sets, make decisions based on those reads, and make non-conflicting writes that violate a global business invariant (e.g., checking if total account balance $> 0$ before withdrawing).

### Isolation Level vs. Anomalies Matrix

```
  Isolation Level  │ Dirty Reads │ Non-Repeatable Reads │ Phantom Reads │ Write Skew
  ─────────────────┼─────────────┼──────────────────────┼───────────────┼─────────────
  Read Uncommitted │   Allowed   │       Allowed        │    Allowed    │   Allowed
  Read Committed   │  Prevented  │       Allowed        │    Allowed    │   Allowed
  Repeatable Read  │  Prevented  │      Prevented       │  Prevented*   │   Allowed
  Serializable     │  Prevented  │      Prevented       │   Prevented   │  Prevented
```
*\*Note: PostgreSQL/InnoDB Repeatable Read prevents Phantom Reads natively using MVCC visibility checks or Next-Key locking, which is stricter than the SQL-92 standard.*

---

## 3. Multi-Version Concurrency Control (MVCC)

Modern databases avoid using locks for simple read operations (following the principle: *"Readers do not block Writers, and Writers do not block Readers"*). They achieve this using **Multi-Version Concurrency Control (MVCC)**, maintaining multiple historical versions of individual rows concurrently.

```
                  PostgreSQL MVCC (Append-Only)
      ┌──────────────────────────────────────────────────┐
      │ Row (xmin=100, xmax=105) ──► Old Version (Dead)  │
      ├──────────────────────────────────────────────────┤
      │ Row (xmin=105, xmax=0)   ──► New Version (Live)  │  <-- Writes append new rows
      └──────────────────────────────────────────────────┘

                  MySQL InnoDB MVCC (Undo Log)
      ┌─────────────────────┐       Undo Logs (Rollback Segment)
      │ Row (Roll_Ptr) ─────┼─────► [Old Version xid=100]
      └─────────────────────┘
         ▲ (Live Row in Table)
```

### A. PostgreSQL MVCC (Append-Only Storage)
* **Mechanism**: Every `UPDATE` is physically written as a brand-new row insertion (`INSERT`) on disk. Old rows are kept in place and marked with system columns:
  - `xmin`: Transaction ID (XID) of the creator.
  - `xmax`: Transaction ID of the updater/deleter (0 if active/live).
* **Visibility**: A transaction reading at Read Committed isolation only sees row versions where `xmin` is committed and `xmax` is either uncommitted or belongs to an active transaction.
* **Garbage Collection (VACUUM)**: Because updates append new records, PostgreSQL accumulates dead rows over time (**Bloat**). A background daemon (**Auto-Vacuum**) must continuously scan tables to prune dead row versions and mark free space pointers as reusable.

### B. MySQL InnoDB MVCC (Rollback Segment / Undo Logs)
* **Mechanism**: InnoDB updates rows **in-place** in the actual tables. It stores historical row versions sequentially inside a separate append-only file called the **Undo Log** (or Rollback Segment).
* **Visibility**: Every row contains a `roll_ptr` pointer to its previous version in the Undo Log. When a transaction needs to read a historical snapshot, the engine traverses the undo chain backward to dynamically reconstruct the row version that matches the transaction's read view.
* **Garbage Collection**: Once no active transactions need a specific Undo Log version, background **Purge Threads** clean up the obsolete Undo records, avoiding database table bloat.

---

## 4. Deadlocks & Distributed Locking

### Deadlock Detection & Resolution
A deadlock occurs when two or more transactions form a circular blocking lock dependency (e.g., TxA locks Row 1 and waits for Row 2; TxB locks Row 2 and waits for Row 1).
* **Detection**: Engines run a background lock-monitor thread that analyzes the lock dependency graph (**Wait-For Graph**).
* **Resolution**: Upon finding a cycle, the engine selects the transaction that has written the least amount of WAL bytes as the "victim," forcibly terminates and rolls it back, releasing its locks so other transactions can proceed.

### Distributed Locking Patterns
When coordinating locks across separate, decoupled physical nodes, standard database locks cannot help. 
* **Redis Redlock**: Implements locking by acquiring locks with a TTL from a majority (e.g., 3 out of 5) of independent Redis nodes.
* **Database Leases / Fencing Tokens**: Acquires a lock with an auto-incrementing integer token (fencing token). During writes to shared storage, the storage engine rejects any write carrying a fencing token lower than the latest committed token, preventing late-arriving clients (due to GC pauses or network delays) from corrupting state.

---

## 5. Highly Technical Interview Q&As

### Q1: What is a Database Deadlock, and how do database engines detect and resolve it?
- **Answer**: A Deadlock occurs when two or more transactions hold locks that the other transactions need to proceed, creating a circular blocking dependency (e.g., TxA locks Row 1, TxB locks Row 2. TxA requests Row 2, TxB requests Row 1. Both block forever).
- **Detection & Resolution**: Database engines run a background lock-monitor thread that analyzes the active lock dependency graph. Upon detecting a circular cycle (deadlock), the engine forcibly **kills and rolls back** one of the transactions (usually the one that has written the least amount of WAL bytes), releasing its locks so the other transaction can complete.

### Q2: Compare MVCC implementation in PostgreSQL vs. MySQL (InnoDB). What are the storage, performance, and garbage-collection trade-offs?
- **Answer**:
  - **PostgreSQL (Append-Only/Out-of-Place Updates)**:
    - *Storage*: Every update is a new physical row insertion on disk. Causes high **Table Bloat** and write amplification on indexes since all indexes must point to the new physical row location (partially mitigated by HOT - Heap-Only Tuples).
    - *Garbage Collection*: Relies on the **Auto-Vacuum** daemon to sweep tables and free space occupied by dead rows. High CPU/IO overhead during heavy write workloads.
    - *Reads/Writes*: Writes are slow due to page allocations. Reads are fast but can degrade if tables are bloated.
  - **MySQL InnoDB (In-Place Updates + Undo Logs)**:
    - *Storage*: Row updates are performed in-place. Index pointers remain stable because the row's physical address does not change. Historical versions are stored sequentially in **Undo Logs**.
    - *Garbage Collection*: Relies on **Purge Threads** to discard Undo Log blocks. Cleanups are fast and sequential.
    - *Reads/Writes*: Faster write throughput. Reads must traverse the undo chain to reconstruct historical states, introducing minor CPU overhead for long-running transactions reading deep history.

### Q3: How do databases implement the Serializable isolation level without relying on heavy table-wide locking?
- **Answer**: Modern engines avoid heavy table locking under Serializable isolation using two main strategies:
  1. **Strict Two-Phase Locking (SS2PL)**: Used by older engines. Transactions acquire Shared locks on reads and Exclusive locks on writes, holding all locks until the transaction commits. To prevent phantom reads, it uses **Range Locks** (locking the index keys and the gaps between keys) rather than locking the entire table.
  2. **Serializable Snapshot Isolation (SSI)**: Used by PostgreSQL. SSI is **optimistic**. Transactions run concurrently on standard MVCC snapshots without locks. However, the database tracks **SIREAD locks** (virtual, non-blocking lock flags on read keys and index gaps). If the engine detects a dependency cycle of read-write conflicts (e.g., TxA wrote to a key TxB read, and TxB wrote to a key TxA read) before commit, it aborts one of the transactions with a `40001` serialization failure.

### Q4: What is "write skew" and how does the Serializable isolation level prevent it?
- **Answer**: **Write Skew** is a serialization anomaly that occurs under the Repeatable Read isolation level.
  - *Scenario*: Imagine a medical on-call system with a rule: "At least one doctor must remain on call." Doctor A and Doctor B are both on call.
  - Concurrent Transactions:
    - TxA reads the database: "How many doctors are on call?" (Result: 2).
    - TxB reads the database: "How many doctors are on call?" (Result: 2).
    - TxA decides: "I can check out." Updates status of Doctor A to "off-call" and commits.
    - TxB decides: "I can check out." Updates status of Doctor B to "off-call" and commits.
  - *Result*: Both transactions commit because they did not write to the same rows. However, 0 doctors are now on call, violating the global system invariant.
  - *Serializable Prevention*: 
    - Under SSI (PostgreSQL), the engine registers virtual SIREAD locks on the doctors' status range read by both transactions. When TxA commits, the engine detects that TxB's concurrent write conflicts with TxA's active read range. The engine aborts TxB's transaction on commit, preventing the write skew.

### Q5: Why is the Redis Redlock algorithm controversial, and what are its security weaknesses?
- **Answer**: Redis Redlock (proposed by Salvatore Sanfilippo) is controversial because it relies on wall-clock time assumptions to enforce mutual exclusion across separate physical nodes. Martin Kleppmann published a famous critique detailing its core security weaknesses:
  1. **Clock Drift Vulnerability**: Redlock assumes all Redis servers increment time at identical rates. If one Redis node's system clock leaps forward (e.g., due to an NTP sync update), its lease lock will expire prematurely. A second client can then acquire the lock while the first client still believes it owns it, violating mutual exclusion.
  2. **Process Pauses (GC/Virtualization)**: If Client A acquires the Redlock, but then undergoes a long stop-the-world garbage collection pause (or hypervisor VM pause) that exceeds the lock's TTL, the Redis nodes will release the lock. Client B can then safely acquire the lock. When Client A wakes up, it continues its write to shared storage, corrupting the state.
  3. **The Solution**: Distributed consensus systems (like ZooKeeper or Etcd) or databases (like PostgreSQL) should utilize **Fencing Tokens** (monotonically increasing epoch counters). Every lock acquisition returns a fencing token. The storage engine enforces that any incoming write carrying a token lower than the latest committed token is rejected, guaranteeing safety regardless of clock drift or client process pauses.

