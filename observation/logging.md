# Structured Logging & Aggregation

Comprehensive interview study guide covering logging practices, structured JSON logs, log levels, and centralized log aggregation architectures (ELK/PLG stacks).

---

## 1. Structured Logging (JSON vs. Plain Text)

Traditional logging outputs plain text:
`[2026-06-04 10:15:30] INFO: User 123 placed order 456 successfully.`
* **The Problem:** Plain text logs are extremely difficult to query, parse, or filter automatically at scale. To find all logs for user `123`, you must write complex regular expressions (`grep`).

**Structured Logging** outputs logs as a standardized, machine-readable structured format (typically **JSON**):
```json
{
  "timestamp": "2026-06-04T10:15:30.452Z",
  "level": "INFO",
  "message": "Order placed successfully",
  "user_id": 123,
  "order_id": 456,
  "service": "order-service",
  "environment": "production"
}
```
* **The Benefit:** Log aggregators can index JSON keys instantly. You can query your logs like a database: `service = "order-service" AND user_id = 123`.

---

## 2. Dynamic Log Levels

| Level | Purpose | Target Environment |
| :--- | :--- | :--- |
| **DEBUG** | Highly verbose diagnostic details, useful strictly during development. | Development / Staging |
| **INFO** | Routine, successful operational events (e.g., startup, completed task). | Production |
| **WARN** | Non-critical anomalies or unexpected paths (e.g., slow API response, retry). | Production |
| **ERROR** | Operation failed, but the application remains running (e.g., DB save failed). | Production (Triggers Alerts) |
| **FATAL** | Critical crash. The process cannot recover and must exit. | Production (Triggers Alerts) |

---

## 3. Centralized Log Aggregation Architectures

In microservices, checking local container logs via `docker logs` is impossible since logs are scattered across dozens of instances. We use centralized log shipping:

### 1. The ELK Stack (Elasticsearch, Logstash, Kibana)
* **Logstash/Filebeat:** Agents installed on nodes that scrape raw log files and forward them.
* **Elasticsearch:** A highly scalable search engine that indexes and stores logs.
* **Kibana:** The frontend visualization dashboard used to search and analyze logs.

### 2. The PLG Stack (Promtail, Loki, Grafana)
* **Loki** is a modern, lightweight alternative to Elasticsearch. Unlike Elasticsearch (which indexes the full text of every log, requiring massive memory/disk), Loki **only indexes metadata labels** (like `service="api"`), compressing the raw log text. This makes Loki 10-100× cheaper to operate.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: Why is structured JSON logging considered a non-negotiable requirement for production backend services?
* **Answer:** In production, systems process millions of requests. If an outage occurs, parsing unstructured plain text logs manually is too slow. Structured JSON logs turn text strings into indexed key-value fields. This allows search engines (like Elasticsearch or Loki) to filter, group, and query massive volumes of logs in milliseconds (e.g., showing all `level="ERROR"` logs grouped by `exception_class`). It also enables automated alerting systems to parse log keys and trigger alerts on specific patterns.

### Q2: What is "Log Sampling" and why is it used in high-traffic applications?
* **Answer:** In high-traffic systems (processing tens of thousands of requests per second), logging every single event (especially `DEBUG` or successful `INFO` events) can generate terabytes of data daily, leading to massive storage costs and network bandwidth congestion. **Log Sampling** is a technique where the logging agent only forwards a specific percentage of routine logs (e.g., 10% of successful HTTP 200 logs) to the central aggregator, while still forwarding 100% of critical errors and warnings, saving storage while retaining debugging signals.

### Q3: How do you prevent sensitive data (like passwords, keys, or SSNs) from being leaked into your production logs?
* **Answer:** Leaking PII (Personally Identifiable Information) or secrets in logs is a major security violation. We prevent this using **Log Masking and Sanitization Filters**:
  1. Standardize your logging libraries to intercept log payloads before they write to stdout.
  2. Implement global sanitization filters (using regex or structural JSON keys) that scan for keywords like `password`, `token`, `credit_card`, `ssn`.
  3. Replace matching values with masked strings (e.g., `{"password": "[MASKED]"}`).
  4. Train developers and enforce linting/static-analysis rules that block raw object serialization (like logging the entire `User` object).
