# Next.js: Caching

This is the hardest part of Next.js and the source of most "why is my data stale?" questions. There are several distinct caches with separate lifetimes, and — importantly — two different caching *models* that coexist. Saying which one you mean is most of the battle.

---

## 1. The Four Layers

```
   Browser                         Server
   ───────                         ──────
┌──────────────┐    ┌─────────────────┐   ┌──────────────┐   ┌───────────┐
│ Router Cache │    │ Full Route Cache│   │  Data Cache  │   │  Request  │
│              │    │                 │   │              │   │Memoization│
│ RSC payload  │    │ Rendered HTML   │   │ fetch/query  │   │ dedupe    │
│ per session  │    │ + RSC payload   │   │ results      │   │ within    │
│              │    │                 │   │              │   │ one render│
│ client-side  │    │ persists across │   │ persists     │   │ lifetime: │
│ navigation   │    │ requests        │   │ across       │   │ one       │
│              │    │                 │   │ deployments  │   │ request   │
└──────────────┘    └─────────────────┘   └──────────────┘   └───────────┘
```

- **Request Memoization** — within a single render pass, the same fetch called in three components runs once. This is what lets each component fetch its own data without you hoisting it to a parent. Automatic, short-lived, nothing to configure.
- **Data Cache** — persists fetch/query results across requests and deployments. Controlled by revalidation settings and cleared by tag or path.
- **Full Route Cache** — the rendered output of a static route, held on the server and typically served from a CDN.
- **Router Cache** — a client-side store of route payloads so back/forward and prefetched links are instant, without a server round trip.

---

## 2. Two Models Coexist — Say Which One You Mean

This is the detail that makes caching discussions confusing, and naming it correctly is a strong interview signal.

**The legacy model** is what you get by default. Its important property today: **`fetch` is not cached by default** — nor are `GET` route handlers, nor page segments in the client Router Cache. Early App Router versions *did* cache these aggressively, which generated a great deal of confused blog writing that is still online. If you want caching in this model you opt in per call (`cache: 'force-cache'`, a `revalidate` value) or use route segment config.

**Cache Components** is the newer, explicit model, enabled with `cacheComponents: true`. It is stable but opt-in, and the framework has said it is the intended future default. Its rule is simpler and stricter:

> Nothing is cached unless you say so. You opt in with a `'use cache'` directive.

```javascript
async function getProducts(categoryId: string) {
  'use cache';
  cacheLife('hours');
  cacheTag(`category-${categoryId}`);
  return db.product.findMany({ where: { categoryId } });
}
```

Turning it on also brings PPR: static shell, dynamic holes, one route. Three constraints follow from it, and they are the ones people hit:

- **You cannot read `cookies()`, `headers()`, or `searchParams` inside a `'use cache'` function.** Read them outside and pass the values in as arguments — the arguments are part of the cache key. A `'use cache: private'` variant exists for per-user caching that needs runtime values, and a `'use cache: remote'` variant for a durable external store.
- **Anything uncached must sit under a `<Suspense>` boundary**, or the build fails and tells you which route is blocking.
- Route segment config (`dynamic`, `revalidate`, `fetchCache`) is replaced by these directives.

The cache key for a `'use cache'` function is derived from the build, the function identity, its serialized arguments, **and the variables it closes over** — which is why passing request values in as arguments works correctly and reaching for them from an outer scope does not.

---

## 3. Invalidating

Four operations, and knowing which to use is a common question:

| API | Effect | Where |
| :--- | :--- | :--- |
| `revalidateTag(tag, profile)` | Marks tagged entries stale, stale-while-revalidate style. Does not re-render the current page. **The second argument is required now** — the single-argument form is deprecated. | Anywhere server-side |
| `updateTag(tag)` | Expires immediately and re-renders, so the user sees their own write | Server Actions only |
| `revalidatePath(path)` | Invalidates everything for a path | Anywhere server-side |
| `refresh()` | Refetches uncached data only | Server Actions only |

The read-your-own-writes distinction between `revalidateTag` and `updateTag` is exactly the kind of thing an interviewer asks to separate people who have shipped from people who have read a tutorial. If a user edits something and then sees their old value, `revalidateTag` alone is usually the reason.

---

## 4. The Debugging Checklist for Stale Data

