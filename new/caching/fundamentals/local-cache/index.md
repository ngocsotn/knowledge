# Local In-Memory Cache

Detailed study guide covering Local In-Memory Caching systems, memory limits, and eviction policies.

## Local Caching Mechanics
Local caches store data directly inside the application's physical RAM memory space (e.g., using `sync.Map` in Go, in-memory Map in Node, or specialized libraries like Guava Cache in Java).
- **Speed:** Sub-microsecond speeds. Fastest possible caching layer since it bypasses network calls entirely.
- **Limitation:** Restricted by the container's heap/RAM limits. Data is lost upon container restart. Unsuited for horizontally scaled applications due to cache discrepancy.

## Eviction Policies
Eviction occurs when the cache is full, and older items must be thrown away to make room for new data:
- **LRU (Least Recently Used):** Evicts the item that has not been accessed for the longest time. Maintains a doubly-linked list of keys.
- **LFU (Least Frequently Used):** Evicts the item with the lowest access count. Uses logarithmic decay counters.
- **FIFO (First In First Out):** Evicts the oldest item based on insertion time.
- **TTL (Time to Live):** Evicts items after a configured lifespan.

## Interview Questions & Answers

### Q1: What is the main drawback of using Local Caching in horizontally scaled applications?
- **Answer:** Cache Discrepancy (Inconsistency). If Server A handles a write request and updates its local in-memory cache, Server B remains completely unaware of this change. When a client requests the same resource from Server B, they receive stale data, leading to inconsistent user experiences across the cluster.
