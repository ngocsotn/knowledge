# Saga Pattern & Distributed Transactions

In a single-instance monolithic database, achieving **Atomicity** and **Isolation** is trivial: the database engine utilizes its local Write-Ahead Log (WAL) and locking mechanisms (such as Two-Phase Locking / 2PL) to guarantee ACID properties. 

In a distributed microservice architecture:
* **Database-per-Service Pattern:** Every microservice owns its isolated datastore.
* **Network Partition Boundary:** A single logical operation (e.g., *Customer buys an item*) crosses multiple network boundaries, requiring state transitions across an Order DB, a Payment DB, and an Inventory DB.
* **The Coordination-Availability Tradeoff (CAP Theorem):** Attempting to enforce immediate global consistency across multiple databases requires blocking coordinator locks, degrading system throughput, increasing transaction latency, and leaving the system vulnerable to cascading timeouts if a single node degrades.

### Distributed Atomicity vs. Distributed Consensus
It is critical to distinguish between these two classes of distributed protocols:
* **Consensus (e.g., Raft, Paxos):** Solves the problem of agreeing on a **single value or sequence of values** across a replicated group of nodes, where only a *majority* (quorum) of healthy nodes is required to proceed ($N/2 + 1$).
* **Atomic Commitment (e.g., 2PC):** Solves the problem of ensuring that **all participants** in a transaction perform the same operation (all commit or all abort). Unlike consensus, 2PC requires **unanimous agreement (100% participation)**. If a single participant is down or unreachable, the transaction cannot commit.

---

## 2. Two-Phase Commit (2PC): Blocking Atomicity

The Two-Phase Commit (2PC) protocol is the classic standard for atomic commitment across distributed datastores.

### Execution Phases

```
          COORDINATOR                             PARTICIPANTS
      
               │                                      │
               │─── 1. PREPARE (Can you commit?) ────►│
               │                                      │ (Locks resources, writes WAL)
               │◄── 2. VOTE_COMMIT / VOTE_ABORT ──────│
               │
      ┌────────┴────────┐
      │ Decide Outcome  │ (Unanimous = COMMIT, else ABORT)
      └────────┬────────┘
               │
               │─── 3. GLOBAL_COMMIT / GLOBAL_ABORT ─►│
               │                                      │ (Applies changes, releases locks)
               │◄── 4. ACKNOWLEDGEMENT ───────────────│
               │                                      │
```

#### Phase 1: Prepare (Voting Phase)
1. The **Coordinator** generates a globally unique transaction ID ($T_x$) and writes a `START_2PC` record to its local WAL.
2. The Coordinator sends a `PREPARE` message to all **Participants** (Resource Managers).
3. Each Participant receives the query, executes all local preparation work (allocating resources, obtaining local locks, executing the database write in memory), and writes its local WAL.
4. Each Participant votes:
   * **VOTE_COMMIT:** If local preparation succeeded, writes `VOTED_COMMIT` to its WAL and returns a success response. **The participant is now locked and cannot unilaterally abort or commit.**
   * **VOTE_ABORT:** If local preparation failed (e.g., constraint violation or deadlock), writes `VOTED_ABORT` to its WAL and returns a failure response.

