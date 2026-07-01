# Go (Golang) Idioms

Core patterns and best practices native to the Go programming language:
1. **Explicit Error Returning:** Avoiding try-catch exception blocks entirely. Functions return values alongside an error interface tuple (`val, err := doSomething()`), forcing explicit, visible error-checking.
2. **Empty Interface (`interface{}` / `any`):** Allows variables to hold values of absolutely any underlying data type.
3. **Defer Statement:** Guarantees deferred cleanup actions (e.g., closing database connections, unlocking mutexes) run automatically at the end of the surrounding function block.

## Interview Questions & Answers

### Q1: Why does Go prefer explicit error returning over try-catch exception models?
- **Answer:** Readability and predictability. Exceptions create invisible exit paths in code, making control flows harder to trace. Go forces errors to be handled as first-class values immediately at the call site, eliminating hidden crashes and encouraging developers to write robust, self-documenting code.
