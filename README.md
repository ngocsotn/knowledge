# Software Engineering Interview Knowledge Repository

Comprehensive, high-quality, structured study guides covering staff-level concepts across 17 major software engineering categories.

---

## Table of Contents

### 1. Backend Basics, Security & Auth
* [Backend Basics](./backend/basic/README.md)
* [Same-Origin Policy (SOP)](./backend/backend-security/SOP/README.md)
* [Cross-Origin Resource Sharing (CORS)](./backend/backend-security/CORS/README.md)
* [Cross-Site Scripting (XSS)](./backend/backend-security/XSS/README.md)
* [Cross-Site Request Forgery (CSRF)](./backend/backend-security/CSRF/README.md)
* [Cookies, Storage & Cache](./backend/backend-security/Cookies-storage-cache/README.md)
* [Distributed Denial of Service (DDoS)](./backend/backend-security/DDOS/README.md)
* [SQL Injection](./backend/backend-security/SQL-injection/README.md)
* [Rate Limiting](./backend/backend-security/Rate-limiting/README.md)
* [Secure File Uploads](./backend/backend-security/File-uploads/README.md)
* [Real-World Implementation Struggles](./backend/implementation-struggles/README.md)
* [Session Authentication](./backend/auth/Session/README.md)
* [JSON Web Tokens (JWT)](./backend/auth/JWT/README.md)
* [OAuth 2.0 & PKCE](./backend/auth/OAuth2/README.md)
* [RESTful API Design](./backend/api-design/restful/README.md)
* [GraphQL Design & Resolvers](./backend/api-design/graphql/README.md)
* [gRPC & RPC Streaming](./backend/api-design/rpc-grpc/README.md)
* [Pagination, Filtering & Sorting](./backend/api-design/pagination-filtering-sorting/README.md)

### 2. Databases & Storage
* [SQL vs NoSQL Databases](./database/SQL-vs-NoSQL/README.md)
* [Database Indexing](./database/Indexing/README.md)
* [Transactions & ACID Properties](./database/Transactions-ACID/README.md)
* [Sharding & Replication](./database/Sharding-replication/README.md)
* [Database Query Optimization](./database/Optimization/README.md)
* [Advanced Database Patterns (Saga, Event Sourcing, Outbox, Multi-Tenancy)](./database/advanced-patterns/README.md)
* [**Vector Databases & Semantic/Image Search (HNSW, IVF, pgvector, CLIP)**](./database/vector-databases/README.md) *(New)*

### 3. NodeJS, JavaScript & TypeScript
* [JavaScript Event Loop](./nodejs-js-ts/event-loop/README.md)
* [Async/Await & Promises](./nodejs-js-ts/async-await-promises/README.md)
* [Closures & Scopes](./nodejs-js-ts/closures/README.md)
* [TypeScript Type System](./nodejs-js-ts/typescript-types/README.md)
* [Garbage Collection (V8)](./nodejs-js-ts/garbage-collection/README.md)
* [JavaScript Core Concepts](./nodejs-js-ts/javascript-core/README.md)
* [Performance, Streams & Worker Threads](./nodejs-js-ts/performance-memory/README.md)
* [Package Managers (pnpm, npm, yarn)](./nodejs-js-ts/package-managers/README.md)

### 4. High-Level System Design & Architecture
* [System Scalability](./architect-system-design/Scalability/README.md)
* [Load Balancers & Consistent Hashing](./architect-system-design/Load-balancers/README.md)
* [Caching Strategies](./architect-system-design/Caching-strategies/README.md)
* [Message Queues (Kafka & RabbitMQ)](./architect-system-design/Message-queues/README.md)
* [CAP Theorem & PACELC](./architect-system-design/CAP-theorem/README.md)
* [High Availability & Disaster Recovery (HA/DR)](./architect-system-design/high-availability-dr/README.md)

