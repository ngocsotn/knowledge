# Apache Kafka: Storage Architecture

Deep-dive covering Sequential I/O, Log-based Storage, and LSM-Tree storage structures.

## Storage Architecture

### 1. Sequential I/O
Kafka writes events to append-only commit logs. Operating systems are highly optimized for sequential disk operations, making Kafka's disk writes nearly as fast as memory access.

### 2. Log-based Storage
Data is saved as physical segment files on disk, accompanied by index files linking offsets to raw file positions, bypassing traditional random-access databases.

### 3. LSM Tree Concept
Log-Structured Merge-trees prioritize write performance by buffering changes in-memory (MemTable) before flushing them sequentially to sorted segment files (SSTables) on disk, with background compaction.

## Interview Questions & Answers

### Q1: Why is Kafka so fast despite storing all data on physical disk?
- **Answer:** Kafka leverages three main operating system level optimizations:
  1. **Sequential I/O:** Disk writes are append-only. Sequential disk access is extremely fast compared to random I/O.
  2. **Page Cache:** Kafka stores disk data in the OS page cache (RAM), avoiding JVM heap overhead.
  3. **Zero-Copy Optimization:** Using the `sendfile` system call, Kafka pipes data directly from the OS page cache to the network socket, bypassing the application-level CPU buffers and saving massive context-switch cycles.
