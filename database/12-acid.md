# ACID Properties

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 5.1 — ACID Properties / Phase 5.3 — Isolation Levels  
> The four critical properties that every transaction in a relational database must guarantee.

---

## Table of Contents

1. [What Is ACID?](#1-what-is-acid)
2. [Atomicity (Tính nguyên tử)](#2-atomicity-tính-nguyên-tử)
3. [Consistency (Tính nhất quán)](#3-consistency-tính-nhất-quán)
4. [Isolation (Tính cô lập)](#4-isolation-tính-cô-lập)
   - [Read Phenomena](#read-phenomena)
   - [Isolation Levels](#isolation-levels)
   - [How Databases Implement Isolation](#how-databases-implement-isolation)
5. [Durability (Tính bền vững)](#5-durability-tính-bền-vững)
6. [Why ACID Matters](#6-why-acid-matters)

---

## 1. What Is ACID?

ACID stands for **four critical properties** that every transaction in a relational database system must satisfy:

| Property | Vietnamese | One-Line Summary |
|----------|-----------|-----------------|
| **A** — Atomicity | Tính nguyên tử | All or nothing — if one query fails, everything rolls back. |
| **C** — Consistency | Tính nhất quán | Data must remain valid according to all defined rules. |
| **I** — Isolation | Tính cô lập | Transactions execute independently of each other. |
| **D** — Durability | Tính bền vững | Committed data survives crashes. |

---

## 2. Atomicity (Tính nguyên tử)

### What

An **atomic transaction** will **rollback all queries** if one or more queries fail, or if the database crashes mid-transaction.

- A transaction acts as a **unit of work** — it cannot be split.
- **Either all occurs, or nothing occurs** (Được ăn cả, ngã về không).

### Why

Without atomicity, data would be **lost or inconsistent**:

- All queries in a transaction **MUST succeed**.
- If **one query fails** or the **database crashes**, all queries in the transaction **MUST be rolled back**.

### Example

```
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;   -- succeeds
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;   -- FAILS (e.g., constraint violation)
ROLLBACK;  -- first UPDATE is also undone — no money is lost
```

> **Plain English**: Atomicity is like an "all-or-nothing" switch. Either the entire transaction completes perfectly, or it's as if it never happened at all.

---

## 3. Consistency (Tính nhất quán)

### What

Consistency requires that **any data changed by a transaction must be valid and allowed** according to all defined rules, including:

- Constraints (`NOT NULL`, `UNIQUE`, `CHECK`, `FOREIGN KEY`)
- Cascades
- Triggers

If a transaction violates **any** constraint or rule, the data changes by that transaction **must be rolled back**.

### Why

Without consistency, data could be changed in **un-allowed, invalid ways** — for example:

- Inserting a row that violates a foreign key constraint.
- Setting a balance to a negative number when a `CHECK (balance >= 0)` exists.

### Example

```sql
-- Table has: CHECK (balance >= 0)
BEGIN;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
-- If balance was 500, this violates the CHECK constraint
-- → Transaction must ROLLBACK, balance stays at 500
```

---

## 4. Isolation (Tính cô lập)

### What

Isolation requires that transactions are **committed and executed independently** of each other.

### Why

To ensure the database **doesn't produce errors** caused by read phenomena — situations that arise when multiple transactions read or write the same rows concurrently.

---

### Read Phenomena

Read phenomena are **undesired side effects** that occur when many transactions read or write on the same row at the same time (concurrently).

#### Dirty Reads

A situation where **Transaction A reads data that was updated by Transaction B but not yet committed**. If Transaction B rolls back, Transaction A has read data that essentially **never existed**.

```
Timeline:
  Tx A                              Tx B
  ─────                             ─────
                                    BEGIN
                                    UPDATE accounts SET balance = 200 WHERE id = 1
  BEGIN
  SELECT balance FROM accounts
    WHERE id = 1
  → reads 200 (UNCOMMITTED!)
                                    ROLLBACK  ← balance is back to original
  → Transaction A used a value
    that was never committed ❌
```

#### Non-Repeatable Reads

A situation where **Transaction A reads the same row twice but gets different values**, because the row was **updated and committed** by Transaction B between the two reads.

```
Timeline:
  Tx A                              Tx B
  ─────                             ─────
  BEGIN
  SELECT balance FROM accounts
    WHERE id = 1
  → reads 500
                                    BEGIN
                                    UPDATE accounts SET balance = 300 WHERE id = 1
                                    COMMIT
  SELECT balance FROM accounts
    WHERE id = 1
  → reads 300 (DIFFERENT!) ❌
```

#### Phantom Reads

A situation where **Transaction A runs a query twice but gets different row sets**, because Transaction B **inserted new rows** that now match Transaction A's query.

```
Timeline:
  Tx A                              Tx B
  ─────                             ─────
  BEGIN
  SELECT * FROM accounts
    WHERE balance > 100
  → returns 3 rows
                                    BEGIN
                                    INSERT INTO accounts (balance) VALUES (500)
                                    COMMIT
  SELECT * FROM accounts
    WHERE balance > 100
  → returns 4 rows (PHANTOM ROW!) ❌
```

---

### Isolation Levels

Isolation levels are **configurations that determine which read phenomena are allowed**.

| Isolation Level | Dirty Reads | Non-Repeatable Reads | Phantom Reads |
|----------------|-------------|---------------------|---------------|
| **Read Uncommitted** | Possible | Possible | Possible |
| **Read Committed** | Not Possible | Possible | Possible |
| **Repeatable Read** | Not Possible | Not Possible | Possible |
| **Snapshot** | Not Possible | Not Possible | Not Possible |
| **Serializable** | Not Possible | Not Possible | Not Possible |

#### Read Uncommitted

- **No isolation** — any change from outside is visible to the transaction, committed or not (Đọc cả những thứ chưa được commit).
- Only SQL Server supports it. PostgreSQL treats it as Read Committed.

#### Read Committed

- A query in a transaction only sees data that was **committed before the query started** (Chỉ đọc được những data đã được commit trước khi **query bắt đầu**).
- **Default level for many databases**, including PostgreSQL 15.

#### Repeatable Read

- The transaction only sees data that was **committed before the transaction started** (Chỉ đọc được những data đã được commit trước khi **transaction đó bắt đầu**).
- If Transaction A reads a row, that row **cannot change** from A's perspective while A is running.
- **Default level for MySQL 8**.

#### Snapshot

- Each query in a transaction only sees changes that have been committed **at the start of the transaction** — like a snapshot version at that moment.
- In **PostgreSQL**, Snapshot and Repeatable Read are the same.

#### Serializable

- Transactions do **not work concurrently** but **sequentially**, one after the other.
- Strongest isolation, but lowest concurrency.

> **Performance note**: PostgreSQL 15 defaults to **Read Committed**; MySQL 8 defaults to **Repeatable Read**. PostgreSQL is often faster in concurrent workloads because its default isolation level is weaker (less locking overhead).

---

### How Databases Implement Isolation

| Strategy | How It Works | Example |
|----------|-------------|---------|
| **Pessimistic Locking** | Acquires **locks** (row-level, table-level, page-level) to prevent concurrent access and avoid lost updates. | `SELECT ... FOR UPDATE` |
| **Optimistic Locking** | Uses **no locks**. Instead, tracks whether data changed since it was read, and **fails the transaction** if a conflict is detected. | Application-level `version_number` column |

---

## 5. Durability (Tính bền vững)

### What

Durability requires that **all data and changes committed by transactions must be persisted in durable, non-volatile storage** — even if the database crashes, the OS crashes, or the power goes out.

### How Databases Achieve Durability

| Mechanism | Description |
|-----------|------------|
| **WAL (Write-Ahead Log)** | All changes are written to a sequential log **before** being applied to data files. On crash recovery, the WAL is replayed to restore committed transactions. |
| **Asynchronous Snapshot** | Periodic snapshots of the database state written to disk. |
| **AOF (Append-Only File)** | Every write operation is appended to a file (common in Redis). |

> **Plain English**: Durability means "once the database says COMMIT succeeded, that data is safe — even if the server catches fire one second later."

---

## 6. Why ACID Matters

ACID properties exist to **ensure data in the database is valid and consistent** at all times.

Without ACID:
- Partial writes could leave data in a broken state (no Atomicity).
- Invalid data could be stored (no Consistency).
- Concurrent transactions could read stale or ghost data (no Isolation).
- Committed data could disappear after a crash (no Durability).

---

## What to Learn Next

1. **MVCC (Multi-Version Concurrency Control)** — How PostgreSQL implements isolation without heavy locking.
2. **WAL internals** — How Write-Ahead Logging works at the page level.
3. **Two-Phase Commit** — How distributed transactions coordinate across multiple databases.
4. **Deadlock detection** — How databases resolve circular lock dependencies.
5. **Transaction performance** — Impact of long-running transactions on VACUUM and bloat.
