# Views & Materialized Views

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 7 — Views & Materialized Views  
> Regular views, materialized views, updatable views, security barrier views, and practical patterns for each.

---

## Table of Contents

1. [Views](#1-views)
   - [Creating a View](#creating-a-view)
   - [How Views Work Internally](#how-views-work-internally)
   - [Benefits of Views](#benefits-of-views)
   - [Security Barrier Views](#security-barrier-views)
   - [Recursive Views](#recursive-views)
   - [Managing Views](#managing-views)
2. [Materialized Views](#2-materialized-views)
   - [Creating a Materialized View](#creating-a-materialized-view)
   - [Refreshing Data](#refreshing-data)
   - [Indexes on Materialized Views](#indexes-on-materialized-views)
   - [Refresh Strategies](#refresh-strategies)
   - [Limitations](#limitations)
3. [Updatable Views](#3-updatable-views)
   - [Auto-Updatable Views](#auto-updatable-views)
   - [WITH CHECK OPTION](#with-check-option)
   - [INSTEAD OF Triggers](#instead-of-triggers)
4. [Views vs Materialized Views](#4-views-vs-materialized-views)
5. [Practical Patterns](#5-practical-patterns)

---

## 1. Views

A **view** is a named query stored in the database catalog. It acts as a **virtual table** — no data is stored. Every time you query a view, PostgreSQL executes the underlying query.

### Creating a View

```sql
CREATE VIEW active_orders AS
SELECT
    o.id,
    o.created_at,
    o.total,
    o.status,
    c.name AS customer_name,
    c.email AS customer_email
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status != 'cancelled';
```

```sql
-- Use it like a table
SELECT * FROM active_orders WHERE total > 1000 ORDER BY created_at DESC;

-- Join with other tables or views
SELECT ao.*, p.name AS product
FROM active_orders ao
JOIN order_items oi ON oi.order_id = ao.id
JOIN products p ON p.id = oi.product_id;
```

### How Views Work Internally

Views are **not pre-computed**. PostgreSQL expands the view definition into the query at parse time:

```sql
-- You write:
SELECT * FROM active_orders WHERE total > 1000;

-- PostgreSQL internally executes:
SELECT o.id, o.created_at, o.total, o.status, c.name, c.email
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status != 'cancelled'
  AND o.total > 1000;   -- your WHERE is merged in
```

The planner optimizes the merged query as a whole — views generally have **no performance overhead** compared to writing the query directly.

### Benefits of Views

| Benefit | Example |
|---------|---------|
| **Simplify complex queries** | Hide multi-table JOINs behind a simple `SELECT * FROM view` |
| **Enforce access control** | Expose only certain columns/rows to specific roles |
| **Decouple application from schema** | Rename underlying columns without breaking app queries (use column aliases in the view) |
| **Reuse logic** | Define business rules once (e.g., "active orders = not cancelled") |
| **Layer security** | Combine with RLS, `SECURITY BARRIER`, and column-level grants |

### Security Barrier Views

A normal view can leak data through **user-defined functions** in the WHERE clause. Security barrier views prevent this.

```sql
-- Problem: leaky view
CREATE VIEW public_users AS
SELECT id, name, email FROM users WHERE role = 'public';

-- An attacker's function that leaks data as a side effect
CREATE FUNCTION leak(text) RETURNS boolean AS $$
BEGIN
    RAISE NOTICE 'Leaked value: %', $1;
    RETURN true;
END;
$$ LANGUAGE plpgsql COST 0.0001;  -- low cost tricks planner into evaluating first

-- This might evaluate leak() on ALL rows, including non-public ones,
-- before applying the view's WHERE filter
SELECT * FROM public_users WHERE leak(email);
```

```sql
-- Fix: security barrier view
CREATE VIEW public_users WITH (security_barrier = true) AS
SELECT id, name, email FROM users WHERE role = 'public';

-- Now PostgreSQL ALWAYS evaluates the view's WHERE clause first,
-- before any user-supplied functions. No data leaks.
```

| Feature | Normal View | Security Barrier View |
|---------|------------|----------------------|
| Evaluation order | Planner decides (may run user functions first) | View's WHERE always evaluated first |
| Performance | Slightly better (planner has full freedom) | Slight overhead (restricted evaluation order) |
| Use with RLS | Not recommended | ✅ Recommended |

> **When to use**: Always use `security_barrier` when the view's purpose is **access control** — hiding rows from certain users.

### Recursive Views

A shorthand for recursive CTEs.

```sql
-- Recursive view for an org hierarchy
CREATE RECURSIVE VIEW org_tree (id, name, manager_id, depth, path) AS
    -- Base case: top-level managers
    SELECT id, name, manager_id, 0, ARRAY[name]
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive case: subordinates
    SELECT e.id, e.name, e.manager_id, t.depth + 1, t.path || e.name
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.id;

SELECT * FROM org_tree ORDER BY path;
```

This is equivalent to:

```sql
CREATE VIEW org_tree AS
WITH RECURSIVE org_tree(id, name, manager_id, depth, path) AS (...)
SELECT * FROM org_tree;
```

### Managing Views

```sql
-- Replace a view (must return same column types, or drop and recreate)
CREATE OR REPLACE VIEW active_orders AS
SELECT ... ;   -- can add columns at the end, but cannot remove or change type

-- Rename
ALTER VIEW active_orders RENAME TO current_orders;

-- Change owner
ALTER VIEW active_orders OWNER TO app_role;

-- Change column names
ALTER VIEW active_orders RENAME COLUMN customer_name TO client_name;

-- Drop
DROP VIEW active_orders;
DROP VIEW IF EXISTS active_orders CASCADE;  -- also drops dependent views

-- List all views
SELECT table_name, view_definition
FROM information_schema.views
WHERE table_schema = 'public';

-- View the definition
SELECT pg_get_viewdef('active_orders'::regclass, true);
```

**View dependency chain**: If view B depends on view A, you cannot drop view A without `CASCADE` (which also drops B). Use `DROP VIEW ... CASCADE` carefully.

---

## 2. Materialized Views

A **materialized view** is a named query whose **result is stored physically on disk**. It's like a snapshot — fast to read, but must be refreshed to reflect changes in the underlying tables.

### Creating a Materialized View

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    date_trunc('month', o.created_at) AS month,
    c.region,
    count(*) AS order_count,
    sum(o.total) AS revenue,
    avg(o.total) AS avg_order_value
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.status = 'completed'
GROUP BY 1, 2
ORDER BY 1 DESC, 2;
```

The query runs **immediately** and stores the results. Subsequent reads hit the stored data, not the underlying tables.

```sql
-- Create without populating (useful when base tables are not ready)
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT ... 
WITH NO DATA;
-- Cannot query until refreshed: SELECT * FROM monthly_revenue → ERROR

-- Populate it later
REFRESH MATERIALIZED VIEW monthly_revenue;
```

### Refreshing Data

Materialized views are **not auto-refreshed**. You must explicitly refresh them.

```sql
-- Full refresh: rewrites all data, acquires ACCESS EXCLUSIVE lock
-- Blocks all reads during refresh
REFRESH MATERIALIZED VIEW monthly_revenue;

-- Concurrent refresh: does NOT block reads
-- Requires a UNIQUE index on the materialized view
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

| Refresh Type | Locks | Reads During Refresh | Requirements |
|-------------|-------|:-------------------:|-------------|
| Regular `REFRESH` | ACCESS EXCLUSIVE | ❌ Blocked | None |
| `REFRESH ... CONCURRENTLY` | EXCLUSIVE (weaker) | ✅ Allowed (reads old data) | UNIQUE index required |

> **Always prefer `CONCURRENTLY`** in production to avoid blocking reads. The trade-off is slightly higher CPU/IO during refresh.

### Indexes on Materialized Views

Materialized views are physically stored tables — you can create **indexes** on them for fast lookups.

```sql
-- Required for CONCURRENTLY refresh
CREATE UNIQUE INDEX idx_monthly_revenue_pk
ON monthly_revenue (month, region);

-- Additional indexes for query patterns
CREATE INDEX idx_monthly_revenue_region
ON monthly_revenue (region);

CREATE INDEX idx_monthly_revenue_month
ON monthly_revenue (month DESC);
```

### Refresh Strategies

| Strategy | How | Best For |
|----------|-----|----------|
| **Cron job** | `cron.schedule('0 * * * *', 'REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue')` using `pg_cron` | Hourly/daily dashboards |
| **Application trigger** | Refresh after batch imports or at end-of-day processing | Event-driven updates |
| **Trigger on base table** | Trigger that calls `REFRESH` after INSERT/UPDATE/DELETE | Small, infrequently updated views (⚠️ expensive for high-write tables) |
| **Lazy refresh** | Track a `last_refreshed_at` timestamp; refresh only when stale and requested | On-demand reporting |

```sql
-- pg_cron example: refresh every hour
SELECT cron.schedule(
    'refresh_monthly_revenue',
    '0 * * * *',
    'REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue'
);

-- Check scheduled jobs
SELECT * FROM cron.job;
```

```sql
-- Lazy refresh pattern with a wrapper function
CREATE OR REPLACE FUNCTION get_monthly_revenue(p_max_age interval DEFAULT '1 hour')
RETURNS SETOF monthly_revenue
LANGUAGE plpgsql
AS $$
DECLARE
    v_last_refresh timestamptz;
BEGIN
    -- Check when the matview was last refreshed
    SELECT last_refresh INTO v_last_refresh
    FROM matview_refresh_log
    WHERE view_name = 'monthly_revenue';

    IF v_last_refresh IS NULL OR v_last_refresh < now() - p_max_age THEN
        REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
        INSERT INTO matview_refresh_log (view_name, last_refresh)
        VALUES ('monthly_revenue', now())
        ON CONFLICT (view_name) DO UPDATE SET last_refresh = now();
    END IF;

    RETURN QUERY SELECT * FROM monthly_revenue;
END;
$$;
```

### Limitations

| Limitation | Detail |
|-----------|--------|
| **No auto-refresh** | You must manage refresh scheduling yourself |
| **Stale data** | Between refreshes, the materialized view may not reflect recent changes |
| **Storage cost** | Stores a full copy of the query result on disk |
| **Full rewrite on refresh** | Even `CONCURRENTLY` rewrites the entire view (no incremental refresh in core PG) |
| **No parameterization** | Cannot pass parameters like a function — the query is fixed |
| **VACUUM required** | Like regular tables, materialized views accumulate dead tuples after refreshes |

> **Incremental refresh**: Not natively supported in PostgreSQL. Third-party extensions like `pg_ivm` (Incremental View Maintenance) provide this for specific use cases.

---

## 3. Updatable Views

### Auto-Updatable Views

PostgreSQL automatically makes a view updatable if it meets these conditions:

| Condition | Requirement |
|-----------|------------|
| Single table | View references exactly **one** table or updatable view in FROM |
| No aggregation | No `GROUP BY`, `HAVING`, `DISTINCT`, `LIMIT`, `OFFSET` |
| No set operations | No `UNION`, `INTERSECT`, `EXCEPT` |
| No window functions | No `OVER()` |
| No subqueries in SELECT | Only simple column references |
| Columns are simple references | No expressions or functions on columns in SELECT |

```sql
-- ✅ Auto-updatable: single table, simple column references
CREATE VIEW pending_orders AS
SELECT id, customer_id, total, status, created_at
FROM orders
WHERE status = 'pending';

-- These all work automatically:
INSERT INTO pending_orders (customer_id, total, status)
VALUES (1, 99.99, 'pending');

UPDATE pending_orders SET total = 149.99 WHERE id = 1;

DELETE FROM pending_orders WHERE id = 1;
```

```sql
-- ❌ NOT auto-updatable: uses JOIN
CREATE VIEW order_details AS
SELECT o.id, o.total, c.name
FROM orders o JOIN customers c ON c.id = o.customer_id;
-- INSERT/UPDATE/DELETE → ERROR

-- ❌ NOT auto-updatable: uses aggregate
CREATE VIEW order_summary AS
SELECT customer_id, count(*), sum(total)
FROM orders GROUP BY customer_id;
-- INSERT/UPDATE/DELETE → ERROR
```

### WITH CHECK OPTION

Prevents inserting or updating rows that **would not be visible** through the view.

```sql
CREATE VIEW pending_orders AS
SELECT id, customer_id, total, status
FROM orders
WHERE status = 'pending'
WITH CHECK OPTION;

-- ✅ Allowed: the new row satisfies status = 'pending'
INSERT INTO pending_orders (customer_id, total, status)
VALUES (1, 99.99, 'pending');

-- ❌ Rejected: status = 'shipped' doesn't satisfy the view's WHERE
INSERT INTO pending_orders (customer_id, total, status)
VALUES (1, 99.99, 'shipped');
-- ERROR: new row violates check option for view "pending_orders"

-- ❌ Rejected: updating status away from 'pending'
UPDATE pending_orders SET status = 'shipped' WHERE id = 1;
-- ERROR: new row violates check option for view "pending_orders"
```

#### LOCAL vs CASCADED

For views defined on top of other views:

```sql
CREATE VIEW all_orders AS
SELECT * FROM orders;

CREATE VIEW pending_orders AS
SELECT * FROM all_orders WHERE status = 'pending'
WITH CASCADED CHECK OPTION;  -- default

CREATE VIEW high_value_pending AS
SELECT * FROM pending_orders WHERE total > 1000
WITH LOCAL CHECK OPTION;
```

| Option | Checks | Behavior |
|--------|--------|----------|
| `WITH LOCAL CHECK OPTION` | Only **this** view's WHERE clause | Allows writes that violate parent views' conditions |
| `WITH CASCADED CHECK OPTION` (default) | **This** view's AND **all parent** views' WHERE clauses | Enforces the full view chain |

### INSTEAD OF Triggers

For views that are **not auto-updatable** (joins, aggregations, etc.), write `INSTEAD OF` triggers to handle DML manually.

```sql
-- Complex view with a JOIN (not auto-updatable)
CREATE VIEW customer_orders AS
SELECT
    c.id AS customer_id,
    c.name AS customer_name,
    c.email,
    o.id AS order_id,
    o.total,
    o.status
FROM customers c
JOIN orders o ON o.customer_id = c.id;

-- Make INSERT work via INSTEAD OF trigger
CREATE OR REPLACE FUNCTION insert_customer_order()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    v_customer_id integer;
BEGIN
    -- Find or create the customer
    SELECT id INTO v_customer_id
    FROM customers WHERE email = NEW.email;

    IF NOT FOUND THEN
        INSERT INTO customers (name, email)
        VALUES (NEW.customer_name, NEW.email)
        RETURNING id INTO v_customer_id;
    END IF;

    -- Create the order
    INSERT INTO orders (customer_id, total, status)
    VALUES (v_customer_id, NEW.total, COALESCE(NEW.status, 'pending'));

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_insert_customer_order
    INSTEAD OF INSERT ON customer_orders
    FOR EACH ROW
    EXECUTE FUNCTION insert_customer_order();

-- Now this works:
INSERT INTO customer_orders (customer_name, email, total)
VALUES ('Alice', 'alice@example.com', 199.99);
```

```sql
-- Make UPDATE work
CREATE OR REPLACE FUNCTION update_customer_order()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    -- Update customer fields
    UPDATE customers SET name = NEW.customer_name, email = NEW.email
    WHERE id = NEW.customer_id;

    -- Update order fields
    UPDATE orders SET total = NEW.total, status = NEW.status
    WHERE id = NEW.order_id;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_update_customer_order
    INSTEAD OF UPDATE ON customer_orders
    FOR EACH ROW
    EXECUTE FUNCTION update_customer_order();
```

---

## 4. Views vs Materialized Views

| Aspect | View | Materialized View |
|--------|------|-------------------|
| **Data storage** | None — virtual query | Physical — stored on disk |
| **Freshness** | Always current | Stale until refreshed |
| **Read speed** | Depends on underlying query complexity | Fast — reads from stored table |
| **Write overhead** | None | Refresh cost (full rewrite) |
| **Indexes** | ❌ Cannot index a view | ✅ Can create indexes |
| **VACUUM needed** | No | Yes |
| **Updatable** | Yes (with conditions) | ❌ No (read-only) |
| **Use case** | Access control, query simplification, API layer | Expensive reports, dashboards, search results |

### Decision Guide

```
Is the query expensive (> 100ms) or frequently executed?
├── No  → Regular VIEW
└── Yes → Can you tolerate stale data?
    ├── No  → Regular VIEW (accept the cost)
    └── Yes → MATERIALIZED VIEW
            └── How often does base data change?
                ├── Rarely → REFRESH on-demand or daily cron
                ├── Moderately → Hourly REFRESH CONCURRENTLY
                └── Constantly → Consider caching layer instead
```

---

## 5. Practical Patterns

### Pattern 1: API Layer — Decouple Schema from Application

```sql
-- Internal schema (can evolve freely)
CREATE TABLE usr (
    usr_id    serial PRIMARY KEY,
    fname     text,
    lname     text,
    eml       text UNIQUE,
    is_active boolean DEFAULT true
);

-- Stable API view (application queries this)
CREATE VIEW users AS
SELECT
    usr_id   AS id,
    fname    AS first_name,
    lname    AS last_name,
    eml      AS email,
    is_active AS active
FROM usr;

-- When you rename a column internally:
-- ALTER TABLE usr RENAME COLUMN eml TO email_address;
-- Just update the view — application code doesn't change
```

### Pattern 2: Dashboard with Materialized View

```sql
-- Expensive aggregation for a dashboard
CREATE MATERIALIZED VIEW dashboard_stats AS
SELECT
    date_trunc('day', o.created_at) AS day,
    count(*) AS orders,
    sum(o.total) AS revenue,
    count(DISTINCT o.customer_id) AS unique_customers,
    avg(o.total) AS avg_order_value
FROM orders o
WHERE o.status = 'completed'
GROUP BY 1;

CREATE UNIQUE INDEX idx_dashboard_stats_day ON dashboard_stats (day);

-- Fast reads for the dashboard
SELECT * FROM dashboard_stats WHERE day >= current_date - 30 ORDER BY day;

-- Schedule hourly refresh
-- pg_cron: SELECT cron.schedule('refresh_dashboard', '0 * * * *',
--   'REFRESH MATERIALIZED VIEW CONCURRENTLY dashboard_stats');
```

### Pattern 3: Row-Level Security with Security Barrier View

```sql
-- Multi-tenant table
CREATE TABLE documents (
    id         serial PRIMARY KEY,
    tenant_id  integer NOT NULL,
    title      text,
    content    text,
    created_at timestamptz DEFAULT now()
);

-- Security barrier view per tenant
CREATE VIEW tenant_documents WITH (security_barrier = true) AS
SELECT id, title, content, created_at
FROM documents
WHERE tenant_id = current_setting('app.tenant_id')::integer;

-- Application sets the tenant context then queries the view
SET app.tenant_id = '42';
SELECT * FROM tenant_documents;  -- only sees tenant 42's documents
```

### Pattern 4: Nested Views with Cascading Check

```sql
-- Base view: all active products
CREATE VIEW active_products AS
SELECT * FROM products WHERE discontinued = false
WITH CASCADED CHECK OPTION;

-- Specialized view: premium products (> $100)
CREATE VIEW premium_products AS
SELECT * FROM active_products WHERE price > 100
WITH CASCADED CHECK OPTION;

-- Insert through premium_products:
-- ✅ Works: discontinued = false AND price > 100
INSERT INTO premium_products (name, price, discontinued)
VALUES ('Luxury Item', 250, false);

-- ❌ Rejected: price <= 100 violates premium_products check
INSERT INTO premium_products (name, price, discontinued)
VALUES ('Cheap Item', 50, false);

-- ❌ Rejected: discontinued = true violates parent active_products check (CASCADED)
INSERT INTO premium_products (name, price, discontinued)
VALUES ('Old Luxury', 250, true);
```

### Pattern 5: Materialized View with Incremental-like Refresh

```sql
-- For append-only data (e.g., event logs), simulate incremental refresh
-- by partitioning time and only refreshing the recent matview

CREATE MATERIALIZED VIEW recent_events AS
SELECT event_type, count(*) AS cnt, date_trunc('hour', created_at) AS hour
FROM events
WHERE created_at >= current_date - 7  -- only last 7 days
GROUP BY 1, 3;

CREATE UNIQUE INDEX idx_recent_events_pk ON recent_events (event_type, hour);

-- Refresh is fast because the underlying query only scans recent data
-- Combine with a historical summary table for older data:
-- SELECT * FROM recent_events
-- UNION ALL
-- SELECT * FROM historical_event_summary WHERE hour < current_date - 7;
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Using matview for real-time data | Stale reads between refreshes | Use a regular view or caching layer |
| `REFRESH` without `CONCURRENTLY` in production | Blocks all reads during refresh | Always use `CONCURRENTLY` (requires unique index) |
| No unique index on matview | Cannot use `REFRESH CONCURRENTLY` | Add a unique index on the primary key columns |
| Forgetting `WITH CHECK OPTION` | Users can insert rows that disappear from the view | Add `WITH CHECK OPTION` to enforce visibility |
| Deeply nested views | Hard to debug and the planner may produce suboptimal plans | Keep view nesting ≤ 2–3 levels; use CTEs or functions for complex logic |
| Using views expecting performance gain | Views are virtual — no speed benefit over the raw query | Use materialized views or indexes for performance |
| Not vacuuming materialized views | Bloat after repeated refreshes | Monitor and vacuum like regular tables |

---

## What to Learn Next

1. **Stored Procedures & Triggers** — Write trigger functions for `INSTEAD OF` triggers — see [19-stored-procedures-functions-triggers.md](19-stored-procedures-functions-triggers.md).
2. **Row-Level Security** — Combine with security barrier views — see [12-policy.md](12-policy.md).
3. **Schema Design** — Materialized views for denormalization — see [18-schema-design-data-modeling.md](18-schema-design-data-modeling.md).
4. **Indexes** — Index materialized views for fast lookups — see [05-indexes.md](05-indexes.md).
5. **Roles & Privileges** — Control who can query/refresh views — see [08-roles.md](08-roles.md).

---

> *Ref: [Docs — CREATE VIEW](https://www.postgresql.org/docs/18/sql-createview.html) · [Docs — CREATE MATERIALIZED VIEW](https://www.postgresql.org/docs/18/sql-creatematerializedview.html) · [Docs — Rules on Views](https://www.postgresql.org/docs/18/rules-views.html) · [Neon — PostgreSQL Tutorial](https://neon.com/postgresql/tutorial)*
