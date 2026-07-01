# Dependency Injection (DI)

Dependency Injection is a creational pattern where an object receives its dependencies from external coordinators, rather than constructing them internally.
- **Approach:** Dependencies are defined as interfaces, injected via constructors or parameters, decoupling the consumer from concrete implementations.

## Interview Questions & Answers

### Q1: What is the difference between Dependency Injection and a Service Locator?
- **Answer:** **Dependency Injection** is a push model—the container actively injects dependencies into the object constructor upon creation. **Service Locator** is a pull model—the object explicitly queries a global registry to resolve its dependencies. DI is preferred because it avoids global state dependencies, keeping classes easier to isolate and unit test.
