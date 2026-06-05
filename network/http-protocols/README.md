# HTTP Protocols: From HTTP/0.9 to HTTP/3 (QUIC)

Comprehensive interview study guide covering the complete evolution of HTTP, detailed architectural and performance comparisons, binary framing layers, header compression algorithms, TLS handshake latencies, and modern UDP routing.

---

## 1. Complete Evolution & Feature Matrix

The Hypertext Transfer Protocol (HTTP) has evolved from a single-line text protocol to a complex binary, multiplexed transport system.

| Protocol Version | Transport Layer | Format Style | Multiplexing Support | Header Compression | Security / TLS Integration | Latency (Connection Handshake) |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **HTTP/0.9** (1991) | TCP | Raw ASCII Text | None (No headers) | None | None | 1 RTT (TCP) |
| **HTTP/1.0** (1996) | TCP | ASCII Text | None (Non-persistent connection) | None | Optional | 1 RTT (TCP) + 1 RTT per request |
| **HTTP/1.1** (1997) | TCP | ASCII Text | None (**Keep-Alive** persistent serial) | None | Optional (TLS separated) | 1 RTT (TCP) + 2 RTT (TLS 1.2) = 3 RTT |
| **HTTP/2** (2015) | TCP | **Binary Frames** | **Yes** (Multiplexed single TCP) | Yes (**HPACK** Huffman) | Optional (Mandatory in browsers) | 1 RTT (TCP) + 1 RTT (TLS 1.3) = 2 RTT |
| **HTTP/3** (2022) | **UDP (via QUIC)**| **Binary Frames** | **Yes** (Multiplexed independent UDP) | Yes (**QPACK** stream-safe) | **Mandatory** (TLS 1.3 built-in) | **1 RTT (TCP+TLS merged)** / **0 RTT (Resumption)** |

---

## 2. In-Depth Historical Protocol Milestones

### A. HTTP/0.9 (The One-Line Protocol)
- **Mechanics**: Extremely primitive. The client sends a single ASCII line: `GET /index.html` followed by a carriage return.
- **Constraints**: No HTTP headers, no status codes (errors had to be sent as text inside the HTML!), and no metadata. Only supported HTML files.
- **Connection**: Closed instantly by the server after returning the HTML stream.

### B. HTTP/1.0 (The Metadata Expansion)
- **Features**: Introduced **HTTP Headers** for both requests and responses. Enabled sending metadata (e.g., `Content-Type: image/png` or `User-Agent`), **HTTP Status Codes** (`200 OK`, `404 Not Found`), and content negotiation.
- **The Performance Problem (Connection Overhead)**: Connections were non-persistent by default. To download a page with 10 images, the browser had to open and close **11 separate TCP connections** sequentially. Every connection paid the price of a 3-way TCP handshake, causing massive network latency.

### C. HTTP/1.1 (The Persistent Standard)
- **Features**: Introduced **Keep-Alive** persistent connections by default, allowing a single TCP socket to be reused for multiple request-response cycles. Added `Host` header (enabling virtual hosting of multiple domains on a single IP), chunked transfer encoding, and cache-control headers.
- **The Bottleneck (Application HoL Blocking)**: Although the TCP connection remains open, HTTP/1.1 requires requests over a single connection to be resolved **sequentially**. If Request 1 (e.g., a slow database API) is slow, Request 2 (a tiny CSS file) is completely blocked behind it inside the queue. This is called **Application Head-of-Line (HoL) Blocking**.

### D. HTTP/2 (The Binary Multiplexing Leap)
- **Features**: Bypasses text parsing by introducing the **Binary Framing Layer**. Messages are split into independent **Frames** (e.g., `HEADERS` frames, `DATA` frames).
- **Multiplexing**: Interleaves multiple dynamic streams concurrently over a **single TCP connection**. No more queuing at the application layer.
- **Header Compression (HPACK)**: Resolves header redundancy (e.g., sending the same huge `User-Agent` and `Cookie` string on every request) using a static and dynamic table algorithm (HPACK), compressing headers by up to 85%.

