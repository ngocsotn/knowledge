# Distributed Caching

Comprehensive guide to Distributed Caching systems and shared state management.

Caching is the process of storing copies of data in a high-speed, in-memory data store (typically **Redis** or **Memcached**) to bypass slow, resource-heavy disk operations (like SQL queries or complex computations), improving read latency to sub-milliseconds.

---

## Distributed Cache Properties
- **Location:** Centrally stored in dedicated cache systems like **Redis** or **Memcached**.
- **Shared State:** All clustered app instances read/write to the same cache, guaranteeing cache consistency.
- **Scale:** Can store gigabytes to terabytes of data, independent of the application server's heap memory limits.

## Interview Questions & Answers

### Q1: When do you choose Memcached over Redis?
- **Answer:** Choose **Memcached** for simple, static, high-volume key-value lookups where advanced data structures are unnecessary. Memcached is multithreaded and highly optimized for raw string keys, making it slightly faster and more memory-efficient under simple query patterns. Choose **Redis** for rich data structures (hashes, sorted sets, lists), persistence (AOF/RDB), replication, pub/sub messaging, and high availability.
