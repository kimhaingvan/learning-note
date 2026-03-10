# Database General Concepts

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> A practical guide to understanding how databases store data, read/write rows, use indexes, and manage transactions.

---

## Table of Contents

1. [How Tables and Indexes Are Stored on Disk](#1-how-tables-and-indexes-are-stored-on-disk)
2. [Storage Concepts](#2-storage-concepts)
3. [Database Types: Row-based vs Column-based](#3-database-types-row-based-vs-column-based)
4. [Transactions](#4-transactions)

---

## 1. How Tables and Indexes Are Stored on Disk

### Table Organization

- A table is stored **row by row**, with **no inherent ordering**.
- When you insert data, each new row is appended after the previous one on disk.

---

### Reading a Row

When you read a row from a table, this is what happens step by step:

1. The database finds the **page** where the row lives.
2. It identifies the **file and byte offset** on disk where that page is located.
3. It asks the OS to read from that file at that offset for the **length of one page**.
4. The OS checks its **filesystem cache** — if the data is already there, it returns it immediately. Otherwise, it issues a physical **disk read** and loads the page into memory.
5. The database places the page into its own memory pool called the **buffer pool** (also called shared buffer pool).

> **Key insight**: Once a page is in the buffer pool, you get access to **all rows on that page**, not just the one you requested. This is why smaller rows are more efficient — more rows fit per page, giving you more data per I/O operation.

---

### Writing a Row

When you write (insert/update/delete) a row:

- The database modifies the relevant **page in the buffer pool** first (in memory).
- The change is written to the **WAL (Write-Ahead Log)** for durability.
- The actual page on disk is updated later in a background flush process.

> This is why writes are fast — they hit memory first, not disk directly.

---

## 2. Storage Concepts

### Table

The **logical structure** that holds your data, organized into rows and columns. This is what you interact with via SQL.

---

### Row ID (`row_id`)

- An **internal, system-maintained** identifier for each row.
- In **MySQL** (InnoDB): the `row_id` is the same as the **primary key**.
- In **PostgreSQL**: there is a hidden system column called `ctid` (tuple ID) that encodes `(page_number, row_index_within_page)`.

> You rarely interact with `row_id` directly, but the database uses it internally to locate rows on disk.

---

### Page

A **fixed-size block** of data — the smallest unit the database reads/writes at a time.

| Database   | Page Size |
|------------|-----------|
| PostgreSQL | 8 KB      |
| MySQL      | 16 KB     |

**Why it matters**:
- The database never reads a single row in isolation — it always reads a full page.
- A single page can hold **many rows**, so one I/O can return a lot of data.
- `SELECT *` and `SELECT name` cost the **same I/O** — the database fetches the whole page, then discards what you don't need.

---

### I/O (Input/Output)

An **I/O** is a single read or write request made to disk.

- I/O is the **most expensive operation** in a database — it's orders of magnitude slower than reading from memory.
- The fewer I/Os a query requires, the faster it runs.
- Everything in database performance optimization revolves around **reducing I/O**.

> **Plain English**: Think of I/O like a trip to a warehouse to pick up a box. Every trip costs time. You want to pick up as much as possible in one trip.

---

### Heap (Data Structure)

The **heap** is the main storage area where your actual table data lives on disk.

```
Heap (disk storage)
┌─────────────────────────────────┐
│  Page 1                         │
│  [row1][row2][row3][row4]...    │
├─────────────────────────────────┤
│  Page 2                         │
│  [row5][row6][row7][row8]...    │
├─────────────────────────────────┤
│  Page 3 ...                     │
└─────────────────────────────────┘
```

- Rows are stored **in insertion order** — there is no sorting.
- To find a specific row without an index, the database must **scan every page** (called a **sequential scan** or **full table scan**).

---

### Index (B-Tree Data Structure)

An **index** is a **separate data structure** (stored on disk alongside the heap) that holds **pointers back into the heap**.

#### Why Indexes Exist

Without an index, finding a row requires scanning every page — O(n) time. An index provides O(log n) lookup.

#### How a B-Tree Index Works

```
B-Tree Index (disk)          Heap (disk)
┌─────────────────┐          ┌──────────────┐
│  Index entries  │          │  Page 3      │
│  value → (page, │─ IO #1 ─▶│  row_id = 42 │─ IO #2 ─▶ full row data
│  row_id)        │          └──────────────┘
└─────────────────┘
```

**Two-step lookup process**:
1. **IO #1** — Search the index B-tree to find `(page_number, row_id)` for the desired value.
2. **IO #2** — Go to the heap at that exact page and row to fetch the actual data.

#### Key Notes

- The index itself is stored as pages and costs I/O to read.
- **Smaller indexes = faster searches** — fewer pages to read.
- The database uses the index to jump **directly** to the right page in the heap, instead of scanning all pages.

---

### Primary Key vs Secondary Key

| Feature          | Primary Key                              | Secondary Index                          |
|------------------|------------------------------------------|------------------------------------------|
| Uniqueness       | Must be unique, not null                 | Can be non-unique                        |
| Storage (MySQL)  | Data is **physically sorted** by PK (clustered) | Stores PK value as pointer to heap |
| Storage (Postgres)| Separate B-tree index, heap is unordered | Same structure as any other index       |
| Lookup cost      | Single B-tree lookup (MySQL: direct data)| B-tree lookup → then heap lookup        |

---

### Example: What Happens on a Query

```sql
SELECT name FROM employees WHERE age = 30;
```

**Without index on `age`**:
- Database scans every page in the heap.
- Reads each row, checks if `age = 30`.
- Discards rows that don't match.
- Returns only the `name` column — but the full page was still loaded.

**With index on `age`**:
1. Traverse B-tree index → find `(page=5, row_id=12)`.
2. Go directly to Page 5, Row 12 in the heap.
3. Return `name`.

> Cost goes from **O(n pages)** to **O(log n)** index pages + **O(1)** heap pages.

---

## 3. Database Types: Row-based vs Column-based

### Row-based Database

> Examples: PostgreSQL, MySQL, SQL Server, Oracle

**How data is stored on disk**:
```
Page 1: [id=1, name="Alice", age=30, salary=5000]
        [id=2, name="Bob",   age=25, salary=6000]
        [id=3, name="Carol", age=35, salary=7000]
```

Each page contains **full rows**. One I/O gives you several complete rows.

**How queries work**:
1. Database scans pages sequentially.
2. For each row on a page, it checks if the row qualifies (matches the `WHERE` clause).
3. If it qualifies, it returns **all columns** (or filters to the requested ones in memory).

**Cost note**: `SELECT *` and `SELECT name` have the **same I/O cost** — the whole page is fetched either way.

---

### Column-based Database

> Examples: Apache Cassandra (wide-column), Amazon Redshift, ClickHouse, Apache Parquet

**How data is stored on disk**:
```
Column "name":   [Alice, Bob, Carol, Dave, ...]   ← stored together
Column "age":    [30, 25, 35, 22, ...]            ← stored together
Column "salary": [5000, 6000, 7000, 4500, ...]    ← stored together
```

Each page holds **one column's values** across many rows.

**How queries work**:
1. For `SELECT name WHERE age > 28`:
   - Read only the `age` column pages to find matching `row_id`s.
   - Read only the `name` column pages to fetch those `row_id`s.
   - Never touch `salary` at all.

---

### Pros & Cons Comparison

| Feature                        | Row-based DB                          | Column-based DB                        |
|-------------------------------|---------------------------------------|----------------------------------------|
| **Best for**                  | OLTP (many small reads/writes)        | OLAP (analytics, aggregations)         |
| **Read full row**             | 1 I/O gets whole row ✅               | Requires reading from each column ❌   |
| **Read few columns**          | Wastes I/O loading unused columns ❌  | Loads only needed columns ✅           |
| **Writes/updates**            | Fast — append single row ✅           | Slower — must update each column file ❌|
| **Compression**               | Poor (mixed types per page)           | Excellent (same type per page) ✅      |
| **Aggregations** (SUM, AVG)   | Slow — must scan all rows             | Very fast — reads one column ✅        |
| **Typical use case**          | Web apps, transactions, user records  | Data warehouses, reporting, BI tools   |

> **Plain English**:
> - Row DB = a filing cabinet where each drawer has one person's full file.
> - Column DB = a spreadsheet where each column is stored separately — great for summing a column, bad for reading one person's row.

---

## 4. Transactions

### What Is a Transaction?

A **transaction** is a **unit of work** that groups one or more SQL queries together, treating them as a single atomic operation.

- If all queries succeed → **commit** (persist changes).
- If any query fails → **rollback** (undo everything).

> **Key rule**: You are **always inside a transaction** when running SQL. If you don't start one explicitly, the database starts one implicitly and commits it immediately after each statement.

---

### When to Use a Transaction

| Scenario                                | Use a Transaction? |
|-----------------------------------------|--------------------|
| Modifying data (INSERT, UPDATE, DELETE) | **Yes — always**   |
| Read-only queries (SELECT)              | Optional — useful for consistent snapshots |
| Complex multi-step operations           | **Yes — required** |

**Example — Bank transfer (must be atomic)**:
```sql
BEGIN;

UPDATE accounts SET balance = balance - 500 WHERE id = 1;  -- debit
UPDATE accounts SET balance = balance + 500 WHERE id = 2;  -- credit

COMMIT;
```

If the second `UPDATE` fails, the `ROLLBACK` undoes the first one. Money is never lost or duplicated.

---

### Transaction Lifespan

```
BEGIN
  │
  ├── query 1
  ├── query 2
  ├── query 3
  │
  ├── COMMIT   ← changes are persisted to disk
  │    OR
  └── ROLLBACK ← all changes are discarded
```

| Command      | What it does |
|--------------|--------------|
| `BEGIN`      | Opens a transaction. Tells the DB: *"Start tracking my work."* |
| `COMMIT`     | Closes the transaction successfully. Persists all changes to disk. |
| `ROLLBACK`   | Aborts the transaction. Discards all changes since `BEGIN`. |

---

### Implicit vs Explicit Transactions

**Implicit (auto-commit)**:
```sql
-- No BEGIN — database wraps each statement in its own transaction
INSERT INTO users (name) VALUES ('Alice');
-- ↑ automatically: BEGIN → INSERT → COMMIT
```

**Explicit**:
```sql
BEGIN;
INSERT INTO orders (user_id, total) VALUES (1, 99.99);
INSERT INTO order_items (order_id, product_id) VALUES (LASTVAL(), 5);
COMMIT;
```

> Use explicit transactions whenever **two or more operations must succeed or fail together**.

---

## What to Learn Next

1. **ACID Properties** — Atomicity, Consistency, Isolation, Durability: the guarantees transactions provide.
2. **Transaction Isolation Levels** — READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE.
3. **Concurrency problems** — Dirty reads, non-repeatable reads, phantom reads.
4. **Index types** — Hash indexes, partial indexes, composite indexes, covering indexes.
5. **Query execution plans** — How to use `EXPLAIN ANALYZE` to understand what your database actually does.
6. **Database locking** — Row locks, table locks, deadlocks, and how to avoid them.
7. **WAL (Write-Ahead Log)** — How databases guarantee durability without writing data to disk on every write.
