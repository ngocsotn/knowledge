# Advanced TypeScript Type System

Detailed study guide for advanced TypeScript type operations, compilation phases, structural typing, and covariance/contravariance.

---

## 1. Type Aliases vs. Interfaces

While both declare structured shapes, they have fundamental behavioral and structural differences:

| Feature | Interface | Type Alias |
| :--- | :--- | :--- |
| **Declaration Merging** | **Yes:** Multiple declarations with the same name merge their fields. | **No:** Duplicate type definitions throw a compile-time error. |
| **Union & Tuples** | **No:** Cannot represent unions (`A \| B`) or tuples directly. | **Yes:** Can represent unions, intersections, primitives, and tuples. |
| **Extensibility** | Extended using the `extends` keyword. | Extended using intersection operators (`&`). |
| **Object Representation** | Limited strictly to object shapes and functions. | Can represent any primitive, array, tuple, union, or object. |

### Code Example of Declaration Merging
```typescript
interface User { name: string; }
interface User { age: number; }

// Resulting type is merged: { name: string; age: number; }
const user: User = { name: "Alice", age: 30 };
```

---

## 2. The TypeScript Compiler (`tsc`) & Compilation Phases

The TS compiler is a multi-phase pipeline that translates TypeScript code into pure JavaScript, while verifying type safety.

```
Source Code ──► Scanner (Tokens) ──► Parser (AST) ──► Binder (Symbols)
                                                         │
  ┌──────────────────────────────────────────────────────┘
  ▼
Type Checker (Verifies types) ──► Emitter (Generates JS & .d.ts)
```

1. **Scanner:** Converts raw source code characters into a linear stream of syntax **tokens**.
2. **Parser:** Takes tokens and builds a tree structure called the **Abstract Syntax Tree (AST)**.
3. **Binder:** Traverses the AST and creates a map of **Symbols** (linking declarations across files to their physical scope).
4. **Type Checker:** The core engine. It checks AST nodes against the Symbol map to verify type safety and compatibility.
5. **Emitter:** Converts the AST into output JavaScript files (`.js`), Source Maps (`.map`), and type definitions (`.d.ts`), ignoring types entirely in the output (Type Erasure).

---

## 3. Structural (Duck) Typing vs. Nominal Typing

