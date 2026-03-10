# Git Merging

> **Category**: Version Control | **Level**: Intermediate

---

## How Git Merge Works Internally

When you merge, Git uses **three key snapshots**:

1. **Current branch tip** (e.g., `main`)
2. **Branch being merged in** (e.g., `feature`)
3. **Common ancestor** of both — the **merge base**

Git performs a **three-way merge**: compares the merge base with the two branch tips, applies the differences, and creates a **new merge commit** if needed.

---

## Types of Merges

### Fast-Forward Merge

```
*---*---*---A---B (main)
                 \
                  C---D (feature)
```

- **When:** The current branch (`main`) has **no new commits** since `feature` branched off.
- Git simply moves the `main` pointer forward to `D` — **no merge commit created**.

### Recursive (Three-Way) Merge

```
A---B----C-----M (main)
 \           /
  D-------E (feature)
```

- **When:** Both branches have progressed independently.
- Git finds the shared ancestor (`A`), diffs `A→B→C` and `A→D→E`, combines the changes.
- Creates a **new merge commit `M`** with two parents: `C` and `E`.

### Merge Conflicts

Occurs when **both branches edit the same part** of a file. Git adds conflict markers:

```
<<<<<<< HEAD
your change
=======
their change
>>>>>>> feature
```

Resolution steps:
1. Manually edit the file to the desired content.
2. `git add <resolved-file>`
3. `git commit`

---

## Cheat Sheet

### Basic Merge

```bash
git checkout main
git merge feature
```

### Control Fast-Forward Behavior

```bash
git merge --ff-only feature     # only merge if fast-forward is possible (safe)
git merge --no-ff feature       # always create a merge commit
git merge --ff feature          # allow fast-forward (default)
```

### Commit Control

```bash
git merge --no-commit feature   # stage the merge but don't auto-commit
git merge --no-edit feature     # use default merge commit message
git merge --log feature         # include abbreviated commit messages in merge commit
git merge --signoff feature     # add a "Signed-off-by" line
```

### Conflict Handling

```bash
git merge -X ours feature       # on conflicts, favor our changes
git merge -X theirs feature     # on conflicts, favor incoming changes
git merge --abort               # cancel merge, return to pre-merge state
git merge --quit                # stop merge, keep index/working tree as-is
```

### Merge Strategies

```bash
git merge -s ort feature                        # default strategy for 2-head merges
git merge -s octopus feature1 feature2          # merge multiple branches at once
git merge -s ours feature                       # keep our content entirely (record ancestry only)
git merge --allow-unrelated-histories other-branch  # merge repos with no shared history
```

### Squash Merge

```bash
git merge --squash feature
git commit -m "Squash merge feature branch"
# Combines all feature commits into a single commit; no merge ancestry
```

---

## Merge vs Fast-Forward Visual

| Scenario | Fast-Forward | No-FF |
|----------|-------------|--------|
| Main has no new commits | Pointer just moves | Creates merge commit |
| Main has new commits | Not possible | Three-way merge + commit |
| History clarity | Linear (no merge commit) | Explicit record of branch integration |

---

## Pitfalls

- **Fast-forward loses branch context** — use `--no-ff` when you want to preserve the fact that a feature branch existed.
- **Merge commits on shared branches** can clutter history — consider rebasing feature branches before merging.
- Never `git merge --abort` after resolving all conflicts — only abort if you want to cancel the entire merge.

---

## What to Learn Next

- [10-git-rebasing.md](10-git-rebasing.md) — Alternative to merging: reapply commits on top of another branch
- [11-git-cherry-pick-and-rewriting.md](11-git-cherry-pick-and-rewriting.md) — Pick specific commits
