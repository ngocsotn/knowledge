# Operating System Fundamentals & Language Concurrency Models

Comprehensive, staff-level study guide covering low-level Operating System internals, CPU scheduling, synchronization primitives, system memory, disk I/O, streams, and concurrency models across various programming languages.

---

## 1. Operating System Fundamentals

### Process vs. Thread

```
Process Boundary (Virtual Address Space / Page Tables)
┌────────────────────────────────────────────────────────┐
│  Process Memory (Text, Data, BSS, Heap)                │
│                                                        │
│  Thread 1                      Thread 2                │
│  ┌────────────────────────┐    ┌────────────────────────┐│
│  │ Stack | registers | TLS│    │ Stack | registers | TLS││
│  └────────────────────────┘    └────────────────────────┘│
└────────────────────────────────────────────────────────┘
```

* **Process**: An isolated execution environment created by the OS. 
  - **Memory Space**: Has its own dedicated virtual address space mapped via page tables. Process memory is fully isolated from other processes.
  - **Resources**: Owns its own file descriptors, security contexts, environment variables, and network sockets.
  - **Creation**: Expensive (on Linux, `fork()` clones page tables, descriptors, and security tokens).
  - **Communication**: Requires Inter-Process Communication (IPC) mechanisms (Sockets, Pipes, gRPC, Shared Memory, Message Queues).
* **Thread**: The smallest schedulable unit of execution within a process.
  - **Shared Memory**: Shares the parent process’s virtual address space, heap, open file descriptors, and global variables.
  - **Private Context**: Possesses its own execution stack, CPU register state, Program Counter (PC), and Thread Local Storage (TLS).
  - **Creation**: Extremely lightweight compared to a process.
  - **Communication**: Direct and high-performance via shared memory variables (requires strict synchronization).

#### Thread Mapping Models (1:1 vs. M:N)
* **1:1 Model (Kernel Threading)**:
  - Every application/user thread maps directly to one kernel thread managed by the OS scheduler (e.g., Linux `pthreads`, Java Platform Threads, C++, Rust).
  - *Pros*: Full utilization of multi-core hardware. If one thread blocks on I/O, other threads continue running on separate cores.
  - *Cons*: Expensive creation and context switching. Scaling is limited by kernel memory structures (typically 1MB - 8MB stack per thread).
* **M:N Model (Hybrid Threading / Green Threads)**:
  - Multiplexes $M$ user-level lightweight threads on top of $N$ native kernel threads (e.g., Go Goroutines, Java 21+ Virtual Threads / Project Loom, Erlang processes).
  - *Pros*: Ultra-lightweight. Creation costs microseconds; memory usage starts at 2KB - 4KB. Millions of concurrent green threads can run in a single process. Context switches occur in user-space, avoiding costly OS context switch traps.
  - *Cons*: Extremely complex runtime scheduler. A single blocking system call (e.g., disk read) can starve the carrier kernel thread unless intercepted by the runtime and delegated dynamically.

---

### Context Switching Mechanics

A context switch is the process of storing the CPU state of a running execution unit (process/thread) so that it can be paused, and loading the saved state of another thread to resume execution.

#### Kernel-Level Context Switch Sequence:
1. **Interrupt Trigger**: A hardware timer interrupt (for preemption) or a software trap (blocking system call, yield) triggers a privilege escalation, trapping the CPU from **User Mode (Ring 3)** into **Kernel Mode (Ring 0)**.
2. **State Saving**: The OS kernel saves the CPU register values (including the Stack Pointer `ESP/RSP` and Program Counter `EIP/RIP`) of the current thread onto its dedicated **Kernel Stack**.
3. **Scheduling Phase**: The OS Scheduler runs its selection algorithm (e.g., CFS) to choose the next eligible thread from the ready queue.
4. **Memory Map Switch (Process Only)**: If the scheduler switches to a thread in a *different* process, the CPU must switch virtual memory maps. This involves updating the page directory base register (`CR3` on x86) to point to the new process’s Page Directory.
   - **Crucial Penalty**: Switching page directories flushes the **Translation Lookaside Buffer (TLB)**, which caches virtual-to-physical address translations. This causes massive L1/L2 cache misses on subsequent memory accesses.
