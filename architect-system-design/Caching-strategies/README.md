# Caching Strategies & Eviction Policies

Comprehensive interview study guide covering distributed caching patterns, write paths, cache-aside strategies, memory eviction policies, and Redis Cluster architecture under scale.

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

## 3. Distributed Cache Architecture: Redis Cluster

When a single Redis node cannot handle the write/read throughput or memory storage requirements of an enterprise-scale system, **Redis Cluster** provides a distributed, highly available architecture.

```
                  ┌───────────────────────────────┐
                  │         Redis Cluster         │
                  │   [16,384 Hash Slots Total]   │
                  └───────────────┬───────────────┘
          ┌───────────────────────┼───────────────────────┐
          ▼                       ▼                       ▼
    Master Node 1           Master Node 2           Master Node 3
  [Slots: 0 - 5460]      [Slots: 5461 - 10922]  [Slots: 10923 - 16383]
          │                       │                       │
      (Replica)               (Replica)               (Replica)
    Replica Node 1          Replica Node 2          Replica Node 3
```

### A. Sharding and Hash Slots
- **No Consistent Hashing**: Unlike some systems that use consistent hashing rings, Redis Cluster uses **Hash Slots**.
- **16384 Slots**: There are exactly 16,384 logical slots in a cluster. The slots are divided amongst the master nodes.
- **Key-to-Slot Mapping**: Every key is mapped to one of these slots using the CRC16 checksum formula:
  $$\text{Slot} = \text{CRC16}(key) \pmod{16384}$$
- **Cross-Slot Operations (Hash Tags)**: Normally, multi-key operations (like `MGET` or transactions) are forbidden in a cluster if the keys hash to different slots on different nodes. 
  - *Solution*: Use **Hash Tags** `{}` in key names. Only the text inside the curly braces is hashed.
  - *Example*: `{user123}:profile` and `{user123}:orders` are guaranteed to land on the exact same hash slot and node, allowing safe multi-key operations.

### B. High Availability: Sentinel vs. Native Cluster Failover
- **Redis Sentinel**: Best for non-sharded configurations (single master, multiple replicas). Sentinels monitor nodes, handle automatic failover, and act as a configuration provider for clients. Sentinels run as a separate service layer.
- **Native Cluster Failover**: In a Redis Cluster, master nodes monitor each other through gossip protocols. If a majority of master nodes detect Node 1 has failed, they vote to promote Node 1's replica to be the new master. No external Sentinel cluster is required.

---

## 4. Deep-Dive Eviction Policies

When Redis reaches its maximum memory limit (`maxmemory`), it must evict existing keys to make room for new writes based on the configured `maxmemory-policy`.

### A. How Redis Implements LRU (Least Recently Used)
- **True LRU**: Requires maintaining a doubly linked list of all keys in memory, moving a key to the head on every read/write. This introduces extreme memory overhead and lock contention.
- **Approximated LRU**: To avoid this overhead, Redis stores a 24-bit timestamp of the last access time in every object's header. When eviction is triggered, Redis selects a random sample of keys (e.g., 5 or 10 keys) and evicts the one with the oldest timestamp among them. This achieves nearly identical performance to True LRU with zero memory or latency overhead.

### B. How LFU (Least Frequently Used) Is Implemented
Redis LFU maps frequency into a single 8-bit log counter and an 8-bit decay timestamp inside the object's header:
1. **Logarithmic Counter**: Instead of incrementing linearly, the counter is updated logarithmically. The higher the counter, the less likely it is to increment on subsequent reads, capping the counter at 255.
2. **Decay Period**: A background routine decreases the counter over time based on how long ago the key was last accessed (decay timestamp), ensuring that historical spikes in key usage decay if the key becomes idle.

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: What is the Cache Invalidation problem, and why should you "delete" a key on write instead of "updating" it?
**Answer**:
If two concurrent write requests attempt to update the same record in an **update-on-write** cache, a race condition can cause the DB and cache to drift:
1. Writer A updates the DB.
2. Writer B updates the DB.
3. Writer B updates the cache.
4. Writer A updates the cache (network delay on step 3).
- **Result**: The database has Writer B's data, but the cache has Writer A's stale data.
- **The Solution**: Always **delete (invalidate) the cache key on write**. By doing this, we eliminate the race condition completely. The next read operation will safely experience a cache miss and fetch the single authoritative source of truth from the database.

