# Frontend Performance Optimization

Comprehensive interview study guide covering Core Web Vitals, asset optimization, code splitting, layout thrashing, and rendering optimizations.

---

## 1. Core Web Vitals (The Performance Metrics)

Core Web Vitals are user-centric performance metrics defined by Google to measure actual page experience:

| Metric | What it Measures | Target Score | Key Tech to Fix |
| :--- | :--- | :--- | :--- |
| **LCP** (Largest Contentful Paint) | Loading performance (visual page load speed) | **< 2.5 seconds** | Image optimization, CDN caching, remove render-blocking JS/CSS |
| **CLS** (Cumulative Layout Shift) | Visual stability (accidental page movement) | **< 0.1** | Reserve dimensions for images/ads, avoid dynamic content insertion |
| **INP** (Interaction to Next Paint) | Responsiveness to user inputs (replaces FID) | **< 200 milliseconds**| Offload heavy JS tasks via web workers, break up long tasks |

---

## 2. Critical Optimization Strategies

### 1. Code Splitting & Lazy Loading
* **The Problem:** Delivering a massive JavaScript bundle (monolithic bundle) forces the browser to block rendering while downloading, parsing, and compiling unused code.
* **The Fix:** Use dynamic imports (e.g., `React.lazy` and `Suspense`) to split JavaScript by route or heavy component. The browser loads only the exact code needed for the active view, reducing initial bundle size.

### 2. Asset & Resource Pre-loading
Instruct the browser on resource priorities using link tags:
* **`dns-prefetch`:** Resolves DNS lookup for an external domain early (e.g., `<link rel="dns-prefetch" href="https://api.example.com">`).
* **`preconnect`:** Resolves DNS, TCP handshake, and SSL negotiation early.
* **`preload`:** Forces high-priority download of an asset needed on the current page (e.g., font files or hero images).
* **`prefetch`:** Downloads low-priority assets in background idle time for future pages.

### 3. Rendering Optimizations
* **Virtual Lists (Windowing):** Rendering thousands of DOM nodes in a massive list degrades memory and scroll performance. Virtual lists (e.g., `react-window`) render **only the visible nodes** in the viewport, reusing elements as the user scrolls.
* **Layout Thrashing:** Writing to the DOM and then immediately reading layout properties (like `element.offsetHeight`) forces the browser to run Reflow synchronously multiple times in a single frame. Always batch reads first, then batch writes.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is "Layout Thrashing" and how do you prevent it in high-frequency events (like scrolling)?
* **Answer:** **Layout Thrashing** occurs when code repeatedly writes to the DOM and reads layout values in a fast sequence (e.g., inside a scroll handler). Every time you write, the browser marks layout as "dirty." If you immediately read a layout property, the browser is forced to run synchronous Reflow instantly to return the accurate value. To prevent this:
  1. Use **`requestAnimationFrame`** to batch all DOM write operations to the next frame.
  2. Implement **Throttling** or **Debouncing** to limit execution rates.
  3. Keep DOM writes decoupled from scroll/mouse tracking.

### Q2: What is the difference between `defer` and `async` when loading scripts via HTML?
* **Answer:**
  * **Default (`<script>`):** Parsing HTML blocks immediately. The script is downloaded and executed before HTML parsing resumes.
  * **`async`:** The script downloads asynchronously in parallel with HTML parsing. The moment it finishes downloading, **HTML parsing blocks** while the script executes. Highly unpredictable ordering.
  * **`defer`:** The script downloads asynchronously in parallel with HTML parsing. Execution is **deferred** until HTML parsing is completely finished, preserving script execution order. Best for standard application bundles.

### Q3: How do you optimize modern web fonts to prevent cumulative layout shifts (CLS)?
* **Answer:** Web fonts can cause a flash of unstyled text (FOUT) or invisible text (FOIT), which shifts surrounding text blocks upon final render. To fix this:
  1. Add `font-display: swap;` in your `@font-face` CSS to display a system fallback font instantly while loading.
  2. Preload critical font files using `<link rel="preload" as="font" crossorigin>`.
  3. Set matching font metrics (line-height, letter-spacing) on the fallback font to minimize layout shift differences when the custom font swaps in.
