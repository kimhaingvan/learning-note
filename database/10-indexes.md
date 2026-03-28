# Indexes — PostgreSQL 18

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 2 — Index Fundamentals · Phase 4 — Indexes & Query Performance  
> **Source**: [PostgreSQL 18 Docs — Chapter 11](https://www.postgresql.org/docs/18/indexes.html)  
> Index types, multicolumn indexes, expression & partial indexes, covering indexes, scanning methods, query optimizer decisions, EXPLAIN deep dive, query optimization techniques, and planner statistics.

---

## Table of Contents

1. [Statistics Collection](#1-statistics-collection)
2. [Index Type Matters](#2-index-type-matters)
3. [EXPLAIN & Examining Index Usage](#3-explain-examining-index-usage)
4. [Scanning Methods](#4-scanning-methods)
5. [How the Query Optimizer Decides to Use Indexes](#5-how-the-query-optimizer-decides-to-use-indexes)
6. [Selectivity](#6-selectivity)
7. [Query Pattern Compatibility](#7-query-pattern-compatibility)
8. [Joins and Sorting](#8-joins-and-sorting)
9. [Caching and Parallelism](#9-caching-and-parallelism)
10. [Partial Indexes](#10-partial-indexes)
11. [Cost Estimation](#11-cost-estimation)
12. [Statistics & the Planner](#12-statistics-the-planner)
13. [How UUIDs Affect Performance](#13-how-uuids-affect-performance)
14. [Query Optimization Techniques](#14-query-optimization-techniques)
15. [The Cost of Long-Running Transactions](#15-the-cost-of-long-running-transactions)
16. [Operator Classes and Operator Families](#16-operator-classes-and-operator-families)
17. [Index-Only Scans and Covering Indexes](#17-index-only-scans-and-covering-indexes)

---

## 1. Statistics Collection

Databases maintain **metadata** about tables and indexes:

- Number of rows (`nrows`)
- Distinct values (cardinality)
- Value distribution (histogram)
- Null counts
- Index depth / size

These let the optimizer estimate **how selective** an index will be.

```sql
ANALYZE table_name;   -- Updates statistics
```

#### 2. Index Type Matters

| Index Type | Best Use Case |
|------------|--------------|
| **B-tree** | Equality, range, sorting |
| **Hash** | Exact matches only |
| **GIN / GiST** | Full-text search, JSON, geometric data |
| **BRIN** | Huge tables with physically sorted data |

#### 3. EXPLAIN & Examining Index Usage

### EXPLAIN Basics

`EXPLAIN` retrieves information about what execution plan PostgreSQL will use for a given SQL statement.

#### Reading EXPLAIN Output

Example output line:

```
Seq Scan on order_item oi  (cost=0.01..158.71 rows=2971 width=798)
```

| Field | Meaning |
|-------|---------|
| **cost=0.01** | Estimated time (in arbitrary cost units) to fetch the **first row**. Can increase if work is needed upfront (e.g., sorting, aggregating). |
| **cost=..158.71** | Estimated **total cost** to complete the query. |
| **rows=2971** | Estimated number of rows the query will return. |
| **width=798** | Sum of all bytes across all columns returned per row. |
| **Heap Fetches** | (Shown in `EXPLAIN ANALYZE`) The count of times the database hit the heap to retrieve records. |

> **Tip**: Use `EXPLAIN ANALYZE` to get **actual** execution times and row counts, not just estimates.

### EXPLAIN Variants

```sql
-- Estimated plan only (does NOT execute the query)
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- Execute + show actual times and row counts
EXPLAIN ANALYZE SELECT * FROM orders WHERE customer_id = 42;

-- Add buffer hit/miss statistics
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;

-- JSON output (for visualization tools like explain.dalibo.com)
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT * FROM orders WHERE customer_id = 42;

-- Show WAL usage (useful for write queries)
EXPLAIN (ANALYZE, BUFFERS, WAL) UPDATE orders SET status = 'shipped' WHERE id = 1;

-- Verbose: show output column list and schema-qualified names
EXPLAIN (VERBOSE) SELECT * FROM orders WHERE customer_id = 42;

-- Settings: show non-default planner settings that affect the plan
EXPLAIN (SETTINGS) SELECT * FROM orders WHERE customer_id = 42;
```

> **Warning**: `EXPLAIN ANALYZE` actually **executes** the query. For `UPDATE`/`DELETE`, wrap in a transaction and `ROLLBACK`:
> ```sql
> BEGIN;
> EXPLAIN ANALYZE DELETE FROM orders WHERE status = 'expired';
> ROLLBACK;
> ```

### Reading a Full Plan

```
Sort  (cost=1500.00..1520.00 rows=8000 width=120) (actual time=12.5..13.2 rows=7823 loops=1)
  Sort Key: created_at DESC
  Sort Method: quicksort  Memory: 1024kB
  ->  Hash Join  (cost=200.00..1000.00 rows=8000 width=120) (actual time=2.1..8.5 rows=7823 loops=1)
        Hash Cond: (o.customer_id = c.id)
        ->  Seq Scan on orders o  (cost=0.00..700.00 rows=25000 width=80) (actual time=0.01..3.2 rows=25000 loops=1)
        ->  Hash  (cost=150.00..150.00 rows=500 width=40) (actual time=1.8..1.8 rows=500 loops=1)
              Buckets: 1024  Batches: 1  Memory Usage: 40kB
              ->  Seq Scan on customers c  (cost=0.00..150.00 rows=500 width=40) (actual time=0.01..1.0 rows=500 loops=1)
Planning Time: 0.25 ms
Execution Time: 14.0 ms
```

| Field | Meaning |
|-------|---------|
| `cost=startup..total` | Estimated cost in planner units (not milliseconds) |
| `rows=N` | Estimated rows (vs `actual rows=N` with ANALYZE) |
| `width=N` | Average row width in bytes |
| `actual time=start..end` | Real wall-clock time in milliseconds |
| `loops=N` | How many times this node was executed (e.g., inner side of nested loop) |
| `Buffers: shared hit=N read=N` | Pages found in cache vs read from disk |
| `Planning Time` | Time spent generating the plan |
| `Execution Time` | Total wall-clock execution time |

### Key Signals to Look For

| Signal | What It Means | Action |
|--------|--------------|--------|
| `Seq Scan` on large table | Missing index or low selectivity | Add index if selectivity is high |
| `estimated rows` ≫ `actual rows` (or vice versa) | Stale statistics | Run `ANALYZE tablename` |
| `Sort Method: external merge Disk` | Sort spilled to disk | Increase `work_mem` or avoid sort |
| `Buffers: shared read` ≫ `shared hit` | Cold cache or table too large for memory | Check `shared_buffers`, consider index-only scan |
| `loops=N` with high N | Inner side of nested loop executed many times | Consider hash join (`enable_nestloop=off` to test) |
| `Heap Fetches: N` on index-only scan | Visibility map not current | Run `VACUUM` on the table |

### Join Algorithms

The planner chooses between three join strategies:

| Algorithm | How It Works | Best When |
|-----------|-------------|-----------|
| **Nested Loop** | For each row in outer table, scan inner table (using index if available) | Small outer table, indexed inner table, or LIMIT queries |
| **Hash Join** | Build a hash table from the smaller table, probe with the larger | One table fits in `work_mem`, equality joins |
| **Merge Join** | Sort both inputs, then merge (like merge sort) | Both inputs already sorted (e.g., by index), large tables |

```sql
-- Force a specific join for testing (don't use in production)
SET enable_hashjoin = off;   -- disable hash join
SET enable_nestloop = off;   -- disable nested loop
SET enable_mergejoin = off;  -- disable merge join
-- Then run EXPLAIN ANALYZE to compare plans
-- Remember to SET ... = on; afterward
```

### Examining Index Usage with System Catalogs

Use `pg_stat_user_indexes` and `pg_stat_user_tables` to see real usage over time.

```sql
-- See the plan
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- See the plan + actual execution stats
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_id = 42;

-- Find unused indexes (idx_scan = 0 since last stats reset)
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) AS index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find tables being sequentially scanned a lot
SELECT
    relname,
    seq_scan,
    idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > 1000
ORDER BY seq_scan DESC;
```

### Example — Real-World Flow

```sql
-- 1. Slow query reported
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'alice@example.com';
-- Output shows: Seq Scan → no index on email

-- 2. Create index
CREATE INDEX CONCURRENTLY idx_users_email ON users (email);

-- 3. Verify plan changed
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE email = 'alice@example.com';
-- Output shows: Index Scan using idx_users_email

-- 4. Wait a few days. Check usage:
SELECT indexname, idx_scan
FROM pg_stat_user_indexes
WHERE tablename = 'users';
-- idx_users_email: 15000  ← actively used
-- idx_users_old_name: 0   ← drop it
```

### Pitfalls

- Stats reset after `pg_stat_reset()` or a crash — low `idx_scan` does not always mean the index is truly unused.
- The planner may ignore an index because `ANALYZE` stats are stale — run `ANALYZE` first.
- Index may not be used when `enable_indexscan = off` (session-level override, useful for testing).
- Do not drop indexes without checking if they support a unique constraint or FK.

### Memory Aid

> `EXPLAIN` shows the plan. `pg_stat_user_indexes` shows the history.  
> Zero scans = candidate for removal. Seq scan on large table = candidate for an index.

---

## 4. Scanning Methods

PostgreSQL uses **5 scanning methods**:

| Method | When Used |
|--------|-----------|
| Sequential Scan | No index available, or most rows are fetched |
| Index Scan | Index exists, few rows match |
| Index Only Scan | Index covers all selected columns |
| Bitmap Scan | Index exists, moderate number of rows match |
| TID Scan | Direct access by tuple ID |

---

### Sequential Scan

#### What

Scans **all pages** of the table sequentially. If the table has 100 pages with 1,000 records per page, it reads all 100,000 records to find those matching the condition.

#### When

PostgreSQL uses a Sequential Scan when at least one of these is true:
- **No index** exists for the columns in the `WHERE` clause.
- The **majority of rows** would be fetched by the query (low selectivity).

---

### Index Scan

#### What

A two-step scan:

1. **Step 1**: Scan the index data structure (B-tree, hash, etc.) to find the **TID** (page number + offset) of matching rows.
2. **Step 2**: For each TID, jump directly to the corresponding heap page and fetch the full row data.

The heap access is needed because the query requires columns **not available in the index**.

---

### Index Only Scan

#### What

Same as Index Scan, except the query only selects columns that **are part of the index**. No heap access is required — all needed data is in the index itself.

> This is the **fastest scan type** when applicable. Design indexes with `INCLUDE` columns to enable more Index Only Scans. See [§10 — Index-Only Scans and Covering Indexes](#10-index-only-scans-and-covering-indexes).

---

### Bitmap Scan

#### What

A hybrid between Index Scan and Sequential Scan. Works in two phases:

#### Step 1: Bitmap Index Scan

PostgreSQL creates a **bitmap hash** where:
- **Key** = page number
- **Value** = 1 (page has qualifying rows) or 0 (page does not)

```
Bitmap: { Page 0: 1, Page 1: 0, Page 2: 1, Page 3: 0, Page 4: 1 }
```

Process:
1. Scan the index to find qualifying rows.
2. For each match, note the page number (from the TID) and set its bitmap flag to 1.
3. **Do NOT** go to the heap yet — just build the bitmap.

#### Step 2: Bitmap Heap Scan

1. Read the bitmap to find which pages have qualifying rows.
2. Fetch those pages from the heap **sequentially** (not randomly).
3. **Recheck** the `WHERE` clause on each row in the page to filter out non-qualifying rows.
4. Return the qualifying rows.

#### Combining Multiple Indexes

If the query has multiple conditions on different indexed columns, PostgreSQL can create **multiple bitmaps** and combine them:

- `AND` condition → **BitmapAnd** (intersection of bitmaps)
- `OR` condition → **BitmapOr** (union of bitmaps)

---

### TID Scan

#### What

Returns rows by their **Tuple ID (TID)** — a 6-byte value consisting of:
- 4 bytes: page number
- 2 bytes: tuple index within the page

Used for direct access when you know the exact physical location of a row.

---

### Key Differences

#### Index Scan vs Bitmap Scan

| Aspect | Index Scan | Bitmap Scan |
|--------|-----------|-------------|
| **Heap access pattern** | Jumps to heap **per index entry** (random I/O) | Builds bitmap first, then accesses heap pages **sequentially** |
| **Switching** | Alternates between index and heap frequently | Reads index once, then reads heap in batch |
| **Best for** | Very few matching rows | Moderate number of matching rows |

#### Index Scan vs Index Only Scan

| Aspect | Index Scan | Index Only Scan |
|--------|-----------|----------------|
| **Heap access** | Required (for non-indexed columns) | **Not required** (all data is in the index) |
| **Performance** | Good | Best (fewer I/Os) |
| **Requirement** | Index on filter columns | Index covers **all** selected columns |

---

## 5. How the Query Optimizer Decides to Use Indexes

The **query optimizer** in PostgreSQL decides to use an index **only if it predicts** that using the index will be **faster than scanning the entire table**.

### Decision Factors

| Factor | Description |
|--------|------------|
| **Selectivity** | Fewer matching rows → higher chance to use index |
| **Data size** | Large tables benefit more from indexes |
| **Query pattern** | Must match index column order and type |
| **Statistics** | Optimizer estimates cost from collected stats |
| **Cost model** | Chooses plan with lowest total I/O + CPU cost |

### Step-by-Step: How the Optimizer Decides

#### 6. Selectivity

**Selectivity** = the fraction of rows that match the condition.

- **High selectivity (few rows match)** → index is efficient.
- **Low selectivity (many rows match)** → index scan may be slower than a full table scan.

```sql
SELECT * FROM users WHERE user_id = 42;
-- user_id is unique → highly selective → use index ✅

SELECT * FROM users WHERE is_active = true;
-- 95% of users are active → low selectivity → full table scan is cheaper ✅
```

#### 7. Query Pattern Compatibility

**Index-friendly patterns** ✅:

```sql
WHERE column = value
WHERE column BETWEEN a AND b
WHERE column LIKE 'abc%'
ORDER BY column
JOIN ON indexed_column
```

**Not index-friendly** ❌:

```sql
WHERE LOWER(column) = 'abc'      -- function call breaks index usage
WHERE column + 1 = 100           -- expression breaks index usage
WHERE column LIKE '%abc%'        -- leading wildcard disables index
```

#### 8. Joins and Sorting

- If a query joins on an **indexed foreign key** → optimizer can use an **index nested loop join**.
- If `ORDER BY` matches the **index order** → skip sorting entirely.
- Otherwise, a full table scan + sort might be cheaper.

#### 9. Caching and Parallelism

The optimizer considers whether data is likely **already in the buffer pool** (cached in memory) and whether **parallel workers** can be used to speed up the scan.

---

## 10. Partial Indexes

### Intuition

- Why index 1,000,000 rows when you only query 10,000 of them?
- A partial index is an index with a `WHERE` clause — it only includes rows that match.
- Smaller, faster, cheaper to maintain.

### What

- [Official docs] A partial index is built over a subset of rows defined by a `WHERE` predicate.
- PostgreSQL uses the partial index only when the query predicate implies the index predicate.

### How

```sql
-- Index only active users (deleted_at IS NULL)
CREATE INDEX idx_active_users ON users (email) WHERE deleted_at IS NULL;

-- Index only unpaid orders (very selective subset)
CREATE INDEX idx_unpaid ON orders (customer_id) WHERE status = 'unpaid';

-- Query: planner uses partial index
SELECT * FROM orders WHERE status = 'unpaid' AND customer_id = 42;
-- ✅ index predicate (status = 'unpaid') is implied by query condition
```

### Example — Real-World Flow

```sql
-- Full orders table: 50M rows, 99% completed, 1% pending
-- Full index on status would be huge and mostly useless
CREATE INDEX idx_pending_orders ON orders (created_at)
WHERE status = 'pending';

-- Dashboard query (fast — tiny index)
SELECT * FROM orders
WHERE status = 'pending'
ORDER BY created_at ASC
LIMIT 100;

-- New order inserted
INSERT INTO orders (customer_id, status, total) VALUES (7, 'pending', 99.99);
-- Only this 1% of rows enters the index → minimal write cost
```

### Pitfalls

- The query `WHERE` clause must logically imply the index predicate for PostgreSQL to use it.
- Using non-immutable expressions in the index predicate is not allowed.
- Do not use for very volatile predicates — too many index updates.

### Memory Aid

> Partial index = index with a bouncer: only rows that pass the `WHERE` get in.  
> Tiny index, tiny maintenance cost, query must "know" it qualifies.

---

## 11. Cost Estimation

The optimizer assigns **costs** to each possible execution plan:

- **I/O cost**: how many pages must be read.
- **CPU cost**: how many rows must be compared.
- **Memory cost**: sorting, joining, caching.

Decision: if `Index Scan cost < Sequential Scan cost` → use the index.

- **Index Scan cost** = index traversal + random table lookups.
- **Sequential Scan cost** = reading the entire table sequentially.

#### 12. Statistics & the Planner

### How the Planner Estimates

The planner relies on **table statistics** to estimate row counts and choose execution plans. Stale or inaccurate statistics → bad plans → bad performance.

### ANALYZE

Collects statistics about table contents and stores them in `pg_statistic`.

```sql
-- Analyze a specific table
ANALYZE orders;

-- Analyze all tables
ANALYZE;

-- Analyze a specific column
ANALYZE orders (customer_id);
```

> **Autovacuum** runs `ANALYZE` automatically, but after bulk loads or major data changes, run it manually.

### pg_stats — Viewing Statistics

`pg_stats` is a readable view over the internal `pg_statistic` catalog.

```sql
SELECT
    attname,
    n_distinct,        -- estimated distinct values (-1 = unique, 0.5 = half of rows)
    null_frac,         -- fraction of NULL values (0.0 to 1.0)
    avg_width,         -- average column width in bytes
    most_common_vals,  -- most frequent values
    most_common_freqs, -- frequencies of the most common values
    histogram_bounds   -- evenly-spaced boundary values for non-MCV values
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

| Statistic | What It Tells the Planner |
|-----------|--------------------------|
| `n_distinct` | How many unique values to expect. Affects group count estimates. |
| `null_frac` | What fraction is NULL. Affects `IS NULL`/`IS NOT NULL` estimates. |
| `most_common_vals` + `freqs` | Exact frequencies for top values. Enables accurate estimates for `WHERE status = 'pending'`. |
| `histogram_bounds` | Distribution of non-MCV values. Enables accurate range estimates. |
| `correlation` | How well physical row order matches logical order. Affects index vs seq scan decisions. |

### default_statistics_target

Controls the granularity of statistics histograms. Default is 100 (100 histogram buckets + up to 100 MCV entries).

```sql
-- Check current target
SHOW default_statistics_target;  -- 100

-- Increase for a specific column (skewed data benefits from more granularity)
ALTER TABLE orders ALTER COLUMN status SET STATISTICS 500;

-- Then re-collect
ANALYZE orders;
```

Costs:
- Higher target → more accurate estimates → better plans.
- Higher target → `ANALYZE` takes longer and stores more data in `pg_statistic`.
- **Rule of thumb**: Increase for columns with highly skewed distributions that appear in WHERE clauses.

### Extended Statistics (PG 10+)

Standard per-column statistics miss **cross-column correlations**. Extended statistics fix this.

```sql
-- Problem: planner assumes city and zip_code are independent
-- Estimates: P(city='Hanoi') × P(zip='100000') → far too low
-- Reality: city and zip are perfectly correlated

-- Fix: create extended statistics
CREATE STATISTICS stat_city_zip (dependencies) ON city, zip_code FROM addresses;
ANALYZE addresses;

-- Now the planner knows city and zip_code are correlated
```

Types of extended statistics:
| Type | What It Captures |
|------|-----------------|
| `dependencies` | Functional dependencies between columns (A determines B) |
| `ndistinct` | Number of distinct values for column combinations |
| `mcv` | Most common value lists for column combinations (PG 12+) |

```sql
-- Full extended statistics (all types)
CREATE STATISTICS stat_orders_cust_status (dependencies, ndistinct, mcv)
ON customer_id, status FROM orders;
ANALYZE orders;

-- Inspect
SELECT * FROM pg_statistic_ext WHERE stxname = 'stat_orders_cust_status';
```

### Planner Cost Parameters

These settings tune how the planner estimates I/O and CPU costs. **Adjust to match your hardware.**

| Parameter | Default | Meaning | Tune For |
|-----------|---------|---------|----------|
| `seq_page_cost` | 1.0 | Cost of a sequential page read | Baseline (usually leave at 1.0) |
| `random_page_cost` | 4.0 | Cost of a random page read | **Lower for SSD** (1.1–1.5) |
| `cpu_tuple_cost` | 0.01 | Cost of processing one row | Rarely changed |
| `cpu_index_tuple_cost` | 0.005 | Cost of processing one index entry | Rarely changed |
| `cpu_operator_cost` | 0.0025 | Cost of executing one operator/function | Rarely changed |
| `effective_cache_size` | 4GB | Estimated total cache (PG buffers + OS cache) | Set to ~50-75% of RAM |
| `parallel_tuple_cost` | 0.1 | Cost of transferring a tuple from worker to leader | Lower to encourage parallelism |

```sql
-- For SSDs: lower random I/O cost so planner prefers indexes more
SET random_page_cost = 1.1;

-- Tell planner how much cache is available
SET effective_cache_size = '24GB';

-- Check effective values
SELECT name, setting, unit FROM pg_settings
WHERE name IN ('random_page_cost', 'effective_cache_size', 'seq_page_cost');
```

> **Key insight**: If your queries default to sequential scans on SSD when you expect index scans, lower `random_page_cost` to 1.1–1.5. On SSD, random reads are nearly as fast as sequential reads.

### Diagnosing Bad Estimates

```sql
-- Run EXPLAIN ANALYZE and compare estimated vs actual rows
EXPLAIN ANALYZE SELECT * FROM orders WHERE status = 'pending';

-- If estimated rows = 10000, actual rows = 50:
-- 1. Run ANALYZE orders;
-- 2. Check pg_stats for accuracy
-- 3. Consider extended statistics if multiple columns are correlated
-- 4. Increase statistics_target for that column

-- Quick diagnostic query
SELECT
    relname,
    n_live_tup,                               -- actual rows
    n_dead_tup,                               -- dead rows awaiting vacuum
    last_analyze,                             -- when stats were last collected
    last_autoanalyze
FROM pg_stat_user_tables
WHERE relname = 'orders';
```

---

## 13. How UUIDs Affect Performance

### The Problem

Random UUIDs in B-tree indexes **hurt write performance** due to:
- **Random inserts** that cause page splits throughout the tree.
- **Large key size** (16 bytes vs 4 bytes for `INTEGER` or 8 bytes for `BIGINT`).

### Background

PostgreSQL's default index type (B-tree) works best when inserted keys are:
- **Sequentially increasing** — new entries go to the "rightmost" page.
- **Clustered** — related keys are close in physical order.

A **UUID** (especially v4) is:
- **Randomly distributed** by design.
- **16 bytes per key**, larger than `INTEGER` (4 bytes) or `BIGINT` (8 bytes).

### Impact on Insert Performance

- Each new random UUID lands in a **random spot** in the B-tree.
- This causes **page splits**, **random I/O**, and **cache misses**.
- As the table grows, inserts can become **3–5× slower** than with sequential keys (`BIGSERIAL` or monotonic sequences).

### Recommendation

- Prefer **sequential UUIDs** (v1 or v7) that include a timestamp component.
- Use **integers** (`BIGSERIAL`) for internal-only keys.
- Reserve random UUIDs (v4) for external-facing IDs where unpredictability is required.

---

## 14. Query Optimization Techniques

### Keyset Pagination (Seek Method)

```sql
-- ❌ Slow for deep pages: OFFSET scans and discards rows
SELECT * FROM events ORDER BY id LIMIT 20 OFFSET 1000000;

-- ✅ Fast: keyset pagination uses the index directly
SELECT * FROM events
WHERE id > $last_seen_id
ORDER BY id
LIMIT 20;
```

| Method | Complexity | Why |
|--------|-----------|-----|
| `OFFSET N` | O(N) | Must scan and discard N rows |
| Keyset (`WHERE id > X`) | O(log N) | Index seek directly to the position |

### Batch Lookups

```sql
-- ❌ Slow: N separate queries in a loop
SELECT * FROM products WHERE id = 1;
SELECT * FROM products WHERE id = 2;
-- ... 1000 times

-- ✅ Fast: single query with ANY
SELECT * FROM products WHERE id = ANY(ARRAY[1, 2, 3, ..., 1000]);

-- ✅ Fast: use unnest + JOIN for very large batches
SELECT p.* FROM products p
JOIN unnest(ARRAY[1, 2, 3, ..., 1000]) AS t(id) ON p.id = t.id;

-- ✅ Fast: bulk INSERT with VALUES
INSERT INTO products (name, price) VALUES
    ('A', 10), ('B', 20), ('C', 30);  -- single round-trip
```

### Prepared Statements

Reuse query plans for repeated similar queries.

```sql
-- Prepare a parameterized statement
PREPARE get_orders(integer) AS
SELECT * FROM orders WHERE customer_id = $1;

-- Execute with specific parameters
EXECUTE get_orders(42);
EXECUTE get_orders(99);

-- Deallocate when done
DEALLOCATE get_orders;
```

**Plan types**:
- **Custom plan** (first 5 executions) — optimized for the specific parameter values.
- **Generic plan** (6th execution onward) — reused plan if it's no worse than custom plans on average.

> **Gotcha**: A generic plan can be suboptimal for skewed data distributions. Force custom plans with `plan_cache_mode = force_custom_plan` if needed.

### OR to UNION Rewrite

```sql
-- ❌ OR on different columns: planner may choose seq scan
SELECT * FROM orders WHERE customer_id = 42 OR status = 'urgent';

-- ✅ UNION ALL: each branch can use its own index
SELECT * FROM orders WHERE customer_id = 42
UNION ALL
SELECT * FROM orders WHERE status = 'urgent' AND customer_id != 42;
```

---

## 15. The Cost of Long-Running Transactions

### How PostgreSQL Handles Row Versions

PostgreSQL uses **MVCC** — it creates a new **version** of any row that a transaction touches. This means:

- If a long transaction creates **millions of rows** and then **rolls back**, those millions of row versions are now **dead** (invalid).

### What Happens to Dead Rows

- PostgreSQL runs a **VACUUM** command periodically to remove dead rows and free up space on pages.
- Until VACUUM runs, dead rows still **physically exist** on disk.

### What If VACUUM Hasn't Run Yet?

The dead rows are **still filtered out correctly** — new transactions will not read them. PostgreSQL checks the **transaction state** (via `xmin`/`xmax`) of the transaction that created each row.

**However**, this check is **expensive**:
- Every query must inspect and skip over dead rows.
- More dead rows = more checking = **slower performance**.

> **Plain English**: Long-running transactions that create or modify lots of rows (and then fail) leave behind "garbage" that slows down everyone else until VACUUM cleans it up.

### Best Practices

- Keep transactions **as short as possible**.
- Monitor **autovacuum** to ensure it keeps up with dead row accumulation.
- Avoid massive batch operations inside a single transaction — break them into smaller chunks.

---

## 16. Operator Classes and Operator Families

### Intuition

- Not every data type has an obvious "sort" order — what does it mean to compare two JSONB documents? Two IP addresses?
- Operator classes tell PostgreSQL: "when using this index on this column, use *these* comparison operators."
- Most of the time the defaults just work; this becomes relevant with custom types or full-text search.

### What

- [Official docs] An operator class specifies which operators can use an index and what semantics they express.
- Every index column has an operator class — usually implicit from the data type.
- An operator family groups related operator classes that are compatible with each other.

### How

```sql
-- Default: PostgreSQL picks the right operator class automatically
CREATE INDEX idx_name ON users (name);

-- Explicit operator class: case-insensitive text comparison
CREATE INDEX idx_name_ci ON users (name text_pattern_ops);
-- text_pattern_ops: enables LIKE 'prefix%' matching with locale-independent comparisons

-- For LIKE queries across locales
CREATE INDEX idx_name_pattern ON users (name varchar_pattern_ops);

-- Check available operator classes
SELECT am.amname, opc.opcname
FROM pg_opclass opc
JOIN pg_am am ON opc.opcmethod = am.oid
WHERE am.amname = 'btree'
ORDER BY opc.opcname;
```

### Pitfalls

- Using a `C` locale makes `text_pattern_ops` unnecessary (the default covers LIKE).
- In other locales, `LIKE 'prefix%'` does NOT use a plain text B-tree index — you need `text_pattern_ops` or `varchar_pattern_ops`.

### Memory Aid

> Operator class = "rules for comparing this column in this index."  
> Usually invisible, but matters for LIKE, locale, and custom types.

---

## 17. Index-Only Scans and Covering Indexes

### Intuition

- Normally an index tells PostgreSQL *where* rows are, then PostgreSQL fetches those rows from the heap (table pages).
- An **index-only scan** skips the heap entirely — all needed data is already in the index.
- A **covering index** adds extra columns to the index (`INCLUDE`) so the heap fetch is unnecessary.

### What

- [Official docs] An index-only scan is possible when all columns needed by the query are present in the index.
- PostgreSQL still must check the **visibility map** to confirm rows are visible (heap is not accessed for data, but may be for visibility on recently modified pages).
- `INCLUDE` columns are stored in leaf pages but not used for ordering or searching.

### Requirements

- All `SELECT`, `WHERE`, and `ORDER BY` columns must be in the index (key or `INCLUDE`).
- Visibility map must be current (run `VACUUM` if `Heap Fetches` is high in EXPLAIN output).

### How

```sql
-- Without INCLUDE: must fetch heap to get 'name'
CREATE INDEX idx_users_email ON users (email);
SELECT name FROM users WHERE email = 'alice@example.com';
-- Index Scan (heap fetch needed for 'name')

-- With INCLUDE: name is in the index leaf
CREATE INDEX idx_users_email_cover ON users (email) INCLUDE (name);
SELECT name FROM users WHERE email = 'alice@example.com';
-- Index Only Scan (no heap fetch needed)
```

### Example — Real-World Flow

```sql
-- API: look up username and avatar_url by session token (hot path)
CREATE UNIQUE INDEX idx_sessions_token
ON sessions (token)
INCLUDE (user_id, username, avatar_url);

SELECT user_id, username, avatar_url
FROM sessions
WHERE token = 'abc123xyz';
-- Index Only Scan: entire result comes from index, zero heap access
```

### Pitfalls

- `INCLUDE` columns add size to the index — do not add large text/JSONB columns needlessly.
- Index-only scan still hits the heap if the visibility map is not current (shortly after bulk writes).
- Running `VACUUM` regularly keeps the visibility map fresh and makes index-only scans more effective.

### Memory Aid

> Index-only scan = query answered entirely from the index "card catalog."  
> `INCLUDE` = staple the extra data you need directly to the card.

---

## What to Learn Next

1. **B-Tree and B+Tree internals** — How the tree structure works at the page level — see [04-btree.md](04-btree.md).
2. **Transactions & Concurrency** — ACID, MVCC, isolation levels — see [01-acid.md](01-acid.md), [02-transactions.md](02-transactions.md).
3. **VACUUM and Autovacuum tuning** — Controlling dead row cleanup — see [26-monitoring-diagnostics-bloat.md](26-monitoring-diagnostics-bloat.md).
4. **`pg_stat_statements`** — Find the slowest queries, then decide which indexes to build.
5. **Configuration Tuning** — `shared_buffers`, `work_mem`, `maintenance_work_mem` — see [25-performance-tuning.md](25-performance-tuning.md).
6. **Covering indexes in depth** — `INCLUDE` design patterns for hot API paths.
7. **GIN tuning** — `fastupdate`, `gin_pending_list_limit` for high-write JSONB workloads.
8. **`pgvector` extension** — HNSW and IVFFlat index types for machine learning vector similarity search.

---

> *Ref: [Docs — Chapter 11: Indexes](https://www.postgresql.org/docs/18/indexes.html) · [Docs — EXPLAIN](https://www.postgresql.org/docs/18/sql-explain.html) · [Docs — Planner Statistics](https://www.postgresql.org/docs/18/planner-stats.html) · [Docs — Planner Cost Constants](https://www.postgresql.org/docs/18/runtime-config-query.html) · [Neon — PostgreSQL Indexes](https://neon.com/postgresql/postgresql-indexes)*
