
# Job Priorities

Priority lets urgent jobs run before less important jobs when workers are constrained.

Example:

```text
priority 1: payment confirmation
priority 5: welcome email
priority 10: analytics export
```

```mermaid
flowchart TD
    Queue[Waiting jobs] --> High[High priority]
    Queue --> Normal[Normal priority]
    Queue --> Low[Low priority]
    High --> Worker[Worker]
    Normal --> Worker
    Low --> Worker
```

## Why Priorities Matter

Without priorities, large batch jobs can delay customer-facing work. Priorities protect business-critical latency.

## Priority Risks

High-priority jobs can starve low-priority jobs. Use aging, separate queues, quotas, or reserved worker capacity.

Example:

```text
80% workers: customer jobs
20% workers: reports and cleanup
```

Separate queues often make capacity policy clearer than one queue with many priority values.

## When to Use

Use priorities when jobs share queue and resource pool but business urgency differs. Use separate queues when workloads need different owners, concurrency, retry, or scaling.

## Interview Questions

### Can priority guarantee SLA?

No. Priority only influences scheduling. Worker capacity, downstream latency, and retries still control completion time.

### How prevent starvation?

Reserve capacity, cap high-priority rate, or increase priority of waiting jobs over time.

### Example

Payment receipt job has high priority; nightly analytics export has low priority. During sale traffic, receipt jobs run first, while export continues with reserved low-priority workers.

## Pros, Cons, and Cost

**Pros:** protects urgent work, simple business ordering, better latency under contention.  
**Cons:** starvation, fairness complexity, misleading SLA expectations.  
**Cost:** reserved worker capacity and possible lower utilization for low-priority queues.

## Advanced Design

Priority is not isolation. If high-priority and low-priority jobs share Redis and downstream dependencies, high-priority work can still suffer. Use queue isolation when failure domains differ.

## Fairness Strategies

```mermaid
flowchart LR
    High[High priority] --> Reserved[Reserved urgent workers]
    Normal[Normal priority] --> Shared[Shared workers]
    Low[Low priority] --> Shared
    Reserved --> Payment[Payment work]
    Shared --> Reports[Reports and cleanup]
```

Use reserved workers for hard latency requirements. Use weighted scheduling or aging when every class must make progress.

## Priority Inversion

High-priority job may depend on a resource held by low-priority job, such as a database lock or tenant quota. Priority queue alone cannot solve dependency priority inversion. Keep critical transactions short and reserve downstream capacity.

## More Interview Questions

### Is lower numeric value always higher priority?

Library configuration decides interpretation. Document convention and test it. Do not assume priority semantics from another queue system.

### Should every job have priority?

No. Default priority is simpler. Add priority only when business latency differs and team can define fairness policy.