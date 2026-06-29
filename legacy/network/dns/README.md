# DNS (Domain Name System)

Comprehensive interview study guide covering the Domain Name System, resolution flow, record types, caching, and TTL mechanics.

---

## 1. Meaning & The Global Lookup

The **Domain Name System (DNS)** is the hierarchical, decentralized database that translates human-readable domain names (like `google.com`) into computer-readable IP addresses (like `142.250.190.46`).

---

## 2. Hierarchical DNS Resolution Flow

When you type `blog.example.com` into your browser for the first time:

```
Browser ──► 1. Recursive Resolver ──► 2. Root Servers (".")
                    ▲                     │
                    │ (Returns TLD IP)    ▼
                    │◄────────────────────┘
                    │──► 3. TLD Nameservers (".com")
                    ▲                     │
                    │ (Returns Auth IP)   ▼
                    │◄────────────────────┘
                    │──► 4. Authoritative Nameservers ("example.com")
                    ▲                     │
                    │ (Returns IP)        ▼
                    │◄────────────────────┘
```

1. **Browser / OS Cache:** Checks local host files and resolver cache first.
2. **Recursive Resolver (e.g., Google 8.8.8.8, Cloudflare 1.1.1.1):** Queries upstream hierarchy on the client's behalf.
3. **Root Servers (`.`):** Directs the resolver to the appropriate Top-Level Domain (TLD) server (e.g., `.com`, `.org`, `.vn`).
4. **TLD Nameserver:** Directs the resolver to the Authoritative Nameserver of the specific domain.
5. **Authoritative Nameserver:** Holds the official DNS records. It returns the exact target IP address (e.g., `A` record) to the resolver, which caches it and returns it to your browser.

---

## 3. Critical DNS Record Types

* **`A` Record:** Maps a hostname directly to an **IPv4 address** (e.g., `example.com` ──► `93.184.216.34`).
* **`AAAA` Record:** Maps a hostname to an **IPv6 address**.
* **`CNAME` (Canonical Name):** Maps an alias name to another canonical domain name (e.g., `www.example.com` ──► `example.com`).
* **`MX` (Mail Exchanger):** Specifies mail servers responsible for receiving emails on behalf of the domain.
* **`TXT` (Text):** Stores arbitrary text data, heavily used for email security validation (SPF, DKIM, DMARC) and domain ownership proof.
* **`NS` (Name Server):** Specifies the Authoritative Nameservers for the domain.

---

## 4. DNS Caching & TTL (Time To Live)

**TTL (Time To Live)** is a setting (configured in seconds) attached to every DNS record that dictates how long local resolvers, OS, and browsers are allowed to cache that record before querying the Authoritative Nameserver again.

* **High TTL (e.g., 86400s / 24 hours):** Reduces DNS query traffic, improving latency. However, if you change server IPs, users will experience cached routing errors until the TTL expires.
* **Low TTL (e.g., 60s):** Highly dynamic, ideal for fast failover or load balancing (e.g., AWS Route 53 multi-IP routing).

---

## 5. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between a Recursive DNS Query and an Iterative DNS Query?
* **Answer:**
  * **Recursive Query:** The client asks a resolver (e.g., Cloudflare 1.1.1.1) to find the IP. The resolver is legally bound to either return the final IP address or a solid "Not Found" error. The resolver handles all the hard work of traversing the root, TLD, and authoritative servers.
  * **Iterative Query:** The resolver queries upstream servers. Instead of finding the IP itself, an upstream server returns a reference: "I don't know the IP, but ask this TLD server instead." The resolver then queries that TLD server, iterating through references until it hits the authoritative source.

### Q2: Why is a CNAME record discouraged for the root zone (naked domain like `example.com`), and what is the workaround?
* **Answer:** According to DNS RFC specifications, a **CNAME record cannot coexist with other record types** for the same hostname. Since the root zone (`example.com`) *must* contain `NS` and `SOA` records to function, setting a CNAME at the root violates RFCs and can break email or resolver systems. The workaround is using an **`ALIAS` (or ANAME) record**, supported by modern DNS providers (like AWS Route 53 or Cloudflare). An ALIAS acts like a CNAME but resolves the target IP dynamically at the DNS server level, returning a clean `A` record representation to the client.

### Q3: What is "DNS Cache Poisoning" (DNS Spoofing), and how does DNSSEC mitigate it?
* **Answer:** **DNS Cache Poisoning** occurs when an attacker intercepts recursive queries and injects a fake IP address response into a resolver's cache before the real authoritative response arrives. Resolvers will serve this malicious IP to all future clients. **DNSSEC** (Domain Name System Security Extensions) mitigates this by adding digital cryptographic signatures (using public-key cryptography) to every DNS record. Resolvers verify these signatures against public keys chained up to the Root Zone trust anchor, automatically discarding unsigned or spoofed responses.
