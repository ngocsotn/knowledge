# Database Storage Engine Internals

Comprehensive study guide covering physical database layout, page structures, buffer pools, Write-Ahead Logs (WAL), checkpoint mechanics, and crash-recovery algorithms.

---

## 1. Physical Layout: Pages & Slotted-Page Architecture

At the lowest level, database storage engines organize data into uniform binary blocks called **Pages** (or **Blocks**).
* **Page Sizes**: 
  * **PostgreSQL**: Default is $8\text{ KB}$.
  * **MySQL (InnoDB)**: Default is $16\text{ KB}$.
* **I/O Boundary**: Databases never read or write individual rows to disk. All disk/memory I/O operations are strictly performed on whole Pages.

### The Slotted-Page Structure
To handle variable-length rows (e.g., text, JSON) and avoid fragmentation, databases use a **Slotted-Page Architecture**:

```
┌───────────────────────────────────────────────────────────────┐
│ Page Header                                                   │
│ (LSN, free space pointers, flags, transaction visibility)     │
├───────────────────────────────────────────────────────────────┤
│ Line Pointers (Array of [Offset, Length])                     │
│ ┌─────────┬─────────┬─────────┐                               │
│ │ ptr #1  │ ptr #2  │ ptr #3  │...                            │
│ └────┬────┴────┬────┴────┬────┘                               │
├──────┼─────────┼─────────┼────────────────────────────────────┤
│      │         │         │                                    │
│      │         │         └──────────────┐                     │
│      ▼         └──────────────┐         │   (Free Space grows │
│                               │         │    downward)        │
│                               ▼         ▼                     │
│                                                               │
│                               ▲         ▲                     │
│                               │         │   (Tuples grow      │
│                               │         └─────────┐  upward)  │
│                               └─────────┐         │           │
│                                         │         │           │
├─────────────────────────────────────────┴─────────┴───────────┤
│ Tuple #3                                                      │
├───────────────────────────────────────────────────────────────┤
│ Tuple #2                                                      │
├───────────────────────────────────────────────────────────────┤
│ Tuple #1 (Raw Row Data)                                       │
└───────────────────────────────────────────────────────────────┘
```

* **Line Pointers (Slots)**: Grow **downward** from the top of the page. Each contains the exact offset and length of the corresponding row.
* **Raw Tuples**: Grow **upward** from the bottom of the page.
* **Free Space**: The gap in the middle. When inserting a new row, a new pointer is added at the top, and the raw tuple is written at the bottom of the free space. This layout allows safe in-place updates and variable row sizes.

---

## 2. Memory Buffer Pool & Eviction Strategies

Reading from spinning disks or SSDs is orders of magnitude slower than reading from RAM. To accelerate performance, databases allocate a large portion of RAM to the **Buffer Pool** (or **Shared Buffers** in PostgreSQL) to act as an in-memory page cache.

```
                   RAM (Buffer Pool)
             ┌─────────────────────────────┐
             │ Cached Page 1 │Cached Page 2│
             ├───────────────┼─────────────┤
             │ Cached Page 3 │ Cached Page4│
             └───────────────┴─────────────┘
                ▲                       │
           Read │ (Cache Hit: 0ms)      │ Checkpoint Write
                │                       ▼ (Dirty Page Flush)
             ┌─────────────────────────────┐
             │        Physical Disk        │
             │ [Page 1] [Page 2] [Page 3]  │
             └─────────────────────────────┘
```

### Read and Write Pathways
* **Reads**: When a query requests a row, the engine resolves its page ID.
  * **Cache Hit**: Page is already in the Buffer Pool. Instantly read.
  * **Cache Miss**: The engine loads the page from disk into the Buffer Pool.
* **Writes**: To keep write latency low, updates are **NOT** written directly to the main table files on disk. Instead, the page is modified directly in the RAM Buffer Pool and marked as a **Dirty Page**.

### Buffer Pool Eviction Algorithms
When the Buffer Pool is full and a new page must be loaded from disk, the engine must evict an existing page:
1. **LRU (Least Recently Used)**: Evicts the page that hasn't been accessed for the longest time. Vulnerable to "sequential scan pollution" (a full table scan loads thousands of unneeded pages once, evicting hot pages).
2. **Clock-Sweep (2Q / LIRS)**: Used by PostgreSQL/MySQL. Uses a "usage count" bit. A pointer sweeps through pages. If a page has a usage count $> 0$, the pointer decrements it and passes; if $0$, it evicts that page. This protects the cache from table-scan pollution.

---

## 3. The WAL (Write-Ahead Logging) Engine

If a page is modified in RAM but not yet flushed to disk, a sudden power failure or crash would result in catastrophic data loss. Writing table pages to disk on every transaction is impossible because table pages are scattered randomly across the disk, leading to slow **Random I/O**.

To solve this, databases write all modifications sequentially to an append-only on-disk file called the **Write-Ahead Log (WAL)** (or **Redo Log** in MySQL) *before* applying changes to any pages.

```
 Client ────► [ Write Transaction ]
                    │
                    ├── 1. Append sequentially (Sequential I/O - fast) ──► Write-Ahead Log (WAL)
                    │
                    └── 2. Modify page in RAM (Dirty Page in Memory)  ──► Buffer Pool
```

### Sequential I/O vs. Random I/O
* **Sequential I/O**: Writing to the end of a single pre-allocated file (WAL). Highly optimized by disk controllers ($100\text{x} - 1000\text{x}$ faster than random writes).
* **Random I/O**: Accessing separate physical sectors on disk to update actual tables. Done asynchronously in the background.

