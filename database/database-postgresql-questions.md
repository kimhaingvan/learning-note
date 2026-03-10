# PostgreSQL — Study Roadmap & Questions

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> A structured roadmap of PostgreSQL topics to study, with key resources and exercises.

---

## Table of Contents

1. [Official Resources](#1-official-resources)
2. [Core Topics](#2-core-topics)
3. [Why Pagination with OFFSET Is Slow](#3-why-pagination-with-offset-is-slow)
4. [Practice Exercises](#4-practice-exercises)

---

## 1. Official Resources

| Resource | URL |
|----------|-----|
| PostgreSQL Official Site | https://www.postgresql.org/ |
| PostgreSQL Tutorial | https://www.postgresqltutorial.com/ |
| Server Reference Docs | https://www.postgresql.org/docs/current/reference-server.html |
| Internals Documentation | https://www.postgresql.org/docs/current/internals.html |

---

## 2. Core Topics

### Data Modeling and Normalization

- **Normalization Principles** — 1NF, 2NF, 3NF, BCNF: eliminating redundancy and ensuring data integrity through decomposition.
- **Denormalization Techniques** — Intentionally introducing redundancy to improve read performance (common in OLAP/reporting workloads).
- **Constraints and Data Integrity** — `NOT NULL`, `UNIQUE`, `CHECK`, `PRIMARY KEY`, `FOREIGN KEY`, `EXCLUDE` constraints to enforce valid data at the database level.

### Transactions and Concurrency Control

- **ACID Properties** — Atomicity, Consistency, Isolation, Durability.
- **Two-Phase Commit** — A protocol for distributed transactions across multiple systems.
- **Transaction Logs** — WAL (Write-Ahead Logging) for durability and crash recovery.

### Isolation Levels

- READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE.
- PostgreSQL's MVCC-based implementation.

### Multi-Version Concurrency Control (MVCC)

- How PostgreSQL provides snapshot isolation without blocking readers.
- Tuple versioning with `xmin` / `xmax` system columns.

### Locking Mechanisms

- Table-level locks, row-level locks, advisory locks.
- Deadlock detection.

### Data Types and Extensions

- **Arrays** — Store and query multi-valued columns.
- **Enumerated Types** — Custom types with a fixed set of values.
- **JSON and JSONB** — Semi-structured data. JSONB is binary, indexable, and more performant.
- **HSTORE** — Key-value pairs within a single column.
- **Handling Time Zones and Dates** — `TIMESTAMP WITH TIME ZONE` vs `TIMESTAMP WITHOUT TIME ZONE`, `AT TIME ZONE`, and timezone-aware best practices.

### Stored Procedures and Functions

- **Procedural Languages**: PL/pgSQL (default), PL/Python, PL/Perl, PL/Tcl.
- **Function Types**:
  - `IMMUTABLE` — Same inputs always return the same output. Safe to cache. Can be used in index expressions.
  - `STABLE` — Returns same result within a single table scan (can read DB but not modify).
  - `VOLATILE` — May return different results on each call. Default. Cannot be used in index definitions.
- **Advanced Data Manipulation**:
  - `UPSERT` with `INSERT ... ON CONFLICT DO UPDATE`
  - `INSERT ... ON CONFLICT DO NOTHING`
  - `ON DELETE CASCADE` / `SET NULL` / `SET DEFAULT` / `RESTRICT`

### Triggers

- **Trigger Types**:
  - **Row-Level Triggers** — Fire once per affected row.
  - **Statement-Level Triggers** — Fire once per SQL statement.
- **Trigger Timing**:
  - `BEFORE` — Runs before the operation; can modify the new row or cancel the operation.
  - `AFTER` — Runs after the operation; sees final row state.
  - `INSTEAD OF` — Replaces the operation (used on views).

### Backup and Recovery

- SQL Dump (`pg_dump`, `pg_dumpall`)
- File System Level Backup
- Continuous Archiving / Point-in-Time Recovery (PITR)

### PostgreSQL Architecture

- **Process Model** — Postmaster (main process) forks a backend process per connection.
- **Client-Server Model** — Clients connect over TCP/IP; server manages all data access.
- **Data Storage** — Heap files, TOAST (for large values), FSM (free space map).
- **Background Processes** — Autovacuum, WAL writer, checkpointer, stats collector, background writer.
- **Tablespaces** — Control physical storage placement of database objects.
- **Page Structure** — 8 KB fixed-size pages: page header, item pointers, tuples, special space.
- **Write-Ahead Logging (WAL)** — All changes are written to WAL before data files, ensuring crash recovery.

### Advanced SQL Queries

- **Subqueries and CTEs** — Common Table Expressions (`WITH` clauses) for readability and recursive logic.
- **Recursive Queries** — `WITH RECURSIVE` for tree/graph traversal (e.g., org charts, category hierarchies).
- **Set Operations** — `UNION`, `INTERSECT`, `EXCEPT`.
- **Full-Text Search** — `tsvector`, `tsquery`, `to_tsvector()`, `to_tsquery()`, GIN indexes.

### Indexing and Performance Optimization

- **Index Types**: B-tree (default), Hash, GIN (inverted), GiST (generalized search), SP-GiST, BRIN.
- **Indexing Strategies**: Partial Indexes, Expression Indexes, Covering Indexes (`INCLUDE`), Multicolumn Indexes.
- **Query Optimization**: `EXPLAIN`, `EXPLAIN ANALYZE`, understanding execution plans (Seq Scan, Index Scan, Bitmap Scan, etc.).
- **Vacuuming and Analyze**: `VACUUM`, `VACUUM FULL`, `ANALYZE`, autovacuum tuning.

### Partitioning and Sharding

- Declarative partitioning: RANGE, LIST, HASH.
- Sub-partitioning.
- Partition pruning and performance implications.

### Integration with Other Systems

- Using PostgreSQL with message brokers like **Kafka** and **RabbitMQ**.
- Logical replication for event-driven architectures.
- Foreign Data Wrappers (`postgres_fdw`) for cross-system queries.

---

## 3. Why Pagination with OFFSET Is Slow

`OFFSET` is slow because the database must still **scan and discard** all the skipped rows. It does **not** jump directly to row N.

```sql
SELECT *
FROM orders
ORDER BY created_at
LIMIT 10 OFFSET 10000;

-- OFFSET 10     → scan 10 rows, discard 10
-- OFFSET 1,000  → scan 1,000 rows, discard 1,000
-- OFFSET 1,000,000 → scan 1,000,000 rows, discard all of them 😱
```

### Why This Happens

- The database **reads rows 1 through 10,000** and throws them all away.
- It then returns only rows 10,001 through 10,010.
- As the offset grows, query time grows **linearly**.

### Better Alternative: Keyset Pagination (Cursor-based)

Instead of skipping rows by count, filter by the **last seen value**:

```sql
-- First page
SELECT * FROM orders ORDER BY created_at LIMIT 10;

-- Next page (using the last created_at from previous page)
SELECT * FROM orders
WHERE created_at > '2025-06-15T12:00:00Z'
ORDER BY created_at
LIMIT 10;
```

This approach uses an **index seek** instead of a scan+discard, and performs consistently regardless of how deep into the dataset you paginate.

---

## 4. Practice Exercises

| Platform | URL | Notes |
|----------|-----|-------|
| Mode SQL Tutorial | https://mode.com/sql-tutorial/sql-select-statement | Recommended |
| LeetCode Database | https://leetcode.com/problemset/database/ | SQL problem sets |
| HackerRank SQL | https://www.hackerrank.com/domains/sql | Beginner → Advanced |
| PG Exercises | https://pgexercises.com/questions/basic | PostgreSQL-specific |
| Stack Overflow (PostgreSQL) | https://stackoverflow.com/questions/tagged/postgresql | Community Q&A |

---

## What to Learn Next

1. Study each core topic above in depth, starting with **Transactions & ACID**, then **Indexing**.
2. Practice with **PG Exercises** and **LeetCode Database** to solidify SQL skills.
3. Read the PostgreSQL **Internals** documentation to understand MVCC, WAL, and the query planner.
4. Set up a local PostgreSQL instance and experiment with `EXPLAIN ANALYZE` on real queries.
