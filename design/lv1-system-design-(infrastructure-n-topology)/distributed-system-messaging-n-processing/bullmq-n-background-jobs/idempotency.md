
# Idempotency

Idempotency means repeating same operation produces same business result, not duplicate side effect.

## Why Jobs Need It

Worker can:

1. Complete external side effect.
2. Crash before marking job complete.
3. Receive same job again.

At-least-once execution makes duplicates normal.

```mermaid
sequenceDiagram
    participant Q as Queue
    participant W as Worker
    participant P as Provider
    Q->>W: Deliver job
    W->>P: Charge with idempotency key
    P-->>W: Success
    W--xQ: Crash before acknowledge
    Q->>W: Redeliver same job
    W->>P: Repeat same key
    P-->>W: Return original result
```

## Techniques

### Unique Database Constraint

Store operation key:

```text
unique(payment_attempt_id)
```

Second execution sees existing result.

### Provider Idempotency Key

Send stable key to payment, email, or shipping provider when supported.

### Processed-Event Table

Record `(consumer, event_id)` after successful transaction. Duplicate checks existing row.

### Deterministic Output

Writing same object version or using upsert can make repeat safe.

## Idempotency Boundary

Queue job ID alone may not protect side effect. Job can be recreated with new queue ID. Business operation ID must remain stable.

## Pros, Cons, and Cost

**Pros:** safe retries, crash recovery, easier replay, protection from duplicate charges.<br>
**Cons:** durable key storage, retention policy, race-condition design, provider limitations.<br>
**Cost:** idempotency table storage, unique indexes, provider features, and cleanup jobs.

## Advanced Design

Reserve idempotency key and outcome atomically. Concurrent requests with same key must wait for or return one result, not execute side effect twice. Retain keys at least as long as late retries can arrive.

## Atomic State Machine

```mermaid
stateDiagram-v2
    [*] --> Absent
    Absent --> Processing: reserve key
    Processing --> Succeeded: side effect succeeds
    Processing --> Failed: permanent failure
    Processing --> Processing: retry or recovery
    Succeeded --> Succeeded: duplicate request returns saved result
    Failed --> Failed: duplicate request returns saved failure
```

Store status, request hash, result reference, and timestamps. Reject same key with different payload unless API contract explicitly allows it.

## Idempotency Versus Concurrency

Two workers can receive same business operation concurrently. Unique constraint or atomic `INSERT ... ON CONFLICT` must decide one owner. A check-then-insert sequence without transaction can race.

## Interview Questions and Answers


#### Is idempotency same as exactly once?

No. Execution may happen multiple times; business effect becomes one logical result.

#### Where store idempotency key?

Durable database or provider system, not worker memory.


#### How long retain idempotency keys?

At least longer than maximum client retry, queue retry, provider retry, and reconciliation window. Delete only after late duplicate cannot arrive.

#### What if first attempt remains `processing` forever?

Use lease expiration and recovery worker. Do not let permanent `processing` block operation indefinitely.

#### Why hash request payload?

Same idempotency key with different payload is usually client bug or abuse. Hash detects conflict and prevents ambiguous result.


#### What should an idempotency record contain?

Store the key, operation type, normalized request or resource identity, state, response or result reference, timestamps, and an expiry policy. The unique constraint must cover the business scope, such as `(tenant_id, operation, external_id)`, not merely a random request ID.

#### How do you distinguish a duplicate from a conflicting request?

A same-key request with the same normalized intent can return the original result. A same key with different material parameters should fail as a conflict, because silently applying the first request hides a client bug or replay attack.

#### Does idempotency remove the need for locking?

No. Idempotency prevents repeated final effects; locking or compare-and-set controls concurrent attempts. Use both when two workers could race before either has written the completion record.

### Examples and Diagrams

#### Example

`capture-payment` carries `payment_attempt_id`. Worker retries after timeout. Payment service returns prior capture result instead of charging twice.

#### Practical example: one shipment label

```mermaid
flowchart LR
    J[Create-label job] --> I[(Idempotency table)]
    I -->|new| P[Carrier API with key]
    P --> I
    I -->|existing success| R[Return saved label]
    I -->|processing timeout| Q[Reconcile status]
```

The worker never creates a second label solely because its first carrier request timed out. It queries the carrier using the same idempotency key or reconciles the shipment state.
