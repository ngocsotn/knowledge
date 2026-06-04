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

## 5. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between Non-Repeatable Read and Phantom Read?
* **Answer:** A **Non-Repeatable Read** applies to a **single row**: a concurrent transaction updates or deletes that exact row, causing subsequent reads to see changed attributes. A **Phantom Read** applies to a **range query**: a concurrent transaction inserts new rows matching the filter range, causing subsequent range queries to return extra, newly created records.

### Q2: How does Multi-Version Concurrency Control (MVCC) solve the reader-writer blocking problem?
* **Answer:** Under traditional locking, a write transaction blocks all read transactions on the target row to prevent dirty reads. **MVCC** solves this by keeping multiple versions of a row concurrently. When a write occurs, a new version is created while the old version is preserved. Concurrent readers are directed to read the old, consistent version matching their transaction start timestamp, enabling concurrent reads and writes without lock-based blocking.

### Q3: What is a Database Deadlock, and how does the engine resolve it?
* **Answer:** A deadlock occurs when two or more transactions hold locks on resources the other transaction needs to proceed, creating a circular dependency block (e.g., Tx 1 locks Row A and waits for Row B; Tx 2 locks Row B and waits for Row A). Relational engines resolve this by running active **Deadlock Detection** background threads that trace lock-dependency cycles. When a cycle is detected, the engine forcibly aborts one of the transactions (the "deadlock victim"), rolling back its changes and releasing its locks so the surviving transaction can proceed.
