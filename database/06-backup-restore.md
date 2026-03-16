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

### What PITR Really Means

**PITR** stands for **Point-In-Time Recovery**. It is PostgreSQL's method of recovering a database cluster to a chosen moment by combining:

- a **base backup** — a full physical copy of the whole cluster, and
- a **continuous WAL archive** — every change recorded after that backup.

The core idea:

```
base backup  =  photo of the database
WAL archive  =  security camera recording every change after the photo
PITR restore =  start from the photo → replay the recording → stop at 10:32:15
```

This is why PostgreSQL requires a **continuous, unbroken sequence of archived WAL** stretching back to at least the start of the base backup. A gap in WAL means recovery can stop early or fail entirely.

### Key Distinctions

- `pg_dump` / `pg_dumpall` are **not usable** as part of PITR — they are logical backups, not physical ones.
- PITR always restores the **whole cluster** — not a single database or table.
- `wal_level = minimal` is **not sufficient** for PITR; use `replica` or higher.

### The Overall PITR Lifecycle

```
Phase A — Prepare backup infrastructure
  └─ Enable WAL archiving
  └─ Confirm archived WAL is being stored safely
  └─ Test that archived WAL can actually be restored

Phase B — Take a base backup
  └─ pg_basebackup captures a consistent physical cluster snapshot
  └─ backup_label records the exact starting LSN for recovery

Phase C — Normal operation
  └─ PostgreSQL keeps generating WAL
  └─ WAL files continue being archived
  └─ New base backups taken periodically to reduce future recovery time

Phase D — Restore when disaster happens
  └─ Stop PostgreSQL on the restore server
  └─ Replace data directory with the chosen base backup
  └─ Set restore_command (where to fetch archived WAL from)
  └─ Set recovery target (where to stop replay)
  └─ Start PostgreSQL — it replays WAL and stops at the target
  └─ Promote to read-write on a new timeline
```

### What Each Component Means

| Component | Role |
|-----------|------|
| **Base backup** | Full physical cluster snapshot at a starting moment — the foundation of recovery |
| **WAL** | Write-Ahead Log records every change in 16 MB segment files — the recording after the photo |
| **WAL archive** | Safe external storage of past WAL segments — must be complete and continuous |
| **`backup_label`** | Written into the base backup; records the starting LSN so recovery knows exactly where to begin |
| **`restore_command`** | Command PostgreSQL runs during recovery to fetch each archived WAL segment back into place |
| **Recovery target** | The exact stop point: timestamp, restore point, transaction ID, or LSN |
| **Timeline** | A branch of database history created when recovery ends and the cluster resumes normal operation |

### The Most Important Rule

> **A PITR backup is only as good as its WAL archive continuity.**
> A pristine base backup with a gap in archived WAL cannot recover past that gap.
> The only backup that truly matters is one you have actually tested restoring.

### Common Misunderstandings

| Misunderstanding | Reality |
|-----------------|---------|
| "I have `pg_dump`, isn't that enough?" | `pg_dump` is logical backup; PITR needs physical base backup + WAL. They solve different problems. |
| "Can I PITR one table only?" | No — base backup and WAL replay work at whole-cluster level. |
| "Can I use `wal_level = minimal`?" | No — `minimal` does not contain enough information for point-in-time recovery. |
| "Can I skip testing and rely on the config?" | Risky. PostgreSQL recommends testing archiving before taking the first base backup. |

### PITR Flow Diagram

```
Normal operation
─────────────────
Client writes data
      │
      ▼
PostgreSQL writes WAL ──── archive_command ────► WAL archive (/backup/archive/)
      │
      └──── pg_basebackup at time T0 ──────────► Base backup (/backup/base/)

Disaster recovery
─────────────────
Restore base backup (T0) into /var/lib/postgresql/18/main/
      │
      ▼
create recovery.signal  ← tells PostgreSQL to enter archive recovery
      │
      ▼
Start PostgreSQL
      │
      ▼
restore_command fetches WAL segments from /backup/archive/
      │
      ▼
Replay WAL forward from T0
      │
      ▼
Stop at recovery_target_time / restore_point / LSN
      │
      ▼
PostgreSQL removes recovery.signal automatically
      │
      ▼
pg_wal_replay_resume() → database opens on new timeline
```

