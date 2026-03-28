# Table Basics & Constraints

> **Level**: Beginner–Intermediate | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 1.4  
> Creating tables with proper constraints is the foundation of data integrity. This covers `CREATE TABLE`, all constraint types, `ALTER TABLE`, temporary tables, and best practices.

---

## Table of Contents

1. [CREATE TABLE](#1-create-table)
2. [PRIMARY KEY](#2-primary-key)
3. [FOREIGN KEY](#3-foreign-key)
4. [UNIQUE Constraint](#4-unique-constraint)
5. [NOT NULL Constraint](#5-not-null-constraint)
6. [CHECK Constraint](#6-check-constraint)
7. [DEFAULT Values](#7-default-values)
8. [EXCLUDE Constraint](#8-exclude-constraint)
9. [ALTER TABLE](#9-alter-table)
10. [DROP TABLE](#10-drop-table)
11. [Temporary Tables](#11-temporary-tables)
12. [Unlogged Tables](#12-unlogged-tables)
13. [Table Naming & Design Conventions](#13-table-naming--design-conventions)
14. [Constraint Naming Conventions](#14-constraint-naming-conventions)
15. [Practical Example: Complete Table Design](#15-practical-example-complete-table-design)
16. [Common Mistakes](#16-common-mistakes)
17. [What to Learn Next](#17-what-to-learn-next)

---

## 1. CREATE TABLE

### Basic Syntax

```sql
CREATE TABLE table_name (
    column_name data_type [constraints],
    column_name data_type [constraints],
    ...
    [table_constraints]
);
```

### Simple Example

```sql
CREATE TABLE employees (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    first_name text NOT NULL,
    last_name text NOT NULL,
    email text NOT NULL UNIQUE,
    salary numeric(10, 2) CHECK (salary > 0),
    department_id integer REFERENCES departments(id),
    hired_at timestamptz NOT NULL DEFAULT NOW(),
    active boolean NOT NULL DEFAULT true
);
```

### Column Constraints vs Table Constraints

Constraints can be defined in two places:

```sql
-- Column constraint (inline with the column)
CREATE TABLE users (
    email text NOT NULL UNIQUE   -- constraint on one column
);

-- Table constraint (after all columns — required for multi-column constraints)
CREATE TABLE order_items (
    order_id integer NOT NULL,
    product_id integer NOT NULL,
    quantity integer NOT NULL CHECK (quantity > 0),
    PRIMARY KEY (order_id, product_id)  -- composite PK, must be table-level
);
```

### CREATE TABLE IF NOT EXISTS

```sql
-- No error if the table already exists
CREATE TABLE IF NOT EXISTS logs (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    message text NOT NULL,
    created_at timestamptz NOT NULL DEFAULT NOW()
);
```

### CREATE TABLE AS (from a query)

```sql
-- Create a new table from a SELECT result
CREATE TABLE active_users AS
SELECT id, name, email
FROM users
WHERE active = true;

-- Note: constraints and indexes are NOT copied — only the data and column types.
```

### CREATE TABLE LIKE (copy structure)

```sql
-- Copy column definitions from another table
CREATE TABLE users_archive (LIKE users INCLUDING ALL);
-- INCLUDING ALL copies: defaults, constraints, indexes, comments, storage settings
-- WITHOUT constraints: CREATE TABLE users_archive (LIKE users INCLUDING DEFAULTS);
```

---

## 2. PRIMARY KEY

Uniquely identifies each row in a table. Every table should have a primary key.

### Properties

- Combines `UNIQUE` + `NOT NULL`.
- Automatically creates a unique B-tree index.
- Only **one** primary key per table (but it can span multiple columns).

### Single-Column Primary Key

```sql
-- Preferred: identity column
CREATE TABLE users (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    email text NOT NULL
);

-- UUID primary key (for distributed systems)
CREATE TABLE events (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    type text NOT NULL
);
```

### Composite Primary Key

```sql
CREATE TABLE order_items (
    order_id integer NOT NULL REFERENCES orders(id),
    product_id integer NOT NULL REFERENCES products(id),
    quantity integer NOT NULL DEFAULT 1,
    PRIMARY KEY (order_id, product_id)
);
```

### Natural Key vs Surrogate Key

| Type | Description | Example | Pros | Cons |
|------|-------------|---------|------|------|
| **Surrogate** | System-generated, meaningless ID | `id serial` / `id uuid` | Stable, compact, no business logic coupling | Extra column |
| **Natural** | Business-meaningful value | `email`, `isbn`, `country_code` | No extra column, self-documenting | Can change, may be large |

> **Recommendation**: Use surrogate keys (`integer IDENTITY` or `uuid`) for most tables. Add unique constraints on natural keys.

---

## 3. FOREIGN KEY

Enforces referential integrity — ensures a value in one table references an existing row in another table.

### Basic Foreign Key

```sql
CREATE TABLE departments (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL UNIQUE
);

CREATE TABLE employees (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL,
    department_id integer REFERENCES departments(id)
);
```

### Referential Actions

What happens when the referenced row is deleted or its key is updated?

```sql
CREATE TABLE employees (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL,
    department_id integer REFERENCES departments(id)
        ON DELETE SET NULL        -- Set to NULL when department is deleted
        ON UPDATE CASCADE         -- Update FK when department ID changes
);
```

| Action | ON DELETE | ON UPDATE |
|--------|-----------|-----------|
| `CASCADE` | Delete the child row too | Update the FK value to match |
| `SET NULL` | Set FK column to NULL | Set FK column to NULL |
| `SET DEFAULT` | Set FK column to its DEFAULT | Set FK column to its DEFAULT |
| `RESTRICT` | Block deletion if children exist (checked immediately) | Block update |
| `NO ACTION` (default) | Block deletion if children exist (checked at end of statement) | Block update |

### RESTRICT vs NO ACTION

Both prevent the operation, but timing differs:

- **RESTRICT** — Checks immediately. Fails even if a later trigger would fix the violation.
- **NO ACTION** — Checks at end of statement. Allows triggers to fix violations first.

> In practice, `NO ACTION` (the default) is almost always what you want.

### Named Foreign Key

```sql
CREATE TABLE employees (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL,
    department_id integer,
    CONSTRAINT fk_employees_department
        FOREIGN KEY (department_id) REFERENCES departments(id)
        ON DELETE SET NULL
);
```

### Multi-Column Foreign Key

```sql
CREATE TABLE line_items (
    id serial PRIMARY KEY,
    order_id integer NOT NULL,
    product_id integer NOT NULL,
    quantity integer NOT NULL,
    FOREIGN KEY (order_id, product_id) REFERENCES order_items(order_id, product_id)
);
```

### Self-Referencing Foreign Key

```sql
CREATE TABLE employees (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name text NOT NULL,
    manager_id integer REFERENCES employees(id) ON DELETE SET NULL
);
```

### Performance Note

> **Important**: PostgreSQL does **not** automatically create an index on foreign key columns. You should almost always create one manually:

```sql
CREATE INDEX idx_employees_department_id ON employees(department_id);
```

Without this index, cascading deletes and joins on the FK column will require sequential scans.

---

## 4. UNIQUE Constraint

Ensures no two rows have the same value(s) in the specified columns.

```sql
-- Single column
CREATE TABLE users (
    id serial PRIMARY KEY,
    email text NOT NULL UNIQUE
);

-- Multi-column unique (the combination must be unique)
CREATE TABLE enrollments (
    student_id integer NOT NULL,
    course_id integer NOT NULL,
    enrolled_at timestamptz NOT NULL DEFAULT NOW(),
    UNIQUE (student_id, course_id)
);
```

### NULLs in UNIQUE Constraints

```sql
-- PostgreSQL allows multiple NULLs in a UNIQUE column (NULL ≠ NULL)
INSERT INTO users (email) VALUES (NULL);  -- OK
INSERT INTO users (email) VALUES (NULL);  -- OK (second NULL is allowed!)

-- To prevent this, combine UNIQUE with NOT NULL
```

### UNIQUE vs UNIQUE INDEX

```sql
-- These are functionally identical:
ALTER TABLE users ADD CONSTRAINT uq_email UNIQUE (email);
CREATE UNIQUE INDEX uq_email ON users (email);

-- The index form allows more control (partial, expression, INCLUDE)
CREATE UNIQUE INDEX uq_active_email ON users (email) WHERE active = true;
```

---

## 5. NOT NULL Constraint

Prevents NULL values in a column.

```sql
CREATE TABLE users (
    id serial PRIMARY KEY,
    email text NOT NULL,    -- required
    nickname text            -- optional (NULL allowed)
);
```

### When to Use NOT NULL

Use `NOT NULL` aggressively — make columns nullable only when NULL has a clear business meaning.

| Do use NOT NULL | Don't use NOT NULL |
|----------------|-------------------|
| `email text NOT NULL` | `deleted_at timestamptz` (NULL = not deleted) |
| `name text NOT NULL` | `manager_id integer` (NULL = no manager) |
| `active boolean NOT NULL DEFAULT true` | `notes text` (optional free-form) |

> **Key insight**: In PostgreSQL 18, `NOT NULL` constraints can be properly inherited across partition hierarchies — see [Neon PG18 Features](https://neon.com/postgresql/postgresql-18-new-features).

---

## 6. CHECK Constraint

Validates column values against a boolean expression.

### Column-Level CHECK

```sql
CREATE TABLE products (
    id serial PRIMARY KEY,
    name text NOT NULL,
    price numeric(10, 2) NOT NULL CHECK (price > 0),
    discount numeric(3, 2) CHECK (discount >= 0 AND discount <= 1),
    stock integer NOT NULL DEFAULT 0 CHECK (stock >= 0)
);
```

### Table-Level CHECK (Multi-Column)

```sql
CREATE TABLE events (
    id serial PRIMARY KEY,
    name text NOT NULL,
    starts_at timestamptz NOT NULL,
    ends_at timestamptz NOT NULL,
    CONSTRAINT chk_event_dates CHECK (ends_at > starts_at)
);
```

### Named CHECK

```sql
CREATE TABLE employees (
    salary numeric(10, 2) NOT NULL,
    CONSTRAINT chk_salary_positive CHECK (salary > 0)
);
-- Named constraints produce clearer error messages on violation
```

### CHECK vs Enum vs Domain

| Approach | Add/Remove values | Reusable | Type safety |
|----------|-------------------|----------|-------------|
| `CHECK (status IN ('a','b','c'))` | Easy (ALTER TABLE) | No | Weak (any text accepted at compile time) |
| `ENUM type` | Hard (can add, can't remove) | Yes | Strong |
| `DOMAIN` type | Easy (ALTER DOMAIN) | Yes | Strong |

---

## 7. DEFAULT Values

Provides a value when none is specified during INSERT.

```sql
CREATE TABLE articles (
    id integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    title text NOT NULL,
    status text NOT NULL DEFAULT 'draft',
    view_count integer NOT NULL DEFAULT 0,
    created_at timestamptz NOT NULL DEFAULT NOW(),
    updated_at timestamptz NOT NULL DEFAULT NOW(),
    metadata jsonb NOT NULL DEFAULT '{}',
    tags text[] NOT NULL DEFAULT '{}'
);

-- INSERT without specifying defaults
INSERT INTO articles (title) VALUES ('My First Post');
-- id=1, status='draft', view_count=0, created_at=<now>, metadata='{}', tags='{}'
```

### Expression Defaults

```sql
CREATE TABLE orders (
    id uuid PRIMARY KEY DEFAULT uuidv7(),                    -- Function call
    order_number text DEFAULT 'ORD-' || to_char(NOW(), 'YYYYMMDD'),  -- Expression
    created_at timestamptz DEFAULT CURRENT_TIMESTAMP         -- SQL standard
);
```

> **Tip**: Defaults are evaluated at INSERT time, not at table creation time. `NOW()` gives the current time of each INSERT, not the time the table was created.

---

## 8. EXCLUDE Constraint

A generalized uniqueness constraint — prevents rows from having overlapping or conflicting values. Requires a GiST or SP-GiST index.

### Use Case: Prevent Overlapping Bookings

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE room_bookings (
    id serial PRIMARY KEY,
    room text NOT NULL,
    during tstzrange NOT NULL,
    CONSTRAINT no_overlapping_bookings
        EXCLUDE USING gist (room WITH =, during WITH &&)
);

-- This succeeds
INSERT INTO room_bookings (room, during)
VALUES ('Room A', '[2026-03-27 09:00, 2026-03-27 10:00)');

-- This fails: overlaps with Room A's existing booking
INSERT INTO room_bookings (room, during)
VALUES ('Room A', '[2026-03-27 09:30, 2026-03-27 11:00)');
-- ERROR: conflicting key value violates exclusion constraint

-- This succeeds: different room
INSERT INTO room_bookings (room, during)
VALUES ('Room B', '[2026-03-27 09:30, 2026-03-27 11:00)');
```

### How It Reads

`EXCLUDE USING gist (room WITH =, during WITH &&)` means:
> "Exclude any new row where `room` **equals** an existing room AND `during` **overlaps** with an existing time range."

---

## 9. ALTER TABLE

Modify an existing table's structure.

### Add a Column

```sql
ALTER TABLE employees ADD COLUMN phone text;
ALTER TABLE employees ADD COLUMN age integer CHECK (age >= 18);
```

### Drop a Column

```sql
ALTER TABLE employees DROP COLUMN phone;
ALTER TABLE employees DROP COLUMN IF EXISTS phone;  -- no error if missing
ALTER TABLE employees DROP COLUMN department_id CASCADE;  -- drops dependent objects
```

### Rename

```sql
ALTER TABLE employees RENAME TO staff;
ALTER TABLE staff RENAME COLUMN first_name TO given_name;
```

### Change Data Type

```sql
ALTER TABLE employees ALTER COLUMN salary TYPE numeric(12, 2);

-- With explicit conversion
ALTER TABLE employees ALTER COLUMN age TYPE integer USING age::integer;
```

### Add/Drop Constraints

```sql
-- Add a constraint
ALTER TABLE employees ADD CONSTRAINT chk_salary CHECK (salary > 0);
ALTER TABLE employees ADD UNIQUE (email);
ALTER TABLE employees ALTER COLUMN email SET NOT NULL;

-- Drop a constraint
ALTER TABLE employees DROP CONSTRAINT chk_salary;
ALTER TABLE employees ALTER COLUMN email DROP NOT NULL;
```

### Set/Drop Default

```sql
ALTER TABLE employees ALTER COLUMN active SET DEFAULT true;
ALTER TABLE employees ALTER COLUMN active DROP DEFAULT;
```

### Important Locking Behavior

| Operation | Lock Level | Blocks Reads? | Blocks Writes? |
|-----------|-----------|---------------|----------------|
| `ADD COLUMN` (no default/not null) | `ACCESS EXCLUSIVE` | Very briefly | Very briefly |
| `ADD COLUMN` with `DEFAULT` (PG 11+) | `ACCESS EXCLUSIVE` | Very briefly | Very briefly |
| `DROP COLUMN` | `ACCESS EXCLUSIVE` | Very briefly | Very briefly |
| `ALTER TYPE` (requires rewrite) | `ACCESS EXCLUSIVE` | Yes (rewrites table) | Yes |
| `ADD/DROP CONSTRAINT` (CHECK) | `ACCESS EXCLUSIVE` | Very briefly | Very briefly |
| `ADD CONSTRAINT FOREIGN KEY NOT VALID` | `SHARE ROW EXCLUSIVE` | No | Partial |
| `VALIDATE CONSTRAINT` | `SHARE UPDATE EXCLUSIVE` | No | No |

> **Best practice for large tables**: Add foreign keys as `NOT VALID`, then `VALIDATE CONSTRAINT` separately to avoid long locks.

```sql
-- Two-step FK addition (minimal locking on large tables)
ALTER TABLE employees ADD CONSTRAINT fk_dept
    FOREIGN KEY (department_id) REFERENCES departments(id) NOT VALID;

-- Runs a scan but only takes a lightweight lock
ALTER TABLE employees VALIDATE CONSTRAINT fk_dept;
```

---

## 10. DROP TABLE

Remove a table and its data permanently.

```sql
DROP TABLE employees;

-- No error if the table doesn't exist
DROP TABLE IF EXISTS employees;

-- Drop dependent objects too (views, FK references)
DROP TABLE departments CASCADE;

-- Drop multiple tables
DROP TABLE employees, departments, orders;
```

> **Warning**: `DROP TABLE` is **irreversible** (unless inside a transaction). Always use `IF EXISTS` in scripts.

---

## 11. Temporary Tables

Session-scoped tables that are automatically dropped when the session ends.

```sql
-- Visible only to the current session
CREATE TEMPORARY TABLE temp_results (
    id integer,
    score numeric
);

-- Shorthand
CREATE TEMP TABLE temp_results (id integer, score numeric);

-- Dropped automatically when the session ends (or ON COMMIT)
CREATE TEMP TABLE temp_results (id integer)
    ON COMMIT DROP;          -- Drop at end of transaction
    -- ON COMMIT DELETE ROWS; -- Keep table, truncate data at commit
    -- ON COMMIT PRESERVE ROWS; -- Keep everything (default)
```

### Properties

- Created in a special `pg_temp` schema — invisible to other sessions.
- Not WAL-logged (faster writes, but not crash-safe).
- Indexes and constraints work normally.
- Temporary tables **shadow** regular tables with the same name in the current session.

> **Use case**: Intermediate computation results, staging data for batch processing.

---

## 12. Unlogged Tables

Persistent tables that skip WAL logging for performance.

```sql
CREATE UNLOGGED TABLE cache_entries (
    key text PRIMARY KEY,
    value jsonb NOT NULL,
    expires_at timestamptz NOT NULL
);
```

### Properties

| Feature | Logged Table | Unlogged Table |
|---------|------------|----------------|
| WAL logging | Yes | No |
| Crash recovery | Survives crashes | **Truncated** on crash recovery |
| Replication | Replicated to standbys | **Not replicated** |
| Write speed | Normal | 2-5x faster |
| Persistence | Full | Survives normal restarts, NOT crashes |

> **Use cases**: Caches, session stores, staging tables, materialized temp data. Never for critical data.

---

## 13. Table Naming & Design Conventions

| Convention | Example | Notes |
|------------|---------|-------|
| **Lowercase** | `users`, `order_items` | PostgreSQL folds unquoted names to lowercase |
| **snake_case** | `created_at`, `first_name` | Consistent with PostgreSQL system columns |
| **Plural table names** | `users`, `orders` | Common convention (some prefer singular) |
| **Singular type names** | `order_status`, `user_role` | For enum and domain types |
| **Explicit PK column** | `id` or `user_id` | `id` is most common |
| **FK matches referenced column** | `department_id` → `departments.id` | Self-documenting |
| **Timestamps** | `created_at`, `updated_at`, `deleted_at` | Consistent `_at` suffix for timestamps |

> **Avoid**: CamelCase names (require double-quoting), reserved words as names, spaces in identifiers.

---

## 14. Constraint Naming Conventions

Always name your constraints. PostgreSQL auto-generates names, but explicit names are clearer in error messages and migrations.

| Constraint Type | Convention | Example |
|----------------|-----------|---------|
| Primary key | `pk_{table}` | `pk_users` |
| Foreign key | `fk_{table}_{column}` | `fk_employees_department_id` |
| Unique | `uq_{table}_{column(s)}` | `uq_users_email` |
| Check | `chk_{table}_{description}` | `chk_products_price_positive` |
| Exclusion | `excl_{table}_{description}` | `excl_bookings_no_overlap` |

```sql
CREATE TABLE users (
    id integer GENERATED ALWAYS AS IDENTITY,
    email text NOT NULL,
    CONSTRAINT pk_users PRIMARY KEY (id),
    CONSTRAINT uq_users_email UNIQUE (email)
);
```

---

## 15. Practical Example: Complete Table Design

A realistic multi-table schema demonstrating all concepts:

```sql
-- Enable required extensions
CREATE EXTENSION IF NOT EXISTS btree_gist;

-- Enum type for order status
CREATE TYPE order_status AS ENUM ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled');

-- Users table
CREATE TABLE users (
    id integer GENERATED ALWAYS AS IDENTITY,
    email text NOT NULL,
    display_name text NOT NULL,
    active boolean NOT NULL DEFAULT true,
    created_at timestamptz NOT NULL DEFAULT NOW(),
    updated_at timestamptz NOT NULL DEFAULT NOW(),

    CONSTRAINT pk_users PRIMARY KEY (id),
    CONSTRAINT uq_users_email UNIQUE (email),
    CONSTRAINT chk_users_email_format CHECK (email ~* '^.+@.+\..+$')
);

-- Products table
CREATE TABLE products (
    id integer GENERATED ALWAYS AS IDENTITY,
    name text NOT NULL,
    price numeric(10, 2) NOT NULL,
    stock integer NOT NULL DEFAULT 0,

    CONSTRAINT pk_products PRIMARY KEY (id),
    CONSTRAINT chk_products_price CHECK (price > 0),
    CONSTRAINT chk_products_stock CHECK (stock >= 0)
);

-- Orders table
CREATE TABLE orders (
    id uuid PRIMARY KEY DEFAULT uuidv7(),
    user_id integer NOT NULL,
    status order_status NOT NULL DEFAULT 'pending',
    total numeric(12, 2) NOT NULL DEFAULT 0,
    notes text,
    created_at timestamptz NOT NULL DEFAULT NOW(),
    updated_at timestamptz NOT NULL DEFAULT NOW(),

    CONSTRAINT fk_orders_user FOREIGN KEY (user_id)
        REFERENCES users(id) ON DELETE RESTRICT,
    CONSTRAINT chk_orders_total CHECK (total >= 0)
);

-- Order items (composite PK)
CREATE TABLE order_items (
    order_id uuid NOT NULL,
    product_id integer NOT NULL,
    quantity integer NOT NULL DEFAULT 1,
    unit_price numeric(10, 2) NOT NULL,

    CONSTRAINT pk_order_items PRIMARY KEY (order_id, product_id),
    CONSTRAINT fk_order_items_order FOREIGN KEY (order_id)
        REFERENCES orders(id) ON DELETE CASCADE,
    CONSTRAINT fk_order_items_product FOREIGN KEY (product_id)
        REFERENCES products(id) ON DELETE RESTRICT,
    CONSTRAINT chk_order_items_quantity CHECK (quantity > 0),
    CONSTRAINT chk_order_items_price CHECK (unit_price > 0)
);

-- Indexes on foreign keys (PostgreSQL doesn't auto-create these!)
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_product_id ON order_items(product_id);

-- Index for common query pattern
CREATE INDEX idx_orders_status ON orders(status) WHERE status != 'delivered';
```

---

## 16. Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| No primary key | No reliable way to identify rows | Always define a PK |
| No index on FK columns | Slow JOINs and cascading deletes | Create indexes on FK columns manually |
| Auto-generated constraint names | Unclear error messages in logs | Name all constraints explicitly |
| `NOT NULL` on everything | Some columns legitimately should be nullable | Apply `NOT NULL` intentionally, not blindly |
| Using `serial` in new code | Legacy, weaker ownership | Use `GENERATED ALWAYS AS IDENTITY` |
| `varchar(255)` out of habit | PostgreSQL has no performance difference between `text` and `varchar` | Use `text` unless a max length is a business rule |
| Forgetting `IF EXISTS` / `IF NOT EXISTS` | Scripts fail on rerun | Always use in migration scripts |
| Adding FK with `VALID` on large tables | Long `ACCESS EXCLUSIVE` lock | Use `NOT VALID` + `VALIDATE CONSTRAINT` |

---

## 17. What to Learn Next

1. **Indexes** — B-tree, GIN, GiST, BRIN index types — see [05-indexes.md](05-indexes.md).
2. **Transactions** — `BEGIN`, `COMMIT`, `ROLLBACK`, isolation levels — see [02-transactions.md](02-transactions.md).
3. **Normalization & Data Modeling** — Designing proper schemas — Phase 3 (Roadmap).
4. **Sequences** — `SERIAL`, `IDENTITY`, custom sequences — see [11-sequences.md](11-sequences.md).
5. **Managing Databases** — Templates, schemas, tablespaces — see [03-managing-databases.md](03-managing-databases.md).

---

> *Ref: [Docs — DDL](https://www.postgresql.org/docs/18/ddl.html) · [Docs — Constraints](https://www.postgresql.org/docs/18/ddl-constraints.html) · [Docs — ALTER TABLE](https://www.postgresql.org/docs/18/sql-altertable.html) · [Neon Tutorial — Create Table](https://neon.com/postgresql/tutorial)*
