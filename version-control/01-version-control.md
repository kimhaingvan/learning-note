# Version Control & Git Fundamentals

> **Category**: Version Control | **Level**: Intermediate

---

## What is Version Control?

**Version control** is a system that manages and tracks changes to files over time, allowing multiple people to collaborate on a project while maintaining a history of all modifications. It records changes to files (code, documents, config files) and stores them in a **repository**.

**Why it matters:**
- Tracks every modification with a record of who changed what and why.
- Allows multiple developers to work on the same codebase simultaneously without overwriting each other's work.
- Essential for rollbacks, audits, and collaboration.

---

## What is Git?

**Git is a distributed version control system** that tracks changes by **taking snapshots** of your project over time, rather than storing differences (deltas) like older systems (CVS, Subversion).

### Three File States

Every file in Git lives in one of three states:

| State | Meaning |
|-------|---------|
| **Modified** | You changed a file but haven't committed it yet |
| **Staged** | You marked a modified file to go into your next commit snapshot |
| **Committed** | The data is safely stored in your local database |

**Cycle:** Modified → Staged → Committed

---

## How Git Stores Data

### Snapshot-Based (Not Changesets)

- Git **does NOT store diffs**.
- Each commit is a **full snapshot** of your project at that moment in time.

### Why Snapshots Aren't Heavy

Git reuses unchanged objects. Files are stored as **blobs** (content-only, no filenames). If a file hasn't changed between commits, Git **doesn't store it again** — it reuses the same blob by referencing its SHA-1 hash.

**Example:**
1. Commit a project with 100 files.
2. Change **just one** file and commit again.

Git only stores:
- 1 new blob (the changed file)
- 1 new tree object
- 1 new commit object

The other 99 files are **not duplicated**.

> Under the hood: Git uses **content-addressed storage** — every file is stored by its SHA-1 checksum. If two files are identical, Git stores **just one copy**, even across branches or commits.

---

## What Happens During a Commit?

### Object Graph

```
Commit:  d3e1a6b   ← (HEAD, master) #2
│
├── Message: "Update test.rb"
├── Parent: a1b2c3d   ← Commit #1
└── Tree:   e4f5a7c
              │
              ├── README    → Blob b6a8d1e  (same as before)
              ├── test.rb   → Blob f9c3e2b  (new content)
              └── LICENSE   → Blob 3d4e5f1  (same as before)
------------------------------------------------------------
Commit:  a1b2c3d   ← Initial Commit #1
│
├── Message: "Initial commit"
└── Tree:   c7d8e9f
              │
              ├── README    → Blob b6a8d1e
              ├── test.rb   → Blob a4b5c6d
              └── LICENSE   → Blob 3d4e5f1
```

### Step-by-Step

1. **Stage files** — Git computes a SHA-1 hash for each file and stores them as **Blobs**:
   ```bash
   git add README test.rb LICENSE
   ```

2. **First commit** — Git creates:
   - ✅ **1 Tree Object** `c7d8e9f` — pointers to all 3 blobs
   - ✅ **1 Commit** `a1b2c3d` — metadata (author, date, message) + pointer to tree; Parent: NULL

3. **Next commit** (only `test.rb` changed):
   ```bash
   git add test.rb
   git commit -m "Update test.rb"
   ```
   Git creates:
   - ✅ **1 new Blob** for `test.rb` (new content `f9c3e2b`)
   - ✅ **1 new Tree** pointing to: old README blob, new test.rb blob, old LICENSE blob
   - ✅ **1 new Commit** with Parent = `a1b2c3d`

---

## What to Learn Next

- [02-git-config-and-aliases.md](02-git-config-and-aliases.md) — Set up your identity and shortcuts
- [03-git-repository-and-staging.md](03-git-repository-and-staging.md) — Init, clone, and record changes
- Reference: [roadmap.sh/git-github](https://roadmap.sh/git-github)
