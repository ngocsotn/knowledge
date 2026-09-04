# Angular Interview Questions & Answers

Comprehensive study guide of real Angular-position interview questions encountered in the field. Covers framework internals, component architecture, state management, performance optimization, error handling, and TypeScript design decisions.

## Table of Contents

- [Q1: Eager and Lazy Evaluations](#q1-eager-and-lazy-evaluations)
- [Q2: How Would You Design a Component?](#q2-how-would-you-design-a-component)
- [Q3: What Options Do You Have to Handle Heavy Computations?](#q3-what-options-do-you-have-to-handle-heavy-computations)
- [Q4: How Do You Avoid Uncontrolled/Unaware Data Mutations?](#q4-how-do-you-avoid-uncontrolledunaware-data-mutations)
- [Q5: What's the Benefit of Top-Level Functions?](#q5-whats-the-benefit-of-top-level-functions)
- [Q6: How Would You Handle Errors, Exceptions, and Unforeseeable Issues?](#q6-how-would-you-handle-errors-exceptions-and-unforeseeable-issues)
- [Q7: What Might Cause Memory Leaks in Angular?](#q7-what-might-cause-memory-leaks-in-angular)
- [Q8: Role-Based Feature Design With and Without Ternary Clauses](#q8-role-based-feature-design-with-and-without-ternary-clauses)
- [Q9: Options to Handle Heavy Frontend Queries on Deeply Nested Objects](#q9-options-to-handle-heavy-frontend-queries-on-deeply-nested-objects)
- [Q10: State Management on Local and Global Scale](#q10-state-management-on-local-and-global-scale)
- [Q11: Between Class and Interface — Which Is Better and for What Purpose?](#q11-between-class-and-interface--which-is-better-and-for-what-purpose)

---

## Q1: Eager and Lazy Evaluations

### What Are They?

**Eager Evaluation** means a value, expression, module, or resource is computed, loaded, or initialized **immediately** — regardless of whether it is actually needed at the moment of consumption. The system pays the full upfront cost during the initial bootstrap phase.

**Lazy Evaluation** means a value, expression, module, or resource is computed, loaded, or initialized **only at the point of first demand**. The system defers all work until the consumer explicitly requests it.

```
┌────────────────────────────────────────────────────────┐
│                    APPLICATION BOOT                     │
├─────────────────────────┬──────────────────────────────┤
│    EAGER (Immediate)    │      LAZY (On-Demand)        │
│                         │                              │
│  AppModule              │  Feature routes              │
│  Core services          │  Dialog/modal modules        │
│  Root guards            │  Admin panels                │
│  HTTP interceptors      │  Heavy chart libraries       │
│  Global state stores    │  PDF/export engines          │
└─────────────────────────┴──────────────────────────────┘
```

### Angular Context: Module & Route Loading

In Angular, the most visible application of this concept is **module loading strategy**:

- **Eager Loading:** Every module declared in `AppModule.imports` is bundled into the main JavaScript chunk. The browser downloads and parses all of it before the first pixel renders.
- **Lazy Loading:** Feature modules are split into separate JavaScript chunks via dynamic `import()` and are only downloaded when the user navigates to the corresponding route.

```typescript
// Eager: included in the main bundle at startup
@NgModule({
  imports: [
    CommonModule,
    CoreModule,       // Always needed
    SharedModule,     // Always needed
  ]
})
export class AppModule {}

// Lazy: downloaded only when user navigates to /admin
const routes: Routes = [
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  }
];
```

### Beyond Modules: Observable Streams

RxJS Observables in Angular are **lazy by default**. An Observable pipeline does not execute any logic until `.subscribe()` is called. This is a critical distinction from Promises (which are eager — a `new Promise(executor)` fires the executor immediately upon construction).

```typescript
// LAZY: This does nothing until subscribed
const data$ = this.http.get('/api/items').pipe(
  map(items => items.filter(i => i.active)),
  tap(items => console.log('Filtered:', items.length))
);

// Execution starts HERE
data$.subscribe(result => this.items = result);
```

### When to Use Which

| Scenario | Strategy | Rationale |
|---|---|---|
| Core layout, navigation, auth guards | **Eager** | Required on every single page load — deferring them adds latency to every interaction |
| Feature modules behind routes (admin, reports) | **Lazy** | Users may never visit these routes; loading them wastes bandwidth and parse time |
| Singleton services (auth, logging, telemetry) | **Eager** | Must be available globally before any component renders |
| Heavy third-party libraries (charts, PDF, maps) | **Lazy** | Enormous bundle sizes that penalize initial Time-to-Interactive (TTI) |
| Observables and streams | **Lazy** (default) | Subscribe only when data is actually needed; avoids unnecessary HTTP calls and computations |
| Preloading after initial paint | **Lazy + PreloadAllModules** | Combines lazy splitting with background preloading after the app becomes interactive |

### Key Takeaway
Eager evaluation is appropriate when the cost of deferral exceeds the cost of upfront initialization. Lazy evaluation is appropriate when the probability of consumption is low or the payload is large. In Angular, the default recommendation is: **eagerly load the shell, lazily load the features**, and use `PreloadAllModules` or custom preloading strategies to warm lazy chunks during idle time.

---

## Q2: How Would You Design a Component?

### The Step-by-Step Component Design Process

Designing a component is not just writing a `.ts` file. It is a deliberate architectural decision that determines reusability, testability, and maintainability. The following steps should be considered sequentially:

```
┌─────────────────────────────────────────────────┐
│            COMPONENT DESIGN PIPELINE            │
├─────────────────────────────────────────────────┤
│  1. Identify Responsibility (Single Purpose)    │
│  2. Classify: Smart vs. Presentational          │
│  3. Define the API Contract (Inputs/Outputs)    │
│  4. Determine State Ownership                   │
│  5. Define Change Detection Strategy            │
│  6. Plan Composition & Projection               │
│  7. Handle Lifecycle & Cleanup                  │
│  8. Ensure Accessibility                        │
│  9. Write Unit Tests Against the Contract       │
└─────────────────────────────────────────────────┘
```

### Step 1: Identify the Single Responsibility
Every component must answer: *"What is the one thing I render and manage?"* If a component does two unrelated things (e.g., renders a user list AND manages a shopping cart), it must be split. This maps directly to the **Single Responsibility Principle (SRP)**.

### Step 2: Classify — Smart (Container) vs. Presentational (Dumb)
- **Smart Components** own the data. They inject services, dispatch actions, manage subscriptions, and orchestrate child components. They know *where* data comes from.
- **Presentational Components** receive data exclusively through `@Input()` and emit events exclusively through `@Output()`. They know *nothing* about services, stores, or HTTP. They are pure rendering units.

```typescript
// SMART (Container) — owns the data pipeline
@Component({
  selector: 'app-user-list-page',
  template: `<app-user-table [users]="users$ | async" (select)="onSelect($event)" />`
})
export class UserListPageComponent {
  users$ = this.userService.getAll();
  constructor(private userService: UserService) {}
  onSelect(user: User) { this.router.navigate(['/users', user.id]); }
}

// PRESENTATIONAL (Dumb) — pure I/O contract
@Component({
  selector: 'app-user-table',
  changeDetection: ChangeDetectionStrategy.OnPush,
  template: `
    <table>
      <tr *ngFor="let user of users" (click)="select.emit(user)">
        <td>{{ user.name }}</td>
      </tr>
    </table>
  `
})
export class UserTableComponent {
  @Input() users: User[] = [];
  @Output() select = new EventEmitter<User>();
}
```

### Step 3: Define the API Contract
The component's `@Input()` and `@Output()` bindings are its **public API**. Design them as you would design a function signature:
- Inputs should be **typed, documented, and defaulted** where reasonable.
- Outputs should emit **domain events**, not internal implementation details.
- Avoid passing entire service references or complex objects when a primitive or a narrow interface suffices.

### Step 4: Determine State Ownership
Decide who owns the state. A component should either **own** its state (and expose changes via outputs) or **receive** state (and be a pure projection of inputs). Mixing both leads to synchronization bugs and unpredictable rendering.

### Step 5: Change Detection Strategy
Presentational components should use `ChangeDetectionStrategy.OnPush`. This tells Angular to skip change detection on the component unless:
- An `@Input()` reference changes.
- An event handler within the component fires.
- An Observable bound via the `async` pipe emits.

This drastically reduces unnecessary re-rendering in large component trees.

### Step 6: Composition & Content Projection
Use `<ng-content>` and `<ng-template>` for composable layouts instead of hard-coding child structures. This enables consumer-side customization without modifying the component internals.

### Step 7: Lifecycle & Cleanup
Implement `OnDestroy` to unsubscribe from manual subscriptions, detach observers, and release external resources. Prefer the `async` pipe or `takeUntilDestroyed()` (Angular 16+) to automate cleanup.

### Step 8: Accessibility (a11y)
Apply ARIA attributes, keyboard event handlers, and semantic HTML. A component that cannot be navigated via keyboard or screen reader is incomplete.

### Step 9: Test Against the Contract
Unit tests should assert the component's public contract — its inputs, outputs, and rendered DOM — not its internal implementation. Use `TestBed` to configure dependencies and verify behavior through the component's external interface.

---

## Q3: What Options Do You Have to Handle Heavy Computations?

As a frontend developer, the browser's **main thread** is the single bottleneck. Any computation that blocks it beyond ~16ms (60fps budget) causes visible UI jank, frozen interactions, and degraded user experience. There are multiple strategies to handle this:

### Option 1: Web Workers (True Parallelism)

Web Workers execute JavaScript in a **separate OS-level thread** with its own memory space. They communicate with the main thread via `postMessage()` serialization.

```typescript
// app.component.ts — Offload heavy computation to a worker
if (typeof Worker !== 'undefined') {
  const worker = new Worker(new URL('./heavy.worker', import.meta.url));
  worker.postMessage({ dataset: largeArray });
  worker.onmessage = ({ data }) => {
    this.processedResult = data.result;
  };
}
```

```typescript
// heavy.worker.ts
addEventListener('message', ({ data }) => {
  const result = expensiveTransformation(data.dataset);
  postMessage({ result });
});
```

- **When to use:** CPU-intensive tasks (sorting millions of records, image processing, cryptographic hashing, complex mathematical models).
- **Limitation:** No DOM access. Data must be serialized (structured clone), so transferring massive objects has overhead. Use `Transferable` objects (ArrayBuffer) for zero-copy transfers.

### Option 2: `requestIdleCallback` / Task Yielding (Cooperative Scheduling)

Break a long synchronous computation into small chunks and yield control back to the browser between chunks.

```typescript
function processInChunks(items: any[], chunkSize: number, processFn: (item: any) => void) {
  let index = 0;
  function nextChunk(deadline: IdleDeadline) {
    while (index < items.length && deadline.timeRemaining() > 0) {
      processFn(items[index++]);
    }
    if (index < items.length) {
      requestIdleCallback(nextChunk);
    }
  }
  requestIdleCallback(nextChunk);
}
```

- **When to use:** Non-urgent background processing where results don't need to appear instantly (analytics aggregation, pre-computation of UI data).

### Option 3: Virtual Scrolling (Angular CDK)

When the "computation" is really about rendering thousands of DOM nodes, don't compute less — **render less**.

```html
<cdk-virtual-scroll-viewport itemSize="48" class="viewport">
  <div *cdkVirtualFor="let item of items" class="item">{{ item.name }}</div>
</cdk-virtual-scroll-viewport>
```

- **When to use:** Lists or tables with 1,000+ rows. The CDK viewport only creates DOM nodes for the visible window (typically 10–20 items) regardless of total list size.

### Option 4: Memoization & Pure Pipes

Cache expensive function results so they are only recomputed when inputs change.

```typescript
@Pipe({ name: 'heavyFilter', pure: true })
export class HeavyFilterPipe implements PipeTransform {
  transform(items: Item[], query: string): Item[] {
    return items.filter(item => item.name.includes(query));
  }
}
```

- **When to use:** Expensive transformations called repeatedly in templates. Pure pipes only re-execute when the input reference changes, while method calls in templates execute on every change detection cycle.

### Option 5: Server-Side Offloading

Move the computation entirely off the client. Send the raw data to a backend endpoint, let the server (or a serverless function) process it, and return only the final result.

- **When to use:** When the dataset exceeds what is reasonable to hold in browser memory (>50MB), or when the computation requires resources unavailable to the browser (database joins, ML inference).

### Option 6: WebAssembly (Wasm)

Compile performance-critical algorithms in C, C++, or Rust to WebAssembly and invoke them from JavaScript. Wasm executes at near-native speed inside the browser sandbox.

- **When to use:** Extreme computation scenarios: video encoding/decoding, physics simulations, cryptographic operations, image manipulation at scale.

### Decision Matrix

| Strategy | Thread | DOM Access | Best For |
|---|---|---|---|
| Web Workers | Separate | No | CPU-heavy transforms, parsing |
| `requestIdleCallback` | Main (yielded) | Yes | Low-priority background work |
| Virtual Scrolling | Main | Yes | Rendering large lists |
| Memoization / Pure Pipes | Main | Yes | Repeated expensive calculations |
| Server Offloading | Server | N/A | Massive datasets, ML, aggregation |
| WebAssembly | Main or Worker | No | Near-native computation |

---

## Q4: How Do You Avoid Uncontrolled/Unaware Data Mutations?

Uncontrolled mutation is the root cause of the most insidious bugs in frontend applications: components rendering stale data, change detection not triggering, state becoming inconsistent across the tree, and debugging becoming a needle-in-a-haystack exercise because the mutation happened somewhere upstream with no trace.

### The Core Problem

```typescript
// DANGEROUS: Mutating the original array in-place
this.users.push(newUser);
// Angular's OnPush change detection does NOT detect this.
// The array reference is identical. The view does not update.
```

### Strategy 1: Immutable Data Patterns

Never modify an object or array in place. Always produce a **new reference**.

```typescript
// SAFE: New array reference — OnPush detects this
this.users = [...this.users, newUser];

// SAFE: New object reference
this.user = { ...this.user, name: 'Updated Name' };
```

This works because Angular's `OnPush` change detection compares `@Input()` references using strict equality (`===`). A new reference triggers re-render; a mutation on the same reference does not.

### Strategy 2: `readonly` and TypeScript Enforcement

Leverage TypeScript's type system to prevent mutations at compile time:

```typescript
interface User {
  readonly id: number;
  readonly name: string;
  readonly roles: readonly string[];
}

// Compile error: Cannot assign to 'name' because it is a read-only property
user.name = 'hacked';

// Use Readonly<T> for deep protection on types
type ImmutableState = Readonly<AppState>;
```

### Strategy 3: Signals (Angular 16+)

Angular Signals enforce mutation awareness by design. You cannot silently mutate a signal's value — you must call `.set()`, `.update()`, or `.mutate()` explicitly, which automatically notifies all consumers.

```typescript
// Signal: mutations are explicit and tracked
users = signal<User[]>([]);

// Explicit mutation — Angular knows about it
this.users.update(current => [...current, newUser]);
```

### Strategy 4: Centralized State with Redux/NgRx

State is stored in a single immutable store. Mutations happen **only** through dispatched actions processed by pure reducer functions. Components receive state via selectors (read-only projections). No component can ever directly mutate the store.

```typescript
// Reducer: pure function, always returns new state
export const userReducer = createReducer(
  initialState,
  on(addUser, (state, { user }) => ({
    ...state,
    users: [...state.users, user]   // New reference, always
  }))
);
```

### Strategy 5: Defensive Copying at Boundaries

When receiving data from external sources (API responses, third-party libraries, parent components), deep-clone at the boundary to prevent upstream mutations from leaking into your component's internal state:

```typescript
@Input() set config(value: Config) {
  this._config = structuredClone(value); // Deep copy at boundary
}
```

### Key Takeaway
The philosophy is: **make mutations visible, intentional, and traceable**. Use immutable patterns for data flow, `readonly` for compile-time safety, Signals or NgRx for controlled state transitions, and `OnPush` change detection to enforce that only reference changes trigger re-renders.

---

## Q5: What's the Benefit of Top-Level Functions?

Top-level functions are standalone functions declared at the module scope rather than as methods inside a class. In Angular's evolution (especially from Angular 14+ onward), there has been a deliberate architectural shift toward top-level functions and away from class-based ceremony.

### Concrete Benefits

**1. Tree-Shakability**
Bundlers (Webpack, esbuild, Vite/Rollup) can statically analyze and eliminate unused top-level functions during dead-code elimination. Class methods are bound to the class prototype and cannot be individually tree-shaken — if any part of the class is imported, the entire class is included in the bundle.

```typescript
// Tree-shakable: if unused, bundler removes it entirely
export function createUserGuard(): CanActivateFn {
  return () => inject(AuthService).isAuthenticated();
}

// Not tree-shakable at the method level
@Injectable()
export class UserGuard implements CanActivate {
  canActivate() { return this.auth.isAuthenticated(); }
  unusedMethod() { /* still bundled */ }
}
```

**2. Reduced Boilerplate**
No class declaration, no constructor injection ceremony, no `implements` clause. The function does one thing, takes explicit parameters, and returns a value.

```typescript
// Functional interceptor (Angular 15+)
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();
  const cloned = req.clone({ setHeaders: { Authorization: `Bearer ${token}` } });
  return next(cloned);
};

// vs. Class-based interceptor (legacy)
@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  constructor(private auth: AuthService) {}
  intercept(req: HttpRequest<any>, next: HttpHandler): Observable<HttpEvent<any>> {
    const cloned = req.clone({ setHeaders: { Authorization: `Bearer ${this.auth.getToken()}` } });
    return next.handle(cloned);
  }
}
```

**3. Composability**
Functions compose naturally via standard function composition, piping, and higher-order functions — a paradigm that is far simpler than class inheritance or decorator chains.

**4. Testability**
A pure function with explicit inputs and outputs is trivially testable without `TestBed`, mocking, or DI container setup. Pass in arguments, assert the return value.

**5. Alignment with Modern Angular APIs**
Angular's modern API surface has deliberately moved to functional patterns:
- `inject()` function replaces constructor DI
- Functional guards (`CanActivateFn`) replace class guards
- Functional interceptors (`HttpInterceptorFn`) replace class interceptors
- Functional resolvers replace class resolvers
- `provideRouter()`, `provideHttpClient()` replace `NgModule` imports

### When to Still Use Classes
- **Services with mutable internal state** that must be shared across consumers (e.g., a `BehaviorSubject`-backed cache).
- **Components and Directives** — Angular still requires the class + decorator pattern for these.
- **Complex object modeling** where encapsulation of internal state behind a public API boundary is genuinely needed.

---

## Q6: How Would You Handle Errors, Exceptions, and Unforeseeable Issues?

### The Two Scales: Local and Global

```
┌─────────────────────────────────────────────────────────┐
│                    ERROR HANDLING LAYERS                 │
├───────────────────────────┬─────────────────────────────┤
│      LOCAL (Component)    │       GLOBAL (App-Wide)      │
│                           │                              │
│  try/catch in methods     │  Global ErrorHandler class   │
│  catchError in pipes      │  HTTP Interceptor fallback   │
│  Form validation errors   │  Router error handlers       │
│  Signal/Observable error  │  Zone.js uncaught handler    │
│  UI error state rendering │  Global error boundary UI    │
└───────────────────────────┴─────────────────────────────┘
```

### Local Error Handling

**1. RxJS `catchError` Operator for Stream-Level Recovery**

```typescript
this.userService.getUser(id).pipe(
  catchError(error => {
    this.errorMessage = 'Failed to load user profile.';
    this.logger.warn('User fetch failed', { id, error });
    return of(null); // Return a fallback value so the stream survives
  })
).subscribe(user => this.user = user);
```

**2. Explicit Error State in Component Templates**

```html
@if (error) {
  <app-error-banner [message]="error" (retry)="reload()" />
} @else if (loading) {
  <app-skeleton-loader />
} @else {
  <app-user-profile [user]="user" />
}
```

**3. Form-Level Validation Errors**
Angular Reactive Forms provide built-in error state management via `FormControl.errors`, `FormGroup.errors`, and custom validators — these are local, scoped, and declarative.

### Global Error Handling

**1. Custom `ErrorHandler` — The Last Line of Defense**

Angular provides a global `ErrorHandler` class. Override it to catch **every** unhandled exception across the entire application — including template binding errors, lifecycle hook crashes, and unhandled promise rejections.

```typescript
@Injectable()
export class GlobalErrorHandler implements ErrorHandler {
  constructor(private logger: LoggingService, private notifier: NotificationService) {}

  handleError(error: unknown): void {
    const parsedError = this.extractError(error);

    // 1. Log to external telemetry (Sentry, Datadog, etc.)
    this.logger.fatal(parsedError);

    // 2. Show user-friendly notification
    this.notifier.showError('An unexpected error occurred. Our team has been notified.');

    // 3. Optionally navigate to a dedicated error page
    // this.router.navigate(['/error']);
  }

  private extractError(error: unknown): Error {
    if (error instanceof HttpErrorResponse) return new Error(error.message);
    if (error instanceof Error) return error;
    return new Error(String(error));
  }
}

// Register in AppModule or app.config.ts
{ provide: ErrorHandler, useClass: GlobalErrorHandler }
```

**2. HTTP Interceptor — Centralized API Error Policy**

```typescript
export const errorInterceptor: HttpInterceptorFn = (req, next) => {
  return next(req).pipe(
    retry({ count: 2, delay: 1000 }),  // Auto-retry transient failures
    catchError((error: HttpErrorResponse) => {
      if (error.status === 401) {
        inject(AuthService).logout();
        inject(Router).navigate(['/login']);
      }
      if (error.status >= 500) {
        inject(NotificationService).showError('Server error. Please try again later.');
      }
      return throwError(() => error);
    })
  );
};
```

**3. Router Error Handling**

```typescript
provideRouter(routes, withNavigationErrorHandler((error) => {
  inject(LoggingService).error('Navigation failed', error);
  inject(Router).navigate(['/not-found']);
}))
```

### Key Principle
**Local handlers recover. Global handlers report and protect.** Components should attempt graceful degradation (show fallback UI, retry, display inline errors). The global handler exists as the safety net — it catches what local handlers missed, logs it to telemetry, and prevents the entire application from crashing silently.

---

## Q7: What Might Cause Memory Leaks in Angular?

### Common Sources of Memory Leaks

**1. Unsubscribed Observables**
The most common cause. When a component subscribes to a long-lived Observable (e.g., a `Subject`, `interval`, router events, store selectors) and the component is destroyed without unsubscribing, the subscription callback retains a closure reference to the destroyed component, preventing garbage collection.

```typescript
// LEAK: subscription lives forever after component destruction
ngOnInit() {
  this.dataService.stream$.subscribe(data => this.data = data);
}

// FIX: takeUntilDestroyed (Angular 16+)
stream$ = this.dataService.stream$.pipe(takeUntilDestroyed());

// FIX: async pipe (auto-unsubscribes on component destroy)
// template: {{ stream$ | async }}

// FIX: Manual destroy subject (legacy pattern)
private destroy$ = new Subject<void>();
ngOnInit() {
  this.dataService.stream$.pipe(takeUntil(this.destroy$)).subscribe(data => this.data = data);
}
ngOnDestroy() { this.destroy$.next(); this.destroy$.complete(); }
```

**2. Event Listeners on Global Objects**
Adding listeners to `window`, `document`, or `body` inside a component without removing them on destroy.

```typescript
// LEAK: listener persists after component dies
ngOnInit() {
  window.addEventListener('resize', this.onResize);
}

// FIX
ngOnDestroy() {
  window.removeEventListener('resize', this.onResize);
}

// BETTER: Use Renderer2 or RxJS fromEvent with takeUntilDestroyed
resize$ = fromEvent(window, 'resize').pipe(takeUntilDestroyed());
```

**3. `setInterval` / `setTimeout` Not Cleared**

```typescript
// LEAK
ngOnInit() { this.intervalId = setInterval(() => this.poll(), 5000); }

// FIX
ngOnDestroy() { clearInterval(this.intervalId); }
```

**4. Closures Capturing Component References**
Third-party libraries (map SDKs, charting libraries, editors) that retain callback references to Angular component instances.

**5. Detached DOM Nodes**
Dynamically created DOM elements (via `Renderer2` or direct DOM APIs) that are removed from the visual tree but still referenced by JavaScript variables.

**6. Circular References in Services**
Singleton services that accumulate references to destroyed component instances (e.g., push to an array on init, never remove on destroy).

### Systematic Prevention Strategy

```
┌────────────────────────────────────────────────────┐
│         MEMORY LEAK PREVENTION CHECKLIST           │
├────────────────────────────────────────────────────┤
│  1. Default to async pipe (auto-cleanup)           │
│  2. Use takeUntilDestroyed() for imperative subs   │
│  3. Audit all ngOnInit for paired ngOnDestroy       │
│  4. Use Renderer2 instead of raw DOM APIs          │
│  5. Profile with Chrome DevTools Memory tab         │
│  6. Enforce lint rules (no-unsubscribed-observable) │
│  7. Weak references for caches and registries      │
└────────────────────────────────────────────────────┘
```

### Detection Tooling
- **Chrome DevTools → Memory Tab → Heap Snapshots:** Take a snapshot before navigating to a component, navigate to it, navigate away, force garbage collection, take another snapshot. Compare retained objects — any component instances that survived are leaked.
- **`allocation instrumentation on timeline`:** Records allocations over time; highlights objects allocated but never freed.
- **Angular DevTools:** Inspect component tree, verify components are properly destroyed when navigated away.

---

## Q8: Role-Based Feature Design With and Without Ternary Clauses

### The Scenario
You have a feature group with a **core function** that behaves identically for all users. However, each role (Admin, Manager, Viewer) has **extra features** that complement and interact with the core function bidirectionally.

### Approach 1: WITHOUT Ternary Clauses — Structural Design Pattern

This is the recommended approach. Instead of littering templates with conditional logic, use Angular's **structural architecture** to separate role-specific concerns into distinct components and load them dynamically.

**Strategy: Component Registry + Dynamic Loading**

```typescript
// 1. Define a role-feature component registry
const ROLE_FEATURE_MAP: Record<UserRole, Type<any>> = {
  admin:   AdminFeaturePanelComponent,
  manager: ManagerFeaturePanelComponent,
  viewer:  ViewerFeaturePanelComponent,
};

// 2. Core component dynamically projects the role-specific panel
@Component({
  selector: 'app-feature-shell',
  template: `
    <section class="core-feature">
      <app-core-function
        [data]="data"
        (action)="handleCoreAction($event)" />
    </section>
    <section class="role-feature">
      <ng-container *ngComponentOutlet="roleComponent; inputs: roleInputs" />
    </section>
  `
})
export class FeatureShellComponent {
  roleComponent: Type<any>;
  roleInputs: Record<string, any>;

  constructor(private auth: AuthService) {
    const role = this.auth.currentUser().role;
    this.roleComponent = ROLE_FEATURE_MAP[role];
    this.roleInputs = { data: this.data, onAction: this.handleRoleAction.bind(this) };
  }
}
```

**Benefits:**
- **Open/Closed Principle:** Adding a new role means adding a new component and a single registry entry — zero modification to the core or shell.
- **Testability:** Each role-specific panel is tested in isolation.
- **Bundle optimization:** Role-specific components can be lazy-loaded.

**Alternative: Directive-Based Feature Toggling**

```typescript
@Directive({ selector: '[appRequiresRole]' })
export class RequiresRoleDirective {
  @Input() appRequiresRole!: UserRole | UserRole[];

  constructor(
    private templateRef: TemplateRef<any>,
    private viewContainer: ViewContainerRef,
    private auth: AuthService
  ) {}

  ngOnInit() {
    const roles = Array.isArray(this.appRequiresRole) ? this.appRequiresRole : [this.appRequiresRole];
    if (roles.includes(this.auth.currentUser().role)) {
      this.viewContainer.createEmbeddedView(this.templateRef);
    }
  }
}
```

```html
<app-core-function [data]="data" />

<div *appRequiresRole="'admin'">
  <app-admin-actions [data]="data" />
</div>
<div *appRequiresRole="'manager'">
  <app-manager-dashboard [data]="data" />
</div>
```

### Approach 2: WITH Ternary Clauses — Inline Conditional

This is simpler but does not scale. Acceptable only for **2–3 simple, small conditions** in small applications.

```html
<app-core-function [data]="data" />

@switch (userRole) {
  @case ('admin') {
    <app-admin-actions [data]="data" (override)="handleOverride($event)" />
  }
  @case ('manager') {
    <app-manager-dashboard [data]="data" />
  }
  @default {
    <app-viewer-summary [data]="data" />
  }
}
```

**Drawbacks:**
- Violates Open/Closed Principle — every new role requires editing the template.
- Template becomes bloated with conditional logic.
- All role-specific components are eagerly bundled regardless of the active user's role.

### Recommendation
Use the structural pattern (Approach 1) by default. Reserve inline ternary/switch for trivially simple, stable role distinctions where adding architectural abstraction costs more than the flexibility it provides.

---

## Q9: Options to Handle Heavy Frontend Queries on Deeply Nested Objects

When dealing with large, deeply nested data structures (trees, recursive objects, 10,000+ record datasets), naive querying and traversal on the main thread becomes a performance bottleneck. Here are the options:

### Option 1: Data Normalization (Flattening)

Transform deeply nested structures into **flat, dictionary-based lookups** at the point of ingestion. This converts O(n × depth) traversals into O(1) map lookups.

```typescript
// BEFORE: Deeply nested
interface Department {
  id: number;
  teams: { id: number; members: { id: number; name: string }[] }[];
}

// AFTER: Normalized flat maps
interface NormalizedState {
  departments: Record<number, { id: number; teamIds: number[] }>;
  teams:       Record<number, { id: number; memberIds: number[] }>;
  members:     Record<number, { id: number; name: string }>;
}
```

This is the pattern used by NgRx Entity, normalizr, and Redux state conventions. Lookups, updates, and deletions become O(1) dictionary operations instead of recursive tree walks.

### Option 2: Memoized Selectors

Compute derived views from normalized state using memoized selectors that cache results and only recompute when upstream inputs change.

```typescript
// NgRx memoized selector — recomputes only when the relevant slice changes
export const selectActiveTeamMembers = createSelector(
  selectTeams,
  selectMembers,
  selectActiveTeamId,
  (teams, members, activeId) => {
    const team = teams[activeId];
    return team?.memberIds.map(id => members[id]) ?? [];
  }
);
```

### Option 3: Indexed Search Structures

Build client-side indexes for frequent search patterns. Libraries like **Fuse.js** (fuzzy search) or **Lunr.js** (full-text search) build inverted indexes over datasets, reducing search from O(n) linear scan to O(log n) or O(1).

### Option 4: Web Workers for Query Offloading

Move the filtering, sorting, or aggregation logic to a Web Worker. The main thread sends the query parameters; the worker returns only the final result set.

### Option 5: Pagination and Cursor-Based Fetching

Don't load all 50,000 records into the browser. Use server-side cursor pagination to fetch only the visible page, and pre-fetch the next page during idle time.

### Option 6: Virtual Scrolling + On-Demand Hydration

Render only the visible rows using CDK Virtual Scroll. Combine with lazy data hydration — fetch detail payloads only when a row enters the viewport.

### Option 7: `structuredClone` + `Transferable` + Worker Pipeline

For truly massive datasets, use `Transferable` ArrayBuffers to send data to a worker with **zero-copy** (no serialization cost), process it, and transfer the result back.

### Combined Strategy

In practice, the optimal approach layers multiple strategies:

```
  API Response → Normalize (flatten) → Store in NgRx/Signal Store
                                            │
                          ┌─────────────────┴─────────────────┐
                          ▼                                   ▼
                  Memoized Selectors               Web Worker (heavy queries)
                          │                                   │
                          ▼                                   ▼
                  Virtual Scroll Viewport          Return filtered result
```

---

## Q10: State Management on Local and Global Scale

### The Spectrum of State Management Options

```
┌────────────────────────────────────────────────────────────────────┐
│                     STATE MANAGEMENT SPECTRUM                      │
├──────────────────┬─────────────────────┬─────────────────────────┤
│   LOCAL (Micro)  │  SHARED (Meso)      │  GLOBAL (Macro)          │
│                  │                     │                          │
│  Component state │  Service + Subject  │  NgRx / NGXS / Akita    │
│  Template vars   │  Signal-based store │  Signal Store (NgRx 18+)│
│  Form state      │  Input/Output chain │  URL/Router state        │
│  Signal()        │  ContextProviders   │  Server cache (TanStack) │
└──────────────────┴─────────────────────┴─────────────────────────┘
```

### Local State Options

**1. Component-Level Variables / Signals**
The simplest form. State lives inside a single component and dies with it.

```typescript
@Component({ ... })
export class CounterComponent {
  count = signal(0);
  increment() { this.count.update(c => c + 1); }
}
```

- **When:** State is consumed only by this component and its template.
- **Benefit:** Zero coupling, zero infrastructure overhead.

**2. Reactive Forms State**
Angular Reactive Forms are a built-in local state manager for form-centric components. `FormGroup` tracks values, validity, dirty/pristine state, and disabled state reactively.

### Shared State Options

**3. Service + BehaviorSubject (Classic Pattern)**

```typescript
@Injectable({ providedIn: 'root' })
export class CartService {
  private _items = new BehaviorSubject<CartItem[]>([]);
  items$ = this._items.asObservable();

  addItem(item: CartItem) {
    this._items.next([...this._items.value, item]);
  }
}
```

- **When:** 2–5 components need to share state, but a full store is overkill.
- **Benefit:** Lightweight, uses standard Angular DI.

**4. Signal-Based Lightweight Store (Angular 17+)**

```typescript
@Injectable({ providedIn: 'root' })
export class CartStore {
  private _items = signal<CartItem[]>([]);
  items = this._items.asReadonly();
  totalPrice = computed(() => this._items().reduce((sum, i) => sum + i.price, 0));

  addItem(item: CartItem) { this._items.update(items => [...items, item]); }
}
```

- **When:** Same as above, but prefer signal semantics over Observable ceremony.

### Global State Options

**5. NgRx Store (Redux Pattern)**

Full unidirectional data flow: **Action → Reducer → State → Selector → Component**.

```typescript
// Actions
export const loadUsers = createAction('[Users] Load');
export const loadUsersSuccess = createAction('[Users] Load Success', props<{ users: User[] }>());

// Reducer
export const userReducer = createReducer(
  initialState,
  on(loadUsersSuccess, (state, { users }) => ({ ...state, users, loaded: true }))
);

// Selector
export const selectAllUsers = createSelector(selectUserState, state => state.users);

// Effect (side effect isolation)
loadUsers$ = createEffect(() => this.actions$.pipe(
  ofType(loadUsers),
  switchMap(() => this.userService.getAll().pipe(
    map(users => loadUsersSuccess({ users }))
  ))
));
```

- **When:** Enterprise applications with complex state transitions, multi-team development, time-travel debugging requirements, or audit-trail needs.
- **Benefit:** Predictable, testable, debuggable (Redux DevTools), enforces immutability.
- **Cost:** Boilerplate. Small apps don't need this.

**6. NgRx SignalStore (Modern Alternative)**

Combines the discipline of NgRx with the simplicity of Signals. Less boilerplate than classic NgRx Store.

```typescript
export const UsersStore = signalStore(
  { providedIn: 'root' },
  withState<UsersState>({ users: [], loading: false }),
  withMethods((store, usersService = inject(UsersService)) => ({
    async loadUsers() {
      patchState(store, { loading: true });
      const users = await usersService.getAll();
      patchState(store, { users, loading: false });
    }
  })),
  withComputed(({ users }) => ({
    activeUsers: computed(() => users().filter(u => u.active))
  }))
);
```

### Best Practices for Extensibility, Reusability, and Non-Coupling

1. **Use the lightest tool that fits.** Don't install NgRx for a counter. Don't use component variables for cross-cutting authentication state.
2. **Separate read from write.** Expose read-only projections (selectors, readonly signals) and dedicated mutation methods. Never expose the raw `BehaviorSubject` or mutable signal.
3. **Keep stores feature-scoped.** Don't create one monolithic global store. Create feature stores (`UsersStore`, `CartStore`, `NotificationsStore`) that each manage a bounded context.
4. **Facade pattern for store access.** Wrap store interactions behind a Facade service so components don't couple to the store library's API. Swapping NgRx for SignalStore becomes a one-file change.
5. **Derive, don't duplicate.** Use `computed()` or `createSelector()` to derive values from base state rather than storing duplicate copies.

---

## Q11: Between Class and Interface — Which Is Better and for What Purpose?

### The Core Distinction

| Aspect | Interface | Class |
|---|---|---|
| **Exists at runtime?** | No — erased during TypeScript compilation | Yes — compiled to JavaScript constructor function |
| **Bundle size impact** | Zero | Adds code to the bundle |
| **Can instantiate?** | No | `new MyClass()` |
| **Can hold implementation?** | No (shape declaration only) | Yes (methods, constructors, state) |
| **Can be used with `instanceof`?** | No | Yes |
| **Can be used as DI token?** | No (TypeScript-only construct) | Yes (Angular DI resolves classes) |
| **Supports multiple inheritance?** | Yes (`implements A, B`) | Single class inheritance only |

### When to Use Interfaces

Use interfaces when you need to define **the shape of data or a contract** without any behavioral implementation:

```typescript
// Data Transfer Object shapes — no behavior, no instantiation needed
export interface UserDto {
  id: number;
  name: string;
  email: string;
}

// Service contracts — defines WHAT a service must do, not HOW
export interface Logger {
  info(message: string): void;
  error(message: string, context?: unknown): void;
}

// Configuration typing
export interface AppConfig {
  apiUrl: string;
  featureFlags: Record<string, boolean>;
}
```

### When to Use Classes

Use classes when you need **runtime behavior, instantiation, DI tokens, or encapsulated state**:

```typescript
// Angular Services — must be classes for DI
@Injectable({ providedIn: 'root' })
export class UserService {
  private cache = new Map<number, User>();

  getUser(id: number): Observable<User> {
    if (this.cache.has(id)) return of(this.cache.get(id)!);
    return this.http.get<User>(`/api/users/${id}`).pipe(
      tap(user => this.cache.set(id, user))
    );
  }
}

// Domain models with behavior
export class Money {
  constructor(private amount: number, private currency: string) {}
  add(other: Money): Money {
    if (this.currency !== other.currency) throw new Error('Currency mismatch');
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

### The Angular-Specific Caveat

In Angular's dependency injection system, **interfaces cannot be used as injection tokens** because they are erased at compile time. If you need to inject an abstraction, you must use either:
- An abstract class (acts as both a type and a runtime token)
- An `InjectionToken<T>`

```typescript
// Abstract class as DI token (preserves runtime identity)
export abstract class Logger {
  abstract info(msg: string): void;
  abstract error(msg: string): void;
}

// Provide a concrete implementation
{ provide: Logger, useClass: ConsoleLogger }

// InjectionToken for interface-typed values
export const APP_CONFIG = new InjectionToken<AppConfig>('app.config');
{ provide: APP_CONFIG, useValue: { apiUrl: '/api', featureFlags: {} } }
```

### The Rule of Thumb
- **Default to interfaces** for data shapes, DTOs, and contracts. They add zero runtime cost.
- **Escalate to classes** when you need instantiation, runtime type checking, encapsulated state, or Angular DI integration.
- **Use abstract classes** when you need both an interface contract *and* a DI-resolvable token.