5. **State Restoration**: The OS loads the CPU registers from the next thread's Kernel Stack.
6. **Resuming Execution**: The CPU drops privilege levels back to User Mode (Ring 3), changing the Program Counter register to point to the restored thread’s next instruction.

---

### CPU Scheduling

#### Preemptive vs. Cooperative Scheduling
* **Preemptive Scheduling**: The OS kernel controls scheduling. It forcibly preempts running threads after their assigned time slice (quantum) expires, or when a higher-priority task enters the ready queue. 
  - *Mitigates*: CPU starvation caused by rogue loops. Enforces fair CPU share.
* **Cooperative Scheduling**: The executing thread holds CPU ownership. The scheduler cannot interrupt it; the thread must explicitly yield control (via `yield`, `sleep`, or blocking I/O) to let other tasks run.
  - *Risk*: A single non-yielding CPU-bound thread can starve and freeze the entire application runtime.

#### Schedulers
* **Round Robin (RR)**: Assigns a fixed time slice (quantum) to each process in a FIFO queue. Highly fair for identical workloads but exhibits poor average turnaround times for tasks of widely varying lengths.
* **Multi-Level Feedback Queue (MLFQ)**: An adaptive scheduling system. It maintains multiple queues with different priority levels.
  - Tasks that use the CPU heavily (CPU-bound) are dynamically demoted to lower-priority queues.
  - Tasks that block frequently (I/O-bound, interactive) are promoted to high-priority, low-latency queues.
  - Prevents starvation via "priority boosting" (periodically moving all tasks to the top queue).
* **Completely Fair Scheduler (CFS / EEVDF)**: The default Linux scheduler.
  - **CFS**: Replaces fixed priorities with **virtual runtime (vruntime)** tracking. It models an ideal, multi-tasking CPU where $N$ tasks get $1/N$ of the CPU power. The scheduler stores tasks in a self-balancing Red-Black Tree indexed by `vruntime`. It always schedules the task with the *lowest* `vruntime`.
  - **EEVDF (Earliest Eligible Virtual Deadline First)**: Enhances CFS in newer Linux kernels. It dynamically schedules tasks based on "virtual lag" and eligible deadlines, improving latency bounds for interactive, real-time tasks.

---

### Concurrency & Synchronization

#### Core Concurrency Failures
* **Race Condition**: A state where the output of execution depends on the non-deterministic sequence, timing, or interleaving of concurrent execution units accessing shared resources.
* **Data Race**: A specific class of race condition where two or more threads concurrently access the same memory location, at least one access is a write, and there is no synchronization (e.g., lock, atomic) separating the accesses. Leads to undefined behavior in compiled languages.
* **Deadlock**: A permanent blocking state where a set of threads are unable to progress because each is waiting for a resource held by another thread in the set.
  - **The 4 Coffman Conditions (Must ALL hold for a deadlock to occur)**:
    1. **Mutual Exclusion**: At least one resource must be held in a non-shareable mode (only one thread can use it at a time).
    2. **Hold and Wait**: A thread must hold at least one resource while waiting to acquire additional resources held by other threads.
    3. **No Preemption**: Resources cannot be forcibly taken from a thread; they must be released voluntarily.
    4. **Circular Wait**: A closed chain of threads must exist, where each thread waits for a resource held by the next thread in the chain.
* **Starvation**: A thread is permanently denied CPU time or resource access because the scheduler or locking mechanism continuously prioritizes other threads.

#### Concurrency Primitives
* **Mutex (Mutual Exclusion)**: A locking mechanism used to serialize access to shared critical sections.
  - **Ownership**: Exclusive. Only the exact thread that locked the mutex is allowed to unlock it.
  - **Hybrid Implementation**: Modern production mutexes (like Go’s `sync.Mutex` or C++ `std::mutex`) do not immediately block threads in the kernel. They execute a fast-path **Compare-And-Swap (CAS)** atomic operation in user-space first. If the lock is held, they spin-lock briefly in CPU loops. If the lock remains held, they execute the slow-path: trapping into the kernel via a **futex (fast userspace mutex)** call to park the thread in a sleep queue until signaled.