> **Memory aid**: `Archive first → Base backup second → Keep WAL → Restore base → Replay WAL → Stop at target`
> That sequence reflects the real dependency order: WAL archiving must be working *before* a base backup is useful, and restore needs both together.

### Environment Used in All Examples

The same paths are used consistently throughout every subsection below:

| Path | Purpose |
|------|---------|
| `/var/lib/postgresql/18/main` | PostgreSQL data directory |
| `/backup/archive` | WAL archive destination |
| `/backup/base` | Full base backup destination |
| `/backup/inc1` | Incremental backup destination |
| `/backup/combined` | Combined synthetic backup (for incremental restore) |
| `/var/log/postgresql` | Server log directory |

---

### 6.1 Setting Up WAL Archiving

> **Why this step exists**: The base backup alone is not enough for PITR. It is only the starting point. To move forward to any later moment, PostgreSQL must read the WAL records generated *after* that backup. Without a working archive, those records are lost when PostgreSQL recycles old WAL segments.

WAL segment files would normally be recycled. Archiving means **"save each completed segment before PostgreSQL reuses it."**

WAL is split into segment files, normally **16 MB each** (unless changed at `initdb`). Without archiving, old segments are recycled. With archiving, each completed segment must be safely copied elsewhere first.

#### Required Configuration

```ini
# postgresql.conf
wal_level = replica          # or higher — replica is sufficient for archiving
archive_mode = on
archive_command = 'test ! -f /backup/archive/%f && cp %p /backup/archive/%f'
```

```bash
# archive_mode = on requires a full server restart (not just reload)
pg_ctl restart -D /var/lib/postgresql/18/main

# Verify archiving is active
psql -c "SELECT pg_switch_wal();"      # force-archive current segment
ls /backup/archive/                    # one new file should appear
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

> **Why this step exists**: WAL archiving alone cannot recover a cluster from nothing — you need a starting snapshot to replay WAL *from*. The base backup is that snapshot. It also produces a `backup_label` file recording the exact starting LSN, so recovery knows precisely where to begin reading the WAL archive.

A base backup is the **starting physical copy** for recovery. Without it, WAL replay has no starting frame to build on.

#### Tools

`pg_basebackup` is the easiest tool:
- Creates a full base backup as regular files or tar archive.
- Works **while server is running**.
- Always backs up the **entire cluster**.
- Uses the **replication protocol** — needs a user with `REPLICATION` permission or superuser.
- `pg_hba.conf` must allow replication connections.
- `max_wal_senders` must be high enough.

```bash
# Full base backup — tar format, gzip compressed, WAL streamed, progress shown
# -Ft : tar format  |  -z : gzip  |  -Xs : stream WAL  |  -P : progress
pg_basebackup -D /backup/base -Ft -z -Xs -P

# Nightly example: date-stamped destination
pg_basebackup -D /backup/base_2026_03_10 -Ft -z -Xs -P

# What gets created:
# /backup/base/base.tar.gz         — cluster data + backup_label
# /backup/base/pg_wal.tar.gz       — WAL streamed during backup
# /backup/base/backup_manifest     — list of files + checksums
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

> **Why this step exists**: Full base backups copy every relation block — even blocks unchanged since the last backup. For large databases where most data changes slowly, this wastes time and storage. Incremental backup copies only changed blocks. WAL is still required after combining, so this is an optimization of the base backup step, not a replacement for the full PITR workflow.

PostgreSQL 18 lets `pg_basebackup` make **incremental backups** — copying only changed blocks plus metadata, instead of all relation blocks.

#### Requirements

- The **backup manifest** from an earlier backup on the same server is required.
- Server must have the necessary **WAL summary files** (`pg_wal/summaries/`).
- Summaries must cover all LSNs from the prior backup start LSN to the current backup start LSN.

#### How It Works

