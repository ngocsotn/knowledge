# Rate Limiting & Throttling

Rate limiting is an API security and resource management technique that restricts the number of incoming requests a client can make within a specified timeframe.

---

## 1. Core Rate Limiting Algorithms

There are five classical algorithms utilized to enforce rate limits:

```
Token Bucket          Leaky Bucket          Fixed Window          Sliding Window
┌───────────┐         ┌───────────┐         ┌───────────┐         ┌───────────┐
│Tokens add │         │Requests   │         │Counter resets       │Logs exact │
│on timer.  │         │drip out   │         │on boundary.         │timestamps │
│Bursts OK. │         │constantly.│         │Boundary spikes.     │accurately.│
└───────────┘         └───────────┘         └───────────┘         └───────────┘
```

### 1. Token Bucket
* **Mechanism:** A bucket holds a maximum capacity of tokens. Tokens are added to the bucket at a constant rate. Each incoming request consumes one token. If the bucket is empty, the request is rejected.
* **Pros/Cons:** Allows burst traffic up to the bucket capacity while maintaining an average long-term request rate.
* **Usage:** Standard API rate limiters.

### 2. Leaky Bucket
* **Mechanism:** Requests enter a bucket and are queued. The bucket leaks requests at a constant, steady rate. If requests enter faster than the leak rate, the bucket overflows, and excess requests are immediately dropped.
* **Pros/Cons:** Guarantees a perfectly smooth, constant egress output rate. Adds latency to burst traffic because requests must wait in the queue.
* **Usage:** Traffic shaping, preventing database connection spikes.

### 3. Fixed Window Counter
* **Mechanism:** Divides time into fixed intervals (e.g., 1-minute windows). A simple counter increments on each request. When the counter exceeds the limit, requests are blocked until the next window starts, resetting the counter.
* **Pros/Cons:** Extremely simple to implement and memory efficient. However, it suffers from **boundary spikes**: a client can send 100% of their limit right before a window ends, and another 100% right after, effectively doubling the traffic limit in a split second.

### 4. Sliding Window Log
* **Mechanism:** Stores a sorted set of exact timestamps for every request a user makes. To check the limit, the algorithm counts how many logs fall within the current sliding timeframe (e.g., last 60 seconds) and drops older logs.
* **Pros/Cons:** Extremely precise. Avoids boundary spikes completely. However, it has very high memory consumption since it must store timestamps for every single request.

### 5. Sliding Window Counter
* **Mechanism:** A hybrid of Fixed Window and Sliding Window Log. It keeps track of counters for the current window and the previous window. It estimates request density dynamically using a weighted formula:
  $$\text{Requests} = \text{Previous Window Count} \times \left(1 - \frac{\text{Current Window Progress}}{\text{Total Window Duration}}\right) + \text{Current Window Count}$$
* **Pros/Cons:** Extremely memory efficient and resolves boundary spikes without storing infinite request timestamps.

---

## 2. Distributed Architecture: Redis-Based Rate Limiting

In microservices or clustered environments, rate-limiting must be coordinated across multiple server nodes.

* **Redis** is the standard cache database used to persist rate-limiting states because of its speed and atomic operations.
* **The Redis Race Condition:**
  * When executing `GET` -> `INCR` -> `SET` sequence across parallel requests, a race condition can cause clients to exceed limits.
  * *Mitigation:* Execute atomic **Lua Scripts** or use Redis commands like `INCR` directly which execute isolated and sequentially.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between Token Bucket and Leaky Bucket?
* **Answer:** **Token Bucket** allows a degree of burst traffic: if tokens have accumulated, a client can execute multiple requests simultaneously without latency. **Leaky Bucket** completely flattens traffic, enforcing a strict, steady processing rate. Burst traffic is buffered in a queue and processed slowly, adding latency, or dropped if the buffer queue is full.

### Q2: How do you choose the right key to identify a client for rate-limiting?
* **Answer:** It depends on the context:
  1. **Authenticated Clients:** Rate-limit by unique `user_id`, `account_id`, or `api_key`. This is resilient to network/IP changes.
  2. **Unauthenticated Clients:** Rate-limit by client IP. To prevent issues with proxies, you must securely resolve the true client IP using trusted header offsets (e.g., `X-Forwarded-For` or `X-Real-IP`).
  3. **High-Risk Endpoints (Login/Register):** Rate-limit by a combination of IP and the targeted credential identifier (e.g., email) to mitigate credential-stuffing attacks.

### Q3: How do you prevent a rate-limiting database (like Redis) from becoming a single point of failure (SPOF)?
* **Answer:** If Redis crashes, a secure system must fail gracefully. You can configure a fallback to local **in-memory rate-limiting** (using sliding windows in Go/Node RAM) with a slightly higher threshold, or implement a **fail-open** policy while raising alerts, prioritizing uptime over rate protection during database outages.
