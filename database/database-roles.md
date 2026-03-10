# PostgreSQL Database Roles

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> A complete guide to PostgreSQL's unified role system — creating users, groups, granting permissions, membership options (INHERIT/SET), and safely dropping roles.

---

## Table of Contents

1. [What Are Roles?](#1-what-are-roles)
2. [Why Roles Matter](#2-why-roles-matter)
3. [When to Use Roles](#3-when-to-use-roles)
4. [How Roles Work](#4-how-roles-work)
5. [Membership Options: INHERIT vs SET](#5-membership-options-inherit-vs-set)
6. [Dropping Roles Safely](#6-dropping-roles-safely)
7. [Practical Examples](#7-practical-examples)

---

## 1. What Are Roles?

Roles are PostgreSQL's **unified model for users and groups**. There is no separate "user" vs "group" concept — everything is a **role**.

A role can:

| Capability | Description |
|-----------|------------|
| **Log in** | If it has the `LOGIN` attribute |
| **Own objects** | Tables, schemas, functions, databases, etc. |
| **Receive object privileges** | Via `GRANT ... ON ...` |
| **Inherit privileges** through membership | Via `GRANT role TO role` |
| **Switch identity** in-session | Via `SET ROLE` |

> Roles are **cluster-wide** (not per-database) and are **separate from OS users**.

### Three Things to Keep Separate

| Concept | What It Controls | Example |
|---------|-----------------|---------|
| **Role attributes** | Flags on the role itself | `LOGIN`, `CREATEDB`, `CREATEROLE`, `REPLICATION`, `BYPASSRLS` |
| **Role membership** | One role granted to another | `GRANT parent_role TO member_role` |
| **Object privileges** | Permissions on specific objects | `GRANT SELECT ON TABLE reports TO reporting` |

---

## 2. Why Roles Matter

Most real PostgreSQL administration is **role design**:

- Who can **log in**?
- What can they do **directly** vs via **group roles**?
- Which powers require **explicit switching** with `SET ROLE`?
- How do you **clean up** safely when a role must be removed?

### Common Mistakes from Misunderstanding Roles

- Accidental "superuser-like" access through careless membership grants.
- Broken **least-privilege boundaries** (especially with membership/ownership).
- Failed `DROP ROLE` because of ownership/grants left behind.

---

## 3. When to Use Roles

| Action | When |
|--------|------|
| **Creating identities** for people, services, apps | `CREATE ROLE ... LOGIN` |
| **Creating group roles** that bundle permissions | `CREATE ROLE ... NOLOGIN` |
| **Granting access** | Via membership (teams) or object privileges (tables, schemas) |
| **Delegating admin** safely | Limited use of `CREATEROLE`, `WITH ADMIN OPTION` |
| **Using powerful capabilities** only when needed | Via `SET ROLE` |
| **Decommissioning accounts** | Ownership transfer + privilege cleanup + drop |

---

## 4. How Roles Work

### Available Role Attributes

```sql
SUPERUSER | NOSUPERUSER
CREATEDB | NOCREATEDB
CREATEROLE | NOCREATEROLE
INHERIT | NOINHERIT
LOGIN | NOLOGIN
REPLICATION | NOREPLICATION
BYPASSRLS | NOBYPASSRLS
CONNECTION LIMIT connlimit
PASSWORD 'password' | PASSWORD NULL
VALID UNTIL 'timestamp'
IN ROLE role_name [, ...]
ROLE role_name [, ...]
ADMIN role_name [, ...]
```

### Create a Group Role (Permission Bundle)

```sql
CREATE ROLE name [ [ WITH ] option [ ... ] ]

-- Example: a group role for reporting access
CREATE ROLE reporting NOLOGIN;
```

### Create a Login Role (Real Identity)

```sql
CREATE ROLE alice LOGIN PASSWORD '...';
```

### Alter / Update a Role

Changes role attributes, renames a role, or sets role-specific session defaults. Does **not** add or remove memberships — use `GRANT`/`REVOKE` for that.

```sql
ALTER ROLE role_specification [ WITH ] option [ ... ];

ALTER ROLE name RENAME TO new_name;

-- Set a per-role session default
ALTER ROLE { role_specification | ALL } [ IN DATABASE database_name ]
  SET configuration_parameter { TO | = } { value | DEFAULT };

-- Reset a per-role session default
ALTER ROLE { role_specification | ALL } [ IN DATABASE database_name ]
  RESET configuration_parameter;

ALTER ROLE { role_specification | ALL } [ IN DATABASE database_name ]
  RESET ALL;
```

### Grant Membership (Role → Role)

```sql
GRANT reporting TO alice WITH INHERIT TRUE, SET FALSE;
```

### Grant Object Privileges (Object → Role)

```sql
GRANT SELECT ON TABLE reports TO reporting;
```

### Revoke Role Membership

```sql
REVOKE [ { ADMIN | INHERIT | SET } OPTION FOR ]
    role_name [, ...] FROM role_specification [, ...]
    [ GRANTED BY role_specification ]
    [ CASCADE | RESTRICT ]
```

### Switch Role and Reset Role

```sql
-- Switch to another role (for permission checking)
SET [ SESSION | LOCAL ] ROLE role_name;
SET ROLE admin_role;

-- Return to session_user
SET ROLE NONE;

-- Go back to the connection-time role setting
RESET ROLE;
```

---

## 5. Membership Options: INHERIT vs SET

These two options control **how** a member uses a granted role's privileges:

| Option | Meaning |
|--------|---------|
| `WITH INHERIT TRUE` | Member **automatically gets** the role's object privileges |
| `WITH SET TRUE` | Member can **explicitly become** the role via `SET ROLE` |

These are **independent**. You can combine them in four ways:

| INHERIT | SET | Effect |
|---------|-----|--------|
| TRUE | TRUE | Privileges active automatically + can `SET ROLE` |
| TRUE | FALSE | Privileges active automatically, but cannot `SET ROLE` |
| FALSE | TRUE | Privileges off by default; must `SET ROLE` to activate |
| FALSE | FALSE | Membership exists but is basically unusable |

### Memory Shortcut

- `INHERIT TRUE` = privileges are **active automatically**.
- `SET TRUE` = you **can** `SET ROLE` into that role.
- `SET FALSE` = you **cannot** `SET ROLE` into that role.
- `INHERIT FALSE, SET FALSE` = membership exists but is basically unusable.

---

## 6. Dropping Roles Safely

### Typical Workflow

```sql
-- 1. Transfer ownership you want to keep
REASSIGN OWNED BY old_role TO successor_role;

-- 2. Clean up remaining ownership + privileges (run per database)
DROP OWNED BY old_role;

-- 3. Drop the role
DROP ROLE old_role;
```

> **Important**: `REASSIGN OWNED` and `DROP OWNED` operate per-database. If the role owns objects in multiple databases, you must connect to each database and run these commands.

---

## 7. Practical Examples

### Recommended Pattern: Login Role + Group Roles

```sql
-- Permission bundles (group roles)
CREATE ROLE app_read NOLOGIN;
CREATE ROLE app_write NOLOGIN;

-- Real identity (login role)
CREATE ROLE app_user LOGIN PASSWORD '...';

-- Membership: auto-inherit read/write privileges
GRANT app_read  TO app_user WITH INHERIT TRUE, SET FALSE;
GRANT app_write TO app_user WITH INHERIT TRUE, SET FALSE;

-- Object privileges granted to the group roles
GRANT USAGE ON SCHEMA app TO app_read;
GRANT SELECT ON ALL TABLES IN SCHEMA app TO app_read;

GRANT USAGE ON SCHEMA app TO app_write;
GRANT INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_write;
```

### Admin Role That Requires Explicit Switching

```sql
CREATE ROLE app_admin NOLOGIN CREATEDB CREATEROLE;
GRANT app_admin TO app_user WITH INHERIT FALSE, SET TRUE;

-- Later, when admin work is needed:
SET ROLE app_admin;
-- ... do admin work ...
RESET ROLE;
```

### INHERIT vs SET — Three-Case Demo

**Setup:**

```sql
CREATE ROLE reporting NOLOGIN;
CREATE ROLE alice LOGIN PASSWORD 'secret';
CREATE TABLE sales(id int);
REVOKE ALL ON sales FROM PUBLIC;
GRANT SELECT ON sales TO reporting;
```

#### Case 1: INHERIT TRUE, SET TRUE

```sql
GRANT reporting TO alice WITH INHERIT TRUE, SET TRUE;

-- After login as alice:
SELECT * FROM sales;    -- ✅ works immediately (inherited)
SET ROLE reporting;     -- ✅ allowed
```

**Good fit**: Convenience-oriented group roles (`read_only`, `analyst`, `app_writer`) in trusted environments.

#### Case 2: INHERIT FALSE, SET TRUE

```sql
REVOKE reporting FROM alice;
GRANT reporting TO alice WITH INHERIT FALSE, SET TRUE;

-- After login as alice:
SELECT * FROM sales;    -- ❌ fails (not inherited)
SET ROLE reporting;     -- ✅ allowed
SELECT * FROM sales;    -- ✅ now works
```

**Good fit**: Higher-risk roles (maintenance, data export, incident response) where you want "normal mode" vs "elevated mode".

#### Case 3: INHERIT TRUE, SET FALSE

```sql
REVOKE reporting FROM alice;
GRANT reporting TO alice WITH INHERIT TRUE, SET FALSE;

-- After login as alice:
SELECT * FROM sales;    -- ✅ works immediately (inherited)
SET ROLE reporting;     -- ❌ fails
```

**Good fit**: A "pure permission bundle" where you don't want explicit identity switching. Avoid letting such a role own SQL objects (to reduce escalation paths).

---

## What to Learn Next

1. **Row-Level Security (RLS)** — Policies that restrict which rows a role can see or modify.
2. **`pg_hba.conf`** — Host-based authentication that controls *who can connect* from *where*.
3. **Default privileges** — `ALTER DEFAULT PRIVILEGES` for automatically granting permissions on future objects.
4. **Schema-level security** — Using `GRANT USAGE ON SCHEMA` and `REVOKE CREATE ON SCHEMA`.
5. **Audit logging** — Tracking who did what with `pgaudit` extension.
