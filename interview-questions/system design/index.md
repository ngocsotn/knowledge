# System Design Interview Questions & Answers

Comprehensive study guide of real interview questions encountered while applying for an Angular frontend position. These questions are Angular-related but heavily centered on system design principles, software architecture, and cross-cutting engineering concerns.

## Table of Contents

- [Q1: How Would You Layer Your Apps?](#q1-how-would-you-layer-your-apps)
- [Q2: What Are Responsibility Ownership and How Do You Make It Clear?](#q2-what-are-responsibility-ownership-and-how-do-you-make-it-clear)
- [Q3: What Paradigm Is Used to Solve Which Set of Problems?](#q3-what-paradigm-is-used-to-solve-which-set-of-problems)
- [Q4: Idempotency and Their Role in Systems](#q4-idempotency-and-their-role-in-systems)
- [Q5: What Are Monads and the Function Signature Ambiguity Problem?](#q5-what-are-monads-and-the-function-signature-ambiguity-problem)
- [Q6: How Do You Handle Schema Mismatches Between Boundaries?](#q6-how-do-you-handle-schema-mismatches-between-boundaries)
- [Q7: How Do You Design Your System to Be Modular?](#q7-how-do-you-design-your-system-to-be-modular)

---

## Q1: How Would You Layer Your Apps?

### The Problem Layering Solves

Without explicit layering, business logic leaks into UI components, data-access details leak into domain rules, and every change cascades unpredictably across the codebase. Layering imposes a **directed dependency rule**: each layer may only depend on the layer directly below it, never upward or sideways.

### The Standard Layered Architecture (N-Tier)

```
┌─────────────────────────────────────────────────────────┐
│                  PRESENTATION LAYER                     │
│  Components, Templates, Directives, Pipes               │
│  Responsibility: Render UI, capture user input           │
│  Depends on: Application Layer                           │
├─────────────────────────────────────────────────────────┤
│                  APPLICATION LAYER                       │
│  Facades, Use Cases, Orchestrators                       │
│  Responsibility: Coordinate workflows, enforce rules     │
│  Depends on: Domain Layer                                │
├─────────────────────────────────────────────────────────┤
│                    DOMAIN LAYER                          │
│  Models, Value Objects, Business Rules, Validators       │
│  Responsibility: Pure business logic, zero dependencies  │
│  Depends on: Nothing (the innermost core)                │
├─────────────────────────────────────────────────────────┤
│                 INFRASTRUCTURE LAYER                     │
│  HTTP Services, Storage, Third-Party SDKs, Adapters      │
│  Responsibility: External communication, persistence     │
│  Depends on: Domain interfaces (inverted)                │
└─────────────────────────────────────────────────────────┘
```

### Each Layer Solves Specific Problems

**Presentation Layer**
- **Solves:** UI rendering, user interaction capture, visual state management.
- **Contains:** Angular Components, Directives, Pipes, Template logic.
- **Rule:** No HTTP calls, no business logic, no direct state mutations. Components are thin — they delegate to the Application Layer.

**Application Layer (Facades / Use Cases)**
- **Solves:** Workflow orchestration, cross-concern coordination, mapping between domain models and UI view models.
- **Contains:** Facade services, use-case classes, command/query handlers.
- **Rule:** Orchestrates domain objects and infrastructure services but contains no domain rules itself.

```typescript
@Injectable({ providedIn: 'root' })
export class OrderFacade {
  constructor(
    private orderRepo: OrderRepository,
    private paymentService: PaymentService,
    private notificationService: NotificationService
  ) {}

  async placeOrder(cart: Cart): Promise<OrderConfirmation> {
    const order = Order.createFrom(cart);         // Domain logic
    const payment = await this.paymentService.charge(order.total);  // Infrastructure
    const saved = await this.orderRepo.save(order);                 // Infrastructure
    this.notificationService.notify(saved);                         // Infrastructure
    return OrderConfirmation.from(saved, payment);                  // Application mapping
  }
}
```

**Domain Layer**
- **Solves:** Business rule enforcement, validation, domain invariants.
- **Contains:** Pure TypeScript classes and functions with zero framework imports.
- **Rule:** No Angular imports, no HTTP, no RxJS. This layer is portable across frameworks and testable with zero infrastructure.

```typescript
// Pure domain model — zero dependencies
export class Order {
  static createFrom(cart: Cart): Order {
    if (cart.items.length === 0) throw new DomainError('Cannot create order from empty cart');
    return new Order(crypto.randomUUID(), cart.items, OrderStatus.Pending);
  }

  get total(): number {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }
}
```

**Infrastructure Layer**
- **Solves:** External system communication (APIs, databases, browser storage, third-party SDKs).
- **Contains:** `HttpClient` services, WebSocket handlers, LocalStorage adapters, Firebase wrappers.
- **Rule:** Implements interfaces defined by the Domain Layer (Dependency Inversion). The domain defines *what* it needs; infrastructure provides *how*.

```typescript
// Domain defines the contract
export abstract class OrderRepository {
  abstract save(order: Order): Promise<Order>;
  abstract findById(id: string): Promise<Order | null>;
}

// Infrastructure implements it
@Injectable({ providedIn: 'root' })
export class HttpOrderRepository extends OrderRepository {
  constructor(private http: HttpClient) { super(); }
  save(order: Order): Promise<Order> {
    return firstValueFrom(this.http.post<Order>('/api/orders', order));
  }
  findById(id: string): Promise<Order | null> {
    return firstValueFrom(this.http.get<Order>(`/api/orders/${id}`));
  }
}
```

### Advanced: Feature-Sliced Architecture

In large Angular applications, layers are further partitioned by **feature domain** (vertical slicing):

```
src/
├── features/
│   ├── orders/
│   │   ├── presentation/    (components, pages)
│   │   ├── application/     (facades, use-cases)
│   │   ├── domain/          (models, rules)
│   │   └── infrastructure/  (HTTP, adapters)
│   ├── users/
│   │   ├── presentation/
│   │   ├── application/
│   │   ├── domain/
│   │   └── infrastructure/
├── shared/                  (cross-cutting utilities)
└── core/                    (singleton services, guards, interceptors)
```

This combines the discipline of horizontal layers with the autonomy of vertical feature slices, enabling teams to work independently on isolated feature domains.

---

## Q2: What Are Responsibility Ownership and How Do You Make It Clear?

### Definition

Responsibility Ownership defines **which module, component, service, or layer is accountable for a specific concern** — and, critically, which ones are NOT. When ownership is ambiguous, developers add duplicate logic, create conflicting behaviors, or leave gaps where no one handles the concern at all.

### The Three Dimensions of Ownership

```
┌─────────────────────────────────────────────────────────┐
│              RESPONSIBILITY OWNERSHIP                   │
├───────────────┬───────────────────┬─────────────────────┤
│   DATA OWNER  │  BEHAVIOR OWNER   │  LIFECYCLE OWNER    │
│               │                   │                     │
│ Who is the    │ Who validates,    │ Who creates,        │
│ source of     │ transforms, and   │ initializes, and    │
│ truth?        │ enforces rules?   │ destroys it?        │
└───────────────┴───────────────────┴─────────────────────┘
```

**Data Ownership:** Exactly one entity is the source of truth for any piece of state. If `UserService` owns the current user, no component should independently fetch or cache user data. Every consumer reads from `UserService`.

**Behavior Ownership:** Business rules, validations, and transformations belong to the Domain Layer. If "an order must have at least one item," that rule is enforced inside `Order.createFrom()`, not in a component, not in an API interceptor, not in a template `*ngIf`.

**Lifecycle Ownership:** The creator of a resource is responsible for destroying it. If a component creates a subscription, that component must unsubscribe. If a service opens a WebSocket, that service must close it.

### Techniques to Make Ownership Clear

**1. Single Responsibility Principle (SRP)**
Each class, service, or function has exactly one reason to change. If a service handles both authentication AND notification, it owns two responsibilities — split it.

**2. Directory Structure as Ownership Declaration**
The file system itself should make ownership visible:

```
features/billing/
├── domain/
│   ├── invoice.model.ts          ← Owns Invoice business rules
│   └── invoice.validator.ts      ← Owns validation logic
├── application/
│   └── billing.facade.ts         ← Owns workflow orchestration
├── infrastructure/
│   └── billing-api.service.ts    ← Owns HTTP communication
└── presentation/
    └── invoice-list.component.ts ← Owns rendering
```

A developer looking for "where is billing validation handled?" navigates directly to `billing/domain/`.

**3. Access Modifiers and Encapsulation**
Use TypeScript `private` and `protected` to prevent external code from reaching into a module's internal state. Export only the public API.

```typescript
@Injectable({ providedIn: 'root' })
export class CartService {
  // Private: only CartService can mutate this
  private readonly _items = signal<CartItem[]>([]);

  // Public: consumers get a read-only projection
  readonly items = this._items.asReadonly();
  readonly total = computed(() => this._items().reduce((s, i) => s + i.price, 0));

  // Public: mutation is controlled through explicit methods
  addItem(item: CartItem): void { this._items.update(items => [...items, item]); }
  removeItem(id: string): void { this._items.update(items => items.filter(i => i.id !== id)); }
}
```

**4. Interface Contracts at Boundaries**
When two modules interact, define an explicit interface contract. The provider implements the interface; the consumer depends only on the interface. Ownership of the *contract* belongs to the consumer (Dependency Inversion).

**5. Documentation and ADRs (Architecture Decision Records)**
For cross-cutting concerns that span multiple modules, write an ADR that explicitly names the owner. Example: *"Authentication token refresh is owned by `CoreModule/AuthService`. No other module may directly call the refresh endpoint."*

### Key Takeaway
Ownership is clear when any developer on the team can answer three questions without asking anyone: *"Who owns this data? Who owns this behavior? Who manages this lifecycle?"* If the answer requires a meeting, the architecture has failed.

---

## Q3: What Paradigm Is Used to Solve Which Set of Problems?

### The Four Major Paradigms

```
┌─────────────────────────────────────────────────────────────────┐
│                    PARADIGM COMPARISON                          │
├────────────────────┬────────────────────────────────────────────┤
│  PARADIGM          │  CORE PHILOSOPHY                           │
├────────────────────┼────────────────────────────────────────────┤
│  Imperative        │  Tell the machine HOW to do it, step by   │
│                    │  step. You control flow explicitly.        │
├────────────────────┼────────────────────────────────────────────┤
│  Declarative       │  Tell the machine WHAT you want. It       │
│                    │  figures out the how.                      │
├────────────────────┼────────────────────────────────────────────┤
│  Object-Oriented   │  Model the world as interacting objects   │
│  (OOP)             │  with encapsulated state and behavior.    │
├────────────────────┼────────────────────────────────────────────┤
│  Functional        │  Model computation as pure mathematical   │
│  (FP)              │  functions with no side effects.          │
└────────────────────┴────────────────────────────────────────────┘
```

### Imperative Programming
- **Solves:** Low-level control, performance-critical loops, hardware interaction, sequential algorithms.
- **Characteristics:** Mutable variables, explicit loops (`for`, `while`), step-by-step mutation.
- **Example use case:** Writing a custom sorting algorithm, managing a manual DOM manipulation sequence, controlling WebSocket byte streams.
- **Weakness:** Hard to reason about at scale because state changes are spread across many lines and branches.

### Declarative Programming
- **Solves:** UI rendering, data queries, configuration, infrastructure-as-code.
- **Characteristics:** You describe the desired end state; the framework resolves how to achieve it.
- **Example use case:** Angular templates (declare what the UI looks like for a given state), SQL queries (declare what data you want, not how to traverse tables), CSS (declare visual rules).
- **Weakness:** Limited control over execution order and performance tuning.

```html
<!-- Declarative: Angular template describes WHAT to render -->
@for (user of users; track user.id) {
  <app-user-card [user]="user" />
}
```

### Object-Oriented Programming (OOP)
- **Solves:** Modeling complex, long-lived business domains with stateful entities that interact over time.
- **Characteristics:** Encapsulation, inheritance, polymorphism, abstraction.
- **Example use case:** Enterprise domain models (User, Order, Invoice), Angular services with internal state, game entities with behavior.
- **Weakness:** Inheritance hierarchies can become rigid. Shared mutable state in objects leads to coupling.

```typescript
// OOP: stateful entity with encapsulated behavior
export class ShoppingCart {
  private items: CartItem[] = [];

  addItem(item: CartItem): void {
    const existing = this.items.find(i => i.sku === item.sku);
    if (existing) existing.quantity += item.quantity;
    else this.items.push({ ...item });
  }

  get total(): number {
    return this.items.reduce((sum, i) => sum + i.price * i.quantity, 0);
  }
}
```

### Functional Programming (FP)
- **Solves:** Data transformation pipelines, concurrent/parallel processing, predictable state management, avoiding side-effect bugs.
- **Characteristics:** Pure functions, immutability, higher-order functions, composition, no side effects.
- **Example use case:** RxJS Observable pipelines, Redux/NgRx reducers, data transformation chains, utility functions.
- **Weakness:** Steep learning curve for teams unfamiliar with it. IO and side effects require monadic wrappers or explicit effect isolation.

```typescript
// FP: pure data transformation pipeline
const activeAdminEmails = users
  .filter(u => u.active)
  .filter(u => u.role === 'admin')
  .map(u => u.email);

// NgRx reducer: pure function, always returns new state
export const cartReducer = createReducer(
  initialState,
  on(addItem, (state, { item }) => ({
    ...state,
    items: [...state.items, item]
  }))
);
```

### When to Combine Paradigms (Multi-Paradigm)

Modern applications mix paradigms at different layers:

| Layer | Primary Paradigm | Rationale |
|---|---|---|
| Templates | **Declarative** | Describe UI state, let Angular handle rendering |
| Data pipelines (RxJS) | **Functional** | Pure transformations, no side effects, composable |
| Services with state | **OOP** | Encapsulate mutable state behind controlled APIs |
| Reducers / Selectors | **Functional** | Predictable state transitions |
| DOM manipulation scripts | **Imperative** | Fine-grained control where needed |

### Key Takeaway
No single paradigm is universally superior. The discipline is choosing the right paradigm for the right problem boundary, and keeping boundaries clean so paradigms don't leak across layers.

---

## Q4: Idempotency and Their Role in Systems

### Definition

An operation is **idempotent** if applying it once produces the same result as applying it multiple times. The system's state does not change beyond the initial application, regardless of how many times the operation is repeated.

```
f(x) = f(f(x)) = f(f(f(x)))
```

### Why Idempotency Matters

In distributed systems — and even in frontend applications — operations can be **retried** due to:
- Network timeouts (client doesn't know if the server received the request)
- User double-clicks
- Browser back/forward navigation replaying form submissions
- Message queue redelivery (at-least-once delivery guarantees)
- HTTP interceptor auto-retry logic

Without idempotency, retries cause **duplicate side effects**: double charges, duplicate records, conflicting state mutations.

### Idempotent vs. Non-Idempotent Operations

| HTTP Method | Idempotent? | Explanation |
|---|---|---|
| `GET` | ✅ Yes | Reading data never changes state |
| `PUT` | ✅ Yes | Replaces the entire resource; repeating produces same state |
| `DELETE` | ✅ Yes | Deleting an already-deleted resource is a no-op |
| `POST` | ❌ No | Creates a new resource each time; repeating creates duplicates |
| `PATCH` | ⚠️ Depends | If the patch sets absolute values, yes. If it increments, no. |

### Implementing Idempotency

**1. Idempotency Keys (Client-Generated)**

The client generates a unique key (UUID) for each logical operation and sends it with the request. The server checks if this key has already been processed.

```typescript
// Frontend: generate idempotency key before submitting
const idempotencyKey = crypto.randomUUID();

this.http.post('/api/payments', payload, {
  headers: { 'Idempotency-Key': idempotencyKey }
}).subscribe();
```

```
Server Logic:
1. Receive request with Idempotency-Key header
2. Check if key exists in Redis/database
   → If YES: return the stored response (no re-execution)
   → If NO: execute the operation, store the result keyed by the idempotency key, return the response
```

**2. Upsert (PUT Semantics)**

Instead of "create a new record," use "set this record to this exact state." Repeating a `PUT /users/123 { name: "Alice" }` always results in the same state.

**3. Conditional Writes (Optimistic Concurrency)**

Use version numbers or ETags. The write only succeeds if the current version matches the expected version, preventing duplicate effects from stale retries.

**4. Frontend Debouncing & Disable-on-Submit**

Prevent duplicate user-initiated requests at the UI level:

```typescript
// Disable button after first click
@Component({
  template: `<button [disabled]="submitting" (click)="submit()">Pay</button>`
})
export class PaymentComponent {
  submitting = false;
  submit() {
    this.submitting = true;
    this.paymentService.charge(this.order).pipe(
      finalize(() => this.submitting = false)
    ).subscribe();
  }
}
```

### Idempotency in Frontend Context

Even pure frontend operations benefit from idempotent thinking:
- **NgRx reducers** are idempotent by design — dispatching the same action with the same payload always produces the same state.
- **Signal `.set()`** is idempotent — setting the same value multiple times doesn't create side effects.
- **Router navigation** to the same route should be idempotent — no duplicate subscriptions or side effects.

### Key Takeaway
Design every state-changing operation to be safely retriable. At the API layer, use idempotency keys. At the UI layer, debounce and disable. At the state layer, use pure reducers and set-semantics. Idempotency is not an optimization — it is a **correctness guarantee** in a world where retries are inevitable.

---

## Q5: What Are Monads and the Function Signature Ambiguity Problem?

### The Problem: Function Signature Ambiguity

Consider this function signature:

```typescript
function getUser(id: number): User { ... }
```

This signature **lies**. It promises to return a `User`, but in reality it might:
- Return `null` or `undefined` if the user doesn't exist
- Throw an exception if the database is down
- Return stale data from a cache
- Block indefinitely on a slow network call

The caller has no way to know from the type signature alone what can go wrong. This is the **function signature ambiguity problem** — the gap between what the type promises and what actually happens at runtime.

### What Is a Monad?

A Monad is a **design pattern from functional programming** that wraps a value inside a computational context and provides two operations:

1. **`of` (or `return`):** Wraps a plain value into the monadic context.
2. **`flatMap` (or `bind` / `chain`):** Takes the wrapped value, applies a function that returns another monad, and flattens the result to avoid nesting.

The key insight: **Monads make the computational context explicit in the type signature.** Instead of lying about what a function returns, the return type honestly declares: "this might fail," "this might be empty," or "this will be asynchronous."

### Monad Types That Solve Signature Ambiguity

**1. `Option<T>` / `Maybe<T>` — Explicit Absence**

Instead of returning `User | null` (which callers forget to check), return an `Option<User>`:

```typescript
function getUser(id: number): Option<User> {
  const user = database.find(id);
  return user ? Some(user) : None();
}

// Caller MUST handle the absence — the type system enforces it
getUser(42)
  .map(user => user.name)       // Only executes if Some
  .getOrElse('Unknown User');   // Provides fallback for None
```

The function signature now honestly says: *"I might not return a User."*

**2. `Result<T, E>` / `Either<L, R>` — Explicit Errors**

Instead of throwing exceptions (which are invisible in the type signature), return a `Result`:

```typescript
function parseConfig(raw: string): Result<Config, ParseError> {
  try {
    return Ok(JSON.parse(raw));
  } catch (e) {
    return Err(new ParseError('Invalid JSON'));
  }
}

// Caller MUST handle both success and failure paths
parseConfig(input)
  .map(config => config.apiUrl)
  .mapErr(error => logError(error))
  .unwrapOr(defaultConfig);
```

The function signature honestly says: *"I might fail with a ParseError."*

**3. `Observable<T>` — Explicit Asynchrony and Multiplicity**

RxJS Observables in Angular are monads. When a function returns `Observable<User>`:

```typescript
function getUser(id: number): Observable<User> { ... }
```

The signature honestly says: *"This is asynchronous, may emit zero, one, or many values over time, and may error."* The caller handles this explicitly via `subscribe`, `pipe`, error callbacks, and completion handlers.

**4. `Promise<T>` — Explicit Single Async Result**

```typescript
async function getUser(id: number): Promise<User> { ... }
```

Honestly says: *"This is asynchronous and will resolve once with a User or reject with an error."*

### How Monads Chain (flatMap)

The power of monads is in composition. Without monads, nested nullability creates the pyramid of doom:

```typescript
// ❌ Without monads: nested null checks
const user = getUser(id);
if (user) {
  const address = user.address;
  if (address) {
    const city = address.city;
    if (city) { ... }
  }
}

// ✅ With monadic chaining: flat, readable, and safe
getUser(id)
  .flatMap(user => user.address)
  .flatMap(address => address.city)
  .map(city => city.toUpperCase())
  .getOrElse('UNKNOWN');
```

Each `.flatMap()` short-circuits on `None` — no nested `if` statements, no forgotten null checks.

### Practical Application in Angular

While Angular/TypeScript doesn't have a built-in `Option` or `Result` type, the patterns are applied through:
- **RxJS Observables:** The primary monad in Angular applications.
- **Optional chaining (`?.`):** A lightweight syntax for `Maybe`-like access.
- **TypeScript strict null checks:** Forces handling of `T | undefined` at the type level.
- **Libraries:** `fp-ts`, `neverthrow`, `oxide.ts` bring `Option`, `Result`, and `Either` types to TypeScript.

### Key Takeaway
Monads solve function signature ambiguity by encoding the **context** (absence, failure, asynchrony, multiplicity) directly into the return type. The type signature becomes honest, and the compiler forces the caller to handle every possible outcome — eliminating entire categories of runtime bugs.

---

## Q6: How Do You Handle Schema Mismatches Between Boundaries?

### The Problem

In any non-trivial application, data crosses boundaries:
- **Backend API → Frontend:** The API returns `snake_case` JSON; the frontend uses `camelCase` TypeScript interfaces.
- **API v1 → API v2:** A field is renamed, a type changes from `string` to `object`, or a required field becomes optional.
- **Microservice A → Microservice B:** Different teams model the same concept differently.
- **Third-party SDK → Your Domain:** External libraries use their own data shapes.

When these schemas diverge — and they always do — consuming raw external data directly in your domain or presentation layer creates **tight coupling** to external contracts you don't control.

### The Solution: Anti-Corruption Layer (ACL) with Mappers

Intercept external data at the boundary and transform it into your internal domain model using dedicated **mapper functions**. Your internal code never touches the raw external schema.

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────────┐
│   External   │────►│  Anti-Corruption │────►│  Internal Domain │
│   Schema     │     │  Layer (Mapper)   │     │  Model           │
│  (API DTO)   │     │                  │     │  (Clean types)   │
└──────────────┘     └──────────────────┘     └──────────────────┘
```

### Implementation Pattern

```typescript
// 1. Define the EXTERNAL schema (what the API actually returns)
interface UserApiResponse {
  user_id: number;
  full_name: string;
  email_address: string;
  is_active: boolean;
  created_at: string;   // ISO string from backend
  roles: { role_name: string; role_level: number }[];
}

// 2. Define the INTERNAL domain model (what your app uses)
interface User {
  id: number;
  name: string;
  email: string;
  active: boolean;
  createdAt: Date;
  roles: UserRole[];
}

// 3. Define the MAPPER at the boundary (Anti-Corruption Layer)
function mapApiResponseToUser(response: UserApiResponse): User {
  return {
    id: response.user_id,
    name: response.full_name,
    email: response.email_address,
    active: response.is_active,
    createdAt: new Date(response.created_at),
    roles: response.roles.map(r => ({
      name: r.role_name as UserRoleName,
      level: r.role_level,
    })),
  };
}

// 4. Use the mapper in the infrastructure service
@Injectable({ providedIn: 'root' })
export class UserApiService {
  constructor(private http: HttpClient) {}

  getUser(id: number): Observable<User> {
    return this.http.get<UserApiResponse>(`/api/users/${id}`).pipe(
      map(response => mapApiResponseToUser(response))  // ACL boundary
    );
  }
}
```

### Advanced Strategies

**1. Runtime Validation with Zod / io-ts**

Don't just map — **validate** at the boundary. If the backend silently changes a field type, catch it immediately instead of letting a `undefined` propagate through your app:

```typescript
import { z } from 'zod';

const UserApiSchema = z.object({
  user_id: z.number(),
  full_name: z.string(),
  email_address: z.string().email(),
  is_active: z.boolean(),
  created_at: z.string().datetime(),
});

// Validate + transform in one step
function parseUser(raw: unknown): User {
  const validated = UserApiSchema.parse(raw);  // Throws ZodError on mismatch
  return mapApiResponseToUser(validated);
}
```

**2. Versioned Mappers for API Evolution**

When the API evolves, create version-specific mappers. A factory function selects the correct mapper based on API version:

```typescript
function createUserMapper(apiVersion: string): (raw: unknown) => User {
  switch (apiVersion) {
    case 'v1': return mapV1ToUser;
    case 'v2': return mapV2ToUser;
    default:   throw new Error(`Unsupported API version: ${apiVersion}`);
  }
}
```

**3. Adapter Pattern for Third-Party SDKs**

Wrap third-party library types inside your own interface so your application code never directly depends on the SDK's schema:

```typescript
// Your internal interface
interface MapMarker {
  id: string;
  lat: number;
  lng: number;
  label: string;
}

// Adapter: translates your domain model to the SDK's format
class GoogleMapsAdapter {
  toGoogleMarker(marker: MapMarker): google.maps.MarkerOptions {
    return { position: { lat: marker.lat, lng: marker.lng }, title: marker.label };
  }
}
```

### Key Takeaway
Never let external schemas leak past the boundary. Define clear internal models, write dedicated mappers at the boundary, validate with runtime schemas (Zod), and treat every external integration as an untrusted input source. When the external contract changes, only the mapper changes — the rest of the application remains untouched.

---

## Q7: How Do You Design Your System to Be Modular?

### The Core Tension

Every engineering team claims their system is "modular," yet most codebases degrade into tight coupling within months. The reason: **modularity is not a default property — it is an enforced constraint.** Without active enforcement, developers take shortcuts: importing internal functions from sibling modules, sharing mutable state through global services, and creating hidden dependency chains.

### What Modularity Actually Means

A system is modular when:
1. **Each module can be understood in isolation.** Reading `BillingModule` doesn't require understanding `UserModule`.
2. **Each module can be modified independently.** Changing billing logic doesn't break user authentication.
3. **Each module can be tested independently.** `BillingModule` tests pass without starting the database or the user service.
4. **Each module can be replaced entirely.** Swapping Stripe for PayPal affects only `BillingModule/infrastructure/`.

### The Principles That Enforce Modularity

**1. Explicit Public APIs (Barrel Exports)**

Each module exposes only its public interface through a single entry point. Internal files are never imported directly by external modules.

```
features/billing/
├── index.ts                 ← PUBLIC API (barrel file)
├── billing.facade.ts        ← Exported via index.ts
├── billing.model.ts         ← Exported via index.ts
├── internal/
│   ├── tax-calculator.ts    ← INTERNAL: never imported externally
│   └── stripe-adapter.ts   ← INTERNAL: never imported externally
```

```typescript
// features/billing/index.ts — the only import path for consumers
export { BillingFacade } from './billing.facade';
export { Invoice, BillingPeriod } from './billing.model';
// tax-calculator and stripe-adapter are NOT exported
```

**2. Dependency Inversion at Module Boundaries**

Modules depend on abstractions (interfaces), not on each other's concrete implementations. If `BillingModule` needs user data, it depends on a `UserProvider` interface — not on `UserService` from `UserModule`.

```typescript
// Defined in billing module's domain (the consumer defines the contract)
export abstract class UserProvider {
  abstract getCurrentUser(): Observable<User>;
}

// Provided by the app's root configuration
{ provide: UserProvider, useExisting: UserService }
```

**3. Enforce Boundaries with Lint Rules**

Use ESLint rules (e.g., `eslint-plugin-boundaries`, `@nx/enforce-module-boundaries`) to make illegal cross-module imports a build error:

```json
// .eslintrc: Prevent billing from directly importing user internals
{
  "rules": {
    "@nx/enforce-module-boundaries": ["error", {
      "depConstraints": [
        { "sourceTag": "scope:billing", "onlyDependOnLibsWithTags": ["scope:shared", "scope:billing"] }
      ]
    }]
  }
}
```

**4. Inversion of Control (IoC) via Dependency Injection**

Angular's DI system is a built-in modularization tool. Services are provided at the module or component level, making their scope explicit:

```typescript
// Service scoped to the billing feature — not visible globally
@Component({
  providers: [BillingCalculationService]  // Scoped to this component tree
})
export class BillingPageComponent { }
```

**5. Communication Through Contracts, Not Implementation**

Modules communicate via:
- **Event buses / Observables:** `BillingModule` emits a `PaymentCompleted` event; `NotificationModule` listens. Neither imports the other.
- **Shared interfaces in a common library:** DTOs and event types live in a shared `contracts` package.

### The Modular Architecture Checklist

```
┌──────────────────────────────────────────────────────────┐
│              MODULARITY ENFORCEMENT CHECKLIST             │
├──────────────────────────────────────────────────────────┤
│  ✅  Each module has a single barrel export (index.ts)    │
│  ✅  No deep imports across module boundaries             │
│  ✅  Dependencies flow inward (Dependency Inversion)      │
│  ✅  Cross-module communication via events or interfaces  │
│  ✅  Lint rules enforce boundary violations at build time │
│  ✅  Each module has its own test suite that runs solo     │
│  ✅  Shared code lives in explicit `shared/` libraries    │
│  ✅  No circular dependencies between modules             │
└──────────────────────────────────────────────────────────┘
```

### Key Takeaway
Modularity is achieved through **enforced boundaries**, not good intentions. Use barrel exports, dependency inversion, lint rules, and Angular's DI scoping to create hard walls between modules. If a developer can accidentally import an internal function from a sibling module without a lint error, the modularity is illusory.
