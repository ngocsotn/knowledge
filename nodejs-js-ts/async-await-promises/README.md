# Async/Await & Promises

Comprehensive interview study guide covering JavaScript Promises, asynchronous control flow, error handling, and parallel concurrency methods.

---

## 1. Promises: The State Machine

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

## 2. Advanced Promise Concurrency APIs

When managing multiple parallel asynchronous tasks, JavaScript provides four core execution methods:

| Method | Behavior | Success Condition | Failure Condition |
| :--- | :--- | :--- | :--- |
| **Promise.all()** | Runs all in parallel. All-or-nothing. | Resolves when **all** promises fulfill. | Rejects immediately when **any** promise rejects. |
| **Promise.allSettled()** | Runs all in parallel. Never rejects. | Resolves when **all** promises settle (either fulfill or reject). | Never rejects. Returns array of `{status, value/reason}`. |
| **Promise.race()** | Runs all in parallel. First responder wins. | Resolves if the **first** settled promise fulfills. | Rejects if the **first** settled promise rejects. |
| **Promise.any()** | Runs all in parallel. First success wins. | Resolves if **any** promise fulfills. | Rejects (with `AggregateError`) only if **all** fail. |

---

## 3. Async / Await Syntactic Sugar

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

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between Promise.all() and Promise.allSettled()?
* **Answer:** **Promise.all()** is an all-or-nothing operation. If any promise in the array fails, the entire batch rejects immediately, discarding the successes of other resolved promises. **Promise.allSettled()** is resilient; it waits for every promise to complete, whether it succeeds or fails, and returns a structured array detailing the outcome (value or error reason) for every single element, ensuring no results are lost.

### Q2: What does `await` actually do to the execution of code below it?
* **Answer:** When JS encounters an `await <promise>` statement, it pauses the execution of the current `async` function. The code below the `await` statement is packaged as a **micro-task callback** (equivalent to a `.then` block) and queued in the micro-task queue. Execution control is immediately yielded back to the main thread's Call Stack to handle other events. Once the awaited promise settles, the micro-task executes, resuming the function block.

### Q3: Why is error handling different between Promises and Async/Await?
* **Answer:** With standard Promises, errors are propagated down the chain and must be caught using the `.catch(err)` method. If neglected, they trigger an `"UnhandledPromiseRejection"` warning. With `async/await`, asynchronous errors act like native synchronous exceptions, allowing you to use standard `try / catch` blocks. This unifies synchronous and asynchronous error handling into a single, clean code structure.
