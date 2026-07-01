# SQL Data Modeling: Normalization & Denormalization

Guide covering database relational modeling, normal forms, and denormalization.

## Data Modeling Principles

### 1. Normalization (Relational Standard)
The process of organizing data fields to minimize redundancy, eliminate anomalies, and guarantee data integrity.
- **1NF (First Normal Form):** Atomic values; no repeating groups.
- **2NF (Second Normal Form):** Meets 1NF; zero partial dependencies (every non-prime attribute depends on the entire primary key).
- **3NF (Third Normal Form):** Meets 2NF; zero transitive dependencies (non-prime attributes depend only on the primary key, not on other non-prime attributes).

### 2. Denormalization (Read Optimization)
The deliberate introduction of redundancy (e.g., duplicating aggregate columns or nested details inside a table) to speed up complex queries by avoiding slow multi-table JOINs.
- **Trade-off:** High write complexity; risk of data inconsistency.

## Interview Questions & Answers

### Q1: When is it highly beneficial to Denormalize your SQL database?
- **Answer:** Denormalization is beneficial in read-heavy systems where database execution is bottlenecked by complex, multi-table `JOIN` operations (e.g., analytical dashboards, e-commerce home screens). By duplicating specific fields (like pre-calculating total order prices directly inside the `Users` table), reads can be executed instantly at $O(1)$ disk page lookups. However, you must implement strict application-level or trigger-based background jobs to update duplicate columns during mutations to prevent stale-data inconsistencies.