* **Semaphore**: A synchronization primitive that manages resources using an internal counter.
  - **Binary Semaphore**: Counter ranges only between 0 and 1. Similar to a mutex, but has **no ownership restriction**; any thread can signal a binary semaphore to unlock it.
  - **Counting Semaphore**: Counter represents $N$ available slots of a shared resource.
    - `P` (Proberen / Wait): Decrements the counter. If the counter is $< 0$, the calling thread blocks until signaled.
    - `V` (Verhogen / Signal): Increments the counter. If the counter is $\le 0$, it wakes up a parked thread.
* **Atomic Operations**: Hardware-level, lock-free memory mutations executed in a single, non-interruptible CPU cycle (e.g., `CAS`, `AtomicIncrement`). Uses CPU cache-coherency protocols (like MESI) and memory fence instructions to bypass high Mutex scheduling overheads.

---

## 2. Runtimes & Language Concurrency Models

### NodeJS (Single-Threaded Event Loop & libuv)

```
[ JS Call Stack ] ◄─── executes on Single Main Thread
        │
  (Async I/O Call)
        ▼
   [ libuv ] ──► Async OS Poll (epoll/kqueue) 
        │
        ├─► [ Thread Pool ] (File System, Crypto, DNS Lookup)
        │
   (Callback Ready)
        ▼
[ libuv Event Loop Phases ] (Timers -> Pending -> Poll -> Check -> Close)
```

* **The Single-Threaded Core**: JavaScript execution in Node.js is single-threaded. This completely eliminates race conditions on JavaScript variables, CPU context-switching overhead, and complex lock mechanics.
* **libuv Multi-Threading**: Behind the scenes, Node.js delegates asynchronous blocking operations to the native C/C++ library **libuv**:
  - **Non-Blocking OS I/O**: libuv uses low-level, non-blocking kernel polling selectors (e.g., `epoll` on Linux, `kqueue` on macOS, `IOCP` on Windows) to manage active network sockets.
  - **The libuv Thread Pool**: Synchronous, blocking operations that cannot be polled asynchronously by the OS kernel (such as file system read/writes, cryptographic hashes like `bcrypt`, and DNS lookups) are delegated to an internal thread pool (default size: `4`, configurable up to `1024` via `UV_THREADPOOL_SIZE`).
* **Worker Threads (`worker_threads`)**: Allows true multi-threaded CPU execution in Node.js. Each worker thread runs its own isolated V8 engine instance, Event Loop, and context. Communication occurs via structured JSON message passing (`postMessage`) or high-performance, zero-copy shared memory using `SharedArrayBuffer` and thread-safe lock-free `Atomics`.
* **Cluster Module**: A multi-process scaling solution. Spawns multiple identical processes (using `fork()`) that share the master process's port. The master process routes incoming TCP connections to worker processes using a Round-Robin algorithm.

---

### Python (GIL & Cooperative AsyncIO)
* **The Global Interpreter Lock (GIL)**: A mutual exclusion lock in CPython designed to protect the interpreter's internal memory management (which is not thread-safe by default). The GIL ensures that only one native OS thread executes Python bytecode at any given microsecond.
  - *Impact*: Native threads (`threading` module) in Python cannot utilize multiple CPU cores for CPU-bound computations.
* **Bypassing the GIL**:
  - **Multiprocessing**: Spawns independent OS processes via the `multiprocessing` module. Each process has its own dedicated python interpreter, memory space, and GIL, enabling true parallel CPU execution (at the expense of duplicate memory usage).
  - **GIL Release**: C extensions (such as NumPy, OpenCV, or standard library cryptographic libraries) explicitly release the GIL during native CPU-intensive computations, allowing parallel multi-core math across native threads.
