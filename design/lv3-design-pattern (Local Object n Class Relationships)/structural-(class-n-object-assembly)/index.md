# Structural Design Patterns: Adapter, Decorator, and Facade

**Structural Patterns** explain how to assemble objects and classes into larger structures while keeping these structures flexible and efficient. They utilize inheritance or composition to allow separate objects to collaborate seamlessly.

---

## 2. Core Structural Patterns

### 1. Adapter Pattern
* **Intent:** Allows objects with incompatible interfaces to collaborate. It acts as a translator.
* **Real-world Example:** Integrating a legacy logger that expects XML formatting into a modern service that outputs JSON. The `XMLToJSONAdapter` wraps the legacy class and translates inputs on-the-fly.

### 2. Decorator Pattern
* **Intent:** Attaches new behaviors to objects dynamically by placing these objects inside special wrapper classes.
* **Real-world Example:** Building a custom coffee order: `SimpleCoffee` can be dynamically wrapped with a `MilkDecorator`, then a `SugarDecorator`, each adding to the final price and behavior without modifying the base class.

### 3. Facade Pattern
* **Intent:** Provides a simplified, high-level interface to a complex set of classes, library, or subsystem.
* **Real-world Example:** A single `OrderFacade` that abstracts a highly complex order process involving checking inventory, authorizing payments, computing taxes, and notifying shipping. The client simply calls `orderFacade.placeOrder(id)`.

### 4. Proxy Pattern
* **Intent:** Provides a placeholder or surrogate for another object to control access to it (e.g., lazy loading, logging, access control, or caching).
* **Real-world Example:** A `CachedDatabaseProxy` that intercept database queries, returning cached results from Redis before forwarding heavy read requests to the real SQL database.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between the Adapter and Decorator pattern?
* **Answer:**
  * **Adapter** changes or translates the interface of an existing object to match a client's expectation, allowing incompatible systems to integrate. It does not add new behaviors.
  * **Decorator** preserves the original interface but **extends or adds new behaviors** to an object dynamically at runtime by wrapping it.

### Q2: How does the Proxy Pattern implement "Lazy Loading" in Hibernate or other ORMs?
* **Answer:** When an ORM loads a parent entity (e.g., `User`) containing a massive one-to-many relationship (e.g., `orders`), it doesn't query the orders table immediately (which would degrade query speeds). Instead, it instantiates a **Proxy** subclass representing the orders array. This proxy intercepts any read access (e.g., calling `user.getOrders()`). Only upon first call does the proxy run the actual SQL query to fetch orders from the database, caching them locally for future reads.

### Q3: Why is the Facade Pattern highly useful when migrating a monolithic system to microservices?
* **Answer:** Migrating to microservices can break frontend client integrations if clients must suddenly orchestrate calls across 10 separate backend microservices. By placing an **API Gateway (acting as a distributed Facade)** at the entry point, the frontend continues to call a single simplified endpoint (e.g., `/checkout`). The Gateway Facade handles the internal orchestration (calling payments, orders, and shipping microservices) on behalf of the client, isolating the client from the underlying architectural migration.

---

## 4. Production-Grade Deep Dive: The Decorator Pattern

The Decorator pattern is standard in production backend engineering, commonly appearing as **HTTP middleware pipelines**. Middlewares dynamically "wrap" or decorate core business handlers with cross-cutting concerns (logging, rate-limiting, authentication) without the core handler ever knowing.

### 1. Go Middleware Decorator (net/http)
In Go, functions are first-class citizens. We can leverage functional closures to dynamically chain decorators.

```go
package main

import (
	"log"
	"net/http"
	"time"
)

// HTTPHandlerDecorator defines our functional decorator signature
type HTTPHandlerDecorator func(http.Handler) http.Handler

// Core Business Logic (The component to decorate)
func processPaymentHandler() http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		log.Println("[CORE] processing transaction...")
		w.WriteHeader(http.StatusOK)
		w.Write([]byte(`{"status":"success"}`))
	})
}

// LoggingDecorator (Decorator 1)
func WithLogging() HTTPHandlerDecorator {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			start := time.Now()
			log.Printf("[LOG START] %s %s", r.Method, r.URL.Path)
			
			// Call the wrapped component
			next.ServeHTTP(w, r)
			
			log.Printf("[LOG END] Completed in %v", time.Since(start))
		})
	}
}

// AuthenticationDecorator (Decorator 2)
func WithAuth() HTTPHandlerDecorator {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			token := r.Header.Get("Authorization")
			if token != "Bearer secret-token" {
				log.Println("[AUTH FAIL] Unauthenticated access rejected.")
				http.Error(w, "Unauthorized", http.StatusUnauthorized)
				return
			}
			log.Println("[AUTH SUCCESS] Valid token detected.")
			next.ServeHTTP(w, r)
		})
	}
}

// Pipeline builder chains multiple decorators sequentially
func Chain(handler http.Handler, decorators ...HTTPHandlerDecorator) http.Handler {
	for i := len(decorators) - 1; i >= 0; i-- {
		handler = decorators[i](handler)
	}
	return handler
}

func main() {
	coreHandler := processPaymentHandler()
	
	// Dynamically decorate core business handler with logging & auth
	decoratedPipeline := Chain(coreHandler, WithLogging(), WithAuth())
	
	http.Handle("/api/v1/payment", decoratedPipeline)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

### 2. TypeScript/JavaScript Middleware Decorator (Express-style)
In Node.js, we can construct standard request-response pipelines where decorator functions pass execution down the chain using a `next()` callback.

```typescript
import express, { Request, Response, NextFunction } from 'express';

type Middleware = (req: Request, res: Response, next: NextFunction) => void;

// Base Controller (The Core Component)
const processPaymentController = (req: Request, res: Response) => {
  console.log("[CORE] Processing secure transaction...");
  res.status(200).json({ status: "success" });
};

// Logging Decorator (Decorator 1)
const withLogging: Middleware = (req, res, next) => {
  const start = Date.now();
  console.log(`[LOG START] ${req.method} ${req.url}`);
  
  // Track response completion
  res.on('finish', () => {
    console.log(`[LOG END] Completed in ${Date.now() - start}ms`);
  });
  
  next(); // Pass control to next decorator
};

// Authentication Decorator (Decorator 2)
const withAuth: Middleware = (req, res, next) => {
  const token = req.headers.authorization;
  if (token !== 'Bearer secret-token') {
    console.log("[AUTH FAIL] Invalid credentials.");
    res.status(401).send("Unauthorized");
    return;
  }
  console.log("[AUTH SUCCESS] Token accepted.");
  next();
};

// Registering the decorated routing pipeline
const app = express();
app.post('/api/v1/payment', withLogging, withAuth, processPaymentController);

app.listen(3000, () => {
  console.log("TypeScript middleware pipeline listening on port 3000");
});
```

## Interview Questions & Answers

### Q1: What is the structural difference between the Decorator and Adapter patterns?
- **Answer:** **Decorator** matches the exact interface of the wrapped object and extends its behavior dynamically without altering the interface. **Adapter** translates a foreign or legacy interface into a completely different, expected target interface.
