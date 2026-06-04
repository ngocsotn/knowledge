# Load Balancing in Distributed Systems

Comprehensive interview study guide covering Layer 4 vs. Layer 7 load balancers, routing algorithms, and consistent hashing.

---

## 1. Layer 4 (L4) vs. Layer 7 (L7) Load Balancing

Load balancers operate at different layers of the OSI model:

```
        ┌───────────────────────────────────────────────────────┐
        │ Layer 7: Application (NGINX, HAProxy, AWS ALB)        │ ◄── Inspects cookies, headers, HTTP paths
        ├───────────────────────────────────────────────────────┤
        │ Layer 4: Transport (AWS NLB, F5 Big-IP, HAProxy L4)   │ ◄── Inspects IP addresses & TCP/UDP ports
        └───────────────────────────────────────────────────────┘
```

### 1. Layer 4 (L4) Load Balancing (Transport Layer)
* **Mechanism:** Routes traffic based strictly on packet attributes in the transport layer, such as IP address and TCP/UDP port numbers. It has zero knowledge of the actual HTTP/JSON contents.
* **Pros:** Extremely fast and memory efficient since it doesn't need to decrypt TLS, parse HTTP headers, or read payloads.
* **Cons:** Cannot perform smart routing based on URL paths, cookies, or user attributes.

### 2. Layer 7 (L7) Load Balancing (Application Layer)
* **Mechanism:** Decrypts TLS (SSL Termination) and inspects the application-layer payload (HTTP methods, URL paths, headers, cookies, query parameters).
* **Pros:** Enables smart routing (e.g., routing `/api/v1/checkout` to the billing service and `/static/*` to a CDN), sticky sessions based on cookies, and targeted rate limiting.
* **Cons:** Higher CPU and latency overhead due to packet parsing and TLS decryption.

---

## 2. Load Balancing Algorithms

To distribute traffic, load balancers utilize various routing algorithms:

1. **Round Robin:**
   * *How it works:* Distributes requests sequentially down the list of servers.
   * *Best for:* Clusters where all servers have identical hardware specifications and jobs have equal workloads.
2. **Weighted Round Robin:**
   * *How it works:* Servers are assigned weight values based on capacity. More powerful servers receive a higher percentage of requests.
3. **Least Connections:**
   * *How it works:* Directs new requests to the server node with the lowest number of active active connections.
   * *Best for:* Long-lived transactions or operations with highly variable CPU workloads.
4. **IP Hash:**
   * *How it works:* Hashes the client's IP address and maps it to a server index.
   * *Best for:* Ensuring a specific user always hits the same server (sticky session) without storing session state.

---

## 3. Consistent Hashing

Traditional hash mapping (`hash(key) % num_servers`) is highly fragile: if you add a new server node or a server crashes, the modulo value changes for almost all keys, resulting in a **cache stampede** or loss of session mapping across the entire system.

* **Consistent Hashing** maps both database/cache nodes and data keys to a logical **360-degree circle (the Hash Ring)**.
* **How it works:** A key is hashed, placed on the ring, and routed to the next closest database node in a clockwise direction.
* **Why it scales:** Adding or removing a server node only impacts keys located between it and its immediate neighbor on the ring (affecting only $1 / N$ of keys), preserving cluster cache stability.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is SSL Termination, and why is it handled at the Load Balancer level?
* **Answer:** SSL/TLS Termination is the process of decrypting HTTPS requests at the load balancer level and transmitting them as plain HTTP to the internal backend servers. This offloads the high CPU cost of cryptographic handshakes and decryption from the individual application servers, letting them focus strictly on business logic. It also simplifies certificate management, requiring updates on only the load balancers rather than every backend instance.

### Q2: Why does Traditional Modulo Hashing fail when scaling a cache cluster horizontally, and how does Consistent Hashing fix it?
* **Answer:** In traditional hashing (`hash(key) % N`), changing the number of servers ($N$) by adding or removing a node recalculates the destination index for nearly 100% of keys. In a caching cluster, this instantly invalidates the entire cache, overwhelming the underlying databases. **Consistent Hashing** maps servers and keys to a shared hash ring. Adding a server node only triggers a re-route for a small fraction of keys ($1/N$), keeping the remaining $N-1$ cache mappings completely unaffected.

### Q3: How does a Load Balancer detect if a backend server has crashed?
* **Answer:** Load balancers use continuous **Active Health Checks**. They are configured to ping a specific endpoint (e.g., `GET /health`) on each backend server at regular intervals (e.g., every 5 seconds). If a server fails to respond with a `2xx` or `3xx` status, or times out multiple times consecutively, the load balancer marks the node as unhealthy and immediately stops routing new traffic to it until it passes health checks again.