* **AsyncIO & Coroutines**: Single-threaded cooperative multitasking.
  - Coroutines use generator-based `async/await` state machines.
  - Coroutines voluntarily yield CPU control back to the central `asyncio` Event Loop whenever they hit an I/O wait (e.g., `await client.get()`), allowing a single thread to multiplex thousands of concurrent network connections without thread context-switch overhead.

---

### Java (Thread Pools & Virtual Threads)
* **Traditional Platform Threads (1:1 Model)**:
  - Java historically mapped `java.lang.Thread` directly to OS native threads.
  - **Executor Service**: Since native threads are expensive, Java pools them using `ExecutorService` (e.g., `ThreadPoolExecutor`). Tasks are queued in a thread-safe `BlockingQueue` and consumed sequentially by pooled native threads.
* **Virtual Threads (Project Loom, Java 21+ - M:N Model)**:
  - Extremely lightweight threads mapped $M$ to $N$ native **Carrier Threads**.
  - **Non-Blocking Yielding**: When code running inside a Virtual Thread performs a blocking I/O operation (e.g., socket read, database call, or lock acquisition), the JVM intercepts the block, unmounts the Virtual Thread's stack frame from the native Carrier Thread, and parks it in user-space.
  - The Carrier Thread is immediately freed to execute other ready Virtual Threads. Once the I/O event resolves, the Virtual Thread is remounted onto an available Carrier Thread and resumes.
  - *Result*: Allows Java applications to scale to millions of concurrent sessions using simple, synchronous programming patterns instead of highly complex reactive code (e.g., WebFlux).

---

### Go (Goroutines & The GMP Scheduler)

```
  Go GMP Scheduler Context
  
     M1 (OS Thread) ◄─── bound ─── P1 (Logical Processor)
                                    │
                               [Run Queue] (Local)
                                    ├── G1 (Goroutine - Active)
                                    ├── G2 (Goroutine - Ready)
                                    └── G3
```

* **Goroutines**: Go's lightweight user-space threads. They start with an initial dynamic stack of only 2KB that grows or shrinks as needed in the contiguous heap, completely avoiding the 1MB static OS thread stack penalty.
* **The GMP Scheduling Engine**:
  - **G (Goroutine)**: Represents the goroutine runtime structure, stack, and program counter.
  - **M (Machine)**: Represents a native OS kernel thread managed by the OS scheduler.
  - **P (Processor)**: Represents a logical processor or scheduling resource context. The count of $P$ is capped by `GOMAXPROCS` (typically matching the physical CPU core count).
* **Work Stealing**: If local run-queues associated with processor $P1$ become empty, machine $M1$ attempts to steal half of the goroutines from another processor $P2$'s local queue, or pulls from the shared global run-queue.
* **Syscall Interception (Preemption)**: When Goroutine `G1` executes a blocking system call (e.g., synchronous disk write):
  - The Go runtime detaches the physical machine `M1` executing `G1` from logical processor `P1`.
  - `P1` is assigned to a new or cached native thread `M2` to continue executing other queued goroutines (`G2`, `G3`) in parallel.
  - Once the blocking syscall completes, `G1` is placed back onto a logical processor's run-queue, and `M1` is parked in a thread pool.

---

### .NET (Task-based Asynchronous Pattern - TAP)
* **async/await Compilation**: The C# compiler transforms `async` methods into complex state machines. When an `await` instruction is encountered, the compiler generates a continuation callback and yields control of the current execution thread back to the caller.
* **The Task Scheduler & CLR Thread Pool**:
  - Upon completion of the awaited task, the **Task Scheduler** executes the continuation.
  - .NET manages a highly optimized native thread pool. The thread pool engine uses an internal **Hill-Climbing Algorithm** to dynamically adjust the active thread count based on real-time throughput metrics, scaling up when workloads are high and pruning threads to mitigate context-switch overhead when idle.

---

## 3. System Memory Management

### Stack vs. Heap

```
Virtual Memory Address Space
┌──────────────────────────────────────┐
│  Stack (Fast, LIFO, linear, down)    │
│    │                                 │
│    ▼                                 │
│    ▲                                 │
│    │                                 │
│  Heap (Dynamic, slow, random, up)    │
└──────────────────────────────────────┘
```

