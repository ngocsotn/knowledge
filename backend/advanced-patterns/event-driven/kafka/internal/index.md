# Apache Kafka: Internal Architecture

Under the hood guide covering Partitions, Consumer Groups, Offsets, and Leader/Follower replication.

## Core Internal Mechanics

### 1. Partition
Topics are divided into multiple append-only, ordered commit logs called Partitions. Partitions allow Kafka to scale horizontally and support parallel processing.
- **Scale:** Partitions are distributed across brokers; each partition is read sequentially.

### 2. Consumer Group
A group of cooperating consumers reading from one or more topics. Each partition is assigned to exactly one consumer in the group to guarantee sequential processing.
- **Scale:** Speeds up consumption; handles automatic rebalancing.

### 3. Offset
A sequential integer ID assigned to each message inside a partition. Offsets represent the logical consumer position in the commit log.

### 4. Leader/Follower Replication
Each partition has one Leader broker and zero or more Follower brokers.
- **Leader:** Handles all read and write requests for the partition.
- **Followers:** Silently replicate the leader's logs to guarantee high availability.

## Interview Questions & Answers

### Q1: What happens if you have more Consumers in a Consumer Group than Partitions in a Topic?
- **Answer:** Any extra consumers in the group will remain completely idle. Kafka enforces a strict constraint: each partition within a topic can be assigned to at most one consumer in a consumer group at any given time. This guarantees in-order processing of partition logs. To scale consumption, you must increase the number of partitions.
