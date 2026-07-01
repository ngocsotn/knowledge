# Registry Pattern

The Registry Pattern provides a centralized, well-known directory/map that stores and resolves objects, algorithms, or components.
- **Usage:** Typically maps unique string identifiers to active singleton service instances or dynamic algorithm strategies.

## Interview Questions & Answers

### Q1: How does a Strategy Router leverage the Registry Pattern?
- **Answer:** A Strategy Router stores interchangeable algorithm objects in a key-value registry map. When a request arrives (e.g., specifying a payment provider like `stripe`), the router looks up the key in its registry map and executes the matching payment class, avoiding long, unmaintainable nested `if-else` loops.
