# Async/Await & Promises

Comprehensive interview study guide covering the paradigms of synchronous and asynchronous programming, process vs. thread concurrency, and the evolution of JS/TS asynchronous control flows (Callbacks, Promises, Async/Await, and Defer).

---

## 1. Synchronous vs. Asynchronous Programming

At its core, programming execution paradigms dictate how tasks are ordered and how CPU execution time is managed relative to slow hardware inputs/outputs (I/O like network, disk, DB).

```
[Synchronous Execution]
Task A ──────────────────► [Blocks CPU] ──► Task B ──────────────────► [Blocks CPU]

[Asynchronous Execution]
Task A ──► [Starts I/O] ──────────────────────────┬──► Callback A Runs (Resume)
           Task B ──► [Starts I/O] ──► Task C ────┘
```

* **Synchronous Programming (Blocking):**
  * **Mechanism:** Instructions execute sequentially, one after another, in a strict blocking fashion.
  * **Behavior:** When a task performs a blocking operation (e.g., synchronous file read, synchronous network call, heavy CPU computation), the executing thread stalls and halts all forward progress until that operation completes.
  * **Resource Cost:** Wasteful. The execution thread remains idle during I/O waits, consuming OS thread resources and stack memory without doing active work.
* **Asynchronous Programming (Non-blocking):**
  * **Mechanism:** Tasks can be triggered to run out-of-order. When an asynchronous task is initiated, the executing thread does not wait. Instead, it registers a continuation mechanism (callback, promise resolution, event handler) and immediately continues executing other subsequent tasks.
  * **Behavior:** Once the long-running asynchronous operation settles (success or failure), the runtime notifies the host process, scheduling the registered callback to execute.
  * **Resource Cost:** Highly efficient. Allows a single execution thread to multiplex thousands of active concurrent I/O operations without needing thousands of physical OS threads.

---

## 2. Concurrency Primitives: Process vs. Thread

Operating systems handle concurrent task execution by managing two physical/virtual primitives: **Processes** and **Threads**.

| Dimension | Process | Thread |
| :--- | :--- | :--- |
| **Definition** | An isolated instance of an active computer program run by the operating system. | The smallest dispatchable unit of execution inside a Process. |
| **Memory Isolation** | **Complete**. Has private virtual address space, heap, stack, page tables, and file descriptors. Cannot access another process's memory. | **Shared**. Shares the parent process's heap, static memory, address space, and file descriptors. Each thread has only its private **Stack**. |
| **Communication** | **Heavy**. Requires explicit Inter-Process Communication (IPC): UNIX sockets, pipes, TCP/IP, or explicitly shared OS memory segments. | **Instant**. Reads/writes shared variables directly on the shared process heap. Requires synchronization (Mutex/CAS) to prevent race conditions. |
| **Context Switch Cost**| **High**. OS must swap virtual page tables, flush hardware CPU caches, and clear the Translation Lookaside Buffer (TLB). | **Low**. Thread context switches preserve the active virtual memory page tables and CPU cache state; only registers and stack pointers are swapped. |
| **System Stability** | **High**. If a process experiences a segmentation fault or crashes, the OS isolates it; other system processes are unaffected. | **Low**. If a single thread crashes (e.g., via a segmentation fault or unhandled native exception), the entire parent process crashes. |

---

## 3. Promises: The State Machine

A `Promise` is an object representing the eventual completion (or failure) of an asynchronous operation. A promise behaves like a state machine with three disjoint states:

```
                       ┌───────────────┐
                       │    Pending    │ (Initial state)
                       └───────┬───────┘
            ┌──────────────────┴──────────────────┐
            ▼                                     ▼
    ┌───────────────┐                     ┌───────────────┐
    │   Fulfilled   │ (.then)             │   Rejected    │ (.catch)
    └───────────────┘                     └───────────────┘
```

* **Rules:**
  * A promise transitions from `Pending` to either `Fulfilled` (success) or `Rejected` (failure).
  * This transition is **immutable** and permanent. Once resolved or rejected, a promise cannot change states again.

---

