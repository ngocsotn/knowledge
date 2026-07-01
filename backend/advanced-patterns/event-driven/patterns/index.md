# Event-Driven Architecture Patterns

Detailed guide covering foundational asynchronous communication patterns, integration strategies, and high-performance microservices routing architectures.

---

## 1. Core Messaging Patterns

### A. Fire-and-Forget
The producer publishes an event or message to the broker and immediately resumes its work without waiting for any confirmation or processing response from downstream consumers.
- **Use Case:** Non-critical background telemetry, audit logging, sending notifications.
- **Trade-off:** High performance, but requires at-least-once configuration if data loss is unacceptable.

### B. Competing Consumers (Worker Queue)
Multiple identical instances of a consumer service subscribe to a single shared queue.
- **Mechanism:** The queue broker distributes incoming tasks (typically round-robin) across the available workers. Each individual task is processed by **exactly one** worker instance.
- **Use Case:** Background intensive processing, image/video encoding, generating reports.

```
                    ┌───► [ Worker Instance 1 ] (Processes Job A)
[ Shared Queue ] ───┼───► [ Worker Instance 2 ] (Processes Job B)
                    └───► [ Worker Instance 3 ] (Processes Job C)
```

### C. Fan-Out (Publish-Subscribe Routing)
A single message published by a producer is duplicated by the broker and routed to multiple independent destination queues, allowing different business services to react to the same event concurrently.
- **Use Case:** An `OrderPlacedEvent` triggers:
  1. Billing Service (starts credit card capture)
  2. Inventory Service (allocates physical items)
  3. Notification Service (sends user receipt email)
- **Mechanism:** Decouples services entirely; the order service has zero knowledge of billing, inventory, or notification existence.

```
                     ┌──► [ Queue A ] ──► [ Billing Service ]
[ OrderPlaced ] ────►┼──► [ Queue B ] ──► [ Inventory Service ]
                     └──► [ Queue C ] ──► [ Notification Service ]
```

### D. Asynchronous Request-Reply (RPC over MQ)
Enables synchronous-style client-server conversation over asynchronous message brokers.
- **Mechanism:**
  1. **Client** creates a temporary or shared response queue (the **Reply-To** queue).
  2. **Client** publishes a request with a unique **Correlation ID** and a `ReplyTo` queue name set in the message metadata headers.
  3. **Server** receives and processes the request.
  4. **Server** publishes the result back to the specified `ReplyTo` queue, attaching the identical **Correlation ID**.
  5. **Client** listens to the `ReplyTo` queue, reads the incoming correlation ID, matches it to its in-memory pending request, and unblocks the execution thread.

---

## 2. Advanced Integration Patterns

### A. Event Notification vs. Event-Carried State Transfer (ECST)
When designing event schemas, you must choose between these two core philosophies:

1. **Event Notification (Thin Events):**
   - *Concept:* The event contains only minimal metadata notifying that something occurred (e.g., `{"eventId": "999", "status": "Shipped"}`).
   - *Pros:* Small network footprint, simple schema evolution.
   - *Cons:* High back-channel traffic. When the Billing Service receives this event, it must make a synchronous REST/gRPC callback to the Order Service to fetch the order total before charging, re-coupling the services.

2. **Event-Carried State Transfer (ECST) (Fat Events):**
   - *Concept:* The event contains all necessary state data required by downstream consumers (e.g., `{"eventId": "999", "status": "Shipped", "items": [...], "price": 450.00, "customer": {...}}`).
   - *Pros:* Zero back-channel coupling. Consumers can complete their entire workflow independently using only the event payload.
   - *Cons:* Heavy payload sizes, complex schema governance, potential security concerns (over-sharing PII in public logs).

---

## 3. High-Impact Interview Questions & Answers

### Q1: Detail the step-by-step lifecycle of an asynchronous Request-Reply transaction. How does the client handle timeout errors?
* **Answer:**
  1. **Handshake Setup:** The client starts up and subscribes to a shared response queue (e.g., `client-response-queue`).
  2. **Request Generation:** The client creates a request payload, generates a unique UUID (the `Correlation ID`), and puts both into a message. It sets the `ReplyTo` header to `client-response-queue` and `CorrelationId` header to the UUID.
  3. **In-Memory Tracking:** The client registers the UUID in an in-memory map associated with a pending promise/future object and triggers a timer (e.g., 5000ms timeout).
  4. **Execution:** The server processes the request, computes the response, and publishes it back to `client-response-queue` with the same `CorrelationId`.
  5. **Resolution:** The client's queue listener receives the response, retrieves the `CorrelationId`, pulls the corresponding promise from its map, resolves it with the payload, and cancels the active timer.
  6. **Timeout Handling:** If the timer fires before the response arrives, the client deletes the UUID entry from its in-memory map and rejects the request promise with a `Gateway Timeout` error, freeing memory blocks.

### Q2: Why is Event-Carried State Transfer (ECST) preferred over Event Notification in high-scale microservices? Contrast the architectural trade-offs.
* **Answer:**
  - **Why ECST is Preferred:** ECST completely decouples services at runtime. Under Event Notification (thin events), downstream services must continuously query the upstream source system for details. During traffic bursts, this creates a massive wave of synchronous REST/gRPC queries that can crash the upstream service, defeating the purpose of asynchronous event-driven design.
  - **Trade-offs:**
    - **Thin Events:** Simple schemas, low bandwidth, but creates tight runtime query dependency (cascade failure risk).
    - **Fat Events (ECST):** Excellent resilience, zero back-channel queries, but requires strict schema evolution protocols (e.g., Avro compatibility) and increases data replication footprint since downstream services must store duplicates of upstream entity states locally.

### Q3: What are the risks of using a single temporary (ephemeral) response queue per request in the Request-Reply pattern?
* **Answer:**
  - **Resource Exhaustion:** Creating and tearing down a temporary queue in RabbitMQ or ActiveMQ for every single HTTP request consumes significant broker CPU and RAM resources. Under heavy load, this can overwhelm the message broker.
  - **Performance Hit:** Ephemeral queue creation incurs synchronous round-trips to the broker, slowing down request execution.
  - **The Engineering Solution:** Use a **shared, long-lived response queue** per client instance. To route messages to different concurrent threads, use the `Correlation ID` inside the application process to multiplex responses, rather than relying on separate physical queues.

### Q4: Contrast the "Choreography" vs. "Orchestration" patterns for distributed sagas. When do you use each?
* **Answer:**
  - **Choreography (Decentralized):** There is no central controller. Each service listens to events, performs its local action, and publishes a new event that triggers the next service.
    - *Pros:* High performance, highly decoupled, no single point of failure.
    - *Cons:* Hard to understand, debug, or trace as the system grows (spaghetti event flows).
    - *Use Case:* Simple, small distributed workflows with 2-4 steps.
  - **Orchestration (Centralized):** A central coordinator service (the Orchestrator) directs the workflow, telling each downstream service what to do via command-reply patterns and managing compensating transactions on failure.
    - *Pros:* Easy to trace, monitor, and enforce complex branching logic or timeouts.
    - *Cons:* The orchestrator can become a single point of failure and a bottleneck.
    - *Use Case:* Complex financial workflows, multi-step hotel/flight booking engines where state tracking is critical.