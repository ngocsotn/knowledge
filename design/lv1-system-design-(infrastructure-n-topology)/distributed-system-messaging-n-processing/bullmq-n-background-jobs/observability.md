
# BullMQ Observability

Queue systems fail gradually before they fail visibly. Monitoring queue age and worker health catches problems earlier than checking only failed count.

## Key Metrics

### Queue Metrics

- Waiting jobs.
- Active jobs.
- Delayed jobs.
- Completed jobs.
- Failed jobs.
- Oldest waiting age.
- Throughput.

### Worker Metrics

- Active worker count.
- Job duration.
- Success rate.
- Retry rate.
- Stalled jobs.
- Process CPU and memory.
- Graceful shutdown duration.

### Dependency Metrics

- Database latency.
- Provider response codes.
- Rate-limit responses.
- Connection pool usage.
- Redis latency and memory.

```mermaid
flowchart TD
    Queue[Queue metrics] --> Dashboard[Dashboard]
    Worker[Worker metrics] --> Dashboard
    Dependency[Dependency metrics] --> Dashboard
    Dashboard --> Alert[Alerts]
    Alert --> Runbook[Operator runbook]
```

## Important Alerts

Alert on:

- Oldest job age above SLA.
- Queue depth growing continuously.
- Consumer count unexpectedly zero.
- Stalled jobs increasing.
- Failed or DLQ rate spike.
- Redis memory near limit.
- Redis replication or persistence failure.

Queue depth alone is weak. Ten thousand fast jobs may be healthier than ten old jobs blocked by one dependency.

## Tracing

Propagate:

- Trace ID.
- Request ID.
- Job ID.
- Business ID.
- Parent job ID.

Trace:

```text
HTTP request -> enqueue -> worker claim -> database -> external provider
```

OpenTelemetry spans can separate enqueue latency, queue wait time, handler execution, and downstream calls. Keep `jobId`, queue name, and bounded job type as span attributes, but do not put secrets or unbounded payload fields into metrics labels.

## Pros, Cons, and Cost

**Pros:** faster diagnosis, measurable SLOs, safer scaling, evidence-based incident response.<br>
**Cons:** metric cardinality, log volume, trace overhead, alert fatigue.<br>
**Cost:** metrics storage, log ingestion, tracing, dashboards, and on-call maintenance.

## Advanced Design

Track end-to-end latency from job creation to completion, not only worker duration. Partition metrics by queue, job type, outcome, dependency, and tenant only where cardinality remains affordable.

## Incident Investigation

```mermaid
flowchart TD
    Alert[Queue age alert] --> QueueCheck[Check queue depth and oldest age]
    QueueCheck --> WorkerCheck[Check worker count and stalls]
    WorkerCheck --> DependencyCheck[Check DB and provider latency]
    DependencyCheck --> RedisCheck[Check Redis memory, latency, persistence]
    RedisCheck --> Action[Throttle, scale, fix, or replay]
```

Use correlation IDs to connect API logs, worker logs, database traces, and provider requests. Avoid logging full job payloads when they contain personal or secret data.

## Cardinality Control

Do not create metrics label for unrestricted user ID, job ID, or request ID. Put those values in logs or traces. Metrics should aggregate by queue, job type, status, and bounded tenant class.

## Interview Questions and Answers


#### Which metric detects user-visible delay?

Oldest waiting age or end-to-end completion latency. Queue depth does not show age.

#### How detect stalled workers?

Track heartbeats, active-job duration, worker process health, and stalled-job count.


#### Why monitor oldest job age?

It reflects user-visible delay better than queue count. A small queue can still contain one job older than SLA.

#### What should alert page versus ticket?

Page on imminent business SLO breach, zero workers, Redis failure, or payment backlog. Ticket slower analytics lag or routine failed jobs below threshold.

#### Why keep queue events separate from business events?

Queue events describe processing mechanics. Business events describe domain facts and may need durable consumers or replay.


#### Which dimensions belong in queue metrics?

Track queue depth, oldest job age, arrival and completion rates, retry count, failure rate, active workers, and processing latency. Break down by queue and bounded workload class; avoid labels such as raw user ID that create unbounded cardinality.

#### How do you connect a job to a user request?

Propagate a correlation ID and trace context in job metadata, but keep the payload free of secrets. Emit spans for enqueue, wait time, handler execution, and downstream calls so queue delay is visible separately from processing time.

#### What is a useful first incident query?

Compare arrival rate, completion rate, oldest age, retry causes, worker availability, Redis latency, and downstream error rate over the same time window. This separates producer bursts from worker or dependency regressions quickly.

### Examples and Diagrams

#### Dashboard Example

```text
email queue:
  oldest job: 8s
  waiting: 120
  active: 20
  failure rate: 0.2%
  provider 429: 0
```

If oldest age rises while active stays full, add capacity only after checking provider and database limits.

#### Example incident

Email queue grows. Worker count is healthy, but provider returns 429. Correct action is lower rate, backoff, and inspect provider quota, not blindly add workers.

#### SLO Example

```text
99% of email jobs complete within 60 seconds
99.9% of payment jobs complete or enter review within 30 seconds
failed-job rate below 0.5%
```

Measure from enqueue time to final outcome. Worker execution time alone misses waiting and retry delays.

#### Practical example: alerting on user-visible delay

```mermaid
flowchart LR
    Q[Queue] --> M[Metrics]
    W[Worker] --> M
    D[Dependencies] --> M
    M --> G[Dashboard]
    M --> A[Alert: oldest age]
```

Alert on oldest age and SLO burn, not only depth: ten large jobs may be more harmful than a thousand tiny jobs. Link the alert to a runbook with drain and replay steps.
