# PostgreSQL Row-Level Security & CREATE POLICY

> **Level**: Intermediate–Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 8.3 — Row-Level Security  
> A complete guide to PostgreSQL's Row-Level Security (RLS) — defining policies with `CREATE POLICY`, understanding PERMISSIVE vs RESTRICTIVE, per-command behavior, expression evaluation order, multi-policy combination, and practical real-world patterns.

> See also: [08-roles.md](08-roles.md) for Roles & Privileges (Phase 8.1/8.2).

---

## Table of Contents

1. [What Is Row-Level Security?](#1-what-is-row-level-security)
2. [Why RLS Matters](#2-why-rls-matters)
3. [When to Use RLS](#3-when-to-use-rls)
4. [How RLS Works — The Big Picture](#4-how-rls-works--the-big-picture)
5. [CREATE POLICY — Syntax & Parameters](#5-create-policy--syntax--parameters)
6. [USING vs WITH CHECK — The Two Expressions](#6-using-vs-with-check--the-two-expressions)
7. [PERMISSIVE vs RESTRICTIVE Policies](#7-permissive-vs-restrictive-policies)
8. [Per-Command Policy Behavior](#8-per-command-policy-behavior)
9. [Policies Applied by Command Type (Table 300)](#9-policies-applied-by-command-type-table-300)
10. [Application of Multiple Policies](#10-application-of-multiple-policies)
11. [Practical Examples](#11-practical-examples)
12. [Important Notes & Caveats](#12-important-notes--caveats)
13. [Common Mistakes & Misunderstandings](#13-common-mistakes--misunderstandings)
14. [Best Practices](#14-best-practices)
15. [Quick Revision Points](#15-quick-revision-points)

---

## 1. What Is Row-Level Security?

### Definition

**Row-Level Security (RLS)** is a PostgreSQL feature that allows you to control which rows in a table are visible or modifiable to specific database roles — on a **per-row** basis.

While traditional `GRANT`/`REVOKE` controls access at the **table** or **column** level, RLS operates at the **row** level.

### Analogy

> Think of a table like a **shared filing cabinet**. Traditional privileges decide *who can open the drawer*. RLS decides *which files inside the drawer each person is allowed to see and touch*.

Every user opens the same drawer (queries the same table), but each sees a **different subset of rows** based on the policy.

### The Two Pieces

| Piece | What It Does |
|-------|-------------|
| `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` | **Activates** the RLS mechanism on the table |
| `CREATE POLICY ... ON table_name` | **Defines the rules** for which rows each role can see/modify |

> **Critical**: Enabling RLS without defining any policies triggers a **"default deny"** — no rows are visible or updatable for non-owner roles.

---

## 2. Why RLS Matters

### The Problem Without RLS

Without RLS, row-level access control must be implemented in application code:

```
┌────────────────────────────────────────────────────────────┐
│  WITHOUT RLS                                               │
│                                                            │
│  App Layer:  WHERE tenant_id = current_tenant()            │
│              ↑ Must be added to EVERY query                │
│              ↑ One missed filter = data leak               │
│              ↑ Cannot enforce from DB level                │
│                                                            │
│  WITH RLS                                                  │
│                                                            │
│  DB Layer:   Policy: USING (tenant_id = current_setting    │
│                ('app.current_tenant')::int)                 │
│              ↑ Enforced at database level                   │
│              ↑ Applied automatically to ALL queries          │
│              ↑ Cannot be bypassed unless BYPASSRLS          │
└────────────────────────────────────────────────────────────┘
```

### Key Benefits

| Benefit | Description |
|---------|-------------|
| **Defense in depth** | Even if application logic has a bug, the DB enforces row access |
| **Multi-tenant isolation** | Each tenant only sees their own data, enforced at the DB layer |
| **Simplified application code** | No need to sprinkle `WHERE tenant_id = ...` on every query |
| **Consistent enforcement** | Works across all access paths — ORMs, raw SQL, reporting tools, `psql` |
| **Composable** | Multiple policies can be combined (PERMISSIVE ORed, RESTRICTIVE ANDed) |

---

## 3. When to Use RLS

| Scenario | Why RLS Helps |
|----------|--------------|
| **Multi-tenant SaaS** | Each tenant only sees their rows; one table, many tenants |
| **User data isolation** | Users see only their own records (orders, messages, profiles) |
| **Role-based data access** | Managers see their department's data; employees see only their own |
| **Regulatory compliance** | GDPR, HIPAA — enforce data visibility boundaries at DB level |
| **Shared databases** | Multiple applications share the same tables with different access rules |

### When NOT to Use RLS

| Scenario | Why |
|----------|-----|
| **Superuser bypass** | Superusers and table owners bypass RLS by default (unless `FORCE ROW LEVEL SECURITY` is set) |
| **Performance-critical hot paths** | Policy expressions add overhead to every query; for ultra-low-latency scenarios, measure first |
| **Simple single-user databases** | No benefit if there's only one access pattern |

---

## 4. How RLS Works — The Big Picture

### Step-by-Step Flow

```
┌──────────────────────────────────────────────────────────┐
│                  RLS Lifecycle                            │
│                                                          │
│  Step 1          Step 2          Step 3       Step 4     │
│  ─────────       ─────────       ─────────    ───────    │
│  Enable RLS      Create          Query        Policy     │
│  on the table    policies        arrives      is applied │
│                                                          │
│  ALTER TABLE     CREATE POLICY   SELECT *     PostgreSQL │
│  t ENABLE ROW    p ON t          FROM t       appends    │
│  LEVEL SECURITY  USING (expr)   WHERE ...    USING expr │
│                  WITH CHECK(..)              to the plan │
└──────────────────────────────────────────────────────────┘
```

### The Three Commands

```sql
-- Step 1: Enable RLS on the table
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Step 2: Define a policy
CREATE POLICY user_isolation ON orders
    FOR ALL
    TO PUBLIC
    USING (user_id = current_user_id());

-- Step 3: Query the table (policy is applied automatically)
SELECT * FROM orders;
-- PostgreSQL internally rewrites this to:
-- SELECT * FROM orders WHERE user_id = current_user_id();
```

### Default Deny

If RLS is enabled but **no policy** matches the current role and command, PostgreSQL applies a **default deny**:

```sql
-- RLS enabled, no policies defined
SELECT * FROM orders;  -- Returns 0 rows (not an error — just empty)
INSERT INTO orders ... -- ERROR: new row violates row-level security policy
```

---

## 5. CREATE POLICY — Syntax & Parameters

### Full Syntax

```sql
CREATE POLICY name ON table_name
    [ AS { PERMISSIVE | RESTRICTIVE } ]
    [ FOR { ALL | SELECT | INSERT | UPDATE | DELETE } ]
    [ TO { role_name | PUBLIC | CURRENT_ROLE | CURRENT_USER | SESSION_USER } [, ...] ]
    [ USING ( using_expression ) ]
    [ WITH CHECK ( check_expression ) ]
```

### Parameter Reference

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `name` | Yes | — | Policy name. Must be unique **per table** (same name can exist on different tables) |
| `table_name` | Yes | — | The table this policy applies to (optionally schema-qualified) |
| `AS PERMISSIVE \| RESTRICTIVE` | No | `PERMISSIVE` | Policy type — controls how this policy combines with others |
| `FOR command` | No | `ALL` | Which command this policy applies to: `ALL`, `SELECT`, `INSERT`, `UPDATE`, `DELETE` |
| `TO role_name` | No | `PUBLIC` | Which role(s) this policy applies to |
| `USING (expression)` | No | — | Filters **existing** rows (used for reads and row selection) |
| `WITH CHECK (expression)` | No | — | Validates **new or modified** rows (used for inserts and updates) |

### Important Defaults

```
┌──────────────────────────────────────────────────────────┐
│  Omit AS        → PERMISSIVE                             │
│  Omit FOR       → ALL (applies to every command)         │
│  Omit TO        → PUBLIC (applies to every role)         │
│  Omit USING     → No row filter (all existing rows pass) │
│  Omit WITH CHECK → Falls back to the USING expression    │
│                    (for ALL and UPDATE policies only)     │
└──────────────────────────────────────────────────────────┘
```

---

## 6. USING vs WITH CHECK — The Two Expressions

This is the most important distinction in RLS. These are two fundamentally different questions:

| Expression | Question It Answers | Applied To | On Failure |
|-----------|-------------------|-----------|-----------|
| `USING (expr)` | "Can this role **see** this existing row?" | Reads (`SELECT`), row selection for `UPDATE`/`DELETE` | Row is **silently hidden** (filtered out) |
| `WITH CHECK (expr)` | "Is this role **allowed to write** this new/modified row?" | Writes (`INSERT`, `UPDATE`) | **Error is thrown** (policy violation) |

### Visual Analogy

```
                    USING                           WITH CHECK
                    ─────                           ──────────
   ┌──────────┐    "Which rows can        ┌──────────┐    "Is this new row
   │ Existing │ ──→  you SEE?"            │ New/     │ ──→  allowed to
   │ Rows     │                           │ Modified │      EXIST?"
   └──────────┘                           │ Row      │
        │                                  └──────────┘
        ▼                                       │
   true  → row is visible                       ▼
   false → row is silently hidden          true  → write proceeds
   null  → row is silently hidden          false → ERROR thrown
                                           null  → ERROR thrown
```

### Key Rule for ALL and UPDATE Policies

> If a policy has a `USING` expression but **no** `WITH CHECK` expression, the `USING` expression is used for **both** purposes — filtering existing rows AND validating new/modified rows.

```sql
-- This policy:
CREATE POLICY see_own ON orders FOR ALL
    USING (user_id = current_user);
-- Is equivalent to:
CREATE POLICY see_own ON orders FOR ALL
    USING (user_id = current_user)
    WITH CHECK (user_id = current_user);
```

### WITH CHECK Evaluation Order

For `INSERT`, `UPDATE`, and `MERGE`:

1. **BEFORE triggers** fire first (and may modify the row)
2. **WITH CHECK** expression is evaluated (against the row as modified by triggers)
3. **Other constraints** are checked (NOT NULL, CHECK, UNIQUE, FK)
4. Actual data modification happens

> This means a `BEFORE ROW` trigger can change the data being inserted, which affects whether the security policy passes or fails.

---

## 7. PERMISSIVE vs RESTRICTIVE Policies

### PERMISSIVE (Default)

- Multiple permissive policies are combined with **OR**.
- Each permissive policy **adds** to the set of accessible rows.
- Think of it as: "You can see a row if **any** permissive policy says yes."

```sql
-- Users can see their own orders
CREATE POLICY own_orders ON orders AS PERMISSIVE
    FOR SELECT USING (user_id = current_user);

-- Users can also see public orders
CREATE POLICY public_orders ON orders AS PERMISSIVE
    FOR SELECT USING (is_public = true);

-- Result: user sees rows WHERE (user_id = current_user) OR (is_public = true)
```

### RESTRICTIVE

- Multiple restrictive policies are combined with **AND**.
- Each restrictive policy **narrows** the set of accessible rows.
- Think of it as: "You can only see a row if **all** restrictive policies say yes."

```sql
-- Restrictive: only active orders are ever visible
CREATE POLICY active_only ON orders AS RESTRICTIVE
    FOR SELECT USING (status != 'archived');

-- Restrictive: only orders from the current fiscal year
CREATE POLICY current_year ON orders AS RESTRICTIVE
    FOR SELECT USING (created_at >= date_trunc('year', now()));
```

### The Critical Rule

> **There must be at least one PERMISSIVE policy granting access before RESTRICTIVE policies have any effect.**

If only restrictive policies exist → **no rows are accessible** (there's nothing to restrict).

### How They Combine

```
Final access =
    (permissive_1 OR permissive_2 OR ... OR permissive_N)
    AND
    restrictive_1
    AND
    restrictive_2
    AND
    ...
    AND
    restrictive_N
```

### Visual Example

```
┌─────────────────────────────────────────────────────┐
│  All rows in the table                              │
│  ┌───────────────────────────────────────────┐      │
│  │  Permissive Policy A (OR)                 │      │
│  │  ┌───────────────────────────────┐        │      │
│  │  │  Permissive Policy B (OR)     │        │      │
│  │  │                               │        │      │
│  │  │  ═══════════════════════════   │        │      │
│  │  │  Union of A ∪ B = base set    │        │      │
│  │  └───────────────────────────────┘        │      │
│  └───────────────────────────────────────────┘      │
│       ↓ AND                                         │
│  ┌───────────────────────────────────────────┐      │
│  │  Restrictive Policy X (AND)               │      │
│  │  ┌───────────────────────────────┐        │      │
│  │  │  Restrictive Policy Y (AND)   │        │      │
│  │  │                               │        │      │
│  │  │  Intersection within base set │        │      │
│  │  └───────────────────────────────┘        │      │
│  └───────────────────────────────────────────┘      │
│       ↓                                             │
│  Final visible rows = (A ∪ B) ∩ X ∩ Y              │
└─────────────────────────────────────────────────────┘
```

---

## 8. Per-Command Policy Behavior

Each command type has specific rules about which expressions (`USING`/`WITH CHECK`) are applicable.

### ALL

- Applies to **every** command type.
- Accepts both `USING` and `WITH CHECK`.
- If only `USING` is defined, it's used for both reading and writing.
- Coexists with command-specific policies — both the `ALL` policy and the specific policy apply.

```sql
CREATE POLICY tenant_isolation ON orders FOR ALL
    USING (tenant_id = current_setting('app.tenant_id')::int)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::int);
```

### SELECT

- Applies to `SELECT` queries and any command requiring `SELECT` permissions on the relation.
- Uses only `USING` — **cannot have `WITH CHECK`**.
- If a `RETURNING` clause is used on `INSERT`/`UPDATE`/`DELETE`, newly written rows must also satisfy SELECT policies (error thrown if not, **never silently ignored**).
- `INSERT ... ON CONFLICT` requires SELECT permissions; rows proposed for insertion are checked against SELECT policies (error thrown if they don't pass).
- `MERGE` applies SELECT policies to both source and target relations before joining.

```sql
CREATE POLICY read_own ON orders FOR SELECT
    USING (user_id = current_user);
```

### INSERT

- Applies to `INSERT` commands and `MERGE ... THEN INSERT` actions.
- Uses only `WITH CHECK` — **cannot have `USING`** (there are no existing rows to filter).
- Rows that don't pass → **error** (entire command aborted).
- For `INSERT ... ON CONFLICT DO NOTHING/UPDATE` — INSERT policies' `WITH CHECK` is evaluated for **all** proposed rows, regardless of whether a conflict occurs.

```sql
CREATE POLICY insert_own ON orders FOR INSERT
    WITH CHECK (user_id = current_user);
```

### UPDATE

- Applies to `UPDATE`, `SELECT FOR UPDATE/SHARE`, `ON CONFLICT DO UPDATE`, and `MERGE ... THEN UPDATE`.
- Accepts **both** `USING` and `WITH CHECK`:
  - `USING` → which existing rows can be selected for updating
  - `WITH CHECK` → whether the modified row is allowed to be stored
- If only `USING` is specified, it's used for both cases.
- Typically requires `SELECT` rights too (for `WHERE`, `RETURNING`, `SET` expressions) → SELECT/ALL policies also apply.
- For `ON CONFLICT DO UPDATE`: if the existing row doesn't pass `USING`, an **error is thrown** (never silently skipped — unlike standalone `UPDATE` which silently filters).

```sql
CREATE POLICY update_own ON orders FOR UPDATE
    USING (user_id = current_user)           -- can only update your own rows
    WITH CHECK (user_id = current_user);     -- can't reassign to another user
```

### DELETE

- Applies to `DELETE` commands and `MERGE ... THEN DELETE` actions.
- Uses only `USING` — **cannot have `WITH CHECK`** (there's no new row to validate).
- Rows not passing `USING` are silently filtered for standalone `DELETE`.
- For `MERGE ... THEN DELETE`: if a row visible via SELECT policy doesn't pass the DELETE policy, an **error is thrown**.
- Typically requires `SELECT` rights too → SELECT/ALL policies also apply.

```sql
CREATE POLICY delete_own ON orders FOR DELETE
    USING (user_id = current_user);
```

### Summary: Which Expressions Are Allowed Per Command

| Command Policy | `USING` Allowed? | `WITH CHECK` Allowed? |
|----------------|------------------|-----------------------|
| `ALL` | Yes | Yes (falls back to `USING` if omitted) |
| `SELECT` | Yes | **No** |
| `INSERT` | **No** | Yes |
| `UPDATE` | Yes | Yes (falls back to `USING` if omitted) |
| `DELETE` | Yes | **No** |

---

## 9. Policies Applied by Command Type (Table 300)

This table shows exactly how each policy type's expressions interact with each command. **"Filter"** means the row is silently hidden if the expression returns false/null. **"Check"** means an error is thrown if the expression returns false/null.

| Command | SELECT/ALL `USING` | INSERT/ALL `WITH CHECK` | UPDATE/ALL `USING` | UPDATE/ALL `WITH CHECK` | DELETE/ALL `USING` |
|---------|-------------------|------------------------|-------------------|------------------------|-------------------|
| `SELECT` | Filter existing row | — | — | — | — |
| `SELECT FOR UPDATE/SHARE` | Filter existing row | — | Filter existing row | — | — |
| `INSERT` | Check new row ᵃ | Check new row | — | — | — |
| `UPDATE` | Filter existing row ᵃ & check new row ᵃ | — | Filter existing row | Check new row | — |
| `DELETE` | Filter existing row ᵃ | — | — | — | Filter existing row |
| `INSERT ... ON CONFLICT` | Check new row ᵇᶜ | Check new row ᶜ | — | — | — |
| `ON CONFLICT DO UPDATE` | Check existing & new rows ᵈ | — | Check existing row | Check new row ᵈ | — |
| `MERGE` | Filter source & target rows | — | — | — | — |
| `MERGE ... THEN INSERT` | Check new row ᵃ | Check new row | — | — | — |
| `MERGE ... THEN UPDATE` | Check new row | — | Check existing row | Check new row | — |
| `MERGE ... THEN DELETE` | — | — | — | — | Check existing row |

**Footnotes:**

- **ᵃ** If read access is required to either the existing or new row (e.g., a `WHERE` or `RETURNING` clause that refers to columns from the relation).
- **ᵇ** If an arbiter index or constraint is specified.
- **ᶜ** Row proposed for insertion is checked regardless of whether or not a conflict occurs.
- **ᵈ** New row of the auxiliary `UPDATE` command, which might be different from the new row of the original `INSERT` command.

---

## 10. Application of Multiple Policies

### Different Command Types on the Same Command

When policies of **different** command types apply to a single command (e.g., a `SELECT` policy and an `UPDATE` policy both apply to an `UPDATE` command), the user must satisfy **both** types. The expressions are combined with **AND**.

```
-- An UPDATE command needs:
--   Permission to SELECT the row  AND  permission to UPDATE the row

(SELECT policy expressions) AND (UPDATE policy expressions)
```

### Same Command Type — Multiple Policies

When multiple policies of the **same** command type apply:

1. All **PERMISSIVE** policies are combined with **OR**
2. All **RESTRICTIVE** policies are combined with **AND**
3. The two results are combined with **AND**

> At least one permissive policy must grant access, otherwise access is denied entirely.

### ALL Policies in the Mix

`ALL` policies are treated as the **same type** as whichever specific command type is being applied. So in an `UPDATE` requiring both SELECT and UPDATE permissions:

- `ALL` policies are combined with SELECT-specific policies for the SELECT check
- `ALL` policies are combined with UPDATE-specific policies for the UPDATE check

### Full Combination Formula

For an `UPDATE` command requiring both SELECT and UPDATE permissions:

```sql
-- The full expression that must evaluate to TRUE:

    expression from RESTRICTIVE SELECT/ALL policy 1
AND expression from RESTRICTIVE SELECT/ALL policy 2
AND ...
AND (
        expression from PERMISSIVE SELECT/ALL policy 1
     OR expression from PERMISSIVE SELECT/ALL policy 2
     OR ...
    )
AND expression from RESTRICTIVE UPDATE/ALL policy 1
AND expression from RESTRICTIVE UPDATE/ALL policy 2
AND ...
AND (
        expression from PERMISSIVE UPDATE/ALL policy 1
     OR expression from PERMISSIVE UPDATE/ALL policy 2
     OR ...
    )
```

---

## 11. Practical Examples

### Example 1: Multi-Tenant SaaS Isolation

The most common RLS use case — each tenant sees only their own data:

```sql
-- Setup
CREATE TABLE documents (
    id          serial PRIMARY KEY,
    tenant_id   int NOT NULL,
    title       text,
    content     text,
    created_by  text DEFAULT current_user
);

-- Step 1: Enable RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Step 2: Create policy using a session variable for tenant identity
CREATE POLICY tenant_isolation ON documents
    FOR ALL
    TO PUBLIC
    USING (tenant_id = current_setting('app.tenant_id')::int)
    WITH CHECK (tenant_id = current_setting('app.tenant_id')::int);

-- Step 3: Application sets the tenant context before queries
SET app.tenant_id = '42';
SELECT * FROM documents;  -- Only sees tenant 42's documents
INSERT INTO documents (tenant_id, title) VALUES (42, 'OK');    -- Succeeds
INSERT INTO documents (tenant_id, title) VALUES (99, 'FAIL');  -- ERROR: policy violation
```

### Example 2: Users See Only Their Own Rows

```sql
CREATE TABLE messages (
    id        serial PRIMARY KEY,
    sender    text NOT NULL,
    recipient text NOT NULL,
    body      text,
    sent_at   timestamptz DEFAULT now()
);

ALTER TABLE messages ENABLE ROW LEVEL SECURITY;

-- Users can see messages they sent or received
CREATE POLICY see_own_messages ON messages FOR SELECT
    USING (sender = current_user OR recipient = current_user);

-- Users can only insert messages as themselves
CREATE POLICY send_as_self ON messages FOR INSERT
    WITH CHECK (sender = current_user);

-- Users can only delete messages they sent
CREATE POLICY delete_own ON messages FOR DELETE
    USING (sender = current_user);
```

### Example 3: PERMISSIVE + RESTRICTIVE Together

```sql
CREATE TABLE reports (
    id           serial PRIMARY KEY,
    department   text,
    is_published boolean DEFAULT false,
    created_at   timestamptz DEFAULT now()
);

ALTER TABLE reports ENABLE ROW LEVEL SECURITY;

-- PERMISSIVE: users in 'engineering' can see engineering reports
CREATE POLICY eng_access ON reports AS PERMISSIVE FOR SELECT
    TO engineering
    USING (department = 'engineering');

-- PERMISSIVE: users in 'management' can see all departments
CREATE POLICY mgmt_access ON reports AS PERMISSIVE FOR SELECT
    TO management
    USING (true);

-- RESTRICTIVE: regardless of other policies, only published reports are visible
CREATE POLICY published_only ON reports AS RESTRICTIVE FOR SELECT
    USING (is_published = true);

-- For 'engineering' role, final filter:
--   (department = 'engineering')   ← permissive
--   AND (is_published = true)      ← restrictive
--
-- For 'management' role, final filter:
--   (true)                         ← permissive
--   AND (is_published = true)      ← restrictive
```

### Example 4: Separate Read/Write Policies

```sql
CREATE TABLE inventory (
    id          serial PRIMARY KEY,
    warehouse   text,
    product     text,
    quantity    int,
    updated_by  text
);

ALTER TABLE inventory ENABLE ROW LEVEL SECURITY;

-- All warehouse staff can read items in their warehouse
CREATE POLICY read_warehouse ON inventory FOR SELECT
    USING (warehouse = current_setting('app.warehouse'));

-- Staff can update items only in their warehouse, and must stamp their name
CREATE POLICY update_warehouse ON inventory FOR UPDATE
    USING (warehouse = current_setting('app.warehouse'))
    WITH CHECK (
        warehouse = current_setting('app.warehouse')
        AND updated_by = current_user
    );
```

### Example 5: Force RLS on Table Owner

By default, the table owner **bypasses** RLS. To enforce policies even on the owner:

```sql
ALTER TABLE documents FORCE ROW LEVEL SECURITY;

-- Now even the table owner is subject to policies
-- (Superusers still bypass unless they explicitly SET ROLE)
```

### Example 6: Complete Hands-On Walkthrough (Role-Based Department Access)

A full end-to-end example you can follow in `psql`. Each manager only sees the department they manage.

```sql
-- Step 1: Create a new database and connect to it
CREATE DATABASE hr;
\c hr

-- Step 2: Create the departments table
CREATE TABLE departments (
    id      serial PRIMARY KEY,
    name    VARCHAR(255) NOT NULL UNIQUE,
    manager VARCHAR(255) NOT NULL
);

-- Step 3: Insert sample data
INSERT INTO departments (name, manager)
VALUES
    ('Sales',     'alice'),
    ('Marketing', 'bob'),
    ('IT',        'jack');

-- Step 4: Create a group role (no login — used as a permission bundle)
CREATE ROLE managers;

-- Step 5: Grant SELECT on all tables in public schema to the group
GRANT SELECT ON ALL TABLES IN SCHEMA public TO managers;

-- Step 6: Create login roles as members of the managers group
CREATE ROLE alice WITH LOGIN PASSWORD 'SecurePass1' IN ROLE managers;
CREATE ROLE bob   WITH LOGIN PASSWORD 'SecurePass2' IN ROLE managers;
CREATE ROLE jack  WITH LOGIN PASSWORD 'SecurePass3' IN ROLE managers;
-- These roles inherit SELECT privileges from managers automatically.

-- Step 7: Enable RLS on the departments table
ALTER TABLE departments ENABLE ROW LEVEL SECURITY;

-- Step 8: Create a policy — each manager sees only their department
CREATE POLICY department_managers ON departments
    TO managers
    USING (manager = current_user);
```

Now test from separate sessions:

```sql
-- Session A: connect as alice
-- psql -U alice -d hr

SELECT * FROM departments;
--  id | name  | manager
-- ----+-------+---------
--   1 | Sales | alice
-- (1 row)
```

```sql
-- Session B: connect as bob
-- psql -U bob -d hr

SELECT * FROM departments;
--  id |   name    | manager
-- ----+-----------+---------
--   2 | Marketing | bob
-- (1 row)
```

```sql
-- Session C: connect as jack
-- psql -U jack -d hr

SELECT * FROM departments;
--  id | name | manager
-- ----+------+---------
--   3 | IT   | jack
-- (1 row)
```

> Each role queries the **same table** with the **same SQL**, but PostgreSQL's policy silently filters rows so each user only sees their own department. This is RLS in action.

---

## 12. Important Notes & Caveats

### Ownership and Superuser Bypass

- **Table owners** bypass RLS by default. Use `ALTER TABLE ... FORCE ROW LEVEL SECURITY` to change this.
- **Superusers** always bypass RLS. To test policies as a superuser, use `SET ROLE` to switch to a non-superuser role.
- The `BYPASSRLS` role attribute allows a non-superuser role to bypass all RLS policies.

### Referential Integrity Information Leaks

Policies are **not** applied during internal referential integrity checks or constraint validation. This means:

- Inserting a duplicate into a unique column reveals that the value exists (even if the existing row is hidden by policy).
- Inserting into a FK column reveals whether the referenced value exists in the parent table.

> **Mitigation**: Use surrogate keys (e.g., UUIDs) instead of keys with external meaning, and carefully restrict INSERT/DELETE/UPDATE permissions.

### LEAKPROOF Functions

- PostgreSQL evaluates policy filter conditions **before** user-supplied qualifications (to prevent leaking protected data to untrusted functions).
- Functions marked as `LEAKPROOF` may be evaluated **before** policy expressions, because they are trusted not to leak data.

### Policy Expressions Run as the Querying User

Policy expressions run with the permissions of the **user executing the query**, not the policy creator. If the expression references other tables or functions, the querying user must have access to them — otherwise they get a permission denied error.

### Views and RLS

Views referenced in policies use the **view owner's** rights (not the querying user's), following normal view permission rules. Exception: views created with `security_invoker = true` use the invoker's rights.

### MERGE and Policies

There is no separate policy type for `MERGE`. Instead, the individual policies for `SELECT`, `INSERT`, `UPDATE`, and `DELETE` are applied depending on which actions the `MERGE` executes.

---

## 13. Common Mistakes & Misunderstandings

| Mistake | What Actually Happens | Fix |
|---------|----------------------|-----|
| Enabling RLS but creating no policies | **All rows become invisible** (default deny) | Always create at least one permissive policy |
| Only creating RESTRICTIVE policies | No rows accessible (nothing to restrict) | Add a permissive policy first to grant base access |
| Forgetting that table owners bypass RLS | Owner sees all rows despite policies | Use `FORCE ROW LEVEL SECURITY` if owner should be restricted |
| Using `WITH CHECK` on a `SELECT` policy | Syntax error — SELECT policies only support `USING` | Remove the `WITH CHECK` clause |
| Using `USING` on an `INSERT` policy | Syntax error — INSERT policies only support `WITH CHECK` | Remove the `USING` clause |
| Assuming policy hides data from superusers | Superusers **always** bypass RLS | Use `SET ROLE` to test as a normal user |
| Not setting session variables before queries | Policy expressions referencing `current_setting()` return NULL or error | Set variables (`SET app.tenant_id = '...'`) before querying |
| Expecting silent skip on `ON CONFLICT DO UPDATE` | Unlike standalone UPDATE, fails with **error** if existing row doesn't pass USING | Ensure conflicting rows satisfy UPDATE policies |
| Not granting SELECT for UPDATE/DELETE | UPDATE/DELETE typically need access to row data (WHERE, RETURNING) | Ensure SELECT or ALL policies also cover these rows |

---

## 14. Best Practices

### Design

1. **Start with a permissive ALL policy** for the most common access pattern, then add restrictive policies to further limit access where needed.
2. **Use session variables** (`current_setting('app.xyz')`) for tenant/user context — this decouples policies from role names.
3. **Keep policy expressions simple** — avoid subqueries and complex joins in USING/WITH CHECK. Performance matters since these run on every row.
4. **Name policies descriptively** — `tenant_isolation_select` is better than `policy1`. Policy names are per-table, so be specific about intent.

### Security

5. **Always test with `SET ROLE`** — switch to the actual application role and verify that policies work as intended.
6. **Use `FORCE ROW LEVEL SECURITY`** if the table owner should also be subject to policies.
7. **Watch for information leaks** via unique constraints and FK references — use surrogate keys.
8. **Avoid `LEAKPROOF` on untrusted functions** — only system-level or thoroughly audited functions should be marked `LEAKPROOF`.

### Operations

9. **Monitor policy performance** — use `EXPLAIN` to see how policy expressions are applied to query plans.
10. **Document your policies** — RLS logic is invisible in application code; ensure the team knows which policies exist and why.
11. **Version your policies** — include `CREATE POLICY` statements in migration scripts alongside schema changes.

---

## 15. Quick Revision Points

- **RLS** = row-level access control enforced by the database, not the application.
- **Two steps**: (1) `ALTER TABLE ... ENABLE ROW LEVEL SECURITY`, (2) `CREATE POLICY`.
- **No policies + RLS enabled** = **default deny** (no rows visible).
- **`USING`** = filter existing rows (silent); **`WITH CHECK`** = validate new/modified rows (error on fail).
- **PERMISSIVE** policies combine with **OR** (add access); **RESTRICTIVE** policies combine with **AND** (narrow access).
- **At least one permissive** policy must pass before restrictive policies matter.
- **Table owners** bypass RLS unless `FORCE ROW LEVEL SECURITY` is set.
- **Superusers** always bypass RLS — use `SET ROLE` to test.
- **`ALL` policies** apply to every command and are treated as the same type as the specific policy being evaluated.
- **INSERT** → `WITH CHECK` only. **SELECT/DELETE** → `USING` only. **ALL/UPDATE** → both.
- Policy expressions run as the **querying user** — they must have access to any referenced tables/functions.
- **`MERGE`** has no dedicated policy type — it uses the individual command policies as needed.

---

> **References**:  
> - [PostgreSQL Official Documentation — CREATE POLICY](https://www.postgresql.org/docs/current/sql-createpolicy.html)  
> - [PostgreSQL Official Documentation — Row Security Policies (Section 5.9)](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)  
> - [Neon — PostgreSQL Row-Level Security Tutorial](https://neon.com/postgresql/postgresql-administration/postgresql-row-level-security)