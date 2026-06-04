# Scope & Closures in JavaScript

Comprehensive interview study guide covering execution contexts, lexical environments, scope chains, closures, and practical memory management.

---

## 1. Context, Scope, and Lexical Environments

To understand closures, you must first understand how JavaScript executes code:

* **Execution Context:** The wrapper environment where code is executed. It contains the active variables, arguments, and the `this` binding.
* **Lexical Environment:** The physical structure in the JS engine that maps variable names to values. It consists of:
  1. **Environment Record:** The actual storage map of variables.
  2. **Outer Reference:** A pointer to the parent outer Lexical Environment (the Scope Chain).

---

## 2. What is a Closure?

A **Closure** is the combination of a function bundled together with references to its surrounding state (its **Lexical Environment**).

> **In plain terms:** A closure allows an inner function to retain access to variables, arguments, and parameters declared in its parent outer function, even after that parent outer function has finished executing and returned.

### Code Demonstration
```javascript
function createCounter() {
  let count = 0; // Lexical variable
  
  return function increment() {
    count++; // Accesses outer scope variable
    return count;
  };
}

const counter = createCounter();
console.log(counter()); // 1
console.log(counter()); // 2
```
* **Why this works:** When `createCounter` returns, its execution context is popped off the Call Stack. However, because the returned `increment` function maintains an outer reference to `createCounter`'s Lexical Environment, the JS garbage collector cannot delete `count` from memory.

---

## 3. Practical Use Cases

1. **Data Encapsulation / Private Variables:**
   * JavaScript class fields were historically public. Closures let you enforce strict private state:
     ```javascript
     function User(name) {
       return {
         getName: () => name // name cannot be accessed or changed directly from outside
       };
     }
     ```
2. **Function Currying & Partial Application:**
   * Customizing function setups (e.g., `const add5 = add(5); console.log(add5(10));`).

---

## 4. Closure Memory Leaks

Because closures retain references to outer scopes, they can prevent large objects from being garbage collected.

* **The Trap:** If an inner closure references even a *single* variable of a massive outer scope, the **entire outer scope** remains pinned in memory.
* **The Fix:** Explicitly set large unneeded outer variables to `null` inside or after execution, or avoid nesting functions unnecessarily.

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: Explain closures to a non-technical person.
* **Answer:** Imagine you write a letter in a room and seal it in an envelope. You then walk out of that room and close the door. Even though you are no longer in that room, you carry the envelope with you, giving you permanent access to the secrets written inside no matter where you travel. In programming, the room is the parent function, and the envelope is the inner function (the closure) that carries the parent function's variables with it wherever it is called.

### Q2: What is the output of this classic loop interview question, and how do you fix it?
```javascript
for (var i = 1; i <= 3; i++) {
  setTimeout(() => console.log(i), 1000);
}
```
* **Answer:** The output is **`4, 4, 4`** (printed 3 times). Because `var` is function-scoped (not block-scoped), a single variable `i` is shared across all iterations. By the time the async `setTimeout` callbacks execute (1 second later), the loop has finished, and `i` has resolved to `4`.
* **The Fix:**
  1. Change `var i` to `let i`. Because `let` is block-scoped, a brand-new variable scope binding is created for every loop iteration.
  2. Use an IIFE (Immediately Invoked Function Expression) closure to freeze the current value of `i` inside a parameter scope.

### Q3: How do closures cause memory leaks, and how do you prevent them?
* **Answer:** Closures cause memory leaks when they hold references to large outer scope objects or DOM nodes that are no longer needed by the application. Because the inner function is kept alive (e.g., as an event listener), the garbage collector cannot reclaim any variables in the parent lexical environment. Prevent this by removing inactive event listeners, clearing intervals, and explicitly setting variables to `null` once they are out of scope.
