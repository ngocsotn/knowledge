# Metrics & Alerting Systems

Comprehensive interview study guide covering system metrics, Pull vs. Push collection models (Prometheus vs. StatsD), and the Four Golden Signals.

---

## 1. Metrics vs. Logs

* **Logs:** Focus on **specific individual events** in high detail (e.g., "User 123 clicked buy at 10:15"). Excellent for debugging specific bugs. High disk storage overhead.
* **Metrics:** Focus on **numerical, aggregate values over time** (e.g., "Average CPU usage over the last 5 minutes is 64%", "API error rate is 0.5%"). Excellent for real-time dashboards, capacity planning, and automated alerting. Extremely lightweight on storage.

---

## 2. Pull vs. Push Metrics Collection Models

To build a centralized metrics dashboard, you must collect data from ephemeral microservices:

```
      PULL MODEL (Prometheus)                      PUSH MODEL (StatsD)
┌─────────────┐     Scrapes (/metrics)       ┌─────────────┐   Pushes (UDP)
│ Prometheus  ├──────────────┐               │ Application ├─────────────┐
└─────────────┘              ▼               └─────────────┘             ▼
                       ┌─────────────┐                             ┌─────────────┐
                       │ Application │                             │ StatsD/Agent│
                       └─────────────┘                             └─────────────┘
```

### 1. Pull-Based Model (e.g., Prometheus)
* **Mechanism:** The application exposes a read-only HTTP endpoint (typically `/metrics`) displaying current metric states. The central metrics server periodically pings (scrapes) this endpoint to gather data.
* **Pros:** Decouples applications from metrics server locations; simpler to manage; if an application crashes, the server detects the scrape failure instantly (acting as an implicit health-check).

### 2. Push-Based Model (e.g., StatsD, InfluxDB)
* **Mechanism:** The application actively pushes metrics payloads to a collector agent over UDP.
* **Pros:** Highly suited for short-lived, ephemeral tasks (like AWS Lambda serverless functions) that terminate before a pull server can scrape them. UDP pushes are extremely fast and non-blocking.

---

## 3. The SRE Four Golden Signals

Google's Site Reliability Engineering (SRE) manual defines the **Four Golden Signals** as the critical metrics to monitor in any user-facing system:

1. **Latency:** The time it takes to service a request (e.g., average, 95th, and 99th percentile response durations).
2. **Traffic:** A measure of how much demand is being placed on your system (e.g., HTTP requests per second, active database connections).
3. **Errors:** The rate of requests that fail (e.g., HTTP 5xx error rate, database transaction timeouts).
4. **Saturation:** A measure of how "full" your system resources are, indicating resource bottlenecks (e.g., memory utilization, disk IO, thread pool exhaustion).

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: Why is monitoring the 99th percentile (p99) latency far more critical than monitoring average (mean) latency?
* **Answer:** Average latency is a deceptive metric that hides severe user experience issues. If 98 users experience a blazing-fast response of 10ms, but 2 users experience a frozen response of 5 seconds, the average latency is a comfortable ~110ms, indicating everything is healthy. However, **p99 latency (the threshold below which 99% of requests fall)** will accurately reflect the 5-second bottleneck. Monitoring percentiles (p50, p90, p99, p99.9) ensures you capture edge cases, database locks, or garbage collection pauses that impact your unluckiest users.

### Q2: What is the "Prometheus Alertmanager" and how do you prevent Alert Fatigue?
* **Answer:** **Alertmanager** is a component that handles alerts sent by client applications or Prometheus queries. It de-duplicates, groups, and routes them to appropriate receivers (like Slack, PagerDuty, or Email). To prevent **Alert Fatigue** (where engineers ignore notifications because they receive hundreds of spam alerts daily):
  1. Only alert on **actionable, symptom-based issues** (e.g., "p99 latency is >2s", not "CPU is 90%").
  2. Implement **Alert Grouping**: combine multiple related alerts (e.g., if a database node goes down, group 50 dependent service alerts into a single database alert).
  3. Tune thresholds based on historical percentiles.

### Q3: Why is UDP typically used to push metrics (like in StatsD) instead of TCP?
* **Answer:** UDP is connectionless and non-blocking. When an application pushes a metric (like incrementing a counter `requests_total+1`), it does not want to block execution or wait for a network handshake acknowledgment. Using **UDP ensures zero performance impact on the application's main thread**. If the StatsD collector is offline or the network is congested, the UDP packets are simply discarded silently. While some metrics are lost, the core application's latency remains unaffected.
