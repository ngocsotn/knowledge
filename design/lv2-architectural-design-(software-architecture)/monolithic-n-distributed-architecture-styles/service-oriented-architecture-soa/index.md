# Service-Oriented Architecture (SOA)

SOA is an enterprise architectural pattern where distinct business applications share capabilities through standard reusable service interfaces over an Enterprise Service Bus (ESB).
- **Microservices vs. SOA:** SOA focuses on enterprise application integration via an ESB, sharing schemas. Microservices focus on absolute decoupling, utilizing dumb pipes (e.g., lightweight HTTP or Kafka) and private databases.

## Interview Questions & Answers

### Q1: What is the role of the Enterprise Service Bus (ESB) in SOA, and why do microservices reject it?
- **Answer:** In SOA, the ESB acts as a smart central mediator handling message translation, orchestration, and complex routing. Microservices reject this because smart pipes create central bottlenecks and tight coupling. Microservices enforce "Smart endpoints, dumb pipes," utilizing simple HTTP or Kafka brokers while keeping routing logic inside independent services.
