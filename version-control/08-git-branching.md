# Git Branching

> **Category**: Version Control | **Level**: Intermediate

---

## What is a Branch?

A **branch** is a **movable pointer** to a commit. Branching means you diverge from the main line of development and continue work without affecting it.

Key properties:
- Branches are **lightweight and instantaneous** — no file duplication like older VCS tools.
- Encourages **frequent branching and merging**.
- `HEAD` points to the **currently active branch**.

---

## Creating and Switching Branches

**Create a new branch:**
```bash
git branch testing
```

**Switch to a branch:**
```bash
git checkout testing
# or (Git 2.23+):
git switch testing
```

**Create and switch in one step:**
```bash
git checkout -b testing
# or (Git 2.23+):
git switch -c testing
```

**Return to previous branch:**
```bash
git switch -
```

---

## HEAD and Branch Pointers

- `HEAD` always points to the **currently active branch**.
- When you commit, the branch `HEAD` points to **moves forward** to the new commit.
- Switching branches **changes your working directory** to reflect that branch's state.

---

## Viewing Branches

```bash
git branch              # list all local branches (* = current)
git branch -v           # show last commit on each branch
git branch -a           # show local + remote branches
git branch -r           # show remote branches only
```

---

## Merged vs Unmerged

```bash
# Branches already merged into current (safe to delete):
git branch --merged

# Branches NOT yet merged (risky to delete):
git branch --no-merged
```

---

## Deleting Branches

```bash
# Delete a merged branch (safe):
git branch -d branch-name

# Force delete an unmerged branch:
git branch -D branch-name
```

---

## Renaming Branches

```bash
# Rename locally:
git branch --move old-name new-name

# Push the renamed branch and set as upstream:
git push --set-upstream origin new-name

# Delete the old remote branch:
git push origin --delete old-name
```

---

## Changing the Default Branch (e.g., `master` → `main`)

```bash
# Step 1: Rename locally
git branch --move master main

# Step 2: Push to remote and set upstream
git push --set-upstream origin main

# Step 3: Delete old remote branch (after updating all integrations/settings)
git push origin --delete master
```

---

## Typical Branch Workflow

```bash
# Create feature branch from main:
git checkout -b feature/my-feature main

# ... make changes ...

git add .
git commit -m "Add my feature"

# Merge back into main:
git checkout main
git merge feature/my-feature

# Clean up:
git branch -d feature/my-feature
```

---

## Pitfalls

- Switching branches with **uncommitted changes** can cause conflicts — commit or stash first.
- Deleting a branch only removes the pointer, not the commits (until garbage collected).
- Always check `git branch --no-merged` before pruning branches.

---

## What to Learn Next

- [09-git-merging.md](09-git-merging.md) — Integrate branch changes
- [10-git-rebasing.md](10-git-rebasing.md) — Alternative to merging for linear history
