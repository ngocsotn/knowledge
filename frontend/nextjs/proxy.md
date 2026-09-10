# Next.js: Proxy (Formerly Middleware)

Code that runs before a request reaches a route. It is a small surface with one big trap: it looks like the natural place to put authentication, and it is not.

---

## 1. What It Is, and What Changed

A single file at the project root intercepts matching requests before the route renders — redirects, rewrites, headers, locale detection.

**The name changed.** This was `middleware.ts` with an exported `middleware` function; it is now **`proxy.ts` exporting `proxy`**. `middleware.ts` still works but is deprecated, and a codemod converts it. Config flags were renamed to match.

The rename came with a substantive change: **proxy runs on the Node.js runtime**, so the old edge-runtime constraints on which APIs you could use no longer apply there. Much of the older writing on this topic — "middleware runs at the edge, so no Node APIs, no database driver" — no longer describes the default. `middleware.ts` is retained mainly for the edge-runtime cases that still want it.

---

## 2. What It Is For

Cheap, request-level decisions:

- Redirects and rewrites
- Setting or forwarding headers
- Locale and geo detection
- A/B bucketing
- At most a coarse gate: "is there a session cookie at all?"

It sits on the critical path of **every matched request**, so cost here is cost on everything. Keep it small even though the runtime now permits more.

---

## 3. Why Auth Does Not Belong Here

This is the part interviews probe, and the framework's own guidance is now explicit: treat proxy as a last resort for authorization rather than the place it lives.

The concrete reason is worth being able to state:

> A Server Action POST is routed against the **page's** matcher. A proxy rule protecting one path does not necessarily intercept the action that mutates its data.

So a proxy that redirects unauthenticated users away from `/admin` may not run at all when someone posts directly to the Server Action that `/admin` would have called. You have protected the view and left the mutation open.

Beyond that, checking a cookie's *presence* is not authentication — it does not verify the session is valid, unexpired, or belongs to a user with the right role, and doing a real lookup on every request is exactly the expensive work you did not want here.

**Where authorization actually belongs:** inside each Server Action, in the page or layout that loads the data, and in a `server-only` data access layer — where you have the actual user and the actual record. See [Mutations](./mutations.md).

Use proxy for the cheap redirect that improves the experience; do not let it be the thing standing between an attacker and your data.

---

## 4. The Matcher

The matcher config decides which requests run your proxy. A careless one runs it on every static asset, image, and prefetch — multiplying its cost across requests that did not need it at all. Exclude static paths explicitly, and remember that `<Link>` prefetching means route requests fire more often than users navigate.

---

## 5. Common Mistakes

- **Using proxy as the authorization boundary.** Covered above; it can miss Server Action POSTs entirely.
- **Treating cookie presence as authentication.** It proves nothing about validity or role.
- **A matcher that catches static assets**, adding latency to everything.
- **Heavy work on the critical path** — a database round trip here is paid by every matched request.
- **Assuming it runs at the edge.** `proxy.ts` is Node.js runtime.

---

## 6. Interview Questions & High-Impact Answers

### Q1: What is middleware good for, and what should it not do?
- **Answer:** Cheap request-level decisions before the route renders — redirects, rewrites, locale detection, setting headers, and at most a coarse "is there a session cookie at all?" gate. Two updates worth mentioning: it is now `proxy.ts` exporting `proxy`, with `middleware.ts` deprecated, and it runs on the Node.js runtime, so the old "no Node APIs at the edge" constraint no longer describes the default. What has not changed is that it is not an authorization boundary — the framework's own guidance treats it as a last resort for auth. The concrete reason is that a Server Action POST routes against the page's matcher, so a proxy rule can miss the mutation entirely. Real authorization goes inside each action and in a `server-only` data access layer.

### Q2: Someone puts an auth check in proxy and says the app is secured. What do you tell them?
- **Answer:** That they have secured the page view and possibly nothing else. A Server Action posts against the page's matcher, so the rule may not intercept the mutation at all — the read is gated and the write is open. Separately, a presence check on a cookie does not validate the session, its expiry, or the user's role, and doing that properly means a lookup on every matched request, which is the work you were trying to avoid there. I would keep the proxy redirect for the user-experience benefit and add the real check inside each action and in the data access layer.

### Q3: Why did middleware get renamed to proxy?
- **Answer:** The rename came with a runtime change — it runs on Node.js rather than being constrained to the edge runtime — and the new name describes what it actually is: a request-level proxy for redirects, rewrites, and headers, not a general-purpose middleware layer where application logic belongs. The framework had been steering people away from putting auth and business logic there, and the name was part of the problem. `middleware.ts` still works and is deprecated rather than removed, with a codemod for the migration.

---

## Related Guides

- [Mutations](./mutations.md) — where authorization actually belongs
- [Routing & Layouts](./routing-and-layouts.md)
- [Authentication & Authorization](../../auth/index.md)
- [Web Frontend Security](../security/index.md)
- [Next.js overview](./index.md)
