# Multi-Region Database Replication

Multi-region replication distributes database nodes globally across distinct continents or cloud data centers.
- **Async replication:** Essential due to the speed-of-light physical speed limit across global fiber networks (cross-continental latency can exceed 150ms).
- **Synchronous replication over regions:** Unviable under high write loads as it forces clients to block for hundreds of milliseconds waiting for multi-datacenter network round-trips.

## Interview Questions & Answers

### Q1: What is the biggest challenge of active-active multi-region replication?
- **Answer:** Data Consistency and Write Conflicts. If a user in New York updates a record and a user in Tokyo updates the identical record at the same second, both regional masters accept writes locally. Reconciling these concurrent writes across oceans without sacrificing availability or locking up database threads requires complex multi-version databases or conflict-free CRDT algorithms.
