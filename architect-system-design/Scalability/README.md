# System Scalability, HA, and Reliability

Comprehensive interview study guide covering system scaling patterns, High Availability (HA), Fault Tolerance, and reliability metrics.

---

## 1. Vertical vs. Horizontal Scaling (The Scalability Pivot)

* **Vertical Scaling (Scale-up):**
  * *Mechanism:* Adding more CPU, RAM, or faster NVMe storage to a single server node.
  * *When to use:* Early-stage prototypes, databases requiring strict sub-millisecond local latency, or single-threaded workflows.
  * *Limits:* Hardware ceiling limits (cannot grow indefinitely), expensive cost curves, and introduces a Single Point of Failure (SPOF).
* **Horizontal Scaling (Scale-out):**
  * *Mechanism:* Spreading user load across multiple cheap, independent application/database server nodes.
  * *When to use:* Modern distributed microservices, web apps experiencing rapid growth, stateless APIs.
  * *Limits:* Adds massive operational, deployment, and networking complexity.

---

## 2. High Availability (HA) vs. Fault Tolerance

High Availability and Fault Tolerance are related but distinct engineering goals:

| Metric | High Availability (HA) | Fault Tolerance |
| :--- | :--- | :--- |
| **Goal** | Minimize system downtime. | Guarantee **zero** service interruption or data loss. |
| **Acceptable Interruption** | Minor transition blip allowed during failover (e.g., active session re-auth). | Absolute transparency; no user detects a node failover. |
| **Redundancy Pattern** | Active-Passive / Active-Active clustering. | Hardware mirroring (duplicate hardware running in lock-step). |
| **Cost & Complexity** | Standard / Moderate. | Extremely High (requires real-time state replication and custom hardware). |

### Designing HA Systems (The Five 9s Goal)
HA is measured in uptime percentages:
* **99.9% ("Three Nines"):** ~8.76 hours of downtime per year.
* **99.999% ("Five Nines"):** ~5.26 minutes of downtime per year. (The gold standard for critical services).

---

## 3. SLA, SLO, and SLI

In site reliability engineering (SRE), service quality is tracked using three boundaries:

1. **SLA (Service Level Agreement):**
   * *What it is:* The legally binding promise made to users, including financial penalties if missed.
   * *Example:* "If our uptime drops below 99.9% this month, we will refund 10% of your subscription cost."
2. **SLO (Service Level Objective):**
   * *What it is:* The internal target metric targetted by the team. Must be stricter than the SLA.
   * *Example:* "Internal objective: 99.99% uptime."
3. **SLI (Service Level Indicator):**
   * *What it is:* The actual, real-time value measured by monitors.
   * *Example:* "Current measurement: 99.95% of incoming HTTP requests returned `< 500` status."

---

## 4. Distributed ID Generation (The Database Scale-Out Challenge)

When a database is sharded (horizontally partitioned) across multiple independent physical servers, traditional database-level auto-increment columns (e.g., `id SERIAL` or `AUTO_INCREMENT`) fail.
* **The Conflict:** If Shard Node A and Shard Node B both generate auto-incrementing IDs in parallel, they will both generate `1`, `2`, `3`, leading to catastrophic primary key collisions when merging or referencing records.

### A. Comparison of Distributed ID Strategies

| Pattern | Storage Footprint | Index Efficiency | Creation Overhead | Single Point of Failure (SPOF) |
| :--- | :---: | :---: | :---: | :---: |
| **UUIDv4** | 128-bit (36 chars) | **Extremely Poor** (Causes B-Tree fragmentation) | **Near Zero** (Decentralized local generation) | **None** |
| **Snowflake ID** | 64-bit integer | **Excellent** (Chronological sequence) | Low (Requires machine coordinate check) | **None** (Once machine ID is assigned) |
| **ULID** | 128-bit (26 chars) | **Good** (Lexicographically sortable) | Near Zero (Decentralized local generation) | **None** |
| **Ticket Server** | 64-bit integer | **Excellent** (Strict sequence) | **High** (Requires synchronous DB write) | **Yes** (Generates IDs from single DB) |

### B. Deep Dive: Key Generation Techniques

#### 1. UUIDv4 (Universally Unique Identifier)
Generates 128 bits of pseudo-random data.
* **Pros:** 100% decentralized. Any service instance can generate a UUIDv4 locally in microseconds without coordinating with other servers. Practically zero chance of collision.
* **The Indexing Disaster (B-Tree splits):** Because UUIDv4 is completely random, inserting new rows with UUID keys forces database engines to write to arbitrary pages of the B-Tree index structure. This triggers **Index Page Splits** (the database must allocate new storage blocks and relocate existing rows to maintain sorted index lists), severely degrading write throughput under high concurrency.

