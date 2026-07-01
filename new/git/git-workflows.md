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
* **Main Branches:**
  * `main` (Production): Stores the official release history. Every commit here represents a production deployment.
  * `develop` (Integration): Acts as the integration branch for completed features.
* **Supporting Branches (Lifetimes):**
  * `feature/*`: Branched off of `develop`, merged back into `develop`. Exists only during active feature implementation.
  * `release/*`: Branched off of `develop`, merged into `main` and back into `develop` once complete. Created for final release staging, testing, and documentation changes.
  * `hotfix/*`: Branched directly off of `main`, merged into `main` (with release tags) and back into `develop` (or active release branch). Used for immediate production bug fixes.
* **Drawback:** Massive configuration overhead. Large merge conflicts are common due to branches living separately for weeks.

### 2. Trunk-Based Development (The Agile Champion)
* **How it works:** All developers merge their code changes directly into the single main branch (the "Trunk") multiple times a day. Feature branches are extremely short-lived (typically less than 24 hours).
* **Guarantees:** Relies heavily on **Feature Flags** (toggles) to hide unfinished code in production, allowing developers to safely merge incomplete features to the trunk without breaking the system.

---

## 3. Deep Dive: Git Conflicts & Conflict Resolution

A **Merge Conflict** occurs when two developers modify the exact same lines of the exact same file on different branches, or when one developer deletes a file while another developer is modifying it, and Git cannot automatically determine which version takes precedence.

```
       Git Conflict on DAG (Directed Acyclic Graph)
       
              [Commit C1 (Common Ancestor)]
                      /            \
                     /              \
       [Commit C2 (Local)]       [Commit C3 (Incoming)]
        Added: x = "local"        Added: x = "incoming"
                     \              /
                      ▼            ▼
             [ !!! MERGE CONFLICT !!! ]
```

### A. Anatomy of Conflict Markers
When Git detects a conflict during a merge, rebase, or cherry-pick, it stops execution and writes visual conflict markers directly into the affected source files:

```
<<<<<<< HEAD
const databaseUrl = process.env.LOCAL_DB_URL;
=======
const databaseUrl = process.env.PRODUCTION_DB_URL;
>>>>>>> feature/prod-deployment
```

* **`<<<<<<< HEAD` (Local Changes):** Denotes the start of the conflict block. The code below it represents changes on your active, current branch (where HEAD is pointing).
* **`=======` (The Boundary Line):** The middle divider separating the two competing sets of changes.
* **`>>>>>>> feature/prod-deployment` (Incoming Changes):** Denotes the end of the conflict block. The code above it represents the incoming changes from the source branch being merged.

### B. Step-by-Step CLI Resolution Procedure

To resolve a conflict manually on the command line:

1. **Locate conflicted files:** Run `git status` to identify files marked as `both modified`.
2. **Open and inspect:** Open the files in an editor, locate the `<<<<<<<`, `=======`, and `>>>>>>>` markers, and analyze the competing lines.
3. **Edit and choose:** Delete the markers entirely and write the final merged code. You can choose to preserve your local changes, apply the incoming changes, or synthesize a completely new combination of both.
4. **Stage and save:** Add the resolved files to the Git staging index:
   ```bash
   git add <resolved-file-path>
   ```
5. **Finalize the transition:**
   * If resolving a **merge conflict**: Run `git commit` to complete the merge commit.
   * If resolving a **rebase conflict**: Run `git rebase --continue`.
   * If you get stuck or corrupt the state: Run `git merge --abort` or `git rebase --abort` to return instantly to the pre-merge state.

---

## 4. Popular Interview Questions & High-Impact Answers

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

### Q4: Under the hood, how does Git detect a merge conflict? What algorithm does it use?
* **Answer:** Git utilizes a **three-way merge algorithm**. When you merge Branch B into Branch A, Git looks at three commits on the DAG: the tip of Branch A, the tip of Branch B, and the **Common Ancestor** commit where they originally split. Git compares the diffs of both Branch A and Branch B against the Common Ancestor. If a line was modified in Branch A but remained unchanged in Branch B, Git auto-applies the change. However, if the exact same line was modified differently in *both* Branch A and Branch B relative to the ancestor, Git halts, raising a **merge conflict** because there is no clean logical precedence.

### Q5: Describe the lifecycle of branches in Git Flow. How does it compare to Trunk-Based Development's branches?
* **Answer:** In **Git Flow**, branches have distinct, long lifecycles: `main` and `develop` are infinite branches that never die. Supporting branches like `feature/*` can live for weeks before merging back. `release/*` branches isolate QA processes, and `hotfix/*` branches quickly bypass staging. In **Trunk-Based Development**, branches are short-lived. Developers pull short-lived branches off `main` and merge them back to the trunk within hours (typically $< 24$ hours). This forces continuous code integration and eliminates the extensive merge coordination common to Git Flow.

### Q6: What is the difference between `git merge --ours` and checking out a file with `git checkout --ours` during a conflict?
* **Answer:** They operate at completely different scopes:
  * **`git merge -s ours` (or `git merge --ours`)** is a merge strategy that operates across the entire branch merge. It completes the merge, creating a merge commit, but completely ignores all incoming changes from the source branch, keeping the files of your current branch exactly as they were.
  * **`git checkout --ours <file-path>`** operates at the individual **file scope** during an active, paused merge conflict. It tells Git to overwrite the conflict in that specific file by selecting your current branch's local version, discarding only that file's incoming changes, while leaving other conflicted files to be resolved manually.

