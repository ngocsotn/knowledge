# Saga Pattern in Microservices

Comprehensive interview study guide covering distributed transactions, the limits of 2-Phase Commit (2PC), and Saga design patterns (Orchestration vs. Choreography).

---

## 1. The Distributed Transaction Challenge

In a monolithic database, ensuring transaction consistency across multiple tables is easy—you use standard SQL ACID transactions.

In a microservices architecture:
* Every service owns its isolated database (Database-per-Service pattern).
* A single business transaction (e.g., placing an order) requires writing data across the Orders DB, Inventory DB, and Payment DB.
* You **cannot** use standard SQL transactions.
* **Why Two-Phase Commit (2PC) fails:** While 2PC provides distributed ACID consistency, it is a **blocking protocol**. It requires locking database resources across all services until every node agrees to commit. At scale, this introduces severe latency, high deadlock rates, and degrades system availability.

---

## 2. Meaning & The Saga Pattern

The **Saga Pattern** is a design pattern used to manage data consistency across distributed microservices without locking resources.

* **How it works:** A Saga is a sequence of **local transactions**. Each local transaction updates data within a single service's database and emits an event or message. This event triggers the next local transaction in the succeeding service.
* **Handling Failures:** If any local transaction fails, the Saga must execute a series of **Compensating Transactions** to reverse the changes made by the prior successful local transactions (acting as a roll-back mechanism).

---

## 3. Saga Design Topologies

There are two primary ways to coordinate a Saga:

### 1. Choreography-Based Saga (Decentralized / Event-Driven)
* **Mechanism:** No central coordinator exists. Services communicate by publishing and subscribing to events using a message broker (e.g., Kafka).
* **Execution Flow:**
  * Order Service creates order in pending state, emits `OrderCreated` event.
  * Payment Service receives `OrderCreated`, processes payment, emits `PaymentSucceeded` event.
  * Inventory Service receives `PaymentSucceeded`, reserves stock, emits `SagaCompleted` event.
* **Pros/Cons:** Simple to understand, high decoupling, no single point of coordination. However, it can become hard to track as the number of services grows (leading to "spaghetti" event flows).

### 2. Orchestration-Based Saga (Centralized)
* **Mechanism:** A centralized service (the **Orchestrator** or State Machine) explicitly defines, manages, and coordinates the execution flow of all local transactions.
* **Execution Flow:**
  * The Orchestrator calls the Order Service, receives success.
  * The Orchestrator calls the Payment Service, receives success.
  * The Orchestrator calls the Inventory Service. If it fails, the Orchestrator explicitly calls compensating endpoints on Payment and Order services to undo changes.
* **Pros/Cons:** Clear execution paths, easy debugging, centralized state tracking. However, it introduces a single point of logic and coordination.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is a Compensating Transaction in the Saga pattern, and how does it differ from a standard SQL ROLLBACK?
* **Answer:** A standard SQL `ROLLBACK` is automatic and low-level: it utilizes write-ahead logs to instantly undo uncommitted database changes in memory/disk, returning the system to its initial state as if the query never ran. A **Compensating Transaction** is an explicit, application-level business action written by developers to undo a *committed* local transaction. For example, if a payment transaction committed successfully, but the downstream inventory reservation fails, the compensating transaction is an explicit API call to execute a refund, creating a new offsetting record in the database.

### Q2: When would you choose Orchestration over Choreography for a Saga?
* **Answer:** Choose **Choreography** for simple workflows with 2-4 services, where event flows are straightforward and decoupling is the primary goal. Choose **Orchestration** for complex business workflows (5+ services, branching logic, loops, conditional waits) where tracking state across decentralized events becomes too hard to debug, or when you need a single authoritative service to monitor SLA/SLO deadlines.

### Q3: How do you handle idempotency inside Saga consumers?
* **Answer:** Since message brokers guarantee At-Least-Once delivery, consumers can receive the same Saga event multiple times. To guarantee consistency, all Saga event consumers must be completely **Idempotent**. This is typically implemented using the **Idempotent Consumer Pattern**:
  1. Every Saga execution carries a unique `Saga_ID` or `Transaction_ID`.
  2. The receiving service checks a deduplication table in its database to see if the `Transaction_ID` has already been processed.
  3. If present, the service discards the message immediately. If absent, it processes the task and saves the ID in the database within a single local transaction.
