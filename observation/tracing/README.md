# Distributed Tracing

Comprehensive interview study guide covering Distributed Tracing in microservices, Context Propagation, Trace/Span IDs, OpenTelemetry, and Jaeger.

---

## 1. The Distributed Debugging Challenge

In a monolithic application, tracing a request is simple: you check a single application log, or trace the thread's execution stack frame.

In a microservices architecture:
* A single user action (e.g., placing an order) triggers a cascade of nested HTTP/gRPC requests across multiple independent services (e.g., Gateway ──► Order Service ──► Inventory ──► Shipping).
* If the request fails or is slow, checking individual service logs is useless because there is no obvious way to connect them.
* **Distributed Tracing** solves this by tracking the complete path of a request as it traverses across physical network boundaries.

---

## 2. Trace IDs, Span IDs, & Context Propagation

Distributed tracing relies on propagating unique metadata headers alongside every network call:

```
  Client ──► [Trace_ID: XYZ] ──► [API Gateway] ──► [Trace_ID: XYZ] ──► [Order Service]
            [Span_ID:  A  ]                      [Span_ID:  B  ]
                                                 [Parent_ID: A ]
```

* **Trace:** Represents the entire end-to-end journey of a single transaction/request from start to finish.
* **Trace ID:** A globally unique 128-bit identifier attached to the transaction upon entry. It remains identical across all microservice network hops.
* **Span:** Represents a single unit of work (e.g., an HTTP request, a DB query, or a function execution).
* **Span ID:** A unique identifier for that specific unit of work.
* **Parent Span ID:** Identifies which span triggered the current span, allowing tracing backbones to reconstruct the call graph hierarchically.

### Context Propagation
The process of serializing trace metadata (Trace ID, Span ID) into network request headers (e.g., standard W3C `traceparent` headers) so that downstream services can read them, create new child spans, and pass them further downstream.

---

## 3. OpenTelemetry (OTel) & Backends

* **OpenTelemetry (OTel):** A vendor-neutral, industry-standard collection of APIs, SDKs, and tooling used to generate, collect, and export telemetry data (metrics, logs, and traces).
* **Tracing Backends (e.g., Jaeger, Zipkin):** Receive exported OTel traces, store them, and provide UI search screens to visualize call flows as Gantt charts.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is "Context Propagation" in distributed tracing, and how does it traverse network boundaries?
* **Answer:** **Context Propagation** is the mechanism of transferring trace metadata across thread and network boundaries. When Service A executes an HTTP call to Service B:
  1. The tracing library **injects** the active trace metadata into the HTTP headers (e.g., W3C headers like `traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`).
  2. Service B receives the request and **extracts** the `traceparent` header.
  3. Service B's local tracing library starts a new child span, copying the parsed Trace ID, setting its `Parent_ID` to the received Span ID, and saving this metadata in local thread-local storage (or context) for future downstream propagation.

### Q2: What are the primary performance overheads of Distributed Tracing, and how do you mitigate them?
* **Answer:** Distributed tracing can degrade application performance if not configured carefully. Every span generates metadata, consumes memory, and requires network bandwidth to export logs to collectors (like Jaeger). We mitigate this using **Sampling**:
  * **Head-based Sampling:** The system decides whether to trace a request the moment it enters the gateway (e.g., tracing exactly 1% of requests). If selected, the flag propagates downstream. This is lightweight and predictable.
  * **Tail-based Sampling:** The system collects all spans and decides whether to save the trace *after* the request finishes. This is highly useful for saving 100% of slow requests (>500ms) or errors, while discarding boring successful traces, but requires a dedicated collector proxy layer.

### Q3: How do you propagate trace context across asynchronous Message Queues (like Kafka or RabbitMQ)?
* **Answer:** Context propagation across message queues works exactly like HTTP, but utilizes message metadata pockets instead of HTTP headers. In **Kafka**, you inject the Trace ID and Span ID into the **Kafka Record Headers** (key-value metadata arrays attached to every message). The consumer service extracts these headers from the received record, restores the trace context, and starts a child span. This allows you to track the exact latency starting from when the publisher sent the message, through the queue duration, up to the consumer's asynchronous execution.
