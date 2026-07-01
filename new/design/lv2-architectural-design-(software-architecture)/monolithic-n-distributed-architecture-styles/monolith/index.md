# Monolithic Architecture

A Monolith organizes all software layers, business domains, and API routers into a single, unified codebase compiled and running as a single process.
- **Pros:** Fast development startup; simple deployments; easy debugging; $O(1)$ in-memory function calls.
- **Cons:** Single point of failure; deployment bottleneck; hard to partition across massive development teams.

## Interview Questions & Answers

### Q1: When is a monolithic architecture highly preferred over microservices?
- **Answer:** In early-stage startups or greenfield projects. Monoliths prioritize development speed, rapid refactoring, and minimal operations overhead, allowing developers to pivot quickly without dealing with distributed systems complexity.
