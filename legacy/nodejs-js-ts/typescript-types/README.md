# TypeScript Type System

Comprehensive interview study guide covering TypeScript types vs. interfaces, generics, utility types, structural typing, and advanced type operations.

---

## 1. Type vs. Interface

While both can declare structured shapes, they have fundamental behavioral differences:

| Feature | Interface | Type Alias |
| :--- | :--- | :--- |
| **Declaration Merging** | **Supported.** Multiple definitions with the same name automatically merge. | Not supported. Triggers duplicate identifier error. |
| **Extension Syntax** | Uses `extends` (e.g., `interface B extends A`). | Uses intersection `&` (e.g., `type B = A & C`). |
| **Primitive Aliasing** | Cannot alias primitive types or unions (e.g., `type ID = string | number`). | Supported. Flexible definition styles. |
| **Objects & Classes** | Ideal for declaring API records, OOP class blueprints. | Ideal for complex unions, tuples, or functional parameters. |

---

## 2. Advanced Type Operations

TypeScript supports complex operations to map, transform, and constraint types:

### 1. Union (`|`) vs. Intersection (`&`)
* **Union (`A | B`):** The value must conform to either shape `A` **or** shape `B`.
* **Intersection (`A & B`):** The value must contain all attributes of shape `A` **and** shape `B`.

### 2. Generics
Generics allow you to write reusable components that operate over a variety of types rather than a single static type, preserving type safety:
```typescript
interface ResponseEnvelope<T> {
  status: "success" | "error";
  data: T; // Dynamic type parameter
}
```

---

## 3. Standard Utility Types

TypeScript provides built-in utilities to transform types:

* **`Partial<T>`:** Makes all properties in `T` optional.
* **`Required<T>`:** Makes all properties in `T` mandatory.
* **`Readonly<T>`:** Makes all properties in `T` read-only (constant).
* **`Pick<T, K>`:** Extracts a subset of keys `K` from type `T`.
* **`Omit<T, K>`:** Removes a subset of keys `K` from type `T`.
* **`Record<K, T>`:** Constructs an object type with keys `K` and value type `T`.

---

## 4. Structural Typing vs. Nominal Typing

* **Nominal Typing (e.g., Java, C++):** Equivalence is decided strictly by explicit class names or declarations.
* **Structural Typing (TypeScript):** Equivalence is decided strictly by the **shape and structure** of the data.
  * If object $X$ has all properties required by type $Y$, $X$ is considered compatible with $Y$, regardless of inheritance declarations. This is commonly known as **Duck Typing**.

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: When should you use `interface` vs. `type` in TypeScript?
* **Answer:** Use **`interface`** by default when defining object structures, components props, or class contracts, as interfaces support **declaration merging** (essential for extensible library types) and produce faster compile-time type-checking. Use **`type`** when you need primitive aliases, tuple types, mapped types, or complex Union/Intersection structures which interfaces are physically incapable of representing.

### Q2: What is the difference between `unknown` and `any` in TypeScript?
* **Answer:** **`any`** completely disables compile-time type-checking; it allows you to read any property, execute any method, or assign the value to any variable, bypassing TypeScript's safety guarantees completely. **`unknown`** is the type-safe counterpart. It represents any value, but the compiler will throw errors if you attempt to interact with it until you explicitly perform a type-guard assertion (such as `typeof x === "string"` or `instanceof User`) to verify its shape first.

### Q3: What is the difference between `interface extends` and type intersection (`&`) when dealing with property collisions?
* **Answer:** If you use `interface extends` and declare a property with the same name but a conflicting type, the TypeScript compiler will throw a **clear compile-time error** explaining the type collision. If you use type intersection (`&`), TypeScript allows the compile step but merges the colliding properties into an impossible **`never`** type, resulting in silent type issues during variable assignments.
