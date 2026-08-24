# RESTful API Design

Comprehensive study guide for designing scalable, maintainable, and robust RESTful APIs for modern web applications.

---

## 1. What is REST?

**Representational State Transfer (REST)** is an architectural style designed by Roy Fielding in 2000 for distributed hypermedia systems. It is not a protocol, but rather a set of constraints that systems must follow to be considered "RESTful."

### Six Core Architectural Constraints of REST

1. **Client-Server Architecture**: Separates user interface concerns from data storage concerns. Allows independent evolution of frontend and backend.
2. **Statelessness**: Every request from client to server must contain all necessary information to understand and process the request. The server must not store session state about the client.
3. **Cacheable**: Servers must label responses as cacheable or non-cacheable. Clients and intermediaries (CDN, proxies) can cache responses to improve performance and reduce server load.
4. **Uniform Interface**: The crucial constraint that simplifies and decouples the architecture:
   - **Resource Identification in Requests**: Resources are identified using URIs (Uniform Resource Identifiers).
   - **Resource Manipulation through Representations**: When a client holds a representation of a resource (e.g., JSON), it has enough information to modify or delete it.
   - **Self-Descriptive Messages**: Each message contains enough information to describe how to process it (e.g., `Content-Type: application/json`).
   - **HATEOAS (Hypermedia As The Engine Of Application State)**: Clients discover available actions dynamically through hyperlinks returned in responses.
5. **Layered System**: A client cannot tell whether it is connected directly to the end server or to an intermediary (load balancer, reverse proxy, gateway). This enables scalability and security policies.
6. **Code on Demand (Optional)**: Server can temporarily extend client functionality by transferring executable code (e.g., JavaScript).

---

## 2. Richardson Maturity Model (RMM)

A framework developed by Leonard Richardson to grade APIs based on their adherence to REST constraints.

| Level | Name | Description | Example |
| :--- | :--- | :--- | :--- |
| **Level 0** | **The Swamp of POX** | Plain Old XML / Single Endpoint. Uses HTTP purely as a transport protocol. All requests go to one URI with custom payloads. | `POST /api` with body `<getProfile id="123"/>` |
| **Level 1** | **Resources** | Introduces individual URIs for distinct resources, but still uses a single HTTP method (usually `POST`). | `POST /api/users/123` with body `{"action": "get"}` |
| **Level 2** | **HTTP Verbs** | Uses correct HTTP methods (GET, POST, PUT, DELETE, PATCH) and standard status codes. | `GET /api/users/123` returning `200 OK` |
| **Level 3** | **Hypermedia Controls** | Fully RESTful. Includes HATEOAS links to guide the client on valid next transitions/actions. | `GET /api/users/123` returns user details and a link `{"rel": "deactivate", "href": "/api/users/123/deactivate"}` |

---

## 3. Best Practices & Design Patterns

### URI Naming Conventions
- Use **nouns**, not verbs, to represent resources:
  - ❌ `GET /api/getUsers`
  -  `GET /api/users`
- Use plural nouns consistently: `/api/users`, `/api/articles`.
- Use hierarchical relationships: `/api/users/123/orders/456`.
- Use kebab-case for multi-word paths: `/api/billing-transactions`.

### HTTP Methods & Idempotency
- **Idempotency** means making multiple identical requests has the same side-effect on the server as making a single request.
- **Safety** means the request does not modify the state of the resource (read-only).

| Method | Description | Safe? | Idempotent? | Successful Response Code |
| :--- | :--- | :--- | :--- | :--- |
| **GET** | Retrieve a representation of a resource. | **Yes** | **Yes** | `200 OK` |
| **POST** | Create a new resource or execute non-idempotent operations. | No | No | `201 Created` |
| **PUT** | Replace an existing resource completely, or create if non-existent. | No | **Yes** | `200 OK` / `204 No Content` |
| **PATCH** | Apply partial modifications to a resource. | No | No (usually) | `200 OK` / `204 No Content` |
| **DELETE** | Remove a resource. | No | **Yes** | `200 OK` / `204 No Content` |

### API Versioning Strategies
1. **URI Versioning (Most Popular)**: `/api/v1/users`
   - *Pros*: Simple, visible, works well with caching proxies.
   - *Cons*: Breaks URI design theory (changing version changes the resource identity).
2. **Query Parameter Versioning**: `/api/users?version=1`
   - *Pros*: Easy to set defaults if parameter is missing.
   - *Cons*: Harder to configure routing rules in API Gateways.
