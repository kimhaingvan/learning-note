# Git Repository & Recording Changes

> **Category**: Version Control | **Level**: Intermediate

---

## Git Repository

### Two Ways to Get a Repository

#### 1. Initialize a New Repository

```bash
# Navigate to your project directory:
cd /path/to/your/project

# Initialize Git:
git init

# Start tracking files and make first commit:
git add *.c
git add LICENSE
git commit -m "Initial project version"
```

> `git init` sets up Git, but you must **add** files and make an **initial commit** to start version control.

#### 2. Clone an Existing Repository

```bash
git clone <repository-url>

# Example:
git clone https://github.com/libgit2/libgit2

# Clone into a custom directory name:
git clone https://github.com/libgit2/libgit2 mylibgit
```

### Supported Transfer Protocols

| Protocol | Example |
|----------|---------|
| **HTTPS** | `https://github.com/user/repo.git` |
| **Git** | `git://server/repo.git` |
| **SSH** | `user@server:path/to/repo.git` |

> For secure, authenticated access (pushing changes), SSH is most commonly used.

---

## Recording Changes to the Repository

### File States

| State | Description |
|-------|-------------|
| **Untracked** | Not yet known to Git |
| **Unmodified** | Tracked, no changes since last commit |
| **Modified** | Tracked, changed but not staged |
| **Staged** | Marked to go into the next commit |

**Lifecycle:** Untracked → Staged → Committed → (Unmodified) → Modified → Staged …

---

### Quick Reference

| Action | Command |
|--------|---------|
| Check file status | `git status` |
| Stage a file | `git add <file>` |
| Commit staged files | `git commit -m "message"` |
| Ignore files | Create `.gitignore` |
| View unstaged changes | `git diff` |
| View staged changes | `git diff --staged` |
| Remove file from repo | `git rm <file>` |
| Rename file | `git mv old new` |

---

### Checking File Status

```bash
git status

# Short format:
git status -s
# ?? = untracked
# A  = staged new file
# M  = modified
```

---

### Staging Files

```bash
git add <filename>
```

> If you edit a file after staging it, the new changes are NOT staged. You must `git add` again.

---

### Ignoring Files

Create a `.gitignore` file at the repo root:

```gitignore
*.log
*~
build/
!lib.a      # exception: do track this file
```

---

### Viewing Changes (Diffs)

```bash
# Changes NOT staged (before git add):
git diff

# Changes staged for commit (after git add):
git diff --staged
# or:
git diff --cached
```

---

### Committing Changes

```bash
# Opens editor for commit message:
git commit

# Inline commit message:
git commit -m "your commit message"

# Stage all modified tracked files and commit (skips git add):
git commit -a -m "quick commit message"
```

---

### Removing Files

```bash
# Remove from disk AND stop tracking:
git rm <file>

# Stop tracking but KEEP the file on disk:
git rm --cached <file>
```

---

### Renaming or Moving Files

```bash
git mv old_name new_name
```

This is equivalent to:
```bash
mv old_name new_name
git rm old_name
git add new_name
```

---

## What to Learn Next

- [04-git-log.md](04-git-log.md) — Inspect commit history
- [05-git-undo-and-reset.md](05-git-undo-and-reset.md) — Undo mistakes
