# DRY, KISS, & YAGNI

Comprehensive interview study guide covering the three foundational pragmatism-driven software design principles, premature optimization, and refactoring trade-offs.

---

## 1. Principles Overview

| Principle | Meaning | Key Target | Anti-Pattern |
| :--- | :--- | :--- | :--- |
| **DRY** (Don't Repeat Yourself) | Every piece of knowledge must have a single, unambiguous representation within a system. | Redundancy & Copy-Paste | Duplicated logic across files |
| **KISS** (Keep It Simple, Stupid) | Systems work best if they are kept simple rather than made complicated. | Over-engineering | Complex abstractions for simple CRUD |
| **YAGNI** (You Aren't Gonna Need It) | Always implement things when you actually need them, never when you foresee needing them. | Speculative development | Writing generic pluggable DB adapters on Day 1 |

---

## 2. Core Concepts & Practical Trade-offs

### 1. DRY (The Cost of Duplication)
Duplicating code makes systems fragile. If a bug is found in copied code, you must locate and fix it in multiple places. However, **"Duplication is far cheaper than the wrong abstraction"** (Sandi Metz).
* **The Trap:** Dry-ing up code too early can tightly couple two unrelated business flows just because they share a similar data structure today, making future changes extremely difficult.

### 2. KISS (The Cost of Over-Engineering)
Many developers confuse "smart" with "complex." They write highly abstract, generic, or multi-layered classes containing design patterns that are unneeded for simple CRUD tasks.
* **The Fix:** Prioritize readability and simplicity. Write code that a junior developer can understand in 5 seconds. Avoid building dynamic factories or registry managers if a simple `switch` case handles the requirement perfectly.

### 3. YAGNI (Speculative Development)
YAGNI targets "just-in-case" coding: writing features, endpoints, databases, or generic configurations that you think you *might* need in six months.
* **The Reality:** Requirements change constantly. When you eventually need that feature, your speculative implementation will likely be obsolete or unfit, wasting engineering hours and bloating the codebase.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: Is it possible to over-apply the DRY (Don't Repeat Yourself) principle? Explain when duplication is preferred.
* **Answer:** Yes, over-applying DRY is a common architectural pitfall. If two completely separate domain contexts (e.g., the User registration form and the Admin profile editor) happen to share a few identical layout CSS properties or validation checks today, merging them into a single shared abstraction can couple them tightly. When the Product team requests a change *only* for the admin editor, you are forced to add complex, nested conditional flags inside the shared abstraction, making code hard to read. In such cases, **"accidental duplication"** is preferred over tight coupling.

### Q2: How do you strike the perfect balance between KISS and YAGNI when designing a system's initial architecture?
* **Answer:** The key is to **design for extension, but implement strictly what is needed today**:
  1. Define clean **interfaces** at boundaries, which allows swapping concrete implementations later without modifying core logic.
  2. Avoid building concrete "pluggable engines," microservice brokers, or complex horizontal scaling configurations until actual metrics or production traffic demand them.
  3. Write clean, modular, and readable code—this ensures that when you actually *do* need to extend the system later, refactoring is trivial and risk-free.

### Q3: What is "Premature Optimization" and why is it considered the root of all evil in programming?
* **Answer:** **Premature Optimization** is the act of optimizing code for performance before you have measured and proven that a bottleneck actually exists. This typically leads to highly complex, unreadable, and bug-prone code structures (e.g., writing manual bitwise operations or complex caching layers from Day 1). Developers should prioritize code readability, simplicity, and architectural correctness first. Once the system is running under realistic loads, use profiling tools to locate exact bottlenecks and optimize *only* those hot paths.
