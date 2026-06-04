# gRPC vs. REST APIs

Comprehensive interview study guide comparing REST and gRPC API architectures, payloads, network protocols, and communication paradigms.

---

## 1. REST vs. gRPC Comparison

| Attribute | REST | gRPC |
| :--- | :--- | :--- |
| **Protocol** | HTTP/1.1 (standard) or HTTP/2 | Strictly **HTTP/2** (requires HTTP/2 capabilities) |
| **Payload Format**| JSON (standard), XML, Plain Text | Binary serialization via **Protocol Buffers** (Protobuf) |
| **Typing** | Weak/Implicit (JSON has no native schema validation) | Strong/Strictly typed (defined in static `.proto` files) |
| **Performance** | Moderate (large verbose text payloads, no multiplexing) | Extremely High (tiny binary payloads, HTTP/2 multiplexing) |
| **Streaming** | Request-Response only (Server-Sent Events/WS separate) | Full bidirectional streaming, client streaming, server streaming |
| **Usage** | Public APIs, Browser-to-Backend integrations. | Internal Microservice-to-Microservice communication. |

---

## 2. Deep Dive: Why gRPC is Highly Performant

gRPC (Google Remote Procedure Call) utilizes two core technologies to optimize network and compute performance:

### 1. HTTP/2 Transport
Unlike HTTP/1.1, which establishes separate TCP connections or forces sequential request handling (Keep-Alive queues), HTTP/2 supports:
* **Multiplexing:** Allows clients and servers to send and receive multiple requests/responses simultaneously over a **single TCP connection**, eliminating network socket overhead and handshake delays.
* **Header Compression (HPACK):** Compresses heavy HTTP headers, saving network bandwidth.
* **Bidirectional Streaming:** Both client and server can open continuous data streams simultaneously.

### 2. Protocol Buffers (Protobuf)
* **How it works:** You define your data shapes and API routes inside a static schema file (`.proto`). A compiler generated code bindings for your chosen language (Go, Node, Python).
* **Binary Serialization:** JSON is verbose (storing keys like `"user_id"` and `"email"` as plain text in every record). Protobuf encodes fields into tiny, indexed binary formats, reducing payload sizes by up to 70-80% and skipping expensive string parsing steps.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: Why is gRPC highly preferred for internal microservices, but REST is still standard for public web APIs?
* **Answer:** Internal microservices communicate constantly over highly active networks, making gRPC's low-latency, tiny binary payloads, and HTTP/2 multiplexed connections ideal for saving CPU and bandwidth. However, public APIs target third-party developers and web browsers. Browsers cannot easily execute raw gRPC/HTTP/2 calls without proxy layers (like gRPC-Web), and developers expect standard, human-readable JSON payloads they can test easily via standard tools like Postman, cURL, or browser network inspectors.

### Q2: What is the difference between a gRPC Unary call and Streaming?
* **Answer:**
  * **Unary:** The classic request-response model. The client sends a single request payload and waits for the server to return a single response.
  * **Server Streaming:** The client sends one request, and the server returns a continuous stream of multiple responses (e.g., stock price updates or chat messages).
  * **Client Streaming:** The client streams a large volume of chunks (e.g., file upload), and the server returns a single final status.
  * **Bidirectional Streaming:** Both client and server stream data to each other simultaneously in real-time.

### Q3: How do you handle backward compatibility and API versioning in gRPC/Protobuf without breaking changes?
* **Answer:** Protobuf handles versioning statelessly using field index tags (e.g., `string email = 2;`). To maintain backward compatibility:
  1. **Never change the index tag** of an existing field.
  2. If you deprecate a field, do not delete it; mark it as `deprecated = true` or leave its index tag untouched so older clients can safely ignore it.
  3. New fields must be assigned a unique, unused index tag. Older servers will simply discard the new field during deserialization without crashing.
