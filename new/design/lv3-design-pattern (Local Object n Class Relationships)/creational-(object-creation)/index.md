# Creational Design Patterns: Singleton, Factory, and Builder

**Creational Patterns** abstract the object instantiation process. They help make a system independent of how its objects are created, composed, and represented. Instead of directly instantiating objects using hardcoded `new` operators, these patterns delegate instantiation logic to specialized abstractions.

---

## 2. Core Creational Patterns

### 1. Singleton Pattern
* **Intent:** Guarantees that a class has **only one instance** and provides a global access point to it.
* **Real-world Example:** A central Database Connection Pool or an Application Logger.
* **Trade-off:** Often considered an anti-pattern if overused because it introduces global state, makes unit testing difficult (due to shared state across tests), and violates the Single Responsibility Principle.

### 2. Factory Method Pattern
* **Intent:** Defines an interface for creating an object but lets subclasses decide which class to instantiate.
* **Real-world Example:** A payment processing system where a `PaymentProcessorFactory` returns different instances (e.g., `StripeProcessor`, `PaypalProcessor`) dynamically based on an input string.

### 3. Builder Pattern
* **Intent:** Separates the construction of a complex object from its representation, allowing the same construction process to create different representations.
* **Real-world Example:** Constructing an HTTP client request payload with many optional headers, parameters, and body values using a fluent method chaining API:
  `Request.builder().setURL("api/").addHeader("Auth").setBody(payload).build()`

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: Why is the Singleton Pattern often considered an anti-pattern, and how do you write a thread-safe Singleton in Java/Go?
* **Answer:** Singleton is considered an anti-pattern because it introduces **global state** into your application. This tightly couples classes to the global instance, hiding dependencies and making unit tests brittle because state can leak between individual test cases.
* **Thread-safe implementation:** To prevent race conditions in multi-threaded environments, you must use proper synchronization:
  * In **Go**, utilize `sync.Once` which guarantees a function executes exactly once:
    ```go
    var instance *Database
    var once sync.Once
    func GetInstance() *Database {
        once.Do(func() {
            instance = &Database{}
        })
        return instance
    }
    ```
  * In **Java**, use Double-Checked Locking with a `volatile` instance variable to prevent JVM instruction reordering issues.

### Q2: What is the difference between the Factory Method and the Abstract Factory pattern?
* **Answer:**
  * **Factory Method** uses inheritance to let subclasses decide which concrete product class to instantiate. It exposes a single creation method (e.g., `createButton()`).
  * **Abstract Factory** uses composition. It defines an interface for creating a **family of related or dependent objects** without specifying their concrete classes. For example, a `GUIFactory` defines methods to create *both* buttons AND scrollbars (`createButton()`, `createScrollbar()`). A concrete implementation like `MacGUIFactory` will return `MacButton` and `MacScrollbar`, ensuring visual consistency.

### Q3: When is the Builder Pattern preferred over standard class constructors or setters?
* **Answer:** The Builder pattern is preferred when:
  1. An object requires **many optional parameters** to initialize (avoiding "telescoping constructors" like `User(id, name, null, null, null, true)`).
  2. The object must be **immutable** after creation (setters make objects mutable, whereas a builder gathers parameters in temporary memory and returns a read-only, final object upon calling `.build()`).
  3. Construction requires complex validation or ordering of parameters.

## Interview Questions & Answers

### Q1: Why is the Singleton pattern considered an anti-pattern by many developers?
- **Answer:** Hard to test and tight coupling. Singletons introduce global state into an application. Testing code that depends on a Singleton is extremely difficult because you cannot easily mock the instance or reset its state between isolated test runs. Pair with Dependency Injection (DI) instead to manage single-instance lifecycles.

### Q2: What is the main benefit of the Builder pattern?
- **Answer:** Step-by-step construction. The Builder pattern isolates the construction of a complex object from its representation, allowing you to enforce validation rules, set default attributes, and construct representations of an object (e.g., using method chaining).
