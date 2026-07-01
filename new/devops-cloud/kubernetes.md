# Kubernetes Orchestration

Comprehensive interview study guide covering Kubernetes architecture, control plane, worker nodes, pods, services, deployments, and routing.

---

## 1. Meaning of Kubernetes

**Kubernetes (K8s)** is an open-source container orchestration platform designed to automate the deployment, scaling, healing, and management of containerized applications across clusters of physical or virtual machines.

---

## 2. Hierarchical Cluster Architecture

A Kubernetes cluster is divided into two primary logical sections:

```
                  ┌─────────────────────────────────┐
                  │          CONTROL PLANE          │
                  │   API Server   ◄──►   etcd      │
                  │   Scheduler    ◄──►   Manager   │
                  └────────┬────────────────────────┘
                           │ (Communicates via TLS)
            ┌──────────────┼──────────────┐
            ▼ (Node 1)     ▼ (Node 2)     ▼ (Node 3)
      ┌───────────┐  ┌───────────┐  ┌───────────┐
      │  Kubelet  │  │  Kubelet  │  │  Kubelet  │
      │ Kube-Proxy│  │ Kube-Proxy│  │ Kube-Proxy│
      │   Pods    │  │   Pods    │  │   Pods    │
      └───────────┘  └───────────┘  └───────────┘
```

### 1. Control Plane (The Brain)
* **API Server (`kube-apiserver`):** The central entry point. All administrative commands, CLI tools (kubectl), and worker communications talk to this via JSON over TLS.
* **etcd:** A highly consistent, distributed key-value database that stores the authoritative **state and configuration** of the entire cluster.
* **Scheduler (`kube-scheduler`):** Watches for newly created pods with no assigned node, and selects an optimal worker node for them to run on based on resource availability.
* **Controller Manager (`kube-controller-manager`):** Runs continuous background controller loops that compare the actual state of the cluster with the desired state (e.g., ensuring 3 healthy pod replicas are running).

### 2. Worker Nodes (The Muscle)
* **Kubelet:** A tiny agent running on every node. It ensures that containers defined in pod specifications are running and completely healthy.
* **Kube-Proxy (`kube-proxy`):** Implements Kubernetes network services on each node, managing IP routing tables and performing simple load balancing across pods.
* **Container Runtime:** The physical software that runs containers (e.g., `containerd` or Docker).

---

## 3. Core Objects & Network Routing

* **Pod:** The **smallest deployable unit** in Kubernetes. It represents a single running process and wraps one or more closely coupled containers (sharing a network namespace and volume storage).
* **Deployment:** A declarative resource that manages the lifecycle of Pods, supporting rolling updates, horizontal scaling, and rollbacks.
* **Service:** An abstraction that defines a logical set of Pods and a policy to access them. Types of services:
  * **ClusterIP (Default):** Exposes the service on a cluster-internal IP, making it accessible *only* within the cluster.
  * **NodePort:** Exposes the service on a static port on each worker node's IP, allowing external access.
  * **LoadBalancer:** Provisions a physical external load balancer in your cloud provider (e.g., AWS, GCP) and routes traffic directly to NodePort/ClusterIP.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between a Pod and a Container?
* **Answer:** A **Container** is an isolated process running on host namespaces, packaged from a single Docker image. A **Pod** is a Kubernetes-specific abstraction. It is a **sandbox wrapping one or more containers**. Containers inside a single Pod share the exact same network namespace (meaning they communicate via `localhost`), the same IP address, port spaces, and shared storage volumes. This is highly useful for "sidecar" helper containers (e.g., a logger container scraping logs from a main server container).

### Q2: How does Kubernetes handle high availability and self-healing when a worker node crashes?
* **Answer:**
  1. The **Kubelet** on worker nodes periodically reports health metrics to the **API Server**.
  2. If a node crashes, the API Server stops receiving heartbeats.
  3. The **Controller Manager** detects that the active pod replicas dropped below the desired count defined in the Deployment spec.
  4. The **Scheduler** instantly locates other healthy, active worker nodes with sufficient resource capacity and schedules replacement Pods on them, restoring the desired state without human intervention.

### Q3: How do Rolling Updates work in a Kubernetes Deployment, and how does it prevent downtime?
* **Answer:** A **Rolling Update** replaces the old version of an application with the new version progressively:
  1. The deployment creates a new ReplicaSet alongside the old one.
  2. It launches a small percentage of new pods (e.g., 25%).
  3. It waits for the new pods to pass their **Readiness Probes** (proving they are fully ready to accept user traffic).
  4. Once healthy, it directs the Service proxy to route traffic to the new pods and tears down a corresponding number of old pods. This process repeats incrementally until 100% of traffic is migrated, ensuring zero downtime.

