# Memory Management & Garbage Collection in V8

Comprehensive interview study guide covering the V8 engine memory layout, heap partitions, garbage collection algorithms, and common memory leaks.

---

## 1. V8 Engine Memory Layout

The V8 engine (which powers NodeJS, Chrome, and Edge) partitions memory into two primary regions:

```
Resident Set (Total memory allocated from OS)
  ├─► Stack (Fast execution, frames, primitive local variables)
  └─► Heap (Dynamic allocations, objects, closures, functions)
        ├─► New Space (Short-lived items, managed by Scavenger)
        └─► Old Space (Long-lived items, managed by Mark-Sweep-Compact)
```

1. **Stack:** Stores local primitive variables, execution context frames, and pointers to Heap objects. Extremely fast allocation/deallocation managed automatically by the CPU.
2. **Heap:** Stores reference types like Objects, Arrays, Functions, and closures. The size of the Heap is monitored and reclaimed by the **Garbage Collector (GC)**.

---

## 2. V8 Garbage Collection Process

V8 separates the Heap into generations based on the **Generational Hypothesis**: "Most objects die young."

### 1. Minor GC (Scavenger Collection)
* **Target:** **New Space** (where new objects are allocated, typically small: 1MB to 64MB).
* **Algorithm (Cheney's Copying Algorithm):**
  * The space is split into two equal halves: **To-Space** (Active) and **From-Space** (Inactive).
  * New allocations land in To-Space.
  * During GC, active referenced objects are copied to From-Space. Unreferenced objects are discarded.
  * The roles of To-Space and From-Space are swapped.
  * If an object survives two Scavenger cycles, it is promoted (moved) to the Old Space.

### 2. Major GC (Mark-Sweep-Compact)
* **Target:** **Old Space** (where surviving long-lived data resides).
* **Algorithm:**
  1. **Marking:** Traverses the memory tree starting from "GC Roots" (Stack pointers, global objects) to identify and mark all accessible active objects.
  2. **Sweeping:** Sequentially scans unreferenced allocations, freeing their memory addresses.
  3. **Compacting:** Moves remaining fragmented active data into contiguous blocks to eliminate memory fragmentation, preventing out-of-memory errors on large allocations.

---

## 3. Common JavaScript Memory Leaks

A memory leak occurs when unneeded allocations are unintentionally kept reachable from GC roots, preventing the engine from reclaiming space.

1. **Accidental Global Variables:** Variables declared without `let`/`const` attach to the `global` (NodeJS) or `window` (browser) root, persisting forever.
2. **Uncleared Intervals & Timeouts:** Running `setInterval` holds references to outer-scope variables in its callback. If the interval is never cleared, those variables remain pinned.
3. **Out-of-DOM References:** Retaining references to deleted HTML elements in client-side arrays.
4. **Closures:** Inner functions keeping outer lexical environments in memory indefinitely.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: How does Cheney's Copying Algorithm work in V8's New Space?
* **Answer:** Cheney's algorithm partitions V8's New Space into two active regions: To-Space and From-Space. When a GC cycle is triggered, the engine starts from GC roots, identifying alive objects in To-Space and copying them directly as contiguous blocks into From-Space (which compacts them automatically). Unreferenced young objects are thrown away. Once copied, the active spaces swap names, instantly reclaiming the fragmented memory and preparing for new allocations.

### Q2: Why does V8 use a Generational Garbage Collector instead of a single sweeping process?
* **Answer:** Reclaiming memory across a massive single heap requires scanning millions of objects, causing long pauses (blocking the main JS thread). Generation-based GC relies on the fact that **most allocations are temporary and die young**. By separating memory, V8 can run a super-fast, cheap Scavenger cycle over a tiny New Space (taking <1ms). It only falls back to the heavy, resource-intensive Mark-Sweep-Compact major GC on the Old Space when heap thresholds are crossed.

### Q3: How do you trace and diagnose a memory leak in a production NodeJS application?
* **Answer:** To locate memory leaks under production loads, follow this pipeline:
  1. **Observe Heap Metrics:** Monitor RAM consumption using APM tools or NodeJS standard `process.memoryUsage().heapUsed` to identify a constant, step-like growth pattern over time.
  2. **Capture Heap Snapshots:** Run the NodeJS process with inspect flags (`node --inspect`) and use Chrome DevTools to capture multiple **Heap Snapshots** spaced minutes apart.
  3. **Compare Snapshots:** Use the DevTools Comparison view to analyze which specific constructors/classes are growing in count (often showing Uncleared Timers or leaked Closure variables), pointing to the leaking file source.
