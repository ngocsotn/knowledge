# Browser Content Compression & Asset Optimization

Deep dive into how assets are optimized, compressed, and served efficiently to browsers.

## Compression Algorithms

### 1. Gzip (GNU Zip)
The traditional standard for web file compression. Leverages the DEFLATE algorithm (combining LZ77 and Huffman coding).
- **Usage:** Universally supported; fast compression/decompression speeds.

### 2. Brotli
Modern standard developed by Google. Uses a dictionary-based compression format specifically optimized for web text payloads (HTML, CSS, JS).
- **Performance:** Achieves up to **20-30% smaller file sizes** than Gzip at similar compression levels.
- **Support:** Supported by all modern browsers. Typically served with the header:
  `Accept-Encoding: br, gzip`

## Asset Optimization Strategies

### 1. Code Splitting
Breaking a monolithic JavaScript bundle into smaller, dynamically loaded chunks.
- **Impact:** Decreases Initial Page Load time; browsers only load the JS necessary for the current route.

### 2. Lazy Loading
Deferring the loading of non-critical resources (images, video, below-the-fold content) until they enter the viewport.
- **Implementation:** `loading="lazy"` attribute, or using dynamic `IntersectionObserver` API in JS.

### 3. Tree Shaking
Dead-code elimination. The bundler (Webpack, Vite, Rollup) analyzes ES6 import/export statements and strips unused code from the production bundle.

## Interview Questions & Answers

### Q1: Why does Brotli compress web assets (HTML, CSS, JS) much better than Gzip?
- **Answer:** Brotli uses a static, pre-defined dictionary containing common words, tags, and strings frequently found in web assets (e.g., `<div`, `class=`, `javascript`, common English words). This means instead of calculating back-references dynamically from scratch like Gzip's DEFLATE, Brotli can reference its dictionary index for common web terms, drastically reducing the binary size of compressed text files.
