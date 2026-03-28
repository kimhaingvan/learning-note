# PostgreSQL Locking Modes

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 5.5 — Locking Mechanisms / Phase 5.6 — Deadlocks  
> A comprehensive guide to PostgreSQL's table-level and row-level locking mechanisms — what they are, when they're acquired, how they interact, lock monitoring, deadlocks, and timeout configuration.

---

## Table of Contents

1. [What Are Locks?](#1-what-are-locks)
2. [Why Locking Exists](#2-why-locking-exists)
3. [Locking Levels Overview](#3-locking-levels-overview)
4. [Table-Level Locks (Detailed)](#4-table-level-locks-detailed)
5. [Row-Level Locks (Detailed)](#5-row-level-locks-detailed)
6. [Page-Level and Advisory Locks](#6-page-level-and-advisory-locks)
7. [Deadlocks](#7-deadlocks)
8. [Lock Monitoring](#8-lock-monitoring)
9. [Lock and Statement Timeouts](#9-lock-and-statement-timeouts)

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

## 7. Deadlocks

### What Is a Deadlock?

A **deadlock** occurs when two or more transactions **wait for each other's locks**, creating a circular dependency that none can resolve on their own.

```
Timeline:
  Tx A                              Tx B
  ─────                             ─────
  BEGIN                              BEGIN
  UPDATE accounts SET bal=100
    WHERE id = 1
  -- holds lock on row 1
                                     UPDATE accounts SET bal=200
                                       WHERE id = 2
                                     -- holds lock on row 2
  UPDATE accounts SET bal=300
    WHERE id = 2
  -- ⏳ BLOCKED waiting for Tx B
                                     UPDATE accounts SET bal=400
                                       WHERE id = 1
                                     -- ⏳ BLOCKED waiting for Tx A

  ══════════════ DEADLOCK! ══════════════
  Neither transaction can proceed.
```

### How PostgreSQL Detects Deadlocks

1. When a transaction has been **waiting** for a lock longer than `deadlock_timeout` (default: **1 second**), PostgreSQL runs a **deadlock detection algorithm**.
2. It builds a **wait-for graph** of all waiting transactions.
3. If a **cycle** is found → deadlock detected.
4. PostgreSQL **aborts one transaction** (the one it detects last) and lets the other proceed.
5. The aborted transaction receives:

```
ERROR:  deadlock detected
DETAIL: Process 12345 waits for ShareLock on transaction 67890;
        blocked by process 12346.
        Process 12346 waits for ShareLock on transaction 67889;
        blocked by process 12345.
HINT:   See server log for query details.
```

### deadlock_timeout

```sql
-- Check current setting
SHOW deadlock_timeout;  -- default: '1s'

-- Adjust (only affects when detection runs, NOT how long to wait for locks)
SET deadlock_timeout = '2s';

-- In postgresql.conf for permanent change:
-- deadlock_timeout = 1s
```

| Misconception | Reality |
|--------------|--------|
| "deadlock_timeout is how long to wait before giving up" | It's how long to wait before **checking** for deadlocks. If there's no deadlock, the transaction continues waiting. |
| "Lower timeout = faster deadlock resolution" | True, but the detection algorithm is expensive. Running it too frequently wastes CPU. 1s is a good default. |

### Prevention Strategies

| Strategy | How | Example |
|----------|-----|--------|
| **Lock in consistent order** | Always acquire locks on rows/tables in the same order across all code paths | Always lock accounts by ascending `id` |
| **Keep transactions short** | Less time holding locks = less chance of deadlock | Don't do HTTP calls inside transactions |
| **Use `SELECT FOR UPDATE` with `NOWAIT` or `SKIP LOCKED`** | Fail fast or skip locked rows instead of waiting | Queue processing patterns |
| **Reduce lock scope** | Lock only what you need | Use row-level locks instead of table-level |

```sql
-- Strategy: Consistent ordering
-- Always lock accounts in ascending ID order
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Now both rows are locked in a predictable order
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
```

```sql
-- NOWAIT: fail immediately if lock is not available
SELECT * FROM tasks WHERE id = 1 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row

-- SKIP LOCKED: skip rows that are locked by other transactions
-- Perfect for job queues
SELECT * FROM job_queue
WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

---

## 8. Lock Monitoring

### pg_locks — All Current Locks

`pg_locks` is a system view that shows **every lock currently held or awaited** in the database.

```sql
SELECT
    l.pid,
    l.locktype,
    l.mode,
    l.granted,                    -- true = holding; false = waiting
    l.relation::regclass AS table,
    a.state,
    a.query
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation IS NOT NULL
  AND a.datname = current_database()
ORDER BY l.granted, l.pid;
```

| Column | Meaning |
|--------|---------|
| `locktype` | Type of lockable object: `relation`, `tuple`, `transactionid`, `advisory`, etc. |
| `mode` | Lock mode: `AccessShareLock`, `RowExclusiveLock`, `ExclusiveLock`, etc. |
| `granted` | `true` = lock is held; `false` = waiting to acquire |
| `relation` | The table OID (cast to `::regclass` for name) |
| `pid` | Backend process ID |

### Finding Blocked Queries

```sql
-- Which queries are blocked, and who is blocking them?
SELECT
    blocked.pid        AS blocked_pid,
    blocked.query      AS blocked_query,
    blocked.state      AS blocked_state,
    blocker.pid        AS blocker_pid,
    blocker.query      AS blocker_query,
    blocker.state      AS blocker_state,
    now() - blocked.query_start AS wait_duration
FROM pg_stat_activity blocked
JOIN LATERAL (
    SELECT unnest(pg_blocking_pids(blocked.pid)) AS pid
) bp ON true
JOIN pg_stat_activity blocker ON blocker.pid = bp.pid
WHERE blocked.state = 'active'
  AND cardinality(pg_blocking_pids(blocked.pid)) > 0;
```

### pg_blocking_pids()

Returns an array of PIDs that are blocking a given backend:

```sql
-- Which processes are blocking PID 12345?
SELECT pg_blocking_pids(12345);
-- → {12346, 12347}
```

### Lock Chains (Cascading Blocks)

```sql
-- Recursive CTE to find full lock chains
WITH RECURSIVE lock_chain AS (
    SELECT
        pid,
        pg_blocking_pids(pid) AS blockers,
        query,
        0 AS depth
    FROM pg_stat_activity
    WHERE cardinality(pg_blocking_pids(pid)) > 0

    UNION ALL

    SELECT
        a.pid,
        pg_blocking_pids(a.pid),
        a.query,
        lc.depth + 1
    FROM pg_stat_activity a
    JOIN lock_chain lc ON a.pid = ANY(lc.blockers)
    WHERE lc.depth < 5  -- prevent infinite recursion
)
SELECT DISTINCT pid, blockers, query, depth
FROM lock_chain
ORDER BY depth;
```

### Terminating Blocking Sessions

```sql
-- Politely cancel the query (sends SIGINT)
SELECT pg_cancel_backend(12345);

-- Forcefully terminate the session (sends SIGTERM)
SELECT pg_terminate_backend(12345);
```

| Function | Effect | When to Use |
|----------|--------|------------|
| `pg_cancel_backend(pid)` | Cancels the current query; session stays open | First attempt — less disruptive |
| `pg_terminate_backend(pid)` | Kills the entire session | When cancel doesn't work or session is idle-in-transaction |

---

## 9. Lock and Statement Timeouts

### lock_timeout

Maximum time to **wait for a lock** before giving up with an error.

```sql
-- Set per-session
SET lock_timeout = '5s';

-- Set per-transaction
BEGIN;
SET LOCAL lock_timeout = '3s';
ALTER TABLE users ADD COLUMN age integer;
-- If the table lock can't be acquired within 3s:
-- ERROR: canceling statement due to lock timeout
COMMIT;
```

### statement_timeout

Maximum time a **single statement** can run before being cancelled.

```sql
-- Set per-session
SET statement_timeout = '30s';

-- Long query will be killed after 30 seconds
SELECT * FROM huge_table WHERE complex_condition;
-- ERROR: canceling statement due to statement timeout
```

### idle_in_transaction_session_timeout

Maximum time a session can sit **idle inside an open transaction** before being terminated.

```sql
-- Set in postgresql.conf for all sessions
-- idle_in_transaction_session_timeout = '10min'

-- Set per-session
SET idle_in_transaction_session_timeout = '5min';
```

### Comparison

| Timeout | What It Limits | Default | Recommended |
|---------|---------------|---------|-------------|
| `lock_timeout` | Time waiting to acquire a lock | 0 (disabled) | 5–30s for DDL migrations |
| `statement_timeout` | Total execution time of one statement | 0 (disabled) | 30s–5min for web apps |
| `idle_in_transaction_session_timeout` | Idle time inside an open transaction | 0 (disabled) | 5–10min |
| `deadlock_timeout` | Time before checking for deadlocks | 1s | 1s (rarely change) |

### Best Practice: Safe DDL Migrations

```sql
-- Don't let a migration block the entire application
BEGIN;
SET LOCAL lock_timeout = '5s';
SET LOCAL statement_timeout = '30s';

ALTER TABLE orders ADD COLUMN tracking_number text;
-- If the lock can't be acquired in 5s, the migration fails gracefully
-- instead of blocking all queries on the table indefinitely

COMMIT;
```

---

## What to Learn Next

1. **MVCC** — How PostgreSQL avoids many locks by using row versioning — see [10-mvcc.md](10-mvcc.md).
2. **Transactions** — Transaction lifecycle, savepoints, and two-phase commit — see [02-transactions.md](02-transactions.md).
3. **ACID Properties** — Isolation levels that determine locking behavior — see [01-acid.md](01-acid.md).
4. **`CREATE INDEX CONCURRENTLY`** — How to build indexes without blocking writes.
5. **Connection Pooling** — PgBouncer/Pgpool-II to manage connection limits and lock contention.

---

> *Ref: [Docs — Explicit Locking](https://www.postgresql.org/docs/18/explicit-locking.html) · [Docs — Deadlocks](https://www.postgresql.org/docs/18/explicit-locking.html#LOCKING-DEADLOCKS) · [Docs — pg_locks](https://www.postgresql.org/docs/18/view-pg-locks.html) · [Docs — Lock Management](https://www.postgresql.org/docs/18/runtime-config-locks.html)*
