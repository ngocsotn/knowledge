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

## 3. The W3C Trace Context Specification

To ensure cross-vendor compatibility, the W3C Trace Context standard defines an exact, universally parsed format for the HTTP `traceparent` header.

```
                  W3C traceparent Header Format
                  
  traceparent: 00 - 4bf92f3577b34da6a3ce929d0e0e4736 - 00f067aa0ba902b7 - 01
               │   └────────────────┬───────────────┘ └────────┬───────┘   └─┬┘
             Version            Trace ID                    Span ID      Trace Flags
             (8-bit)           (128-bit)                    (64-bit)       (8-bit)
```

1. **Version (2 hex characters):** Currently `00`. Identifies the spec version.
2. **Trace ID (32 hex characters):** 16-byte (128-bit) unique identifier for the overall trace. Shared across all nodes.
3. **Parent ID / Span ID (16 hex characters):** 8-byte (64-bit) unique identifier for the parent caller span.
4. **Trace Flags (2 hex characters):** Currently `01` or `00`. `01` represents **Recorded/Sampled** (tell downstream nodes to record this trace); `00` means unsampled.

---

## 4. Production Go Implementation: Manual Context Propagation

In high-performance microservices, understanding how the trace context is injected and extracted at the byte level is critical. Below is a raw, dependency-free implementation in Go illustrating context propagation across a custom HTTP client and server:

```go
package main

import (
	"context"
	"crypto/rand"
	"encoding/hex"
	"fmt"
	"net/http"
	"strings"
)

type TraceContext struct {
	TraceID string
	SpanID  string
}

// key is a custom type to prevent context key collisions
type key int
const traceContextKey key = 0

// GenerateRandomHex creates cryptographic IDs
func GenerateRandomHex(bytes int) string {
	b := make([]byte, bytes)
	_, _ = rand.Read(b)
	return hex.EncodeToString(b)
}

// InjectTraceHeader serializes TraceContext to W3C format
func InjectTraceHeader(ctx context.Context, req *http.Request) {
	if tCtx, ok := ctx.Value(traceContextKey).(TraceContext); ok {
		// Format: version-traceId-spanId-flags
		headerVal := fmt.Sprintf("00-%s-%s-01", tCtx.TraceID, tCtx.SpanID)
		req.Header.Set("traceparent", headerVal)
	}
}

// ExtractTraceHeader parses W3C header and returns a context with TraceContext
func ExtractTraceHeader(req *http.Request) context.Context {
	headerVal := req.Header.Get("traceparent")
	if headerVal == "" {
		// No trace header found; initialize a fresh root trace
		return context.WithValue(req.Context(), traceContextKey, TraceContext{
			TraceID: GenerateRandomHex(16), // 128-bit
			SpanID:  GenerateRandomHex(8),  // 64-bit
		})
	}

	parts := strings.Split(headerVal, "-")
	if len(parts) != 4 || parts[0] != "00" {
		return req.Context()
	}

	tCtx := TraceContext{
		TraceID: parts[1],
		SpanID:  parts[2], // Becomes the parent span ID
	}
	return context.WithValue(req.Context(), traceContextKey, tCtx)
}

func main() {
	// 1. Client Side: Create root context
	rootCtx := context.WithValue(context.Background(), traceContextKey, TraceContext{
		TraceID: GenerateRandomHex(16),
		SpanID:  GenerateRandomHex(8),
	})

	req, _ := http.NewRequestWithContext(rootCtx, "GET", "http://internal-service/api", nil)
	InjectTraceHeader(rootCtx, req)

	fmt.Println("Sent Header:", req.Header.Get("traceparent"))

	// 2. Server Side: Extract context
	extractedCtx := ExtractTraceHeader(req)
	parsedCtx := extractedCtx.Value(traceContextKey).(TraceContext)
	fmt.Printf("Received - TraceID: %s, ParentSpanID: %s\n", parsedCtx.TraceID, parsedCtx.SpanID)
}
```

---

## 5. OpenTelemetry (OTel) & Backends

* **OpenTelemetry (OTel):** A vendor-neutral, industry-standard collection of APIs, SDKs, and tooling used to generate, collect, and export telemetry data (metrics, logs, and traces).
* **Tracing Backends (e.g., Jaeger, Zipkin):** Receive exported OTel traces, store them, and provide UI search screens to visualize call flows as Gantt charts.

---

## 6. Popular Interview Questions & High-Impact Answers

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

### Q4: Explain the exact W3C traceparent header byte layout. How does it dictate sampling behavior?
* **Answer:** The W3C `traceparent` header is a hyphen-delimited string split into four hexadecimal fields: `version-traceId-parentId-traceFlags`.
  1. `version` (8-bit, e.g., `00`): Defines the specification version.
  2. `traceId` (128-bit, 32 hex characters): Globally unique ID for the end-to-end trace.
  3. `parentId` (64-bit, 16 hex characters): Unique ID for the active caller's span.
  4. `traceFlags` (8-bit, 2 hex characters, e.g., `01` or `00`): Represents recording/sampling decisions. The least significant bit (LSB) acts as the **Sampled Flag** (`01` = Sampled, `00` = Not Sampled). If set to `01`, the downstream services are instructed to record and export trace span metrics to the telemetry backends.

### Q5: How do you prevent trace context from being lost when spawning asynchronous worker pools or thread transitions?
* **Answer:** In languages like Go or Java, trace context is attached to the executive thread's local storage (ThreadLocal) or passed explicitly via a `context.Context` object. When spinning up asynchronous pools (e.g., executing a task inside a new Go goroutine or a Java Executor pool):
  1. **Explicit Propagation:** You must capture the active trace context from the spawning thread *before* the transition.
  2. **Context Injection:** Pass the context explicitly as an argument to the worker task closure.
  3. **Re-activation:** Inside the asynchronous background thread, re-activate the captured context to the thread's local tracer instance before executing any database or HTTP child spans, preventing "detached spans" that float outside the main transaction trace graph.
