# Authentication & SSL/TLS Encryption

> **Level**: Intermediate–Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 8.4 — Authentication (pg_hba.conf) / Phase 8.5 — SSL/TLS Encryption  
> How PostgreSQL controls who can connect, which authentication method is used, and how to encrypt connections with SSL/TLS.

---

## Table of Contents

1. [Authentication Overview](#1-authentication-overview)
2. [pg_hba.conf — Host-Based Authentication](#2-pg_hbaconf--host-based-authentication)
   - [File Location](#file-location)
   - [Record Format](#record-format)
   - [Connection Types](#connection-types)
   - [Authentication Methods](#authentication-methods)
   - [Match Rules](#match-rules)
   - [Common Configurations](#common-configurations)
3. [Password Authentication](#3-password-authentication)
4. [SSL/TLS Encryption](#4-ssltls-encryption)
   - [Why SSL Matters](#why-ssl-matters)
   - [Server-Side Setup](#server-side-setup)
   - [Client sslmode Options](#client-sslmode-options)
   - [Client Certificate Authentication (mTLS)](#client-certificate-authentication-mtls)
5. [pg_ident.conf — User Name Mapping](#5-pg_identconf--user-name-mapping)
6. [Reloading Configuration](#6-reloading-configuration)
7. [Best Practices](#7-best-practices)

---

## 1. Authentication Overview

PostgreSQL authentication happens in two phases:

```
Client Connection Attempt
         │
         ▼
┌─────────────────────┐
│  Phase 1: pg_hba    │  "Can this client connect at all?"
│  (Host-Based Auth)  │  Matches connection type, database, user, IP
│                     │  Determines which auth method to use
└─────────┬───────────┘
          │ Match found
          ▼
┌─────────────────────┐
│  Phase 2: Auth      │  "Does the client prove their identity?"
│  Method Check       │  Runs the chosen method (password, cert, peer, etc.)
└─────────┬───────────┘
          │ Success
          ▼
     Connection Established
```

If no `pg_hba.conf` rule matches → connection is **rejected**.

---

## 2. pg_hba.conf — Host-Based Authentication

### File Location

```sql
-- Find from SQL
SHOW hba_file;
-- Typically: /etc/postgresql/18/main/pg_hba.conf
-- Or:        /var/lib/pgsql/18/data/pg_hba.conf
```

### Record Format

Each line in `pg_hba.conf` is a rule with this format:

```
TYPE    DATABASE    USER    ADDRESS         METHOD    [OPTIONS]
```

```
# Example pg_hba.conf
# TYPE   DATABASE   USER       ADDRESS            METHOD
local    all        all                           peer
host     all        all        127.0.0.1/32       scram-sha-256
host     all        all        ::1/128            scram-sha-256
hostssl  mydb       app_user   10.0.0.0/8         scram-sha-256
hostssl  all        all        0.0.0.0/0          scram-sha-256
```

### Connection Types

| Type | Meaning |
|------|---------|
| `local` | Unix domain socket connections (same machine) |
| `host` | TCP/IP connections (both SSL and non-SSL) |
| `hostssl` | TCP/IP connections — **only SSL** (rejects non-SSL) |
| `hostnossl` | TCP/IP connections — **only non-SSL** (rejects SSL) |
| `hostgssenc` | TCP/IP with GSSAPI encryption |
| `hostnogssenc` | TCP/IP without GSSAPI encryption |

### Authentication Methods

| Method | Security | How It Works | Best For |
|--------|:--------:|-------------|----------|
| `trust` | ❌ None | No password required — anyone matching the rule gets in | Local development only. **Never in production.** |
| `peer` | ✅ Good | OS username must match PostgreSQL role name | `local` connections (Unix socket) on the same machine |
| `ident` | ⚠️ Moderate | Like `peer` but for TCP connections; relies on ident server | Rarely used; not recommended |
| `md5` | ⚠️ Weak | MD5-hashed password challenge | Legacy systems. **Prefer scram-sha-256.** |
| `scram-sha-256` | ✅ Strong | SCRAM authentication (salted, iterated) | **Recommended for all password auth** |
| `cert` | ✅ Strong | Client must present a valid TLS certificate | Automated services, zero-password auth |
| `ldap` | ✅ Strong | Authenticates against an LDAP/AD directory | Enterprise SSO integration |
| `gss` | ✅ Strong | Kerberos/GSSAPI authentication | Enterprise SSO with Active Directory |
| `radius` | ⚠️ Moderate | RADIUS server authentication | Centralized auth with RADIUS infrastructure |
| `pam` | ✅ Good | Pluggable Authentication Modules | Linux PAM integration |
| `reject` | N/A | Always reject the connection | Explicitly block specific users/IPs |

### Match Rules

Rules are evaluated **top to bottom**. The **first matching rule** determines the auth method.

```
# ⚠️ ORDER MATTERS!

# Rule 1: Block user 'danger' from all databases
host    all    danger    0.0.0.0/0    reject

# Rule 2: Allow app_user to mydb via SSL with password
hostssl mydb   app_user  10.0.0.0/8   scram-sha-256

# Rule 3: Allow all other users with password
host    all    all       10.0.0.0/8   scram-sha-256

# If 'danger' tries to connect, Rule 1 matches first → rejected
# If 'app_user' connects to mydb over SSL → Rule 2 matches
# If 'app_user' connects to mydb without SSL → Rule 3 matches (not Rule 2)
```

**Special values:**

| Field | Special Value | Meaning |
|-------|--------------|---------|
| DATABASE | `all` | Any database |
| DATABASE | `sameuser` | Database matching the user's role name |
| DATABASE | `samerole` | Database matching any role the user is a member of |
| DATABASE | `replication` | Replication connections |
| USER | `all` | Any user |
| USER | `+grouprole` | Any member of `grouprole` |
| ADDRESS | `all` | Any IP |
| ADDRESS | `samehost` | Server's own IPs |
| ADDRESS | `samenet` | Server's subnet |

### Common Configurations

#### Development

```
# Allow everything locally (NEVER in production)
local   all   all                 trust
host    all   all   127.0.0.1/32  trust
host    all   all   ::1/128       trust
```

#### Production

```
# Local connections: OS user must match PG role
local   all   all                           peer

# Loopback: strong password
host    all   all   127.0.0.1/32            scram-sha-256
host    all   all   ::1/128                 scram-sha-256

# Application servers: SSL required, strong password
hostssl mydb  app_user  10.0.1.0/24         scram-sha-256

# Replication: SSL + strong password from specific IPs
hostssl replication  repl_user  10.0.2.0/24 scram-sha-256

# Admin: SSL + client certificate
hostssl all   admin     10.0.0.0/8          cert

# Block everything else
host    all   all   0.0.0.0/0               reject
host    all   all   ::/0                    reject
```

---

## 3. Password Authentication

### scram-sha-256 (Recommended)

The current best practice for password auth. Uses SCRAM (Salted Challenge Response Authentication Mechanism).

```sql
-- Ensure passwords are stored as SCRAM
SET password_encryption = 'scram-sha-256';   -- default in PG 14+

-- Create a role with password
CREATE ROLE app_user LOGIN PASSWORD 'strong_random_password';

-- Or change an existing password
ALTER ROLE app_user PASSWORD 'new_strong_password';
```

**How SCRAM works:**

```
Client                              Server
  │                                    │
  │── Username ──────────────────────► │
  │                                    │ looks up stored SCRAM verifier
  │◄── Salt + iteration count ──────── │
  │                                    │
  │── Proof (salted, iterated hash) ──►│
  │                                    │ verifies proof against stored verifier
  │◄── Server signature ────────────── │
  │                                    │
  │   (both sides verify each other)   │
```

- The **plaintext password never travels** over the wire.
- The server never stores the plaintext password — only a **verifier**.
- Resistant to replay attacks and eavesdropping.

### md5 (Legacy — Avoid)

```sql
-- Old style (still works but weaker)
SET password_encryption = 'md5';
CREATE ROLE old_user LOGIN PASSWORD 'password';
```

| Feature | md5 | scram-sha-256 |
|---------|-----|--------------|
| Server stores | MD5 hash | SCRAM verifier (salted + iterated) |
| Replay resistant | ❌ Partially | ✅ Yes |
| Offline brute-force resistant | ❌ Weak hash | ✅ Strong (PBKDF2-like) |
| Mutual authentication | ❌ No | ✅ Yes |

### Password Expiration

```sql
-- Password expires on a specific date
ALTER ROLE app_user VALID UNTIL '2027-01-01';

-- No expiration (default)
ALTER ROLE app_user VALID UNTIL 'infinity';

-- Check expiration
SELECT rolname, rolvaliduntil FROM pg_roles WHERE rolname = 'app_user';
```

---

## 4. SSL/TLS Encryption

### Why SSL Matters

Without SSL, all data between client and server travels in **plaintext** — including passwords, queries, and results. Anyone on the network can eavesdrop.

```
Without SSL:
Client ──── [password, queries, data in PLAINTEXT] ──── Server
                    ↑ Network attacker can read everything

With SSL:
Client ──── [encrypted tunnel] ──── Server
                    ↑ Attacker sees only encrypted bytes
```

### Server-Side Setup

#### 1. Generate Certificates

```bash
# Self-signed certificate for development/testing
openssl req -new -x509 -days 3650 -nodes \
    -out server.crt -keyout server.key \
    -subj "/CN=postgres-server"

# Set permissions (PostgreSQL refuses to start if key is world-readable)
chmod 600 server.key
chown postgres:postgres server.crt server.key
```

#### 2. Configure postgresql.conf

```ini
# Enable SSL
ssl = on
ssl_cert_file = '/etc/postgresql/server.crt'
ssl_key_file = '/etc/postgresql/server.key'

# Optional: CA certificate for verifying client certs
ssl_ca_file = '/etc/postgresql/root.crt'

# Recommended: restrict TLS versions and ciphers
ssl_min_protocol_version = 'TLSv1.3'          # PG 12+
ssl_ciphers = 'HIGH:!aNULL:!MD5'               # strong ciphers only
```

#### 3. Require SSL in pg_hba.conf

```
# Only allow SSL connections from external networks
hostssl  all  all  0.0.0.0/0  scram-sha-256
```

#### 4. Restart PostgreSQL

```bash
sudo systemctl restart postgresql
```

### Client sslmode Options

The client controls SSL behavior via the `sslmode` connection parameter:

| sslmode | Encryption | Verifies Server | Verifies Hostname | Use Case |
|---------|:----------:|:--------------:|:-----------------:|----------|
| `disable` | ❌ | ❌ | ❌ | Never use SSL (testing only) |
| `allow` | Maybe | ❌ | ❌ | Try non-SSL first, fall back to SSL |
| `prefer` (default) | Maybe | ❌ | ❌ | Try SSL first, fall back to non-SSL |
| `require` | ✅ | ❌ | ❌ | Encrypt, but don't verify server identity |
| `verify-ca` | ✅ | ✅ | ❌ | Encrypt + verify the server's certificate is signed by a trusted CA |
| `verify-full` | ✅ | ✅ | ✅ | **Encrypt + verify CA + verify hostname matches certificate** |

```bash
# Connection string examples
psql "host=db.example.com dbname=mydb user=app sslmode=verify-full sslrootcert=/path/to/root.crt"

# Environment variable
export PGSSLMODE=verify-full
export PGSSLROOTCERT=/path/to/root.crt
```

> **Production**: Always use `verify-full`. Anything less is vulnerable to MITM attacks.

### Client Certificate Authentication (mTLS)

Mutual TLS — the client also proves its identity with a certificate, eliminating passwords entirely.

#### Server Setup

```ini
# postgresql.conf
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
ssl_ca_file = 'root.crt'     # CA that signed client certificates
```

```
# pg_hba.conf — use 'cert' auth method
hostssl  all  all  0.0.0.0/0  cert  clientcert=verify-full
```

#### Client Setup

```bash
# Client needs: client cert, client key, root CA cert
psql "host=db.example.com dbname=mydb \
      sslmode=verify-full \
      sslcert=/path/to/client.crt \
      sslkey=/path/to/client.key \
      sslrootcert=/path/to/root.crt"
```

| File | Purpose |
|------|---------|
| `sslcert` | Client's certificate (proves identity to server) |
| `sslkey` | Client's private key |
| `sslrootcert` | CA certificate (verifies the server's certificate) |

> **When to use mTLS**: Automated services (CI/CD, replication, microservices) where managing passwords is harder than managing certificates.

### Checking SSL Status

```sql
-- Is the current connection using SSL?
SELECT ssl, version AS ssl_version, cipher, bits
FROM pg_stat_ssl
WHERE pid = pg_backend_pid();

-- All connections' SSL status
SELECT
    a.pid, a.usename, a.client_addr,
    s.ssl, s.version, s.cipher
FROM pg_stat_activity a
JOIN pg_stat_ssl s ON a.pid = s.pid
WHERE a.datname = current_database();
```

---

## 5. pg_ident.conf — User Name Mapping

Maps external (OS/certificate) usernames to PostgreSQL role names. Referenced from `pg_hba.conf` with the `map=` option.

```
# pg_hba.conf
local   all   all   peer   map=mymap

# pg_ident.conf
# MAPNAME    SYSTEM-USERNAME    PG-USERNAME
mymap        root               postgres
mymap        deploy             app_user
mymap        /^(.*)$            \1               # identity mapping (OS name = PG name)
```

| Field | Meaning |
|-------|---------|
| MAPNAME | Name referenced in `pg_hba.conf` |
| SYSTEM-USERNAME | OS username (or certificate CN). Supports regex with `/^...$` |
| PG-USERNAME | PostgreSQL role name |

```sql
-- Find ident file location
SHOW ident_file;
```

---

## 6. Reloading Configuration

After editing `pg_hba.conf`, `pg_ident.conf`, or SSL-related `postgresql.conf` settings:

```sql
-- From SQL (no restart needed for pg_hba and pg_ident)
SELECT pg_reload_conf();

-- From command line
sudo systemctl reload postgresql
-- or
pg_ctl reload -D /var/lib/pgsql/18/data
```

> **Note**: Changing `ssl = on/off` or SSL certificate paths requires a **full restart**, not just a reload.

```sql
-- Verify pg_hba.conf was loaded without errors
SELECT * FROM pg_hba_file_rules WHERE error IS NOT NULL;

-- View all active pg_hba rules
SELECT line_number, type, database, user_name, address, auth_method
FROM pg_hba_file_rules
ORDER BY line_number;
```

---

## 7. Best Practices

### Authentication

| Practice | Why |
|----------|-----|
| Use `scram-sha-256` for all password auth | Strongest built-in password method |
| Use `peer` for local (Unix socket) connections | No password needed; OS handles identity |
| Use `reject` as the last rule | Explicit deny instead of implicit |
| Use `hostssl` instead of `host` in production | Ensures encryption is enforced |
| Use `cert` for automated services | Eliminate password management entirely |
| Set `password_encryption = 'scram-sha-256'` globally | Ensure new passwords aren't stored as md5 |

### SSL

| Practice | Why |
|----------|-----|
| Use `sslmode=verify-full` on clients | Prevents MITM attacks |
| Set `ssl_min_protocol_version = 'TLSv1.3'` | Disable older vulnerable protocols |
| Rotate certificates before expiry | Prevent service disruptions |
| Use separate CAs for server and client certs | Limit blast radius of a compromised CA |
| Monitor `pg_stat_ssl` for unencrypted connections | Detect misconfigured clients |

### General

| Practice | Why |
|----------|-----|
| Put most specific rules first in `pg_hba.conf` | First match wins — specific before general |
| Test changes with `pg_hba_file_rules` | Catch syntax errors before reload |
| Use connection limits per role | `ALTER ROLE app_user CONNECTION LIMIT 50` — prevent connection exhaustion |
| Set `idle_in_transaction_session_timeout` | Kill abandoned connections |

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using `trust` in production | Anyone can connect as any user | Use `scram-sha-256` or `cert` |
| Ordering rules incorrectly | A broader rule matches before a specific one | Put specific rules first |
| Using `sslmode=require` and thinking it's secure | Doesn't verify the server's identity (vulnerable to MITM) | Use `sslmode=verify-full` |
| World-readable `server.key` | PostgreSQL refuses to start | `chmod 600 server.key` |
| Forgetting to reload after pg_hba changes | Old rules still active | `SELECT pg_reload_conf()` |
| Using md5 password encryption | Weak against offline brute-force | Switch to `scram-sha-256` |

---

## What to Learn Next

1. **Roles & Privileges** — Who can access which objects — see [08-roles.md](08-roles.md).
2. **Row-Level Security** — Per-row access control — see [12-policy.md](12-policy.md).
3. **Auditing** — Tracking who did what — see [22-auditing.md](22-auditing.md).
4. **Backup & Restore** — Disaster recovery strategies — Phase 9.
5. **Replication** — Streaming replication with SSL — Phase 10.

---

> *Ref: [Docs — Client Authentication](https://www.postgresql.org/docs/18/client-authentication.html) · [Docs — pg_hba.conf](https://www.postgresql.org/docs/18/auth-pg-hba-conf.html) · [Docs — SSL Support](https://www.postgresql.org/docs/18/ssl-tcp.html) · [Docs — SCRAM Authentication](https://www.postgresql.org/docs/18/sasl-authentication.html) · [Neon — PostgreSQL Administration](https://neon.com/postgresql/postgresql-administration)*
