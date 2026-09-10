# Next.js: Mutations

How you change data — Server Actions, Route Handlers, and the security boundary that separates a working app from a breach. The security section here is the one that most reliably comes up in interviews, because it is the mistake teams actually make.

---

## 1. Server Actions

A Server Action is a function that runs on the server and can be called from the client as if it were local.

```jsx
// app/actions.ts
'use server';

export async function createPost(formData: FormData) {
  const session = await auth();                     // authenticate
  if (!session) throw new Error('Unauthorized');    // authorize

  const title = formData.get('title');
  const parsed = PostSchema.parse({ title });       // validate

  await db.post.create({ data: parsed });
  revalidatePath('/blog');                          // invalidate the cache
}
```

```jsx
// a form, with no client JavaScript required
<form action={createPost}>
  <input name="title" />
  <button type="submit">Create</button>
</form>
```

No API route, no `fetch`, no manual serialization. And because it is a real form submission, it **works before JavaScript loads** — genuine progressive enhancement.

On the React side, `useActionState`, `useFormStatus`, and `useOptimistic` handle the pending state, the errors, and the optimistic UI — see [React: Actions & Data APIs](../react/actions-and-data.md).

A vocabulary note: the docs now distinguish **Server Functions** (any function marked `'use server'`) from **Server Actions** (a Server Function invoked from a form action or a transition). In conversation people use "Server Action" for both.

---

## 2. The Security Point Interviewers Look For

> A Server Action compiles to a public HTTP endpoint. Anyone can call it with any arguments.

The `'use server'` directive is not an access-control mechanism. It does not mean "only my components can call this." Every action must, on its own:

- **Authenticate** — who is this?
- **Authorize** — may this user do this to this record? This is the object-level check, and skipping it is how you get an IDOR vulnerability.
- **Validate** — treat every argument as hostile input; parse it with a schema.

Do not rely on the fact that the UI only shows the delete button to admins. The endpoint exists regardless of what your UI renders. This is the same discipline you would apply to any REST endpoint, and forgetting it is the most common serious Next.js security mistake.

Two supporting practices the docs recommend, worth naming because they show depth:

- **A `server-only` data access layer.** Centralize the auth-plus-query logic in modules marked `server-only`, and return DTOs rather than raw database rows, so a stray field cannot leak through a component into the client payload.
- **Know what the framework does and does not do for you.** Action IDs are non-deterministic and rotated, unused actions are removed at build time, closed-over variables are encrypted, there is an Origin/Host check against CSRF, and there is a request body size limit. None of that is authorization. It reduces the attack surface; it does not decide whether *this user* may delete *this record*.

---

## 3. Behavioural Details That Surprise People

**Actions dispatch sequentially.** The client sends Server Actions one at a time. Wrapping several action calls in `Promise.all` does not run them in parallel. If you need parallelism, do the concurrent work inside a single action, or use a Route Handler.

**The response can carry a re-render.** If the action revalidates, mutates cookies, or redirects, the reply includes both the return value and a fresh RSC payload for the current route — so a mutation and a UI update are a single round trip rather than two.

**Closures are serialized.** Variables an action closes over are encrypted and sent to the client so they can be sent back. That is convenient, and it means you should not close over anything enormous — or anything you would not want leaving the server even in encrypted form.

---

## 4. Server Actions vs Route Handlers

Both run server code. They are for different things.

| | Server Action | Route Handler (`route.ts`) |
| :--- | :--- | :--- |
| Called by | Your own components and forms | Anything that speaks HTTP |
| Shape | A function call | A request and response |
| Good for | Mutations from your own UI | Public APIs, webhooks, third-party callbacks, file streaming, anything non-browser |
| Progressive enhancement | Yes, via `<form action>` | No |
| Cache integration | Revalidation helpers built in | Manual |
| Parallelism | Serialized by the client | Full control |

Rule of thumb: **your own UI mutating your own data → Server Action. Something outside your app needs to talk to you → Route Handler.** A Stripe webhook is a Route Handler. A "publish post" button is a Server Action.

---

## 5. Common Mistakes

- **Assuming a Server Action is private** because it is not imported anywhere public. Any bundled action is a reachable POST endpoint.
- **Checking permissions in the page instead of in the action.** Page-level gating is not a boundary.
- **Returning raw database rows** from an action, leaking columns the client should never see.
- **Calling several actions in `Promise.all`** expecting parallelism.
- **Using a Server Action for a webhook.** You need a Route Handler and control over the HTTP response.
- **Forgetting to invalidate** after a mutation, then wondering why the UI is stale — see [Caching](./caching.md).

---

## 6. Interview Questions & High-Impact Answers

### Q1: What is a Server Action, and what is the main security risk?
- **Answer:** A function marked `'use server'` that runs on the server and can be invoked directly from a component or a form action, with no API route and no manual serialization. Used with `<form action>` it works before JavaScript hydrates, which is real progressive enhancement. The risk is that it compiles to a public HTTP endpoint — anyone can call it with any arguments. `'use server'` is not access control. So every action has to authenticate, authorize against the specific record, and validate its inputs with a schema, exactly as a REST endpoint would. Hiding the button in the UI protects nothing.

### Q2: When would you use a Route Handler instead of a Server Action?
- **Answer:** When the caller is not my own UI. Webhooks, public APIs, OAuth callbacks, file downloads, anything that needs control over the HTTP response or is consumed by a non-browser client. Server Actions are for mutations initiated by my own components, where I want a function call rather than a request/response and I want the built-in cache revalidation. There is also a throughput consideration: the client serializes Server Action dispatches, so genuinely parallel work belongs in a Route Handler or inside a single action.

### Q3: How would you secure a Server Action that deletes a record?
- **Answer:** Inside the action, in this order: resolve the session and reject if there is none; load the record and verify this user is allowed to delete *that specific* record, which is the object-level check that prevents IDOR; validate and parse the id rather than trusting it; perform the delete; then revalidate the affected cache tag or path. I would put that auth-and-query logic in a `server-only` data access module so it is not duplicated per action, and return a DTO rather than the row. What I would not do is rely on the page having gated the UI, or on a proxy rule, since the action's endpoint is reachable independently.

### Q4: Does hiding a button prevent the action behind it from being called?
- **Answer:** No. The action is bundled and reachable as a POST endpoint whether or not any UI renders it — the only thing that removes it is dead-code elimination when nothing references it at all. This is the same reasoning as never trusting a disabled input or a hidden field. The server has to make the decision, on every call, with the actual user and the actual resource in hand.

---

## Related Guides

- [Caching](./caching.md) — invalidating after a mutation
- [Proxy](./proxy.md) — and why auth does not live there
- [React: Actions & Data APIs](../react/actions-and-data.md) — the client-side hooks
- [Web Frontend Security](../security/index.md)
- [Backend Security](../../backend/security/index.md)
- [Next.js overview](./index.md)
