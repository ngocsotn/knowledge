# React

Most React confusion is not about syntax. It is about not knowing *when React runs your function again*, and what your variables mean when it does. These guides build that model from the ground up, then go deep enough for a senior interview.

Read them in order if React is new to you. If you are revising, each page stands alone and ends with its own interview questions.

## Subcategories

- [The Rendering Model](./rendering-model.md) — render vs commit, and the three things that cause a re-render
- [State & Hooks](./state-and-hooks.md) — state as a snapshot, the updater form, and why the rules of hooks exist
- [Effects & Refs](./effects-and-refs.md) — what an effect is actually for, and the effects you should delete
- [Performance & Concurrency](./performance.md) — memoization, the React Compiler, transitions, Suspense
- [Actions & Data APIs](./actions-and-data.md) — `useActionState`, `useOptimistic`, `use()`
- [Server Components](./server-components.md) — two places your code can run, and the boundary between them

---

## Sources and Version Notes

Written **September 2026**, against **React 19.3** (the 19.x line began December 2024; there is no React 20). React's core model — render/commit, hook ordering, state as a snapshot, effects as synchronization — has been stable for years and is the durable part of these guides.

Version-sensitive points worth re-checking against [react.dev](https://react.dev):

- The React Compiler reached 1.0 but remains an opt-in Babel plugin, off by default in frameworks that integrate it.
- `forwardRef` still works and has *not* been removed; ref-as-prop is the recommended approach for new code.
- Suspense reveal behaviour has changed within the 19.x line — siblings are pre-warmed, and streaming SSR batches boundary reveals — so React 18-era mental models about fallbacks are not reliable.
- Server Component APIs are supplied by your framework, not by React directly, so their exact shape depends on the framework version.

Two React Server Components security advisories were issued in December 2025 affecting Flight deserialization; if you maintain an RSC application, confirm you are on a patched version.

---

## Related Guides

- [The DOM & Virtual DOM](../dom-vdom/index.md)
- [Frontend State Management](../state-management/index.md)
- [Web Rendering Patterns](../rendering-patterns/index.md)
- [Next.js](../nextjs/index.md)
- [Frontend Performance Optimization](../performance/index.md)
- [Frontend Implementation Struggles](../implementation-struggles/index.md)
- [Advanced JavaScript Language Core](../../specific-language/nodejs/javascript/index.md)
