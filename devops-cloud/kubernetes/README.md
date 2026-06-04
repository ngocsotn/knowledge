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
