# Connection Pooling & Load Balancing

> **Level**: Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 10.4 — Connection Pooling (PgBouncer) / Phase 10.5 — Load Balancing  
> Managing PostgreSQL connections at scale — connection pooling with PgBouncer, load balancing reads across replicas, and routing patterns.

---

## Table of Contents

1. [Why Connection Pooling?](#1-why-connection-pooling)
2. [PgBouncer](#2-pgbouncer)
   - [Architecture](#architecture)
   - [Pool Modes](#pool-modes)
   - [Configuration](#configuration)
   - [Authentication Setup](#authentication-setup)
   - [Monitoring & Administration](#monitoring--administration)
   - [Transaction Pooling Caveats](#transaction-pooling-caveats)
3. [Alternative Poolers](#3-alternative-poolers)
4. [Load Balancing](#4-load-balancing)
   - [Application-Level Routing](#application-level-routing)
   - [HAProxy for PostgreSQL](#haproxy-for-postgresql)
   - [PgPool-II](#pgpool-ii)
   - [Read/Write Splitting Patterns](#readwrite-splitting-patterns)
5. [Production Architecture Patterns](#5-production-architecture-patterns)

---

## 1. Why Connection Pooling?

Every PostgreSQL connection spawns a **dedicated backend process** (~5-10 MB RAM each). Without pooling:

```
100 app servers × 10 connections each = 1,000 PostgreSQL backends
Each backend: ~10 MB → 10 GB RAM just for connections
Plus: process fork overhead, context switching, shared buffer contention
```

| Problem | Impact |
|---------|--------|
| Backend process per connection | High memory consumption |
| Fork overhead | Slow connection creation (~5-10 ms) |
| `max_connections` limit | Hard cap (default 100, max practical ~500-1000) |
| Connection storms | App restarts/deploys open hundreds of connections at once |
| Idle connections | Waste memory and slots doing nothing |

**Connection pooling** sits between your application and PostgreSQL, maintaining a smaller pool of reusable connections:

```
App Servers (1,000 connections)          PostgreSQL (50 backends)
┌─────┐                                  ┌──────────┐
│ App │──┐                          ┌──► │ Backend  │
│ App │──┤   ┌──────────────┐       │    │ Backend  │
│ App │──┼──►│  PgBouncer   │───────┤    │ Backend  │
│ App │──┤   │ (50 pool)    │───────┤    │ ...      │
│ App │──┘   └──────────────┘       └──► │ Backend  │
└─────┘                                  └──────────┘
       1000:50 multiplexing ratio
```

---

## 2. PgBouncer

PgBouncer is the most widely deployed PostgreSQL connection pooler — lightweight, single-threaded, and handles thousands of client connections with minimal resource usage.

### Architecture

```
                PgBouncer Process (~2-5 MB RAM)
                ┌─────────────────────────────────┐
                │  Client connections  (listen)    │
Client ────────►│  ┌───────────────────────────┐  │
Client ────────►│  │ Connection Queue           │  │── Server connections ──► PostgreSQL
Client ────────►│  │ (waiting for a pool slot)  │  │       (pool_size)
Client ────────►│  └───────────────────────────┘  │
                │  Stats collector               │
                └─────────────────────────────────┘
```

| Property | Value |
|----------|-------|
| Memory footprint | ~2 KB per connection |
| Protocol | PostgreSQL wire protocol (transparent to apps) |
| Concurrency model | Single-threaded, event-driven (libevent) |
| Scalability | 10,000+ client connections easily |

### Pool Modes

| Mode | Connection returned to pool after… | Best For | Limitations |
|------|-----------------------------------|----------|------------|
| **Session** | Client disconnects | Legacy apps, session-level features | Low multiplexing — almost no benefit |
| **Transaction** | Each transaction completes | ✅ Most applications | No session state across transactions |
| **Statement** | Each statement completes | Simple autocommit workloads | No multi-statement transactions |

#### Session Pooling

```
Client A: BEGIN → query → query → COMMIT → query → query → DISCONNECT
          └────────── same server connection the entire session ──────────┘
```

#### Transaction Pooling (Recommended)

```
Client A: BEGIN → query → COMMIT    BEGIN → query → COMMIT
          └── server conn 1 ──┘     └── server conn 2 ──┘   (may differ!)

Client B:                       ↑ uses server conn 1 while A is idle
```

#### Statement Pooling

```
Client A: SELECT ...  INSERT ...  SELECT ...
          └─ conn 1 ┘ └─ conn 2 ┘ └─ conn 3 ┘   (each statement, different conn)
```

### Configuration

#### `pgbouncer.ini`

```ini
[databases]
# database = connection string to actual PostgreSQL
mydb = host=127.0.0.1 port=5432 dbname=mydb

# Wildcard — any database name forwards to same-named database on server
* = host=127.0.0.1 port=5432

# Read replica pool
mydb_ro = host=replica1 port=5432 dbname=mydb

[pgbouncer]
# Listen address and port
listen_addr = 0.0.0.0
listen_port = 6432

# Authentication
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
# Or delegate to PostgreSQL:
# auth_type = hba
# auth_hba_file = /etc/pgbouncer/pg_hba.conf
# auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1

# Pool mode
pool_mode = transaction

# Pool sizing
default_pool_size = 20          # server connections per user/database pair
min_pool_size = 5               # keep at least this many server connections open
reserve_pool_size = 5           # extra connections for burst traffic
reserve_pool_timeout = 3        # seconds before using reserve pool

# Connection limits
max_client_conn = 1000          # max client connections
max_db_connections = 50         # max server connections per database
max_user_connections = 40       # max server connections per user

# Timeouts
server_idle_timeout = 600       # close idle server connection after 10 min
client_idle_timeout = 0         # 0 = disabled (don't close idle clients)
client_login_timeout = 60       # timeout for initial client authentication
query_timeout = 0               # 0 = disabled; max time for a query
query_wait_timeout = 120        # max time a client waits for a server connection

# Logging
log_connections = 1
log_disconnections = 1
log_pooler_errors = 1
stats_period = 60               # log stats every 60 seconds

# TLS (optional)
; client_tls_sslmode = require
; client_tls_key_file = /etc/pgbouncer/server.key
; client_tls_cert_file = /etc/pgbouncer/server.crt
```

#### Key Sizing Formula

```
default_pool_size ≥ (max concurrent transactions) / (number of databases × users)

# Example: 200 peak concurrent transactions, 1 database, 1 app user
default_pool_size = 200 / (1 × 1) = 200
# But PostgreSQL max_connections = 200, so:
default_pool_size = 150  (leave room for admin connections)
```

### Authentication Setup

#### `userlist.txt`

```txt
"app_user" "SCRAM-SHA-256$4096:salt$StoredKey:ServerKey"
"admin_user" "SCRAM-SHA-256$4096:salt$StoredKey:ServerKey"
```

Generate password hashes:

```bash
# Get hash from PostgreSQL
psql -c "SELECT rolname, rolpassword FROM pg_authid WHERE rolname = 'app_user';"
# Copy the SCRAM hash into userlist.txt
```

#### Auth Passthrough (Recommended for Large Deployments)

Instead of maintaining `userlist.txt`, delegate authentication to PostgreSQL:

```ini
# pgbouncer.ini
auth_type = scram-sha-256
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1
auth_user = pgbouncer_auth    # user that runs the auth_query
```

```sql
-- On PostgreSQL: create the auth lookup user
CREATE ROLE pgbouncer_auth LOGIN PASSWORD '...';
GRANT SELECT ON pg_shadow TO pgbouncer_auth;
```

### Monitoring & Administration

Connect to PgBouncer's admin console:

```bash
psql -h 127.0.0.1 -p 6432 -U admin pgbouncer
```

```sql
-- Pool status
SHOW POOLS;
-- Columns: database, user, cl_active, cl_waiting, sv_active, sv_idle, pool_mode

-- Client connections
SHOW CLIENTS;

-- Server connections
SHOW SERVERS;

-- Statistics
SHOW STATS;
-- total_xact_count, total_query_count, avg_xact_time, avg_query_time

-- Detailed per-second stats
SHOW STATS_AVERAGES;

-- Configuration
SHOW CONFIG;

-- Reload config without restart
RELOAD;

-- Pause/resume a database (for maintenance)
PAUSE mydb;
RESUME mydb;

-- Graceful shutdown
SHUTDOWN;
```

Key metrics to monitor:

| Metric | Where | Alert When |
|--------|-------|-----------|
| `cl_waiting` | `SHOW POOLS` | > 0 for sustained periods (clients waiting for server connection) |
| `sv_active` vs `pool_size` | `SHOW POOLS` | `sv_active` consistently = `pool_size` (pool exhausted) |
| `avg_xact_time` | `SHOW STATS` | Increasing over time (long transactions hogging connections) |
| `total_xact_count` | `SHOW STATS` | Dropping (potential connectivity issue) |

### Transaction Pooling Caveats

When using `pool_mode = transaction`, each transaction may get a **different server connection**. Session-level state is **lost between transactions**.

| Feature | Works? | Workaround |
|---------|:------:|-----------|
| **Prepared statements** | ❌ | Use `DEALLOCATE ALL` or set `server_reset_query = DEALLOCATE ALL; DISCARD ALL` |
| **SET / session variables** | ❌ | Use `SET LOCAL` inside transactions, or `server_reset_query` |
| **LISTEN/NOTIFY** | ❌ | Use session pooling for notification listeners |
| **Temporary tables** | ❌ | Create and use within a single transaction |
| **Advisory locks** | ❌ (session-level) | Use `pg_advisory_xact_lock()` (transaction-level) instead |
| **Cursors (WITH HOLD)** | ❌ | Use cursors only within transactions |
| **`search_path` changes** | ❌ | Set in connection string or use `SET LOCAL` |

```ini
# Recommended server_reset_query for transaction mode
server_reset_query = DISCARD ALL
# Resets: prepared statements, temp tables, session variables, etc.
```

---

## 3. Alternative Poolers

| Pooler | Architecture | Strengths | Weaknesses |
|--------|-------------|-----------|------------|
| **PgBouncer** | Single process, event-driven | Ultra-lightweight, battle-tested | Single-threaded, no built-in HA |
| **PgPool-II** | Multi-process | Load balancing, query cache, watchdog HA | Complex config, heavier resource use |
| **Odyssey** (Yandex) | Multi-threaded | Better multi-core utilization | Less widely adopted |
| **pgcat** (Supabase) | Multi-threaded, Rust | Sharding, load balancing built-in | Newer, smaller community |
| **Connection pooling in app** (HikariCP, pgx pool) | In-process | No extra infrastructure | Per-app pool, no global multiplexing |

---

## 4. Load Balancing

### Application-Level Routing

The simplest approach — use separate connection strings for read and write operations.

```python
# Application code (Python / psycopg example)
import psycopg

# Write connection (primary)
write_conn = psycopg.connect("host=primary port=5432 dbname=mydb")

# Read connection (replica, via PgBouncer)
read_conn = psycopg.connect("host=pgbouncer port=6432 dbname=mydb_ro")
```

#### `target_session_attrs` (Built-in libpq Feature)

PostgreSQL's libpq can automatically select the right server:

```python
# Connect to any available read-write server
conn = psycopg.connect(
    "host=node1,node2,node3 port=5432,5432,5432 "
    "dbname=mydb target_session_attrs=read-write"
)

# Connect to any available read-only server (standby)
read_conn = psycopg.connect(
    "host=node1,node2,node3 port=5432,5432,5432 "
    "dbname=mydb target_session_attrs=standby"
)
```

| `target_session_attrs` Value | Connects To |
|------------------------------|-------------|
| `any` | First available host |
| `read-write` | Primary (read-write) only |
| `read-only` | Any server in read-only mode |
| `primary` | Primary server |
| `standby` | Standby (hot standby) server |
| `prefer-read` | Standby if available, otherwise primary |
| `prefer-standby` | Standby if available, otherwise primary |

### HAProxy for PostgreSQL

HAProxy provides TCP-level load balancing with health checks.

```
Clients
   │
   ├──► HAProxy :5000 (read-write) ──► Primary
   │
   └──► HAProxy :5001 (read-only)  ──► Standby 1
                                    ──► Standby 2  (round-robin)
```

```haproxy
# /etc/haproxy/haproxy.cfg

global
    maxconn 1000

defaults
    mode tcp
    timeout connect 5s
    timeout client 30m
    timeout server 30m

# Frontend for writes — single primary
frontend pg_write
    bind *:5000
    default_backend pg_primary

# Frontend for reads — round-robin across standbys
frontend pg_read
    bind *:5001
    default_backend pg_standbys

# Primary backend (single server)
backend pg_primary
    option httpchk GET /primary    # Patroni REST API health check
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.0.0.1:5432 check port 8008
    server node2 10.0.0.2:5432 check port 8008
    server node3 10.0.0.3:5432 check port 8008

# Standby backend (round-robin read replicas)
backend pg_standbys
    option httpchk GET /replica    # Patroni REST API health check
    http-check expect status 200
    balance roundrobin
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 10.0.0.1:5432 check port 8008
    server node2 10.0.0.2:5432 check port 8008
    server node3 10.0.0.3:5432 check port 8008
```

> HAProxy works particularly well with **Patroni**, which exposes REST endpoints (`/primary`, `/replica`, `/health`) for health checking.

### PgPool-II

PgPool-II is a middleware that provides connection pooling, load balancing, query routing, and HA in one package.

```
Client → PgPool-II → Primary (writes)
                   → Standby 1 (reads)
                   → Standby 2 (reads)
```

| Feature | PgPool-II | PgBouncer + HAProxy |
|---------|:---------:|:-------------------:|
| Connection pooling | ✅ | ✅ (PgBouncer) |
| Load balancing | ✅ Built-in | ✅ (HAProxy) |
| Query-level read/write split | ✅ Automatic | ❌ App-level |
| Query caching | ✅ In-memory cache | ❌ |
| Watchdog (HA for pooler) | ✅ Built-in | Separate (keepalived/VIP) |
| Resource usage | Higher | Lower |
| Complexity | Higher | Lower per component |

```ini
# pgpool.conf key settings
backend_hostname0 = 'primary'
backend_port0 = 5432
backend_weight0 = 0           # no reads to primary
backend_flag0 = 'ALWAYS_PRIMARY'

backend_hostname1 = 'standby1'
backend_port1 = 5432
backend_weight1 = 1           # reads go here

backend_hostname2 = 'standby2'
backend_port2 = 5432
backend_weight2 = 1           # reads go here

# Load balancing
load_balance_mode = on
statement_level_load_balance_mode = on

# Streaming replication check
sr_check_user = 'pgpool_check'
sr_check_period = 10
```

### Read/Write Splitting Patterns

| Pattern | How | Pros | Cons |
|---------|-----|------|------|
| **Application-level** | Separate connection strings for read/write | Full control, no middleware | Code changes, developer discipline |
| **`target_session_attrs`** | libpq auto-routing | No middleware needed | Connection-level only, not query-level |
| **PgPool-II** | Automatic SQL parsing | Transparent to app | Heavy, complex, occasional parsing errors |
| **HAProxy + Patroni** | Port-based routing (5000=write, 5001=read) | Simple, proven, lightweight | App must use correct port |
| **ProxySQL (experimental)** | SQL-aware proxy | Query-level routing | PostgreSQL support is newer |

#### Application-Level Split Example (Go)

```go
import (
    "github.com/jackc/pgx/v5/pgxpool"
)

// Create separate pools for read and write
writePool, _ := pgxpool.New(ctx,
    "host=primary port=5432 dbname=mydb pool_max_conns=50")

readPool, _ := pgxpool.New(ctx,
    "host=standby1,standby2 port=5432,5432 dbname=mydb "+
    "target_session_attrs=standby pool_max_conns=100")

// Route queries
func GetOrder(ctx context.Context, id int) (*Order, error) {
    row := readPool.QueryRow(ctx, "SELECT * FROM orders WHERE id = $1", id)
    // ...
}

func CreateOrder(ctx context.Context, o *Order) error {
    _, err := writePool.Exec(ctx, "INSERT INTO orders ...", o.Fields()...)
    return err
}
```

---

## 5. Production Architecture Patterns

### Small Scale (< 100 connections)

```
App → PostgreSQL (no pooler needed)
```

- Direct connections are fine
- Set `max_connections = 200`

### Medium Scale (100-1,000 connections)

```
App Servers → PgBouncer → Primary
                        → Standby (reads)
```

- PgBouncer in transaction mode
- Separate PgBouncer instances or database entries for read/write

### Large Scale (1,000+ connections) with HA

```
                    ┌──────────────────────────────┐
                    │         HAProxy VIP           │
                    │  :5000 (write)  :5001 (read)  │
                    └───────┬──────────┬────────────┘
                            │          │
                    ┌───────▼──────────▼────────────┐
                    │     PgBouncer (per node)       │
                    └───────┬──────────┬────────────┘
                            │          │
               ┌────────────┼──────────┼────────────┐
               ▼            ▼          ▼            ▼
          ┌─────────┐  ┌─────────┐ ┌─────────┐
          │ Patroni  │  │ Patroni │ │ Patroni │
          │ Primary  │  │ Standby │ │ Standby │
          └─────────┘  └─────────┘ └─────────┘
                            │
                    ┌───────▼───────┐
                    │   etcd / DCS  │
                    └───────────────┘
```

Typical stack:
- **Patroni** — HA and automatic failover
- **PgBouncer** — Connection pooling (co-located with each PG node or centralized)
- **HAProxy** — TCP load balancing with Patroni health checks
- **etcd** — Distributed consensus for leader election

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| No connection pooler with 500+ connections | Memory exhaustion, performance degradation | Deploy PgBouncer |
| Transaction pooling with `LISTEN/NOTIFY` | Notifications delivered to random clients | Use session pooling for notification listeners |
| `DISCARD ALL` not in `server_reset_query` | Session state leaks between clients | Set `server_reset_query = DISCARD ALL` |
| PgBouncer `default_pool_size` too high | Exceeds PostgreSQL `max_connections` | `pool_size × db_count × user_count < max_connections` |
| Using PgPool-II just for pooling | Overkill — resource heavy, complex | Use PgBouncer for pooling; add HAProxy for LB |
| Ignoring `cl_waiting` metric | Clients silently queuing → timeouts | Alert when `cl_waiting > 0` persists |
| App reconnection storms after failover | PgBouncer overwhelmed | Set `min_pool_size`, use `PAUSE`/`RESUME` during failovers |
| No health checks in HAProxy | Sends traffic to dead nodes | Use Patroni REST endpoints for health checks |

---

## What to Learn Next

1. **Replication & HA** — Streaming and logical replication that feeds these read replicas — see [23-replication-high-availability.md](23-replication-high-availability.md).
2. **Authentication & SSL** — Securing connections between pooler and PostgreSQL — see [21-authentication-ssl.md](21-authentication-ssl.md).
3. **Performance Tuning** — Server-level tuning that complements connection pooling — Phase 11.
4. **Monitoring** — Observability for the full stack — Phase 11.

---

> *Ref: [PgBouncer Docs](https://www.pgbouncer.org/) · [PgPool-II Docs](https://www.pgpool.net/) · [Docs — libpq Connection Strings](https://www.postgresql.org/docs/18/libpq-connect.html#LIBPQ-PARAMKEYWORDS) · [HAProxy Docs](https://www.haproxy.com/documentation/) · [Patroni Docs](https://patroni.readthedocs.io/)*
