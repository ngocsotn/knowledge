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

## 3. TCP Sliding Window & Head-of-Line (HOL) Blocking

### 1. Sliding Window Mechanics (Flow Control)
TCP operates on a byte-stream model where every sent byte is assigned a unique sequence number.
* **Sender Window:** Divided into three segments:
  1. *Sent & Acknowledged:* Bytes already received and confirmed by the receiver.
  2. *Sent but Unacknowledged:* Bytes currently in flight. Must not exceed the receiver's advertised window.
  3. *Not Yet Sent but Ready:* Bytes within the allowed window that can be sent immediately.
* **Receiver Window (rwnd):** Advertised in the TCP header of every ACK packet. It communicates the exact buffer size available in the receiver's kernel space. If `rwnd = 0` (Zero Window), the sender must freeze transmission and periodically send 1-byte **Zero Window Probes** to check if buffer space has cleared.

### 2. TCP Head-of-Line (HOL) Blocking
Because TCP guarantees strict in-order delivery of bytes, a single lost packet blocks the delivery of all subsequent packets—even if those packets arrived safely in the receiver's buffer. The OS kernel refuses to pass out-of-order bytes to the application layer.

```
 Packet 1       Packet 2 (LOST)      Packet 3       Packet 4
┌─────────┐      ┌─────────┐      ┌─────────┐      ┌─────────┐
│ Seq=100 │      │ Seq=200 │      │ Seq=300 │      │ Seq=400 │
└────┬────┘      └────┬────┘      └────┬────┘      └────┬────┘
     │                │                │                │
     ▼                ▼                ▼                ▼
┌────────────────────────────────────────────────────────────┐
│ Receiver Buffer (Blocked! Wait for retransmit of Seq=200)   │
└────────────────────────────────────────────────────────────┘
```

* **The HTTP/2 HOL Bottleneck:** In HTTP/2, multiple logical streams are multiplexed over a **single TCP connection**. If one TCP packet containing bytes for Stream A is lost, it blocks *all other streams* (Stream B, Stream C) sharing that TCP connection, causing systemic latency.
* **The HTTP/3 (QUIC) Solution:** QUIC runs over UDP. It implements independent, stream-level framing. A lost packet in Stream A only halts Stream A; Stream B and Stream C continue streaming unimpeded.

---

## 4. Congestion Control Algorithms (Tahoe vs. Reno vs. Cubic vs. BBR)

Congestion control dynamically sizes the sender's congestion window (`cwnd`) to match network capacity.

```
       cwnd (Congestion Window)
        ▲
        │           Reno (Fast Recovery)
        │                / \     /\
        │               /   \---/  \
        │   Tahoe      /     \     
        │    /\       /       \    
        │   /  \     /         \   
        │  /    \___/           \  
        │ /                       \
        └───────────────────────────────► Time
```

### 1. TCP Tahoe (Loss-Based, Classic)
* **Behavior:** On packet loss (detected via RTO timeout or 3 duplicate ACKs), Tahoe immediately drops `cwnd` back to **1 MSS (Maximum Segment Size)**, resets the slow start threshold (`ssthresh = cwnd / 2`), and re-enters the exponential Slow Start phase.
* **Flaw:** Extremely aggressive drop penalty. Causes massive throughput drops on high-bandwidth networks.

### 2. TCP Reno (Loss-Based, Classic Modern)
* **Behavior:** Differentiates between RTO timeout and 3 duplicate ACKs:
  * *RTO Timeout:* Drop `cwnd` to 1 MSS (same as Tahoe).
  * *3 Duplicate ACKs (Fast Recovery):* Instead of dropping to 1, Reno halves `cwnd` (`cwnd = cwnd / 2`), sets `ssthresh = cwnd`, and continues growing linearly (Congestion Avoidance) from that halved value.
* **Flaw:** Assumes *all* packet loss is caused by network congestion. Over wireless or noisy networks, random packet drops trigger unnecessary throttling.