## 4. Advanced Promise Concurrency APIs

When managing multiple parallel asynchronous tasks, JavaScript provides four core execution methods:

| Method | Behavior | Success Condition | Failure Condition |
| :--- | :--- | :--- | :--- |
| **Promise.all()** | Runs all in parallel. All-or-nothing. | Resolves when **all** promises fulfill. | Rejects immediately when **any** promise rejects. |
| **Promise.allSettled()** | Runs all in parallel. Never rejects. | Resolves when **all** promises settle (either fulfill or reject). | Never rejects. Returns array of `{status, value/reason}`. |
| **Promise.race()** | Runs all in parallel. First responder wins. | Resolves if the **first** settled promise fulfills. | Rejects if the **first** settled promise rejects. |
| **Promise.any()** | Runs all in parallel. First success wins. | Resolves if **any** promise fulfills. | Rejects (with `AggregateError`) only if **all** fail. |

---

## 5. Async / Await Syntactic Sugar

The `async` and `await` keywords are syntactic sugar built on top of standard native Promises, making asynchronous code read like synchronous sequential statements.

* **Under the hood:** `async/await` uses **Generators** and **yield** steps to yield execution control back to the event loop while waiting for micro-tasks to settle.
* **Syntax conversion:**
  ```javascript
  // Traditional Promise Chaining
  function fetchUser() {
    return getUser().then(user => user.name);
  }

  // Modern Async/Await equivalent
  async function fetchUser() {
    const user = await getUser();
    return user.name;
  }
  ```

---

## 6. The Evolution of Asynchronous JS/TS: Callback, Promise, Async/Await, and Defer

Asynchronous patterns in JavaScript and TypeScript have evolved to optimize readability, error resilience, and execution flow.

```
Callbacks ──────────► Promises ──────────► Async/Await ──────────► Defer Patterns
(Nesting, IOC)        (Flat Chaining)      (Try/Catch, Yield)      (Decoupled Resolution)
```

### A. Callbacks (Legacy Paradigm)
* **Mechanics:** A function passed as an argument to another function, invoked once the asynchronous operation finishes.
* **The Pitfalls:**
  * **Callback Hell:** Deeply nested callbacks make code unreadable and unmaintainable.
  * **Inversion of Control (IOC):** You surrender control of your function's execution to a third-party library, trusting it to invoke your callback exactly once, at the correct time, with the correct parameters (a major reliability risk).
  * **Complex Error Handling:** Errors must be handled inside each nesting layer individually (often via error-first callback conventions, e.g., `(err, data) => {}`).

### B. Promises (ES6/ES2015 Standard)
* **Mechanics:** Returns an object instantly representing the asynchronous value state.
* **The Wins:**
  * Resolves the Inversion of Control issue by returning control back to your code—you decide when and how to attach `.then()` handlers.
  * Flattens nesting into clean, linear chains of `.then()` and `.catch()`.
  * Centralizes error propagation; unhandled errors bubble up to the nearest `.catch()` block.

### C. Async/Await (ES2017 standard)
* **Mechanics:** Integrates generator-like suspension into standard functions.
* **The Wins:**
  * Eliminates boilerplate `.then()` wrapper noise, allowing asynchronous code to be written with a clean, linear, synchronous look.
  * Standardizes error handling using normal `try/catch` blocks.
  * Pause points are scoped to the function; the main event loop is never blocked.

### D. Defer Patterns
In JS/TS, "defer" appears in two distinct contexts: the **Deferred Object Pattern** (asynchronous logic) and **HTML Script Defer** (browser loading).

#### 1. The Deferred Promise Pattern (Software Architecture)
* **Concept:** Decouples the promise state machine from the executor function.
* **Mechanics:** In standard Promises, you must resolve or reject inside the constructor block:
  ```javascript
  const p = new Promise((resolve, reject) => {
    // Execution and resolution logic must live inside this closure!
  });
  ```
  A **Deferred** pattern extracts the control handles (`resolve` and `reject`) onto a separate object, allowing any outside code to resolve or reject the promise at any time:
  ```javascript
  class Deferred {
    constructor() {
      this.promise = new Promise((resolve, reject) => {
        this.resolve = resolve;
        this.reject = reject;
      });
    }
  }

  // Usage
  const deferred = new Deferred();
  
  // Any external asynchronous event can trigger resolution later:
  ws.on('message', (data) => deferred.resolve(data));
  
  await deferred.promise;
  ```

