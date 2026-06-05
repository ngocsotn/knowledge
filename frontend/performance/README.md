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

## 3. Modern Browser Media Compression: WebP, AVIF, and WebM

Unoptimized images and videos typically represent 60% to 80% of an average website's total page weight. Delivering heavy legacy formats (like raw PNG, JPEG, GIF, and standard MP4) directly degrades the **Largest Contentful Paint (LCP)** core web vital, wasting user data and increasing load times.

---

### A. Comparative Media Matrix

| Format / Codec | Media Type | Compression Style | Size Reduction (vs. Legacy) | Alpha Transparency | Animation Support | Browser Compatibility | Best Production Use Case |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **WebP** (Google) | Image | Lossy & Lossless | **~30% smaller** than JPEG/PNG | **Yes** (replaces heavy PNG) | **Yes** (replaces heavy GIF) | **Universal** (>97% global support) | All standard web images, product galleries |
| **AVIF** (AOMedia) | Image | Lossy & Lossless | **~50% smaller** than JPEG / **~20% smaller** than WebP | **Yes** (replaces heavy PNG) | **Yes** (high quality, small size) | **High** (>93%, modern browsers) | Hero graphics, high-fidelity photography |
| **WebM (VP9/AV1)** | Video | Lossy | **~50% smaller** than H.264 MP4 | **Yes** (transparent video overlays) | Yes | **High** (all modern browsers) | Inline video backgrounds, loop micro-animations |

---

### B. Next-Gen Image Formats

#### 1. WebP (Universal Modern Standard)
Developed by Google, WebP provides superior lossless and lossy compression for web images.
- **Lossy Compression**: Uses predictive coding (similar to VP8 video keyframes) to predict adjacent pixel values, writing only the residual differences.
- **Lossless Compression**: Utilizes specialized Huffman coding, color cache, and spatial local entropy, making WebP lossless files **26% smaller** than equivalent PNGs while fully supporting alpha channels.

#### 2. AVIF (Next-Gen High Fidelity)
Based on the next-gen **AV1** open-source video codec, AVIF is the most advanced image compression format available to browsers.
- **Visual Fidelity**: Supports **10-bit and 12-bit color depth**, High Dynamic Range (HDR), and wider color gamuts. It eliminates "color banding" artifacts in gradients that frequently ruin highly compressed JPEGs.
- **The Compression Benefit**: At high compression rates, AVIF maintains sharp details and edges where WebP and JPEG begin to experience "smearing" or blocky artifacts.

---

### C. Next-Gen Video Formats

#### WebM (VP9 / AV1)
WebM is a highly compressed media container format developed specifically for HTML5 browser video.
- **The Codec Leap**: Traditional `.mp4` files use the H.264 (MPEG-4) codec, which is highly inefficient for high-definition web playback. WebM using **VP9** or **AV1** compression engines reduces video size by **50%** compared to MP4, ensuring instant start times and zero buffering.
- **Transparent Web Video**: Unlike MP4 (which does not support alpha channel transparency natively), WebM supports **video transparency**, enabling developers to overlay complex, pre-rendered motion graphics directly on top of dynamic HTML elements.

---

### D. Implementation Best Practices (Progressive Enhancement)

To deliver the smallest possible next-gen formats while maintaining safety fallbacks for legacy browsers, utilize the HTML5 **`<picture>`** element:

```html
<picture>
  <!-- 1. Serve AVIF to cutting-edge browsers (smallest size) -->
  <source srcset="hero-image.avif" type="image/avif">
  
  <!-- 2. Serve WebP to other modern browsers -->
  <source srcset="hero-image.webp" type="image/webp">
  
  <!-- 3. Fallback to standard optimized JPEG/PNG for old legacy clients -->
  <img 
    src="hero-image.jpg" 
    alt="Optimized Hero Graphic" 
    loading="lazy" 
    width="1200" 
    height="600" 
    decoding="async">
</picture>
```
- **`loading="lazy"`**: Instructs the browser to defer downloading off-screen images until they approach the viewport, saving network bandwidth.
- **`decoding="async"`**: Offloads image decoding from the main UI thread, preventing frame drops during loading.
- **`width` and `height`**: Explicitly declaring dimensions reserves layout aspect ratio space *before* the image downloads, **completely preventing Cumulative Layout Shift (CLS)**.

---

## 4. Popular Interview Questions & High-Impact Answers

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

### Q4: Why is AVIF preferred over WebP, and what is the computational trade-off of using AVIF?
* **Answer:** **AVIF** is preferred over WebP because it achieves up to **20% better compression density** and supports **10-bit and 12-bit color depths (HDR)**, which prevents "color banding" artifacts on smooth gradients. The main trade-off is **CPU compilation overhead**: encoding images to AVIF at build time is highly resource-intensive and can be up to **10x slower** than encoding WebPs or JPEGs, which can significantly slow down static-site generation (SSG) deployment pipelines if not properly cached.

### Q5: Why is declaring `width` and `height` attributes on HTML `<img>` elements critical for Cumulative Layout Shift (CLS), even when utilizing responsive CSS rules like `width: 100%; height: auto;`?
* **Answer:** Without HTML dimension attributes, the browser pre-allocates a **0px height** box for an image before its file is downloaded. Once loaded, the layout suddenly "shifts" downward to accommodate the dynamic height, triggering high CLS. Modern browsers read the HTML `width` and `height` attributes to **pre-calculate the aspect ratio** before downloading starts. This aspect ratio, combined with responsive CSS rules, allows the browser to reserve the exact vertical space in the layout, ensuring complete visual stability as the image is parsed and rendered.
