# System & Software Design: The 4-Tier Hierarchy

Master study guides covering structural systems, software patterns, and coding paradigms organized across 4 distinct scales of size.

```
┌────────────────────────────────────────────────────────┐
│  1. SYSTEM DESIGN (Infrastructure, Scale, Topology)    │  <-- Biggest Scale (Macro)
└───────────────────────────┬────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────┐
│  2. ARCHITECTURAL PATTERNS (Monolith, MVC, Layers)     │  <-- High-level code structure
└───────────────────────────┬────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────┐
│  3. DESIGN PATTERNS (Singleton, Strategy, Observer)    │  <-- Medium scale (Subsystems & Classes)
└───────────────────────────┬────────────────────────────┘
                            ▼
┌────────────────────────────────────────────────────────┐
│  4. IDIOMS (Language-specific patterns like RAII)      │  <-- Smallest Scale (Micro)
└────────────────────────────────────────────────────────┘
```

---

## The 4 Levels of Design

### ☁️ [LEVEL 1: System Design (System Topology & Infrastructure)](./lv1-system-design-(infrastructure-n-topology)/index.md)
At this level, you do not care about the code itself. You care about how massive physical or virtual components are organized across the internet and data centers.
- **Scope:** Global. The entire network, hardware, and physical deployment topology.
- **Key Question:** *"How do we reliably serve 100 million users with high availability, low latency, and fault tolerance?"*
- **The Artifacts:** Network topology diagrams, cloud architecture maps, data flow pipelines, failover strategies.
- **Key Goals:** Scalability, reliability, security, cost efficiency, and physical performance.
- **Examples:**
  - **Load Balancing & CDNs:** Distributing incoming web traffic across multiple servers (NGINX, AWS CloudFront).
  - **Database Scaling:** Master-Slave Replication, Sharding (splitting a database horizontally), and Caching (Redis/Memcached).
  - **Data Partitioning Laws:** Applying the CAP Theorem and PACELC Theorem to choose between data consistency or fast system response during a network partition.

### 🏛️ [LEVEL 2: Architectural Design (Software Architecture)](./lv2-architectural-design-(software-architecture)/index.md)
We now zoom past the cloud infrastructure and enter the codebase. This level defines the structural rules, constraints, and organizational philosophies of your software.
- **Scope:** Application-wide. The high-level blueprint of the codebase, directories, and internal boundaries.
- **Key Question:** *"How do we partition our business logic so that 50 developers can write code simultaneously without stepping on each other's toes?"*
- **The Artifacts:** Component diagrams, package structures, directory layouts, dependency rules.
- **Key Goals:** Modularity, ease of testing, developer velocity, clear boundaries, and maintainability.
- **Examples:**
  - **Architectural Styles:** Microservices (splitting business capabilities into independent running programs) vs. Monolith (all code in one package).
  - **Code Organization Patterns:** MVC (Model-View-Controller), Layered (N-Tier) Architecture, and Clean/Hexagonal Architecture (isolating core business logic from database and UI frameworks).
  - **Integration Patterns:** Event-Driven Architecture (EDA) (services talking asynchronously via a message broker) and Domain-Driven Design (DDD) (modeling code strictly around business domain language).

### 🔌 [LEVEL 3: Design Patterns (Local Object & Class Relationships)](./lv3-design-pattern%20(Local%20Object%20n%20Class%20Relationships)/index.md)
We zoom inside a single service or package. We are looking at specific files, classes, interfaces, and methods.
- **Scope:** Subsystem or Module-wide. Typically involves fewer than 10 classes working together.
- **Key Question:** *"How do we design this specific subsystem so that we can easily plug in a new feature next week without modifying the existing code?"*
- **The Artifacts:** Class diagrams, object relationships, and interfaces.
- **Key Goals:** Reusability, flexibility, loose coupling, and adhering to SOLID design principles.
- **Examples (The "Gang of Four" standards):**
  - **Creational:** Singleton (ensuring only one logger object exists), Builder (building complex query objects step-by-step).
  - **Structural:** Adapter (translating XML data structure into JSON for a legacy API), Facade (hiding database complexity behind one clean manager class).
  - **Behavioral:** Observer (triggering automatic UI updates when database values change), Strategy (switching from stripe payment to paypal payment with an interchangeable class).

