# Git Undo & Reset

> **Category**: Version Control | **Level**: Intermediate

---

## Git Undo

Tools to **undo actions** in Git: modifying commits, unstaging files, and reverting working directory changes.

### Quick Reference

| Situation | Command |
|-----------|---------|
| Amend last commit (fix mistake) | `git commit --amend` |
| Unstage file (keep changes) | `git restore --staged <file>` or `git reset HEAD <file>` |
| Discard local changes (**danger!**) | `git restore <file>` or `git checkout -- <file>` |

---

### Amending Commits

Use when you committed too early (missed a file, typo in message). The amended commit **replaces** the last one entirely.

```bash
git commit -m "Initial commit"
git add forgotten_file
git commit --amend
```

> ⚠️ **Only amend local commits** that haven't been pushed. Never amend commits already shared with others.

---

### Unstaging a Staged File

Move files back from **staged → modified**:

```bash
# Modern method (Git 2.23+):
git restore --staged <file>

# Traditional method:
git reset HEAD <file>
```

Example:
```bash
git add CONTRIBUTING.md README.md
git restore --staged CONTRIBUTING.md
# → CONTRIBUTING.md is now unstaged; README.md stays staged
```

---

### Discarding Local Changes

**Permanently discards** unsaved local changes, reverting the file to its last committed state:

```bash
# Modern method (preferred):
git restore <file>

# Traditional method:
git checkout -- <file>
```

> ⚠️ This is **irreversible** — local changes are gone.

---

## Git Reset

### What

`git reset` is a **local history editing tool** that moves your branch tip and optionally syncs the staging area (index) and working files.

**Two forms:**

| Form | Syntax | Effect |
|------|--------|--------|
| **Pathspec** | `git reset [<tree-ish>] -- <paths>` | Updates the **index only** for those paths. Equivalent to `git restore --staged`. Does NOT move your branch. |
| **Commit-moving** | `git reset [--mode] <commit>` | **Moves HEAD/branch** and may update index and working tree. Default mode: `--mixed`. |

---

### When to Use Which Mode

| Use-case | Recommended |
|----------|-------------|
| Just unstage files (keep edits) | `git reset -- <paths>` or `git reset -p` |
| Go back to `<commit>`, keep changes **staged** | `git reset --soft <commit>` |
| Go back to `<commit>`, keep changes **unstaged** | `git reset <commit>` (default `--mixed`) |
| Discard all local edits — go back precisely to `<commit>` | `git reset --hard <commit>` ⚠️ dangerous |
| Go back but keep unstaged local mods if possible | `git reset --merge <commit>` or `--keep` |

---

### How — Examples

**Unstage everything (keep edits):**
```bash
echo C >> file.txt
git add file.txt
git reset           # index cleared; file still has "C"
```

**Unstage one file only:**
```bash
echo D >> file.txt && git add file.txt
git reset -- file.txt
```

**Unstage only part of a file (interactive):**
```bash
printf "\nE1\nE2\nE3\n" >> file.txt
git add file.txt
git reset -p -- file.txt   # pick hunks to unstage
```

**Undo last commit, keep changes staged:**
```bash
git reset --soft HEAD~1
# → now run git commit --amend or make a new commit
```

**Undo last commit, keep changes unstaged:**
```bash
git reset HEAD~1    # default --mixed
```

**Squash last 3 commits into one (without rebase):**
```bash
git reset --soft HEAD~3
git commit -m "Squashed work"
```

**Hard reset to a specific commit:**
```bash
git log --oneline   # copy <sha>
git reset --hard <sha>
```

**Recover from an accidental hard reset:**
```bash
git reflog                  # find the previous HEAD, e.g. HEAD@{1}
git reset --hard HEAD@{1}
```

**Already pushed? Use revert instead:**
```bash
# On a shared branch, prefer revert (creates a new commit — safe for others)
git revert <sha_to_undo>
git push
```

---

## Pitfalls

- `git reset --hard` is **permanent for uncommitted work** — use `--soft` or `--mixed` first if unsure.
- On shared branches, prefer `git revert` — it creates a new "undo" commit instead of rewriting history.
- Save your position before any reset: `git tag backup-point` or note the hash from `git log`.

---

## Memory Aid

> `--soft` = soft landing (changes stay staged). `--mixed` = medium (changes back to modified). `--hard` = hard crash (changes gone).

## What to Learn Next

- [06-git-remote.md](06-git-remote.md) — Work with remote repositories
- [11-git-cherry-pick-and-rewriting.md](11-git-cherry-pick-and-rewriting.md) — Advanced history editing
