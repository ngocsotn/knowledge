# HTTP Protocols: HTTP/1.1, HTTP/2, & HTTP/3

Comprehensive interview study guide covering HTTP evolution, multiplexing, TCP vs. UDP transport, status codes, and security.

---

## 1. Evolution of HTTP Protocols

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
| :--- | :--- | :--- | :--- |
| **Transport Layer** | TCP | TCP | **UDP** (via QUIC protocol) |
| **Connection Multiplexing** | No (Head-of-Line blocking) | Yes (Single TCP connection) | Yes (Single UDP connection) |
| **Header Compression** | None (Plain Text) | Yes (**HPACK** algorithm) | Yes (**QPACK** algorithm) |
| **Security (TLS)** | Optional | Optional (Strictly HTTPS in browsers) | **Mandatory** (TLS 1.3 integrated in QUIC) |
| **Head-of-Line Blocking** | At Application layer | At TCP layer (packet loss blocks all) | **None** (independent QUIC streams) |

---

## 2. Core Concepts

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

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is "TCP Head-of-Line (HoL) Blocking" and how does HTTP/3 resolve it?
* **Answer:** **TCP Head-of-Line Blocking** occurs when a packet is dropped or delayed at the network layer. Since TCP guarantees in-order delivery of bytes, the operating system kernel blocks the *entire* TCP connection—including all independent HTTP/2 multiplexed streams—until the lost packet is retransmitted and acknowledged. **HTTP/3** replaces TCP with **QUIC** (which runs over UDP). QUIC implements its own stream-aware congestion control. If a packet is lost, QUIC blocks *only* the specific stream that lost the packet, allowing all other streams to continue processing immediately.

### Q2: What is the difference between `PUT` and `PATCH` HTTP methods, and which is idempotent?
* **Answer:**
  * **`PUT`** is **idempotent**. It is used to *replace* an entire resource. The request payload must contain the complete updated entity. Executing it multiple times always results in the exact same resource state.
  * **`PATCH`** is **not idempotent**. It is used to apply *partial updates* to a resource (e.g., updating just one field). If the payload contains relative instructions (e.g., `{ "op": "add", "path": "/balance", "value": 10 }`), executing this PATCH request three times will add 30 to the balance, modifying state each time.

### Q3: What is the significance of the "101 Switching Protocols" status code?
* **Answer:** The **`101 Switching Protocols`** status code is returned by a server to indicate that it is accepting the client's request to upgrade the connection protocol. It is most commonly seen during the **WebSocket Handshake**, where the client sends an HTTP GET request with `Upgrade: websocket` headers, and the server returns a 101 status code to signal that it is transitioning the TCP socket from standard HTTP request-response to a persistent, full-duplex WebSocket connection.
