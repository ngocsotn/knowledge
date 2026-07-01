# API Gateway Pattern

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

## Interview Questions & Answers

### Q1: Why is Authentication Offloading at the API Gateway level highly recommended?
- **Answer:** Simplified downstream security. Offloading signature checks and token parsing to the gateway means downstream microservices do not need cryptographic keys or caching libraries. They receive trusted, parsed HTTP headers (like `X-User-ID`), keeping microservices lightweight, fast, and secure.
