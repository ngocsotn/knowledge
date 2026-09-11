# React: State & Hooks

State is where React stops being obvious. The single idea in this guide — that state is a snapshot rather than a variable — explains a large share of the bugs people write in their first year, and it is asked about constantly.

Assumes you have read [The Rendering Model](./rendering-model.md).

---

## 1. State Is a Snapshot, Not a Variable

This is the most important idea on the page.

```javascript
function Counter() {
  const [count, setCount] = useState(0);

  function handleClick() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    console.log(count);   // 0  — not 3, not even 1
  }
  ...
}
```

The counter goes up by **one**, and the log prints **0**.

Why: `count` is a `const` captured in this particular render's scope. It is a photograph of the state at the moment this render happened. Calling `setCount` does not reassign it — nothing can, it is a const. It schedules a *future* render in which `count` will be a different value.

So all three calls compute `0 + 1`. React sees "set to 1" three times.

**The fix is the updater form,** which receives the latest pending value rather than the captured one:

```javascript
setCount(c => c + 1);
setCount(c => c + 1);
setCount(c => c + 1);   // now it is 3
```

> Use the updater form whenever the next state depends on the previous state. Use the direct form when you are setting a value that comes from somewhere else.

---

## 2. Batching

React groups multiple state updates into a single re-render. Setting three pieces of state in one handler produces one render, not three. Modern React batches consistently — including inside promises, timeouts, and native event handlers — so you can generally stop thinking about it. The relevant part is the consequence above: within one handler, the state variables do not change mid-function.

---

## 3. Lazy Initial State

```javascript
const [data, setData] = useState(expensiveComputation());     // runs EVERY render
const [data, setData] = useState(() => expensiveComputation()); // runs once
```

In the first line, the argument is evaluated on every render and then ignored on all but the first. Passing a function defers it. Small detail, real cost, common interview question.

---

## 4. State Belongs to a Position in the Tree

State is not attached to your component "the class." It is attached to a *position in the rendered tree*. Same component at the same position keeps its state; different position, or a different `key`, gets fresh state.

This gives you a deliberate technique: **change the `key` to reset a component's state.**

```jsx
<UserProfile key={userId} userId={userId} />
```

When `userId` changes, React unmounts the old one and mounts a new one with clean state — no effect needed to reset seven fields manually. This is idiomatic, not a hack.

---

## 5. Never Mutate State

```javascript
// Wrong: same array reference, so React may skip the render entirely
items.push(newItem);
setItems(items);

// Right: a new array
setItems([...items, newItem]);
```

React compares the previous and next values with `Object.is`. Mutating in place and passing the same reference back means the comparison says "unchanged" and nothing updates. It also breaks `React.memo` further down the tree and defeats the React Compiler, which skips components it detects mutating values.

The same applies to objects, nested objects, and anything held in state. If a deep update is getting painful, that is a signal the state shape is wrong — flatten it, or use `useReducer`.

---

## 6. Hooks: Why the Rules Exist

The rules of hooks — only call them at the top level, only from React functions, never inside conditions or loops — sound arbitrary until you know the implementation.

React does not know the *names* of your hooks. Internally, each component instance holds an ordered list, and each `useState` / `useEffect` call consumes the next slot **by position**:

```
render 1:  useState('a')  → slot 0
           useState('b')  → slot 1
           useEffect(...) → slot 2

render 2:  useState('a')  → slot 0   ✓ matches
           useState('b')  → slot 1   ✓ matches
           useEffect(...) → slot 2   ✓ matches
```

Put one behind an `if` and on some render the list shifts by one. Slot 1 now hands your effect's data to a state hook. That is why the rule exists, and saying so in an interview is a much better answer than reciting the rule.

---

## 7. The Stale Closure Trap

Because each render's functions capture that render's variables, a function that outlives its render sees old values:

```javascript
useEffect(() => {
  const id = setInterval(() => {
    console.log(count);   // forever the count from the render this effect ran in
  }, 1000);
  return () => clearInterval(id);
}, []);   // empty deps: this effect ran once, capturing count = 0
```

