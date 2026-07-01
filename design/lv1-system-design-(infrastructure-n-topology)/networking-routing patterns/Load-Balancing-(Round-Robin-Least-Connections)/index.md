# Load Balancing: L4 vs. L7 & Algorithms

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

## 2. Load Balancing Algorithms & Decision Matrix

To distribute traffic, load balancers utilize various routing algorithms. Choosing the correct algorithm depends directly on the system's statefulness, server hardware uniformity, and transaction workloads.

### A. Algorithm Comparison Matrix

| Algorithm | How it Works | Ideal Use Case | Pitfalls / Drawbacks |
| :--- | :--- | :--- | :--- |
| **Round Robin** | Routes requests sequentially down the list of servers. | Stateless apps with **homogeneous** (identical) server specs and uniform request processing times. | Severely unbalances cluster load if some servers have lower specs or if some requests take 10x longer to process. |
| **Weighted Round Robin** | Assigns static weights (e.g., Server A = 3, Server B = 1) based on capacity, routing a proportional fraction of traffic. | Stateless apps running on **heterogeneous** hardware specs (e.g., mixing a 16-core and 8-core machine). | Does not account for real-time traffic spikes or dynamic load fluctuations on the servers. |
| **Least Connections** | Routes new incoming requests to the server with the lowest number of active TCP/HTTP connections. | Stateful/long-lived transactions, or workflows with **highly variable request processing times** (e.g., database reports). | Higher memory/CPU overhead on the load balancer to track active connection states in real-time. |
| **Weighted Least Connections** | Combines real-time active connection tracking with static server capacity weights. | Long-lived transactions distributed across a cluster of **unequal hardware specs**. | High tracking overhead; vulnerable to "thundering herd" if a recovered node suddenly has zero connections. |
| **IP Hash / Cookie Sticky** | Hashes the client's IP address or reads a custom session cookie to map them permanently to a specific server. | Legacy **stateful applications** where session data is cached on local server filesystem/RAM. | Fails to distribute load evenly if many users hide behind a single corporate NAT/Proxy (skewed hash distribution). |
| **Consistent Hashing** | Maps both servers and data keys to a 360-degree hash ring. | **Distributed caching layers** (Redis, Memcached) or partitioned/sharded databases. | Complex to implement; requires virtual nodes (vnodes) to prevent hotspots and cascade crashes. |
| **Least Response Time** | Routes requests to the server with the lowest active connections and the fastest response time (latency). | **Geographically dispersed nodes** or cloud environments with highly dynamic network delays. | Can cause traffic to fluctuate wildly ("flapping load") between nodes as response times change. |

---

## 3. Consistent Hashing & Virtual Nodes

Traditional hash mapping (`hash(key) % num_servers`) is highly fragile: if you add a new server node or a server crashes, the modulo value changes for almost all keys, resulting in a **cache stampede** or loss of session mapping across the entire system.

* **Consistent Hashing** maps both database/cache nodes and data keys to a logical **360-degree circle (the Hash Ring)**.
* **How it works:** A key is hashed, placed on the ring, and routed to the next closest database node in a clockwise direction.
* **Why it scales:** Adding or removing a server node only impacts keys located between it and its immediate neighbor on the ring (affecting only $1 / N$ of keys), preserving cluster cache stability.

### The Hotspot / Non-Uniform Distribution Problem
In a simple consistent hashing ring with few physical servers, nodes are mapped to arbitrary positions. Because hashing is probabilistic, large intervals (gaps) can form between servers, causing some nodes to inherit a disproportionate amount of the ring's namespace. This leads to **load hotspots** where one server gets overloaded while others sit idle.

```
       Simple Ring (Hotspot)                    Ring with Virtual Nodes (Uniform)
             [Server A]                                    [A-vn1]
            /          \                                  /       \
     [Key] *            \                            [B-vn2]     [C-vn1]
          /              \                           /             \
      O(1) Gap            O(10) Gap             [Key]*             [A-vn2]
        /                  \                         \             /
    [Server B] -------- [Server C]                   [C-vn2]     [B-vn1]
                                                          \       /
                                                           [A-vn3]
```

### The Solution: Virtual Nodes (Vnodes)
To enforce uniform distribution, consistent hashing implementations map physical servers to multiple **Virtual Nodes (vnodes)** across the ring:
1. Instead of hashing a node's IP address directly (`hash("10.0.0.1")`), the system hashes the node ID appended with a sequence index ($M$ times):
   $$hash("10.0.0.1#1"), \quad hash("10.0.0.1#2"), \quad \dots, \quad hash("10.0.0.1#M")$$
