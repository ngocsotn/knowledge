# Clean Architecture (Ports & Adapters)

Clean Architecture (or Hexagonal/Ports & Adapters) isolates core business logic from external frameworks, databases, and UI layers.
- **The Dependency Rule:** Dependencies must always point inward. The inner Core (entities and use cases) must never know anything about outer layers (like PostgreSQL, Express/Spring, or React).
- **Ports (Interfaces):** Inner core defines the interface contract (e.g., `UserRepository` interface).
- **Adapters (Implementations):** Outer layer implements the interface (e.g., `PostgreSQLUserRepository` struct).

## Interview Questions & Answers

### Q1: Why is Clean Architecture highly valued for enterprise systems?
- **Answer:** Independent of databases and frameworks. Because the core business logic has zero external imports, you can swap your database from PostgreSQL to MongoDB, or change your router framework, by simply writing a new adapter class without touching or risking your core business rules.
