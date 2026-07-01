# Frontend State Management

Comprehensive interview study guide covering local, global, and server states, Redux architecture, prop drilling, and unidirectional data flow.

---

## 1. Classification of State

In modern frontend applications, state is categorized into three core buckets:

| State Type | Definition | Best-Fit Tools |
| :--- | :--- | :--- |
| **Local State** | UI state restricted to a single component or immediate parent-child boundary. | `useState`, `useReducer` |
| **Global State** | Shared state across multiple unrelated component branches. | Zustand, Redux Toolkit, React Context |
| **Server State** | Cached data fetched from an external API (requires sync, retry, loading indicators). | TanStack Query (React Query), SWR |

---

## 2. Redux vs. Context vs. Zustand

### 1. React Context
* **Mechanism:** Native dependency injection mechanism.
* **Performance Trap:** When a value in a Context Provider changes, **all components subscribing to that context re-render**, regardless of whether they consume the specific field that changed. This is highly inefficient for fast-changing global states.

### 2. Redux (Unidirectional Flow)
* **Architecture:** Explicit unidirectional data flow:
  `View ──► Dispatch Action ──► Reducer ──► Store Update ──► View re-renders`
* **Performance:** Uses memoized selectors (`useSelector`) to track specific state changes, preventing redundant component re-renders.

### 3. Zustand
* **Architecture:** Simple, lightweight, store-based state management using hooks. No complex boilerplate, supports selector-based performance optimization natively.

---

## 3. Unidirectional Flow vs. Bidirectional Data Binding

```
Unidirectional (React, Redux)                Bidirectional (Angular, Vue v-model)
 ┌─────────┐      Dispatches                 ┌───────────┐  Syncs State  ┌───────────┐
 │  State  ├──────────────┐                  │           │ ◄────────────►│           │
 └────▲────┘              ▼                  │ Component │               │ DOM Input │
      │               ┌────────┐             │   State   │ ◄────────────►│  Element  │
      └───────────────┤ Action │             └───────────┘               └───────────┘
        State Update  └────────┘
```

* **Unidirectional (One-way):** State flows in one direction. Views cannot modify state directly; they must dispatch actions. This makes state updates deterministic and easy to debug.
* **Bidirectional (Two-way):** State changes automatically update the UI element, and user inputs automatically update the application state without dispatching events. Highly productive but harder to trace or debug in complex data flows.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is "Prop Drilling" and what are the best ways to resolve it?
* **Answer:** **Prop Drilling** occurs when you need to pass state through multiple layers of nested components that do not actually consume the data, just to deliver it to a deeply nested child. To resolve it:
  1. **Component Composition:** Pass the children directly (e.g., `<Parent><Child data={data} /></Parent>`), bypassing the intermediate components.
  2. **React Context:** Use context to inject state directly into the target child.
  3. **Global State Store:** Use Zustand or Redux to allow any component to subscribe to state anywhere in the tree.

### Q2: Why is TanStack Query (React Query) often preferred over standard Redux for handling API data?
* **Answer:** Standard Redux requires massive boilerplate (writing actions, thunks, reducers, and selectors) just to fetch, store, and display API data. Furthermore, Redux does not natively support server-state concepts. **TanStack Query** is built specifically for Server State. It handles caching, automatic background revalidation, query retries, request deduplication, loading/error states, and garbage collection of stale data out of the box with zero boilerplate.

### Q3: How do you prevent unnecessary re-renders when consuming global state in React?
* **Answer:**
  1. **Selector-based subscription:** In Zustand or Redux, use granular selectors (e.g., `const user = useSelector(state => state.user)`) instead of pulling the entire store object, preventing re-renders when unrelated properties change.
  2. **Context Splitting:** Split large React Context providers into smaller, focused contexts (e.g., separate `UserContext` from `ThemeContext`).
  3. **React.memo:** Wrap expensive leaf components in `React.memo` to skip re-renders if their received props remain unchanged.
