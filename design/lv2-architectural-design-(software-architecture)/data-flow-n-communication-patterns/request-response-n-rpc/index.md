# Request-Response & RPC Patterns

The traditional synchronous communication paradigm.
- **Request-Response:** The client opens an HTTP connection, sends a request payload, blocks, and waits for the server to process and return a response.
- **RPC (Remote Procedure Call):** Allows a program to execute a function on a remote server as if it were a local function call, abstracting network layers (e.g., gRPC using Protocol Buffers and HTTP/2).

## Interview Questions & Answers

### Q1: When do you choose gRPC over REST for service-to-service communication?
- **Answer:** Choose **gRPC** for internal microservice communication where high throughput, low latency, and strict API schemas are critical. gRPC's binary Protobuf serialization and multiplexed HTTP/2 transport are up to 10x faster than REST's text-based JSON over HTTP/1.1. Choose **REST** for external, public-facing APIs where universal client compatibility and simple caching are prioritized.
