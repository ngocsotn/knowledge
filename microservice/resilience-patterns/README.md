# Microservices Resilience Patterns

Comprehensive study guide for building fault-tolerant, resilient distributed systems that handle transient network failures, service crashes, and slow downstream dependencies gracefully.

---

## 1. The Circuit Breaker Pattern

In a distributed microservice mesh, a slow or unresponsive downstream service can quickly consume resources (threads, sockets) of upstream services, causing cascading failures. A **Circuit Breaker** acts as an electrical safety switch to protect the system.

### The Three States of a Circuit Breaker

```
         ┌──────────────────┐
         │                  │ (Failure rate < threshold)
         ▼                  │
   ┌──────────┐  (Failure rate > threshold)  ┌──────────┐
   │  CLOSED  │────────────────────────────>│   OPEN   │
   └──────────┘                             └──────────┘
         ▲                                        │
         │                                        │ (Sleep window expires)
         │           ┌─────────────┐              │
         └───────────│  HALF-OPEN  │<─────────────┘
      (Success rate  └─────────────┘
       > threshold)
```

1. **Closed**: Normal state. Requests flow freely to the downstream service. The circuit breaker monitors success/failure rates.
2. **Open**: When the failure rate (or slow request percentage) exceeds a configured threshold (e.g., 50% failures over a 10-second window), the circuit **trips**.
   - In the Open state, all incoming requests are **instantly rejected** (failed fast) without calling the downstream service, saving network resources and upstream threads.
   - A fallback or cached response is returned immediately.
3. **Half-Open**: After a configured "sleep window" (e.g., 30 seconds), the circuit enters Half-Open.
   - It allows a small, restricted number of trial requests to pass through.
   - If these trial requests succeed, the breaker assumes the downstream service has recovered and transitions back to **Closed**.
   - If any trial request fails, the breaker trips back to **Open**, restarting the sleep timer.

---

## 2. The Bulkhead Pattern

The **Bulkhead Pattern** isolates elements of an application into distinct pools so that if one fails, the others will continue to function. Named after the physical partitions in a ship's hull that prevent the entire ship from sinking if a single section is flooded.

### Dual Implementations:
- **Thread Pool Isolation (Most Common)**: Create dedicated, size-limited thread pools for each downstream dependency.
  - *Example*: An API Gateway assigns a pool of 10 threads for calling the "Reviews" service, and 100 threads for "Checkout." If "Reviews" slows down, it can at most exhaust its own 10 threads; the remaining 100 threads for "Checkout" stay completely unaffected.
- **Semaphore Isolation**: Uses a shared counter (semaphore) to limit concurrent requests instead of separate threads. This has lower CPU overhead but cannot handle timeouts asynchronously.

---

## 3. Retries, Exponential Backoff, and Jitter

When a network request fails due to a transient issue, retrying immediately is a natural reaction. However, simple retries can easily cause a **Retry Storm** (Thundering Herd Problem), completely overwhelming a struggling backend database or service.

### The Resolution: Backoff and Jitter
1. **Exponential Backoff**: Instead of retrying at constant intervals (e.g., every 1s), multiply the wait time exponentially after each failure:
   $$\text{delay} = \text{base\_delay} \times 2^{\text{attempt}}$$
   - Attempt 1: 100ms
   - Attempt 2: 200ms
   - Attempt 3: 400ms
   - Attempt 4: 800ms
2. **Jitter (Randomness)**: Adding randomized noise to the delay calculations is critical. Without Jitter, all clients that failed concurrently will retry at the *exact same millisecond boundaries*, creating massive spikes of load. Adding Jitter distributes the retries evenly over time:
   $$\text{delay} = \text{random}(0, \text{base\_delay} \times 2^{\text{attempt}})$$

---

## 4. The Fallback Pattern

Defines alternative behavior when a service invocation fails.
- **Default Static Value**: Return empty structures, dummy data, or placeholder templates.
- **Cached Read**: Fetch the last known good response from a fast Redis cache, serving stale but functional data to the user.
- **Functional Degradation**: Turn off non-essential UI widgets or features completely without breaking core workflows (e.g., hiding personalized recommendations on the homepage during peak sale traffic).

---

## 5. Hard Interview Questions & Deep Answers

### Q1: How do you configure a Circuit Breaker's sliding window in a high-throughput microservice? Compare Count-based vs. Time-based windows.
**Answer**:
Configuring sliding windows requires balancing sensitivity vs. stability:
- **Count-Based Window**: Measures failures over the last $N$ requests (e.g., last 100 requests).
  - *Pros*: Predictable. Great for low-to-medium throughput services where time metrics can take hours to accumulate.
  - *Cons*: Highly sensitive to sudden bursts. During high-throughput spikes, 50 bad requests within a single second can trip the circuit immediately, even if the service recovered the very next millisecond.
- **Time-Based Window**: Measures failures over the last $T$ seconds, divided into bucket increments (e.g., a 10-second window divided into ten 1-second buckets).
  - *Pros*: Smooths out transient spikes and accurately reflects the current real-time health of the network.
  - *Cons*: Needs a minimum volume threshold. If a service receives only 1 request in 10 seconds and that request fails, the failure rate is 100%, but tripping the circuit based on a single sample is a false positive. Always set `minimumNumberOfCalls` (e.g., at least 20 calls in the window before calculating rate).

### Q2: Why is Thread Pool isolation preferred over Semaphore isolation in Hystrix/Resilience4j, and what are the operational trade-offs?
**Answer**:
- **Thread Pool Isolation**:
  - *Mechanics*: Incoming requests are queued and executed on a separate, dedicated thread pool.
  - *Advantage*: **True Asynchronous Timeout Capability**. If a downstream service freezes, the calling thread can trigger a timeout and walk away immediately, while the background worker thread remains blocked.
  - *Trade-off*: Higher context-switching overhead, memory usage, and CPU consumption due to thread scheduling.
- **Semaphore Isolation**:
  - *Mechanics*: Executes requests directly on the calling/tomcat thread, using a counter to limit concurrency.
  - *Advantage*: Extremely lightweight. Zero context-switching overhead.
  - *Trade-off*: No timeout capability. If the downstream service blocks, the calling/tomcat thread remains frozen until the network socket timeout expires (which could be minutes), bypassing the circuit breaker's timeout limits and exhausting server resources.

### Q3: What is a "Retry Storm," and how do you design a cluster of microservices to prevent self-inflicted Denial of Service (DoS) during recovery?
**Answer**:
- **Retry Storm**: When a core database restarts, all upstream services that depend on it begin failing. If 1,000 active clients each retry 5 times with short timeouts, the moment the database comes back online, it is instantly hit with 5,000 sequential queries (the thundering herd), causing it to crash again immediately.
- **Prevention Designs**:
  1. **Maximum Retry Cap**: Limit retries to at most 2 or 3 attempts.
  2. **Exponential Backoff + Full Jitter**: Ensure all retries are spaced out randomly.
  3. **Token Bucket Rate Limiting (Retry Budget)**: Implement a client-side retry budget. For example, a service can only allocate 10% of its normal request budget for retries. If the retry rate exceeds 10% of successful requests, it halts retries entirely and returns fallbacks, protecting the downstream system from retry overload.
