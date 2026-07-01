# Git Internals: Under the Hood of Version Control

Comprehensive staff-level study guide covering Git's internal object storage architecture, Directed Acyclic Graphs (DAG), packfiles compaction, reflog disaster recovery, and the mechanics of three-way merges.

---

## 1. Git is a Content-Addressable Storage (CAS) System

Unlike traditional version control systems (like SVN or CVS) that store file differences (deltas) over time, **Git is a snapshot-based file system**. Under the hood, Git is a simple **key-value database** where keys are 40-character SHA-1 hashes (160-bit) and values are compressed data payloads.

### The SHA-1 Cryptographic Hash Formula
Every object written to the Git database (`.git/objects/`) is hashed using a standardized header prefix:

$$\text{hash} = \text{SHA-1}\left( \text{object\_type} \ + \ \text{" "} \ + \ \text{content\_size\_in\_bytes} \ + \ \text{"\backslash 0"} \ + \ \text{raw\_content} \right)$$

For example, a text file containing exactly "Hello" is written with a header `blob 5\0`:

```bash
$ echo -n "Hello" | git hash-object --stdin -w
f572d396fae9206628714fb2ce00f72e94f225af
```

---

## 2. The Four Git Object Types

The entire Git Directed Acyclic Graph (DAG) is composed of only four primitive object types residing inside the `.git/objects/` directory (sharded by the first two hex characters of their hashes to prevent directory file-limit performance drops):

```
                       ┌─────────────────────────┐
                       │      Commit Object      │
                       │ - tree: a48f12...       │
                       │ - parent: 92fb54...     │
                       │ - author & message      │
                       └────────────┬────────────┘
                                    │
                                    ▼
                       ┌─────────────────────────┐
                       │       Tree Object       │
                       │ - 100644 blob f572d3... │ (maps index.js)
                       │ - 040000 tree b219f2... │ (maps sub-folder)
                       └──────┬────────────┬─────┘
                              │            │
                              ▼            ▼
                 ┌───────────────┐  ┌───────────────┐
                 │  Blob Object  │  │  Tree Object  │
                 │ (Raw content) │  │ (Sub-folder)  │
                 └───────────────┘  └───────────────┘
```

### 1. Blob Objects (Binary Large Objects)
* **What it stores:** Only the raw compressed file bytes.
* **What it lacks:** Zero metadata. A blob does not store the filename, the directory path, creation dates, or file permissions.

### 2. Tree Objects
* **What it stores:** Directory structures. A tree object is a flat list mapping filenames, execution permissions (`100644` for standard files, `100755` for executables, `040000` for subdirectories) to the target blob or sub-tree hashes.

### 3. Commit Objects
* **What it stores:** Metadata pointing to a root `tree` hash representing the project snapshot at that exact second, array references to one or more `parent` commit hashes, the author, the committer, and the commit message.

### 4. Annotated Tag Objects
* **What it stores:** A persistent reference pointing to a specific commit hash, signed with an optional GPG signature and author message.

---

## 3. Packfiles and Delta Compression

When you commit files, Git creates "loose" object files compressed with zlib. If you modify a 10MB file by changing 1 character, Git creates a completely brand-new 10MB loose blob file, which quickly balloons repository disk usage.

### Compaction via `git gc` (Garbage Collection)
To keep repositories compact and fast, Git runs **packfile compaction**:
1. It gathers thousands of loose objects and compresses them into a single binary **Packfile** (`.pack`) accompanied by an index file (`.idx`) for $O(1)$ fast lookups.
2. **Delta Compression:** Git compares similar blobs (usually files sharing the same path name) and stores one as a base, and the other as a delta (the differences).
3. **The Reverse-Delta Trick:** To optimize performance, Git stores the **newest (latest) version of the file as the full base**, and the older historical versions as delta differences. This is because HEAD checkouts represent 99% of developer operations and must run with zero delta calculation overhead.

---

## 4. Disaster Recovery: The Reflog

If you run a destructive command like `git reset --hard HEAD~5` or delete an unmerged feature branch, you rewrite the pointers of your branch refs (like `refs/heads/main`). 
* **The Reality:** The commits are not deleted. They simply lose all pointing references, turning into **Dangling Commits** in the DAG.
* **The Safeguard:** Git keeps a local, append-only ledger of every single movement of your local HEAD pointer inside `.git/logs/HEAD`, known as the **Reflog**.

```
Reflog records all pointer movements (e.g., commit, checkout, reset, rebase)
[HEAD@{0}]: Reset to HEAD~1 (Lost commits!)
[HEAD@{1}]: Commit: Implement User Authentication (Dangling, but hash exists here!)
[HEAD@{2}]: Checkout: moving from main to dev
```

### The Recovery Pipeline

#### Step 1: Inspect local HEAD history
```bash
$ git reflog
c7e7eb2 HEAD@{0}: reset: moving to HEAD~1
fae4044 HEAD@{1}: commit: Implement User Authentication
```

