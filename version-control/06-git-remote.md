# Git Remote, Fetch & Pull

> **Category**: Version Control | **Level**: Intermediate

---

## Git Remote

A **remote repository** is a version of your project hosted elsewhere (online, network, or local). Collaborating means pushing, pulling, and fetching from remotes.

### Key Terms

| Term | Meaning |
|------|---------|
| **origin** | Default name for the remote you cloned from |
| **Tracked branch** | Local branch linked to a remote branch |
| **Fetch** | Download data from remote (no merge) |
| **Pull** | Fetch + merge |
| **Push** | Upload local commits to remote |

---

### Viewing Remotes

```bash
git remote           # list remotes by shortname
git remote -v        # show fetch & push URLs

# Output:
# origin  https://github.com/user/repo.git (fetch)
# origin  https://github.com/user/repo.git (push)
```

### Adding a Remote

```bash
git remote add <name> <url>

# Example:
git remote add pb https://github.com/paul/repo
```

### Inspecting a Remote

```bash
git remote show origin
# Shows: URLs, tracked branches, push/pull mappings
```

### Renaming a Remote

```bash
git remote rename old-name new-name
```

### Removing a Remote

```bash
git remote remove paul
# or:
git remote rm paul
```

### Clean Up Stale References

```bash
git remote prune origin
# Removes local refs to branches deleted on the remote
```

### Pushing to a Remote

```bash
git push <remote> <branch>

# Example:
git push origin master
```

---

## Git Fetch

`git fetch` **downloads** new commits, branches, and tags from a remote but **does NOT merge** anything. Your working directory stays unchanged.

The fetched changes appear in references like `origin/main`, but they're not merged into your local `main` until you explicitly do so:

```bash
git merge origin/main
```

### Fetch vs Pull

| Command | Effect |
|---------|--------|
| `git fetch` | Download only — no changes to working branch |
| `git pull` | Fetch + merge (or rebase) |

---

### Fetch Cheat Sheet

**Basics:**
```bash
git fetch                   # Update from default remote (origin)
git fetch origin            # Explicit remote
git fetch origin branch     # Fetch only one branch
git fetch --all             # Fetch from all remotes
```

**Cleaning up:**
```bash
git fetch --prune           # Remove local refs to deleted remote branches
git fetch --prune --tags    # Also prune tags

# Set pruning as default:
git config --global fetch.prune true
git config remote.origin.prune true
```

**Tags:**
```bash
git fetch --tags            # Fetch all tags
git fetch --no-tags         # Disable auto tag fetching
```

**Shallow & partial:**
```bash
git fetch --depth=1         # Fetch only last 1 commit
git fetch --depth=N         # Fetch last N commits
git fetch --unshallow       # Convert shallow clone to full
git fetch --deepen=N        # Add N more commits to history
```

**Custom refspec:**
```bash
git fetch origin main:local-main         # Save remote main as local branch
git fetch origin featureX:myFeature      # Custom refspec
```

---

## Git Pull

`git pull` = `git fetch` + `git merge` (or rebase). It updates your local branch **and immediately integrates** the fetched changes.

### Cheat Sheet

**Basic:**
```bash
git pull                    # fetch + merge from default remote branch
git pull origin main        # fetch + merge remote main into current branch
```

**Choose integration style:**
```bash
git pull --merge                  # fetch + merge (default)
git pull --rebase                 # fetch + rebase (linear history)
git pull --rebase=merges          # rebase, preserving merge commits
git pull --rebase=interactive     # interactive rebase after fetch
git pull --no-rebase              # disable rebase if set in config
```

**Fast-forward behavior:**
```bash
git pull --ff-only                # update only if fast-forward possible (safest)
git pull --ff                     # allow fast-forward (default)
git pull --no-ff                  # always create a merge commit
```

**Merge conflict strategy:**
```bash
git pull -X ours                  # on conflicts, prefer our changes
git pull -X theirs                # on conflicts, prefer incoming changes
git pull --no-edit                # use default merge message without editor
```

**Fetch options during pull:**
```bash
git pull --prune                  # drop remote-tracking refs deleted upstream
git pull --tags                   # fetch all tags
git pull --depth=1                # shallow fetch before integrating
git pull --all                    # fetch from all remotes
```

---

## Pitfalls

- `git pull` with diverged history can create unwanted merge commits — prefer `git pull --rebase` for cleaner history.
- Never `git push --force` on a shared branch without team agreement.

---

## What to Learn Next

- [08-git-branching.md](08-git-branching.md) — Create and manage branches
- [09-git-merging.md](09-git-merging.md) — Merge strategies explained