---

## 5. Architectural Deep Dive: Kubernetes Internals

For Senior/Staff infrastructure positions, you must understand the low-level communication loops, consensus consistency, and network packet paths inside the cluster.

### 1. The Pod Lifecycle, Container Hooks, & Health Probes

```
                       Pod Created (Pending State)
                                    │
                                    ▼
                        Run Init Containers (Seq)
                                    │
                                    ▼
                      Main Container Startup (PostStart Hook)
                                    │
                                    ├──────────────────────────┐
                                    ▼                          ▼
                          Startup Probe (Blocks others)    PreStop Hook (On delete)
                                    │                          │
                                    ▼                          ▼
                          Liveness / Readiness Probe       SIGTERM -> SIGKILL
```

#### Detailed Lifecycle Phases
* **Pending:** Pod manifest accepted by API server, but container images downloading or scheduling not completed.
* **Running:** Pod bound to a worker node, all containers initialized, and at least one container actively running or restarting.
* **Succeeded:** All containers in the Pod terminated successfully with exit code `0` (typically run-to-completion Jobs).
* **Failed:** At least one container terminated with a non-zero exit code.
* **CrashLoopBackOff:** A container keeps crashing, forcing the Kubelet to wait with exponential backoff delay ($10s, 20s, 40s...$ capped at $5m$) before restarting it.

#### Hooks & Graceful Termination
1. **`PostStart` Hook:** Executes immediately after container creation. Runs asynchronously with the container's entrypoint (no order guarantee).
2. **`PreStop` Hook:** Blocks the termination request. Triggered before the `SIGTERM` signal is dispatched. Ideal for flushing files, closing DB pools, or telling discovery endpoints to stop routing traffic to the pod.
3. **Termination Sequence:** 
   $$\text{PreStop Hook} \ \longrightarrow \ \text{SIGTERM} \ \longrightarrow \ \text{Wait terminationGracePeriodSeconds (default 30s)} \ \longrightarrow \ \text{SIGKILL}$$

#### Triple Probes Hierarchy
* **Startup Probe:** Checks if the application inside the container has fully initialized. **Blocks Liveness and Readiness probes** from running until it passes. Prevents slow-starting containers from being prematurely killed by Liveness probes.
* **Liveness Probe:** Checks if the container needs a hard reboot. If it fails, Kubelet kills the container and initiates restart policies.
* **Readiness Probe:** Checks if the container can accept HTTP/TCP traffic. If it fails, the Endpoint Controller **removes the Pod IP from the matching Service's routing table**, stopping all incoming user traffic from reaching it.

---

### 2. ETCD Consistency, Quorum, & Raft Architecture

ETCD is a strongly consistent, distributed key-value storage engine using the **Raft consensus algorithm**.

#### Quorum Formulation
ETCD requires a **strict majority quorum** of active members to commit writes and elect leaders. This prevents split-brain partition corruptions:

$$\text{Quorum} = \lfloor N/2 \rfloor + 1$$

| Cluster Size ($N$) | Max Allowed Failures ($F$) | Quorum Needed | Why Odd Sizes are Mandatory |
|:---:|:---:|:---:|---|
| **3** | **1** | **2** | If split ($2$ and $1$), the majority group ($2$) still retains quorum and accepts writes. |
| **4** | **1** | **3** | No extra fault tolerance over 3 nodes, but requires more network overhead. |
| **5** | **2** | **3** | Can survive 2 node outages. |

#### Write-Path Consensus Execution Flow
1. **Client Proposal:** A write request is sent to `kube-apiserver`, which writes it to the ETCD Leader node.
2. **AppendEntries RPC:** The Leader appends the entry to its local WAL (Write-Ahead Log) and broadcasts the entry to all Follower nodes.
3. **Follower Verification:** Followers append the entry to their WALs and send an acknowledgment (ACK) back to the Leader.
4. **Leader Commit:** Once the Leader receives ACKs from a **quorum** of nodes, it commits the entry to its state machine and replies success to the API Server.
5. **Follower Apply:** The Leader notifies Followers to commit the entry to their local state machines on the next heartbeat.

---

### 3. Container Network Interface (CNI) & IP Packet Routing

