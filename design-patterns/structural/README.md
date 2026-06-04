# Structural Design Patterns

Comprehensive interview study guide covering Structural design patterns, focusing on class and object composition, adapters, decorators, facades, and proxies.

---

## 1. Meaning of Structural Patterns

**Structural Patterns** explain how to assemble objects and classes into larger structures while keeping these structures flexible and efficient. They utilize inheritance or composition to allow separate objects to collaborate seamlessly.

---

## 2. Core Structural Patterns

### 1. Adapter Pattern
* **Intent:** Allows objects with incompatible interfaces to collaborate. It acts as a translator.
* **Real-world Example:** Integrating a legacy logger that expects XML formatting into a modern service that outputs JSON. The `XMLToJSONAdapter` wraps the legacy class and translates inputs on-the-fly.

### 2. Decorator Pattern
* **Intent:** Attaches new behaviors to objects dynamically by placing these objects inside special wrapper classes.
* **Real-world Example:** Building a custom coffee order: `SimpleCoffee` can be dynamically wrapped with a `MilkDecorator`, then a `SugarDecorator`, each adding to the final price and behavior without modifying the base class.

### 3. Facade Pattern
* **Intent:** Provides a simplified, high-level interface to a complex set of classes, library, or subsystem.
* **Real-world Example:** A single `OrderFacade` that abstracts a highly complex order process involving checking inventory, authorizing payments, computing taxes, and notifying shipping. The client simply calls `orderFacade.placeOrder(id)`.

### 4. Proxy Pattern
* **Intent:** Provides a placeholder or surrogate for another object to control access to it (e.g., lazy loading, logging, access control, or caching).
* **Real-world Example:** A `CachedDatabaseProxy` that intercept database queries, returning cached results from Redis before forwarding heavy read requests to the real SQL database.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between the Adapter and Decorator pattern?
* **Answer:**
  * **Adapter** changes or translates the interface of an existing object to match a client's expectation, allowing incompatible systems to integrate. It does not add new behaviors.
  * **Decorator** preserves the original interface but **extends or adds new behaviors** to an object dynamically at runtime by wrapping it.

### Q2: How does the Proxy Pattern implement "Lazy Loading" in Hibernate or other ORMs?
* **Answer:** When an ORM loads a parent entity (e.g., `User`) containing a massive one-to-many relationship (e.g., `orders`), it doesn't query the orders table immediately (which would degrade query speeds). Instead, it instantiates a **Proxy** subclass representing the orders array. This proxy intercepts any read access (e.g., calling `user.getOrders()`). Only upon first call does the proxy run the actual SQL query to fetch orders from the database, caching them locally for future reads.

### Q3: Why is the Facade Pattern highly useful when migrating a monolithic system to microservices?
* **Answer:** Migrating to microservices can break frontend client integrations if clients must suddenly orchestrate calls across 10 separate backend microservices. By placing an **API Gateway (acting as a distributed Facade)** at the entry point, the frontend continues to call a single simplified endpoint (e.g., `/checkout`). The Gateway Facade handles the internal orchestration (calling payments, orders, and shipping microservices) on behalf of the client, isolating the client from the underlying architectural migration.