- **Nominal Typing (e.g., Java, C#):** Type compatibility is determined strictly by the class name/declaration. If two classes have identical fields but different names, they are not compatible.
- **Structural Typing (TypeScript):** Type compatibility is determined solely by the shape and members of the type. If an object has the required fields, it is accepted (popularly known as **"Duck Typing"**: *If it walks like a duck and quacks like a duck, it is a duck*).

### Emulating Nominal Types (Branding/Opaque Types)
To prevent runtime bugs where structurally identical identifiers are mistakenly interchanged (e.g., passing a `UserId` into a function expecting a `ProductId`), we employ **Type Branding**:

```typescript
type Brand<K, T> = K & { __brand: T };

type UserId = Brand<string, "UserId">;
type ProductId = Brand<string, "ProductId">;

function getUser(id: UserId) { /* ... */ }

const pId = "123" as ProductId;
// getUser(pId); // Compile Error! ProductId is not assignable to UserId
```

---

## 4. Advanced Type Operations & Concepts

### A. Conditional Types
Conditional types allow you to declare type transitions using ternary-like logic:
`T extends U ? X : Y`

Used heavily alongside the **`infer`** keyword, which introduces a placeholder type variable to be resolved dynamically by the compiler:

```typescript
type GetReturnType<T> = T extends (...args: any[]) => infer R ? R : never;

type TargetFunction = () => number;
type Resolved = GetReturnType<TargetFunction>; // number
```

### B. Mapped Types
Mapped types iterate over keys to dynamically build new object schemas:

```typescript
type ReadonlyMapped<T> = {
  readonly [P in keyof T]: T[P];
};
```

### C. Template Literal Types
Allows string manipulation and pattern matching within the type system:

```typescript
type Direction = "top" | "bottom";
type Padding = `padding-${Direction}`; // "padding-top" | "padding-bottom"
```

---

## 5. Covariance, Contravariance, and Bivariance

These terms define how type compatibility of complex structures (like arrays, functions, or objects) scales relative to their subtype hierarchies (e.g., `Dog extends Animal`).

### A. Covariance (Preserves Direction)
A structure is **covariant** if it scales in the same direction as the subtype.
- **Rule:** If `Dog extends Animal`, then `Dog[]` can be assigned to `Animal[]`.
- **Implementation:** Read-only structures, getters, and function return values are covariant.

### B. Contravariance (Reverses Direction)
A structure is **contravariant** if it scales in the opposite direction of the subtype.
- **Rule:** If `Dog extends Animal`, then a function expecting an `Animal` can be assigned to a function expecting a `Dog` (because the function parameter only utilizes `Animal` fields, which `Dog` is guaranteed to have).
- **Implementation:** Function parameters (under `strictFunctionTypes: true`) are contravariant.

### C. Bivariance (Both Directions)
A structure is **bivariant** if compatibility works in both directions.
- **Implementation:** Method declarations inside classes/interfaces are bivariant by default to simplify standard JavaScript patterns (like array callbacks).

---

## 6. Inner Implementations of Common Utility Types

TypeScript provides global utility types. Their internal configurations are written using advanced operators:

### `Partial<T>`
Makes all properties of an object optional.
```typescript
type Partial<T> = {
  [P in keyof T]?: T[P];
};
```

### `Required<T>`
Removes the optional marker (`?`) from all fields.
```typescript
type Required<T> = {
  [P in keyof T]-?: T[P];
};
```

### `Omit<T, K>`
Constructs a type by picking all properties from `T` and then removing `K`.
```typescript
type Omit<T, K extends keyof any> = Pick<T, Exclude<keyof T, K>>;
```

---

## 7. High-Impact Interview Questions & Answers

### Q1: Write a custom utility type `DeepReadonly<T>` that recursively marks all properties (including nested objects and arrays) as readonly.
* **Answer:**
  ```typescript
  type DeepReadonly<T> = {
    readonly [P in keyof T]: T[P] extends Function
      ? T[P]
      : T[P] extends object
      ? DeepReadonly<T[P]>
      : T[P];
  };
  ```

### Q2: What is the purpose of the `any` type versus the `unknown` type, and when should you choose one over the other?
* **Answer:**
  - **`any`:** Turns off the type checker entirely. You can read any property, invoke any method, or assign `any` to any other type. It is completely type-unsafe.
  - **`unknown`:** The type-safe counterpart of `any`. It represents any value, but the TypeScript compiler restricts you from doing *anything* with an `unknown` variable until you perform explicit type narrowing (e.g., using `typeof`, `instanceof`, or user-defined type guards):
    ```typescript
    function processValue(val: unknown) {
      // val.toUpperCase(); // Error! Property 'toUpperCase' does not exist on type 'unknown'.
      if (typeof val === "string") {
        val.toUpperCase(); // Allowed! Type narrowed to string.
      }
    }
    ```

### Q3: How do you implement a User-Defined Type Guard, and how does it differ from a boolean return type?
* **Answer:** A user-defined type guard uses a special return type predicate: **`parameterName is Type`**.
- **Difference:** Returning a simple `boolean` tells the compiler the function succeeded, but does *not* refine the type of the variable inside the parent block. Returning `is Type` instructs the compiler to narrow the variable type directly inside the conditional branches.
- **Example:**
  ```typescript
  interface Bird { fly(): void; }
  interface Fish { swim(): void; }

  function isFish(animal: Bird | Fish): animal is Fish {
    return (animal as Fish).swim !== undefined;
  }

  function move(animal: Bird | Fish) {
    if (isFish(animal)) {
      animal.swim(); // Narrowed to Fish
    } else {
      animal.fly();  // Narrowed to Bird
    }
  }
  ```

### Q4: Why does the compiler allow assigning a function with fewer parameters to a callback expecting more parameters? Give an example.
* **Answer:** This is a design feature in TypeScript to support standard, idiomatic JavaScript array and callback patterns. If a callback expects three parameters (e.g., `value`, `index`, `array` in `Array.prototype.forEach`), you are allowed to pass a function that ignores the index and array arguments.
- **Example:**
  ```typescript
  const arr = [1, 2, 3];
  // forEach expects: (val: number, idx: number, arr: number[]) => void
  // We pass: (val) => void (which ignores idx and arr)
  arr.forEach((val) => console.log(val)); // Allowed and safe.
  ```

### Q5: What is the difference between `const` assertions (`as const`) and standard variable declarations?
* **Answer:** `as const` instructs the TypeScript compiler to infer the most specific, literal, and deeply immutable type possible for an object or array:
  1. Primitive types are narrowed to their literal values (e.g., `"GET"` instead of `string`).
  2. Object properties become `readonly`.
  3. Arrays become `readonly` tuples.
- **Example:**
  ```typescript
  const config = {
    url: "https://api.com",
    method: "GET"
  } as const;

  // config.method is typed strictly as the literal "GET" (not string).
  // config.url = "test"; // Error! Cannot assign to 'url' because it is a read-only property.
  ```

### Q6: How do you solve type errors when dynamically index-accessing an object using a string variable?
* **Answer:**
- **The Problem:**
  ```typescript
  const prices = { gold: 100, silver: 50 };
  function getPrice(key: string) {
    // return prices[key]; // Error! Element implicitly has an 'any' type because expression of type 'string' can't be used to index type.
  }
  ```
- **The Solution:** Use a generic constraint or narrow the parameter to keys of the target object using `keyof typeof`:
  ```typescript
  function getPriceSafe(key: keyof typeof prices) {
    return prices[key]; // Safe! key is restricted to "gold" | "silver"
  }
  ```