2. Each physical server now controls hundreds of logical points (typically $M = 100 \text{ to } 200$ vnodes).
3. **Statistical Balancing:** With vnodes, the standard deviation of load across nodes drops drastically (usually keeping key distribution deviation $< 5\%$). If a node goes offline, its load is evenly dissipated to *all* other remaining nodes across the ring, preventing any single backup node from being overwhelmed.

### High-Performance Go Implementation
The lookup phase must be highly optimized. To resolve which node owns a key, consistent hashing uses binary search (bisect/`sort.Search`) over a sorted slice of vnode hashes to yield $O(\log (\text{num\_physical\_nodes} \times M))$ lookup latency.

```go
package main

import (
	"crypto/sha256"
	"encoding/binary"
	"fmt"
	"sort"
	"strconv"
)

type HashRing struct {
	vnodes      int              // Number of virtual nodes per physical node
	ring        []uint32         // Sorted list of vnode hashes
	nodeMap     map[uint32]string // Maps vnode hash to physical node name
	nodes       map[string]bool  // Set of active physical nodes
}

func NewHashRing(vnodes int) *HashRing {
	return &HashRing{
		vnodes:  vnodes,
		nodeMap: make(map[uint32]string),
		nodes:   make(map[string]bool),
	}
}

func (h *HashRing) getHash(key string) uint32 {
	hasher := sha256.New()
	hasher.Write([]byte(key))
	sum := hasher.Sum(nil)
	return binary.BigEndian.Uint32(sum[0:4]) // Extract 32-bit integer
}

func (h *HashRing) AddNode(node string) {
	if h.nodes[node] {
		return
	}
	h.nodes[node] = true
	for i := 0; i < h.vnodes; i++ {
		vnodeKey := node + "#" + strconv.Itoa(i)
		hash := h.getHash(vnodeKey)
		h.ring = append(h.ring, hash)
		h.nodeMap[hash] = node
	}
	sort.Slice(h.ring, func(i, j int) bool { return h.ring[i] < h.ring[j] })
}

func (h *HashRing) RemoveNode(node string) {
	if !h.nodes[node] {
		return
	}
	delete(h.nodes, node)
	for i := 0; i < h.vnodes; i++ {
		vnodeKey := node + "#" + strconv.Itoa(i)
		hash := h.getHash(vnodeKey)
		h.removeHashFromRing(hash)
		delete(h.nodeMap, hash)
	}
}

func (h *HashRing) removeHashFromRing(hash uint32) {
	idx := sort.Search(len(h.ring), func(i int) bool { return h.ring[i] >= hash })
	if idx < len(h.ring) && h.ring[idx] == hash {
		h.ring = append(h.ring[:idx], h.ring[idx+1:]...)
	}
}

func (h *HashRing) Get(key string) string {
	if len(h.ring) == 0 {
		return ""
	}
	hash := h.getHash(key)
	// Binary search to find closest vnode hash greater than key's hash
	idx := sort.Search(len(h.ring), func(i int) bool { return h.ring[i] >= hash })
	
	// If hash is greater than all vnode hashes in the ring, wrap around to index 0
	if idx == len(h.ring) {
		idx = 0
	}
	return h.nodeMap[h.ring[idx]]
}

func main() {
	hr := NewHashRing(100)
	hr.AddNode("server-1")
	hr.AddNode("server-2")
	hr.AddNode("server-3")

	fmt.Println("user_8241 maps to:", hr.Get("user_8241")) // O(log M) lookup
}
```

---

## 4. Advanced Health Checking: Active & Passive

To maintain maximum availability, load balancers continuously run dual active and passive health-checking routines to detect and isolate malfunctioning nodes.

### 1. Active Health Checking (In-Band Probing)
The load balancer actively initiates synthetic requests to individual backends to verify status.
* **Flapping Dampening:** A common failure mode is "flapping" where a node constantly transitions between healthy and unhealthy, causing upstream router turbulence.
  - **Mitigation:** Force state changes to be hysteretic (sticky). Configured with threshold counts:
    - `rise = 3`: A node must succeed 3 times consecutively before being marked healthy and returned to the cluster pool.
    - `fall = 2`: A node must fail 2 times consecutively to be declared dead.
* **HTTP Target Customization:** Instead of simple TCP socket connection tests (which pass even if the HTTP server is stuck in an infinite loop), active probes send HTTP GET requests to `/healthz` or `/live`. The endpoint must verify database availability, file system read-write status, and memory pressures before returning `200 OK`.

### 2. Passive Health Checking (Out-of-Band / Circuit Breaker)
Instead of synthetic requests, the load balancer inspects live customer transactions flowing through the gateway.
* **TCP & HTTP Observation:** If any connection attempts to a backend trigger a TCP handshake reset, connection timeouts, or return $X$ consecutive `502 Bad Gateway`/`503 Service Unavailable` statuses, the node is immediately quarantined.
* **Graceful Backoff:** If triggered, the node is isolated for a configurable period (e.g., `fail_timeout = 30s`). When the quarantine expires, the load balancer slowly leaks a small percentage of live traffic to the node to probe if it has recovered, serving as an automated circuit breaker.

