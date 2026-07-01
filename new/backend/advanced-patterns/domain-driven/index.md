# Domain-Driven Design (DDD)

Comprehensive study guide covering Domain-Driven Design principles, Strategic Patterns, and Tactical Patterns.

## Strategic Design Patterns
Domain-Driven Design (DDD) aligns the software architecture directly with the business domain. It is divided into **Strategic** and **Tactical** patterns.

```
DDD Strategic Boundaries (Context Mapping)
┌─────────────────────────────────┐       ┌─────────────────────────────────┐
│        Billing Context          │       │        Inventory Context        │
│                                 │       │                                 │
│  [Entity: Invoice]              │       │  [Entity: StockItem]            │
│  [ValueObject: Currency]        │       │  [ValueObject: SKU]             │
└───────────────┬─────────────────┘       └────────────────┬────────────────┘
                │ (Upstream / Downstream Link via Event)   │
                ▼                                          ▼
   [Message Broker / Integration Event: OrderPaid] ────────┘
```

### A. Strategic Design Patterns
1. **Ubiquitous Language**: A shared, precise vocabulary used equally by developers, domain experts, and product managers to eliminate translation errors.
2. **Bounded Context**: A strict logical boundary (typically mapping to a microservice) within which a domain model has a single, unambiguous meaning. The word "User" might mean "Buyer" in the Checkout Context, but "Recipient" in the Shipping Context.
3. **Context Mapping**: Defining how different Bounded Contexts integrate and communicate (e.g., Upstream/Downstream, Shared Kernel, Customer/Supplier).

### B. Tactical Design Patterns
- **Entity**: An object defined by its unique thread of continuity and **Identity** (e.g., a `User` with a unique ID). Two entities with different IDs but identical names are different.
- **Value Object**: An immutable object defined strictly by its **Attributes/Values** (e.g., `Money{amount: 100, currency: "USD"}`). Two value objects with identical attributes are completely interchangeable. They possess no identity and no lifecycle.
- **Aggregate Root**: A cluster of associated Entities and Value Objects treated as a single transaction boundary. External objects can only reference the Aggregate through its single parent, the **Aggregate Root** (e.g., an `Order` is the Root; `OrderItem` entities inside cannot be modified directly without going through `Order`).
- **Domain Event**: A record of a business-significant state change that has already occurred (e.g., `OrderSubmittedEvent`).
- **Domain Service**: Stateless business logic that does not naturally belong inside a single Entity or Value Object (e.g., a `FundsTransferService` coordinating two Account entities).

---

## Popular Interview Questions & Answers

### Q1: What is the Ubiquitous Language in DDD?
- **Answer:** Ubiquitous Language is a common, shared language developed by both domain experts and software developers. It ensures that technical terms, class names, method names, and database schemas align perfectly with real-world business terms, eliminating translation errors and communication gaps between business and technology.

### Q2: What is a Bounded Context?
- **Answer:** A Bounded Context is a strategic boundary within which a specific domain model applies. Inside this boundary, terms and concepts have unique, unambiguous meanings. For example, in an e-commerce system, a "Product" model in the Inventory Context might track physical dimensions and warehouse locations, while "Product" in the Billing Context tracks pricing and tax rates.
