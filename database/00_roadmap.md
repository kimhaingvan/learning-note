# PostgreSQL Mastery Roadmap

> **Level**: Beginner → Advanced | **Tone**: Technical / Teacher-like
> A structured roadmap covering every major PostgreSQL concept — from fundamentals to internals — with brief descriptions and references to official documentation.

**Official References:**

| Resource | URL |
|----------|-----|
| PostgreSQL 18 Official Docs | https://www.postgresql.org/docs/18/index.html |
| Neon PostgreSQL Tutorial | https://neon.com/postgresql/tutorial |
| Neon PostgreSQL Administration | https://neon.com/postgresql/postgresql-administration |
| Neon PostgreSQL 18 New Features | https://neon.com/postgresql/postgresql-18-new-features |
| Neon PostgreSQL Indexes | https://neon.com/postgresql/postgresql-indexes |

---

## Table of Contents

- [Phase 1 — Foundations](#phase-1--foundations)
  - [1.1 Installation & Setup](#11-installation--setup)
  - [1.2 SQL Fundamentals](#12-sql-fundamentals)
  - [1.3 Data Types](#13-data-types)
  - [1.4 Table Basics & Constraints](#14-table-basics--constraints)
- [Phase 2 — Querying & Data Manipulation](#phase-2--querying--data-manipulation)
  - [2.1 Filtering & Sorting](#21-filtering--sorting)
  - [2.2 Joins](#22-joins)
  - [2.3 Grouping & Aggregation](#23-grouping--aggregation)
  - [2.4 Subqueries & CTEs](#24-subqueries--ctes)
  - [2.5 Set Operations](#25-set-operations)
  - [2.6 Window Functions](#26-window-functions)
- [Phase 3 — Schema Design & Data Modeling](#phase-3--schema-design--data-modeling)
  - [3.1 Normalization](#31-normalization)
  - [3.2 Denormalization](#32-denormalization)
  - [3.3 Table Inheritance & Partitioning](#33-table-inheritance--partitioning)
  - [3.4 Schemas & Namespaces](#34-schemas--namespaces)
  - [3.5 Sequences & Identity Columns](#35-sequences--identity-columns)
- [Phase 4 — Indexes & Query Performance](#phase-4--indexes--query-performance)
  - [4.1 Index Types](#41-index-types)
  - [4.2 Index Strategies](#42-index-strategies)
  - [4.3 EXPLAIN & Query Plans](#43-explain--query-plans)
  - [4.4 Query Optimization Techniques](#44-query-optimization-techniques)
  - [4.5 Statistics & the Planner](#45-statistics--the-planner)
- [Phase 5 — Transactions & Concurrency](#phase-5--transactions--concurrency)
  - [5.1 ACID Properties](#51-acid-properties)
  - [5.2 Transaction Control](#52-transaction-control)
  - [5.3 Isolation Levels](#53-isolation-levels)
  - [5.4 MVCC (Multi-Version Concurrency Control)](#54-mvcc-multi-version-concurrency-control)
  - [5.5 Locking Mechanisms](#55-locking-mechanisms)
  - [5.6 Deadlocks](#56-deadlocks)
- [Phase 6 — Stored Procedures, Functions & Triggers](#phase-6--stored-procedures-functions--triggers)
  - [6.1 PL/pgSQL Language](#61-plpgsql-language)
  - [6.2 User-Defined Functions](#62-user-defined-functions)
  - [6.3 Stored Procedures](#63-stored-procedures)
  - [6.4 Triggers & Event Triggers](#64-triggers--event-triggers)
  - [6.5 Rules System](#65-rules-system)
- [Phase 7 — Views & Materialized Views](#phase-7--views--materialized-views)
  - [7.1 Views](#71-views)
  - [7.2 Materialized Views](#72-materialized-views)
  - [7.3 Updatable Views](#73-updatable-views)
- [Phase 8 — Administration & Security](#phase-8--administration--security)
  - [8.1 Roles & Users](#81-roles--users)
  - [8.2 Privileges & GRANT/REVOKE](#82-privileges--grantrevoke)
  - [8.3 Row-Level Security (RLS)](#83-row-level-security-rls)
  - [8.4 Authentication (pg_hba.conf)](#84-authentication-pg_hbaconf)
  - [8.5 SSL/TLS Encryption](#85-ssltls-encryption)
  - [8.6 Auditing](#86-auditing)
- [Phase 9 — Backup, Restore & Disaster Recovery](#phase-9--backup-restore--disaster-recovery)
  - [9.1 Logical Backup (pg_dump / pg_restore)](#91-logical-backup-pg_dump--pg_restore)
  - [9.2 Physical Backup & PITR](#92-physical-backup--pitr)
  - [9.3 WAL Archiving](#93-wal-archiving)
  - [9.4 Continuous Archiving & Recovery](#94-continuous-archiving--recovery)
- [Phase 10 — Replication & High Availability](#phase-10--replication--high-availability)
  - [10.1 Streaming Replication](#101-streaming-replication)
  - [10.2 Logical Replication](#102-logical-replication)
  - [10.3 Failover & Switchover](#103-failover--switchover)
  - [10.4 Connection Pooling (PgBouncer)](#104-connection-pooling-pgbouncer)
  - [10.5 Load Balancing](#105-load-balancing)
- [Phase 11 — Performance Tuning & Monitoring](#phase-11--performance-tuning--monitoring)
  - [11.1 Configuration Tuning (postgresql.conf)](#111-configuration-tuning-postgresqlconf)
  - [11.2 Memory Architecture](#112-memory-architecture)
  - [11.3 VACUUM & Autovacuum](#113-vacuum--autovacuum)
  - [11.4 Monitoring & Diagnostics](#114-monitoring--diagnostics)
  - [11.5 pg_stat Catalog Views](#115-pg_stat-catalog-views)
  - [11.6 Bloat Management](#116-bloat-management)
- [Phase 12 — Advanced Data Features](#phase-12--advanced-data-features)
  - [12.1 JSON / JSONB](#121-json--jsonb)
  - [12.2 Arrays](#122-arrays)
  - [12.3 Hstore](#123-hstore)
  - [12.4 Full-Text Search](#124-full-text-search)
  - [12.5 PostGIS (Geospatial)](#125-postgis-geospatial)
  - [12.6 Range Types](#126-range-types)
  - [12.7 Domain Types & Custom Types](#127-domain-types--custom-types)
  - [12.8 Generated Columns](#128-generated-columns)
- [Phase 13 — Extensions & Ecosystem](#phase-13--extensions--ecosystem)
  - [13.1 Extension System](#131-extension-system)
  - [13.2 Popular Extensions](#132-popular-extensions)
  - [13.3 Foreign Data Wrappers (FDW)](#133-foreign-data-wrappers-fdw)
  - [13.4 Logical Decoding & Change Data Capture](#134-logical-decoding--change-data-capture)
- [Phase 14 — PostgreSQL Internals](#phase-14--postgresql-internals)
  - [14.1 Storage Architecture](#141-storage-architecture)
  - [14.2 WAL (Write-Ahead Logging)](#142-wal-write-ahead-logging)
  - [14.3 Buffer Manager & Shared Buffers](#143-buffer-manager--shared-buffers)
  - [14.4 Query Processing Pipeline](#144-query-processing-pipeline)
  - [14.5 Catalog System](#145-catalog-system)
  - [14.6 TOAST (The Oversized-Attribute Storage Technique)](#146-toast-the-oversized-attribute-storage-technique)
- [Phase 15 — PostgreSQL 18 New Features](#phase-15--postgresql-18-new-features)

---

## Phase 1 — Foundations

### 1.1 Installation & Setup

Setting up a working PostgreSQL environment and connecting with client tools.

- **Installing PostgreSQL** — Package managers (apt, brew, dnf), Docker images, or cloud-managed instances (Neon, RDS, Supabase).
- **psql CLI** — The default interactive terminal. Connect, run queries, use meta-commands (`\dt`, `\d+`, `\l`, `\du`).
- **pgAdmin / DBeaver** — GUI tools for visual database management and query building.
- **Creating a Database** — `CREATE DATABASE`, locale/encoding settings, template databases.
- **Configuration Files** — `postgresql.conf` (server settings), `pg_hba.conf` (client authentication), `pg_ident.conf` (user mapping).

> *Ref: [Neon Tutorial — Getting Started](https://neon.com/postgresql/tutorial) · [Docs — Server Setup](https://www.postgresql.org/docs/18/runtime.html)*

---

### 1.2 SQL Fundamentals

Core SQL statements that form the backbone of all database operations.

- **SELECT** — Retrieve data from one or more tables.
- **INSERT** — Add new rows. Single row, multi-row, and `INSERT ... ON CONFLICT` (upsert).
- **UPDATE** — Modify existing rows with `SET` clauses and `WHERE` filters.
- **DELETE** — Remove rows. Use `WHERE` to avoid deleting everything.
- **TRUNCATE** — Fast removal of all rows from a table (no row-by-row scanning).
- **Column Aliases** — Rename output columns with `AS`.
- **ORDER BY / LIMIT / OFFSET** — Control result ordering and pagination.

> *Ref: [Neon Tutorial — SQL Basics](https://neon.com/postgresql/tutorial) · [Docs — SQL Commands](https://www.postgresql.org/docs/18/sql-commands.html)*

---

### 1.3 Data Types

PostgreSQL has one of the richest type systems of any RDBMS.

- **Numeric** — `smallint`, `integer`, `bigint`, `decimal`/`numeric`, `real`, `double precision`, `serial`/`bigserial`.
- **Character** — `char(n)`, `varchar(n)`, `text`. In practice, `text` + CHECK constraint is idiomatic.
- **Boolean** — `true`, `false`, `null`.
- **Date/Time** — `date`, `time`, `timestamp`, `timestamptz`, `interval`. Always prefer `timestamptz` for real-world timestamps.
- **UUID** — `uuid` type with `gen_random_uuid()` (built-in since PG 13).
- **JSON / JSONB** — Semi-structured data. `jsonb` is binary-stored and indexable.
- **Arrays** — Native array support for any built-in type (e.g., `integer[]`, `text[]`).
- **Network Types** — `inet`, `cidr`, `macaddr`.
- **Enum** — User-defined enum types for fixed sets of values.
- **Range Types** — `int4range`, `tsrange`, `daterange`, etc. Support containment and overlap operators.
- **Composite Types** — Struct-like types combining multiple fields.
- **Bytea** — Binary data storage.

> *Ref: [Docs — Data Types](https://www.postgresql.org/docs/18/datatype.html)*

---

### 1.4 Table Basics & Constraints

Creating tables with proper constraints is the foundation of data integrity.

- **CREATE TABLE** — Define columns, types, and inline constraints.
- **PRIMARY KEY** — Uniquely identifies each row. Implicitly creates a unique B-tree index.
- **FOREIGN KEY** — Enforces referential integrity between tables. Supports `CASCADE`, `SET NULL`, `RESTRICT`.
- **UNIQUE** — Ensures no two rows share the same value in the specified columns.
- **NOT NULL** — Prevents null values in a column.
- **CHECK** — Validates column values against a boolean expression.
- **DEFAULT** — Provides a fallback value when none is specified during INSERT.
- **EXCLUDE** — Prevents rows from violating a given exclusion condition (uses GiST index).
- **ALTER TABLE** — Add/drop columns, rename, add constraints, change types.
- **DROP TABLE** — Remove a table and its data. `CASCADE` drops dependent objects.
- **Temporary Tables** — Session-scoped tables that auto-drop on disconnect.

> *Ref: [Docs — DDL](https://www.postgresql.org/docs/18/ddl.html) · [Neon Tutorial — Create Table](https://neon.com/postgresql/tutorial)*

---

## Phase 2 — Querying & Data Manipulation

### 2.1 Filtering & Sorting

Narrowing down result sets efficiently.

- **WHERE** — Filter rows using comparison (`=`, `<>`, `>`, `<`), pattern (`LIKE`, `ILIKE`), range (`BETWEEN`), and NULL checks (`IS NULL`).
- **AND / OR / NOT** — Combine conditions with logical operators.
- **IN / NOT IN** — Match against a set of values or subquery results.
- **EXISTS** — Check for the existence of rows in a correlated subquery.
- **ORDER BY** — Sort results by one or more columns, `ASC` / `DESC`, `NULLS FIRST/LAST`.
- **DISTINCT / DISTINCT ON** — Deduplicate result rows. `DISTINCT ON` is a PostgreSQL-specific extension.

> *Ref: [Docs — Queries](https://www.postgresql.org/docs/18/queries.html)*

---

### 2.2 Joins

Combining data from multiple tables.

- **INNER JOIN** — Returns rows where there is a match on both sides.
- **LEFT JOIN (LEFT OUTER JOIN)** — Returns all rows from the left table, and matched rows from the right (or NULL).
- **RIGHT JOIN** — The reverse of LEFT JOIN.
- **FULL OUTER JOIN** — Returns all rows from both tables, with NULLs where there is no match.
- **CROSS JOIN** — Cartesian product of two tables.
- **SELF JOIN** — Joining a table with itself (e.g., employee-manager hierarchies).
- **LATERAL JOIN** — Allows the subquery on the right side to reference columns from the left. Powerful for correlated lookups.
- **NATURAL JOIN** — Automatically joins on columns with the same name (generally avoided in production).

> *Ref: [Docs — Table Expressions / Joins](https://www.postgresql.org/docs/18/queries-table-expressions.html)*

---

### 2.3 Grouping & Aggregation

Summarizing data across groups of rows.

- **GROUP BY** — Partition rows into groups for aggregate computation.
- **HAVING** — Filter groups after aggregation (unlike WHERE which filters rows before).
- **Aggregate Functions** — `COUNT`, `SUM`, `AVG`, `MIN`, `MAX`, `STRING_AGG`, `ARRAY_AGG`, `JSON_AGG`.
- **GROUPING SETS** — Compute aggregates over multiple grouping combinations in one query.
- **ROLLUP** — Hierarchical subtotals (e.g., year → month → day totals).
- **CUBE** — All possible combinations of grouping columns.
- **FILTER** — Apply conditional aggregation: `COUNT(*) FILTER (WHERE status = 'active')`.

> *Ref: [Docs — Aggregate Functions](https://www.postgresql.org/docs/18/functions-aggregate.html)*

---

### 2.4 Subqueries & CTEs

Structuring complex queries with nested or named result sets.

- **Scalar Subqueries** — A subquery that returns exactly one value, usable anywhere an expression is allowed.
- **Correlated Subqueries** — References outer query columns; re-evaluated per outer row.
- **Common Table Expressions (CTEs)** — `WITH` clauses that name reusable subqueries. Improve readability.
- **Recursive CTEs** — `WITH RECURSIVE` for tree traversal, graph walks, and hierarchical data.
- **CTE Materialization** — PostgreSQL 12+ allows `MATERIALIZED` / `NOT MATERIALIZED` hints to control CTE execution.

> *Ref: [Docs — WITH Queries](https://www.postgresql.org/docs/18/queries-with.html)*

---

### 2.5 Set Operations

Combining results from multiple queries.

- **UNION** — Combine results, removing duplicates.
- **UNION ALL** — Combine results, keeping duplicates (faster — no dedup sort).
- **INTERSECT** — Rows present in both result sets.
- **EXCEPT** — Rows in the first result set but not the second.

> *Ref: [Docs — Combining Queries](https://www.postgresql.org/docs/18/queries-union.html)*

---

### 2.6 Window Functions

Perform calculations across related rows without collapsing them.

- **OVER()** — Defines the window (partition + ordering) for the function.
- **PARTITION BY** — Split rows into independent groups for the window function.
- **Ranking** — `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `NTILE()`.
- **Value** — `LAG()`, `LEAD()`, `FIRST_VALUE()`, `LAST_VALUE()`, `NTH_VALUE()`.
- **Aggregate as Window** — `SUM() OVER(...)`, `AVG() OVER(...)` for running totals, moving averages.
- **Frame Clause** — `ROWS BETWEEN` / `RANGE BETWEEN` to fine-tune the window frame.
- **Named Windows** — `WINDOW w AS (...)` for reuse across multiple columns.

> *Ref: [Docs — Window Functions](https://www.postgresql.org/docs/18/tutorial-window.html)*

---

## Phase 3 — Schema Design & Data Modeling

### 3.1 Normalization

Organizing tables to minimize redundancy and dependency anomalies.

- **1NF** — Atomic values, no repeating groups.
- **2NF** — Remove partial dependencies (every non-key column depends on the whole primary key).
- **3NF** — Remove transitive dependencies.
- **BCNF** — Every determinant is a candidate key.
- **When to Normalize** — OLTP workloads with heavy writes benefit most from normalization.

---

### 3.2 Denormalization

Intentionally adding redundancy for read performance.

- **Materialized Aggregates** — Pre-computed totals stored in dedicated columns.
- **Duplicated Columns** — Copying frequently-joined data to avoid JOINs.
- **JSON Columns** — Storing semi-structured denormalized data in `jsonb`.
- **Trade-offs** — Faster reads at the cost of data integrity complexity and write overhead.

---

### 3.3 Table Inheritance & Partitioning

Splitting large tables for manageability and performance.

- **Table Inheritance** — PostgreSQL's object-relational feature where child tables inherit parent columns. Limited use in modern applications.
- **Declarative Partitioning** — `PARTITION BY RANGE`, `LIST`, or `HASH`. PostgreSQL handles routing automatically.
- **Partition Pruning** — The planner skips irrelevant partitions during query execution.
- **Partition Maintenance** — Adding, detaching, and dropping partitions. `DEFAULT` partition for unmatched values.

> *Ref: [Docs — Partitioning](https://www.postgresql.org/docs/18/ddl-partitioning.html)*

---

### 3.4 Schemas & Namespaces

Logical grouping of database objects.

- **Public Schema** — Default schema where objects are created unless otherwise specified.
- **Creating Schemas** — `CREATE SCHEMA` for multi-tenant isolation, logical grouping, or access control.
- **search_path** — Controls which schemas are searched when an unqualified object name is used.
- **Schema-Based Multi-Tenancy** — Each tenant gets a dedicated schema (common SaaS pattern).

> *Ref: [Docs — Schemas](https://www.postgresql.org/docs/18/ddl-schemas.html)*

---

### 3.5 Sequences & Identity Columns

Auto-incrementing value generators.

- **Sequences** — Standalone objects that produce incrementing values. `CREATE SEQUENCE`, `nextval()`, `currval()`, `setval()`.
- **SERIAL / BIGSERIAL** — Legacy shorthand that auto-creates a sequence and links it to a column.
- **GENERATED ALWAYS AS IDENTITY** — SQL-standard identity columns (preferred over SERIAL in modern PostgreSQL).
- **Sequence Gaps** — Sequences do NOT guarantee gap-free values. Rollbacks and crashes create gaps by design.

> *Ref: [Docs — Sequences](https://www.postgresql.org/docs/18/sql-createsequence.html)*

---

## Phase 4 — Indexes & Query Performance

### 4.1 Index Types

PostgreSQL supports multiple index access methods, each optimized for different workloads.

- **B-tree** — Default index. Supports equality and range queries (`=`, `<`, `>`, `BETWEEN`, `IN`). Ideal for most use cases.
- **Hash** — Optimized for simple equality lookups (`=`). WAL-logged since PG 10. Smaller than B-tree for equality-only workloads.
- **GiST (Generalized Search Tree)** — Supports complex data types: geometric, full-text, range, network types. Lossy index that may require rechecks.
- **SP-GiST (Space-Partitioned GiST)** — For non-balanced tree structures: quad-trees, k-d trees, radix trees. Good for non-uniform data distributions.
- **GIN (Generalized Inverted Index)** — Optimized for composite values with many elements: arrays, JSONB, full-text search (`tsvector`). Fast lookups, slower updates.
- **BRIN (Block Range Index)** — Extremely compact index storing min/max per block range. Best for naturally ordered data (e.g., time-series with append-only inserts).

> *Ref: [Neon — PostgreSQL Indexes](https://neon.com/postgresql/postgresql-indexes) · [Docs — Index Types](https://www.postgresql.org/docs/18/indexes-types.html)*

---

### 4.2 Index Strategies

Advanced indexing techniques to optimize specific query patterns.

- **Composite (Multi-Column) Index** — Index on multiple columns. Column order matters — leftmost prefix rule applies.
- **Covering Index (INCLUDE)** — Add non-key columns to an index to enable index-only scans without visiting the heap.
- **Partial Index** — Index a subset of rows using a `WHERE` clause. Reduces index size and maintenance cost.
- **Expression Index** — Index on an expression or function result (e.g., `CREATE INDEX ON tbl (lower(email))`).
- **Unique Index** — Enforces uniqueness. Automatically created by `UNIQUE` and `PRIMARY KEY` constraints.
- **Concurrent Index Creation** — `CREATE INDEX CONCURRENTLY` avoids locking writes during index build (takes longer but non-blocking).

> *Ref: [Neon — PostgreSQL Indexes](https://neon.com/postgresql/postgresql-indexes) · [Docs — Indexes](https://www.postgresql.org/docs/18/indexes.html)*

---

### 4.3 EXPLAIN & Query Plans

Understanding how PostgreSQL executes your queries.

- **EXPLAIN** — Shows the estimated query plan without executing it.
- **EXPLAIN ANALYZE** — Executes the query and shows actual runtimes and row counts.
- **EXPLAIN (BUFFERS, FORMAT JSON)** — Adds buffer hit/miss info and outputs in JSON for tooling.
- **Scan Types** — Seq Scan, Index Scan, Index Only Scan, Bitmap Scan, TID Scan.
- **Join Algorithms** — Nested Loop, Hash Join, Merge Join. The planner chooses based on cost estimates.
- **Cost Model** — `startup_cost..total_cost` representing estimated I/O and CPU units.

> *Ref: [Docs — EXPLAIN](https://www.postgresql.org/docs/18/sql-explain.html)*

---

### 4.4 Query Optimization Techniques

Practical techniques to make queries faster.

- **Index-Only Scans** — When all needed columns are in the index. Requires recent VACUUM for visibility map.
- **Avoiding Sequential Scans** — Ensure appropriate indexes exist for WHERE, JOIN, and ORDER BY clauses.
- **Pagination** — Use keyset pagination (`WHERE id > $last_id ORDER BY id LIMIT N`) instead of `OFFSET` for large datasets.
- **Batch Operations** — Use `unnest()` + JOIN or `VALUES` lists for bulk lookups instead of many single queries.
- **Prepared Statements** — Reuse query plans for repeated similar queries. Generic vs custom plans.

---

### 4.5 Statistics & the Planner

How PostgreSQL decides which plan to use.

- **pg_statistic / pg_stats** — Table-level statistics (most common values, histograms, null fractions, distinct counts).
- **ANALYZE** — Collects fresh statistics. Run after bulk loads or significant data changes.
- **default_statistics_target** — Controls the granularity of statistics histograms (default 100, increase for skewed data).
- **Extended Statistics** — `CREATE STATISTICS` for multi-column correlations and functional dependencies (PG 10+).
- **Planner Cost Parameters** — `seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, etc. Tune to match your hardware.

> *Ref: [Docs — Planner Statistics](https://www.postgresql.org/docs/18/planner-stats.html)*

---

## Phase 5 — Transactions & Concurrency

### 5.1 ACID Properties

The fundamental guarantees of a relational transaction.

- **Atomicity** — All operations in a transaction succeed or all are rolled back.
- **Consistency** — Transactions bring the database from one valid state to another.
- **Isolation** — Concurrent transactions don't interfere with each other.
- **Durability** — Once committed, data survives crashes (via WAL).

---

### 5.2 Transaction Control

Managing transaction boundaries.

- **BEGIN / COMMIT / ROLLBACK** — Explicit transaction control.
- **SAVEPOINT** — Create rollback points within a transaction for partial recovery.
- **Autocommit** — Default mode in psql; each statement is its own transaction unless wrapped in `BEGIN`.
- **Two-Phase Commit** — `PREPARE TRANSACTION` / `COMMIT PREPARED` for distributed systems.

> *Ref: [Docs — Transaction Management](https://www.postgresql.org/docs/18/tutorial-transactions.html)*

---

### 5.3 Isolation Levels

Control how transactions see each other's changes.

- **Read Committed** — Default in PostgreSQL. Each statement sees the latest committed data at statement start.
- **Repeatable Read** — Transaction sees a consistent snapshot from its start. Detects serialization conflicts.
- **Serializable** — Strongest level. Uses SSI (Serializable Snapshot Isolation) to detect all anomalies. May abort transactions with serialization failures.
- **Read Uncommitted** — PostgreSQL treats this as Read Committed (dirty reads are never allowed).
- **Anomalies** — Dirty reads, non-repeatable reads, phantom reads, serialization anomalies.

> *Ref: [Docs — Transaction Isolation](https://www.postgresql.org/docs/18/transaction-iso.html)*

---

### 5.4 MVCC (Multi-Version Concurrency Control)

PostgreSQL's core concurrency model — readers never block writers.

- **Tuple Versioning** — Each row version has `xmin` (creating transaction) and `xmax` (deleting/updating transaction).
- **Snapshot Isolation** — Transactions see a frozen-in-time view of the database based on active transactions at snapshot time.
- **Visibility Rules** — A row is visible if `xmin` is committed and `xmax` is not committed (from the viewer's perspective).
- **Dead Tuples** — Old row versions remain on disk until VACUUM reclaims them.
- **Transaction IDs (XIDs)** — 32-bit counter. Wraparound prevention requires periodic aggressive vacuuming.

> *Ref: [Docs — MVCC](https://www.postgresql.org/docs/18/mvcc.html)*

---

### 5.5 Locking Mechanisms

Ensuring safe concurrent access to shared resources.

- **Row-Level Locks** — `FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`. Acquired implicitly on writes.
- **Table-Level Locks** — Eight lock modes from `ACCESS SHARE` (read) to `ACCESS EXCLUSIVE` (DDL). See lock compatibility matrix.
- **Advisory Locks** — Application-defined locks managed by PostgreSQL. `pg_advisory_lock()`, `pg_try_advisory_lock()`.
- **pg_locks** — System view to inspect all current locks and detect contention.

> *Ref: [Docs — Explicit Locking](https://www.postgresql.org/docs/18/explicit-locking.html)*

---

### 5.6 Deadlocks

When two transactions block each other indefinitely.

- **Detection** — PostgreSQL automatically detects deadlocks and aborts one transaction.
- **deadlock_timeout** — How long to wait before checking for deadlocks (default 1s).
- **Prevention** — Lock resources in a consistent order across all code paths. Keep transactions short.

---

## Phase 6 — Stored Procedures, Functions & Triggers

### 6.1 PL/pgSQL Language

PostgreSQL's built-in procedural language for writing server-side logic.

- **Variables & Assignment** — `DECLARE`, `:=`, `INTO`.
- **Control Flow** — `IF/ELSIF/ELSE`, `CASE`, `LOOP`, `WHILE`, `FOR`, `FOREACH`.
- **Exception Handling** — `BEGIN ... EXCEPTION WHEN ... THEN ...` blocks.
- **Dynamic SQL** — `EXECUTE` for building queries at runtime (use `format()` or `quote_ident()`/`quote_literal()` to prevent injection).
- **RAISE** — Debug logging and error signaling: `RAISE NOTICE`, `RAISE EXCEPTION`.

> *Ref: [Docs — PL/pgSQL](https://www.postgresql.org/docs/18/plpgsql.html)*

---

### 6.2 User-Defined Functions

Reusable SQL or procedural logic that returns a value.

- **SQL Functions** — Pure SQL body, inlined by the planner for optimization.
- **PL/pgSQL Functions** — Procedural logic with conditionals, loops, and exception handling.
- **RETURNS TABLE** — Functions that return a set of rows (table-valued functions).
- **RETURNS SETOF** — Stream multiple rows back to the caller.
- **IMMUTABLE / STABLE / VOLATILE** — Volatility categories that affect planner optimization and index eligibility.
- **SECURITY DEFINER vs INVOKER** — Execute with the function owner's privileges or the caller's.

> *Ref: [Docs — CREATE FUNCTION](https://www.postgresql.org/docs/18/sql-createfunction.html)*

---

### 6.3 Stored Procedures

Transaction-aware server-side routines (introduced in PG 11).

- **CREATE PROCEDURE** — Unlike functions, procedures can manage transaction control (`COMMIT` / `ROLLBACK` inside).
- **CALL** — Invoke a stored procedure.
- **Use Cases** — Batch processing, ETL pipelines, complex multi-step operations that need intermediate commits.

> *Ref: [Docs — CREATE PROCEDURE](https://www.postgresql.org/docs/18/sql-createprocedure.html)*

---

### 6.4 Triggers & Event Triggers

Automatic actions in response to data changes or DDL events.

- **Row-Level Triggers** — Fire once per affected row (`FOR EACH ROW`).
- **Statement-Level Triggers** — Fire once per statement (`FOR EACH STATEMENT`).
- **BEFORE / AFTER / INSTEAD OF** — Timing of trigger execution relative to the operation.
- **Transition Tables** — Access `OLD TABLE` and `NEW TABLE` in AFTER triggers for bulk processing.
- **Event Triggers** — Fire on DDL commands (`CREATE`, `ALTER`, `DROP`) for schema auditing and governance.
- **Trigger Functions** — Must return `TRIGGER` type and use `NEW` / `OLD` record references.

> *Ref: [Docs — Triggers](https://www.postgresql.org/docs/18/trigger-definition.html)*

---

### 6.5 Rules System

PostgreSQL's query rewriting system (less common than triggers).

- **CREATE RULE** — Rewrite queries before execution (e.g., redirect INSERTs to a different table).
- **Rules vs Triggers** — Rules rewrite at the query level; triggers fire at the row level. Triggers are generally more predictable.
- **Updatable Views** — Rules can make views writable (though `INSTEAD OF` triggers are more common now).

> *Ref: [Docs — Rules](https://www.postgresql.org/docs/18/rules.html)*

---

## Phase 7 — Views & Materialized Views

### 7.1 Views

Named queries stored as virtual tables.

- **CREATE VIEW** — Define a reusable query. No data stored — executes underlying query on each access.
- **Benefits** — Simplify complex queries, enforce access control, decouple application from schema changes.
- **Security Barrier Views** — Prevent information leakage through user-defined functions in WHERE clauses.

> *Ref: [Docs — Views](https://www.postgresql.org/docs/18/sql-createview.html)*

---

### 7.2 Materialized Views

Cached query results stored as physical tables.

- **CREATE MATERIALIZED VIEW** — Executes the query and stores the result set.
- **REFRESH MATERIALIZED VIEW** — Re-execute the query and replace stored data.
- **CONCURRENTLY** — Refresh without locking reads (requires a unique index).
- **Use Cases** — Dashboard aggregations, expensive report queries, search results.
- **Limitations** — Not auto-refreshed; your application must manage refresh schedules.

> *Ref: [Docs — Materialized Views](https://www.postgresql.org/docs/18/sql-creatematerializedview.html)*

---

### 7.3 Updatable Views

Views that support INSERT, UPDATE, DELETE.

- **Simple Updatable Views** — Auto-updatable if the view is based on a single table with no aggregation, DISTINCT, GROUP BY, etc.
- **WITH CHECK OPTION** — Ensures rows inserted/updated through the view still satisfy the view's WHERE clause.
- **INSTEAD OF Triggers** — For complex views, write triggers to handle DML manually.

---

## Phase 8 — Administration & Security

### 8.1 Roles & Users

PostgreSQL unifies users and groups into a single "role" concept.

- **CREATE ROLE / CREATE USER** — `CREATE USER` is just `CREATE ROLE` with `LOGIN` privilege.
- **Role Attributes** — `SUPERUSER`, `CREATEDB`, `CREATEROLE`, `LOGIN`, `REPLICATION`, `BYPASSRLS`.
- **Role Membership** — `GRANT role TO another_role` for group-based permission management.
- **Predefined Roles** — `pg_read_all_data`, `pg_write_all_data`, `pg_monitor`, etc. (PG 14+).

> *Ref: [Neon — PostgreSQL Administration](https://neon.com/postgresql/postgresql-administration) · [Docs — Roles](https://www.postgresql.org/docs/18/user-manag.html)*

---

### 8.2 Privileges & GRANT/REVOKE

Fine-grained access control on database objects.

- **GRANT** — Give permissions (`SELECT`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `EXECUTE`, `USAGE`, etc.).
- **REVOKE** — Remove permissions.
- **Default Privileges** — `ALTER DEFAULT PRIVILEGES` to set permissions for future objects automatically.
- **Column-Level Privileges** — Grant `SELECT` or `UPDATE` on specific columns.
- **Schema Privileges** — `USAGE` (access objects) and `CREATE` (create objects) on schemas.

> *Ref: [Docs — Privileges](https://www.postgresql.org/docs/18/ddl-priv.html)*

---

### 8.3 Row-Level Security (RLS)

Restrict which rows a user can see or modify.

- **ENABLE ROW LEVEL SECURITY** — Activate RLS on a table (no access by default once enabled).
- **CREATE POLICY** — Define rules using `USING` (read filter) and `WITH CHECK` (write filter) clauses.
- **Policy Types** — `PERMISSIVE` (OR-combined) vs `RESTRICTIVE` (AND-combined).
- **current_user / current_setting()** — Common functions used in policy expressions for tenant isolation.
- **FORCE ROW LEVEL SECURITY** — Apply RLS even to table owners.

> *Ref: [Docs — Row Security Policies](https://www.postgresql.org/docs/18/ddl-rowsecurity.html)*

---

### 8.4 Authentication (pg_hba.conf)

Controls who can connect and how they authenticate.

- **Connection Types** — `local` (Unix socket), `host` (TCP/IP), `hostssl`, `hostnossl`.
- **Auth Methods** — `trust`, `md5`, `scram-sha-256` (preferred), `cert`, `ldap`, `gss`, `peer`, `ident`.
- **Match Rules** — Evaluated top to bottom; first match wins.
- **Best Practice** — Use `scram-sha-256` for password auth. Never use `trust` in production.

> *Ref: [Docs — Client Authentication](https://www.postgresql.org/docs/18/client-authentication.html)*

---

### 8.5 SSL/TLS Encryption

Encrypting connections between client and server.

- **Server Certificates** — Configure `ssl_cert_file`, `ssl_key_file` in `postgresql.conf`.
- **Client Certificates** — Mutual TLS for strong identity verification.
- **sslmode** — Client connection parameter: `disable`, `allow`, `prefer`, `require`, `verify-ca`, `verify-full`.
- **Best Practice** — Use `verify-full` in production for maximum security.

> *Ref: [Docs — SSL Support](https://www.postgresql.org/docs/18/ssl-tcp.html)*

---

### 8.6 Auditing

Tracking who did what and when.

- **log_statement** — Log all SQL statements or categories (`none`, `ddl`, `mod`, `all`).
- **pgAudit Extension** — Session and object-level auditing with detailed logging.
- **pg_stat_statements** — Track query execution statistics (not auditing per se, but useful for forensics).
- **Event Triggers** — Custom DDL auditing by writing trigger functions that log schema changes.

---

## Phase 9 — Backup, Restore & Disaster Recovery

### 9.1 Logical Backup (pg_dump / pg_restore)

Exporting database objects and data as SQL or custom format.

- **pg_dump** — Dump a single database. Supports plain SQL (`-Fp`), custom (`-Fc`), directory (`-Fd`), tar (`-Ft`) formats.
- **pg_dumpall** — Dump all databases including global objects (roles, tablespaces).
- **pg_restore** — Restore from custom/directory/tar format dumps. Supports parallel restore (`-j`).
- **Selective Restore** — Restore specific tables, schemas, or data-only / schema-only.

> *Ref: [Neon — PostgreSQL Administration](https://neon.com/postgresql/postgresql-administration) · [Docs — Backup](https://www.postgresql.org/docs/18/backup.html)*

---

### 9.2 Physical Backup & PITR

File-system level backup for large databases.

- **pg_basebackup** — Takes a binary copy of the entire data directory. Foundation for replicas and PITR.
- **Point-in-Time Recovery (PITR)** — Restore to any moment in time by replaying WAL segments up to a target timestamp/transaction ID.
- **Recovery Target** — `recovery_target_time`, `recovery_target_lsn`, `recovery_target_xid`.

> *Ref: [Docs — Continuous Archiving and PITR](https://www.postgresql.org/docs/18/continuous-archiving.html)*

---

### 9.3 WAL Archiving

Continuously archiving WAL segments for disaster recovery.

- **archive_mode / archive_command** — Enable archiving and define how WAL files are copied to safe storage.
- **archive_library** — Module-based archiving (PG 15+).
- **WAL Retention** — `wal_keep_size` and replication slots ensure WAL segments are retained for replicas.

---

### 9.4 Continuous Archiving & Recovery

End-to-end disaster recovery workflow.

- **Base Backup + WAL Replay** — Combine `pg_basebackup` with archived WAL segments for full recovery capability.
- **Standby Mode** — Configure a standby server to continuously replay WAL for warm/hot standby.
- **recovery.signal / standby.signal** — Signal files that tell PostgreSQL to enter recovery or standby mode on startup.

---

## Phase 10 — Replication & High Availability

### 10.1 Streaming Replication

Real-time, byte-level replication of WAL data.

- **Primary / Standby** — The primary streams WAL to one or more standbys. Standbys continuously apply changes.
- **Synchronous Replication** — Primary waits for standby confirmation before committing. Guarantees zero data loss.
- **Asynchronous Replication** — Primary doesn't wait. Higher performance, slight risk of data loss on failover.
- **Hot Standby** — Standby server accepts read-only queries while replaying WAL.
- **Replication Slots** — Ensure the primary retains WAL segments until the standby has consumed them.

> *Ref: [Docs — Streaming Replication](https://www.postgresql.org/docs/18/warm-standby.html)*

---

### 10.2 Logical Replication

Table-level, selective replication using the publish/subscribe model.

- **Publication** — Define which tables to replicate from the publisher.
- **Subscription** — A subscriber connects and receives changes for the specified publication.
- **Use Cases** — Cross-version upgrades, selective replication, data distribution, multi-datacenter setups.
- **Limitations** — DDL is not replicated. Sequences, large objects have limited support.

> *Ref: [Docs — Logical Replication](https://www.postgresql.org/docs/18/logical-replication.html)*

---

### 10.3 Failover & Switchover

Promoting a standby to primary when things go wrong.

- **pg_promote()** — Promote a standby to primary role.
- **Automatic Failover** — Use Patroni, pg_auto_failover, or cloud-managed solutions.
- **Switchover** — Planned role swap between primary and standby for maintenance.
- **Split Brain Prevention** — Fencing and quorum mechanisms to prevent two primaries.

---

### 10.4 Connection Pooling (PgBouncer)

Managing database connections efficiently.

- **PgBouncer** — Lightweight connection pooler that sits between application and PostgreSQL.
- **Pool Modes** — Session pooling, transaction pooling (most common), statement pooling.
- **Benefits** — Reduces connection overhead, enables thousands of application connections with few server connections.

---

### 10.5 Load Balancing

Distributing read queries across replicas.

- **Application-Level Routing** — Direct writes to primary, reads to standbys.
- **HAProxy / PgPool-II** — Middleware for transparent query routing and load balancing.
- **Connection String Targets** — `target_session_attrs=read-write` / `read-only` for automatic routing.

---

## Phase 11 — Performance Tuning & Monitoring

### 11.1 Configuration Tuning (postgresql.conf)

Key parameters that dramatically affect performance.

- **shared_buffers** — PostgreSQL's main cache. Typically 25% of total RAM.
- **effective_cache_size** — Hint to the planner about total OS + PG cache. Typically 50-75% of RAM.
- **work_mem** — Memory per sort/hash operation. Set carefully — multiplied by concurrent operations.
- **maintenance_work_mem** — Memory for VACUUM, CREATE INDEX, ALTER TABLE. Can be larger than `work_mem`.
- **wal_buffers** — WAL write buffer. Auto-sized based on `shared_buffers` (usually fine).
- **max_connections** — Limit concurrent connections. Lower is better; use a connection pooler.
- **random_page_cost** — Lower for SSDs (1.1–1.5) vs HDDs (4.0). Affects index vs seq scan decisions.
- **effective_io_concurrency** — Set higher for SSDs to enable more parallel I/O.

> *Ref: [Docs — Server Configuration](https://www.postgresql.org/docs/18/runtime-config.html)*

---

### 11.2 Memory Architecture

Understanding PostgreSQL's memory model.

- **Shared Buffers** — Main buffer pool shared across all connections.
- **WAL Buffers** — Write-ahead log staging area.
- **Per-Backend Memory** — Each connection gets private `work_mem`, `temp_buffers`, `maintenance_work_mem`.
- **OS Page Cache** — PostgreSQL relies on the OS filesystem cache as a second layer.
- **Huge Pages** — Reduce TLB misses for large `shared_buffers` allocations.

---

### 11.3 VACUUM & Autovacuum

Reclaiming space from dead tuples and updating statistics.

- **VACUUM** — Marks dead tuple space as reusable. Does not return space to the OS.
- **VACUUM FULL** — Rewrites the entire table compactly. Acquires `ACCESS EXCLUSIVE` lock — avoid on busy tables.
- **Autovacuum** — Background daemon that automatically runs VACUUM and ANALYZE on tables.
- **Tuning Autovacuum** — `autovacuum_vacuum_threshold`, `autovacuum_vacuum_scale_factor`, `autovacuum_naptime`.
- **Freeze** — Prevent transaction ID wraparound. `vacuum_freeze_min_age`, `autovacuum_freeze_max_age`.
- **pg_stat_user_tables** — Monitor `n_dead_tup`, `last_autovacuum`, `last_autoanalyze`.

> *Ref: [Docs — Routine Vacuuming](https://www.postgresql.org/docs/18/routine-vacuuming.html)*

---

### 11.4 Monitoring & Diagnostics

Observing database health and detecting issues.

- **pg_stat_activity** — See all current connections, their queries, states, and wait events.
- **pg_stat_statements** — Extension that tracks execution statistics for all normalized queries.
- **pg_stat_bgwriter** — Buffer allocation and checkpoint statistics.
- **pg_stat_io** — Detailed I/O statistics by backend type (PG 16+).
- **Log Analysis** — `log_min_duration_statement` to log slow queries. `auto_explain` extension for automatic plan logging.
- **External Tools** — pgMonitor, Prometheus + postgres_exporter, Datadog, pgBadger for log analysis.

> *Ref: [Docs — Monitoring Stats](https://www.postgresql.org/docs/18/monitoring-stats.html)*

---

### 11.5 pg_stat Catalog Views

Key system views for operational insight.

- **pg_stat_user_tables / pg_stat_user_indexes** — Row counts, scan counts, tuple operations, index usage.
- **pg_stat_replication** — Streaming replication status: lag, state, write/flush/replay LSN.
- **pg_stat_wal** — WAL generation statistics.
- **pg_stat_progress_vacuum / _create_index / _copy** — Real-time progress tracking for long operations.
- **pg_locks** — All current locks with granted/waiting status.

---

### 11.6 Bloat Management

Dealing with wasted space from MVCC overhead.

- **Table Bloat** — Dead tuples inflate table size. Monitor with `pgstattuple` or bloat estimation queries.
- **Index Bloat** — Indexes grow with dead entries. `REINDEX CONCURRENTLY` to rebuild without downtime.
- **pg_repack** — Extension for online table reorganization without `ACCESS EXCLUSIVE` locks.
- **Prevention** — Properly tuned autovacuum is the best defense against bloat.

---

## Phase 12 — Advanced Data Features

### 12.1 JSON / JSONB

Semi-structured data storage and querying.

- **JSON vs JSONB** — `json` stores raw text; `jsonb` stores parsed binary (faster queries, supports indexing).
- **Operators** — `->` (get by key), `->>` (get as text), `#>` (path), `@>` (containment), `?` (key exists).
- **Indexing JSONB** — GIN indexes on `jsonb` columns for fast containment and existence queries.
- **Functions** — `jsonb_build_object()`, `jsonb_agg()`, `jsonb_each()`, `jsonb_to_record()`, `jsonb_set()`, `jsonb_path_query()` (SQL/JSON path).
- **SQL/JSON Standard** — `JSON_TABLE`, `JSON_QUERY`, `JSON_VALUE`, `JSON_EXISTS` (PG 17+).

> *Ref: [Docs — JSON Types](https://www.postgresql.org/docs/18/datatype-json.html)*

---

### 12.2 Arrays

First-class array support for any data type.

- **Declaration** — `integer[]`, `text[][]` (multi-dimensional).
- **Operators** — `@>` (contains), `<@` (contained by), `&&` (overlap), `||` (concatenation).
- **Functions** — `array_agg()`, `unnest()`, `array_length()`, `array_position()`, `array_remove()`.
- **Indexing** — GIN indexes for element containment and overlap queries.

> *Ref: [Docs — Arrays](https://www.postgresql.org/docs/18/arrays.html)*

---

### 12.3 Hstore

Key-value store in a single column (extension).

- **Usage** — `CREATE EXTENSION hstore`. Store flat key-value pairs as `'key1=>val1, key2=>val2'::hstore`.
- **Operators** — `->` (get value), `?` (key exists), `||` (merge), `-` (delete key).
- **When to Use** — Simple tag/attribute storage. For nested data, prefer `jsonb`.

> *Ref: [Docs — hstore](https://www.postgresql.org/docs/18/hstore.html)*

---

### 12.4 Full-Text Search

Built-in document search without external engines.

- **tsvector** — Parsed and normalized document representation.
- **tsquery** — Search query with boolean operators (`&`, `|`, `!`, `<->`).
- **to_tsvector() / to_tsquery()** — Convert text to searchable formats.
- **GIN Index** — Index `tsvector` columns for fast full-text lookups.
- **Ranking** — `ts_rank()`, `ts_rank_cd()` for relevance scoring.
- **Dictionaries & Configurations** — Language-specific stemming, stop words, and synonym support.
- **Phrase Search** — `<->` (adjacent), `<N>` (within N words).

> *Ref: [Docs — Full Text Search](https://www.postgresql.org/docs/18/textsearch.html)*

---

### 12.5 PostGIS (Geospatial)

Industry-standard geospatial extension.

- **Geometry vs Geography** — Planar vs spherical coordinate systems.
- **Spatial Functions** — `ST_Distance`, `ST_Contains`, `ST_Intersects`, `ST_Buffer`, `ST_Within`.
- **Spatial Indexing** — GiST indexes on geometry columns for fast spatial queries.
- **Use Cases** — Location-based services, mapping, geofencing, route planning.

> *Ref: [PostGIS Documentation](https://postgis.net/documentation/)*

---

### 12.6 Range Types

Represent a range of values in a single column.

- **Built-in Ranges** — `int4range`, `int8range`, `numrange`, `tsrange`, `tstzrange`, `daterange`.
- **Operators** — `@>` (contains element), `<@` (contained by), `&&` (overlap), `-|-` (adjacent).
- **Exclusion Constraints** — `EXCLUDE USING gist (room WITH =, during WITH &&)` — no overlapping bookings.
- **Use Cases** — Scheduling, versioning, temporal data, IP range management.

> *Ref: [Docs — Range Types](https://www.postgresql.org/docs/18/rangetypes.html)*

---

### 12.7 Domain Types & Custom Types

Creating application-specific types.

- **Domain Types** — Named types built on base types with constraints: `CREATE DOMAIN email AS text CHECK (VALUE ~ '^.+@.+$')`.
- **Composite Types** — Struct-like types: `CREATE TYPE address AS (street text, city text, zip text)`.
- **Enum Types** — `CREATE TYPE mood AS ENUM ('happy', 'sad', 'neutral')`. Ordered, compact storage.

> *Ref: [Docs — CREATE DOMAIN](https://www.postgresql.org/docs/18/sql-createdomain.html)*

---

### 12.8 Generated Columns

Columns computed automatically from other columns.

- **STORED** — Value computed and physically stored on disk during INSERT/UPDATE.
- **Syntax** — `col_name type GENERATED ALWAYS AS (expression) STORED`.
- **Use Cases** — Derived attributes, computed search columns, denormalized aggregates.
- **Limitations** — Cannot reference other generated columns. Expression must be immutable.

> *Ref: [Docs — Generated Columns](https://www.postgresql.org/docs/18/ddl-generated-columns.html)*

---

## Phase 13 — Extensions & Ecosystem

### 13.1 Extension System

PostgreSQL's modular architecture for extending functionality.

- **CREATE EXTENSION** — Install an extension into the current database.
- **EXTENSION Contents** — Can include types, functions, operators, indexes, tables, and configuration.
- **pg_available_extensions** — List all extensions available to install.
- **Versioning** — `ALTER EXTENSION ... UPDATE TO 'version'` for seamless upgrades.

> *Ref: [Docs — Extensions](https://www.postgresql.org/docs/18/extend-extensions.html)*

---

### 13.2 Popular Extensions

Essential extensions every PostgreSQL user should know.

| Extension | Purpose |
|-----------|---------|
| **pg_stat_statements** | Query performance statistics |
| **pgcrypto** | Cryptographic functions (hashing, encryption) |
| **uuid-ossp** | UUID generation functions (legacy; prefer `gen_random_uuid()`) |
| **postgis** | Geospatial data types and functions |
| **pg_trgm** | Trigram-based similarity search and fuzzy matching |
| **hstore** | Key-value storage in a column |
| **pgAudit** | Session and object-level auditing |
| **pg_repack** | Online table reorganization without locks |
| **auto_explain** | Automatic query plan logging for slow queries |
| **pg_cron** | Scheduled jobs inside PostgreSQL |
| **pg_partman** | Automated partition management |
| **timescaledb** | Time-series data optimization |
| **pgvector** | Vector similarity search for AI/ML embeddings |
| **citext** | Case-insensitive text type |
| **btree_gist / btree_gin** | B-tree operators for GiST/GIN indexes (enables exclusion constraints) |

---

### 13.3 Foreign Data Wrappers (FDW)

Query external data sources as if they were local tables.

- **postgres_fdw** — Connect to other PostgreSQL servers. Supports pushdown of WHERE, JOIN, aggregate.
- **file_fdw** — Read CSV or other flat files as foreign tables.
- **Other FDWs** — MySQL, Oracle, MongoDB, Redis, Elasticsearch, S3/Parquet wrappers available.
- **Architecture** — `CREATE SERVER` → `CREATE USER MAPPING` → `CREATE FOREIGN TABLE` or `IMPORT FOREIGN SCHEMA`.

> *Ref: [Docs — Foreign Data Wrappers](https://www.postgresql.org/docs/18/ddl-foreign-data.html)*

---

### 13.4 Logical Decoding & Change Data Capture

Stream database changes to external systems.

- **Logical Decoding** — Extract row-level changes from WAL in a structured format.
- **Output Plugins** — `pgoutput` (built-in), `wal2json`, `test_decoding`.
- **Replication Slots** — Track consumer position in the WAL stream.
- **CDC Pipelines** — Debezium, Kafka Connect with PostgreSQL connector, or custom consumers.
- **Use Cases** — Event sourcing, cache invalidation, data synchronization, microservice integration.

> *Ref: [Docs — Logical Decoding](https://www.postgresql.org/docs/18/logicaldecoding.html)*

---

## Phase 14 — PostgreSQL Internals

### 14.1 Storage Architecture

How data lives on disk.

- **Data Directory ($PGDATA)** — Contains all database files: `base/`, `global/`, `pg_wal/`, `pg_xact/`.
- **Pages** — Fixed 8KB blocks. Every table, index, and WAL segment is organized into pages.
- **Heap Files** — Tables are stored as "heaps" — unordered collections of pages.
- **Item Pointers (ctid)** — `(page_number, offset)` physical location of a tuple.
- **Free Space Map (FSM)** — Tracks available space per page for efficient inserts.
- **Visibility Map (VM)** — Tracks which pages contain only all-visible tuples (enables index-only scans).
- **Filenode Mapping** — `pg_class.relfilenode` maps tables to physical files.

> *Ref: [Docs — Database File Layout](https://www.postgresql.org/docs/18/storage-file-layout.html)*

---

### 14.2 WAL (Write-Ahead Logging)

Ensuring durability and enabling replication.

- **WAL Principle** — Every change is written to WAL before modifying data pages. Guarantees durability on crash.
- **WAL Segments** — 16MB files in `pg_wal/`. Named by LSN (Log Sequence Number).
- **Checkpoints** — Periodic flushing of dirty buffers to disk. After a checkpoint, WAL before that point can be recycled.
- **wal_level** — `minimal` (standalone), `replica` (replication), `logical` (logical decoding).
- **Full Page Writes** — After a checkpoint, the first modification to a page writes the full page image to WAL (prevents torn page reads).
- **LSN (Log Sequence Number)** — Monotonically increasing pointer into the WAL stream.

> *Ref: [Docs — WAL](https://www.postgresql.org/docs/18/wal.html)*

---

### 14.3 Buffer Manager & Shared Buffers

PostgreSQL's in-memory cache layer.

- **Shared Buffer Pool** — Fixed-size cache in shared memory (configured by `shared_buffers`).
- **Buffer Tags** — Each buffer holds one 8KB page, identified by `(tablespace, database, relation, fork, block)`.
- **Clock Sweep** — Eviction algorithm used to replace least-recently-used pages.
- **Pin / Unpin** — Pin count prevents eviction while a backend is using a page.
- **Dirty Buffers** — Modified pages not yet flushed to disk. Written out by the background writer or checkpointer.

---

### 14.4 Query Processing Pipeline

The journey of a SQL query from text to results.

1. **Parser** — Tokenize SQL and build a parse tree. Catches syntax errors.
2. **Analyzer** — Resolve names, types, and permissions. Produces a query tree.
3. **Rewriter** — Apply rules (e.g., view expansion).
4. **Planner/Optimizer** — Generate possible execution plans and choose the cheapest based on cost estimation.
5. **Executor** — Run the chosen plan, reading/writing data from/to the buffer pool.

> *Ref: [Docs — Query Processing](https://www.postgresql.org/docs/18/overview.html)*

---

### 14.5 Catalog System

PostgreSQL's metadata store.

- **System Catalogs** — `pg_class` (tables/indexes), `pg_attribute` (columns), `pg_type` (types), `pg_index`, `pg_namespace` (schemas), `pg_proc` (functions).
- **Information Schema** — SQL-standard views over the catalog. More portable, less detailed.
- **pg_catalog Schema** — Contains all system catalog tables. Always in `search_path`.
- **Self-Describing** — The catalog describes itself — catalog tables are just regular tables.

> *Ref: [Docs — System Catalogs](https://www.postgresql.org/docs/18/catalogs.html)*

---

### 14.6 TOAST (The Oversized-Attribute Storage Technique)

Handling values larger than a single page.

- **Automatic** — When a row exceeds ~2KB, PostgreSQL compresses and/or moves large attributes to a TOAST table.
- **Strategies** — `PLAIN` (no TOAST), `EXTENDED` (compress then out-of-line), `EXTERNAL` (out-of-line, no compression), `MAIN` (compress in-line first).
- **TOAST Pointers** — The main table row stores a pointer to the TOAST table entry.
- **Columnar Access** — Only the accessed column's TOAST data is fetched, not the entire row.

> *Ref: [Docs — TOAST](https://www.postgresql.org/docs/18/storage-toast.html)*

---

## Phase 15 — PostgreSQL 18 New Features

Key improvements in PostgreSQL 18 (released 2025).

- **Virtual Generated Columns** — Computed on read, not stored. Saves disk space compared to `STORED` generated columns.
- **UUIDv7 Support** — `uuidv7()` generates time-ordered UUIDs — better index locality than random UUIDv4.
- **MAINTAIN Privilege** — New privilege for VACUUM, ANALYZE, REINDEX, CLUSTER without full ownership.
- **Asynchronous I/O (AIO)** — Internal async I/O framework enabling more efficient disk operations (foundational work).
- **OAuth/OIDC Authentication** — Native OAuth 2.0 device authorization flow for passwordless client authentication.
- **pg_basebackup Incremental** — Incremental physical backup for faster backup/restore cycles.
- **JSON_TABLE Improvements** — Enhanced SQL/JSON support for transforming JSON into relational rows.
- **NOT NULL Inheritance** — `NOT NULL` constraints can now be properly inherited and enforced across partition hierarchies.
- **Event Trigger Enhancements** — `login` event trigger fires on user login for session-level auditing/setup.
- **Logical Replication Improvements** — Failover slots, conflict detection, apply worker parallelism.
- **COPY Performance** — Faster multi-row COPY operations with reduced overhead.
- **Planner Improvements** — Better join ordering, improved statistics usage, faster query planning.

> *Ref: [Neon — PostgreSQL 18 New Features](https://neon.com/postgresql/postgresql-18-new-features) · [PG 18 Release Notes](https://www.postgresql.org/docs/18/release-18.html)*

---

## Quick Reference: Learning Path Summary

```
Phase 1  Foundations               ← Start here. Install, connect, write basic SQL.
Phase 2  Querying                  ← Master joins, aggregations, window functions.
Phase 3  Schema Design             ← Model data properly. Normalize, then denormalize with intent.
Phase 4  Indexes & Performance     ← Understand EXPLAIN. Build the right indexes.
Phase 5  Transactions              ← ACID, MVCC, isolation levels, locking.
Phase 6  Functions & Triggers      ← Server-side logic with PL/pgSQL.
Phase 7  Views                     ← Abstraction layer over raw tables.
Phase 8  Administration            ← Roles, RLS, authentication, security hardening.
Phase 9  Backup & Recovery         ← pg_dump, WAL archiving, PITR.
Phase 10 Replication & HA          ← Streaming/logical replication, failover.
Phase 11 Performance Tuning        ← Configuration, VACUUM, monitoring.
Phase 12 Advanced Data Features    ← JSONB, full-text search, PostGIS, arrays, ranges.
Phase 13 Extensions & Ecosystem    ← FDW, pgvector, pg_cron, CDC.
Phase 14 Internals                 ← Storage, WAL, buffer manager, query pipeline.
Phase 15 PG 18 New Features        ← UUIDv7, virtual generated columns, AIO, OAuth.
```

---

*Last updated: March 2026*
