# Behavioral Design Patterns

Comprehensive interview study guide covering Behavioral design patterns, focusing on object communication, algorithms, state management, and responsibilities.

---

## 1. Meaning of Behavioral Patterns

**Behavioral Patterns** are specifically concerned with algorithms and the assignment of responsibilities between objects. They describe not just patterns of objects or classes but also the patterns of communication between them.

---

## 2. Core Behavioral Patterns

### 1. Observer Pattern
* **Intent:** Defines a one-to-many dependency between objects so that when one object (the **Subject**) changes state, all its dependents (the **Observers**) are notified and updated automatically.
* **Real-world Example:** Event-driven message brokers, newsletter subscriptions, or custom reactive UI frameworks (like React's state effect hooks).

### 2. Strategy Pattern
* **Intent:** Defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.
* **Real-world Example:** A shopping cart system that accepts interchangeable shipping methods (e.g., `FedexStrategy`, `UPSStrategy`, `DHLStrategy`) dynamically without modifying the shopping cart calculation.

### 3. Command Pattern
* **Intent:** Encapsulates a request as an independent object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.
* **Real-world Example:** An Undo/Redo editor system where every text insertion or deletion is saved as a discrete `Command` object with an execute and rollback method.

### 4. State Pattern
* **Intent:** Allows an object to alter its behavior when its internal state changes. The object will appear to change its class.
* **Real-world Example:** A document publication workflow: a `Document` can transit across `Draft`, `InReview`, and `Published` states. Calling `.publish()` behaves completely differently depending on the active state.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: How do you prevent memory leaks when implementing the Observer Pattern?
* **Answer:** Memory leaks occur in the Observer pattern because the **Subject maintains a hard reference to all registered Observers** in its collection. If an observer goes out of scope but fails to explicitly unsubscribe, the garbage collector cannot reclaim it because the Subject is still holding its reference (the "lapsed listener" problem). To prevent this, always implement an explicit `unsubscribe()` method that removes the observer, or use **Weak References** (e.g., `WeakMap` in JavaScript) to store observers so they can be garbage collected when no other references exist.

### Q2: What is the difference between the State and Strategy pattern?
* **Answer:** While both patterns utilize composition and delegate work to subclasses, their intents differ:
  * **Strategy** is usually chosen and set once by the client upon initialization (e.g., choosing "PayPal" as the payment method). The strategies are completely independent and unaware of each other.
  * **State** allows behaviors to transition dynamically over time. The individual concrete State classes often trigger state transitions (e.g., `Draft` state explicitly changing the context to `InReview` after approval), making them tightly coupled to the system's lifecycle flow.

### Q3: When should you use the Chain of Responsibility Pattern?
* **Answer:** Use the Chain of Responsibility pattern when you need to process a request through a sequential **pipeline of independent handlers**, where each handler decides either to process the request or pass it to the next link in the chain. A classic example is **HTTP Middleware**: an incoming request passes through an `IPFilterMiddleware`, then an `AuthMiddleware`, then a `RateLimiterMiddleware`. If any middleware fails validation, it breaks the chain and returns a response, preventing downstream handlers from running.
