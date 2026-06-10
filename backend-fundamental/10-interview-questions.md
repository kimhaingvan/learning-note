# Backend Interview Questions

> **Level**: Intermediate → Advanced | **Tone**: Technical / Teacher-like  
> Common backend interview questions with detailed answers, diagrams, SQL/code examples, and cross-references.

---

## Table of Contents

1. [Database & Concurrency](#1-database--concurrency)

---

## 1. Database & Concurrency

### Q1. How do you prevent two transactions from operating on the same record at the same time?

> **Summary**: This is a **concurrent write** (race condition) problem on the same row. There are **5 approaches** from simple to advanced: Atomic UPDATE, Pessimistic Locking, Optimistic Locking, Serializable Isolation, Advisory Lock.

---

#### Context — How Does the Problem Occur?

```
Transaction A                          Transaction B
    │                                      │
    │── BEGIN                              │── BEGIN
    │── SELECT balance FROM accounts       │── SELECT balance FROM accounts
    │   WHERE id = 1;  → balance = 1000   │   WHERE id = 1;  → balance = 1000
    │                                      │
    │── UPDATE accounts                    │── UPDATE accounts
    │   SET balance = 1000 - 200           │   SET balance = 1000 - 300
    │   WHERE id = 1;  → 800              │   WHERE id = 1;  → 700
    │── COMMIT                             │── COMMIT
    │                                      │
    ▼                                      ▼
    Expected: 1000 - 200 - 300 = 500
    Actual:   700 (Transaction B overwrites A's result)
    → LOST UPDATE!
```

Both transactions read `balance = 1000`, then compute based on the stale value → the result of the first transaction is lost.

---

#### Approach 1: Pessimistic Locking (`SELECT ... FOR UPDATE`)

**Idea**: Lock the row immediately when reading — other transactions must **wait** until the lock is released.

**When to use**: When **conflict probability is high** (many users modifying the same record).

```sql
-- Transaction A
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- ← LOCK row
-- balance = 1000
UPDATE accounts SET balance = 1000 - 200 WHERE id = 1;
COMMIT;  -- ← RELEASE lock

-- Transaction B (runs concurrently)
BEGIN;
SELECT balance FROM accounts WHERE id = 1 FOR UPDATE;  -- ⏳ BLOCKED — waits for A to commit
-- After A commits → reads balance = 800
UPDATE accounts SET balance = 800 - 300 WHERE id = 1;  -- → 500 ✅
COMMIT;
```

**Variants**:

| Clause | Behavior |
|--------|----------|
| `FOR UPDATE` | Strongest lock — blocks both FOR UPDATE reads and writes |
| `FOR NO KEY UPDATE` | Lighter — does not block `FOR KEY SHARE` (use when not changing PK/FK) |
| `FOR SHARE` | Shared lock — multiple transactions can read, but none can write |
| `FOR KEY SHARE` | Lightest — only blocks changes to key columns |
| `NOWAIT` | Don't wait — returns error immediately if row is locked |
| `SKIP LOCKED` | Skip locked rows — very useful for **job queue** pattern |

```sql
-- NOWAIT: Fail immediately if locked
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row

-- SKIP LOCKED: Skip locked rows (job queue pattern)
SELECT * FROM tasks WHERE status = 'pending'
ORDER BY created_at
LIMIT 1
FOR UPDATE SKIP LOCKED;
```

**Pros**: Simple, absolutely guarantees no lost updates.  
**Cons**: Reduces throughput (transactions must wait for each other), can cause **deadlocks**.

---

#### Approach 2: Optimistic Locking (Version Column)

**Idea**: Don't lock when reading. When writing, check if the data has changed since it was read. If it has → **retry**.

**When to use**: When **conflict probability is low** (few users modifying the same record concurrently).

**Step 1**: Add a `version` column to the table.

```sql
CREATE TABLE accounts (
    id      SERIAL PRIMARY KEY,
    balance NUMERIC NOT NULL,
    version INT NOT NULL DEFAULT 0
);
```

**Step 2**: Read the row along with its version.

```sql
SELECT balance, version FROM accounts WHERE id = 1;
-- balance = 1000, version = 5
```

**Step 3**: Update with a version-match condition.

```sql
UPDATE accounts
SET balance = 1000 - 200, version = version + 1
WHERE id = 1 AND version = 5;
-- Rows affected: 1 → Success ✅
-- Rows affected: 0 → Someone else changed it → RETRY ❌
```

**Full flow**:

```
Transaction A                          Transaction B
    │                                      │
    │── SELECT (version=5, balance=1000)   │── SELECT (version=5, balance=1000)
    │                                      │
    │── UPDATE WHERE version=5             │
    │   → rows affected = 1 ✅             │
    │   → version becomes 6               │
    │── COMMIT                             │
    │                                      │── UPDATE WHERE version=5
    │                                      │   → rows affected = 0 ❌
    │                                      │   → RETRY: re-read, re-calculate
```

**Go implementation**:

```go
func transferMoney(db *sql.DB, accountID int, amount float64) error {
    maxRetries := 3
    for i := 0; i < maxRetries; i++ {
        // Step 1: Read current state
        var balance float64
        var version int
        err := db.QueryRow(
            "SELECT balance, version FROM accounts WHERE id = $1", accountID,
        ).Scan(&balance, &version)
        if err != nil {
            return err
        }

        // Step 2: Business logic
        newBalance := balance - amount
        if newBalance < 0 {
            return fmt.Errorf("insufficient balance")
        }

        // Step 3: Conditional update
        result, err := db.Exec(
            "UPDATE accounts SET balance = $1, version = version + 1 WHERE id = $2 AND version = $3",
            newBalance, accountID, version,
        )
        if err != nil {
            return err
        }

        rowsAffected, _ := result.RowsAffected()
        if rowsAffected == 1 {
            return nil // Success
        }
        // rowsAffected == 0 → conflict → retry
    }
    return fmt.Errorf("optimistic lock: max retries exceeded")
}
```

**Pros**: Doesn't block other transactions, higher throughput than pessimistic.  
**Cons**: Must handle retry logic, not suitable when conflict rate is high.

---

#### Approach 3: Atomic UPDATE (Simplest)

**Idea**: Use a **single UPDATE statement** instead of SELECT then UPDATE — the database handles row-level locking internally.

```sql
-- ❌ Wrong: SELECT then UPDATE (race condition)
SELECT balance FROM accounts WHERE id = 1;     -- 1000
UPDATE accounts SET balance = 1000 - 200 WHERE id = 1;

-- ✅ Correct: Atomic UPDATE (database locks the row automatically)
UPDATE accounts SET balance = balance - 200 WHERE id = 1;
```

When using `SET balance = balance - 200`, PostgreSQL automatically:
1. Acquires a lock on the row.
2. Reads the current value.
3. Computes and writes.
4. Releases the lock on commit.

Transaction B waits for A to commit before executing → correct result.

```sql
-- Transaction A: UPDATE accounts SET balance = balance - 200 WHERE id = 1;
-- Transaction B: UPDATE accounts SET balance = balance - 300 WHERE id = 1;
-- Result: 1000 - 200 - 300 = 500 ✅
```

**Pros**: Simplest approach, no extra code or columns needed.  
**Cons**: Only works when the logic is simple enough to express in a single SQL statement (no need to read the value first for complex application-level computation).

---

#### Approach 4: Serializable Isolation Level

**Idea**: Tell the database to guarantee that transactions execute **as if they were sequential** — if a conflict is detected, the database aborts one transaction.

```sql
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
SELECT balance FROM accounts WHERE id = 1;  -- 1000
UPDATE accounts SET balance = 1000 - 200 WHERE id = 1;
COMMIT;
```

If 2 serializable transactions conflict → PostgreSQL aborts one of them with this error:

```
ERROR: could not serialize access due to concurrent update
```

The application must catch this error and **retry**.

```go
func transferWithSerializable(db *sql.DB, accountID int, amount float64) error {
    maxRetries := 3
    for i := 0; i < maxRetries; i++ {
        tx, err := db.BeginTx(ctx, &sql.TxOptions{
            Isolation: sql.LevelSerializable,
        })
        if err != nil {
            return err
        }

        var balance float64
        err = tx.QueryRow("SELECT balance FROM accounts WHERE id = $1", accountID).Scan(&balance)
        if err != nil {
            tx.Rollback()
            return err
        }

        _, err = tx.Exec("UPDATE accounts SET balance = $1 WHERE id = $2", balance-amount, accountID)
        if err != nil {
            tx.Rollback()
            // Check if serialization failure → retry
            if isSerializationError(err) {
                continue
            }
            return err
        }

        err = tx.Commit()
        if err != nil {
            if isSerializationError(err) {
                continue
            }
            return err
        }
        return nil // Success
    }
    return fmt.Errorf("serializable: max retries exceeded")
}
```

**Pros**: Database guarantees correctness, no extra columns or complex logic needed.  
**Cons**: Highest performance cost, must retry on abort, adds overhead to all transactions.

---

#### Approach 5: Advisory Lock (Application-Level Lock)

**Idea**: Lock on an **arbitrary key** (usually an ID) — doesn't lock the row itself, but locks at the application logic level.

```sql
-- Lock on account_id = 1 (session-level lock)
SELECT pg_advisory_lock(1);

-- Perform logic
SELECT balance FROM accounts WHERE id = 1;
UPDATE accounts SET balance = balance - 200 WHERE id = 1;

-- Unlock
SELECT pg_advisory_unlock(1);
```

```sql
-- Transaction-level advisory lock (auto-unlocks on commit/rollback)
BEGIN;
SELECT pg_advisory_xact_lock(1);  -- lock for account_id = 1
-- ... logic ...
COMMIT;  -- automatically unlocked
```

**When to use**: When you need to lock based on **business logic** (not just a single row, but a group of related operations).

**Pros**: Flexible, lighter than row locks, locks based on business meaning.  
**Cons**: Must manage lock/unlock carefully, not automatic like row locks.

---

#### Summary — Which Approach to Choose?

| Approach | Conflict Rate | Complexity | Throughput | Best For |
|----------|---------------|------------|------------|----------|
| **Atomic UPDATE** | Any | ⭐ Very simple | ⭐⭐⭐ High | Simple logic (increment, decrement) |
| **Pessimistic Lock** (`FOR UPDATE`) | High | ⭐⭐ Simple | ⭐⭐ Medium | Financial transactions, inventory |
| **Optimistic Lock** (version) | Low | ⭐⭐⭐ Medium | ⭐⭐⭐ High | CMS, form editing, low-conflict CRUD |
| **Serializable** | Any | ⭐⭐ Simple (DB handles it) | ⭐ Low | Critical correctness, complex logic |
| **Advisory Lock** | Custom | ⭐⭐⭐ Complex | ⭐⭐ Varies | Job queue, distributed locking, batch |

**General guidelines**:
1. **Prefer Atomic UPDATE** if the logic is simple enough.
2. **Pessimistic Lock** when conflict rate is high and absolute correctness is required.
3. **Optimistic Lock** when conflict rate is low and you want high throughput.
4. **Serializable** when logic is complex and you need the database to guarantee correctness.
5. **Advisory Lock** when you need locking based on business logic beyond row-level.

---

#### Sample Interview Answer

> "There are several approaches depending on the context:
>
> **The simplest** is an **atomic UPDATE** (`SET balance = balance - X`) — the database handles row-level locking internally.
>
> If you need to SELECT first then UPDATE, there are 2 directions:
> - **Pessimistic locking** (`SELECT ... FOR UPDATE`) — locks the row immediately on read, suitable when conflict rate is high (e.g., money transfers, inventory deduction).
> - **Optimistic locking** (add a `version` column) — doesn't lock on read, checks version on write, retries on conflict. Suitable when conflict rate is low.
>
> You can also use **Serializable isolation** (database auto-aborts on conflict) or **Advisory Locks** (lock on a business key).
>
> In practice, I typically use **atomic UPDATE** for simple logic and **pessimistic lock** for complex read-compute-write scenarios."

> Cross-reference: [14-mvcc.md](../database/14-mvcc.md) · [15-locking-modes.md](../database/15-locking-modes.md) · [13-transactions.md](../database/13-transactions.md) · [12-acid.md](../database/12-acid.md)
