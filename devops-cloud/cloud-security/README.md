# Cloud Security and Infrastructure Hardening

Comprehensive DevOps & Cloud study guide covering secure architecture design, Identity & Access Management, network segmentation, and threat modeling in modern cloud environments.

---

## 1. Identity & Access Management (IAM)

IAM governs authentication and authorization of human users and automated software workloads inside your cloud provider (e.g., AWS, GCP, Azure).

### Core Concepts & Best Practices:
- **Principle of Least Privilege (PoLP)**: Give users and services *only* the absolute minimum permissions required to perform their tasks. Never use wildcard `*` permissions in production policy documents.
- **IAM Roles vs. Users**:
  - **IAM Users**: Long-lived credentials (Access Key / Secret Key). Highly vulnerable to leaking (e.g., accidentally pushing keys to public GitHub).
  - **IAM Roles**: Used by applications and cloud services (e.g., EC2, Lambda, Kubernetes pods). Roles do not have permanent credentials; they use **temporary, short-lived security tokens** (via Security Token Service - STS) that automatically expire and rotate.

---

## 2. VPC Networking and Security

A **Virtual Private Cloud (VPC)** isolates your infrastructure network from other tenants in the cloud.

```
+-------------------------------------------------------------------------+
| VPC                                                                     |
|  +---------------------------+       +-------------------------------+  |
|  | Public Subnet (Allows IP) |       | Private Subnet (Strict Secure)|  |
|  |                           |       |                               |  |
|  |  [Nginx Load Balancer] ───┼──────>│  [API App Server]             |  |
|  |  [NAT Gateway] <──────────┼───────┼─── (Accesses third-party APIs)|  |
|  +---------------------------+       +-------------------------------+  |
+-------------------------------------------------------------------------+
```

### A. Public vs. Private Subnets
- **Public Subnet**: Direct route to the Internet Gateway. Contains load balancers or bastion hosts that need public IP addresses.
- **Private Subnet**: No direct route from the internet. Contains database servers, cache nodes, and application containers. They can access the internet (e.g., to fetch third-party packages or updates) *only* through a **NAT Gateway** located in the public subnet.

### B. Security Groups vs. Network Access Control Lists (NACLs)

| Dimension | Security Groups | NACLs |
| :--- | :--- | :--- |
| **Working Layer**| Instance Level (computes, DBs) | Subnet Level (entire subnet boundary) |
| **Statefulness** | **Stateful**: If you allow incoming traffic, outgoing response is automatically allowed. | **Stateless**: You must explicitly configure both inbound and outbound traffic rules. |
| **Rule Order** | Evaluates all rules before deciding. | Evaluates rules in strict sequential order (by rule number). |
| **Supported Rules** | "Allow" rules only. | Supports both "Allow" and "Deny" rules. |

### C. VPC Endpoints (PrivateLink)
- **Problem**: When a private subnet container needs to write to an S3 bucket or call AWS Secrets Manager, standard routing sends that traffic out through the NAT Gateway onto the public internet to reach the service API.
- **Solution**: **VPC Endpoints** provision private network interfaces directly inside your private subnet. Traffic to S3 or Secrets Manager flows strictly over AWS's private fiber network, bypassing the public internet entirely.

---

## 3. Secrets Management and Leakage Protection

Storing secrets (API keys, database passwords, private keys) safely is a critical security gate.

### Environment Variables vs. Encrypted Files
- **Accidental Environment Leakage**: While injecting secrets via environment variables (`process.env.DB_PASSWORD`) is common, it is **operationally insecure** because:
  1. Any subprocess or library spawned by your app can read all env variables.
  2. Crash dumps, stack traces, and log exporters frequently write out the entire environment state, exposing secrets to third-party logs.
- **FileSystem Injection (Best Practice)**: Mount secrets as temporary, encrypted, in-memory files (e.g., Kubernetes Secrets mounted on `/run/secrets/` or tmpfs). Applications read the secret directly from the file and never expose it in logs or env vars.

---

## 4. Threat Modeling (STRIDE)

**STRIDE** is a security model developed by Microsoft to identify and mitigate threat categories.

| Threat | Security Property | Mitigation Strategy |
| :--- | :--- | :--- |
| **S**poofing | Authentication | Implement mTLS, strong passwords, and multi-factor auth (MFA). |
| **T**ampering | Integrity | Secure signatures (HMAC), input validation, file integrity checks. |
| **R**epudiation | Non-repudiability | Comprehensive logging, audit trails, secure digital signatures. |
| **I**nformation Disclosure | Confidentiality | Encryption at rest and in transit (TLS, AES-256), least privilege access. |
| **D**enial of Service | Availability | Rate limiting, CAPTCHAs, redundant resources, load shedding. |
| **E**levation of Privilege | Authorization | Role-Based Access Control (RBAC), strict IAM validation boundaries. |

---

## 5. Hard Interview Questions & Deep Answers

### Q1: What is "SSRF" (Server-Side Request Forgery) in cloud environments, and how does accessing Instance Metadata Service (IMDS) escalate this vulnerability?
**Answer**:
- **SSRF**: An attacker exploits an API endpoint that fetches data from an external URL (e.g., uploading a profile picture via URL). By passing an internal IP address (like `127.0.0.1` or `169.254.169.254`), they trick the backend server into making requests to internal endpoints.
- **IMDS Vulnerability**:
  - Cloud providers run the **Instance Metadata Service** at the link-local IP `169.254.169.254`.
  - Under IMDSv1, anyone on the server can execute `GET http://169.254.169.254/latest/meta-data/iam/security-credentials/` to retrieve the active IAM role's temporary credentials.
  - If SSRF is exploited, the attacker can retrieve these temporary security credentials, assume the server's IAM role, and fully compromise other cloud resources (e.g., reading S3, modifying databases).
- **Mitigation**:
  1. Upgrade to **IMDSv2**, which requires a session token handshake using `PUT` and session-header validation, blocking SSRF.
  2. Implement strict URL whitelisting and network firewalls to block outgoing server traffic to `169.254.169.254` from application containers.

### Q2: How do you design secure Kubernetes pods to enforce runtime container isolation?
**Answer**:
By default, Docker containers run as root and share the host OS kernel, leaving them vulnerable to container breakout exploits.
1. **Run as Non-Root User**: Never run container processes as root. Use `USER 1000` inside the Dockerfile.
2. **Read-Only Root Filesystem**: Configure the container's security context to use a read-only root filesystem (`readOnlyRootFilesystem: true`). This prevents attackers from writing malicious binaries or modifying existing libraries if they execute code injection.
3. **Disable Privilege Escalation**: Set `allowPrivilegeEscalation: false` to block standard `sudo` or SUID escalations.
4. **Use gVisor or Kata Containers (Runtime Isolation)**: For highly sensitive or untrusted code execution workloads, bypass standard container runtime (runc) and use **gVisor** (a secure user-space kernel proxy written in Go) to intercept and sandbox all system calls, preventing host kernel takeover.
