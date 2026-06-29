# Backend Basics: Core HTTP & REST API Design

Comprehensive interview study guide covering core web protocols, REST API design, HTTP methods, and status codes.

---

## 1. Core HTTP Concepts

### HTTP Methods: Safe vs. Idempotent
* **Safe Methods:** Do not modify resources. They are read-only.
  * *Methods:* `GET`, `HEAD`, `OPTIONS`.
* **Idempotent Methods:** Multiple identical requests have the same side-effect as a single request.
  * *Methods:* `GET`, `PUT`, `DELETE`, `HEAD`, `OPTIONS`.
  * *Why `POST` is NOT idempotent:* Sending `POST /orders` multiple times creates multiple orders.
  * *Why `PATCH` is NOT idempotent (usually):* `PATCH` can be idempotent, but a patch payload like `{"increment": 1}` changes the resource state with each call.

| Method | Safe | Idempotent | Usage |
| :--- | :---: | :---: | :--- |
| **GET** | Yes | Yes | Retrieve a representation of a resource. |
| **POST** | No | No | Create a new resource or execute non-idempotent operations. |
| **PUT** | No | Yes | Replace an entire resource or create it if its ID is client-determined. |
| **PATCH** | No | No/Yes | Partially update a resource. |
| **DELETE** | No | Yes | Remove a resource. |

### HTTP Status Codes: The Essential Checklist
* **2xx (Success)**
  * `200 OK`: Request succeeded. Returned with a body.
  * `201 Created`: Resource successfully created. Includes a `Location` header.
  * `204 No Content`: Request succeeded, but no response body (e.g., successful `DELETE` or empty `PUT`).
* **3xx (Redirection)**
  * `301 Moved Permanently`: Resource has a new permanent URL. Browser caches this.
  * `302 Found` / `307 Temporary Redirect`: Resource is temporarily elsewhere. `307` guarantees the HTTP method cannot change.
  * `304 Not Modified`: Cached version is still valid (uses `ETag` or `If-None-Match`).
* **4xx (Client Errors)**
  * `400 Bad Request`: Malformed payload, validation failure.
  * `401 Unauthorized`: Client lacks credentials or session is invalid. (Means "Unauthenticated").
  * `403 Forbidden`: Client is authenticated but lacks permission for the specific resource.
  * `404 Not Found`: Resource does not exist.
  * `409 Conflict`: Conflict with current state (e.g., duplicate unique field like email).
  * `429 Too Many Requests`: Rate limit exceeded.
* **5xx (Server Errors)**
  * `500 Internal Server Error`: Unhandled server exception.
  * `502 Bad Gateway`: Upstream server returned invalid response (e.g., Nginx proxying to crashed Go process).
  * `503 Service Unavailable`: Server down for maintenance or overloaded.
  * `504 Gateway Timeout`: Upstream server took too long to respond.

---

## 2. Best Practices in REST API Design

1. **Use Nouns, Not Verbs, for URIs:**
   * *Bad:* `GET /getUserDetails?id=123` or `POST /createNewUser`
   * *Good:* `GET /api/v1/users/123` or `POST /api/v1/users`
2. **Represent Relationships with Nesting:**
   * `GET /api/v1/users/123/posts`: Retrieve posts written by user 123.
3. **Use Filtering, Sorting, and Pagination on Collections:**
   * Keep base URIs clean. Pass modifiers as query parameters:
   * `GET /api/v1/posts?status=published&sort=-created_at&page=2&limit=20`
4. **Consistent Error Payloads:**
   * Always return a structured JSON error body instead of raw text.
   * Format:
     ```json
     {
       "error": {
         "code": "validation_failed",
         "message": "The request payload failed validation.",
         "details": {
           "email": "must be a valid email address"
         }
       }
     }
     ```
5. **API Versioning:**
   * Path-based versioning is highly preferred for caching and predictability: `/api/v1/posts`.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between PUT and PATCH?
* **Answer:** `PUT` replaces the entire target resource with the request payload. If any fields are omitted from the request, they are set to null or default values. `PATCH` applies a partial update, modifying only the fields specified in the request while leaving other attributes untouched.

### Q2: What is the difference between 401 Unauthorized and 403 Forbidden?
* **Answer:** `401 Unauthorized` specifically means **unauthenticated**. The user has not provided valid credentials or has not logged in. `403 Forbidden` means **unauthorized**. The user's identity is verified, but they do not have the required role or permissions to access the requested resource.

### Q3: How do you handle pagination in a high-scale REST API?
* **Answer:** There are two main patterns:
  1. **Offset-based Pagination (`LIMIT X OFFSET Y`):** Good for simple use cases, but scales poorly (`O(N)` DB lookup) and suffers from drift issues (items added/deleted while scrolling causes duplicate/skipped items).
  2. **Cursor-based Pagination (`LIMIT X WHERE id > last_seen_id`):** Highly performant (`O(log N)` lookup on index) and resilient to drift. However, it does not support jumping to arbitrary pages (e.g., page 5) and requires a unique, sequentially ordered cursor field.