3. **Accept Header (Content Negotiation) Versioning**: `Accept: application/vnd.company.v1+json`
   - *Pros*: Elegant, leaves URIs clean.
   - *Cons*: Difficult to test in browser; complicates HTTP caching.

---

## 4. Advanced Architectural Challenges & Engineering Solutions

Operating large-scale REST services introduces edge cases around concurrent edits, double-payment creations, and standardized error-handling taxonomies.

### A. Complete vs. Partial Updates: Managing Concurrency in Updates
When clients submit modifications, you must choose between standard complete state overwrites (`PUT`) or targeted delta updates (`PATCH`).

```
Optimistic Concurrency Control (OCC) Flow:
Client ─────────── GET /users/123 ───────────> Server (Returns ETag: "v1")
Client (User A) ─── PATCH /users/123 ────────> Server (If-Match: "v1") -> Succeeds (ETag becomes "v2")
Client (User B) ─── PATCH /users/123 ────────> Server (If-Match: "v1") -> Fails (412 Precondition Failed)
```

1. **PUT vs. PATCH:**
   * **`PUT`:** Completely replaces the target resource. The client sends the *entire* state representation. If a client omits fields, the server must nullify or default them. `PUT` is inherently **idempotent**.
   * **`PATCH`:** Applies partial modifications. It is **non-idempotent** by default (e.g., executing relative operations like `{"increment": 10}`).
2. **PATCH Formats:**
   * **JSON Merge Patch (RFC 7396):** Sending a simple JSON object containing only mutated keys. Passing a key with a value of `null` deletes the key.
     * *Limit:* Cannot handle complex array operations or partial string updates.
   * **JSON Patch (RFC 6902):** A strict array of operations to execute in order:
     `[{"op": "replace", "path": "/email", "value": "new@email.com"}, {"op": "remove", "path": "/temporaryToken"}]`
