# Next.js: Rendering & Data

The question to ask about every line you write in the App Router is *where does this run?* Get that right and static versus dynamic, streaming, and bundle size all follow from it. Get it wrong and you ship a database client to the browser.

---

## 1. Where Does This Code Run?

There are three answers, and confusing them is the most common source of bugs.

```
   Build time            Request time (server)          Browser
   ──────────            ─────────────────────          ───────
   Static generation     Dynamic rendering              Hydration,
   of pages and data     Server Components              Client Components,
                         Route Handlers                 event handlers,
                         Server Actions                 effects, state
```

**Components are Server Components by default.** They can `await` a database call, read environment secrets, and import heavy libraries that never reach the browser. They cannot use `useState`, `useEffect`, event handlers, or browser APIs.

**`'use client'` opts a subtree into the browser.** As covered in [React: Server Components](../react/server-components.md), it marks a *boundary*: everything imported beneath it becomes client code too. Keep it as low in the tree as you can — a page does not need to be a Client Component just because it contains one interactive button.

**Request-scoped data comes from async APIs.** Reading cookies, headers, or the route's params is asynchronous, because touching them tells the framework this route depends on the request:

```javascript
export default async function Page({ params }) {
  const { slug } = await params;
  const cookieStore = await cookies();
  const theme = cookieStore.get('theme')?.value;
  ...
}
```

If you learned Next.js from older material where these were synchronous, this is the change most likely to trip you up — the synchronous path was removed, not merely deprecated.

---

## 2. Static, Dynamic, and Revalidated

Next.js decides per route whether to render it once at build time or on every request. You mostly do not choose this explicitly — you *cause* it.

**A route becomes dynamic when it uses request-specific information:** `cookies()`, `headers()`, `searchParams`, or a data read explicitly marked as uncached. Otherwise Next.js will try to render it statically at build time.

**Static** is the fast path: HTML generated once, served from a CDN, no server work per visitor. Ideal for marketing pages, docs, and blog posts.

**Dynamic** renders per request. Necessary for anything personalized — a dashboard, a cart, anything behind a login.

**Revalidation (ISR)** is the middle ground and the one worth understanding well: serve static HTML, but regenerate it in the background on a schedule or on demand. A product page can be CDN-fast for every visitor and still update within a minute of a price change. On-demand revalidation after a mutation is the more precise version — see [Caching](./caching.md).

---

## 3. Streaming

Streaming is how you avoid one slow query holding up the whole page. A `loading.tsx` file, or an explicit `<Suspense>`, lets Next.js send the shell immediately and stream the slow section in when it is ready:

```jsx
export default function Page() {
  return (
    <>
      <Header />                         {/* sent immediately */}
      <Suspense fallback={<Skeleton />}>
        <SlowRecommendations />          {/* streamed in later */}
      </Suspense>
    </>
  );
}
```

The user sees a usable page in a fraction of the time, even though total time to complete is unchanged. This matters for perceived performance and for Core Web Vitals — see [Frontend Performance](../performance/index.md).

**Partial Prerendering (PPR)** is the natural conclusion: one route that is *both*, with a static shell served instantly from the CDN and dynamic holes streamed in per request. PPR is no longer a standalone experiment — it is the built-in behaviour of Cache Components, which is stable but opt-in. If you read older material describing an `experimental.ppr` flag or a per-route `experimental_ppr` export, that is out of date; those were removed.

---

## 4. Avoiding Waterfalls

Two kinds, and they need different fixes.

**Server-side waterfall** — two independent queries awaited in sequence:

```javascript
const user = await getUser(id);        // 80ms
const orders = await getOrders(id);    // 120ms  → 200ms total

const [user, orders] = await Promise.all([getUser(id), getOrders(id)]);  // 120ms
```

**Client-side waterfall** — a component that must mount and render before it can request anything, then renders again with the data. This is the structural problem Server Components remove: fetch on the server, in parallel, before any HTML is sent.

The subtle case is a Server Component that awaits data *and* renders a child that awaits its own data. That is sequential by construction. If the child's request does not depend on the parent's result, start it in the parent and pass the promise down, or wrap the child in its own Suspense boundary so it streams independently rather than blocking.

---

## 5. Bundle and Asset Discipline

The framework gives you good defaults, and you can undo them.