```bash
# Step 1: Full base backup (same as any normal base backup)
pg_basebackup -D /backup/base -Ft -z -Xs -P

# Step 2: Incremental backup — provide the manifest from the prior backup
pg_basebackup --incremental=/backup/base/backup_manifest -D /backup/inc1 -Ft -z -Xs -P

# Step 3: Combine into a synthetic full backup before restore
# Arguments: list prior backups in order from oldest to newest
pg_combinebackup -o /backup/combined /backup/base /backup/inc1

# Step 4: Restore exactly like a normal base backup — pg_combinebackup output
# is a plain directory (not tar), so copy directly into the data directory
rm -rf /var/lib/postgresql/18/main
cp -a /backup/combined/. /var/lib/postgresql/18/main/
# Then continue with recovery.signal + restore_command as in section 6.5
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

> **Why this step exists**: `pg_basebackup` handles entering and leaving backup mode automatically. When you need tighter control — custom copy tools, specific exclusions, or snapshot-system integration — you can drive backup mode manually with `pg_backup_start()` / `pg_backup_stop()`. The same rules apply: write out the returned `backup_label` and `tablespace_map` exactly, and ensure all required WAL has reached the archive before releasing the backup.

The manual workflow behind online physical backup when you need more control than `pg_basebackup`.

#### Steps

```sql
-- 1. Start backup mode (keep this connection open until step 3)
SELECT pg_backup_start(label => 'my_backup', fast => false);
-- fast => false : waits for normal checkpoint rhythm (less I/O impact)
-- fast => true  : forces immediate checkpoint (faster start, heavier I/O)
-- Backup starts at the beginning of the checkpoint
```

```bash
# 2. Copy the cluster data directory while the server is still running
#    Exclude pg_wal/ — it is handled separately
tar --exclude=/var/lib/postgresql/18/main/pg_wal \
    -czf /backup/base/base.tar.gz \
    /var/lib/postgresql/18/main
# Normal database activity may continue during this phase
```

```sql
-- 3. Stop backup and retrieve the metadata (same connection as step 1)
SELECT lsn, labelfile, spcmapfile FROM pg_backup_stop(wait_for_archive => true);
-- write labelfile  value → /var/lib/postgresql/18/main/backup_label   (byte-for-byte)
-- write spcmapfile value → /var/lib/postgresql/18/main/tablespace_map (if non-empty)
-- wait_for_archive => true ensures all required WAL has reached the archive
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

> **Why this step exists**: This is the actual PITR restore. The base backup returns the cluster to its state at backup time; WAL replay moves it forward to the chosen target. The base backup alone leaves you "too early" — `restore_command` and the WAL archive are what drive you forward to the exact moment you want.

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
restore_command = 'cp /backup/archive/%f %p'
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

#### Signal Files

PostgreSQL uses signal files — empty marker files placed in the data directory — to decide what startup mode to enter. The server can still start without them, but the behavior is very different:

| Signal file | What PostgreSQL does on startup |
|-------------|--------------------------------|
| `recovery.signal` | Enters **targeted archive recovery**. Replays archived WAL, then stops at the `recovery_target` (or when WAL runs out). After completing, PostgreSQL **removes the file automatically** and opens for normal read-write operation. |
| `standby.signal` | Enters **standby / streaming replica mode**. Keeps replaying WAL continuously from archive and/or primary — never stops on its own. If both files exist, `standby.signal` wins. |
| *(neither)* | **Normal startup** only. PostgreSQL does ordinary crash recovery if needed, but does **not** enter archive recovery or PITR. |

> **Important nuance**: Without a signal file, PostgreSQL can still do **crash recovery** after an unclean shutdown — that is always automatic. What the signal files control is entry into *archive recovery / PITR*, which is a different, explicit operation.

**Easy mental model**:
```
No signal file      →  normal server boot (crash recovery only if needed)
recovery.signal     →  boot, do PITR / targeted archive recovery, then open writable
standby.signal      →  boot as standby and keep recovering indefinitely
```

**PITR startup flow**:
```
Restore base backup
      │
      ▼
Create recovery.signal
      │
      ▼
Set restore_command + optional recovery_target
      │
      ▼
pg_ctl start
      │
      ▼
Replay archived WAL
      │
      ▼
Reach recovery target (or WAL exhausted)
      │
      ▼
PostgreSQL removes recovery.signal automatically
      │
      ▼
Database opens read-write on a new timeline
```

#### Pitfalls

- Restoring files with wrong owner/permissions.
- Forgetting to remove stale `pg_wal` files from the backup copy.
- Forgetting `recovery.signal` — without it, PostgreSQL starts normally and **skips PITR entirely**.
- Misreading normal "file not found" end-of-recovery behavior as failure.
- Choosing a target time inside the backup window.

