# OAuth 2.0 & OIDC: Backend Perspective

Machine-to-machine integration, federations, multi-tenant architectures, and resource server token introspection.

## Machine-to-Machine Credentials
Used for **Machine-to-Machine (M2M)** integrations (e.g., background cron jobs, billing microservices).
- **Mechanism**: No user interface or end-user interaction.
- The client sends its `client_id` and `client_secret` directly to the Auth Server via a POST request.
- The Auth Server returns a highly restricted, short-lived Access Token.

---

## Identity Federation & Single Sign-On
Identity Federation bridges user directories across distinct organizational boundaries.
* **Trust Protocol**: Built on secure trust relationships established using exchange certificates (SAML 2.0 or OIDC).
* **Enterprise Login (e.g., Active Directory / SAML)**: An employee logs into a SaaS tool using their internal company credentials. The SaaS tool (Service Provider) delegates authentication to the company’s internal Active Directory (Identity Provider) via SAML or OIDC redirects.
* **Social Login (e.g., Google, Facebook)**: A consumer signs into an e-commerce store with "Log in with Google". The e-commerce tool trusts Google’s signed OIDC tokens to verify the customer’s email and identity.

---

## Multi-Tenant JWT Architectures
In SaaS environments where multiple enterprise customers (tenants) share a single software deployment, the authentication architecture must enforce strict data isolation.

```
 Client (Tenant Acme) ──(Header: JWT iss=auth.com/acme)──► [API Gateway] ──(Injects X-Tenant-ID)──► [Downstream Services]
                                                               │                                      │
                                                               ├─► Fetch Public Key via               └─► Database Isolation
                                                               │   auth.com/acme/jwks.json                (Row-Level Security)
```

### A. Core Multi-Tenant JWT Design
1. **Tenant Claims**: Inject tenant identifier claims into the JWT payload during generation:
   - `tid` (Tenant ID): `acme_corp`
   - `iss` (Issuer): Tenant-specific issuer string (e.g., `https://auth.provider.com/acme_corp`).
2. **Tenant-Specific Signing Keys**:
   - For high isolation, sign each tenant's tokens using a tenant-specific private key.
   - Expose public keys at tenant-specific JWKS paths: `https://auth.provider.com/tenants/<tenant_id>/.well-known/jwks.json`.
3. **Gateway Routing and Verification**:
   - The Gateway inspects the token's `iss` claim or extracts a subdomain (e.g., `acme.saas.com`) to identify the tenant.
   - It fetches the tenant-specific public key from the matching JWKS endpoint and verifies the signature.
   - The Gateway injects `X-Tenant-ID` into downstream headers to drive database routing or PostgreSQL Row-Level Security (RLS).

---

## Interview Questions & Answers

### Q1: How do you design and scale JWT-based authentication for a SaaS platform with strict tenant isolation?
- **Answer:** 
  1. Embed a tenant identifier claim (`tid`) and tenant-scoped issuer (`iss`) in every JWT.
  2. Use tenant-specific private/public keys, exposing public keys at tenant JWKS paths: `/tenants/<tenant_id>/.well-known/jwks.json`.
  3. API Gateway verifies the tenant-specific public keys, extracts the tenant ID, and injects a sanitized `X-Tenant-ID` downstream.
  4. Microservices enforce database multi-tenancy using Row-Level Security (RLS) or connection routing.
