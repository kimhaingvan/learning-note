# Git Tags

> **Category**: Version Control | **Level**: Intermediate

---

## What is Tagging?

Tags mark **specific commits** — most commonly used to label **releases** (e.g., `v1.0`, `v2.0`).

### Two Tag Types

| Type | Description |
|------|-------------|
| **Annotated** | Full Git objects with message, tagger info, date, optional GPG signature |
| **Lightweight** | Simple pointer to a commit — no metadata |

---

## Quick Reference

| Task | Command |
|------|---------|
| List all tags | `git tag` |
| Filter tags | `git tag -l "v1.*"` |
| Create annotated tag | `git tag -a v1.0 -m "Initial release"` |
| Create lightweight tag | `git tag v1.0-lw` |
| Tag an older commit | `git tag -a v1.2 <commit-hash>` |
| View tag details | `git show v1.0` |
| Push a tag | `git push origin v1.0` |
| Push all tags | `git push origin --tags` |
| Delete local tag | `git tag -d v1.0` |
| Delete remote tag | `git push origin --delete v1.0` |
| Checkout tag (detached HEAD) | `git checkout v1.0` |
| Create branch from tag | `git checkout -b fix-v1.0 v1.0` |

---

## Listing Tags

```bash
git tag                              # list all tags (alphabetically)
git tag -l "v1.*"                    # by pattern
git tag -n2                          # show first 2 lines of annotation
git tag --sort=version:refname       # semantic-like version sort
git tag --contains <commit>          # tags that include a specific commit
git tag --merged main                # tags reachable from main
git tag --points-at <object>         # tags pointing at an object (e.g., HEAD)
```

---

## Creating Tags

**Annotated tag** (recommended for releases):
```bash
git tag -a v1.4 -m "Release version 1.4"
```

**Lightweight tag** (just a pointer):
```bash
git tag v1.4-lw
```

**Tag an older commit** (using its hash):
```bash
git tag -a v1.2 9fceb02
```

---

## Viewing Tag Details

```bash
git show v1.4
# Shows: tagger info, date, message, and the tagged commit
```

---

## Pushing Tags to Remote

```bash
# Push a single tag:
git push origin v1.5

# Push all tags:
git push origin --tags

# Push only annotated tags (recommended):
git push origin --follow-tags
```

> By default, `git push` does **not** push tags — you must push them explicitly.

---

## Deleting Tags

**Delete a local tag:**
```bash
git tag -d v1.4-lw
```

**Delete a remote tag (two equivalent approaches):**
```bash
git push origin :refs/tags/v1.4-lw
# or:
git push origin --delete v1.4-lw
```

**Delete and re-tag at a different commit:**
```bash
git tag -d v1.2.0
git tag -f -a v1.2.0 -m "Fix tag to correct commit"
```

---

## Checking Out a Tag

```bash
# Detached HEAD state (read-only):
git checkout v2.0.0

# Recommended — create a branch from the tag:
git checkout -b version2 v2.0.0
```

> Checking out a tag puts Git in **detached HEAD** state. Create a branch from it if you need to make commits.

---

## What to Learn Next

- [08-git-branching.md](08-git-branching.md) — Work with branches
- [12-cicd.md](12-cicd.md) — Automate deployments using CI/CD pipelines triggered by tags
