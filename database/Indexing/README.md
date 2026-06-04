# Database Indexing

Comprehensive interview study guide covering indexing mechanisms, B-Tree and Hash structures, composite indexes, and index scan vs. seek.

---

## 1. Meaning & Core Mechanism

A database index is a separate, optimized data structure maintained by the database engine to speed up data retrieval operations. Without an index, the database must perform a **Full Table Scan (Sequential Scan)**, reading every single page on the disk to find matching rows—resulting in `O(N)` time complexity.

---

## 2. Common Index Data Structures

Modern relational databases (like PostgreSQL and MySQL) use two main types of index data structures:

### 1. B-Tree (and B+ Tree) Indexes (Default)
* **Mechanism:** A self-balancing search tree designed to store sorted data. B+ Trees store actual row pointers or row data only in the leaf nodes, while internal nodes store strictly index search keys.
* **Why B+ Tree is preferred for Databases:**
  * Leaf nodes are linked together in a doubly linked list, enabling extremely fast **range queries** (e.g., `WHERE age BETWEEN 18 AND 30`) and sequential scanning.
  * Shallow height: Large fan-out (hundreds of child nodes per node) keeps the tree depth small (usually 3-4 levels), minimizing slow disk I/O operations.
* **Complexity:** `O(log N)` for reads, writes, and range scans.

### 2. Hash Indexes
* **Mechanism:** Uses a hash table to map keys directly to row addresses.
* **Limitations:**
  * Does **not** support range queries (e.g., `WHERE salary > 5000` is impossible to look up via Hash).
  * Does not support sorting (`ORDER BY`).
  * Only works for exact equality lookups (`WHERE id = 123`).
* **Complexity:** `O(1)` average lookup time.

---

## 3. Advanced Indexing Strategies

### 1. Composite (Multi-Column) Indexes
An index built on multiple columns (e.g., `INDEX (last_name, first_name)`).
* **The Leftmost Prefix Rule:** A composite index can only be used if the query filters match columns starting from the left of the index definition.
  * `WHERE last_name = 'Smith' AND first_name = 'John'` ──► **Uses Index**
  * `WHERE last_name = 'Smith'` ──► **Uses Index**
  * `WHERE first_name = 'John'` ──► **FAILS leftmost rule (Full Table Scan)**

### 2. Clustered vs. Non-Clustered Indexes
* **Clustered Index:** Defines the physical storage order of rows on disk. A table can have **only one** clustered index (typically the Primary Key).
* **Non-Clustered Index:** A separate structure containing the indexed keys and pointers back to the physical rows (the clustered index key).

---

## 4. Index Scan vs. Index Seek

When checking an execution plan (via `EXPLAIN`), look for these operations:

* **Index Seek:**
  * *Definition:* The database engine traverses the index tree structure directly to locate specific matching keys.
  * *Efficiency:* Highly efficient. Only reads the exact branches needed. `O(log N)`.
* **Index Scan:**
  * *Definition:* The engine traverses the entire index structure sequentially from start to finish.
  * *Efficiency:* Less efficient. Used when a range is too broad, or the leftmost prefix rule is violated but scanning the smaller index is still faster than scanning the massive raw data table.

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: Why should we not index every column in a table?
* **Answer:** While indexes speed up read operations (`SELECT`), they introduce overhead to write operations (`INSERT`, `UPDATE`, `DELETE`). For every write, the database must dynamically update, rebalance, or split the index structures (e.g., B-Trees) on disk and in memory. Furthermore, indexes consume valuable RAM and disk space, and over-indexing can confuse the query optimizer into choosing inefficient execution plans.

### Q2: How does a B+ Tree compare to a standard Binary Search Tree (BST) for database storage?
* **Answer:** A standard BST has a fan-out of 2, making the tree extremely deep for millions of records. This requires many sequential node evaluations. Because database pages reside on slow persistent disks, each node traversal represents a slow disk I/O. A **B+ Tree** has a huge fan-out (often 100+ keys per node), keeping the tree extremely shallow (3-4 levels maximum) and ensuring we only need 3-4 disk seek operations to find any record.

### Q3: What is a "Covering Index" and how does it optimize queries?
* **Answer:** A covering index is a non-clustered index that contains all the columns requested by a specific query in its select and filter clauses. When this occurs, the database engine can retrieve all required data directly from the lightweight index structure in RAM, completely skipping the slow step of referencing physical table pages on disk (Index lookup without Row Lookup).

---

## 6. Deep Dive: Physical Storage & Disk IO

### A. Clustered vs. Secondary Indexes (Under the Hood)

#### 1. Clustered Index (Primary Key Storage)
The **Clustered Index** *is* the table itself. The leaf nodes of the Clustered B+Tree do not store pointers; they store the **actual, complete data rows**.
* Because the physical rows can only be sorted in one order on disk, there can be **only one** clustered index per table.
* **Access Path:** 
  `Query PK` ──► `Traverse Clustered B+Tree` ──► `Leaf Node (Contains Full Data Row)` ──► `Return Row`.

