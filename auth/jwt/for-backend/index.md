# JWT & Authentication: Backend Perspective

Backend cryptographic validation, JWKS caching, and high-throughput revocation models.

## JSON Web Key Set (JWKS) Verification
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

## JWT Propagation in Microservices
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

## High-Throughput Token Revocation (Blacklist)
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

## Interview Questions & Answers

### Q1: How do you verify asymmetric signatures dynamically using JWKS?
- **Answer:** The backend extracts the `kid` claim from the incoming token's header. It matches the `kid` against its locally cached public keys (JWKS). If there is a cache miss, it dynamically fetches the latest keys from `/.well-known/jwks.json`, updates its cache (with strict rate-limiting to prevent DDoS), and performs the cryptographic signature check using the correct public key.
