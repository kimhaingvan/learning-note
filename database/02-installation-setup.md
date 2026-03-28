# PostgreSQL Installation & Setup

> **Level**: Beginner | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 1.1  
> Everything you need to install PostgreSQL, connect with client tools, understand the configuration files, and create your first database.

---

## Table of Contents

1. [Installing PostgreSQL](#1-installing-postgresql)
2. [The psql CLI](#2-the-psql-cli)
3. [GUI Tools — pgAdmin & DBeaver](#3-gui-tools--pgadmin--dbeaver)
4. [PostgreSQL Server Architecture](#4-postgresql-server-architecture)
5. [Configuration Files](#5-configuration-files)
6. [Creating Your First Database](#6-creating-your-first-database)
7. [Verifying the Installation](#7-verifying-the-installation)
8. [Common Pitfalls](#8-common-pitfalls)
9. [What to Learn Next](#9-what-to-learn-next)

---

## 1. Installing PostgreSQL

### Installation Methods

| Method | Platform | Best For |
|--------|----------|----------|
| `apt install postgresql` | Debian/Ubuntu | Production Linux servers |
| `dnf install postgresql-server` | Fedora/RHEL | Production Linux servers |
| `brew install postgresql@18` | macOS | Local development |
| Docker (`postgres:18`) | Any | Isolated dev environments, CI/CD |
| Cloud-managed (Neon, RDS, Supabase) | Any | Zero-ops, serverless, production |
| Official installer (EDB) | Windows/macOS | GUI-based setup |

### Installing on Ubuntu/Debian

```bash
# Add the official PostgreSQL APT repository
sudo sh -c 'echo "deb https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Import the repository signing key
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -

# Update and install
sudo apt update
sudo apt install postgresql-18
```

After installation:
- The `postgres` system user is created automatically.
- A default cluster is initialized at `/var/lib/postgresql/18/main/`.
- The `postgres` superuser role is created (matches the OS user).

### Installing on macOS (Homebrew)

```bash
brew install postgresql@18
brew services start postgresql@18
```

### Installing with Docker

```bash
docker run --name pg18 \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  -d postgres:18
```

> **Tip**: For persistent data, add `-v pgdata:/var/lib/postgresql/data`.

### Cloud-Managed (Neon)

Neon provides serverless PostgreSQL — no installation needed:
1. Sign up at https://neon.tech
2. Create a project → a database is provisioned instantly.
3. Copy the connection string and connect from `psql` or your application.

> *Ref: [Neon PostgreSQL Tutorial](https://neon.com/postgresql/tutorial)*

---

## 2. The psql CLI

`psql` is PostgreSQL's interactive terminal client. It is the most powerful and commonly used tool for working with PostgreSQL.

### Connecting

```bash
# Connect as the current OS user to the default database
psql

# Connect to a specific database as a specific user
psql -U username -d dbname -h localhost -p 5432

# Using a connection string
psql "postgresql://username:password@localhost:5432/dbname"

# Connect via Unix socket (local, no password if peer auth)
sudo -u postgres psql
```

### Essential Meta-Commands

Meta-commands start with a backslash (`\`) and are processed by `psql` itself, not sent to the server.

| Command | Description |
|---------|-------------|
| `\l` | List all databases |
| `\c dbname` | Connect to a different database |
| `\dt` | List tables in the current schema |
| `\dt+` | List tables with size and description |
| `\d tablename` | Describe a table (columns, types, constraints) |
| `\d+ tablename` | Extended description (storage, stats, description) |
| `\di` | List indexes |
| `\dv` | List views |
| `\df` | List functions |
| `\du` | List roles (users) |
| `\dn` | List schemas |
| `\ds` | List sequences |
| `\dx` | List installed extensions |
| `\timing` | Toggle query execution time display |
| `\x` | Toggle expanded (vertical) display |
| `\e` | Open query in `$EDITOR` |
| `\i filename` | Execute commands from a file |
| `\o filename` | Send query output to a file |
| `\copy` | Client-side COPY (import/export CSV) |
| `\q` | Quit psql |
| `\?` | Show all meta-commands |
| `\h COMMAND` | Show syntax help for a SQL command |

### Useful psql Settings

```bash
# In ~/.psqlrc — applies to every psql session
\set QUIET 1
\pset null '(null)'         -- Display NULL explicitly
\set HISTSIZE 10000          -- Longer command history
\set COMP_KEYWORD_CASE upper -- Auto-uppercase SQL keywords
\timing on                   -- Always show query duration
\x auto                      -- Auto-toggle expanded display
\set QUIET 0
```

### Multi-Line Queries

```
dbname=# SELECT
dbname-#   id,
dbname-#   name
dbname-# FROM users
dbname-# WHERE active = true;
```

- `=#` means psql is ready for a new command.
- `=#` changes to `-#` when a statement is not yet terminated (missing `;`).
- End every SQL statement with a **semicolon** (`;`).

---

## 3. GUI Tools — pgAdmin & DBeaver

### pgAdmin 4

The official PostgreSQL GUI administration tool.

| Feature | Detail |
|---------|--------|
| **Type** | Web-based (runs in browser) |
| **Best for** | Visual database management, query builder, explain viewer |
| **Install** | `apt install pgadmin4` or download from pgadmin.org |
| **Platforms** | Windows, macOS, Linux |

Key capabilities:
- Visual query editor with syntax highlighting.
- Graphical EXPLAIN plan viewer (very useful for understanding query plans).
- Dashboard with server activity, locks, and sessions.
- Object browser for tables, indexes, functions, triggers.

### DBeaver

A universal database tool that supports PostgreSQL and many other databases.

| Feature | Detail |
|---------|--------|
| **Type** | Desktop application (Java-based) |
| **Best for** | Multi-database workflows, ER diagrams |
| **Install** | `brew install --cask dbeaver-community` or download from dbeaver.io |
| **Platforms** | Windows, macOS, Linux |

Key capabilities:
- ER diagram generation from existing schemas.
- Data editor with inline editing and filtering.
- SQL editor with auto-completion and formatting.
- Import/export data in CSV, JSON, XML formats.

> **Recommendation**: Use `psql` for daily work and scripting. Use pgAdmin or DBeaver for visual exploration and EXPLAIN analysis.

---

## 4. PostgreSQL Server Architecture

Understanding the process model helps when troubleshooting.

```
                  ┌─────────────────────────────────┐
                  │       PostgreSQL Cluster          │
                  │  (one data directory = $PGDATA)   │
                  │                                   │
                  │  ┌─────────────────────────────┐  │
 Client ──TCP──▶  │  │  postmaster (main process)  │  │
                  │  │  Listens on port 5432        │  │
                  │  └──────────┬──────────────────┘  │
                  │             │ forks                │
                  │  ┌──────────▼──────────────────┐  │
                  │  │  Backend process (per conn)  │  │
                  │  │  Handles one client session  │  │
                  │  └─────────────────────────────┘  │
                  │                                   │
                  │  Background processes:             │
                  │  ├── checkpointer                  │
                  │  ├── background writer              │
                  │  ├── WAL writer                     │
                  │  ├── autovacuum launcher             │
                  │  ├── stats collector                 │
                  │  └── logical replication launcher    │
                  └─────────────────────────────────┘
```

### Key Concepts

- **Cluster** — A single PostgreSQL instance managing one data directory. Contains multiple databases.
- **Postmaster** — The main process that listens for connections and forks a backend process for each client.
- **Backend** — One per connection. Executes SQL, manages transactions, accesses shared buffers.
- **Background Workers** — Perform maintenance tasks (VACUUM, checkpoints, WAL flushing) without client interaction.

> **Key insight**: Each connection = one OS process. This is why connection limits matter and why you need a connection pooler (PgBouncer) at scale.

---

## 5. Configuration Files

PostgreSQL is configured through three main files, typically located in the data directory (`$PGDATA`) or `/etc/postgresql/18/main/` on Debian/Ubuntu.

### 5.1 postgresql.conf — Server Settings

The main configuration file controlling all server behavior.

```
# Location
/etc/postgresql/18/main/postgresql.conf   # Debian/Ubuntu
$PGDATA/postgresql.conf                     # Default

# Common settings
listen_addresses = 'localhost'   # Which interfaces to listen on ('*' for all)
port = 5432                      # TCP port
max_connections = 100            # Maximum concurrent connections
shared_buffers = 256MB           # Main memory cache (typically 25% of RAM)
work_mem = 4MB                   # Memory per sort/hash operation
wal_level = replica              # WAL detail level (minimal, replica, logical)
log_statement = 'ddl'            # What to log (none, ddl, mod, all)
```

**How to apply changes**:
```sql
-- Most settings: reload is enough
SELECT pg_reload_conf();
-- Or from shell: pg_ctl reload

-- Some settings require a restart (e.g., shared_buffers, max_connections)
-- Check: SELECT name, pending_restart FROM pg_settings WHERE pending_restart;
```

### 5.2 pg_hba.conf — Client Authentication

Controls **who can connect**, from **where**, and **how** they authenticate.

```
# TYPE    DATABASE    USER        ADDRESS             METHOD
local     all         all                             peer
host      all         all         127.0.0.1/32        scram-sha-256
host      all         all         ::1/128             scram-sha-256
host      mydb        appuser     192.168.1.0/24      scram-sha-256
```

| Field | Meaning |
|-------|---------|
| `TYPE` | `local` (Unix socket), `host` (TCP/IP), `hostssl` (SSL only) |
| `DATABASE` | Database name, `all`, or `sameuser` |
| `USER` | Role name, `all`, or a group with `+groupname` |
| `ADDRESS` | IP range in CIDR notation (for `host` types) |
| `METHOD` | `peer`, `scram-sha-256`, `md5`, `trust`, `cert`, `ldap` |

**Rules are evaluated top-to-bottom** — first match wins.

> **Security warning**: Never use `trust` method in production. Always use `scram-sha-256`.

### 5.3 pg_ident.conf — User Name Mapping

Maps operating system user names to PostgreSQL role names.

```
# MAPNAME    SYSTEM-USERNAME    PG-USERNAME
mymap        os_alice           pg_alice
mymap        os_bob             pg_bob
```

Used when `pg_hba.conf` specifies `map=mymap` with `peer` or `ident` authentication.

### Viewing Active Settings

```sql
-- Show a specific setting
SHOW shared_buffers;
SHOW max_connections;

-- Show all settings with source information
SELECT name, setting, unit, source
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'max_connections');

-- Show settings that differ from defaults
SELECT name, setting, boot_val, source
FROM pg_settings
WHERE setting != boot_val;
```

---

## 6. Creating Your First Database

> **Deep dive**: See [03-managing-databases.md](03-managing-databases.md) for full coverage of template databases, tablespaces, per-database configuration, and database vs schema decisions.

### Quick Start

```sql
-- Connect as the postgres superuser
sudo -u postgres psql

-- Create a role for yourself
CREATE ROLE myuser WITH LOGIN PASSWORD 'mypassword';

-- Create a database owned by your role
CREATE DATABASE mydb OWNER myuser;

-- Connect to the new database
\c mydb myuser
```

### From the Command Line

```bash
# Create a database (wrapper around CREATE DATABASE)
createdb -U postgres mydb

# Create a role
createuser -U postgres --login --pwprompt myuser

# Connect
psql -U myuser -d mydb -h localhost
```

### Verify

```sql
-- List all databases
\l

-- Show current connection info
\conninfo

-- Check PostgreSQL version
SELECT version();
```

---

## 7. Verifying the Installation

A quick checklist to confirm everything works:

```bash
# 1. Check PostgreSQL service is running
sudo systemctl status postgresql
# or
pg_isready

# 2. Check the version
psql --version
# psql (PostgreSQL) 18.x

# 3. Connect and run a test query
sudo -u postgres psql -c "SELECT 1 AS test;"

# 4. Check the data directory
sudo -u postgres psql -c "SHOW data_directory;"
# /var/lib/postgresql/18/main

# 5. Check listening address and port
sudo -u postgres psql -c "SHOW listen_addresses;"
sudo -u postgres psql -c "SHOW port;"

# 6. Check configuration file locations
sudo -u postgres psql -c "SHOW config_file;"
sudo -u postgres psql -c "SHOW hba_file;"
```

---

## 8. Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| `psql: FATAL: role "myuser" does not exist` | Create the role first: `CREATE ROLE myuser WITH LOGIN PASSWORD '...'` |
| `psql: FATAL: Peer authentication failed` | Use `-h localhost` to force TCP (password auth) instead of Unix socket (peer) |
| `psql: could not connect to server: Connection refused` | Check `listen_addresses` in `postgresql.conf` and ensure the service is running |
| `FATAL: no pg_hba.conf entry for host` | Add a matching line in `pg_hba.conf` and `pg_ctl reload` |
| `CREATE DATABASE` inside a transaction | Not allowed — run outside `BEGIN` / `COMMIT` |
| Forgot to set a password | `ALTER ROLE myuser PASSWORD 'newpassword';` |
| Can't connect remotely | Set `listen_addresses = '*'` and add a `host` entry in `pg_hba.conf` |

---

## 9. What to Learn Next

1. **SQL Fundamentals** — `SELECT`, `INSERT`, `UPDATE`, `DELETE` — see [14-sql-fundamentals.md](14-sql-fundamentals.md).
2. **Data Types** — PostgreSQL's rich type system — see [15-data-types.md](15-data-types.md).
3. **Managing Databases** — Templates, tablespaces, schemas — see [03-managing-databases.md](03-managing-databases.md).
4. **Roles & Security** — Authentication and authorization — see [08-roles.md](08-roles.md).

---

> *Ref: [Neon Tutorial — Getting Started](https://neon.com/postgresql/tutorial) · [Docs — Server Setup and Operation](https://www.postgresql.org/docs/18/runtime.html) · [Docs — psql](https://www.postgresql.org/docs/18/app-psql.html)*
