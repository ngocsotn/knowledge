# HTTP Status Codes

Comprehensive guide to HTTP status codes used in backend development.

## The Essential Checklist

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