Kubernetes mandates that **every Pod gets a unique, routable IP address within the cluster**, eliminating host port conflicts.

```
┌───────────────────────────────── Worker Node ──────────────────────────────────┐
│                                                                                │
│   Pod A (Network Namespace)               Pod B (Network Namespace)            │
│   ┌───────────────────────┐               ┌───────────────────────┐            │
│   │        eth0           │               │        eth0           │            │
│   └──────────┬────────────┘               └──────────┬────────────┘            │
│              │ (veth pair)                           │ (veth pair)             │
│              ▼                                       ▼                         │
│         ┌────┴────┐                             ┌────┴────┐                    │
│         │ veth_A  │                             │ veth_B  │                    │
│         └────┬────┘                             └────┬────┘                    │
│              │                                       │                         │
│              ▼                                       ▼                         │
│   ┌──────────┴───────────────────────────────────────┴──────────┐              │
│   │                       cni0 (Bridge)                         │              │
│   └──────────────────────────┬──────────────────────────────────┘              │
│                              │                                                 │
│                              ▼                                                 │
│   ┌──────────────────────────┴──────────────────────────────────┐              │
│   │                        eth0 (Physical)                      │              │
│   └─────────────────────────────────────────────────────────────┘              │
└────────────────────────────────────────────────────────────────────────────────┘
```

#### Node-Local Packet Path (Pod A to Pod B on same node)
1. **Pod Virtual Interface:** Pod A dispatches an IP packet to its local virtual interface `eth0`.
2. **Veth Pair Conduit:** The packet travels through a **veth (Virtual Ethernet) pair** connecting the Pod network namespace to the host network namespace (e.g., `veth_A`).
3. **Host Bridge Routing:** The packet exits the host side of the veth pair and lands on the host network bridge (e.g., `cni0` or `docker0`).
4. **Direct Bridge Forwarding:** The bridge reads the destination MAC/IP, determines the destination Pod B is attached to the same bridge, and forwards the packet through Pod B's veth conduit (`veth_B`) into Pod B's namespace.

#### Inter-Node Packet Path (Pod A to Pod C on Node 2)
1. **Default Gateway Forwarding:** If Pod C resides on a different node, the local bridge `cni0` realizes the subnet does not match, forwarding the packet to the node's main physical gateway interface `eth0`.
2. **Overlay Encapsulation (VXLAN/Geneve) - *e.g., Flannel/Calico Overlay*:**
   * The local CNI daemon encapsulates the raw pod-to-pod IP packet inside an outer **UDP packet** (destination port `4789`).
   * The outer IP header sets the Source Node IP as the source and Target Node IP as the destination.
   * This allows standard physical switches to route the packet across local subnets without knowing about Pod IP spaces.
3. **Direct Routing (BGP) - *e.g., Calico Peer-to-Peer Routing*:**
   * No UDP encapsulation overhead. 
   * Nodes run a BGP (Border Gateway Protocol) client, advertising their local Pod subnets to all other nodes (acting as virtual routers).
   * Packets travel raw and unencapsulated across the host physical network, resulting in higher throughput.

---

### 4. The Controller Reconciliation Loop Mechanics

The Controller Manager operates on a continuous, level-triggered **Reconciliation Loop** designed to drive actual state towards the desired state.

```
 ┌───────────────┐
 │ Desired State │ (defined in etcd manifest)
 └───────┬───────┘
         │
         ▼
  ┌─────────────┐       No change       ┌─────────────┐
  │  Compare()  ├──────────────────────►│    Sleep    │
  └──────┬──────┘                       └─────────────┘
         │ Difference detected
         ▼
  ┌─────────────┐
  │  Reconcile()│ (creates/deletes pods, adjusts network routing)
  └─────────────┘
```

#### Low-Level Architecture (The Informer Pattern)
To prevent overloading the API server with polling queries, controllers use **Informers**:
1. **Reflector:** Initiates a `List` query to fetch initial resources and then establishes a persistent HTTP connection to `Watch` for real-time state change events (add, update, delete).
2. **DeltaFIFO Queue:** Emitted watch events are pushed to a FIFO buffer queue.
3. **Local Store (Indexer / Cache):** Events are consumed from DeltaFIFO, updating a fast, local in-memory cache of cluster resources. This ensures controllers query the local memory store instead of making heavy API server trips.
4. **WorkQueue:** Changed resources are pushed to a workqueue where multiple worker threads dequeue them and execute the custom controller reconciliation function (`Reconcile(req)`), aligning cluster state.

