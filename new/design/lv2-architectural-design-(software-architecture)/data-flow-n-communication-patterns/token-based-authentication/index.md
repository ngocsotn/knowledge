# Token-Based Authentication

Token-Based Authentication is a stateless security pattern where the server issues a signed cryptographic token (such as a JWT) to the client upon successful authentication.
- **Statelessness:** The server does not store active sessions; it verifies the signature of incoming tokens locally using cryptography, enabling trivial horizontal scaling.

## Interview Questions & Answers

### Q1: What is the main security challenge of token-based authentication?
- **Answer:** Revocation. Because tokens are self-contained and validated statelessly, revoking a token before its natural expiration requires maintaining a distributed blacklist cache (e.g., using Redis JTI lookups), which introduces a stateful boundary to an otherwise stateless system.