#### Worked Example: Full End-to-End PITR Scenario

This scenario walks through the **complete lifecycle** — from first-time WAL archiving setup, through taking a base backup, through a disaster, all the way to a verified recovery. All paths follow the environment table defined at the start of Section 6. Every command is shown in sequence so you can follow along or adapt it directly.

**Full Timeline**:
```
09:00  →  configure postgresql.conf, pg_hba.conf, restart server  (Phase 1)
09:15  →  create database, table, insert first 2 rows             (Phase 2)
09:30  →  pg_basebackup taken                                    (Phase 3)
11:00  →  INSERT rows 3 and 4, force WAL archive                  (Phase 4)
11:30  →  DROP TABLE khach_hang  ← disaster                      (Phase 5)
11:31  →  begin recovery                                         (Phase 6)
```

---

##### Phase 1 — Configure PostgreSQL for WAL Archiving (09:00)

###### Step 1.1 — Create the archive directory

```bash
# Create the WAL archive destination and set ownership
mkdir -p /backup/archive
chown postgres:postgres /backup/archive
ls -ld /backup/archive
# drwxr-xr-x 2 postgres postgres 4096 Mar 15 09:00 /backup/archive
```

###### Step 1.2 — Configure `postgresql.conf`

```bash
# Edit postgresql.conf (location varies; find it with pg_config or SHOW config_file)
psql -c "SHOW config_file;"
# /etc/postgresql/18/main/postgresql.conf
```

```ini
# /etc/postgresql/18/main/postgresql.conf — add or update these settings
wal_level = replica          # minimum required for PITR (minimal is not enough)
archive_mode = on            # turn on WAL archiving
archive_command = 'cp %p /backup/archive/%f'
                             # %p = source WAL file path
                             # %f = WAL filename only
                             # test ! -f guards against overwriting existing files
```

> **Why `test ! -f` matters**: PostgreSQL may occasionally re-archive the same segment. Without this guard, a re-archive would silently overwrite the already-safe copy, which could corrupt it.

###### Step 1.3 — Configure `pg_hba.conf` for replication connections

`pg_basebackup` uses the replication protocol, so the `postgres` user must be allowed to make replication connections.

```bash
# Find pg_hba.conf
psql -c "SHOW hba_file;"
# /etc/postgresql/18/main/postgresql.conf
```

```ini
# /etc/postgresql/18/main/pg_hba.conf — ensure this line exists
# TYPE  DATABASE     USER      ADDRESS    METHOD
local   replication  postgres             trust
# or for TCP:
# host  replication  postgres  127.0.0.1/32  trust
```

###### Step 1.4 — Restart PostgreSQL to apply changes

`archive_mode` requires a **full restart** (not just `reload`) because it changes how the WAL writer initialises.

```bash
pg_ctl restart -D /var/lib/postgresql/18/main

# Confirm archive_mode is now on
psql -c "SHOW archive_mode;"
# archive_mode
# --------------
#  on

psql -c "SHOW wal_level;"
# wal_level
# -----------
#  replica
```

###### Step 1.5 — Verify archiving is working

```sql
-- Force a WAL segment switch to trigger the archive_command immediately
SELECT pg_switch_wal();
--  pg_switch_wal
-- ---------------
--  0/2000000
```

```bash
# Wait 1-2 seconds, then confirm a segment arrived in the archive
ls /backup/archive/
# 000000010000000000000001

# Check pg_stat_archiver for any failures
psql -c "SELECT last_archived_wal, last_archived_time, failed_count FROM pg_stat_archiver;"
#  last_archived_wal                 | last_archived_time            | failed_count
# -----------------------------------+-------------------------------+--------------
#  000000010000000000000001          | 2026-03-15 09:00:30+00        | 0
```

`failed_count = 0` means the archive is healthy. If it is non-zero, fix `archive_command` before proceeding — a base backup taken against a broken archive is not useful for PITR.

---

##### Phase 2 — Create the Database and Initial Data (09:15)

