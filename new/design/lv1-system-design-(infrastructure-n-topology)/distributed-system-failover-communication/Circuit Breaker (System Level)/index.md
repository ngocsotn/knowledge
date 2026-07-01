# Circuit Breaker (System Level)

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

## Interview Questions & Answers

### Q1: What is the purpose of the Half-Open state in a Circuit Breaker?
- **Answer:** Canary testing. In the **Half-Open** state, the circuit breaker allows a highly restricted volume of client requests to pass to the failing service. If these canary requests succeed, the breaker assumes the downstream service has recovered and closes the circuit (resuming normal traffic). If they fail, it immediately trips back to the **Open** state, continuing to fail-fast and save system resources.
