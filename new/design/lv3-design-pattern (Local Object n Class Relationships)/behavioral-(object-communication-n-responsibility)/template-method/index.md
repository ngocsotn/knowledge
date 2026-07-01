# Template Method Pattern

Template Method defines the skeleton of an algorithm in a method, deferring some steps to subclasses.
- **Mechanism:** Subclasses redefine specific steps of an algorithm without changing its overall structure (the Hollywood Principle: "Don't call us, we'll call you").

## Interview Questions & Answers

### Q1: What is the difference between Strategy and Template Method?
- **Answer:** **Template Method** uses class inheritance to vary parts of an algorithm (subclasses override specific hook methods of a parent class). **Strategy** uses object composition to switch the entire algorithm dynamically at runtime (client delegates the task to a strategy interface).
