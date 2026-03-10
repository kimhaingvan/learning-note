# PostgreSQL Locking Modes

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> A comprehensive guide to PostgreSQL's table-level and row-level locking mechanisms — what they are, when they're acquired, and how they interact.

---

## Table of Contents

1. [What Are Locks?](#1-what-are-locks)
2. [Why Locking Exists](#2-why-locking-exists)
3. [Locking Levels Overview](#3-locking-levels-overview)
4. [Table-Level Locks (Detailed)](#4-table-level-locks-detailed)
5. [Row-Level Locks (Detailed)](#5-row-level-locks-detailed)
6. [Page-Level and Advisory Locks](#6-page-level-and-advisory-locks)

---

## 1. What Are Locks?

When two lock modes **conflict**, they cannot be held on the same data at the same time by different transactions.

If one transaction has already acquired a conflicting lock, the later transaction **must wait** until the first transaction releases its lock (usually at `COMMIT` or `ROLLBACK`).

### Analogy

Imagine a table is a **room**:

- Some people just **look around** → weak locks
- Some people **rearrange furniture** → medium locks
- Some people **renovate the entire room** → strong locks

> The **stronger your action**, the **stronger your lock** → stronger locks **block more people**.

### Example

```sql
-- SESSION 1
BEGIN;                               -- (1)
SELECT * FROM users;                 -- (2) acquires ACCESS SHARE lock
-- ... still in transaction ...
COMMIT;                              -- (4) releases lock

-- SESSION 2 (while Session 1 is active)
TRUNCATE users;                      -- (3) needs ACCESS EXCLUSIVE lock
-- ⏳ BLOCKED — must wait for Session 1 to COMMIT
```

`ACCESS SHARE` conflicts with `ACCESS EXCLUSIVE`, so Session 2 waits.

---

## 2. Why Locking Exists

- **Data consistency** — prevent concurrent transactions from corrupting data.
- **Conflict prevention** — ensure the same data isn't updated by different transactions at the same time.
- **Controlled concurrent access** — allow maximum parallelism while maintaining correctness.

---

## 3. Locking Levels Overview

PostgreSQL has four categories of locks:

| Level | Scope | Acquired By |
|-------|-------|-------------|
| **Table-Level** | Entire table | SQL commands or explicit `LOCK TABLE` |
| **Row-Level** | Individual rows | `SELECT FOR ...`, `UPDATE`, `DELETE` |
| **Page-Level** | Data pages | Internal (rarely visible to users) |
| **Advisory** | Application-defined | Explicit `pg_advisory_lock()` calls |

All locks are released at the **end of the transaction** (or during savepoint rollback).

---

## 4. Table-Level Locks (Detailed)

### Which Commands Acquire Which Locks

| Command | Lock Acquired |
|---------|--------------|
| `SELECT` | ACCESS SHARE |
| `SELECT FOR UPDATE/SHARE` | ROW SHARE |
| `INSERT`, `UPDATE`, `DELETE` | ROW EXCLUSIVE |
| `CREATE INDEX` | SHARE |
| `CREATE TRIGGER` | SHARE ROW EXCLUSIVE |
| `DROP TABLE`, `TRUNCATE` | ACCESS EXCLUSIVE |

### Conflict Matrix

Below is the full conflict matrix. **X** = these two lock types **conflict** (cannot be held simultaneously by different transactions).

| Requested ↓ \ Held → | AS | RS | RE | SUX | S | SRE | E | AE |
|----------------------|----|----|----|----|---|-----|---|----|
| **Access Share (AS)** | | | | | | | | X |
| **Row Share (RS)** | | | | | | | X | X |
| **Row Exclusive (RE)** | | | | | X | X | X | X |
| **Share Update Excl (SUX)** | | | | X | X | X | X | X |
| **Share (S)** | | | X | X | | X | X | X |
| **Share Row Excl (SRE)** | | | X | X | X | X | X | X |
| **Exclusive (E)** | | X | X | X | X | X | X | X |
| **Access Exclusive (AE)** | X | X | X | X | X | X | X | X |

---

### Access Share

- **Purpose**: "I only want to **read**."
- **Acquired by**: `SELECT`
- **Conflicts with**: Access Exclusive only
- **Does not conflict with**: Everything else

```sql
-- Session A (HOLD)
BEGIN;
SELECT * FROM demo_lock;

-- Session B (CONFLICT)
BEGIN;
DROP TABLE demo_lock;
-- ⏳ Blocked

-- Session C (NOT CONFLICT)
BEGIN;
UPDATE demo_lock SET name = 'C' WHERE id = 1;
-- ✅ Not blocked
```

---

### Row Share

- **Purpose**: "I'm reading and **may update specific rows**."
- **Acquired by**: `SELECT ... FOR UPDATE`, `SELECT ... FOR SHARE`
- **Conflicts with**: Exclusive, Access Exclusive
- **Does not conflict with**: Everything else

```sql
-- Session A (HOLD)
BEGIN;
SELECT * FROM demo_lock FOR UPDATE;

-- Session B (CONFLICT)
BEGIN;
LOCK TABLE demo_lock IN ACCESS EXCLUSIVE MODE;
-- ⏳ Blocked

-- Session C (NOT CONFLICT)
BEGIN;
INSERT INTO demo_lock VALUES (3, 'C');
-- ✅ Not blocked
```

---

### Row Exclusive

- **Purpose**: "I'm **changing some rows**."
- **Acquired by**: `INSERT`, `UPDATE`, `DELETE`
- **Conflicts with**: Share, Share Row Exclusive, Exclusive, Access Exclusive
- **Does not conflict with**: Access Share, Row Share, Row Exclusive, Share Update Exclusive

```sql
-- Session A (HOLD)
BEGIN;
UPDATE demo_lock SET name = 'A' WHERE id = 1;

-- Session B (CONFLICT)
BEGIN;
CREATE INDEX idx1 ON demo_lock(name);
-- ⏳ Blocked (CREATE INDEX needs SHARE lock)

-- Session C (NOT CONFLICT)
BEGIN;
INSERT INTO demo_lock VALUES (4, 'D');
-- ✅ Not blocked (also Row Exclusive — compatible)
```

---

### Share Update Exclusive

- **Purpose**: "I'm doing **maintenance**, don't do other maintenance."
- **Acquired by**: `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY`
- **Conflicts with**: Share Update Exclusive, Share, Share Row Exclusive, Exclusive, Access Exclusive
- **Does not conflict with**: Access Share, Row Share, Row Exclusive

```sql
-- Session A (HOLD)
BEGIN;
VACUUM demo_lock;

-- Session B (CONFLICT)
BEGIN;
LOCK TABLE demo_lock IN SHARE MODE;
-- ⏳ Blocked

-- Session C (NOT CONFLICT)
BEGIN;
UPDATE demo_lock SET name = 'X' WHERE id = 2;
-- ✅ Not blocked
```

---

### Share

- **Purpose**: "**Read-only** — no one writes."
- **Acquired by**: `CREATE INDEX` (non-concurrent)
- **Conflicts with**: Row Exclusive, Share Update Exclusive, Share Row Exclusive, Exclusive, Access Exclusive
- **Does not conflict with**: Access Share, Row Share, Share

```sql
-- Session A (HOLD)
BEGIN;
LOCK TABLE demo_lock IN SHARE MODE;

-- Session B (CONFLICT)
BEGIN;
UPDATE demo_lock SET name = 'Z' WHERE id = 1;
-- ⏳ Blocked

-- Session C (NOT CONFLICT)
BEGIN;
SELECT * FROM demo_lock;
-- ✅ Not blocked
```

---

### Share Row Exclusive

- **Purpose**: "I'm doing **sensitive structural work**."
- **Acquired by**: `CREATE TRIGGER`
- **Conflicts with**: Row Exclusive, Share Update Exclusive, Share, Share Row Exclusive, Exclusive, Access Exclusive
- **Does not conflict with**: Access Share, Row Share

```sql
-- Session A (HOLD)
BEGIN;
CREATE TRIGGER ... ;

-- Session B (CONFLICT)
BEGIN;
INSERT INTO demo_lock VALUES (5, 'E');
-- ⏳ Blocked

-- Session C (NOT CONFLICT)
BEGIN;
SELECT * FROM demo_lock;
-- ✅ Not blocked
```

---

### Exclusive

- **Purpose**: "**Only I write**."
- **Acquired by**: `ALTER TABLE` (some forms)
- **Conflicts with**: Everything except Access Share
- **Does not conflict with**: Access Share only

```sql
-- Session A (HOLD)
BEGIN;
LOCK TABLE demo_lock IN EXCLUSIVE MODE;

-- Session B (CONFLICT)
BEGIN;
INSERT INTO demo_lock VALUES (6, 'F');
-- ⏳ Blocked

-- Session C (NOT CONFLICT)
BEGIN;
SELECT * FROM demo_lock;
-- ✅ Not blocked
```

---

### Access Exclusive

- **Purpose**: "I **own the table**. Everyone else out."
- **Acquired by**: `DROP TABLE`, `TRUNCATE`, `VACUUM FULL`, some `ALTER TABLE` forms
- **Conflicts with**: **ALL lock types**
- **Does not conflict with**: Nothing

```sql
-- Session A (HOLD)
BEGIN;
DROP TABLE demo_lock;

-- Session B (CONFLICT)
BEGIN;
SELECT * FROM demo_lock;
-- ⏳ Blocked — even reads are blocked!
```

---

## 5. Row-Level Locks (Detailed)

Row-level locks are acquired on **individual rows**, not the whole table.

### Strength Order (Weakest → Strongest)

```
FOR KEY SHARE → FOR SHARE → FOR NO KEY UPDATE → FOR UPDATE
```

### Conflict Matrix

| Lock Mode | Blocks UPDATE | Blocks DELETE | Blocks KEY CHANGE | Allows Other SHARE Locks |
|-----------|:------------:|:------------:|:-----------------:|:-----------------------:|
| **FOR KEY SHARE** | ❌ | ✅ | ✅ | ✅ |
| **FOR SHARE** | ✅ | ✅ | ✅ | ✅ |
| **FOR NO KEY UPDATE** | ✅ | ✅ | ❌ | ❌ |
| **FOR UPDATE** | ✅ | ✅ | ✅ | ❌ |

---

### FOR KEY SHARE

```sql
SELECT ... FOR KEY SHARE;
-- "Don't change WHO this row IS."
```

- Protects **key columns only**. Non-key columns can still be updated.
- Other transactions **CANNOT**: `DELETE` the row, change key columns.
- Other transactions **CAN**: Update non-key columns, read normally.

---

### FOR SHARE

```sql
SELECT ... FOR SHARE;
-- "Don't change this row at all."
```

- Other transactions **CANNOT**: `DELETE` the row, `UPDATE` the row.
- Other transactions **CAN**: Also acquire `FOR SHARE`, read normally.

---

### FOR NO KEY UPDATE

```sql
-- Acquired automatically by:
UPDATE table SET non_key_column = value;
```

- Protects **non-key columns**. Only key columns can be updated by others.
- Other transactions **CANNOT**: `DELETE`, `UPDATE`, `FOR SHARE`, `FOR UPDATE`.
- Other transactions **CAN**: `FOR KEY SHARE`, read normally.

---

### FOR UPDATE

```sql
SELECT ... FOR UPDATE;
DELETE ...;
```

- **Strongest row lock**. Full exclusive access to the row.
- Other transactions **CANNOT**: Any row lock, `UPDATE`, `DELETE`.
- Other transactions **CAN**: Plain `SELECT` only.

---

### FOR NO KEY UPDATE vs FOR UPDATE — Automatic Selection

PostgreSQL automatically chooses the appropriate row lock based on what you're updating:

| Operation | Lock Acquired |
|-----------|--------------|
| `UPDATE users SET balance = 100 WHERE id = 1;` (non-key column) | **FOR NO KEY UPDATE** |
| `UPDATE users SET id = 2 WHERE id = 1;` (key column) | **FOR UPDATE** |

---

## 6. Page-Level and Advisory Locks

### Page-Level Locks

- Used **internally** by PostgreSQL for page-level operations.
- Rarely visible or relevant to application developers.

### Advisory Locks

- **Application-defined** locks — PostgreSQL provides the mechanism, you decide the semantics.
- Useful for preventing duplicate cron jobs, coordinating distributed workers, etc.

```sql
-- Acquire an advisory lock (blocks if already held)
SELECT pg_advisory_lock(12345);

-- Try without blocking (returns true/false)
SELECT pg_try_advisory_lock(12345);

-- Release
SELECT pg_advisory_unlock(12345);
```

---

## What to Learn Next

1. **Deadlocks** — What happens when two transactions wait for each other's locks, and how PostgreSQL detects and resolves them.
2. **Lock monitoring** — Using `pg_locks`, `pg_stat_activity`, and `pg_blocking_pids()`.
3. **MVCC** — How PostgreSQL avoids many locks by using row versioning.
4. **Statement timeouts and lock timeouts** — `SET lock_timeout = '5s'` to prevent indefinite waiting.
5. **CREATE INDEX CONCURRENTLY** — How to build indexes without blocking writes.