### 5. Microservices Resilience
* [API Gateway](./microservice/API-gateway/README.md)
* [Service Discovery](./microservice/Service-discovery/README.md)
* [Saga Pattern (Distributed Transactions)](./microservice/Saga-pattern/README.md)
* [CQRS (Command Query Responsibility Segregation)](./microservice/CQRS/README.md)
* [gRPC vs REST](./microservice/gRPC-vs-REST/README.md)
* [Resilience Patterns (Circuit Breaker, Bulkheads, Retries)](./microservice/resilience-patterns/README.md)

### 6. Frontend Architecture & Performance
* [Rendering Patterns (SSR, SSG, ISR, Hydration)](./frontend/rendering-patterns/README.md)
* [DOM vs Virtual DOM](./frontend/dom-vdom/README.md)
* [Frontend Performance Optimization](./frontend/performance/README.md)
* [State Management](./frontend/state-management/README.md)
* [Advanced Architecture (Module Federation, Shadow DOM, Service/Web Workers)](./frontend/advanced-architecture/README.md)

### 7. Computer Networks
* [HTTP Protocols (HTTP/1.1, HTTP/2, HTTP/3)](./network/http-protocols/README.md)
* [TCP vs UDP](./network/tcp-udp/README.md)
* [Domain Name System (DNS)](./network/dns/README.md)
* [WebSockets & Socket Scaling](./network/websockets/README.md)
* [Network Security & TLS Handshakes](./network/security/README.md)

### 8. Software Engineering Principles
* [SOLID Principles](./principles/solid/README.md)
* [DRY, KISS, YAGNI](./principles/dry-kiss-yagni/README.md)
* [Architectural & Design Paradigms](./principles/design-paradigms/README.md)

### 9. Software Testing & Methodologies
* [Unit, Integration & E2E Testing](./testing/unit-integration-e2e/README.md)
* [TDD & BDD Methodologies](./testing/tdd-bdd/README.md)

### 10. Design Patterns
* [Creational Patterns](./design-patterns/creational/README.md)
* [Structural Patterns](./design-patterns/structural/README.md)
* [Behavioral Patterns](./design-patterns/behavioral/README.md)

### 11. DevOps & Cloud
* [Docker Containers](./devops-cloud/docker/README.md)
* [Kubernetes Orchestration](./devops-cloud/kubernetes/README.md)
* [CI/CD Pipelines](./devops-cloud/ci-cd/README.md)
* [Virtualization & Container Primitives](./devops-cloud/virtualization/README.md)
* [Cloud & Infrastructure Security](./devops-cloud/cloud-security/README.md)

### 12. Observability & Telemetry
* [Structured Logging](./observation/logging/README.md)
* [System Metrics](./observation/metrics/README.md)
* [Distributed Tracing](./observation/tracing/README.md)

### 13. Git Version Control
* [Git Workflows](./git/git-workflows/README.md)
* [Rebase vs Merge](./git/rebase-merge/README.md)
* [Git Internals & DAG Architecture](./git/internals/README.md)

### 14. Data Structures & Algorithms (DSA)
* [Time & Space Complexity](./dsa/complexity/README.md)
* [Arrays & Strings](./dsa/arrays-strings/README.md)
* [Linked Lists](./dsa/linked-lists/README.md)
* [Trees & Graphs](./dsa/trees-graphs/README.md)
* [Sorting & Searching](./dsa/sorting-searching/README.md)
* [Dynamic Programming & Greedy Algorithms](./dsa/dynamic-programming-greedy/README.md)

### 15. Project Management Methodologies
* [Agile Methodologies](./Project%20methodologies/agile/README.md)
* [Scrum Framework](./Project%20methodologies/scrum/README.md)
* [Kanban System](./Project%20methodologies/kanban/README.md)
* [Waterfall & V-Model](./Project%20methodologies/waterfall/README.md)

### 16. Culture Fit & Behavioral
* [STAR Method & Behavioral Scenarios](./culture-fit/README.md)

### 17. Staff Mock Interview Scenarios
* [Comprehensive System Architecture Scenarios](./interview-scenarios/README.md)
