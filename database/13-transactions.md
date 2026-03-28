# Transactions

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 5.2 — Transaction Control  
> Understanding what transactions are, when to use them, how their lifecycle works, savepoints, two-phase commit, and transaction states.

---

## Table of Contents

1. [What Is a Transaction?](#1-what-is-a-transaction)
2. [When to Use Transactions](#2-when-to-use-transactions)
3. [Transaction Lifespan](#3-transaction-lifespan)
4. [Implicit vs Explicit Transactions](#4-implicit-vs-explicit-transactions)
5. [Savepoints](#5-savepoints)
6. [Two-Phase Commit](#6-two-phase-commit)
7. [Transaction States](#7-transaction-states)

---

## 1. What Is a Transaction?

A **transaction** is a **unit of work** that includes one or more SQL queries, treated as a single logical operation.

- It is a **collection of queries** that either all succeed together or all fail together.
- You are **always inside a transaction**. If you execute a single statement (e.g., an `INSERT` or `UPDATE`) without an explicit `BEGIN`, the database will:
  1. Start a transaction **implicitly**.
  2. Execute the statement.
  3. **Immediately commit** the result.

> **Plain English**: A transaction is like a contract — either the entire deal goes through, or nothing happens at all. No half-done state.

---

## 2. When to Use Transactions

| Scenario | Use Transaction? |
|----------|-----------------|
| Changing or modifying data (`INSERT`, `UPDATE`, `DELETE`) | **Yes — always** |
| Read-only queries | Optional — use when you want a **consistent snapshot** of data at the time the transaction began |
| Multiple related operations that must succeed or fail together | **Yes — required** |

**Example — Transferring money between accounts**:

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;   -- debit
UPDATE accounts SET balance = balance + 100 WHERE id = 2;   -- credit
COMMIT;
```

If the second `UPDATE` fails, `ROLLBACK` undoes the first — no money is lost or duplicated.

---

## 3. Transaction Lifespan

A transaction has three key lifecycle commands:

```
BEGIN
  │
  ├── query 1
  ├── query 2
  ├── query 3
  │
  ├── COMMIT     ← persist all changes to disk
  │    OR
  └── ROLLBACK   ← discard all changes
```

| Command | What It Does |
|---------|-------------|
| `BEGIN` | Opens a transaction. Tells the database: *"Hey, I'm starting my work — open a transaction for me."* |
| `COMMIT` | Closes the transaction successfully. Tells the database: *"Hey, I finished my work — please commit and persist these changes to disk."* |
| `ROLLBACK` | Aborts the transaction. Tells the database: *"Hey, forget all these changes — don't persist them to disk."* |

---

## 4. Implicit vs Explicit Transactions

### Implicit (Auto-Commit)

When you don't use `BEGIN`, the database wraps **each individual statement** in its own transaction and commits it immediately:

```sql
-- No BEGIN — each statement is its own transaction
INSERT INTO users (name) VALUES ('Alice');
-- ↑ internally: BEGIN → INSERT → COMMIT (automatic)
```

### Explicit

When you use `BEGIN`, you control exactly when to commit or rollback:

```sql
BEGIN;
INSERT INTO orders (user_id, total) VALUES (1, 99.99);
INSERT INTO order_items (order_id, product_id) VALUES (currval('orders_id_seq'), 5);
COMMIT;
```

> **Rule of thumb**: Use explicit transactions whenever **two or more operations must succeed or fail together**.

---

## 5. Savepoints

### What

A **SAVEPOINT** creates a named checkpoint inside a running transaction. You can **rollback to** that checkpoint without aborting the entire transaction.

### Why

Without savepoints, any error inside a transaction forces a **complete rollback**. Savepoints allow **partial recovery** — roll back just the failed part and continue.

### Syntax

```sql
BEGIN;

INSERT INTO orders (user_id, total) VALUES (1, 99.99);
SAVEPOINT before_items;

INSERT INTO order_items (order_id, product_id) VALUES (1, 999);
-- Oops, product 999 violates a FK constraint!

ROLLBACK TO SAVEPOINT before_items;
-- The order INSERT is still intact — only the item INSERT was undone

INSERT INTO order_items (order_id, product_id) VALUES (1, 5);  -- correct product
RELEASE SAVEPOINT before_items;  -- optional: discard the savepoint

COMMIT;
```

### Key Rules

| Rule | Detail |
|------|--------|
| Savepoints are **transaction-local** | They don't exist outside `BEGIN ... COMMIT` |
| `ROLLBACK TO` does **not** end the transaction | The transaction continues after rollback |
| `RELEASE SAVEPOINT` destroys the savepoint | You can no longer rollback to it |
| Savepoints can **nest** | `SAVEPOINT a` → `SAVEPOINT b` → `ROLLBACK TO a` also destroys `b` |
| PostgreSQL enters **aborted state** on error | Without savepoints, *any* error in a transaction makes all subsequent commands fail with `ERROR: current transaction is aborted` |

### Nested Savepoints

```sql
BEGIN;
SAVEPOINT sp1;
  INSERT INTO t VALUES (1);
  SAVEPOINT sp2;
    INSERT INTO t VALUES (2);
    ROLLBACK TO sp2;          -- undoes INSERT (2) only
  INSERT INTO t VALUES (3);   -- this works
COMMIT;                        -- rows 1 and 3 are committed
```

### Common Use Case — Error Recovery in PL/pgSQL

```sql
DO $$
BEGIN
    INSERT INTO users (email) VALUES ('alice@example.com');
EXCEPTION WHEN unique_violation THEN
    -- PL/pgSQL internally uses savepoints for EXCEPTION blocks
    RAISE NOTICE 'Email already exists, skipping.';
END $$;
```

---

## 6. Two-Phase Commit

### What

**Two-Phase Commit (2PC)** is a protocol for coordinating transactions across multiple databases or services. It ensures all participants either commit or abort together.

### How It Works

```
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  Coordinator │       │  Database A  │       │  Database B  │
└──────┬───────┘       └──────┬───────┘       └──────┬───────┘
       │  Phase 1: PREPARE    │                      │
       │ ─────────────────►   │                      │
       │  PREPARE TRANSACTION │                      │
       │  'txn_abc'           │                      │
       │  ◄──── OK ────────── │                      │
       │ ──────────────────────────────────────────►  │
       │                      │  PREPARE TRANSACTION  │
       │                      │  'txn_abc'            │
       │  ◄─────────────────────────────── OK ─────── │
       │                      │                      │
       │  Phase 2: COMMIT     │                      │
       │ ─────────────────►   │                      │
       │  COMMIT PREPARED     │                      │
       │  'txn_abc'           │                      │
       │ ──────────────────────────────────────────►  │
       │                      │  COMMIT PREPARED      │
       │                      │  'txn_abc'            │
```

### PostgreSQL Syntax

```sql
-- Phase 1: Prepare (locks are held, WAL is flushed, but not committed)
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
PREPARE TRANSACTION 'transfer_abc';
-- Transaction is now in a "prepared" state — survives crashes

-- Phase 2a: Commit (when all participants are ready)
COMMIT PREPARED 'transfer_abc';

-- Phase 2b: Rollback (if any participant failed to prepare)
ROLLBACK PREPARED 'transfer_abc';
```

### Configuration

```
# postgresql.conf — must set max_prepared_transactions > 0
max_prepared_transactions = 10   # default is 0 (disabled)
```

### Monitoring Prepared Transactions

```sql
-- View all currently prepared transactions
SELECT gid, prepared, owner, database
FROM pg_prepared_xacts;
```

### Key Rules and Risks

| Rule | Detail |
|------|--------|
| Prepared transactions **survive crashes** | They persist until explicitly committed or rolled back |
| Prepared transactions **hold locks** | They block other transactions just like active ones |
| **Orphaned prepared transactions** are dangerous | If the coordinator crashes and never resolves them, locks are held indefinitely |
| Requires `max_prepared_transactions > 0` | Disabled by default in PostgreSQL |
| Transaction name must be **globally unique** | Use UUIDs or application-generated identifiers |

> **Warning**: In practice, most applications use **application-level sagas** or **outbox patterns** instead of 2PC, because 2PC has failure modes (coordinator crash) that are hard to handle correctly. Use 2PC only when you truly need distributed ACID guarantees.

---

## 7. Transaction States

A PostgreSQL connection is always in one of these transaction states:

```
     ┌─────────────────┐
     │      idle        │  ← No transaction active
     └────────┬────────┘
              │ BEGIN
              ▼
     ┌─────────────────┐
     │ idle in          │  ← Transaction open, waiting for commands
     │ transaction      │
     └────────┬────────┘
              │ SQL error occurs
              ▼
     ┌─────────────────┐
     │ idle in          │  ← Transaction is aborted; only ROLLBACK
     │ transaction      │    or ROLLBACK TO SAVEPOINT works
     │ (aborted)        │
     └─────────────────┘
```

### Checking Transaction State

```sql
-- From pg_stat_activity
SELECT pid, state, xact_start, query
FROM pg_stat_activity
WHERE datname = current_database();
```

| State | Meaning |
|-------|--------|
| `idle` | Connection is open, no transaction |
| `active` | Currently executing a query |
| `idle in transaction` | Inside a `BEGIN` block, but between statements |
| `idle in transaction (aborted)` | An error occurred inside a transaction; awaiting `ROLLBACK` |
| `fastpath function call` | Executing a fast-path function |
| `disabled` | Tracking disabled for this backend |

### The "idle in transaction" Problem

Connections stuck in `idle in transaction` are dangerous:

- They **hold locks** that block other transactions.
- They **pin old MVCC snapshots**, preventing VACUUM from cleaning dead tuples.
- They **consume connection slots** (limited by `max_connections`).

```sql
-- Find idle-in-transaction connections older than 5 minutes
SELECT pid, now() - xact_start AS duration, query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND now() - xact_start > interval '5 minutes';

-- Terminate a stuck session (use with caution)
SELECT pg_terminate_backend(pid);
```

### Automatic Timeout Protection

```sql
-- Kill idle-in-transaction sessions after 10 minutes
SET idle_in_transaction_session_timeout = '10min';

-- Set globally in postgresql.conf
-- idle_in_transaction_session_timeout = '10min'
```

---

## What to Learn Next

1. **ACID Properties** — The four guarantees every transaction provides — see [01-acid.md](01-acid.md).
2. **Isolation Levels** — How concurrent transactions interact with each other — see [01-acid.md § Isolation](01-acid.md).
3. **MVCC** — How PostgreSQL implements isolation without heavy locking — see [10-mvcc.md](10-mvcc.md).
4. **Locking Modes** — How databases prevent conflicts between concurrent transactions — see [07-locking-modes.md](07-locking-modes.md).
5. **Deadlocks** — Detection, prevention, and resolution — see [07-locking-modes.md § Deadlocks](07-locking-modes.md).

---

> *Ref: [Docs — SAVEPOINT](https://www.postgresql.org/docs/18/sql-savepoint.html) · [Docs — PREPARE TRANSACTION](https://www.postgresql.org/docs/18/sql-prepare-transaction.html) · [Docs — Transaction Management](https://www.postgresql.org/docs/18/tutorial-transactions.html)*
