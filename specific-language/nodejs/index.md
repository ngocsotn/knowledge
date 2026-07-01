# Advanced Node.js Core, Performance & Tools

Comprehensive guide covering Node.js architecture, streams, worker threads, and package management.

---

## 1. Buffers vs. Streams

Understanding how Node.js handles data in memory is crucial for building high-scale backends.

### A. Buffers
A **Buffer** represents a fixed-size chunk of memory allocated outside the V8 heap (using raw C++ memory).
- **How it works:** Reads the entire file/payload into physical RAM at once before processing.
- **Problem:** If you try to read a 4GB file using `fs.readFile()`, a standard server with 2GB of RAM will crash instantly with an `Out of Memory` (OOM) error because V8 cannot accommodate a buffer of that size.

### B. Streams
A **Stream** handles data chunk-by-chunk (in small sequential pieces, typically 64KB buffers), allowing processing of massive files with a constant, small memory footprint.
- **Types:**
  - `Readable` (e.g., `fs.createReadStream`)
  - `Writable` (e.g., `fs.createWriteStream`)
  - `Duplex` (e.g., TCP socket)
  - `Transform` (e.g., `zlib.createGzip` - modifies data while reading/writing)
- **Example (No RAM issues):**
  ```javascript
  const fs = require('fs');
  const zlib = require('zlib');

  const readable = fs.createReadStream('huge-4gb-file.txt');
  const compress = zlib.createGzip();
  const writable = fs.createWriteStream('huge-file.txt.gz');

  readable.pipe(compress).pipe(writable); // Pipeline handles streaming seamlessly
  ```

#### Handling Backpressure
- **Backpressure** occurs when a Readable stream pushes data much faster than the Writable stream can write it to the physical destination (e.g., slow network socket or slow disk).
- **How it's resolved:**
  1. The Writable stream's internal buffer hits its `highWaterMark` capacity threshold (default 16KB for object streams, 64KB for buffers).
  2. The `.write()` method returns `false`, indicating that the buffer is full.
  3. The stream coordinator pauses the Readable stream.
  4. Once the Writable stream clears its queue and writes data to disk, it emits the **`'drain'`** event.
  5. The coordinator resumes the Readable stream.
  * *Note: Using `.pipe()` or `stream.pipeline()` handles backpressure automatically under the hood.*

---

## 2. Worker Threads vs. Clustering vs. PM2

To scale across multiple CPU cores, Node.js provides different concurrency and multiprocessing modules:

| Feature | Worker Threads (`worker_threads`) | Cluster Module (`cluster`) | PM2 (Process Manager) |
| :--- | :--- | :--- | :--- |
| **Model** | **Multi-threading:** Share same OS process. | **Multi-processing:** Master spawns child processes. | **Multi-processing:** Spawns and manages independent instances. |
| **Memory** | Share memory space (via `ArrayBuffer`/`SharedArrayBuffer`). | Completely isolated memory spaces. | Completely isolated memory spaces. |
| **CPU Sharing**| Good for heavy CPU math inside one app. | Good for scaling web servers (ports sharing). | Good for production zero-downtime clustering & monitoring. |
| **Port Sharing**| Cannot share the same network socket port. | Yes, Master handles load balancing (Round-Robin) across workers on same port. | Yes, Nginx/PM2 forwards traffic across instances. |

### Code Example: Worker Threads
```javascript
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
  // Spawns thread
  const worker = new Worker(__filename);
  worker.on('message', message => console.log(`Result: ${message}`));
  worker.postMessage(40); // Send CPU task
} else {
  // Inside worker thread
  parentPort.on('message', (num) => {
    const result = fibonacci(num); // Intensive CPU task
    parentPort.postMessage(result);
  });
}

function fibonacci(n) {
  return n < 2 ? n : fibonacci(n - 1) + fibonacci(n - 2);
}
```

---

## 3. Package Management & node_modules Architecture

Understanding how different package managers lay out physical dependencies on disk prevents dependency hell and optimizes deployment pipelines.

### A. Core Architectural Comparison

| Dimension | npm (Default) | pnpm | Yarn (Classic v1) | Yarn (PnP - v2+) |
| :--- | :--- | :--- | :--- | :--- |
| **Folder Structure** | Flat (Duplicated if version mismatch) | Nested symlink/hard-link store | Flat (Duplicated if version mismatch) | Zero `node_modules` (PnP map file) |
| **Disk Space** | Heavy. Duplicates files across projects. | Minimal. Uses central hard-link content-addressable store. | Heavy. Duplicates files across projects. | Minimal. Compressed zip archives on disk. |
| **Phantom Dependencies** | **Vulnerable:** App can import packages not listed in `package.json` due to flat hoisting. | **Immune:** Strict nested structure. Only declared packages are importable. | **Vulnerable:** Uses flat hoisting. | **Immune:** Zero-hoisting. Imports resolved strictly. |