### 💻 [LEVEL 4: Idioms (Language-Specific Paradigms)](./lv4-idioms%20(Language-Specific%20Paradigms)/index.md)
We zoom in to the smallest possible detail: the syntax, syntax tricks, and best practices of a specific programming language.
- **Scope:** Localized. Usually restricted to a few lines of code or a single function.
- **Key Question:** *"What is the most elegant, performant, and standard way to write this specific logic in this specific language?"*
- **The Artifacts:** Raw source code.
- **Key Goals:** Code elegance, readability, memory safety, and matching native language paradigms.
- **Examples:**
  - **Python:** Context Managers (using the `with open() as file:` block to automatically close files) and List Comprehensions (`[x for x in list]`).
  - **Go (Golang):** The `defer` keyword to guarantee cleanup operations execute at the end of a function, or using goroutines with channels for concurrency.
  - **C++:** RAII (Resource Acquisition Is Initialization) to bind resource life cycles to object lifetimes, preventing memory leaks automatically.
  - **JavaScript/TypeScript:** Using Destructuring (`const { name, age } = user`) or Optional Chaining (`user?.address?.city`) to safely access nested object properties.

---

## Interview Questions & Answers

### Q1: How do choices at Level 1 (System Design) cascade down to affect Level 2, Level 3, and Level 4 decisions? Give a concrete example.
* **Answer:** Let's trace a high-level requirement: **"Handle a massive peak-load spike during a flash-sale event."**
  1. **Level 1 (System Design):** We opt for an asynchronous, decoupled topology. We add a highly scalable Message Queue (like Apache Kafka) as an entry buffer to absorb sudden bursts of traffic, shielding our databases from crashing.
  2. **Level 2 (Architectural Design):** To handle this Kafka message flow, we adopt an **Event-Driven Architecture (EDA)** pattern inside our codebase. We isolate the ingestion code from the ordering processing code using independent worker layers or microservices.
  3. **Level 3 (Design Patterns):** Inside our order processor component, we use the **Strategy Pattern** to dynamically route messages to different processing engines (e.g., standard orders vs. high-priority VIP customer orders) without altering the main consumer class loop.
  4. **Level 4 (Idioms):** Inside the Go order-processing worker, we launch multiple goroutines listening on a worker-pool channel and utilize `defer` to clean up database connection locks to avoid deadlock-induced starvation under extreme concurrency.

### Q2: What is the difference between a Design Pattern (Level 3) and an Idiom (Level 4)?
* **Answer:**
  - **Design Pattern:** A language-agnostic, conceptual blueprint for solving common structural or behavioral object-relationship problems. It can be implemented in almost any object-oriented language (e.g., implementing the *Observer Pattern* in Java, C#, or Python looks conceptually identical).
  - **Idiom:** A highly localized, language-specific syntax trick or best practice that is dictated by the design of that specific language. For instance, the `defer` statement is a Go idiom, while `try-with-resources` or RAII are the respective C# / C++ idiomatic equivalents for guaranteeing safe object cleanup. You cannot apply a Go idiom directly in C++.

### Q3: Why do SOLID principles map heavily to Level 3 (Design Patterns) but have limited direct relevance to Level 1 (System Design)?
* **Answer:**
  - **SOLID Principles** (Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion) are rules designed for managing compile-time dependencies, object-oriented subclass hierarchies, and class interfaces inside a single memory execution context.
  - **Level 1 System Design** deals with runtime networking, hardware, and physical separation. Concepts like network partitions, latency, serialization overhead, data consensus (Raft/Paxos), and replication lag dominate Level 1. You cannot use "Interface Segregation" or "Dependency Inversion" to solve a split-brain scenario in a partitioned database cluster.

### Q4: How do you handle Cross-Cutting Concerns (e.g., Security/Authentication) across each of these four design levels?
* **Answer:**
  - **Level 1 (System Design):** We enforce perimeter firewalls, setup VPC Peering, use CDN Edge policies, and place an API Gateway at the edge to reject unauthenticated requests before they reach internal microservice subnets.
  - **Level 2 (Architectural Design):** We structure the codebase around **Clean Architecture**, adding dedicated Middleware filters and interceptors at the entry controller layer of each service to validate JWT claims and forward normalized user context objects.
  - **Level 3 (Design Patterns):** We use the **Proxy Pattern** or **Decorator Pattern** to wrap core business service objects with security-checking layers dynamically, ensuring security logic is completely decoupled from business calculations.
  - **Level 4 (Idioms):** Inside our language of choice, we use standard syntax features (like TypeScript decorators `@Authorized()` or Python decorators `@requires_auth`) to safely declare security constraints on individual endpoint functions.
