# Stateful Session Authentication

# Stateful Session Authentication

Comprehensive interview study guide covering traditional cookie-based, stateful session authentication, security, and storage architecture.

---

## 1. Meaning & Mechanism

Stateful session authentication relies on the server maintaining a persistent record (session store) of logged-in users, while the browser stores only a unique reference identifier (session ID) in a cookie.

### Authentication Flow
```
Browser                                                  Server (API)
  │                                                            │
  ├─► POST /login (Credentials) ───────────────────────────────┤
  │                                                            │ (Validates DB)
  │                                                            │ (Creates Session)
  │                                                            │ (Stores in Redis)
  ├─◄ Set-Cookie: session_id=xyz123 (HttpOnly, Secure) ────────┤
  │                                                            │
  │ (Subsequent request)                                       │
  ├─► GET /dashboard (Cookie automatically sent) ──────────────┤
  │                                                            │ (Reads Redis for xyz123)
  ├─◄ Return HTML/JSON ────────────────────────────────────────┤
```

---

## Session-Based vs. Token-Based (JWT) Auth
| Dimension | Stateful Session | Stateless JWT |
| :--- | :--- | :--- |
| **Server State** | High. Server stores every active session. | Zero. Server is completely stateless. |
| **Scalability** | Harder. Requires distributed cache (Redis) or sticky sessions. | Easy. No shared state database required. |
| **Revocation** | Instant. Delete session from store. | Delayed. Must wait for expiration or use a blacklist. |
| **Payload Size** | Tiny (contains only session ID). | Large (contains full payload + signature). |

---
