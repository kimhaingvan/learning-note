# Performance Tuning — Configuration, Memory & VACUUM

> **Level**: Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 11.1 — Configuration Tuning / Phase 11.2 — Memory Architecture / Phase 11.3 — VACUUM & Autovacuum  
> Tuning PostgreSQL for optimal performance — key `postgresql.conf` parameters, memory architecture, and the VACUUM system that keeps PostgreSQL healthy.

---

## Table of Contents

1. [Configuration Tuning (postgresql.conf)](#1-configuration-tuning-postgresqlconf)
   - [Memory Parameters](#memory-parameters)
   - [WAL & Checkpoint Parameters](#wal--checkpoint-parameters)
   - [Planner Parameters](#planner-parameters)
   - [Connection Parameters](#connection-parameters)
   - [Parallel Query Parameters](#parallel-query-parameters)
   - [Quick-Start Tuning Recipe](#quick-start-tuning-recipe)
2. [Memory Architecture](#2-memory-architecture)
   - [Shared Memory](#shared-memory)
   - [Per-Backend Memory](#per-backend-memory)
   - [OS Page Cache](#os-page-cache)
   - [Huge Pages](#huge-pages)
   - [Memory Budget Planning](#memory-budget-planning)
3. [VACUUM & Autovacuum](#3-vacuum--autovacuum)
   - [Why VACUUM Exists](#why-vacuum-exists)
   - [VACUUM Variants](#vacuum-variants)
   - [Autovacuum](#autovacuum)
   - [Tuning Autovacuum](#tuning-autovacuum)
   - [Transaction ID Wraparound](#transaction-id-wraparound)
   - [Monitoring VACUUM](#monitoring-vacuum)

---

## 1. Configuration Tuning (postgresql.conf)

### Memory Parameters

| Parameter | Default | Recommended | Notes |
|-----------|---------|-------------|-------|
| `shared_buffers` | 128 MB | **25% of RAM** | PostgreSQL's main in-memory cache. Sweet spot is 25% — beyond that, OS page cache is more effective |
| `effective_cache_size` | 4 GB | **50-75% of RAM** | Not an allocation — a hint to the planner about total available cache (PG + OS). Affects index scan decisions |
| `work_mem` | 4 MB | **4-64 MB** | Per-operation (not per-connection) memory for sorts, hashes, and joins. Be cautious: a single complex query may use multiple `work_mem` allocations |
| `maintenance_work_mem` | 64 MB | **256 MB – 2 GB** | Used by VACUUM, CREATE INDEX, ALTER TABLE ADD FK. Can safely be large since few concurrent maintenance operations |
| `temp_buffers` | 8 MB | 8-32 MB | Per-session temporary table cache |

#### `work_mem` Deep Dive

```
work_mem is per SORT/HASH OPERATION, not per query or connection!

Example query with 3 sort operations:
  SELECT ... FROM a JOIN b ... ORDER BY ... GROUP BY ...
  Each sort/hash can use up to work_mem

Worst case memory for a single query:
  work_mem × (number of sort/hash nodes in plan)

Total memory risk:
  work_mem × sort_nodes × max_connections
  = 32 MB × 3 × 200 = 19.2 GB  ← can be dangerous!
```

**Strategy**: Keep `work_mem` modest globally, increase per-session for analytics:

```sql
-- Global: conservative
ALTER SYSTEM SET work_mem = '16MB';

-- Per-session: generous for reporting queries
SET work_mem = '256MB';
-- Or per-transaction:
SET LOCAL work_mem = '256MB';
```

### WAL & Checkpoint Parameters

| Parameter | Default | Recommended | Notes |
|-----------|---------|-------------|-------|
| `wal_buffers` | -1 (auto) | **Auto** (~3% of shared_buffers, max 64 MB) | Usually fine at auto |
| `min_wal_size` | 80 MB | 1-2 GB | Minimum WAL size before recycling |
| `max_wal_size` | 1 GB | **4-8 GB** | Maximum WAL before forced checkpoint. Larger = less frequent checkpoints |
| `checkpoint_completion_target` | 0.9 | **0.9** | Spread checkpoint writes over 90% of the checkpoint interval. Less I/O spikes |
| `checkpoint_timeout` | 5 min | **10-15 min** | Time between checkpoints. Longer reduces I/O, but increases recovery time |
| `wal_compression` | off | **on** (PG 15+: `lz4` or `zstd`) | Compress WAL. Reduces I/O at slight CPU cost |

```
Checkpoint I/O spike without tuning:
    ████████████████████▓░░░░░░░░░░░░░░░░░░░  (big burst)
         checkpoint          idle

With checkpoint_completion_target = 0.9:
    ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░  (spread evenly)
         writes spread over 90% of interval
```

### Planner Parameters

| Parameter | Default | SSD Value | HDD Value | Notes |
|-----------|---------|-----------|-----------|-------|
| `random_page_cost` | 4.0 | **1.1–1.5** | 4.0 | Relative cost of random I/O. Lower → planner prefers index scans |
| `seq_page_cost` | 1.0 | 1.0 | 1.0 | Baseline cost for sequential I/O |
| `effective_io_concurrency` | 1 | **200** | 2-4 | Hint for bitmap heap scans — how many I/O requests the OS can handle concurrently |
| `maintenance_io_concurrency` | 10 | **200** | 10 | Same but for maintenance operations (VACUUM, CREATE INDEX) |
| `cpu_tuple_cost` | 0.01 | 0.01 | 0.01 | Rarely needs changing |
| `default_statistics_target` | 100 | 100-500 | 100-500 | Higher = more accurate stats but slower ANALYZE. Increase for columns with skewed distributions |

```sql
-- Set SSD-optimized planner settings
ALTER SYSTEM SET random_page_cost = 1.1;
ALTER SYSTEM SET effective_io_concurrency = 200;
ALTER SYSTEM SET maintenance_io_concurrency = 200;
SELECT pg_reload_conf();
```

### Connection Parameters

| Parameter | Default | Recommended | Notes |
|-----------|---------|-------------|-------|
| `max_connections` | 100 | **50-200** (use pooler) | Each connection = process + memory. Lower is better |
| `superuser_reserved_connections` | 3 | 3-5 | Reserve slots for admin access |
| `idle_in_transaction_session_timeout` | 0 | **30s – 5min** | Kill sessions left idle in a transaction |
| `statement_timeout` | 0 | **30s – 5min** (per-role) | Kill queries running too long |

```sql
-- Protect against idle-in-transaction sessions
ALTER SYSTEM SET idle_in_transaction_session_timeout = '60s';

-- Set statement timeout per role (not globally)
ALTER ROLE web_app SET statement_timeout = '30s';
ALTER ROLE analytics SET statement_timeout = '300s';  -- 5 min for reports
```

### Parallel Query Parameters

| Parameter | Default | Notes |
|-----------|---------|-------|
| `max_parallel_workers_per_gather` | 2 | Max workers per plan node. 2-4 is reasonable |
| `max_parallel_workers` | 8 | Total parallel workers across all queries |
| `max_parallel_maintenance_workers` | 2 | Workers for `CREATE INDEX`, `VACUUM` |
| `parallel_tuple_cost` | 0.1 | Cost of passing a tuple to the leader |
| `parallel_setup_cost` | 1000 | Startup cost of launching workers |
| `min_parallel_table_scan_size` | 8 MB | Minimum table size before parallel scan is considered |
| `min_parallel_index_scan_size` | 512 kB | Minimum index size before parallel scan |

```sql
-- Increase parallelism for large analytical queries
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET max_parallel_workers = 16;
ALTER SYSTEM SET max_parallel_maintenance_workers = 4;
SELECT pg_reload_conf();
```

### Quick-Start Tuning Recipe

For a dedicated PostgreSQL server with **32 GB RAM, SSD storage, 4 cores**:

```ini
# Memory
shared_buffers = '8GB'              # 25% of RAM
effective_cache_size = '24GB'       # 75% of RAM
work_mem = '32MB'                   # conservative; increase per-session
maintenance_work_mem = '1GB'        # generous for VACUUM/INDEX
temp_buffers = '32MB'

# WAL
wal_buffers = '64MB'
min_wal_size = '2GB'
max_wal_size = '8GB'
checkpoint_timeout = '15min'
checkpoint_completion_target = 0.9
wal_compression = lz4

# Planner (SSD)
random_page_cost = 1.1
effective_io_concurrency = 200
maintenance_io_concurrency = 200

# Connections
max_connections = 200               # use PgBouncer in front
idle_in_transaction_session_timeout = '60s'

# Parallel
max_parallel_workers_per_gather = 2
max_parallel_workers = 8
max_parallel_maintenance_workers = 2

# Logging
log_min_duration_statement = '250ms'    # log queries > 250ms
log_checkpoints = on
log_lock_waits = on
```

> **Tool**: Use [PGTune](https://pgtune.leopard.in.ua) to generate initial settings based on your hardware, then adjust based on workload.

---

## 2. Memory Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     PostgreSQL Process                        │
│                                                              │
│  ┌───────────────── Shared Memory ──────────────────────┐    │
│  │  ┌──────────────┐  ┌──────────┐  ┌───────────────┐  │    │
│  │  │ Shared        │  │ WAL      │  │ Lock Table    │  │    │
│  │  │ Buffers       │  │ Buffers  │  │               │  │    │
│  │  │ (shared_      │  │ (wal_    │  │               │  │    │
│  │  │  buffers)     │  │  buffers)│  │               │  │    │
│  │  └──────────────┘  └──────────┘  └───────────────┘  │    │
│  │  ┌──────────────┐  ┌──────────────────────────────┐  │    │
│  │  │ CLOG         │  │ Other: visibility map,       │  │    │
│  │  │ (pg_xact)    │  │ free space map, etc.         │  │    │
│  │  └──────────────┘  └──────────────────────────────┘  │    │
│  └──────────────────────────────────────────────────────┘    │
│                                                              │
│  ┌────── Per-Backend (× max_connections) ───────────┐        │
│  │  work_mem (per sort/hash), temp_buffers,          │        │
│  │  catalog cache, plan cache, maintenance_work_mem  │        │
│  └───────────────────────────────────────────────────┘        │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│                  Operating System                             │
│  ┌──────────────────────────────────────────────────────┐    │
│  │            OS Page Cache (filesystem cache)           │    │
│  │  Caches data files, index files, WAL files            │    │
│  │  Size: (Total RAM - shared_buffers - per-backend)     │    │
│  └──────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### Shared Memory

| Component | Parameter | Purpose |
|-----------|-----------|---------|
| **Shared Buffers** | `shared_buffers` | Main cache for data pages (8 KB blocks). All backends read/write through this |
| **WAL Buffers** | `wal_buffers` | Staging area for WAL records before flushed to disk |
| **Lock Table** | (auto-sized) | Stores all active lock information |
| **CLOG** | (auto-sized) | Transaction commit status (committed, aborted, in-progress) |

**Buffer management** — PostgreSQL uses a **clock-sweep** eviction algorithm (similar to simplified LRU):

```
Page read:
1. Check shared buffers (buffer hit → fast ✅)
2. Miss → read from OS page cache or disk
3. Evict least-recently-used page if buffers full
4. Insert new page

Buffer hit ratio should be > 99% for OLTP workloads:
```

```sql
-- Check buffer hit ratio
SELECT
    sum(blks_hit) AS buffer_hits,
    sum(blks_read) AS disk_reads,
    round(100.0 * sum(blks_hit) / nullif(sum(blks_hit) + sum(blks_read), 0), 2) AS hit_ratio
FROM pg_stat_database;

-- Per-table buffer usage
SELECT
    schemaname, relname,
    heap_blks_hit, heap_blks_read,
    round(100.0 * heap_blks_hit / nullif(heap_blks_hit + heap_blks_read, 0), 2) AS hit_ratio
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC
LIMIT 20;
```

### Per-Backend Memory

Each PostgreSQL backend (connection) has its own private memory:

| Component | Size | Notes |
|-----------|------|-------|
| `work_mem` | 4 MB default | Allocated per sort/hash operation in a query plan |
| `temp_buffers` | 8 MB default | Cache for temporary tables |
| `maintenance_work_mem` | 64 MB default | Only for VACUUM, INDEX, ALTER operations |
| Catalog cache | Variable | Cached system catalog entries |
| Plan cache | Variable | Prepared statement plans |
| Relcache | Variable | Relation metadata cache |

**Total per-backend worst case**:

```
~10 MB base + work_mem × N_operations + temp_buffers
```

### OS Page Cache

PostgreSQL performs **double caching** by design:

```
Data file on disk
    ↓ read()
OS Page Cache (kernel manages, uses free RAM)
    ↓ memcpy()
Shared Buffers (PostgreSQL manages)
    ↓
Query result
```

This is why `shared_buffers` = 25% is recommended — the remaining RAM is used by the OS page cache, which caches the same data files. PostgreSQL benefits from both layers.

> **Key insight**: `effective_cache_size` should reflect **both** shared_buffers + OS page cache. It doesn't allocate memory — it tells the planner how much data is likely cached.

### Huge Pages

Standard memory pages are 4 KB. With large `shared_buffers`, the CPU's TLB (Translation Lookaside Buffer) can thrash. **Huge pages** (2 MB or 1 GB) reduce TLB pressure.

```ini
# postgresql.conf
huge_pages = try    # 'on', 'off', or 'try' (default: try)
```

```bash
# Linux: calculate and configure huge pages
# Number of huge pages needed (2 MB each):
postgres_shared_mem=$(pg_controldata | grep 'shared memory' | awk '{print $NF}')
# Or estimate: shared_buffers + ~200 MB overhead
huge_pages_needed=$((8192 / 2 + 100))   # for 8 GB shared_buffers

# Set in /etc/sysctl.conf
echo "vm.nr_hugepages = $huge_pages_needed" >> /etc/sysctl.conf
sysctl -p
```

### Memory Budget Planning

#### Example: 64 GB RAM Server

```
Total RAM:                          64 GB
─────────────────────────────────────────
shared_buffers:                     16 GB   (25%)
OS Page Cache (target):             ~32 GB  (50%)
Per-backend memory:                 ~12 GB  (reserve)
  200 connections × ~60 MB worst case
OS + PostgreSQL overhead:           ~4 GB

effective_cache_size:               48 GB   (shared_buffers + OS cache)
maintenance_work_mem:               2 GB
work_mem:                           32 MB   (conservative global)
```

---

## 3. VACUUM & Autovacuum

### Why VACUUM Exists

PostgreSQL uses **MVCC** (Multi-Version Concurrency Control) — see [10-mvcc.md](10-mvcc.md):

```
UPDATE orders SET status = 'shipped' WHERE id = 1;

Before UPDATE:
┌───────┬──────────┬──────────┐
│  id   │  status  │  xmax    │
│  1    │ pending  │  0 (live)│
└───────┴──────────┴──────────┘

After UPDATE (both versions exist):
┌───────┬──────────┬──────────┐
│  1    │ pending  │  100 ←── │  dead tuple (old version, marked by xmax)
│  1    │ shipped  │  0       │  live tuple (new version)
└───────┴──────────┴──────────┘

Without VACUUM:
  ❌ Dead tuples accumulate → table bloat
  ❌ Indexes still point to dead tuples → slow scans
  ❌ Transaction ID exhaustion → database freezes
```

VACUUM reclaims space from dead tuples and prevents transaction ID wraparound.

### VACUUM Variants

| Command | What It Does | Locks | Returns Space to OS? |
|---------|-------------|-------|:---:|
| `VACUUM` | Marks dead tuple space as reusable within the table | `ShareUpdateExclusiveLock` (allows concurrent DML) | ❌ |
| `VACUUM FULL` | Rewrites table compactly, reclaims disk space | `AccessExclusiveLock` (**blocks everything**) | ✅ |
| `VACUUM VERBOSE` | Same as VACUUM with detailed output | Same as VACUUM | ❌ |
| `VACUUM (PARALLEL n)` | Parallel vacuum of indexes (PG 13+) | Same as VACUUM | ❌ |
| `VACUUM FREEZE` | Aggressively freezes old tuples to prevent wraparound | Same as VACUUM | ❌ |
| `ANALYZE` | Updates table statistics (row counts, value distributions) | `ShareUpdateExclusiveLock` | N/A |
| `VACUUM ANALYZE` | VACUUM + ANALYZE in one pass | Same as VACUUM | ❌ |

```sql
-- Standard vacuum on a specific table
VACUUM orders;

-- Vacuum with statistics update
VACUUM ANALYZE orders;

-- Verbose (see what happened)
VACUUM (VERBOSE, ANALYZE) orders;

-- Parallel vacuum (PG 13+) — parallelize index cleanup
VACUUM (PARALLEL 4) large_table;

-- VACUUM FULL — rewrites table (use sparingly!)
VACUUM FULL orders;   -- ⚠️ blocks all reads and writes!
```

#### VACUUM vs VACUUM FULL

```
Regular VACUUM:
Before: [live][dead][live][dead][dead][live][    free space    ]
After:  [live][free][live][free][free][live][    free space    ]
                ↑ space reusable for new rows, but file size unchanged

VACUUM FULL:
Before: [live][dead][live][dead][dead][live][    free space    ]
After:  [live][live][live]
                         ↑ file shrunk, space returned to OS
```

> **Rule**: Use regular `VACUUM` (via autovacuum) routinely. Use `VACUUM FULL` only when a table has extreme bloat and you need to reclaim disk space. Consider `pg_repack` as a non-blocking alternative.

### Autovacuum

The **autovacuum launcher** is a background daemon that automatically runs VACUUM and ANALYZE on tables as needed.

```
autovacuum launcher (1 process)
    │
    ├── autovacuum worker 1 → vacuuming table orders
    ├── autovacuum worker 2 → vacuuming table products
    └── autovacuum worker 3 → analyzing table users
    
    (max autovacuum_max_workers concurrent workers)
```

#### When Autovacuum Triggers

```
VACUUM trigger:
  dead_tuples > autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor × table_rows)
  
  Default: dead_tuples > 50 + (0.2 × table_rows)
  → For a 10,000-row table: triggers at 2,050 dead tuples
  → For a 10,000,000-row table: triggers at 2,000,050 dead tuples  ← too late!

ANALYZE trigger:
  changed_tuples > autovacuum_analyze_threshold + (autovacuum_analyze_scale_factor × table_rows)
  
  Default: changed_tuples > 50 + (0.1 × table_rows)
```

### Tuning Autovacuum

#### Global Settings

| Parameter | Default | Recommended | Notes |
|-----------|---------|-------------|-------|
| `autovacuum` | on | **on** | Never turn off globally |
| `autovacuum_max_workers` | 3 | **3-6** | Concurrent autovacuum workers |
| `autovacuum_naptime` | 1 min | **30s – 1 min** | How often launcher checks for tables needing vacuum |
| `autovacuum_vacuum_threshold` | 50 | 50 | Minimum dead tuples before triggering |
| `autovacuum_vacuum_scale_factor` | 0.2 | **0.05-0.1** | Fraction of table size (lower = more frequent) |
| `autovacuum_analyze_threshold` | 50 | 50 | Minimum changed tuples before analyze |
| `autovacuum_analyze_scale_factor` | 0.1 | **0.02-0.05** | Fraction of table size |
| `autovacuum_vacuum_cost_delay` | 2 ms | **2-10 ms** | Pause between autovacuum I/O bursts (throttle) |
| `autovacuum_vacuum_cost_limit` | -1 (uses `vacuum_cost_limit`) | **1000-2000** | How much I/O autovacuum can do before pausing |

```ini
# postgresql.conf — aggressive autovacuum for large tables
autovacuum_max_workers = 4
autovacuum_naptime = '30s'
autovacuum_vacuum_scale_factor = 0.05    # vacuum at 5% dead, not 20%
autovacuum_analyze_scale_factor = 0.02   # analyze at 2% changed
autovacuum_vacuum_cost_limit = 1500      # more I/O budget
```

#### Per-Table Tuning (Critical for Large Tables)

For a 100M-row table, the default scale factor (0.2) means vacuum triggers after **20 million dead tuples** — far too late.

```sql
-- Override autovacuum settings for a specific table
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,     -- vacuum at 1% dead (1M rows)
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.005,   -- analyze at 0.5% changed
    autovacuum_vacuum_cost_delay = 0            -- no throttle for this table
);

-- For high-write tables, consider even more aggressive:
ALTER TABLE events SET (
    autovacuum_vacuum_scale_factor = 0,
    autovacuum_vacuum_threshold = 10000,        -- vacuum every 10K dead tuples
    autovacuum_analyze_scale_factor = 0,
    autovacuum_analyze_threshold = 10000
);
```

### Transaction ID Wraparound

PostgreSQL uses 32-bit transaction IDs (XIDs). After ~2 billion transactions, XIDs wrap around — old data could become "future" and invisible.

```
XID counter: 0 → 1 → 2 → ... → 2,147,483,647 → WRAP → 0
                                                    ↑
                                           DANGER: old data becomes
                                           "in the future" and invisible
```

**VACUUM FREEZE** prevents this by marking old tuples with a special "frozen" XID:

```sql
-- Check how close tables are to wraparound
SELECT
    c.oid::regclass AS table_name,
    age(c.relfrozenxid) AS xid_age,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE c.relkind = 'r'
    AND n.nspname NOT IN ('pg_catalog', 'information_schema')
ORDER BY age(c.relfrozenxid) DESC
LIMIT 20;

-- ⚠️ If age approaches 200,000,000 → autovacuum_freeze_max_age triggers aggressive vacuum
-- ⚠️ If age approaches 2,000,000,000 → database will SHUT DOWN to prevent corruption
```

Key freeze parameters:

| Parameter | Default | Notes |
|-----------|---------|-------|
| `vacuum_freeze_min_age` | 50,000,000 | Minimum XID age before a tuple can be frozen |
| `vacuum_freeze_table_age` | 150,000,000 | Table age that triggers aggressive (whole-table) vacuum |
| `autovacuum_freeze_max_age` | 200,000,000 | Maximum age before forced anti-wraparound vacuum (cannot be disabled) |

### Monitoring VACUUM

```sql
-- Last vacuum/analyze times per table
SELECT
    schemaname, relname,
    n_live_tup, n_dead_tup,
    round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
    last_vacuum, last_autovacuum,
    last_analyze, last_autoanalyze,
    vacuum_count, autovacuum_count
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;

-- Tables that need vacuuming (high dead tuple ratio)
SELECT
    schemaname || '.' || relname AS table_name,
    n_dead_tup,
    n_live_tup,
    round(100.0 * n_dead_tup / nullif(n_live_tup, 0), 2) AS dead_pct,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- Currently running autovacuum workers
SELECT
    pid, query, state,
    now() - xact_start AS duration,
    wait_event_type, wait_event
FROM pg_stat_activity
WHERE query LIKE 'autovacuum:%';

-- VACUUM progress (PG 9.6+)
SELECT
    p.relid::regclass AS table_name,
    p.phase,
    p.heap_blks_total,
    p.heap_blks_scanned,
    p.heap_blks_vacuumed,
    round(100.0 * p.heap_blks_vacuumed / nullif(p.heap_blks_total, 0), 1) AS pct_complete
FROM pg_stat_progress_vacuum p;
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Setting `shared_buffers` to 50%+ of RAM | Leaves no room for OS page cache, performance drops | Keep at 25% of RAM |
| Setting `work_mem` globally to 256 MB | 200 connections × 256 MB × 3 sorts = 150 GB (!!) | Keep low globally, `SET` per-session |
| Turning off autovacuum | Dead tuples accumulate, table bloat, wraparound risk | Never disable globally |
| Default `autovacuum_vacuum_scale_factor` on huge tables | Vacuums trigger too late (20% of 100M = 20M dead tuples) | Set per-table: 0.01 or use threshold only |
| Ignoring `xid_age` on tables | Wraparound → forced vacuum → long freeze → database stall | Monitor age, ensure autovacuum runs |
| Using `VACUUM FULL` routinely | Blocks all reads/writes, locks the table | Use regular VACUUM + `pg_repack` for bloat |
| `random_page_cost = 4.0` on SSDs | Planner avoids index scans, chooses slow seq scans | Set to 1.1 for SSDs |
| Not setting `idle_in_transaction_session_timeout` | Long idle transactions block VACUUM | Set to 60s for web apps |

---

## What to Learn Next

1. **MVCC Internals** — How dead tuples and transaction visibility work — see [10-mvcc.md](10-mvcc.md).
2. **Monitoring & Diagnostics** — Observing the effects of tuning — see [26-monitoring-diagnostics-bloat.md](26-monitoring-diagnostics-bloat.md).
3. **Indexes & Query Performance** — Query-level optimization complements server tuning — see [05-indexes.md](05-indexes.md).
4. **Connection Pooling** — Reducing `max_connections` with PgBouncer — see [24-connection-pooling-load-balancing.md](24-connection-pooling-load-balancing.md).
5. **Locking** — Understanding lock impacts on VACUUM — see [07-locking-modes.md](07-locking-modes.md).

---

> *Ref: [Docs — Server Configuration](https://www.postgresql.org/docs/18/runtime-config.html) · [Docs — Routine Vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html) · [Docs — Monitoring Stats](https://www.postgresql.org/docs/18/monitoring-stats.html) · [PGTune](https://pgtune.leopard.in.ua) · [Neon — PostgreSQL Administration](https://neon.com/postgresql/postgresql-administration)*
