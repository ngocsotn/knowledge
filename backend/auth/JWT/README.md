# Advanced Authentication & Authorization (JWT & Key Management)

Comprehensive staff-level study guide covering JWT internals, symmetric vs. asymmetric cryptosystems, JSON Web Key Sets (JWKS), microservices propagation, Token Lifecycle Management, and high-throughput Revocation patterns.

---

## 1. Cryptographic Key Management: Symmetric vs. Asymmetric

Modern security systems rely on either shared secrets or asymmetric mathematical key pairs to guarantee token integrity.

```
Symmetric (HS256)
[Auth Server / API] ─── (Sign with Secret) ───► [JWT] ─── (Verify with SAME Secret) ───► [Auth Server / API]

Asymmetric (RS256)
[Auth Server] ─────── (Sign with Private Key) ──► [JWT] ─── (Verify with Public Key) ───► [Downstream Services]
```

### Symmetric Encryption (HMAC with SHA-256 / HS256)
* **Mechanism**: Uses a single private secret key for both generating the signature (signing) and validating the signature (verifying).
* **Speed**: Extremely fast. Uses symmetric cryptographic hash functions ($O(1)$ overhead).
* **Compromise Surface**: High. If any downstream service consuming the JWT is breached, the shared secret is exposed, allowing the attacker to forge valid tokens for any user in the ecosystem.
* **Best Fit**: Monolithic architectures, or secure internal service-to-service communication where secrets can be safely shared via a private parameter store.

### Asymmetric Encryption (RSA/ECDSA with SHA-256 / RS256/ES256)
* **Mechanism**: Uses a mathematically linked key pair:
 - **Private Key**: Kept strictly confidential by the Identity Provider (IdP) / Auth Server to **sign** tokens.
 - **Public Key**: Shared publicly with anyone to **verify** signatures.
* **Speed**: Slower. Mathematical operations (exponentiation over large prime factors in RSA, or elliptic curve point multiplication) make verification $5\times$ to $10\times$ more CPU-intensive than HS256.
* **Compromise Surface**: Minimal. If a downstream verification service is breached, the attacker only gains access to the *public key*, meaning they can verify but **cannot forge** new tokens.
* **Best Fit**: Distributed microservices, public web/mobile clients, third-party API integrations, and single sign-on (SSG/SSO) providers.

---

## 2. JSON Web Key Set (JWKS) — RFC 7517

A **JSON Web Key Set (JWKS)** is a standard JSON structure containing a set of cryptographic public keys used to verify signatures issued by an Authorization Server. It is typically exposed publicly at a well-known endpoint (e.g., `https://auth.domain.com/.well-known/jwks.json`).

### A. JWKS Payload Schema
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

* **`keys`**: Array of JWK objects.
* **`kty` (Key Type)**: Identifies the cryptographic algorithm family (e.g., `RSA` or `EC`).
* **`use` (PublicKey Use)**: Intended use. `sig` denotes signature verification; `enc` denotes encryption.
* **`alg` (Algorithm)**: The specific signature algorithm (e.g., `RS256`).
* **`kid` (Key ID)**: Unique identifier for this specific key. Used by verifying APIs to select the correct public key matching the token's header `kid` claim.
* **`n` & `e`**: Modulus and Exponent values representing the RSA public key.

### B. High-Availability Caching & Resiliency Strategies
To prevent JWKS network latency or endpoint failures from taking down the authentication layer of your entire downstream microservice cluster, you must implement the **Dynamic JWKS Client** pattern:

```
Verifying Service
 Incoming JWT (kid=v2) ──► [In-Memory Cache] ─(Hit)─► Verify Signature (Fast)
                                │
                             (Miss)
                                ▼
                    [JWKS Endpoint Rate-Gate] ──► Fetch /.well-known/jwks.json ──► Update Cache
```

1. **In-Memory Cache with high TTL**: Cache fetched JWKeys in-memory (e.g., LRU cache) with a high Time-to-Live (e.g., 24 hours). This keeps token verification local, highly performant ($O(1)$), and independent of Auth Server uptime.
2. **Stale-While-Revalidate (SWR)**: When the cache expires, continue serving the stale public key to verify incoming tokens while triggering an asynchronous, non-blocking background network request to fetch the latest JWKS.
3. **Reactive Fetching with Rate-Limiting**: If a token arrives with an unknown `kid` (not present in the cache), do not immediately fail. Perform a synchronous lookup against the JWKS endpoint, as the Auth Server may have rotated keys. **Security Guard**: Restrict dynamic fetches to at most once per 10-30 seconds (using a sliding-window rate-limiter) to prevent attackers from sending fake `kid` headers to DDoS your Auth Server via the verifying service.
4. **Key Rotation Grace Period**: When rotating keys on the Auth Server, keep the old public key in the JWKS payload for at least 48 hours. Active sessions signed with the old key remain valid while verification caches synchronize.

---

## 3. JWT Propagation in Microservices Architecture

Authentication (who you are) and Authorization (what you can do) should be decoupled to maximize throughput and minimize network hops.

