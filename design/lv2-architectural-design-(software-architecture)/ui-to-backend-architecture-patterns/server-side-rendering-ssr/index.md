# Server-Side Rendering (SSR)

* **How it works:** On every client request, the server fetches data from databases or APIs, renders the complete HTML string on-demand, and returns the fully populated HTML to the browser.
* **Trade-off:** Fast FCP and excellent SEO, but higher server CPU load and slower TTFB (since the server must fetch data and build the HTML before responding).

## Interview Questions & Answers

### Q1: What is the difference between SSR and SSG?
- **Answer:** **Server-Side Rendering (SSR)** generates HTML dynamically on the server for **every** request, making it ideal for highly dynamic or user-specific data (e.g., social feeds). **Static Site Generation (SSG)** generates the entire site's HTML once during the **build step**, serving static files from a CDN, which makes it incredibly fast but unsuited for live user-specific data.
