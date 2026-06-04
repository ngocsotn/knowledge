# Caching Strategies & Eviction Policies

Comprehensive interview study guide covering distributed caching patterns, write paths, cache-aside strategies, and memory eviction policies.

---

## 1. Meaning & Core Concept

Caching is the process of storing copies of data in a high-speed, in-memory data store (typically **Redis** or **Memcached**) to bypass slow, resource-heavy disk operations (like SQL queries or complex computations), improving read latency to sub-milliseconds.

---

## 2. Caching Access Patterns

Depending on the system goals, different write and read caching strategies are deployed:

```
Cache-Aside (Lazy)     Read-Through            Write-Through           Write-Back (Behind)
┌─────────────────┐    ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ App queries     │    │ App queries     │     │ App writes      │     │ App writes      │
│ cache first.    │    │ cache. Cache    │     │ simultaneously  │     │ to cache only.  │
│ DB fallback.    │    │ fetches from DB.│     │ to cache & DB.  │     │ Async DB sync.  │
└─────────────────┘    └─────────────────┘     └─────────────────┘     └─────────────────┘
```

### 1. Cache-Aside (Lazy Loading)
* **Read Path:**
  1. Application checks the cache first.
  2. **Cache Hit:** Return data to client.
  3. **Cache Miss:** App queries the database, writes the result to the cache, and returns it.
* **Write Path:** Application writes updates directly to the database and invalidates (deletes) the corresponding cache key.
* **Pros/Cons:** Safe and highly efficient; cache only holds frequently accessed data. However, a cache miss adds slight latency to the first request.

### 2. Read-Through
* **Mechanism:** The application treats the cache as the main data store. On a cache miss, the *cache library itself* handles reading from the database and populating the key before returning it.

### 3. Write-Through
* **Mechanism:** When writing data, the application writes to the cache and the database **simultaneously** in a single transaction.
* **Pros/Cons:** Guarantees cache-database consistency. However, it adds latency to write operations since both stores must commit before success.

### 4. Write-Back (Write-Behind)
* **Mechanism:** The application writes updates strictly to the in-memory cache, which acknowledges success immediately. A background worker periodically batches these updates and writes them **asynchronously** to the database.
* **Pros/Cons:** Extremely high write throughput. However, if the cache server crashes before background sync commits, data is permanently lost.

---

## 3. Cache Eviction Policies

Since in-memory caches have finite RAM limits, they must drop old keys when they run out of space using predefined eviction algorithms:

* **LRU (Least Recently Used):** Discards the keys that haven't been accessed for the longest period. (The industry standard default).
* **LFU (Least Frequently Used):** Tracks access frequency. Discards keys with the lowest access count.
* **FIFO (First In, First Out):** Discards keys in the order they were created, regardless of access patterns.
* **TTL (Time To Live):** Keys expire and are evicted automatically after a predefined time-frame.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is the Cache Invalidation problem, and why should you "delete" a key on write instead of "updating" it?
* **Answer:** If two concurrent write requests attempt to update the same record in an **update-on-write** cache, a race condition can cause the DB and cache to drift (e.g., Writer 1 updates DB, Writer 2 updates DB, Writer 2 updates cache, Writer 1 updates cache—leaving the cache with stale Writer 1 data while DB has Writer 2 data). By **deleting (invalidating) the cache key on write**, we force the next read request to safely perform a cache-aside lookup, pulling the latest authoritative data directly from the DB.

### Q2: Explain Cache Penetration, Cache Avalanche, and Cache Shear (Stampede), and how you mitigate them.
* **Answer:**
  * **Cache Penetration:** Requests target keys that *never* exist in the DB (e.g., negative IDs), forcing queries to hit the DB every time. *Fix:* Cache empty/null results with a short TTL, or use a **Bloom Filter** at the edge.
  * **Cache Avalanche:** Many popular keys expire at the exact same second, or the cache server crashes, causing millions of requests to flood the database simultaneously. *Fix:* Add a **random jitter (noise)** to TTL expirations, and deploy high-availability Redis clusters.
  * **Cache Stampede (Cache Shear):** A highly popular key expires, and thousands of concurrent threads detect the miss at the same time, executing identical slow SQL queries to rebuild it. *Fix:* Implement **Mutex Locking (Singleflight)** so only the first thread performs the SQL read while others wait for the cached result.

### Q3: When is a Write-Back (Write-Behind) caching strategy suitable, and what is its main danger?
* **Answer:** Write-back caching is ideal for write-heavy systems where database write performance is a major bottleneck, such as multiplayer game state updates, IoT telemetry ingestion, or real-time analytics tracking. The primary danger is **data loss**: because the write is acknowledged as successful once stored in volatile RAM, any power outage, node crash, or container restart before the asynchronous database sync completes results in unrecoverable data loss.
