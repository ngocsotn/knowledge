# Network Security and TLS/SSL

Comprehensive study guide for understanding network security protocols, cryptographical fundamentals, handshakes, and transport layer security (TLS) in distributed systems.

---

## 1. Cryptography Fundamentals

Secure network transport relies on combining two distinct cryptographic models:

- **Asymmetric Cryptography (Public Key)**:
  - Uses a pair of mathematically linked keys: a **Public Key** (distributed to anyone) and a **Private Key** (kept strictly secret).
  - *Use Case*: Authentication and secure key exchange.
  - *Characteristics*: Secure but extremely CPU-intensive; not suitable for encrypting large streams of data.
- **Symmetric Cryptography (Shared Secret)**:
  - Uses the **same single secret key** for both encryption and decryption (e.g., AES-GCM).
  - *Use Case*: Fast encryption of bulk data streams.
  - *Characteristics*: Highly efficient, near zero overhead.

---

## 2. The TLS Handshake (TLS 1.2 vs. TLS 1.3)

The **Transport Layer Security (TLS)** handshake negotiates cipher suites, authenticates the server, and establishes a secure symmetric session key.

### TLS 1.2 Handshake (2 Round-Trips - 2 RTT)

```
Client                                                   Server
  |                                                        |
  | ─── ClientHello (Ciphers, random number) ────────────> |
  | <── ServerHello (Selected cipher, server random) ───── |
  | <── Server Certificate (Public Key) ────────────────── |
  |                                                        |
  | ─── Client Key Exchange (Pre-master secret) ─────────> |
  | ─── ChangeCipherSpec (Switch to symmetric keys) ──────> |
  | ─── Finished (Symmetric encrypted check) ────────────> |
  |                                                        |
  | <── ChangeCipherSpec ───────────────────────────────── |
  | <── Finished ───────────────────────────────────────── |
  V                                                        V
             [Symmetric Encrypted Session Begins]
```

### TLS 1.3 Handshake (1 Round-Trip - 1 RTT - Optimized)
TLS 1.3 optimizes transport performance by eliminating redundant round-trips. It is a mandatory **1-RTT (Round-Trip Time)** handshake compared to TLS 1.2's **2-RTT** default.

```
Client                                                   Server
  |                                                        |
  | ─── ClientHello ─────────────────────────────────────> | (1-RTT)
  |     (Supported Ciphers + Key Share guess)              |
  |                                                        |
  | <── ServerHello (Cipher choice + Key Share selected) ── |
  | <── Encrypted Extensions & Certificate ────────────── |
  | <── Server Finished ────────────────────────────────── |
  |                                                        |
  | ─── Client Finished (Encrypted Data begins) ─────────> |
  V                                                        V
```

#### How TLS 1.3 achieves 1-RTT:
1. **Key Share Speculation:** Instead of sending a blank ClientHello to ask what key exchange algorithm the server supports, the client aggressively speculates. It sends its list of supported cipher suites AND immediately attaches its **Public Key Share** guesses (e.g., using popular curves like x25519 or secp256r1) directly inside the initial `ClientHello`.
2. **Immediate Session Key Derivation:** The server receives the `ClientHello`. It selects a compatible curve from the client's guesses, combines it with its own generated ephemeral key, and immediately derives the symmetric master key.
3. **Encrypted Parameters:** The server's response (`ServerHello`, Certificate, Signature, and Finished signals) is returned to the client and **encrypted** using this newly derived master key. This significantly minimizes cleartext metadata visible to network sniffers.
4. **Cipher Suite Reduction:** TLS 1.3 reduced the list of permitted cipher suites from dozens of insecure choices in TLS 1.2 to only **5 highly secure AEAD (Authenticated Encryption with Associated Data)** ciphers (e.g., `TLS_AES_256_GCM_SHA384` and `TLS_CHACHA20_POLY1305_SHA256`).

#### Complete Deprecation of Static RSA Key Exchange:
In TLS 1.2, servers could use a static RSA key to decrypt client session keys, which broke **Perfect Forward Secrecy (PFS)**. TLS 1.3 **completely outlaws RSA key transport** and static Diffie-Hellman, making Ephemeral Diffie-Hellman (**ECDHE** or **DHE**) the only permitted key exchange mechanism.

---

### Zero-RTT (0-RTT) Resumption & Replay Attacks
TLS 1.3 introduces **Zero-RTT (0-RTT) Resumption**, allowing a client who has previously connected to a server to transmit application data (such as an HTTP GET request) directly inside the *very first packet* of a reconnecting handshake, eliminating connection latency entirely.

