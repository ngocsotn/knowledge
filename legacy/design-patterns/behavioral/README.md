# Behavioral Design Patterns

Comprehensive interview study guide covering Behavioral design patterns, focusing on object communication, algorithms, state management, and responsibilities.

---

## 1. Meaning of Behavioral Patterns

**Behavioral Patterns** are specifically concerned with algorithms and the assignment of responsibilities between objects. They describe not just patterns of objects or classes but also the patterns of communication between them.

---

## 2. Core Behavioral Patterns

### 1. Observer Pattern
* **Intent:** Defines a one-to-many dependency between objects so that when one object (the **Subject**) changes state, all its dependents (the **Observers**) are notified and updated automatically.
* **Real-world Example:** Event-driven message brokers, newsletter subscriptions, or custom reactive UI frameworks (like React's state effect hooks).

### 2. Strategy Pattern
* **Intent:** Defines a family of algorithms, encapsulates each one, and makes them interchangeable. Strategy lets the algorithm vary independently from clients that use it.
* **Real-world Example:** A shopping cart system that accepts interchangeable shipping methods (e.g., `FedexStrategy`, `UPSStrategy`, `DHLStrategy`) dynamically without modifying the shopping cart calculation.

### 3. Command Pattern
* **Intent:** Encapsulates a request as an independent object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.
* **Real-world Example:** An Undo/Redo editor system where every text insertion or deletion is saved as a discrete `Command` object with an execute and rollback method.

### 4. State Pattern
* **Intent:** Allows an object to alter its behavior when its internal state changes. The object will appear to change its class.
* **Real-world Example:** A document publication workflow: a `Document` can transit across `Draft`, `InReview`, and `Published` states. Calling `.publish()` behaves completely differently depending on the active state.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: How do you prevent memory leaks when implementing the Observer Pattern?
* **Answer:** Memory leaks occur in the Observer pattern because the **Subject maintains a hard reference to all registered Observers** in its collection. If an observer goes out of scope but fails to explicitly unsubscribe, the garbage collector cannot reclaim it because the Subject is still holding its reference (the "lapsed listener" problem). To prevent this, always implement an explicit `unsubscribe()` method that removes the observer, or use **Weak References** (e.g., `WeakMap` in JavaScript) to store observers so they can be garbage collected when no other references exist.

### Q2: What is the difference between the State and Strategy pattern?
* **Answer:** While both patterns utilize composition and delegate work to subclasses, their intents differ:
  * **Strategy** is usually chosen and set once by the client upon initialization (e.g., choosing "PayPal" as the payment method). The strategies are completely independent and unaware of each other.
  * **State** allows behaviors to transition dynamically over time. The individual concrete State classes often trigger state transitions (e.g., `Draft` state explicitly changing the context to `InReview` after approval), making them tightly coupled to the system's lifecycle flow.

### Q3: When should you use the Chain of Responsibility Pattern?
* **Answer:** Use the Chain of Responsibility pattern when you need to process a request through a sequential **pipeline of independent handlers**, where each handler decides either to process the request or pass it to the next link in the chain. A classic example is **HTTP Middleware**: an incoming request passes through an `IPFilterMiddleware`, then an `AuthMiddleware`, then a `RateLimiterMiddleware`. If any middleware fails validation, it breaks the chain and returns a response, preventing downstream handlers from running.

---

## 4. Production-Grade Deep Dive: Dynamic Strategy Router

The **Strategy Pattern** is highly useful for replacing massive `if/else` or `switch` statements with an interchangeable family of algorithms. In production, we combine the **Strategy Pattern** with a **Registry (Map)** to build a **Dynamic Strategy Router** that resolves the correct processor at runtime based on payload metadata.

### 1. Go Payment Strategy Router
Instead of manually matching processors in long branches, we register strategies into a central map on startup.

```go
package main

import (
	"errors"
	"fmt"
	"log"
)

// PaymentStrategy interface defines our strategy contract
type PaymentStrategy interface {
	Process(amount float64) error
}

// StripeStrategy implements PaymentStrategy
type StripeStrategy struct{}
func (s *StripeStrategy) Process(amount float64) error {
	log.Printf("Successfully routed $%.2f to Stripe API", amount)
	return nil
}

// PayPalStrategy implements PaymentStrategy
type PayPalStrategy struct{}
func (p *PayPalStrategy) Process(amount float64) error {
	log.Printf("Successfully routed $%.2f to PayPal gateway", amount)
	return nil
}

// WireTransferStrategy implements PaymentStrategy
type WireTransferStrategy struct{}
func (w *WireTransferStrategy) Process(amount float64) error {
	log.Printf("Successfully processed $%.2f SWIFT wire transfer", amount)
	return nil
}

// PaymentRouter acts as the strategy registry & runtime resolver
type PaymentRouter struct {
	strategies map[string]PaymentStrategy
}

func NewPaymentRouter() *PaymentRouter {
	return &PaymentRouter{
		strategies: make(map[string]PaymentStrategy),
	}
}

// Register adds strategies to our Map registry on startup
func (pr *PaymentRouter) Register(paymentMethod string, strategy PaymentStrategy) {
	pr.strategies[paymentMethod] = strategy
}

// Route dynamic routes matching runtime parameters without switch blocks
func (pr *PaymentRouter) Route(paymentMethod string, amount float64) error {
	strategy, exists := pr.strategies[paymentMethod]
	if !exists {
		return errors.New("unsupported payment method: " + paymentMethod)
	}
	return strategy.Process(amount)
}

func main() {
	router := NewPaymentRouter()
	
	// Registry initialization
	router.Register("stripe", &StripeStrategy{})
	router.Register("paypal", &PayPalStrategy{})
	router.Register("wire", &WireTransferStrategy{})
	
	// Dynamic client requests
	requests := []struct {
		method string
		amount float64
	}{
		{"stripe", 250.00},
		{"paypal", 99.50},
		{"wire", 12500.00},
		{"crypto", 10.00}, // Should trigger fallback error
	}
	
	for _, req := range requests {
		if err := router.Route(req.method, req.amount); err != nil {
			log.Printf("[ERROR] Route failed: %v", err)
		}
	}
}
```

### 2. TypeScript Payment Strategy Router (using Class/Object maps)
```typescript
interface PaymentStrategy {
  process(amount: number): Promise<void>;
}

class StripeStrategy implements PaymentStrategy {
  async process(amount: number): Promise<void> {
    console.log(`[Stripe] Charging $${amount}`);
  }
}

class PayPalStrategy implements PaymentStrategy {
  async process(amount: number): Promise<void> {
    console.log(`[PayPal] Capturing $${amount}`);
  }
}

class ApplePayStrategy implements PaymentStrategy {
  async process(amount: number): Promise<void> {
    console.log(`[ApplePay] Express-authorized $${amount}`);
  }
}

class PaymentRouter {
  // Hash map registry mapping strings to interface strategies
  private registry: Map<string, PaymentStrategy> = new Map();

  register(method: string, strategy: PaymentStrategy): void {
    this.registry.set(method, strategy);
  }

  async execute(method: string, amount: number): Promise<void> {
    const strategy = this.registry.get(method);
    if (!strategy) {
      throw new Error(`Unsupported transaction pathway: ${method}`);
    }
    await strategy.process(amount);
  }
}

// Application instantiation
const router = new PaymentRouter();
router.register('stripe', new StripeStrategy());
router.register('paypal', new PayPalStrategy());
router.register('applepay', new ApplePayStrategy());

// Executing dynamic runtime routes
(async () => {
  await router.execute('stripe', 142.50);
  await router.execute('applepay', 89.00);
})();
```

---

## 5. Architectural Deep Dive: The Saga Orchestrator (Behavioral & Distributed Systems)

In microservices, where transactions span multiple databases, we cannot rely on local databases or traditional two-phase locking (2PC) over networks because of scaling/network-partition blocks. Instead, we use the **Saga Pattern**. 

A Saga is a sequence of local transactions. Each transaction updates database tables within a single service. If a local step fails, the **Orchestrator** triggers **compensating transactions** sequentially in reverse order to roll back state changes and ensure eventual consistency.

```
                  ┌──────────────────────┐
                  │   SagaOrchestrator   │
                  └─┬────┬──────────┬────┘
                    │    │          │
         Success    │    │          │    Failure (Triggers compensations)
     ┌──────────────┘    │          └──────────────┐
     ▼                   ▼                         ▼
┌──────────────┐   ┌──────────────┐         ┌──────────────┐
│ OrderService │   │ StockService │         │ PayService   │
│ - Create()   │   │ - Reserve()  │         │ - Charge()   │
└──────────────┘   └──────────────┘         └──────────────┘
```

### Saga Compensations Orchestrator (Go Implementation)

```go
package main

import (
	"errors"
	"log"
)

// SagaStep defines execution and fallback routines
type SagaStep struct {
	Name       string
	Execute    func() error
	Compensate func() error
}

type SagaOrchestrator struct {
	steps []SagaStep
}

func NewSagaOrchestrator() *SagaOrchestrator {
	return &SagaOrchestrator{}
}

func (so *SagaOrchestrator) AddStep(step SagaStep) {
	so.steps = append(so.steps, step)
}

// Run executes transactions. On failure, triggers compensating transactions in reverse order.
func (so *SagaOrchestrator) Run() error {
	completedSteps := 0
	
	for _, step := range so.steps {
		log.Printf("[SAGA] Executing step: %s", step.Name)
		if err := step.Execute(); err != nil {
			log.Printf("[SAGA FAILURE] Step '%s' failed: %v. Initiating rollback...", step.Name, err)
			so.rollback(completedSteps)
			return err
		}
		completedSteps++
	}
	
	log.Println("[SAGA SUCCESS] Distributed transaction finalized with eventual consistency.")
	return nil
}

func (so *SagaOrchestrator) rollback(completedCount int) {
	// Traverse completed steps in REVERSE order
	for i := completedCount - 1; i >= 0; i-- {
		step := so.steps[i]
		log.Printf("[SAGA ROLLBACK] Executing compensation for step: %s", step.Name)
		if err := step.Compensate(); err != nil {
			// In production, unfailed compensations are queued for retry, written to DLQs, or flagged for manual SRE intervention.
			log.Printf("[CRITICAL ERROR] Compensation failed for step %s: %v", step.Name, err)
		}
	}
}

func main() {
	orchestrator := NewSagaOrchestrator()

	// Step 1: Reserve Order Room
	orchestrator.AddStep(SagaStep{
		Name: "Reserve Order Record",
		Execute: func() error {
			log.Println("Database: Order created status=pending")
			return nil
		},
		Compensate: func() error {
			log.Println("Database: Order cancelled status=cancelled")
			return nil
		},
	})

	// Step 2: Reserve Inventory Stock
	orchestrator.AddStep(SagaStep{
		Name: "Reserve Inventory Stock",
		Execute: func() error {
			log.Println("Warehouse: Deducted 1 unit of stock")
			return nil
		},
		Compensate: func() error {
			log.Println("Warehouse: Credited 1 unit of stock back")
			return nil
		},
	})

	// Step 3: Authorize Card Payment (Fails to demonstrate compensation)
	orchestrator.AddStep(SagaStep{
		Name: "Charge Payment Gateway",
		Execute: func() error {
			return errors.New("insufficient credit limit on visa card")
		},
		Compensate: func() error {
			log.Println("Payment Gateway: Refund issued")
			return nil
		},
	})

	// Run transaction pipeline
	orchestrator.Run()
}
```

