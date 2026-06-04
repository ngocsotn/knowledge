# Senior & Staff-Level CS Mock Interview Scenarios

This study guide presents high-signal, staff-level system architecture mock interview scenarios. Each scenario simulates an engineering design committee interview, detailing deep problem statements, architectural tradeoffs, step-by-step resolution logs, and exact verbal responses to impress senior panel interviewers.

---

## Scenario 1: Flash Sale Cache Stampede & Redis Cluster Overload

### 1. Problem Statement
Your company is launching an ultra-high-profile flash sale where 50,000 users will simultaneously load a single "Active Discount Vouchers" dashboard at exactly 12:00:00 UTC. 
* **The Risk:** If the cache expires at 12:00:00 or is evicted, 50,000 requests will instantly fall through to the PostgreSQL database in a single second, inducing a database connection pool blowout, locking transaction threads, and causing a catastrophic service outage (**Cache Stampede** / **Thundering Herd**).
* **The Hashing Risk:** These voucher records use keys like `voucher:active:user_123`, `voucher:active:user_456`. When distributed across a 3-node Redis Cluster, multi-key transaction pipelines fail because keys hash to different Redis hash slots.

---

### 2. High-Impact Architectural Solution

#### Component Architecture Diagram
```
                     50,000 Concurrent Requests
                                 │
                                 ▼
                     ┌───────────────────────┐
                     │   API Gateways (L7)   │
                     └───────────┬───────────┘
                                 │
                        (In-Flight Deduplication)
                                 │
                     ┌───────────▼───────────┐
                     │   Singleflight Pool   │
                     └───────────┬───────────┘
                                 │
                        (Cache Miss / Hit)
                                 │
            ┌────────────────────┴────────────────────┐
            ▼                                         ▼
  ┌───────────────────┐                     ┌───────────────────┐
  │   Redis Cluster   │                     │ PostgreSQL Master │
  │ (Using `{user}`)  │                     │ (Only 1 query!)   │
  └───────────────────┘                     └───────────────────┘
```

#### Code Defense: Go Singleflight Integration
We intercept cache misses with an in-memory **Singleflight** (Request Deduplicator) pool. If 10,000 concurrent goroutines trigger a cache miss for the exact same key, `singleflight.Group` suppresses 9,999 calls. It spawns only **one** database transaction, sharing the single returned query result with all 10,000 waiting threads.

```go
package main

import (
	"context"
	"fmt"
	"sync"
	"time"

	"golang.org/x/sync/singleflight"
)

type VoucherService struct {
	cacheGroup singleflight.Group
	redisCache map[string]string // Mock Redis
	dbConn     *sync.Mutex       // Mock Database Lock
	queryCount int
}

func (s *VoucherService) FetchVoucherData(ctx context.Context, key string) (string, error) {
	// 1. Attempt Cache Read
	if val, exists := s.redisCache[key]; exists {
		return val, nil
	}

	// 2. Cache Miss - Execute Singleflight to collapse duplicate queries
	v, err, shared := s.cacheGroup.Do(key, func() (interface{}, error) {
		// Only the first thread enters here to execute the expensive DB query
		return s.queryDatabaseDirect(key)
	})

	if err != nil {
		return "", err
	}

	fmt.Printf("Key: %s | Result Shared across threads: %v\n", key, shared)
	return v.(string), nil
}

func (s *VoucherService) queryDatabaseDirect(key string) (string, error) {
	s.dbConn.Lock()
	defer s.dbConn.Unlock()
	s.queryCount++ // Tracks exact physical DB queries made
	time.Sleep(100 * time.Millisecond) // Simulate slow query
	return "Voucher-Data-Payload", nil
}
```

#### Redis Hashing Slots Alignment via Hash Tags
To execute multi-key operations (like transaction scripts, Lua executions, or pipeline merges) across keys without slot failures, we wrap the sharding key portion in **curly brackets `{}`**:
* `user:10001:voucher:active` and `user:10001:profile` hash to different slots.
* `{user:10001}:voucher:active` and `{user:10001}:profile` force Redis to hash only the bracketed string `user:10001`, routing both keys to the **exact same Redis Cluster node**, enabling high-speed atomic transactions.

