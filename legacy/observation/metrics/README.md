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

## 4. Why Global Percentiles Need Histogram Cumulative Buckets

In low-scale systems, calculating the 99th percentile (p99) latency of a single server is straightforward: you sort all request durations in memory over a window and pick the value at the 99% mark.

### The Aggregation Fallacy
In high-scale distributed systems, your application is split across 50 independent container instances.
* **The Trap:** Each container calculates and exposes its own local p99 latency (e.g., `p99 = 250ms`). 
* **The Error:** You cannot calculate the global cluster-wide p99 by taking the average, median, or sum of these 50 individual p99 metrics. This is mathematically invalid because percentiles are non-additive. If one server handles 10x more traffic than another, its p99 is heavily weighted, but simple mathematical averaging ignores this population skew.

### The Solution: Cumulative Histogram Buckets
To compute correct global percentiles across any dynamic slice of your cluster, systems use **Histograms** with static cumulative bucket boundaries:
1. Each application instance counts request durations into pre-configured, cumulative counter buckets (e.g., requests taking `le="0.1s"`, `le="0.2s"`, `le="0.5s"`, `le="1.0s"`, and `le="+Inf"`).
2. Instead of exposing pre-calculated percentiles, the containers expose these raw counters over the `/metrics` endpoint.
3. **Mathematical Estimation via PromQL:** The Prometheus server pulls these counter buckets from all 50 instances, sums them dynamically, and applies **Linear Interpolation** inside the bucket boundaries to estimate the global p99 percentile:

```promql
# Computes the global p99 response latency over a rolling 5-minute window
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))
```

* **Why it scales:** Summing raw bucket counters preserves population weight, allowing accurate calculation of arbitrary percentiles across any dynamically grouped cluster dimension (e.g., grouped by zone, region, or service version).

---

## 5. Alert Dampening & Flapping Prevention

A primary operational failure in observability is **Flapping Alerts**—where a metric hovers right around an alert threshold, triggering dozens of quick fire-and-resolved notifications that wake up engineers in the middle of the night.

### Dampening with Alerting Rules
To prevent notification noise, alerting systems enforce a temporal hold window using the `for` directive in declarative alerting files:

```yaml
groups:
  - name: API-Performance-Alerts
    rules:
      - alert: High-API-Latency-Global
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 2.0
        for: 5m # Metric must remain strictly breached for 5 consecutive minutes before firing
        labels:
          severity: critical
        annotations:
          summary: "Global p95 latency exceeds 2 seconds"
          description: "Active p95 latency is {{ $value }}s on cluster."
```

* **The Lifecycle:**
  1. **Active:** The Latency metric spikes above 2s. The alert status enters `Active (Pending)`. No notification is sent yet.
  2. **Transient Spike Cleared:** If latency drops below 2s at minute 3, the timer resets. The alert returns to normal (no alert was ever fired).
  3. **Alert Fired:** If latency remains $>2\text{s}$ for 5 continuous minutes, the alert transitions to `Firing`, sending a single actionable page to the SRE on-call rotation.

---

## 6. Popular Interview Questions & High-Impact Answers

### Q1: Why is monitoring the 99th percentile (p99) latency far more critical than monitoring average (mean) latency?
* **Answer:** Average latency is a deceptive metric that hides severe user experience issues. If 98 users experience a blazing-fast response of 10ms, but 2 users experience a frozen response of 5 seconds, the average latency is a comfortable ~110ms, indicating everything is healthy. However, **p99 latency (the threshold below which 99% of requests fall)** will accurately reflect the 5-second bottleneck. Monitoring percentiles (p50, p90, p99, p99.9) ensures you capture edge cases, database locks, or garbage collection pauses that impact your unluckiest users.

### Q2: What is the "Prometheus Alertmanager" and how do you prevent Alert Fatigue?
* **Answer:** **Alertmanager** is a component that handles alerts sent by client applications or Prometheus queries. It de-duplicates, groups, and routes them to appropriate receivers (like Slack, PagerDuty, or Email). To prevent **Alert Fatigue** (where engineers ignore notifications because they receive hundreds of spam alerts daily):
  1. Only alert on **actionable, symptom-based issues** (e.g., "p99 latency is >2s", not "CPU is 90%").
  2. Implement **Alert Grouping**: combine multiple related alerts (e.g., if a database node goes down, group 50 dependent service alerts into a single database alert).
  3. Tune thresholds based on historical percentiles.

### Q3: Why is UDP typically used to push metrics (like in StatsD) instead of TCP?
* **Answer:** UDP is connectionless and non-blocking. When an application pushes a metric (like incrementing a counter `requests_total+1`), it does not want to block execution or wait for a network handshake acknowledgment. Using **UDP ensures zero performance impact on the application's main thread**. If the StatsD collector is offline or the network is congested, the UDP packets are simply discarded silently. While some metrics are lost, the core application's latency remains unaffected.

### Q4: Why is it mathematically incorrect to calculate a global cluster-wide 99th percentile (p99) latency by taking the average of all local servers' local p99 values?
* **Answer:** Percentiles are non-additive. Local p99 latency figures discard the raw population size (total sample count) that contributed to that percentile value. If Server A handles 10,000 requests with a `p99 = 100ms`, and Server B handles exactly 1 request with a `p99 = 2,000ms`, the mathematical average of their p99s is `1,050ms`. However, the true global p99 across all 10,001 requests is extremely close to `100ms`, because the single slow request on Server B resides deep in the global tail well beyond the 99% boundary. Averaging percentiles leads to extreme mathematical errors under skewed loads.

### Q5: How does Prometheus calculate quantiles on the server-side from pre-bucketed counters, and what is its main limitation?
* **Answer:** Prometheus uses the `histogram_quantile` function which reads cumulative bucket counter values (tracked as time-series metrics appended with `_bucket`).
  1. It first aggregates the rate of requests per bucket across the targeted instances.
  2. It locates the target bucket that contains the desired quantile index (e.g., locating the boundary bucket for the 95th percentile).
  3. It performs **Linear Interpolation** within that bucket's lower and upper boundaries to estimate the exact latency value.
  - **The Limitation:** Because the calculation relies on linear interpolation inside static bucket ranges, the precision of the resulting quantile is heavily dependent on how the bucket boundaries are configured. If the bucket range is too wide (e.g., a single bucket spanning from `0.1s` to `10.0s`), the estimated quantile value will be highly inaccurate. Buckets must be designed to cluster tightly around the expected SLO thresholds.
