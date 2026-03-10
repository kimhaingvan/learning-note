# Git Config & Aliases

> **Category**: Version Control | **Level**: Intermediate

---

## Git Config

Git uses the `git config` tool to manage configuration variables controlling everything from your identity to the default text editor.

### Configuration Levels

| Level | Flag | File | Scope |
|-------|------|------|-------|
| **System** | `--system` | `/etc/gitconfig` | All users on the system (requires admin) |
| **Global** | `--global` | `~/.gitconfig` or `~/.config/git/config` | Current user only |
| **Local** | `--local` (default) | `.git/config` | Specific repository only |

> **Precedence:** Local → Global → System (local wins)

---

### Viewing Configurations

```bash
# View all settings with their source file:
git config --list --show-origin

# Check a specific value:
git config user.name

# Find where a specific config came from:
git config --show-origin user.name

# List all current settings:
git config --list
```

---

### Set Your Identity

Required for every commit:

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

---

### Default Branch Name

Set the default branch name for new repositories (Git 2.28+):

```bash
git config --global init.defaultBranch main
```

---

### Pull Behavior

```bash
# Default to merge strategy on pull:
git config --global pull.rebase false

# Default to rebase strategy on pull:
git config --global pull.rebase true
```

---

## Git Aliases

Aliases let you create **short custom commands** for frequently used Git operations.

```bash
git config --global alias.<shortcut> "<command>"
```

### Common Aliases

```bash
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.st status
```

After setting these, `git co main` is the same as `git checkout main`.

### Alias for External Commands

Prefix with `!` to run shell commands:

```bash
git config --global alias.visual '!gitk'
```

---

## What to Learn Next

- [03-git-repository-and-staging.md](03-git-repository-and-staging.md) — Init, clone, and record changes
- [04-git-log.md](04-git-log.md) — Inspect commit history
