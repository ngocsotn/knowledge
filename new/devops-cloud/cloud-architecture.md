# Cloud Architecture & Serverless Computing

Comprehensive, production-grade interview study guide covering Cloud Computing Service Models (IaaS, PaaS, SaaS, FaaS), Cloud Network Topologies (VPC, Subnets, NAT Gateways, Security Groups vs. NACLs), and Serverless Architecture (AWS Lambda, execution lifecycles, and Cold Start optimization).

---

## 1. Cloud Computing Service Models

Cloud infrastructure isolates administrative and physical hardware control across four primary abstractions:

```
[ Traditional On-Premise ]   [ IaaS (Infrastructure) ]   [ PaaS (Platform) ]   [ FaaS / Serverless ]
┌────────────────────────┐   ┌───────────────────────┐   ┌─────────────────┐   ┌───────────────────┐
│ Applications           │   │ Applications          │   │ Applications    │   │ Function Code     │
├────────────────────────┤   ├───────────────────────┤   ├─────────────────┤   └───────────────────┘
│ Data / OS / VM         │   │ Data / OS / VM        │   └─────────────────┘   (Infrastructure, OS,
├────────────────────────┤   └───────────────────────┘   (Managed Runtime,     Runtimes, & scaling
│ Hardware / Networking  │   (Physical servers and      OS, & virtualization  completely handled
└────────────────────────┘    network managed by cloud)  completely managed)   by Cloud Provider)
```

1. **IaaS (Infrastructure as a Service):**
   * **What you get:** Raw virtualized compute (VMs), storage, and networking layers.
   * **Your responsibility:** Installing and patching the Operating System, managing runtimes, setting up firewalls, and scaling instances.
   * **Examples:** AWS EC2, GCP Compute Engine, Azure VMs.
2. **PaaS (Platform as a Service):**
   * **What you get:** A managed application runtime environment. The cloud provider manages the physical server, OS, database, and virtualization layers.
   * **Your responsibility:** Writing and deploying application code.
   * **Examples:** AWS Elastic Beanstalk, Heroku, Google App Engine.
3. **SaaS (Software as a Service):**
   * **What you get:** A complete, fully functional software application accessed over the web.
   * **Examples:** Slack, Salesforce, Microsoft 365.
4. **FaaS (Function as a Service / Serverless Compute):**
   * **What you get:** On-demand execution of individual, ephemeral code functions triggered by events. Runtimes, scaling, OS, and physical infrastructure are 100% managed by the provider.
   * **Examples:** AWS Lambda, Google Cloud Functions, Azure Functions.

---

## 2. Cloud Network Architecture: Virtual Private Clouds (VPC)

A **Virtual Private Cloud (VPC)** is a logically isolated virtual network mapped inside a cloud provider's physical data center infrastructure.

```
                         Virtual Private Cloud (VPC)
  ┌────────────────────────────────────────────────────────────────────────┐
  │                                                                        │
  │   ┌──────────────────────────┐          ┌──────────────────────────┐   │
  │   │ Public Subnet            │          │ Private Subnet           │   │
  │   │ (Direct Route to IGW)    │          │ (No Route to IGW)        │   │
  │   │                          │          │                          │   │
  │   │ [Nginx Public Proxy]     │          │ [Internal API Server]    │   │
  │   │ [NAT Gateway (Static IP)]│──Routing─► [PostgreSQL DB]          │   │
  │   └──────────┬───────────────┘          └────────────┬─────────────┘   │
  │              │                                       │                 │
  └──────────────┼───────────────────────────────────────┼─────────────────┘
                 ▼                                       ▼
        Internet Gateway (IGW)                 NAT Gateway (Outbound Only)
```

### A. Subnets: Public vs. Private
* **Public Subnet:** A subnet configured with a route table that directs outbound traffic directly to an **Internet Gateway (IGW)**, allowing resources inside to bind to public IP addresses and accept incoming traffic from the public internet (e.g., Load Balancers, public-facing proxies).
* **Private Subnet:** A subnet with no direct routing path to the public internet. Resources have private IP addresses only, completely isolating them from internet scanning and external entry (e.g., Databases, internal business API containers).
* **NAT Gateway (Network Address Translation):** To allow resources inside a private subnet to securely fetch outbound updates (e.g., package downloads) without exposing them to incoming requests:
  1. Deploy a **NAT Gateway** inside the **Public Subnet** (with a static elastic IP).
  2. Route the private subnet's outbound internet traffic (`0.0.0.0/0`) through the NAT Gateway.
  3. The NAT Gateway intercepts requests, translates private IPs to its public static IP, fetches the target data, and forwards it back safely.

### B. Security Groups vs. Network ACLs (NACLs)

Cloud engineers enforce network defense in depth by coordinating stateful and stateless firewalls:

| Feature | Security Group (SG) | Network ACL (NACL) |
| :--- | :--- | :--- |
| **Operational Scope**| **Instance Level** (Applies to a specific VM or container network interface). | **Subnet Level** (Applies to all resources inside the entire subnet). |
| **Statefulness** | **Stateful**. If an inbound request is permitted, the outbound response is automatically allowed, ignoring outbound rules. | **Stateless**. Inbound and outbound traffic rules must be explicitly and independently declared. |
| **Rules Support** | Allows **Allow Rules** only. Everything else is denied by default. | Supports both **Allow and Deny Rules** (evaluated sequentially based on rule numbers). |
| **Evaluation Order** | All rules are compiled and evaluated as a single policy (order does not matter). | Evaluated sequentially in strict ascending numerical order (e.g., Rule 100 before Rule 200). |

