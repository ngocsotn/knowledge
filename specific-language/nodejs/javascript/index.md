# Advanced JavaScript Language Internals

Comprehensive study guide covering Core JavaScript, closures, event loop, async/await, and V8 memory management.

---

## 1. ECMAScript vs. JavaScript

- **ECMAScript (ES):** The standardized scripting language specification managed by the **TC39** committee (under Ecma International). It defines the syntax, rules, and core features of the language (e.g., how arrays, promises, and loops must behave).
- **JavaScript (JS):** The actual **implementation** of ECMAScript. It includes the core ES engine (like V8 in Chrome/Node, JavaScriptCore in Safari, SpiderMonkey in Firefox) along with platform-specific **Host APIs** (e.g., `window`, `document`, `fetch` in browsers; `fs`, `path`, `process` in Node.js).

---

## 2. Closures & Lexical Scopes

To understand closures, you must first understand how JavaScript executes code:

* **Execution Context:** The wrapper environment where code is executed. It contains the active variables, arguments, and the `this` binding.
* **Lexical Environment:** The physical structure in the JS engine that maps variable names to values. It consists of:
  1. **Environment Record:** The actual storage map of variables.
  2. **Outer Reference:** A pointer to the parent outer Lexical Environment (the Scope Chain).

### Creation vs. Execution Phase
1. **Creation Phase:** The engine compiles the function, sets up the Outer Reference scope chain, and initializes variables with `undefined` (or leaves them uninitialized in the "Temporal Dead Zone" if declared with `let` or `const`).
2. **Execution Phase:** Values are assigned to variables, and functions are executed line-by-line.

### What is a Closure?
A **closure** is the combination of a function bundled together with references to its surrounding state (its **lexical environment**). In other words, a closure gives an inner function access to the outer function's scope even after the outer function has finished executing and popped off the Call Stack.

```javascript
function makeCounter() {
  let count = 0; // Lexical environment variable
  return function() {
    return ++count; // Inner function forms a closure over count
  };
}

const counter = makeCounter();
console.log(counter()); // 1
console.log(counter()); // 2
```

### Memory Footprint & Garbage Collection of Closures
Usually, when a function finishes executing, its Lexical Environment is cleaned up from memory. However, if an inner function is returned and assigned to a global or long-lived variable (like `counter` above), the inner function maintains an **outer reference pointer** to the outer Lexical Environment.
- **GC Behavior:** The garbage collector cannot reclaim the outer lexical scope because a live reference (the returned inner function) still points to it.
- **Memory Leaks:** If you accidentally retain closures or keep references to returned functions in massive arrays or global variables, you prevent the GC from clearing the captured scope, leading to a memory leak. To free this memory, set the reference to `null`: `counter = null`.

---

## 3. Event Loop & Asynchronous Execution

JavaScript is **single-threaded**—the V8 engine has exactly one **Call Stack** and executes one instruction at a time on the main thread.

However, the **Node.js Runtime** is multi-threaded. It uses the **Libuv** C++ library, which provides:
1. **Host I/O APIs:** Translates operations to non-blocking OS kernel multiplexing (`epoll` on Linux, `kqueue` on macOS).
2. **Thread Pool:** By default, maintains 4 threads (configurable via `UV_THREADPOOL_SIZE`) to handle blocking tasks that the OS cannot run asynchronously (such as file I/O `fs`, cryptographic calculations `crypto`, compression `zlib`, and DNS lookups).

```
  ┌────────────────────────────────────────────────────────┐
  │                   V8 ENGINE (Main Thread)              │
  │  ┌───────────────────────┐   ┌───────────────────────┐  │
  │  │      Call Stack       │   │       Memory Heap     │  │
  │  └───────────┬───────────┘   └───────────────────────┘  │
  └──────────────┼─────────────────────────────────────────┘
                 │ (Registers Async I/O Task)
                 ▼
  ┌────────────────────────────────────────────────────────┐
  │                 LIBUV (Background Thread Pool)         │
  │   - fs (Disk I/O)         - crypto                     │
  │   - zlib                  - DNS lookups                │
  └──────────────┬─────────────────────────────────────────┘
                 │ (Pushes callback on task completion)
                 ▼
  ┌────────────────────────────────────────────────────────┐
  │                      QUEUES                            │
  │  Microtasks: process.nextTick -> Promises              │
  │  Macrotasks: Timers -> Poll -> Check                   │
  └────────────────────────────────────────────────────────┘
```

### The 6 Phases of the Libuv Event Loop
Every cycle of the event loop (a "tick") goes through these phases in order:

