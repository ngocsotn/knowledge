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

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is a Single Point of Failure (SPOF), and how do you design systems to eliminate it?
* **Answer:** A Single Point of Failure is any individual component inside an architecture which, if it crashes, causes the entire system to stop functioning. For example, a single database server or a single load balancer. Eliminate SPOFs by introducing **redundancy** at every layer:
  1. **DNS Layer:** Use multiple nameservers and Geo-DNS routing.
  2. **Application Layer:** Deploy stateless APIs in auto-scaling groups behind multi-AZ load balancers.
  3. **Database Layer:** Maintain active-passive primary-secondary replicas with automated heartbeat failovers.

### Q2: What is the difference between an Active-Active and an Active-Passive cluster configuration?
* **Answer:** In an **Active-Active** configuration, all server nodes in the cluster simultaneously receive and process incoming client traffic. This provides full resource utilization and automatic failover capacity. In an **Active-Passive** configuration, only the primary ("Active") node handles user requests, while standby ("Passive") nodes keep their state synchronized in the background. If the Active node fails, a heartbeat observer automatically promotes the Passive node to the primary role to resume operations.

### Q3: What is "Graceful Degradation" and how does it improve system reliability?
* **Answer:** Graceful Degradation is the architectural pattern of designing services to continue running with limited, fallback features when upstream dependencies or database instances crash, prioritizing system availability over complete feature sets. For example, if a recommendation engine API goes offline, a streaming app degrades gracefully by displaying a hardcoded static list of popular movies instead of crashing the entire homepage layout.
