# Data-Driven Design

Data-Driven Design is a software development philosophy where the codebase, service boundaries, and objects are modeled directly around database tables, relations, and data schemas.
- **Approach:** Prioritizes optimal data storage, quick CRUD scaffolding, and relational mappings first.

## Interview Questions & Answers

### Q1: What is the main drawback of Data-Driven Design compared to Domain-Driven Design?
- **Answer:** Anemic Domain Model. In Data-Driven systems, entities are typically dumb, state-only objects mapped to database rows, while the business logic gets scattered across procedural service classes. As the application grows, maintaining business constraints becomes difficult because there is no isolated, cohesive object layer capturing domain rules.
