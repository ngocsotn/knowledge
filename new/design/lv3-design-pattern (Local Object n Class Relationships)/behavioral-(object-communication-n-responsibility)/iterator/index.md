# Iterator Pattern

Iterator provides a standard way to access the elements of an aggregate object sequentially without exposing its underlying representation (e.g., list, tree, graph).
- **Core Role:** Exposes methods like `hasNext()` and `next()` to navigate collections uniformly.

## Interview Questions & Answers

### Q1: Why is the Iterator pattern highly useful for custom graph structures?
- **Answer:** Decoupling traversal logic. A graph can be traversed in multiple ways (Breadth-First Search, Depth-First Search). Instead of polluting the `Graph` class with multiple traversal methods, you write separate `BFSIterator` and `DFSIterator` classes, allowing clients to traverse the identical graph structure using clean, uniform interfaces.
