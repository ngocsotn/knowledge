# Content Delivery Networks (CDNs)

A globally distributed network of proxy servers that cache static and dynamic assets (images, video, JS/CSS, HTML) close to the physical location of users to minimize latency.
- **Edge Cache:** Caching content directly on physical CDN edge nodes.
- **Dynamic Routing:** Routing requests through optimized private CDN pathways to accelerate dynamic API calls.

## Interview Questions & Answers

### Q1: How do you handle cache invalidation on a global CDN?
- **Answer:**
  1. **Purge API:** Force the CDN to invalidate and clear its cache for specific URLs globally. (Can take seconds to minutes).
  2. **Cache-Busting (Versioned URLs):** Embed unique asset hashes or versions into resource filenames (e.g., `bundle.a8b12f.js`). When a new version is released, the URL changes, automatically forcing the CDN to cache-miss and fetch the new file from the origin, bypassing invalidation lag completely.
