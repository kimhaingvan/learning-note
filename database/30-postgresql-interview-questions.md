# PostgreSQL Interview Questions & Answers

> **Level**: Intermediate → Expert | **Tone**: Technical / Teacher-like  
> The most commonly asked PostgreSQL interview questions in real-world engineering interviews — with detailed answers, SQL examples, and cross-references to deep-dive notes.

---

## Table of Contents

1. [Fundamentals & Architecture](#1-fundamentals--architecture)
2. [Data Types & Schema Design](#2-data-types--schema-design)
3. [Indexes & Query Performance](#3-indexes--query-performance)
4. [Transactions, MVCC & Locking](#4-transactions-mvcc--locking)
5. [VACUUM & Maintenance](#5-vacuum--maintenance)
6. [Replication & High Availability](#6-replication--high-availability)
7. [Backup & Recovery](#7-backup--recovery)
8. [Security & Administration](#8-security--administration)
9. [Advanced SQL & Features](#9-advanced-sql--features)
10. [Performance Tuning & Monitoring](#10-performance-tuning--monitoring)
11. [Practical Scenario Questions](#11-practical-scenario-questions)

---

## 1. Fundamentals & Architecture

### Q1. What is PostgreSQL and how does it differ from MySQL?

PostgreSQL is an open-source, object-relational database management system (ORDBMS) known for standards compliance and extensibility.

| Feature | PostgreSQL | MySQL |
|---------|-----------|-------|
| Type system | Rich — arrays, JSONB, ranges, composites, domains, custom types | Simpler — basic types + JSON |
| MVCC | Full MVCC — readers never block writers | InnoDB uses MVCC, but with different implementation |
| Concurrency | True SERIALIZABLE isolation via SSI | Primarily gap locks for SERIALIZABLE |
| Standards compliance | Very high (SQL:2016) | Moderate |
| Replication | Streaming + logical built-in | Binary log replication |
| Extensibility | Custom types, operators, index methods, procedural languages, FDW | Limited plugin architecture |
| Full-text search | Built-in (tsvector/tsquery) | Built-in but less powerful |
| Partitioning | Declarative (RANGE, LIST, HASH) | Similar from MySQL 8.0 |
| License | PostgreSQL License (permissive) | GPL (or commercial) |

**When to choose PostgreSQL**: Complex queries, strict data integrity, JSONB workloads, geospatial (PostGIS), advanced analytics, need for extensibility.

> Deep dive: [13-installation-setup.md](13-installation-setup.md)

---

### Q2. Explain the PostgreSQL process architecture.

PostgreSQL uses a **multi-process architecture** (not multi-threaded):

```
Client connections
    │    │    │
    ▼    ▼    ▼
┌────────────────────────────────────┐
│         Postmaster (main)          │  ← Listens on port 5432
│   Forks a backend per connection   │
└──────────┬─────────────────────────┘
           │
    ┌──────┼──────────────────────────────────┐
    │      │                                   │
    ▼      ▼                                   ▼
Backend  Backend  Backend     Background Processes:
(conn 1) (conn 2) (conn 3)   ├── Background Writer
                              ├── WAL Writer
                              ├── Checkpointer
                              ├── Autovacuum Launcher + Workers
                              ├── Stats Collector
                              └── Logical Replication Launcher
```

Key points:
- **Postmaster** is the supervisor process — forks one **backend** per client connection
- Each backend has its own private memory (`work_mem`, `temp_buffers`)
- All backends share **shared memory** (shared buffers, WAL buffers, lock table)
- Backend crash → postmaster restarts all backends (to protect shared memory integrity)
- This is why **connection pooling** (PgBouncer) is critical — each connection = one OS process

> Deep dive: [28-postgresql-internals.md](28-postgresql-internals.md) · [24-connection-pooling-load-balancing.md](24-connection-pooling-load-balancing.md)

---

### Q3. What is the data directory structure ($PGDATA)?

```
$PGDATA/
├── base/          ← Per-database directories (named by OID)
├── global/        ← Cluster-wide shared catalogs (pg_database, pg_authid)
├── pg_wal/        ← WAL segment files (16 MB each)
├── pg_xact/       ← Transaction commit status (CLOG)
├── pg_tblspc/     ← Tablespace symlinks
├── postgresql.conf, pg_hba.conf, pg_ident.conf
└── PG_VERSION, postmaster.pid
```

Tables are stored as **heap files** organized into **8 KB pages**. Each page contains a header, line pointers, free space, and tuples.

> Deep dive: [28-postgresql-internals.md § Storage Architecture](28-postgresql-internals.md)

---

### Q4. What is the query processing pipeline in PostgreSQL?

Every SQL query goes through 5 stages:

```
SQL text → Parser → Analyzer → Rewriter → Planner → Executor → Results
```

1. **Parser** — Tokenizes SQL, builds parse tree. Catches syntax errors.
2. **Analyzer** — Resolves table/column names against the system catalog. Type-checks.
3. **Rewriter** — Applies rules (e.g., expands views into their underlying queries).
4. **Planner/Optimizer** — Generates execution plans, estimates costs, picks the cheapest plan.
5. **Executor** — Runs the plan using a pull-based iterator model (Volcano model).

The planner uses **statistics** from `ANALYZE` (stored in `pg_statistic`) to estimate row counts and costs.

> Deep dive: [28-postgresql-internals.md § Query Processing Pipeline](28-postgresql-internals.md)

---

## 2. Data Types & Schema Design

### Q5. What is the difference between `json` and `jsonb`?

| Feature | `json` | `jsonb` |
|---------|:------:|:-------:|
| Storage | Raw text (preserved exactly) | Parsed binary format |
| Duplicate keys | Preserves all | Last value wins |
| Key ordering | Preserves insertion order | No guaranteed order |
| Whitespace | Preserves | Stripped |
| Indexing (GIN) | ❌ No | ✅ Yes |
| Query speed | Slower (must re-parse on every access) | ✅ Faster |
| Write speed | Slightly faster (no parsing) | Slightly slower (must parse) |

**Almost always use `jsonb`** — it's indexable, faster to query, and supports containment operators (`@>`, `?`, `?|`).

```sql
-- GIN index on JSONB
CREATE INDEX idx_data_gin ON events USING GIN (data);

-- Containment query (uses GIN index)
SELECT * FROM events WHERE data @> '{"type": "click"}';

-- SQL/JSON path (PG 12+)
SELECT * FROM events
WHERE jsonb_path_exists(data, '$.items[*] ? (@.price > 100)');
```

> Deep dive: [15-data-types.md § JSON/JSONB](15-data-types.md) · [27-advanced-data-features.md § JSON/JSONB](27-advanced-data-features.md)

---

### Q6. What are the normalization forms and when should you denormalize?

| Form | Rule | Example Violation |
|------|------|-------------------|
| **1NF** | Atomic values, no repeating groups | `tags = "a,b,c"` in one column |
| **2NF** | 1NF + no partial dependencies on composite PK | Non-key column depends on only part of a composite key |
| **3NF** | 2NF + no transitive dependencies | `zip_code → city` (city depends on zip, not on PK) |
| **BCNF** | Every determinant is a candidate key | Rare edge cases beyond 3NF |

**When to denormalize**:
- Read-heavy OLAP/reporting workloads
- Reducing JOINs for latency-critical queries
- Materialized views or pre-aggregated summary tables
- JSONB columns for flexible attributes that rarely need joining

**Rule of thumb**: Normalize first for data integrity, then denormalize selectively for proven performance needs.

> Deep dive: [18-schema-design-data-modeling.md](18-schema-design-data-modeling.md)

---

### Q7. What are `SERIAL` vs `IDENTITY` columns? Which should you use?

```sql
-- Legacy approach (SERIAL)
CREATE TABLE orders (
    id SERIAL PRIMARY KEY  -- creates a sequence + DEFAULT, but id is technically nullable
);

-- Modern approach (IDENTITY — SQL standard, PG 10+)
CREATE TABLE orders (
    id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
```

| Feature | `SERIAL` | `IDENTITY` |
|---------|:--------:|:----------:|
| SQL standard | ❌ PostgreSQL-specific | ✅ SQL:2003 |
| Can INSERT explicit value | ✅ Always (can bypass) | ❌ `GENERATED ALWAYS` prevents it (use `OVERRIDING SYSTEM VALUE` to force) |
| Owner of sequence | Implicit (can get orphaned) | Tightly coupled to column |
| `pg_dump` handling | Sequence + DEFAULT | Single column definition |

**Use `GENERATED ALWAYS AS IDENTITY`** for new tables. It's the SQL standard, prevents accidental value injection, and has cleaner semantics.

> Deep dive: [15-data-types.md § Auto-Incrementing](15-data-types.md) · [11-sequences.md](11-sequences.md)

---

### Q8. Explain `UUID` vs auto-increment integer for primary keys.

| Feature | Auto-increment (`int`/`bigint`) | `UUID` (v4) | `UUID` (v7) |
|---------|:------------------------------:|:-----------:|:-----------:|
| Size | 4 or 8 bytes | 16 bytes | 16 bytes |
| Index-friendly | ✅ Sequential inserts, excellent B-tree locality | ❌ Random — causes page splits, poor cache locality | ✅ Time-ordered — good B-tree locality |
| Globally unique | ❌ Only within a table | ✅ Yes | ✅ Yes |
| Predictable | ⚠️ Yes (enumerable) | ✅ No | ⚠️ Time component leaks creation time |
| Distributed systems | ❌ Needs coordination | ✅ No coordination needed | ✅ No coordination needed |

```sql
-- UUIDv4 (random)
SELECT gen_random_uuid();

-- UUIDv7 (time-ordered, PG 18+)
SELECT uuidv7();
```

**Recommendation:**
- Single database, no security concern about ID enumeration → **auto-increment `bigint`**
- Distributed systems, need to generate IDs on the client → **UUIDv7** (PG 18+) or UUIDv4
- Avoid UUIDv4 for high-write tables (B-tree index fragmentation)

> Deep dive: [15-data-types.md § UUID](15-data-types.md)

---

## 3. Indexes & Query Performance

### Q9. What types of indexes does PostgreSQL support?

| Index Type | Best For | Key Operators |
|-----------|---------|:------------:|
| **B-tree** (default) | Equality, range, sorting | `=`, `<`, `>`, `BETWEEN`, `IN`, `IS NULL` |
| **Hash** | Equality only | `=` only |
| **GIN** (Generalized Inverted) | Full-text search, JSONB, arrays | `@>`, `?`, `?|`, `@@` |
| **GiST** (Generalized Search Tree) | Geometry, ranges, full-text (lossy) | `&&`, `@>`, `<@`, `<<`, `>>` |
| **SP-GiST** (Space-Partitioned GiST) | Non-balanced data (IP addresses, phone numbers) | Same as GiST |
| **BRIN** (Block Range Index) | Large naturally-ordered tables (time-series) | `<`, `<=`, `=`, `>=`, `>` |

```sql
-- Partial index — only index what you query
CREATE INDEX idx_active_orders ON orders (created_at)
    WHERE status = 'active';

-- Expression index — index a computed value
CREATE INDEX idx_lower_email ON users (lower(email));

-- Covering index — avoid heap lookups entirely (index-only scan)
CREATE INDEX idx_orders_cover ON orders (customer_id)
    INCLUDE (total, status);

-- Multicolumn index — leftmost prefix rule applies
CREATE INDEX idx_orders_multi ON orders (customer_id, created_at DESC);
```

> Deep dive: [05-indexes.md](05-indexes.md)

---

### Q10. How do you read an EXPLAIN plan? What should you look for?

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT) SELECT * FROM orders WHERE customer_id = 42;
```

```
Index Scan using idx_orders_customer on orders  (cost=0.43..8.45 rows=5 width=64)
                                                 (actual time=0.023..0.030 rows=5 loops=1)
  Index Cond: (customer_id = 42)
  Buffers: shared hit=4
Planning Time: 0.085 ms
Execution Time: 0.052 ms
```

**Key things to look for**:

| Check | What to Look For | Red Flag |
|-------|-----------------|----------|
| **Scan type** | Seq Scan vs Index Scan vs Bitmap Scan | Seq Scan on large tables with WHERE clause |
| **Row estimate vs actual** | `rows=5` (estimated) vs `rows=5` (actual) | Large discrepancy → stale statistics → run `ANALYZE` |
| **Buffers** | `shared hit` vs `shared read` | High `shared read` → data not in cache |
| **Loops** | In nested loops, multiply rows × loops | High loop count with many rows per loop → slow |
| **Sort method** | `Sort Method: quicksort Memory: 25kB` | `external merge Disk: ...` → `work_mem` too low |
| **Rows removed by filter** | `Rows Removed by Filter: 999995` | Full scan + filter → need a better index |

**Most common fixes:**
1. **Missing index** → Create one matching the WHERE/JOIN/ORDER BY
2. **Stale statistics** → `ANALYZE table_name`
3. **Planner chooses wrong plan** → Check `random_page_cost` (too high on SSDs)
4. **Sort spilling to disk** → Increase `work_mem` for that session

> Deep dive: [05-indexes.md § EXPLAIN Deep Dive](05-indexes.md)

---

### Q11. What is the difference between Index Scan, Index Only Scan, and Bitmap Scan?

| Scan Type | How It Works | When Used |
|-----------|-------------|-----------|
| **Index Scan** | Reads index → for each match, fetches the heap page | Few rows, scattered across pages |
| **Index Only Scan** | Reads only the index — never touches the heap | All needed columns are in the index AND visibility map confirms all-visible |
| **Bitmap Scan** | Phase 1: scans index → builds a bitmap of matching pages. Phase 2: reads those pages sequentially | Moderate number of rows, or combining multiple indexes |

```
                        Few rows          Moderate rows         Many rows
                       ┌──────────────┬──────────────────┬──────────────┐
                       │  Index Scan  │  Bitmap Scan     │  Seq Scan    │
                       │ (random I/O) │  (sorts → seq)   │ (sequential) │
                       └──────────────┴──────────────────┴──────────────┘
```

**Index Only Scan** is the fastest — it requires:
1. A **covering index** (all needed columns in the index)
2. Recent `VACUUM` so the visibility map is up-to-date

```sql
-- Create covering index for index-only scan
CREATE INDEX idx_orders_cover ON orders (customer_id) INCLUDE (total, created_at);

-- This can be an Index Only Scan:
SELECT total, created_at FROM orders WHERE customer_id = 42;
```

> Deep dive: [05-indexes.md](05-indexes.md)

---

### Q12. Why is `OFFSET` slow and what's the alternative?

`OFFSET` forces PostgreSQL to **scan and discard** all skipped rows:

```sql
-- OFFSET 1,000,000 → scans 1,000,000 rows, throws them all away
SELECT * FROM orders ORDER BY created_at LIMIT 10 OFFSET 1000000;
```

**Solution: Keyset (cursor-based) pagination** — use the last seen value as a filter:

```sql
-- First page
SELECT * FROM orders ORDER BY created_at, id LIMIT 10;

-- Next page (pass the last created_at + id from previous page)
SELECT * FROM orders
WHERE (created_at, id) > ('2024-06-15T12:00:00Z', 12345)
ORDER BY created_at, id
LIMIT 10;
```

This uses an **index seek** — constant performance regardless of page depth.

| Feature | OFFSET | Keyset |
|---------|:------:|:------:|
| Performance at page 1 | ✅ Fast | ✅ Fast |
| Performance at page 10,000 | ❌ Very slow | ✅ Still fast |
| Can jump to arbitrary page | ✅ Yes | ❌ No (sequential only) |
| Handles concurrent inserts | ⚠️ Can skip/duplicate rows | ✅ Stable cursor |

> Deep dive: [05-indexes.md § Keyset Pagination](05-indexes.md)

---

## 4. Transactions, MVCC & Locking

### Q13. Explain MVCC in PostgreSQL. How does it work?

**MVCC (Multi-Version Concurrency Control)** — PostgreSQL keeps **multiple versions** of each row. Readers see a snapshot; they never block writers, and writers never block readers.

```
UPDATE orders SET status = 'shipped' WHERE id = 1;

Heap page (both versions co-exist):
┌──────────────────────────────────────────┐
│ (id=1, status='pending',  xmin=100, xmax=200)  ← dead tuple (old)
│ (id=1, status='shipped',  xmin=200, xmax=0)    ← live tuple (new)
└──────────────────────────────────────────┘
```

Each tuple has hidden system columns:
- **`xmin`** — Transaction ID that created this row version
- **`xmax`** — Transaction ID that deleted/updated it (0 = still alive)

**Visibility rules**: A tuple is visible to transaction T if:
1. `xmin` is committed AND `xmin` < T's snapshot
2. `xmax` is 0 (not deleted) OR `xmax` is not committed OR `xmax` > T's snapshot

This is why PostgreSQL needs **VACUUM** — dead tuples accumulate and must be cleaned up.

> Deep dive: [10-mvcc.md](10-mvcc.md) · [28-postgresql-internals.md § Heap Tuple Structure](28-postgresql-internals.md)

---

### Q14. What are the transaction isolation levels in PostgreSQL?

| Level | Dirty Read | Non-Repeatable Read | Phantom Read | Serialization Anomaly |
|-------|:----------:|:-------------------:|:------------:|:---------------------:|
| **Read Uncommitted** (treated as Read Committed) | ❌ Not possible | ✅ Possible | ✅ Possible | ✅ Possible |
| **Read Committed** (default) | ❌ | ✅ Possible | ✅ Possible | ✅ Possible |
| **Repeatable Read** | ❌ | ❌ | ❌ (snapshot-based in PG) | ✅ Possible |
| **Serializable** (SSI) | ❌ | ❌ | ❌ | ❌ |

Key points:
- PostgreSQL **never** allows dirty reads — even `READ UNCOMMITTED` behaves as `READ COMMITTED`
- `REPEATABLE READ` in PG prevents phantoms too (unlike the SQL standard) because it uses **snapshot isolation**
- `SERIALIZABLE` uses **Serializable Snapshot Isolation (SSI)** — detects dependency cycles and aborts one transaction with `ERROR: could not serialize access`

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ...
COMMIT; -- may get: ERROR: could not serialize access due to read/write dependencies
        -- → application must retry the transaction
```

> Deep dive: [01-acid.md](01-acid.md) · [10-mvcc.md](10-mvcc.md)

---

### Q15. What types of locks exist in PostgreSQL?

#### Table-Level Locks

| Lock Mode | Conflicts With | Acquired By |
|-----------|---------------|-------------|
| `ACCESS SHARE` | `ACCESS EXCLUSIVE` | `SELECT` |
| `ROW SHARE` | `EXCLUSIVE`, `ACCESS EXCLUSIVE` | `SELECT FOR UPDATE/SHARE` |
| `ROW EXCLUSIVE` | `SHARE`, `SHARE ROW EXCLUSIVE`, `EXCLUSIVE`, `ACCESS EXCLUSIVE` | `INSERT`, `UPDATE`, `DELETE` |
| `ACCESS EXCLUSIVE` | Everything | `DROP TABLE`, `TRUNCATE`, `VACUUM FULL`, some `ALTER TABLE` |

#### Row-Level Locks

```sql
-- Exclusive row lock (for update)
SELECT * FROM orders WHERE id = 1 FOR UPDATE;

-- Share row lock (allow other FOR SHARE, block FOR UPDATE)
SELECT * FROM orders WHERE id = 1 FOR SHARE;

-- Skip locked rows (great for job queues)
SELECT * FROM jobs WHERE status = 'pending'
    ORDER BY created_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED;

-- Don't wait for locks
SELECT * FROM orders WHERE id = 1 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row (instead of waiting)
```

#### Advisory Locks

Application-defined locks that don't correspond to any database object:

```sql
-- Session-level advisory lock (held until session ends or explicit unlock)
SELECT pg_advisory_lock(12345);
SELECT pg_advisory_unlock(12345);

-- Transaction-level (released on COMMIT/ROLLBACK)
SELECT pg_advisory_xact_lock(12345);
```

> Deep dive: [07-locking-modes.md](07-locking-modes.md)

---

### Q16. What is a deadlock and how does PostgreSQL handle it?

A **deadlock** occurs when two or more transactions wait on each other's locks in a cycle:

```
Transaction A: holds lock on row 1, waiting for lock on row 2
Transaction B: holds lock on row 2, waiting for lock on row 1
→ Neither can proceed = DEADLOCK
```

PostgreSQL detects deadlocks automatically:
1. After `deadlock_timeout` (default: 1s), the waiting backend checks for cycles in the **wait-for graph**
2. If a cycle is found, PostgreSQL **aborts one transaction** with `ERROR: deadlock detected`
3. The application should **retry** the aborted transaction

**Prevention strategies:**
- Always lock resources in **consistent order** (e.g., always lock lower ID first)
- Keep transactions **short**
- Use `NOWAIT` or `SKIP LOCKED` to avoid waiting
- Avoid locking many rows in interactive transactions

```sql
-- Consistent ordering prevents deadlocks:
-- Both transactions lock rows in the same order (by id ASC)
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- ...
COMMIT;
```

> Deep dive: [07-locking-modes.md § Deadlocks](07-locking-modes.md)

---

### Q17. What is the difference between optimistic and pessimistic locking?

| Approach | Mechanism | When to Use |
|----------|----------|-------------|
| **Pessimistic** | `SELECT ... FOR UPDATE` — lock the row when you read it | High contention, short transactions |
| **Optimistic** | Read without locking, check for changes before writing (using a version column or `xmin`) | Low contention, long user interactions |

```sql
-- Pessimistic: lock the row immediately
BEGIN;
SELECT * FROM products WHERE id = 1 FOR UPDATE;
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- Optimistic: check version before updating
-- Read with version
SELECT id, stock, version FROM products WHERE id = 1;
-- → stock=10, version=5

-- Later, update only if version hasn't changed
UPDATE products
SET stock = stock - 1, version = version + 1
WHERE id = 1 AND version = 5;
-- If 0 rows updated → someone else modified it → retry
```

---

## 5. VACUUM & Maintenance

### Q18. Why does PostgreSQL need VACUUM?

Because of MVCC — `UPDATE` and `DELETE` don't physically remove old row versions. They leave **dead tuples** that:

1. **Waste disk space** (table bloat)
2. **Slow down sequential scans** (must skip over dead tuples)
3. **Waste index space** (indexes still point to dead tuples)
4. **Risk transaction ID wraparound** (32-bit XIDs wrap after ~2 billion transactions → database shuts down)

`VACUUM`:
- Marks dead tuple space as **reusable** (but doesn't return space to OS)
- Updates the **visibility map** (enables index-only scans)
- Updates the **free space map** (enables efficient inserts)
- **Freezes** old tuples to prevent XID wraparound

`VACUUM FULL`:
- Rewrites the entire table compactly → returns space to OS
- Requires `ACCESS EXCLUSIVE` lock (blocks everything) — **avoid in production**
- Use **`pg_repack`** as a non-blocking alternative

> Deep dive: [25-performance-tuning.md § VACUUM & Autovacuum](25-performance-tuning.md) · [26-monitoring-diagnostics-bloat.md § Bloat Management](26-monitoring-diagnostics-bloat.md)

---

### Q19. How does autovacuum work and how do you tune it?

**Autovacuum** is a background daemon that automatically runs VACUUM and ANALYZE on tables when dead tuples exceed a threshold:

```
Trigger formula:
  dead_tuples > autovacuum_vacuum_threshold + (autovacuum_vacuum_scale_factor × reltuples)

Default: dead_tuples > 50 + (0.2 × table_row_count)
```

**Problem with defaults on large tables**: For a 100M-row table, VACUUM triggers only after **20 million dead tuples** — far too late.

**Tuning:**

```sql
-- Per-table override (critical for large tables)
ALTER TABLE orders SET (
    autovacuum_vacuum_scale_factor = 0.01,      -- vacuum at 1% dead (not 20%)
    autovacuum_vacuum_threshold = 1000,
    autovacuum_analyze_scale_factor = 0.005
);

-- For very high-write tables
ALTER TABLE events SET (
    autovacuum_vacuum_scale_factor = 0,
    autovacuum_vacuum_threshold = 50000          -- vacuum every 50K dead tuples
);
```

```ini
# postgresql.conf — global tuning
autovacuum_max_workers = 4
autovacuum_naptime = '30s'
autovacuum_vacuum_scale_factor = 0.05
autovacuum_vacuum_cost_limit = 1500
```

> Deep dive: [25-performance-tuning.md § Tuning Autovacuum](25-performance-tuning.md)

---

### Q20. What is transaction ID wraparound and why is it dangerous?

PostgreSQL uses **32-bit transaction IDs** (XIDs). After ~2 billion transactions, the counter wraps around. If old tuples aren't "frozen" before this happens, they become invisible (appear to be "in the future").

**At ~2 billion unfrozen XIDs → PostgreSQL forces a shutdown to prevent data loss.**

Prevention:
- **Autovacuum** automatically freezes old tuples (the `autovacuum_freeze_max_age` threshold)
- **Monitor** XID age: `SELECT max(age(relfrozenxid)) FROM pg_class WHERE relkind = 'r';`
- If age exceeds **200 million** → autovacuum triggers aggressive (anti-wraparound) vacuum
- If age exceeds **2 billion** → emergency shutdown

```sql
-- Check which tables are closest to wraparound
SELECT c.oid::regclass, age(c.relfrozenxid) AS xid_age,
       pg_size_pretty(pg_total_relation_size(c.oid)) AS size
FROM pg_class c
WHERE c.relkind = 'r'
ORDER BY age(c.relfrozenxid) DESC
LIMIT 10;
```

> Deep dive: [25-performance-tuning.md § Transaction ID Wraparound](25-performance-tuning.md)

---

## 6. Replication & High Availability

### Q21. What is the difference between streaming and logical replication?

| Feature | Streaming (Physical) | Logical |
|---------|:-------------------:|:-------:|
| **Replicates** | Entire WAL byte stream (all databases) | Selected tables via publish/subscribe |
| **Cross-version** | ❌ Same major version | ✅ Different versions |
| **DDL** | ✅ Automatic | ❌ Manual on subscriber |
| **Selective tables** | ❌ All or nothing | ✅ Per-table |
| **Standby writes** | ❌ Read-only | ✅ Full read-write |
| **Failover** | ✅ Promote standby | ⚠️ Not designed for failover |
| **Best for** | HA, disaster recovery, read scaling | Migration, selective sync, integration |

```sql
-- Logical replication setup
-- Publisher:
CREATE PUBLICATION my_pub FOR TABLE orders, customers;

-- Subscriber:
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=publisher dbname=mydb user=repl_user'
    PUBLICATION my_pub;
```

> Deep dive: [23-replication-high-availability.md](23-replication-high-availability.md)

---

### Q22. How does failover work? What is Patroni?

**Manual failover:**
```sql
-- On the standby
SELECT pg_promote();   -- promotes standby to primary
```

**Patroni** is the most popular automatic HA solution:
- Runs as an agent alongside PostgreSQL on each node
- Uses a **DCS (etcd/Consul/ZooKeeper)** for leader election
- Automatically promotes the best standby when primary fails
- Prevents **split brain** via DCS leader lock
- Provides switchover (planned) and failover (unplanned)

```
Patroni + etcd Architecture:
  Node 1 (Patroni + PG Primary) ─┐
  Node 2 (Patroni + PG Standby) ─┼──► etcd (leader election)
  Node 3 (Patroni + PG Standby) ─┘
```

> Deep dive: [23-replication-high-availability.md § Failover & Switchover](23-replication-high-availability.md)

---

### Q23. What is a connection pooler and why do you need one?

Each PostgreSQL connection = one **OS process** (~5-10 MB RAM). Without pooling:
- 1,000 app connections → 1,000 PostgreSQL backends → **10 GB RAM** just for connections
- Fork overhead, context switching, shared buffer contention

**PgBouncer** sits between app and PostgreSQL, multiplexing many client connections onto a smaller pool of server connections.

| Pool Mode | Connection returned to pool after… | Best For |
|-----------|-----------------------------------|---------|
| **Session** | Client disconnects | Legacy apps |
| **Transaction** (recommended) | Transaction completes | ✅ Most applications |
| **Statement** | Each statement | Simple autocommit only |

```
1,000 app connections → PgBouncer (50 pool) → PostgreSQL (50 backends)
```

**Transaction mode caveat**: Session-level state is lost between transactions — no prepared statements, no SET, no LISTEN/NOTIFY, no temp tables across transactions.

> Deep dive: [24-connection-pooling-load-balancing.md](24-connection-pooling-load-balancing.md)

---

## 7. Backup & Recovery

### Q24. What backup strategies does PostgreSQL support?

| Method | Type | Online? | PITR? | Granularity | Speed |
|--------|:----:|:-------:|:-----:|:-----------:|:-----:|
| `pg_dump` | Logical | ✅ | ❌ | Per-table/database | Slow for large DBs |
| `pg_dumpall` | Logical | ✅ | ❌ | Entire cluster | Slow |
| `pg_basebackup` | Physical | ✅ | ✅ (with WAL archiving) | Entire cluster | Faster |
| File system snapshot | Physical | ⚠️ Needs consistent snapshot | ✅ | Entire cluster | Fastest |

**Point-in-Time Recovery (PITR):**
1. Take a base backup (`pg_basebackup`)
2. Archive WAL segments continuously (`archive_command`)
3. To restore: restore base backup + replay WAL up to the target time

```sql
-- Recovery target in postgresql.conf:
recovery_target_time = '2024-06-15 14:30:00'
recovery_target_action = 'promote'
```

> Deep dive: [06-backup-restore.md](06-backup-restore.md)

---

### Q25. What is the difference between `pg_dump` and `pg_basebackup`?

| Feature | `pg_dump` | `pg_basebackup` |
|---------|:---------:|:---------------:|
| Type | Logical (SQL/custom format) | Physical (binary file copy) |
| Restores to | Any PG version (forward-compatible) | Same major version only |
| Granularity | Single table, schema, or database | Entire cluster |
| PITR support | ❌ | ✅ (with WAL archiving) |
| Speed | Slower (reads and serializes all data) | Faster (streams raw files) |
| Consistency | Snapshot at start of dump | Consistent with WAL |
| Custom format | ✅ `pg_restore` with selective restore | ❌ All or nothing |

**Use `pg_dump`** for: single database backup, migration between versions, selective restore.  
**Use `pg_basebackup`** for: disaster recovery, PITR, setting up streaming replicas.

> Deep dive: [06-backup-restore.md](06-backup-restore.md)

---

## 8. Security & Administration

### Q26. How does PostgreSQL authentication work (pg_hba.conf)?

`pg_hba.conf` controls **who can connect, from where, and how**:

```
# TYPE    DATABASE    USER    ADDRESS          METHOD
local     all         all                      peer
host      all         all     127.0.0.1/32     scram-sha-256
host      all         all     10.0.0.0/8       scram-sha-256
host      replication repl    10.0.0.0/8       scram-sha-256
host      all         all     0.0.0.0/0        reject
```

| Method | Security | Use Case |
|--------|:--------:|---------|
| `peer` | ✅ OS user name must match DB user | Local Unix socket connections |
| `scram-sha-256` | ✅ Challenge-response, salted | **Production standard** |
| `md5` | ⚠️ Weaker hash | Legacy — migrate to SCRAM |
| `password` | ❌ Cleartext | Never use |
| `cert` | ✅ Client TLS certificate | Highest security (mTLS) |
| `reject` | N/A | Deny access |

Rules are evaluated **top-to-bottom, first match wins**.

> Deep dive: [21-authentication-ssl.md](21-authentication-ssl.md)

---

### Q27. What is Row-Level Security (RLS)?

RLS restricts which **rows** a user can see or modify — policies are applied transparently to every query:

```sql
-- Enable RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Users can only see their own orders
CREATE POLICY user_orders ON orders
    FOR ALL
    USING (user_id = current_setting('app.current_user_id')::int);

-- Set the user context (typically done by the application)
SET app.current_user_id = '42';

SELECT * FROM orders;
-- Automatically adds: WHERE user_id = 42
```

**Key rule**: RLS policies are **not applied to the table owner** by default. Use `ALTER TABLE orders FORCE ROW LEVEL SECURITY;` if even the owner should be restricted.

> Deep dive: [12-policy.md](12-policy.md) · [08-roles.md](08-roles.md)

---

## 9. Advanced SQL & Features

### Q28. What are CTEs and when should you use (or avoid) them?

**CTE (Common Table Expression)** — a named temporary result set within a query:

```sql
WITH active_orders AS (
    SELECT customer_id, sum(total) AS total_spent
    FROM orders
    WHERE status = 'active'
    GROUP BY customer_id
)
SELECT c.name, ao.total_spent
FROM customers c
JOIN active_orders ao ON c.id = ao.customer_id
WHERE ao.total_spent > 1000;
```

| CTE Behavior | PG Version | Effect |
|-------------|:----------:|--------|
| **Optimization fence** (materialized) | PG 11 and earlier | CTE always materializes — planner can't push WHERE into it |
| **Inlined by default** | PG 12+ | Non-recursive CTEs are inlined (optimized like subqueries) |
| **Force materialization** | PG 12+ | `WITH cte AS MATERIALIZED (...)` |

**Use CTEs for:** readability, recursive queries, writeable CTEs (`WITH ... INSERT/UPDATE/DELETE ... RETURNING`).

**Avoid CTEs when:** performance matters and you're on PG 11 or earlier (use subqueries instead).

> Deep dive: [17-querying-data-manipulation.md § CTEs](17-querying-data-manipulation.md)

---

### Q29. Explain window functions with an example.

Window functions compute values across a set of rows **related to the current row** — without collapsing rows like `GROUP BY`:

```sql
SELECT
    department,
    name,
    salary,
    rank() OVER (PARTITION BY department ORDER BY salary DESC) AS dept_rank,
    salary - avg(salary) OVER (PARTITION BY department) AS diff_from_avg,
    sum(salary) OVER (ORDER BY hire_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM employees;
```

Common window functions:

| Function | Purpose |
|----------|---------|
| `ROW_NUMBER()` | Unique sequential number (1, 2, 3…) |
| `RANK()` | Rank with gaps (1, 2, 2, 4…) |
| `DENSE_RANK()` | Rank without gaps (1, 2, 2, 3…) |
| `LAG(col, n)` / `LEAD(col, n)` | Access previous/next row |
| `FIRST_VALUE(col)` / `LAST_VALUE(col)` | First/last in the window frame |
| `NTILE(n)` | Divide rows into N equal buckets |
| Aggregates (`SUM`, `AVG`, `COUNT`) | Running/moving aggregations over a frame |

> Deep dive: [17-querying-data-manipulation.md § Window Functions](17-querying-data-manipulation.md)

---

### Q30. What is a recursive CTE? Give a practical example.

Recursive CTEs solve hierarchical/graph traversal problems:

```sql
-- Org chart: find all reports (direct and indirect) for a manager
WITH RECURSIVE reports AS (
    -- Base case: direct reports of manager_id = 1
    SELECT id, name, manager_id, 1 AS depth
    FROM employees
    WHERE manager_id = 1

    UNION ALL

    -- Recursive case: reports of reports
    SELECT e.id, e.name, e.manager_id, r.depth + 1
    FROM employees e
    JOIN reports r ON e.manager_id = r.id
    WHERE r.depth < 10    -- safety limit to prevent infinite recursion
)
SELECT * FROM reports ORDER BY depth, name;
```

Common use cases:
- Org chart / management hierarchy
- Category tree navigation
- Bill of materials
- Graph traversal (shortest path)
- Generating series

> Deep dive: [17-querying-data-manipulation.md](17-querying-data-manipulation.md)

---

### Q31. Explain the difference between `DELETE`, `TRUNCATE`, and `DROP`.

| Feature | `DELETE` | `TRUNCATE` | `DROP` |
|---------|:--------:|:----------:|:------:|
| **Removes** | Rows (can use WHERE) | All rows | Entire table |
| **Logged in WAL** | ✅ Row by row | Minimal logging | Minimal logging |
| **Triggers fired** | ✅ Row-level triggers | Only `TRUNCATE` triggers | No |
| **MVCC overhead** | ✅ Creates dead tuples | ❌ No dead tuples | N/A |
| **Lock** | `ROW EXCLUSIVE` | `ACCESS EXCLUSIVE` | `ACCESS EXCLUSIVE` |
| **Recoverable with ROLLBACK** | ✅ | ✅ (within transaction) | ✅ (within transaction) |
| **Speed for millions of rows** | Slow | ✅ Very fast | ✅ Very fast |
| **Resets sequences** | ❌ | Optional (`RESTART IDENTITY`) | N/A |

```sql
DELETE FROM orders WHERE status = 'cancelled';   -- fine for selective deletes
TRUNCATE orders RESTART IDENTITY CASCADE;         -- fast full wipe
DROP TABLE IF EXISTS temp_orders;                 -- remove table entirely
```

---

### Q32. How does PostgreSQL handle full-text search?

PostgreSQL has **built-in** full-text search — no Elasticsearch needed for many use cases:

```sql
-- Create a tsvector column with GIN index
ALTER TABLE articles ADD COLUMN search_vector tsvector;

UPDATE articles SET search_vector =
    setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
    setweight(to_tsvector('english', coalesce(body, '')), 'B');

CREATE INDEX idx_search ON articles USING GIN (search_vector);

-- Search with ranking
SELECT title,
       ts_rank(search_vector, query) AS rank
FROM articles,
     websearch_to_tsquery('english', 'postgresql replication -mysql') AS query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

Features: stemming, stop words, weights (A > B > C > D), phrase search (`<->` operator), prefix matching (`:*`), `websearch_to_tsquery` for natural language input.

> Deep dive: [27-advanced-data-features.md § Full-Text Search](27-advanced-data-features.md)

---

## 10. Performance Tuning & Monitoring

### Q33. What are the most important postgresql.conf parameters to tune?

| Parameter | Default | Recommended | Why |
|-----------|---------|-------------|-----|
| `shared_buffers` | 128 MB | **25% of RAM** | Main in-memory cache |
| `effective_cache_size` | 4 GB | **50-75% of RAM** | Hint to planner about total cache |
| `work_mem` | 4 MB | **16-64 MB** | Per-sort/hash operation — be cautious |
| `maintenance_work_mem` | 64 MB | **512 MB – 2 GB** | For VACUUM, CREATE INDEX |
| `random_page_cost` | 4.0 | **1.1** (SSD) | Affects index vs seq scan decisions |
| `effective_io_concurrency` | 1 | **200** (SSD) | Parallel I/O hint for bitmap scans |
| `max_connections` | 100 | **50-200** (use pooler) | Lower is better — each = one process |
| `max_wal_size` | 1 GB | **4-8 GB** | Reduces checkpoint frequency |
| `checkpoint_completion_target` | 0.9 | **0.9** | Spread checkpoint I/O evenly |

**Critical mistake**: Setting `work_mem` globally to 256 MB with 200 connections × 3 sort operations = 150 GB memory risk!

> Deep dive: [25-performance-tuning.md](25-performance-tuning.md)

---

### Q34. How do you find and fix slow queries?

**Step 1: Enable pg_stat_statements** (must-have extension)

```sql
CREATE EXTENSION pg_stat_statements;

-- Top queries by total time
SELECT calls, round(total_exec_time::numeric, 2) AS total_ms,
       round(mean_exec_time::numeric, 2) AS avg_ms,
       left(query, 100) AS query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

**Step 2: Log slow queries**

```ini
log_min_duration_statement = 250   # log queries > 250ms
```

**Step 3: Analyze execution plans**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

**Step 4: Common fixes**

| Symptom | Diagnosis | Fix |
|---------|----------|-----|
| Seq Scan on large table with WHERE | Missing index | Create appropriate index |
| Estimate vs actual rows mismatch | Stale statistics | `ANALYZE table_name` |
| Sort on disk | `work_mem` too low | `SET work_mem = '128MB'` for the session |
| Nested Loop with high loops | Wrong join strategy | Check statistics, consider `SET enable_nestloop = off` to test |
| High `shared read` in BUFFERS | Data not cached | Increase `shared_buffers` or check if working set fits in RAM |

> Deep dive: [26-monitoring-diagnostics-bloat.md](26-monitoring-diagnostics-bloat.md) · [05-indexes.md § EXPLAIN Deep Dive](05-indexes.md)

---

### Q35. How do you monitor PostgreSQL in production?

**Essential views:**

```sql
-- Active sessions
SELECT pid, state, wait_event_type, wait_event,
       now() - query_start AS runtime, left(query, 80)
FROM pg_stat_activity WHERE state = 'active';

-- Table health
SELECT relname, n_live_tup, n_dead_tup,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC;

-- Unused indexes
SELECT indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes WHERE idx_scan = 0;

-- Replication lag
SELECT application_name, replay_lag
FROM pg_stat_replication;
```

**Monitoring checklist:**

| Metric | Alert Threshold |
|--------|:---------------:|
| Connection count | > 80% of `max_connections` |
| Replication lag | > 10 seconds |
| Buffer hit ratio | < 99% |
| Dead tuple ratio | > 10% of live tuples |
| XID age | > 500 million |
| Long idle-in-transaction | > 5 minutes |

> Deep dive: [26-monitoring-diagnostics-bloat.md](26-monitoring-diagnostics-bloat.md)

---

## 11. Practical Scenario Questions

### Q36. Design a job queue in PostgreSQL.

```sql
CREATE TABLE jobs (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    queue text NOT NULL DEFAULT 'default',
    payload jsonb NOT NULL,
    status text NOT NULL DEFAULT 'pending',
    attempts int NOT NULL DEFAULT 0,
    max_attempts int NOT NULL DEFAULT 3,
    scheduled_at timestamptz NOT NULL DEFAULT now(),
    locked_by text,
    locked_at timestamptz,
    completed_at timestamptz,
    created_at timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX idx_jobs_pending ON jobs (scheduled_at)
    WHERE status = 'pending';

-- Worker claims a job (atomic, no race conditions)
WITH next_job AS (
    SELECT id FROM jobs
    WHERE status = 'pending'
        AND scheduled_at <= now()
        AND attempts < max_attempts
    ORDER BY scheduled_at
    LIMIT 1
    FOR UPDATE SKIP LOCKED      -- ← key: skips jobs locked by other workers
)
UPDATE jobs SET
    status = 'processing',
    locked_by = 'worker-1',
    locked_at = now(),
    attempts = attempts + 1
FROM next_job
WHERE jobs.id = next_job.id
RETURNING *;
```

**`FOR UPDATE SKIP LOCKED`** is the key — multiple workers safely claim different jobs without blocking each other.

---

### Q37. How would you handle database migrations safely in production?

**Rules for safe migrations:**

| Rule | Why |
|------|-----|
| Never `ALTER TABLE ... ADD COLUMN ... DEFAULT value` on PG < 11 | Rewrites entire table (fixed in PG 11+) |
| `CREATE INDEX CONCURRENTLY` | Regular CREATE INDEX locks the table |
| `ALTER TABLE ... ADD COLUMN ... NOT NULL DEFAULT` is safe in PG 11+ | Stores default in catalog, doesn't rewrite |
| Use `lock_timeout` before DDL | Prevent long waits that queue behind your migration |
| Test on a copy first | EXPLAIN on prod-like data sizes |

**Safe DDL pattern:**

```sql
-- Set a short lock timeout to avoid blocking other queries for too long
SET lock_timeout = '5s';

-- Add a nullable column (instant, no rewrite)
ALTER TABLE orders ADD COLUMN new_col text;

-- Create index without blocking writes
CREATE INDEX CONCURRENTLY idx_orders_new ON orders (new_col);

-- Add NOT NULL constraint (PG 12+ validates without full lock)
ALTER TABLE orders ALTER COLUMN new_col SET NOT NULL;

-- Or use CHECK constraint (validated in background):
ALTER TABLE orders ADD CONSTRAINT chk_new_col CHECK (new_col IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT chk_new_col;  -- doesn't hold ACCESS EXCLUSIVE
```

---

### Q38. How would you migrate from one PostgreSQL major version to another?

Three approaches:

| Method | Downtime | Risk | Complexity |
|--------|:--------:|:----:|:----------:|
| **`pg_dump` / `pg_restore`** | ✅ High (full dump/restore) | Low | Low |
| **`pg_upgrade`** | ⚠️ Medium (minutes for link mode) | Medium | Medium |
| **Logical replication** | ✅ Minimal (switchover) | Higher (conflict handling) | Higher |

**Best for minimal downtime — Logical replication:**
1. Set up PG 18 subscriber
2. `CREATE SUBSCRIPTION` pointing to PG 16 publisher
3. Initial data sync + ongoing replication
4. When caught up → switch application to PG 18
5. Drop subscription

```sql
-- On PG 18 (subscriber):
CREATE SUBSCRIPTION upgrade_sub
    CONNECTION 'host=pg16_host dbname=mydb user=repl_user'
    PUBLICATION all_tables;
```

> Deep dive: [23-replication-high-availability.md § Logical Replication Use Cases](23-replication-high-availability.md)

---

### Q39. You have a table with 500M rows and SELECT queries are slow. How do you fix it?

**Systematic approach:**

```
1. DIAGNOSE
   └─ EXPLAIN (ANALYZE, BUFFERS) the slow query
   └─ Check pg_stat_statements for the query pattern
   └─ Check pg_stat_user_tables for dead tuples, seq_scan count

2. QUICK WINS
   ├─ Run ANALYZE if statistics are stale
   ├─ Add missing index (CREATE INDEX CONCURRENTLY)
   ├─ Fix random_page_cost if using SSD (set to 1.1)
   └─ Increase work_mem for the session if sorting

3. SCHEMA OPTIMIZATION
   ├─ Partition the table (RANGE by date is common)
   ├─ Add covering index (INCLUDE) for index-only scans
   ├─ Use partial index if only querying active/recent rows
   └─ Consider BRIN index for time-series data

4. INFRASTRUCTURE
   ├─ Ensure shared_buffers = 25% of RAM
   ├─ Add read replicas for read-heavy workloads
   └─ Use connection pooling (PgBouncer)

5. APPLICATION
   ├─ Use keyset pagination instead of OFFSET
   ├─ Cache frequently accessed data (Redis/Memcached)
   └─ Batch queries where possible
```

---

### Q40. How do you prevent and handle table bloat?

```sql
-- 1. Detect bloat
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0 * n_dead_tup / nullif(n_live_tup + n_dead_tup, 0), 2) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY n_dead_tup DESC;

-- 2. Tune autovacuum for large tables
ALTER TABLE big_table SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 5000
);

-- 3. Fix existing bloat without downtime
-- Install pg_repack
CREATE EXTENSION pg_repack;
-- Run from command line:
-- pg_repack -d mydb -t big_table

-- 4. For index bloat:
REINDEX INDEX CONCURRENTLY idx_big_table_name;

-- 5. Prevention:
--    - Set idle_in_transaction_session_timeout = '60s'
--    - Use fillfactor < 100 for UPDATE-heavy tables
--    - Partition large tables
--    - Monitor n_dead_tup and last_autovacuum
```

> Deep dive: [26-monitoring-diagnostics-bloat.md § Bloat Management](26-monitoring-diagnostics-bloat.md)

---

### Q41. Explain the TOAST mechanism and when it matters.

When a row exceeds ~2 KB, PostgreSQL automatically compresses and/or moves large column values to a separate **TOAST table**:

```
Strategies:
  EXTENDED (default for text/jsonb) — compress first, then move out-of-line
  EXTERNAL — move out-of-line without compression (pre-compressed data)
  MAIN — try hard to keep in-line, compress aggressively
  PLAIN — never TOAST (fixed-size types like integer)
```

**When it matters in interviews:**
- `SELECT *` on tables with large text/JSON columns is slow — it fetches and decompresses every TOASTed value
- `SELECT id, name` on the same table is fast — TOAST data for unselected columns is **never read**
- This is why you should **always select only the columns you need**

> Deep dive: [28-postgresql-internals.md § TOAST](28-postgresql-internals.md)

---

### Q42. How do you implement soft deletes and what are the trade-offs?

```sql
-- Soft delete pattern
ALTER TABLE orders ADD COLUMN deleted_at timestamptz;

-- "Delete"
UPDATE orders SET deleted_at = now() WHERE id = 1;

-- All queries must filter:
SELECT * FROM orders WHERE deleted_at IS NULL;

-- Use partial index for performance:
CREATE INDEX idx_orders_active ON orders (customer_id, created_at)
    WHERE deleted_at IS NULL;

-- Use a view for convenience:
CREATE VIEW active_orders AS
    SELECT * FROM orders WHERE deleted_at IS NULL;
```

| Trade-off | Benefit | Drawback |
|-----------|---------|----------|
| Recoverability | ✅ Can "undelete" | Every query needs `WHERE deleted_at IS NULL` |
| Audit trail | ✅ Know when something was deleted | Table grows forever (VACUUM can't reclaim soft-deleted rows) |
| Foreign keys | ✅ References remain valid | Cascade rules don't apply to soft deletes |
| Performance | | ❌ Table bloat over time — partition by time or archive old soft-deleted rows |

**Alternative**: Use an `archived_orders` table and `DELETE ... RETURNING` to move rows:

```sql
WITH deleted AS (
    DELETE FROM orders WHERE id = 1 RETURNING *
)
INSERT INTO archived_orders SELECT * FROM deleted;
```

> Deep dive: [18-schema-design-data-modeling.md](18-schema-design-data-modeling.md)

---

### Q43. How do you handle concurrent counter updates without race conditions?

```sql
-- ❌ WRONG: Read-then-write race condition
SELECT stock FROM products WHERE id = 1;  -- reads 10
-- Another transaction also reads 10
UPDATE products SET stock = 9 WHERE id = 1;  -- both write 9, lost update!

-- ✅ CORRECT: Atomic update
UPDATE products SET stock = stock - 1 WHERE id = 1 AND stock > 0;
-- PostgreSQL's row-level lock ensures only one transaction modifies at a time

-- ✅ CORRECT: SELECT FOR UPDATE
BEGIN;
SELECT stock FROM products WHERE id = 1 FOR UPDATE;  -- locks the row
-- application logic
UPDATE products SET stock = stock - 1 WHERE id = 1;
COMMIT;

-- ✅ CORRECT: Advisory lock for application-level locking
SELECT pg_advisory_xact_lock(hashtext('product:' || '1'));
-- ... do complex logic ...
-- lock released on COMMIT
```

---

### Q44. Compare partitioning vs sharding in PostgreSQL.

| Feature | Partitioning | Sharding |
|---------|:-----------:|:-------:|
| **Where** | Single PostgreSQL instance | Multiple PostgreSQL instances |
| **Built-in** | ✅ Declarative (`RANGE`, `LIST`, `HASH`) | ❌ Requires external tooling (Citus, app-level) |
| **Scaling** | Vertical — helps query performance | Horizontal — scales beyond one machine |
| **Query routing** | Transparent (partition pruning) | Application or middleware responsibility |
| **Transactions** | ✅ Full ACID across partitions | ⚠️ Distributed transactions needed |
| **Complexity** | Low | High |

```sql
-- Declarative range partitioning
CREATE TABLE events (
    id bigint GENERATED ALWAYS AS IDENTITY,
    created_at timestamptz NOT NULL,
    data jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_q1 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE events_2024_q2 PARTITION OF events
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Queries automatically prune irrelevant partitions:
SELECT * FROM events WHERE created_at >= '2024-05-01';
-- Only scans events_2024_q2
```

> Deep dive: [09-partitioning.md](09-partitioning.md) · [18-schema-design-data-modeling.md](18-schema-design-data-modeling.md)

---

### Q45. What are the key differences between a stored function and a stored procedure?

| Feature | Function | Procedure (PG 11+) |
|---------|:--------:|:------------------:|
| Has `RETURN` value | ✅ Required | ❌ Optional (`INOUT` params for output) |
| Called with | `SELECT my_func()` | `CALL my_proc()` |
| Transaction control | ❌ Cannot `COMMIT`/`ROLLBACK` inside | ✅ Can `COMMIT`/`ROLLBACK` inside |
| Used in SQL expressions | ✅ `WHERE my_func(col)` | ❌ Only via `CALL` |
| Volatility categories | `IMMUTABLE`, `STABLE`, `VOLATILE` | Not applicable |
| Use in indexes | ✅ `IMMUTABLE` functions | ❌ |

```sql
-- Function: returns a value, usable in queries
CREATE FUNCTION get_total(order_id int) RETURNS numeric AS $$
    SELECT sum(price * quantity) FROM order_items WHERE order_id = $1;
$$ LANGUAGE sql STABLE;

SELECT get_total(42);

-- Procedure: long-running batch with intermediate commits
CREATE PROCEDURE archive_old_orders(cutoff_date date) AS $$
DECLARE
    batch_size int := 1000;
BEGIN
    LOOP
        WITH moved AS (
            DELETE FROM orders
            WHERE created_at < cutoff_date
            LIMIT batch_size
            RETURNING *
        )
        INSERT INTO archived_orders SELECT * FROM moved;

        EXIT WHEN NOT FOUND;
        COMMIT;  -- commit each batch (only procedures can do this)
    END LOOP;
END;
$$ LANGUAGE plpgsql;

CALL archive_old_orders('2023-01-01');
```

> Deep dive: [19-stored-procedures-functions-triggers.md](19-stored-procedures-functions-triggers.md)

---

## Quick Reference: Top 10 Interview Tips

| # | Tip |
|---|-----|
| 1 | **Always mention MVCC** when discussing concurrency — it's PostgreSQL's core innovation |
| 2 | **Know EXPLAIN output** — interviewers love asking you to read execution plans |
| 3 | **Understand why VACUUM exists** — it's the most common "PostgreSQL gotcha" |
| 4 | **Connection pooling** — mention PgBouncer whenever discussing scaling |
| 5 | **Index types** — know when to use B-tree vs GIN vs GiST vs BRIN |
| 6 | **`SELECT ... FOR UPDATE SKIP LOCKED`** — the answer to every "concurrent processing" question |
| 7 | **Keyset pagination** — always prefer over OFFSET for large datasets |
| 8 | **`pg_stat_statements`** — the first tool you install on any PostgreSQL instance |
| 9 | **Streaming replication + Patroni** — the standard HA answer |
| 10 | **`CREATE INDEX CONCURRENTLY`** — always mention for production index creation |

---

> *Cross-references: This file ties together concepts from all notes in this repository. Use the linked files for in-depth study of any topic.*
