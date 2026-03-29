# PostgreSQL Sequences

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> Everything about PostgreSQL sequence objects — what they are, how they work internally, how to create/alter/drop them, the manipulation functions (`nextval`, `currval`, `setval`, `lastval`), their relationship to `SERIAL`/`IDENTITY`, and the traps you must avoid.

---

## Table of Contents

1. [What Is a Sequence?](#1-what-is-a-sequence)
2. [Why Sequences Exist](#2-why-sequences-exist)
3. [When to Use Sequences](#3-when-to-use-sequences)
4. [How Sequences Work — The Flow](#4-how-sequences-work--the-flow)
   - [Step 1 — Create the Sequence Object](#step-1--create-the-sequence-object)
   - [Step 2 — Generate Values with `nextval`](#step-2--generate-values-with-nextval)
   - [Step 3 — Read the Current Value](#step-3--read-the-current-value)
   - [Step 4 — Reset or Jump with `setval`](#step-4--reset-or-jump-with-setval)
   - [Step 5 — Drop the Sequence](#step-5--drop-the-sequence)
5. [CREATE SEQUENCE — Full Syntax](#5-create-sequence--full-syntax)
6. [Sequence Manipulation Functions](#6-sequence-manipulation-functions)
7. [ALTER SEQUENCE](#7-alter-sequence)
8. [Sequences and SERIAL / IDENTITY Columns](#8-sequences-and-serial--identity-columns)
9. [Sequences Across Tables](#9-sequences-across-tables)
10. [Listing and Inspecting Sequences](#10-listing-and-inspecting-sequences)
11. [Common Mistakes & Misunderstandings](#11-common-mistakes--misunderstandings)
12. [Best Practices](#12-best-practices)
13. [Quick Revision Points](#13-quick-revision-points)

---

## 1. What Is a Sequence?

A **sequence** is a special single-row table created with `CREATE SEQUENCE`. Its only job is to produce a series of unique integers — typically used for primary keys.

### Analogy

Think of a sequence as a **ticket dispenser** at a deli counter:

- Each customer pulls the next number — no two customers get the same number.
- Numbers only go forward (you can't "un-pull" a ticket).
- Even if a customer leaves without ordering, that number is **gone** — it is not recycled.

> **Plain English**: A sequence is a database-managed counter that guarantees unique, ordered integers across concurrent sessions.

---

## 2. Why Sequences Exist

| Problem | How Sequences Solve It |
|---------|----------------------|
| Need unique IDs for new rows | Sequences generate globally unique integers, even across concurrent `INSERT`s |
| Manual `MAX(id) + 1` is unsafe under concurrency | Two sessions could compute the same max — sequences are **atomic** |
| Need IDs shared across multiple tables | One sequence can serve many tables |
| Need control over starting value, increment, min/max | Sequences are fully configurable |

> Without sequences, generating unique IDs safely requires application-level locking or UUIDs. Sequences are PostgreSQL's purpose-built, high-performance solution.

---

## 3. When to Use Sequences

| Scenario | Use a Sequence? |
|----------|----------------|
| Auto-incrementing primary key | **Yes** — via `SERIAL`, `IDENTITY`, or explicit sequence |
| Unique ID shared across multiple tables | **Yes** — one sequence, multiple tables calling `nextval` |
| Custom numbering (e.g., invoice numbers starting at 10000) | **Yes** — configure `START`, `INCREMENT` |
| Gapless numbering (every number must be used) | **No** — sequences **cannot guarantee gapless** numbering |
| UUIDs or random identifiers | **No** — use `gen_random_uuid()` instead |
| Non-integer identifiers | **No** — sequences produce integers only |

---

## 4. How Sequences Work — The Flow

The lifecycle of a sequence has five steps:

```
┌──────────────────────────────────────────────────────────────┐
│                   Sequence Lifecycle                          │
│                                                              │
│  Step 1         Step 2         Step 3       Step 4   Step 5  │
│  ──────         ──────         ──────       ──────   ──────  │
│  CREATE         nextval()      currval()    setval() DROP    │
│  SEQUENCE       generates      reads last   resets   SEQUENCE│
│  (define it)    next value     value        position (remove)│
└──────────────────────────────────────────────────────────────┘
```

---

### Step 1 — Create the Sequence Object

You define the sequence with `CREATE SEQUENCE`. This creates a special single-row table in the catalog that holds the sequence state.

```sql
CREATE SEQUENCE order_id_seq
    START 1
    INCREMENT 1
    MINVALUE 1
    NO MAXVALUE
    CACHE 1;
```

**Why this step exists**: The sequence object must exist before any session can request values from it. The parameters define the rules: where to start, how to increment, and what the boundaries are.

---

### Step 2 — Generate Values with `nextval`

When you call `nextval('order_id_seq')`, PostgreSQL:

1. **Advances** the sequence to its next value.
2. **Returns** that value to the caller.
3. This is done **atomically** — even if 100 sessions call `nextval` at the same instant, each gets a distinct value.

```sql
SELECT nextval('order_id_seq');  -- returns 1
SELECT nextval('order_id_seq');  -- returns 2
SELECT nextval('order_id_seq');  -- returns 3
```

**Why this step exists**: This is the core purpose of a sequence — producing unique integers safely under concurrency.

**Critical behavior**: The value obtained by `nextval` is **never reclaimed**, even if the calling transaction rolls back. This is by design — reclaiming would require blocking concurrent transactions, defeating the purpose of sequences.

```
Session A                          Session B
─────────                          ─────────
BEGIN;
INSERT INTO orders ...
  nextval → 1
                                   BEGIN;
                                   INSERT INTO orders ...
                                     nextval → 2
                                   COMMIT;  ✅
ROLLBACK;  ❌
-- Value 1 is GONE — not reused
-- Next nextval → 3
```

#### Concurrency Guarantees

If two sessions call `nextval()` on the same sequence at the same time, each call is **atomic** — each session gets a **different value**. There is no collision and no "both got the same number" outcome.

```sql
-- Sequence is at 100. Two concurrent sessions:
-- Session A: SELECT nextval('my_seq');  → 101
-- Session B: SELECT nextval('my_seq');  → 102
-- Even if Session A rolls back, 101 is gone. Next call → 103.
```

Three important consequences:

| Consequence | Explanation |
|-------------|-------------|
| **Increments are not rolled back** | If a transaction gets 101 and later fails, 101 is still consumed. This avoids blocking concurrent transactions. |
| **Gaps are normal** | Missing numbers appear from rollbacks, failed inserts, or unused cached values. This applies to `SERIAL` / `BIGSERIAL` too, since they are backed by sequences. |
| **Commit order ≠ sequence order** | A transaction that got 102 might commit before the one that got 101. Sequence values are **unique but not a reliable representation of commit time** — because `nextval()` allocates independently of transaction outcome. |

> **Bottom line**: PostgreSQL sequences are designed for concurrent use — they guarantee **uniqueness**, not gaplessness or transactional rollback of allocated numbers.

#### Example — Full Concurrent Flow

```
Time ──►

Session A                              Session B
──────────────────────                 ──────────────────────

BEGIN;
INSERT INTO orders (id, name)
VALUES (nextval('order_seq'),          BEGIN;
        'Order Alpha');
-- nextval → 101                       INSERT INTO orders (id, name)
                                       VALUES (nextval('order_seq'),
                                               'Order Beta');
                                       -- nextval → 102

                                       COMMIT;  ✅
                                       -- Row (102, 'Order Beta') is saved

ROLLBACK;  ❌
-- Row (101, 'Order Alpha') is gone
-- BUT 101 is NOT returned to the sequence

                                       BEGIN;
                                       INSERT INTO orders (id, name)
                                       VALUES (nextval('order_seq'),
                                               'Order Gamma');
                                       -- nextval → 103  (not 101!)
                                       COMMIT;  ✅
```

**Result in table `orders`**:

| id  | name        |
|-----|-------------|
| 102 | Order Beta  |
| 103 | Order Gamma |

- **101 is missing** — this is a gap caused by Session A's rollback.
- **102 committed before 101's transaction ended** — commit order ≠ sequence order.
- **103 follows 102**, skipping 101 — the sequence never goes backward.

---

### Step 3 — Read the Current Value

Two functions let you peek at the current state **without advancing** the sequence:

| Function | What It Returns |
|----------|----------------|
| `currval('order_id_seq')` | The last value returned by `nextval` **in this session** |
| `lastval()` | The last value returned by `nextval` for **any sequence** in this session |

```sql
SELECT nextval('order_id_seq');   -- returns 4
SELECT currval('order_id_seq');   -- returns 4 (does NOT advance)
SELECT lastval();                 -- returns 4 (most recent nextval in this session)
```

**Why this step exists**: After inserting a row, you often need to know the ID that was just generated — for example, to insert child rows in a related table.

**Important**: `currval` and `lastval` are **session-local**. They return what *your* session last got from `nextval`. They are **not affected** by other sessions calling `nextval`.

```
Session A                          Session B
─────────                          ─────────
nextval('seq') → 10
                                   nextval('seq') → 11
currval('seq') → 10               currval('seq') → 11
-- Each session sees its own last value
```

> **Error trap**: Calling `currval` or `lastval` **before** calling `nextval` in the current session will raise an error — there is no "current value" yet.

---

### Step 4 — Reset or Jump with `setval`

`setval` lets you manually reposition a sequence. This is an administrative operation.

```sql
-- Two-parameter form: set last_value = 1000, is_called = true
-- Next nextval will return 1001
SELECT setval('order_id_seq', 1000);

-- Three-parameter form: set last_value = 1000, is_called = false
-- Next nextval will return exactly 1000
SELECT setval('order_id_seq', 1000, false);
```

| Call | `is_called` | Next `nextval` returns |
|------|-------------|----------------------|
| `setval('seq', 42)` | `true` | 43 |
| `setval('seq', 42, true)` | `true` | 43 |
| `setval('seq', 42, false)` | `false` | 42 |

**Why this step exists**: After bulk data imports (e.g., `pg_dump` / `COPY`), the sequence position may be stale. You need `setval` to resync it with the actual maximum ID in the table.

**Critical behavior**: `setval` changes are **immediately visible** to other transactions and are **not undone** if the calling transaction rolls back. This is the same non-transactional behavior as `nextval`.

---

### Step 5 — Drop the Sequence

```sql
-- Drop explicitly:
DROP SEQUENCE order_id_seq;

-- Drop only if it exists:
DROP SEQUENCE IF EXISTS order_id_seq;

-- Drop and cascade to dependent objects:
DROP SEQUENCE order_id_seq CASCADE;
```

If a sequence is **owned by** a table column (via `OWNED BY` or auto-created by `SERIAL`/`IDENTITY`), dropping the table or column will **automatically drop** the sequence.

**Why this step exists**: Cleanup. Orphaned sequences waste catalog space and cause confusion.

---

## 5. CREATE SEQUENCE — Full Syntax

```sql
CREATE SEQUENCE [ IF NOT EXISTS ] sequence_name
    [ AS { SMALLINT | INT | BIGINT } ]
    [ INCREMENT [ BY ] increment ]
    [ MINVALUE minvalue | NO MINVALUE ]
    [ MAXVALUE maxvalue | NO MAXVALUE ]
    [ START [ WITH ] start ]
    [ CACHE cache ]
    [ [ NO ] CYCLE ]
    [ OWNED BY { table_name.column_name | NONE } ];
```

### Parameter Reference

| Parameter | Description | Default |
|-----------|------------|---------|
| `AS data_type` | Data type of the sequence (`SMALLINT`, `INT`, `BIGINT`). Determines min/max boundaries. | `BIGINT` |
| `INCREMENT BY` | Value added to the current position on each `nextval` call. Positive = ascending, negative = descending. | `1` |
| `MINVALUE` / `NO MINVALUE` | Floor of the sequence. | `1` (ascending) or data type min (descending) |
| `MAXVALUE` / `NO MAXVALUE` | Ceiling of the sequence. | Data type max (ascending) or `-1` (descending) |
| `START WITH` | First value returned by the first `nextval` call. | `MINVALUE` (ascending) or `MAXVALUE` (descending) |
| `CACHE` | How many values to preallocate in memory per session for faster access. | `1` (no cache) |
| `CYCLE` / `NO CYCLE` | Whether to wrap around when max/min is reached. `CYCLE` restarts; `NO CYCLE` raises an error. | `NO CYCLE` |
| `OWNED BY` | Associates the sequence with a table column. When that column or table is dropped, the sequence is dropped too. | `NONE` |

### Data Type Ranges

| Type | Min | Max |
|------|-----|-----|
| `SMALLINT` | -32,768 | 32,767 |
| `INT` | -2,147,483,648 | 2,147,483,647 |
| `BIGINT` | -9,223,372,036,854,775,808 | 9,223,372,036,854,775,807 |

### Examples

#### Ascending sequence starting at 100, incrementing by 5

```sql
CREATE SEQUENCE mysequence
    INCREMENT 5
    START 100;

SELECT nextval('mysequence');  -- 100
SELECT nextval('mysequence');  -- 105
SELECT nextval('mysequence');  -- 110
```

#### Descending sequence that cycles (3 → 2 → 1 → 3 → 2 → 1 ...)

```sql
CREATE SEQUENCE countdown
    INCREMENT -1
    MINVALUE 1
    MAXVALUE 3
    START 3
    CYCLE;

SELECT nextval('countdown');  -- 3
SELECT nextval('countdown');  -- 2
SELECT nextval('countdown');  -- 1
SELECT nextval('countdown');  -- 3 (cycled!)
```

#### Sequence owned by a table column

```sql
CREATE TABLE order_details (
    order_id SERIAL,
    item_id  INT NOT NULL,
    item_text VARCHAR NOT NULL,
    price    DECIMAL(10,2) NOT NULL,
    PRIMARY KEY (order_id, item_id)
);

CREATE SEQUENCE order_item_id
    START 10
    INCREMENT 10
    MINVALUE 10
    OWNED BY order_details.item_id;

-- Use the sequence in INSERT:
INSERT INTO order_details (order_id, item_id, item_text, price)
VALUES
    (100, nextval('order_item_id'), 'DVD Player', 100.00),   -- item_id = 10
    (100, nextval('order_item_id'), 'Android TV', 550.00),   -- item_id = 20
    (100, nextval('order_item_id'), 'Speaker',    250.00);   -- item_id = 30
```

---

## 6. Sequence Manipulation Functions

All four functions operate on sequence objects. They require `USAGE` or appropriate privilege on the sequence.

| Function | Returns | Side Effect | Session-Local? |
|----------|---------|-------------|----------------|
| `nextval(regclass)` | `bigint` — the next value | **Advances** the sequence | No — globally visible |
| `currval(regclass)` | `bigint` — last value from `nextval` in this session | **None** | Yes — session-local |
| `lastval()` | `bigint` — last value from `nextval` for any sequence in this session | **None** | Yes — session-local |
| `setval(regclass, bigint [, boolean])` | `bigint` — the set value | **Repositions** the sequence | No — globally visible |

### `nextval` — The Core Function

```sql
SELECT nextval('order_id_seq');
-- or inside an INSERT:
INSERT INTO orders (id, name) VALUES (nextval('order_id_seq'), 'New Order');
```

**Key behaviors**:
- **Atomic**: concurrent sessions each get a distinct value.
- **Non-transactional**: the value is consumed even if the transaction rolls back.
- **Gaps are normal**: rollbacks, `ON CONFLICT` clauses, and crashes all create gaps.

---

### `currval` — Read Last Generated Value

```sql
-- Must call nextval first in this session!
SELECT nextval('order_id_seq');   -- 42
SELECT currval('order_id_seq');   -- 42
```

**Use case**: After `INSERT`, fetch the ID that was just generated — without `RETURNING`.

```sql
-- Pattern: insert then fetch the ID
INSERT INTO orders (id, name) VALUES (nextval('order_id_seq'), 'Widget');
SELECT currval('order_id_seq');  -- the ID used in the INSERT above
```

> **Better alternative**: Use `INSERT ... RETURNING id` instead — it's a single statement and avoids the `currval` pitfall.

---

### `lastval` — Read Last Value from Any Sequence

```sql
SELECT nextval('seq_a');  -- 10
SELECT nextval('seq_b');  -- 55
SELECT lastval();         -- 55 (most recent nextval, regardless of which sequence)
```

**Warning**: `lastval` is fragile. If a trigger or function calls `nextval` on a different sequence, `lastval` changes silently. Prefer `currval` with an explicit sequence name.

---

### `setval` — Reposition the Sequence

```sql
-- After bulk import, resync the sequence:
SELECT setval('order_id_seq', (SELECT MAX(id) FROM orders));
-- Next nextval will return MAX(id) + 1
```

**Three-parameter form**:

```sql
-- Next nextval returns exactly 500 (not 501):
SELECT setval('order_id_seq', 500, false);
```

---

## 7. ALTER SEQUENCE

You can change any parameter of an existing sequence:

```sql
ALTER SEQUENCE sequence_name
    [ AS { SMALLINT | INT | BIGINT } ]
    [ INCREMENT [ BY ] increment ]
    [ MINVALUE minvalue | NO MINVALUE ]
    [ MAXVALUE maxvalue | NO MAXVALUE ]
    [ START [ WITH ] start ]
    [ RESTART [ [ WITH ] restart ] ]
    [ CACHE cache ]
    [ [ NO ] CYCLE ]
    [ OWNED BY { table_name.column_name | NONE } ];
```

### Common Operations

```sql
-- Change the increment:
ALTER SEQUENCE order_id_seq INCREMENT BY 10;

-- Restart the sequence at a specific value:
ALTER SEQUENCE order_id_seq RESTART WITH 1000;

-- Change ownership to a different column:
ALTER SEQUENCE order_id_seq OWNED BY new_table.new_column;

-- Remove ownership association:
ALTER SEQUENCE order_id_seq OWNED BY NONE;

-- Rename:
ALTER SEQUENCE order_id_seq RENAME TO new_order_id_seq;

-- Move to a different schema:
ALTER SEQUENCE order_id_seq SET SCHEMA new_schema;
```

---

## 8. Sequences and SERIAL / IDENTITY Columns

### SERIAL (Legacy Approach)

`SERIAL` is a shorthand that auto-creates a sequence behind the scenes:

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

-- PostgreSQL internally does:
-- 1. CREATE SEQUENCE users_id_seq AS INTEGER;
-- 2. Sets column default: DEFAULT nextval('users_id_seq')
-- 3. ALTER SEQUENCE users_id_seq OWNED BY users.id
```

| Shorthand | Actual Type | Sequence Type |
|-----------|------------|---------------|
| `SERIAL` | `INT` | `INT` sequence |
| `BIGSERIAL` | `BIGINT` | `BIGINT` sequence |
| `SMALLSERIAL` | `SMALLINT` | `SMALLINT` sequence |

**Drawback**: `SERIAL` does **not** prevent manual inserts that bypass the sequence. Someone can `INSERT INTO users (id, name) VALUES (999, 'Eve')` — this does not advance the sequence, causing a future collision.

---

### IDENTITY (Modern Approach — Recommended)

`GENERATED ... AS IDENTITY` is the SQL-standard way, available since PostgreSQL 10:

```sql
CREATE TABLE users (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name TEXT NOT NULL
);
```

| Option | Behavior |
|--------|----------|
| `GENERATED ALWAYS` | PostgreSQL **always** generates the value. Manual inserts are **rejected** unless you use `OVERRIDING SYSTEM VALUE`. |
| `GENERATED BY DEFAULT` | PostgreSQL generates the value if you don't provide one. Manual inserts are **allowed** (like `SERIAL`). |

**Why prefer IDENTITY over SERIAL**:

| Feature | `SERIAL` | `IDENTITY` |
|---------|----------|-----------|
| SQL standard | No (PostgreSQL-specific) | Yes |
| Prevents manual ID bypass | No | Yes (`GENERATED ALWAYS`) |
| Visible in `\d` as identity | No (shows as default) | Yes |
| Sequence auto-dropped with column | Yes (via `OWNED BY`) | Yes (built-in) |
| Works with `pg_dump` / `pg_restore` | Quirky (separate sequence) | Clean (integrated) |

---

## 9. Sequences Across Tables

A single sequence can serve **multiple tables**. This is useful when you need a globally unique ID across related tables.

```sql
-- One shared sequence:
CREATE SEQUENCE global_id_seq START 1;

-- Used by multiple tables:
CREATE TABLE customers (
    id BIGINT DEFAULT nextval('global_id_seq') PRIMARY KEY,
    name TEXT NOT NULL
);

CREATE TABLE suppliers (
    id BIGINT DEFAULT nextval('global_id_seq') PRIMARY KEY,
    name TEXT NOT NULL
);

-- IDs are unique across BOTH tables:
INSERT INTO customers (name) VALUES ('Alice');  -- id = 1
INSERT INTO suppliers (name) VALUES ('Acme');   -- id = 2
INSERT INTO customers (name) VALUES ('Bob');    -- id = 3
```

> **Caution**: When using `OWNED BY`, a sequence can only be owned by **one** column. If you drop that column/table, the shared sequence disappears for all tables.

---

## 10. Listing and Inspecting Sequences

### List All Sequences in the Current Database

```sql
-- Via pg_class:
SELECT relname AS sequence_name
FROM pg_class
WHERE relkind = 'S';

-- Via information_schema:
SELECT sequence_name, data_type, start_value, increment, minimum_value, maximum_value
FROM information_schema.sequences
WHERE sequence_schema = 'public';
```

### Inspect a Specific Sequence

```sql
-- Treat the sequence as a table and SELECT from it:
SELECT * FROM order_id_seq;
```

This returns:

| Column | Meaning |
|--------|---------|
| `last_value` | The last value returned by `nextval` (or set by `setval`) |
| `log_cnt` | Internal — how many `nextval` calls remain before a WAL write |
| `is_called` | `true` if `nextval` has been called at least once; `false` means next `nextval` returns `last_value` |

### psql Shortcuts

```
\ds              -- List all sequences
\ds+ seq_name    -- Show details of a specific sequence
\d table_name    -- Shows the default column values (reveals sequence usage)
```

---

## 11. Common Mistakes & Misunderstandings

| Misunderstanding | Reality |
|-----------------|---------|
| "Sequences produce gapless numbers" | **No.** Rollbacks, crashes, `ON CONFLICT`, and `CACHE > 1` all create gaps. PostgreSQL explicitly warns: sequences **cannot** be used for gapless numbering. |
| "If my transaction rolls back, the sequence value is returned" | **No.** `nextval` is non-transactional. The consumed value is gone forever. |
| "SERIAL prevents duplicate IDs" | **No.** `SERIAL` sets a default, but nothing stops a manual `INSERT` with an explicit ID that collides with future sequence values. Use `GENERATED ALWAYS AS IDENTITY` instead. |
| "`currval` shows the global current value" | **No.** `currval` is **session-local** — it shows what *your* session last got from `nextval`. |
| "`setval` is rolled back if my transaction fails" | **No.** `setval` changes are **immediate and permanent**, even if the calling transaction aborts. |
| "CACHE makes sequences transactional" | **No.** `CACHE` preallocates values in memory for performance. If a session crashes, all cached-but-unused values are **lost** (creating gaps). |
| "Dropping a table always drops its sequences" | Only if the sequence is **owned by** a column in that table (true for `SERIAL` and `IDENTITY`, but not for manually created sequences with `OWNED BY NONE`). |
| "I can use `lastval` safely" | Be careful — triggers and functions may call `nextval` on other sequences, silently changing what `lastval` returns. Prefer `currval` or `RETURNING`. |

---

## 12. Best Practices

### Design

- **Prefer `GENERATED ALWAYS AS IDENTITY`** over `SERIAL` for new tables — it's SQL-standard and prevents accidental manual ID insertion.
- **Use `BIGINT` / `BIGSERIAL`** for high-volume tables — `INT` maxes out at ~2.1 billion, which large systems can exhaust.
- **One sequence per table** unless you genuinely need cross-table uniqueness.
- **Don't rely on sequences for gapless numbering** — use a separate counter table with `FOR UPDATE` locking if you truly need gapless numbers.

### Operations

- **Resync sequences after bulk imports** — `COPY` and `pg_restore` data may not advance the sequence:

```sql
-- After importing data into the orders table:
SELECT setval('orders_id_seq', (SELECT COALESCE(MAX(id), 0) FROM orders));
```

- **Monitor sequence exhaustion** on `INT` sequences in high-volume tables:

```sql
SELECT
    sequencename,
    last_value,
    CASE data_type
        WHEN 'smallint' THEN 32767
        WHEN 'integer'  THEN 2147483647
        WHEN 'bigint'   THEN 9223372036854775807
    END AS max_value,
    ROUND(
        last_value::numeric /
        CASE data_type
            WHEN 'smallint' THEN 32767
            WHEN 'integer'  THEN 2147483647
            WHEN 'bigint'   THEN 9223372036854775807
        END * 100, 2
    ) AS pct_used
FROM pg_sequences
WHERE schemaname = 'public';
```

- **Use `CACHE` wisely** — higher cache values improve `nextval` throughput but create larger gaps on session crashes. For most workloads, `CACHE 1` (default) is fine.

### Application Code (Go Example)

```go
// Preferred: use RETURNING to get the generated ID
var orderID int64
err := db.QueryRowContext(ctx,
    `INSERT INTO orders (name, total) VALUES ($1, $2) RETURNING id`,
    "New Order", 99.99,
).Scan(&orderID)
if err != nil {
    return fmt.Errorf("insert order: %w", err)
}
// orderID now holds the generated sequence value
```

```go
// Alternative: explicit nextval (rarely needed)
var id int64
err := db.QueryRowContext(ctx,
    `SELECT nextval('order_id_seq')`,
).Scan(&id)
if err != nil {
    return fmt.Errorf("nextval: %w", err)
}

_, err = db.ExecContext(ctx,
    `INSERT INTO orders (id, name, total) VALUES ($1, $2, $3)`,
    id, "New Order", 99.99,
)
```

---

## 13. Quick Revision Points

- **Sequence** = a database object that generates unique integers. Internally, it's a special single-row table.
- **`nextval`** advances the sequence and returns the next value. It is **atomic** (safe under concurrency) and **non-transactional** (not rolled back on abort).
- **`currval`** returns the last value from `nextval` in the **current session** — it does not advance the sequence.
- **`lastval`** is like `currval` but for whichever sequence was last used in the session — fragile if triggers call `nextval` on other sequences.
- **`setval`** repositions the sequence manually — changes are **immediate and permanent** regardless of transaction outcome.
- **Gaps are normal and expected** — rollbacks, crashes, cached values, and `ON CONFLICT` all cause gaps. Sequences are **not** gapless counters.
- **`SERIAL`** = shorthand that auto-creates a sequence + sets it as the column default. Legacy approach.
- **`GENERATED ALWAYS AS IDENTITY`** = SQL-standard replacement for `SERIAL`. Prevents manual ID bypass. Use this for new tables.
- **`OWNED BY`** ties a sequence's lifetime to a column — dropping the column drops the sequence.
- **`CACHE`** preallocates values in memory for speed, but unused cached values are lost on session/crash.
- **After bulk imports**, always resync with `setval('seq', (SELECT MAX(id) FROM table))`.
- **Prefer `INSERT ... RETURNING id`** over `currval` to get the generated ID in application code.

### One-Sentence Summary

> A PostgreSQL sequence is a high-performance, concurrency-safe integer generator whose values are consumed permanently on each `nextval` call — even if the transaction rolls back — and must be resynced manually after bulk data operations.

---

## Related Topics

- [Transactions](02-transactions.md) — Sequences are non-transactional; understanding this requires knowing how transactions work.
- [ACID Properties](01-acid.md) — `nextval` and `setval` are exceptions to normal transactional atomicity.
- [Indexes](05-indexes.md) — Primary keys generated by sequences are typically indexed.
- [MVCC](10-mvcc.md) — Sequence operations bypass normal MVCC visibility rules for performance.

---

## Resources

- [PostgreSQL Official Docs — CREATE SEQUENCE](https://www.postgresql.org/docs/current/sql-createsequence.html)
- [PostgreSQL Official Docs — Sequence Manipulation Functions](https://www.postgresql.org/docs/current/functions-sequence.html)
- [PostgreSQL Official Docs — ALTER SEQUENCE](https://www.postgresql.org/docs/current/sql-altersequence.html)
- [Neon — PostgreSQL Sequences Tutorial](https://neon.com/postgresql/postgresql-tutorial/postgresql-sequences)
