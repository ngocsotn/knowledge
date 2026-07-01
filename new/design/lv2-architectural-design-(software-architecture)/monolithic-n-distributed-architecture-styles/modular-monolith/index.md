# Modular Monolith

A Modular Monolith keeps all code in a single physical repository and running process, but strictly enforces structural boundaries between distinct domain modules.
- **Enforcement:** Modules can only communicate with each other through explicit, public interface contracts (APIs) or in-memory events, banning direct code imports or cross-domain package references.

## Interview Questions & Answers

### Q1: What is the main advantage of a Modular Monolith?
- **Answer:** Balanced complexity. It offers the strict modularity and boundary separation of microservices (making it easy to split into independent microservices later if needed) without the distributed network latency, deployment overhead, or remote transaction complexity of multi-service systems.
