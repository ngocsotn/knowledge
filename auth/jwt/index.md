# JSON Web Tokens (JWT): Architecture & Security

Comprehensive study guide covering JWT internals, symmetric vs. asymmetric cryptosystems, JSON Web Key Sets (JWKS), token propagation in microservices, stateless revocation models, client-side storage boundaries, and Refresh Token Rotation (RTR).

---

## 1. Cryptographic Key Management: Symmetric vs. Asymmetric

Modern token-based authentication relies on either shared secrets or asymmetric mathematical key pairs to guarantee token integrity and authenticity.

```
Symmetric (HS256)
[Auth Server / API] ─── (Sign with Secret) ───► [JWT] ─── (Verify with SAME Secret) ───► [Auth Server / API]

Asymmetric (RS256)
[Auth Server] ─────── (Sign with Private Key) ──► [JWT] ─── (Verify with Public Key) ───► [Downstream Services]
```

### A. Symmetric Encryption (HMAC with SHA-256 / HS256)
* **Mechanism:** Uses a single private secret key for both generating the signature (signing) and validating the signature (verifying).
* **Speed:** Extremely fast. Uses symmetric cryptographic hash functions ($O(1)$ overhead).
* **Compromise Surface:** High. If any downstream service consuming the JWT is breached, the shared secret is exposed, allowing the attacker to forge valid tokens for any user in the ecosystem.
* **Best Fit:** Monolithic architectures, or secure internal service-to-service communication where secrets can be safely shared via a private parameter store.

### B. Asymmetric Encryption (RSA/ECDSA with SHA-256 / RS256/ES256)
* **Mechanism:** Uses a mathematically linked key pair:
  - **Private Key:** Kept strictly confidential by the Identity Provider (IdP) / Auth Server to **sign** tokens.
  - **Public Key:** Shared publicly with anyone to **verify** signatures.
* **Speed:** Slower. Mathematical operations (exponentiation over large prime factors in RSA, or elliptic curve point multiplication) make verification $5\times$ to $10\times$ more CPU-intensive than HS256.
* **Compromise Surface:** Minimal. If a downstream verification service is breached, the attacker only gains access to the *public key*, meaning they can verify but **cannot forge** new tokens.
* **Best Fit:** Distributed microservices, public web/mobile clients, third-party API integrations, and single sign-on (SSO) providers.

---

## 2. JWT vs. Stateful Sessions Comparison

| Dimension | Stateful Session | Stateless JWT |
| :--- | :--- | :--- |
| **Server State** | **Stateful:** Server must store active session records in memory/Redis. | **Stateless:** Token contains all state; server is completely stateless. |
| **Scalability** | **Harder:** Requires sticky routing or high-performance shared cache databases. | **Trivial:** Verified locally using cryptography; zero database bottlenecks. |
| **Revocation** | **Instant:** Deleting the session ID from Redis immediately terminates access. | **Delayed:** Token remains valid until `exp` passes, unless blacklisted. |
| **Payload Size** | **Minimal:** Session ID is a small string (~32 bytes), saving bandwidth. | **Large:** Contains headers, claims, signatures (~500–1024+ bytes), adding overhead. |
| **Network Traffic** | **High DB/Cache load:** Every request requires a database or cache query. | **Zero DB load:** Requests are validated purely via CPU-bound cryptography. |

---

## 3. Backend Perspective: JWKS, Propagation, & Revocation

### A. JSON Web Key Set (JWKS) Verification
A **JSON Web Key Set (JWKS)** is a standard JSON structure containing a set of cryptographic public keys used to verify signatures issued by an Authorization Server. It is typically exposed publicly at a well-known endpoint (e.g., `https://auth.domain.com/.well-known/jwks.json`).

