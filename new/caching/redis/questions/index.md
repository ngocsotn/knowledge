# Redis Interview Challenges & Mitigations

Deep dive covering Cache Stampede, Cache Penetration, Cache Avalanche, and system protection.

## Major Cache Failure Modes

### 1. Cache Stampede (Thundering Herd)
Occurs when a hot cache key expires under extremely high concurrency. Thousands of simultaneous requests fail to find the key, and flood the backend database concurrently to recalculate it, causing database exhaustion and crashes.
- **Mitigation:** **Mutex Locking (Singleflight):** Force concurrent reads to acquire a lock, allowing only one thread to query the DB and write back to cache, while other requests block/wait for the result.

### 2. Cache Penetration
Occurs when queries target keys that never exist in the database (e.g., querying `user_id = -999` repeatedly). These queries bypass the cache entirely and flood the database, wasting resources.
- **Mitigation:**
  1. **Cache Null Values:** Store `key: null` in Redis with a short TTL.
  2. **Bloom Filter:** Place a space-efficient probabilistic Bloom Filter before the cache layer to quickly verify if the key definitely does not exist.

### 3. Cache Avalanche
Occurs when thousands of cached keys expire at the exact same second, or if the entire Redis server crashes. Subsequent client traffic floods the backend database instantly.
- **Mitigation:**
  1. **Randomized TTL Jitter:** Add a random offset (e.g., `TTL = 3600 + random(0, 300)`) to prevent synchronized mass expiration.
  2. **High Availability (Sentinel/Cluster):** Protect cache cluster with replication and failover support.

## Interview Questions & Answers

### Q1: How does a Bloom Filter mitigate Cache Penetration?
- **Answer:** A Bloom Filter is a space-efficient, probabilistic data structure used to test if an element is a member of a set.
  - If the filter returns **No**: The element is **definitely not** in the set. The API rejects the request immediately without checking Redis or the database.
  - If the filter returns **Yes**: The element **might be** in the set. The request proceeds to cache and database. This blocks 100% of malicious non-existent queries from hitting the database.
