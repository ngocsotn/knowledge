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

Unoptimized images, videos, and audio typically represent 60% to 80% of an average website's total page weight. Delivering heavy legacy formats (like raw PNG, JPEG, GIF, standard MP4, and MP3) directly degrades the **Largest Contentful Paint (LCP)** core web vital, wasting user data and increasing load times.

---

### A. Comparative Media Matrix

| Format / Codec | Media Type | Compression Style | Size Reduction (vs. Legacy) | Alpha Channel | Key Performance Features | Browser Support | Best Production Use Case |
| :--- | :---: | :--- | :--- | :---: | :--- | :---: | :--- |
| **WebP** (Google) | Image | Lossy & Lossless | **~30% smaller** than JPEG/PNG | **Yes** | Replaces heavy PNG/GIF | **Universal** (>97%) | General web assets, product photos |
| **AVIF** (AOMedia) | Image | Lossy & Lossless | **~50% smaller** than JPEG / **~20% vs WebP** | **Yes** | 10/12-bit color, AV1-based | **High** (>93%) | High-fidelity photography, hero graphics |
| **JPEG XL (JXL)** | Image | Lossy & Lossless | **~60% smaller** than JPEG | **Yes** | **Lossless JPEG recompression**, progressive decoding | **Experimental** (Safari native, Chrome flags) | High-res photo archiving, rapid progressive loads |
| **AV1 Codec** | Video | Lossy | **~50% smaller vs H.264** / **~30% vs HEVC** | **Yes** | Royalty-free, next-gen high density | **High** (>92%) | High-definition streaming, micro-animations |
| **Opus Codec** | Audio | Lossy | **Extreme density** (6kbps - 510kbps) | N/A | Dynamic adaptive bit-rate, 5ms latency | **Universal** (>98%) | WebRTC voice calls, high-res audio streaming |

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

#### 3. JPEG XL (The Ultimate Image Standard)
JPEG XL (JXL) is designed to replace both baseline JPEG and legacy raw formats, optimized for both web delivery and professional photography.
- **Lossless JPEG Transcoding**: JXL can seamlessly transcode existing legacy JPEGs into JPEG XL files, **reducing file size by 20% with zero quality loss** (fully reversible back to the original byte-identical JPEG).
- **Responsive Progressive Decoding**: Built-in, high-efficiency progressive decoding allows browsers to render a low-resolution preview using only **15% of the file data**, progressively refining the image as more bytes arrive. This completely eliminates visual layout shifts.
- **Speed & Hardware**: Supports up to 32-bit float channels, up to 9999 channels (for multispectral data), and is mathematically designed for rapid, highly parallelized software encoding/decoding without needing dedicated hardware acceleration.

---

### C. Advanced Video Codecs Comparison

To deliver high-definition inline videos and micro-animations, selecting the correct video codec is critical:

1. **H.264 (AVC - Advanced Video Coding)**:
   - *Status*: Legacy industry workhorse.
   - *Pros*: Universal compatibility on 99.9% of devices globally. Supports hardware decoding on virtually all CPUs and GPUs.
   - *Cons*: Highly inefficient compression compared to modern codecs. Large file sizes at 1080p and 4K.
2. **H.265 (HEVC - High Efficiency Video Coding)**:
   - *Status*: Successor to H.264.
   - *Pros*: Excellent compression density (**50% smaller than H.264**). Standard in native mobile video capture (iOS/Android).
   - *Cons*: Burdened by complex, expensive royalty licensing fees. This delayed browser adoption historically, although it is now supported in modern browsers with matching hardware decoders.
3. **VP9 (Google open standard)**:
   - *Status*: Free, royalty-free standard.
   - *Pros*: Competes directly with HEVC in compression density, universally supported across all browsers without licensing bottlenecks.
4. **AV1 (AOMedia Open Standard)**:
   - *Status*: Modern royalty-free champion.
   - *Pros*: **30% more efficient than VP9/HEVC** and **50% more efficient than H.264**. Backed by major technology companies (Apple, Google, Netflix, Microsoft).
   - *Cons*: CPU-intensive encoding at build time. Requires modern hardware decoding support (integrated in Apple M3+, Intel 11th Gen+, AMD RDNA2+, Nvidia RTX 30+).

---

### D. Advanced Audio Codecs Optimization

1. **AAC (Advanced Audio Coding)**:
   - *Status*: The classic successor to MP3.
   - *Pros*: Universal compatibility, excellent performance at standard bit rates (128kbps - 256kbps).
   - *Cons*: Lacks dynamic adaptability and has high latency (not suitable for real-time applications).
2. **FLAC (Free Lossless Audio Codec)**:
   - *Status*: Professional lossless standard.
   - *Pros*: Compresses audio with **zero quality loss**.
   - *Cons*: File sizes are too large (typically 5x to 10x larger than lossy formats) for general web delivery, reserved strictly for audiophile platforms.
3. **Opus (The Web Audio Champion)**:
   - *Status*: The absolute king of interactive and streaming web audio. Developed by Xiph.Org and Skype.
   - *Pros*:
     - **Hybrid Architecture**: Merges **SILK** (optimized for human speech) and **CELT** (optimized for ultra-high-fidelity music).
     - **Dynamic Adaptability**: Can adjust its bit rate seamlessly on-the-fly from **6 kbps to 510 kbps** based on network conditions without dropping audio frames.
     - **Ultra-Low Latency**: Operates at a microscopic latency frame size (**5ms to 20ms**), making it the mandatory standard for WebRTC, VOIP, multiplayer gaming, and real-time audio streams.

---

### E. Implementation Best Practices (Progressive Enhancement)

To deliver the smallest possible next-gen formats while maintaining safety fallbacks for legacy browsers, utilize the HTML5 **`<picture>`** element:

```html
<picture>
  <!-- 1. Serve JXL to advanced browsers (if supported) -->
  <source srcset="hero-image.jxl" type="image/jxl">
  
  <!-- 2. Serve AVIF to cutting-edge browsers (smallest size) -->
  <source srcset="hero-image.avif" type="image/avif">
  
  <!-- 3. Serve WebP to other modern browsers -->
  <source srcset="hero-image.webp" type="image/webp">
  
  <!-- 4. Fallback to standard optimized JPEG/PNG for old legacy clients -->
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
