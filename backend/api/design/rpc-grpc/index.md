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

## 4. Hard Interview Questions & Deep Answers

### Q1: Why cannot browsers natively call gRPC services, and how does gRPC-Web resolve this limitation?
**Answer**:
- **Why Browsers Can't Call gRPC**:
  1. **Low-Level HTTP/2 Control**: Browsers do not expose low-level control over HTTP/2 frames to JavaScript. Standard browser APIs (Fetch or XMLHttpRequests) cannot mandate or inspect specific HTTP/2 frame structures, such as trailers (gRPC uses **HTTP/2 Trailers** for returning status codes and metadata at the end of a stream).
  2. **Binary Framing**: Browsers struggle with the raw binary framing of standard gRPC over HTTP/2 without middleware.
- **How gRPC-Web Solves This**:
  1. **gRPC-Web Protocol**: Translates standard gRPC frames into browser-compatible HTTP/1.1 or HTTP/2 streams. It encodes HTTP/2 trailers into the response body as a special base64 or binary chunk at the end.
  2. **Envoy Proxy**: A reverse proxy (like Envoy) is deployed in front of the gRPC services. It accepts incoming `gRPC-Web` requests from browsers (HTTP/1.1 or HTTP/2), parses them, strips the custom encoding, translates them into standard, high-performance `gRPC` over HTTP/2, and routes them to backend gRPC services.

### Q2: What are "Field Tags" in Protobuf, and why can you never change a field tag once a service is deployed?
**Answer**:
- **Field Tags**: In a `.proto` file, fields are defined with tags: `string email = 3;`. In binary format, Protobuf does not write the word `"email"`. It writes a binary header containing the field number (`3`) and wire type, followed by the serialized value.
- **Why tags cannot be changed**:
  1. **Backward & Forward Compatibility**: If you change `string email = 3;` to `string email = 4;`, existing clients running old compiled code will still expect email at tag `3`. When they receive tag `4`, they will treat it as an unknown field and discard it. Conversely, if you assign a new field to tag `3`, old clients will misinterpret the new data as the old field.
  2. **Rules for Schema Evolution**:
     - Never change the numeric tags of existing fields.
     - Never delete fields; instead, mark them as `reserved` (e.g., `reserved 3;`) to prevent other developers from accidentally reusing that tag number in the future.

### Q3: How do you implement load balancing for gRPC microservices, given HTTP/2's long-lived connection model?
**Answer**:
- **The gRPC Load Balancing Problem**:
  In HTTP/1.1, clients create and tear down TCP connections frequently. L4 (TCP-level) load balancers can distribute these connections easily.
  In HTTP/2 (and therefore gRPC), connections are long-lived and multiplexed. Once a client establishes a connection to a specific backend instance, **all subsequent RPC requests flow over that single connection**. A standard L4 load balancer will route all client traffic to a single instance, leading to massive hot spots and idle servers.
- **Solutions**:
  1. **Client-Side Load Balancing (Thick Client)**:
     - Client queries a Service Registry (e.g., Consul, Kubernetes DNS) to discover all available backend IP addresses.
     - The client maintains a sub-connection pool to all IPs and uses client-side algorithms (e.g., Round Robin, Least Loaded) to distribute RPC requests across connections.
     - *Cons*: Highly language-dependent; changes to balancing policies require updating and redeploying all client services.
  2. **L7 Proxy Load Balancing (Thin Client - Recommended)**:
     - All gRPC clients connect to a stateless Layer 7 proxy (e.g., Linkerd, Envoy, Traefik).
     - The proxy terminates the incoming HTTP/2 connections, acts as a high-performance multiplexer, inspects individual RPC requests, and routes them to backends on a request-by-request basis.
     - *Cons*: Adds a minor network hop latency; increases infrastructure management overhead.
