# Chain of Responsibility Pattern

Chain of Responsibility avoids coupling the sender of a request to its receiver by giving more than one object a chance to handle the request.
- **Mechanism:** Links receiving objects in a chain. Each handler receives the request, and either handles it or forwards it to the next link in the chain.

## Interview Questions & Answers

### Q1: Give a real-world example of Chain of Responsibility in web development.
- **Answer:** HTTP Middleware Pipelines. In web frameworks (such as Go's `net/http` middleware chain or Express in Node), incoming HTTP requests traverse a chain of middlewares (e.g., Request Logging -> Authentication -> Rate Limiting -> Route Handler). Each middleware checks the request, either aborts/handles it, or calls `next()` to pass it downstream.
