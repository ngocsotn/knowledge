# Database Sharding

Sharding splits databases horizontally based on a shard key.

## Interview Questions & Answers

### Q1: How do you choose a good Shard Key for database sharding?
- **Answer:** A good shard key must have **High Cardinality** (many unique values, e.g., `user_id` instead of `country_code`) and **Even Distribution** (data is written equally across shards). Selecting a poor shard key (like `created_at` date) causes all new writes to hit the same newest shard, creating an expensive hot-spot bottleneck.