---

## 5. Nginx Dynamic Upstream & Service Discovery

In traditional static deployments, Nginx relies on compile-time configuration or hardcoded IPs within the `upstream` block:

```nginx
upstream application_pool {
    server 10.0.0.10:8080;
    server 10.0.0.11:8080;
}
```
**The Problem:** In dynamic cloud environments (Kubernetes, AWS Auto Scaling, Docker Compose), containers are constantly spawned and destroyed, changing their IPs. Running `nginx -s reload` on every change causes a slow, CPU-expensive re-evaluation of the configuration, can leak file descriptors, and terminates transient connection pools under high loads.

### Dynamic Resolution Techniques

#### 1. DNS Resolution with Runtime Resolvers (Short TTL)
Nginx can be configured to resolve upstream domains dynamically at runtime instead of caching them once during startup.

```nginx
resolver 10.0.0.2 valid=5s; # DNS resolver address and cache TTL

server {
    location /api/ {
        # Using a variable forces Nginx to query DNS at runtime
        set $upstream_backend "app-service.internal";
        proxy_pass http://$upstream_backend:8080;
    }
}
```

#### 2. Service Discovery Sidecars & API Gateways
For production-scale dynamic environments, systems deploy **Control-Plane Sidecars** (e.g., HashiCorp Consul, Envoy, Nginx Plus with `/api` endpoint):
* **Consul Template:** Continually monitors Consul's service directory and writes dynamic changes to local disk, running optimized hot-reloads of HAProxy/Nginx configurations only when changes occur.
* **Envoy Discovery Services (xDS APIs):** Envoy acts as a data plane. It connects to a centralized control plane (like Istio or Consul) and subscribes to real-time endpoint listings. Upstreams are updated instantly in memory using zero-downtime, lock-free routing table swaps without ever needing process restarts.

---

## 6. Load Balancer vs. API Gateway (Microservice Topology)

In modern microservice architectures, **Load Balancers** and **API Gateways** operate in tandem at different tiers of the network topology. A common design is to deploy them sequentially to manage ingress traffic:

```
                      [ Client Requests (Public Internet) ]
                                        │
                                        ▼ (HTTPS / DNS Geo-Routing)
                      ┌──────────────────────────────────┐
                      │    Edge Load Balancer (L4/L7)    │ ◄── Handles SSL termination, DDoS defense,
                      └────────────────┬─────────────────┘     and routes to the API Gateway Cluster
                                       │
                                       ▼ (HTTP / Private Virtual Network)
                      ┌──────────────────────────────────┐
                      │     API Gateway Cluster (L7)     │ ◄── Handles JWT auth, rate limiting, logging,
                      └────────────────┬─────────────────┘     and dynamic microservices orchestration
                                       │
            ┌──────────────────────────┼──────────────────────────┐
            ▼                          ▼                          ▼
     ┌─────────────┐            ┌─────────────┐            ┌─────────────┐
     │Auth Service │            │Order Service│            │Cart Service │ ◄── Microservices (Private IP / Pods)
     └─────────────┘            └─────────────┘            └─────────────┘
```

### A. Architectural Roles & Differences

| Dimension | Load Balancer (LB) | API Gateway |
| :--- | :--- | :--- |
| **Primary Focus** | **Traffic distribution & high availability**. Ensures no individual server is overwhelmed by raw request volume. | **Application orchestration & api management**. Coordinates business features and protects microservices. |
| **OSI Layer** | Operates at **Layer 4** (Transport) or simple **Layer 7** (Application). | Operates deeply at **Layer 7** (Application). |
| **Routing Logic** | Uses generic network routing (IP address, port, HTTP path prefixes `/static/*` or `/api/*`). | Uses complex, dynamic application rules (URL parameters, custom headers, tenant IDs, API versions). |
| **Security Role** | Handles SSL/TLS termination, basic IP blacklisting, and high-volume DDoS shielding. | Handles fine-grained application security: **JWT signature verification**, OAuth2 flows, RBAC authorization, API-key billing tracking. |
| **Cross-Cutting Concerns**| None. Focuses strictly on packet forwarding. | **Offloads common business concerns**: request/response body transformations, response caching, metrics aggregation (Prometheus/Jaeger), circuit breaking. |
| **Dynamic Service Discovery**| Static IP targets or basic DNS lookups. | Deep integration with cloud-native service registries (Consul, Kubernetes DNS, Eureka) to route to dynamic ephemeral IP addresses. |