---

### 3. Interview Q&A Script

**Interviewer:** *"If the cache is empty during a peak load spike, how do you prevent your database from melting down?"*

**Your Verbal Response:**
> "I mitigate this by layering an in-memory execution barrier using the **Singleflight pattern** inside the application middleware layer, coupled with **probabilistic cache pre-expiration**. 
> 
> When a cache miss occurs under high concurrency, instead of allowing all threads to query the database, we route the key through a singleflight group. The singleflight group intercepts the requests, allows only the first thread to execute the physical database transaction, and holds the other threads in a blocked state. Once the single database query returns, the proxy populates the cache and multiplexes the single result back to all waiting requests. 
> 
> For highly active keys, I also implement **XFetch (probabilistic early expiration)**. As the cache lifetime nears expiry, background threads probabilistically trigger an early background cache refresh based on query frequency, ensuring the cache is refreshed *before* it actually expires, keeping database misses at absolute zero."

---

## Scenario 2: Sharding a Global Financial Ledger (Zero-Downtime Scaling)

### 1. Problem Statement
Your system manages a global ledger where users register millions of multi-currency transactions daily. The main database is a single massive PostgreSQL server. CPU utilization is at 98%, write locks are queuing up, and vertical scaling limits have been reached.
* **Requirements:** Shard the ledger database horizontally to support infinite write scaling.
* **Constraints:** Must preserve transactional consistency for single-account transfers, eliminate cross-shard joins, and handle resharding (adding new shards) with absolutely **zero downtime**.

---

### 2. High-Impact Architectural Solution

#### Database Schema & Routing Strategy
1. **Choose Sharding Key:** `account_id` (not `transaction_id` or `timestamp`).
   - *Why:* High cardinality ensures uniform key distribution. Most ledger queries are account-centric (e.g., fetch statement for `account_id`), allowing the router to target a single physical shard instead of performing slow scatter-gather queries across the entire network.
2. **Handle Cross-Account Transfers:** 
   - A transfer between `account_A` (Shard 1) and `account_B` (Shard 2) cannot run on a single local transaction. We utilize the **Saga Pattern** or **Transactional Outbox with CDC (Change Data Capture)** to guarantee eventual consistency without using slow distributed two-phase commit (2PC) locks.

#### Data Migration Ring for Zero-Downtime Resharding
To scale from 4 physical shards to 8 physical shards without taking the platform offline, we employ a **Consistent Hashing Ring** with a **Dual-Write, Log-Replay Migration Pipeline**:

```
                  Consistent Hashing Ring
                         [Shard 1]
                        /         \
                 [New Shard 5]   [Shard 2]
                     │               │
                     └─ Dual-Writes ─┘
```

1. **Step 1: Expand Topology:** Map the new shard nodes (`Shard 5`, `Shard 6`) onto the consistent hashing ring. Only keys mapped between the new node and its counter-clockwise neighbor are targeted for movement (exactly $1/N$ of keys).
2. **Step 2: Dual-Writing:** Update the routing proxies (e.g., Vitess or custom router) to write new entries to *both* the old shard and the new shard simultaneously for the targeted key space. Reads still point to the old shard.
3. **Step 3: Historical Backfill:** Stream historical WAL logs from the old shard to the new shard using CDC tools (like Debezium) up to the dual-write checkpoint.
4. **Step 4: Cutover:** Once replication lag drops to zero, swap the routing proxy reads to the new shard and terminate the dual-write to the old shard, completing the migration with zero downtime and zero data loss.

---

### 3. Interview Q&A Script

**Interviewer:** *"How do you handle a transfer between two users whose accounts reside on completely different physical shards without locking up the database?"*

