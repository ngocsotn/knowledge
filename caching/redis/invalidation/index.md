# Redis Cache Invalidation Strategies

Guide covering TTL, Event-Based, Version-Based, and Tag-Based invalidation patterns.

## Invalidation Strategies

### 1. TTL (Time to Live)
The simplest invalidation pattern. Keys are assigned an expiration timestamp.
- **Pros:** Automatic self-cleanup; easy implementation.
- **Cons:** Risk of serving stale data during the TTL window.

### 2. Event-Based (Active Invalidation)
The application listens to database mutation events (or publishes event messages) and explicitly deletes/evicts the corresponding cache keys immediately.
- **Pros:** Highly consistent data.
- **Cons:** Implementation complexity; vulnerable to distributed race conditions.

### 3. Version-Based
Every database entity has an associated version integer. Cache keys embed this version (e.g., `user:101:v5`). When the database updates, the version changes, automatically causing subsequent requests to cache-miss and fetch fresh data.

### 4. Tag-Based (Cache Purging)
Keys are tagged with one or more labels. Purging a tag (e.g., `tag:products`) invalidates all cached objects tagged with that label in a single operation.

## Interview Questions & Answers

### Q1: Why is "Cache Invalidation" considered one of the two hardest problems in computer science?
- **Answer:** Cache invalidation requires coordinating state across decoupled, distributed systems. If you invalidate too early (high write rate), you suffer from cache stampedes and database load. If you invalidate too late (high TTL), you serve stale, incorrect data. Handling race conditions where concurrent read-misses overwrite new writes in cache requires complex mutex locks, versioning, or transactional singleflight operations.
