# Cache-Aside (Lazy Loading)

The application coordinates state: first reads from the cache; if a cache miss occurs, reads from the primary database, writes back to the cache, and returns the resource.
- **Pros:** Only requested data is cached, preserving memory.
- **Cons:** Cache-miss latency penalty; risk of serving stale data if database updates occur independently.

## Interview Questions & Answers

### Q1: What is the primary mitigation for stale data in Cache-Aside patterns?
- **Answer:** Pair the write path with active cache eviction (delete the key immediately upon database update), or set an aggressive Time-to-Live (TTL) on cached keys to force automatic self-cleaning.
