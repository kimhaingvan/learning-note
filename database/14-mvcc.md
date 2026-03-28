# Multi-Version Concurrency Control (MVCC)

> **Level**: Intermediate–Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 5.4 — MVCC  
> How PostgreSQL allows multiple transactions to read and write the same data concurrently without blocking each other — by keeping multiple versions of every row.

---

## Table of Contents

1. [Definition / Idea](#1-definition--idea)
2. [When MVCC Matters](#2-when-mvcc-matters)
3. [Main Approach — How MVCC Works](#3-main-approach--how-mvcc-works)
   - [Phase A — Transaction Starts & Snapshot](#phase-a--transaction-starts--snapshot)
   - [Phase B — Reads Check Visibility](#phase-b--reads-check-visibility)
   - [Phase C — Writes Create New Versions](#phase-c--writes-create-new-versions)
   - [Phase D — Cleanup (VACUUM)](#phase-d--cleanup-vacuum)
4. [Life of a Row — End-to-End Example](#4-life-of-a-row--end-to-end-example)
5. [Key Internal Concepts](#5-key-internal-concepts)
6. [MVCC and Isolation Levels](#6-mvcc-and-isolation-levels)
7. [MVCC Does NOT Mean "No Locking"](#7-mvcc-does-not-mean-no-locking)
8. [Common Mistakes & Misunderstandings](#8-common-mistakes--misunderstandings)
9. [Best Practices](#9-best-practices)
10. [Quick Revision Points](#10-quick-revision-points)

---

## 1. Definition / Idea

### Definition

**MVCC** stands for **Multi-Version Concurrency Control**. It is PostgreSQL's concurrency model for handling multiple sessions reading and writing the same data at the same time while preserving consistency.

### Idea — How to Think About It

> When data changes, PostgreSQL creates a **new row version** instead of overwriting the old one, and each transaction reads the versions its **snapshot** is allowed to see.

Think of it as a **library** where every edit creates a new page rather than erasing the old one. Different readers can look at different editions of the same page depending on when they walked in.

The core idea:

- A row can have **multiple physical versions** at the same time.
- Each transaction gets a **snapshot** — a "camera frame" that defines what is visible.
- PostgreSQL decides which row versions are visible to that snapshot.
- Old versions remain alive so other transactions can still see the correct past state.
- **VACUUM** later cleans up versions that are no longer visible to anyone.

### Purpose — Why We Use It

The PostgreSQL docs emphasize the main benefit:

> **Reads do not block writes, and writes do not block reads** under normal MVCC behavior.

Without MVCC, a pure lock-based system would force readers to wait for writers (and vice versa), severely limiting throughput on mixed workloads.

---

## 2. When MVCC Matters

| Scenario | Why MVCC Helps |
|----------|---------------|
| Many users reading and writing the same table concurrently | Readers never block writers; writers never block readers |
| Long-running analytical queries alongside OLTP writes | The analytical query sees a consistent snapshot — writes proceed in parallel |
| Different isolation levels (Read Committed, Repeatable Read, Serializable) | MVCC + snapshots are the mechanism that implements these levels |
| Understanding `VACUUM`, bloat, and performance tuning | MVCC is the *reason* dead tuples exist and why vacuum is essential |

> **Plain English**: If your database has more than one user, MVCC is the engine that makes concurrency safe and fast. Understanding it is foundational to PostgreSQL performance tuning.

---

## 3. Main Approach — How MVCC Works

The MVCC lifecycle has four phases:

```
┌─────────────────────────────────────────────────────────────┐
│                    MVCC Lifecycle                            │
│                                                             │
│  Phase A          Phase B          Phase C       Phase D    │
│  ─────────        ─────────        ─────────     ───────    │
│  Transaction      Reads check      Writes        VACUUM     │
│  starts →         row visibility   create new    removes    │
│  gets snapshot    via snapshot     row versions   dead rows  │
└─────────────────────────────────────────────────────────────┘
```

---

### Phase A — Transaction Starts & Snapshot

When a transaction begins, PostgreSQL assigns it a **snapshot** — a set of rules that define what committed data is visible to that transaction.

```
Transaction starts
      │
      ▼
PostgreSQL assigns a snapshot
      │
      ▼
Snapshot defines: which transactions are
  • committed (visible)
  • in-progress (invisible)
  • not yet started (invisible)
```

**Why this step exists**: Without a stable snapshot, a query could see an inconsistent mix of old and new states mid-execution.

---

### Phase B — Reads Check Visibility

When PostgreSQL reads a row, it does **not** simply ask "does the row exist?" — it asks:

1. Who **created** this version? Was that transaction committed?
2. Was this version **deleted or superseded**?
3. Does my **snapshot** consider that visible?

```
For each row version on disk:
      │
      ├── Is the creator transaction committed?
      │     └── No  → invisible
      │     └── Yes ↓
      ├── Was the row deleted/superseded?
      │     └── No  → VISIBLE ✅
      │     └── Yes → is the deleter committed AND visible to my snapshot?
      │               └── Yes → invisible (row is dead to me)
      │               └── No  → VISIBLE ✅ (deletion not yet visible)
```

> **Key insight**: Two transactions can read the **same table** at the **same time** but see **different row versions** depending on their snapshots. That is the "multi-version" part.

---

### Phase C — Writes Create New Versions

MVCC avoids making readers fight writers by creating **new versions** instead of modifying in place.

| Operation | What Happens |
|-----------|-------------|
| `INSERT` | Creates a **new row version**. Other transactions see it only when visibility rules allow. |
| `UPDATE` | Conceptually: creates a **new version** and marks the **old version as obsolete** for newer snapshots. This is why updates leave old row versions behind. |
| `DELETE` | Marks the row version as **dead for future snapshots**, but older snapshots may still need it. The row is **not instantly erased**. |

```sql
-- Conceptual UPDATE internals
UPDATE accounts SET balance = 150 WHERE id = 1;

-- Internally PostgreSQL does something like:
-- 1. Find current visible version (V1: balance = 100)
-- 2. Create new version          (V2: balance = 150)
-- 3. Mark V1 as obsolete for newer snapshots
-- 4. V1 still visible to older active snapshots
```

---

### Phase D — Cleanup (VACUUM)

MVCC naturally accumulates **dead tuples** — old row versions that no active transaction needs anymore.

PostgreSQL's `VACUUM` process:

- Removes dead row versions in tables and indexes.
- Updates the **visibility map** (used to optimize index-only scans).
- Protects against **transaction ID wraparound** — a critical maintenance concern.

```
MVCC creates new versions  ──►  Old versions become dead  ──►  VACUUM cleans them up
         (concurrency)              (cleanup cost)               (long-term health)
```

> **Plain English**: MVCC gives you concurrency benefits **now**; VACUUM pays the cleanup cost **later**.

---

## 4. Life of a Row — End-to-End Example

Consider a table with one row: `id = 1, balance = 100`.

```
Step 1: Initial committed row
─────────────────────────────
  V1: balance = 100  (committed) ✅

Step 2: Transaction B updates it
─────────────────────────────────
  UPDATE accounts SET balance = 150 WHERE id = 1;

  PostgreSQL creates:
    V1: balance = 100  (old version, still exists)
    V2: balance = 150  (new version)

Step 3: Visibility depends on snapshot timing
──────────────────────────────────────────────
  Tx A (started before B committed) → sees V1 (balance = 100)
  Tx C (started after  B committed) → sees V2 (balance = 150)

Step 4: Old version becomes dead
─────────────────────────────────
  Once NO active transaction needs V1 → V1 is removable dead data

Step 5: VACUUM reclaims space
──────────────────────────────
  VACUUM removes dead tuple V1
  Space is available for reuse
```

### Timeline Diagram

```
Timeline:
  Tx A                    Tx B                      Tx C
  ─────                   ─────                     ─────
  BEGIN
  (snapshot taken)
                          BEGIN
                          UPDATE balance → 150
                          COMMIT
  SELECT balance
  → reads 100 (V1) ✅                              BEGIN
                                                    (snapshot taken)
                                                    SELECT balance
                                                    → reads 150 (V2) ✅
  COMMIT
       ↓
  V1 is now dead (no active snapshot needs it)
       ↓
  VACUUM removes V1
```

---

## 5. Key Internal Concepts

### Snapshot

The transaction's **visibility window** — what committed work it is allowed to see. Depending on the isolation level, a snapshot is taken:

| Isolation Level | Snapshot Timing |
|----------------|----------------|
| **Read Committed** | At the start of **each statement** |
| **Repeatable Read** | At the start of **the transaction** |
| **Serializable** | At the start of **the transaction** (+ SSI conflict detection) |

---

### Tuple Version (Row Version)

A row version stored in the table **heap**. One logical row may have **multiple physical versions** over time because updates produce new versions rather than replacing the old one for everybody.

---

### xmin / xmax

PostgreSQL tracks row versions using hidden system columns:

| Column | Meaning |
|--------|---------|
| `xmin` | The **transaction ID** that **created** (inserted) this row version |
| `xmax` | The **transaction ID** that **deleted or obsoleted** this row version (0 if still live) |

```sql
-- You can inspect them directly:
SELECT xmin, xmax, * FROM accounts WHERE id = 1;
```

Visibility rules use `xmin` and `xmax` to decide whether a row version is visible to a given snapshot.

---

### Dead Tuple

An old row version that is **no longer visible** to any active transaction. It occupies space but serves no purpose. VACUUM removes dead tuples.

---

### Visibility Map

A per-table bitmap that tracks which pages contain **only visible tuples** (i.e., no cleanup needed). Used by:

- **VACUUM**: to skip pages that don't need scanning.
- **Index-only scans**: to avoid fetching the heap page when all columns are in the index.

---

### Autovacuum

The background maintenance daemon that automatically runs `VACUUM` and `ANALYZE` to keep MVCC storage healthy and prevent transaction ID wraparound.

```sql
-- Check autovacuum activity:
SELECT relname, last_autovacuum, last_autoanalyze, n_dead_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

---

## 6. MVCC and Isolation Levels

MVCC is the **engine** that implements isolation levels. The isolation level determines **when snapshots are taken** and **what anomalies are possible**.

### Read Committed (Default in PostgreSQL)

- Each **statement** gets a fresh snapshot of committed data.
- Two statements in the same transaction **can see different committed data** if another transaction commits in between.
- Can observe **non-repeatable reads** across statements.

```
Timeline:
  Tx A                              Tx B
  ─────                             ─────
  BEGIN
  SELECT balance FROM accounts
    WHERE id = 1
  → reads 500
                                    BEGIN
                                    UPDATE balance = 300 WHERE id = 1
                                    COMMIT
  SELECT balance FROM accounts
    WHERE id = 1
  → reads 300 (new snapshot per statement!)
```

---

### Repeatable Read

- The transaction keeps a **stable snapshot** for its entire lifetime.
- Repeated reads see a **consistent past view** — even if other transactions commit new data.
- PostgreSQL's implementation is stronger than the SQL standard minimum — it also prevents phantom reads.

```
Timeline:
  Tx A                              Tx B
  ─────                             ─────
  BEGIN (snapshot taken)
  SELECT balance FROM accounts
    WHERE id = 1
  → reads 500
                                    BEGIN
                                    UPDATE balance = 300 WHERE id = 1
                                    COMMIT
  SELECT balance FROM accounts
    WHERE id = 1
  → still reads 500 (same snapshot!)
```

---

### Serializable

- Uses **Serializable Snapshot Isolation (SSI)** to emulate safe serial execution.
- PostgreSQL detects dangerous read/write patterns and **aborts transactions** when necessary.
- Strongest correctness — but may require **retry logic** in application code.

```go
// Go pseudocode — retry pattern for Serializable
for retries := 0; retries < maxRetries; retries++ {
    tx, err := db.BeginTx(ctx, &sql.TxOptions{
        Isolation: sql.LevelSerializable,
    })
    if err != nil {
        return err
    }

    err = doWork(tx)
    if err != nil {
        tx.Rollback()
        if isSerializationFailure(err) {
            continue // retry
        }
        return err
    }

    err = tx.Commit()
    if err == nil {
        return nil
    }
    if isSerializationFailure(err) {
        continue // retry
    }
    return err
}
```

---

## 7. MVCC Does NOT Mean "No Locking"

This is a critical distinction. MVCC **reduces** read/write blocking but does **not eliminate** locking entirely.

| What MVCC Handles | What Locks Still Handle |
|-------------------|------------------------|
| Read/write visibility (no blocking) | Writer/writer conflicts on the same row |
| Consistent snapshots for readers | DDL operations (`ALTER TABLE`, `DROP`, `TRUNCATE`) |
| Non-blocking reads | Serializable conflict detection (SSI) |
| | Explicit `SELECT FOR UPDATE` / `SELECT FOR SHARE` |

> **Correct mental model**: MVCC handles **visibility**. Locks handle **conflicts** that visibility alone cannot solve.

```sql
-- Two writers updating the same row STILL conflict:
-- Session 1
BEGIN;
UPDATE accounts SET balance = 200 WHERE id = 1;
-- holds row-level lock

-- Session 2
BEGIN;
UPDATE accounts SET balance = 300 WHERE id = 1;
-- ⏳ BLOCKED — waiting for Session 1 to COMMIT or ROLLBACK
```

---

## 8. Common Mistakes & Misunderstandings

| Misunderstanding | Reality |
|-----------------|---------|
| "UPDATE replaces the old row immediately for everyone" | PostgreSQL keeps old row versions long enough for snapshots that still need them. |
| "DELETE removes the row instantly" | Older snapshots may still see the row. The row becomes invisible to future snapshots at the appropriate time. |
| "MVCC means no bloat" | MVCC **creates** dead tuples naturally. Without proper vacuuming, tables and indexes **will** bloat. |
| "MVCC means no locks" | PostgreSQL still uses locking for writer/writer conflicts, DDL, and serializable conflict control. |
| "Read Committed gives one fixed view for the whole transaction" | In Read Committed, **each statement** can get a fresh snapshot. Only Repeatable Read and Serializable keep a stable transaction-wide snapshot. |
| "VACUUM is optional maintenance" | VACUUM is **essential** to MVCC health. Without it, dead tuples accumulate, bloat grows, and transaction ID wraparound can halt the entire database. |
| "Long-running transactions are harmless" | They prevent VACUUM from cleaning up old row versions, causing **table and index bloat**. |

---

## 9. Best Practices

### Operational

- **Don't disable autovacuum** — it is central to MVCC health, not optional maintenance.
- **Keep transactions short** — long-running transactions hold old snapshots alive, preventing VACUUM from cleaning dead tuples.
- **Monitor dead tuples** — watch `n_dead_tup` in `pg_stat_user_tables` to spot vacuum lag.
- **Tune autovacuum for high-churn tables** — tables with heavy UPDATE/DELETE traffic may need more aggressive autovacuum settings.

```sql
-- Per-table autovacuum tuning example:
ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.01,   -- vacuum after 1% of rows are dead
    autovacuum_analyze_scale_factor = 0.005
);
```

### Application Design

- **Use appropriate isolation levels** — don't default to Serializable everywhere; Read Committed is sufficient for most use cases.
- **Implement retry logic for Serializable** — SSI may abort your transaction; your application must be prepared to retry.
- **Avoid `SELECT FOR UPDATE` on rows you only read** — it escalates to a row-level lock unnecessarily.
- **Batch large UPDATE/DELETE operations** — processing millions of rows in a single transaction creates millions of dead tuples at once.

```sql
-- Instead of one massive DELETE:
DELETE FROM logs WHERE created_at < '2025-01-01';

-- Batch it:
DO $$
BEGIN
    LOOP
        DELETE FROM logs
        WHERE ctid IN (
            SELECT ctid FROM logs
            WHERE created_at < '2025-01-01'
            LIMIT 10000
        );
        EXIT WHEN NOT FOUND;
        PERFORM pg_sleep(0.1);  -- let autovacuum breathe
    END LOOP;
END $$;
```

### Monitoring

```sql
-- Check for bloated tables:
SELECT
    schemaname, relname,
    n_live_tup, n_dead_tup,
    ROUND(n_dead_tup::numeric / GREATEST(n_live_tup, 1) * 100, 2) AS dead_pct,
    last_autovacuum
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;

-- Check for long-running transactions (MVCC bloat risk):
SELECT
    pid, now() - xact_start AS duration,
    state, query
FROM pg_stat_activity
WHERE state != 'idle'
  AND xact_start IS NOT NULL
ORDER BY duration DESC;
```

---

## 10. Quick Revision Points

- **MVCC** = Multi-Version Concurrency Control — PostgreSQL's way of allowing concurrent reads and writes without blocking.
- **Core rule**: A transaction sees row versions according to its **snapshot**, not simply "the latest bytes on disk."
- **Reads don't block writes; writes don't block reads** — this is the fundamental MVCC benefit.
- **UPDATE** ≈ create new version + mark old version obsolete (not an in-place overwrite).
- **DELETE** ≈ mark row as dead for future snapshots (not an instant erase).
- **Snapshot timing** depends on isolation level: per-statement (Read Committed) vs per-transaction (Repeatable Read / Serializable).
- **Dead tuples** = old row versions no longer visible to any active snapshot → cleaned up by VACUUM.
- **VACUUM** is not optional — it removes dead tuples, updates the visibility map, and prevents transaction ID wraparound.
- **Autovacuum** runs in the background; monitor it, tune it for hot tables, never disable it.
- **MVCC ≠ no locks** — writer/writer conflicts, DDL, and Serializable SSI still use locking.
- **Long-running transactions** are the #1 enemy of MVCC health — they pin old snapshots and block cleanup.

### One-Sentence Summary

> MVCC in PostgreSQL means each transaction reads from a snapshot, writes create new row versions, old versions remain until no snapshot needs them, and VACUUM later cleans them up.

---

## Related Topics

- [ACID Properties](01-acid.md) — Isolation is implemented via MVCC.
- [Transactions](02-transactions.md) — The lifecycle of a transaction and how snapshots fit in.
- [Locking Modes](07-locking-modes.md) — What MVCC does *not* handle — lock-based conflict resolution.
- [Indexes](05-indexes.md) — Index-only scans depend on the visibility map.
- [Partitioning](09-partitioning.md) — Vacuum behavior across partitioned tables.

---

## Resources

- [PostgreSQL Official Docs — Concurrency Control (Chapter 13)](https://www.postgresql.org/docs/current/mvcc.html)
- [Viblo — PostgreSQL MVCC](https://viblo.asia/p/011-postgresql-multi-version-concurrency-control-6J3ZgdGLlmB)
- [Heroku — PostgreSQL Concurrency](https://devcenter.heroku.com/articles/postgresql-concurrency)
