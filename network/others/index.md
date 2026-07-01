# Advanced Network Architecture & Lookups

Guide covering DNS mechanics, WebSocket scaling backplanes, and polling methods.

## 1. Meaning & The Global Lookup

The **Domain Name System (DNS)** is the hierarchical, decentralized database that translates human-readable domain names (like `google.com`) into computer-readable IP addresses (like `142.250.190.46`).

## Scaling WebSockets at Distributed Scale
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

## 4. Why WebSockets Run Over TCP and Not UDP

One of the most common system design questions is why the WebSocket protocol was designed on top of TCP rather than UDP, especially since real-time applications prioritize low latency.

```
┌────────────────────────────────────────────────────────┐
│ Why TCP?                                               │
├──────────────────────────┬─────────────────────────────┤
│ 1. HTTP Compatibility    │ Start with HTTP GET Upgrade │
│ 2. Reliability & Order   │ Guaranteed framing delivery │
│ 3. Port Multiplexing     │ Uses port 80/443 (Firewalls)│
│ 4. Congestion Control    │ Prevent network collapse    │
└──────────────────────────┴─────────────────────────────┘
```

### 1. HTTP Upgrade Compatibility (The Handshake)
The core requirement of the WebSocket protocol is that it must start with an HTTP Upgrade request (`101 Switching Protocols`). Because HTTP is structurally built on top of TCP, the initial WebSocket handshake *must* occur over a TCP connection. 
UDP has no concept of a connection or structured request/response cycles. Trying to implement an HTTP upgrade over UDP would require implementing an entire custom session management layer.

### 2. Message Framing and Strict Ordering
WebSockets transmit discrete text or binary frames. If bytes arrive out of order, or if a single frame's header is lost, the parser cannot reconstruct the message, resulting in corrupt data or socket termination.
* **TCP** guarantees that every packet is delivered and reassembled in the exact sequence it was sent.
* **UDP** is completely packet-oriented and unordered. If a frame was split across multiple UDP packets and one packet was lost or arrived late, the entire WebSocket message would be corrupted.

### 3. Firewall and Port Multiplexing (Standard Web Ports)
WebSockets multiplex traffic over standard web ports: **80** (HTTP) and **443** (HTTPS).
* Corporate firewalls and proxies are highly restrictive. They generally allow TCP port 443 through because it is standard HTTPS traffic. They routinely block arbitrary UDP ports to prevent DDoS attacks, DNS amplification, and security leaks.
* Operating over TCP on port 443 ensures WebSockets work seamlessly on almost any internet connection worldwide without special network/firewall configurations.

### 4. Built-in Congestion Control
Stateful, real-time communication can transmit high volumes of data. Without TCP's sliding window and congestion control (Cubic/BBR), a massive burst of WebSocket messages could easily overwhelm intermediate network routers, resulting in packet drop cascades and network collapse.

---

## 5. Proxy/Reverse-Proxy Configuration (Nginx, AWS ALB, and Timeouts)

When running WebSockets in production, you never expose the application server directly to the internet. You run it behind a load balancer or reverse proxy like Nginx or AWS Application Load Balancer (ALB). This requires custom protocol configuration.

### 1. Nginx WebSocket Configuration
By default, Nginx terminates connections after a short period of inactivity (typically 60 seconds). To keep a WebSocket open and allow headers upgrade, use this production-ready configuration:

```nginx
http {
    # Map upgrade headers dynamically
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    upstream websocket_backend {
        server 127.0.0.1:8080;
        keepalive 32; # Keep-alive idle connections to backend
    }

    server {
        listen 443 ssl;
        server_name api.example.com;

        location /ws/ {
            proxy_pass http_websocket_backend;
            
            # Enable HTTP/1.1 (required for Upgrade)
            proxy_http_version 1.1;
            
            # Forward WebSocket handshake headers
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection $connection_upgrade;
            
            # Forward client IP metadata
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

            # Custom Timeout Adjustments
            # Set to 1 day (86400s) so Nginx does not close idle connections
            proxy_read_timeout 86400s;
            proxy_write_timeout 86400s;
        }
    }
}
```

### 2. AWS Application Load Balancer (ALB) Configuration
AWS ALBs natively support WebSockets because they support HTTP/1.1. However, they enforce a default **Idle Timeout** of 60 seconds.
* If no data is transmitted in either direction for 60 seconds, the ALB terminates the TCP connection.
* **The Solution:** You must either increase the ALB's Idle Timeout in the AWS Console (up to a maximum of 4000 seconds) OR implement a periodic application-level keep-alive (Ping/Pong or custom heartbeat) every 30-50 seconds to keep the connection active.

---

## 6. Popular Interview Questions & High-Impact Answers

### Q1: How do you configure Load Balancers (like Nginx) to support WebSockets?
* **Answer:** By default, load balancers expect short-lived, request-response HTTP connections. To support persistent WebSockets, you must:
  1. **Configure Protocol Upgrades:** Instruct Nginx to forward the `Upgrade` and `Connection` headers to upstream servers (using the `map` directive in Nginx or setting headers explicitly).
  2. **Enable Sticky Sessions (Session Affinity) if needed:** Ensure reconnecting client handshakes land on the same container if the application maintains local connection state, or decouple state via a Redis pub/sub backplane.
  3. **Tune Timeouts:** Increase proxy read/write timeouts (e.g., `proxy_read_timeout 86400s`) so Nginx does not forcefully disconnect idle sessions.

### Q2: How does the WebSocket protocol handle connection "Keep-Alives" (Liveness)?
* **Answer:** WebSockets natively support **Ping and Pong frames** at the protocol layer to detect dead connections (where the client crashes or loses Wi-Fi silently without sending a closing handshake):
  1. The server periodically transmits a binary **Ping** frame to the client.
  2. The client is legally required to respond immediately with a **Pong** frame.
  3. If the server misses multiple consecutive Pong frames within a timeout window, it terminates the socket locally to free up server file descriptors.

### Q3: What is the significance of the "Max File Descriptors" limit when building WebSocket servers?
* **Answer:** Since every WebSocket client maintains an active TCP connection, the operating system allocates one **File Descriptor (FD)** per socket. Linux systems enforce default limits (e.g., 1024 FDs per process). If your server hits this limit, it will reject all new connections with `EMFILE (Too many open files)` errors, regardless of available CPU or memory. To scale a Node.js or Go WebSocket application to support hundreds of thousands of concurrent connections, you must modify system resource limits (`ulimit -n`) and sysctl configurations (`fs.file-max`).
