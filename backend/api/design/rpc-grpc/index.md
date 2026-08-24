# RPC and gRPC API Design

Comprehensive study guide for understanding, designing, and optimizing highly efficient RPC and gRPC APIs for microservices, low-latency communication, and high-performance distributed systems.

---

## 1. What is RPC & gRPC?

**RPC (Remote Procedure Call)** is an architectural paradigm where a computer program causes a procedure (subroutine/function) to execute in a different address space (commonly on another physical machine) without the programmer explicitly writing the details for this remote interaction. It abstractly makes remote API calls look like local function calls.

**gRPC** is an open-source, high-performance, universal RPC framework developed by Google in 2015. It builds upon:
1. **Protocol Buffers (Protobuf)** as the Interface Definition Language (IDL) and message serialization format.
2. **HTTP/2** as the underlying transport protocol.

---

## 2. Protocol Buffers (Protobuf) vs. JSON

Protocol Buffers is a strongly-typed binary serialization format.

```protobuf
syntax = "proto3";

package user;

message UserRequest {
  int32 id = 1;         // Field tag/number (essential for binary encoding)
  string locale = 2;
}

message UserResponse {
  int32 id = 1;
  string name = 2;
  string email = 3;
}

service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
}
```

### Why Protobuf is Highly Efficient:
- **Binary Encoding**: JSON is text-based and highly verbose (repeating field names in every message). Protobuf serializes data into a compact binary format, substituting keys with **Field Tags/Numbers** (e.g., field tag `1` instead of `"id"`).
- **Smaller Payload Size**: Protobuf payloads are typically **3 to 10 times smaller** than equivalent JSON, reducing network bandwidth.
- **Fast CPU Serialization**: Text parsing in JSON is highly CPU-intensive. Binary serialization/deserialization in Protobuf is **6 to 10 times faster** on CPU.
- **Schema-First Code Generation**: Automatically compiles client stubs and server interfaces across multiple languages (Go, Java, Node.js, C#, Python) from a single `.proto` file.

---

## 3. HTTP/2 Transport & Streaming Modes

gRPC runs strictly over **HTTP/2**, unlocking capabilities that traditional REST over HTTP/1.1 cannot support:

1. **Multiplexing**: Allows sending multiple concurrent requests and responses over a single TCP connection. Eliminates Head-of-Line (HoL) blocking at the HTTP/1.1 connection level.
2. **Header Compression (HPACK)**: Compresses HTTP headers. HPACK maintains an index of already-seen headers, sending only diffs to eliminate repetitive payload headers (e.g., `User-Agent`).
3. **True Bidirectional Streaming**: Enables low-latency client and server streaming over a single TCP connection.

### Four gRPC Streaming Paradigms

| Type | Description | Use Case |
| :--- | :--- | :--- |
| **Unary** | Standard request-response. Client sends one request, server returns one response. | Traditional CRUD APIs, fetch profiles. |
| **Server-Streaming** | Client sends one request; server returns a stream of multiple responses. | Real-time stock tickers, sports scores, log streaming. |
| **Client-Streaming** | Client sends a stream of multiple requests; server returns a single consolidated response. | Uploading large batches of telemetry, log ingestion. |
| **Bidirectional Streaming** | Both client and server send streams of independent messages concurrently. | Real-time chat apps, multiplayer gaming, video calling. |

---

## 4. Production-Scale Challenges & Engineering Solutions

Operating multiplexed gRPC connections across scalable cloud infrastructures introduces challenges regarding client browser networking, backward-compatible API schema updates, and L4 load-balancer hot spots.

### A. Browser Communication & The gRPC-Web Proxy Bridge
Standard browser environments cannot natively call gRPC endpoints directly due to fundamental browser-level HTTP restrictions.

1. **The Browser Constraint:**
   * **No Low-Level HTTP/2 Control:** Modern browser fetch/XHR APIs do not expose low-level control over raw HTTP/2 framing.
   * **Trailers Support:** gRPC relies heavily on **HTTP/2 Trailers** (transmitting custom headers after the response body completes, carrying the `grpc-status` and execution error messages). Browsers do not support trailers or inspect raw multiplexed frames.
2. **The gRPC-Web Solution:**
   To connect web frontends to gRPC microservices, implement **gRPC-Web**:
   * **The Client Protocol:** The client-side library compiles payloads into a specialized HTTP/1.1 or standard HTTP/2 payload format, encoding HTTP/2 trailers directly into the tail end of the binary body chunk.
   * **The Translator Proxy (Envoy):** A reverse proxy (e.g., Envoy, Traefik) sits in front of the backend services. It accepts the browser's incoming `gRPC-Web` requests, translates them back into standard RFC-compliant binary gRPC over HTTP/2, forwards them to the microservices, and translates responses back.

---

### B. Binary Encoding & Protocol Buffer Schema Evolution
Unlike JSON where field names are explicitly serialized (`{"email": "test@test.com"}`), Protobuf uses a compact, keyless binary serialization system governed by **Field Tags** (`string email = 3;`).

```
Protobuf Binary Layout:
[Field Number: 3 | Wire Type: 2] -> [Value Length: 13] -> [Bytes: "test@test.com"]
```

1. **Why Tags are Permanent:**
   The compiled client code reads the wire byte stream using offset numbers. If you alter a tag (e.g., modifying `string email = 3;` to `string email = 4;`), old client versions will continue looking for the email value at tag `3`. When they receive the new payload, they will fail to find tag `3`, discard tag `4` as an unknown field, and the user's email will silently resolve to an empty string.
2. **Rules for Safe Schema Evolution:**
   To guarantee seamless backward and forward compatibility:
   * **Never reuse tag numbers:** Never assign a retired field's tag to a new field.
   * **Use the `reserved` Keyword:** When deleting a field, declare its tag and name as reserved in the proto file to prevent other developers from reusing them in future versions:
     ```protobuf
     message UserResponse {
       reserved 3, 10 to 15;
       reserved "temporaryToken";
       // ... active fields
     }
     ```
   * **Never modify field types:** Changing `int32 id = 1` to `int64 id = 1` changes the wire-level encoding format, immediately corrupting deserialization pipelines for legacy clients.

---

### C. Overcoming HTTP/2 Multiplexed Load Balancing Hot Spots
Standard Layer 4 (TCP-level) load balancers fail catastrophically when distributing gRPC traffic across multiple backend nodes due to HTTP/2's long-lived connection model.

```
THE L4 BALANCER PROBLEM (HTTP/2 Connection Persistence)
Client A ───(Single TCP Conn / Multiplexed Streams)───> L4 Balancer ───> Server Instance 1 (100% Load)
Client B ───(Single TCP Conn / Multiplexed Streams)───> L4 Balancer ───> Server Instance 1 (Overloaded!)
                                                                         Server Instance 2 (Idle)
```

1. **The L4 Connection Persistence Problem:**
   In HTTP/1.1 REST, connections are short-lived. An L4 balancer distributes traffic by shifting TCP handshakes across servers.
   In HTTP/2 (gRPC), the client establishes **one** long-lived TCP connection, and multiplexes thousands of concurrent RPC requests over it. An L4 balancer only balances the *initial connection*. This causes all client traffic to stick to a single server instance, creating massive hot spots while neighboring servers sit idle.
2. **Engineering Solutions:**
   * **Option 1: L7 Proxy Load Balancing (Recommended):**
     Deploy a high-performance Layer 7 proxy sidecar or gateway (e.g., Envoy, Linkerd) in front of the gRPC backends. The proxy terminates incoming client HTTP/2 TCP connections, reads individual multiplexed RPC requests, and distributes them on a **request-by-request** basis across the backend server pool.
   * **Option 2: Client-Side Load Balancing:**
     The gRPC client utilizes a "thick" library that regularly queries a Service Registry (e.g., Kubernetes DNS, Consul) to discover all active backend server IP addresses. The client maintains an internal TCP connection pool to all IPs and uses a client-side algorithm (like Round-Robin or Least Loaded) to distribute RPC requests directly.

---

## 5. Interview Masterclass: High-Impact Q&As

### Q1: Why cannot standard web browsers make native gRPC calls directly, and how does gRPC-Web bypass this limitation?
* **Answer:** Standard browser clients cannot make direct gRPC calls for two key reasons:
  1. **HTTP/2 Trailers Control:** gRPC uses HTTP/2 trailers to transmit execution statuses and error metadata *after* the response body is fully sent. Browser Fetch/XHR APIs do not expose or support Trailer headers.
  2. **Low-Level Framing Access:** gRPC requires raw binary framing access over the wire, which browser execution environments strictly sandbox.
  * *The gRPC-Web Solution:* The client library wraps binary payloads in a custom HTTP/1.1 or standard HTTP/2 format, encoding the HTTP/2 trailers inside the actual tail end of the response body. A reverse proxy like **Envoy** is deployed in front of the gRPC backends to intercept browser requests, strip the custom encoding, translate the frames back into native gRPC, and route them downstream.

### Q2: What are "Field Tags" in Protobuf, and why is changing a field tag number after deployment a breaking change?
* **Answer:** In Protocol Buffers, field keys are not serialized as strings (e.g., `"email"`). Instead, they are encoded as highly compressed binary numbers called **Field Tags** (`string email = 3;`).
  * If a tag number is changed (e.g., to `4`), any legacy client still running old compiled code will continue reading the byte stream looking for tag `3`. When they receive the new payload, they will fail to locate tag `3`, discard tag `4` as an unknown field, and resolve the property to its empty default value, causing silent data loss.
  * *Deletion Rule:* To safely delete fields, declare the field tag and name as **`reserved`** in the `.proto` file to prevent future developers from accidentally reassigning that tag number.

### Q3: Explain why traditional L4 Load Balancers create backend "Hot Spots" in HTTP/2 / gRPC microservices, and how L7 proxies solve this.
* **Answer:**
  * **The Problem:** L4 (TCP-level) load balancers distribute traffic by balancing TCP connection handshakes. In HTTP/1.1 REST, connections are short-lived, so traffic balances naturally. In HTTP/2 (and gRPC), connections are highly persistent and multiplexed. Once a client establishes a connection, **all concurrent streams flow over that single TCP socket indefinitely**. An L4 balancer only balances the initial connection, leaving one backend node heavily overloaded with streams while others sit completely idle.
  * **The L7 Solution:** Deploying a Layer 7 proxy (e.g., Envoy or Linkerd) terminates incoming client HTTP/2 TCP sockets at the proxy boundary. The proxy reads and parses individual multiplexed gRPC stream frames inside the TCP tunnel, distributing them on a **request-by-request** basis across the downstream backend servers, ensuring optimal load distribution.

### Q4: Compare gRPC streaming paradigms and identify typical microservice use cases for each.
* **Answer:** gRPC supports four core communication patterns:
  1. **Unary (Request-Response):** Standard single request, single response. Used for typical CRUD APIs, fetching profiles, or editing records.
  2. **Server-Streaming (One-to-Many):** Client sends one request, and the server returns a persistent stream of multiple frames. Used for real-time stock price tickers, push notification feeds, or live logs.
  3. **Client-Streaming (Many-to-One):** Client streams multiple payload frames, and the server aggregates them into a single response once the stream closes. Used for massive IoT sensor telemetry ingestion or uploading multi-part files.
  4. **Bidirectional-Streaming (Many-to-Many):** Both sides stream data concurrently and independently over a single persistent TCP tunnel. Used for real-time collaborative whiteboards, multiplayer gaming, and video calling backends.