#### 2. Twitter Snowflake ID (Time-Ordered 64-Bit Integers)
Pioneered by Twitter, Snowflake generates 64-bit time-ordered integers.
```
 1 bit      41 bits (Timestamp)       10 bits (Worker ID)  12 bits (Sequence)
┌─────┬──────────────────────────────┬───────────────────┬──────────────────┐
│  0  │ Milliseconds since custom    │ Unique machine ID │ Increment count  │
│     │ epoch (allows ~69 years)     │ (assign via etcd) │ (cap at 4096/ms) │
└─────┴──────────────────────────────┴───────────────────┴──────────────────┘
```
* **Pros:** Fits inside a standard 64-bit integer (extremely compact). **Strictly chronological and sequential**, meaning new rows append perfectly to the end of database B-Tree index pages (zero page splits, highly performant writes).
* **Cons:** Requires a centralized directory service (like Consul or ZooKeeper) to dynamically allocate and coordinate unique `Worker ID` coordinates to servers during auto-scaling to prevent ID collisions.

#### 3. ULID (Universally Unique Lexicographically Sortable Identifier)
A 128-bit identifier consisting of a 48-bit timestamp followed by 80 bits of cryptographically secure random data, encoded in Crockford's Base32.
* **Pros:** Universally unique (like UUID) but **lexicographically sortable** (like Snowflake), protecting database indexes from fragmentation. Easily fits inside a standard UUID column natively supported by databases (like PostgreSQL `UUID`).
* **Cons:** Larger storage footprint than Snowflake (128-bit vs. 64-bit).

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: What is a Single Point of Failure (SPOF), and how do you design systems to eliminate it?
* **Answer:** A Single Point of Failure is any individual component inside an architecture which, if it crashes, causes the entire system to stop functioning. For example, a single database server or a single load balancer. Eliminate SPOFs by introducing **redundancy** at every layer:
  1. **DNS Layer:** Use multiple nameservers and Geo-DNS routing.
  2. **Application Layer:** Deploy stateless APIs in auto-scaling groups behind multi-AZ load balancers.
  3. **Database Layer:** Maintain active-passive primary-secondary replicas with automated heartbeat failovers.

### Q2: What is the difference between an Active-Active and an Active-Passive cluster configuration?
* **Answer:** In an **Active-Active** configuration, all server nodes in the cluster simultaneously receive and process incoming client traffic. This provides full resource utilization and automatic failover capacity. In an **Active-Passive** configuration, only the primary ("Active") node handles user requests, while standby ("Passive") nodes keep their state synchronized in the background. If the Active node fails, a heartbeat observer automatically promotes the Passive node to the primary role to resume operations.

### Q3: What is "Graceful Degradation" and how does it improve system reliability?
* **Answer:** Graceful Degradation is the architectural pattern of designing services to continue running with limited, fallback features when upstream dependencies or database instances crash, prioritizing system availability over complete feature sets. For example, if a recommendation engine API goes offline, a streaming app degrades gracefully by displaying a hardcoded static list of popular movies instead of crashing the entire homepage layout.

### Q4: Why is using raw UUIDv4 as a database primary key considered a performance anti-pattern in relational databases, and what are the alternatives?
* **Answer:** Relational databases (like MySQL InnoDB or PostgreSQL) store tables as clustered indexes sorted by the primary key using a B-Tree structure. Because UUIDv4 is completely random, newly inserted primary keys land on random B-Tree index pages. This forces the engine to perform frequent **B-Tree Page Splits** and random disk writes to reorganize the sorted index nodes in memory, degrading write performance under load. Alternatives include **Twitter Snowflake IDs** (64-bit time-ordered integers) or **ULIDs** (128-bit lexicographically sortable IDs), both of which are sequential and naturally append to the end of the B-Tree index, avoiding fragmentation.

### Q5: How does the bit structure of a Snowflake ID guarantee time-ordering and coordinate unique IDs across a distributed cluster?
* **Answer:** A Snowflake ID uses a 64-bit binary layout: the first bit is 0 (keeping the number positive), followed by **41 bits representing the millisecond timestamp** (providing chronological time-ordering), then **10 bits for the unique Machine/Worker ID** (guaranteeing that different physical servers never overlap namespaces), and finally **12 bits for a sequence number** (which increments atomically within that millisecond). Because the timestamp sits at the most significant bits of the integer, sorting Snowflake IDs naturally sorts them chronologically. Mutual exclusion is achieved locally on each node by using the machine ID and the local sequence lock.

