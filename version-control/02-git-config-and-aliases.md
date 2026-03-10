# Git Config & Aliases

> **Category**: Version Control | **Level**: Intermediate

---

## The Mental Model — Three Separate Layers

A very common source of confusion: people think changing their name in Git config fixes their SSH access, or vice versa. These are **three completely independent mechanisms**:

| Layer | Config Key | What It Controls |
|-------|-----------|-----------------|
| **Commit identity** | `user.name`, `user.email` | Author/committer metadata baked into each commit object |
| **Repository configuration** | `.git/config`, `~/.gitconfig` | Remote URL, repo-specific overrides, default behaviors |
| **SSH authentication** | `~/.ssh/` keys, `~/.ssh/config` | Whether the remote server permits you to push/pull at all |

**The key point:**
- `user.name` / `user.email` → decides what **appears in the commit log**
- SSH key → decides **whether the server lets you access the repo**
- Remote URL → decides **where Git connects**

These two identities often represent the same person, but they are **not the same mechanism**.

---

## Git Config

Git uses the `git config` tool to manage configuration variables controlling everything from your identity to the default text editor.

### Configuration Levels

Git reads config from multiple places, in order of increasing priority:

| Level | Flag | File | Scope |
|-------|------|------|-------|
| **System** | `--system` | `/etc/gitconfig` | All users on the system (requires admin) |
| **Global** | `--global` | `~/.gitconfig` or `~/.config/git/config` | Current user only |
| **Local** | `--local` (default) | `.git/config` inside the repo | This repository only |

> **Precedence:** Local overrides Global, Global overrides System. Writing with `git config` (no flag) writes to the repo-local config.

---

### Viewing Configurations

```bash
# Best debugging command — shows value AND which file it came from:
git config --list --show-origin

# Check a specific value:
git config user.name

# Check where a value comes from:
git config --show-origin user.name

# Check global vs local explicitly:
git config --global --get user.email
git config --local --get user.email

# See the remote URLs:
git remote -v
```

---

### Set Your Identity

Required for every commit. Git records these inside the commit object itself.

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

**Safety setting** — prevents Git from guessing your identity:

```bash
git config --global user.useConfigOnly true
```

This forces you to explicitly configure identity per-repo. Essential when you use multiple emails.

---

### Repo-specific Identity Override

Work and personal repos usually need different emails. Override inside the repo:

```bash
cd ~/src/open-source-project
git config --local user.name "Alice Nguyen"
git config --local user.email "alice@users.noreply.example.com"
```

Because local config takes precedence, only this repo uses the local email. All other repos fall back to global.

---

### Auto-apply Identity by Directory (`includeIf`)

The cleanest multi-identity setup for engineers who switch between work and personal repos:

```ini
# ~/.gitconfig
[includeIf "gitdir:~/work/"]
    path = ~/.gitconfig-work

[includeIf "gitdir:~/personal/"]
    path = ~/.gitconfig-personal
```

```ini
# ~/.gitconfig-work
[user]
    name = Alice Nguyen
    email = alice@company.com
```

```ini
# ~/.gitconfig-personal
[user]
    name = Alice Nguyen
    email = alice.personal@example.com
```

Git automatically applies the matching config based on which directory the repo lives in. No manual per-repo setup needed.

---

### Default Branch Name

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

## SSH Authentication for Git

SSH handles **remote access control** — separate from who you are in commits.

### How It Works

```
git push
  └─▶ reads remote.origin.url from .git/config
       └─▶ connects via SSH to the remote host
            └─▶ SSH key authenticates your account
                 └─▶ server checks if you have write permission
```

### Step-by-Step Setup

**1. Check for existing keys**

```bash
ls -al ~/.ssh
```

Look for `id_ed25519.pub` or `id_rsa.pub`. If they exist, you may be able to skip key generation.

**2. Generate a new SSH key**

```bash
# Ed25519 (recommended):
ssh-keygen -t ed25519 -C "you@example.com"

# RSA 4096 (fallback for legacy systems):
ssh-keygen -t rsa -b 4096 -C "you@example.com"
```

