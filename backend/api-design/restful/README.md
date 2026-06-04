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
| **POST** | Create a new resource or execute non-idempotent operations. | No | No | `21 Created` |
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

## 4. Hard Interview Questions & Deep Answers

### Q1: What is the difference between PUT and PATCH, and how do you implement PATCH safely to handle concurrent updates?
**Answer**:
- **PUT** is used for complete replacement of a resource. The payload contains the entire updated representation of the resource. If fields are omitted, the server should set them to null or their default values. PUT is **idempotent**.
- **PATCH** is used for partial updates. Only the modified fields are sent. Patch is **non-idempotent** by default (e.g., appending an item to an array or relative operations like `{"increment": 1}`).
- **Safe Patch Implementations**:
  1. **JSON Merge Patch (RFC 7396)**: Sending a partial JSON object. Null values represent deletions. Standard, but cannot handle array manipulation or partial string appends.
  2. **JSON Patch (RFC 6902)**: An array of operations: `[{"op": "replace", "path": "/email", "value": "new@email.com"}]`. Highly precise but more complex to parse and construct.
- **Concurrent Updates Safeguard**:
  Use **Optimistic Concurrency Control (OCC)** using the `If-Match` header and `ETag` (Entity Tag) or `Last-Modified` timestamps.
  1. Client fetches resource: gets `ETag: "version1"`.
  2. Client submits PATCH with `If-Match: "version1"`.
  3. If another update modified the resource to `"version2"` first, the server rejects the request with `412 Precondition Failed`, preventing the second user from overwriting the first user's changes silently (Lost Update Problem).

### Q2: How do you design an idempotent POST endpoint (e.g., for creating a charge or transaction)?
**Answer**:
Making a `POST` request idempotent is critical to avoid duplicate side effects (e.g., charging a customer twice due to a network timeout retry). This is solved using an **Idempotency Key Pattern**:
1. **Idempotency Key**: Client generates a unique UUID (Idempotency Key) and sends it in the header: `Idempotency-Key: f47ac10b-58cc-4372-a567-0e02b2c3d479`.
2. **Key Store**: Server uses a fast, distributed, transactional Key-Value store (like Redis with a TTL of 24 hours) to track keys.
3. **Execution Steps**:
   - **Check**: Server checks if the key exists in Redis inside a lock or atomic transaction (`SETNX`).
   - **In Progress**: If the key exists and the request is still processing, the server returns `409 Conflict` (or a custom header indicating in-progress processing).
   - **Completed**: If the key exists and has an associated cached response, the server returns that cached response directly without running the business logic again (along with a header like `X-Cache-Idempotent: true`).
   - **New Request**: If the key does not exist:
     1. Store key in Redis with status `PROCESSING`.
     2. Execute the payment/creation logic inside a database transaction.
     3. Save the response body and status code in Redis, updating status to `COMPLETED`.
     4. Return the response to the client.

### Q3: What HTTP status codes would you return for validation errors, unauthorized requests, database failures, and resource conflicts?
**Answer**:
Returning precise HTTP status codes enables client-side error-handling automation:
- **Validation Errors (e.g., invalid email, missing fields)**: `422 Unprocessable Entity` is the best practice (RFC 4918) because the syntax is correct (so not `400 Bad Request`), but the business rules are violated. Alternatively, `400 Bad Request` can be used.
- **Authentication Failures (missing or invalid token)**: `401 Unauthorized` (means unauthenticated).
- **Authorization Failures (authenticated, but lacking permissions)**: `403 Forbidden`.
- **Database/Infrastructure Failures (connection timeout, panic)**: `500 Internal Server Error`.
- **Resource Conflicts (e.g., duplicate username, stale revision)**: `409 Conflict`.
- **Resource Not Found**: `404 Not Found`.
- **Rate Limiting**: `429 Too Many Requests`.
- **Service Unavailable (maintenance, temporary overload)**: `503 Service Unavailable`.