The fixes: include the value in the dependency array (the effect re-subscribes with fresh values), use the updater form so you never need to read current state, or keep the value in a ref. What you should not do is lie to the dependency array to make the warning go away — that warning is describing a real bug.

React also provides `useEffectEvent` for the specific case of reading a value inside an effect *without* re-running when it changes; see [Actions & Data APIs](./actions-and-data.md).

---

## 8. `useState` or `useReducer`?

Reach for `useReducer` when:

- The next state depends on several existing values, so update logic is spreading across handlers.
- Several pieces of state always change together — a form, a wizard, a request lifecycle.
- You want the update logic testable in isolation, or written down as named actions.

`useState` is fine for independent, simple values. The signal to switch is usually a handler that calls three setters in a row.

---

## 9. Common Mistakes

- **Deriving state into state.** Storing something in state that could be computed from existing state or props. It will go out of sync eventually — compute it during render instead.
- **Mutating state directly.** Same reference, so React may skip the render entirely.
- **Reading state right after setting it** and expecting the new value. It is the old snapshot until the next render.
- **Calling `useState(expensive())`** instead of `useState(() => expensive())`.
- **Resetting state with an effect** when a `key` does the job in one line.

---

## 10. Interview Questions & High-Impact Answers

### Q1: Why does calling `setCount(count + 1)` three times only increment by one?
- **Answer:** Because state is a snapshot of the render it was read in, not a live variable. `count` is a `const` captured in that render's closure, so all three calls compute the same `0 + 1` and React records "set to 1" three times. The updater form `setCount(c => c + 1)` queues functions that each receive the latest pending value, which gives 3. The rule I follow is to use the updater form whenever the next state depends on the previous one.

### Q2: Why are there rules of hooks?
- **Answer:** React tracks hooks per component instance as an ordered list, matched by call order rather than by name — the first `useState` is slot 0 on every render, and so on. A hook inside a condition or loop can shift that ordering between renders, so slot 1 might return the wrong hook's data. Calling them unconditionally at the top level is what keeps the order stable. That is also why the linter is strict about it: the failure is silent and confusing, not a clean error.

### Q3: How do you reset a component's state when a prop changes?
- **Answer:** Give it a `key` tied to that prop — `<Profile key={userId} />`. React treats a changed key as a different component at that position, so it unmounts the old instance and mounts a fresh one with clean state. This is idiomatic React, not a trick: state is bound to a position in the tree, and the key is what identifies that position. The alternative — an effect that resets seven state variables — is more code, runs a render late, and drifts as fields are added.

### Q4: What is a stale closure and how do you avoid one?
- **Answer:** Each render's functions capture that render's variables, so a function that outlives its render — an interval callback, a subscription handler, an event listener registered once — keeps reading the values from the render that created it. The honest fixes are to put the value in the dependency array so the effect re-subscribes with fresh values, to use the state updater form so you never read current state at all, or to hold the value in a ref. What I would not do is remove the dependency to silence the linter, because the warning is describing the bug rather than causing it.

### Q5: Why can't you mutate state directly?
- **Answer:** React compares previous and next state with `Object.is`. Push onto an array and pass the same reference back and the comparison says nothing changed, so the render is skipped. Even when a re-render does happen for another reason, mutation breaks `React.memo` comparisons down the tree, and the React Compiler detects mutation and silently declines to optimize that component. Creating a new array or object is also what makes the update traceable — you can tell what changed by comparing references.

### Q6: When would you use `useReducer` instead of `useState`?
- **Answer:** When update logic starts to spread. If a handler calls three setters in a row, if the next state depends on several current values, or if the same transitions happen from multiple places, a reducer collects that into one named, testable function. It also makes state transitions describable as actions, which helps when debugging. For independent simple values `useState` is clearer, and I would not convert just for consistency.

---

## Related Guides

- [The Rendering Model](./rendering-model.md)
- [Effects & Refs](./effects-and-refs.md)
- [Frontend State Management](../state-management/index.md) — Context, Redux, Zustand, server state
- [React overview](./index.md)