3. **Optimistic Concurrency Control (OCC):**
   To resolve the **Lost Update Problem** (where User B silently overwrites User A's changes), implement OCC via headers:
   * The server returns an **`ETag`** hash or `Last-Modified` timestamp in response headers.
   * The client must cache this ETag. When sending `PUT` or `PATCH`, it includes the ETag in the **`If-Match`** request header.
   * The server verifies the ETag before applying changes. If another client has updated the resource first (modifying the database ETag), the server rejects the request with **`412 Precondition Failed`**, forcing the client to re-fetch.

---

### B. Designing Idempotent Write Operations (The Idempotency-Key Pattern)
Double-submitting a `POST` request (due to a mobile connection retry or network socket timeout) can result in dual payments or duplicate database entries. Implementing an **Idempotency Key Pattern** makes `POST` endpoints safe.

```
                  THE IDEMPOTENCY KEY LIFECYCLE
                                │
                      [POST Request Arrives]
                 (Idempotency-Key: <UUID> Header)
                                │
                                v
               ┌─────────────────────────────────┐
               │  Does Key Exist in Redis Cache? │
               └────────────────┬────────────────┘
                                │
                     YES        │        NO
          ┌─────────────────────┴─────┐  └───────────────┬──────────────────────┐
          ▼                           ▼                  ▼                      ▼
  [Status: PROCESSING]    [Status: COMPLETED]    [Store Key: PROCESSING]  [Execute DB Transaction]
          │                           │                  │                      │
          v                           v                  v                      v
[Return 409 Conflict]   [Return Cached Response] [Save Response in Redis] [Return Successful Response]
```

1. **Client Setup:** The client generates a unique UUID locally for the action and sends it in the header: `Idempotency-Key: b72bc38f-a9cb...`
2. **Check & Lock:** The server checks if this key exists in a fast distributed memory store (like Redis with a 24-hour TTL) using an atomic operation like `SETNX`.
3. **The State Machine Logic:**
   * **If Status is `PROCESSING`:** If the key exists but the initial request is still executing in the background, return **`409 Conflict`** (or a custom header directing retry spacing).
   * **If Status is `COMPLETED`:** If the request has already been successfully executed, return the *original cached response payload* directly without hitting the business database again (along with a header like `X-Cache-Idempotent: true`).
   * **If Key is New:** Store the key with status `PROCESSING`. Execute the database operations inside a database transaction. Save the response body and HTTP status code in Redis, update the status to `COMPLETED`, and return the response.

---

### C. Standardized Error Handling Protocols
Standardizing API error reporting enables client-side SDK automated parsing and clean error logging. Use HTTP status codes logically:

1. **Validation & Business Logic Failures:**
   Use **`422 Unprocessable Entity`** (RFC 4918) when the request format is correct (so not a `400 Bad Request`), but the internal business validations fail (e.g., a username is already taken, or an password lacks capital letters).
2. **Identity & Authorization Gates:**
   * **`401 Unauthorized`:** Implies *unauthenticated*. The client has provided invalid or missing credentials.
   * **`403 Forbidden`:** The client is authenticated but does not possess the permissions or RBAC roles necessary to access the resource.
3. **Resource Availability:**
   * **`404 Not Found`:** The target resource does not exist.
   * **`410 Gone`:** The resource was permanently deleted and will never be available again (excellent for system cleanup optimization).
4. **Rate Limiting:**
   Use **`429 Too Many Requests`**. Always accompany this with a **`Retry-After: 3600`** header, specifying how many seconds the client must wait before retrying.
5. **System Resiliency Failures:**
   * **`500 Internal Server Error`:** General catch-all for database connection pools crashing, internal code panics, or uncaught exceptions.
   * **`503 Service Unavailable`:** The server is temporarily overloaded or down for planned maintenance.

---

## 5. Interview Masterclass: High-Impact Q&As

### Q1: What is the mechanical difference between PUT and PATCH, and how do you safely coordinate concurrent PATCH requests?
* **Answer:**
  * **`PUT`:** Used for complete resource replacement. The client sends the *entire* state layout. Missing properties must be defaulted or set to null by the server. PUT is **idempotent**.
  * **`PATCH`:** Used for partial updates (deltas). Only specified properties are updated. Patch is **non-idempotent** by default.
  * **Safe Concurrency Control:** To avoid the **Lost Update Problem** (User B overwriting User A's changes), implement **Optimistic Concurrency Control (OCC)**. The server returns a cryptographic hash in the **`ETag`** response header. When a client sends a `PATCH` request, they must attach this ETag inside the **`If-Match`** request header. If the server detects that the database record's ETag has changed (meaning another user has completed an update first), it rejects the client's PATCH with a **`412 Precondition Failed`** error, forcing a safe re-fetch.

### Q2: Explain how the Idempotency Key Pattern makes REST POST calls safe against network drops and client retries.
* **Answer:** Duplicate submissions of a state-changing POST request (like a credit card charge) are blocked using an **Idempotency Key**:
  1. The client generates a unique UUID locally and sends it in the header: `Idempotency-Key: <UUID>`.
  2. The server tries to store this key in Redis with a status of `PROCESSING` inside a lock or atomic transaction (such as `SETNX` with a 24-hour TTL).
  3. If the key already exists and its status is `PROCESSING`, the server returns a `409 Conflict` (meaning the original request is still in-progress).
  4. If the status is `COMPLETED`, the server returns the *original cached response payload* directly, bypassing the database entirely.
  5. If the key is new, the server processes the charge inside a database transaction, updates the Redis key status to `COMPLETED` along with the response payload, and returns the successful response.

### Q3: Outline the levels of the Richardson Maturity Model (RMM) and their architectural significance.
* **Answer:** RMM grades API RESTfulness on a scale of 0 to 3:
  * **Level 0 (Sw Swamp of POX):** Uses HTTP strictly as a transport protocol. All requests go to a single endpoint (e.g., `POST /api`) with raw XML/JSON payloads detailing custom operations.
  * **Level 1 (Resources):** Introduces individual URIs for distinct entities (e.g., `GET /api/users/123`), but still uses a single HTTP method (`POST`) for all actions.
  * **Level 2 (HTTP Verbs):** Employs correct HTTP verbs (GET, POST, PUT, DELETE, PATCH) and maps standard HTTP status codes correctly.
  * **Level 3 (Hypermedia Controls / HATEOAS):** Fully RESTful. The server returns hypermedia links along with resource payloads to dynamically direct the client on what actions are available next (e.g., a checkout resource response includes a link to `cancel` or `pay`).

### Q4: Why is returning `422 Unprocessable Entity` preferred over `400 Bad Request` for data validation errors?
* **Answer:**
  * **`400 Bad Request`:** Indicates syntactic errors in the request structure. It means the server failed to compile or parse the JSON string itself (e.g., broken braces, malformed strings).
  * **`422 Unprocessable Entity`:** Indicates semantic errors. The JSON syntax is completely correct and parsed successfully, but the business rules or values inside the payload violate system specifications (e.g., email is missing an `@` symbol, or an integer is negative). Using `422` allows the client's HTTP client library to distinguish between a malformed transmission issue and a business logic validation failure.

