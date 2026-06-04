# API Gateway Architecture

Comprehensive interview study guide covering the role of API Gateways, routing, cross-cutting concern offloading, and reverse proxy comparisons.

---

## 1. Meaning & Core Concept

An **API Gateway** is an architectural pattern where a single entry point server intercepts all incoming client requests, handles cross-cutting system operations, and routes them to appropriate downstream microservices.

```
                      Client (SPA / Mobile)
                        │
                        ▼ (HTTPS)
                  ┌───────────┐
                  │API Gateway│ (SSL, Rate-limiting, Auth Offload)
                  └─────┬─────┘
            ┌───────────┼───────────┐
            ▼ (HTTP)    ▼ (HTTP)    ▼ (HTTP)
        ┌───────┐   ┌───────┐   ┌───────┐
        │User-db│   │Order  │   │Payment│ (Microservices)
        └───────┘   └───────┘   └───────┘
```

---

## 2. Key Capabilities & Features

An API Gateway offloads common operations from individual microservices, keeping them thin and business-focused:

1. **Request Routing (Reverse Proxying):** Maps client request paths to physical microservice network locations (e.g., routing `/orders/*` to the Orders service).
2. **Authentication & Authorization Offloading:** Verifies JWT signatures, sessions, or API keys at the edge, rejecting unauthenticated requests before they consume downstream resources.
3. **Rate Limiting & Throttling:** Enforces request quotas per IP, User ID, or API key.
4. **SSL Termination:** Decrypts HTTPS at the gateway, routing unencrypted HTTP internally within the private network to save downstream CPU cycles.
5. **Request & Response Transformation:** Modifies headers, sanitizes inputs, or converts payloads (e.g., SOAP to JSON) dynamically.

---

## 3. API Gateway vs. Reverse Proxy

* **Reverse Proxy (e.g., standard Nginx, HAProxy):** Designed for simple load balancing, static file serving, and path-based routing. It has zero awareness of application-level authentication, billing, or custom API policies.
* **API Gateway (e.g., Kong, AWS API Gateway, Apigee):** Built specifically for APIs. It supports dynamic plugin systems (for OAuth, CORS, analytics, and custom JWT parsing) and can integrate directly with service registries.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: Why is Authentication Offloading at the API Gateway level highly recommended in microservices?
* **Answer:** Authenticating requests at the API Gateway level prevents code duplication: instead of implementing JWT verification, public-key fetching, and error-handling inside every individual microservice, the gateway handles it once at the edge. On success, the gateway attaches verified user metadata (like `X-User-ID: 123` or roles) to the request headers and forwards it to downstream services, which can trust this header implicitly within the secure private network.

### Q2: What are the main drawbacks or risks of deploying an API Gateway?
* **Answer:** While powerful, an API Gateway introduces three main challenges:
  1. **Single Point of Failure (SPOF):** If the gateway crashes, the entire application becomes inaccessible. This is mitigated by running multiple redundant gateway instances behind a Layer 4 load balancer.
  2. **Latency Overhead:** Every request must traverse an extra network hop and undergo parsing/verification, adding minor latency.
  3. **Operational Bottleneck:** If multiple teams share a single gateway configuration, updating routing rules can lead to configuration lockouts or deployment conflicts.

### Q3: What is "BFF" (Backend For Frontend) pattern, and how does it relate to API Gateways?
* **Answer:** The **BFF** pattern is a variation of the API Gateway. Instead of having a single monolithic gateway for all clients, you build **separate, dedicated gateways** for each client type (e.g., one BFF for the iOS app, one BFF for the web app, and one for third-party public integrations). This allows each BFF to tailor API responses (like stripping unneeded fields for mobile screens, or batching multiple requests into a single response payload), optimizing performance for specific device constraints.
