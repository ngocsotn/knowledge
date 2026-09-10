# React: The Rendering Model

Before hooks, before Server Components, before any API at all — this is the model everything else sits on. If you can say precisely when React calls your component function again, most React bugs stop being mysterious.

---

## 1. The One Idea

React's entire premise fits in one line:

> Your UI is a function of your state. You change the state; React figures out the DOM.

You never write "find that `<span>` and set its text." You describe what the screen should look like for the current state, and React makes the real DOM match.

That happens in two distinct phases, and keeping them separate in your head solves a lot of problems:

```
   state changes
        │
        ▼
┌───────────────┐   React calls your component functions.
│    RENDER     │   Produces a description of the UI (elements).
│  (pure, can   │   Compares with the previous description.
│   be thrown   │   Nothing has touched the DOM yet.
│    away)      │
└───────┬───────┘
        │
        ▼
┌───────────────┐   React applies the minimal set of DOM changes.
│    COMMIT     │   Refs are attached. Effects run.
│ (side effects)│   The user sees the result.
└───────────────┘
```

Two consequences worth memorizing now, because almost every React bug traces back to one of them:

1. **Render must be pure.** React may call your component and throw the result away, or call it twice. Do not mutate anything, do not fetch, do not touch the DOM during render.
2. **"Re-render" does not mean "update the DOM."** Re-rendering is React calling your function to see what changed. If nothing changed, the DOM is not touched. This is why "too many re-renders" is usually a smaller performance problem than beginners assume — and why the real cost is the work *inside* your component.

The diffing algorithm that decides what to update — reconciliation, keys, why `index` is a bad key — is covered in [The DOM & Virtual DOM](../dom-vdom/index.md). These guides are about everything above that layer.

---

## 2. What Actually Triggers a Re-Render

There are exactly three causes. Learn them as a closed list, because interviewers ask this and most candidates get it half right.

1. **Its own state changed** (`useState` / `useReducer` setter called with a different value).
2. **Its parent re-rendered.** By default, when a component re-renders, React re-renders all of its children. It does not check whether their props changed first.
3. **A context it consumes changed value.**

Now the myth to kill:

> "A component re-renders when its props change."

Backwards. A component re-renders when its *parent* re-renders — which usually happens to coincide with props changing, but not always. A child whose props are identical still re-renders when the parent does, unless you wrap it in `React.memo`. And a child *inside* `props.children` passed down from a grandparent does **not** re-render when the middle component re-renders, because its element was created higher up. That surprises people, and it is a legitimate performance technique.

**Why identity matters.** Two objects with the same contents are not the same object:

```javascript
{ a: 1 } === { a: 1 }   // false
```

Every render creates fresh objects, arrays, and functions. So a memoized child receiving `style={{ color: 'red' }}` sees a "new" prop every single time and re-renders anyway. This one fact explains most of the confusion around `memo`, `useMemo`, and `useCallback` — see [Performance & Concurrency](./performance.md).

---

## 3. Component Identity

React decides whether to *update* an existing component or *destroy and recreate* it based on two things: its position in the tree and its type.

This leads to a mistake that looks harmless and is not:

```jsx
function Parent() {
  // Wrong: a brand-new component type on every render
  function Child() { return <input />; }
  return <Child />;
}
```

Every render of `Parent` creates a new `Child` function. React compares the old type with the new type, sees two different functions, and concludes this is a *different component* — so it unmounts the old one and mounts a fresh one. All state inside is lost, the input loses focus, and any effects re-run. Define components at module scope, always.

The same rule read the other way gives you a useful tool: changing a component's `key` deliberately destroys and recreates it, which is how you reset state. That is covered in [State & Hooks](./state-and-hooks.md).

---

## 4. Common Mistakes

- **Defining a component inside another component.** A new component type each render, so React remounts it and loses its state.
- **Doing work during render.** Fetching, mutating a module-level variable, writing to a ref, touching the DOM. Render may be called twice or discarded; anything with a side effect belongs in an event handler or an effect.
- **`index` as a key** in a reorderable or filterable list. Covered in [DOM & VDOM](../dom-vdom/index.md).
- **Assuming a re-render is expensive.** It is often not. Profile before you optimize.

---

## 5. Interview Questions & High-Impact Answers

### Q1: What causes a component to re-render?
- **Answer:** Three things: its own state changed, its parent re-rendered, or a context it consumes changed. Notably *not* "its props changed" — by default a child re-renders because the parent did, whether or not props actually differ, and `React.memo` is what opts out of that. One useful exception: children passed via `props.children` were created by an ancestor, so they do not re-render when the intermediate component does.

### Q2: What is the difference between the render phase and the commit phase?
- **Answer:** Render is React calling your component functions to build a description of the UI and diff it against the previous one — pure, interruptible, and possibly thrown away. Commit is where React applies the minimal DOM changes, attaches refs, and runs effects. The practical rule is that anything with a side effect belongs in commit, not render, because React may render speculatively and discard the result. Concurrent rendering makes this stricter, not looser.

### Q3: Why must render be pure?
- **Answer:** Because React does not guarantee it calls your component exactly once per update. It may render, discard the work, and render again — that is what makes concurrent features like transitions possible — and StrictMode double-invokes in development to surface violations. So a component that mutates a variable or fetches during render produces duplicated or inconsistent effects. Purity is also what the React Compiler relies on: it silently skips components that break the rules, so an impure component quietly loses its optimization.

### Q4: Why does defining a component inside another component lose its state?
- **Answer:** React identifies a component by its position in the tree and its type. A function declared inside the parent is a *new function object* on every render, so the type comparison fails and React treats it as a different component — unmounting the old instance and mounting a new one. State is discarded, inputs lose focus, and effects re-run. The fix is to declare components at module scope and pass data through props.

---

## Related Guides

- [State & Hooks](./state-and-hooks.md)
- [Performance & Concurrency](./performance.md)
- [The DOM & Virtual DOM](../dom-vdom/index.md)
- [React overview](./index.md)
