# NodeJS Event Loop & Libuv

Comprehensive interview study guide covering the NodeJS Event Loop architecture, phases, task queues, and asynchronous processing.

---

## 1. Meaning & Single-Threaded Architecture

NodeJS is often described as **single-threaded**, which means that its main thread executes JavaScript code in a single sequential flow (called the **Call Stack**).

However, NodeJS is only single-threaded *for user JavaScript execution*. Under the hood, NodeJS uses the **Libuv** C++ library, which provides a thread pool (default size: 4 threads, configurable via `UV_THREADPOOL_SIZE`) to handle heavy asynchronous system tasks such as disk I/O, cryptographic operations (`crypto`), and compression (`zlib`).

---

## 2. The Phases of the Event Loop

The Event Loop is a continuous cycle executed by Libuv that checks for completed asynchronous tasks and schedules their callbacks to run on the main JS thread.

```
                    ┌───────────────────────────────┐
                    │      1. Timers Phase          │ ◄── setTimeout, setInterval
                    └───────────────┬───────────────┘
                    ┌───────────────▼───────────────┐
                    │      2. Pending Callbacks     │ ◄── TCP errors, OS anomalies
                    └───────────────┬───────────────┘
                    ┌───────────────▼───────────────┐
                    │      3. Poll Phase            │ ◄── incoming requests, I/O callbacks
                    └───────────────┬───────────────┘
                    ┌───────────────▼───────────────┐
                    │      4. Check Phase           │ ◄── setImmediate
                    └───────────────┬───────────────┘
                    ┌───────────────▼───────────────┐
                    │      5. Close Callbacks       │ ◄── socket.on('close')
                    └───────────────────────────────┘
```

### 1. Timers
* Executes callbacks scheduled by `setTimeout()` and `setInterval()` whose expiration threshold has passed.
### 2. Pending Callbacks
* Executes system-related callbacks, such as TCP connection errors.
### 3. Idle, Prepare
* Used internally by NodeJS; developers do not interact with this phase.
### 4. Poll
* Retrieves new I/O events (incoming network requests, file reads). The event loop will block here waiting for I/O if no other callbacks are queued.
### 5. Check
* Executes callbacks scheduled by `setImmediate()`.
### 6. Close Callbacks
* Executes callbacks for closed resources (e.g., `socket.on('close', ...)`).

---

## 3. Macro-Tasks vs. Micro-Tasks

Callbacks are divided into two priority queues that execute during the Event Loop:

1. **Macro-tasks (Tasks):** Standard callbacks like `setTimeout`, `setInterval`, `setImmediate`, and I/O tasks.
2. **Micro-tasks:** High-priority operations including `process.nextTick()` and **Promise resolve/reject callbacks** (`.then`, `.catch`, `await`).

### The Golden Rule of Micro-tasks
Micro-tasks do **not** run in a specific Event Loop phase. Instead, **the Micro-task queue is flushed immediately after the current JavaScript execution finishes, before moving to the next Event Loop phase, and between every single macro-task.**

*Note:* `process.nextTick()` has higher priority than standard Promises and is executed *before* the Promise micro-task queue.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between setImmediate() and setTimeout(fn, 0)?
* **Answer:**
  * `setTimeout(fn, 0)` is designed to run in the **Timers Phase** once its timer expires. However, because of OS clock precision and Event Loop start times, there is a minimum delay of ~1ms, meaning its execution depends on event loop latency.
  * `setImmediate(fn)` is designed to execute immediately in the **Check Phase**, which occurs directly after the Poll phase.
  * *Context Rule:* If called inside an I/O cycle (e.g., inside `fs.readFile`), `setImmediate` is guaranteed to execute *before* `setTimeout(fn, 0)` because the Event Loop enters the Check phase directly after completing I/O.

### Q2: What is Libuv, and does NodeJS execute all asynchronous operations in its thread pool?
* **Answer:** Libuv is the C/C++ library that manages the NodeJS Event Loop and asynchronous tasks. It maintains a **thread pool** to handle blocking, synchronous-by-design system operations like file I/O (`fs`), cryptography (`crypto.pbkdf2`), and DNS lookups. However, **network I/O (like HTTP fetch/sockets) does NOT use the thread pool**. Instead, Libuv registers network sockets directly with the operating system's native non-blocking multiplexers (like `epoll` on Linux, `Kqueue` on macOS, or `IOCP` on Windows), keeping execution extremely lightweight and saving thread overhead.

### Q3: What happens if you run an infinite loop (like `while(true)`) in a NodeJS server?
* **Answer:** Because user JavaScript executes on a single main thread, running an infinite loop blocks the Call Stack completely. The main thread will never yield back to the Libuv Event Loop. As a result, the Event Loop cannot advance to poll for new network events, causing the entire server to freeze, drop incoming connections, and become completely unresponsive.
