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
