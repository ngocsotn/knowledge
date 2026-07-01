# JSON Web Tokens (JWT) Fundamentals

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

## JWT vs. Stateful Sessions Comparison
| Dimension | Stateful Session | Stateless JWT |
| :--- | :--- | :--- |
| **Server State** | **Stateful**: Server must store active session records in memory/Redis. | **Stateless**: Token contains all state; server is completely stateless. |
| **Scalability** | **Harder**: Requires sticky routing or high-performance shared cache databases. | **Trivial**: Verified locally using cryptography; zero database bottlenecks. |
| **Revocation** | **Instant**: Deleting the session ID from Redis immediately terminates access. | **Delayed**: Token remains valid until `exp` passes, unless blacklisted. |
| **Payload Size** | **Minimal**: Session ID is a small string (~32 bytes), saving bandwidth. | **Large**: Contains headers, claims, signatures (~500–1024+ bytes), adding overhead. |
| **Network Traffic** | **High DB/Cache load**: Every request requires a database or cache query. | **Zero DB load**: Requests are validated purely via CPU-bound cryptography. |

---