### 3. TCP Cubic (Loss-Based, Linux/macOS Default)
* **Behavior:** Growth is modeled as a cubic function ($C = t - K$) of time since the last packet loss, rather than growing linearly over RTT.
* **Benefits:** 
  * Independent of RTT, making it highly equitable for streams with different RTTs.
  * Grows very fast initially, plateaus near the last known capacity bottleneck to probe safely, then accelerates again if no loss occurs.
  * Highly optimized for **LFNs (Long Fat Networks)** with high bandwidth-delay products.

### 4. Google BBR (Bottleneck Bandwidth and RTT - Model-Based)
* **Behavior:** Ignores packet loss as a primary signal. Instead, BBR continuously measures actual **Maximum Bottleneck Bandwidth (MaxBW)** and **Minimum Round-Trip Time (MinRTT)**.
* **Optimal Operating Point:** It keeps the inflight data volume exactly equal to the **BDP (Bandwidth-Delay Product)**:
  $$\text{BDP} = \text{MaxBW} \times \text{MinRTT}$$
* **Benefits:** 
  * Prevents **Bufferbloat** (routers queueing excessive packets, causing latency spikes).
  * Maintains near-maximum throughput even in noisy environments with 15% random packet loss, where Reno/Cubic would throttle throughput to zero.

---

## 5. Detailed TCP State Machine and Packet Dissection

The OS kernel tracks connection state transitions for every TCP socket.

```
      +---------+
      |  CLOSED |
      +----+----+
           | Active Open
           v
      +----+----+
      |SYN_SENT |
      +----+----+
           | SYN-ACK
           v
      +----+----+        Passive Open       +---------+
      | ESTABL- |◄──────────────────────────┤  LISTEN |
      |  ISHED  |                           +---------+
      +----+----+
           | Active Close (FIN)
           v
      +----+----+
      |FIN_WAIT1|
      +----+----+
           | ACK
           v
      +----+----+
      |FIN_WAIT2|
      +----+----+
           | FIN (from Server)
           v
      +----+----+
      |TIME_WAIT| ====> (Wait 2 * MSL) ====> CLOSED
      +---------+
```

### Dissecting Key Connection States
* **LISTEN:** Server is waiting for incoming client SYN packets.
* **SYN_SENT / SYN_RCVD:** Handshake in progress. SYN flood attacks target sockets in `SYN_RCVD` state.
* **ESTABLISHED:** Bi-directional data transfer channel active.
* **FIN_WAIT_1:** Client sent FIN, waiting for ACK from server.
* **CLOSE_WAIT:** Server received client's FIN, sent ACK, but waiting for server-side application to call `close()` to send its own FIN. High count of sockets in `CLOSE_WAIT` indicates **application-level resource leaks**.
* **TIME_WAIT:** Connection closed. Client waits 2×MSL to ensure delayed packets drain. High count of sockets in `TIME_WAIT` indicates high-frequency connection opening/closing (mitigated via pooling).

---

## 6. Popular Interview Questions & High-Impact Answers

### Q1: Why does TCP require a 3-way handshake instead of a 2-way handshake?
* **Answer:** A 3-way handshake is mathematically required to **prevent delayed, stale connection requests** from establishing phantom sessions. In a 2-way handshake (where the connection is active immediately upon server SYN-ACK), if an old client SYN packet was delayed in the network, it could arrive at the server *after* the client already disconnected. The server would respond with an ACK and allocate resources for a connection that the client will ignore, causing memory leaks. The third ACK from the client proves that both sides are active and agree on sequence numbers.

### Q2: What is the purpose of the `TIME_WAIT` state during TCP teardown, and why is it usually 2×MSL (Maximum Segment Lifetime)?
* **Answer:** The client enters the `TIME_WAIT` state after sending its final ACK for two reasons:
  1. **Ensure Reliable Teardown:** If the final ACK is lost, the server will assume its FIN failed and retransmit it. The client must stay alive to receive this retransmitted FIN and return another ACK.
  2. **Prevent Port Pollution:** It allows any delayed, duplicate packets belonging to the active connection to naturally expire and drain from the network, preventing them from being misread as valid data in a future connection using the same IP and port.

