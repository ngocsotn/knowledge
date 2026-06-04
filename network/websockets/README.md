# WebSockets

Comprehensive interview study guide covering the WebSocket protocol, connection handshakes, comparison with polling mechanisms, and distributed scaling strategies.

---

## 1. WebSockets vs. Polling Mechanisms

| Attribute | Short Polling | Long Polling | Server-Sent Events (SSE) | WebSockets |
| :--- | :--- | :--- | :--- | :--- |
| **Direction** | Client-Pull | Client-Pull | Server-Push only | **Bidirectional** (Full-Duplex) |
| **Connection Type** | Ephemeral HTTP | Semi-persistent HTTP | Persistent HTTP (text) | **Persistent TCP** (Binary/Text)|
| **Overhead** | Very High (repeated handshakes) | High (open connections block) | Low | **Extremely Low** (2-byte frame header) |
| **Use Case** | Dashboards with infrequent updates. | Legacy chat widgets. | Live news feeds, price tickers. | Real-time chat, collaborative docs, gaming. |

---

## 2. The WebSocket Connection Handshake

WebSockets run over standard TCP sockets but utilize HTTP to initiate the connection. This bypasses firewalls that typically block non-HTTP traffic.

1. **Client Request (The Upgrade HTTP GET):**
   ```http
   GET /chat HTTP/1.1
   Host: server.example.com
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
   Sec-WebSocket-Version: 13
   ```
2. **Server Response (The Acceptance 101):**
   The server computes a SHA-1 hash of the client's key appended with a standard UUID, verifying it accepts the upgrade:
   ```http
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
   ```

Upon 101 status, the HTTP protocol is discarded, and both sides communicate over the established TCP socket using lightweight WebSocket binary frames.

---

## 3. Scaling WebSockets (The Distributed Challenge)

Unlike stateless REST APIs, WebSockets are **highly stateful**. A client establishes a long-lived TCP connection directly to a **specific server instance**.

```
                   Client A      Client B
                      │             │
                      ▼             ▼ (WS Connections)
               ┌───────────┐ ┌───────────┐
               │ Server 1  │ │ Server 2  │
               └─────┬─────┘ └─────┬─────┘
                     │             │
                     ▼             ▼ (Pub/Sub)
               ┌─────────────────────────┐
               │     Redis Backplane     │ (Broadcast Messages)
               └─────────────────────────┘
```

### The Messaging Gap
If Client A is connected to Server 1, and Client B is connected to Server 2, Server 1 cannot send messages to Client B directly because it has no open socket to Client B.

### The Solution: Redis Pub/Sub Backplane
1. When Client A sends a chat message, Server 1 publishes the message payload to a Redis Pub/Sub channel (e.g., `channels:lobby`).
2. Server 2 subscribes to `channels:lobby` and receives the broadcast.
3. Server 2 instantly locates Client B's active WebSocket connection in its local memory and pushes the message down the socket.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: How do you configure Load Balancers (like Nginx) to support WebSockets?
* **Answer:** By default, load balancers expect short-lived, request-response HTTP connections. To support persistent WebSockets, you must:
  1. **Configure Protocol Upgrades:** Instruct Nginx to forward the `Upgrade` and `Connection` headers to upstream servers.
  2. **Enable Sticky Sessions (Session Affinity):** Ensure reconnecting client handshakes land on the same container if needed, or use a Redis backplane.
  3. **Tune Timeouts:** Increase read/write timeouts (e.g., `proxy_read_timeout 86400s`) to prevent Nginx from forcefully dropping quiet connections due to inactivity.

### Q2: How does the WebSocket protocol handle connection "Keep-Alives" (Liveness)?
* **Answer:** WebSockets natively support **Ping and Pong frames** at the protocol layer to detect dead connections (where the client crashes or loses Wi-Fi silently without sending a closing handshake):
  1. The server periodically transmits a binary **Ping** frame to the client.
  2. The client is legally required to respond immediately with a **Pong** frame.
  3. If the server misses multiple consecutive Pong frames within a timeout window, it terminates the socket locally to free up server file descriptors.

### Q3: What is the significance of the "Max File Descriptors" limit when building WebSocket servers?
* **Answer:** Since every WebSocket client maintains an active TCP connection, the operating system allocates one **File Descriptor (FD)** per socket. Linux systems enforce default limits (e.g., 1024 FDs per process). If your server hits this limit, it will reject all new connections with `EMFILE (Too many open files)` errors, regardless of available CPU or memory. To scale a Node.js or Go WebSocket application to support hundreds of thousands of concurrent connections, you must modify system resource limits (`ulimit -n`) and sysctl configurations (`fs.file-max`).