#### 2. HTML Script Loading: Default vs. Async vs. Defer (Browser Concurrency)
* **`<script src="app.js">` (Blocking):** HTML parsing halts immediately. The script file is downloaded and executed synchronously. HTML parsing resumes only after execution completes.
* **`<script async src="app.js">` (Asynchronous / Non-Deterministic):** Script download happens asynchronously in parallel with HTML parsing. The moment the file is downloaded, **HTML parsing blocks** to execute the script. Execution order is non-deterministic (first finished downloading runs first).
* **`<script defer src="app.js">` (Deferred / Sequential):** Script download happens asynchronously in parallel with HTML parsing. Execution is strictly deferred until **HTML parsing completes**. Script execution order matches their position in the HTML document, making it highly safe and predictable.

---

## 7. Popular Interview Questions & High-Impact Answers

### Q1: What is the fundamental difference between Synchronous and Asynchronous programming?
* **Answer:** **Synchronous programming** is blocking and sequential; the execution thread stops and waits for a task (such as database queries or disk writes) to fully complete before starting the next one, wasting CPU cycles on idle waits. **Asynchronous programming** is non-blocking; the execution thread initiates a task, registers a continuation callback/promise, and immediately continues executing other instructions. The callback is executed once the background operation completes, enabling massive resource efficiency.

### Q2: Compare Callbacks, Promises, and Async/Await. How do they handle errors and Inversion of Control?
* **Answer:**
  * **Callbacks** require passing a function to an external routine. This leads to **Inversion of Control** (yielding execution trust to third-party code) and nested "Callback Hell." Error handling is manual and repetitive (error-first pattern).
  * **Promises** resolve Inversion of Control by immediately returning a state-machine object under your control, allowing clean method chaining via `.then()`. Errors bubble up automatically and are handled centrally via `.catch()`.
  * **Async/Await** is syntactic sugar over Promises using Generators under the hood. It allows asynchronous code to look synchronous, simplifying readability. It handles errors using native `try/catch` blocks, merging synchronous and asynchronous error logic.

### Q3: What is a "Deferred" object pattern, and how is it different from a standard Promise?
* **Answer:** A standard **Promise** wraps its asynchronous execution block inside its constructor callback, making its resolution logic self-contained within that closure. A **Deferred** object pattern decouples the state machine from the execution logic by exposing the `resolve` and `reject` functions as properties of the object itself. This allows external, decoupled parts of an application (e.g., event handlers or WebSocket listeners) to manually trigger the promise's completion or rejection from outside its original scope.

### Q4: What is the difference between a Process and a Thread? How does Node.js utilize them?
* **Answer:** A **Process** is a fully isolated instance of an active application with its own dedicated virtual memory space, heap, stack, and descriptors, requiring heavy IPC for communication. A **Thread** is the smallest executable unit within a process, sharing the parent process's heap, descriptors, and memory space, making communication near-instantaneous but introducing crash vulnerability (one thread crash takes down the entire process). **Node.js** executes JavaScript single-threaded on its main event loop, but utilizes Libuv’s multi-threaded C++ thread pool to execute blocking OS tasks (like cryptography and file I/O) in the background without blocking execution.

### Q5: What is the difference between `<script>`, `<script async>`, and `<script defer>`?
* **Answer:**
  * **`<script>`** blocks HTML parsing immediately to download and execute the script.
  * **`<script async>`** downloads the script in parallel with HTML parsing but immediately halts parsing to execute it once downloaded, making execution order unpredictable.
  * **`<script defer>`** downloads the script in parallel with HTML parsing but waits to execute it until HTML parsing is fully complete, guaranteeing sequential execution in the order they appear.

