# Other API Paradigms & Protocols

Comprehensive study guide covering SOAP, Webhooks, Server-Sent Events (SSE), and Long Polling, with architectural comparisons.

## 1. SOAP (Simple Object Access Protocol)
SOAP is a highly structured, strictly-defined protocol for exchanging data (using XML payloads only).
- **Core Features:** Strict contract definitions using WSDL (Web Services Description Language), built-in WS-Security, and ACID transactional guarantees.
- **Drawbacks:** Highly verbose; high CPU/network parse overhead compared to JSON REST or binary gRPC.

## 2. Webhooks (Reverse APIs)
Webhooks are user-defined HTTP callback endpoints. When an event occurs in the source system (e.g., Stripe processes a payment), the source system triggers an outgoing HTTP POST call directly to the target system's webhook URL asynchronously.
- **Security:** Always verify incoming signatures (e.g., `X-Signature` HMAC hash using a shared webhook secret) to prevent request-spoofing attacks.

## 3. Server-Sent Events (SSE)
SSE is a lightweight server-to-client unidirectional streaming protocol.
- **Mechanism:** Leverages standard persistent HTTP connections with headers:
  `Content-Type: text/event-stream`
  `Cache-Control: no-cache`
- **Use Cases:** Live news tickers, stock ticker updates, ChatGPT-style real-time text streaming.

## 4. Long Polling
An legacy real-time communication technique.
- **Mechanism:** Client requests data; server holds the HTTP connection open until new data arrives or a timeout occurs, returns the response, and forces the client to immediately open a new connection.
- **Drawback:** Inefficient; consumes massive concurrent connection pools under scale.

## Interview Questions & Answers

### Q1: What is the difference between WebSockets and Server-Sent Events (SSE)?
- **Answer:**
  - **WebSockets:** Bi-directional communication channel. Supports real-time read and write from both client and server over a single TCP socket. Essential for multiplayer games or collaborative editors.
  - **SSE:** Uni-directional (Server-to-Client) only. Runs over standard HTTP/1.1 or HTTP/2 protocols out-of-the-box, supporting automatic reconnection, event IDs, and firewall traversal without custom proxy configuration, making it simpler and lighter than WebSockets for simple feeds or text streams.
