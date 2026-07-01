# Web Rendering Patterns

Comprehensive interview study guide covering CSR, SSR, SSG, ISR, hydration mechanics, and their architectural trade-offs.

---

## 1. Core Rendering Patterns

| Pattern | Render Location | Render Time | TTFB | FCP / LCP | SEO Friendly |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **CSR** (Client-Side) | Browser | On-demand (in browser) | Fast | Slow | Poor (No initial HTML) |
| **SSR** (Server-Side) | Server | On-demand (per request)| Slow | Fast | Excellent |
| **SSG** (Static Gen) | Build Server | Build Time | Fast | Fast | Excellent |
| **ISR** (Incremental) | Build Server/CDN | Build Time + Background | Fast | Fast | Excellent |

---

## 2. Core Concepts

### 1. CSR (Client-Side Rendering)
* **How it works:** The server sends a barebones HTML shell containing a single div container (e.g., `<div id="root"></div>`) and a large bundle of JavaScript. The browser downloads and executes this JavaScript, which builds the DOM nodes and fetches data dynamically from APIs.
* **Trade-off:** Fast Time-to-First-Byte (TTFB) but slow First Contentful Paint (FCP) and poor SEO since web crawlers see an empty page initially.

### 2. SSR (Server-Side Rendering)
* **How it works:** On every client request, the server fetches data from databases or APIs, renders the complete HTML string on-demand, and returns the fully populated HTML to the browser.
* **Trade-off:** Fast FCP and excellent SEO, but higher server CPU load and slower TTFB (since the server must fetch data and build the HTML before responding).

### 3. SSG (Static Site Generation)
* **How it works:** The entire website is pre-rendered into static HTML, CSS, and JS files at **build time**. These static files are uploaded directly to a CDN.
* **Trade-off:** Ultra-fast performance, zero server overhead at runtime. However, if content changes, you must trigger a full site rebuild.

### 4. ISR (Incremental Static Regeneration)
* **How it works:** Allows you to rebuild specific static pages in the background as requests come in, without rebuilding the entire site. It relies on a stale-while-revalidate caching header.

---

## 3. Hydration Process

**Hydration** is the process where client-side JavaScript attaches event listeners, hooks, and application state to the static, server-rendered HTML received from the server, making the static page interactive.

```
       Server Sends Static HTML            Browser Downloads JS Bundle
            ┌──────────────┐                     ┌──────────────┐
            │ <div>        │                     │ bundle.js    │
            │  <button>    │                     └──────┬───────┘
            │   Click Me   │                            │
            │  </button>   │                            ▼
            │ </div>       │                   ┌─────────────────┐
            └──────┬───────┘                   │  Hydration:     │
                   │                           │  Attach Event   │
                   ▼                           │  Listeners      │
            Non-Interactive                    └────────┬────────┘
             Static Visuals                             │
                                                        ▼
                                                 Fully Interactive
```

* **The Performance Cost ("Uncanny Valley"):** Between the time the user sees the server-rendered visual content (FCP) and the time hydration finishes (Time to Interactive - TTI), the page looks active but buttons do not respond to clicks. This period is known as the "uncanny valley."

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between Time to First Byte (TTFB) and Time to Interactive (TTI), and how do CSR vs. SSR affect them?
* **Answer:** **TTFB** is the duration a browser waits to receive the first byte of data from the server. **TTI** is the time it takes for a page to become fully interactive (interactive elements respond within 50ms). In **CSR**, TTFB is extremely fast because the server returns a simple static blank HTML shell instantly; however, TTI is slow because the browser must download and execute a massive JS bundle before rendering. In **SSR**, TTFB is slower because the server must block to fetch API data and render the page on-demand; however, visual paint (FCP) is fast because the client gets pre-populated HTML.

### Q2: What is "Selective Hydration" (implemented in React 18/Server Components)?
* **Answer:** Traditional hydration is monolithic: the browser must download the *entire* application's JS bundle and hydrate the *entire* page from top to bottom before any element becomes interactive. **Selective Hydration** utilizes React 18's `<Suspense>` to slice pages into independent, isolated chunks. The browser can hydrate high-priority components (like a button the user just clicked) immediately, even if other low-priority chunks (like a heavy footer or comment block) are still loading, eliminating monolithic hydration blocks.

### Q3: Why is ISR (Incremental Static Regeneration) highly valuable for massive e-commerce sites?
* **Answer:** E-commerce sites can have millions of product pages, making complete build-time **SSG** impossible due to massive build durations. **ISR** resolves this by pre-rendering only high-traffic pages (e.g., top 1,000 products) at build time. All other product pages are generated on-demand upon the first user visit, cached statically on CDNs, and revalidated asynchronously in the background. This guarantees blazing-fast loading speeds for millions of pages while keeping build times under a few minutes.
