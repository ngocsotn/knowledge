# TCP vs. UDP Transport Layer

Comprehensive interview study guide covering the Transport Layer, TCP 3-Way Handshake, 4-Way Teardown, Congestion/Flow control, and UDP mechanics.

---

## 1. TCP vs. UDP Comparison

| Attribute | TCP (Transmission Control Protocol) | UDP (User Datagram Protocol) |
| :--- | :--- | :--- |
| **Connection State** | Connection-oriented (requires handshake) | Connectionless (fire and forget) |
| **Reliability** | Guaranteed delivery (retransmission) | Best-effort delivery (packet loss possible)|
| **Data Ordering** | Guaranteed in-order byte stream | Unordered datagram packets |
| **Control Features** | Flow Control & Congestion Control | None |
| **Speed / Overhead** | Slower (20-byte header, handshake latency) | Faster (8-byte header, zero latency) |
| **Best-Fit Cases** | Web browsing (HTTP), Email (SMTP), SSH, DB. | Live streaming, DNS, VoIP, Online gaming. |

---

## 2. Core TCP Lifecycles

### 1. TCP 3-Way Handshake (Establish Connection)
Before any data can be sent, TCP must establish a synchronized connection:
1. **SYN (Synchronize):** Client chooses a random sequence number `Seq=x` and sends a SYN packet to the server.
2. **SYN-ACK:** Server receives SYN, chooses its own random sequence number `Seq=y`, and returns a SYN-ACK packet with acknowledgement number `Ack=x+1`.
3. **ACK:** Client receives SYN-ACK, and sends an ACK packet with `Ack=y+1` to complete the handshake.

```
      Client                               Server
        │                                    │
        │─── SYN (Seq=x) ───────────────────►│ (Listen)
        │                                    │
        │◄── SYN-ACK (Seq=y, Ack=x+1) ───────│ (SYN Received)
        │                                    │
        │─── ACK (Ack=y+1) ─────────────────►│ (Established)
        ▼                                    ▼
```

### 2. TCP 4-Way Teardown (Close Connection)
Since TCP is bidirectional, both sides must close their transmission channels independently:
1. **FIN:** Client sends a FIN packet to signal it has no more data to send.
2. **ACK:** Server returns an ACK, closing the client's write channel.
3. **FIN:** Server finishes sending its outstanding data and sends its own FIN packet.
4. **ACK:** Client returns an ACK, closing the server's write channel. Client enters `TIME_WAIT` state before fully releasing the connection.

---

## 3. TCP Congestion vs. Flow Control

* **Flow Control (Sliding Window):** Prevents the sender from overwhelming the **receiver's memory**. The receiver advertises its "Receive Window" (available buffer size) in every ACK packet, telling the sender how much data it can safely transmit.
* **Congestion Control:** Prevents the sender from overwhelming the **network infrastructure** (routers/switches). It manages a "Congestion Window" (cwnd) dynamically using four core phases:
  1. **Slow Start:** Starts with a small cwnd and doubles it every round-trip time (RTT).
  2. **Congestion Avoidance:** When cwnd reaches a threshold, it transitions to linear growth (+1 packet per RTT) to probe network limits safely.
  3. **Fast Retransmit:** If three duplicate ACKs are received, TCP assumes a packet was lost, instantly retransmits it without waiting for timeout, and cuts cwnd in half.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: Why does TCP require a 3-way handshake instead of a 2-way handshake?
* **Answer:** A 3-way handshake is mathematically required to **prevent delayed, stale connection requests** from establishing phantom sessions. In a 2-way handshake (where the connection is active immediately upon server SYN-ACK), if an old client SYN packet was delayed in the network, it could arrive at the server *after* the client already disconnected. The server would respond with an ACK and allocate resources for a connection that the client will ignore, causing memory leaks. The third ACK from the client proves that both sides are active and agree on sequence numbers.

### Q2: What is the purpose of the `TIME_WAIT` state during TCP teardown, and why is it usually 2×MSL (Maximum Segment Lifetime)?
* **Answer:** The client enters the `TIME_WAIT` state after sending its final ACK for two reasons:
  1. **Ensure Reliable Teardown:** If the final ACK is lost, the server will assume its FIN failed and retransmit it. The client must stay alive to receive this retransmitted FIN and return another ACK.
  2. **Prevent Port Pollution:** It allows any delayed, duplicate packets belonging to the active connection to naturally expire and drain from the network, preventing them from being misread as valid data in a future connection using the same IP and port.

### Q3: How does UDP-based QUIC implement reliability if UDP itself is unreliable?
* **Answer:** UDP is a barebones, connectionless transport layer with no concept of handshakes, sequence numbers, or retransmissions. However, QUIC implements **reliability, flow control, congestion control, and connection tracking entirely at the Application Layer**. QUIC manages sequence numbers, detects packet loss, and triggers retransmissions inside its own software protocol, bypassing the operating system kernel's TCP stack for faster performance and lower latency.
