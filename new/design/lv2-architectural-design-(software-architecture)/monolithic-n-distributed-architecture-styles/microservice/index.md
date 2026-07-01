# Microservice Architecture

Microservice architecture structures an application as a collection of small, autonomous, independently deployable services modeled around business domain capabilities.
- **Service Boundaries:** Each microservice owns its private database; direct database-sharing between services is strictly prohibited to prevent tight coupling.

## Interview Questions & Answers

### Q1: Why must each microservice own its database in a distributed system?
- **Answer:** To prevent tight coupling. If Microservice A and Microservice B share a database schema, any table modification forces both services to coordinate releases, destroying independent deployment velocity and violating service boundaries.
