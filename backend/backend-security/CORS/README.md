# Cross-Origin Resource Sharing (CORS)

CORS is a browser-enforced mechanism that uses HTTP headers to allow scripts running in a browser to interact with resources from a different origin.

---

## 1. Meaning & Mechanism

Since the **Same-Origin Policy (SOP)** blocks cross-origin reads by default, CORS acts as a controlled exception mechanism. It allows servers to declare who is permitted to read their responses.

### Simple Requests vs. Preflight Requests

#### 1. Simple Requests
Requests that do not trigger a preflight. They must meet all these conditions:
* Method is `GET`, `POST`, or `HEAD`.
* Content-Type is limited to `application/x-www-form-urlencoded`, `multipart/form-data`, or `text/plain`.
* No custom headers (like `Authorization` or `X-Requested-With`) are sent.

The browser sends these requests directly and checks the returned `Access-Control-Allow-Origin` header to decide whether the script can read the response.

#### 2. Preflight Requests
If a request violates any simple request condition (e.g., sends JSON payload via `application/json`, uses a `PUT`/`PATCH`/`DELETE` method, or includes custom headers), the browser automatically performs a **preflight** check:
1. Browser sends an HTTP `OPTIONS` request to the server first.
2. It includes headers:
   * `Origin`: The calling website origin.
   * `Access-Control-Request-Method`: The intended HTTP method.
   * `Access-Control-Request-Headers`: The custom headers.
3. Server must respond with standard CORS approval headers and a `200` or `204` success status.
4. If approved, the browser sends the actual request.

---

## 2. Core CORS Headers

| Header | Source | Description |
| :--- | :---: | :--- |
| **Origin** | Client | Sent automatically by the browser to specify the calling origin. |
| **Access-Control-Allow-Origin** | Server | Specifies which origin(s) are allowed to access the resource (e.g., `https://example.com` or `*`). |
| **Access-Control-Allow-Methods** | Server | Comma-separated list of approved HTTP methods (e.g., `GET, POST, PUT, DELETE`). |
| **Access-Control-Allow-Headers** | Server | Comma-separated list of approved custom headers. |
| **Access-Control-Allow-Credentials** | Server | Set to `true` if the server allows credentials (cookies, HTTP auth, SSL client certificates) to be sent on cross-origin requests. |
| **Access-Control-Max-Age** | Server | How long (in seconds) the preflight response can be cached by the browser to avoid redundant `OPTIONS` calls. |

---

## 3. Dynamic Whitelisting Code Implementation (Go)

When supporting multiple origins with session cookies (`Access-Control-Allow-Credentials: true`), you **cannot** use the wildcard `*` for the origin. The backend must dynamically validate the incoming origin against a trusted whitelist and echo it back.

### Production-Grade Go CORS Middleware:
```go
package middleware

import (
	"net/http"
)

// Allowed origins whitelist
var allowedOrigins = map[string]bool{
	"https://ngocsotn.com":         true,
	"https://admin.ngocsotn.com":   true,
	"http://localhost:3000":        true, // local development
}

func CORS(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		origin := r.Header.Get("Origin")

		// 1. Dynamic Origin Validation
		if allowedOrigins[origin] {
			w.Header().Set("Access-Control-Allow-Origin", origin)
			w.Header().Set("Access-Control-Allow-Credentials", "true")
		}

		// 2. Set allowed methods & headers for approved requests
		w.Header().Set("Access-Control-Allow-Methods", "GET, POST, PUT, PATCH, DELETE, OPTIONS")
		w.Header().Set("Access-Control-Allow-Headers", "Accept, Content-Type, Content-Length, Accept-Encoding, X-CSRF-Token, Authorization")
		
		// 3. Set preflight caching age (2 hours)
		w.Header().Set("Access-Control-Max-Age", "7200")

		// 4. Handle Preflight OPTIONS requests instantly with no auth check
		if r.Method == "OPTIONS" {
			w.WriteHeader(http.StatusNoContent) // 204
			return
		}

		next.ServeHTTP(w, r)
	})
}
```

---

## 4. Best Practices

1. **Never use wildcard `*` with Credentials:**
   * If `Access-Control-Allow-Credentials: true` is set, `Access-Control-Allow-Origin` **cannot** be `*`. The server must dynamically return the exact request's `Origin` header after validation.
2. **Handle OPTIONS Methods on Public/Unauthenticated Routers:**
   * Ensure your backend framework does not intercept `OPTIONS` requests for authentication checks (e.g., blocking preflights because they lack a Bearer JWT). Preflights must always return unauthenticated `204 No Content`.
3. **Whitelist Specific Origins in Production:**
   * Restrict access to trusted subdomains/domains. Avoid permissive configurations.
4. **Leverage Preflight Caching:**
   * Provide a reasonable `Access-Control-Max-Age` (e.g., 2 hours / 7200 seconds) to reduce network latency caused by redundant preflights.

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: Why do we see a CORS error in the browser console, but the same endpoint works fine in Postman, cURL, or a backend service?
* **Answer:** CORS is purely a **browser-enforced security standard**. It does not exist inside non-browser runtimes like Node.js, Python, cURL, or API clients like Postman. Those tools do not implement the Same-Origin Policy and ignore CORS headers, making the request and reading the response directly without restriction.

### Q2: What happens if a backend API doesn't handle OPTIONS requests?
* **Answer:** The preflight request will fail (often returning `404 Not Found`, `405 Method Not Allowed`, or `500 Internal Server Error`). When the browser's automatic preflight fails, the browser blocks the execution of the actual HTTP request entirely, resulting in a CORS network error on the frontend.

### Q3: How do you handle CORS when your backend supports multiple frontend environments (e.g., staging, production)?
* **Answer:** The backend must dynamically inspect the incoming `Origin` header of the request, check if it matches an allowed whitelist (e.g., configured via environment variables), and if valid, echo that origin back in the `Access-Control-Allow-Origin` response header.

