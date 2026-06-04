# SOLID Design Principles

Comprehensive interview study guide covering the five SOLID object-oriented design principles with clear, real-world examples and refactoring guidelines.

---

## 1. SOLID Overview

| Principle | Meaning | Key Goal |
| :--- | :--- | :--- |
| **S** - Single Responsibility | A class should have only **one reason to change**. | High Cohesion |
| **O** - Open/Closed | Software entities should be **open for extension, closed for modification**. | Refactor-free extension |
| **L** - Liskov Substitution | Subtypes must be completely **substitutable for their base types** without breaking correctness. | Reliable polymorphism |
| **I** - Interface Segregation | Clients should not be forced to depend on **interfaces they do not use**. | Lean, focused contracts |
| **D** - Dependency Inversion | Depend on **abstractions, not concretions**. | Decoupling through interfaces |

---

## 2. Deep Dive & Examples

### 1. Single Responsibility Principle (SRP)
* **Violation:** A `User` class containing business properties AND database storage queries AND email notification methods. If the database schema changes OR email API provider changes, this class must modify.
* **Refactoring:** Split into three classes: `User` (holds entity data), `UserRepository` (handles DB storage), and `EmailService` (handles notifications).

### 2. Open/Closed Principle (OCP)
* **Violation:** An `AreaCalculator` class containing `if/else` checks for every shape type (e.g., `if shape == 'circle' ... else if shape == 'rectangle'`). Adding a new shape forces modifying this core calculator class.
* **Refactoring:** Create a `Shape` interface with a `getArea()` method. Every shape class (e.g., `Circle`, `Rectangle`) implements `Shape`. The `AreaCalculator` simply accepts a list of `Shape` and loops through them safely, open for new shapes without code modification.

### 3. Liskov Substitution Principle (LSP)
* **Violation:** A `Square` class inheriting from a `Rectangle` class. If the parent class has `setWidth()` and `setHeight()` set independently, a `Square` subtype will break the client's expectations (since setting width dynamically changes height in squares), violating LSP.
* **Refactoring:** If a subtype breaks parent assumptions, **do not use inheritance**. Create a shared `Shape` interface instead, or keep them decoupled entirely.

### 4. Interface Segregation Principle (ISP)
* **Violation:** A giant `Worker` interface containing `work()` and `eat()` methods. A `Robot` class implementing `Worker` is forced to implement `eat()` with empty dummy logic because robots do not eat.
* **Refactoring:** Segregate the interface into smaller, focused interfaces: `Workable` with `work()`, and `Feedable` with `eat()`. Humany workers implement both, while Robots implement only `Workable`.

### 5. Dependency Inversion Principle (DIP)
* **Violation:** A high-level `OrderService` class directly instantiating a concrete low-level `MySQLDatabase` class inside its constructor (`this.db = new MySQLDatabase()`). If you switch to MongoDB, you must rewrite the high-level business service.
* **Refactoring:** The `OrderService` should depend on a `Database` interface. The concrete `MySQLDatabase` implements `Database`. You inject the interface instance into `OrderService`'s constructor (Dependency Injection).

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: How does Dependency Inversion (DIP) differ from Dependency Injection (DI)?
* **Answer:** **Dependency Inversion** is an abstract **design principle** (the "D" in SOLID) which dictates that high-level modules should not depend on low-level modules; both should depend on abstractions (interfaces). **Dependency Injection (DI)** is a concrete **implementation technique** used to realize DIP. It is the process of passing dependencies (concretions) into a class from the outside (typically via constructor or setter) rather than letting the class instantiate them internally.

### Q2: What is a clear sign that code is violating Liskov Substitution Principle (LSP)?
* **Answer:** A classic warning sign of an LSP violation is the presence of **type checking checks** or **empty method overrides** inside subtypes. For example, if a client receives a list of base objects and has to execute code like `if (child instanceof Bird && !(child instanceof Penguin))` or if a penguin subclass throws `UnsupportedOperationException` for parent methods like `fly()`, it indicates the inheritance structure is broken. Subclasses must be completely substitutable for their parent classes without clients knowing or altering behavior.

### Q3: Why is violating OCP (Open/Closed Principle) highly dangerous in large codebases?
* **Answer:** When code violates OCP, adding a new feature forces you to modify existing, thoroughly tested, and deployed source code. This introduces high risk: you can easily break existing functionalities, invalidate unit tests, and cause regressions. Adhering to OCP allows you to write *new* code in separate files (e.g., implementing an interface) without touching stable core classes, keeping deployments and builds safe.