**Your Verbal Response:**
> "I explicitly reject distributed two-phase commit (2PC) protocols because they introduce blocking coordinators, increase network round-trip overhead, and drastically lower write availability under load. Instead, I enforce eventual consistency using an **asynchronous Saga pattern mediated by a Transactional Outbox**.
> 
> When Account A on Shard 1 transfers money to Account B on Shard 2, Shard 1 executes a local ACID transaction. This transaction does two things: it debits Account A and writes an `AccountDebited` event into a local `outbox` table in the exact same database transaction.
> 
> A Change Data Capture (CDC) daemon (like Debezium) continuously tail-reads Shard 1's WAL log and publishes the event to an Apache Kafka partition keyed by `AccountB`. Shard 2's consumer group reads the message, executes a local transaction to credit Account B, and returns an acknowledgment. This isolates physical database locks to single-shard transactions, keeping the write pipeline non-blocking and highly available."

---

## Scenario 3: Hardening public Single-Sign-On Auth against Replay Attacks

### 1. Problem Statement
You are architecting the Single-Sign-On (SSO) login flow for a multi-tenant client fleet containing a public web SPA (React 19) and a native mobile application.
* **The Vulnerability:** Public clients cannot securely store a `client_secret` because client-side source code is completely visible to attackers. Standard OAuth2 Authorization Code flow is vulnerable to **Authorization Code Interception Attacks** (where malware on the device intercepts the code from the redirect URL and exchanges it for a token).
* **The Storage Risk:** If the Refresh Token is leaked from browser memory or local storage, an attacker can replay the token to generate new access tokens indefinitely, compromising user accounts.

---

### 2. High-Impact Architectural Solution

#### Secure Authentication Flow: OAuth2 PKCE Handshake
To protect public clients, we mandate **Proof Key for Code Exchange (PKCE)**. This enforces a dynamic, single-use cryptographic challenge that prevents intercepted codes from being exchanged by malicious actors:

```
[Public Client]                                      [Authorization Server]
       │                                                        │
       │─── 1. Generate Verifier & Challenge (SHA256) ─────────►│
       │─── 2. Redirect with Challenge ────────────────────────►│
       │                                                        │
       │◄── 3. Return Authorization Code (Malware Intercepts) ──│
       │                                                        │
       │─── 4. Send Code + Original Code Verifier ─────────────►│
       │                                                        │ (Server hashes Verifier & matches Challenge)
       │◄── 5. Issue Access & Cryptographic Refresh Token ──────│
```

#### Code Defense: Refresh Token Rotation (RTR) & Reuse Detection
The authorization server enforces **Refresh Token Rotation (RTR)**. Every time a client exchanges a refresh token for a new access token, the old refresh token is instantly invalidated, and a **brand new** refresh token is returned to the client in a secure, `HttpOnly`, `SameSite=Strict`, custom path-scoped cookie.

```go
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"errors"
	"fmt"
	"sync"
)

type TokenFamily struct {
	ActiveTokenHash string
	IsRevoked       bool
}

type TokenRegistry struct {
	mu       sync.Mutex
	families map[string]*TokenFamily // Key: Refresh Token Family ID
}

func (r *TokenRegistry) RotateToken(familyID, incomingToken string) (string, error) {
	r.mu.Lock()
	defer r.mu.Unlock()

	family, exists := r.families[familyID]
	if !exists {
		return "", errors.New("invalid_family")
	}

	incomingHash := hashSHA256(incomingToken)

	// Detect Token Reuse (Malicious Replay Attempt)
	if family.IsRevoked || family.ActiveTokenHash != incomingHash {
		// ALARM: This token was already used! Revoke the entire family instantly.
		family.IsRevoked = true
		return "", errors.New("security_alert_token_reuse_detected_session_terminated")
	}

	// Token is valid. Rotate: generate new token, update registry
	newToken := "new-secure-token-" + hex.EncodeToString([]byte(incomingToken[0:4]))
	family.ActiveTokenHash = hashSHA256(newToken)

	return newToken, nil
}

func hashSHA256(data string) string {
	h := sha256.Sum256([]byte(data))
	return hex.EncodeToString(h[:])
}
```

---

### 3. Interview Q&A Script

**Interviewer:** *"If an attacker successfully steals a Refresh Token from local storage, how does your system detect this and protect the user?"*