```
Client                                                   Server
  |                                                        |
  | ─── ClientHello + Session Ticket ────────────────────> | (0-RTT!)
  |     + Early Application Data (e.g., GET /index)        |
  V                                                        V
```

#### The Replay Attack Vulnerability:
Because 0-RTT data is sent before a fresh, synchronized handshake is established, it does not have the protection of a unique, newly negotiated key. It relies on a pre-shared key (PSK) generated from the *previous* session.

```
Client               Sniffer/Attacker               Server
  |                         |                          |
  | ─── GET /pay ──────────>| (Intercepts 0-RTT pkt)   |
  |                         |                          |
  |                         | ─── Replayed GET /pay ──>| (Executes payment)
  |                         |                          |
  |                         | ─── Replayed GET /pay ──>| (Executes payment AGAIN!)
```

1. **Interception:** An attacker on the local network sniffs the client's initial 0-RTT packet (containing the ClientHello, Session Ticket, and the encrypted data `POST /api/v1/checkout`).
2. **Replay:** The attacker replicates this exact packet and forwards it to the server. Because the packet's cryptographic parameters are completely valid, the server accepts it, decrypts it, and executes the checkout transaction a second time.
3. **Result:** The attacker can trigger multiple identical transactions or state mutations on the server without needing to crack the encryption.

#### Engineering Mitigations against 0-RTT Replays:
To safely run 0-RTT in production, you must implement the following safeguards:
* **Enforce Strict Idempotency:** Load balancers and web servers (like Nginx) must be configured to **only allow 0-RTT on safe, idempotent HTTP methods (GET/HEAD)**. Any state-changing methods (`POST`, `PUT`, `DELETE`, `PATCH`) must be rejected if they arrive via 0-RTT and forced to undergo a full 1-RTT handshake.
* **Single-Use Session Tickets (Anti-Replay Cache):** The server maintains a distributed, low-latency cache (like Redis) tracking the unique serial numbers of used Session Tickets. If a ticket is presented a second time within its expiration window, it is instantly rejected.
* **Timestamp Windowing:** The server compares the timestamp embedded inside the session ticket with the current server time. If the drift exceeds a tight window (e.g., 2-5 seconds), the 0-RTT data is discarded.


---

## 3. Mutual TLS (mTLS)

Standard TLS (like visiting `https://google.com`) is one-way: the client authenticates the server, but the server does not know who the client is.
In **mTLS (Mutual TLS)**, both parties authenticate each other's certificates concurrently.

```
Client                                                   Server
  |                                                        |
  | ─── ClientHello ─────────────────────────────────────> |
  | <── ServerHello + Server Certificate ───────────────── |
  | <── CertificateRequest (Asks client for cert) ──────── |
  |                                                        |
  | ─── Client Certificate ──────────────────────────────> |
  | ─── CertificateVerify (Signed hash proving owner) ───> |
  V                                                        V
```

### Why mTLS is essential for Microservices:
Inside a secure service mesh (e.g., Istio, Linkerd), mTLS guarantees:
- **Authentication**: Service A knows for certain that Service B is who they claim to be.
- **Encryption**: All inter-service traffic is secure against sniffing and MITM (Man-in-the-Middle) attacks.
- **Authorization/Access Control**: Combined with RBAC (Role-Based Access Control), we can declare that only authenticated certs from Service A are allowed to write to Service B.

---

## 4. Digital Certificates and PKI

