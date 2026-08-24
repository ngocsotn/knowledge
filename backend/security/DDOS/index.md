# Distributed Denial of Service (DDoS)

DDoS is a malicious attempt to disrupt the normal traffic of a targeted server, service, or network by overwhelming the target or its surrounding infrastructure with a flood of Internet traffic.

---

## 1. Types of DDoS Attacks

DDoS attacks are categorized by the layer of the OSI model they target:

```
┌───────────────────────────────────────┐
│ Layer 7: Application (HTTP Floods)    │ ◄── Overwhelms web server processing
├───────────────────────────────────────┤
│ Layer 4: Transport (SYN Floods)       │ ◄── Overwhelms TCP connection tables
├───────────────────────────────────────┤
│ Layer 3: Network (UDP Floods, ICMP)   │ ◄── Overwhelms total network bandwidth
└───────────────────────────────────────┘
```

### 1. Volumetric (Network Layer - Layer 3/4)
* **Goal:** Exhaust the network bandwidth of the target.
* **Examples:** UDP Floods, DNS Amplification (exploiting open DNS resolvers to bounce massive payloads to the target).
* **Mitigation:** Requires high upstream network capacity and routing scrubbing services (e.g., Anycast IP routing).

### 2. Protocol Attacks (Transport Layer - Layer 4)
* **Goal:** Exhaust server connection state tables or intermediate network equipment (firewalls, load balancers).
* **Examples:**
  * **SYN Flood:** Exploits the TCP three-way handshake. The attacker sends thousands of TCP `SYN` requests with spoofed source IPs but never responds to the server's `SYN-ACK`. The server leaves these connections half-open, exhausting its finite connection pool.

```
                    SYN FLOOD EXPLOIT & SYN COOKIE SHIELD
       
VULNERABLE BACKLOG ALLOCATION (Memory Allocation on SYN)
Client ───[SYN]───> Server (Allocates socket state in Backlog Queue memory)
Client <──[SYN-ACK]─ Server
[Attacker drops socket! Never sends final ACK] ───> Backlog Queue Fills up & Crashes!

STATELESS SYN COOKIE DEFENSE (No memory allocated initially)
Client ───[SYN]───> Server (Calculates cryptographically secure Sequence Number)
Client <──[SYN-ACK (with Cookie in Seq)]─ Server (Allocates ZERO memory!)
                                                      │
                                           ┌──────────┴──────────┐
                                           │  Does final ACK     │
                                           │  contain valid Seq? │
                                           └──────────┬──────────┘
                                                      │
                                           YES        │        NO
                                    ┌─────────────────┴─────────────────┐
                                    ▼                                   ▼
                      [Allocate socket memory & CONNECT]          [Silently discard]
```

* **Mitigation:** SYN Cookies (stateless connection tracking), aggressive connection timeouts.

### 3. Application Layer Attacks (Layer 7)
* **Goal:** Overwhelm server resources (CPU, RAM, DB pools) by mimicking legitimate client traffic.
* **Examples:**
  * **HTTP Flood:** Flooding CPU-heavy API endpoints (like search or login) with standard requests.
  * **Slowloris:** Initiating hundreds of HTTP connections and sending headers extremely slowly. The server must keep these threads open waiting for the requests to finish, exhausting worker pools.
* **Mitigation:** Web Application Firewalls (WAF), rate limiting, short keep-alive timeouts.

---

## 2. Mitigation Strategies & Architecture

1. **Leverage Anycast CDNs (Cloudflare, AWS CloudFront):**
   * Spreads the incoming attack volume globally across hundreds of edge locations instead of routing all traffic to a single backend origin server.
2. **Web Application Firewalls (WAF):**
   * Analyzes Layer 7 request patterns, automatically blocking known botnets, suspicious User-Agents, or geographic sources.
3. **Connection Management & Timeouts:**
   * Configure load balancers and reverse proxies (like Nginx) to enforce low connection timeout thresholds to drop slow-sending clients (defending against Slowloris).
4. **Implement Rate Limiting & Captchas:**
   * Intercept spike loads before they hit application runtimes or database engines.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: How does a SYN Flood exploit the TCP handshake, and how does "SYN Cookies" mitigate it?
* **Answer:** In a standard TCP handshake, the server allocates memory (connection state) upon receiving a `SYN` request, waiting for the final `ACK`. An attacker floods `SYN` packets from spoofed IPs and never completes the handshake, exhausting server memory. **SYN Cookies** mitigates this by making the handshake stateless: the server encodes connection details into the `SYN-ACK` sequence number itself and allocates *zero* memory. Only when a valid final `ACK` arrives (carrying the correct sequence number) does the server decode the cookie and allocate resources.

### Q2: What is an application-layer (Layer 7) HTTP flood, and why is it harder to detect than Layer 3/4 attacks?
* **Answer:** A Layer 7 HTTP flood targets application resources using valid HTTP GET/POST requests that appear to be normal user actions. Because these requests do not utilize spoofed IPs or malformed packets, and they complete standard handshakes, they blend in with legitimate traffic. Mitigating Layer 7 attacks requires behavior analysis, rate-limiting on unique client identifiers, and challenge-response prompts (like CAPTCHAs) to distinguish bots from human users.

### Q3: How do you protect a backend origin server when using a CDN like Cloudflare?
* **Answer:** To prevent attackers from bypassing Cloudflare and flooding the origin directly, you must:
  1. Restrict origin port access (80/443) using firewalls (e.g., AWS Security Groups) to **only accept traffic from Cloudflare's published IP ranges**.
  2. Implement Authenticated Origin Pulls (mTLS certificates exchanged between CDN and Origin).
