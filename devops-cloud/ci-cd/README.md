# Continuous Integration & Delivery (CI/CD)

Comprehensive interview study guide covering CI/CD pipelines, automation flows, validation gates, and advanced deployment strategies.

---

## 1. Meaning of CI/CD

* **CI (Continuous Integration):** A development practice where developers merge their code changes back into the main branch frequently. Every merge triggers an automated pipeline that builds and tests the code to detect integration bugs early.
* **CD (Continuous Delivery):** The practice of automating code releases. Every successfully validated build is packaged and automatically pushed to a staging environment, ready for manual one-click deployment to production.
* **CD (Continuous Deployment):** One step further than delivery. Every change that passes all automated pipeline gates is **automatically deployed directly to production** without human intervention.

---

## 2. Pipelines Phases & Quality Gates

A standard CI/CD pipeline consists of six major phases:

```
Merge PR ──► 1. LINT ──► 2. TEST ──► 3. BUILD ──► 4. SCAN ──► 5. DEPLOY
```

1. **Linting & Formatting:** Checks syntax consistency and code style guidelines (e.g., ESLint, Go vet) before compiling.
2. **Automated Testing:** Runs unit tests, integration tests, and coverage checks.
3. **Build & Package:** Compiles source code, bundles assets, and builds immutable artifacts (like Docker images).
4. **Security Scanning (SAST/DAST):** Scans dependencies for known vulnerabilities (Snyk) and checks code for hardcoded secrets or CVEs.
5. **Release & Push:** Signs artifacts and uploads them to registries (e.g., Docker Hub, AWS ECR).
6. **Deployment:** Provisions infrastructure and rolls out the new version to target environments.

---

## 3. High-Impact Deployment Strategies

To minimize risk and prevent downtime during production rollouts, teams utilize advanced deployment patterns:

### 1. Rolling Update
* **Mechanism:** Progressively replaces old instances of the application with new ones.
* **Pros/Cons:** Zero downtime, low resource usage. However, it can take a long time to complete at scale, and the system must tolerate running two different versions of the code simultaneously.

### 2. Blue-Green Deployment
* **Mechanism:** Maintains two identical production environments: "Blue" (active) and "Green" (inactive). You deploy the new version to Green, run post-deployment tests, and swap the router/load-balancer target instantly to point to Green.
* **Pros/Cons:** Zero downtime, instant rollback (swap router back to Blue if bugs are found). However, it requires double the infrastructure cost.

### 3. Canary Deployment
* **Mechanism:** Deploys the new version to a small subset of servers (e.g., 5% of traffic) and routes real user requests to it. You monitor error rates and performance. If stable, you roll out the change to 100% of servers.
* **Pros/Cons:** Isolates damage to a small user group. However, routing logic is complex.

---

## 4. Popular Interview Questions & High-Impact Answers

### Q1: What is the difference between Continuous Delivery and Continuous Deployment?
* **Answer:**
  * In **Continuous Delivery**, the pipeline validates and packages code successfully, but the final release to the production server requires a **manual human approval/trigger** (e.g., clicking a button in GitHub Actions or Jenkins).
  * In **Continuous Deployment**, there are **no manual gates**. Every code change that passes all linting, unit, integration, security, and build checks is automatically deployed directly to the live production server within minutes of merging.

### Q2: What is the purpose of SAST (Static Application Security Testing) in a CI/CD pipeline?
* **Answer:** **SAST** analyzers scan raw source code, configurations, and dependency trees at rest before compilation to locate known security vulnerabilities (like SQL Injection paths, unsafe deserialization, or outdated libraries containing CVEs) and hardcoded credentials/secrets. Running SAST in the pipeline prevents security regressions from reaching production, acting as an automated compliance and security gate.

### Q3: What is "Configuration Drift" and how do you prevent it in infrastructure deployments?
* **Answer:** **Configuration Drift** occurs when manual hotfixes or configurations are made directly on production servers (e.g., modifying firewall rules or updating files via SSH) without updating the primary CI/CD pipeline or configuration scripts. This leads to discrepancies between the staging and production environments, causing future automated deployments to fail mysteriously. It is prevented by using **IaC (Infrastructure as Code)** tools like Terraform or Ansible and enforcing strict "no-SSH" write policies on production.