#### JWKS Payload Schema
```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "alg": "RS256",
      "kid": "v2_rotation_key_2026_06",
      "n": "u1W_NzS-T_g08gN3s...",
      "e": "AQAB"
    }
  ]
}
```
- **`kty` (Key Type):** Identifies the cryptographic algorithm family (e.g., `RSA` or `EC`).
- **`use` (PublicKey Use):** Intended use. `sig` denotes signature verification; `enc` denotes encryption.
- **`alg` (Algorithm):** The specific signature algorithm (e.g., `RS256`).
- **`kid` (Key ID):** Unique identifier for this specific key. Used by verifying APIs to select the correct public key matching the token's header `kid` claim.
- **`n` & `e`:** Modulus and Exponent values representing the RSA public key.

#### Dynamic JWKS Client Caching & Resiliency
To prevent JWKS network latency or endpoint failures from taking down downstream microservices, implement the **Dynamic JWKS Client** pattern:
1. **In-Memory Cache with high TTL:** Cache fetched JWKeys in-memory (e.g., LRU cache) with a high TTL (e.g., 24 hours) to keep validation local and sub-millisecond.
2. **Stale-While-Revalidate (SWR):** When the cache expires, serve the stale public key to verify incoming tokens while triggering a background, non-blocking network request to fetch the latest JWKS.
3. **Reactive Fetching with Rate-Limiting:** If a token arrives with an unknown `kid`, perform a synchronous lookup against the JWKS endpoint (as the Auth Server may have rotated keys). **Security Guard:** Restrict dynamic fetches to at most once per 10-30 seconds to prevent attackers from sending fake `kid` headers to DDoS your Auth Server.

### B. JWT Propagation in Microservices
Authentication (who you are) and Authorization (what you can do) should be decoupled at the edge:

```
Client ──(External JWT)──► [API Gateway] ──(Internal Clean Headers)──► [Microservices]
                            │   │
                            │   └─(Verifies JWT signature via cached JWKS)
                            ▼
                   [Centralized Auth DB] (Only queried on /login and /refresh)
```

1. **Edge Authentication (Gateway Translation):**
   - The **API Gateway** acts as the Gatekeeper. It intercepts incoming client requests, verifies the external JWT signature, strips the client JWT, and extracts user claims (e.g., `user_id`, `roles`).
   - The Gateway injects these claims as simple downstream HTTP headers:
     - `X-User-ID: 10948`
     - `X-User-Roles: billing_admin, editor`
   - Downstream services (e.g., `Order Service`, `Billing Service`) do not perform any cryptographic token validation or database calls. They read the trusted `X-User-*` headers, keeping them simple and extremely fast.
   - **VPC Gating Security:** The gateway must strip any client-supplied `X-User-*` headers at the edge to prevent header-spoofing. Microservices must reject any traffic not routed through the Gateway.
2. **Microservice-to-Microservice Auth (Zero Trust):**
   - If the internal network is not fully trusted, the API Gateway translates the external JWT into a short-lived (e.g., 60-second expiration), restricted **Internal JWT** signed with a symmetric cluster key. Downstream microservices verify this internal JWT's signature locally.

### C. High-Throughput Token Revocation (Blacklist)
To support instant global revoking (on logout, password change, or account suspension) without turning JWTs back into stateful, database-bound sessions, use a **Hybrid Revocation Cache**:

```
     Client Request (with JWT)
               │
               ▼
     [Step 1: Cryptographic Validation] (Signature, Expiration, Issuer)
               │
            (Valid)
               ▼
     [Step 2: $O(1)$ Blacklist Check]  ──► Query Redis (Key: jti:10293)
               │
           (Not Found)
               ▼
     Allow Access (Stateless & Fast)
```

1. **The Unique Identifier (`jti`):** Inject a unique, high-entropy cryptographic UUID claim (`jti` - JWT ID) into every Access Token during generation.
2. **The Redis Blacklist Store:** When a user triggers an invalidation event (logs out, changes password, or gets banned):
   - Determine the remaining time-to-live of the token: `TTL = Token.exp - CurrentTime`.
   - Write the `jti` to Redis as a blacklisted key: `SETEX blacklist:jti:<jti> <TTL> "revoked"`.
