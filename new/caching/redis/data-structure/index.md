# Redis Data Structures & Internals

Deep dive covering String, Hash, List, Set, and Sorted Set structures and their memory architectures.

## Core Data Structures

### 1. String
The fundamental key-value format. Capable of storing strings, integers, or raw binary payloads up to 512MB.
- **Under the hood:** Dynamic SDS (Simple Dynamic String) headers, offering $O(1)$ length operations.

### 2. Hash
A map of string fields and values. Perfect for storing objects (e.g., `user:101`).
- **Under the hood:** Encoded as highly compressed `ziplist` for small hashes, and upgraded to `hashtable` at scale.

### 3. List
An ordered collection of string elements.
- **Under the hood:** Implemented as a doubly-linked `quicklist` (combining ziplists and linked lists) supporting $O(1)$ push/pop from head and tail.

### 4. Set
An unordered, unique collection of string elements.
- **Under the hood:** Encoded as a sorted integer array `intset` for numbers, upgraded to `hashtable` at scale.

### 5. Sorted Set (ZSET)
A collection of unique strings, where each element is mapped to a floating-point score. Ordered automatically by score.
- **Under the hood:** Leverages a **Skip List (skiplist)** combined with a hash table to guarantee $O(\log N)$ insertions, searches, and range queries.

## Interview Questions & Answers

### Q1: What is a Skip List, and why does Redis use it for Sorted Sets?
- **Answer:** A Skip List is a probabilistic, multi-level linked list data structure that supports search, insertion, and deletion in logarithmic time ($O(\log N)$), similar to balanced binary trees (like AVL or Red-Black trees).
  - Redis chooses Skip Lists because they are simpler to implement, consume less memory overhead per node, and perform significantly faster for concurrent range queries (`ZRANGEBYSCORE`) than balanced trees since they avoid complex rotation algorithms during mutations.
