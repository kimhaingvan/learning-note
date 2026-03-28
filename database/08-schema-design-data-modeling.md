# Schema Design & Data Modeling

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 3 (3.1 – 3.5)  
> How to structure your database for correctness, performance, and maintainability — normalization, denormalization, partitioning, schemas, and sequences.

---

## Table of Contents

1. [Normalization](#1-normalization)
   - [Why Normalize?](#11-why-normalize)
   - [Functional Dependencies](#12-functional-dependencies)
   - [1NF — First Normal Form](#13-1nf--first-normal-form)
   - [2NF — Second Normal Form](#14-2nf--second-normal-form)
   - [3NF — Third Normal Form](#15-3nf--third-normal-form)
   - [BCNF — Boyce-Codd Normal Form](#16-bcnf--boyce-codd-normal-form)
   - [Beyond 3NF: 4NF, 5NF](#17-beyond-3nf-4nf-5nf)
   - [When to Stop Normalizing](#18-when-to-stop-normalizing)
2. [Denormalization](#2-denormalization)
   - [Why Denormalize?](#21-why-denormalize)
   - [Denormalization Techniques](#22-denormalization-techniques)
   - [Trade-offs](#23-trade-offs)
3. [Table Inheritance & Partitioning](#3-table-inheritance--partitioning)
   - [Table Inheritance (Legacy)](#31-table-inheritance-legacy)
   - [Declarative Partitioning](#32-declarative-partitioning)
4. [Schemas & Namespaces](#4-schemas--namespaces)
   - [What Is a Schema?](#41-what-is-a-schema)
   - [The search_path](#42-the-search_path)
   - [Schema-Based Multi-Tenancy](#43-schema-based-multi-tenancy)
   - [Schema Best Practices](#44-schema-best-practices)
5. [Sequences & Identity Columns](#5-sequences--identity-columns)
6. [Practical Schema Design Patterns](#6-practical-schema-design-patterns)
   - [One-to-Many](#61-one-to-many)
   - [Many-to-Many](#62-many-to-many)
   - [One-to-One](#63-one-to-one)
   - [Self-Referencing (Hierarchy)](#64-self-referencing-hierarchy)
   - [Polymorphic Associations](#65-polymorphic-associations)
   - [Soft Deletes](#66-soft-deletes)
   - [Audit Columns](#67-audit-columns)
   - [EAV (Entity-Attribute-Value)](#68-eav-entity-attribute-value)
7. [Common Mistakes](#7-common-mistakes)
8. [What to Learn Next](#8-what-to-learn-next)

---

## 1. Normalization

Normalization is the process of organizing tables to **minimize redundancy** and **prevent data anomalies** (insertion, update, deletion anomalies).

### 1.1 Why Normalize?

#### Data Anomalies (The Problem)

Consider a single denormalized table:

```
orders_flat
┌──────────┬───────────┬──────────────┬───────────┬──────────┐
│ order_id │ product   │ product_price│ customer  │ cust_city│
├──────────┼───────────┼──────────────┼───────────┼──────────┤
│ 1        │ Widget    │ 29.99        │ Alice     │ Hanoi    │
│ 2        │ Widget    │ 29.99        │ Bob       │ HCMC     │
│ 3        │ Gadget    │ 49.99        │ Alice     │ Hanoi    │
└──────────┴───────────┴──────────────┴───────────┴──────────┘
```

| Anomaly | What Goes Wrong |
|---------|----------------|
| **Update anomaly** | Changing Widget's price requires updating **every row** that references it. Miss one → inconsistent data. |
| **Insertion anomaly** | Can't add a new product without creating a fake order. |
| **Deletion anomaly** | Deleting order #3 loses the only record that the Gadget product exists. |

#### Normalized Solution

```
products             customers            orders
┌────┬────────┬───┐  ┌────┬───────┬─────┐  ┌────┬─────────┬──────────┐
│ id │ name   │ $ │  │ id │ name  │ city│  │ id │ prod_id │ cust_id  │
├────┼────────┼───┤  ├────┼───────┼─────┤  ├────┼─────────┼──────────┤
│ 1  │ Widget │ 30│  │ 1  │ Alice │ HN  │  │ 1  │ 1       │ 1        │
│ 2  │ Gadget │ 50│  │ 2  │ Bob   │ HCM │  │ 2  │ 1       │ 2        │
└────┴────────┴───┘  └────┴───────┴─────┘  │ 3  │ 2       │ 1        │
                                            └────┴─────────┴──────────┘
```

Now: change a price once in `products`, add products without orders, delete orders without losing products.

### 1.2 Functional Dependencies

Normalization is based on **functional dependencies** — when one column's value determines another.

```
Notation: X → Y  ("X determines Y")
```

| Dependency | Example | Meaning |
|------------|---------|---------|
| `employee_id → name` | ID 42 always maps to "Alice" | Name depends on employee_id |
| `{order_id, product_id} → quantity` | A specific order-product combination determines the quantity | Quantity depends on the composite key |

**Key types**:
- **Full dependency**: `{A, B} → C` and C depends on the **whole** composite key, not just A or B alone.
- **Partial dependency**: `{A, B} → C` but actually A alone determines C. (Violates 2NF)
- **Transitive dependency**: `A → B → C` where A determines B, and B determines C. (Violates 3NF)

### 1.3 1NF — First Normal Form

**Rule**: Each column contains a **single atomic value**. No repeating groups, no arrays-as-CSV.

```sql
-- ❌ Violates 1NF: multiple values in one column
CREATE TABLE orders_bad (
    id serial PRIMARY KEY,
    customer text,
    products text  -- 'Widget, Gadget, Thingamajig'  ← NOT atomic
);

-- ✅ 1NF: separate rows (or a join table)
CREATE TABLE orders (
    id serial PRIMARY KEY,
    customer_id integer REFERENCES customers(id)
);

CREATE TABLE order_items (
    order_id integer REFERENCES orders(id),
    product_id integer REFERENCES products(id),
    quantity integer NOT NULL DEFAULT 1,
    PRIMARY KEY (order_id, product_id)
);
```

> **PostgreSQL nuance**: PostgreSQL supports `text[]` arrays and `jsonb` natively. Storing tags as `text[]` is technically not 1NF, but it's a pragmatic choice when the "nested" data has no independent meaning. The key question: **does the repeated data need to be queried/joined independently?** If yes → normalize. If no → arrays/JSONB can be acceptable.

### 1.4 2NF — Second Normal Form

**Rule**: 1NF + no **partial dependencies**. Every non-key column must depend on the **entire** primary key, not just part of it.

Only relevant for tables with **composite primary keys**.

```sql
-- ❌ Violates 2NF: product_name depends only on product_id, not on (order_id, product_id)
CREATE TABLE order_items_bad (
    order_id integer,
    product_id integer,
    quantity integer,
    product_name text,    -- depends only on product_id (partial dependency)
    PRIMARY KEY (order_id, product_id)
);

-- ✅ 2NF: move product_name to the products table
CREATE TABLE order_items (
    order_id integer REFERENCES orders(id),
    product_id integer REFERENCES products(id),
    quantity integer NOT NULL,
    PRIMARY KEY (order_id, product_id)
);

CREATE TABLE products (
    id serial PRIMARY KEY,
    name text NOT NULL  -- product_name lives here
);
```

### 1.5 3NF — Third Normal Form

**Rule**: 2NF + no **transitive dependencies**. Non-key columns must not depend on other non-key columns.

```sql
-- ❌ Violates 3NF: city depends on zip_code, not directly on employee_id
CREATE TABLE employees_bad (
    id serial PRIMARY KEY,
    name text NOT NULL,
    zip_code text,
    city text          -- id → zip_code → city (transitive)
);

-- ✅ 3NF: extract the transitive dependency
CREATE TABLE employees (
    id serial PRIMARY KEY,
    name text NOT NULL,
    zip_code text REFERENCES zip_codes(code)
);

CREATE TABLE zip_codes (
    code text PRIMARY KEY,
    city text NOT NULL
);
```

### 1.6 BCNF — Boyce-Codd Normal Form

**Rule**: Every **determinant** (left side of a functional dependency) must be a **candidate key**.

BCNF is stricter than 3NF. It matters when there are **overlapping candidate keys**.

```
Example: A table of course schedules
  student_id, subject → teacher      (one teacher per student-subject)
  teacher → subject                   (each teacher teaches one subject)

The second dependency violates BCNF because 'teacher' is a determinant 
but NOT a candidate key (it doesn't uniquely identify a row by itself).

Fix: Decompose into:
  Table 1: teacher, subject        (teacher → subject)
  Table 2: student_id, teacher     (student_id, teacher → row)
```

> **In practice**: Most schemas that satisfy 3NF also satisfy BCNF. The distinction matters in rare cases with overlapping composite keys.

### 1.7 Beyond 3NF: 4NF, 5NF

| Form | Rule | Rare? |
|------|------|-------|
| **4NF** | No multi-valued dependencies. If A determines a set of B values AND a set of C values independently, split into separate tables. | Yes |
| **5NF** | No join dependencies that can't be reconstructed from smaller joins. | Very rare |

> **Practical advice**: 3NF / BCNF is sufficient for almost all real-world OLTP applications. 4NF and 5NF are academic completeness.

### 1.8 When to Stop Normalizing

| Workload | Normal Form | Why |
|----------|-------------|-----|
| OLTP (transactional) | 3NF / BCNF | Write consistency matters more than read speed |
| OLAP (analytics) | 2NF or denormalized | Read speed matters more; data loaded in bulk |
| Reporting / dashboards | Denormalized / star schema | Optimized for aggregate queries |
| Search / caching | Heavily denormalized | Optimized for fast lookups |

---

## 2. Denormalization

Intentionally introducing redundancy to improve **read performance** at the cost of write complexity.

### 2.1 Why Denormalize?

- **JOINs are expensive** at scale — denormalization avoids multi-table joins.
- **Read-heavy workloads** benefit from pre-computed or duplicated data.
- **Reporting queries** that scan millions of rows are faster on wide, flat tables.

> **Rule**: Normalize first, then denormalize **specific** tables/columns with clear justification.

### 2.2 Denormalization Techniques

#### Duplicated Columns

Copy frequently-joined data to avoid the JOIN.

```sql
-- Normalized: need to JOIN orders → employees to get employee_name
CREATE TABLE orders (
    id serial PRIMARY KEY,
    employee_id integer REFERENCES employees(id),
    total numeric(12, 2)
);

-- Denormalized: store the name directly
CREATE TABLE orders (
    id serial PRIMARY KEY,
    employee_id integer REFERENCES employees(id),
    employee_name text NOT NULL,  -- duplicated from employees
    total numeric(12, 2)
);
```

**Maintaining consistency**: Use a trigger or application logic to update `employee_name` when the employee's name changes.

#### Pre-Computed Aggregates

Store computed values instead of recalculating on every query.

```sql
-- Add a counter cache
ALTER TABLE departments ADD COLUMN employee_count integer NOT NULL DEFAULT 0;

-- Maintain with a trigger
CREATE OR REPLACE FUNCTION update_dept_count() RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE departments SET employee_count = employee_count + 1 WHERE id = NEW.department_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE departments SET employee_count = employee_count - 1 WHERE id = OLD.department_id;
    ELSIF TG_OP = 'UPDATE' AND OLD.department_id != NEW.department_id THEN
        UPDATE departments SET employee_count = employee_count - 1 WHERE id = OLD.department_id;
        UPDATE departments SET employee_count = employee_count + 1 WHERE id = NEW.department_id;
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_dept_count
    AFTER INSERT OR DELETE OR UPDATE OF department_id ON employees
    FOR EACH ROW EXECUTE FUNCTION update_dept_count();
```

#### Materialized Views

Pre-compute complex queries and store the result as a table.

```sql
CREATE MATERIALIZED VIEW monthly_revenue AS
SELECT
    DATE_TRUNC('month', created_at) AS month,
    SUM(total) AS revenue,
    COUNT(*) AS order_count
FROM orders
GROUP BY DATE_TRUNC('month', created_at);

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY monthly_revenue;
```

#### JSON Columns for Nested Data

Store related data as `jsonb` to avoid joins.

```sql
CREATE TABLE products (
    id serial PRIMARY KEY,
    name text NOT NULL,
    attributes jsonb NOT NULL DEFAULT '{}'
    -- Instead of a separate attributes table:
    -- {"color": "red", "weight": "1.5kg", "dimensions": "10x20x5"}
);
```

#### Star Schema (Analytics/OLAP)

```
           ┌──────────────┐
           │  dim_date     │
           └──────┬───────┘
                  │
┌──────────┐     │     ┌──────────────┐
│ dim_product├────┤     │ dim_customer  │
└──────────┘     │     └──────┬───────┘
                 │            │
           ┌─────▼────────────▼──┐
           │    fact_sales        │
           │  date_key            │
           │  product_key         │
           │  customer_key        │
           │  quantity            │
           │  revenue             │
           └─────────────────────┘
```

- **Fact table**: stores measurable events (sales, clicks, transactions).
- **Dimension tables**: stores descriptive attributes (who, what, when, where).
- Optimized for aggregation queries — minimal JOINs to dimension tables.

### 2.3 Trade-offs

| Aspect | Normalized | Denormalized |
|--------|-----------|-------------|
| **Data integrity** | Single source of truth | Risk of inconsistency |
| **Write speed** | Fast (one table to update) | Slower (must update copies) |
| **Read speed** | Slower (needs JOINs) | Faster (pre-joined/pre-computed) |
| **Storage** | Less | More (redundant data) |
| **Complexity** | Simple writes, complex reads | Complex writes, simple reads |
| **Best for** | OLTP, write-heavy | OLAP, read-heavy, reporting |

---

## 3. Table Inheritance & Partitioning

### 3.1 Table Inheritance (Legacy)

PostgreSQL's object-relational inheritance feature. Child tables inherit columns from a parent.

```sql
CREATE TABLE vehicles (
    id serial PRIMARY KEY,
    make text NOT NULL,
    model text NOT NULL,
    year integer NOT NULL
);

CREATE TABLE cars (
    doors integer NOT NULL DEFAULT 4
) INHERITS (vehicles);

CREATE TABLE trucks (
    payload_tons numeric NOT NULL
) INHERITS (vehicles);
```

```sql
-- Querying the parent includes child rows
SELECT * FROM vehicles;         -- Returns cars + trucks + vehicles
SELECT * FROM ONLY vehicles;    -- Returns only rows in the parent table itself
```

#### Limitations

- **Unique constraints and foreign keys don't span parent + children** — a `UNIQUE` on `vehicles.id` does NOT prevent duplicates across `cars` and `trucks`.
- **Indexes on the parent don't cover children**.
- **Most modern PostgreSQL usage prefers declarative partitioning** over inheritance.

> **Recommendation**: Avoid table inheritance for new designs. Use **declarative partitioning** (below) for splitting large tables, or standard relational patterns (separate tables + FK) for type hierarchies.

### 3.2 Declarative Partitioning

> **Full coverage**: See [09-partitioning.md](09-partitioning.md) for step-by-step setup, partition pruning, attach/detach patterns, and operational maintenance.

Quick reference:

```sql
-- Range partitioning (most common — time-series data)
CREATE TABLE events (
    id bigint GENERATED ALWAYS AS IDENTITY,
    created_at timestamptz NOT NULL,
    data jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2026_q1 PARTITION OF events
    FOR VALUES FROM ('2026-01-01') TO ('2026-04-01');
CREATE TABLE events_2026_q2 PARTITION OF events
    FOR VALUES FROM ('2026-04-01') TO ('2026-07-01');

-- List partitioning (categorical data)
CREATE TABLE orders (
    id serial,
    region text NOT NULL,
    total numeric
) PARTITION BY LIST (region);

CREATE TABLE orders_apac PARTITION OF orders FOR VALUES IN ('VN', 'TH', 'SG', 'JP');
CREATE TABLE orders_emea PARTITION OF orders FOR VALUES IN ('UK', 'DE', 'FR');

-- Hash partitioning (uniform distribution)
CREATE TABLE sessions (
    id uuid,
    user_id integer,
    data jsonb
) PARTITION BY HASH (user_id);

CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE sessions_2 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE sessions_3 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Default partition (catches unmatched values)
CREATE TABLE events_default PARTITION OF events DEFAULT;
```

#### When to Partition

| Scenario | Partition? | Strategy |
|----------|-----------|----------|
| Time-series data (logs, events, metrics) | **Yes** | `RANGE` by date |
| Multi-region / multi-tenant | Yes | `LIST` by region/tenant |
| Table > 10-100 GB | Consider | Depends on query patterns |
| Small table (< 1 GB) | **No** | Overhead not justified |
| Need to retain/purge by time window | **Yes** | Drop/detach old partitions |
| Uniform load distribution | Yes | `HASH` |

#### Key Rules

- The partition key must be part of any `UNIQUE` or `PRIMARY KEY` constraint.
- Indexes must be created on each partition (or defined on the parent to auto-propagate).
- Foreign keys referencing a partitioned table work since PG 12.
- `PARTITION BY` cannot be added to an existing table — you must recreate and migrate.

---

## 4. Schemas & Namespaces

### 4.1 What Is a Schema?

A schema is a **logical namespace** within a database that groups tables, views, functions, and other objects.

```
Database "myapp"
├── Schema: public          ← default
│   ├── users
│   ├── orders
│   └── products
├── Schema: analytics
│   ├── daily_stats
│   └── monthly_revenue
├── Schema: staging
│   └── raw_imports
└── Schema: audit
    └── event_log
```

> **Analogy**: If a database is a building, schemas are floors. Each floor has its own rooms (tables), but they share the building's infrastructure (roles, connections).

```sql
-- Create a schema
CREATE SCHEMA analytics;

-- Create a table in a specific schema
CREATE TABLE analytics.daily_stats (
    day date NOT NULL,
    metric text NOT NULL,
    value numeric NOT NULL
);

-- Access with qualified name
SELECT * FROM analytics.daily_stats;

-- Drop a schema and all its objects
DROP SCHEMA staging CASCADE;
```

> **Deep dive on databases vs schemas**: See [03-managing-databases.md](03-managing-databases.md) for when to use separate databases vs schemas.

### 4.2 The search_path

Controls which schemas PostgreSQL looks in when you use an **unqualified** table name.

```sql
-- Show current search_path
SHOW search_path;
-- "$user", public     (default — looks in user's schema first, then public)

-- Set search_path for the session
SET search_path TO myapp, public;

-- Now unqualified names are resolved in order: myapp → public
SELECT * FROM users;  -- looks for myapp.users first, then public.users

-- Set per-role default
ALTER ROLE myuser SET search_path TO myapp, public;

-- Set per-database default
ALTER DATABASE mydb SET search_path TO myapp, public;
```

#### How Resolution Works

```
SELECT * FROM users;

1. Check myapp.users    → found? Use it.
2. Check public.users   → found? Use it.
3. Error: relation "users" does not exist.
```

> **Security warning**: If `public` is in `search_path`, any user who can create objects in the `public` schema can shadow your tables. In multi-user environments, revoke `CREATE` on `public` or remove it from the search_path.

```sql
-- Remove public create privilege (recommended for multi-user setups)
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
```

### 4.3 Schema-Based Multi-Tenancy

Each tenant gets its own schema, all within one database.

```sql
-- Create a schema per tenant
CREATE SCHEMA tenant_acme;
CREATE SCHEMA tenant_globex;

-- Each has its own tables
CREATE TABLE tenant_acme.users (id serial PRIMARY KEY, name text);
CREATE TABLE tenant_globex.users (id serial PRIMARY KEY, name text);

-- Route at connection time
SET search_path TO tenant_acme, public;
SELECT * FROM users;  -- queries tenant_acme.users
```

#### Pros and Cons

| Pros | Cons |
|------|------|
| Strong logical isolation | Schema count limited (~10,000 practical) |
| Easy per-tenant backup (`pg_dump -n tenant_x`) | DDL changes must be applied to all schemas |
| `search_path` routing is simple | Hard to run cross-tenant queries |
| Familiar SQL (no tenant_id column everywhere) | Migration tooling must handle N schemas |

#### Alternatives

| Strategy | How | Best For |
|----------|-----|----------|
| **Schema per tenant** | Separate schemas | Small-to-medium tenant count, strong isolation |
| **Row-level (shared table)** | `tenant_id` column + RLS | Large tenant count, operational simplicity |
| **Database per tenant** | Separate databases | Maximum isolation, independent lifecycle |

### 4.4 Schema Best Practices

| Practice | Why |
|----------|-----|
| Use schemas for logical grouping | `public` for app tables, `analytics` for reports, `audit` for logs |
| Revoke `CREATE` on `public` schema | Prevent privilege escalation in multi-user environments |
| Set explicit `search_path` per role | Prevent accidental cross-schema access |
| Don't put application tables in `pg_catalog` or `information_schema` | Reserved for system use |
| Name schemas descriptively | `tenant_acme`, `staging`, `v2` — not `schema1` |

---

## 5. Sequences & Identity Columns

> **Full coverage**: See [11-sequences.md](11-sequences.md) for complete details on `CREATE SEQUENCE`, `nextval`/`currval`/`setval`/`lastval`, `ALTER SEQUENCE`, cross-table sequences, and common traps.

Quick reference for schema design decisions:

| Approach | Syntax | Recommended? |
|----------|--------|-------------|
| `GENERATED ALWAYS AS IDENTITY` | `id integer GENERATED ALWAYS AS IDENTITY` | **Yes** — SQL standard, clear ownership |
| `GENERATED BY DEFAULT AS IDENTITY` | `id integer GENERATED BY DEFAULT AS IDENTITY` | Yes — allows manual override |
| `serial` / `bigserial` | `id serial PRIMARY KEY` | Legacy — still works, widely used |
| Explicit sequence | `CREATE SEQUENCE` + `DEFAULT nextval()` | When sharing across tables |
| `uuid` + `uuidv7()` | `id uuid DEFAULT uuidv7()` | For distributed systems (PG18) |

### Key Points for Schema Design

```sql
-- Preferred: identity column (modern PostgreSQL)
CREATE TABLE users (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL
);

-- Override identity for data migration
INSERT INTO users OVERRIDING SYSTEM VALUE (id, name) VALUES (999, 'Migrated User');

-- Shared sequence across multiple tables
CREATE SEQUENCE global_id_seq;

CREATE TABLE posts (
    id bigint PRIMARY KEY DEFAULT nextval('global_id_seq'),
    title text
);
CREATE TABLE comments (
    id bigint PRIMARY KEY DEFAULT nextval('global_id_seq'),
    body text
);
```

### Gaps Are Normal

Sequences **never guarantee gap-free numbering**. Gaps are created by:
- Transaction rollbacks
- Server crashes
- Cached values (`CACHE` option)
- Deleted rows

> If you need gap-free numbering (invoices, receipts), use a **transactional counter table** with row locking — not a sequence.

---

## 6. Practical Schema Design Patterns

### 6.1 One-to-Many

The most common relationship. A parent row has many child rows.

```sql
-- One department → many employees
CREATE TABLE departments (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL UNIQUE
);

CREATE TABLE employees (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL,
    department_id integer NOT NULL REFERENCES departments(id) ON DELETE RESTRICT
);

CREATE INDEX idx_employees_dept ON employees(department_id);
```

### 6.2 Many-to-Many

Use a **junction table** (also called join table, bridge table, or association table).

```sql
-- Students ↔ Courses (many-to-many)
CREATE TABLE students (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL
);

CREATE TABLE courses (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title text NOT NULL
);

CREATE TABLE enrollments (
    student_id integer REFERENCES students(id) ON DELETE CASCADE,
    course_id integer REFERENCES courses(id) ON DELETE CASCADE,
    enrolled_at timestamptz NOT NULL DEFAULT NOW(),
    grade text,
    PRIMARY KEY (student_id, course_id)
);

-- Index for querying "which students are in this course?"
CREATE INDEX idx_enrollments_course ON enrollments(course_id);
```

### 6.3 One-to-One

Typically: split a table to isolate rarely-accessed or sensitive columns.

```sql
-- Core user data vs extended profile (loaded separately)
CREATE TABLE users (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email text NOT NULL UNIQUE,
    name text NOT NULL
);

CREATE TABLE user_profiles (
    user_id integer PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    bio text,
    avatar_url text,
    settings jsonb NOT NULL DEFAULT '{}'
);
```

> The `PRIMARY KEY` on `user_profiles.user_id` (instead of a separate `id`) enforces the 1:1 relationship.

### 6.4 Self-Referencing (Hierarchy)

```sql
-- Adjacency list (simplest hierarchy)
CREATE TABLE categories (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL,
    parent_id integer REFERENCES categories(id) ON DELETE CASCADE
);

CREATE INDEX idx_categories_parent ON categories(parent_id);
```

#### Querying the Hierarchy

```sql
-- Full tree with depth using recursive CTE
WITH RECURSIVE tree AS (
    SELECT id, name, parent_id, 0 AS depth, name::text AS path
    FROM categories WHERE parent_id IS NULL
    UNION ALL
    SELECT c.id, c.name, c.parent_id, t.depth + 1, t.path || ' > ' || c.name
    FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree ORDER BY path;
```

#### Alternative Hierarchy Patterns

| Pattern | Description | Read | Write | Best For |
|---------|-------------|------|-------|----------|
| **Adjacency list** | `parent_id` FK | Recursive CTE | Simple | Dynamic trees, small depth |
| **Materialized path** | `path text` (e.g., `'/1/3/7/'`) | `LIKE '/1/3/%'` | Update all descendants | Read-heavy, breadcrumbs |
| **Nested set** | `lft` + `rgt` integers | Range query | Expensive (rebalance) | Static trees, rare writes |
| **Closure table** | Separate table listing all ancestor-descendant pairs | Simple JOIN | Insert + N rows | Frequent subtree queries |
| **ltree extension** | `ltree` type | Built-in operators | Built-in | PostgreSQL-specific, powerful |

### 6.5 Polymorphic Associations

When multiple tables reference the "same" target differently.

```sql
-- ❌ Anti-pattern: nullable FKs (one NULL column per row)
CREATE TABLE comments (
    id serial PRIMARY KEY,
    body text NOT NULL,
    post_id integer REFERENCES posts(id),       -- one of these will be NULL
    photo_id integer REFERENCES photos(id),     -- on every row
    video_id integer REFERENCES videos(id),
    CHECK (
        (post_id IS NOT NULL)::int +
        (photo_id IS NOT NULL)::int +
        (video_id IS NOT NULL)::int = 1
    )
);

-- ✅ Better: separate join tables
CREATE TABLE post_comments (
    comment_id integer PRIMARY KEY REFERENCES comments(id),
    post_id integer NOT NULL REFERENCES posts(id)
);
CREATE TABLE photo_comments (
    comment_id integer PRIMARY KEY REFERENCES comments(id),
    photo_id integer NOT NULL REFERENCES photos(id)
);

-- ✅ Also acceptable: generic reference (trade-off: no FK enforcement)
CREATE TABLE comments (
    id serial PRIMARY KEY,
    body text NOT NULL,
    commentable_type text NOT NULL,   -- 'post', 'photo', 'video'
    commentable_id integer NOT NULL
);
CREATE INDEX idx_comments_target ON comments(commentable_type, commentable_id);
```

### 6.6 Soft Deletes

Mark rows as deleted instead of physically removing them.

```sql
CREATE TABLE users (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email text NOT NULL UNIQUE,
    name text NOT NULL,
    deleted_at timestamptz  -- NULL = active, non-null = soft-deleted
);

-- Partial index: only active users (most queries filter on this)
CREATE INDEX idx_users_email_active ON users(email) WHERE deleted_at IS NULL;

-- Application always adds: WHERE deleted_at IS NULL
SELECT * FROM users WHERE deleted_at IS NULL AND email = 'alice@example.com';
```

> **Alternative**: Use a `boolean` column (`is_deleted`), but `deleted_at` gives you a timestamp for auditing and retention policies.

### 6.7 Audit Columns

Standard columns for tracking row lifecycle.

```sql
CREATE TABLE any_table (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    -- ... business columns ...
    created_at timestamptz NOT NULL DEFAULT NOW(),
    updated_at timestamptz NOT NULL DEFAULT NOW(),
    created_by text,   -- or integer REFERENCES users(id)
    updated_by text
);

-- Auto-update updated_at with a trigger
CREATE OR REPLACE FUNCTION set_updated_at() RETURNS trigger AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_set_updated_at
    BEFORE UPDATE ON any_table
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

### 6.8 EAV (Entity-Attribute-Value)

A flexible but problematic pattern. Usually avoid it.

```sql
-- ❌ EAV: every attribute is a row (not a column)
CREATE TABLE properties (
    entity_id integer,
    attribute_name text,
    attribute_value text,  -- everything stored as text — type safety lost
    PRIMARY KEY (entity_id, attribute_name)
);

-- Query for a specific attribute requires pivot:
SELECT entity_id,
    MAX(CASE WHEN attribute_name = 'color' THEN attribute_value END) AS color,
    MAX(CASE WHEN attribute_name = 'size' THEN attribute_value END) AS size
FROM properties
GROUP BY entity_id;
```

> **Why EAV is an anti-pattern**: No type safety, no constraints, complex queries, terrible performance at scale. Instead use **jsonb** columns for flexible attributes, or normal columns + ALTER TABLE for known attributes.

---

## 7. Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| No normalization (giant flat tables) | Update anomalies, data inconsistency | Normalize to at least 3NF for OLTP |
| Over-normalization | Excessive JOINs hurt read performance | Denormalize with materialized views or duplicated columns |
| EAV pattern for known attributes | No type safety, impossible to index | Use normal columns or `jsonb` |
| No index on FK columns | Slow JOINs and cascading deletes | Always create indexes on FK columns |
| Table inheritance for data modeling | Broken FK/UNIQUE across hierarchy | Use normal tables + FK, or partitioning |
| Nullable FK everywhere for polymorphism | Sparse nulls, complex CHECK constraints | Use separate join tables or generic reference |
| `public` schema open to all (`CREATE`) | Privilege escalation attack vector | Revoke `CREATE ON SCHEMA public FROM PUBLIC` |
| Forgot `search_path` security | Shadow table attacks | Set explicit `search_path` per role |
| Using sequences for gap-free numbering | Gaps are normal and expected | Use transactional counter table |

---

## 8. What to Learn Next

1. **Indexes & Query Performance** — How to make your schema fast — see [05-indexes.md](05-indexes.md) and Phase 4 in the roadmap.
2. **Transactions & Concurrency** — ACID, MVCC, isolation levels — see [01-acid.md](01-acid.md), [02-transactions.md](02-transactions.md).
3. **Partitioning (deep dive)** — Step-by-step setup, pruning, attach/detach — see [09-partitioning.md](09-partitioning.md).
4. **Sequences (deep dive)** — `nextval`, `currval`, `setval`, traps — see [11-sequences.md](11-sequences.md).
5. **Row-Level Security** — For multi-tenant designs — see [12-policy.md](12-policy.md) and Phase 8.

---

> *Ref: [Docs — DDL](https://www.postgresql.org/docs/18/ddl.html) · [Docs — Schemas](https://www.postgresql.org/docs/18/ddl-schemas.html) · [Docs — Partitioning](https://www.postgresql.org/docs/18/ddl-partitioning.html) · [Neon Tutorial](https://neon.com/postgresql/tutorial) · [Neon Administration](https://neon.com/postgresql/postgresql-administration)*
