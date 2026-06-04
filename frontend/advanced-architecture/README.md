# Advanced Frontend Architecture

Comprehensive study guide covering modern microfrontends, browser-level multi-threading, service workers caching strategies, and performance engineering in enterprise client-side applications.

---

## 1. Microfrontends

**Microfrontends** apply the architectural ideas of microservices to the frontend. An application is split into individual, autonomous, domain-focused frontend modules that are composed into a single, cohesive user experience.

### Integration Strategies:
1. **Build-Time Integration (npm packages)**:
   - *Mechanics*: Each microfrontend is compiled and published as a private npm package. The parent container installs them and builds a single monolithic bundle.
   - *Pros*: Simple, compile-time type-safety, robust tooling.
   - *Cons*: Releasing a change to one microfrontend requires recompiling and redeploying the *entire parent container* (coupled deployment).
2. **Runtime Integration (Module Federation - Recommended)**:
   - *Mechanics*: Each microfrontend is compiled and deployed independently as a separate host endpoint. The container loads these dynamic entries at runtime over the network using Webpack Module Federation or Vite Federation.
   - *Pros*: Complete independent deployments. Releasing Microfrontend A instantly updates the live application without touching other modules.
   - *Cons*: Complex routing orchestration; risk of runtime script failures; complex dependency sharing (avoiding loading React 18 multiple times).

### Critical Operational Struggles:
- **CSS Leakage**: Microfrontend A's CSS styles can pollute Microfrontend B's elements.
  - *Fix*: Use strict CSS Modules, BEM (Block-Element-Modifier) naming conventions, or wrap each microfrontend inside a **Shadow DOM** to guarantee style encapsulation.
- **State Sharing**: Sharing state across microfrontends should be avoided to prevent tight coupling.
  - *Fix*: Use native browser **Custom Events** (`window.dispatchEvent()`) for decoupled, asynchronous pub/sub messaging.

---

## 2. Service Workers & PWA Caching

A **Service Worker** is a type of Web Worker: a background script written in JavaScript that the browser runs separately from the main thread. It acts as a client-side network proxy, intercepting and caching outgoing HTTP requests.

### Core Caching Strategies

```
      Cache-First (Fast Asset)                   Network-First (Stale Data Risk)
[App] ──> [Cache] (Hit! Returns)           [App] ──> [Network] (Try Online)
                                                        |
                                                        ├── (Fail!) ──> [Cache] (Fallback)
```

1. **Cache-First (Cache-Falling-Back-To-Network)**:
   - *Mechanics*: Check the Cache first. If found, return immediately. If not found, fetch from the network, cache the result, and return.
   - *Use Case*: Static assets (JS chunks, CSS files, images, custom fonts).
2. **Network-First (Network-Falling-Back-To-Cache)**:
   - *Mechanics*: Attempt to fetch from the network. If successful, return and cache the latest response. If network fails (offline), load from the local cache.
   - *Use Case*: Critical dynamic data (e.g., inbox emails, product stock, account dashboard).
3. **Stale-While-Revalidate (Highly Popular)**:
   - *Mechanics*: Instantly return the cached version (fast response), while asynchronously initiating a network request to fetch the latest version, update the cache, and update the UI if data changed.
   - *Use Case*: Feeds, blog posts, configurations.

---

## 3. Web Workers (Multi-Threading in Browser)

By default, JavaScript in browsers runs on a **Single Thread (the UI/Main Thread)**. If you run a heavy computation (e.g., sorting 10 million rows, parsing large JSON files, executing canvas image manipulations), the main thread freezes, locking the UI and degrading user experience (Jank).

### Web Workers to the Rescue:
A **Web Worker** spawns a true secondary OS thread, running JavaScript in the background without blocking the UI.

```javascript
// main.js (Main Thread)
const worker = new Worker('worker.js');

// Send data to worker
worker.postMessage({ items: [3, 1, 5] });

// Listen for processed results
worker.onmessage = function(event) {
  console.log('Sorted list:', event.data.sorted);
};

// worker.js (Background Thread)
onmessage = function(event) {
  const sorted = event.data.items.sort();
  postMessage({ sorted }); // Return results
};
```

---

## 4. Hard Interview Questions & Deep Answers

### Q1: How do you prevent dependency duplication and manage shared libraries in dynamic Module Federation?
**Answer**:
When using Webpack Module Federation, if five microfrontends all depend on `react` and `react-dom`, you do not want the browser to download five identical copies of React, as this ruins performance.
- **The Solution: Declaring Shared Singletons**:
  Configure Module Federation properties inside the bundler setup (`webpack.config.js` or `vite.config.js`) to treat common frameworks as **shared singletons**:
  ```javascript
  // webpack.config.js
  new ModuleFederationPlugin({
    shared: {
      react: {
        singleton: true,
        strictVersion: true,
        requiredVersion: '^18.2.0',
      },
      'react-dom': {
        singleton: true,
        strictVersion: true,
        requiredVersion: '^18.2.0',
      },
    },
  });
  ```
  - `singleton: true`: Instructs the microfrontend loader to resolve and use only a single active instance of React across all modules.
  - `strictVersion: true`: If a microfrontend requests an incompatible React version, the runtime will throw an error or log a warning rather than loading multiple different versions silently.

### Q2: What is "Service Worker Lifecycle," and how do you handle updating an active Service Worker without locking old tabs?
**Answer**:
Service workers have a strict, secure lifecycle:
1. **Register**: Registered by the main thread.
2. **Install**: Service Worker files are downloaded and parsed. If successful, enters `installed` (waiting) state.
3. **Activate**: The service worker takes control of the page.
- **The "Waiting/Lock" Problem**:
  If a user has three tabs of your web app open and you deploy an updated Service Worker file, the browser downloads the new file, installs it, but **refuses to activate it**. It enters the `waiting` state because the older active service worker is still controlling open browser tabs. The new service worker will only activate once *all tabs are closed*.
- **The Production Fixes**:
  1. **`self.skipWaiting()`**: Inside the Service Worker `install` event, call `self.skipWaiting()`. This forces the waiting worker to instantly terminate the old worker and take control.
  2. **Reload Promotion (UX-friendly)**: Listen for the `controlling` state change in the main thread. Prompt the user with a toast notice: *"New version available. Reload to update."* On click, call `postMessage({ type: 'SKIP_WAITING' })` to trigger the change and programmatically reload the page (`window.location.reload()`).
