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
* **Performance:** `useSelector` tracks selected Redux state to avoid unnecessary re-renders. `createSelector` memoizes derived data to avoid unnecessary recalculation and can be reused across components

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

* **Unidirectional (One-way):** Data flows in one direction. State updates the UI, and user actions update the state through functions or actions. This makes the flow easier to follow and debug.
  * **Use when:** State logic is complex, updates need to be predictable, or the application has many components sharing and changing state.

* **Bidirectional (Two-way):** State and UI update each other automatically. When state changes, the UI updates, and when the user changes the UI, the state updates too. It is convenient, but can be harder to trace in complex flows.
  * **Use when:** You need simple and direct synchronization between form inputs and state, especially for forms or small UI interactions.

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
  3. **Shallow Comparison:** In Zustand, use `useShallow` when selecting multiple values as an object or array to avoid re-renders caused by new references.

### Q4: When should you use local state instead of global state?
* **Answer:** Keep state local when it is only used by one component or a small component tree. Use global state only when the same state needs to be shared across unrelated parts of the application.

### Q5: When should you use React Context instead of Redux or Zustand?
* **Answer:**
  1. **React Context:** Use Context for simple shared values that do not change often, such as theme, locale, or feature flags.
  2. **Redux / Zustand:** Use Redux or Zustand when the state is more complex, changes frequently, or needs better control over subscriptions and updates.

### Q6: What is the difference between client state and server state?
* **Answer:**
  1. **Client State:** State owned by the frontend, such as modal state, theme, or local UI preferences.
  2. **Server State:** Data owned by the backend and cached on the frontend, such as users, products, or orders.
  3. **Key Difference:** Server state needs synchronization, caching, refetching, retry, and stale-data handling, while client state usually does not.

### Q7: When should you use `useReducer` instead of `useState`?
* **Answer:**
  1. **useState:** Use it for simple and independent state updates, such as toggles, inputs, or counters.
  2. **useReducer:** Use it when state logic is more complex, multiple values change together, or updates depend on different actions.
  3. **Key Benefit:** `useReducer` keeps state transition logic in one place, making complex state easier to understand and maintain.

### Q8: What is an optimistic update?
* **Answer:**
  1. **Optimistic Update:** Update the UI immediately before the server confirms the change.
  2. **Benefits:** It makes the application feel faster and more responsive.
  3. **Risk:** If the request fails, the UI should rollback or show an error.