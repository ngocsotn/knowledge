# Language-Specific Paradigms

Study guides covering runtime models, execution engines, and idioms of specific languages.

## Subcategories

- [Node.js & JavaScript Fundamentals](./nodejs/index.md)
  - [JavaScript Language Internals](./nodejs/javascript/index.md)
  - [TypeScript Type System](./nodejs/typescript/index.md)
- [.NET Platform: Runtime, Compilation, Memory & Ecosystem](./dotnet/index.md)
  - [C# Language Deep Dive (C# 5 to C# 14)](./dotnet/csharp/index.md)
  - [ASP.NET: From 4.5 to Modern ASP.NET Core](./dotnet/aspnet/index.md)
    - [Classic ASP.NET on .NET Framework (4.5 to 4.8)](./dotnet/aspnet/aspnet-framework/index.md)
    - [Modern ASP.NET Core (.NET Core 1.0 to .NET 10)](./dotnet/aspnet/aspnet-core/index.md)
  - [Data Access in .NET: ADO.NET, Dapper, EF6 & EF Core](./dotnet/data-access/index.md)

## Node.js vs .NET at a Glance

Both are popular choices for backend services. They share a lot (async I/O, package managers, JIT-compiled runtimes) but differ in how they run code in parallel.

| Topic | Node.js | .NET |
| :--- | :--- | :--- |
| Runtime | V8 JavaScript engine + libuv | CLR (CoreCLR) with a JIT compiler, optional Native AOT |
| Main languages | JavaScript, TypeScript | C# (also F#, VB.NET) |
| Typing | Dynamic (TypeScript adds compile-time types that are erased) | Static, and types exist at run time (reified generics) |
| Concurrency | One event loop thread + a small helper thread pool | Multi-threaded thread pool + `async`/`await` |
| CPU-heavy work | Blocks the event loop unless moved to worker threads | Runs in parallel across all cores |
| Web frameworks | Express, NestJS, Fastify | ASP.NET Core (Controllers, Minimal APIs) |
| Package manager | npm / pnpm / yarn | NuGet |

**Shared concepts:** `async`/`await` (promises in Node.js, tasks in .NET), middleware pipelines, dependency injection (NestJS in Node.js, built in with ASP.NET Core) and garbage-collected memory.
