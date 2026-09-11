# React: Effects & Refs

`useEffect` is the most misused hook in React, and most of the effects in a typical codebase should be deleted rather than fixed. This guide covers what an effect is actually for, and the escape hatch — refs — for the values React should not be tracking at all.

Assumes you have read [State & Hooks](./state-and-hooks.md).

---

## 1. What an Effect Is For

`useEffect` is not "run this after render." It is:

> **Synchronize this component with an external system.**

External means outside React: a WebSocket, a browser API, a subscription, a third-party widget, an analytics SDK, `document.title`.

If you cannot name the external system being synchronized, you are probably not looking at an effect.

---

## 2. You Probably Do Not Need an Effect

This is the highest-yield section in the React guides, because unnecessary effects are the number one source of bad React code.

**Do not use an effect to compute derived state.** Just compute it during render:

```javascript
// Bad: extra render, state that can go out of sync
const [fullName, setFullName] = useState('');
useEffect(() => { setFullName(first + ' ' + last); }, [first, last]);

// Good
const fullName = first + ' ' + last;
```

If the computation is genuinely expensive, wrap it in `useMemo` — still not an effect.

**Do not use an effect to respond to a user event.** If something should happen because the user clicked, put it in the click handler. An effect cannot tell *why* it ran; the handler knows exactly why.

```javascript
// Bad: this also fires if the cart is loaded from a URL, a refresh, anything
useEffect(() => { if (cart.length) showToast('Added!'); }, [cart]);

// Good
function handleAdd(item) {
  addToCart(item);
  showToast('Added!');
}
```

**Do not use an effect to reset state when a prop changes.** Use a `key` — see [State & Hooks](./state-and-hooks.md).

**Do not chain effects.** An effect that sets state, triggering another effect that sets more state, is a render loop wearing a disguise. Compute the whole result in one place.

The remaining legitimate uses are narrow: subscriptions, manual DOM measurement, non-React widgets, and — in a client-only app without a data library — fetching.

---

## 3. The Dependency Array Means What It Says

It is not a "when to run" configuration. It is a list of reactive values the effect reads. React re-runs the effect when any of them changed (compared with `Object.is`).

If you find yourself fighting it — removing a dependency to stop a loop, or adding `// eslint-disable` — the effect is usually doing something that is not synchronization. Fix the effect, not the array.

---

## 4. Cleanup, and Why StrictMode Runs Effects Twice

Every effect that sets something up should tear it down:

```javascript
useEffect(() => {
  const socket = connect(roomId);
  return () => socket.disconnect();   // cleanup
}, [roomId]);
```

In development, StrictMode deliberately mounts, unmounts, and remounts your component — so effects run setup → cleanup → setup. This is not a bug and you should not try to defeat it with a ref guard. It is a test: if your effect breaks when run twice, it will break in production too, when React remounts a component or restores state. Write a correct cleanup and the double-invocation is invisible.

---

## 5. Fetching in an Effect Has a Race Condition

```javascript
useEffect(() => {
  let ignore = false;
  fetchUser(userId).then(data => {
    if (!ignore) setUser(data);   // discard responses from stale requests
  });
  return () => { ignore = true; };
}, [userId]);
```

Without the `ignore` flag, a slow request for user 1 can resolve *after* a fast request for user 2 and overwrite the correct data. That is the answer interviewers are looking for when they ask "what is wrong with fetching in useEffect?" — followed by "and in practice I would use a data library or a server component instead, because it also handles caching, deduplication, and revalidation."

An `AbortController` is the other standard answer, and it has the advantage of actually cancelling the request rather than just ignoring the response.

---

## 6. Refs: The Escape Hatch

`useRef` gives you a mutable box that survives renders and **does not trigger a re-render when changed**.

Use it for:
- DOM access (focus, scroll, measure, play a video).
- Values you need to remember but never render: a timeout ID, a previous value, a "has this already run" flag.

Do not use it as a replacement for state. If the UI needs to reflect the value, it belongs in state — mutating a ref will not update the screen.

The distinction in one line: **state is for values the UI is derived from; refs are for values the UI does not depend on.**

Reading or writing `ref.current` during render breaks purity. Do it in effects or event handlers.

**A modern convenience:** `ref` is an ordinary prop on function components now, so `forwardRef` is not needed for new code. It still works and has not been removed, so "forwardRef is deprecated" is slightly ahead of reality — the accurate phrasing is that it is expected to be deprecated eventually.

---

## 7. Common Mistakes

- **Effects as glue.** Chains of effects that each set state and trigger the next. Almost always a sign the logic belongs in an event handler or in render.
- **Lying to the dependency array** to silence the linter, then wondering why a value is stale.
- **Guarding against StrictMode's double-invoke with a ref** instead of writing a cleanup function.
- **Reaching for `useEffect` to fetch** in an app that has a data library or a server available.
- **Using a ref to store something the UI displays.** Changing it will not re-render.
- **Reading `ref.current` during render.** It breaks purity and misbehaves under concurrent rendering.

---

## 8. Interview Questions & High-Impact Answers

### Q1: When should you actually use `useEffect`?
- **Answer:** To synchronize with something outside React — a subscription, a WebSocket, a browser API, a non-React widget, `document.title`. Most effects I see in review should not exist: derived values should be computed during render (with `useMemo` if expensive), things that happen because of a user action belong in the event handler, and resetting state when a prop changes should use a `key` instead. My rule of thumb is that if I cannot name the external system being synchronized, it is not an effect.

### Q2: Why does React run my effects twice in development?
- **Answer:** StrictMode intentionally mounts, unmounts, and remounts components so effects run setup, cleanup, setup. It is a correctness test, not a bug: React reserves the right to remount a component and restore its state, so an effect that breaks when run twice is genuinely broken. The right response is to write a proper cleanup function, not to guard with a ref. It also only happens in development builds.

### Q3: What is wrong with fetching data in `useEffect`?
- **Answer:** Several things, but the one interviewers want is the race condition: if the dependency changes quickly, an earlier request can resolve after a later one and overwrite correct data with stale data. The fix is a cleanup function setting an `ignore` flag, or an `AbortController`. Beyond that, it creates a client-side waterfall — you cannot fetch until the component has mounted and rendered — and gives you no caching, deduplication, or revalidation. In practice I would use a data library like TanStack Query, or fetch on the server with a Server Component.

### Q4: What is the difference between state and a ref?
- **Answer:** Both persist across renders; only state triggers a re-render when it changes. So state is for values the UI is derived from, and refs are for values the UI does not depend on — a timeout ID, a DOM node, a previous value, a "did this already run" flag. The failure mode of confusing them is a ref holding something the user should see, which silently never updates. Refs also should not be read or written during render, because that breaks purity.

### Q5: An effect is running in an infinite loop. How do you debug it?
- **Answer:** Almost always a dependency whose identity changes every render — an object, array, or function literal created in the component body — where the effect then sets state and triggers another render. I check what is in the dependency array and whether each entry is referentially stable. The fix depends on the cause: memoize the value, move it outside the component if it is constant, use the state updater form so the effect does not need to depend on current state, or, most often, delete the effect because the work belongs in render or in an event handler.

---

## Related Guides

- [State & Hooks](./state-and-hooks.md)
- [Actions & Data APIs](./actions-and-data.md) — `useEffectEvent`, and the modern alternatives to fetching in effects
- [Performance & Concurrency](./performance.md)
- [React overview](./index.md)