- **PKI (Public Key Infrastructure)**: The system of hardware, software, policies, and certificates that bind public keys to identities (such as websites or services).
- **Certificate Authority (CA)**: Trusted third-party institutions (e.g., Let's Encrypt) that issue digital certificates.
- **Chain of Trust**:
  - The browser/OS pre-installs a list of trusted **Root Certificates** (Root CAs).
  - Websites use **Intermediate Certificates** signed by Root CAs.
  - The client verifies a website's leaf certificate by validating each signature up the chain to the pre-installed trusted root.

---

## 5. Hard Interview Questions & Deep Answers

### Q1: What is the "Diffie-Hellman Key Exchange" (DHKE), and how does it enable PFS (Perfect Forward Secrecy)?
**Answer**:
- **Diffie-Hellman**: A cryptographic algorithm that allows two parties to establish a shared secret over an insecure channel without actually sending the secret itself over the wire. They exchange public parameters, combine them with their private numbers, and mathematically arrive at the exact same shared key.
- **Perfect Forward Secrecy (PFS)**:
  - If a hacker records all encrypted traffic of a server for years, and then subsequently steals the server's private RSA key, they **cannot** decrypt the historical traffic.
  - **How DHKE achieves this**: For each connection, the server and client generate a unique, temporary (ephemeral) Diffie-Hellman key pair (`DHE` or `ECDHE`). Once the session ends, these keys are permanently deleted from memory. Since the server's master private key is only used to *sign* the exchange (authentication), not to *encrypt* it, compromising the master key provides zero access to historical session keys.

### Q2: What is the difference between asymmetric decryption and checking a digital signature?
**Answer**:
They are inverse operations of the same mathematical foundation:
- **Asymmetric Decryption**:
  - Anyone can use your **Public Key** to encrypt a message. Only you (the owner) can use your **Private Key** to decrypt and read it.
  - *Goal*: Confidentiality.
- **Digital Signature**:
  - You (the owner) encrypt a hash of a document using your **Private Key** (this is the signature).
  - Anyone can use your **Public Key** to decrypt the signature and compare the resulting hash to the document. If they match, it proves that the signature was created by you and that the document has not been altered.
  - *Goal*: Integrity and Non-repudiation.

### Q3: How do you implement secure client-certificate revocation checks in high-performance mTLS systems? Compare CRL vs. OCSP.
**Answer**:
When a client certificate is compromised (e.g., a laptop is stolen), it must be invalidated before its expiration date.
- **CRL (Certificate Revocation List)**:
  - The server regularly downloads a complete list of revoked certificate serial numbers from the CA.
  - *Trade-off*: High database storage overhead. If the CRL file is large (megabytes), it degrades connection setup performance. Also, there is lag between revocation and CRL updates.
- **OCSP (Online Certificate Status Protocol)**:
  - During the handshake, the server queries the CA's OCSP responder API in real-time to check the status of the client's cert.
  - *Trade-off*: Adds network latency to every handshake. If the OCSP responder goes down, the server must choose to either "soft-fail" (allow insecure access) or "hard-fail" (block valid clients).
- **OCSP Stapling (Optimization - for server certs)**:
  - The server queries the OCSP responder itself every hour, gets a signed and timestamped proof of validity, and "staples" this proof directly inside the initial handshake to the client. This offloads OCSP network round-trips from the client.

### Q4: [Asymmetric Key Payload Trade-off] Why is asymmetric cryptography not used to encrypt the actual application data payload in network protocols like HTTP/2/3, and how is it used during the handshake to solve the key exchange problem?
**Answer**:
* **Payload Encryption**: Asymmetric encryption (like RSA-4096 or ECC) relies on extremely complex modular exponentiation or elliptic curve point multiplication. This is computationally expensive and slow—about 100 to 1,000 times slower than symmetric block ciphers like AES-GCM or ChaCha20-Poly1305, which run in hardware-accelerated CPU instructions (AES-NI).
* **Handshake Phase**: Asymmetric encryption is used *only* during the initial handshake to safely authenticate the identities of the nodes and securely negotiate a shared secret (session key) over an untrusted public channel.
* **Hybrid Cryptography**: Once the handshake completes and both parties agree on a symmetric session key, the asymmetric keys are put to sleep, and the entire high-throughput application data payload is encrypted using the fast symmetric key.

### Q5: [Asymmetric Key Security Struggle] How does Ephemeral Diffie-Hellman (ECDHE) with digital signatures provide Forward Secrecy, and what happens if an attacker steals the server's private asymmetric key years after recording encrypted network traffic?
**Answer**:
* **Traditional Key Exchange (Non-Forward Secrecy)**: In older RSA handshakes, the client encrypts a symmetric key using the server's public key and sends it to the server. If an attacker records all encrypted traffic over several years, and then later steals the server's private key, they can decrypt the recorded handshake, retrieve the symmetric key, and decrypt *all historical traffic*.
* **Forward Secrecy (ECDHE)**: Under Ephemeral Diffie-Hellman, the server and client generate a *random, throwaway (ephemeral)* key pair for *each individual session*. They perform a Diffie-Hellman key exchange, compute a shared secret, and immediately destroy the ephemeral keys after the session.
* **The Asymmetric Role**: The server's persistent asymmetric private key is used *only* to digitally sign the ephemeral DH parameters, proving authenticity and preventing Man-In-The-Middle (MITM) attacks.
* **The Compromise Outcome**: If the server's private key is stolen years later, the attacker *cannot* decrypt historical traffic because the shared symmetric keys were never sent across the wire, nor were they derived from the persistent private key. The attacker can only use the stolen key to impersonate the server in *future* handshakes.

