# Monitoring, Diagnostics & Bloat Management

> **Level**: Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 11.4 — Monitoring & Diagnostics / Phase 11.5 — pg_stat Catalog Views / Phase 11.6 — Bloat Management  
> How to observe, diagnose, and maintain PostgreSQL — system catalog views, monitoring tools, slow query detection, and managing table/index bloat.

---

## Table of Contents

1. [pg_stat_activity — Live Session Monitoring](#1-pg_stat_activity--live-session-monitoring)
2. [pg_stat_statements — Query Performance](#2-pg_stat_statements--query-performance)
3. [pg_stat Catalog Views](#3-pg_stat-catalog-views)
   - [Table Statistics](#table-statistics)
   - [Index Statistics](#index-statistics)
   - [I/O Statistics](#io-statistics)
   - [Replication Statistics](#replication-statistics)
   - [WAL Statistics](#wal-statistics)
   - [Progress Views](#progress-views)
4. [Slow Query Detection](#4-slow-query-detection)
5. [Wait Events & Bottleneck Analysis](#5-wait-events--bottleneck-analysis)
6. [Bloat Management](#6-bloat-management)
   - [Detecting Table Bloat](#detecting-table-bloat)
   - [Detecting Index Bloat](#detecting-index-bloat)
   - [Fixing Bloat](#fixing-bloat)
   - [Preventing Bloat](#preventing-bloat)
7. [External Monitoring Tools](#7-external-monitoring-tools)

---

## 1. pg_stat_activity — Live Session Monitoring

The **single most important** diagnostic view — shows every active connection and what it's doing.

```sql
SELECT
    pid,
    usename,
    datname,
    client_addr,
    application_name,
    state,
    wait_event_type,
    wait_event,
    backend_start,
    xact_start,
    query_start,
    now() - query_start AS query_duration,
    left(query, 100) AS query_preview
FROM pg_stat_activity
WHERE datname IS NOT NULL
ORDER BY query_start ASC NULLS LAST;
```

### Session States

| State | Meaning |
|-------|---------|
| `active` | Executing a query right now |
| `idle` | Connected but not running anything |
| `idle in transaction` | ⚠️ In an open transaction, not executing. Blocks VACUUM |
| `idle in transaction (aborted)` | Transaction errored, waiting for ROLLBACK |
| `fastpath function call` | Executing a fast-path function |
| `disabled` | Track activity disabled for this backend |

### Common Diagnostic Queries

```sql
-- Connections by state
SELECT state, count(*)
FROM pg_stat_activity
WHERE datname IS NOT NULL
GROUP BY state;

-- Long-running queries (> 1 minute)
SELECT pid, usename, now() - query_start AS runtime, query
FROM pg_stat_activity
WHERE state = 'active'
    AND now() - query_start > interval '1 minute'
ORDER BY runtime DESC;

-- Idle-in-transaction sessions (these block VACUUM!)
SELECT pid, usename, now() - xact_start AS idle_time, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_time DESC;

-- Kill a problematic query (graceful)
SELECT pg_cancel_backend(12345);     -- cancels current query, keeps connection

-- Kill a problematic connection (forceful)
SELECT pg_terminate_backend(12345);  -- terminates entire connection
```

---

## 2. pg_stat_statements — Query Performance

The essential extension for identifying slow and resource-heavy queries. Tracks execution statistics for all **normalized** (parameterized) SQL statements.

```sql
-- Enable the extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 10000          # max tracked statements
pg_stat_statements.track = 'top'        # track top-level statements
pg_stat_statements.track_utility = on   # include non-DML (CREATE, ALTER, etc.)
```

### Top Queries by Total Time

```sql
SELECT
    queryid,
    calls,
    round(total_exec_time::numeric, 2) AS total_time_ms,
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    round(stddev_exec_time::numeric, 2) AS stddev_ms,
    rows,
    round(100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0), 2) AS hit_ratio,
    left(query, 120) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

### Top Queries by I/O (Shared Buffer Reads)

```sql
SELECT
    queryid,
    calls,
    shared_blks_read,
    shared_blks_hit,
    round(100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0), 2) AS hit_ratio,
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    left(query, 120) AS query
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 20;
```

### Most Called Queries

```sql
SELECT
    queryid,
    calls,
    round(total_exec_time::numeric, 2) AS total_time_ms,
    round(mean_exec_time::numeric, 2) AS avg_time_ms,
    rows / calls AS avg_rows,
    left(query, 120) AS query
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 20;
```

### JIT-Compiled Queries (PG 15+)

```sql
SELECT
    queryid,
    calls,
    jit_functions,
    round(jit_generation_time::numeric, 2) AS jit_gen_ms,
    round(jit_inlining_time::numeric, 2) AS jit_inline_ms,
    round(jit_optimization_time::numeric, 2) AS jit_opt_ms,
    round(jit_emission_time::numeric, 2) AS jit_emit_ms,
    left(query, 100) AS query
FROM pg_stat_statements
WHERE jit_functions > 0
ORDER BY jit_generation_time DESC
LIMIT 10;
```

```sql
-- Reset statistics (do periodically after collecting baselines)
SELECT pg_stat_statements_reset();
```

---

## 3. pg_stat Catalog Views

### Table Statistics

```sql
-- Table activity — scans, tuples, modifications
SELECT
    schemaname,
    relname,
    seq_scan,                           -- sequential scans (full table scans)
    seq_tup_read,                       -- rows read by seq scans
    idx_scan,                           -- index scans
    idx_tup_fetch,                      -- rows fetched by index scans
    n_tup_ins AS inserts,
    n_tup_upd AS updates,
    n_tup_del AS deletes,
    n_tup_hot_upd AS hot_updates,       -- HOT updates (no index update needed)
    n_live_tup,
    n_dead_tup,
    last_autovacuum,
    last_autoanalyze
FROM pg_stat_user_tables
ORDER BY seq_scan DESC;
```

| Column | What It Tells You |
|--------|------------------|
| `seq_scan` high + `idx_scan` low | Missing index — query doing full table scans |
| `n_dead_tup` high | VACUUM not running often enough |
| `n_tup_hot_upd / n_tup_upd` ratio | HOT update efficiency — higher is better |
| `last_autovacuum` = NULL | Autovacuum never ran on this table (not enough dead tuples or misconfigured) |

### Index Statistics

```sql
-- Index usage — find unused indexes
SELECT
    schemaname,
    relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;    -- least-used indexes first

-- Unused indexes (candidates for removal)
SELECT
    schemaname || '.' || relname AS table_name,
    indexrelname AS index_name,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
    AND indexrelname NOT LIKE '%_pkey'    -- keep primary keys
ORDER BY pg_relation_size(indexrelid) DESC;
```

### I/O Statistics

```sql
-- pg_stat_io (PG 16+) — detailed I/O by backend type
SELECT
    backend_type,
    object,
    context,
    reads, writes, extends,
    op_bytes,
    hits, evictions,
    round(100.0 * hits / nullif(hits + reads, 0), 2) AS hit_ratio
FROM pg_stat_io
WHERE reads > 0 OR writes > 0
ORDER BY reads + writes DESC;

-- Checkpoint and bgwriter stats
SELECT
    checkpoints_timed,
    checkpoints_req,        -- requested checkpoints (too frequent = max_wal_size too low)
    buffers_checkpoint,
    buffers_clean,
    buffers_backend,        -- high = backends doing their own writes (bad)
    maxwritten_clean
FROM pg_stat_bgwriter;
```

| Metric | Healthy | Problem |
|--------|---------|---------|
| `buffers_backend` | Low | High — backends flushing dirty buffers themselves (increase shared_buffers or bgwriter) |
| `checkpoints_req` | Low relative to `checkpoints_timed` | High — frequent forced checkpoints (increase max_wal_size) |
| `maxwritten_clean` | 0 | > 0 — bgwriter stopped because it hit `bgwriter_lru_maxpages` |

### Replication Statistics

```sql
-- On primary: connected standbys and lag
SELECT
    pid, usename, application_name, client_addr,
    state, sync_state,
    sent_lsn, write_lsn, flush_lsn, replay_lsn,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes,
    write_lag, flush_lag, replay_lag
FROM pg_stat_replication;

-- Replication slot status
SELECT
    slot_name, slot_type, active,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained_bytes
FROM pg_replication_slots;
```

### WAL Statistics

```sql
-- pg_stat_wal (PG 14+)
SELECT
    wal_records,
    wal_fpi,                    -- full page images (high after checkpoint)
    wal_bytes,
    pg_size_pretty(wal_bytes) AS wal_generated,
    wal_buffers_full,           -- should be 0; if > 0, increase wal_buffers
    wal_write, wal_sync,
    stats_reset
FROM pg_stat_wal;
```

### Progress Views

Track long-running operations in real-time:

```sql
-- VACUUM progress
SELECT relid::regclass, phase, heap_blks_total, heap_blks_vacuumed,
       round(100.0 * heap_blks_vacuumed / nullif(heap_blks_total, 0), 1) AS pct
FROM pg_stat_progress_vacuum;

-- CREATE INDEX progress
SELECT relid::regclass, phase, lockers_total, lockers_done,
       blocks_total, blocks_done,
       tuples_total, tuples_done
FROM pg_stat_progress_create_index;

-- COPY progress
SELECT relid::regclass, command, type, bytes_processed, bytes_total,
       tuples_processed
FROM pg_stat_progress_copy;

-- ANALYZE progress (PG 13+)
SELECT relid::regclass, phase, sample_blks_total, sample_blks_scanned
FROM pg_stat_progress_analyze;

-- Base backup progress
SELECT phase, backup_total, backup_streamed,
       round(100.0 * backup_streamed / nullif(backup_total, 0), 1) AS pct
FROM pg_stat_progress_basebackup;

-- CLUSTER / VACUUM FULL progress
SELECT relid::regclass, command, phase,
       heap_blks_total, heap_blks_scanned
FROM pg_stat_progress_cluster;
```

---

## 4. Slow Query Detection

### Built-in Logging

```ini
# postgresql.conf

# Log queries taking longer than threshold
log_min_duration_statement = 250      # ms — log queries > 250ms. Set to 0 for ALL queries.

# Log checkpoints (detect checkpoint pressure)
log_checkpoints = on

# Log lock waits (detect contention)
log_lock_waits = on
deadlock_timeout = '1s'

# Log temp file usage (detect work_mem too low)
log_temp_files = 0          # log all temp file usage (0 = all, -1 = disabled)

# Log auto-explain for slow queries
# (add to shared_preload_libraries)
```

### auto_explain Extension

Automatically logs execution plans for slow queries — no need to run EXPLAIN manually.

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements, auto_explain'

# auto_explain settings
auto_explain.log_min_duration = '500ms'    # log plans for queries > 500ms
auto_explain.log_analyze = on              # include actual times (like EXPLAIN ANALYZE)
auto_explain.log_buffers = on              # include buffer usage
auto_explain.log_timing = on
auto_explain.log_nested_statements = on    # include sub-statements from functions
auto_explain.log_format = 'json'           # easier to parse
```

---

## 5. Wait Events & Bottleneck Analysis

Wait events reveal **what** processes are waiting on — the key to diagnosing bottlenecks.

```sql
-- Current wait events
SELECT
    pid, usename, state,
    wait_event_type, wait_event,
    left(query, 80) AS query
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
    AND state = 'active';

-- Wait event distribution (snapshot)
SELECT
    wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event IS NOT NULL
GROUP BY wait_event_type, wait_event
ORDER BY count(*) DESC;
```

| Wait Event Type | Common Events | Meaning |
|----------------|---------------|---------|
| **LWLock** | `buffer_mapping`, `WALWriteLock` | Lightweight lock contention (internal) |
| **Lock** | `relation`, `tuple`, `transactionid` | Row/table lock contention — see [07-locking-modes.md](07-locking-modes.md) |
| **BufferPin** | `BufferPin` | Waiting for a buffer pin (rare) |
| **IO** | `DataFileRead`, `WALSync`, `DataFileWrite` | I/O operations — disk is the bottleneck |
| **Client** | `ClientRead`, `ClientWrite` | Waiting for client to send query or read results |
| **Activity** | `AutoVacuumMain`, `WalWriterMain` | Background worker idle (normal) |

---

## 6. Bloat Management

### Detecting Table Bloat

Bloat = wasted space from dead tuples that VACUUM has reclaimed internally but not returned to the OS.

#### pgstattuple Extension (Accurate)

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;

-- Detailed bloat analysis
SELECT * FROM pgstattuple('orders');
-- Returns: table_len, tuple_count, tuple_len, tuple_percent,
--          dead_tuple_count, dead_tuple_len, dead_tuple_percent,
--          free_space, free_percent
```

#### Estimation Query (No Extension Needed)

```sql
-- Estimated table bloat
SELECT
    schemaname || '.' || tablename AS table_name,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) AS total_size,
    pg_size_pretty(
        pg_total_relation_size(schemaname || '.' || tablename) -
        pg_relation_size(schemaname || '.' || tablename)
    ) AS index_size,
    CASE WHEN n_live_tup > 0 THEN
        round(100.0 * n_dead_tup / n_live_tup, 2)
    ELSE 0 END AS dead_pct
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC
LIMIT 20;
```

#### pgstattuple for Indexes

```sql
-- Index bloat analysis
SELECT * FROM pgstatindex('orders_pkey');
-- Returns: version, tree_level, index_size, root_block_no,
--          internal_pages, leaf_pages, empty_pages, deleted_pages,
--          avg_leaf_density, leaf_fragmentation
```

### Detecting Index Bloat

```sql
-- B-tree index bloat estimation
SELECT
    i.indexrelname AS index_name,
    i.relname AS table_name,
    pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size,
    i.idx_scan AS scans,
    s.leaf_fragmentation
FROM pg_stat_user_indexes i
CROSS JOIN LATERAL pgstatindex(i.indexrelid) s
WHERE pg_relation_size(i.indexrelid) > 10 * 1024 * 1024    -- > 10 MB
ORDER BY s.leaf_fragmentation DESC
LIMIT 20;
```

### Fixing Bloat

| Method | Blocking? | Reclaims Space to OS? | Notes |
|--------|:---------:|:---------------------:|-------|
| `VACUUM` | ❌ No | ❌ No | Marks space reusable. First step. |
| `VACUUM FULL` | ✅ **Yes** (AccessExclusiveLock) | ✅ Yes | Rewrites table. Blocks everything. |
| `REINDEX CONCURRENTLY` | ❌ No | ✅ Yes (for indexes) | Rebuilds index without blocking writes (PG 12+) |
| `pg_repack` | ❌ No | ✅ Yes | Extension — online table + index repack |
| `CLUSTER` | ✅ **Yes** | ✅ Yes | Rewrites table in index order |

#### pg_repack (Recommended for Production)

```bash
# Install
sudo apt install postgresql-18-repack   # or build from source

# Enable
psql -c "CREATE EXTENSION pg_repack;"
```

```bash
# Repack a specific table (online, non-blocking)
pg_repack -d mydb -t orders

# Repack a specific index
pg_repack -d mydb -i orders_customer_id_idx

# Repack all tables in a database
pg_repack -d mydb

# Dry run
pg_repack -d mydb -t orders --dry-run
```

How `pg_repack` works:
1. Creates a shadow table
2. Copies live data to shadow table (sorted by cluster index if specified)
3. Replays changes captured via trigger during copy
4. Swaps shadow table with original (brief exclusive lock)

#### REINDEX CONCURRENTLY

```sql
-- Rebuild a specific index without blocking
REINDEX INDEX CONCURRENTLY orders_customer_id_idx;

-- Rebuild all indexes on a table
REINDEX TABLE CONCURRENTLY orders;

-- Rebuild all indexes in a schema
REINDEX SCHEMA CONCURRENTLY public;
```

### Preventing Bloat

| Strategy | How |
|----------|-----|
| **Tune autovacuum** | Lower `scale_factor`, increase `cost_limit` — see [25-performance-tuning.md](25-performance-tuning.md) |
| **Kill idle-in-transaction** | Set `idle_in_transaction_session_timeout` — long transactions block cleanup |
| **Avoid long-running transactions** | They hold MVCC snapshots, preventing dead tuple cleanup |
| **Fill factor** | `CREATE TABLE ... WITH (fillfactor = 90)` — leave 10% room for HOT updates, reducing dead tuple creation |
| **HOT updates** | When only non-indexed columns change, PostgreSQL can update in-place without creating dead index entries |
| **Partitioning** | Partition large tables — VACUUM runs per-partition, faster and more granular |
| **Monitor regularly** | Track `n_dead_tup`, `last_autovacuum`, bloat metrics |

```sql
-- Set fillfactor for a table that gets frequent updates
ALTER TABLE orders SET (fillfactor = 80);   -- leave 20% for HOT updates
-- Note: requires VACUUM FULL or pg_repack to take effect on existing data
```

---

## 7. External Monitoring Tools

| Tool | Type | Strengths |
|------|------|-----------|
| **pgBadger** | Log analyzer | Beautiful HTML reports from PostgreSQL logs. Top queries, error rates, connections |
| **pg_stat_monitor** | Extension | Enhanced pg_stat_statements — buckets, histograms, query plans |
| **Prometheus + postgres_exporter** | Metrics pipeline | Time-series metrics. Alerting. Grafana dashboards |
| **pgMonitor** | Dashboard suite | Crunchy Data's monitoring stack (Prometheus + Grafana + alerting) |
| **PgHero** | Web UI | Quick performance insights, suggested indexes, space analysis |
| **Datadog / New Relic** | SaaS monitoring | Full observability with PostgreSQL integration |
| **check_postgres** | Nagios plugin | Health checks for traditional monitoring |
| **temBoard** | Web admin UI | Activities, monitoring, maintenance, configuration |

### Monitoring Checklist

| What | How | Alert Threshold |
|------|-----|----------------|
| Connection count | `pg_stat_activity` | > 80% of `max_connections` |
| Active queries | `pg_stat_activity WHERE state='active'` | > CPU cores |
| Idle-in-transaction | `pg_stat_activity WHERE state='idle in transaction'` | Duration > 5 min |
| Replication lag | `pg_stat_replication` | > 10 seconds |
| Buffer hit ratio | `pg_stat_database` | < 99% |
| Dead tuple ratio | `pg_stat_user_tables` | > 10% of live tuples |
| XID age | `pg_class.relfrozenxid` | > 500,000,000 |
| Disk usage | OS metrics | > 85% |
| Table bloat | `pgstattuple` or estimation | > 30% free space |
| Unused indexes | `pg_stat_user_indexes` | `idx_scan = 0` for > 30 days |
| Lock waits | `pg_locks` + `pg_stat_activity` | Any sustained blocking |

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Not installing `pg_stat_statements` | Flying blind — no query performance data | Install it on every PostgreSQL instance |
| Monitoring only at the OS level | Miss PostgreSQL-specific issues (lock contention, bloat, wraparound) | Combine OS + PG monitoring |
| Not logging slow queries | Only find problems when users complain | Set `log_min_duration_statement = 250` |
| Ignoring `buffers_backend` in bgwriter stats | Don't realize backends are doing too many writes | Tune bgwriter, increase shared_buffers |
| Running `VACUUM FULL` routinely | Blocks everything, causes downtime | Use autovacuum + pg_repack for bloat |
| Not monitoring replication slots | Dead slots retain WAL forever → disk full | Alert on inactive slots |
| Ignoring `checkpoints_req` count | Frequent forced checkpoints indicate `max_wal_size` too low | Increase `max_wal_size` |

---

## What to Learn Next

1. **Performance Tuning** — Server-level tuning that these metrics inform — see [25-performance-tuning.md](25-performance-tuning.md).
2. **MVCC** — Understanding why dead tuples and bloat exist — see [10-mvcc.md](10-mvcc.md).
3. **Indexes** — EXPLAIN plans and query optimization — see [05-indexes.md](05-indexes.md).
4. **Locking** — Lock monitoring and diagnostics — see [07-locking-modes.md](07-locking-modes.md).
5. **Auditing** — `pg_stat_statements` for audit and performance — see [22-auditing.md](22-auditing.md).

---

> *Ref: [Docs — Monitoring Stats](https://www.postgresql.org/docs/18/monitoring-stats.html) · [Docs — pg_stat_statements](https://www.postgresql.org/docs/18/pgstatstatements.html) · [Docs — pgstattuple](https://www.postgresql.org/docs/18/pgstattuple.html) · [pg_repack](https://reorg.github.io/pg_repack/) · [pgBadger](https://pgbadger.darold.net/) · [Neon — PostgreSQL Administration](https://neon.com/postgresql/postgresql-administration)*
