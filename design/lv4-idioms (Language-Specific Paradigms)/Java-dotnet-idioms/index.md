# Java & .NET (C#) Idioms

Core patterns native to enterprise compiled managed languages:
1. **Properties / Getters and Setters:** Encapsulating class attributes inside public accessor methods to protect internal state mutation boundaries.
2. **Dispose Pattern (try-with-resources / `using`):** Guarantees instant release of unmanaged OS resources (like file streams, network sockets, database connections) as soon as the execution block exits.
3. **LINQ (Language Integrated Query):** Syntactic query engine built directly into C# to cleanly filter, map, and transform local data collections or remote databases.

## Interview Questions & Answers

### Q1: Why is the Dispose Pattern critical for unmanaged system resources?
- **Answer:** Managed garbage collectors (GC) only track and clean up managed heap memory (RAM). They do not track unmanaged OS handles (file descriptors, sockets). Failing to explicitly dispose of these resources immediately after use causes "Resource Leaking," which quickly exhausts available system file descriptors and crashes the operating system.