### Log Sequence Numbers (LSN)
Every single WAL record is stamped with a unique, monotonically increasing 64-bit integer called a **Log Sequence Number (LSN)**. Each database page header also stores the LSN of the last WAL record that modified it (`page_LSN`). This is critical for crash recovery.

---

## 4. Checkpoint Mechanics: Sharp vs. Fuzzy

**Checkpoints** are periodic sync events that flush "Dirty Pages" from the RAM Buffer Pool to physical table storage on disk. This limits the size of the WAL log and shortens database startup recovery times.

### A. Sharp Checkpoints
The engine halts all active transaction writes and flushes **every single dirty page** in the Buffer Pool to disk in one massive batch.
* **Downside**: Causes severe write stalls, latency spikes, and system choking (commonly called "checkpoint spikes").

### B. Fuzzy Checkpoints
Instead of flushing everything at once, a background process continuously writes small batches of dirty pages to disk over time. 
* **Mechanism**: The engine maintains a pointer in the WAL corresponding to the oldest dirty page in memory. During a checkpoint, it only writes pages older than a specific LSN threshold and records this safe LSN to the control file. It does not block active user transactions.

### C. Double-Write / Full-Page Write Protection
Because operating system page sizes (typically $4\text{ KB}$) are smaller than database page sizes ($8\text{ KB}$ or $16\text{ KB}$), a physical crash mid-write can result in a **Torn Page** (half the database page is written, the other half is stale, corrupting the page's checksum).
* **MySQL InnoDB (Doublewrite Buffer)**: InnoDB writes dirty pages to a sequential, contiguous disk area called the doublewrite buffer first, then flushes them to their actual table spaces. If a crash occurs mid-table-write, InnoDB restores the page from the doublewrite buffer.
* **PostgreSQL (Full-Page Writes)**: After a checkpoint, the first modification to a page causes PostgreSQL to write the **entire** page contents into the WAL log. Subsequent writes to that page write only the standard delta. During recovery, torn pages are restored by replaying the complete page image from the WAL.

---

## 5. Highly Technical Interview Q&As

### Q1: Why does a database write to a WAL file before modifying actual table files?
- **Answer**: Performance and Durability. Modifying actual table files requires **Random I/O** (writing to various scattered page blocks on disk), which is extremely slow. Writing to a WAL file is **Sequential I/O** (appending to the end of a single file), which is near-instantaneous. Writing transactions sequentially to WAL guarantees **Durability** (survives crashes) while keeping write latency minimal, deferring random table disk flushes to background checkpoint threads.

### Q2: How does the database recover from a sudden crash if some modified pages were not yet flushed to disk?
- **Answer**: It uses the **ARIES (Algorithms for Recovery and Isolation Exploiting Semantics)** recovery protocol. Upon reboot, the database scans its control file to find the last valid checkpoint LSN and executes a 3-phase process:
  1. **Analysis Phase**: Scans the WAL forward from the checkpoint to identify all dirty pages in memory and active transactions that were incomplete at the time of the crash.
  2. **Redo Phase (Repeat History)**: Scans forward and replays all log records to restore the database to the exact state it was in right before the crash. It compares the `page_LSN` on disk; if the page's LSN is equal to or greater than the WAL record's LSN, it skips the write, avoiding redundant disk I/O.
  3. **Undo Phase**: Scans backward and rolls back all transactions that were active but uncommitted when the crash occurred, writing compensating log records (CLRs) to ensure the database returns to a completely consistent state.

### Q3: What is "torn page write" and how do InnoDB and PostgreSQL protect against it?
- **Answer**: A torn page occurs because of a mismatch between database page sizes ($8\text{ KB}/16\text{ KB}$) and operating system block sizes ($4\text{ KB}$). If the OS crashes or loses power while writing a database page, only a portion of the page is saved, corrupting the page checksum and making it unrecoverable.
  - **MySQL InnoDB** solves this using the **Doublewrite Buffer**. Dirty pages are written first to a contiguous, sequential block on disk (Doublewrite Buffer) before being flushed to the actual table file. If a crash occurs during the final table flush, InnoDB restores the uncorrupted page image from the Doublewrite Buffer.
  - **PostgreSQL** solves this using **Full-Page Writes**. Following a checkpoint, the very first write to any page dumps the entire binary page image into the WAL log. If a page becomes torn during a crash, PostgreSQL reconstructs it by copying the clean, full page image from the WAL before replaying subsequent delta records.

### Q4: Explain the difference between B-Trees and LSM-Trees (Log-Structured Merge-Trees) in terms of read/write amplification and optimal use cases.
- **Answer**:
  - **B-Trees (Read-Optimized)**:
    - **Structure**: Sorted, self-balancing tree structure where nodes correspond to physical pages on disk. Updates are done in-place by reading a page, modifying it in memory, and writing it back to its specific disk block.
    - **Amplification**: High Write Amplification (modifying a single 100-byte row forces the write of a whole 8KB/16KB page to disk). Low Read Amplification ($O(\log N)$ pointing directly to pages).
    - **Use Case**: Relational databases (PostgreSQL, MySQL, Oracle) with heavy read workloads and point queries.
  - **LSM-Trees (Write-Optimized)**:
    - **Structure**: Consists of an in-memory sorted buffer (**MemTable**) and multiple immutable, sequential sorted files (**SSTables**) arranged in hierarchical levels on disk. Updates append to the MemTable. When full, MemTable is flushed sequentially to Level 0 SSTables. Background **Compaction** merges and deduplicates SSTables across levels.
    - **Amplification**: Extremely Low Write Amplification (appends are buffered in memory and written sequentially to disk). High Read Amplification (must search across multiple SSTable levels to find the latest version of a row).
    - **Use Case**: Write-heavy systems, time-series databases, and NoSQL stores (Cassandra, RocksDB, ClickHouse).

