# Event-Driven Architecture Concepts

Detailed guide covering core messages, events, queues, topics, and pub/sub mechanisms.

## Core Concepts

### 1. Message
A Message is a raw package of data containing an payload and metadata (headers). It represents a command or data delivery directed to a specific destination.
- **Characteristics:** Intended for a single consumer; usually synchronous or transactional.

### 2. Event
An Event is a record of an action or change of state that occurred in the past (e.g., `OrderPlaced`, `PaymentFailed`).
- **Characteristics:** Fact-based, immutable, and directed to whoever is interested (broadcast/multicast).

### 3. Queue
A point-to-point communication channel. Messages are placed in the queue and pulled by exactly one consumer.
- **Characteristics:** FIFO (First-In-First-Out) delivery guarantees; workload distribution across multiple workers.

### 4. Topic
A publish-subscribe logical channel where events are categorized and stored. Multiple subscribers can listen to the same topic.
- **Characteristics:** Broadcast/multicast; event stream storage.

### 5. Pub/Sub (Publish-Subscribe)
A messaging pattern where publishers send events without knowing who the subscribers are, and subscribers consume specific events asynchronously.

## Interview Questions & Answers

### Q1: What is the difference between a Message and an Event?
- **Answer:** A Message represents an intention or command—it tells a specific recipient to do something (e.g., `CreateInvoiceMessage`). An Event represents a historical fact—it records that something has already happened (e.g., `InvoiceCreatedEvent`). Events are broadcast to any interested parties, whereas messages are directed point-to-point.
