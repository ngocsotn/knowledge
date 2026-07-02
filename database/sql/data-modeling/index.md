# SQL Data Modeling: Normalization, Denormalization & Outbox Sync

Comprehensive study guide covering database relational modeling, normal forms (1NF through BCNF), write anomalies, denormalization, transactional outbox pattern, and Change Data Capture (CDC).

---

## 1. Relational Data Modeling & Normal Forms

Normalization organizes database fields to minimize data redundancy, eliminate write anomalies, and guarantee data integrity.

### The Write Anomalies
To understand why normalization is necessary, we must analyze the three classic **Write Anomalies**:
1. **Insert Anomaly**: Inability to insert a fact because another unrelated fact is missing (e.g., cannot record a new course in a student-course table until at least one student enrolls).
2. **Update Anomaly**: Modifying a duplicated field requires updating multiple rows on disk. If a single row is missed, the database enters an inconsistent state.
3. **Delete Anomaly**: Deleting a record unintentionally wipes out an unrelated fact (e.g., deleting a student removes the only record of a course's existence).

### The Progression of Normal Forms

```
  ┌────────────────────────────────────────────────────────┐
  │ 1NF: Atomic values & Primary Keys                       │
  └───────────────────────────┬────────────────────────────┘
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │ 2NF: No Partial Dependencies (Every column on all PK)  │
  └───────────────────────────┬────────────────────────────┘
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │ 3NF: No Transitive Dependencies (A ──► B ──► C)        │
  └───────────────────────────┬────────────────────────────┘
                              ▼
  ┌────────────────────────────────────────────────────────┐
  │ BCNF: Every Determinant must be a Candidate Key        │
  └────────────────────────────────────────────────────────┘
```

* **1NF (First Normal Form)**: Columns must contain atomic values (no nested arrays or comma-separated lists), and each table must have a unique primary key.
* **2NF (Second Normal Form)**: Meets 1NF, and contains **zero partial dependencies**. Every non-prime column must depend on the *entire* primary key (highly relevant for composite keys).
* **3NF (Third Normal Form)**: Meets 2NF, and contains **zero transitive dependencies**. Non-prime columns must depend *only* on the primary key, and not on other non-prime columns (i.e., if $A \to B$ and $B \to C$, then $C$ is transitively dependent on $A$ and must be split into a separate table).
* **BCNF (Boyce-Codd Normal Form)**: A stricter variant of 3NF. It states that for every non-trivial functional dependency $X \to Y$, the determinant $A$ **must be a candidate key**.

---

## 2. Denormalization Mechanics & Read Optimization

While normalization is the golden standard for write integrity, it comes with a major performance penalty: **high read latency** due to expensive multi-table `JOIN` operations.

To speed up reads, read-heavy architectures deliberately introduce redundancy (**Denormalization**) to pre-calculate values or nest relationships, turning multi-table lookups into $O(1)$ disk page reads.

### Synchronization Patterns for Denormalized Data
* **Application-Level Sync**: The backend service writes updates to both the primary table and the denormalized summary table within a single transaction. (Risk of performance bottlenecks and index locking).
* **Database Triggers**: Executed natively by the database engine upon mutation. (Consumes database CPU, hard to debug, hides business logic).
* **Materialized Views**: A physical database table that stores query results. Can be refreshed periodically:
  ```sql
  REFRESH MATERIALIZED VIEW CONCURRENTLY active_user_stats;
  ```

---

## 3. Microservice Sync Patterns: Outbox & CDC

In distributed microservices, databases must often propagate state updates to external systems (e.g., streaming order updates to Kafka for a shipping service to consume).

### A. The Dual-Write Problem (Anti-Pattern)
Attempting to write to a local database and publish to a Message Queue in one step is a dangerous anti-pattern. If the database commit succeeds but the network connection to the message broker fails, the systems enter an inconsistent state.

```
                     The Dual-Write Failure
      ┌──────────────────────────────────────────────────┐
      │                   API Request                    │
      └──────┬────────────────────────────────────┬──────┘
             │                                    │
             ▼ (Commit Success)                   ▼ (FAIL - Broker Down)
     ┌──────────────┐                     ┌──────────────┐
     │  Local SQL   │                     │ Kafka Broker │
     │  Order DB    │                     │ (Missing Evt)│
     └──────────────┘                     └──────────────┘
```

### B. The Transactional Outbox Pattern
Ensures reliable transaction delivery of domain events without using slow 2-Phase Commit (2PC) distributed locks.

```
 1. Write Data + Event in One Transaction
 ┌────────────────────────────────────────────────────────────────────────┐
 │ BEGIN TRANSACTION;                                                     │
 │   INSERT INTO orders (id, user_id, total) VALUES (101, 5, 250);        │
 │   INSERT INTO outbox (id, event_type, payload) VALUES (1, 'created', '{"id": 101}'); │
 │ COMMIT;                                                                │
 └────────────────────────────────────────────────────────────────────────┘
                                     │
                                     ▼
 2. Background Poller / CDC  ────────┼────────► 3. Publish to Message Broker
   - Scans outbox table              │            - Kafka / RabbitMQ
   - Deletes/Marks rows processed    ▼
```

- **Mechanism**:
  1. The application writes to the target tables (e.g., `orders`) and appends a row to a dedicated `outbox` table inside the **same local database transaction**.
  2. A separate background process (Outbox Poller or CDC) polls the `outbox` table for unsent records.
  3. The poller publishes the events to the Message Queue, and deletes or marks the outbox rows as sent only *after* receiving a successful broker acknowledgement.

### C. Change Data Capture (CDC)
An alternative to database polling. A specialized utility (e.g., Debezium) directly streams raw Write-Ahead Log (WAL) byte-level insertions to Kafka.
* **Pros**: Zero application overhead, captures all changes (including direct database SQL edits), near-real-time ($<50\text{ms}$).

---

## 4. Highly Technical Interview Q&As

### Q1: When is it highly beneficial to Denormalize your SQL database?
- **Answer**: Denormalization is beneficial in read-heavy systems where database execution is bottlenecked by complex, multi-table `JOIN` operations (e.g., analytical dashboards, e-commerce home screens). By duplicating specific fields (like pre-calculating total order prices directly inside the `Users` table), reads can be executed instantly at $O(1)$ disk page lookups. However, you must implement strict application-level or trigger-based background jobs to update duplicate columns during mutations to prevent stale-data inconsistencies.

### Q2: Explain the differences between 3NF and Boyce-Codd Normal Form (BCNF) with a concrete scenario.
- **Answer**: 3NF permits functional dependencies where the determinant is *not* a candidate key, provided the dependent attribute is a prime attribute (part of a candidate key). BCNF removes this exception: every determinant must be a candidate key.
  - *Scenario*: A student-tutoring database table has columns: `[StudentID, Subject, Tutor]`.
  - *Constraints*: 
    - Each student has one tutor per subject: `(StudentID, Subject) -> Tutor`.
    - Each tutor teaches only one subject: `Tutor -> Subject`.
  - *Analysis*:
    - **Candidate Keys**: `(StudentID, Subject)` and `(StudentID, Tutor)`.
    - **3NF Check**: The table is in 3NF. In the dependency `Tutor -> Subject`, `Tutor` is *not* a candidate key. However, `Subject` is a prime attribute (part of the candidate key `(StudentID, Subject)`). Thus, 3NF is satisfied, but we still have data redundancy (tutor-subject mapping is repeated).
    - **BCNF Violation**: BCNF is violated because, in the dependency `Tutor -> Subject`, `Tutor` is a determinant but is **not a candidate key**.
    - **BCNF Resolution**: Split the table into two tables: `Tutor_Subject([Tutor, Subject])` and `Student_Tutor([StudentID, Tutor])`. This eliminates redundancy and write anomalies.

### Q3: What are database anomalies (Insert, Update, Delete) and how does normalization resolve them? Provide concrete tables.
- **Answer**: Database anomalies occur due to data redundancy, where unrelated facts are mixed inside a single table.
  - *Unnormalized Table*: `Employee_Departments([EmpID, Name, DeptName, DeptManager])`
  - *Anomalies*:
    - **Insert Anomaly**: Cannot add a new department (e.g., `Research`) until we hire at least one employee to work in it (since `EmpID` is part of the primary key and cannot be NULL).
    - **Update Anomaly**: If the manager of `HR` changes, we must update `DeptManager` across hundreds of employee rows. If a single row is missed, the database is corrupt.
    - **Delete Anomaly**: If we fire the only employee in `IT`, we unintentionally wipe out all record of the IT department's existence and who manages it.
  - *Resolution via 3NF Normalization*: Split into two tables:
    - `Employees([EmpID, Name, DeptID])`
    - `Departments([DeptID, DeptName, DeptManager])`
    - Now, we can add departments independently (resolves Insert), update managers in a single row (resolves Update), and delete employees without losing department records (resolves Delete).

### Q4: How does the Transactional Outbox Pattern solve the dual-write problem when propagating database mutations to an external Message Queue?
- **Answer**: The dual-write problem occurs when an application attempts to write to a local database and publish to a Message Queue sequentially. If the database transaction commits but the network call to the broker fails (or the broker is down), the database state diverges from the event stream.
  - *Outbox Solution*: The application writes to the local tables and inserts a row to an `outbox` table inside the **same local database transaction**. This guarantees ACID atomic execution: either both writes succeed or both fail. A background outbox-poller process (or Debezium streaming database WAL changes) reads the `outbox` table and publishes the events to the broker. The outbox row is deleted or marked processed only *after* the broker successfully acknowledges receipt of the event, ensuring **at-least-once** event delivery.

### Q5: What is Change Data Capture (CDC), and how does it compare to application-level outbox polling?
- **Answer**: Change Data Capture (CDC) is an architectural pattern that tracks and streams state changes directly from the database level to external systems.
  - *CDC (Log-Based)*: A utility (like Debezium) parses the database's Write-Ahead Log (WAL) or binlog directly, bypassing the SQL engine entirely. When a WAL insert is parsed, it formats it as a JSON/Avro record and streams it to Kafka.
  - *Comparison*:
    - **Performance**: CDC has near-zero overhead on the database CPU since it reads append-only WAL files asynchronously. Outbox polling requires continuous `SELECT ... LIMIT` queries on the database engine, creating constant transactional read locks.
    - **Code Coupling**: CDC requires no changes to application code or domain models. Outbox polling requires developers to manually construct and save event payloads inside every mutation pathway.
    - **Robustness**: CDC captures all changes, including manual administrative SQL edits. Outbox polling only captures changes executed via the application logic.

