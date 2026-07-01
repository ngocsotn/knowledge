# Single Page Application (SPA)

* **How it works:** The server sends a barebones HTML shell containing a single div container (e.g., `<div id="root"></div>`) and a large bundle of JavaScript. The browser downloads and executes this JavaScript, which builds the DOM nodes and fetches data dynamically from APIs.
* **Trade-off:** Fast Time-to-First-Byte (TTFB) but slow First Contentful Paint (FCP) and poor SEO since web crawlers see an empty page initially.

## Interview Questions & Answers

### Q1: What is Hydration in client-side framework architectures?
- **Answer:** Hydration is the process where the client-side JavaScript bundle reads the server-rendered static HTML, attaches event listeners, boots up state stores, and turns the static page into a fully interactive, reactive SPA.
