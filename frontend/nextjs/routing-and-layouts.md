# Next.js: Routing & Layouts

Next.js has two routing systems and you will be asked about both. This guide covers the difference, the file conventions of the App Router, and the one property of nested layouts that makes navigation feel different from a traditional web app.

---

## 1. Two Routers, One Framework

**Pages Router** (`pages/`) is the original. Every page is a Client Component, and you fetch data by exporting `getServerSideProps` (per request) or `getStaticProps` (at build). It is simple, mature, and well understood.

**App Router** (`app/`) is the modern one, built on React Server Components. Components are server-side by default, layouts nest, data is fetched inline with `async`/`await`, and rendering can stream.

```
  Pages Router                          App Router
  ────────────                          ──────────
  pages/blog/[id].tsx                   app/blog/[id]/page.tsx
  export getServerSideProps             export default async function Page()
  every component ships to client       server by default, opt in with 'use client'
  one layout via _app.tsx               nested layouts, preserved across navigation
  full page transitions                 streaming + partial updates
```

The honest interview answer about which to use: **App Router for new projects**, because that is where the framework's investment goes and it is what the React model now assumes. Pages Router is still supported and maintained with no announced removal timeline, and the two can coexist in one application — so a gradual migration route by route is the normal path, not a big-bang rewrite.

Be precise about the status. Pages Router is not deprecated; it simply does not receive the newer features, and some of them are App Router only.

---

## 2. File-System Routing

A folder is a route segment. A `page` file makes that segment publicly reachable. Special filenames have specific jobs:

```
app/
├── layout.tsx          ← wraps everything; root layout is required
├── page.tsx            ← /
├── loading.tsx         ← shown while this segment loads (a Suspense boundary)
├── error.tsx           ← catches errors below it (must be a Client Component)
├── not-found.tsx       ← 404 UI
├── blog/
│   ├── layout.tsx      ← wraps every /blog/* route
│   ├── page.tsx        ← /blog
│   └── [slug]/
│       └── page.tsx    ← /blog/hello-world
└── api/
    └── health/
        └── route.ts    ← an HTTP endpoint, not a page
```

Other pieces of the vocabulary:

- `[slug]` — a dynamic segment. `[...slug]` catches all; `[[...slug]]` catches all optionally.
- `(marketing)` — a **route group**: organizes files without adding a URL segment.
- `_components` — a folder prefixed with an underscore is private and never routed.
- `template.tsx` — like a layout, but it remounts on navigation instead of persisting.
- `@modal` — a **parallel route** slot, rendered alongside the page. Each slot now requires a `default.tsx` for the case where it has no matching state.

---

## 3. Layouts Do Not Re-Render on Navigation

This is the property that matters most, and the one that changes how you structure an app.

Navigating from `/blog/a` to `/blog/b` re-renders the page but **preserves the layout** — including any state inside it, scroll position in a sidebar, an open dropdown, and a playing video. That is a genuine advantage over full page transitions, and it is why the App Router feels closer to an SPA than to a traditional server-rendered site.

It also explains a constraint people trip over: **`layout.tsx` cannot receive the dynamic params of the routes below it**, because it is not re-rendered when those params change. If you need the current slug, read it in the page, or use a client hook.

---

## 4. Error and Loading Boundaries

Two file conventions worth understanding as React concepts rather than magic filenames:

- **`loading.tsx`** is a `<Suspense>` boundary for that route segment. Next.js wraps the page in it automatically. That is all it is.
- **`error.tsx`** is an error boundary for everything below it, and it must be a Client Component because error boundaries need state. It receives the error and a `reset` function. A separate `global-error.tsx` catches failures in the root layout itself, which `error.tsx` cannot.

Newer versions also expose a `catchError` utility for custom error boundaries with a retry that re-fetches Server Components, and it deliberately does not swallow the framework's own `notFound()` and `redirect()` signals.

---

## 5. Common Mistakes

- **Expecting a layout to re-render** when a dynamic param changes. It does not, by design.
- **Reaching for `template.tsx`** when you actually wanted a `key` on a component inside the layout.
- **Forgetting `default.tsx`** on a parallel route slot, which now fails rather than rendering nothing.
- **Putting `error.tsx` at the root** and expecting it to catch root layout errors. That is what `global-error.tsx` is for.
- **Assuming Pages Router is deprecated.** It is maintained; it just does not get new features.

---

## 6. Interview Questions & High-Impact Answers

### Q1: What does Next.js give you that React does not?
- **Answer:** Routing, a server rendering pipeline with static generation and streaming, a data-fetching story that does not need a separate API layer, mutations through Server Actions, a configured build with per-route code splitting, and image and font optimization. React is a rendering library with no opinion about where code runs or when data is fetched; Next.js answers exactly those questions. The cost is a framework-shaped project structure and a deployment target that can run server code.

### Q2: App Router or Pages Router for a new project, and why?
- **Answer:** App Router, because it is where the framework's development is focused and it is built on the React Server Components model that React itself now assumes. Practically it gives nested layouts that persist across navigation, streaming with Suspense, server-side data fetching without an API layer, and a smaller client bundle. Pages Router is not deprecated and has no removal timeline, so for an existing app I would migrate incrementally rather than rewrite — the two run side by side, route by route.

### Q3: What is the difference between a layout and a template?
- **Answer:** A layout persists across navigation within its segment — it does not re-render, so state inside it survives, which is what makes sidebar scroll position and open menus stick. A template remounts on every navigation, giving fresh state and re-running effects each time. Layout is the default and the one you want almost always; template is for the narrow case where you deliberately need per-navigation state or an enter animation on every visit.

### Q4: Why can't a layout read the dynamic params of the page below it?
- **Answer:** Because layouts are not re-rendered when those params change — that persistence is the whole feature. If a layout could depend on the slug, it would have to re-render on every navigation and you would lose the preserved state. So the current param belongs in the page, which does re-render, or in a client component using the routing hooks. Newer versions do expose *root* params, which are stable for the layout's lifetime, for cases like a locale segment.

---

## Related Guides

- [Rendering & Data](./rendering-and-data.md)
- [Caching](./caching.md)
- [React: Server Components](../react/server-components.md)
- [Next.js overview](./index.md)
