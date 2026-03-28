# Database Partitioning

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> How to split large tables into smaller physical partitions for better query performance, easier data retention, and operational flexibility in PostgreSQL.

---

## Table of Contents

1. [What Is Partitioning?](#1-what-is-partitioning)
2. [Why Partition?](#2-why-partition)
3. [How Partitioning Works](#3-how-partitioning-works)
4. [Step-by-Step Setup](#4-step-by-step-setup)
5. [Partition Pruning](#5-partition-pruning)
6. [Load Then Attach Pattern](#6-load-then-attach-pattern)
7. [Dropping Partitions](#7-dropping-partitions)
8. [PARTITION OF vs ATTACH PARTITION](#8-partition-of-vs-attach-partition)

---

## 1. What Is Partitioning?

**Partitioning** = splitting **1 logical big table** into many **physical smaller tables** (partitions).

- The goal is to keep each partition small enough that the "hot" part of data plus its indexes **fits in memory**, making operations cheaper.
- Applications query the **parent table** — PostgreSQL routes reads/writes to the correct partition transparently.

---

## 2. Why Partition?

### Faster Queries

- Biggest win when most frequently accessed rows live in **one or a small number of partitions**.
- Partitioning acts like "upper levels of an index tree" — the planner can **skip irrelevant partitions** entirely.

### Sequential Scan Can Beat Index Scan

- If a query touches a **large percentage of one partition**, a sequential scan on that small partition can be faster than an index scan with random I/O.

### Fast Bulk Delete / Retention

- Remove old data by **dropping a partition** (`DROP TABLE partition`) or **detaching** it (`ALTER TABLE ... DETACH PARTITION`).
- This is **much faster than `DELETE`** and avoids the heavy `VACUUM` work created by bulk deletes.

### Hot vs Cold Storage

- Move old/seldom-used partitions to **cheaper/slower storage** using tablespaces.

---

## 3. How Partitioning Works

### Partition Types (Declarative)

PostgreSQL supports three built-in partition types:

| Type | Syntax | How It Works |
|------|--------|-------------|
| **RANGE** | `FOR VALUES FROM (x) TO (y)` | Partitions cover **non-overlapping value ranges**. Boundaries are **lower-inclusive, upper-exclusive**. Example: `[1,10)` and `[10,20)` — value 10 goes to the second partition. |
| **LIST** | `FOR VALUES IN (x, y, z)` | Partitions cover **explicit lists of values**. |
| **HASH** | `FOR VALUES WITH (MODULUS m, REMAINDER r)` | Rows placed where `hash(key) % modulus = remainder`. |

> Need other types? Use legacy methods like inheritance + `UNION ALL` views, but you lose some benefits of declarative partitioning.

### Key Concepts

- A partitioned table must be **created as partitioned from the start** (or you must recreate/migrate into a new partitioned parent).
- The **partitioned table is "virtual"** — it holds **no storage**.
- **Partitions (child tables) are normal tables** that hold the actual data.
- Inserting into the parent → row is **routed** to the correct partition via the partition key.
- **Updating the partition key** can cause the row to **move between partitions**.

### Sub-Partitioning

- A partition can **itself be partitioned** (multi-level partitioning).
- Each partition can have its **own indexes, constraints, and defaults** (but must have the same columns as the parent).

### Attach / Detach

- You can't "convert" a normal table into a partitioned table directly, but you can:
  - **ATTACH** an existing table as a partition.
  - **DETACH** a partition (turning it into a standalone table).

---

## 4. Step-by-Step Setup

### Step 1: Create the Parent (Partitioned) Table

The parent defines the schema and partition strategy. It holds **no data** itself.

**RANGE example:**

```sql
CREATE TABLE events (
  id          bigserial,
  created_at  timestamptz NOT NULL,
  user_id     bigint NOT NULL,
  payload     jsonb
) PARTITION BY RANGE (created_at);
```

**LIST example:**

```sql
CREATE TABLE tenant_data (
  tenant_id  int NOT NULL,
  id         bigserial,
  created_at timestamptz NOT NULL,
  data       text
) PARTITION BY LIST (tenant_id);
```

**HASH example:**

```sql
CREATE TABLE user_activity (
  user_id    bigint NOT NULL,
  ts         timestamptz NOT NULL,
  action     text
) PARTITION BY HASH (user_id);
```

### Step 2: Create Partitions (Child Tables)

> **Recommendation**: Always add a `DEFAULT` partition — it catches unexpected keys and prevents insert failures.

**RANGE:**

```sql
CREATE TABLE events_2026_01
  PARTITION OF events
  FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE events_default
  PARTITION OF events
  DEFAULT;
```

**LIST:**

```sql
CREATE TABLE tenant_data_t1
  PARTITION OF tenant_data
  FOR VALUES IN (1);

CREATE TABLE tenant_data_default
  PARTITION OF tenant_data
  DEFAULT;
```

**HASH:**

```sql
CREATE TABLE user_activity_p0
  PARTITION OF user_activity
  FOR VALUES WITH (MODULUS 8, REMAINDER 0);

-- Create p1 through p7 for the remaining remainders...
```

### Step 3: Constraints & Keys

#### Primary Key

The primary key **must include the partition key** (common gotcha):

```sql
ALTER TABLE events
  ADD CONSTRAINT events_pk PRIMARY KEY (id, created_at);
```

- PostgreSQL needs uniqueness enforceable **per partition**.
- If you need a globally unique `id` alone, consider UUIDs or accepting that the constraint won't be database-enforced across partitions.

#### Check Constraints

Partition bounds imply constraints automatically. Additional `CHECK` constraints are allowed per partition.

#### Indexes

- Creating an index on the **parent** automatically creates the same index on **all child partitions**.
- Indexes are **per partition** — this is usually beneficial (smaller indexes), but queries touching many partitions will use many separate indexes.

```sql
CREATE INDEX ON events (created_at);
CREATE INDEX ON events (user_id, created_at);
```

### Step 4: Insert Data

Inserts into the parent are **automatically routed** to the correct partition:

```sql
INSERT INTO events (created_at, user_id, payload)
VALUES (now(), 123, '{"type":"click"}');
```

If no partition matches:
- **Has DEFAULT partition** → routed to default.
- **No DEFAULT partition** → **insert fails** with an error.

---

## 5. Partition Pruning

Partition pruning allows the planner to **skip partitions** that cannot contain matching rows.

```sql
-- Enable (this is the default)
SET enable_partition_pruning = on;

-- The planner only scans partitions whose bounds overlap the WHERE clause
EXPLAIN ANALYZE
SELECT *
FROM events
WHERE created_at >= '2026-02-01' AND created_at < '2026-03-01';
```

| Setting | Behavior |
|---------|----------|
| `enable_partition_pruning = on` (default) | Planner removes partitions that **cannot match** the `WHERE` clause from the execution plan. |
| `enable_partition_pruning = off` | Planner scans **all partitions** regardless of the filter. |

> **Important**: Pruning is driven by **partition bounds**, not by indexes. You get pruning even without any indexes.

---

## 6. Load Then Attach Pattern

An alternative approach when you already have a regular table (possibly loaded with data) and want to make it a partition.

### Steps

```sql
-- 1. Create staging table matching parent schema
CREATE TABLE events_2026_04_staging
  (LIKE events INCLUDING DEFAULTS INCLUDING CONSTRAINTS);

-- 2. Add CHECK constraint for the partition range (important!)
ALTER TABLE events_2026_04_staging
  ADD CONSTRAINT events_2026_04_chk
  CHECK (created_at >= '2026-04-01' AND created_at < '2026-05-01');

-- 3. Load data into staging
INSERT INTO events_2026_04_staging (created_at, user_id, payload)
VALUES ('2026-04-10', 42, '{"type":"login"}');

-- 4. Attach as a partition (validates data against partition bounds)
ALTER TABLE events
  ATTACH PARTITION events_2026_04_staging
  FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');

-- 5. Drop the now-redundant CHECK constraint
ALTER TABLE events_2026_04_staging
  DROP CONSTRAINT events_2026_04_chk;
```

> **Why the CHECK constraint?** When you `ATTACH`, PostgreSQL must verify all existing rows match the partition bounds. If the CHECK constraint already guarantees this, PostgreSQL **skips the full table scan**, making the attach operation instant.

---

## 7. Dropping Partitions

### Option A: Detach Then Drop (Recommended)

```sql
-- Detach — the partition becomes a standalone table
ALTER TABLE events DETACH PARTITION events_2025_01;

-- Optional: backup/export first
-- pg_dump -t events_2025_01 ...
-- COPY events_2025_01 TO ...

-- Then drop
DROP TABLE events_2025_01;
```

### Option B: Drop Directly (Faster, More Locking Impact)

```sql
DROP TABLE events_2025_01;
```

**Benefits of dropping a partition**:
- Deletes millions of rows **instantly** without row-by-row `DELETE`.
- Avoids `VACUUM` bloat entirely.
- Much faster than `DELETE FROM events WHERE created_at < '2025-02-01'`.

---

## 8. PARTITION OF vs ATTACH PARTITION

| Approach | Use When |
|----------|----------|
| `PARTITION OF` | Creating **new, empty** future partitions. Minimal fuss. |
| `ATTACH PARTITION` | You already have a table (possibly with data) you want to convert. You want to load data separately first, then bring it under the parent. Useful when migrating from non-partitioned to partitioned design. |

---

## What to Learn Next

1. **Sub-partitioning** — Partitioning a partition (e.g., partition by tenant, then by date).
2. **Automating partition creation** — Using `pg_partman` extension for automatic partition management.
3. **Partition-wise joins and aggregates** — How PostgreSQL optimizes joins across partitioned tables.
4. **Sharding** — Distributing partitions across multiple database servers (Citus, logical replication).
5. **Monitoring partition sizes** — Keeping track of partition growth and planning retention policies.
