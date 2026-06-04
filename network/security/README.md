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
TLS 1.3 eliminates the secondary key exchange round-trip by sending key-share guesses along with the initial client hello.
1. **ClientHello**: Sends cipher choices AND a public key share guess (Diffie-Hellman Key Exchange parameters).
2. **ServerHello**: Returns the selected cipher, server's public key share, server certificate, and signature. Both client and server calculate the symmetric session key instantly.
3. **Finished**: Server sends encrypted Finished signal. Client returns Finished.

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

