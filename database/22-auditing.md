# Auditing

> **Level**: Intermediate–Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 8.6 — Auditing  
> Tracking who did what and when — using built-in logging, pgAudit extension, pg_stat_statements, and event triggers.

---

## Table of Contents

1. [Why Auditing Matters](#1-why-auditing-matters)
2. [Built-in Statement Logging](#2-built-in-statement-logging)
3. [pgAudit Extension](#3-pgaudit-extension)
   - [Session Auditing](#session-auditing)
   - [Object Auditing](#object-auditing)
4. [pg_stat_statements](#4-pg_stat_statements)
5. [DDL Auditing with Event Triggers](#5-ddl-auditing-with-event-triggers)
6. [DML Auditing with Table Triggers](#6-dml-auditing-with-table-triggers)
7. [Best Practices](#7-best-practices)

---

## 1. Why Auditing Matters

| Need | What Auditing Provides |
|------|----------------------|
| **Security** | Detect unauthorized access or data exfiltration |
| **Compliance** | Meet GDPR, HIPAA, SOX, PCI-DSS requirements for data access logging |
| **Forensics** | Reconstruct events after a security incident |
| **Debugging** | Trace which queries modified specific data |
| **Accountability** | Know which user/role performed each action |

---

## 2. Built-in Statement Logging

PostgreSQL can log SQL statements via `postgresql.conf` settings. No extensions required.

### log_statement

```ini
# postgresql.conf
log_statement = 'none'    # none | ddl | mod | all
```

| Value | What It Logs | Use Case |
|-------|-------------|----------|
| `none` | Nothing | Default — no overhead |
| `ddl` | `CREATE`, `ALTER`, `DROP`, etc. | Schema change tracking |
| `mod` | DDL + `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE` | Data modification tracking |
| `all` | Every statement | Full audit trail (high I/O cost) |

### Per-Role Logging

```sql
-- Enable full logging for a specific role only
ALTER ROLE audited_user SET log_statement = 'all';

-- Normal roles keep the default
ALTER ROLE app_user SET log_statement = 'none';
```

### Related Logging Settings

```ini
# Log statements that take longer than N ms (useful for slow query detection)
log_min_duration_statement = 1000    # log queries > 1 second

# Log all statement durations (even fast ones)
log_duration = on

# Include useful context in log lines
log_line_prefix = '%t [%p] %u@%d '    # timestamp, pid, user@database

# Log connections and disconnections
log_connections = on
log_disconnections = on

# Log lock waits longer than deadlock_timeout
log_lock_waits = on

# Log checkpoints
log_checkpoints = on

# Log temp file creation (indicates work_mem too low)
log_temp_files = 0    # log all temp files; set to N to log only files > N kB
```

### Viewing Logs

```bash
# Log location (depends on configuration)
# Check with:
SHOW log_directory;
SHOW log_filename;
SHOW logging_collector;

# Typical locations:
# /var/log/postgresql/postgresql-18-main.log
# /var/lib/pgsql/18/data/log/postgresql-*.log

# Tail logs
tail -f /var/log/postgresql/postgresql-18-main.log
```

### Limitations

- **No object-level granularity** — cannot log only access to specific tables.
- **Output is plain text** — hard to parse and analyze at scale.
- **No separation of privilege types** — logs the full statement but doesn't categorize access type.
- **Superuser bypass** — A superuser can change `log_statement` to disable logging.

---

## 3. pgAudit Extension

`pgAudit` provides **structured, granular audit logging** that addresses the limitations of built-in logging.

### Installation

```bash
# Debian/Ubuntu
sudo apt install postgresql-18-pgaudit

# RHEL/CentOS
sudo yum install pgaudit_18
```

```ini
# postgresql.conf
shared_preload_libraries = 'pgaudit'    # requires restart
```

```sql
-- Enable the extension
CREATE EXTENSION pgaudit;
```

### Session Auditing

Logs all statements matching specified **statement classes** for the session.

```ini
# postgresql.conf — log all DDL and write operations
pgaudit.log = 'ddl, write'
```

| Class | What It Covers |
|-------|---------------|
| `READ` | `SELECT`, `COPY TO` |
| `WRITE` | `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `COPY FROM` |
| `FUNCTION` | Function calls and `DO` blocks |
| `ROLE` | `GRANT`, `REVOKE`, `CREATE/ALTER/DROP ROLE` |
| `DDL` | All DDL not covered by `ROLE` |
| `MISC` | `DISCARD`, `FETCH`, `CHECKPOINT`, etc. |
| `MISC_SET` | `SET` commands |
| `ALL` | All of the above |

```ini
# Log everything for thorough auditing
pgaudit.log = 'all'

# Log only schema changes and role modifications
pgaudit.log = 'ddl, role'

# Per-role: audit a specific user more closely
ALTER ROLE sensitive_user SET pgaudit.log = 'all';
```

Additional settings:

```ini
# Include the statement text (default: true)
pgaudit.log_statement_once = on     # avoid repeating statement for multi-line results

# Include object name/type
pgaudit.log_relation = on           # log the specific table/view accessed

# Log catalog (system table) access
pgaudit.log_catalog = off           # usually off to reduce noise

# Log parameters (bound values in prepared statements)
pgaudit.log_parameter = on          # shows actual values, not just $1, $2

# Log only for a specific role
pgaudit.role = 'auditor'            # see Object Auditing below
```

### Session Audit Log Output

```
AUDIT: SESSION,1,1,DDL,CREATE TABLE,TABLE,public.orders,
    "CREATE TABLE orders (id serial PRIMARY KEY, total numeric);"

AUDIT: SESSION,2,1,WRITE,INSERT,TABLE,public.orders,
    "INSERT INTO orders (total) VALUES (99.99);"

AUDIT: SESSION,3,1,READ,SELECT,TABLE,public.orders,
    "SELECT * FROM orders WHERE id = 1;"
```

| Field | Meaning |
|-------|---------|
| SESSION/OBJECT | Audit type |
| Statement ID | Unique statement counter |
| Sub-statement ID | For nested statements |
| Class | READ/WRITE/DDL/etc. |
| Command | Specific SQL command |
| Object Type | TABLE/VIEW/FUNCTION/etc. |
| Object Name | Schema-qualified object name |
| Statement | The SQL text |

### Object Auditing

Logs access to **specific objects** (tables, views, functions) based on privileges granted to an **audit role**. This is the most granular approach.

```sql
-- 1. Create an audit role (does not need LOGIN)
CREATE ROLE auditor NOLOGIN;

-- 2. Grant privileges on objects you want audited
GRANT SELECT, INSERT, UPDATE, DELETE ON sensitive_data TO auditor;
GRANT SELECT ON financial_reports TO auditor;
```

```ini
# 3. postgresql.conf — tell pgAudit to use this role
pgaudit.role = 'auditor'
```

Now any access to `sensitive_data` or `financial_reports` by **any user** is logged — because pgAudit checks whether the `auditor` role has a matching privilege.

```
AUDIT: OBJECT,1,1,READ,SELECT,TABLE,public.sensitive_data,
    "SELECT * FROM sensitive_data WHERE id = 42;"
```

> **Object auditing** is preferred for production because it's **targeted** — you only log access to sensitive objects, not every single query.

---

## 4. pg_stat_statements

Tracks **execution statistics** for all normalized SQL statements. Not a true audit tool (doesn't log who/when), but invaluable for forensics and performance analysis.

### Setup

```ini
# postgresql.conf
shared_preload_libraries = 'pg_stat_statements'    # requires restart
pg_stat_statements.max = 10000                      # max tracked statements
pg_stat_statements.track = 'all'                    # none | top | all
```

```sql
CREATE EXTENSION pg_stat_statements;
```

### Key Queries

```sql
-- Top 10 slowest queries by total time
SELECT
    calls,
    round(total_exec_time::numeric, 2) AS total_ms,
    round(mean_exec_time::numeric, 2) AS avg_ms,
    rows,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Most frequently executed queries
SELECT calls, query
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Queries with the most shared buffer reads (disk I/O)
SELECT
    calls,
    shared_blks_hit,
    shared_blks_read,
    round(100.0 * shared_blks_hit / NULLIF(shared_blks_hit + shared_blks_read, 0), 2) AS cache_hit_pct,
    query
FROM pg_stat_statements
ORDER BY shared_blks_read DESC
LIMIT 10;

-- Reset statistics
SELECT pg_stat_statements_reset();
```

### What It Tracks

| Column | Meaning |
|--------|---------|
| `calls` | Number of times executed |
| `total_exec_time` | Total execution time (ms) |
| `mean_exec_time` | Average time per call |
| `rows` | Total rows returned/affected |
| `shared_blks_hit` | Shared buffer cache hits |
| `shared_blks_read` | Blocks read from disk |
| `temp_blks_read/written` | Temp file usage (spill to disk) |

> **Forensic use**: After an incident, check which queries ran most frequently or consumed the most resources. Combine with log timestamps to narrow the window.

---

## 5. DDL Auditing with Event Triggers

For fine-grained DDL auditing with custom logic, use **event triggers** (covered in [19-stored-procedures-functions-triggers.md](19-stored-procedures-functions-triggers.md)).

```sql
-- Audit table for DDL events
CREATE TABLE ddl_audit_log (
    id         serial PRIMARY KEY,
    event_time timestamptz DEFAULT now(),
    username   text DEFAULT current_user,
    command    text,
    object_type text,
    schema_name text,
    object_name text
);

-- Event trigger function
CREATE OR REPLACE FUNCTION log_ddl_event()
RETURNS event_trigger
LANGUAGE plpgsql
AS $$
DECLARE
    v_obj record;
BEGIN
    FOR v_obj IN SELECT * FROM pg_event_trigger_ddl_commands() LOOP
        INSERT INTO ddl_audit_log (command, object_type, schema_name, object_name)
        VALUES (v_obj.command_tag, v_obj.object_type, v_obj.schema_name, v_obj.object_identity);
    END LOOP;
END;
$$;

-- Attach to DDL events
CREATE EVENT TRIGGER audit_ddl_changes
    ON ddl_command_end
    EXECUTE FUNCTION log_ddl_event();
```

```sql
-- Test: create a table
CREATE TABLE test_audit (id serial);

-- Check the log
SELECT * FROM ddl_audit_log;
--  event_time          | username | command      | object_type | schema_name | object_name
--  2026-03-27 14:00:00 | admin    | CREATE TABLE | table       | public      | public.test_audit
```

---

## 6. DML Auditing with Table Triggers

Track row-level changes (INSERT/UPDATE/DELETE) on specific tables.

```sql
-- Generic audit table
CREATE TABLE data_audit_log (
    id           serial PRIMARY KEY,
    event_time   timestamptz DEFAULT now(),
    table_name   text NOT NULL,
    operation    text NOT NULL,
    username     text DEFAULT current_user,
    old_data     jsonb,
    new_data     jsonb
);

-- Generic audit trigger function (reusable across tables)
CREATE OR REPLACE FUNCTION audit_data_changes()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO data_audit_log (table_name, operation, new_data)
        VALUES (TG_TABLE_NAME, 'INSERT', row_to_json(NEW)::jsonb);
        RETURN NEW;
    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO data_audit_log (table_name, operation, old_data, new_data)
        VALUES (TG_TABLE_NAME, 'UPDATE', row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb);
        RETURN NEW;
    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO data_audit_log (table_name, operation, old_data)
        VALUES (TG_TABLE_NAME, 'DELETE', row_to_json(OLD)::jsonb);
        RETURN OLD;
    END IF;
END;
$$;

-- Attach to sensitive tables
CREATE TRIGGER trg_audit_orders
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW EXECUTE FUNCTION audit_data_changes();

CREATE TRIGGER trg_audit_users
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW EXECUTE FUNCTION audit_data_changes();
```

> **Performance note**: Row-level audit triggers add overhead to every write. For high-write tables, consider logging to an unlogged table and periodically flushing to persistent storage, or use `pgAudit` for statement-level logging instead.

---

## 7. Best Practices

### Choosing the Right Tool

| Need | Tool | Why |
|------|------|-----|
| Compliance audit trail | **pgAudit** (object mode) | Structured, targeted, parseable |
| Schema change tracking | **Event triggers** or `log_statement = 'ddl'` | Captures DDL with context |
| Row-level change history | **Table-level audit triggers** | Stores old/new values as JSONB |
| Performance forensics | **pg_stat_statements** | Aggregated query statistics |
| Quick debugging | `log_statement = 'all'` or `log_min_duration_statement` | Low setup cost |

### Operational

| Practice | Why |
|----------|-----|
| Log to a separate partition or volume | Prevent audit logs from filling the data disk |
| Use `log_line_prefix` with timestamp, user, database, PID | Makes logs parseable |
| Ship logs to a SIEM (Splunk, ELK, CloudWatch) | Centralized analysis and alerting |
| Protect audit tables with restrictive permissions | Prevent audited users from modifying audit logs |
| Separate audit role from application role | Application cannot disable its own auditing |
| Set retention policies | Comply with regulations while managing storage |
| Monitor `pg_stat_statements` regularly | Catch unusual query patterns early |

### Securing Audit Data

```sql
-- Audit table owned by a dedicated role
ALTER TABLE data_audit_log OWNER TO audit_admin;
ALTER TABLE ddl_audit_log OWNER TO audit_admin;

-- Revoke all from other roles
REVOKE ALL ON data_audit_log FROM PUBLIC;
REVOKE ALL ON ddl_audit_log FROM PUBLIC;

-- Only audit_admin can read; no one can modify
GRANT SELECT ON data_audit_log TO security_team;
-- No UPDATE, DELETE, or TRUNCATE grants to anyone

-- Enable RLS for extra protection
ALTER TABLE data_audit_log ENABLE ROW LEVEL SECURITY;
ALTER TABLE data_audit_log FORCE ROW LEVEL SECURITY;
CREATE POLICY audit_read_only ON data_audit_log
    FOR SELECT TO security_team
    USING (true);
-- No INSERT/UPDATE/DELETE policies → only the trigger (running as table owner) can write
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `log_statement = 'all'` in production without managing log storage | Disk fills up quickly | Use `pgAudit` with targeted object auditing, or `log_statement = 'ddl'` |
| Not logging `log_connections` | No record of who connected and when | Enable `log_connections = on` |
| Audit table in the same schema as application data | Audited users might have access | Put audit tables in a separate schema with restricted access |
| Using AFTER triggers on unlogged tables | Audit data lost on crash | Use regular (logged) tables for audit storage |
| Not rotating logs | Single log file grows unbounded | Configure `log_rotation_age` and `log_rotation_size` |
| Forgetting `shared_preload_libraries` | Extension not loaded → `CREATE EXTENSION` fails | Add to `postgresql.conf` and restart |

---

## What to Learn Next

1. **Roles & Privileges** — Access control fundamentals — see [08-roles.md](08-roles.md).
2. **Row-Level Security** — Per-row access control — see [12-policy.md](12-policy.md).
3. **Authentication & SSL** — Connection security — see [21-authentication-ssl.md](21-authentication-ssl.md).
4. **Triggers** — Event triggers and table triggers in detail — see [19-stored-procedures-functions-triggers.md](19-stored-procedures-functions-triggers.md).
5. **Backup & Restore** — Disaster recovery — Phase 9.

---

> *Ref: [Docs — Error Reporting and Logging](https://www.postgresql.org/docs/18/runtime-config-logging.html) · [pgAudit — GitHub](https://github.com/pgaudit/pgaudit) · [Docs — pg_stat_statements](https://www.postgresql.org/docs/18/pgstatstatements.html) · [Docs — Event Triggers](https://www.postgresql.org/docs/18/event-triggers.html)*