```sql
-- Connect as superuser
CREATE DATABASE testdb;
\c testdb

CREATE TABLE khach_hang (
    id   SERIAL PRIMARY KEY,
    name TEXT NOT NULL
);

INSERT INTO khach_hang (name) VALUES ('nguyen van a'), ('pham van b');

SELECT * FROM khach_hang;
-- id | name
-- ----+--------------
--   1 | nguyen van a
--   2 | pham van b
```

---

##### Phase 3 — Take a Base Backup (09:30)

```bash
mkdir -p /backup/base

# -Ft  : tar format (produces base.tar.gz and pg_wal.tar.gz)
# -z   : gzip compression
# -P   : show progress
# -Xs  : stream WAL during backup so the backup is self-contained
pg_basebackup -D /backup/base -Ft -z -Xs -P

# Confirm what was created
ls /backup/base/
# base.tar.gz   pg_wal.tar.gz   backup_manifest
```

`base.tar.gz` contains the entire cluster snapshot **plus a `backup_label` file** recording the exact starting LSN. This LSN is how PostgreSQL will know where to begin WAL replay during recovery.

```bash
# Preview backup_label to confirm it recorded the start LSN
tar -xOzf /backup/base/base.tar.gz backup_label
# START WAL LOCATION: 0/3000028 (file 000000010000000000000003)
# CHECKPOINT LOCATION: 0/3000060
# BACKUP METHOD: streamed
# START TIME: 2026-03-13 09:30:05 UTC
# LABEL: pg_basebackup base backup
```

---

##### Phase 4 — Normal Activity: More Inserts (11:00)

```sql
\c testdb

INSERT INTO khach_hang (name) VALUES ('nguyen van c'), ('nguyen van d');

SELECT * FROM khach_hang;
-- id | name
-- ----+--------------
--   1 | nguyen van a
--   2 | pham van b
--   3 | nguyen van c
--   4 | nguyen van d
```

```sql
-- Force the current WAL segment to be archived immediately,
-- so the INSERTs above are captured in the archive before we continue.
SELECT pg_switch_wal();
-- pg_switch_wal
-- ---------------
--  0/4000000
```

```bash
# Confirm the new WAL segment appeared in the archive
ls /backup/archive/
# 000000010000000000000003  000000010000000000000004
```

---

##### Phase 5 — The Disaster (11:30)

```sql
\c testdb

-- Someone accidentally drops the table
DROP TABLE khach_hang;

-- Confirming it is gone
\d khach_hang
-- ERROR:  relation "khach_hang" does not exist
```

---

##### Phase 6 — Recovery

###### Step 1 — Stop the server and discard the corrupted data directory

```bash
pg_ctl stop -D /var/lib/postgresql/18/main

# Remove all corrupted data files
rm -rf /var/lib/postgresql/18/main

# Re-create a clean, empty data directory with correct permissions
mkdir -p /var/lib/postgresql/18/main
chmod 700 /var/lib/postgresql/18/main
chown postgres:postgres /var/lib/postgresql/18/main
```

###### Step 2 — Restore the base backup

```bash
# Extract the base backup into the data directory
tar -xzf /backup/base/base.tar.gz -C /var/lib/postgresql/18/main

# The pg_wal.tar.gz from the backup is intentionally NOT extracted here —
# we will use the archive instead (see Step 4 note).
```

At this point the data directory is in the state it was at **09:30** — before the inserts at 11:00 and before the DROP at 11:30.

```bash
# backup_label is now in the data directory
cat /var/lib/postgresql/18/main/backup_label
# START WAL LOCATION: 0/3000028 (file 000000010000000000000003)
# ...
```

> **What `backup_label` does**: When PostgreSQL starts in recovery mode, it reads this file to find the **starting LSN** (`0/3000028`). It then requests WAL segments from the archive beginning at exactly that point — no guessing, no scanning.

###### Step 3 — Configure recovery target

```bash
# Append recovery settings to postgresql.conf
cat >> /etc/postgresql/18/main/postgresql.conf <<'EOF'

# --- PITR recovery settings ---
restore_command = 'cp /backup/archive/%f "%p"'
recovery_target_time = '2026-03-13 11:00:00'
# Stop AFTER the 11:00 inserts, BEFORE the 11:30 DROP
EOF
```

`restore_command` tells PostgreSQL where to fetch each archived WAL segment (`%f` = filename, `%p` = destination path).
`recovery_target_time` tells it to stop replaying at 11:00 — after the inserts, before the DROP.

