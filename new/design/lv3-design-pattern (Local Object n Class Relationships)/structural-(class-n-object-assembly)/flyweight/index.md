# Flyweight Pattern

Flyweight minimizes memory usage by sharing as much data as possible with other similar objects.
- **Approach:** Splits object state into:
  - **Intrinsic State:** Constant, context-independent data shared globally (cached).
  - **Extrinsic State:** Dynamic, context-dependent data passed to the flyweight method externally.

## Interview Questions & Answers

### Q1: How does the Flyweight pattern optimize memory in a massive simulation (e.g., a forest of 1 million trees)?
- **Answer:** Instead of creating 1 million independent tree objects (each containing heavy meshes, leaves, textures, and coordinates), the Flyweight pattern creates a single shared `TreeType` flyweight containing the heavy meshes and textures (Intrinsic state). The 1 million trees are represented as tiny coordinate-only objects (Extrinsic state) that reference the shared flyweight, reducing memory footprint by up to 99%.
