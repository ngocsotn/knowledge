# Backend-for-Frontend (BFF) Pattern

The Backend-for-Frontend pattern establishes separate, dedicated API gateways customized for specific client interfaces (e.g., one BFF for the iOS App, one BFF for the Web App).
- **Core Role:** Aggregates multiple downstream microservice endpoints, translates schemas, filters out redundant payload fields, and optimizes payloads to match the device's screen or network limits.

## Interview Questions & Answers

### Q1: Why is the BFF pattern highly beneficial for mobile clients?
- **Answer:** Bandwidth and latency optimization. Instead of a mobile client executing 10 sequential API calls to downstream services over slow mobile networks, the client makes a single query to the Mobile BFF. The BFF executes those 10 calls over the high-speed internal VPC network, aggregates the results, and returns a single, minimal JSON payload, reducing network overhead.