#### 2. Secondary Index (Non-Clustered Lookup)
Any index other than the clustered index is a **Secondary Index**. The leaf nodes of a secondary B+Tree store the indexed key along with a pointer to the **Clustered Index Key (the Primary Key)**, *not* the physical disk offset of the row.
* **Access Path (Without Covering Index):**
  `Query Secondary Key` ──► `Traverse Secondary B+Tree` ──► `Leaf Node (Contains Primary Key)` ──► `Traverse Clustered B+Tree with PK` ──► `Leaf Node (Contains Full Data Row)` ──► `Return Row`.
* This double-traversal overhead is called a **Key Lookup** or **Bookmark Lookup**, and it causes massive Random IO.

```
[Secondary B+Tree Seek] ──► Finds PK value '42'
                                 │
                                 ▼
   [Clustered B+Tree Seek] ──► Reads PK '42' from leaf node ──► Full Row Loaded
```

---

### B. Random IO vs. Sequential IO

* **Sequential IO**: Reading adjacent physical blocks on disk. This is extremely fast because disk heads (or SSD controllers) pre-fetch continuous sectors. A **Full Table Scan** or a sequential scan of a B+Tree leaf-node link-chain uses Sequential IO.
* **Random IO**: Jumping between non-contiguous disk addresses. A **Secondary Index Lookup** that returns many rows triggers multiple separate Key Lookups, causing the disk to jump around to fetch non-contiguous physical pages.
* **The Threshold Rule**: If a query's secondary index seek is expected to return more than **15% to 25%** of the table's total rows, the SQL query optimizer will deliberately abandon the index and fall back to a **Full Table Scan** because performing sequential scanning of the entire table on disk is faster than thousands of separate random IO Key Lookups.

---

### C. SQL B+Tree Fragmentation

As records are inserted, updated, or deleted randomly, the B+Tree indexes degrade physically over time:

1. **Leaf Node Splits**: If a B+Tree leaf node page is full (usually 8KB or 16KB) and a new key must be inserted in sorted order, the page is split into two half-empty pages.
2. **Fragmentation**: Frequent page splits cause:
   * **Internal Fragmentation**: Pages have excessive empty space (low page fill factor, e.g., 50% instead of 90%), wasting RAM and disk.
   * **External Fragmentation**: The logical order of leaf nodes (the doubly linked list) no longer matches their physical order on disk, turning fast Sequential Range scans into slow Random IO seeks.
3. **The Cure**: Regularly run `REBUILD INDEX` / `REINDEX` (PostgreSQL) or `OPTIMIZE TABLE` (MySQL InnoDB) to recreate the index sequentially on contiguous disk blocks with an optimal fill factor (e.g., 80% to allow inserts without immediate splits).

---

## 6. Popular Interview Questions & High-Impact Answers (Extended)

### Q4: Explain the difference between Clustered and Secondary indexes. Why do secondary indexes store the Primary Key instead of direct disk block offsets?
* **Answer:** A clustered index stores the physical data rows directly in its B+Tree leaf nodes, meaning the index *is* the table. A secondary index contains only the indexed keys and the corresponding Primary Key. Secondary indexes store the Primary Key instead of physical disk block offsets (pointers) to prevent **cascading updates**. If the database stored direct disk offsets, whenever a row is moved, updated, or when a clustered B+Tree page splits, the database would have to write and update the offset pointer inside *every single secondary index* on that table. Storing the PK isolates secondary indexes from physical row movements at the expense of an extra clustered tree lookup.

### Q5: What is Index Fragmentation, why does it occur, and how do you diagnose and fix it?
* **Answer:** Index fragmentation is the physical disorganization of index pages on disk. It occurs due to **random write splits**: when we insert, update, or delete rows with non-sequential primary keys (like UUID v4), full B+Tree leaf nodes must split to maintain logical sorted order, leaving pages half-empty and scattered across non-contiguous disk sectors. 
  * *Diagnosis*: In PostgreSQL, check the `pgstattuple` extension. In MySQL, run `SHOW INDEX FROM table_name` and check the cardinality and page details.
  * *Fix*: Rebuild the index using `REINDEX INDEX index_name` (PostgreSQL) or `ALTER TABLE table_name ENGINE=InnoDB` / `OPTIMIZE TABLE table_name` (MySQL) to defragment pages and restore physical sequential alignment.

### Q6: [Struggle Question] Why can using UUID v4 as a Primary Key severely degrade database write performance and cause massive index fragmentation in B+Tree databases? How do you solve it?
* **Answer:** 
  1. **The Write Penalty**: UUID v4 values are completely random. When inserting a row, the database cannot simply append it to the end of the clustered index disk block (sequential write). Instead, it must find a random location in the middle of the B+Tree index.
  2. **Page Splits & Random IO**: This random insertion forces pages to split constantly, triggering massive Random IO read/write cycles to disk and leaving the clustered index highly fragmented (lots of empty space inside index pages).
  3. **Buffer Pool Eviction**: Since keys are random, the database cannot keep the "hot" portion of the index in the RAM buffer pool. Every insert requires loading a different index block from disk, destroying cache locality.
  4. **Solution**: Use **ordered, time-sortable identifiers** like **UUID v7**, **ULID**, or a Snowflake ID. These contain a millisecond-precision Unix timestamp in their most significant bits, making them sequentially sortable. The database can append inserts to the rightmost leaf node of the B+Tree sequentially, preventing page splits and maintaining maximum cache hit rates.