3. **Stateless-First Gating:**
   - The Gateway/API service first validates the token's cryptographic signature locally (stateless, $O(1)$ CPU-bound).
   - If the signature is valid, the Gateway performs a sub-millisecond, non-blocking check to verify if the token's `jti` is present in Redis.
   - If the key does **not** exist in the Redis blacklist, the token is accepted.
4. **Self-Cleaning Storage:** Because the Redis keys are saved with a TTL equal to the token's remaining lifespan, keys expire and delete themselves automatically the exact second the token would have naturally expired anyway, preventing memory bloat.

---

## 4. Frontend Perspective: Storage Boundaries & RTR

### A. Storing JWTs: LocalStorage vs. HttpOnly Cookies

| Dimension | LocalStorage | HttpOnly Cookie |
| :--- | :--- | :--- |
| **Storage Location** | Browser LocalStorage API | Browser Native Cookie Store |
| **XSS Protection** | **None:** JavaScript can read all stored data, making token theft trivial. | **High:** JavaScript cannot access the cookie, preventing token theft. |
| **CSRF Protection**| **Immune:** Browser does not send storage keys automatically. | **Vulnerable:** Browser attaches the cookie automatically to cross-site requests. |
| **Cross-Domain** | Simple custom headers (`Authorization: Bearer <token>`). | Complex (requires SameSite, CORS configurations). |

### B. Token Lifecycle Management (Silent Refresh)
- **Access Tokens (AT):** Short-lived (5-15 mins). Stored strictly in-memory (JS variable).
- **Refresh Tokens (RT):** Long-lived (7-30 days). Stored in `HttpOnly`, `Secure`, `SameSite=Lax` cookies.
- **Silent Refresh Flow:** To prevent session disruption, the frontend registers an in-memory background timer or captures a `401 Unauthorized` interceptor. Before the Access Token expires, it triggers an automatic background `POST /refresh` call using the cookie-attached Refresh Token to retrieve a fresh Access Token.

### C. Refresh Token Rotation (RTR)
Under RTR, refresh tokens are strictly single-use:
1. When a client requests a new Access Token, the submitted `RT_1` is invalidated, and a fresh `RT_2` is returned.
2. **Replay/Theft Detection:** If an attacker steals and replays `RT_1`, the Auth Server detects that `RT_1` was already spent.
3. **Breach Mitigation:** The server immediately flags a potential breach, invalidates the entire **Token Family** associated with that session, revokes access, and forces the true user to log in again with primary credentials.

---

## 5. High-Impact Interview Questions & Answers

### Q1: Write a mock JSON payload of a decoded JWT, identifying the Header, Payload, and the difference between Registered and Custom claims.
* **Answer:**
  - **Header:**
    ```json
    {
      "alg": "RS256",
      "typ": "JWT",
      "kid": "v2_rotation_key_2026_06"
    }
    ```
  - **Payload:**
    ```json
    {
      "iss": "https://auth.example.com",
      "sub": "user_10294",
      "exp": 1782806400,
      "iat": 1782802800,
      "jti": "85196200-70c1-4768-98f8-156fc2187cdc",
      "tenant_id": "acme_corp",
      "roles": ["editor", "billing_admin"]
    }
    ```
  - **Registered Claims:** Standard claims defined by RFC 7519 (`iss` - issuer, `sub` - subject, `exp` - expiration, `iat` - issued at, `jti` - JWT ID). Verifying libraries validate these automatically.
  - **Custom Claims:** Application-specific claims (`tenant_id`, `roles`) used for authorization and business routing.

### Q2: What is the "None" algorithm vulnerability in JWT libraries, and how do you protect against it?
* **Answer:**
- **The Vulnerability:** Many early JWT libraries allowed the header `"alg": "none"` to bypass verification. An attacker could edit a token payload (e.g., setting `"roles": ["admin"]`), change the header to `"alg": "none"`, strip the signature block entirely, and submit the un-signed token. If the backend library parsed the `"none"` algorithm without defensive checks, it accepted the token as valid.
- **The Protection:**
  1. Update and use modern, standard-compliant JWT libraries.
  2. Always explicitly restrict the allowed algorithms inside your backend verification middleware config:
     ```javascript
     // Secure implementation
     jwt.verify(token, publicKey, { algorithms: ['RS256'] }); 
     ```
     This forces the library to reject any token utilizing an algorithm not present in the whitelist, including `"none"`.

