# Git Log — Inspecting History

> **Category**: Version Control | **Level**: Intermediate

---

## What

`git log` lets you inspect the **commit history** of a repository. It's essential for reviewing past changes, debugging issues, and identifying when features were added or bugs introduced.

---

## Basic Commit Viewing

```bash
git log

# commit ca82a6dff817ec66f44342007202690a93763949
# Author: Scott Chacon <schacon@gee-mail.com>
# Date:   Mon Mar 17 21:52:11 2008 -0700
#
#    Change version number
```

- Shows commits in **reverse chronological order** (newest first).
- Includes: SHA-1 hash, author, date, and message.

---

## Useful Log Options

### Patch / Diff (`-p`)

Show detailed changes (diff) in each commit:

```bash
git log -p -2
# -2 limits to the last 2 commits
```

### Statistics (`--stat`)

Summarize file changes per commit:

```bash
git log --stat

# commit 3ffca12c6af40667f92d9b478d4e89d5c12dd4d2
# Author: Vinh Tran <lawtrann@gmail.com>
# Date:   Wed Jun 11 00:24:37 2025 +0700
#
#    [LT-75870] Include submission from SP flow
#
# internal/eureka/v2/lo_stats.go | 20 ++++++++++++--------
# 1 file changed, 12 insertions(+), 8 deletions(-)
```

### Compact Format (`--oneline`)

One commit per line:

```bash
git log --oneline

# ca82a6d Change version number
# 085bb3b Remove unnecessary test
```

### Custom Format (`--pretty=format`)

```bash
git log --pretty=format:"%h - %an, %ar : %s"

# ca82a6d - Scott Chacon, 6 years ago : Change version number
# 085bb3b - Scott Chacon, 6 years ago : Remove unnecessary test
```

| Placeholder | Meaning |
|-------------|---------|
| `%h` | Abbreviated commit hash |
| `%an` | Author name |
| `%ar` | Relative author date |
| `%s` | Commit subject/message |

### Graph View

```bash
git log --oneline --graph --all
```

---

## Limiting Log Output

### By Number

```bash
git log -3
# Show last 3 commits
```

### By Date

```bash
git log --since="2 weeks ago"
git log --since="2024-01-01" --until="2024-01-31"
```

### By Author

```bash
git log --author="Scott Chacon"
```

### By Commit Message

```bash
git log --grep="fix bug"
```

### By File / Path

```bash
git log -- path/to/file
```

### Exclude Merge Commits

```bash
git log --no-merges
```

---

## Combining Filters

```bash
git log --author="Alice" --grep="bugfix" --since="2024-01-01"
```

---

## What to Learn Next

- [05-git-undo-and-reset.md](05-git-undo-and-reset.md) — Amend commits, undo mistakes, reset branches