### B. pnpm Link Mechanics
pnpm completely avoids duplicate file downloads across your local disk by implementing a central, global, content-addressable store (usually at `~/.pnpm-store`).
- **Hard Links:** When you install packages in a project, pnpm creates physical **hard links** from the global store to the project's local virtual folder. Hard links point directly to the same physical disk sector, meaning a package is written to disk only once.
- **Symlinks (Symbolic Links):** pnpm then builds a strict nested tree structure in your `node_modules` using symbolic links (shortcuts) to map transitive dependencies, ensuring that Node.js cannot import undeclared packages (no phantom dependencies).

---

## 4. Advanced Node.js Interview Questions & Answers

### Q1: Write a custom implementation of an `EventEmitter` in Node.js.
* **Answer:**
  ```javascript
  class SimpleEventEmitter {
    constructor() {
      this.listeners = {};
    }

    on(event, callback) {
      if (!this.listeners[event]) {
        this.listeners[event] = [];
      }
      this.listeners[event].push(callback);
      return this; // Allows chaining
    }

    emit(event, ...args) {
      if (!this.listeners[event]) return false;
      this.listeners[event].forEach(callback => {
        callback.apply(null, args);
      });
      return true;
    }

    off(event, callback) {
      if (!this.listeners[event]) return this;
      this.listeners[event] = this.listeners[event].filter(cb => cb !== callback);
      return this;
    }
  }
  ```

### Q2: What are "Phantom Dependencies" and "NPM Dependency Hell", and how does pnpm solve them?
* **Answer:**
  - **Phantom Dependencies:** In npm/Yarn, the `node_modules` tree is flattened (hoisted to the top-level folder) to prevent deep duplicates. This allows your code to successfully `require('lodash')` even if `lodash` is not declared in your `package.json` (as long as one of your dependencies depends on it). If that dependency deletes `lodash` in an update, your production app crashes instantly.
  - **Dependency Hell:** Occurs when different sub-dependencies require conflicting versions of the same library, leading to massive nested duplicate folder footprints.
  - **pnpm's Solution:** It keeps a global store and places actual files in a hidden `.pnpm` folder. It then uses **symlinks** to construct a nested structure that matches exactly what you defined in `package.json`. No unlisted dependencies can be imported (preventing phantom dependencies) and identical files are hard-linked once (saving gigabytes of storage).

### Q3: Why is `SharedArrayBuffer` used alongside `Worker Threads`? Explain the threading synchronization problem.
* **Answer:** By default, Worker Threads communicate using message ports via `parentPort.postMessage()`. Under the hood, this uses the **HTML5 Structured Clone Algorithm**, which completely serializes (copies) the message data into a separate memory space for the receiving thread. This incurs heavy CPU and memory overhead for large arrays.
- **SharedArrayBuffer:** Allows sharing a single contiguous block of physical RAM directly across multiple threads without copying.
- **Synchronization Problem:** Multiple threads writing to the same shared memory location simultaneously can cause race conditions and data corruption.
- **The Solution:** Use **`Atomics`** (e.g., `Atomics.store()`, `Atomics.load()`, `Atomics.wait()`) to perform thread-safe, atomic operations on `SharedArrayBuffer` memory views, acting as lock primitives.

### Q4: What is Event Emitter Memory Leak, and how do you prevent it?
* **Answer:** In Node.js, an `EventEmitter` maintains a list of callback functions in memory. If a short-lived object (like a request context) subscribes to a long-lived global event emitter (like a database connection pool or system-wide process signal listener):
  ```javascript
  dbConnection.on('reconnect', () => {
    console.log(this.requestUrl); // closure captures 'this'
  });
  ```
  Every time a request occurs, a new listener is added. When the request finishes, the listener callback is still referenced by the global `dbConnection` object. The GC cannot reclaim the request context, leading to a major memory leak.
- **Prevention:** Always remove listeners when an object is destroyed or the context ends using `.off()` or `removeListener()`, or subscribe using `.once()` so the listener is automatically deleted after it fires once.

### Q5: What is the difference between `setImmediate()` and `process.nextTick()`?
* **Answer:**
  - `process.nextTick()` is **not** part of the event loop. It schedules a callback to run immediately after the current synchronous operation finishes, *before* the event loop moves to the next phase. Recursive calls can easily starve the event loop.
  - `setImmediate()` schedules a callback to run inside the **Check Phase** of the event loop, after I/O and timers have had an opportunity to execute in their respective phases. It is polite and prevents event loop starvation.

### Q6: How does the Cluster Module load-balance incoming HTTP requests across worker processes?
* **Answer:** The Cluster module uses a master process that binds to the target network port (e.g., `:8080`). It distributes incoming TCP connections to worker processes using one of two models:
  1. **Round-Robin (Default on Unix):** The master process listens on the port, accepts all incoming connections, and distributes them sequentially to child worker processes that are currently idle.
  2. **Shared Socket:** The master creates the listening socket and passes it directly to the workers, allowing the OS kernel to wake up any available worker to accept connections. This can lead to highly unequal load distribution due to OS scheduler scheduling.