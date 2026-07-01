# Python Idioms

Best practices and dynamic syntactic structures native to Python:
1. **Duck Typing:** Dynamic typing paradigm where object type checks are ignored; instead, the presence of specific methods/attributes at runtime dictates compatibility ("If it walks and quacks like a duck, it's a duck").
2. **Decorators:** Dynamic function wrappers that intercept and cleanly modify the behavior of a function or method without altering its source code.
3. **Generators (`yield`):** Functions that return lazy iterators producing values one-at-a-time on-the-fly, saving massive system memory when processing heavy lists.

## Interview Questions & Answers

### Q1: How does a Python Generator save system memory compared to a standard List?
- **Answer:** A standard List stores all elements in-memory (RAM) simultaneously. A Generator function uses the `yield` keyword to produce only one element at a time upon request (lazy evaluation). The execution state is suspended and resumed on-the-fly, keeping memory consumption flat ($O(1)$) even when iterating over millions of records.
