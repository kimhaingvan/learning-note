# Stored Procedures, Functions & Triggers

> **Level**: Intermediate–Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 6 — Stored Procedures, Functions & Triggers  
> PL/pgSQL language, user-defined functions, stored procedures, triggers, event triggers, and the rules system.

---

## Table of Contents

1. [PL/pgSQL Language](#1-plpgsql-language)
   - [Structure of a PL/pgSQL Block](#structure-of-a-plpgsql-block)
   - [Variables & Assignment](#variables--assignment)
   - [Control Flow](#control-flow)
   - [Looping](#looping)
   - [Exception Handling](#exception-handling)
   - [Dynamic SQL](#dynamic-sql)
   - [RAISE — Logging & Errors](#raise--logging--errors)
2. [User-Defined Functions](#2-user-defined-functions)
   - [SQL Functions](#sql-functions)
   - [PL/pgSQL Functions](#plpgsql-functions)
   - [RETURNS TABLE & RETURNS SETOF](#returns-table--returns-setof)
   - [Volatility Categories](#volatility-categories)
   - [SECURITY DEFINER vs INVOKER](#security-definer-vs-invoker)
   - [Function Overloading & Defaults](#function-overloading--defaults)
3. [Stored Procedures](#3-stored-procedures)
   - [Functions vs Procedures](#functions-vs-procedures)
   - [Transaction Control in Procedures](#transaction-control-in-procedures)
4. [Triggers & Event Triggers](#4-triggers--event-triggers)
   - [Row-Level Triggers](#row-level-triggers)
   - [Statement-Level Triggers](#statement-level-triggers)
   - [BEFORE / AFTER / INSTEAD OF](#before--after--instead-of)
   - [Trigger Functions, NEW & OLD](#trigger-functions-new--old)
   - [Transition Tables](#transition-tables)
   - [Event Triggers](#event-triggers)
   - [Common Trigger Patterns](#common-trigger-patterns)
5. [Rules System](#5-rules-system)

---

## 1. PL/pgSQL Language

PL/pgSQL is PostgreSQL's **built-in procedural language** for writing server-side logic. It extends SQL with variables, control flow, loops, and exception handling.

### Structure of a PL/pgSQL Block

```sql
[ <<label>> ]
[ DECLARE
    -- variable declarations ]
BEGIN
    -- statements
[ EXCEPTION
    -- error handlers ]
END [ label ];
```

Every PL/pgSQL function or `DO` block follows this structure. Blocks can **nest**:

```sql
DO $$
<<outer>>
DECLARE
    v_total integer := 0;
BEGIN
    <<inner>>
    DECLARE
        v_count integer;
    BEGIN
        SELECT count(*) INTO v_count FROM orders;
        v_total := v_count * 2;
    END inner;

    RAISE NOTICE 'Total: %', v_total;
END outer;
$$;
```

### Variables & Assignment

```sql
DECLARE
    -- Basic types
    v_name     text := 'Alice';
    v_age      integer;
    v_salary   numeric(10, 2) NOT NULL DEFAULT 0.00;
    v_active   boolean := true;

    -- Anchored types (match column types automatically)
    v_user_id  users.id%TYPE;           -- same type as users.id
    v_user_row users%ROWTYPE;           -- entire row type of users table

    -- Record (generic row, type determined at runtime)
    v_rec      record;

    -- Constants
    c_tax_rate CONSTANT numeric := 0.10;
BEGIN
    -- Assignment with :=
    v_age := 30;

    -- Assignment with SELECT INTO
    SELECT name, age INTO v_name, v_age
    FROM users WHERE id = 1;

    -- Check if SELECT INTO found a row
    IF NOT FOUND THEN
        RAISE EXCEPTION 'User not found';
    END IF;

    -- Row type assignment
    SELECT * INTO v_user_row FROM users WHERE id = 1;
    RAISE NOTICE 'Name: %, Email: %', v_user_row.name, v_user_row.email;
END;
```

| Feature | Syntax | Notes |
|---------|--------|-------|
| Typed variable | `v_name type [:= value]` | Declare with explicit type |
| `%TYPE` | `v_id table.column%TYPE` | Matches the column's current type |
| `%ROWTYPE` | `v_row table%ROWTYPE` | Matches entire row structure |
| `record` | `v_rec record` | Dynamic — fields set at runtime |
| `CONSTANT` | `c_val CONSTANT type := expr` | Cannot be reassigned |
| `NOT NULL` | `v_val type NOT NULL := expr` | Must always have a value |
| `FOUND` | `IF FOUND THEN ...` | True after SELECT INTO finds a row, or after INSERT/UPDATE/DELETE affects rows |

### Control Flow

#### IF / ELSIF / ELSE

```sql
IF v_score >= 90 THEN
    v_grade := 'A';
ELSIF v_score >= 80 THEN
    v_grade := 'B';
ELSIF v_score >= 70 THEN
    v_grade := 'C';
ELSE
    v_grade := 'F';
END IF;
```

#### CASE (Simple)

```sql
CASE v_status
    WHEN 'active'   THEN v_label := 'Active';
    WHEN 'inactive' THEN v_label := 'Inactive';
    WHEN 'banned'   THEN v_label := 'Banned';
    ELSE v_label := 'Unknown';
END CASE;
```

#### CASE (Searched)

```sql
CASE
    WHEN v_age < 18  THEN v_category := 'minor';
    WHEN v_age < 65  THEN v_category := 'adult';
    ELSE v_category := 'senior';
END CASE;
```

### Looping

#### Basic LOOP (with EXIT)

```sql
DECLARE
    v_counter integer := 0;
BEGIN
    LOOP
        v_counter := v_counter + 1;
        EXIT WHEN v_counter > 10;  -- break condition
    END LOOP;
END;
```

#### WHILE Loop

```sql
WHILE v_counter <= 10 LOOP
    -- do something
    v_counter := v_counter + 1;
END LOOP;
```

#### FOR Loop (integer range)

```sql
FOR i IN 1..10 LOOP
    RAISE NOTICE 'i = %', i;
END LOOP;

-- Reverse
FOR i IN REVERSE 10..1 LOOP
    RAISE NOTICE 'i = %', i;
END LOOP;

-- With step
FOR i IN 1..20 BY 3 LOOP
    RAISE NOTICE 'i = %', i;  -- 1, 4, 7, 10, 13, 16, 19
END LOOP;
```

#### FOR Loop (query result)

```sql
FOR v_rec IN SELECT id, name FROM users WHERE active = true LOOP
    RAISE NOTICE 'User: % - %', v_rec.id, v_rec.name;
END LOOP;
```

#### FOREACH (over array)

```sql
DECLARE
    v_ids integer[] := ARRAY[1, 2, 3, 4, 5];
    v_id  integer;
BEGIN
    FOREACH v_id IN ARRAY v_ids LOOP
        RAISE NOTICE 'Processing id: %', v_id;
    END LOOP;
END;
```

#### Loop Control

| Statement | Effect |
|-----------|--------|
| `EXIT` | Leave the innermost loop (or named loop: `EXIT label`) |
| `EXIT WHEN condition` | Leave loop if condition is true |
| `CONTINUE` | Skip to next iteration |
| `CONTINUE WHEN condition` | Skip if condition is true |

### Exception Handling

```sql
BEGIN
    INSERT INTO users (email) VALUES ('alice@example.com');
EXCEPTION
    WHEN unique_violation THEN
        RAISE NOTICE 'Email already exists.';
    WHEN not_null_violation THEN
        RAISE NOTICE 'A required field was NULL.';
    WHEN OTHERS THEN
        RAISE NOTICE 'Unexpected error: % — %', SQLSTATE, SQLERRM;
END;
```

> **How it works**: PL/pgSQL internally creates a **SAVEPOINT** before the `BEGIN` block. If an exception occurs, it rolls back to that savepoint and enters the `EXCEPTION` handler. This means **all work in the block is undone** — not just the failing statement.

| Variable | Contains |
|----------|---------|
| `SQLSTATE` | 5-character error code (e.g., `'23505'` for unique violation) |
| `SQLERRM` | Error message text |
| `SQLSTATE` codes | Full list at [Appendix A — Error Codes](https://www.postgresql.org/docs/18/errcodes-appendix.html) |

Common condition names:

| Condition Name | SQLSTATE | Meaning |
|---------------|----------|---------|
| `unique_violation` | 23505 | Duplicate key |
| `not_null_violation` | 23502 | NULL value in NOT NULL column |
| `foreign_key_violation` | 23503 | FK constraint violated |
| `check_violation` | 23514 | CHECK constraint violated |
| `division_by_zero` | 22012 | Division by zero |
| `no_data_found` | P0002 | `SELECT INTO` found no rows (with `STRICT`) |
| `too_many_rows` | P0003 | `SELECT INTO STRICT` found multiple rows |

### Dynamic SQL

Use `EXECUTE` to build and run SQL statements at runtime.

```sql
-- Simple dynamic SQL
EXECUTE 'SELECT count(*) FROM ' || quote_ident(v_table_name) INTO v_count;

-- Using format() — safer and more readable
EXECUTE format('SELECT count(*) FROM %I WHERE %I = $1', v_table_name, v_column_name)
INTO v_count
USING v_filter_value;
```

| Function | Purpose | Example |
|----------|---------|---------|
| `format('%I', name)` | Safely quote an **identifier** (table/column) | `format('%I', 'users')` → `users` |
| `format('%L', value)` | Safely quote a **literal** value | `format('%L', 'O''Brien')` → `'O''Brien'` |
| `format('%s', text)` | Simple string substitution (no quoting) | Use only for trusted input |
| `quote_ident(name)` | Quote an identifier | `quote_ident('user name')` → `"user name"` |
| `quote_literal(value)` | Quote a literal | `quote_literal('hello')` → `'hello'` |
| `USING $1, $2, ...` | Pass parameters to `EXECUTE` | Preferred for values — prevents SQL injection |

> **Security**: Always use `format('%I')` or `quote_ident()` for identifiers and `USING` clause for values. Never concatenate untrusted input directly into SQL strings.

```sql
-- ✅ SAFE: parameterized dynamic query
EXECUTE format('DELETE FROM %I WHERE created_at < $1', v_table_name)
USING v_cutoff_date;

-- ❌ DANGEROUS: direct string concatenation with user input
EXECUTE 'DELETE FROM ' || user_input_table || ' WHERE id = ' || user_input_id;
```

### RAISE — Logging & Errors

```sql
-- Log levels (messages sent to client and/or server log)
RAISE DEBUG 'Debug detail: %', v_value;
RAISE LOG 'Logged to server log only';
RAISE INFO 'Informational message';
RAISE NOTICE 'Notice: processing row %', v_id;     -- most common for debugging
RAISE WARNING 'Warning: deprecated usage';
RAISE EXCEPTION 'Error: user % not found', v_id;    -- aborts the current transaction

-- EXCEPTION with detail and hint
RAISE EXCEPTION 'Invalid amount: %', v_amount
    USING DETAIL = 'Amount must be positive',
          HINT   = 'Check the input validation logic',
          ERRCODE = 'check_violation';
```

| Level | Sent to Client | Sent to Log | Aborts Transaction |
|-------|:-----------:|:--------:|:--:|
| `DEBUG` | No (unless `client_min_messages = debug`) | Depends on `log_min_messages` | No |
| `LOG` | No | Yes | No |
| `INFO` | Yes | Depends | No |
| `NOTICE` | Yes | Depends | No |
| `WARNING` | Yes | Depends | No |
| `EXCEPTION` | Yes | Yes | **Yes** |

---

## 2. User-Defined Functions

### SQL Functions

Pure SQL body — the simplest type. The planner can **inline** them for optimization.

```sql
-- Simple scalar function
CREATE OR REPLACE FUNCTION full_name(first text, last text)
RETURNS text
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT first || ' ' || last;
$$;

SELECT full_name('Kim', 'Hai');  -- 'Kim Hai'
```

```sql
-- SQL function returning a row
CREATE OR REPLACE FUNCTION get_active_user_count()
RETURNS bigint
LANGUAGE sql
STABLE
AS $$
    SELECT count(*) FROM users WHERE active = true;
$$;
```

### PL/pgSQL Functions

Full procedural logic with variables, loops, and exception handling.

```sql
CREATE OR REPLACE FUNCTION transfer_funds(
    p_from_id  integer,
    p_to_id    integer,
    p_amount   numeric
)
RETURNS void
LANGUAGE plpgsql
AS $$
DECLARE
    v_balance numeric;
BEGIN
    -- Check sufficient balance
    SELECT balance INTO v_balance
    FROM accounts WHERE id = p_from_id
    FOR UPDATE;  -- lock the row

    IF v_balance IS NULL THEN
        RAISE EXCEPTION 'Account % not found', p_from_id;
    END IF;

    IF v_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient balance: % < %', v_balance, p_amount;
    END IF;

    -- Perform transfer
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from_id;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to_id;

    RAISE NOTICE 'Transferred % from account % to %', p_amount, p_from_id, p_to_id;
END;
$$;

-- Call it
SELECT transfer_funds(1, 2, 100.00);
```

### RETURNS TABLE & RETURNS SETOF

Functions that return multiple rows.

#### RETURNS TABLE

```sql
CREATE OR REPLACE FUNCTION get_overdue_orders(p_days integer DEFAULT 30)
RETURNS TABLE (
    order_id    integer,
    customer    text,
    total       numeric,
    order_date  date,
    days_overdue integer
)
LANGUAGE plpgsql
STABLE
AS $$
BEGIN
    RETURN QUERY
    SELECT
        o.id,
        c.name,
        o.total,
        o.created_at::date,
        (current_date - o.created_at::date)
    FROM orders o
    JOIN customers c ON c.id = o.customer_id
    WHERE o.status = 'pending'
      AND o.created_at < current_date - p_days;
END;
$$;

-- Use like a table
SELECT * FROM get_overdue_orders(60) WHERE total > 1000;
```

#### RETURNS SETOF

```sql
-- Return existing row type
CREATE OR REPLACE FUNCTION active_users()
RETURNS SETOF users
LANGUAGE sql
STABLE
AS $$
    SELECT * FROM users WHERE active = true;
$$;

-- Use it
SELECT name, email FROM active_users();
```

#### Key Statements for Returning Rows

| Statement | Use Case |
|-----------|----------|
| `RETURN QUERY SELECT ...` | Return results of a query (most common) |
| `RETURN QUERY EXECUTE '...'` | Return results of dynamic SQL |
| `RETURN NEXT` | Return one row at a time (inside a loop) |
| `RETURN` (no value) | Signal end of a set-returning function |

```sql
-- RETURN NEXT example — row-by-row
CREATE OR REPLACE FUNCTION fibonacci(n integer)
RETURNS SETOF bigint
LANGUAGE plpgsql
IMMUTABLE
AS $$
DECLARE
    a bigint := 0;
    b bigint := 1;
    temp bigint;
BEGIN
    FOR i IN 1..n LOOP
        RETURN NEXT a;
        temp := a;
        a := b;
        b := temp + b;
    END LOOP;
END;
$$;

SELECT * FROM fibonacci(10);
-- 0, 1, 1, 2, 3, 5, 8, 13, 21, 34
```

### Volatility Categories

Volatility tells the planner **how the function behaves**, enabling important optimizations.

| Category | Meaning | Contract | Index-safe? |
|----------|---------|----------|:-----------:|
| **IMMUTABLE** | Returns the same result for the same arguments, **always** | No side effects, no database reads | ✅ Yes |
| **STABLE** | Returns the same result within a **single statement** | Reads database but doesn't modify it | ❌ No |
| **VOLATILE** (default) | Can return different results on each call | May have side effects, modify database | ❌ No |

```sql
-- IMMUTABLE: pure computation — can be used in indexes
CREATE FUNCTION area(radius numeric) RETURNS numeric
LANGUAGE sql IMMUTABLE AS $$
    SELECT pi() * radius * radius;
$$;
CREATE INDEX idx_area ON circles (area(radius));

-- STABLE: reads DB state — safe to call many times in one query without re-executing
CREATE FUNCTION current_user_timezone() RETURNS text
LANGUAGE sql STABLE AS $$
    SELECT timezone FROM user_preferences WHERE user_id = current_setting('app.user_id')::integer;
$$;

-- VOLATILE: modifies state or uses random()/now() — called every time
CREATE FUNCTION log_access(p_user_id integer) RETURNS void
LANGUAGE plpgsql VOLATILE AS $$
BEGIN
    INSERT INTO access_log (user_id, accessed_at) VALUES (p_user_id, now());
END;
$$;
```

| Misuse | Problem |
|--------|---------|
| Marking a VOLATILE function as IMMUTABLE | Planner caches results → stale/wrong data, or index corruption |
| Marking a STABLE function as IMMUTABLE | If used in an index, changes to referenced tables won't update the index |
| Using `now()` in an IMMUTABLE function | `now()` is STABLE (constant per transaction), not IMMUTABLE |

### SECURITY DEFINER vs INVOKER

| Mode | Runs As | Use Case |
|------|---------|----------|
| `SECURITY INVOKER` (default) | The **caller's** privileges | Normal functions |
| `SECURITY DEFINER` | The **function owner's** privileges | Controlled escalation — let users perform specific actions they don't otherwise have permission for |

```sql
-- SECURITY DEFINER example: let app users reset passwords
-- without giving them direct UPDATE on the users table
CREATE OR REPLACE FUNCTION reset_password(p_user_id integer, p_new_hash text)
RETURNS void
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public   -- IMPORTANT: prevent search_path attacks
AS $$
BEGIN
    UPDATE users SET password_hash = p_new_hash WHERE id = p_user_id;
END;
$$;

-- Grant execute to the app role
GRANT EXECUTE ON FUNCTION reset_password(integer, text) TO app_role;
```

> **Security**: Always `SET search_path` in `SECURITY DEFINER` functions. Without it, an attacker could create a malicious table/function in a schema that comes first in the search path.

### Function Overloading & Defaults

PostgreSQL supports **function overloading** — multiple functions with the same name but different parameter types.

```sql
-- Overloaded functions
CREATE FUNCTION format_value(v integer) RETURNS text
LANGUAGE sql IMMUTABLE AS $$ SELECT v::text; $$;

CREATE FUNCTION format_value(v numeric) RETURNS text
LANGUAGE sql IMMUTABLE AS $$ SELECT to_char(v, 'FM999,999.00'); $$;

CREATE FUNCTION format_value(v timestamptz) RETURNS text
LANGUAGE sql IMMUTABLE AS $$ SELECT to_char(v, 'YYYY-MM-DD HH24:MI'); $$;

-- PostgreSQL picks the right overload by argument type
SELECT format_value(42);           -- '42'
SELECT format_value(12345.67);     -- '12,345.67'
SELECT format_value(now());        -- '2026-03-27 14:30'
```

```sql
-- Default parameter values
CREATE FUNCTION get_orders(
    p_status text DEFAULT 'pending',
    p_limit  integer DEFAULT 50
)
RETURNS SETOF orders
LANGUAGE sql
STABLE
AS $$
    SELECT * FROM orders WHERE status = p_status ORDER BY created_at LIMIT p_limit;
$$;

-- All are valid calls
SELECT * FROM get_orders();                     -- defaults: 'pending', 50
SELECT * FROM get_orders('shipped');            -- defaults: limit = 50
SELECT * FROM get_orders('shipped', 10);        -- explicit: both
SELECT * FROM get_orders(p_limit => 100);       -- named parameter
```

### Managing Functions

```sql
-- List functions in current schema
SELECT routine_name, routine_type, data_type
FROM information_schema.routines
WHERE routine_schema = 'public'
ORDER BY routine_name;

-- View function source
SELECT pg_get_functiondef(oid) FROM pg_proc WHERE proname = 'transfer_funds';

-- Drop (must match parameter types for overloaded functions)
DROP FUNCTION transfer_funds(integer, integer, numeric);
DROP FUNCTION IF EXISTS transfer_funds(integer, integer, numeric);
```

---

## 3. Stored Procedures

Introduced in **PostgreSQL 11**. Unlike functions, procedures can **manage transaction control** internally.

### Functions vs Procedures

| Feature | Function | Procedure |
|---------|----------|-----------|
| Invocation | `SELECT func()` or in expressions | `CALL proc()` |
| Return value | Yes (`RETURNS type`) | No (use `INOUT` parameters for output) |
| Transaction control | ❌ Cannot `COMMIT`/`ROLLBACK` inside | ✅ Can `COMMIT`/`ROLLBACK` inside |
| Use in SQL expressions | ✅ Yes | ❌ No |
| Use in triggers | ✅ Yes (trigger functions) | ❌ No |
| Created with | `CREATE FUNCTION` | `CREATE PROCEDURE` |

### Creating a Procedure

```sql
CREATE OR REPLACE PROCEDURE archive_old_orders(p_days integer DEFAULT 90)
LANGUAGE plpgsql
AS $$
DECLARE
    v_batch_size integer := 5000;
    v_deleted    integer;
    v_total      integer := 0;
BEGIN
    LOOP
        -- Move a batch to the archive table
        WITH moved AS (
            DELETE FROM orders
            WHERE status = 'completed'
              AND completed_at < current_date - p_days
            RETURNING *
        )
        INSERT INTO orders_archive SELECT * FROM moved;

        GET DIAGNOSTICS v_deleted = ROW_COUNT;
        v_total := v_total + v_deleted;

        -- Commit each batch individually — this is why we need a procedure
        COMMIT;

        RAISE NOTICE 'Archived % rows (total: %)', v_deleted, v_total;
        EXIT WHEN v_deleted < v_batch_size;

        -- Let autovacuum breathe
        PERFORM pg_sleep(0.1);
    END LOOP;

    RAISE NOTICE 'Archival complete. Total rows archived: %', v_total;
END;
$$;

-- Call the procedure
CALL archive_old_orders(180);
```

### Transaction Control in Procedures

```sql
CREATE OR REPLACE PROCEDURE batch_import(p_data jsonb)
LANGUAGE plpgsql
AS $$
DECLARE
    v_item  jsonb;
    v_count integer := 0;
BEGIN
    FOR v_item IN SELECT * FROM jsonb_array_elements(p_data) LOOP
        BEGIN
            INSERT INTO products (name, price)
            VALUES (
                v_item->>'name',
                (v_item->>'price')::numeric
            );
            v_count := v_count + 1;

            -- Commit every 100 rows
            IF v_count % 100 = 0 THEN
                COMMIT;
                RAISE NOTICE 'Committed % rows', v_count;
            END IF;

        EXCEPTION WHEN OTHERS THEN
            RAISE WARNING 'Skipping invalid item: % — %', v_item, SQLERRM;
            -- Continue to next item
        END;
    END LOOP;

    COMMIT;
    RAISE NOTICE 'Import complete: % rows', v_count;
END;
$$;
```

### Key Rules

| Rule | Detail |
|------|--------|
| `COMMIT` / `ROLLBACK` in procedures | Commits/rolls back **the current transaction** and starts a new one |
| Cannot use in explicit transaction blocks | `BEGIN; CALL proc_with_commit(); COMMIT;` → ERROR — the `CALL` must be the top-level transaction |
| `INOUT` parameters for output | Since procedures have no `RETURNS`, use `INOUT` to pass values back |
| Atomicty per batch | Each `COMMIT` inside makes that batch permanent — earlier batches survive even if a later batch fails |

```sql
-- INOUT example — return a value from a procedure
CREATE OR REPLACE PROCEDURE get_count(p_table text, INOUT p_count bigint)
LANGUAGE plpgsql
AS $$
BEGIN
    EXECUTE format('SELECT count(*) FROM %I', p_table) INTO p_count;
END;
$$;

CALL get_count('users', NULL);
-- Returns: p_count = 42
```

---

## 4. Triggers & Event Triggers

### Overview

A **trigger** is a function that PostgreSQL automatically executes **before**, **after**, or **instead of** a data modification event (`INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`) on a table.

### Creating a Trigger (3 Steps)

```sql
-- Step 1: Create the trigger function (MUST return TRIGGER)
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at := current_timestamp;
    RETURN NEW;
END;
$$;

-- Step 2: Attach the trigger to a table
CREATE TRIGGER trg_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION set_updated_at();

-- Step 3: Test it
UPDATE users SET name = 'Bob' WHERE id = 1;
-- updated_at is automatically set to now()
```

### Row-Level Triggers

Fire **once per affected row**. Use `FOR EACH ROW`.

```sql
CREATE TRIGGER trg_audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION audit_order_changes();
```

### Statement-Level Triggers

Fire **once per SQL statement**, regardless of how many rows are affected. Use `FOR EACH STATEMENT`.

```sql
CREATE TRIGGER trg_notify_bulk_delete
    AFTER DELETE ON orders
    FOR EACH STATEMENT
    EXECUTE FUNCTION notify_bulk_operation();
```

### BEFORE / AFTER / INSTEAD OF

| Timing | When It Fires | Can Modify Row? | Can Cancel Operation? | Use `NEW`/`OLD`? |
|--------|--------------|:---------------:|:--------------------:|:----------------:|
| **BEFORE** | Before the row is inserted/updated/deleted | ✅ Yes (modify `NEW`) | ✅ Yes (return `NULL`) | ✅ Yes |
| **AFTER** | After the row is inserted/updated/deleted | ❌ No (changes already made) | ❌ No | ✅ Yes (read-only) |
| **INSTEAD OF** | Replaces the operation (only on views) | N/A — you implement the logic | N/A | ✅ Yes |

```sql
-- BEFORE trigger: validate or transform data
CREATE OR REPLACE FUNCTION validate_order()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    -- Normalize
    NEW.email := lower(trim(NEW.email));

    -- Validate
    IF NEW.total <= 0 THEN
        RAISE EXCEPTION 'Order total must be positive, got: %', NEW.total;
    END IF;

    -- Cancel the INSERT by returning NULL
    IF NEW.status = 'rejected' THEN
        RETURN NULL;  -- silently cancels the row
    END IF;

    RETURN NEW;  -- allow the operation with (possibly modified) data
END;
$$;
```

```sql
-- INSTEAD OF trigger: make a complex view updatable
CREATE VIEW active_user_orders AS
SELECT u.id AS user_id, u.name, o.id AS order_id, o.total
FROM users u JOIN orders o ON o.user_id = u.id
WHERE u.active = true;

CREATE OR REPLACE FUNCTION insert_active_user_order()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO orders (user_id, total)
    VALUES (NEW.user_id, NEW.total);
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_insert_active_user_orders
    INSTEAD OF INSERT ON active_user_orders
    FOR EACH ROW
    EXECUTE FUNCTION insert_active_user_order();
```

### Trigger Functions, NEW & OLD

| Variable | Available In | Contains |
|----------|------------|---------|
| `NEW` | INSERT, UPDATE | The **new** row being inserted or the **updated** row |
| `OLD` | UPDATE, DELETE | The **original** row before the change |
| `TG_OP` | Any trigger | Operation: `'INSERT'`, `'UPDATE'`, `'DELETE'`, `'TRUNCATE'` |
| `TG_TABLE_NAME` | Any trigger | Name of the table that fired the trigger |
| `TG_TABLE_SCHEMA` | Any trigger | Schema of the table |
| `TG_WHEN` | Any trigger | `'BEFORE'`, `'AFTER'`, or `'INSTEAD OF'` |
| `TG_LEVEL` | Any trigger | `'ROW'` or `'STATEMENT'` |
| `TG_NARGS` | Any trigger | Number of arguments passed to the trigger |
| `TG_ARGV` | Any trigger | Array of trigger arguments (text[]) |

```sql
-- Multi-purpose audit trigger using TG_OP
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, new_data)
        VALUES (TG_TABLE_NAME, 'INSERT', row_to_json(NEW));
        RETURN NEW;

    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, new_data)
        VALUES (TG_TABLE_NAME, 'UPDATE', row_to_json(OLD), row_to_json(NEW));
        RETURN NEW;

    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_data)
        VALUES (TG_TABLE_NAME, 'DELETE', row_to_json(OLD));
        RETURN OLD;
    END IF;
END;
$$;

-- Attach to any table
CREATE TRIGGER trg_audit_users
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_changes();

CREATE TRIGGER trg_audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_changes();
```

### WHEN Clause — Conditional Triggers

```sql
-- Only fire when the status column changes
CREATE TRIGGER trg_status_changed
    AFTER UPDATE ON orders
    FOR EACH ROW
    WHEN (OLD.status IS DISTINCT FROM NEW.status)
    EXECUTE FUNCTION notify_status_change();

-- Only fire for significant price changes (> 10%)
CREATE TRIGGER trg_price_alert
    AFTER UPDATE OF price ON products
    FOR EACH ROW
    WHEN (NEW.price > OLD.price * 1.10 OR NEW.price < OLD.price * 0.90)
    EXECUTE FUNCTION alert_price_change();
```

### Trigger Arguments

```sql
-- Pass arguments to a generic trigger function
CREATE TRIGGER trg_soft_delete_users
    BEFORE DELETE ON users
    FOR EACH ROW
    EXECUTE FUNCTION soft_delete('users', 'deleted_at');

CREATE OR REPLACE FUNCTION soft_delete()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    -- TG_ARGV[0] = 'users', TG_ARGV[1] = 'deleted_at'
    EXECUTE format(
        'UPDATE %I SET %I = now() WHERE id = $1',
        TG_ARGV[0], TG_ARGV[1]
    ) USING OLD.id;

    RETURN NULL;  -- cancel the actual DELETE
END;
$$;
```

### Transition Tables

Available in **AFTER statement-level** triggers. Provide access to all rows affected by the statement as virtual tables.

```sql
CREATE OR REPLACE FUNCTION log_bulk_insert()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    v_count integer;
BEGIN
    SELECT count(*) INTO v_count FROM new_rows;  -- "new_rows" is the transition table
    RAISE NOTICE 'Bulk insert: % rows added to %', v_count, TG_TABLE_NAME;

    INSERT INTO insert_log (table_name, row_count, inserted_at)
    VALUES (TG_TABLE_NAME, v_count, now());

    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_log_bulk_orders
    AFTER INSERT ON orders
    REFERENCING NEW TABLE AS new_rows
    FOR EACH STATEMENT
    EXECUTE FUNCTION log_bulk_insert();
```

| Transition Table | Available In | Contains |
|-----------------|-------------|---------|
| `REFERENCING NEW TABLE AS name` | INSERT, UPDATE | All **new** rows from the statement |
| `REFERENCING OLD TABLE AS name` | UPDATE, DELETE | All **old** rows from the statement |
| Both | UPDATE | `REFERENCING OLD TABLE AS old_rows NEW TABLE AS new_rows` |

### Trigger Execution Order

When multiple triggers exist on the same table and event:

1. **BEFORE statement-level** triggers (alphabetical by name)
2. **BEFORE row-level** triggers (alphabetical by name, per row)
3. **The actual operation** (INSERT/UPDATE/DELETE)
4. **AFTER row-level** triggers (alphabetical by name, per row)
5. **AFTER statement-level** triggers (alphabetical by name)

> **Tip**: Prefix trigger names with a number to control ordering: `trg_01_validate`, `trg_02_audit`.

### Event Triggers

Fire on **DDL commands** (`CREATE`, `ALTER`, `DROP`, etc.) — not on data changes.

```sql
-- Event trigger function (returns event_trigger, not trigger)
CREATE OR REPLACE FUNCTION log_ddl_event()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
DECLARE
    v_obj record;
BEGIN
    FOR v_obj IN SELECT * FROM pg_event_trigger_ddl_commands() LOOP
        INSERT INTO ddl_log (
            event_time, command_tag, object_type, schema_name, object_identity
        ) VALUES (
            now(), v_obj.command_tag, v_obj.object_type,
            v_obj.schema_name, v_obj.object_identity
        );
    END LOOP;
END;
$$;

-- Attach to DDL events
CREATE EVENT TRIGGER trg_log_ddl
    ON ddl_command_end
    EXECUTE FUNCTION log_ddl_event();
```

| Event | When It Fires |
|-------|--------------|
| `ddl_command_start` | Before a DDL command executes |
| `ddl_command_end` | After a DDL command executes successfully |
| `table_rewrite` | Before a table is rewritten (e.g., `ALTER TABLE ... ALTER COLUMN TYPE`) |
| `sql_drop` | After an object is dropped |

```sql
-- Prevent dropping tables in production
CREATE OR REPLACE FUNCTION prevent_table_drops()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
DECLARE
    v_obj record;
BEGIN
    FOR v_obj IN SELECT * FROM pg_event_trigger_dropped_objects() LOOP
        IF v_obj.object_type = 'table' AND v_obj.schema_name = 'public' THEN
            RAISE EXCEPTION 'DROP TABLE on public schema is not allowed. Use migration scripts.';
        END IF;
    END LOOP;
END;
$$;

CREATE EVENT TRIGGER trg_prevent_drops
    ON sql_drop
    EXECUTE FUNCTION prevent_table_drops();
```

### Common Trigger Patterns

#### 1. Auto-set `updated_at` Timestamp

```sql
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    NEW.updated_at := current_timestamp;
    RETURN NEW;
END;
$$;

-- Reuse on any table with an updated_at column
CREATE TRIGGER trg_set_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION set_updated_at();
```

#### 2. Maintain Denormalized Count

```sql
CREATE OR REPLACE FUNCTION update_order_count()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE customers SET order_count = order_count + 1 WHERE id = NEW.customer_id;
    ELSIF TG_OP = 'DELETE' THEN
        UPDATE customers SET order_count = order_count - 1 WHERE id = OLD.customer_id;
    ELSIF TG_OP = 'UPDATE' AND OLD.customer_id != NEW.customer_id THEN
        UPDATE customers SET order_count = order_count - 1 WHERE id = OLD.customer_id;
        UPDATE customers SET order_count = order_count + 1 WHERE id = NEW.customer_id;
    END IF;
    RETURN COALESCE(NEW, OLD);
END;
$$;

CREATE TRIGGER trg_order_count
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION update_order_count();
```

#### 3. Prevent Deletion (Soft Delete)

```sql
CREATE OR REPLACE FUNCTION soft_delete_instead()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    UPDATE users SET deleted_at = now() WHERE id = OLD.id;
    RETURN NULL;  -- cancel the real DELETE
END;
$$;

CREATE TRIGGER trg_soft_delete
    BEFORE DELETE ON users
    FOR EACH ROW
    WHEN (OLD.deleted_at IS NULL)
    EXECUTE FUNCTION soft_delete_instead();
```

#### 4. Immutable Columns

```sql
CREATE OR REPLACE FUNCTION prevent_column_change()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
    IF NEW.created_by IS DISTINCT FROM OLD.created_by THEN
        RAISE EXCEPTION 'Column created_by is immutable';
    END IF;
    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_immutable_created_by
    BEFORE UPDATE ON documents
    FOR EACH ROW EXECUTE FUNCTION prevent_column_change();
```

### Managing Triggers

```sql
-- Disable a trigger (still exists, but doesn't fire)
ALTER TABLE users DISABLE TRIGGER trg_audit_users;

-- Disable ALL triggers on a table (including system triggers for FK constraints)
ALTER TABLE users DISABLE TRIGGER ALL;

-- Enable
ALTER TABLE users ENABLE TRIGGER trg_audit_users;

-- Drop
DROP TRIGGER trg_audit_users ON users;
DROP TRIGGER IF EXISTS trg_audit_users ON users;

-- List triggers on a table
SELECT trigger_name, event_manipulation, action_timing, action_orientation
FROM information_schema.triggers
WHERE event_object_table = 'users';
```

> **Warning**: `DISABLE TRIGGER ALL` also disables triggers that enforce FK constraints. Use with extreme caution and only during bulk data loads.

---

## 5. Rules System

PostgreSQL's **query rewriting system** — rewrites queries at the parser level before execution.

### How Rules Work

```sql
-- Example: redirect INSERTs on a view to the underlying table
CREATE RULE insert_to_active_users AS
    ON INSERT TO active_users_view
    DO INSTEAD
    INSERT INTO users (name, email, active)
    VALUES (NEW.name, NEW.email, true);
```

### Rules vs Triggers

| Aspect | Rules | Triggers |
|--------|-------|----------|
| **Mechanism** | Rewrites the query before execution | Fires a function during execution |
| **Level** | Query-level (statement rewriting) | Row-level or statement-level |
| **Predictability** | Can be surprising — the rewritten query may look nothing like the original | More intuitive — functions fire at well-defined points |
| **RETURNING support** | Poor (tricky to handle) | Good (`RETURN NEW/OLD`) |
| **Performance for bulk operations** | Can be faster (single rewritten query) | Function overhead per row (row-level triggers) |
| **Modern recommendation** | **Avoid** — use `INSTEAD OF` triggers for updatable views | **Preferred** for almost all use cases |

```sql
-- Rule to make a view updatable (legacy approach)
CREATE RULE update_active_users AS
    ON UPDATE TO active_users_view
    DO INSTEAD
    UPDATE users SET name = NEW.name, email = NEW.email
    WHERE id = OLD.id;

-- Modern alternative: INSTEAD OF trigger (preferred)
CREATE TRIGGER trg_update_active_users
    INSTEAD OF UPDATE ON active_users_view
    FOR EACH ROW
    EXECUTE FUNCTION handle_active_user_update();
```

### Rules for Preventing Operations

```sql
-- Prevent DELETE on a table (archival table — read-only)
CREATE RULE no_delete AS ON DELETE TO archived_orders DO INSTEAD NOTHING;

-- Prevent INSERT
CREATE RULE no_insert AS ON INSERT TO archived_orders DO INSTEAD NOTHING;
```

> **Recommendation**: Use rules only for simple redirections on legacy systems. For new code, always prefer triggers for their predictable behavior, better error handling, and `RETURNING` support.

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Forgetting `RETURN NEW` in BEFORE trigger | Row modification is silently cancelled | Always `RETURN NEW` (or `RETURN OLD` for DELETE) |
| Marking a STABLE function as IMMUTABLE | Index corruption if used in expression index | Use correct volatility category |
| No `SET search_path` on SECURITY DEFINER | Allows search_path hijacking attacks | Always add `SET search_path = public` |
| Infinite trigger loops | Trigger A updates table → fires trigger B → updates table → fires trigger A | Use `pg_trigger_depth()` to detect recursion |
| Long-running logic in BEFORE triggers | Adds latency to every INSERT/UPDATE | Move heavy work to AFTER triggers or async processing |
| Using rules for complex logic | Unpredictable query rewriting | Use INSTEAD OF triggers |
| Not using `USING` clause in dynamic SQL | SQL injection vulnerability | Always use `EXECUTE ... USING $1` for values |

---

## What to Learn Next

1. **Views & Materialized Views** — Named queries and cached results — Phase 7.
2. **Roles & Privileges** — Access control for functions and procedures — see [08-roles.md](08-roles.md).
3. **Row-Level Security** — Policies that leverage `current_user` and triggers — see [12-policy.md](12-policy.md).
4. **Transactions** — How `COMMIT`/`ROLLBACK` work inside procedures — see [02-transactions.md](02-transactions.md).
5. **Performance** — Impact of trigger overhead on write-heavy tables — Phase 11.

---

> *Ref: [Docs — PL/pgSQL](https://www.postgresql.org/docs/18/plpgsql.html) · [Docs — CREATE FUNCTION](https://www.postgresql.org/docs/18/sql-createfunction.html) · [Docs — CREATE PROCEDURE](https://www.postgresql.org/docs/18/sql-createprocedure.html) · [Docs — Triggers](https://www.postgresql.org/docs/18/trigger-definition.html) · [Docs — Event Triggers](https://www.postgresql.org/docs/18/event-triggers.html) · [Docs — Rules](https://www.postgresql.org/docs/18/rules.html) · [Neon — PostgreSQL Tutorial](https://neon.com/postgresql/tutorial)*
