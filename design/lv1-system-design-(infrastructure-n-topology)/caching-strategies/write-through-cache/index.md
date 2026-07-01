# Write-Through Caching

The application writes data to the cache first, which immediately writes to the primary database synchronously before returning success to the client.
- **Pros:** Guarantees cache consistency; no stale data.
- **Cons:** High write latency due to synchronous, double-write operations.

## Interview Questions & Answers

### Q1: When is Write-Through caching highly preferred?
- **Answer:** In systems with very high read-to-write ratios where data accuracy is critical (e.g., user profiles, financial account balances). Reads are extremely fast and never stale, justifying the write-path latency penalty.