### Q3: How does UDP-based QUIC implement reliability if UDP itself is unreliable?
* **Answer:** UDP is a barebones, connectionless transport layer with no concept of handshakes, sequence numbers, or retransmissions. However, QUIC implements **reliability, flow control, congestion control, and connection tracking entirely at the Application Layer**. QUIC manages sequence numbers, detects packet loss, and triggers retransmissions inside its own software protocol, bypassing the operating system kernel's TCP stack for faster performance and lower latency.

### Q4: Explain the "Nagle's Algorithm vs. Delayed ACK" deadlock. How do you resolve it?
* **Answer:**
  * **Nagle's Algorithm:** Designed to prevent small-packet storms by delaying transmission of outgoing data if there is outstanding, unacknowledged data. Small payloads are buffered and combined into a single MSS packet.
  * **Delayed ACK:** Designed to prevent ACK storms. Instead of acknowledging every packet, the receiver delays the ACK (often by 200ms) to see if it can piggyback the ACK on returning application data.
  * **The Deadlock:** If a client writes two small messages sequentially (e.g., a header and a body) and uses Nagle's, the first write goes out. Nagle's blocks the second write from going out until the first is ACKed. However, the receiver's Delayed ACK is waiting for more data before sending an ACK. Both sides block until the 200ms Delayed ACK timer expires, severely degrading throughput.
  * **Resolution:** Disable Nagle's algorithm by setting the **`TCP_NODELAY`** socket option on the client and server. This is mandatory for real-time, low-latency APIs and WebSockets.

### Q5: What is a "SYN Flood" attack, and how do "SYN Cookies" mitigate it at the OS kernel level?
* **Answer:**
  * **SYN Flood:** A Denial of Service (DoS) attack where an attacker sends a massive stream of SYN packets using spoofed source IPs. The server responds with SYN-ACK and allocates memory in its **SYN Queue (Incomplete Connection Backlog)** waiting for the final ACK. Since the spoofed IPs never respond, the queue quickly fills up, causing the server to reject legitimate incoming connections.
  * **SYN Cookies:** Mitigates this by eliminating the need to allocate memory for half-open connections. When the SYN queue is full, the server computes a cryptographically signed sequence number (the "cookie") as:
    $$\text{Seq} = H(\text{Client IP}, \text{Client Port}, \text{Server IP}, \text{Server Port}, \text{Secret}) + \text{MSS Index}$$
    The server sends this Seq in its SYN-ACK and deletes all local session records. If the client is legitimate, it responds with an ACK containing `Ack = Seq + 1`. The server reconstructs the hash and verifies the signature. If valid, it allocates the socket immediately.

### Q6: How does TCP Window Scaling work, and why is it necessary for Long Fat Networks (LFNs)?
* **Answer:**
  * **The Problem:** The default TCP header allocates only 16 bits for the Receive Window size, limiting the maximum window size to $2^{16} - 1 = 65,535 \text{ bytes}$ (64 KB). On high-bandwidth, high-latency networks (e.g., trans-oceanic 10 Gbps fiber with 100ms RTT), the **Bandwidth-Delay Product (BDP)** is:
    $$\text{BDP} = 10 \text{ Gbps} \times 0.1 \text{ s} = 1 \text{ Gb} \approx 125 \text{ MB}$$
    If the window is capped at 64 KB, the sender can only transmit 64 KB of data per RTT, leaving 99.9% of the cable capacity completely idle.
  * **Window Scaling:** Solves this by adding a **Window Scale (WS)** option in the SYN packets during the handshake. WS specifies a shift count (from 0 to 14) that multiplies the 16-bit window value by up to $2^{14}$. This dynamically expands the maximum possible window size from 64 KB to **1 GB**, allowing full throughput utilization over LFNs.

