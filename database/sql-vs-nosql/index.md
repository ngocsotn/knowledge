# SQL vs. NoSQL Databases

Comprehensive interview study guide comparing Relational (SQL) and Non-Relational (NoSQL) databases, their data models, and architectural trade-offs.

---

## 1. Core Differences: Relational vs. Non-Relational

| Feature | Relational (SQL) | Non-Relational (NoSQL) |
| :--- | :--- | :--- |
| **Data Model** | Tabular (Rows & Columns) | Key-Value, Document, Column-family, Graph |
| **Schema** | Static, predefined schema | Dynamic, flexible schema |
| **Relationships** | Structured relationships via Foreign Keys & Joins | Denormalized data, nested structures, or no strict joins |
| **Scaling** | Vertical scaling (Scale-up) | Horizontal scaling (Scale-out) |
| **Transactions** | Strict ACID guarantees | Eventual consistency / BASE model (often) |
| **Examples** | PostgreSQL, MySQL, SQL Server | MongoDB, Redis, Cassandra, Neo4j |

---

## 2. NoSQL Data Models & Use Cases

Non-relational databases are categorized into four major families:

### 1. Document Stores (e.g., MongoDB, CouchDB)
* **Structure:** Stores data as semi-structured documents (JSON, BSON, XML).
* **Strength:** Excellent for content management, user profiles, and catalogs where schemas evolve quickly or contain nested fields.
* **Querying:** High support for rich, expressive indexing and nested querying.

### 2. Key-Value Stores (e.g., Redis, DynamoDB, Memcached)
* **Structure:** Stores data as a simple schema-less key-to-value map.
* **Strength:** Extremely low-latency reads and writes (`O(1)`).
* **Usage:** Session caching, rate limiting, leaderboards, pub/sub messaging.

### 3. Column-Family (Wide-Column) Stores (e.g., Cassandra, ScyllaDB, HBase)
* **Structure:** Stores data in columns grouped into column families rather than rows.
* **Strength:** Optimized for massive write throughput and high-scale scanning of specific fields across billions of records.
* **Usage:** Time-series telemetry, IoT streams, analytics tracking.

### 4. Graph Databases (e.g., Neo4j, Amazon Neptune)
* **Structure:** Stores data in Nodes (entities), Edges (relationships), and Properties.
* **Strength:** Optimized for traversing complex, highly connected networks of relationships.
* **Usage:** Social networks, recommendation engines, fraud detection.

---

## 3. ACID vs. BASE

Relational databases prioritize consistency, while high-scale NoSQL systems often adopt the BASE model to achieve massive horizontal scalability.

### ACID (Relational Standard)
* **Atomicity:** All operations in a transaction succeed, or all fail (all-or-nothing).
* **Consistency:** A transaction brings the database from one valid state to another, maintaining constraints/indexes.
* **Isolation:** Concurrent execution of transactions yields the same state as if they executed sequentially.
* **Durability:** Once committed, transaction records are permanently saved, even in a power outage.

### BASE (NoSQL Standard)
* **Basically Available:** The database remains functional and available during network partitions, but some nodes might return stale data.
* **Soft state:** The state of the data can drift or change over time without user interaction because of replica synchronization.
* **Eventual consistency:** The data will eventually sync across all replicas, bringing them into a consistent state if no new updates are made.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: When would you choose PostgreSQL over MongoDB for a new project?
* **Answer:** Choose **PostgreSQL** when the domain requires complex, relational queries (joins), data integrity constraints (foreign keys, uniqueness, check constraints), and strict ACID transaction guarantees (e.g., financial ledger or checkout workflows). Choose **MongoDB** when the schema is dynamic or highly polymorphic (e.g., rich content management or user profiles), when data naturally maps to nested JSON structures, and when you need easy horizontal write/read scaling across clusters.

### Q2: What is "Denormalization" and why is it common in NoSQL?
* **Answer:** Denormalization is the practice of storing redundant copies of data within multiple records to optimize read performance and avoid expensive join operations. In SQL databases, data is normalized to eliminate redundancy. In NoSQL databases (like MongoDB), because cross-table/document Joins are either unsupported or highly inefficient, we duplicate related data inside the document (e.g., nesting author details directly within a post document) to allow fast, single-lookup reads.

### Q3: What is the CAP Theorem, and how does it relate to SQL and NoSQL?
* **Answer:** The CAP theorem states that a distributed system can only guarantee two of three attributes: **Consistency (C)**, **Availability (A)**, and **Partition Tolerance (P)**. Since network partitions (P) are inevitable in real-world networks, a distributed system must choose between:
  1. **CP (Consistency & Partition Tolerance):** Rejects requests if it cannot guarantee data is fully updated across all nodes (standard for clustered SQL and MongoDB).
  2. **AP (Availability & Partition Tolerance):** Continues accepting writes/reads on separated nodes, returning stale or conflicting data, resolving consistency eventually (standard for Cassandra and DynamoDB).
