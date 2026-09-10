# Next.js

React renders components. It has no opinion about routing, where the code runs, or when data is fetched. Next.js answers all three — and almost every hard Next.js question, in production and in interviews, turns out to be a question about *caching* or about *where this code runs*.

## What Next.js Actually Adds

| Concern | What Next.js provides |
| :--- | :--- |
| Routing | File-system based — a folder becomes a URL |
| Server rendering | SSR, static generation, streaming, all built in |
| Data | Server Components fetch directly; no API layer required |
| Mutations | Server Actions — call server code from a form or a handler |
| Bundling | Configured build pipeline, code splitting per route |
| Optimization | Image, font, and script components with sensible defaults |
| Caching | Several layers, with defaults that have changed across versions |

The trade you are making is real and worth naming: you gain an enormous amount of infrastructure for free, and you accept the framework's opinions about structure, its build system, and a deployment target that has to be able to run server code.

These guides cover the **App Router**. Read them in order if Next.js is new to you; each page stands alone and ends with its own interview questions.

## Subcategories

- [Routing & Layouts](./routing-and-layouts.md) — App Router vs Pages Router, file conventions, nested layouts
- [Rendering & Data](./rendering-and-data.md) — where code runs, static vs dynamic, streaming, bundle discipline
- [Caching](./caching.md) — the four layers, the two models, and why your data is stale
- [Mutations](./mutations.md) — Server Actions, Route Handlers, and the security boundary
- [Proxy](./proxy.md) — request-level middleware, and why auth does not belong there

---

## Sources and Version Notes

Written **September 2026**, against **Next.js 16.3** (16.0 released October 2025). Next.js changes faster than anything else in this knowledge base, and **the caching defaults in particular have changed across major versions** — a great deal of material still online describes the pre-v15 "cached by default" behaviour, which is wrong for current versions. Verify specifics against [nextjs.org/docs](https://nextjs.org/docs) for your installed version.

Version-sensitive claims made in these guides, worth re-checking:

- Turbopack is the default bundler for both `next dev` and `next build` as of v16; `--webpack` opts out.
- Async request APIs (`await cookies()`, `await params`) are mandatory in v16; the synchronous compatibility path was removed.
- `proxy.ts` replaced `middleware.ts` in v16 and runs on the Node.js runtime.
- Cache Components (`cacheComponents: true`) is stable but opt-in, and subsumes what were previously the separate `ppr`, `dynamicIO`, and `useCache` experimental flags.
- `next lint` was removed in v16; run ESLint or Biome directly.

If you see an article referring to Next.js 17, be sceptical — as of this writing 16.3.x is the current line.

---

## Related Guides

- [React](../react/index.md)
- [Web Rendering Patterns](../rendering-patterns/index.md)
- [Frontend Performance Optimization](../performance/index.md)
- [Frontend Implementation Struggles](../implementation-struggles/index.md)
- [Web Frontend Security](../security/index.md)
- [Frontend State Management](../state-management/index.md)
