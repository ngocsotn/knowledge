# Composite Pattern

Composite compose objects into tree structures to represent part-whole hierarchies, allowing clients to treat individual objects and compositions of objects uniformly.
- **Usage:** File structures (file vs. directory), UI rendering trees (DOM element containing nested children).

## Interview Questions & Answers

### Q1: How does the Composite pattern simplify client code?
- **Answer:** Polymorphism. The client calls a single method (e.g., `render()`) on the component interface. If the target is a leaf node, it renders itself. If it's a composite, it automatically iterates and executes `render()` on all its children recursively, removing the need for client-side type-checking loops.
