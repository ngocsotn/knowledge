# JWT & Authentication: Frontend Perspective

Frontend storage, Silent Refresh flows, and token lifecycle management.

## Storing JWTs: LocalStorage vs. HttpOnly Cookie
```
                      LocalStorage                    HttpOnly Cookie
                ─────────────────────────       ─────────────────────────
 Storage        Browser Storage API             Browser Cookie Store
 XSS Protection  ❌ Vulnerable (JS readable)     ✅ Immune (JS cannot read)
 CSRF Protection ✅ Immune (No auto-sent)        ❌ Vulnerable (Auto-sent with requests)
 Cross-Domain   ✅ Simple custom headers        ⚠️ Complex (requires SameSite config)
```

## Token Lifecycle Management (Silent Refresh)
- **Access Tokens (AT)**: Short-lived (5-15 mins). Stored in-memory.
- **Refresh Tokens (RT)**: Long-lived (7-30 days). Stored in `HttpOnly`, `Secure` cookies.
- **Silent Refresh Flow**: To prevent session disruption, the frontend triggers an automatic background `POST /refresh` call before the Access Token expires.

## Interview Questions & Answers

### Q1: How does Refresh Token Rotation (RTR) protect against token theft?
- **Answer:** Under RTR, refresh tokens are strictly single-use. When a client requests a new Access Token, the submitted `RT_1` is invalidated and a fresh `RT_2` is returned. If an attacker replays `RT_1`, the Auth Server detects the reuse of a spent token, flags a potential breach, and immediately revokes the entire family of linked tokens, forcing the user to log in again with primary credentials.
