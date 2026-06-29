# System & Software Design

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


1. example: Model-View-Controller (MVC), Microservices, or Clean Architecture or Hexagon....

2. example: Monolith, MVC, Layers...

3. example: Singleton, Strategy, Observer...

4. These are specific tricks unique to a programming language (e.g., using defer in Go, or using blocks in C#)

===============

☁️ LEVEL 1: System Design (System Topology & Infrastructure)
At this level, you do not care about the code itself. You care about how massive physical or virtual components are organized across the internet and data centers.

Scope: Global. The entire network, hardware, and physical deployment topology.

Key Question: "How do we reliably serve 100 million users with high availability, low latency, and fault tolerance?"

The Artifacts: Network topology diagrams, cloud architecture maps, data flow pipelines, failover strategies.

Key Goals: Scalability, reliability, security, cost efficiency, and physical performance.

Examples:

Load Balancing & CDNs: Distributing incoming web traffic across multiple servers (NGINX, AWS CloudFront).

Database Scaling: Master-Slave Replication, Sharding (splitting a database horizontally), and Caching (Redis/Memcached).

Data Partitioning Laws: Applying the CAP Theorem and PACELC Theorem to choose between data consistency or fast system response during a network partition.

🏛️ LEVEL 2: Architectural Design (Software Architecture)
We now zoom past the cloud infrastructure and enter the codebase. This level defines the structural rules, constraints, and organizational philosophies of your software.

Scope: Application-wide. The high-level blueprint of the codebase, directories, and internal boundaries.

Key Question: "How do we partition our business logic so that 50 developers can write code simultaneously without stepping on each other's toes?"

The Artifacts: Component diagrams, package structures, directory layouts, dependency rules.

Key Goals: Modularity, ease of testing, developer velocity, clear boundaries, and maintainability.

Examples:

Architectural Styles: Microservices (splitting business capabilities into independent running programs) vs. Monolith (all code in one package).

Code Organization Patterns: MVC (Model-View-Controller), Layered (N-Tier) Architecture, and Clean/Hexagonal Architecture (isolating core business logic from database and UI frameworks).

Integration Patterns: Event-Driven Architecture (EDA) (services talking asynchronously via a message broker) and Domain-Driven Design (DDD) (modeling code strictly around business domain language).

🔌 LEVEL 3: Design Patterns (Local Object & Class Relationships)
We zoom inside a single service or package. We are looking at specific files, classes, interfaces, and methods.

Scope: Subsystem or Module-wide. Typically involves fewer than 10 classes working together.

Key Question: "How do we design this specific subsystem so that we can easily plug in a new feature next week without modifying the existing code?"

The Artifacts: Class diagrams, object relationships, and interfaces.

Key Goals: Reusability, flexibility, loose coupling, and adhering to SOLID design principles.

Examples (The "Gang of Four" standards):

Creational: Singleton (ensuring only one logger object exists), Builder (building complex query objects step-by-step).

Structural: Adapter (translating XML data structure into JSON for a legacy API), Facade (hiding database complexity behind one clean manager class).

Behavioral: Observer (triggering automatic UI updates when database values change), Strategy (switching from stripe payment to paypal payment with an interchangeable class).

💻 LEVEL 4: Idioms (Language-Specific Implementations)
We zoom in to the smallest possible detail: the syntax, syntax tricks, and best practices of a specific programming language.

Scope: Localized. Usually restricted to a few lines of code or a single function.

Key Question: "What is the most elegant, performant, and standard way to write this specific logic in this specific language?"

The Artifacts: Raw source code.

Key Goals: Code elegance, readability, memory safety, and matching native language paradigms.

Examples:

Python: Context Managers (using the with open() as file: block to automatically close files) and List Comprehensions ([x for x in list]).

Go (Golang): The defer keyword to guarantee cleanup operations execute at the end of a function, or using goroutines with channels for concurrency.

C++: RAII (Resource Acquisition Is Initialization) to bind resource life cycles to object lifetimes, preventing memory leaks automatically.

JavaScript: Using Destructuring (const { name, age } = user) or Optional Chaining (user?.address?.city) to safely access nested object properties.

## Subcategories

- [System Design Interview Scenarios](./interview-scenarios/index.md)
- [Level 1: System Design (Infrastructure & Topology)](./lv1-system-design-(infrastructure-n-topology)/index.md)
- [Level 2: Architectural Design (Software Architecture)](./lv2-architectural-design-(software-architecture)/index.md)
- [Level 3: Design Patterns (Local Object & Class Relationships)](./lv3-design-pattern (Local Object n Class Relationships)/index.md)
- [Level 4: Idioms (Language-Specific Paradigms)](./lv4-idioms (Language-Specific Paradigms)/index.md)