###### Step 4 — Clear stale WAL and create the recovery signal

```bash
# Remove the stale pg_wal/ content that came with the base backup.
# These files are partial snapshots from the moment the backup was taken
# and are unreliable. PostgreSQL will fetch the authoritative copies from
# the archive via restore_command instead.
rm -rf /var/lib/postgresql/18/main/pg_wal/*

# Create recovery.signal — without this file PostgreSQL would start normally
# and skip archive recovery entirely. With it, PostgreSQL enters targeted PITR
# mode, replays WAL up to recovery_target_time, then removes the file itself.
touch /var/lib/postgresql/18/main/recovery.signal
```

> **The WAL replay chain**:
> 1. PostgreSQL reads `backup_label` → finds starting LSN `0/3000028`
> 2. Calls `restore_command` to fetch `000000010000000000000003` from `/backup/archive/`
> 3. Replays segment, then fetches the next (`000000010000000000000004`), and so on
> 4. Stops when it reaches `recovery_target_time = 11:00:00` — after the inserts, before the DROP

###### Step 5 — Start PostgreSQL and watch replay

```bash
pg_ctl start -D /var/lib/postgresql/18/main -l /var/log/postgresql/recovery.log

# Tail the log to watch WAL replay happen in real time
tail -f /var/log/postgresql/recovery.log
```

Expected log output:
```
LOG:  starting point-in-time recovery to 2026-03-13 11:00:00+00
LOG:  restored log file "000000010000000000000003" from archive
LOG:  redo starts at 0/3000028
LOG:  consistent recovery state reached at 0/3000100
LOG:  restored log file "000000010000000000000004" from archive
LOG:  recovery stopping before commit of transaction 499, time 2026-03-13 11:30:00.123456+00
LOG:  pausing at the end of recovery
HINT:  Execute pg_wal_replay_resume() to promote.
```

The "file not found" message for the next WAL segment after the target is **normal** — it means replay has reached the requested stop point.

###### Step 6 — Verify and promote

```sql
-- Connect to the recovered instance (read-only until promoted)
\c testdb

SELECT * FROM khach_hang;
-- id | name
-- ----+--------------
--   1 | nguyen van a
--   2 | pham van b
--   3 | nguyen van c  ← restored
--   4 | nguyen van d  ← restored
```

All four rows are present. The DROP that happened at 11:30 never occurred in this recovered timeline.

```sql
-- Promote the server from recovery mode to normal read-write operation.
-- This removes recovery.signal, starts a new timeline, and allows writes.
SELECT pg_wal_replay_resume();
```

```bash
# Confirm server is now accepting writes
psql -c "INSERT INTO khach_hang (name) VALUES ('test write');" testdb
# INSERT 0 1
```

---

> **Key rules to remember**:
> - `recovery_target_time` must be **after** the base backup completed (09:30) and **before** the event you want to undo (11:30 DROP).
> - Always clear stale `pg_wal/` — let `restore_command` supply segments from the archive.
> - `backup_label` is the anchor: it tells PostgreSQL where WAL replay must start.
> - After `pg_wal_replay_resume()`, PostgreSQL creates a **new timeline** (see 6.6) to protect this recovered history from future experiments.

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
archive_command = 'gzip < %p > /backup/archive/%f.gz'
```

```ini
# Recovery
restore_command = 'gunzip < /backup/archive/%f.gz > %p'
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

### 6.9 Best-Practice Mindset

Think of PITR operationally, not as a file-copy exercise:

- **Archive first, base backup second.** Confirm archiving is working and WAL files are arriving in the archive *before* taking the first base backup. A base backup taken against a broken archive is not useful for PITR.
- **Backups are a recoverability system, not just files.** The system has two assets: base backups and WAL archive continuity. Both must be healthy simultaneously.
- **Test the restore, not just the backup.** Periodically restore a base backup to a test server and replay WAL to a recent point. If you have never done this, you do not know whether your backup actually works.
- **Monitor continuously.** Archive failures are silent at first — `pg_wal/` grows quietly until it fills the filesystem and causes a PANIC shutdown. Set up alerting on archive lag.
- **Keep WAL until you have a tested newer base backup.** Do not delete old archived WAL until you have confirmed the newer base backup can be restored independently.

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
