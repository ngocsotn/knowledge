# Computer Network Protocols

Comprehensive study guide covering HTTP, HTTP Versions, HTTPS, WebSockets, gRPC, TCP, and UDP protocols.

## 1. Complete Evolution & Feature Matrix

The Hypertext Transfer Protocol (HTTP) has evolved from a single-line text protocol to a complex binary, multiplexed transport system.

| Protocol Version | Transport Layer | Format Style | Multiplexing Support | Header Compression | Security / TLS Integration | Latency (Connection Handshake) |
| :

## TCP & UDP Transport Layers
## 1. TCP vs. UDP Comparison

| Attribute | TCP (Transmission Control Protocol) | UDP (User Datagram Protocol) |
| :

## WebSockets Handshake
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
