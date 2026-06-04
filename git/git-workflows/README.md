# Git Workflows: Git Flow vs. Trunk-Based Development

Comprehensive interview study guide covering industry-standard Git branch workflows, release management, and development practices.

---

## 1. Branching Workflows Comparison

| Attribute | Git Flow | GitHub Flow | Trunk-Based Development (TBD) |
| :--- | :--- | :--- | :--- |
| **Main Branches** | `main` and `develop` | `main` | `main` (the "Trunk") |
| **Supporting Branches**| `feature/*`, `release/*`, `hotfix/*` | Short-lived feature branches | Short-lived feature branches (hours) |
| **Merge Frequency** | Slow (weeks/months) | Daily | **Multiple times a day** |
| **Release Frequency** | Long cycles (scheduled) | Continuous | Continuous (often automated via CD) |
| **Best-Fit Cases** | Enterprise software, legacy release cycles, medical/safety devices. | Standard SaaS web applications. | High-velocity tech companies, fast-paced CI/CD teams. |

---

## 2. Deep Dive: Key Workflows

### 1. Git Flow (The Heavyweight)
* **How it works:** Highly structured. Features are branched off and merged into `develop`. When a release is ready, a `release/` branch is cut from `develop` for final QA and bug-fixing before merging into `main` (for tag releases) AND merging back into `develop`. Emergency production fixes use separate `hotfix/` branches branched directly from `main`.
* **Drawback:** Massive configuration overhead. Large merge conflicts are common due to branches living separately for weeks.

### 2. Trunk-Based Development (The Agile Champion)
* **How it works:** All developers merge their code changes directly into the single main branch (the "Trunk") multiple times a day. Feature branches are extremely short-lived (typically less than 24 hours).
* **Guarantees:** Relies heavily on **Feature Flags** (toggles) to hide unfinished code in production, allowing developers to safely merge incomplete features to the trunk without breaking the system.

---

## 3. Popular Interview Questions & High-Impact Answers

### Q1: When would you recommend Trunk-Based Development over Git Flow, and what infrastructure is required to support it?
* **Answer:** **Trunk-Based Development** is highly recommended for high-velocity teams aiming to achieve true Continuous Integration and Continuous Deployment (CI/CD). It eliminates long-lived branches, reducing merge conflicts to a minimum.
* **Requirements:** To use Trunk-Based Development safely, your team must have:
  1. **Robust CI Pipeline:** Fast, automated tests must run on every commit to instantly block broken code from entering the trunk.
  2. **Feature Flags:** The capability to hide unfinished code in production dynamically.
  3. **Fast Rollbacks:** The ability to revert commits or toggle flags in seconds if a bug leaks.

### Q2: What is a "Feature Flag" (Feature Toggle), and how does it enable Trunk-Based Development?
* **Answer:** A **Feature Flag** is an application-level conditional statement (e.g., `if (features.isEnabled("new_payment_ui")) { ... } else { ... }`) that reads configuration values dynamically from memory, database, or external flag services (like LaunchDarkly). It decouples code deployment from feature release. Developers can safely deploy half-finished features to the production main branch with the flag set to `false`. Once the feature is finished and tested, product managers can flip the flag to `true` instantly without redeploying the container.

### Q3: What is the significance of the `git cherry-pick` command?
* **Answer:** **`git cherry-pick <commit-hash>`** is a command used to apply a specific, individual commit from an arbitrary branch onto your current active branch. It works by copying the diff of that commit and creating a brand new commit on top of your current HEAD. It is highly useful in hotfix scenarios: if a bug fix was already committed to a development branch, and you need to deploy *only* that fix to production immediately without bringing along unrelated development features, you cherry-pick that single commit onto the production release branch.