#### Step 2: Recover the dangling commits
Since the commits are still fully preserved inside `.git/objects/`, you can instantly restore your repository state by checking out, merging, or cherry-picking the orphaned commit hash directly:
```bash
$ git checkout -b recovery-branch fae4044
```

#### Step 3: Deep Scan for Dangling Objects
If the reflog entry has expired (defaults to 90 days), you can scan the underlying database blocks directly for orphaned objects:
```bash
$ git fsck --lost-found
dangling commit fae40445d0cd87d7b...
```

---

## 5. The Mechanics of a Three-Way Merge

When you merge two branches containing divergent paths, Git performs a **three-way merge** to automatically resolve line changes:

```
               [Ours: Current HEAD]
              /
      [Common Ancestor: Base]
              \
               [Theirs: Incoming Branch]
```

To decide if a file has conflicts, Git compares three versions of the file:
1. **Base:** The most recent common ancestor commit of the two branches.
2. **Ours:** Your current HEAD commit.
3. **Theirs:** The commit you are merging in.

### The Resolving Logic
For every line block, Git compares the states:

| Ours vs. Base | Theirs vs. Base | Git Decision / Resolution |
|:---:|:---:|---|
| **Unchanged** | **Unchanged** | No change. Keep Base line. |
| **Modified** | **Unchanged** | Take **Ours** modification automatically. |
| **Unchanged** | **Modified** | Take **Theirs** modification automatically. |
| **Modified** | **Modified** | **MERGE CONFLICT!** Git suspends merge, writes conflict markers (`<<<<<<<`, `=======`, `>>>>>>>`), and awaits manual developer intervention. |

---

## 6. Popular Interview Questions & High-Impact Answers

### Q1: Explain why Git is described as a "snapshot-based" system rather than a "delta-based" system.
* **Answer:** Traditional VCS like SVN store a base file and append a chain of incremental delta modifications over time. To check out a specific revision, the server must sequentially apply the delta chain from the beginning, which degrades performance as history grows. **Git stores every version of every file as a complete, compressed snapshot (a blob)**. When a file is unchanged in a new commit, Git does not duplicate the file; instead, the new commit's tree object simply references the existing blob's hash. This makes branch checkouts, logs, and resets instantaneous, since checking out a commit is simply a matter of unpacking the pre-computed tree snapshot.

### Q2: What are "Dangling Commits" in Git, and how does `git reflog` protect against data loss?
* **Answer:** A dangling commit is an orphaned commit node in the Git Directed Acyclic Graph (DAG) that is no longer reachable by any active reference (such as a branch, tag, or HEAD pointer). This typically occurs after running commands like `git reset --hard`, `git commit --amend`, or deleting branches. 
  
  **`git reflog`** acts as a safety net by recording every single movement of the local `HEAD` pointer, regardless of branch states. Reflog entries are stored locally and are independent of remote synchronizations. Even if a commit is lost from branch logs, its hash remains recorded in the reflog for up to 90 days, allowing developers to retrieve the hash and instantly rebuild refs using `git checkout -b <branch> <hash>`.

### Q3: How does Git calculate object hashes, and why can two identical files never have different hashes?
* **Answer:** Git hashes objects deterministically using the SHA-1 algorithm. The input to the hash is not just the file content, but a standardized header prefix: `blob [size_in_bytes]\0[content_bytes]`. Because the SHA-1 calculation is cryptographic and depends strictly on the combined size and content bytes, two files containing the exact same characters will always yield the exact same SHA-1 hash, regardless of their filenames or locations in the folder tree. This provides implicit file-level deduplication across the entire repository.

### Q4: What is the difference between a Git merge conflict and a three-way merge?
* **Answer:** A **Three-Way Merge** is the automated algorithm Git uses to combine two divergent branches. It analyzes three commits: the common ancestor base, the current branch (Ours), and the target merge branch (Theirs). If a line block is modified in only one branch compared to the base, Git automatically merges it. A **Merge Conflict** is a specific failure state of this algorithm. It occurs when the three-way comparison detects that the same line block has been modified in different ways by *both* Ours and Theirs compared to the base, forcing Git to pause and request manual resolution.

### Q5: How does `git gc` optimize repository performance under the hood?
* **Answer:** Over time, active development creates thousands of independent "loose" zlib-compressed object files in `.git/objects/`, slowing down file lookup performance. When `git gc` runs:
  1. It packs these loose objects into a single compressed binary **Packfile** (`.pack`) alongside an index file (`.idx`) for $O(1)$ lookup performance.
  2. It executes **delta compression** where similar files (e.g., historical revisions) are grouped, compressing older versions as difference offsets relative to the newest version base.
  3. It identifies and permanently deletes (prunes) dangling objects that are older than the grace period threshold (defaults to 14 days), reclaiming disk space.