### Q3: Explain how the "Refresh Token Rotation (RTR)" token-family tree works to detect and stop token theft.
* **Answer:**
  1. **Standard State:** User has `RT_A` (active) and `AT_A`.
  2. **Refresh Event:** User submits `RT_A` to `/refresh`. The server:
     - Verifies `RT_A`.
     - Invalidates `RT_A` in the database (or marks it as spent).
     - Generates and returns a fresh pair: `RT_B` and `AT_B`.
  3. **Attacker Theft Scenario:** An attacker intercepts or steals `RT_A` and attempts to submit it to `/refresh`.
     - The server queries its token table or Redis and detects that `RT_A` has **already been used**.
     - This indicates a compromise (either the legitimate client or the attacker is replaying).
     - The server executes a **fail-closed defense:** it immediately deletes `RT_B` (the active child token), invalidates the entire linked family tree of tokens, and forces all sessions to be terminated. The user is forced to re-authenticate with credentials.

### Q4: If an attacker modifies the payload of a JWT (e.g., changes `"roles": ["user"]` to `"roles": ["admin"]`), how does the cryptographic signature check detect this modification?
* **Answer:**
  - **How the Signature is Created:** During generation, the server takes the Base64URL-encoded Header and Base64URL-encoded Payload, joins them with a dot (`header.payload`), hashes them (e.g., using SHA-256), and encrypts the hash using its Private Key. This encrypted hash is the signature.
  - **How the Signature is Verified:** During verification, the backend:
    1. Base64URL-decodes the Header and Payload, joins them, and computes a fresh SHA-256 hash.
    2. Decrypts the attached Signature using the server's Public Key to retrieve the original hash.
    3. Compares the freshly computed hash to the decrypted original hash.
    - *If the attacker changed any character in the payload, the newly computed hash will be completely different from the decrypted hash, causing the signature check to fail instantly.*

### Q5: Why is the `exp` claim essential in JWT payloads, and how does it prevent replay attacks?
* **Answer:**
  - **Stateless Expiry:** Since JWTs are stateless, the server does not check a database to see if a token is still active. The `exp` (expiration time) claim is the only boundary that terminates a token's validity.
  - **Replay Mitigation:** If an attacker intercepts an active JWT, they can replay it on subsequent requests to act as the user. The `exp` claim limits the lifespan of this stolen token. Short access token lifespans (e.g., 5-15 minutes) ensure that even if a token is intercepted, it becomes completely useless to the attacker within minutes of the theft.

### Q6: What is a "JWKS Key Rotation", and how do you execute it without causing downtime or active session logouts?
* **Answer:** JWKS Key Rotation is the process of replacing the active signing private key with a new one to comply with security rotation policies or in response to a suspected key leak.
- **The Zero-Downtime Execution Protocol:**
  1. **Phase 1 (Dual Verification):** Generate a new key pair (`Key_New`). Update the Auth Server's JWKS endpoint so that it exposes both the public keys of `Key_Old` and `Key_New`. Continue signing new tokens strictly with `Key_Old`. Let this run for 24-48 hours until all downstream service caches refresh.
  2. **Phase 2 (Switch Signing):** Configure the Auth Server to start signing all newly generated JWTs using the private key of `Key_New`. Incoming requests signed with `Key_Old` are still verified successfully because `Key_Old`'s public key remains present in the JWKS.
  3. **Phase 3 (Deprecate Old Key):** Wait for the maximum lifespan of your access tokens (e.g., 15 minutes) or sessions to expire. Once all active tokens signed with `Key_Old` have expired, remove `Key_Old`'s public key from the JWKS. Key rotation is now complete with zero downtime or user disruption.