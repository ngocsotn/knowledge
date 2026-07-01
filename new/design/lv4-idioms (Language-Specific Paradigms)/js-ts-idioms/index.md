# JavaScript & TypeScript Idioms

Syntax tricks and patterns native to the web programming ecosystem:
1. **IIFE (Immediately Invoked Function Expression):** Executing a JS function immediately upon definition to insulate variables and prevent polluting the global window scopes.
2. **Closures:** A function retaining access to variables inside its outer lexical scope, even after the outer function has fully completed execution.
3. **Currying:** Transforming a function taking multiple parameters into a chain of nested single-argument functions (e.g., `f(a, b)` into `f(a)(b)`), enabling advanced partial application structures.

## Interview Questions & Answers

### Q1: What is a common cause of memory leaks when utilizing JavaScript Closures?
- **Answer:** Accidental reference retention. Because a closure retains references to its outer scope variables, if the closure function itself remains alive in-memory (e.g., attached to a global event listener), any heavy objects stored in its outer lexical scope cannot be garbage collected, leading to creeping memory leaks.
