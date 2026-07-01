# Apache Kafka: Stateful Processing & Streams

Guide covering Stateful Processing, Windowing, and Aggregations in event streams.

## Streaming Patterns

### 1. Stateful Processing
Streaming applications maintain state (e.g., running total, user session profiles) across incoming events, leveraging highly optimized local databases like RocksDB.

### 2. Windowing
Grouping events based on time ranges for temporal analysis.
- **Tumbling Window:** Fixed-size, non-overlapping time blocks (e.g., every 5 minutes).
- **Hopping Window:** Fixed-size, overlapping time blocks (e.g., 5-minute window sliding every 1 minute).
- **Session Window:** Dynamic windows defined by inactivity gaps.

### 3. Aggregation
Combining stream elements to compute running counts, averages, or max/min statistics.

## Interview Questions & Answers

### Q1: How does Kafka Streams maintain state across container restarts?
- **Answer:** Kafka Streams uses local embedded key-value stores (usually RocksDB) to process stateful operations at $O(1)$ speeds. To survive crashes and restarts, this state is continuously backed up to a dedicated, highly durable, compacted Kafka topic called a **Changelog Topic**. Upon restart, the container fetches the changelog topic from the last offset and fully rebuilds its local RocksDB state, ensuring fault-tolerant, stateful streaming.