| Dimension | Stack Memory | Heap Memory |
| :--- | :--- | :--- |
| **Allocation** | **Static/Linear**: Allocated downward sequentially via a hardware Stack Pointer register. | **Dynamic**: Allocated randomly across free memory pools; requires block-searching algorithms. |
| **Speed** | **Incredibly Fast**: Allocation is a single CPU register decrement. | **Slower**: Overhead from memory managers, lock contention, and pool search. |
| **Lifetime** | **Deterministic**: Tied strictly to the function execution block scope (unwound on exit). | **Dynamic**: Managed manually by developer or reclaimed via Garbage Collectors. |
| **Fragmentation** | **Zero**: Continuous linear allocation prevents memory gaps. | **High**: Random allocations/deallocations lead to memory gaps. |
| **Cache Locality** | **Excellent**: High spatial and temporal locality due to packed layout. | **Poorer**: Pointer chasing across disjointed addresses causes cache misses. |
| **Limit Failure** | **Stack Overflow**: Exceeding local frame limits (typically 1MB - 8MB) crashes the process. | **Out of Memory (OOM)**: Exceeding physical/virtual RAM limits triggers OS termination. |

---

### Garbage Collection (GC) Mechanics

#### A. Tracing (Mark-and-Sweep)
The dominant GC paradigm (used by JVM, V8, CLR, Go).
1. **Mark Phase**: The GC pauses mutators (application threads) or uses concurrent barrier markers. Starting from root references (stack registers, global variables, static variables), it traverses the active object reference graph. It sets a "reached" bit on all reachable objects.
2. **Sweep Phase**: Scans the entire heap. The memory addresses of objects without the "reached" bit are reclaimed and added back to free-allocation lists.
3. **Compact Phase**: Moving GC. Re-aligns surviving active objects into contiguous memory blocks. This completely eliminates fragmentation but introduces **Stop-the-World (STW)** pauses because the GC must update every pointer address in application threads to point to the object's new physical memory location.

#### B. Reference Counting
Used natively in CPython, Swift, PHP.
* **Mechanism**: Every heap object has an internal reference counter field.
  - When an object reference is copied, the counter increments.
  - When a reference goes out of scope or is deleted, the counter decrements.
  - The millisecond the counter hits `0`, the memory is immediately reclaimed.
* **The Cyclic Reference Trap**:
  - If Object A references Object B, and Object B references Object A, their counters will never drop to 0, causing a memory leak.
  - *Fix*: Requires a secondary periodic Mark-and-Sweep tracing pass (CPython) or forcing developers to explicitly manage cycle boundaries using `weak` references (Swift).

#### C. Generational Garbage Collection
Optimized for the **Weak Generational Hypothesis**: most objects die shortly after allocation.
* **Heap Segmentation**: Divides the heap into generational segments:
  - **Young Generation (Eden & Survivor Spaces)**: Newly allocated objects go here. This region undergoes frequent, highly optimized, copy-based collections (Minor GC).
  - **Old Generation**: Objects that survive a specified number of Minor GC passes are promoted to the Old Generation. This region is collected less frequently (Major GC / Full GC) as it is significantly larger and computationally expensive.

---

## 4. Disk I/O & Buffering Architectures

### Sequential vs. Random I/O
* **Sequential I/O**: Reading or writing contiguous data blocks on disk.
  - *Performance*: Extremely high.
  - On spinning mechanical Hard Drives (HDD), it bypasses slow mechanical arm seek times and disk rotations.
  - On Solid State Drives (SSD), it maximizes the Flash Controller's internal parallel bus pipelines.
* **Random I/O**: Accessing scattered sectors across arbitrary disk addresses.
  - *Performance*: Poor.
  - On HDDs, it requires physical seek operations (limiting throughput to ~100-200 IOPs).
  - On SSDs, it incurs write amplification penalties and heavy address-translation overhead in the Flash Translation Layer (FTL).