### E. HTTP/3 (The UDP & QUIC Revolution)
- **The Bottleneck (TCP HoL Blocking)**: If a single TCP packet is dropped over a poor network (Wi-Fi/4G), the OS kernel halts the *entire* TCP window, blocking all multiplexed HTTP/2 streams until the missing packet is retransmitted.
- **The Solution (QUIC over UDP)**: HTTP/3 replaces TCP with **QUIC**, which runs on top of connectionless **UDP**. QUIC implements its own stream-aware congestion control. If a packet is lost, QUIC blocks *only* the specific stream that lost the packet, allowing all other streams to continue processing immediately.
- **TLS 1.3 Integration**: QUIC integrates the TLS 1.3 cryptographic handshake directly with the transport handshake, bringing connection establishment latency down to **1 RTT** (or **0 RTT** for subsequent resumptions).

---

## 3. Core Concepts

### 1. HTTP/2 Multiplexing
In HTTP/1.1, browsers had a limit of ~6 concurrent TCP connections per domain. Additional requests had to queue up (Head-of-Line blocking).
* **HTTP/2** solves this by breaking messages down into independent **binary frames** and interleaving them over a **single TCP connection**. This allows downloading HTML, CSS, images, and JS files simultaneously without connection overhead.

### 2. HTTP/3 QUIC (Why UDP?)
Even with HTTP/2 multiplexing, if a single TCP packet is lost over the network:
* TCP halts the entire connection, blocking *all* multiplexed streams until the lost packet is retransmitted (TCP Head-of-Line blocking).
* **HTTP/3** replaces TCP with **QUIC** (built on UDP). QUIC manages multiplexing at the protocol level: each stream is completely independent. If a packet in stream A is lost, stream B and C continue transmitting without delay.

### 3. Safe vs. Idempotent Methods
* **Safe Methods:** Do not modify server state (e.g., `GET`, `HEAD`, `OPTIONS`).
* **Idempotent Methods:** Multiple identical requests have the exact same effect on server state as a single request (e.g., `GET`, `PUT`, `DELETE`, `OPTIONS`). `POST` and `PATCH` are **not** idempotent.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is "TCP Head-of-Line (HoL) Blocking" and how does HTTP/3 resolve it?
* **Answer:** **TCP Head-of-Line Blocking** occurs when a packet is dropped or delayed at the network layer. Since TCP guarantees in-order delivery of bytes, the operating system kernel blocks the *entire* TCP connection—including all independent HTTP/2 multiplexed streams—until the lost packet is retransmitted and acknowledged. **HTTP/3** replaces TCP with **QUIC** (which runs over UDP). QUIC implements its own stream-aware congestion control. If a packet is lost, QUIC blocks *only* the specific stream that lost the packet, allowing all other streams to continue processing immediately.

### Q2: What is the difference between `PUT` and `PATCH` HTTP methods, and which is idempotent?
* **Answer:**
  * **`PUT`** is **idempotent**. It is used to *replace* an entire resource. The request payload must contain the complete updated entity. Executing it multiple times always results in the exact same resource state.
  * **`PATCH`** is **not idempotent**. It is used to apply *partial updates* to a resource (e.g., updating just one field). If the payload contains relative instructions (e.g., `{ "op": "add", "path": "/balance", "value": 10 }`), executing this PATCH request three times will add 30 to the balance, modifying state each time.

### Q3: What is the significance of the "101 Switching Protocols" status code?
* **Answer:** The **`101 Switching Protocols`** status code is returned by a server to indicate that it is accepting the client's request to upgrade the connection protocol. It is most commonly seen during the **WebSocket Handshake**, where the client sends an HTTP GET request with `Upgrade: websocket` headers, and the server returns a 101 status code to signal that it is transitioning the TCP socket from standard HTTP request-response to a persistent, full-duplex WebSocket connection.
