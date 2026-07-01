# Event-Driven Architecture Patterns

Guide covering Fire-and-Forget, Worker Queue, Fan-Out, and Request-Reply patterns.

## Architecture Patterns

### 1. Fire-and-Forget
The producer sends an event to the message broker and immediately continues processing without waiting for a confirmation or response from the consumer.
- **Usage:** Logging, telemetry, non-critical background jobs.

### 2. Worker Queue (Competing Consumers)
Multiple concurrent consumers pull tasks from a single shared queue. Each task is processed by only one worker, distributing workload.
- **Usage:** Image resizing, report generation, video rendering.

### 3. Fan-Out
A single event published to a topic is copied and routed to multiple independent queues, allowing different microservices to process the same event in parallel.
- **Usage:** Triggering billing, inventory update, and email notifications simultaneously when an order is placed.

### 4. Request-Reply (RPC over MQ)
The sender publishes a message with a `ReplyTo` header specifying a temporary reply queue and a unique `CorrelationID`. The consumer processes the message and sends the response back to the designated reply queue matching the ID.

## Interview Questions & Answers

### Q1: How do you implement asynchronous Request-Reply using a Message Broker?
- **Answer:** The client creates a temporary, private callback queue (Reply-To queue). When publishing the request, it generates a unique `CorrelationID` and attaches it along with the reply queue name as headers to the message payload. The server processes the request, extracts these headers, and publishes the response to the specified callback queue with the identical `CorrelationID`. The client listens on the callback queue and matches the response to the original request using the correlation ID.
