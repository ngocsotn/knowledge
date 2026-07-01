# Cookie & Session Authentication: Backend Perspective

Backend architecture, storage systems, and security boundaries for stateful sessions.

## Backend Security & Best Practices
1. **Prevent Session Fixation:**
   * *The Attack:* An attacker traps a user into using a specific session ID. Once the user logs in, the attacker accesses the session.
   * *Mitigation:* **Regenerate the Session ID** immediately upon successful login. Throw away the old pre-login anonymous session ID.
2. **Handle Logout Thoroughly:**
   * Both the client cookie and the server-side record must be deleted.
   * Backend: Clear the session ID key from Redis/database.
   * Frontend: Set the cookie expiration date to the past to instruct the browser to delete it.
3. **Session High-Availability Storage:**
   * *RDBMS (PostgreSQL):* Good for smaller apps; session state survives server restarts.
   * *Key-Value Cache (Redis):* Highly preferred for scale. Redis is extremely fast (`O(1)` read/write), supports automatic keys expiration via TTL, and handles high write volumes effortlessly.

---

## Interview Questions & Answers

### Q1: How do session-based systems scale in horizontally clustered environment?
- **Answer:** Horizontal scaling breaks memory-based sessions because subsequent requests might land on a different server instance. To scale, you must:
  1. **Use a Centralized Session Store:** Store all sessions in a shared database or cache cluster like Redis, making session retrieval fast (`O(1)`) and distributed.
  2. **Sticky Sessions:** Configure the Load Balancer to route a client to the exact same server based on IP or a routing cookie. (Less preferred because it degrades load balancer statistical distribution).