1. **Timers:** Executes callbacks scheduled by `setTimeout()` and `setInterval()`.
2. **Pending Callbacks:** Executes deferred I/O callbacks (e.g., TCP errors or system errors).
3. **Idle, Prepare:** Used only internally by Libuv for house-keeping.
4. **Poll:** Retrieves new I/O events. The engine will block here and wait if there are no pending timers/callbacks in other phases.
5. **Check:** Executes callbacks registered with `setImmediate()`.
6. **Close Callbacks:** Executes close event callbacks (e.g., `socket.on('close', ...)`).

### Microtask Queues vs. Macrotask Queues
After **every single phase** of the event loop, and after **each individual callback** inside a phase finishes executing, the JS engine flushes the **Microtask Queues** before moving forward.
1. **Priority 1: `process.nextTick` Queue:** Strictly flushed first.
2. **Priority 2: Promise Jobs Queue:** Flushed immediately after `process.nextTick`.
3. **Macrotasks (Task Queue):** Standard events (timers, I/O, `setImmediate`) run strictly inside their specific phase.

#### Code Tracing Example:
```javascript
setTimeout(() => console.log("Timeout"), 0);
setImmediate(() => console.log("Immediate"));
process.nextTick(() => console.log("NextTick"));
Promise.resolve().then(() => console.log("Promise"));

// Output Order:
// 1. NextTick (Immediate microtask)
// 2. Promise (Secondary microtask)
// 3. Timeout (Macrotask in Timer Phase)
// 4. Immediate (Macrotask in Check Phase)
```

---

## 4. Async/Await & Promises

### Promise States
A Promise is a state machine with three mutually exclusive states:
- **Pending:** Initial state, neither fulfilled nor rejected.
- **Fulfilled:** Operation completed successfully. Turns immutable.
- **Rejected:** Operation failed with an error. Turns immutable.

### Async/Await Compilation under the Hood
`async`/`await` is syntactic sugar. Under the hood, Babel or V8 transpiles `async` functions into **Generators** wrapped in a runner function that recursively resolves Promises. Every `await` statement yields a Promise and registers the remainder of the function as a microtask callback inside `.then()`.

```javascript
// What you write:
async function getData() {
  const result = await fetch('/api');
  console.log(result);
}

// Conceptually what V8 compiles it to:
function getDataCompiled() {
  return spawn(function* () {
    const result = yield fetch('/api');
    console.log(result);
  });
}
```

---

## 5. Memory Management & Garbage Collection

### Stack vs. Heap Allocation
- **Stack:** Local primitive values (numbers, booleans) and reference pointers. Cleaned up instantly as execution frames leave the stack.
- **Heap:** Reference types (Objects, Arrays, Functions, Closures). Dynamically allocated and managed by the Garbage Collector.

