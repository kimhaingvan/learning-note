# Indexes — PostgreSQL 18 (Chapter 11)

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> **Source**: [PostgreSQL 18 Docs — Chapter 11](https://www.postgresql.org/docs/18/indexes.html)

---

## Table of Contents

1. [What Is an Index?](#1-what-is-an-index)
2. [Index Types](#2-index-types)
   - 2.1 B-tree
   - 2.2 Hash
   - 2.3 GiST
   - 2.4 SP-GiST
   - 2.5 GIN
   - 2.6 BRIN
   - 2.7 Bloom
3. [Multicolumn Indexes](#3-multicolumn-indexes)
4. [Indexes and ORDER BY](#4-indexes-and-order-by)
5. [Combining Multiple Indexes](#5-combining-multiple-indexes)
6. [Unique Indexes](#6-unique-indexes)
7. [Indexes on Expressions](#7-indexes-on-expressions)
8. [Partial Indexes](#8-partial-indexes)
9. [Index-Only Scans and Covering Indexes](#9-index-only-scans-and-covering-indexes)
10. [Operator Classes and Operator Families](#10-operator-classes-and-operator-families)
11. [Examining Index Usage](#11-examining-index-usage)

---

## 1. What Is an Index?

### Intuition

- Imagine scanning 10,000,000 rows to find one — a table without an index does exactly that.
- An index is a **shortcut** built on one or more columns: PostgreSQL first looks up the index, finds the exact location of matching rows, and only reads those.
- Think of it as the index pages at the back of a textbook — instead of reading every page, you go directly to the right one.

### What

- [Official docs] An index is a data structure maintained separately from the table heap.
- It stores a **sorted or structured copy** of one or more column values, together with **TIDs** (tuple identifiers / physical row pointers).
- PostgreSQL automatically uses indexes when the query planner decides the index scan is cheaper than a sequential scan.

### Why

- **Speed** — index lookup is O(log n) instead of O(n) for large tables.
- **Uniqueness enforcement** — `UNIQUE` indexes prevent duplicate values.
- **Foreign key speed** — FK constraint checks use indexes on both sides.

### When

- Large frequently-queried tables.
- Columns used in `WHERE`, `JOIN`, `ORDER BY`, `GROUP BY`.
- Columns with **high cardinality** (many distinct values) benefit most.
- Small tables may be slower with an index than without — the planner decides.

### How

```sql
-- Basic syntax
CREATE INDEX index_name ON table_name (column_name);

-- With index type
CREATE INDEX idx_email ON users USING btree (email);

-- Remove an index
DROP INDEX index_name;

-- Rebuild without blocking reads/writes
REINDEX INDEX CONCURRENTLY index_name;

-- Non-blocking index creation (does not lock writes)
CREATE INDEX CONCURRENTLY idx_email ON users (email);
```

> **`CONCURRENTLY`**: builds the index without holding a lock that blocks writes. Takes longer and slightly more resources, but safe for production.

### Example

```sql
-- Table with 5 million orders
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    customer_id BIGINT,
    status      TEXT,
    created_at  TIMESTAMPTZ
);

-- Without index: sequential scan of 5M rows every time
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- Seq Scan on orders (cost=0.00..98563.00 rows=3 ...)

-- Create index
CREATE INDEX idx_orders_customer ON orders (customer_id);

-- With index: ~5 rows fetched via index lookup
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;
-- Index Scan using idx_orders_customer on orders (cost=0.43..12.90 rows=3 ...)
```

### Pitfalls

- Creating indexes on every column — indexes have a **write cost** on INSERT/UPDATE/DELETE.
- Forgetting that small tables are usually better with sequential scans.
- Not using `CONCURRENTLY` on active production tables.
- Unused indexes still slow down writes and consume storage.

### Memory Aid

> Index = sorted shortcut to physical rows.  
> Fast to read along, expensive to maintain on write.

### Check

1. What does a TID in an index entry point to?
2. When does the planner ignore an index even though it exists?
3. What is the difference between `CREATE INDEX` and `CREATE INDEX CONCURRENTLY`?
4. Mini practice: Create an index on `orders.created_at` without blocking production writes.

---

## 2. Index Types

### 2.1 B-tree

#### Intuition

- The default. Handles ordered data: ranges, equality, sorting.
- Think of it as a sorted tree where every leaf holds the value + a pointer to the row.

#### What

- [Official docs] B-tree indexes can handle equality and range queries on data that can be sorted.
- Supports operators: `<`, `<=`, `=`, `>=`, `>`, `BETWEEN`, `IN`, `IS NULL`, `IS NOT NULL`, `LIKE 'foo%'` (prefix only).

#### How

```sql
-- Default — B-tree is assumed
CREATE INDEX idx_price ON products (price);

-- Explicit
CREATE INDEX idx_price ON products USING btree (price);

-- Descending
CREATE INDEX idx_created_desc ON events (created_at DESC);
```

#### Real-World Flow

```sql
-- Range query — B-tree is ideal
SELECT * FROM events
WHERE created_at BETWEEN '2026-01-01' AND '2026-03-01';

-- LIKE prefix match — B-tree helps
SELECT * FROM users WHERE username LIKE 'kim%';
-- LIKE '%kim' or LIKE '%kim%' → B-tree NOT used (no leading anchor)
```

#### Memory Aid

> B-tree = sorted bookshelf. Works for any "is it before/after/equal?" question.

---

### 2.2 Hash

#### Intuition

- Equality only. Extremely fast for `=`, useless for everything else.
- Think of it as a dictionary lookup — goes straight to the bucket.

#### What

- [Official docs] Hash indexes store a hash of the column value. Only useful for `=` comparisons.
- Smaller than B-tree for purely equality workloads.

#### How

```sql
CREATE INDEX idx_session ON sessions USING hash (session_token);

-- Only helps with:
SELECT * FROM sessions WHERE session_token = 'abc123';
-- Does NOT help with:
SELECT * FROM sessions WHERE session_token > 'abc123';
```

#### Memory Aid

> Hash = dictionary lookup. Only for `=`, nothing else.

---

### 2.3 GiST (Generalized Search Tree)

#### Intuition

- A framework for custom index types. GiST itself is not one index — it is the scaffolding.
- Used for geometric shapes, full-text search, IP ranges, and more.

#### What

- [Official docs] GiST implements a balanced tree structure and provides an interface for user-defined data types to plug in their own comparison logic.
- Supports operators like: `&&` (overlap), `@>` (contains), `<@` (contained by), `~=` (same), `<<` (left of).

#### How

```sql
-- Geospatial (PostGIS — uses GiST internally)
CREATE INDEX idx_location ON places USING gist (location);

-- Range type overlap query
CREATE INDEX idx_booking ON reservations USING gist (during);
SELECT * FROM reservations WHERE during && '[2026-03-01, 2026-03-10)'::tsrange;

-- Full-text with tsvector
CREATE INDEX idx_fts ON articles USING gist (to_tsvector('english', body));
```

#### Memory Aid

> GiST = plug-in tree for complex types: shapes, ranges, text vectors.

---

### 2.4 SP-GiST (Space-Partitioned GiST)

#### Intuition

- Like GiST but for data with natural hierarchical or partitioned structure (quad-trees, trie-based structures).
- Good when the data partitions unevenly.

#### What

- [Official docs] SP-GiST supports partitioned search trees — quad-trees, k-d trees, radix/prefix trees.
- Useful for: 2D geometries, IP addresses (CIDR-prefix trees), text prefix matching.

#### How

```sql
-- IP address range
CREATE INDEX idx_network ON ip_routes USING spgist (prefix inet_ops);

SELECT * FROM ip_routes WHERE prefix >>= '192.168.1.100'::inet;
```

#### Memory Aid

> SP-GiST = tree for naturally partitioned/hierarchical data (tries, kd-trees).

---

### 2.5 GIN (Generalized Inverted Index)

#### Intuition

- When one row contains *many* searchable items (array elements, JSONB keys, text words), you need to look up "which rows contain this element?" — that is an inverted index.
- Think of it as the opposite of a B-tree: instead of "for this row, what's its value?", it says "for this value, which rows contain it?"

#### What

- [Official docs] GIN is optimized for cases where the index key contains multiple values (arrays, JSONB, tsvectors).
- Fast for: `@>`, `<@`, `?`, `?|`, `?&`, `@@` (full-text).
- Larger than GiST but faster at lookup.
- Writes are slower (pending list + cleanup). `fastupdate = on` (default) batches writes.

#### How

```sql
-- JSONB containment
CREATE INDEX idx_tags ON products USING gin (tags);
SELECT * FROM products WHERE tags @> '["electronics", "sale"]';

-- Full-text search
CREATE INDEX idx_fts ON articles USING gin (to_tsvector('english', body));
SELECT * FROM articles WHERE to_tsvector('english', body) @@ to_tsquery('postgres & index');

-- Array containment
CREATE INDEX idx_categories ON posts USING gin (categories);
SELECT * FROM posts WHERE categories @> ARRAY['backend', 'database'];
```

#### Memory Aid

> GIN = inverted index. When one row contains many things, and you want to find rows by any of those things.

---

### 2.6 BRIN (Block Range Index)

#### Intuition

- For enormous tables where data is physically ordered as it was inserted (e.g., event logs, time-series).
- BRIN records only the MIN and MAX value per disk block range — tiny footprint, but only works when data has natural physical order.
- Think of it as "I know pages 1–100 hold January data, pages 101–200 hold February data" — a range map.

#### What

- [Official docs] BRIN stores min/max (or other summary) per range of table pages (`pages_per_range`, default 128).
- Very small index. Works well when column value correlates with physical tuple insertion order.
- Supports: `=`, `<`, `<=`, `>=`, `>`.

#### How

```sql
-- Time-series log table
CREATE TABLE events (
    id         BIGSERIAL PRIMARY KEY,
    ts         TIMESTAMPTZ NOT NULL DEFAULT now(),
    payload    JSONB
);

CREATE INDEX idx_events_brin ON events USING brin (ts);

-- Works well if rows are inserted in time order
SELECT COUNT(*) FROM events WHERE ts > '2026-03-01';
```

#### Pitfalls

- BRIN gives wrong answers if the physical and logical order are very mismatched (e.g., `UPDATE` changes values but rows stay on the same pages).
- Better to use B-tree for randomly inserted timestamp columns.

#### Memory Aid

> BRIN = sticky note on each disk block saying "this block holds values X to Y." Tiny, but requires physical order.

---

### 2.7 Bloom

#### Intuition

- A probabilistic index perfect when you query across many columns simultaneously.
- Can produce false positives (rows that don't match slip through), so PostgreSQL re-checks, but no false negatives (matching rows are never missed).

#### What

- [Official docs] Bloom filters store a bitset per-row for multiple columns. Supports only `=` queries.
- Must `CREATE EXTENSION bloom;` first.
- Best when queries use `AND` across 5–20 columns with equality only.

#### How

```sql
CREATE EXTENSION bloom;

CREATE INDEX idx_bloom ON log_entries
USING bloom (col1, col2, col3, col4, col5)
WITH (length = 80, col1 = 2, col2 = 2);

SELECT * FROM log_entries WHERE col1 = 'A' AND col3 = 'C' AND col5 = 'E';
```

#### Memory Aid

> Bloom = "probably here." Checks multiple columns at once; may need re-check, never misses.

---

### Index Type Summary

| Index | Best For | Operators | Notes |
|-------|---------|-----------|-------|
| **B-tree** | Scalar values, ranges, sorts | `=`, `<`, `<=`, `>=`, `>`, `LIKE 'x%'` | Default |
| **Hash** | Equality only | `=` | Smaller than B-tree for `=` only workloads |
| **GiST** | Geometric, range, FTS | `&&`, `@>`, `<@`, `@@` | Framework; plug-in operator support |
| **SP-GiST** | Hierarchical / tree data | `<<`, `>>`, prefix ops | IP, quadtrees, tries |
| **GIN** | Multi-value items | `@>`, `?`, `@@`, `<@` | Arrays, JSONB, full-text |
| **BRIN** | Very large append-only tables | `=`, `<`, `>` | Tiny; needs physical ordering |
| **Bloom** | Multi-column equality queries | `=` | Probabilistic; extension required |

---

## 3. Multicolumn Indexes

### Intuition

- One index on multiple columns: `(last_name, first_name)`.
- The key rule is the **leftmost prefix rule**: the index is useful if your query filters on the leading column(s), with or without the trailing ones.
- Think of a phone book sorted by last name then first name: you can find "Smith, John" but also all "Smith" entries — but you cannot look up all "John" entries without knowing the last name.

### What

- [Official docs] A multicolumn B-tree index can be used for queries referencing any subset of columns, provided the leftmost column(s) are in the query.
- Column order matters: most selective (or most frequently filtered) column goes first.

### How

```sql
-- Composite index
CREATE INDEX idx_name ON employees (last_name, first_name);

-- Useful for:
SELECT * FROM employees WHERE last_name = 'Smith';
SELECT * FROM employees WHERE last_name = 'Smith' AND first_name = 'John';

-- NOT useful for (missing leftmost column):
SELECT * FROM employees WHERE first_name = 'John';
```

### Example — Real-World Flow

```sql
-- Orders table: frequently filtered by status and created_at together
CREATE INDEX idx_orders_status_date ON orders (status, created_at DESC);

-- Fast: uses full index
SELECT * FROM orders
WHERE status = 'pending'
ORDER BY created_at DESC
LIMIT 20;

-- Still fast: uses status prefix
SELECT COUNT(*) FROM orders WHERE status = 'pending';

-- Slow (ignores index): only created_at, status not provided
SELECT * FROM orders WHERE created_at > '2026-01-01';
-- ↑ planner may use index scan on (created_at) but not on (status, created_at)
```

### Pitfalls

- Putting low-cardinality column last when it is the primary filter.
- Creating redundant indexes — `(a, b)` already covers queries on `(a)`.
- Assuming the index is used even when the leftmost column is missing from the query.

### Memory Aid

> Multicolumn index = phone book. Works left to right, never right to left alone.

### Check

1. Does `CREATE INDEX ON t (a, b)` help a query `WHERE b = 5`?
2. What column should come first — `status` (5 values) or `customer_id` (1M values)?
3. Mini practice: Design an index for queries filtering on `country` and `created_at DESC`.

---

## 4. Indexes and ORDER BY

### Intuition

- B-tree indexes are already sorted — PostgreSQL can scan them in order and skip the sort step entirely.
- Getting an index to satisfy both `WHERE` and `ORDER BY` at once is one of the most important query design patterns.

### What

- [Official docs] A B-tree index can be scanned forward or backward to satisfy `ORDER BY` without an additional sort operation.
- PostgreSQL can use the index to provide pre-sorted output when: columns match, sort direction matches, NULLS ordering matches.

### How

```sql
-- Sort direction in index definition
CREATE INDEX idx_events_ts_desc ON events (ts DESC NULLS LAST);

-- Ascending (default)
CREATE INDEX idx_events_ts_asc ON events (ts ASC NULLS LAST);

-- Query that benefits from descending index
SELECT * FROM events ORDER BY ts DESC LIMIT 10;
```

### Example — Real-World Flow

```sql
-- Pagination pattern: get newest 20 orders for a customer
CREATE INDEX idx_orders_cust_date ON orders (customer_id, created_at DESC);

SELECT id, total, status, created_at
FROM orders
WHERE customer_id = 42
ORDER BY created_at DESC
LIMIT 20
OFFSET 0;

-- EXPLAIN shows "Index Scan Backward" — no sort needed
```

### Pitfalls

- Mixed sort directions in a query (`ORDER BY a ASC, b DESC`) — a single B-tree index on `(a, b)` cannot satisfy this without a sort step.
- Very large OFFSETs still force PostgreSQL to scan many index entries before skipping.

### Memory Aid

> Index = pre-sorted data. Match the index direction to your `ORDER BY` and the sort disappears from the plan.

---

## 5. Combining Multiple Indexes

### Intuition

- PostgreSQL can use more than one index in a single query and merge the results using bitmap operations.
- Think of it as two filtered lists: "rows where status = pending" AND "rows where amount > 100" — combine the lists, fetch only what appears in both.

### What

- [Official docs] PostgreSQL can combine multiple indexes using bitmap index scans.
- **Bitmap AND**: rows in both index results.
- **Bitmap OR**: rows in either index result.
- Each index scan builds an in-memory bitmap of matching heap pages, then they are ANDed/ORed together.

### How

```sql
-- Two separate indexes
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_amount ON orders (amount);

-- PostgreSQL may combine them for this query:
SELECT * FROM orders
WHERE status = 'pending' AND amount > 100;

-- EXPLAIN may show:
-- BitmapAnd
--   Bitmap Index Scan on idx_orders_status
--   Bitmap Index Scan on idx_orders_amount
```

### Pitfalls

- Relying on bitmap combination when a single composite index `(status, amount)` would be faster.
- Bitmap scan does an extra heap fetch step — for very selective queries, a single index scan is usually better.

### Memory Aid

> Bitmap combine = "find rows in list A AND/OR list B, then fetch."  
> Better to design one good composite index than rely on combining.

---

## 6. Unique Indexes

### Intuition

- Enforces that no two rows share the same value — a constraint baked into the index itself.
- When you write `UNIQUE` on a column or `PRIMARY KEY`, PostgreSQL silently creates a unique index.

### What

- [Official docs] A unique index guarantees no duplicate non-NULL entries exist in the indexed columns.
- `NULL` values are not considered equal — multiple NULLs are allowed in a unique index (by SQL standard).
- `PRIMARY KEY` = unique + not null.

### How

```sql
-- Explicit unique index
CREATE UNIQUE INDEX idx_users_email ON users (email);

-- This is equivalent to:
ALTER TABLE users ADD CONSTRAINT uq_email UNIQUE (email);

-- Multicolumn unique
CREATE UNIQUE INDEX idx_enrollment ON enrollments (user_id, course_id);

-- Partial unique
CREATE UNIQUE INDEX idx_unique_active_email
ON users (email)
WHERE deleted_at IS NULL;
-- Multiple deleted users can share email; only active users must be unique
```

### Example — Real-World Flow

```sql
-- UPSERT using conflict on unique index
INSERT INTO users (email, name)
VALUES ('alice@example.com', 'Alice')
ON CONFLICT (email)
DO UPDATE SET name = EXCLUDED.name;
```

### Pitfalls

- Assuming unique indexes prevent `NULL` duplicates (they do not, by standard).
- Forgetting to use partial unique indexes for soft-delete patterns.
- `ON CONFLICT` requires a specific index or constraint to target.

### Memory Aid

> UNIQUE index = constraint + fast lookup in one structure.  
> NULLs are never "equal" to each other — multiple NULLs are always allowed.

---

## 7. Indexes on Expressions

### Intuition

- What if your query always applies a function to a column — `LOWER(email)`, `date_trunc('month', ts)`?
- A regular index on the raw column is useless for those queries.
- Build the index *on the expression itself* so PostgreSQL can use it.

### What

- [Official docs] An index can be built on the result of an arbitrary expression over table columns.
- The index stores the **computed expression value**, not the raw column value.
- PostgreSQL uses the index when the same expression appears in a `WHERE` or `ORDER BY`.

### How

```sql
-- Case-insensitive email lookup
CREATE INDEX idx_users_lower_email ON users (LOWER(email));

-- Query MUST use the same expression:
SELECT * FROM users WHERE LOWER(email) = 'alice@example.com';
-- ✅ uses index

SELECT * FROM users WHERE email = 'Alice@Example.com';
-- ❌ does NOT use index — expression mismatch
```

### Example — Real-World Flow

```sql
-- Monthly billing grouping (expression on truncated date)
CREATE INDEX idx_orders_month ON orders (date_trunc('month', created_at));

SELECT date_trunc('month', created_at) AS month, SUM(total)
FROM orders
GROUP BY date_trunc('month', created_at);
-- Index makes this much faster on large tables

-- JSON field extraction
CREATE INDEX idx_payload_user ON events ((payload->>'user_id'));

SELECT * FROM events WHERE payload->>'user_id' = '42';
```

### Pitfalls

- Forgetting that the query must use **exactly the same expression** as the index definition.
- Expression indexes are more expensive to maintain (the expression is evaluated on every write).
- The expression must be **immutable** (no `now()`, `random()`, etc.).

### Memory Aid

> Expression index = pre-computed column. What you compute in `WHERE`, compute in the index definition instead.

---

## 8. Partial Indexes

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

## 9. Index-Only Scans and Covering Indexes

### Intuition

- Normally an index tells PostgreSQL *where* rows are, then PostgreSQL fetches those rows from the heap (table pages).
- An **index-only scan** skips the heap entirely — all needed data is already in the index.
- A **covering index** adds extra columns to the index (`INCLUDE`) so the heap fetch is unnecessary.

### What

- [Official docs] An index-only scan is possible when all columns needed by the query are present in the index.
- PostgreSQL still must check the **visibility map** to confirm rows are visible (heap is not accessed for data, but may be for visibility on recently modified pages).
- `INCLUDE` columns are stored in leaf pages but not used for ordering or searching.

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

## 10. Operator Classes and Operator Families

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

## 11. Examining Index Usage

### Intuition

- Creating indexes is easy. Knowing which indexes are actually being used (and which are dead weight) requires examining the query plan and system stats.
- Use `EXPLAIN (ANALYZE, BUFFERS)` to see what the planner chose, and `pg_stat_user_indexes` to see usage over time.

### What

- [Official docs] The `EXPLAIN` command shows the query plan chosen by the planner.
- `pg_stat_user_indexes` (`idx_scan`, `idx_tup_read`, `idx_tup_fetch`) shows real usage statistics.
- `pg_stat_user_tables` shows `seq_scan` vs. index-driven access per table.

### How

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

### Check

1. What column in `pg_stat_user_indexes` tells you if an index is being used?
2. What does `EXPLAIN (ANALYZE, BUFFERS)` add over plain `EXPLAIN`?
3. Why might a newly created index not appear in `idx_scan` stats yet?
4. Mini practice: Write a query to find all indexes on the `orders` table that have never been scanned and are larger than 1 MB.

---

## What to Learn Next

1. **`VACUUM` and the visibility map** — understand why regular `VACUUM` makes index-only scans faster.
2. **`pg_stat_statements`** — find the slowest queries, then decide which indexes to build.
3. **Covering indexes in depth** — `INCLUDE` design patterns for hot API paths.
4. **Partial indexes for soft-delete patterns** — keeps unique constraint flexible and index small.
5. **GIN tuning** — `fastupdate`, `gin_pending_list_limit` for high-write JSONB workloads.
6. **`pgvector` extension** — HNSW and IVFFlat index types for machine learning vector similarity search.
