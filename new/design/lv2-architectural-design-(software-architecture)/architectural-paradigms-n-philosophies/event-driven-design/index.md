# Event-Driven Design (EDD)

Event-Driven Design structures the system as a reactive network of **asynchronous event streams**.

### A. Core Concepts
1. **Events**: Immutable records of historical state changes.
   - **Domain Events**: Internal to a single Bounded Context (handled synchronously or asynchronously).
   - **Integration Events**: Published externally across different Bounded Contexts via brokers (e.g., Kafka, RabbitMQ) to sync microservices.
2. **Event Sourcing**: Storing the history of state changes as a sequence of events instead of just overwriting current database rows. (See `database/advanced-patterns/` for details).

### B. Choreography vs. Orchestration Sagas
- **Choreography (Reactive)**: Microservices listen to events and react independently. Highly decoupled, but hard to trace.
- **Orchestration (Command-Driven)**: A central State Machine (Orchestrator) explicitly sends commands to services and handles success/compensations. Highly visible, but introduces centralization.

---

## 4. Data-Driven Design

Data-Driven Design focuses on the **database schema, data flows, and structures** as the primary driver of system design, rather than business behaviors or rich object models.

### A. Characteristics
- **Anemic Domain Model**: Business models are simple data holders (DTOs) with getters and setters but **zero business logic**.
- **Service Layer Bloat**: All business validation, calculations, and decisions are written inside separate procedural Service classes or directly inside database **Stored Procedures and Triggers**.
- **Database Centralization**: The relational schema is defined first. Application code is treated merely as an interface to read/write columns to SQL tables.

### B. Trade-offs
- *Pros*: Extremely fast to build for simple CRUD (Create, Read, Update, Delete) applications; highly optimized SQL query access; simple mapping via standard ORMs.
- *Cons*: As business rules grow complex, logic leaks everywhere (controllers, services, SQL views), leading to untestable "spaghetti" code and high database lock contention.

---

## 5. Command-Driven Design & CQRS

Command-Driven Design enforces strict intent-based actions using **Commands** and segregates them from reads using **CQRS (Command Query Responsibility Segregation)**.

```
CQRS Architecture
                   ┌───► [Command Handler] ───► [Write DB (Normalized)]
                   │                                     │
[User Action] ─────┤                              (Asynchronous Sync)
                   │                                     ▼
                   └───► [Query Handler]   ◄─── [Read DB (Elastic/Redis)]
```

### A. Commands vs. Queries
- **Command**: Represents a user **intent to mutate state**. Named using active, business verbs (e.g., `DeductInventoryCommand`). Commands return `void` or a status code—**never** business data. They validate domain rules and write to the Write DB.
- **Query**: Represents a **request for data**. Named descriptively (e.g., `GetActiveUserListQuery`). Queries are strictly read-only, return DTOs, and **never** mutate state. They can bypass domain aggregates and query optimized Read views directly.

### B. Core Benefits
- Solves read/write performance asymmetry (e.g., 99% of traffic is reading catalog pages, 1% is checkout writes). Write and Read databases can scale independently.

---

## 6. Test-Driven Development (TDD)

Test-Driven Development is an evolutionary coding practice where you write tests **before** implementing the actual production code.

### A. The Red-Green-Refactor Cycle
```
   ┌────────────────────────────────┐
   ▼                                │
[RED: Write failing test]           │
   │                                │
   ▼                                │
[GREEN: Write minimal code to pass] │
   │                                │
   ▼                                │
[REFACTOR: Clean code structure] ───┘
```
1. **Red**: Write a highly focused, automated test for a feature before it exists. Run the test and watch it **fail** (asserting the test is valid).
2. **Green**: Write the **absolute minimum** production code required to make the test pass successfully.
3. **Refactor**: Clean up the code (remove duplication, improve variable naming, apply SOLID design) while ensuring the test suite remains **green** (preventing regressions).

---

## 7. Tactical Implementation Example (DDD Aggregate Root in Go)

We model an `Order` aggregate root enforcing business invariants (cannot add items to a paid order, total calculation must be exact) using DDD tactical patterns:

