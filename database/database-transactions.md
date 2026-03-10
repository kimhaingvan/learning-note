# Transactions

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> Understanding what transactions are, when to use them, and how their lifecycle works.

---

## Table of Contents

1. [What Is a Transaction?](#1-what-is-a-transaction)
2. [When to Use Transactions](#2-when-to-use-transactions)
3. [Transaction Lifespan](#3-transaction-lifespan)
4. [Implicit vs Explicit Transactions](#4-implicit-vs-explicit-transactions)

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

## What to Learn Next

1. **ACID Properties** — The four guarantees every transaction provides.
2. **Isolation Levels** — How concurrent transactions interact with each other.
3. **Read Phenomena** — Dirty reads, non-repeatable reads, phantom reads.
4. **Locking** — How databases prevent conflicts between concurrent transactions.
