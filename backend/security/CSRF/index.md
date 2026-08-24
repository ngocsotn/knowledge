# Cross-Site Request Forgery (CSRF)

CSRF is an attack that forces an authenticated user to execute unwanted actions on a web application in which they are currently logged in.

---

## 1. Meaning & Mechanism

Since browsers automatically attach cookies (including session cookies) to outgoing requests matching the target domain, an attacker can exploit this default behavior.

### How CSRF Works
```
User (Authenticated on Bank.com)
  │
  ├─► User visits Malicious.com in a separate tab
  │
  ├─► Malicious.com runs script/submits a hidden form:
  │     POST https://bank.com/transfer?amount=10000&to=attacker
  │
  └─► Browser automatically sends Bank.com session cookies!
      Bank.com processes request as if the user intended it.
```

---

## 2. Prevention & Mitigation Techniques

### 1. SameSite Cookie Attribute (Primary Modern Defense)
By setting `SameSite=Lax` or `SameSite=Strict` on session cookies:
* The browser will omit cookies on cross-origin writes/requests initiated by third-party domains.
* In the scenario above, the POST request from `Malicious.com` to `Bank.com` would not carry the session cookie, causing Bank.com to reject the call as unauthenticated.

### 2. Anti-CSRF Tokens (Synchronizer Token Pattern)
* **How it works:**
  1. When a user loads a page, the server generates a cryptographically secure, random, single-use token tied to the user's current session.
  2. The server injects this token into a hidden form field or returns it in a custom header.
  3. When a user submits a state-changing request (POST, PUT, DELETE), they must include this token.
  4. The server compares the submitted token with the token stored in the session.
* **Why it works:** A malicious external site cannot read the token from the user's session due to the Same-Origin Policy (SOP), meaning they cannot include the valid token in their forged requests.

### 3. Double Submit Cookie Pattern (Stateless)
* Perfect for stateless/REST APIs (no server session storage):

```
               DOUBLE-SUBMIT COOKIE PATTERN VERIFICATION
       
Client (Browser)                                        Server (REST API)
       │                                                       │
 1.    │─── POST /api/payment ────────────────────────────────>│
       │    Headers:                                           │
       │      - Cookie: csrf-token=abc123xyz                   │
       │      - X-CSRF-Token: abc123xyz                        │
       │                                                       │
 2.    │                                            ┌──────────┴──────────┐
       │                                            │  Token Match Check: │
       │                                            │   Cookie Token ==   |
       │                                            │   Header Token?     │
       │                                            └──────────┬──────────┘
       │                                                       │
       │                                            YES        │        NO
       │                                    ┌──────────────────┴──────────────────┐
       │                                    ▼                                     ▼
 3.    │<── 200 OK (Processed!) ────────────                         [Return 403 Forbidden]
```

1. Server generates a random CSRF token and sets it as a client-side cookie (not necessarily `HttpOnly`, but `Secure`).
2. The frontend JavaScript reads this cookie and copies its value into a custom request header (e.g., `X-CSRF-Token`).
3. The server receives the request and validates that the token value in the cookie matches the token value in the custom header.
* **Why it works:** A malicious site can cause a request to carry the cookie, but due to SOP, it cannot read the cookie to copy its value into the custom header.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: Why do GET requests not need CSRF protection?
* **Answer:** By standard REST and HTTP conventions, `GET` requests must be **safe**—meaning they are read-only and do not alter state on the server. Since CSRF is designed to prevent unauthorized state changes (e.g., transferring funds, changing passwords), protecting `GET` requests is unnecessary as long as the backend properly guarantees safe execution of all read-only methods.

### Q2: Does SameSite=Lax completely eliminate the need for anti-CSRF tokens?
* **Answer:** While `SameSite=Lax` offers excellent default protection, it does not fully replace anti-CSRF tokens. Certain subrequests (like standard links) can still send cookies. If an API has state-changing endpoints exposed via `GET` (violating REST principles), or if a user uses an outdated browser that doesn't support the `SameSite` attribute, the application remains vulnerable. Defense-in-depth dictates combining `SameSite` flags with CSRF tokens.

### Q3: Why does adding a custom header like `X-Requested-With` protect APIs from CSRF?
* **Answer:** Under CORS regulations, if client-side JavaScript attempts a cross-origin request that includes a custom header, the browser is forced to perform a **preflight (OPTIONS)** request first. If the server is not configured to accept requests from the malicious origin, the preflight will fail, and the actual state-changing request containing the session cookie will never be sent.