```
Client ──(External JWT)──► [API Gateway] ──(Internal Clean Headers)──► [Microservices]
                            │   │
                            │   └─(Verifies JWT signature via cached JWKS)
                            ▼
                   [Centralized Auth DB] (Only queried on /login and /refresh)
```

### A. Edge Authentication Pattern (Gateway Translation)
1. **The Gateway acts as the Gatekeeper**: The API Gateway intercepts the incoming client request, reads the external JWT, and verifies the signature using the cached JWKS public key.
2. **Token Stripping and Header Injection**: To protect microservices from heavy parsing logic and external dependencies, the Gateway strips the client JWT. It extracts the raw claims (e.g., `user_id`, `tenant_id`, `roles`) and injects them as simple downstream HTTP headers:
  - `X-User-ID: 10948`
  - `X-User-Roles: billing_admin, editor`
  - `X-Tenant-ID: acme_corp`
3. **Downstream Simplicity**: Downstream services (e.g., `Order Service`, `Billing Service`) do not perform cryptographic token validation or call any database. They read the trusted `X-User-ID` headers and execute logic. This keeps microservices completely stateless and incredibly fast.
4. **Security Boundary (VPC Gating)**: The gateway must strip any client-supplied `X-User-*` headers at the edge to prevent external header-spoofing attacks. Downstream microservices must reject any external traffic not routed through the Gateway.

### B. Microservice-to-Microservice Authentication (Zero Trust JWT)
If the internal VPC network is not fully trusted, or if strict compliance requires cryptographic proof of identity across internal boundaries:
1. **Short-Lived Internal JWTs**: The API Gateway (or a security sidecar) translates the external JWT into a highly restricted, short-lived (e.g., 60-second expiration) **Internal JWT**.
2. **Downstream Verification**: Downstream microservices verify this internal JWT's signature using a shared symmetric cluster key (managed securely via HashiCorp Vault or AWS Secrets Manager) or their own local JWKS client.

---

## 4. Refresh Tokens & Token Rotation (RTR)

To minimize the window of opportunity for an intercepted credential, production systems split session persistence into two separate tokens:

```
 Dimension       Access Token (AT)                Refresh Token (RT)
 ───────────   ───────────────────────────────  ───────────────────────────────────
 Lifetime      Very Short (5 - 15 minutes)      Long (7 - 30 days)
 Purpose       Authorize active resource APIs   Request a new Access Token pair
 Exposure      High (sent on every request)     Low (sent only to /refresh endpoint)
 Storage       Client memory or SessionCookie   HttpOnly, Secure, SameSite Cookie
```

### Refresh Token Rotation (RTR) Sequence & Reuse Detection
To protect against stolen long-lived refresh tokens, authorization servers must enforce **Refresh Token Rotation (RTR)**. Under RTR, refresh tokens are strictly **single-use**.

```
Client                                                   Auth Server
 │                                                           │
 ├─► POST /refresh (Submit RT_1) ───────────────────────────►┤
 │                                                           │ (Verifies RT_1)
 │                                                           │ (Marks RT_1 as "used")
 │                                                           │ (Generates AT_2 + RT_2)
 ◄─ Return (AT_2 + RT_2) ────────────────────────────────────┤
 │                                                           │
 │ ─── ATTACKER REPLAY ATTEMPT ───                           │
 ├─► POST /refresh (Submit RT_1 again) ─────────────────────►┤
 │                                                           │ (Detects REUSE of RT_1!)
 │                                                           │ (Fails Closed: Revokes ALL
 │                                                           │  tokens in RT_1's family)
 ◄─ Return 401 Unauthorized (Force Logout) ──────────────────┤
```

#### How Automatic Reuse Detection Works:
1. **Token Genealogy (Families)**: The Auth Server stores a session family record linking all rotated refresh tokens back to the initial login session.
2. **The Spend Marker**: In the database/Redis, each refresh token is stored with a state: `unused` or `used`.
3. **Normal Path**: When `RT_1` is submitted, the server verifies it is `unused`, generates `AT_2` and `RT_2`, marks `RT_1` as `used`, and stores `RT_2` as `unused`.
4. **The Breach Event**:
  - **Scenario A (Attacker first)**: An attacker steals `RT_1` and submits it. The attacker gets `RT_2`. Later, the legitimate user submits `RT_1`. The server detects `RT_1` is already `used`.
  - **Scenario B (User first)**: The legitimate user refreshes with `RT_1` and gets `RT_2`. Later, the attacker submits the intercepted `RT_1`. The server detects `RT_1` is already `used`.
5. **The Fail-Closed Action**: Upon detecting any attempt to use a spent/rotated refresh token, the server immediately **revokes the entire family** of tokens (invalidating `RT_2` and any active access tokens associated with that family) and logs the user out globally, forcing a clean username/password re-authentication.

---

## 5. Token Revocation & Invalidation

Since JWTs are self-contained and validated cryptographically without querying a database, revoking an active access token prior to its natural expiration (`exp`) is a major architectural challenge.

### High-Throughput Hybrid Revocation Pattern
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

