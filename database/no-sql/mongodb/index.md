# MongoDB Architecture & Internals

Comprehensive study guide covering MongoDB's Document Model, replica sets, sharding, and write/read concerns.

## MongoDB Core Architecture

### 1. Document Model
MongoDB stores data as BSON (Binary JSON) documents. Documents are highly flexible, dynamic, and support nested structures, removing the need for join tables.

### 2. Replica Set (High Availability)
A group of MongoDB processes that maintain the identical data set.
- **Roles:** One Primary node (receives all writes) and multiple Secondary nodes (replicate logs asynchronously).
- **Failover:** If Primary crashes, Secondaries coordinate an election using majority consensus to promote a new Primary.

### 3. Sharding (Horizontal Scaling)
Splitting data across multiple physical shards using a shard key.
- **Components:** Shards (contain subsets of data), Config Servers (store cluster metadata), and `mongos` routers (proxy queries from clients to correct shards).

### 4. Write Concern
Determines the level of durability guarantee for write operations.
- `w: 1` (Default): Returns success as soon as primary node writes to disk/memory.
- `w: majority`: Returns success only after the majority of replica set nodes acknowledge the write.
- `j: true`: Guarantees the write is flushed to the on-disk journal before success.

### 5. Read Concern
Determines the isolation level of read operations.
- `local` / `available`: Returns node's current local data; vulnerable to dirty reads if transaction is rolled back.
- `majority`: Returns data acknowledged by a majority of replica set nodes, preventing dirty reads.
- `linearizable`: Guarantees a read always returns the result of the most recent confirmed write.

## Interview Questions & Answers

### Q1: What is the risk of reading from MongoDB with "local" read concern and writing with "w:1" write concern?
- **Answer:** High risk of **Dirty Reads**. If the Primary node receives a write and immediately reports success, a reader querying a Secondary with `local` read concern sees this new data. However, if the Primary crashes before replicating this write to the secondaries, a new Primary is elected, and the un-replicated write is permanently lost. The reader has read data that "disappeared", violating consistency. Use `w: majority` and `readConcern: majority` to guarantee absolute read consistency.
