# Backup and Restore (PostgreSQL 18)

> **Level**: Intermediate | **Tone**: Technical / Teacher-like  
> A detailed guide to PostgreSQL's three backup families — SQL Dump, File System Level Backup, and Continuous Archiving / PITR — including WAL archiving, incremental backups, timelines, and running backup commands non-interactively.

---

## Table of Contents

1. [Backup Strategy Overview](#1-backup-strategy-overview)
2. [SQL Dump — pg_dump](#2-sql-dump--pg_dump)
3. [pg_dumpall — Whole Cluster Logical Backup](#3-pg_dumpall--whole-cluster-logical-backup)
4. [Handling Large Databases](#4-handling-large-databases)
5. [File System Level Backup](#5-file-system-level-backup)
6. [Continuous Archiving / PITR](#6-continuous-archiving--pitr)
   - [Setting Up WAL Archiving](#61-setting-up-wal-archiving)
   - [Making a Base Backup](#62-making-a-base-backup)
   - [Incremental Backup (PostgreSQL 18)](#63-incremental-backup--postgresql-18)
   - [Low-Level Base Backup API](#64-low-level-base-backup-api)
   - [Recovering Using a Continuous Archive Backup](#65-recovering-using-a-continuous-archive-backup)
   - [Timelines](#66-timelines)
   - [Tips and Examples](#67-tips-and-examples)
   - [Caveats](#68-caveats)
7. [Running Backup Commands Non-Interactively](#7-running-backup-commands-non-interactively)
8. [Master Comparison](#8-master-comparison)
9. [Review Questions](#9-review-questions)

---

## 1. Backup Strategy Overview

### What

PostgreSQL presents **3 backup families** that solve different problems:

| Family | Analogy |
|--------|---------|
| **SQL dump** | "Write the database down as rebuild instructions." The recipe. |
| **File system level backup** | "Pick up the whole machine while it is off." The frozen hard disk. |
| **Continuous archiving / PITR** | "Take a machine image, then keep every change log so you can replay to the right moment." The frozen hard disk + security camera footage. |

### Key Distinction: Logical vs Physical

| Type | What It Understands | Portability |
|------|-------------------|-------------|
| **Logical backup** (`pg_dump`, `pg_dumpall`) | SQL objects | Portable across versions and architectures |
| **Physical backup** (file copy, PITR) | Cluster storage state | Server-version-specific; restores whole cluster |

### Which Method for Which Problem?

These are **not the same backup problem**:

- *"How do I back up production every night?"* → PITR
- *"How do I migrate to a newer PostgreSQL?"* → SQL dump
- *"How do I recover to exactly 10:31 AM before a bad script?"* → PITR

### Quick Decision Guide

| Scenario | Method |
|----------|--------|
| Move to a newer PostgreSQL version | `pg_dump` |
| Back up one DB or selected objects | `pg_dump` |
| Whole cluster logically, including roles/tablespaces | `pg_dumpall` |
| Whole cluster with clean shutdown | File system copy |
| Production disaster recovery and point-in-time restore | Base backup + WAL / PITR |
| PostgreSQL 18 physical backup optimization for large slowly changing datasets | Incremental backup |

> **Memory aid**: logical = portable | physical = fast recovery | PITR = time travel

---

## 2. SQL Dump — pg_dump

### What

`pg_dump` writes out SQL commands (`CREATE TABLE`, `INSERT`, `ALTER`, privileges, ownership, schema definitions) that can rebuild the database as it looked when the dump began.

- Dumps **one database at a time**.
- Is a **normal PostgreSQL client** — takes a consistent snapshot, does not generally block readers/writers.
- Can still conflict with operations that need exclusive locks (e.g., many `ALTER TABLE` forms).
- Does **not have special backup privilege** — must be able to read the objects it dumps (almost always needs superuser or equivalent).

**Major strength**: Output can generally be restored into **newer PostgreSQL versions** and across machine architectures (e.g., 32-bit to 64-bit).

### When to Use

- You need portability.
- You need object-level backup/restore flexibility.
- You want safer cross-version migration.
- Typical use cases: migrate old → new server, export one database, move selected schemas/tables, create a reviewable backup artifact.

### Connection Behavior

```bash
pg_dump dbname > dumpfile

# Connection options
-h   # host
-p   # port
-U   # user
# Defaults can come from PGHOST, PGPORT, PGUSER
```

### Restore: Plain-Text Dump

```bash
# Target database must already exist, created from template0
createdb -T template0 mydb_restore

# Use -X so .psqlrc does not interfere
psql -X mydb_restore < dumpfile

# Stop on first error
psql -X --set ON_ERROR_STOP=on mydb_restore < dumpfile

# Restore in a single transaction (fully succeeds or fully rolls back)
psql -X -1 mydb_restore < dumpfile
```

### Direct Server-to-Server Pipe

```bash
pg_dump -h host1 dbname | psql -X -h host2 dbname
```

### Dump Formats

| Format | Flag | Compressed | Restore Tool | Notes |
|--------|------|-----------|-------------|-------|
| Plain SQL | (default) | No | `psql` | Human-readable |
| Custom | `-Fc` | Yes (default) | `pg_restore` | Flexible for selective restore |
| Directory | `-Fd` | Yes (default) | `pg_restore` | Supports parallel dump `-j N` |
| Tar | `-Ft` | No | `pg_restore` | Less flexible than custom/directory |

### Parallelism

```bash
# Parallel dump (directory format only)
pg_dump -Fd mydb -j 4 -f dumpdir

# Parallel restore (custom or directory format)
pg_restore -j 4 -d mydb dumpdir
# Note: not compatible with --single-transaction
```

### Relationship with PITR

- `pg_dump` is **not part of PITR**.
- Logical dumps do **not** contain the physical information needed for WAL replay.

### Example

```bash
# Basic: dump and restore
pg_dump mydb > mydb.sql
createdb -T template0 mydb_restore
psql -X mydb_restore < mydb.sql

# More realistic: custom format for flexibility
pg_dump -Fc mydb > mydb.dump
pg_restore -C -d postgres mydb.dump
```

### Pitfalls

- Forgetting to create the target DB from `template0`.
- Restoring without pre-creating the needed roles/users.
- Assuming `pg_dump` is ideal for routine large-production disaster backup.
- Assuming `pg_dump` can participate in WAL replay/PITR.
- `pg_dump` and `pg_restore` can execute arbitrary SQL chosen by source superusers — inspect dump before restoring if source superusers are not trusted.

> **Memory aid**: `pg_dump` backs up **meaning, not bytes** — good for moving forward, not the main tool for fast disaster recovery.

---

## 3. pg_dumpall — Whole Cluster Logical Backup

### What

While `pg_dump` handles one database, `pg_dumpall` handles the **whole cluster logically**.

- Dumps **every database** in the cluster.
- Also dumps cluster-wide globals: roles, tablespaces, privilege grants for configuration parameters.

### How

```bash
# Dump everything
pg_dumpall > cluster.sql

# Restore
psql -X -f cluster.sql postgres

# Dump globals only (roles, tablespaces)
pg_dumpall --globals-only > globals.sql
```

### Important Distinctions

- `pg_dumpall` is still **logical** — unlike physical backup, it does not give one physical cluster image.
- It **reconnects once per database**, so database snapshots are not one single synchronized cross-database moment.
- Likely requires **superuser** to dump everything.
- Password auth may prompt repeatedly unless you use `.pgpass`.
- Restore can execute arbitrary source-superuser SQL — trust matters.

> **Memory aid**: `pg_dump` = one database | `pg_dumpall` = all databases + globals

---

## 4. Handling Large Databases

For large databases, plain SQL dumps become awkward. PostgreSQL recommends:

- Use **custom or directory format** for big logical backups.
- Use `pg_restore` for selective or parallel restore.
- Directory format supports **parallel dump jobs** (`-j`).

```bash
# Parallel dump (directory format)
pg_dump -Fd mydb -j 8 -f mydb_dumpdir

# Parallel restore
pg_restore -j 8 -d mydb mydb_dumpdir
```

> **Note**: Large logical dumps are often slower to restore than physical backups, because PostgreSQL must recreate objects and indexes rather than reuse existing raw files.

---

## 5. File System Level Backup

### What

Copying PostgreSQL's raw files directly — the cluster's storage itself.

- Typical example: `tar -cf backup.tar /usr/local/pgsql/data`
- But this is only valid under **strict conditions**.

### When

Appropriate when:
- You can **shut the server down cleanly**.
- You want a **full-cluster raw backup**.
- You do not need partial restore.

### Hard Rules

- **Server must be shut down** — merely blocking connections is not enough.
- Tools like `tar` do not produce an atomic snapshot by themselves.
- Restore also requires server shutdown.

### Why Partial Raw Restore Does Not Work

- Table files alone are **not enough** — they depend on cluster-wide transaction status data (`pg_xact/*`).
- You **cannot restore just one raw table** or one raw database out of the copied files.

### Snapshot-Based Variation

If your filesystem/storage supports a correct consistent frozen snapshot, you may snapshot while PostgreSQL is running:
- When later started, PostgreSQL treats that copy like crash recovery and replays WAL.
- You must **include WAL files**.
- Doing `CHECKPOINT` first can reduce recovery time.

### Multi-Filesystem Caution

- If data, WAL, or tablespaces are on **multiple filesystems**, snapshots must be **simultaneous**.
- If not, snapshot backup may be unsafe — in such cases, continuous-archiving base backup is often better.

### rsync Pattern (Low-Downtime)

```bash
# 1. Run rsync while server is up (reduces what the next sync needs to copy)
rsync -a /var/lib/postgresql/data/ backup/

# 2. Stop server briefly
pg_ctl stop

# 3. Final rsync to catch only the changed files
rsync --checksum -a /var/lib/postgresql/data/ backup/

# 4. Start server again
pg_ctl start
```

### Example

```bash
# With server stopped
pg_ctl stop
tar -cf backup.tar /usr/local/pgsql/data
pg_ctl start
```

### Pitfalls

- Trying to copy while the server is running using ordinary `tar`.
- Trying to restore only one table/database.
- Forgetting tablespaces.
- Trusting multi-volume snapshots that are not truly simultaneous.

> **Memory aid**: raw copy needs raw consistency | whole cluster only | not object-selective

---

## 6. Continuous Archiving / PITR

### What

PostgreSQL constantly writes every data change to WAL in `pg_wal/`. PITR exploits this:

- Take one physical baseline (base backup).
- Keep every WAL change log.
- Replay until the exact moment you want.

```
base backup = starting save file
WAL archive = sequence of moves since then
recovery target = where you stop replay
```

### Why

PITR is the preferred technique for high reliability:
- Not just "do I have a copy?" but also "how much data can I lose?" and "how precisely can I recover?"

### Key Distinctions

- `pg_dump` and `pg_dumpall` are **not usable** as part of continuous archiving.
- PITR restores **whole cluster only** — not selected objects.

> **Memory aid**: PITR = base backup + WAL + stop point

---

### 6.1 Setting Up WAL Archiving

WAL segment files would normally be recycled. Archiving means **"save each completed segment before PostgreSQL reuses it."**

WAL is split into segment files, normally **16 MB each** (unless changed at `initdb`). Without archiving, old segments are recycled. With archiving, each completed segment must be safely copied elsewhere first.

#### Required Configuration

```ini
wal_level = replica          # or higher
archive_mode = on
archive_command = 'test ! -f /mnt/server/archivedir/%f && cp %p /mnt/server/archivedir/%f'
```

| Variable | Meaning |
|----------|---------|
| `%p` | Source path of the WAL file to archive |
| `%f` | WAL filename only (no path) |
| `%%` | Literal `%` |

#### Behavior Rules

- Archive command runs as the **PostgreSQL server OS user**.
- Must return **zero only on success** — nonzero means PostgreSQL retries later.
- If archiving keeps failing, `pg_wal/` **grows until fixed**.
- If `pg_wal/` filesystem fills up because archiving is stuck, PostgreSQL will do a **PANIC shutdown** and remain offline until space is freed.
- Archive logic should **avoid overwriting** existing archive files (PostgreSQL may occasionally re-archive a segment).

#### archive_timeout

```ini
archive_timeout = 60   # seconds — force a segment switch if no WAL generated for this long
```

- Useful when WAL generation is light and you want to limit how long recent transactions remain unarchived.
- But archived files are still **full-sized**, so too-short timeout wastes storage.
- `pg_switch_wal()` can force a switch manually.

#### What WAL Archiving Does NOT Cover

WAL archiving protects database changes made through SQL. It does **not** restore manual changes to:
- `postgresql.conf`
- `pg_hba.conf`
- `pg_ident.conf`

#### Pitfalls

- Using an archive command that silently overwrites files.
- Not monitoring archive lag.
- Setting `archive_timeout` too short.
- Assuming config files are part of WAL recovery.

---

### 6.2 Making a Base Backup

A base backup is the **starting physical copy** for recovery.

#### Tools

`pg_basebackup` is the easiest tool:
- Creates a full base backup as regular files or tar archive.
- Works **while server is running**.
- Always backs up the **entire cluster**.
- Uses the **replication protocol** — needs a user with `REPLICATION` permission or superuser.
- `pg_hba.conf` must allow replication connections.
- `max_wal_senders` must be high enough.

```bash
# Basic base backup
pg_basebackup -D /backup/base1

# Nightly example
pg_basebackup -D /backups/base_2026_03_10
```

#### Backup History File

After each base backup, PostgreSQL creates a **backup history file**:
- Immediately archived into the WAL archive area.
- Named after the first required WAL segment plus an extra suffix.
- Contains: label, backup start/end times, starting/ending WAL segments.

#### Operational Tradeoff

The longer the interval between base backups:
- The **more WAL you must keep**.
- The **longer recovery takes** (more WAL to replay).

#### Standby Backup

`pg_basebackup` can back up from a standby:
- Standby must accept replication connections.
- Primary must have `full_page_writes` enabled.

#### Pitfalls

- Thinking base backup alone is enough for full PITR.
- Forgetting you need WAL from backup start onward.
- Not sizing WAL retention and recovery time together.

> **Memory aid**: base backup = starting frame | WAL = frames after that

---

### 6.3 Incremental Backup — PostgreSQL 18

PostgreSQL 18 lets `pg_basebackup` make **incremental backups** — copying only changed blocks plus metadata, instead of all relation blocks.

#### Requirements

- The **backup manifest** from an earlier backup on the same server is required.
- Server must have the necessary **WAL summary files** (`pg_wal/summaries/`).
- Summaries must cover all LSNs from the prior backup start LSN to the current backup start LSN.

#### How It Works

```bash
# Step 1: Full base backup
pg_basebackup -D /backup/full0

# Step 2: Incremental backup (provide manifest of prior backup)
pg_basebackup --incremental=/backup/full0/backup_manifest -D /backup/inc1

# Step 3: Combine into a synthetic full backup before restore
pg_combinebackup -o /restore/full_synthetic /backup/full0 /backup/inc1
```

#### Restore Model

- Incremental backup **cannot be restored directly**.
- You must keep the incremental backup **and every earlier dependent backup** in the chain.
- Use `pg_combinebackup` to build a **synthetic full backup**.
- Then continue with the normal recovery process using WAL and `recovery.signal`.

#### Failure Conditions

- If required WAL summaries are missing, incremental backup fails.
- Server waits for missing summaries to appear if summarization is just behind.
- If files were removed or the summarizer cannot catch up, backup fails.

#### Dependency-Chain Warning

- PostgreSQL does **not** track backup dependency chains for you.
- If you delete a required earlier backup, later dependent incremental backups may become **unrestorable**.

#### When Is Incremental Backup Worth It?

| Good Fit | Poor Fit |
|----------|----------|
| Relatively large databases | Small databases |
| Substantial portion changes slowly | Heavily modified large databases (almost everything changes) |

#### Standby-Specific Caveat

Incremental backup is only possible if replay would begin from a **later checkpoint** than the previous backup. On a standby, this may fail if there has been too little activity since the previous backup.

#### Pitfalls

- Treating incremental backup as standalone.
- Deleting parent backups too early.
- Forgetting WAL is still required after combine.

> **Memory aid**: incremental = "changed blocks only" | but restore still needs the full family tree

---

### 6.4 Low-Level Base Backup API

The manual workflow behind online physical backup when you need more control than `pg_basebackup`.

#### Steps

```sql
-- 1. Start backup mode
SELECT pg_backup_start(label => 'my_backup', fast => false);
-- The same connection must stay open until backup ends
-- Backup starts at the beginning of a checkpoint

-- 2. Copy files (use tar, cpio, rsync, etc.)
-- Normal database activity may continue during this phase

-- 3. Stop backup and collect metadata
SELECT * FROM pg_backup_stop(wait_for_archive => true);
-- Returns: (lsn, labelfile, spcmapfile)
-- Write second field to 'backup_label' byte-for-byte unchanged
-- Write third field (if nonempty) to 'tablespace_map' byte-for-byte unchanged
```

#### Fast Checkpoint Option

| `fast` Value | Behavior |
|-------------|----------|
| `false` (default) | Wait for normal checkpoint rhythm — less disruptive |
| `true` | Immediate checkpoint — faster start, but heavier I/O impact |

#### `wait_for_archive`

- Default `true` — waits until required WAL has been archived.
- Setting `false` returns earlier, but becomes risky unless you independently verify all required WAL made it to archive.

#### Files to Include / Exclude

| Include | Exclude |
|---------|---------|
| Whole cluster directory | `pg_wal/` |
| External tablespaces | `postmaster.pid`, `postmaster.opts` |
| Symbolic links (preserve them) | `pg_replslot/` (usually) |
| | `pg_dynshmem/`, `pg_notify/`, `pg_serial/`, `pg_snapshots/`, `pg_stat_tmp/`, `pg_subtrans/`, `pgsql_tmp*`, `pg_internal.init` files |

#### Pitfalls

- Losing the session that called `pg_backup_start`.
- Not writing `backup_label` / `tablespace_map`.
- Copying tablespaces incorrectly.
- Including `pg_wal` from the backup copy and confusing restore.

---

### 6.5 Recovering Using a Continuous Archive Backup

#### High-Level Steps

1. **Stop the server** if running.
2. Optionally save old cluster / at least old `pg_wal`.
3. **Remove old data files**.
4. **Restore full backup** directly, or use `pg_combinebackup` for incremental chain first.
5. **Clear out obsolete** `pg_wal`.
6. Copy any **saved unarchived WAL** into `pg_wal`.
7. Set recovery config and **create `recovery.signal`**.
8. **Start the server** — it replays WAL automatically.
9. Inspect the result.

#### Required Recovery Configuration

```ini
# The one absolutely required setting
restore_command = 'cp /mnt/server/archivedir/%f %p'
# %f = wanted WAL filename
# %p = destination path inside recovery process
```

#### Recovery Targets

You can stop recovery at:

| Target | Example |
|--------|---------|
| Date/time | `recovery_target_time = '2026-03-10 09:58:00'` |
| Named restore point | `recovery_target_name = 'before_migration'` |
| Specific transaction ID | `recovery_target_xid = '12345'` |

> **Practical note**: Time and named restore point are **much more usable** in practice than transaction ID, because tooling to identify exact transaction IDs is poor.

#### Important Time Rule

- Recovery target must be **after the end of the base backup**.
- You **cannot recover to a time while that base backup itself was still in progress**.

#### Behavior Rules

- `restore_command` must return **nonzero on failure**.
- Recovery will request files that are not present — **that is normal**.
- An initial missing `00000001.history` can be normal in simple cases.
- Archive copy is **preferred over** files found in `pg_wal/`.
- A normal recovery to latest may end with a "file not found" message — that is often **normal, not a failure**.
- If WAL itself is corrupted, recovery halts and you may need an earlier recovery target.

#### Pitfalls

- Restoring files with wrong owner/permissions.
- Forgetting to remove stale `pg_wal` files from the backup copy.
- Forgetting `recovery.signal`.
- Misreading normal "file not found" end-of-recovery behavior as failure.
- Choosing a target time inside the backup window.

#### Worked Example: Recovering a Dropped Table

**Scenario**: You accidentally drop a table at 11:30. Data was last known good at 11:00. You have a base backup taken at 09:30 and a WAL archive covering everything since.

**Timeline**:
```
09:30  →  pg_basebackup taken
11:00  →  last INSERT (rows 3 and 4 added)
11:30  →  DROP TABLE khach_hang  ← disaster
```

**Step 1 — Stop the server and remove old data:**
```bash
pg_ctl stop
rm -rf /var/lib/pgsql/data
mkdir /var/lib/pgsql/data
chmod 700 /var/lib/pgsql/data
```

**Step 2 — Restore the base backup:**
```bash
tar -xzf /backup/base.tar.gz -C /var/lib/pgsql/data
```
At this point you have data as of 09:30 — before the rows at 11:00 and before the DROP.

**Step 3 — Configure recovery in `postgresql.conf`:**
```ini
restore_command = 'cp /backup/archive/%f "%p"'
recovery_target_time = '2021-06-01 11:00:00'
```
`restore_command` tells PostgreSQL where to find the archived WAL files.
`recovery_target_time` tells it to stop replaying at 11:00 — **after** the inserts, **before** the DROP.

**Step 4 — Clear stale WAL and create the signal file:**
```bash
rm -rf /var/lib/pgsql/data/pg_wal/*    # stale WAL from the base backup
touch /var/lib/pgsql/data/recovery.signal
```
`recovery.signal` tells PostgreSQL to enter recovery mode on startup.

**Step 5 — Start PostgreSQL:**
```bash
pg_ctl start
```
PostgreSQL replays archived WAL segments one by one, stops at 11:00, and opens normally.

**Step 6 — Verify:**
```sql
\c testdb
SELECT * FROM khach_hang;
-- id | name
-- ----+--------------
--   1 | nguyen van a
--   2 | pham van b
--   3 | nguyen van c
--   4 | nguyen van d
```
All four rows are restored. The DROP that happened at 11:30 never occurred in this timeline.

> **Key rule**: `recovery_target_time` must be **after** the base backup completed (09:30 here) and **before** the event you want to undo (11:30 DROP). In this case, 11:00 is the safe spot.

---

### 6.6 Timelines

#### What

Point-in-time recovery creates **branching history**. After archive recovery completes, PostgreSQL creates a **new timeline** and embeds the timeline ID in WAL filenames.

```
WAL filename example:  0000000100001234000055CD
                       ^^^^^^^^ = timeline ID (hex)
```

#### Why

Without timelines, repeated recovery experiments would overwrite or confuse earlier history. With timelines:
- You can recover again from earlier branches later.
- Each new timeline gets a **timeline history file** (archived like WAL — cheap to keep indefinitely).
- History files show where the branch came from.

> **Memory aid**: timeline = branch of database history | like Git branches for WAL history

#### Behavior

- Default recovery behavior: recover to the **latest timeline** found.
- `recovery_target_timeline` can select `current` or a specific child timeline ID.
- You cannot recover into a timeline that branched off **before** the base backup you started from.

---

### 6.7 Tips and Examples

#### Standalone Hot Backup

```bash
# -X includes all WAL needed to use that backup
pg_basebackup -D /backup/hot -X stream
```

Faster to backup/restore than `pg_dump`, though larger — and simpler than full PITR if you just want "recent consistent backup."

#### Compressed Archive Logs

```ini
# postgresql.conf
archive_command = 'gzip < %p > /mnt/server/archivedir/%f.gz'
```

```ini
# Recovery
restore_command = 'gunzip < /mnt/server/archivedir/%f.gz > %p'
```

#### Scripted Archive Command

```ini
archive_command = 'local_backup_script.sh "%p" "%f"'
```

- Useful when archiving involves more than one shell command (e.g., off-site copy, batching, monitoring hooks).
- Enabling `logging_collector` helps capture script stderr in server logs for troubleshooting.

#### Good Real-World Pattern

- Simple `archive_command` in `postgresql.conf` pointing to a shell script.
- Real logic inside the script (compression, off-site upload, monitoring).
- Monitoring on failures.
- **Tested restore procedure** — not just tested backup creation.

---

### 6.8 Caveats

#### CREATE DATABASE During Backup Window

`CREATE DATABASE` during a base backup can behave badly if the copied template DB is modified before the backup finishes.

#### Tablespace Absolute Paths

`CREATE TABLESPACE` replays the **literal absolute path**. This can be dangerous when restoring to another machine with a different directory layout.

- After creating or dropping tablespaces, best practice is to **take a new base backup**.

#### full_page_writes

- Default WAL can be bulky because it includes many full-page images (`full_page_writes`).
- Turning it off can reduce archived WAL volume, but the docs **explicitly warn** to read the crash-safety notes before doing that.
- Increasing checkpoint interval parameters can also reduce page-snapshot frequency while keeping `full_page_writes` on.

> **Memory aid**: PITR is powerful, but tablespaces and template DBs are gotcha zones.

---

## 7. Running Backup Commands Non-Interactively

### The Common Confusion

Beginners often mix up:
- "I don't want to **log into psql manually**" — ✅ already the case with these commands
- "I don't want **any connection to PostgreSQL**" — ❌ impossible for these tools

### What These Commands Actually Are

| Command | What It Is | What It Does |
|---------|-----------|-------------|
| `pg_dump` | Regular PostgreSQL **client application** | Connects to DB, reads objects/data, writes dump |
| `createdb` | Wrapper around SQL `CREATE DATABASE` | Connects to server, issues `CREATE DATABASE` |
| `psql` | PostgreSQL **client** | Connects to DB, executes SQL from file |

> **They connect and do their job non-interactively — but they always require a running PostgreSQL server.**

### 1) Run Directly from Shell with Explicit Connection Options

```bash
pg_dump -h localhost -p 5432 -U postgres mydb > mydb.sql
createdb -h localhost -p 5432 -U postgres -T template0 mydb_restore
psql -h localhost -p 5432 -U postgres -X mydb_restore < mydb.sql
```

### 2) Use Environment Variables (Shorter Commands)

libpq environment variables provide default connection parameters:

```bash
export PGHOST=localhost
export PGPORT=5432
export PGUSER=postgres
export PGDATABASE=postgres
# PGPASSWORD is not recommended in shared environments — use a password file

pg_dump mydb > mydb.sql
createdb -T template0 mydb_restore
psql -X mydb_restore < mydb.sql
```

> **Security note**: `PGPASSWORD` is not recommended because some operating systems allow other users to inspect process environment variables. Use a `.pgpass` password file instead.

### 3) Use a Connection URI

```bash
pg_dump "postgresql://postgres@localhost:5432/mydb" > mydb.sql
createdb "postgresql://postgres@localhost:5432/postgres" -T template0 mydb_restore
psql "postgresql://postgres@localhost:5432/mydb_restore" -X < mydb.sql
```

URI format: `postgresql://[user[:password]@][host[:port]][/dbname][?param=value]`

### 4) Remote Host Example

```bash
pg_dump -h db.example.com -p 5432 -U backup_user mydb > mydb.sql
createdb -h db.example.com -p 5432 -U postgres -T template0 mydb_restore
psql -h db.example.com -p 5432 -U postgres -X mydb_restore < mydb.sql
```

### Important Behavior Notes

- `createdb` connects to an **existing database** on the server to issue `CREATE DATABASE mydb_restore` — it does not connect to the new DB (it doesn't exist yet).
- `psql -X mydb_restore < mydb.sql` requires `mydb_restore` to **already exist** before running.
- `pg_dump` connects to the **source database** (`mydb`), not the restore target.

### Pitfalls

- Thinking `pg_dump` can work offline from just database files (it cannot — it is a client).
- Thinking `createdb` is a local filesystem action (it is a wrapper around SQL `CREATE DATABASE`).
- Forgetting that `mydb_restore` must exist before running `psql ... < mydb.sql`.
- Using `PGPASSWORD` casually in shared environments.

> **Memory aid**: No manual psql session? ✅ Yes. No PostgreSQL connection at all? ❌ No.

---

## 8. Master Comparison

| Feature | SQL Dump | File System Backup | PITR |
|---------|----------|-------------------|------|
| **Granularity** | Per database / table | Entire cluster | Entire cluster |
| **Server downtime** | None | Required (without snapshots) | None |
| **Point-in-time recovery** | ❌ | ❌ | ✅ |
| **Cross-version restore** | ✅ | ❌ | ❌ |
| **Speed (large DBs)** | Slow (rebuild SQL) | Fast (raw copy) | Fast base + incremental WAL |
| **Complexity** | Low | Low | High |
| **Selective restore** | ✅ tables/schemas | ❌ whole cluster | ❌ whole cluster |
| **Best for** | Migration, portability | Clean-shutdown full copy | Production disaster recovery |

> **Final memory aid**:
> - **Logical** backup answers: *"How do I rebuild it?"*
> - **Physical** backup answers: *"How do I copy it?"*
> - **PITR** answers: *"How do I go back to the right moment?"*
> - **Incremental** backup answers: *"How do I copy less physical data?"*
>
> But incremental does not remove the need for WAL or a careful restore workflow.

---

## 9. Review Questions

1. Why can `pg_dump` usually restore into newer PostgreSQL versions, while physical backups usually cannot?
2. Why is ordinary raw file copy unsafe while PostgreSQL is running?
3. Why does PITR require both a base backup **and** a WAL archive?
4. What extra restore step does PostgreSQL 18 incremental backup require?
5. Why are timelines necessary after point-in-time recovery?
6. Is `pg_dump` a server-side command or a client application?
7. Why does `createdb` still require a server connection?
8. What is the difference between using `-h/-p/-U` and using `PGHOST/PGPORT/PGUSER`?
9. Why is `PGPASSWORD` not the best long-term habit?

### Mini Practice

For each scenario, choose the best backup method and explain why in 1–2 lines:

1. Upgrade one database from PostgreSQL 15 to PostgreSQL 18.
2. Restore full production cluster to 09:58 AM before a bad deployment.
3. Take a raw whole-cluster backup during a maintenance shutdown.
4. Back up all databases plus roles and tablespaces logically.
5. Reduce backup size for a large, slowly changing cluster in PostgreSQL 18.

---

## What to Learn Next

1. **`pgBackRest`** — A popular third-party tool that simplifies PITR, incremental backups, parallel restore, and retention management.
2. **Streaming replication** — Using WAL for high availability with hot standby servers.
3. **Backup testing** — Regularly restoring backups to verify they actually work (if you haven't tested the restore, you don't have a backup).
4. **`.pgpass` file** — Secure password storage for scripted client connections.
5. **WAL compression** — `wal_compression = lz4` to reduce WAL volume without disabling `full_page_writes`.
