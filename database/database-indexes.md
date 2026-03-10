# Indexes

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> How database indexes work, how the query optimizer decides to use them, scanning methods, composite indexes, UUID performance impact, and the cost of long-running transactions.

---

## Table of Contents

1. [What Is an Index?](#1-what-is-an-index)
2. [How the Query Optimizer Decides to Use Indexes](#2-how-the-query-optimizer-decides-to-use-indexes)
3. [Understanding the Query Planner with EXPLAIN](#3-understanding-the-query-planner-with-explain)
4. [Scanning Methods](#4-scanning-methods)
5. [Composite Index](#5-composite-index)
6. [How UUIDs Affect Performance](#6-how-uuids-affect-performance)
7. [The Cost of Long-Running Transactions](#7-the-cost-of-long-running-transactions)

---

## 1. What Is an Index?

An **index** is a data structure **separate from the heap** that stores **pointers (TIDs)** to the actual rows in the heap. Its purpose is to locate data quickly without scanning every page.

### Key Properties

- Stored on **disk** alongside the heap.
- Contains entries mapping an **indexed value** to a **page number + row offset** in the heap.
- Each primary key in PostgreSQL has an **index by default**.

### How an Index Lookup Works

```
B-Tree Index (disk)              Heap (disk)
┌───────────────────┐            ┌──────────────────┐
│  Key → (page,     │── IO #1 ──▶│  Page N           │
│         row_id)   │            │  row_id = offset  │── IO #2 ──▶ full row data
└───────────────────┘            └──────────────────┘
```

1. **IO #1** — Traverse the index to find `(page_number, row_id)` for the target value.
2. **IO #2** — Jump directly to that page in the heap and read the row.

### Important Notes

- The index itself is stored as pages and **costs I/O** to read.
- **Smaller indexes = faster searches** — fewer pages to traverse.
- Each primary key column in PostgreSQL has an index **created automatically**.

### Index Scan vs Index Only Scan

| Scan Type | Behavior |
|-----------|----------|
| **Index Scan** | Reads the index to find TIDs, then **jumps to the heap** to retrieve full row data. |
| **Index Only Scan** | Reads the index and retrieves data **directly from the index** — no heap access needed. Only works when the query selects columns that are **all part of the index**. |

---

## 2. How the Query Optimizer Decides to Use Indexes

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

#### 1. Statistics Collection

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

#### 2. Selectivity

**Selectivity** = the fraction of rows that match the condition.

- **High selectivity (few rows match)** → index is efficient.
- **Low selectivity (many rows match)** → index scan may be slower than a full table scan.

```sql
SELECT * FROM users WHERE user_id = 42;
-- user_id is unique → highly selective → use index ✅

SELECT * FROM users WHERE is_active = true;
-- 95% of users are active → low selectivity → full table scan is cheaper ✅
```

#### 3. Cost Estimation

The optimizer assigns **costs** to each possible execution plan:

- **I/O cost**: how many pages must be read.
- **CPU cost**: how many rows must be compared.
- **Memory cost**: sorting, joining, caching.

Decision: if `Index Scan cost < Sequential Scan cost` → use the index.

- **Index Scan cost** = index traversal + random table lookups.
- **Sequential Scan cost** = reading the entire table sequentially.

#### 4. Query Pattern Compatibility

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

#### 5. Index Type Matters

| Index Type | Best Use Case |
|------------|--------------|
| **B-tree** | Equality, range, sorting |
| **Hash** | Exact matches only |
| **GIN / GiST** | Full-text search, JSON, geometric data |
| **BRIN** | Huge tables with physically sorted data |

#### 6. Composite Index Rules

For a multi-column index `(a, b, c)`, only **leftmost prefixes** are usable:

```sql
WHERE a = 1                ✅ uses (a, b, c)
WHERE a = 1 AND b = 2     ✅ uses (a, b, c)
WHERE b = 2               ❌ cannot use (a, b, c)
```

#### 7. Joins and Sorting

- If a query joins on an **indexed foreign key** → optimizer can use an **index nested loop join**.
- If `ORDER BY` matches the **index order** → skip sorting entirely.
- Otherwise, a full table scan + sort might be cheaper.

#### 8. Caching and Parallelism

The optimizer considers whether data is likely **already in the buffer pool** (cached in memory) and whether **parallel workers** can be used to speed up the scan.

---

## 3. Understanding the Query Planner with EXPLAIN

### What

`EXPLAIN` retrieves information about what execution plan PostgreSQL will use for a given SQL statement.

### Reading EXPLAIN Output

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

> This is the **fastest scan type** when applicable. Design indexes with `INCLUDE` columns to enable more Index Only Scans.

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

## 5. Composite Index

### What

An index built on **two or more columns**.

### How It Works

For a composite index `(A, B)`, PostgreSQL can only scan from the **left-hand side**:

| Query Filter | Uses Index? |
|-------------|------------|
| Filter by `A` | ✅ Index Scan |
| Filter by `A` and `B` | ✅ Index Scan |
| Filter by `B` only | ❌ Cannot use this index |

> **Rule**: Composite indexes follow the **leftmost prefix rule**. The index `(A, B, C)` supports queries filtering on `A`, or `A+B`, or `A+B+C` — but NOT `B` alone, `C` alone, or `B+C`.

---

## 6. How UUIDs Affect Performance

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

## 7. The Cost of Long-Running Transactions

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

## What to Learn Next

1. **B-Tree and B+Tree internals** — How the tree structure works at the page level.
2. **Partial Indexes** — Indexes that only cover a subset of rows (`CREATE INDEX ... WHERE ...`).
3. **Covering Indexes** — Using `INCLUDE` to add non-key columns to enable Index Only Scans.
4. **VACUUM and Autovacuum tuning** — Controlling dead row cleanup.
5. **GIN and GiST indexes** — For full-text search, JSONB, and geometric queries.