```go
package domain

import (
	"errors"
	"github.com/google/uuid"
)

// Value Object (Immutable)
type Money struct {
	amount   float64
	currency string
}

func NewMoney(amount float64, currency string) (Money, error) {
	if amount < 0 {
		return Money{}, errors.New("amount cannot be negative")
	}
	return Money{amount: amount, currency: currency}, nil
}

// Entity (Has identity, owned by Order Aggregate)
type OrderItem struct {
	ID       uuid.UUID
	ProductID uuid.UUID
	Price    Money
	Quantity int
}

// Aggregate Root (Enforces transaction boundary)
type Order struct {
	ID         uuid.UUID
	CustomerID uuid.UUID
	Items      []OrderItem
	Status     string // "Pending", "Paid", "Cancelled"
	Total      Money
}

func NewOrder(customerID uuid.UUID) *Order {
	return &Order{
		ID:         uuid.New(),
		CustomerID: customerID,
		Status:     "Pending",
		Total:      Money{amount: 0, currency: "USD"},
	}
}

// Business Invariant: Cannot mutate items of a Paid Order
func (o *Order) AddItem(productID uuid.UUID, price Money, qty int) error {
	if o.Status == "Paid" {
		return errors.New("cannot add items to a completed order")
	}
	if qty <= 0 {
		return errors.New("quantity must be positive")
	}

	item := OrderItem{
		ID:        uuid.New(),
		ProductID: productID,
		Price:     price,
		Quantity:  qty,
	}
	o.Items = append(o.Items, item)
	o.recalculateTotal()
	return nil
}

func (o *Order) recalculateTotal() {
	var total float64
	for _, item := range o.Items {
		total += item.Price.amount * float64(item.Quantity)
	}
	o.Total = Money{amount: total, currency: "USD"}
}
```

---

## 8. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between an Entity and a Value Object in Domain-Driven Design (DDD)?
* **Answer**: An **Entity** is defined by its unique **Identity** which persists across changes and time (e.g., a `User` with an ID of `42`. Even if they change their name, email, and password, they are still the same User). A **Value Object** has no identity and is defined strictly by its **Attributes** (e.g., `Money{amount: 5, currency: "USD"}`). If you swap it with another Money object containing `$5 USD`, the application behaves identically. Value Objects are immutable—mutating one returns a brand-new instance.

### Q2: What is an Anemic Domain Model, and why is it considered an anti-pattern in complex systems?
* **Answer**: An Anemic Domain Model occurs when domain classes contain data columns and getters/setters but **zero business behaviors or invariants** (standard Data-Driven design). All business logic is pushed into separate, stateless service classes. It is considered an anti-pattern in complex systems because it violates OOP encapsulation. The domain models cannot defend their own state, allowing services to write invalid values to columns, scattering business validation rules across dozens of files, and making the system extremely hard to audit or test as rules scale.

### Q3: Explain how CQRS separates commands from queries. Why should a Command never return domain data?
* **Answer**: CQRS separates database mutations from database reads. **Commands** represent user actions that mutate state (e.g., `PlaceOrderCommand`). **Queries** represent read-only fetches (e.g., `GetOrderDetailsQuery`). A Command should never return domain data (only status, validation errors, or created IDs) because returning data forces the write model to load complex read-optimized representations, violating the separation of concerns. By keeping Commands void of read data, you can optimize the write database strictly for low-latency transactions and offload heavy queries to specialized, read-only cache indexes.

### Q4: [Struggle Question] Why does Test-Driven Development (TDD) sometimes lead to poor software designs, and how do you prevent it?
* **Answer**: TDD can lead to poor designs when developers become fixated on "testability" rather than "clean architecture". This causes them to over-interface their code (creating mock interfaces for simple internal helpers) or design components that match the specific mock setups of their test frameworks rather than the natural domain boundaries. This results in rigid, over-engineered codebases. To prevent this, never write tests targeting internal, private implementation details. Instead, **test the public API of the logical Bounded Context / Aggregate Root**. This ensures you can freely refactor internal code and classes without breaking or updating your test suite.

## Interview Questions & Answers

### Q1: What are the primary trade-offs of Event-Driven Design?
- **Answer:**
  - **Pros:** Incredibly decoupled; asynchronous processing handles traffic spikes easily; highly scalable.
  - **Cons:** Hard to debug; tracing transaction flows across 10 microservices requires distributed tracing ID contexts; eventual consistency anomalies.
