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

## 2. Microfrontend Isolation & Security Topologies

To build a reliable microfrontend cluster, systems must strictly isolate styling, script execution, and document objects to prevent one unstable module from crashing the entire host container.

### 1. Style Encapsulation (Shadow DOM vs. CSS containment)
Traditional CSS is global. If Microfrontend A declares `.button { background: red; }`, it can instantly destroy the visual design of Microfrontend B.

#### Solution A: The Shadow DOM (Strong CSS Encapsulation)
The **Shadow DOM** creates a scoped document fragment attached to a DOM element, completely isolated from the main DOM tree.
* **Encapsulation Boundary:** Style rules declared inside a `#shadow-root` do not leak out to the parent page, and global styles (with some exceptions like inherited font-family and custom CSS properties) do not penetrate inward.

```html
<div id="mfe-container">
  <!-- Outer Main DOM -->
  <style>h1 { color: blue; }</style> 
  <h1>Monolith Header</h1>

  <!-- Shadow Host -->
  <div id="mfe-shadow-host"></div>
</div>

<script>
  const host = document.querySelector('#mfe-shadow-host');
  // Create shadow root in 'closed' or 'open' mode
  const shadowRoot = host.attachShadow({ mode: 'open' });
  
  // Scoped styles and elements
  shadowRoot.innerHTML = `
    <style>
      h1 { color: red; } /* Isolated: Only colors elements inside shadow root */
    </style>
    <h1>Microfrontend Scoped Title</h1>
  `;
</script>
```

#### Solution B: Web Components & Custom Element Registry Safety
Web Components wrap shadow roots inside custom HTML tags. However, if Microfrontend A and Microfrontend B both attempt to register the exact same element name (`customElements.define('app-button', Button)`), the browser throws a fatal error.
* **The Production Fix:** Implement namespace prefixing (`customElements.define('mfe-a-button', Button)`) or dynamic registries (`ScopedRegistry` polyfills) to intercept and deduplicate registrations at runtime.

---

### 2. Runtime Script Isolation (Iframes vs. JavaScript Sandboxes)

| Isolation Pillar | Standard Iframes | Shadow DOM + JavaScript Proxies |
|---|---|---|
| **CSS Scoping** | Absolute isolation (native browser level) | Excellent isolation (shadow boundary) |
| **JS Global State (`window`)** | Complete isolation (separate window context) | Shared `window` object (leakage risk) |
| **Performance Overhead** | High (memory cost of multiple document contexts)| Extremely low (lightweight native DOM nodes) |
| **Communication Channel** | Structured `postMessage` protocol | Direct JavaScript function calls or custom events |

#### Production Dynamic Sandbox Pattern (JavaScript Proxies)
When using Module Federation without iframes, advanced hosts wrap each remote module inside an active **ES6 Proxy-based Sandbox** (like Single-SPA or qiankun). This intercepts access to the global `window` object, keeping global variables scoped strictly to the current microfrontend context:

```javascript
class WindowSandbox {
  constructor() {
    this.fakeWindow = {};
    this.proxy = new Proxy(window, {
      set: (target, prop, value) => {
        // Intercept global sets and isolate them to the fake window
        this.fakeWindow[prop] = value;
        return true;
      },
      get: (target, prop) => {
        // Prefer fake window values over true global window values
        if (prop in this.fakeWindow) {
          return this.fakeWindow[prop];
        }
        return target[prop];
      }
    });
  }
}

const sandboxA = new WindowSandbox();
// Microfrontend A executes inside sandboxA.proxy context:
// sandboxA.proxy.globalToken = "A-Secret"; -> Writes to fakeWindow, original window is untouched!
```

---

## 3. Module Federation Internals & Loading Orchestration

**Webpack/Vite Module Federation** is the gold-standard framework for dynamic runtime integration. It compiles microfrontends into separate modular entry points, letting them resolve, load, and execute dependencies dynamically at runtime.

### The Loading Lifecycle Pipeline

```
[Container Main Page] ──► 1. Loads 'remoteEntry.js' script ──► 2. Initializes Remote Scope (Deduplicates)
                                                                            │
[Target Render Node]  ◄── 4. Executes Remote Component  ◄── 3. Fetches shared dependencies & chunk
```

1. **Load `remoteEntry.js`:** The container loads a tiny metadata script (typically `< 5KB`) containing the remote's manifest (all exposed modules and required dependencies).
2. **Container Scope Initialization:** The container reads the remote manifest and initializes the shared dependency pool. If both Container and Remote need `lodash`, they negotiate:
   - If version constraints match, the container registers a single version of `lodash` and shares it.
   - If Remote needs a higher incompatible version, it downloads its own private chunk, preventing version crashes.
3. **Module Resolution:** When the container triggers `import("remoteApp/Button")`, the federation loader executes a dynamic `import()` to download the exact component script chunk and injects it directly into the running application memory.

---

## 4. Service Workers & PWA Caching

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

## 5. Web Workers (Multi-Threading in Browser)

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

## 6. Hard Interview Questions & Deep Answers

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

### Q3: How do you prevent CSS style leakage in microfrontends without using iframe overhead?
**Answer**:
Without the memory and performance cost of iframes, systems utilize three patterns to prevent style leakage:
1. **The Shadow DOM (Hard Browser Isolation):** Render each microfrontend inside a Web Component wrapped in an open Shadow Root (`host.attachShadow({ mode: 'open' })`). This guarantees native style scoping where CSS rules cannot bleed out or in.
2. **CSS Modules (Build-Time Namespace Scoping):** Configure the bundler (Webpack/Vite) to compile CSS classes into unique hashes (e.g., `.button` compiles to `.Button_module_hash_18f9`).
3. **Prefixing / BEM Conventions:** Enforce strict naming rules (e.g., prefixing all rules under `.mfe-billing-`).

### Q4: Explain how ES6 Proxies can be used to construct a global object sandbox for running microfrontends.
**Answer**:
When multiple microfrontends are loaded onto the same window context, they can pollute the global scope or overwrite global flags. To prevent this, a host container can intercept global variables using **ES6 Proxies**:
1. The container creates a lightweight "fake window" object (`{}`) for each microfrontend instance.
2. A Proxy wraps the global `window` object. Any set actions (`proxyWindow.myVar = 'value'`) are trapped and written strictly to the fake window.
3. Any get actions search the fake window first, and fall back to the real `window` if not found.
4. When switching routes, the proxy context is swapped, instantly resetting global side-effects without reloading the browser page.
