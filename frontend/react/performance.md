# React: Performance & Concurrency

Two related topics: how to stop React repeating work it does not need to repeat, and how modern React keeps an expensive update from blocking the interface. Both are heavily interviewed, and both attract cargo-cult answers — so the emphasis here is on when *not* to reach for these tools.

Assumes you have read [The Rendering Model](./rendering-model.md).

---

## 1. Memoization: Three Tools, One Purpose

| Tool | Skips | Returns |
| :--- | :--- | :--- |
| `React.memo(Component)` | Re-rendering the component when props are shallow-equal | A component |
| `useMemo(fn, deps)` | Recomputing a value | The value |
| `useCallback(fn, deps)` | Recreating a function | The function |

`useCallback(fn, deps)` is exactly `useMemo(() => fn, deps)`. There is no third concept.

**The chain that makes them work — or not.** `React.memo` compares props shallowly. If you pass a fresh object or inline arrow function, the comparison fails and `memo` bought you nothing but overhead:

```jsx
<MemoizedChild onClick={() => save(id)} config={{ mode: 'edit' }} />
// new function + new object every render → memo never helps
```

To make that memo effective you must stabilize *every* non-primitive prop with `useCallback` / `useMemo`. This is why memoization tends to spread: one `memo` demands five `useCallback`s upstream.

**When it is genuinely worth it:**
- A component that renders a large list or a heavy subtree.
- A genuinely expensive computation (parsing, sorting thousands of rows).
- A value used in another hook's dependency array, where an unstable identity causes an effect to re-run in a loop. This one is a correctness fix, not an optimization.

**When it is noise:** wrapping every component and every handler by reflex. Each `useMemo` costs an allocation, a dependency comparison, and memory, and it makes the code harder to read. Cheap components are cheaper to re-render than to memoize.

> Measure with the Profiler first. Optimizing a re-render that takes 0.3 ms is not engineering, it is superstition.

---

## 2. The React Compiler

React's optimizing compiler analyzes your components and inserts memoization automatically, making most manual `memo` / `useMemo` / `useCallback` unnecessary. It reached **1.0 and is considered stable**, but three qualifications matter and interviewers use them to check whether you have actually used it:

1. **It is opt-in, not on by default.** It ships as a Babel plugin, and frameworks that integrate it — Next.js among them — still leave it switched off by default. You enable it deliberately.
2. **It only optimizes code that follows the Rules of React.** Impure render, mutation of props or state, a violated hook rule — the compiler detects the violation and *skips that component*, silently. So "I turned the compiler on" does not mean everything got memoized. This is the strongest practical argument for the purity rules in [The Rendering Model](./rendering-model.md).
3. **It cannot help across boundaries it does not control** — referential equality coming from a third-party library, context values, or a parent's state change still cause re-renders.

You can opt a single component in with a `'use memo'` directive, or out with `'use no memo'`, which is the usual escape hatch when the compiler's output misbehaves.

Where it is adopted, the advice becomes: write straightforward code, let the compiler memoize, and reach for manual memoization only where the profiler proves you need it.

---

## 3. Concurrent Rendering: Interruptible Work

The headline capability of modern React is that **rendering can be interrupted**. Historically, once React started rendering it ran to completion, blocking the main thread. Now React can start rendering, pause to handle a more urgent update, and resume or discard the work.

You do not use this directly. You use the APIs built on it.

### Transitions

Mark an update as non-urgent so it cannot block typing or clicking:

```javascript
const [isPending, startTransition] = useTransition();

function handleChange(e) {
  setQuery(e.target.value);                     // urgent: the input must feel instant
  startTransition(() => setResults(search(e.target.value)));  // can be interrupted
}
```

Now typing stays smooth even when the results list is expensive, because React abandons in-progress result rendering when a new keystroke arrives. `isPending` gives you a spinner for free.

`useDeferredValue` is the same idea in a simpler shape: give it a value and it returns a version that "lags behind" during urgent updates. Reach for it when you do not control the state setter — for example, when the value arrives as a prop.

### Suspense

`<Suspense>` declares a loading boundary: while a child is not ready, show the fallback.

```jsx
<Suspense fallback={<Skeleton />}>
  <Comments postId={id} />
</Suspense>
```

Its real power is on the server, where it enables **streaming**: the server sends the shell of the page immediately with fallbacks in place, then streams each section's HTML as it becomes ready. The user sees content much sooner than they would waiting for the slowest query. That, plus selective hydration, is covered from the rendering angle in [Web Rendering Patterns](../rendering-patterns/index.md).

