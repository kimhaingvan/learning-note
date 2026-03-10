# Git Rebasing

> **Category**: Version Control | **Level**: Intermediate

---

## What is `<upstream>`?

In Git, the **upstream** branch is the branch your current branch is tracking — usually the remote version of `main` or `develop`.

Setting upstream explicitly:
```bash
git branch --set-upstream-to=origin/main feature
```

Or when creating a branch from origin:
```bash
git checkout -b feature origin/main
# → origin/main is automatically set as upstream
```

---

## What `git rebase` Does

1. Finds all commits in your branch that are **not** in `<upstream>` / base branch.
2. Saves those commits to a temporary area.
3. Resets your branch to point at the base.
4. **Replays the saved commits one by one** on top of the new base.
5. Commits already present upstream (even with a different hash) are **skipped**.
6. If conflicts arise → resolve them → `git rebase --continue`.

> "Pretend I started my work later, on top of the updated base."

---

## Merge vs Rebase

| Aspect | Merge | Rebase |
|--------|-------|--------|
| History | Preserves branches (merge commits) | Creates linear history |
| Commit hashes | Original hashes preserved | New hashes (commits are replayed) |
| Safe for shared branches | ✅ | ⚠️ Not safe (rewrites history) |
| Readability | Can be noisy with merge commits | Clean, linear |

> **Golden Rule:** Never rebase commits that exist outside your local repository (i.e., already pushed to a shared branch).

---

## Cheat Sheet

### Basic Rebase

```bash
git checkout feature
git rebase main
# Reapply commits from feature on top of main (linear history)

git rebase origin/main
# Same as above, rebasing onto remote main
```

### Interactive Rebase

```bash
git rebase -i HEAD~5
# Interactively rewrite last 5 commits
```

Inside the editor, you can use:

| Action | Effect |
|--------|--------|
| `pick` | Use the commit as-is |
| `reword` | Use commit, but edit the message |
| `edit` | Pause to amend content |
| `squash` | Merge into previous commit (keep messages) |
| `fixup` | Merge into previous commit (discard message) |
| `drop` | Remove this commit entirely |

### Conflict Handling

```bash
git rebase --continue   # after resolving conflicts
git rebase --abort      # cancel rebase, restore original state
git rebase --skip       # skip the problematic commit
```

### Onto Option

Move a range of commits to a completely different base:

```bash
git rebase --onto master next topic
# Replay commits from topic (not in next) onto master
# Useful for rebasing a sub-branch onto a different branch
```

### Dropping Commits

```bash
git rebase --onto topic~5 topic~3 topic
# Remove the range [topic~5, topic~3), keep the rest
```

### Conflict Strategy

```bash
git rebase -X ours main     # on conflict, prefer changes from our branch
git rebase -X theirs main   # on conflict, prefer changes from upstream (main)
```

---

## Common Workflow: Rebase Before PR

```bash
git checkout feature
git fetch origin
git rebase origin/main      # bring feature up-to-date with main
# resolve any conflicts...
git push --force-with-lease  # update remote feature branch (safe force push)
```

---

## Pitfalls

- **Rebasing shared branches** causes problems for everyone who has pulled those commits — their history diverges.
- `--force` push is required after rebasing a pushed branch — always use `--force-with-lease` (it fails if someone else pushed in the meantime).
- An interactive rebase on many commits can be tedious to resolve if there are many conflicts — consider smaller, more frequent rebases.

---

## Memory Aid

> Rebase = cut and paste your commits onto a new base. Merge = staple two histories together.

---

## What to Learn Next

- [11-git-cherry-pick-and-rewriting.md](11-git-cherry-pick-and-rewriting.md) — Pick specific commits, squash history