**3. Start the SSH agent and add the key**

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

The agent stores your decrypted private key in memory — you only type the passphrase once per session.

**4. Copy the public key and upload it**

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the full output and add it to your hosting account (GitHub → Settings → SSH keys). The public key is what you upload; **the private key never leaves your machine**.

**5. Switch the repo remote to SSH**

```bash
# Inspect current remote:
git remote -v

# Switch to SSH:
git remote set-url origin git@github.com:OWNER/REPO.git
```

**6. Test the connection**

```bash
ssh -T git@github.com
# Expected: "Hi username! You've successfully authenticated..."
```

---

## Anatomy of an SSH Remote URL

```
git@github.com:OWNER/REPO.git
│   │          │     │    │
│   │          │     │    └── .git suffix (convention for clone/push URLs)
│   │          │     └─────── repository name on the hosting service
│   │          └───────────── account or organization that owns the repo
│   └──────────────────────── server hostname Git connects to via SSH
└──────────────────────────── SSH user used by the hosting service
```

| Part | Meaning | Example |
|------|---------|---------|
| `git` | SSH login user (GitHub's service user — not your username) | Always `git` on GitHub/GitLab |
| `github.com` | The remote server hostname | `gitlab.com`, `bitbucket.org` |
| `OWNER` | Your GitHub username or organization name | `alice`, `acme-inc` |
| `REPO` | The repository name on the hosting service | `my-app`, `platform-api` |

**Real examples:**

```bash
# Personal repo (username: alice, repo: demo-app)
git@github.com:alice/demo-app.git

# Organization repo (org: acme-inc, repo: platform-api)
git@github.com:acme-inc/platform-api.git
```

> **Important:** `git@` does **not** mean `git config user.name`. The `git` in the URL is the SSH service user on the server. Your real account is identified by the SSH key you uploaded — not by your local `user.name`.

---

## Multiple SSH Keys (Multiple Accounts)

The full pattern: one keypair per identity → one host alias per key → one remote URL per repo.

### Step 1 — Generate a keypair per identity

Use `-f` to set a custom filename so keys don't overwrite each other:

```bash
ssh-keygen -t ed25519 -C "work@company.com"    -f ~/.ssh/id_ed25519_work
ssh-keygen -t ed25519 -C "personal@example.com" -f ~/.ssh/id_ed25519_personal
```

This produces four files:

```
~/.ssh/id_ed25519_work          ← private key (never share)
~/.ssh/id_ed25519_work.pub      ← public key (upload to work GitHub account)
~/.ssh/id_ed25519_personal      ← private key
~/.ssh/id_ed25519_personal.pub  ← public key (upload to personal GitHub account)
```

### Step 2 — Add keys to the SSH agent

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_work
ssh-add ~/.ssh/id_ed25519_personal

# Confirm both are loaded:
ssh-add -l
```

### Step 3 — Upload each public key to the right account

```bash
cat ~/.ssh/id_ed25519_work.pub      # → paste into work GitHub account → Settings → SSH keys
cat ~/.ssh/id_ed25519_personal.pub  # → paste into personal GitHub account
```

### Step 4 — Configure `~/.ssh/config`

`Host` is a **local alias you invent**. SSH uses the `Host` value you give on the command line (or in the Git remote URL) to find the matching block and pick the right key.

```
# ~/.ssh/config
Host github-work
  HostName github.com        # real server to connect to
  User git                   # always "git" for GitHub/GitLab
  IdentityFile ~/.ssh/id_ed25519_work
  IdentitiesOnly yes         # only use this key — don't spray agent keys

Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal
  IdentitiesOnly yes
```

| Directive | What it does |
|-----------|-------------|
| `Host` | Local alias — used in remote URLs and `ssh` commands |
| `HostName` | The actual server SSH connects to |
| `User` | SSH login user on the server (always `git` for hosted Git services) |
| `IdentityFile` | Which private key to authenticate with |
| `IdentitiesOnly yes` | Don't try other agent keys — use only the listed one |

### Step 5 — Point each repo remote to the correct alias

```bash
# Work repo
git remote set-url origin git@github-work:YOUR_COMPANY/YOUR_REPO.git

# Personal repo
git remote set-url origin git@github-personal:YOUR_USERNAME/YOUR_REPO.git
```

When Git opens `git@github-work:...`, SSH looks up the `github-work` Host block and uses `id_ed25519_work`. No manual switching needed.

### Step 6 — Test each identity

```bash
ssh -T git@github-work
# → Hi work-username! You've successfully authenticated...

ssh -T git@github-personal
# → Hi personal-username! You've successfully authenticated...
```

> **Common mistake:** Do not use your GitHub username as the SSH user:
> ```bash
> ssh -T YOUR_GITHUB_USERNAME@github.com  # ❌ will fail
> ssh -T git@github-work                  # ✅ correct
> ```
> All GitHub SSH connections must use `git` as the SSH user. Your actual account is identified by the key, not the login name.

### Recommended file layout

```
~/.ssh/
  id_ed25519_work
  id_ed25519_work.pub
  id_ed25519_personal
  id_ed25519_personal.pub
  config
```

This scales cleanly — add `github-client1`, `gitlab-work`, `bitbucket-legacy` as more Host blocks without touching existing entries.

---

## What Actually Happens on `git commit` + `git push`

```
git commit
  └─▶ creates a commit object with:
       - author name/email  ← from user.name / user.email config
       - committer name/email
       - timestamp
       - parent commit(s)
       - tree snapshot

git push
  └─▶ reads remote.origin.url from .git/config
  └─▶ connects to remote via SSH (if SSH URL)
  └─▶ SSH agent presents your private key
  └─▶ server maps public key → account → checks write permission
```

---

## Minimal Setup from Zero

```bash
# 1. Set default Git identity
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global user.useConfigOnly true

# 2. Generate SSH key
ssh-keygen -t ed25519 -C "you@example.com"

# 3. Start agent and add key
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# 4. Copy public key → add to hosting account
cat ~/.ssh/id_ed25519.pub

# 5. Clone or switch remote to SSH
git clone git@github.com:OWNER/REPO.git
# or:
git remote set-url origin git@github.com:OWNER/REPO.git

# 6. Test
ssh -T git@github.com
```

---

## Common Mistakes

| Mistake | Root Cause | Fix |
|---------|-----------|-----|
| Changed SSH key, commit email still wrong | SSH auth and commit identity are separate mechanisms | Change `user.email`, not the SSH key |
| Changed `user.email`, still can't push | Push depends on SSH auth, not commit metadata | Check remote URL format and SSH key registration |
| Wrong identity appearing in one repo | Repo-local `.git/config` overrides global | Run `git config --list --show-origin` to find the override |
| SSH key works nowhere | Key not added to agent, or public key not uploaded, or remote still using HTTPS | Re-run steps 3–5 in setup above |

---

## Pitfalls

- **`user.name` is not your SSH username.** The `git` in `git@github.com` is the service's SSH user — your account is identified by the key you uploaded.
- **Local config silently overrides global.** If a repo is using the wrong email, check `.git/config` first with `git config --list --show-origin`.
- **`git config` without a flag writes to local**, not global. Always add `--global` when setting defaults.
- **HTTPS remotes ignore SSH keys entirely.** If your remote URL starts with `https://`, SSH setup has no effect — switch it with `git remote set-url`.

---

## Memory Aid

> Git has three separate identity questions:
> 1. **"Who wrote this commit?"** → `user.name` / `user.email`
> 2. **"Where is the remote?"** → `remote.origin.url`
> 3. **"Are you allowed in?"** → SSH key

---

## What to Learn Next

- [03-git-repository-and-staging.md](03-git-repository-and-staging.md) — Init, clone, and record changes
- [04-git-log.md](04-git-log.md) — Inspect commit history