1. **The Unique Identifier (`jti`)**: Inject a unique, high-entropy cryptographic UUID claim (`jti` - JWT ID) into every Access Token during generation.
2. **The Redis Blacklist Store**: Establish an append-only Redis cache cluster. When a user triggers an invalidation event (logs out, changes password, or gets banned):
  - Determine the remaining time-to-live of the token: `TTL = Token.exp - CurrentTime`.
  - Write the `jti` to Redis as a blacklisted key: `SETEX blacklist:jti:<jti> <TTL> "revoked"`.
3. **Stateless-First Gating**:
  - The Gateway/API service first validates the token's cryptographic signature locally (stateless, $O(1)$ CPU-bound).
  - If signature is valid, the Gateway performs a sub-millisecond, non-blocking check to verify if the token's `jti` is present in Redis.
  - If the key does **not** exist in the Redis blacklist, the token is accepted.
4. **Self-Cleaning Storage**: Because the Redis keys are saved with a TTL equal to the token's remaining lifespan, keys expire and delete themselves automatically the exact second the token would have naturally expired anyway. This prevents the blacklist from growing infinitely and preserves Redis memory.

---

## 6. Architectural Comparisons

### JWT vs. Stateful Sessions

| Dimension | Stateful Session | Stateless JWT |
| :--- | :--- | :--- |
| **Server State** | **Stateful**: Server must store active session records in memory/Redis. | **Stateless**: Token contains all state; server is completely stateless. |
| **Scalability** | **Harder**: Requires sticky routing or high-performance shared cache databases. | **Trivial**: Verified locally using cryptography; zero database bottlenecks. |
| **Revocation** | **Instant**: Deleting the session ID from Redis immediately terminates access. | **Delayed**: Token remains valid until `exp` passes, unless blacklisted. |
| **Payload Size** | **Minimal**: Session ID is a small string (~32 bytes), saving bandwidth. | **Large**: Contains headers, claims, signatures (~500–1024+ bytes), adding overhead. |
| **Network Traffic** | **High DB/Cache load**: Every request requires a database or cache query. | **Zero DB load**: Requests are validated purely via CPU-bound cryptography. |

---

### JWT vs. Cookie (Clarifying Category Error)
Comparing "JWT vs. Cookie" is a classic interview trap. **They are not competing technologies**:
- **JWT** is a **token data format** (how data is formatted, structured, and signed).
- **Cookie** is a **storage and transport mechanism** (how the browser stores data and automatically sends it with every HTTP request).

A system can store a **JWT** inside a **Cookie**, or inside **LocalStorage**.

#### Storing a JWT: LocalStorage vs. HttpOnly Cookie

```
                      LocalStorage                    HttpOnly Cookie
                ─────────────────────────       ─────────────────────────
 Storage        Browser Storage API             Browser Cookie Store
 XSS Protection  ❌ Vulnerable (JS readable)     ✅ Immune (JS cannot read)
 CSRF Protection ✅ Immune (No auto-sent)        ❌ Vulnerable (Auto-sent with requests)
 Cross-Domain   ✅ Simple custom headers        ⚠️ Complex (requires SameSite config)
```

#### Best Storage Practice:
Always store **Refresh Tokens** and highly privileged **Access Tokens** inside secure **Cookies** using these strict production-grade flags:
1. **`HttpOnly`**: Blocks JavaScript from accessing the cookie, rendering XSS-based token extraction impossible.
2. **`Secure`**: Guarantees the browser will only transmit the cookie over encrypted TLS (HTTPS) connections.
3. **`SameSite=Strict` or `SameSite=Lax`**: Instructs the browser to only send cookies on first-party requests, mitigating **CSRF (Cross-Site Request Forgery)** attacks. Combine with a custom anti-CSRF request header (like `X-CSRF-Token`) for full defense-in-depth.

---

## 7. Absolute Authentication & Authorization Best Practices

1. **Force Algorithm Verification**: Never trust the `"alg"` claim in the incoming JWT header. Explicitly bind your verification logic to a single expected algorithm (e.g., forcing `RS256`).
2. **Set Aggressive Expirations**: Set Access Token lifetime between 5 to 15 minutes. Long-lived Access Tokens are a massive security liability.
3. **Enable Token Rotation (RTR)**: Always pair long-lived Refresh Tokens with RTR and family-based automatic reuse detection.
4. **Protect the Keys**: Rotate Asymmetric Private Keys periodically. Store them in Hardware Security Modules (HSM) or specialized secret managers (AWS Secrets Manager, HashiCorp Vault). Never commit private keys to source control.
5. **Sanitize downstream headers**: If translating JWTs into plain downstream HTTP headers at your Gateway, ensure the Gateway strips any incoming headers matching the internal format before routing to downstream services.
6. **Limit Payload Size**: Keep claims minimal. Storing complex permissions or profiles in JWT claims creates large headers, degrading HTTP performance.
7. **Sign but Don't Encrypt (by default)**: Standard JWT claims are public. Do not store passwords, PII, or API keys inside the payload unless utilizing JWE (JSON Web Encryption).