1. Is the route statically rendered when you expected dynamic? Reading a request API or an uncached source makes it dynamic.
2. Is the data cached with a revalidation window that has not elapsed?
3. Did the mutation invalidate the right tag or path — and did it use `updateTag` when the user needed to see their own change immediately?
4. Is it fresh on the server but stale in the client Router Cache? **A hard reload of the URL tells you:** fresh on reload but stale on client navigation means it is the Router Cache, and `router.refresh()` is the escape hatch.

That third step — hard reload versus client navigation — is the single most useful diagnostic, because it splits the problem in half in one action.

---

## 5. Common Mistakes

- **Trusting old blog posts.** A large amount of widely-linked material describes the original "cached by default" behaviour, which has not been accurate since v15.
- **Reaching for `router.refresh()` everywhere** instead of finding out which layer is actually serving stale data.
- **Reading `cookies()` inside a `'use cache'` function.** Not allowed; read outside and pass the value as an argument.
- **Calling `revalidateTag` and expecting the current page to update.** It marks stale; it does not re-render. `updateTag` does.
- **Assuming an in-memory cache persists.** It does not survive across serverless invocations or deployments; a durable store needs the remote variant or an external cache.

---

## 6. Interview Questions & High-Impact Answers

### Q1: Explain the caching layers in the App Router.
- **Answer:** Four layers with different lifetimes. Request memoization deduplicates identical fetches within a single render pass, which is what makes per-component data fetching viable. The Data Cache persists results across requests, invalidated by tag or path. The Full Route Cache holds the rendered output of static routes. The client-side Router Cache stores route payloads so back/forward navigation is instant. When someone reports stale data I work down that list to find which layer is serving it. I would also flag that there are now two models: the legacy one, where `fetch` has been *uncached* by default since v15 despite what a lot of older blog posts say, and Cache Components, where nothing is cached until you write `'use cache'` and PPR comes along with it. Which model the project is on changes the answer, so I would establish that first.

### Q2: What is the difference between `revalidateTag` and `updateTag`?
- **Answer:** `revalidateTag` marks tagged cache entries stale in a stale-while-revalidate sense — the next reader may still get the old value while it refreshes in the background, and it does not re-render the current page. `updateTag` expires immediately and re-renders, so the user who just performed the mutation sees their own write. That read-your-own-writes case is the one that matters in practice: after a user edits something, `revalidateTag` alone can show them their old data and look like a bug. Note also that `revalidateTag` now takes a required second argument naming a cache lifetime profile; the single-argument form is deprecated.

### Q3: Your page shows stale data after a mutation. How do you debug it?
- **Answer:** Identify which layer is stale. Did the mutation invalidate the right tag or path? If yes, is the data actually fresh on the server — check by hard-loading the URL directly. If it is fresh on a hard load but stale on client navigation, it is the client Router Cache, and `router.refresh()` after the mutation is the fix. If it is stale on a hard load too, it is the Data Cache or a statically rendered route that has not revalidated. And if the user is looking at their own edit, the likely cause is `revalidateTag` where `updateTag` was needed. Working down the layers in order beats guessing.

### Q4: What does `'use cache'` do, and what can't you do inside it?
- **Answer:** It marks an async function, component, or file as cacheable, with the key derived from the build, the function identity, its arguments, and the variables it closes over. `cacheLife` sets the lifetime profile and `cacheTag` attaches tags for invalidation. The constraint is that you cannot read request-scoped APIs inside it — `cookies()`, `headers()`, `searchParams` — because a cached result cannot depend on a request it will outlive. You read those outside and pass the values in as arguments, so they become part of the key. There are private and remote variants for per-user and durable caching respectively.

### Q5: Why did Next.js move away from caching by default?
- **Answer:** Because implicit caching produced a very common failure mode: developers wrote what looked like a normal fetch, got stale data in production, and had no obvious place to look. The defaults were surprising in the direction that causes bugs rather than the direction that causes slowness. The newer model inverts it — nothing is cached until you write `'use cache'` — so caching becomes a visible, local decision you can read off the code. The trade is that you have to do the work explicitly to get the performance back.

---

## Related Guides

- [Rendering & Data](./rendering-and-data.md)
- [Mutations](./mutations.md)
- [Caching Strategies](../../caching/index.md) — the general principles behind these layers
- [Next.js overview](./index.md)
