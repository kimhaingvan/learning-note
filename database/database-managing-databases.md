# Managing PostgreSQL Databases

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> How PostgreSQL organizes databases within a cluster, template databases, creating/dropping databases, per-database configuration, and tablespaces.

---

## Table of Contents

1. [Intuition — The Hierarchy](#1-intuition--the-hierarchy)
2. [What Is a Database in PostgreSQL?](#2-what-is-a-database-in-postgresql)
3. [Why This Matters](#3-why-this-matters)
4. [When to Use Databases vs Schemas](#4-when-to-use-databases-vs-schemas)
5. [How — Common Operations](#5-how--common-operations)
6. [Template Databases: template1 vs template0](#6-template-databases-template1-vs-template0)
7. [Per-Database Configuration Defaults](#7-per-database-configuration-defaults)
8. [Dropping a Database](#8-dropping-a-database)
9. [Tablespaces (Overview)](#9-tablespaces-overview)
10. [Quick Reference Examples](#10-quick-reference-examples)

---

## 1. Intuition — The Hierarchy

Think in **layers**:

```
PostgreSQL Cluster
├── Database A
│   ├── Schema: public
│   │   ├── Table: users
│   │   └── Table: orders
│   └── Schema: analytics
│       └── Table: events
├── Database B
│   └── Schema: public
│       └── Table: products
├── Roles (cluster-wide)
└── Tablespaces (cluster-wide)
```

**Key rules**:
- A client connection is **inside exactly one database** at a time.
- You do **not** hop across databases in one normal SQL session.
- If apps need to share objects closely, **one database + multiple schemas** is often cleaner than multiple databases.

---

## 2. What Is a Database in PostgreSQL?

### Object Hierarchy

```
cluster → database → schema → table (and other objects)
```

### Cluster-Level Objects

Some names are defined at the **cluster level** (stored in `pg_global` tablespace):
- **Roles** (users and groups)
- **Databases**
- **Tablespaces**

### Isolation

- Databases are **isolated from each other** from the user's perspective, but they share cluster-level objects.
- A client must specify a **database name** when connecting.
- One connection can access **only one database**.
- A client can open **multiple connections** to different databases.

### Cross-Database Access

Cross-database access is **not native** within a single connection, but PostgreSQL provides:
- `postgres_fdw` (Foreign Data Wrapper)
- `dblink` extension

### Security

Database-level security has two pieces:
- **Access control** at connection time (via `pg_hba.conf`)
- **Authorization control** via the `GRANT` system once connected

> **SQL standard note**: A PostgreSQL "database" is practically what the SQL standard calls a **catalog**.

---

## 3. Why This Matters

### Choosing the Wrong Boundary Causes Problems

| Use Case | Recommended Approach |
|----------|---------------------|
| Unrelated projects/user groups | **Separate databases** |
| Interrelated projects/users | **One database**, separated by **schemas** and privileges |

### Important Administration Facts

- Multiple databases in one cluster share the **same WAL** — they are not fully independent for backup/recovery.
- By default, users can often connect broadly unless you explicitly restrict access controls.

### Mental Rule

| Concept | Purpose |
|---------|---------|
| **Database** | Stronger logical **separation** |
| **Schema** | Modular **namespace** inside shared cooperation |

> **Common misunderstanding**: "Different databases in one cluster" are **NOT** the same as "fully separate PostgreSQL servers." They share WAL, roles, and tablespace configuration.

---

## 4. When to Use Databases vs Schemas

| Scenario | Use |
|----------|-----|
| Projects/user groups should be **mostly unaware** of each other | Separate **databases** |
| Projects/users are **related** and should share resources | One database, separate **schemas** |
| Need a **pristine** new database (no local additions) | Create from `template0` |
| Need different **encoding/locale** settings | Create from `template0` |
| A config parameter should default **differently for one database** | `ALTER DATABASE ... SET` |
| Need physical **storage placement control** | Tablespaces |

---

## 5. How — Common Operations

### List Databases

```sql
-- SQL
SELECT datname FROM pg_database;

-- psql meta-command
\l

-- Command line
psql -l
```

### Create a Database

```sql
-- Basic form (owner defaults to current role)
CREATE DATABASE dbname;

-- Shell wrapper
-- createdb dbname
```

**Notes**:
- You must be **superuser** or have `CREATEDB` privilege.
- `CREATE DATABASE` **cannot run inside a transaction block**.
- By default, PostgreSQL **copies `template1`**.

---

## 6. Template Databases: template1 vs template0

| Template | Purpose | Modifiable? |
|----------|---------|-------------|
| `template1` | Default source for new databases. Your **customizable default template**. | Yes |
| `template0` | Pristine baseline (**factory reset**). | No — should not be modified. |

### How Template Copying Works

- Objects added to `template1` are **copied into every database** created from it afterward.
- **Database-level `GRANT` permissions** and **database-level config settings** are **NOT copied**.
- Copying requires **exclusivity** on the source database — no other sessions may be connected during the copy.

### Creating from template1 (Default)

```sql
-- These two are equivalent:
CREATE DATABASE appdb TEMPLATE template1;
CREATE DATABASE appdb;  -- template1 is the default
```

### Why template1 Matters (Propagation Demo)

```sql
-- Connect to template1 as a superuser
\c template1

-- Add something you want every future database to inherit
CREATE SCHEMA shared_tools;

-- Create a new database
CREATE DATABASE demo1 TEMPLATE template1;

-- Verify the copied object
\c demo1
\dn   -- shared_tools schema should appear
```

### When to Use template0

Use `template0` when you need:
- A **pristine** database without any `template1` customizations.
- A **restore target** that avoids duplicate-object conflicts from `template1` additions.
- **Different encoding/locale settings** than `template1` allows.

```sql
CREATE DATABASE music
  LOCALE 'sv_SE.utf8'
  TEMPLATE template0;
```

### Practical Caution

If you customize `template1`, restoring a dump into a new database created from `template1` can cause **duplicate-object conflicts**. In that scenario, create the restore target from `template0`.

---

## 7. Per-Database Configuration Defaults

```sql
-- Set a default for future sessions in this database
ALTER DATABASE mydb SET geqo TO off;

-- Reset to the system default
ALTER DATABASE mydb RESET geqo;
```

> These are defaults for **subsequent connections**, not applied immediately to the current session.

---

## 8. Dropping a Database

```sql
-- Basic
DROP DATABASE dbname;

-- Safer scripting (no error if it doesn't exist)
DROP DATABASE IF EXISTS dbname;

-- Force disconnect existing sessions first
DROP DATABASE dbname WITH (FORCE);
```

**Notes**:
- You **cannot drop** the database you are currently connected to.
- `DROP DATABASE` **cannot run inside a transaction block**.
- `FORCE` can still fail if the database has **prepared transactions**, **active logical replication slots**, or **subscriptions**.

---

## 9. Tablespaces (Overview)

- A database can have a **default tablespace** for storing new objects.
- Built-in tablespaces:
  - `pg_global` — cluster-wide shared objects
  - `pg_default` — default storage location

> **Caveat**: Tablespaces are integral to the cluster. **Losing one can break the entire cluster.** Use them only with clear operational discipline.

---

## 10. Quick Reference Examples

```sql
-- List existing databases
SELECT datname FROM pg_database;
\l
-- psql -l

-- Create a normal database
CREATE DATABASE appdb;

-- Create a database for another owner
CREATE DATABASE appdb OWNER appuser;
-- createdb -O appuser appdb

-- Create a pristine locale-specific database
CREATE DATABASE music
  LOCALE 'sv_SE.utf8'
  TEMPLATE template0;

-- Set a per-database default parameter
ALTER DATABASE mydb SET geqo TO off;

-- Drop a database with forced disconnect
DROP DATABASE olddb WITH (FORCE);
```

---

## What to Learn Next

1. **Schemas** — `CREATE SCHEMA`, `search_path`, and using schemas for multi-tenant applications.
2. **`pg_hba.conf`** — Controlling which roles can connect to which databases from which hosts.
3. **Backup strategies** — `pg_dump` (per database) vs `pg_dumpall` (cluster-wide) vs PITR.
4. **Logical replication** — Replicating specific databases or tables across clusters.
5. **Connection pooling** — Using PgBouncer to manage connections across multiple databases efficiently.