**Your Verbal Response:**
> "I enforce **Refresh Token Rotation (RTR) with Cryptographic Reuse Detection** combined with secure browser-storage isolation. 
> 
> First, I isolate tokens from local storage. Access tokens stay in memory, while Refresh Tokens are stored strictly in `HttpOnly`, `SameSite=Strict`, `Secure` cookies scoped to `/api/v1/auth`, blocking all Cross-Site Scripting (XSS) retrieval vectors.
> 
> Second, if an attacker bypasses this (e.g., via physical device access) and steals a Refresh Token: when the client attempts to rotate the token, the server returns a new token and invalidates the old one. If the client or the attacker attempts to reuse that *old* token a second time, the authorization server's reuse-detection engine flags it instantly. Because a valid client and an attacker cannot both use the same token without one of them triggering a duplicate request, the reuse event triggers an immediate **cascade revocation**—all active sessions descended from that login family are instantly terminated, forcing an immediate re-authentication across all devices."

---

## Scenario 4: Microfrontends Governance & Runtime Isolation

### 1. Problem Statement
Your organization is building a massive enterprise platform managed by five independent engineering teams. The application must be split into five autonomous microfrontends integrated at runtime via Webpack/Vite Module Federation.
* **The Issue:** Team A's dependencies (using React 17) conflict with Team B's dependencies (using React 18).
* **The Styling Risk:** CSS rules and global variables bleed across microfrontends, resulting in layout breakage and race conditions on the global `window` object.
* **The Registry Risk:** Two microfrontends attempt to register the same custom element on the browser's global custom registry, throwing fatal uncaught exceptions.

---

### 2. High-Impact Architectural Solution

#### Encapsulated Microfrontend Lifecycle Architecture
```
┌────────────────────────────────────────────────────────┐
│                   Main Host Shell                      │
│  - Loads Remote Entries                                │
│  - Initializes Sandbox Proxies                         │
└───────────┬────────────────────────────────┬───────────┘
            │                                │
            ▼                                ▼
┌───────────────────────┐        ┌───────────────────────┐
│  MFE-A (Shadow DOM)   │        │  MFE-B (Shadow DOM)   │
│  - Scoped CSS Module  │        │  - Scoped CSS Module  │
│  - Sandbox Proxy A    │        │  - Sandbox Proxy B    │
└───────────────────────┘        └───────────────────────┘
```

1. **Dependency Duplication & Resolution:** Configure the Module Federation plugin to define frameworks as **shared singletons** with `strictVersion: true` and semantic version ranges. If versions are incompatible (e.g., major versions 17 vs 18), the runtime safely falls back to downloading separate chunks for each module, keeping execution decoupled.
2. **Style Bleed Prevention:** Render all remote components inside an open **Shadow DOM root** (`attachShadow({ mode: "open" })`). This constructs a strong browser-native boundary, ensuring remote styles do not bleed into the shell or sister components.
3. **Registry Isolation:** Utilize **Scoped Custom Element Registries** to prevent registration name collisions, allowing Team A and Team B to define their own isolated component registries.

---

### 3. Interview Q&A Script

**Interviewer:** *"When scaling to multiple independent teams, how do you prevent one buggy microfrontend from crashing the entire browser application?"*

**Your Verbal Response:**
> "I implement a **Fail-Closed Isolation Architecture** combining sandboxed execution contexts, declarative dependency singletons, and visual error boundaries.
> 
> To isolate styling and global scripts, each microfrontend component is wrapped in an open **Shadow DOM** and executed inside a **Proxy-based window sandbox**. The proxy captures global modifications (like modifying `window.location` or setting global variables) and scopes them strictly to a virtual state object dedicated to that microfrontend, leaving the true global window clean.
> 
> To handle runtime script crashes, the host shell wraps all dynamic remote imports within React **Error Boundaries** or Svelte error recovery blocks. If Microfrontend A throws a fatal uncaught JavaScript exception, the error boundary catches it, displays a localized, elegant 'Component Temporarily Unavailable' fallback panel, and logs the crash telemetry, while keeping the rest of the application fully functional."
