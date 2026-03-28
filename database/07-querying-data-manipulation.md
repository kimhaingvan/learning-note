# Querying & Data Manipulation

> **Level**: Beginner–Intermediate | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 2 (2.1 – 2.6)  
> Mastering SELECT: filtering, joins, aggregation, subqueries, CTEs, set operations, and window functions — everything you need to extract insights from PostgreSQL.

---

## Table of Contents

1. [Filtering & Sorting](#1-filtering--sorting)
   - [WHERE Clause](#11-where-clause)
   - [Comparison Operators](#12-comparison-operators)
   - [Pattern Matching](#13-pattern-matching)
   - [Logical Operators](#14-logical-operators)
   - [IN / NOT IN](#15-in--not-in)
   - [EXISTS](#16-exists)
   - [BETWEEN](#17-between)
   - [DISTINCT / DISTINCT ON](#18-distinct--distinct-on)
   - [ORDER BY](#19-order-by)
2. [Joins](#2-joins)
   - [INNER JOIN](#21-inner-join)
   - [LEFT JOIN](#22-left-join)
   - [RIGHT JOIN / FULL OUTER JOIN](#23-right-join--full-outer-join)
   - [CROSS JOIN](#24-cross-join)
   - [SELF JOIN](#25-self-join)
   - [LATERAL JOIN](#26-lateral-join)
   - [NATURAL JOIN](#27-natural-join)
   - [Join Performance Tips](#28-join-performance-tips)
3. [Grouping & Aggregation](#3-grouping--aggregation)
   - [GROUP BY](#31-group-by)
   - [HAVING](#32-having)
   - [Aggregate Functions](#33-aggregate-functions)
   - [FILTER Clause](#34-filter-clause)
   - [GROUPING SETS / ROLLUP / CUBE](#35-grouping-sets--rollup--cube)
4. [Subqueries & CTEs](#4-subqueries--ctes)
   - [Scalar Subqueries](#41-scalar-subqueries)
   - [Correlated Subqueries](#42-correlated-subqueries)
   - [Common Table Expressions (CTEs)](#43-common-table-expressions-ctes)
   - [Recursive CTEs](#44-recursive-ctes)
   - [CTE Materialization](#45-cte-materialization)
5. [Set Operations](#5-set-operations)
   - [UNION / UNION ALL](#51-union--union-all)
   - [INTERSECT / EXCEPT](#52-intersect--except)
6. [Window Functions](#6-window-functions)
   - [Anatomy of a Window Function](#61-anatomy-of-a-window-function)
   - [Ranking Functions](#62-ranking-functions)
   - [Value Functions](#63-value-functions)
   - [Aggregate as Window](#64-aggregate-as-window)
   - [Frame Clause](#65-frame-clause)
   - [Named Windows](#66-named-windows)
   - [Practical Examples](#67-practical-examples)
7. [Query Execution Order](#7-query-execution-order)
8. [Common Mistakes](#8-common-mistakes)
9. [What to Learn Next](#9-what-to-learn-next)

---

## Sample Data

All examples in this file use the following tables:

```sql
CREATE TABLE departments (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL UNIQUE
);

CREATE TABLE employees (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name text NOT NULL,
    last_name text NOT NULL,
    email text NOT NULL UNIQUE,
    salary numeric(10, 2) NOT NULL,
    department_id integer REFERENCES departments(id),
    manager_id integer REFERENCES employees(id),
    hired_at date NOT NULL DEFAULT CURRENT_DATE,
    active boolean NOT NULL DEFAULT true
);

CREATE TABLE orders (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    employee_id integer NOT NULL REFERENCES employees(id),
    customer_name text NOT NULL,
    total numeric(12, 2) NOT NULL,
    status text NOT NULL DEFAULT 'pending',
    created_at timestamptz NOT NULL DEFAULT NOW()
);
```

---

## 1. Filtering & Sorting

### 1.1 WHERE Clause

The `WHERE` clause filters rows **before** grouping and aggregation.

```sql
SELECT first_name, last_name, salary
FROM employees
WHERE salary > 80000
  AND department_id = 3
  AND active = true;
```

### 1.2 Comparison Operators

| Operator | Meaning | Example |
|----------|---------|---------|
| `=` | Equal | `WHERE status = 'active'` |
| `<>` or `!=` | Not equal | `WHERE status <> 'deleted'` |
| `<`, `>` | Less / greater than | `WHERE salary > 50000` |
| `<=`, `>=` | Less/greater or equal | `WHERE hired_at >= '2025-01-01'` |
| `IS NULL` | Is null | `WHERE manager_id IS NULL` |
| `IS NOT NULL` | Is not null | `WHERE email IS NOT NULL` |
| `IS DISTINCT FROM` | NULL-safe not-equal | `WHERE a IS DISTINCT FROM b` |

#### IS DISTINCT FROM (NULL-safe comparison)

```sql
-- Regular comparison: NULL = NULL → NULL (not true!)
SELECT * FROM employees WHERE manager_id = NULL;   -- Returns nothing!

-- NULL-safe: treats NULLs as equal
SELECT * FROM employees WHERE manager_id IS NOT DISTINCT FROM NULL;  -- Returns rows where manager_id IS NULL
```

### 1.3 Pattern Matching

```sql
-- LIKE (case-sensitive)
SELECT * FROM employees WHERE last_name LIKE 'Smi%';     -- Starts with 'Smi'
SELECT * FROM employees WHERE email LIKE '%@gmail.com';   -- Ends with '@gmail.com'
SELECT * FROM employees WHERE first_name LIKE '_o%';      -- Second char is 'o'

-- ILIKE (case-insensitive, PostgreSQL-specific)
SELECT * FROM employees WHERE last_name ILIKE '%smith%';

-- POSIX regex (~)
SELECT * FROM employees WHERE email ~ '^[a-z]+@example\.com$';   -- case-sensitive
SELECT * FROM employees WHERE email ~* '^[a-z]+@example\.com$';  -- case-insensitive

-- SIMILAR TO (SQL-standard regex, rarely used)
SELECT * FROM employees WHERE email SIMILAR TO '%@(gmail|yahoo)\.com';
```

| Operator | Type | Case |
|----------|------|------|
| `LIKE` | SQL wildcard (`%`, `_`) | Sensitive |
| `ILIKE` | SQL wildcard | Insensitive |
| `~` | POSIX regex | Sensitive |
| `~*` | POSIX regex | Insensitive |
| `!~` / `!~*` | POSIX regex (negated) | Sensitive / Insensitive |

> **Performance**: `LIKE 'prefix%'` can use a B-tree index. `LIKE '%suffix'` cannot (requires sequential scan or trigram index).

### 1.4 Logical Operators

```sql
-- AND: both conditions must be true
WHERE salary > 50000 AND department_id = 3

-- OR: either condition can be true
WHERE department_id = 1 OR department_id = 2

-- NOT: negate a condition
WHERE NOT active

-- Precedence: NOT > AND > OR
-- These are different:
WHERE a = 1 OR b = 2 AND c = 3     -- means: a=1 OR (b=2 AND c=3)
WHERE (a = 1 OR b = 2) AND c = 3   -- means: (a=1 OR b=2) AND c=3
```

> **Best practice**: Always use parentheses to make operator precedence explicit. Never rely on implicit ordering.

### 1.5 IN / NOT IN

```sql
-- Match against a list
SELECT * FROM employees
WHERE department_id IN (1, 3, 5);

-- Match against a subquery
SELECT * FROM employees
WHERE department_id IN (SELECT id FROM departments WHERE name LIKE 'Eng%');

-- NOT IN
SELECT * FROM employees
WHERE department_id NOT IN (1, 3, 5);
```

#### The NOT IN + NULL Trap

```sql
-- If the subquery returns ANY null, NOT IN returns no rows!
SELECT * FROM employees
WHERE department_id NOT IN (1, 2, NULL);
-- Returns NOTHING — because `department_id <> NULL` is always NULL

-- Fix: use NOT EXISTS instead
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM excluded_depts x WHERE x.id = e.department_id
);
```

> **Rule**: Prefer `NOT EXISTS` over `NOT IN` when the subquery might contain NULLs.

### 1.6 EXISTS

Tests whether a subquery returns **any** rows. Returns `true` or `false`.

```sql
-- Employees who have placed at least one order
SELECT e.first_name, e.last_name
FROM employees e
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.employee_id = e.id
);

-- Employees who have never placed an order (anti-join)
SELECT e.first_name, e.last_name
FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.employee_id = e.id
);
```

> **Performance**: `EXISTS` short-circuits — it stops after finding the first matching row. This makes it efficient for "has any?" checks.

### 1.7 BETWEEN

```sql
-- Inclusive range (equivalent to >= AND <=)
SELECT * FROM employees WHERE salary BETWEEN 50000 AND 100000;

-- Date range
SELECT * FROM orders WHERE created_at BETWEEN '2026-01-01' AND '2026-03-31';

-- NOT BETWEEN
SELECT * FROM employees WHERE salary NOT BETWEEN 50000 AND 100000;
```

> **Warning**: `BETWEEN` includes **both** endpoints. For timestamps, this can be tricky — `BETWEEN '2026-01-01' AND '2026-01-31'` misses times after midnight on the 31st. Prefer explicit `>=` and `<`:

```sql
-- Better for date ranges:
WHERE created_at >= '2026-01-01' AND created_at < '2026-02-01'
```

### 1.8 DISTINCT / DISTINCT ON

```sql
-- Remove duplicate rows (all columns must match)
SELECT DISTINCT department_id FROM employees;

-- DISTINCT ON: keep only the first row per group (PostgreSQL-specific)
-- Get the highest-paid employee per department
SELECT DISTINCT ON (department_id)
    department_id, first_name, last_name, salary
FROM employees
ORDER BY department_id, salary DESC;
```

#### How DISTINCT ON Works

1. PostgreSQL sorts by `department_id, salary DESC`.
2. For each unique `department_id`, it returns only the **first** row in that sort order.
3. The `ORDER BY` must start with the `DISTINCT ON` columns.

> **Key insight**: `DISTINCT ON` is one of PostgreSQL's most powerful non-standard features. It replaces complex `ROW_NUMBER()` + subquery patterns.

### 1.9 ORDER BY

```sql
-- Ascending (default)
SELECT * FROM employees ORDER BY last_name;

-- Descending
SELECT * FROM employees ORDER BY salary DESC;

-- Multiple columns
SELECT * FROM employees ORDER BY department_id ASC, salary DESC;

-- By expression
SELECT first_name, last_name, salary
FROM employees
ORDER BY salary * 12 DESC;

-- NULL ordering
SELECT * FROM employees ORDER BY manager_id NULLS LAST;   -- NULLs at the end
SELECT * FROM employees ORDER BY manager_id DESC NULLS FIRST;

-- By ordinal position (discouraged but valid)
SELECT first_name, salary FROM employees ORDER BY 2 DESC;  -- 2nd column (salary)
```

| Direction | Default NULL position | Override |
|-----------|----------------------|----------|
| `ASC` | `NULLS LAST` | `NULLS FIRST` |
| `DESC` | `NULLS FIRST` | `NULLS LAST` |

---

## 2. Joins

Joins combine rows from two or more tables based on related columns.

### Conceptual Overview

```
Table A                Table B
┌────┬──────┐         ┌────┬──────┐
│ id │ name │         │ id │ a_id │
├────┼──────┤         ├────┼──────┤
│  1 │ Foo  │         │ 10 │   1  │
│  2 │ Bar  │         │ 20 │   1  │
│  3 │ Baz  │         │ 30 │   4  │  ← no match in A
└────┴──────┘         └────┴──────┘
       ↑ no match in B (id=2, id=3)

INNER JOIN:  (1,Foo,10), (1,Foo,20)            ← only matches
LEFT JOIN:   above + (2,Bar,NULL), (3,Baz,NULL) ← all of A
RIGHT JOIN:  (1,Foo,10), (1,Foo,20), (NULL,NULL,30) ← all of B
FULL OUTER:  all of the above                   ← all of A + all of B
CROSS JOIN:  3 × 3 = 9 rows                     ← every combination
```

### 2.1 INNER JOIN

Returns rows only when there is a match in **both** tables.

```sql
SELECT e.first_name, e.last_name, d.name AS department
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;

-- Shorthand (JOIN = INNER JOIN)
SELECT e.first_name, d.name AS department
FROM employees e
JOIN departments d ON e.department_id = d.id;
```

### 2.2 LEFT JOIN

Returns **all rows from the left table**, plus matched rows from the right. Unmatched right-side columns are `NULL`.

```sql
-- All employees, even those without a department
SELECT e.first_name, e.last_name, d.name AS department
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
```

#### Finding Unmatched Rows (Anti-Join)

```sql
-- Employees with no department assigned
SELECT e.first_name, e.last_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.id IS NULL;
```

### 2.3 RIGHT JOIN / FULL OUTER JOIN

```sql
-- RIGHT JOIN: all departments, even those with no employees
SELECT e.first_name, d.name AS department
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;

-- FULL OUTER JOIN: all employees + all departments, unmatched on either side
SELECT e.first_name, d.name AS department
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.id;
```

> **Tip**: `RIGHT JOIN` is rarely used — you can always rewrite it as a `LEFT JOIN` by swapping the table order.

### 2.4 CROSS JOIN

Produces the **Cartesian product** — every row from A combined with every row from B.

```sql
-- Every employee × every department (10 employees × 5 departments = 50 rows)
SELECT e.first_name, d.name
FROM employees e
CROSS JOIN departments d;

-- Equivalent:
SELECT e.first_name, d.name
FROM employees e, departments d;
```

**Use cases**: Generating date grids, creating all combinations for reporting.

```sql
-- Generate a report grid: every department × every month
SELECT d.name, m.month
FROM departments d
CROSS JOIN generate_series('2026-01-01'::date, '2026-12-01'::date, '1 month') AS m(month);
```

### 2.5 SELF JOIN

Joining a table with itself — typically for hierarchical data.

```sql
-- Employee and their manager name
SELECT
    e.first_name AS employee,
    m.first_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Find employees who earn more than their manager
SELECT
    e.first_name AS employee,
    e.salary AS emp_salary,
    m.first_name AS manager,
    m.salary AS mgr_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

### 2.6 LATERAL JOIN

Allows the right-side subquery to **reference columns from the left-side** table. Like a correlated subquery, but as a table source.

```sql
-- For each department, get the top 3 highest-paid employees
SELECT d.name AS department, top.first_name, top.salary
FROM departments d
CROSS JOIN LATERAL (
    SELECT e.first_name, e.salary
    FROM employees e
    WHERE e.department_id = d.id
    ORDER BY e.salary DESC
    LIMIT 3
) AS top;
```

#### LEFT JOIN LATERAL

```sql
-- Same, but include departments with no employees
SELECT d.name AS department, top.first_name, top.salary
FROM departments d
LEFT JOIN LATERAL (
    SELECT e.first_name, e.salary
    FROM employees e
    WHERE e.department_id = d.id
    ORDER BY e.salary DESC
    LIMIT 3
) AS top ON true;
```

> **When to use LATERAL**: When you need the "top N per group" pattern, or when the right-side needs to reference left-side columns dynamically. It's more readable than window-function alternatives for many use cases.

### 2.7 NATURAL JOIN

Automatically joins on **all columns with the same name** in both tables.

```sql
SELECT * FROM employees NATURAL JOIN departments;
-- Equivalent to: JOIN departments ON employees.id = departments.id (if 'id' is the shared name)
```

> **Warning**: Avoid `NATURAL JOIN` in production. If you add a column with a matching name later, the join silently changes. Always specify join conditions explicitly.

### 2.8 Join Performance Tips

| Tip | Why |
|-----|-----|
| Join on indexed columns | Enables index scan instead of sequential scan |
| Index foreign key columns | PostgreSQL does **not** auto-index FKs |
| Prefer `EXISTS` over `IN` for semi-joins | Short-circuits, better with NULLs |
| Prefer `NOT EXISTS` over `NOT IN` | Avoids the NULL trap |
| Prefer `LEFT JOIN ... WHERE x IS NULL` or `NOT EXISTS` for anti-joins | Both efficient; `NOT IN` is dangerous |
| Use `EXPLAIN ANALYZE` to check join strategy | Nested Loop vs Hash Join vs Merge Join |
| Filter early | Place conditions in `WHERE` for inner joins, in `ON` for outer joins |

#### ON vs WHERE in Outer Joins

```sql
-- These are DIFFERENT for outer joins:

-- Filter in ON: returns ALL employees, but only matches active departments
SELECT e.*, d.name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id AND d.active = true;

-- Filter in WHERE: filters the ENTIRE result, removing unmatched rows
SELECT e.*, d.name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id
WHERE d.active = true;  -- This removes rows where d.active IS NULL (no match)
```

> **Rule**: For outer joins, conditions on the **outer** table go in `ON`. Conditions on the **preserved** table go in `WHERE`.

---

## 3. Grouping & Aggregation

### 3.1 GROUP BY

Partitions rows into groups and applies aggregate functions to each group.

```sql
-- Count employees per department
SELECT department_id, COUNT(*) AS employee_count
FROM employees
GROUP BY department_id;

-- Total salary by department
SELECT d.name, SUM(e.salary) AS total_salary, COUNT(*) AS headcount
FROM employees e
JOIN departments d ON e.department_id = d.id
GROUP BY d.name
ORDER BY total_salary DESC;
```

#### GROUP BY Rules

- Every column in `SELECT` must either be in `GROUP BY` or inside an aggregate function.
- You can `GROUP BY` columns not in `SELECT` (to combine groups).
- You can `GROUP BY` expressions: `GROUP BY DATE_TRUNC('month', created_at)`.

```sql
-- Group by expression
SELECT DATE_TRUNC('month', hired_at) AS month, COUNT(*) AS hires
FROM employees
GROUP BY DATE_TRUNC('month', hired_at)
ORDER BY month;

-- Shorthand: group by ordinal position
SELECT DATE_TRUNC('month', hired_at) AS month, COUNT(*) AS hires
FROM employees
GROUP BY 1
ORDER BY 1;
```

### 3.2 HAVING

Filters **groups** (after aggregation), as opposed to `WHERE` which filters **rows** (before aggregation).

```sql
-- Departments with more than 5 employees
SELECT department_id, COUNT(*) AS headcount
FROM employees
GROUP BY department_id
HAVING COUNT(*) > 5;

-- Departments where average salary exceeds 80k
SELECT d.name, AVG(e.salary)::numeric(10,2) AS avg_salary
FROM employees e
JOIN departments d ON e.department_id = d.id
GROUP BY d.name
HAVING AVG(e.salary) > 80000
ORDER BY avg_salary DESC;
```

#### WHERE vs HAVING

| Clause | Filters | Evaluated |
|--------|---------|-----------|
| `WHERE` | Individual rows | **Before** grouping |
| `HAVING` | Groups | **After** grouping |

```sql
-- WHERE filters rows before grouping, HAVING filters groups after
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
WHERE active = true          -- only active employees considered
GROUP BY department_id
HAVING AVG(salary) > 60000   -- only departments with avg > 60k returned
ORDER BY avg_salary DESC;
```

### 3.3 Aggregate Functions

| Function | Description | Example |
|----------|-------------|---------|
| `COUNT(*)` | Count all rows (including NULLs) | `COUNT(*)` |
| `COUNT(expr)` | Count non-null values | `COUNT(manager_id)` |
| `COUNT(DISTINCT expr)` | Count distinct non-null values | `COUNT(DISTINCT department_id)` |
| `SUM(expr)` | Sum of values | `SUM(salary)` |
| `AVG(expr)` | Average (ignores NULLs) | `AVG(salary)` |
| `MIN(expr)` | Minimum value | `MIN(hired_at)` |
| `MAX(expr)` | Maximum value | `MAX(salary)` |
| `STRING_AGG(expr, delimiter)` | Concatenate strings | `STRING_AGG(name, ', ')` |
| `ARRAY_AGG(expr)` | Collect into an array | `ARRAY_AGG(name ORDER BY name)` |
| `JSON_AGG(expr)` | Collect into JSON array | `JSON_AGG(row_to_json(e))` |
| `JSONB_AGG(expr)` | Collect into JSONB array | `JSONB_AGG(data)` |
| `JSON_OBJECT_AGG(key, value)` | Build JSON object from key-value pairs | `JSON_OBJECT_AGG(name, salary)` |
| `BOOL_AND(expr)` | True if ALL true | `BOOL_AND(active)` |
| `BOOL_OR(expr)` | True if ANY true | `BOOL_OR(active)` |
| `BIT_AND(expr)` | Bitwise AND | `BIT_AND(flags)` |
| `BIT_OR(expr)` | Bitwise OR | `BIT_OR(flags)` |
| `PERCENTILE_CONT(frac)` | Continuous percentile (ordered set) | See below |
| `PERCENTILE_DISC(frac)` | Discrete percentile (ordered set) | See below |
| `MODE()` | Most frequent value | See below |

#### Practical Examples

```sql
-- Comma-separated list of employees per department
SELECT d.name,
       STRING_AGG(e.first_name, ', ' ORDER BY e.first_name) AS employees
FROM departments d
JOIN employees e ON d.id = e.department_id
GROUP BY d.name;

-- Array of employee names
SELECT department_id,
       ARRAY_AGG(first_name ORDER BY first_name) AS names
FROM employees
GROUP BY department_id;

-- Median salary (ordered-set aggregate)
SELECT
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees;

-- Mode (most common salary)
SELECT
    MODE() WITHIN GROUP (ORDER BY salary) AS most_common_salary
FROM employees;
```

### 3.4 FILTER Clause

Apply a condition to an aggregate without affecting other aggregates or the `WHERE` clause.

```sql
SELECT
    department_id,
    COUNT(*) AS total,
    COUNT(*) FILTER (WHERE active = true) AS active_count,
    COUNT(*) FILTER (WHERE active = false) AS inactive_count,
    AVG(salary) FILTER (WHERE active = true) AS active_avg_salary,
    SUM(salary) FILTER (WHERE hired_at >= '2025-01-01') AS recent_salary_total
FROM employees
GROUP BY department_id;
```

> **Before FILTER existed**, you'd use `CASE`:
> ```sql
> SUM(CASE WHEN active THEN 1 ELSE 0 END) AS active_count  -- old way
> COUNT(*) FILTER (WHERE active)                            -- modern way
> ```

### 3.5 GROUPING SETS / ROLLUP / CUBE

Compute multiple levels of aggregation in a single query.

#### GROUPING SETS

```sql
-- Individual aggregations + overall total in one query
SELECT department_id, status, COUNT(*) AS cnt
FROM orders o
JOIN employees e ON o.employee_id = e.id
GROUP BY GROUPING SETS (
    (department_id, status),  -- per department + status
    (department_id),          -- per department subtotal
    (status),                 -- per status subtotal
    ()                        -- grand total
)
ORDER BY department_id, status;
```

#### ROLLUP

Hierarchical subtotals — removes columns from right to left.

```sql
-- ROLLUP(a, b, c) = GROUPING SETS ((a,b,c), (a,b), (a), ())
SELECT
    EXTRACT(YEAR FROM created_at) AS year,
    EXTRACT(MONTH FROM created_at) AS month,
    SUM(total) AS revenue
FROM orders
GROUP BY ROLLUP (
    EXTRACT(YEAR FROM created_at),
    EXTRACT(MONTH FROM created_at)
)
ORDER BY year, month;
```

Result includes: per-year-per-month, per-year subtotals, and grand total.

#### CUBE

All possible combinations of grouping columns.

```sql
-- CUBE(a, b) = GROUPING SETS ((a,b), (a), (b), ())
SELECT department_id, status, COUNT(*)
FROM orders o
JOIN employees e ON o.employee_id = e.id
GROUP BY CUBE (department_id, status)
ORDER BY department_id, status;
```

#### GROUPING() Function

Identifies which columns are aggregated (NULL from grouping vs NULL from data).

```sql
SELECT
    department_id,
    status,
    COUNT(*) AS cnt,
    GROUPING(department_id) AS dept_is_agg,   -- 1 if aggregated (NULL = subtotal)
    GROUPING(status) AS status_is_agg
FROM orders o
JOIN employees e ON o.employee_id = e.id
GROUP BY CUBE (department_id, status);
```

---

## 4. Subqueries & CTEs

### 4.1 Scalar Subqueries

A subquery that returns **exactly one value**. Can be used anywhere an expression is allowed.

```sql
-- Compare each employee's salary to the overall average
SELECT
    first_name,
    salary,
    salary - (SELECT AVG(salary) FROM employees) AS diff_from_avg
FROM employees;

-- Use in WHERE
SELECT * FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
```

> If a scalar subquery returns more than one row, PostgreSQL raises an error.

### 4.2 Correlated Subqueries

References columns from the **outer query**. Re-evaluated for each outer row.

```sql
-- Employees earning above their department average
SELECT e.first_name, e.salary, e.department_id
FROM employees e
WHERE e.salary > (
    SELECT AVG(e2.salary)
    FROM employees e2
    WHERE e2.department_id = e.department_id  -- references outer query
);
```

```sql
-- For each order, show how it compares to the employee's average order
SELECT
    o.id,
    o.total,
    o.total - (
        SELECT AVG(o2.total)
        FROM orders o2
        WHERE o2.employee_id = o.employee_id
    ) AS diff_from_personal_avg
FROM orders o;
```

> **Performance**: Correlated subqueries can be slow for large datasets because they execute once per outer row. Consider rewriting as a JOIN or CTE.

### 4.3 Common Table Expressions (CTEs)

Named, reusable "temporary result sets" defined with `WITH`. Improve readability for complex queries.

```sql
WITH dept_stats AS (
    SELECT
        department_id,
        AVG(salary) AS avg_salary,
        COUNT(*) AS headcount
    FROM employees
    GROUP BY department_id
)
SELECT
    e.first_name,
    e.salary,
    ds.avg_salary AS dept_avg,
    e.salary - ds.avg_salary AS diff
FROM employees e
JOIN dept_stats ds ON e.department_id = ds.department_id
WHERE e.salary > ds.avg_salary
ORDER BY diff DESC;
```

#### Multiple CTEs

```sql
WITH
    active_employees AS (
        SELECT * FROM employees WHERE active = true
    ),
    dept_totals AS (
        SELECT department_id, SUM(salary) AS total_salary
        FROM active_employees
        GROUP BY department_id
    ),
    top_departments AS (
        SELECT * FROM dept_totals WHERE total_salary > 500000
    )
SELECT d.name, td.total_salary
FROM top_departments td
JOIN departments d ON td.department_id = d.id
ORDER BY td.total_salary DESC;
```

### 4.4 Recursive CTEs

Traverse hierarchical or graph-structured data using `WITH RECURSIVE`.

#### Structure

```sql
WITH RECURSIVE cte_name AS (
    -- Base case (non-recursive term)
    SELECT ...
    UNION ALL
    -- Recursive step (references cte_name)
    SELECT ... FROM cte_name JOIN ...
)
SELECT * FROM cte_name;
```

#### Example: Employee Hierarchy (org chart)

```sql
WITH RECURSIVE org_chart AS (
    -- Base case: CEO (no manager)
    SELECT id, first_name, manager_id, 1 AS level,
           first_name::text AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive step: find direct reports
    SELECT e.id, e.first_name, e.manager_id, oc.level + 1,
           oc.path || ' → ' || e.first_name
    FROM employees e
    JOIN org_chart oc ON e.manager_id = oc.id
)
SELECT id, first_name, level, path
FROM org_chart
ORDER BY path;
```

#### Example: Generate a Number Series

```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n          -- Base case
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 100  -- Recursive step
)
SELECT n FROM nums;
```

#### Safety: Preventing Infinite Loops

```sql
-- Use CYCLE detection (PG 14+)
WITH RECURSIVE tree AS (
    SELECT id, parent_id, name FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.parent_id, c.name FROM categories c JOIN tree t ON c.parent_id = t.id
)
CYCLE id SET is_cycle USING path
SELECT * FROM tree WHERE NOT is_cycle;

-- Alternative: limit depth manually
WITH RECURSIVE tree AS (
    SELECT id, parent_id, 1 AS depth FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.parent_id, t.depth + 1
    FROM categories c JOIN tree t ON c.parent_id = t.id
    WHERE t.depth < 20  -- safety limit
)
SELECT * FROM tree;
```

### 4.5 CTE Materialization

PostgreSQL 12+ lets you control whether a CTE is materialized (computed once) or inlined (optimized by the planner).

```sql
-- Materialized: computed once, stored in temp table (prevents inlining)
WITH stats AS MATERIALIZED (
    SELECT department_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department_id
)
SELECT * FROM stats WHERE avg_sal > 80000;

-- Not materialized: planner can inline and push predicates through
WITH stats AS NOT MATERIALIZED (
    SELECT department_id, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department_id
)
SELECT * FROM stats WHERE avg_sal > 80000;
```

| Behavior | Default (PG 12+) | When to override |
|----------|------------------|-----------------|
| CTE referenced once | Inlined (not materialized) | Use `MATERIALIZED` if repeated subquery execution is expensive |
| CTE referenced multiple times | Materialized | Use `NOT MATERIALIZED` if inlining enables better predicate pushdown |
| Recursive CTE | Always materialized | Cannot override |

> **Before PG 12**: CTEs were **always** materialized, acting as optimization barriers. This was a common performance pitfall.

---

## 5. Set Operations

Combine results from multiple `SELECT` queries. All queries must return the **same number of columns** with compatible types.

### 5.1 UNION / UNION ALL

```sql
-- UNION: combine and remove duplicates (sorted)
SELECT first_name, last_name FROM employees
UNION
SELECT first_name, last_name FROM contractors;

-- UNION ALL: combine and keep duplicates (faster — no dedup sort)
SELECT first_name, last_name FROM employees
UNION ALL
SELECT first_name, last_name FROM contractors;
```

> **Performance**: Always prefer `UNION ALL` unless you specifically need deduplication. `UNION` requires a sort or hash to remove duplicates.

### 5.2 INTERSECT / EXCEPT

```sql
-- INTERSECT: rows present in BOTH result sets
SELECT email FROM employees
INTERSECT
SELECT email FROM newsletter_subscribers;

-- EXCEPT: rows in the FIRST set but NOT the SECOND
SELECT email FROM employees
EXCEPT
SELECT email FROM unsubscribed_users;

-- EXCEPT ALL: preserves duplicate counts
SELECT department_id FROM employees
EXCEPT ALL
SELECT department_id FROM contractors;
```

#### Set Operation Order

```sql
-- INTERSECT has higher precedence than UNION/EXCEPT
SELECT a FROM t1
UNION
SELECT a FROM t2
INTERSECT           -- evaluated first
SELECT a FROM t3;

-- Equivalent to:
SELECT a FROM t1
UNION
(SELECT a FROM t2 INTERSECT SELECT a FROM t3);
```

> Use parentheses to make precedence explicit.

---

## 6. Window Functions

Perform calculations **across related rows** without collapsing them into a single output row — unlike `GROUP BY` which reduces output.

### 6.1 Anatomy of a Window Function

```sql
function_name(args) OVER (
    [PARTITION BY partition_columns]
    [ORDER BY sort_columns]
    [frame_clause]
)
```

| Part | Purpose |
|------|---------|
| `PARTITION BY` | Divide rows into groups (like GROUP BY, but without collapsing) |
| `ORDER BY` | Define the order within each partition |
| Frame clause | Define which rows relative to current row to include in the window |

```sql
-- Basic example: compare each salary to the department average
SELECT
    first_name,
    department_id,
    salary,
    AVG(salary) OVER (PARTITION BY department_id) AS dept_avg
FROM employees;
```

### 6.2 Ranking Functions

| Function | Description | Ties behavior |
|----------|-------------|---------------|
| `ROW_NUMBER()` | Unique sequential number | 1, 2, 3, 4, 5 (no gaps, no ties) |
| `RANK()` | Rank with gaps after ties | 1, 2, 2, 4, 5 (gap after tie) |
| `DENSE_RANK()` | Rank without gaps | 1, 2, 2, 3, 4 (no gaps) |
| `NTILE(n)` | Divide into n equal buckets | 1, 1, 2, 2, 3 (for NTILE(3) on 5 rows) |

```sql
-- Rank employees by salary within each department
SELECT
    first_name,
    department_id,
    salary,
    ROW_NUMBER() OVER w AS row_num,
    RANK()       OVER w AS rank,
    DENSE_RANK() OVER w AS dense_rank,
    NTILE(4)     OVER w AS quartile
FROM employees
WINDOW w AS (PARTITION BY department_id ORDER BY salary DESC);
```

#### Top N Per Group (Common Pattern)

```sql
-- Top 3 highest-paid employees per department
SELECT * FROM (
    SELECT
        first_name,
        department_id,
        salary,
        ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC) AS rn
    FROM employees
) ranked
WHERE rn <= 3;
```

> **Alternative**: Use `DISTINCT ON` (simpler for top-1) or `LATERAL JOIN` (more flexible for top-N).

### 6.3 Value Functions

Access values from other rows in the window without a self-join.

| Function | Description | Default |
|----------|-------------|---------|
| `LAG(expr, n, default)` | Value from `n` rows **before** current | `n=1`, `default=NULL` |
| `LEAD(expr, n, default)` | Value from `n` rows **after** current | `n=1`, `default=NULL` |
| `FIRST_VALUE(expr)` | First value in the window frame | — |
| `LAST_VALUE(expr)` | Last value in the window frame | — |
| `NTH_VALUE(expr, n)` | Nth value in the window frame | — |

```sql
-- Compare each employee's salary to the next and previous
SELECT
    first_name,
    salary,
    LAG(salary)  OVER (ORDER BY salary) AS prev_salary,
    LEAD(salary) OVER (ORDER BY salary) AS next_salary,
    salary - LAG(salary) OVER (ORDER BY salary) AS diff_from_prev
FROM employees;
```

```sql
-- Month-over-month revenue comparison
SELECT
    DATE_TRUNC('month', created_at) AS month,
    SUM(total) AS revenue,
    LAG(SUM(total)) OVER (ORDER BY DATE_TRUNC('month', created_at)) AS prev_month,
    SUM(total) - LAG(SUM(total)) OVER (ORDER BY DATE_TRUNC('month', created_at)) AS growth
FROM orders
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;
```

#### FIRST_VALUE / LAST_VALUE Gotcha

```sql
-- LAST_VALUE often surprises because of the default frame
SELECT
    first_name,
    salary,
    FIRST_VALUE(first_name) OVER w AS lowest_paid,
    LAST_VALUE(first_name)  OVER w AS highest_paid  -- NOT what you'd expect!
FROM employees
WINDOW w AS (ORDER BY salary);
-- Default frame is RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
-- So LAST_VALUE = current row, not the actual last row!

-- Fix: extend the frame
SELECT
    first_name,
    salary,
    FIRST_VALUE(first_name) OVER w AS lowest_paid,
    LAST_VALUE(first_name)  OVER w AS highest_paid  -- Now correct
FROM employees
WINDOW w AS (ORDER BY salary ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING);
```

### 6.4 Aggregate as Window

Any aggregate function can be used as a window function by adding `OVER()`.

```sql
SELECT
    first_name,
    department_id,
    salary,
    -- Running total of salary (ordered by hire date)
    SUM(salary) OVER (ORDER BY hired_at) AS running_total,
    -- Department total
    SUM(salary) OVER (PARTITION BY department_id) AS dept_total,
    -- Percentage of department salary
    ROUND(salary / SUM(salary) OVER (PARTITION BY department_id) * 100, 1) AS pct_of_dept,
    -- Company-wide count
    COUNT(*) OVER () AS company_size
FROM employees;
```

#### Running Total / Cumulative Sum

```sql
SELECT
    created_at::date AS day,
    SUM(total) AS daily_revenue,
    SUM(SUM(total)) OVER (ORDER BY created_at::date) AS cumulative_revenue
FROM orders
GROUP BY created_at::date
ORDER BY day;
```

#### Moving Average

```sql
-- 7-day moving average
SELECT
    created_at::date AS day,
    SUM(total) AS daily_revenue,
    AVG(SUM(total)) OVER (
        ORDER BY created_at::date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS seven_day_avg
FROM orders
GROUP BY created_at::date
ORDER BY day;
```

### 6.5 Frame Clause

Controls exactly which rows relative to the current row are included in the window calculation.

```
frame_clause = { ROWS | RANGE | GROUPS } BETWEEN frame_start AND frame_end
```

| Frame bound | Meaning |
|-------------|---------|
| `UNBOUNDED PRECEDING` | First row of the partition |
| `N PRECEDING` | N rows before current |
| `CURRENT ROW` | The current row |
| `N FOLLOWING` | N rows after current |
| `UNBOUNDED FOLLOWING` | Last row of the partition |

#### ROWS vs RANGE vs GROUPS

| Mode | Unit | Behavior with ties |
|------|------|--------------------|
| `ROWS` | Physical row count | Each row is distinct (most predictable) |
| `RANGE` | Logical value range | Rows with same `ORDER BY` value are treated as peers |
| `GROUPS` | Number of peer groups | Each group of tied values counts as one |

```sql
-- Default frame (when ORDER BY is present):
-- RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- Explicit ROWS frame for 3-row moving average:
AVG(salary) OVER (
    ORDER BY hired_at
    ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING
)

-- Full partition:
SUM(salary) OVER (
    PARTITION BY department_id
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
)
```

### 6.6 Named Windows

Reuse a window definition across multiple functions using the `WINDOW` clause.

```sql
SELECT
    first_name,
    department_id,
    salary,
    ROW_NUMBER() OVER w AS row_num,
    RANK()       OVER w AS rank,
    SUM(salary)  OVER w AS cumulative_salary,
    AVG(salary)  OVER w AS running_avg
FROM employees
WINDOW w AS (PARTITION BY department_id ORDER BY salary DESC);
```

You can also extend named windows:

```sql
SELECT
    first_name,
    salary,
    SUM(salary) OVER (w ORDER BY hired_at) AS running_total
FROM employees
WINDOW w AS (PARTITION BY department_id);
```

### 6.7 Practical Examples

#### Percentage of Total

```sql
SELECT
    d.name AS department,
    COUNT(*) AS headcount,
    ROUND(COUNT(*)::numeric / SUM(COUNT(*)) OVER () * 100, 1) AS pct_of_company
FROM employees e
JOIN departments d ON e.department_id = d.id
GROUP BY d.name
ORDER BY headcount DESC;
```

#### Gap Detection

```sql
-- Find gaps in a sequence of order IDs
SELECT id, next_id, next_id - id AS gap_size
FROM (
    SELECT id, LEAD(id) OVER (ORDER BY id) AS next_id
    FROM orders
) sub
WHERE next_id - id > 1;
```

#### Running Distinct Count

```sql
-- Unique customers per day (cumulative)
SELECT DISTINCT
    created_at::date AS day,
    COUNT(DISTINCT customer_name) OVER (
        ORDER BY created_at::date
        RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS cumulative_customers
FROM orders;
```

#### De-duplication with ROW_NUMBER

```sql
-- Keep only the latest order per customer
DELETE FROM orders
WHERE id IN (
    SELECT id FROM (
        SELECT id, ROW_NUMBER() OVER (
            PARTITION BY customer_name ORDER BY created_at DESC
        ) AS rn
        FROM orders
    ) sub
    WHERE rn > 1
);
```

---

## 7. Query Execution Order

Understanding the logical evaluation order helps you reason about what's available at each stage.

```
1.  FROM        ← tables and joins
2.  WHERE       ← filter individual rows
3.  GROUP BY    ← partition into groups
4.  HAVING      ← filter groups
5.  SELECT      ← compute output columns, window functions
6.  DISTINCT    ← remove duplicate rows
7.  ORDER BY    ← sort results
8.  LIMIT/OFFSET ← truncate output
```

### Key Implications

| Observation | Explanation |
|-------------|-------------|
| Can't use column aliases in `WHERE` | `WHERE` is evaluated before `SELECT` |
| Can use column aliases in `ORDER BY` | `ORDER BY` is after `SELECT` |
| Can't use window functions in `WHERE` or `HAVING` | Window functions are computed in `SELECT` |
| `HAVING` can use aggregates | `HAVING` is after `GROUP BY` |
| `DISTINCT` sees window function results | `DISTINCT` is after `SELECT` |

```sql
-- This FAILS: alias 'annual' not available in WHERE
SELECT salary * 12 AS annual FROM employees WHERE annual > 100000;

-- Fix: use the expression directly
SELECT salary * 12 AS annual FROM employees WHERE salary * 12 > 100000;

-- Or use a subquery/CTE
WITH t AS (SELECT salary * 12 AS annual FROM employees)
SELECT * FROM t WHERE annual > 100000;
```

```sql
-- This FAILS: window function not allowed in WHERE
SELECT *, ROW_NUMBER() OVER (ORDER BY salary) AS rn
FROM employees
WHERE rn <= 5;

-- Fix: use a subquery
SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (ORDER BY salary) AS rn
    FROM employees
) sub
WHERE rn <= 5;
```

---

## 8. Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `NOT IN` with NULLs | Returns nothing | Use `NOT EXISTS` |
| `WHERE` on outer join's outer table | Converts to inner join | Put condition in `ON` |
| `NATURAL JOIN` | Breaks silently on schema changes | Always explicit `ON` |
| `LAST_VALUE()` without frame | Default frame stops at current row | Add `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` |
| Column alias in `WHERE` | Not visible (evaluated before SELECT) | Use expression or subquery |
| Window function in `WHERE` | Not allowed | Wrap in subquery/CTE |
| `UNION` when `UNION ALL` suffices | Unnecessary dedup sort | Use `UNION ALL` |
| `BETWEEN` for timestamps | End-of-day issues | Use `>= AND <` |
| `SELECT *` with JOINs | Ambiguous if tables share column names | Always qualify with alias |
| Missing `ORDER BY` on `DISTINCT ON` / window | Undefined determinism | Always specify order |
| `GROUP BY` without aggregate | No error but confusing — just use `DISTINCT` | Use `DISTINCT` if no aggregation needed |

---

## 9. What to Learn Next

1. **Schema Design & Normalization** — 1NF through BCNF, denormalization — Phase 3 (Roadmap).
2. **Indexes & Query Performance** — EXPLAIN, B-tree, GIN, partial indexes — see [05-indexes.md](05-indexes.md).
3. **Transactions & Concurrency** — ACID, MVCC, isolation levels — see [01-acid.md](01-acid.md), [02-transactions.md](02-transactions.md).
4. **Full-Text Search** — `tsvector`, `tsquery`, ranking — Phase 12 (Advanced Data Features).

---

> *Ref: [Docs — Queries](https://www.postgresql.org/docs/18/queries.html) · [Docs — Table Expressions / Joins](https://www.postgresql.org/docs/18/queries-table-expressions.html) · [Docs — Aggregate Functions](https://www.postgresql.org/docs/18/functions-aggregate.html) · [Docs — Window Functions](https://www.postgresql.org/docs/18/tutorial-window.html) · [Docs — WITH Queries](https://www.postgresql.org/docs/18/queries-with.html) · [Neon Tutorial](https://neon.com/postgresql/tutorial)*
