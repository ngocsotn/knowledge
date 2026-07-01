# Database Storage Engine Internals

Under the hood guide covering Pages, Buffer Pools, Write-Ahead Logs (WAL), and checkpoints.

## Storage Engine Mechanics

### 1. Page
The smallest unit of database storage on disk (typically 8KB in PostgreSQL, 16KB in InnoDB/MySQL).
- **Structure:** Contains page headers, pointers to row locations, and raw binary tuple fields. All I/O operations are executed at the page level.

### 2. Buffer Pool
An allocated block of RAM memory used to cache physical disk pages.
- **Reads:** If a page is in the Buffer Pool (Cache Hit), it is read instantly. If not, the engine fetches the page from disk into the pool.
- **Writes:** Writes modify pages directly in the Buffer Pool, marking them as **Dirty Pages** in RAM.

### 3. Write-Ahead Log (WAL)
To guarantee ACID durability without forcing slow random disk page writes, databases write all transaction mutations sequentially to an append-only on-disk file called the **Write-Ahead Log (WAL)** *before* modifying any pages.
- **Crash Recovery:** If the database crashes, the engine replays the sequential WAL file upon reboot to restore RAM state.

### 4. Checkpoint
A background process that flushes modified "Dirty Pages" from the RAM Buffer Pool to the physical table files on disk sequentially, and safely truncates spent WAL records.

## Interview Questions & Answers

### Q1: Why does a database write to a WAL file before modifying actual table files?
- **Answer:** Performance and Durability. Modifying actual table files requires **Random I/O** (writing to various scattered page blocks on disk), which is extremely slow. Writing to a WAL file is **Sequential I/O** (appending to the end of a single file), which is near-instantaneous. Writing transactions sequentially to WAL guarantees **Durability** (survives crashes) while keeping write latency minimal, deferring random table disk flushes to background checkpoint threads.
