# Prototype Pattern

The Prototype Pattern creates new object instances by copying or cloning an existing, fully-configured "prototype" instance.
- **Usage:** Creating complex documents, game characters, or network connection states where cloning an existing object is significantly faster than executing expensive constructor logic from scratch.

## Interview Questions & Answers

### Q1: What is the difference between a Shallow Copy and a Deep Copy in the Prototype Pattern?
- **Answer:** A **Shallow Copy** duplicates the top-level object structure, but references to nested objects (arrays, classes) are shared between the original and the clone. A **Deep Copy** recursively duplicates all nested objects, guaranteeing that modifying any attribute of the clone has zero effect on the original state.
