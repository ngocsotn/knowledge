# SQL Concurrency & Locking

Detailed guide covering database locks, lock escalations, Optimistic vs. Pessimistic concurrency, and distributed locking.

## Locking Primitives

### 1. Row Lock vs. Table Lock
- **Row Lock:** Locks a specific target row inside a table. (Maximizes concurrency; higher lock metadata RAM overhead).
- **Table Lock:** Locks the entire physical table. (Minimizes lock metadata; blocks all concurrent writes).

### 2. Pessimistic Locking (`SELECT ... FOR UPDATE`)
Explicitly locks the selected rows at the database level. Any concurrent transactions attempting to modify or lock these rows will block/wait until the current transaction commits or rolls back.
- **Best Fit:** High contention environments where collisions are highly frequent.

### 3. Optimistic Locking (Version Check)
Does not lock any database rows. Instead, rows are stored with a version integer or timestamp. During update:
`UPDATE orders SET status = 'shipped', version = 3 WHERE id = 101 AND version = 2;`
If the update returns `0 affected rows`, a collision occurred. The transaction aborts and retries.
- **Best Fit:** Low contention environments; minimizes locking overhead.

### 4. Distributed Lock
Managing mutual exclusion across distinct, decoupled physical servers (e.g., using Redis Redlock, or ZooKeeper ephemeral nodes).

## Interview Questions & Answers

### Q1: What is a Database Deadlock, and how do database engines detect and resolve it?
- **Answer:** A Deadlock occurs when two or more transactions hold locks that the other transactions need to proceed, creating a circular blocking dependency (e.g., TxA locks Row 1, TxB locks Row 2. TxA requests Row 2, TxB requests Row 1. Both block forever).
- **Detection & Resolution:** Database engines run a background lock-monitor thread that analyzes the active lock dependency graph. Upon detecting a circular cycle (deadlock), the engine forcibly **kills and rolls back** one of the transactions (usually the one that has written the least amount of WAL bytes), releasing its locks so the other transaction can complete.