---

## 3. Serverless Compute: AWS Lambda Lifecycles & Cold Starts

**AWS Lambda** is an event-driven serverless runtime. Instead of keeping servers running continuously, Lambda spawns lightweight virtualization containers on-demand to execute code when triggered.

### A. The MicroVM Hypervisor: AWS Firecracker
AWS Lambda runs on **AWS Firecracker**, an open-source virtualization technology that uses Linux's Kernel-based Virtual Machine (KVM) to spawn highly secure **MicroVMs** in milliseconds. Firecracker bypasses legacy BIOS/PCI device emulation, keeping memory footprint minuscule (~5MB per MicroVM) and enabling extreme startup density.

### B. The Lambda Execution Lifecycle & Cold Starts

```
                       The Cold Start Phase (Slow: >500ms)
┌───────────────────────┬────────────────────────┬────────────────────────┐
│ 1. Download Code      │ 2. Start MicroVM       │ 3. Init Runtime / SDKs │
└───────────────────────┴────────────────────────┴────────────────────────┘
                                      │
                                      ▼
                       The Hot Start Phase (Fast: <10ms)
                        ┌────────────────────────┐
                        │ 4. Execute Handler Code│
                        └────────────────────────┘
```

When a trigger event occurs, AWS Lambda must coordinate execution:
1. **Cold Start (Full Initialization):** If no idle MicroVM is active in the cluster pool, Lambda must perform a Cold Start:
   * Download the function's deployment package (ZIP or Container Image).
   * Initialize a new Firecracker MicroVM container.
   * Bootstrap the language runtime (JVM, Node.js Engine, Python interpreter).
   * Execute the **Initialization Code** (code outside the actual `handler` function, e.g., setting up DB connections, compiling SDK configs).
2. **Hot Start (Warm Execution):** If a MicroVM has executed a request recently and remains in the active container pool, subsequent trigger events bypass the first 3 steps, immediately executing only the `handler` function.

---

## 4. Production Optimization: Mitigating Cold Starts

Cold start latency can degrade API response SLA (sometimes adding 1 to 5 seconds of latency). Senior engineers mitigate cold starts using five strategic patterns:

### A. Keep Deployment Packages Lightweight
* Minimize your deployment package size. Delete unneeded node_modules, strip debug logs, and avoid heavy third-party framework dependencies. Smaller packages download and unzip significantly faster.

### B. Optimize Initialization Code (Lazy Loading)
Global code (written outside the handler) runs during the cold start. Do not initialize heavy libraries or construct expensive database connection pools globally if they are only needed in specific execution branches. Construct connections inside the handler lazily, or optimize the initialization blocks.

### C. Choose the Right Runtime Language
* **Lightweight Runtimes (Fastest):** Go, Rust, Node.js, Python.
* **Heavy Runtimes (Slowest):** Java, C# (.NET).
* *Under the hood:* Go compiles to a native bare-metal binary requiring zero virtual engine setup. Java requires launching a heavy JVM (Java Virtual Machine) and compiling bytecode, which can easily spike cold starts past 3 seconds.

### D. Provisioned Concurrency
If your API cannot tolerate any cold start latency, configure **Provisioned Concurrency**:
* AWS pre-initializes a requested number of warm Firecracker MicroVMs, keeping them completely pre-warmed and ready to execute the handler instantly.
* *Trade-off:* Defeats raw serverless on-demand pricing. You are charged a continuous hourly rate for the pre-warmed capacity, behaves similarly to managed container scaling.

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between Security Groups and Network ACLs (NACLs) in AWS network design?
* **Answer:** **Security Groups** operate at the instance level (virtual network interfaces) and are **stateful**: once an inbound port is permitted, the outbound response is automatically allowed. They only support "Allow" rules. **Network ACLs (NACLs)** operate at the subnet boundary level and are **stateless**: you must explicitly configure both inbound and outbound rules separately. NACLs support both "Allow" and "Deny" rules, and are evaluated sequentially in numerical order.

### Q2: What is an AWS Lambda "Cold Start," and how do you optimize an application to minimize its latency?
* **Answer:** A Cold Start occurs when a Lambda function is triggered but there are no idle container instances (MicroVMs) available in the pool. AWS must download the function code, spawn a new MicroVM (using Firecracker), initialize the runtime (e.g., Node.js or Python), and execute global initialization code before running the handler. Optimize cold starts by:
  1. Selecting fast, native compiled runtimes (like Go or Rust) over heavy virtual machine runtimes (like Java).
  2. Keeping deployment packages small by stripping unneeded dependencies.
  3. Caching and lazily initializing database connections or SDK clients inside the handler rather than running expensive global imports.
  4. Utilizing **Provisioned Concurrency** to pre-warm containers for latency-critical paths.

### Q3: Why does allocating more Memory (RAM) to an AWS Lambda function reduce its execution time, even if the code is not memory-intensive?
* **Answer:** In AWS Lambda's pricing model, **CPU and network allocation scale proportionally with Memory**. You cannot explicitly configure the number of CPU cores or clock speeds. If you configure a Lambda with **128MB** of RAM, it receives a tiny fraction of a CPU core. If you allocate **1.5GB** of RAM, it receives a full CPU core, and at **10GB**, it receives up to 6 CPU cores. For CPU-bound tasks (like image manipulation or cryptographic operations), increasing the memory allocation drastically increases CPU cycles, making the execution finish faster and often reducing the total execution cost.
