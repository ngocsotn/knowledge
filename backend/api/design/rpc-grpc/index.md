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

Protocol Buffers (Protobuf) is a schema-driven system for defining structured messages and serializing them to a compact binary wire format. A `.proto` schema can generate message types, serializers, and service stubs for supported languages.

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
- **Binary Encoding**: JSON is text-based and repeats field names in each message. Protobuf encodes a field number and wire type instead; values are not sent without keys.
- **Payload and CPU costs**: Protobuf can reduce payload size and parsing cost compared with JSON, especially for repeated structured messages. Actual gains depend on data, compression, runtime, and implementation; benchmark representative traffic rather than relying on fixed ratios.
- **Schema-First Code Generation**: Automatically compiles client stubs and server interfaces across multiple languages (Go, Java, Node.js, C#, Python) from a single `.proto` file.
- **Trade-offs**: Binary payloads are less readable in logs and require matching schema/tooling to inspect. Generated code and schema coordination add build and deployment complexity. JSON can be simpler for public APIs, browser clients, ad hoc debugging, or consumers without Protobuf support.

### Schema Basics: Fields, Types, and Presence

The schema defines message shape; each field has a type, a name, and a unique field number. Field numbers identify data on the wire and must remain stable after release.

```protobuf
syntax = "proto3";

message Profile {
  string id = 1;
  optional string display_name = 2;
  repeated string labels = 3;
  Status status = 4;

  enum Status {
    STATUS_UNSPECIFIED = 0;
    STATUS_ACTIVE = 1;
    STATUS_DISABLED = 2;
  }
}
```

- **Scalar fields** include booleans, strings, bytes, floating-point numbers, and signed or unsigned integers. Choose integer types based on expected range and wire characteristics; `sint32`/`sint64` use ZigZag encoding for efficient small negative values.
- **Messages and enums** compose structured data and symbolic values. In proto3, the first enum value must be `0`; use an explicit unspecified value rather than assigning a meaningful state to the default.
- **`repeated`** represents zero or more values. **`map`** represents key/value entries, but map entry order is not meaningful.
- **`oneof`** represents mutually exclusive alternatives. Setting one member clears the others in the generated API. Changing oneof membership can affect older readers, so evolve it carefully.
- **Presence matters**: ordinary proto3 scalar fields use their default when unset and generally do not distinguish absent from explicitly set-to-default. `optional` scalar fields track presence; message fields also have presence. Use presence when the distinction matters to updates or business logic.

Use field numbers from 1 through 15 for frequently used fields when practical; these take fewer bytes in the field key. Avoid numbers 19000 through 19999, reserved by Protobuf.

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
Unlike JSON, which serializes field names as text, Protobuf encodes a field key containing its field number and wire type. The field name and declared type are not repeated for every value; a decoder uses the schema to interpret the bytes.

```
Protobuf Binary Layout:
[Field Key: (Field Number 3, Wire Type 2)] -> [Value Length: 13] -> [Bytes: "test@test.com"]
```

1. **Wire types frame values:** The field key is `(field_number << 3) | wire_type`. Common wire types identify varints, 64-bit values, length-delimited values (strings, bytes, embedded messages, packed repeated fields), and 32-bit values. This lets a decoder skip unknown fields it can frame, but does not make arbitrary schema changes safe.
2. **Field numbers are permanent:** Renumbering a field makes old and new schema versions interpret the same bytes differently. Old readers may treat the new number as unknown, while the renamed field appears absent and receives a default value. This can cause silent data loss.
3. **Rules for safer schema evolution:** Compatibility depends on producer/consumer rollout direction and field usage. Common practices:
   * **Never reuse tag numbers:** After deleting a field, never assign its tag to another field.
   * **Reserve deleted fields:** Record both the number and name to prevent accidental reuse:
     ```protobuf
     message UserResponse {
       reserved 3, 10 to 15;
       reserved "temporaryToken";
       // ... active fields
     }
     ```
   * **Add fields compatibly:** New fields are generally ignored by old readers, but new readers see defaults when old producers omit them. Do not make a new field mandatory in application behavior until all relevant producers supply it.
   * **Do not casually change field types or meaning:** Some types share a wire type but differ in range, interpretation, or generated API. Other changes alter wire type and can make data unreadable. Review each change against Protobuf compatibility rules and deployment order.
   * **Handle enum evolution:** Add enum values without changing existing numbers. Older generated clients may encounter unknown values; application code should handle them safely.
   * **Be cautious with `oneof`, maps, and repeated fields:** Structural changes can lose presence or interpretation across versions even when payloads still parse.

---

### C. API Design and Runtime Behavior

- Keep message and field names clear; field numbers are the wire identity, while names shape generated APIs and JSON mappings.
- Use dedicated request and response messages instead of reusing one message for unrelated operations. This keeps validation and evolution local to each API.
- Use pagination for large collections rather than returning unbounded repeated fields.
- Define timestamps, money, identifiers, and optional values deliberately. Prefer standard well-known types or documented representations over ambiguous primitive conventions.
- Treat generated types as transport contracts, not necessarily as domain models. Map them at service boundaries when domain rules or lifecycle differ.
- Protobuf binary data is not self-describing. Store or transmit the schema/version context needed by tools that inspect data outside the RPC contract.

### D. Performance and Format Choice

Protobuf can reduce repeated field-name overhead and avoid text parsing, but it is not automatically faster or smaller for every workload. Small messages, large strings, compression, generated-runtime overhead, allocation patterns, and language implementation all affect results.

Benchmark serialization, deserialization, payload size, CPU, memory, and end-to-end latency using representative messages and deployed runtimes. Include compression in comparisons when production uses it. Avoid optimizing wire size at the cost of unclear schemas or unsafe compatibility.

Choose Protobuf for schema-driven service contracts, generated clients, and efficient typed communication across supported languages. Choose JSON when human readability, broad interoperability, browser use, or low-friction inspection matters more. Both formats still need validation, versioning discipline, and authorization at the service boundary.

### E. Security and Validation

Protobuf encoding provides neither encryption nor authentication. Use a protected transport such as TLS and apply authentication and authorization independently.

Treat decoded messages as untrusted input:

- Enforce request-size, nesting, collection-count, and processing limits appropriate to the service.
- Validate required business invariants explicitly; a message that parses is not necessarily valid or authorized.
- Avoid logging sensitive fields by default. Binary payloads may be difficult to inspect but can still contain personal or secret data.
- Keep Protobuf libraries and generated code current, and reject malformed or excessively large requests with explicit errors.

### F. Overcoming HTTP/2 Multiplexed Load Balancing Hot Spots
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
* **Answer:** Protobuf encodes each field key from its field number and wire type rather than serializing its name as text. The field number is part of the schema's wire identity. If it changes, old readers may treat the new number as unknown and see the old field as absent, which can cause silent data loss. When deleting a field, reserve both its number and name so neither can be reused.

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

### Q5: How does Protobuf represent field presence in proto3?
* **Answer:** Ordinary proto3 scalar fields use their default value when unset and generally do not distinguish absence from explicitly setting that default. `optional` scalar fields track presence, as do message fields. Choose explicit presence when updates or business rules need to distinguish “not provided” from “provided with the default.”

### Q6: What information is encoded in a Protobuf field key?
* **Answer:** The key combines the field number and wire type: `(field_number << 3) | wire_type`. The wire type tells the decoder how to read or skip the value; it does not identify the full schema type by itself.

### Q7: What are safe practices for evolving a Protobuf schema?
* **Answer:** Never reuse deleted field numbers; reserve deleted numbers and names. Add fields without requiring old producers to provide them immediately. Keep existing field numbers and meanings stable, add enum values without renumbering existing ones, and review changes for producer/consumer rollout order and compatibility.

### Q8: Does Protobuf make an API secure or validate business rules?
* **Answer:** No. Protobuf is serialization, not encryption, authentication, or authorization. Use TLS and access controls, enforce request limits, and validate business invariants after parsing.

### Q9: Is Protobuf always smaller and faster than JSON?
* **Answer:** No. Results depend on message shape, payload size, compression, runtime, and implementation. Benchmark representative production workloads; also weigh readability and interoperability.
