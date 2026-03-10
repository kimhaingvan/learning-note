# Git Cherry-Pick & Rewriting History

> **Category**: Version Control | **Level**: Intermediate

---

## Git Cherry-Pick

### What it Does

Applies the **changes from specific commits** onto your current branch. Creates a **new commit** with a different hash (commit is replayed, not moved).

**Use case:** You want one fix or feature commit without merging the whole branch.

---

### Cheat Sheet

**Basic:**
```bash
git cherry-pick <commit>          # Apply a single commit
git cherry-pick A B C             # Apply multiple commits in order
```

**Ranges:**
```bash
git cherry-pick A..B              # All commits after A up to B (A excluded)
git cherry-pick A^..B             # Include A as well (full range)
```

**Continue / Abort:**
```bash
git cherry-pick --continue        # After resolving conflicts
git cherry-pick --abort           # Cancel, return to pre-pick state
git cherry-pick --quit            # Stop but keep index/working tree as-is
```

**No Commit (stage only):**
```bash
git cherry-pick -n <commit>       # Apply changes but don't commit yet
```

**Edit or Annotate the Commit:**
```bash
git cherry-pick -e <commit>       # Apply and open editor for the message
git cherry-pick -x <commit>       # Append "(cherry picked from …)" to message
```

**Empty Commits:**
```bash
git cherry-pick --allow-empty     # Keep empty commits
```

---

### Pitfalls

- Cherry-pick duplicates commits — if you later merge the original branch, you'll get **duplicate changes** (visible as separate commits with different hashes).
- Prefer cherry-pick for **one-off backports** to older release branches, not as a general integration strategy.

---

## Rewriting History

### What it Means

"Rewriting history" changes commits that exist locally before pushing to a shared remote:

- Changing commit messages
- Editing files in old commits
- Squashing multiple commits together
- Reordering commits
- Removing commits
- Splitting one commit into multiple

Goal: **make your commit history cleaner and more meaningful** before sharing.

---

### Cheat Sheet

**Amend Last Commit:**
```bash
git commit --amend -m "Better commit message"

# Add a missed file without changing the message:
git add missed_file.txt
git commit --amend --no-edit
```

**Interactive Rebase for Multiple Commits:**
```bash
git rebase -i HEAD~3
# Interactively edit the last 3 commits
```

Sample editor content:
```
pick a1b2c3 Add user login
reword d4e5f6 Fix typo in login
squash g7h8i9 Adjust login tests
```

| Keyword | Action |
|---------|--------|
| `pick` | Use commit as-is |
| `reword` | Change commit message |
| `edit` | Stop — allows amending content |
| `squash` | Combine into previous commit (keep all messages) |
| `fixup` | Combine into previous commit (discard this message) |
| `drop` | Remove this commit entirely |

**Reorder Commits:**
```bash
git rebase -i HEAD~4
# Then rearrange lines in the editor:

# Before:
pick 123aaa Add README
pick 456bbb Update docs
pick 789ccc Add API
pick def111 Fix API tests

# After (reordered):
pick 123aaa Add README
pick 789ccc Add API
pick def111 Fix API tests
pick 456bbb Update docs
```

**Squash Last 3 Commits:**
```bash
git rebase -i HEAD~3
# Mark the 2nd and 3rd commits as "squash"
```

**Split a Commit:**
```bash
git rebase -i HEAD~2
# Mark the commit to split as "edit"

# After editor stops at that commit:
git reset HEAD^              # uncommit, leaving changes in working tree
git add file1
git commit -m "Part one of changes"
git add file2
git commit -m "Part two"
git rebase --continue
```

**Drop a Commit:**
```bash
git rebase -i HEAD~5
# Change "pick" to "drop" for any commit to remove it
```

---

### Rules for Safe History Rewriting

> ⚠️ Only rewrite commits that **have not been pushed** to a shared remote.

| Situation | Safe? |
|-----------|-------|
| Local-only commits | ✅ Safe to rewrite |
| Already pushed to your own feature branch (no one else uses it) | ✅ OK with `--force-with-lease` |
| Pushed to shared/main branch | ❌ Never rewrite — use `git revert` instead |

---

## What to Learn Next

- [09-git-merging.md](09-git-merging.md) — Merge strategies
- [10-git-rebasing.md](10-git-rebasing.md) — Rebase deep dive
- [12-cicd.md](12-cicd.md) — CI/CD pipelines and deployment workflows