---

### OS Page Cache (Buffer Cache)

To bridge the performance gap between microsecond CPU execution and millisecond disk storage, the OS kernel intercepts file operations using physical RAM as a **Page Cache**:

```
 [ User Space Application ] ──► System Call (read/write)
                                      │
                                      ▼
                      [ OS Kernel Page Cache (RAM) ]
                               │             ▲
                            (Miss)         (Hit: Bypass disk entirely)
                               ▼             │
                      [ Physical Storage (HDD/SSD) ]
```

* **Read Path**: The kernel reads data in chunks of page sizes (typically 4KB). If the requested block exists in the Page Cache (cache hit), the kernel copies the data directly into user-space RAM, completely bypassing slow storage disk access.
* **Write Path**: When an application writes data, the kernel writes the changes into the Page Cache first (marking the modified pages as **dirty pages**). The write system call returns immediately. An asynchronous kernel process (e.g., `pdflush` or `writeback`) later groups and flushes these dirty pages down to physical disk.
* **Durability Guarantee (`fsync`)**: Since data in the Page Cache is volatile, a sudden power failure before a background flush causes data loss. Applications (like Databases) execute the `fsync()` system call to block execution until the OS forcibly flushes specified dirty pages to persistent physical disk sectors.

---

### File System Buffer Pools (User-Space DB Engines)
High-performance database engines (e.g., PostgreSQL, MySQL InnoDB) bypass or augment the generic OS Page Cache using their own user-space **Buffer Pools**.
- **The Problem with OS Cache**: The OS uses general-purpose cache eviction policies (such as Least Recently Used - LRU). This is highly inefficient for databases where index-tree pages and full-table-scan pages require different eviction heuristics.
- **The Buffer Pool Solution**: DB engines allocate a massive static block of user-space RAM. They implement customized, context-aware page eviction policies (such as 2Q or clock-sweep) optimized for database access patterns, guaranteeing that critical index root nodes are pinned permanently in memory.

---

## 5. Streams & Backpressure Management

### Stream I/O
Stream I/O processes data **chunk-by-chunk** as it arrives, instead of loading the entire payload into physical memory.
- *Advantage*: Enables processing of multi-gigabyte files or infinite data sources (e.g., video feeds, log pipelines) on devices with minimal RAM footprint.

---

### Backpressure Mechanics & Flow Control

Backpressure is a feedback mechanism where a slow data **Consumer** signals a fast data **Producer** to slow down or pause its rate of emission.

```
                  Unmanaged Buffer (No Backpressure)
[ Producer ] ───(Sends 1000 items/sec)───► [ Buffer ] ───(Consumes 10 items/sec)───► [ Consumer ]
                                             │
                                             ▼ (OOM / Crash)

                   Managed Flow Control (With Backpressure)
[ Producer ] ◄───(Only sends N requested)─── [ Consumer (Specifies Request N) ]
```

* **The Buffer Overflow Trap**: If the Producer generates data faster than the Consumer can process it, and there is no flow control, the intermediate buffer/queue grows infinitely. This leads to high memory utilization, latency spikes, and eventually crashes the application via Out-Of-Memory (OOM) failures or drops packets.

#### Production Solutions
1. **TCP Sliding Window**: Hardware-level backpressure. The TCP receiver includes a **Receive Window (RWIN)** field in its packet headers returned to the sender. This tells the sender exactly how much empty space remains in the receiver's hardware buffer. If RWIN drops to `0`, the sender's physical TCP network layer immediately halts transmission until the receiver consumes data and advertises a non-zero RWIN.
2. **Reactive Streams Protocol (Pull-Based Flow Control)**: Software-level backpressure (used in RxJava, Spring Project Reactor, Akka Streams).
   - Instead of the Producer pushing data blindly, the Consumer controls the pipeline using a **Subscription**.
   - The Consumer explicitly requests a precise count of items: `subscription.request(N)`.
   - The Producer is strictly barred from emitting more than $N$ items until the Consumer processes the batch and sends a subsequent `request(M)` signal.