### Q2: Explain Cache Penetration, Cache Avalanche, and Cache Stampede (Shear), and how you mitigate them.
**Answer**:
- **Cache Penetration**: Requests target keys that *never* exist in the database (e.g., negative IDs or spam queries), bypassing the cache and hitting the database every time.
  - *Fix*: Cache empty/null results with a very short TTL, or run a **Bloom Filter** at the gateway layer to instantly reject non-existent IDs.
- **Cache Avalanche**: Many popular keys expire at the exact same second, or the cache cluster crashes, causing millions of concurrent reads to flood the database.
  - *Fix*: Add a **random jitter (noise)** to TTL expirations (e.g., `TTL = 3600 + random(0, 300)` seconds) so keys expire at staggered times, and deploy highly available Redis Clusters with automatic replica promotion.
- **Cache Stampede (Dog-piling)**: A highly popular key (e.g., homepage banner) expires, and thousands of concurrent requests detect the miss at the exact same millisecond, triggering identical slow SQL queries to rebuild the key.
  - *Fix*: Implement **Mutex Locking (Singleflight / Semaphore)** in the application. Only the first thread gets a lock to query the DB and rebuild the cache, while other concurrent threads wait for the lock to release and read the newly cached value.

### Q3: When is a Write-Back (Write-Behind) caching strategy suitable, and what is its main danger?
**Answer**:
Write-back caching is ideal for write-heavy systems where database write performance is a major bottleneck, such as multiplayer game state updates, IoT telemetry ingestion, or real-time analytics tracking. The primary danger is **data loss**: because the write is acknowledged as successful once stored in volatile RAM, any power outage, node crash, or container restart before the asynchronous database sync completes results in unrecoverable data loss.

### Q4: [Caching Invalidation Dilemma] Between "Delete Cache, then Update DB" and "Update DB, then Delete Cache", which sequence is safer for Cache-Aside systems, and what race condition can still occur?
**Answer**:
* **Update DB, then Delete Cache (Safer - Standard)**: 
  * If this fails, the worst case is the cache contains stale data until its TTL expires.
  * **The Split-Second Race Condition**:
    1. Cache is empty.
    2. Client A performs a read, gets a cache miss, and reads the *old value* (e.g., `10`) from the database.
    3. Client B updates the database to `20` and immediately deletes the cache key.
    4. Client A (delayed by network) writes its stale read value (`10`) back into the cache.
    * *Result*: The cache has stale value `10` permanently, while the DB has `20`.
    * *Mitigation*: Use extremely short TTLs or a **Cache-Aside with Mutex Lock (or Singleflight)** during cache rebuilds.
* **Delete Cache, then Update DB (Unsafe)**:
  * Highly vulnerable: Client A deletes the cache key. Client B reads the key, hits a cache miss, reads the *old value* from the DB, and writes it back to the cache before Client A can commit the new value to the DB. The cache stays stale indefinitely.

### Q5: [Caching Method Scenarios] Describe the target production scenarios where you would choose each of the four core caching methods.
**Answer**:
1. **Cache-Aside**: Best for **general-purpose, read-heavy applications** with highly dynamic data where occasional cache misses are acceptable (e.g., e-commerce product catalogs, social media user profiles).
2. **Read-Through**: Best for **microservice clusters** where you want to decouple the data access logic from the core business logic. The application treats the cache as a unified data gateway, simplifying core service code.
3. **Write-Through**: Best for **critical transactional systems** with strict consistency requirements where data is written once but read frequently (e.g., financial ledger balances, active user session details, real-time inventory systems). Bypasses stale cache reads completely.
4. **Write-Back (Write-Behind)**: Best for **ultra-high write-throughput applications** where database disk IO is the absolute system bottleneck and minor data loss during a crash is acceptable (e.g., streaming view counts, multiplayer game coordinates, tracking real-time user clickstreams, IoT sensor telemetry ingestion).


