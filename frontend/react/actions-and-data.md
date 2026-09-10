# React: Actions & Data APIs

React 19 added a group of APIs around the idea of an **Action**: an async function that updates data, with the pending state, errors, and optimistic UI handled for you instead of by hand. These come up constantly in current interviews, and they are simple once you see what problem they replace.

Assumes you have read [Effects & Refs](./effects-and-refs.md).

---

## 1. The Problem They Solve

Every form you have ever written has the same boilerplate: an `isSubmitting` state, an `error` state, a try/catch, and resetting both at the end. Multiply that by every mutation in the app and it is a meaningful share of your component code — and it is the same four lines every time, which is exactly the kind of thing a framework should own.

---

## 2. `useActionState`

```jsx
const [error, submitAction, isPending] = useActionState(
  async (previousState, formData) => {
    const result = await updateName(formData.get('name'));
    if (result.error) return result.error;    // returned value becomes the new state
    redirect('/profile');
    return null;
  },
  null,                                        // initial state
);

<form action={submitAction}>
  <input name="name" />
  <button disabled={isPending}>Save</button>
</form>
```

No `useState` for pending, no manual try/catch wiring. The value your action returns becomes the state on the next render, which is how errors and validation messages travel back to the UI.

Passing the action to `<form action>` also means **the form works before JavaScript hydrates** — real progressive enhancement, not a slogan.

---

## 3. `useFormStatus`

Reads the pending state of the nearest parent `<form>`, so a shared submit button component can show a spinner without prop-drilling:

```jsx
function SubmitButton() {
  const { pending } = useFormStatus();
  return <button disabled={pending}>{pending ? 'Saving…' : 'Save'}</button>;
}
```

The catch that trips people: it only works in a component *rendered inside* that form. Call it in the same component that renders the `<form>` and it always reports not-pending — a classic "why is this always false?" bug.

---

## 4. `useOptimistic`

Show the expected result immediately and reconcile when the server responds:

```jsx
const [optimisticLikes, addOptimisticLike] = useOptimistic(likes);

async function handleLike() {
  addOptimisticLike(likes + 1);   // UI updates instantly
  await likePost(id);             // reverts automatically if this throws
}
```

The difference from ordinary local state is that React automatically discards the optimistic value when the underlying action settles — you do not write the rollback. Hand-rolled optimistic UI usually goes wrong precisely in that rollback path, when an error arrives and the temporary value is left behind.

---

## 5. `use()`

`use` reads a promise or a context during render. Unlike every other hook it **may be called conditionally**, including after an early return.

```jsx
function Comments({ commentsPromise }) {
  const comments = use(commentsPromise);   // suspends until resolved
  return comments.map(c => <p key={c.id}>{c.text}</p>);
}
```

It integrates with Suspense: a component that `use`s an unresolved promise suspends, and the nearest boundary shows its fallback. So you write no loading state at all.

The rule to state carefully: **do not create the promise during render.** Create it in a Server Component or an event handler and pass it down, otherwise every render starts a new request and you have built an infinite fetch loop.

Compared with fetching in an effect, `use` avoids the mount-then-fetch waterfall and the manual race-condition handling — see [Effects & Refs](./effects-and-refs.md) for why that race exists.

---

## 6. `useEffectEvent`

Extracts the non-reactive part of an effect. When an effect needs to *read* a value without *re-running* when it changes, that logic goes in an effect event, which is not listed in the dependency array:

```jsx
const onVisit = useEffectEvent((url) => {
  logVisit(url, currentTheme);   // reads currentTheme, but does not react to it
});

useEffect(() => {
  onVisit(url);
}, [url]);   // only url is reactive
```

It is the sanctioned answer to "I need this value but I do not want it in the deps" — better than stashing it in a ref, and far better than lying to the linter.

---

## 7. Common Mistakes

- **Creating the promise inside the component that `use`s it.** Every render starts a new request.
- **Calling `useFormStatus` in the component that renders the `<form>`.** It must be a child.
- **Hand-rolling optimistic UI** and forgetting the error path, so a failed request leaves a phantom item on screen.
- **Reaching for `useState` + try/catch** for a form when `useActionState` covers it in one call.
- **Assuming Actions require a framework.** `useActionState` and `useOptimistic` are plain React; only *Server* Actions need a framework.

---

## 8. Interview Questions & High-Impact Answers

### Q1: What are Actions, and what does `useActionState` give you?
- **Answer:** An Action is an async function that performs an update, with React managing the surrounding lifecycle. `useActionState` takes that function plus an initial state and returns the current state, a wrapped action to pass to a form, and a pending flag — so the `isSubmitting` state, the try/catch, and the error state all disappear. The value the action returns becomes the next state, which is how validation errors get back to the UI. Passing the wrapped action to `<form action>` also gives you progressive enhancement, since the form submits normally before hydration.

### Q2: What is `useOptimistic` and how is it different from just setting local state?
- **Answer:** It shows the expected result of an async action immediately and reconciles automatically when the action settles — including reverting if it fails. With plain local state you have to write the rollback yourself and keep it in sync with the server response, which is where optimistic UI usually goes wrong. `useOptimistic` ties the temporary value to the lifetime of the action, so when the real data arrives the optimistic layer is simply dropped. It pairs with `useActionState`, which handles the pending and error states for the same action.

### Q3: What is `use()` and how does it differ from fetching in an effect?
- **Answer:** `use()` reads a promise or a context during render and integrates with Suspense — a component reading an unresolved promise suspends and the nearest boundary shows its fallback, so you write no loading state at all. It is also the one hook that may be called conditionally, including after an early return. The important discipline is not to create the promise during render: create it in a Server Component or an event handler and pass it down, or every render kicks off a new request. Compared with fetching in an effect, it avoids the mount-then-fetch waterfall and the manual race-condition handling.

### Q4: Why might `useFormStatus` always report `pending: false`?
- **Answer:** Because it reads the nearest parent `<form>`, so it has to be called from a component *rendered inside* that form. If you call it in the same component that renders the `<form>` element, there is no parent form in scope from its perspective and it reports idle forever. The fix is to extract the submit button into its own component. That constraint is deliberate — it is what lets a shared button component work in any form without props.

### Q5: What problem does `useEffectEvent` solve?
- **Answer:** The case where an effect needs to read a value but should not re-run when it changes — for example logging the current theme on navigation, where `url` is reactive and `theme` is not. Before it, people either added the value to the dependency array and got unwanted re-subscriptions, stashed it in a ref, or suppressed the lint rule and shipped a stale closure. An effect event holds the non-reactive logic and is deliberately excluded from the dependency array, so the effect's reactivity matches what you actually meant.

---

## Related Guides

- [Effects & Refs](./effects-and-refs.md)
- [Server Components](./server-components.md)
- [Performance & Concurrency](./performance.md) — Suspense, which `use()` depends on
- [Next.js: Mutations](../nextjs/mutations.md) — Server Actions, the server-side half
- [React overview](./index.md)
