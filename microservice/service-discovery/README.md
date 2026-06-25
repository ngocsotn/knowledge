# Service Discovery in Microservices

Comprehensive interview study guide covering Service Discovery, Service Registries, Client-side vs. Server-side patterns, and registry consensus.

---

## 1. Meaning & The Scale Challenge

In a traditional architecture, server IPs are static. You can hardcode your database IP directly in your application configurations.

In a dynamically scaling microservice environment (e.g., Kubernetes or AWS ECS):
* Service instances are constantly being created, destroyed, rescheduled, or restarted on different physical nodes.
* IPs and ports are constantly changing, making hardcoded configurations impossible.
* **Service Discovery** provides a dynamic, automated lookup system that allows microservices to locate and communicate with each other over the network.

---

## 2. Service Discovery Architectures

A service discovery setup relies on a central database called the **Service Registry** (e.g., Consul, Eureka, Etcd), which tracks the active network locations of all running service instances.

```
Client-Side Service Discovery                 Server-Side Service Discovery
┌──────────┐     Lookup                       ┌──────────┐     Request
│ ServiceA ├──────────────┐                   │ ServiceA ├──────────────┐
└────┬─────┘              ▼                   └──────────┘              ▼
     │            ┌───────────────┐                             ┌───────────────┐
     │ Route      │ServiceRegistry│                             │ Load Balancer │
     ▼            └───────────────┘                             └───────┬───────┘
┌──────────┐                                                            │ Lookup & Route
│ ServiceB │                                                            ▼
└──────────┘                                                    ┌───────────────┐
                                                                │ ServiceB      │
                                                                └───────────────┘
```

### 1. Client-Side Service Discovery
* **Mechanism:**
  1. When Service A wants to call Service B, Service A queries the **Service Registry** directly to get the list of active IPs/ports for Service B.
  2. Service A caches this list and uses a local load balancing algorithm (e.g., Round Robin) to choose a target node.
  3. Service A executes the HTTP/gRPC request directly to the chosen Node.
* **Examples:** Netflix Eureka client, Spring Cloud LoadBalancer.

### 2. Server-Side Service Discovery
* **Mechanism:**
  1. Service A sends the request to a centralized **Load Balancer** or Proxy (e.g., Kubernetes Ingress/Service).
  2. The Load Balancer queries the **Service Registry** internally to locate Service B instances.
  3. The Load Balancer routes the request to an active Service B node on behalf of Service A.
* **Examples:** Kubernetes Services, AWS ALB.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between Client-side and Server-side Service Discovery, and what are their trade-offs?
* **Answer:** In **Client-side discovery**, the calling service queries the registry directly and performs load-balancing locally. This removes an extra network hop (improving latency) and avoids a single load-balancer bottleneck. However, it tightly couples the application code to the registry's SDK, requiring multi-language library maintenance. In **Server-side discovery**, the caller routes requests through a load balancer. This simplifies application code (it just calls a static DNS name), but introduces an extra network hop and adds a central infrastructure component that must scale and be managed separately.

### Q2: How does a Service Registry know when a microservice instance has crashed or gone offline?
* **Answer:** Service registries use **Heartbeating & Health Checking**:
  1. **Self-Registration:** Upon startup, a service instance registers its IP/port with the registry.
  2. **Active Heartbeats:** The service instance must send a periodic "keep-alive" ping (e.g., every 10 seconds) to the registry.
  3. **Liveness Threshold:** If the registry misses multiple consecutive pings from a specific instance, or if its active health-check probe fails, the registry automatically de-registers (evicts) that instance's IP from its directory, preventing other services from routing traffic to it.

### Q3: What databases are typically used as Service Registries, and why are standard SQL databases unsuitable?
* **Answer:** Standard SQL databases are optimized for relational transactions and complex indexes but scale poorly under massive write/read loads of ephemeral node metadata. Service Registries use highly specialized distributed key-value engines like **Consul**, **Etcd**, or **ZooKeeper**. These systems use consensus algorithms (like **Raft** or **Paxos**) to guarantee strict write consistency across the cluster, ensuring that when an IP changes, all nodes see the update instantly, avoiding routing errors.