### V8 Heap Divisions
1. **New Space (Young Generation):** Highly dynamic, stores short-lived allocations. Managed by the **Scavenger (Cheney's Copying Algorithm)** which splits space into "To" and "From" halves, copying surviving objects between them and moving old survivors to the Old Space.
2. **Old Space (Old Generation):** Stores long-lived objects. Managed by the **Major GC (Mark-Sweep-Compact)** which executes:
   - **Marking:** Tracing active references starting from roots (the stack, globals).
   - **Sweeping:** Deallocating unreferenced dead items.
   - **Compacting:** Shifting live memory blocks to eliminate fragmentation.

### Common Memory Leaks in Node.js
1. **Accidental Globals:** Declaring variables without `let`/`const`, binding them to the global namespace permanently.
2. **Forgotten Timers/Intervals:** `setInterval()` holding closed scope objects in memory because the callback hasn't been cleared via `clearInterval()`.
3. **Detached Handlers & Event Listeners:** Retaining references to closed objects inside global event bus subscriptions.
4. **Closures:** Keeping inner functions that hold references to heavy variables in their parent scopes.

---

## 6. High-Impact Interview Questions & Answers

### Q1: What is the exact execution trace of the following script? Explain the microtask queuing priority.
```javascript
console.log('1');

setTimeout(() => {
  console.log('2');
  Promise.resolve().then(() => console.log('3'));
}, 0);

Promise.resolve().then(() => {
  console.log('4');
  process.nextTick(() => console.log('5'));
});

console.log('6');
```
* **Answer:**
  1. Synchronous operations run first: Prints **`1`** and **`6`**.
  2. The timer `setTimeout(..., 0)` is registered with Libuv and scheduled for the next Timer phase.
  3. The Promise `.then` is pushed to the Promise microtask queue.
  4. The main thread stack empties. The engine checks the Microtask Queue:
     - It pops the Promise job and prints **`4`**.
     - Inside, it registers `process.nextTick`, which goes to the `nextTick` queue.
     - V8 executes the `nextTick` queue immediately before leaving the microtask loop, printing **`5`**.
  5. The Event Loop enters the **Timer Phase**:
     - It pops the timer callback, executing it to print **`2`**.
     - A new Promise `.then` is registered to the microtask queue.
     - Before leaving the Timer phase, the engine flushes the microtask queue, popping the Promise and printing **`3`**.
  * **Final Output:** `1`, `6`, `4`, `5`, `2`, `3`.

### Q2: What is Event Loop Starvation, how do you cause it, and how do you resolve it?
* **Answer:** Event Loop Starvation happens when the Microtask Queue (specifically `process.nextTick`) never empties. Since the event loop is blocked from advancing to subsequent phases (like Timers or Poll) until all microtasks are cleared, background tasks like file reads or database queries are starved of execution cycles.
- **Example of Starvation:**
  ```javascript
  function starve() {
    process.nextTick(starve); // Recursive nextTick fills microtask queue endlessly
  }
  ```
- **Resolution:** Use **`setImmediate()`** instead of `process.nextTick` or recursive Promises. `setImmediate()` schedules tasks in the **Check Phase** of the Event Loop, allowing each tick to process normal I/O, timers, and callbacks before executing the next scheduled immediate task.

### Q3: How do you identify a memory leak in a running Node.js production server?
* **Answer:**
  1. **Symptoms:** Observe the process memory usage graph over time (using PM2 or AWS CloudWatch). If memory shows a constant "sawtooth" upward trend without returning to a baseline after GC, there is a leak.
  2. **Generation of Heap Snapshots:** Run the Node.js process with the `--inspect` flag or trigger snapshots programmatically using the `v8` native module:
     ```javascript
     const v8 = require('v8');
     const fs = require('fs');
     const snapshotStream = v8.getHeapSnapshot();
     snapshotStream.pipe(fs.createWriteStream('heap.heapsnapshot'));
     ```
  3. **Analysis:** Load two snapshots taken at different times under load into Chrome DevTools (Memory Tab). Perform a **Comparison** analysis:
     - Search for objects showing a high count of **Delta** allocations.
     - Inspect the **Retainers** tree to trace what active object (such as a global cache or closure) holds the memory references to the leaked elements.

### Q4: Explain the difference between `V8` Heap's Mark-Sweep and Cheney's Copying (Scavenger) Garbage Collection algorithms.
* **Answer:**
  - **Cheney's Copying (Scavenger):** Used in the **New Space**. It splits the space into equal "From" and "To" semi-spaces. Fresh allocations land in "From". During GC, live objects are copied to the contiguous "To" space (eliminating fragmentation instantly) and the labels are flipped. This is extremely fast because it only touches live objects (which are rare in the New Space).
  - **Mark-Sweep-Compact:** Used in the **Old Space** because memory size is much larger and copying everything would be too expensive. It first **marks** all objects reachable from the root. Then, it **sweeps** (frees) the unmarked dead memory. Finally, it **compacts** (slides live objects together) to eliminate internal fragmentation and preserve large contiguous memory blocks.

### Q5: Why is `UV_THREADPOOL_SIZE` important, and when should you scale it?
* **Answer:** Node.js delegates blocking tasks (like filesystem `fs` calls, crypto computations, and zlib compression) to Libuv's worker thread pool.
- **Limit:** By default, this pool is capped at only **4 threads**. If your application performs heavy cryptography (e.g., calling `bcrypt.hash` on multiple logins) or highly concurrent disk reads, these tasks will queue up waiting for an available thread, slowing down your server.
- **Scaling:** You can increase this size up to a maximum of 128 by setting the environment variable before starting Node:
  `export UV_THREADPOOL_SIZE=16`
  This should be matched to the number of physical CPU cores available to prevent OS thread thrashing.

### Q6: Why is `await` in a `forEach` loop synchronous-blocking safe or unsafe? What is the alternative?
* **Answer:** `Array.prototype.forEach` is **not** promise-aware. If you write:
  ```javascript
  items.forEach(async (item) => {
    await save(item); // Returns control to forEach instantly
  });
  ```
  The `forEach` loop executes synchronously and initiates all the async calls in parallel *without* waiting for each individual `await` to settle. It will complete executing before any of the database writes finish.
- **Alternatives:**
  1. **Sequential Execution:** Use a standard `for...of` loop, which respects `await` control flow:
     ```javascript
     for (const item of items) {
       await save(item); // Strictly sequential wait
     }
     ```
  2. **Parallel Execution:** Use `Promise.all()` to trigger all writes in parallel and wait for all of them to settle:
     ```javascript
     await Promise.all(items.map(item => save(item)));
     ```