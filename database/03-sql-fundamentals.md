# SQL Fundamentals

> **Level**: Beginner | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 1.2  
> The core SQL statements that form the backbone of all PostgreSQL operations — creating, reading, updating, and deleting data.

---

## Table of Contents

1. [SQL Statement Categories](#1-sql-statement-categories)
2. [SELECT — Reading Data](#2-select--reading-data)
3. [INSERT — Adding Data](#3-insert--adding-data)
4. [UPDATE — Modifying Data](#4-update--modifying-data)
5. [DELETE — Removing Data](#5-delete--removing-data)
6. [TRUNCATE — Fast Removal](#6-truncate--fast-removal)
7. [UPSERT — INSERT ... ON CONFLICT](#7-upsert--insert--on-conflict)
8. [Column Aliases](#8-column-aliases)
9. [ORDER BY — Sorting Results](#9-order-by--sorting-results)
10. [LIMIT & OFFSET — Pagination](#10-limit--offset--pagination)
11. [RETURNING Clause](#11-returning-clause)
12. [NULL Handling](#12-null-handling)
13. [Type Casting](#13-type-casting)
14. [Common Operators & Expressions](#14-common-operators--expressions)
15. [Best Practices](#15-best-practices)
16. [What to Learn Next](#16-what-to-learn-next)

---

## 1. SQL Statement Categories

SQL statements are grouped into four categories:

| Category | Name | Statements | Purpose |
|----------|------|------------|---------|
| **DDL** | Data Definition Language | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` | Define and modify database structure |
| **DML** | Data Manipulation Language | `SELECT`, `INSERT`, `UPDATE`, `DELETE` | Read and modify data |
| **DCL** | Data Control Language | `GRANT`, `REVOKE` | Manage permissions |
| **TCL** | Transaction Control Language | `BEGIN`, `COMMIT`, `ROLLBACK`, `SAVEPOINT` | Manage transactions |

> This file focuses on **DML** — the statements you use most often.

---

## 2. SELECT — Reading Data

The most frequently used SQL statement. Retrieves data from one or more tables.

### Basic Syntax

```sql
SELECT column1, column2, ...
FROM table_name;
```

### Select All Columns

```sql
SELECT * FROM employees;
```

> **Warning**: Avoid `SELECT *` in production code. It fetches all columns, which wastes I/O and bandwidth, and breaks if columns are added or reordered.

### Select Specific Columns

```sql
SELECT first_name, last_name, salary
FROM employees;
```

### Computed Columns

```sql
SELECT
    first_name,
    salary,
    salary * 12 AS annual_salary,
    salary * 12 * 0.08 AS annual_tax
FROM employees;
```

### Conditional Expressions

```sql
SELECT
    name,
    CASE
        WHEN salary > 100000 THEN 'Senior'
        WHEN salary > 50000 THEN 'Mid'
        ELSE 'Junior'
    END AS level
FROM employees;
```

### COALESCE — Default for NULLs

```sql
-- Returns the first non-null value
SELECT COALESCE(nickname, first_name, 'Unknown') AS display_name
FROM employees;
```

### NULLIF — Prevent Division by Zero

```sql
-- Returns NULL if both arguments are equal
SELECT total_sales / NULLIF(total_orders, 0) AS avg_order_value
FROM reports;
```

---

## 3. INSERT — Adding Data

### Single Row Insert

```sql
INSERT INTO employees (first_name, last_name, email, salary)
VALUES ('Alice', 'Smith', 'alice@example.com', 85000);
```

### Multi-Row Insert

```sql
INSERT INTO employees (first_name, last_name, email, salary)
VALUES
    ('Bob', 'Jones', 'bob@example.com', 92000),
    ('Carol', 'Davis', 'carol@example.com', 78000),
    ('Dave', 'Wilson', 'dave@example.com', 105000);
```

> **Performance**: Multi-row INSERT is significantly faster than individual INSERTs — one round-trip instead of many.

### INSERT from SELECT

```sql
INSERT INTO archived_employees (first_name, last_name, email)
SELECT first_name, last_name, email
FROM employees
WHERE terminated_at < '2024-01-01';
```

### INSERT with Default Values

```sql
-- Uses DEFAULT for columns with default values defined
INSERT INTO employees (first_name, last_name)
VALUES ('Eve', 'Brown');
-- id, created_at, etc. use their DEFAULT values

-- Insert a row with all defaults
INSERT INTO audit_log DEFAULT VALUES;
```

---

## 4. UPDATE — Modifying Data

### Basic Update

```sql
UPDATE employees
SET salary = 95000
WHERE id = 42;
```

### Update Multiple Columns

```sql
UPDATE employees
SET
    salary = 95000,
    department = 'Engineering',
    updated_at = NOW()
WHERE id = 42;
```

### Update with Expression

```sql
-- Give everyone a 10% raise
UPDATE employees
SET salary = salary * 1.10
WHERE department = 'Engineering';
```

### Update from Another Table

```sql
UPDATE employees e
SET department_name = d.name
FROM departments d
WHERE e.department_id = d.id;
```

> **Warning**: `UPDATE` without `WHERE` modifies **every row** in the table. Always double-check your `WHERE` clause.

### Conditional Update

```sql
UPDATE employees
SET salary = CASE
    WHEN performance_rating >= 5 THEN salary * 1.15
    WHEN performance_rating >= 3 THEN salary * 1.05
    ELSE salary
END
WHERE department = 'Sales';
```

---

## 5. DELETE — Removing Data

### Basic Delete

```sql
DELETE FROM employees
WHERE id = 42;
```

### Delete with Subquery

```sql
DELETE FROM employees
WHERE department_id IN (
    SELECT id FROM departments WHERE name = 'Deprecated'
);
```

### Delete with JOIN (PostgreSQL-specific)

```sql
DELETE FROM employees e
USING departments d
WHERE e.department_id = d.id
  AND d.name = 'Deprecated';
```

> **Warning**: `DELETE` without `WHERE` removes **every row** from the table. Use `TRUNCATE` if that's intentional (it's faster).

---

## 6. TRUNCATE — Fast Removal

Removes **all rows** from a table without scanning each row.

```sql
-- Remove all rows
TRUNCATE TABLE employees;

-- Reset identity/serial counter to 1
TRUNCATE TABLE employees RESTART IDENTITY;

-- Truncate multiple tables
TRUNCATE TABLE employees, departments;

-- Cascade to tables with foreign key references
TRUNCATE TABLE departments CASCADE;
```

### DELETE vs TRUNCATE

| Feature | `DELETE` | `TRUNCATE` |
|---------|----------|------------|
| Removes specific rows | Yes (`WHERE` clause) | No — always all rows |
| Fires row triggers | Yes | No (only statement-level triggers) |
| WAL logging | Each row logged | Minimal logging |
| Speed | Slow for large tables | Near-instant |
| Can be rolled back | Yes | Yes (in PostgreSQL, unlike some DBs) |
| Resets sequences | No | Optional (`RESTART IDENTITY`) |
| Requires `TRUNCATE` privilege | — | Yes |

> **Key**: In PostgreSQL, `TRUNCATE` is **transactional** — it can be rolled back inside a transaction.

---

## 7. UPSERT — INSERT ... ON CONFLICT

Insert a row, or update it if it already exists (based on a unique constraint).

### Do Nothing on Conflict

```sql
INSERT INTO employees (email, first_name, last_name)
VALUES ('alice@example.com', 'Alice', 'Smith')
ON CONFLICT (email) DO NOTHING;
```

### Update on Conflict (True Upsert)

```sql
INSERT INTO employees (email, first_name, last_name, salary)
VALUES ('alice@example.com', 'Alice', 'Smith', 95000)
ON CONFLICT (email)
DO UPDATE SET
    first_name = EXCLUDED.first_name,
    last_name = EXCLUDED.last_name,
    salary = EXCLUDED.salary,
    updated_at = NOW();
```

> `EXCLUDED` refers to the row that was proposed for insertion but conflicted.

### Upsert with Condition

```sql
INSERT INTO products (sku, name, price)
VALUES ('ABC-123', 'Widget', 29.99)
ON CONFLICT (sku)
DO UPDATE SET price = EXCLUDED.price
WHERE products.price <> EXCLUDED.price;  -- Only update if price actually changed
```

---

## 8. Column Aliases

Rename output columns for clarity.

```sql
SELECT
    first_name AS "First Name",   -- Quoted: preserves case and spaces
    last_name AS surname,          -- Unquoted: lowercased by PostgreSQL
    salary * 12 annual_income      -- AS keyword is optional
FROM employees;
```

### Table Aliases

```sql
SELECT e.first_name, d.name AS department
FROM employees AS e
JOIN departments AS d ON e.department_id = d.id;
```

> **Best practice**: Always use table aliases in multi-table queries to avoid ambiguity.

---

## 9. ORDER BY — Sorting Results

```sql
-- Ascending (default)
SELECT * FROM employees ORDER BY last_name;

-- Descending
SELECT * FROM employees ORDER BY salary DESC;

-- Multiple columns
SELECT * FROM employees ORDER BY department ASC, salary DESC;

-- By column position (not recommended for readability)
SELECT first_name, salary FROM employees ORDER BY 2 DESC;

-- NULL ordering
SELECT * FROM employees ORDER BY manager_id NULLS LAST;
SELECT * FROM employees ORDER BY manager_id NULLS FIRST;
```

### How NULLs Sort

| Direction | Default NULL position | Override |
|-----------|----------------------|----------|
| `ASC` | `NULLS LAST` | `NULLS FIRST` |
| `DESC` | `NULLS FIRST` | `NULLS LAST` |

---

## 10. LIMIT & OFFSET — Pagination

### Basic Pagination

```sql
-- First 10 rows
SELECT * FROM employees ORDER BY id LIMIT 10;

-- Skip first 20, then take 10 (page 3)
SELECT * FROM employees ORDER BY id LIMIT 10 OFFSET 20;
```

### SQL Standard Syntax (FETCH)

```sql
-- Equivalent to LIMIT 10
SELECT * FROM employees ORDER BY id FETCH FIRST 10 ROWS ONLY;

-- Equivalent to LIMIT 10 OFFSET 20
SELECT * FROM employees ORDER BY id OFFSET 20 ROWS FETCH NEXT 10 ROWS ONLY;
```

### Why OFFSET Is Slow for Large Datasets

```sql
-- Bad: PostgreSQL must scan and discard 1,000,000 rows
SELECT * FROM events ORDER BY id LIMIT 20 OFFSET 1000000;

-- Good: Keyset pagination — use a WHERE clause instead
SELECT * FROM events
WHERE id > 1000000    -- last seen id from previous page
ORDER BY id
LIMIT 20;
```

> **Keyset pagination** is O(log n) with an index vs O(n) for OFFSET. Always prefer it for large datasets.

---

## 11. RETURNING Clause

Get back data from rows affected by INSERT, UPDATE, or DELETE — without a separate SELECT.

### INSERT with RETURNING

```sql
INSERT INTO employees (first_name, last_name, email)
VALUES ('Alice', 'Smith', 'alice@example.com')
RETURNING id, created_at;
-- Returns: id = 42, created_at = 2026-03-27 10:00:00+00
```

### UPDATE with RETURNING

```sql
UPDATE employees
SET salary = salary * 1.10
WHERE department = 'Engineering'
RETURNING id, first_name, salary AS new_salary;
```

### DELETE with RETURNING

```sql
DELETE FROM sessions
WHERE expires_at < NOW()
RETURNING id, user_id;
```

> **Key advantage**: `RETURNING` saves a round-trip. Without it, you'd need a separate `SELECT` to get the affected data.

---

## 12. NULL Handling

NULL is not a value — it represents the **absence of a value**. This has important implications:

### Rules

```sql
-- NULL comparisons always return NULL (not true or false)
SELECT NULL = NULL;       -- NULL (not true!)
SELECT NULL <> NULL;      -- NULL (not false!)
SELECT NULL > 5;          -- NULL

-- Use IS NULL / IS NOT NULL
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE email IS NOT NULL;

-- NULL in arithmetic propagates
SELECT 100 + NULL;        -- NULL
SELECT NULL || 'hello';   -- NULL (string concatenation)
```

### COALESCE — Replace NULL

```sql
SELECT COALESCE(phone, mobile, 'No contact') AS contact_number
FROM employees;
```

### NULL in Aggregates

```sql
-- Aggregate functions ignore NULLs (except COUNT(*))
SELECT
    COUNT(*)           AS total_rows,        -- counts all rows
    COUNT(manager_id)  AS rows_with_manager, -- counts non-null values
    AVG(salary)        AS avg_salary         -- averages non-null values
FROM employees;
```

### NULL in Boolean Logic

| Expression | Result |
|------------|--------|
| `TRUE AND NULL` | `NULL` |
| `FALSE AND NULL` | `FALSE` |
| `TRUE OR NULL` | `TRUE` |
| `FALSE OR NULL` | `NULL` |
| `NOT NULL` | `NULL` |

---

## 13. Type Casting

PostgreSQL is strongly typed. Use casting to convert between types.

```sql
-- CAST function (SQL standard)
SELECT CAST('42' AS integer);
SELECT CAST('2026-03-27' AS date);

-- :: shorthand (PostgreSQL-specific, more common)
SELECT '42'::integer;
SELECT '2026-03-27'::date;
SELECT 3.14::text;
SELECT '{"name": "Alice"}'::jsonb;

-- Implicit casting (sometimes works, don't rely on it)
SELECT 1 + '2';  -- works: '2' implicitly cast to integer
```

> **Best practice**: Use explicit casts (`::type`) to make your intent clear and avoid surprises.

---

## 14. Common Operators & Expressions

### Comparison Operators

| Operator | Meaning |
|----------|---------|
| `=` | Equal |
| `<>` or `!=` | Not equal |
| `<`, `>` | Less than, greater than |
| `<=`, `>=` | Less or equal, greater or equal |
| `BETWEEN x AND y` | Range (inclusive both ends) |
| `IN (a, b, c)` | Match any value in list |
| `LIKE` / `ILIKE` | Pattern match (`%` = any chars, `_` = one char). `ILIKE` is case-insensitive |
| `SIMILAR TO` | SQL regex pattern matching |
| `~` / `~*` | POSIX regex match / case-insensitive |

### Logical Operators

```sql
-- AND: both must be true
WHERE salary > 50000 AND department = 'Engineering'

-- OR: either can be true
WHERE department = 'Engineering' OR department = 'Product'

-- NOT: negate a condition
WHERE NOT (status = 'inactive')

-- Precedence: NOT > AND > OR (use parentheses for clarity)
WHERE (dept = 'Eng' OR dept = 'Product') AND salary > 50000
```

### String Operators

```sql
-- Concatenation
SELECT 'Hello' || ' ' || 'World';          -- 'Hello World'
SELECT CONCAT('Hello', ' ', 'World');       -- 'Hello World' (NULL-safe)
SELECT CONCAT_WS(', ', 'Alice', 'Bob');     -- 'Alice, Bob'

-- Common string functions
SELECT LENGTH('Hello');                      -- 5
SELECT UPPER('hello');                       -- 'HELLO'
SELECT LOWER('HELLO');                       -- 'hello'
SELECT TRIM('  hello  ');                    -- 'hello'
SELECT SUBSTRING('Hello World' FROM 1 FOR 5); -- 'Hello'
SELECT REPLACE('Hello World', 'World', 'PG'); -- 'Hello PG'
SELECT LEFT('Hello', 3);                     -- 'Hel'
SELECT RIGHT('Hello', 3);                    -- 'llo'
SELECT POSITION('World' IN 'Hello World');   -- 7
```

### Date/Time Expressions

```sql
-- Current date/time
SELECT NOW();                  -- timestamp with time zone
SELECT CURRENT_TIMESTAMP;      -- same as NOW()
SELECT CURRENT_DATE;           -- date only
SELECT CURRENT_TIME;           -- time only

-- Date arithmetic
SELECT NOW() + INTERVAL '7 days';
SELECT NOW() - INTERVAL '1 hour';
SELECT '2026-12-31'::date - '2026-01-01'::date;  -- 365 (integer days)

-- Extract parts
SELECT EXTRACT(YEAR FROM NOW());
SELECT EXTRACT(MONTH FROM NOW());
SELECT DATE_TRUNC('month', NOW());   -- truncate to start of month

-- Format
SELECT TO_CHAR(NOW(), 'YYYY-MM-DD HH24:MI:SS');
```

---

## 15. Best Practices

| Practice | Why |
|----------|-----|
| Never use `SELECT *` in application code | Wastes I/O, breaks if schema changes |
| Always use a `WHERE` clause with `UPDATE` / `DELETE` | Prevents accidental full-table modification |
| Use `RETURNING` instead of separate `SELECT` | Saves a round-trip to the database |
| Prefer multi-row `INSERT` over loops | One round-trip vs N round-trips |
| Use keyset pagination over `OFFSET` | O(log n) vs O(n) for deep pages |
| Use parameterized queries, never string concatenation | Prevents SQL injection |
| Qualify column names in multi-table queries | Avoids ambiguity and future breakage |
| End every statement with `;` | Required in psql and most tools |

---

## 16. What to Learn Next

1. **Data Types** — PostgreSQL's rich type system — see [15-data-types.md](15-data-types.md).
2. **Table Basics & Constraints** — `CREATE TABLE`, primary keys, foreign keys — see [16-table-basics-constraints.md](16-table-basics-constraints.md).
3. **Filtering & Sorting** — `WHERE`, `AND/OR`, `EXISTS`, `DISTINCT` — Phase 2 (Querying).
4. **Joins** — Combining tables — Phase 2 (Querying).
5. **Transactions** — `BEGIN`, `COMMIT`, `ROLLBACK` — see [02-transactions.md](02-transactions.md).

---

> *Ref: [Neon Tutorial — SQL Basics](https://neon.com/postgresql/tutorial) · [Docs — SQL Commands](https://www.postgresql.org/docs/18/sql-commands.html) · [Docs — Functions and Operators](https://www.postgresql.org/docs/18/functions.html)*