Important limitation to state accurately: Suspense is triggered by *Suspense-enabled* data sources — frameworks, `React.lazy`, and the `use` hook. Wrapping a component that calls `fetch` in a plain `useEffect` does nothing.

---

## 4. Fixing a Slow List

The most common real performance problem, and a standard interview scenario. In order:

1. **Profile.** Is the cost rendering many items, or one expensive item? The answers diverge completely.
2. **Check the keys.** Stable identities, not array indices.
3. **Memoize the row** with `React.memo`, and make sure every prop it receives is referentially stable — which usually means `useCallback` on handlers, or passing an id and looking the handler up inside.
4. **Virtualize** if the list is genuinely large. Rendering only the visible window beats any amount of memoization; a thousand memoized rows are still a thousand rows.
5. **Narrow the subscription.** If the data lives in global state, use a selector so only the affected row re-renders rather than the whole list.

---

## 5. Common Mistakes

- **Memoizing by reflex** without profiling.
- **Wrapping a component in `memo`** while still passing inline arrow functions and object literals, so the comparison never succeeds.
- **Assuming a re-render is a DOM update.** Often nothing is committed at all.
- **Optimizing renders when the real cost is a network waterfall** or an oversized bundle. Check what is actually slow.
- **Using `useTransition` for everything**, which just delays updates the user was waiting for.

---

## 6. Interview Questions & High-Impact Answers

### Q1: What is the difference between `useMemo`, `useCallback`, and `React.memo`?
- **Answer:** `React.memo` wraps a component and skips re-rendering it when props are shallow-equal. `useMemo` caches a computed value between renders. `useCallback` caches a function identity — it is literally `useMemo(() => fn, deps)`. They work together: `memo` only helps if every non-primitive prop is stable, which is why memoizing one component often forces `useCallback` on the handlers above it. I profile before adding any of them, since cheap components are cheaper to re-render than to memoize.

### Q2: With the React Compiler enabled, do you still need `useMemo` and `useCallback`?
- **Answer:** Mostly not, but "mostly" is doing work. Three caveats. It is opt-in and off by default even in frameworks that support it, so first confirm it is actually on. It only optimizes components that follow the Rules of React — if a component mutates props or renders impurely, the compiler detects that and silently skips it, so a violation means no memoization and no error. And it cannot control referential identity crossing a boundary it does not own, such as a value from a third-party library or a context. So the workflow becomes: write plain code, let the compiler handle the common case, and use the profiler to find the places it did not.

### Q3: What does "concurrent rendering" actually give you?
- **Answer:** Rendering became interruptible — React can start work, pause for a more urgent update, and resume or throw the work away. The user-facing APIs are transitions and Suspense. `useTransition` marks an update as non-urgent so an expensive re-render cannot block typing, and gives you `isPending` for the loading state. `useDeferredValue` is a simpler form of the same idea. On the server, the same machinery enables streaming HTML with Suspense boundaries and selective hydration.

### Q4: A list re-renders slowly when one item changes. Walk me through fixing it.
- **Answer:** Profile first, to confirm whether the cost is in rendering many items or in one expensive item. If it is the many-items case, wrap the row in `React.memo` and make sure every prop it receives is stable — no inline arrow functions or object literals — which usually means `useCallback` on handlers, or passing the item id and looking the handler up inside. Check that keys are stable identities rather than array indices. If the list is genuinely large, the real answer is virtualization: render only the visible window. And if state is global, use a selector so only the affected row subscribes.

### Q5: When would you use `useTransition` versus `useDeferredValue`?
- **Answer:** They do the same thing from different ends. `useTransition` wraps the *update* — you control the setter, so you mark that state change as non-urgent and get an `isPending` flag. `useDeferredValue` wraps the *value* — useful when you do not own the setter, for instance when the value arrives as a prop from a parent or a library. If I control the state I prefer `useTransition` because the pending flag is free; otherwise `useDeferredValue`.

### Q6: Does `<Suspense>` work with any async component?
- **Answer:** No, and this catches people. Suspense is triggered by data sources that integrate with it — `React.lazy`, the `use` hook, and framework-level data fetching. A component that calls `fetch` inside `useEffect` and sets state never suspends; it just renders with empty state first, so the fallback never appears. If I want a Suspense boundary around my own data, the data has to be exposed as a promise read with `use`, or fetched by a framework that wires it up.

---

## Related Guides

- [The Rendering Model](./rendering-model.md)
- [Actions & Data APIs](./actions-and-data.md)
- [Frontend Performance Optimization](../performance/index.md) — Core Web Vitals, bundles, assets
- [Web Rendering Patterns](../rendering-patterns/index.md)
- [React overview](./index.md)