- **`next/image`** handles sizing, modern formats, lazy loading, and reserves space to prevent layout shift. Always give `width`/`height` or `fill` so CLS stays at zero.
- **`next/font`** self-hosts fonts at build time — no request to a third-party font host, and no layout shift from a late-loading face.
- **`next/dynamic`** defers a heavy client component until it is needed. Charting libraries, rich text editors, and map widgets are the usual candidates.
- **Watch the client bundle.** The biggest wins in the App Router come from *not* shipping code: keep formatting, parsing, and data-shaping libraries inside Server Components.
- **`<Link>` prefetches** routes in the viewport, which is why App Router navigation feels instant. Be aware it costs bandwidth on link-dense pages.
- **Environment variables.** Anything given the public prefix is embedded in the client bundle and is world-readable. Secrets must never carry it.

---

## 6. Common Mistakes

- **`'use client'` at the top of the page** because one child needs interactivity. This drags the whole subtree into the bundle. Push the boundary down.
- **Treating `params` and `searchParams` as plain objects.** They are Promises and must be awaited. So must `cookies()`, `headers()`, and `draftMode()`.
- **Fetching in a `useEffect` inside a Client Component** when the parent Server Component could have fetched it directly.
- **Sequential awaits in a Server Component** for independent queries.
- **Passing a function or class instance across the server/client boundary.** Props must be serializable.
- **Believing `'use server'` marks a Server Component.** It marks a Server Function; only `'use client'` moves a boundary.

---

## 7. Interview Questions & High-Impact Answers

### Q1: How does Next.js decide whether to render a route statically or dynamically?
- **Answer:** It infers it from what the route uses. Reading request-specific information — cookies, headers, search params — or performing an explicitly uncached data read makes the route dynamic; otherwise it is rendered at build time and served from a CDN. You can also force it with route configuration. The middle option is revalidation, where static HTML is regenerated on a schedule or on demand after a mutation, which is usually what you want for content that changes occasionally but is read constantly.

### Q2: What is streaming in Next.js and why does it help?
- **Answer:** Instead of waiting for every data dependency before sending HTML, the server sends the shell immediately and streams slower sections as they resolve, using Suspense boundaries — a `loading.tsx` file creates one automatically for a route segment. Total time to complete is unchanged, but time to first byte and first contentful paint improve a lot, so the page feels dramatically faster and Core Web Vitals improve. The practical benefit is that one slow query no longer blocks the entire page.

### Q3: Why is `'use client'` on a page a problem?
- **Answer:** Because it is a boundary, not a file-level flag: everything imported below it becomes client code and enters the bundle, so one interactive button can drag an entire page's dependency tree to the browser. It also gives up the ability to fetch data directly in those components. The fix is to push the boundary down — keep the page a Server Component, extract the interactive part into its own small Client Component, and pass server-rendered content through as `children` where a client wrapper is needed.

### Q4: How do you keep the client bundle small in the App Router?
- **Answer:** Mostly by not shipping code at all: keep data shaping, date and markdown formatting, and validation inside Server Components, so those dependencies never reach the browser. Push `'use client'` as low as possible. Use `next/dynamic` for genuinely heavy client widgets like editors and charts. Use `next/font` and `next/image` rather than hand-rolling. Then actually look at the bundle analyzer, because the answer is usually one library you did not expect.

### Q5: You are seeing a hydration mismatch error. What causes it and how do you fix it?
- **Answer:** The HTML the server produced does not match what the client rendered on the first pass. Usual culprits are values that differ between the two environments — `Date.now()`, `Math.random()`, locale-dependent formatting, `window` or `localStorage` accessed during render — or invalid HTML nesting that the browser silently corrects, such as a `<div>` inside a `<p>`. The fix depends on the cause: move browser-only values into an effect so the first render matches, use `suppressHydrationWarning` for a genuinely unavoidable difference like a timestamp, or fix the markup. What I would not do is disable SSR for the whole route to make the warning go away.

### Q6: Why must `params` and `cookies()` be awaited now?
- **Answer:** Because reading them is what tells the framework the route depends on the request, and making them async lets the framework begin rendering the static parts before those values are resolved — which is what makes partial prerendering possible. Practically it means a route that never awaits a request API can be fully static. The synchronous forms were removed rather than deprecated, so older tutorials produce code that no longer compiles, and there is a codemod for the migration.

---

## Related Guides

- [Caching](./caching.md)
- [Routing & Layouts](./routing-and-layouts.md)
- [React: Server Components](../react/server-components.md)
- [Web Rendering Patterns](../rendering-patterns/index.md)
- [Frontend Performance Optimization](../performance/index.md)
- [Next.js overview](./index.md)
