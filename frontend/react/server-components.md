# React: Server Components

This is the biggest mental-model shift in modern React, and the place interviews most often find gaps. The core question it forces you to answer about every component you write is simply: *where does this run?*

Assumes you have read [The Rendering Model](./rendering-model.md).

---

## 1. Two Places Your Code Can Run

**Server Components run on the server, once, and never ship to the browser.** They can `await` directly, read a database, use secrets — and they cannot use state, effects, or event handlers, because none of that exists on a server.

**Client Components** are ordinary React: state, effects, handlers, browser APIs. They are what you have always written.

```jsx
// app/page.jsx  — a Server Component by default in an RSC framework
import LikeButton from './LikeButton';   // a Client Component

export default async function Page() {
  const post = await db.post.findFirst();   // runs on the server, no API route needed
  return (
    <article>
      <h1>{post.title}</h1>
      <LikeButton postId={post.id} />       {/* interactivity is opted into */}
    </article>
  );
}
```

```jsx
// app/LikeButton.jsx
'use client';
import { useState } from 'react';

export default function LikeButton({ postId }) {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(!liked)}>{liked ? 'Liked' : 'Like'}</button>;
}
```

---

## 2. Why This Exists

Three real wins:

- **Bundle size.** A heavy markdown, date, or validation library used only in a Server Component ships zero bytes to the browser.
- **Data locality.** The component that needs data fetches it — no API layer in between, and no client-side waterfall where a component must mount before it can request anything.
- **Security.** Secrets and database access stay on the server by construction, not by convention.

The cost is a genuinely more complicated mental model, a hard boundary you must think about, and a framework dependency — you cannot use RSC without a framework and bundler that implement it.

---

## 3. The Things People Get Wrong

**1. "Client Components only run in the browser."** They do not. A Client Component is server-rendered to HTML first, then hydrated in the browser. "Client" means *this component can be interactive*, not *this component never runs on the server*. This is why code in a Client Component that touches `window` during render still crashes the server render, and it is the root of a lot of hydration confusion.

**2. `'use server'` does not mark a Server Component.** Components are Server Components by *default* — there is no directive for it. `'use server'` marks a **Server Function**, something callable from the client that runs on the server. The only directive that moves a component boundary is `'use client'`. Mixing these up is one of the most common vocabulary slips in interviews.

**3. `'use client'` marks a boundary, not a file.** Everything imported by that file, and everything below it in the tree, becomes client code too. It is the *entry point* to the client half of the tree, not a per-file toggle. Put it as low in the tree as you can.

**4. A Client Component can still render a Server Component — as `children`.** This confuses everyone. You cannot `import` a Server Component into a Client Component, but you can pass one *through*:

```jsx
// Server Component
<ClientLayout>
  <ServerSidebar />   {/* rendered on the server, passed as an already-rendered child */}
</ClientLayout>
```

This is the standard escape hatch for keeping a client-side layout wrapper without dragging its contents to the client.

**5. Props crossing the boundary must be serializable.** Strings, numbers, plain objects, arrays, and — specially handled — JSX and Server Functions. **Not** functions you wrote, class instances, `Date` methods, or `Map` (support varies). You are sending data over a wire; treat the boundary like an API.

**6. Server Components are not SSR.** SSR renders your whole component tree to HTML on each request, then hydrates all of it on the client. Server Components *never* hydrate, and their code never enters the bundle at all. They compose with SSR rather than replacing it. If you say "RSC is just SSR" in an interview, that is the moment the interviewer decides how deep to go.

---

## 4. Where the Boundary Should Sit

The practical skill is placing `'use client'` well.

```
   Bad                              Good
   ───                              ────
   app/page.jsx  'use client'       app/page.jsx        (server)
     ├─ Header                        ├─ Header         (server)
     ├─ ProductList                   ├─ ProductList    (server, fetches directly)
     │    └─ ProductCard              │    └─ ProductCard (server)
     └─ AddToCartButton               └─ AddToCartButton  'use client'

   everything ships to the          only the button ships
   browser, nothing can await       everything else stays server-side
```

One interactive button does not make a page interactive. Extract the interactive leaf, mark that, and leave the rest on the server.

---

## 5. Common Mistakes

- **`'use client'` at the top of a page** because one child needs interactivity, dragging the whole subtree into the bundle.
- **Passing a function or class instance across the boundary.** Props must be serializable.
- **Trying to `import` a Server Component into a Client Component.** Pass it as `children` instead.
- **Using `window` or `localStorage` during a Client Component's render.** It still runs on the server first.
- **Saying "RSC replaces SSR."** They compose; a real app uses both.

---

## 6. Interview Questions & High-Impact Answers

### Q1: How do Server Components differ from server-side rendering?
- **Answer:** SSR renders the whole tree to HTML per request and then hydrates all of it on the client, so every component's code still ships in the bundle. Server Components run only on the server, never hydrate, and their code never reaches the browser at all — so a heavy dependency used only there costs zero bytes on the client. They also let a component fetch its own data directly, removing an API layer and a client-side waterfall. They complement SSR rather than replacing it; a typical app uses both, plus Client Components for anything interactive.

### Q2: What are the constraints on the client/server boundary?
- **Answer:** `'use client'` marks a boundary, not a single file — everything imported below it becomes client code, so it belongs as low in the tree as possible. Props crossing the boundary must be serializable: plain data, JSX, and server functions, but not arbitrary functions or class instances. A Client Component cannot import a Server Component, but it can render one passed as `children`, which is the standard way to keep an interactive wrapper without pulling its contents to the client. And Server Components have no state, effects, or event handlers, because those concepts do not exist on the server.

### Q3: Do Client Components run on the server?
- **Answer:** Yes, and this is the misconception I see most often. A Client Component is prerendered to HTML on the server and then hydrated in the browser — "client" means it *can* be interactive, not that it never touches the server. The practical consequence is that touching `window`, `document`, or `localStorage` during render still breaks the server pass, so browser-only work has to move into an effect or a browser-only escape hatch. It is also why hydration mismatches happen at all: two renders in two environments have to agree.

### Q4: A Client Component needs to display data from your database. How do you get it there?
- **Answer:** Fetch it in the Server Component that renders it and pass it down as props — that is the whole point of the model, and it avoids an API route and a client-side waterfall. If the Client Component is a wrapper rather than a leaf, pass the server-rendered content through as `children` so the data-bearing part stays on the server. What I would avoid is adding a `useEffect` fetch inside the Client Component, which reintroduces the waterfall, the race condition, and a public endpoint you now have to secure.

### Q5: What does `'use server'` do?
- **Answer:** It marks a **Server Function** — a function that runs on the server and can be called from client code, which the bundler turns into an endpoint. It does *not* mark a Server Component; components are server-side by default and there is no directive for that. The only directive that moves a component boundary is `'use client'`. The security consequence of `'use server'` matters: the function becomes a public POST endpoint, so it must authenticate, authorize, and validate its own inputs regardless of which UI calls it.

---

## Related Guides

- [The Rendering Model](./rendering-model.md)
- [Actions & Data APIs](./actions-and-data.md)
- [Next.js: Rendering & Data](../nextjs/rendering-and-data.md) — the framework side of the boundary
- [Web Rendering Patterns](../rendering-patterns/index.md)
- [Frontend Implementation Struggles](../implementation-struggles/index.md) — hydration mismatches
- [React overview](./index.md)
