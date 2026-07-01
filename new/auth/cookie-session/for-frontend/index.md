# Cookie & Session Authentication: Frontend Perspective

Comprehensive guide on how frontend applications handle and protect stateful cookies and sessions.

## Storage and Security Flag Controls
Cookies are managed and automatically transmitted by the browser. For proper security:
1. **HttpOnly**: JavaScript cannot read the cookie. This makes token theft via XSS impossible.
2. **Secure**: Cookies are only sent over encrypted HTTPS.
3. **SameSite=Strict**: Mitigates CSRF attacks by ensuring the cookie is only sent on first-party requests.

## Frontend Session Security Checklist
- **Session Fixation Prevention**: Always destroy the session and request a clean session ID from the backend immediately upon login or role changes.
- **CSRF Tokens**: Inject a unique, cryptographically signed token into headers (e.g., `X-CSRF-Token`) for all POST/PUT/DELETE requests to protect session cookies.

## Interview Questions & Answers

### Q1: Is it secure to store the session ID in LocalStorage or in Cookie?
- **Answer:** It is vastly more secure to store session IDs in an `HttpOnly`, `Secure` cookie. If a session ID is stored in `LocalStorage`, any JavaScript running on your page (including third-party analytics, chat widgets, or compromised npm packages) can access it and hijack the user session. An `HttpOnly` cookie blocks all JavaScript access, keeping the session ID safe.