#### Phase 2: Commit (Decision Phase)
1. If **all** participants voted `VOTE_COMMIT`, the Coordinator writes a `COMMIT` record to its WAL and sends a `GLOBAL_COMMIT` message to all participants.
2. If **any** participant voted `VOTE_ABORT` (or if the Coordinator's timeout expired before receiving all votes), the Coordinator writes an `ABORT` record to its WAL and sends a `GLOBAL_ABORT` message to all participants.
3. Participants receive the decision, write `DECISION_COMMIT`/`DECISION_ABORT` to their local WAL, apply the changes (or release lock state), release all locked database resources, and reply with an `ACK`.
4. Once all ACKs are received, the Coordinator writes an `END_2PC` log to its WAL, completing the transaction.

### Critical Failure Modes & The Blocking Problem
The fundamental vulnerability of 2PC is that it is a **blocking protocol**:

1. **Coordinator Crash Mid-Decision:** If the Coordinator crashes after participants have voted `VOTE_COMMIT` but before broadcasting the `GLOBAL_COMMIT` / `GLOBAL_ABORT` decision:
   * Participants are left in a **blocked, uncertain state**.
   * A participant cannot unilaterally abort because another participant might have received a `GLOBAL_COMMIT`.
   * A participant cannot unilaterally commit because another participant might have voted `VOTE_ABORT`, triggering a `GLOBAL_ABORT`.
   * **Result:** All prepared resources (database rows, transactional locks) remain locked indefinitely until the coordinator is recovered and replays its WAL.
2. **High Latency & Scalability Bottleneck:** Because locks are held across multiple network round-trips (from the initial SQL execution in Phase 1 until the final global commit ACK in Phase 2), the transaction rate is limited by the slowest network link or database instance.

---

## 3. Three-Phase Commit (3PC): Non-Blocking but Impractical

The Three-Phase Commit (3PC) protocol was designed to eliminate the indefinite blocking vulnerability of 2PC by introducing an intermediate state and explicit timeout rules.

### Transition Mechanics & Timeout Rules

3PC splits the decision phase of 2PC into two sub-phases, adding a **Pre-Commit** step to ensure no node commits until everyone has agreed.

```
          COORDINATOR                            PARTICIPANTS
               │                                      │
               │─── 1. CAN-COMMIT? (Query to vote) ──►│
               │◄── 2. VOTE_COMMIT / VOTE_ABORT ──────│
               │
      ┌────────┴────────┐
      │ If all agree    │
      └────────┬────────┘
               │─── 3. PRE-COMMIT (Enter pre state) ─►│
               │◄── 4. ACK (Enter prepared) ──────────│
               │
      ┌────────┴────────┐
      │ If ACKs complete│
      └────────┬────────┘
               │─── 5. DO-COMMIT ────────────────────►│
               │◄── 6. ACK (Release locks) ───────────│
```

1. **Can-Commit Phase:** Identical to 2PC Prepare. Coordinator asks participants if they can commit. If all vote yes, the system transitions to Pre-Commit.
2. **Pre-Commit Phase:** The coordinator sends a `PRE-COMMIT` command. Participants write this state to their WAL and reply with an ACK. At this stage, participants are guaranteed that **all** other participants voted yes. No node is allowed to commit yet.
3. **Do-Commit Phase:** Once the coordinator collects ACKs from all participants, it broadcasts `DO-COMMIT`. Participants perform the physical commit and release locks.

### Why 3PC is Non-Blocking
* **Participant Timeout in Pre-Commit State:** If a participant is in the `PRE-COMMIT` state and loses contact with the coordinator, it waits for a timeout. Because the participant knows that *all* nodes successfully passed the `Can-Commit` phase (otherwise the coordinator would never have sent `PRE-COMMIT`), the participant can safely **timeout-commit** under the assumption that the coordinator's intended decision was to commit.
* **Participant Timeout in Can-Commit State:** If a participant is in the `CAN-COMMIT` state and times out waiting for the coordinator, it safely **timeout-aborts**.

### Why 3PC is Rarely Used: The Network Partition Vulnerability
3PC assumes a fail-stop model where nodes crash but the network remains reliable. In real-world cloud networks, **network partitions (split-brain)** occur:
* If a network partition splits the cluster, participants on one side of the partition may time out in the `PRE-COMMIT` state and execute a **Commit**.
* Simultaneously, participants on the other side of the partition may fail to receive the coordinator's commands during `Can-Commit` and execute an **Abort**.
* **Result:** Dual-state divergence (split-brain), violating absolute consistency. 

Because 3PC cannot guarantee atomicity under network partitions, high-throughput microservice architectures utilize **Saga Patterns (eventual consistency)** for application flows and **Consensus Protocols (Raft/Paxos)** for infrastructure-level replication.

---

## 4. Saga Topologies: Choreography vs. Orchestration

The **Saga Pattern** manages distributed business transactions through a sequence of local database transactions. Rather than holding global locks, each step commits its changes locally. If a downstream step fails, the system executes explicit **Compensating Transactions** to reverse committed changes in reverse order.

### Comparative Architectural Analysis

| Dimension | Choreography-Based Saga (Decentralized) | Orchestration-Based Saga (Centralized) |
| :--- | :--- | :--- |
| **Coordination** | Peer-to-peer. Services listen to events and trigger local actions. | Central controller (state machine) directs all steps. |
| **Coupling** | Loosely coupled. Services only depend on the message broker. | Tight logical coupling. Orchestrator must know details of all participant APIs. |
| **Complexity** | High cognitive overhead. Hard to trace end-to-end execution flow. | Centralized business logic. Highly readable and trackable. |
| **Failure Recovery** | Complex. Compensating events must be manually chain-routed. | Simple. Orchestrator explicitly invokes rollback steps in order. |
| **Performance** | Lower latency. Event-driven, parallelizable message routing. | Marginally higher latency due to centralized hop routing. |
| **Single Point of Failure** | None. Completely decentralized. | Yes (The Orchestrator). Requires redundant deployment & database persistence. |

---

## 5. Saga Semantic Anomalies & Isolation Deficiencies

Because Saga transactions commit locally at each step, they lack the **Isolation (I)** of traditional ACID database transactions. Changes made by early steps are visible to other concurrent transactions *before* the entire Saga is complete. This introduces three semantic anomalies:

### 1. Lost Updates
* **Scenario:** Saga $A$ updates a resource (e.g., decreases inventory from 10 to 9). Before Saga $A$ completes, Saga $B$ reads the inventory (9), decreases it (8), and commits. If Saga $A$ subsequently fails and executes a compensating transaction (restoring inventory to 10), Saga $B$'s update is overwritten and permanently lost.

### 2. Dirty Reads
* **Scenario:** Saga $A$ reserves a seat on a flight. Customer $C$ executes a search, reads that the seat is booked, and chooses a different flight. Saga $A$ subsequently fails and cancels the seat reservation. Customer $C$ read uncommitted, temporary state that was rolled back (Dirty Read).

### 3. Non-Repeatable Reads
* **Scenario:** A service reads a balance of \$100. A concurrent Saga modifies the balance to \$50 and commits. The first service reads the balance again and gets \$50.

### Production Countermeasures (Saga Design Patterns)
To mitigate isolation deficiencies, developers must implement the following design patterns:

* **Semantic Locks (Pessimistic State):** 
  Instead of committing a final state change immediately, a local transaction sets an intermediate status (e.g., `ORDER_PENDING`, `FUNDS_HELD`). Downstream services read this status and treat it as a lock, blocking modifications to that record until the Saga completes or is rolled back.
* **Commutative Updates:** 
  Design compensating transactions to be commutative (order-independent). For example, instead of setting a balance to a hard value (`SET balance = 100`), use relative increments/decrements (`ADD balance 50` or `SUBTRACT balance 50`). This prevents overwriting concurrent writes.
* **Pessimistic View Value Gating:**
  If a Saga fails and rolls back, any payment or resource release is subject to a pessimistic check. For example, if a user tries to withdraw refunded cash *before* the compensating rollback completes, the withdrawal is gated by verifying the status of the ongoing Saga.

---

## 6. The Transactional Outbox Pattern

When a microservice executes a local database transaction *and* publishes an event to a message broker (a **dual-write**), a network partition or process crash can result in inconsistency:
* If you write to the DB first and then publish to the queue: the process may crash after the DB commit but before publishing the event, leaving downstream services unaware of the change.
* If you publish to the queue first and then write to the DB: the database write may fail (e.g., due to a constraint violation), but the event is already out, triggering incorrect downstream actions.

### Architecture & Mechanics

The **Transactional Outbox Pattern** solves the dual-write problem by saving the message to a dedicated table (`outbox`) *within the same local database transaction* as the business state change, guaranteeing atomic execution.

```
┌─────────────────────────────────────────────────────────┐
│                     MY_SERVICE                          │
│                                                         │
│  1. BEGIN TRANSACTION                                   │
│  2. INSERT INTO orders (order_id, status) VALUES (...)  │
│  3. INSERT INTO outbox (msg_id, payload, status) ...    │
│  4. COMMIT TRANSACTION                                  │
└──────────────────────────┬──────────────────────────────┘
                           │ (Atomic local write)
                           ▼
                  ┌─────────────────┐
                  │    Local DB     │
                  │ ┌─────────────┐ │
                  │ │Orders Table │ │
                  │ └─────────────┘ │
                  │ ┌─────────────┐ │
                  │ │Outbox Table │ │
                  │ └──────┬──────┘ │
                  └────────┼────────┘
                           │
             ┌─────────────┴─────────────┐
             ▼                           ▼
  [Polling Publisher]         [Transaction Log Miner (CDC)]
  SELECT * FROM outbox         Reads binary log (WAL) e.g.,
  WHERE status = 'PENDING'     via Debezium/Postgres Replication
             │                           │
             └─────────────┬─────────────┘
                           │
                           ▼ (Publish event)
                  ┌─────────────────┐
                  │  Kafka/RabbitMQ │
                  └─────────────────┘
```

### 1. Polling Publisher (Database Polling)
* **Mechanism:** A background thread periodically queries the outbox table:
  ```sql
  SELECT * FROM outbox 
  WHERE status = 'PENDING' 
  ORDER BY created_at ASC 
  LIMIT 100;
  ```
  It publishes these events to the message broker, and marks them `PROCESSED` (or deletes them) upon receiving a broker ACK.
* **Pros/Cons:** Simple to implement; works on any database. However, frequent polling introduces database CPU overhead, latency, and scanning index overhead under high transaction rates.

### 2. Change Data Capture (CDC / Transaction Log Mining)
* **Mechanism:** An external tool (e.g., Debezium) tails the database's Write-Ahead Log (WAL) or transaction log directly. It extracts writes to the `outbox` table and streams them into the message broker.
* **Pros/Cons:** Near-zero database CPU overhead, sub-millisecond publishing latency, completely decoupled. Requires database administrative access to replication logs and operational upkeep of a CDC cluster.

---

## 7. High-Impact Interview Questions & Principal-Level Answers

### Q1: [The Saga Coordinator Crash] If an Orchestrator state-machine process crashes midway through a long-running, multi-step Saga, how does the system recover its state and ensure consistency?
**Answer:**
A production-grade Saga Orchestrator must persist its execution state at every step using an **Event Sourcing** or **State-Store Database** (the "Saga Log") within a local transaction *before* dispatching commands to participant services.
1. **Crash Recovery:** When the Orchestrator service boots or a backup instance takes over, it reads the persisted state store to find active Sagas.
2. **Idempotent Replay:** It inspects the last successfully recorded step and re-dispatches the command. Since message brokers guarantee at-least-once delivery, downstream services may receive duplicate requests; thus, **all participants must be idempotent**.
3. **Rollback Re-evaluation:** If the crashed orchestrator had already written a transition to its log indicating a step failed, but crashed before issuing compensating transactions, the newly recovered coordinator reads the failure state and starts executing the compensating chain.

### Q2: [Outbox Poller Scaling Race] Under high concurrency, we scale out our Outbox Polling service to 10 parallel instances. How do you prevent multiple pollers from picking up and publishing the exact same outbox row twice?
**Answer:**
If multiple pollers run the same query, they will lock the same indexes or fetch duplicate rows. This is solved using three database patterns:
1. **Pessimistic Locking with SKIP LOCKED (Recommended):**
   Use Postgres/MySQL `SELECT ... FOR UPDATE SKIP LOCKED`. This allows each thread to lock a batch of rows and instructs concurrent queries to skip locked rows and fetch the next available records immediately, preventing duplicate processing and contention:
   ```sql
   UPDATE outbox
   SET status = 'PROCESSING', locked_by = :worker_id
   WHERE id IN (
       SELECT id FROM outbox
       WHERE status = 'PENDING'
       ORDER BY created_at ASC
       LIMIT 50
       FOR UPDATE SKIP LOCKED
   )
   RETURNING *;
   ```
2. **Database Partitioning:**
   Partition the outbox table by a hash of the partition key (e.g., modulo of `order_id` or `user_id`). Dedicate specific worker nodes to poll specific database partitions, ensuring no two workers read the same set of rows.
3. **Distributed Locks (e.g., Redis Redlock):**
   Workers acquire a distributed lock on specific row ranges or shards before polling. This is less performant and introduces external dependencies, making `SKIP LOCKED` the industry standard.

### Q3: [Out-of-Order Message Anomaly] Under heavy load, a network partition delays a Saga's step 1 transaction. The orchestrator times out, decides to abort, and publishes a Compensating Event (e.g., "Cancel Order"). Due to network re-routing, the "Cancel Order" message arrives at the Inventory Service BEFORE the initial "Reserve Inventory" message. How do you prevent a permanent resource leak?
**Answer:**
This is the **Out-of-Order Compensating Message** problem. If a service processes the cancel message first, it does nothing because the reservation doesn't exist yet. When the delayed reservation message subsequently arrives, it executes successfully, leaving the inventory permanently leaked.
* **Solution (Tombstoning / Reservation Gating):**
  1. Maintain a state tracking table in the participant database containing `Saga_ID` and its `status` (`HELD`, `RELEASED`, `CANCELLED`).
  2. When the "Cancel Order" event arrives first, the Inventory Service writes a **Tombstone Record** to the state tracking table: `INSERT INTO saga_states (saga_id, status) VALUES ('Saga_123', 'CANCELLED')`.
  3. When the delayed "Reserve Inventory" message eventually arrives, the service first queries the state tracking table. Seeing that `Saga_123` is already marked `CANCELLED`, it discards the reservation request immediately, avoiding the leak.

### Q4: [Kafka Partition Count Change Danger] What happens to message ordering if you dynamically increase the partition count of a Kafka topic while producers are publishing hashed keys?
**Answer:**
Increasing partition count breaks Key-to-Partition ordering. Kafka hashes message keys using $\text{MurmurHash2}(Key) \pmod{\text{Partition Count}}$ to target a partition.
* **The Failure:** If the partition count increases (e.g., from 10 to 12), the modulo hash calculation changes. Messages with the same Key will now be routed to a *different* partition.
* **The Consequences:** Upstream events for account `X` were written to Partition 3. Post-scale events for account `X` land on Partition 5. A consumer reading Partition 5 will process updates *before* the consumer reading Partition 3 has completed processing older history, violating sequential ordering and causing data corruption.
* **The Mitigation:** If strict ordering is required, you must create a **new topic** with the desired partition count, configure a routing bridge to replicate events from the old topic, and drain the old topic completely before cutting over producers and consumers.

### Q5: [The Multi-Tenant Noise Neighbor Problem] In a multi-tenant SaaS architecture utilizing a shared message queue, a single heavy tenant floods the system with 1 million background tasks, starving all other tenants. How do you design the queueing topology to enforce fairness?
**Answer:**
To isolate heavy tenants from starving smaller tenants, you implement a **Fair-Share Queueing Topology**:

```
                               ┌─────────────────┐
                       ┌──────►│ Tenant_A Queue  ├──► [Worker Pool A] (Dedicated)
                       │       └─────────────────┘
 ┌─────────────────┐   │       ┌─────────────────┐
 │ Tenant Router   ├───┼──────►│ Tenant_B Queue  ├──► [Worker Pool B] (Dedicated)
 └─────────────────┘   │       └─────────────────┘
                       │       ┌─────────────────┐
                       └──────►│ Shared Pool Q   ├──► [Shared Worker Pool] (Token Bucket)
                               └─────────────────┘
```

1. **Routing and Dynamic Priority Queuing:**
   Do not route all tenant messages to a single monolithic queue. Use a routing layer that inspects the `tenant_id` in the message header.
2. **Tenant-Isolated Queues (Virtual or Physical):**
   Assign each tenant a dedicated queue, or utilize a **sharded queue array**. 
3. **Token Bucket Rate Limiting on Consumers:**
   Workers read from queues using a round-robin scheduler across tenant queues rather than consuming from a single queue. If a tenant's message rate exceeds their SLA limit, their specific queue consumption is throttled, allowing other tenant queues to be drained.
4. **Shared Pool with Fallback Overflow:**
   Assign high-value tenants to dedicated workers, and route all noisy, low-tier tenants to a shared, throttled pool where they compete only with each other, preserving the performance of enterprise tenants.

## Interview Questions & Answers

### Q1: What is the difference between Orchestration and Choreography Sagas?
- **Answer:** In **Orchestration**, a centralized controller (Orchestrator) explicitly coordinates the execution sequence of local transactions and triggers compensating rollbacks on downstream failures. In **Choreography**, there is no central controller; microservices listen to event streams and execute their local transactions independently, making choreography more decoupled but harder to audit.