---

## 7. Popular Interview Questions & High-Impact Answers

### Q1: What is SSL Termination, and why is it handled at the Load Balancer level?
* **Answer:** SSL/TLS Termination is the process of decrypting HTTPS requests at the load balancer level and transmitting them as plain HTTP to the internal backend servers. This offloads the high CPU cost of cryptographic handshakes and decryption from the individual application servers, letting them focus strictly on business logic. It also simplifies certificate management, requiring updates on only the load balancers rather than every backend instance.

### Q2: Why does Traditional Modulo Hashing fail when scaling a cache cluster horizontally, and how does Consistent Hashing fix it?
* **Answer:** In traditional hashing (`hash(key) % N`), changing the number of servers ($N$) by adding or removing a node recalculates the destination index for nearly 100% of keys. In a caching cluster, this instantly invalidates the entire cache, overwhelming the underlying databases. **Consistent Hashing** maps servers and keys to a shared hash ring. Adding a server node only triggers a re-route for a small fraction of keys ($1/N$), keeping the remaining $N-1$ cache mappings completely unaffected.

### Q3: How does a Load Balancer detect if a backend server has crashed?
* **Answer:** Load balancers use continuous **Active Health Checks**. They are configured to ping a specific endpoint (e.g., `GET /health`) on each backend server at regular intervals (e.g., every 5 seconds). If a server fails to respond with a `2xx` or `3xx` status, or times out multiple times consecutively, the load balancer marks the node as unhealthy and immediately stops routing new traffic to it until it passes health checks again.

### Q4: What is the "flapping node" problem in load balancing health checks, and how is it resolved?
* **Answer:** A flapping node is a server that is unstable, repeatedly transitioning between healthy and unhealthy states in rapid succession (e.g., due to memory thrashing or intermittent network drops). If the load balancer immediately updates its routing tables on every single health check success/failure, this causes extreme routing turbulence and drops transient connection pools. This is resolved using **Flapping Dampening / Hysteresis** which configures threshold bounds: the node must fail $Y$ consecutive times to be taken offline, and must pass $X$ consecutive times (often larger than $Y$, e.g., `rise = 3`, `fall = 2`) before being returned to active service.

### Q5: How do Virtual Nodes (Vnodes) prevent "cascade overload" when a node fails in Consistent Hashing?
* **Answer:** Without virtual nodes, physical servers are mapped directly to a single location on the hash ring. If a physical node fails, 100% of its keys shift clockwise to its immediate neighbor on the ring. This sudden 100% load injection can overwhelm that neighbor, causing it to crash as well, creating a destructive **cascade failure (domino effect)** across the cluster. **Virtual Nodes (Vnodes)** solve this by distributing hundreds of virtual markers for each physical node uniformly across the ring. When a physical server fails, its virtual nodes disappear, and its key load is dissipated proportionally and evenly across *all* remaining active nodes, ensuring no single server takes the brunt of the failure.

### Q6: When would you choose Least Connections over Round Robin as a load balancing algorithm?
* **Answer:** Round Robin assumes that all requests impose an identical computational load and that all servers have identical hardware specifications. Least Connections should be preferred when **request processing times are highly variable** (e.g., mixing simple HTTP GETs with heavy, multi-second SQL reports) or when connections are **long-lived** (e.g., WebSockets, gRPC streams, or SSH tunnels). In these scenarios, Least Connections prevents "load hotspots" where one server becomes backlogged with multiple slow requests while other servers sit idle.

### Q7: What is the difference between a Load Balancer and an API Gateway, and how do they work together in a microservice architecture?
* **Answer:** A **Load Balancer** focuses on high-throughput packet forwarding, network routing (Layer 4/7), SSL termination, and horizontal high availability. An **API Gateway** is a rich Layer 7 orchestrator that coordinates application concerns such as JWT verification, rate limiting, request/response transformations, and service aggregation. In a microservice topology, they work together: the edge Load Balancer sits facing the public internet to distribute incoming requests evenly to a cluster of API Gateways (ensuring the gateway layer itself scales), and the API Gateways then dynamically route, secure, and compose those requests to the private internal microservices.

## Interview Questions & Answers

### Q1: What is the statistical performance difference between Consistent Hashing and Round Robin?
- **Answer:** Round Robin distributes requests equally but is stateless, meaning subsequent user requests land on random servers, defeating local session/in-memory caches. Consistent Hashing routes requests based on a hash of a key (e.g., `user_id`), ensuring a specific user always routes to the identical backend node. Using **Consistent Hashing with Virtual Nodes** keeps server load deviation within a tight <5% boundary, maximizing local caching efficiency while maintaining balanced resource load.
