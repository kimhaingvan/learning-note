# PostgreSQL Internals

> **Level**: Expert | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 14.1 — Storage Architecture / 14.2 — WAL / 14.3 — Buffer Manager / 14.4 — Query Processing Pipeline / 14.5 — Catalog System / 14.6 — TOAST  
> How PostgreSQL works under the hood — the data directory, page layout, WAL mechanics, buffer management, query execution, system catalogs, and oversized attribute storage.

---

## Table of Contents

1. [Storage Architecture](#1-storage-architecture)
   - [Data Directory ($PGDATA)](#data-directory-pgdata)
   - [Page Layout](#page-layout)
   - [Heap Files & Tuple Structure](#heap-files--tuple-structure)
   - [Free Space Map & Visibility Map](#free-space-map--visibility-map)
   - [Filenode Mapping](#filenode-mapping)
   - [Forks](#forks)
2. [WAL (Write-Ahead Logging)](#2-wal-write-ahead-logging)
   - [WAL Principle & Crash Recovery](#wal-principle--crash-recovery)
   - [WAL Segments & LSN](#wal-segments--lsn)
   - [Full Page Writes](#full-page-writes)
   - [Checkpoints](#checkpoints)
   - [WAL Levels](#wal-levels)
   - [WAL Internals: Record Format](#wal-internals-record-format)
3. [Buffer Manager & Shared Buffers](#3-buffer-manager--shared-buffers)
   - [Buffer Pool Architecture](#buffer-pool-architecture)
   - [Buffer Tags & Hash Table](#buffer-tags--hash-table)
   - [Clock Sweep Eviction](#clock-sweep-eviction)
   - [Pin & Lock Protocol](#pin--lock-protocol)
   - [Background Writer & Checkpointer](#background-writer--checkpointer)
   - [Ring Buffer Strategy](#ring-buffer-strategy)
4. [Query Processing Pipeline](#4-query-processing-pipeline)
   - [1. Parser](#1-parser)
   - [2. Analyzer (Semantic Analysis)](#2-analyzer-semantic-analysis)
   - [3. Rewriter](#3-rewriter)
   - [4. Planner / Optimizer](#4-planner--optimizer)
   - [5. Executor](#5-executor)
   - [Pipeline Visualization](#pipeline-visualization)
5. [Catalog System](#5-catalog-system)
   - [Key System Catalogs](#key-system-catalogs)
   - [Information Schema vs pg_catalog](#information-schema-vs-pg_catalog)
   - [Exploring the Catalog](#exploring-the-catalog)
6. [TOAST (The Oversized-Attribute Storage Technique)](#6-toast-the-oversized-attribute-storage-technique)
   - [How TOAST Works](#how-toast-works)
   - [TOAST Strategies](#toast-strategies)
   - [Inspecting TOAST](#inspecting-toast)

---

## 1. Storage Architecture

### Data Directory ($PGDATA)

Everything PostgreSQL stores lives in a single directory tree:

```
$PGDATA/
├── base/                      # Per-database directories
│   ├── 1/                     # Template database (template1)
│   ├── 16384/                 # User database (OID = 16384)
│   │   ├── 16385              # Table file (relfilenode = 16385)
│   │   ├── 16385_fsm          # Free Space Map fork
│   │   ├── 16385_vm           # Visibility Map fork
│   │   ├── 16386              # Index file
│   │   └── ...
│   └── ...
├── global/                    # Cluster-wide tables (pg_database, pg_authid, etc.)
├── pg_wal/                    # WAL segment files (16 MB each)
│   ├── 000000010000000000000001
│   ├── 000000010000000000000002
│   └── ...
├── pg_xact/                   # Transaction commit status (CLOG)
├── pg_multixact/              # Multi-transaction status (for row locks)
├── pg_subtrans/               # Subtransaction status
├── pg_tblspc/                 # Tablespace symlinks
├── pg_stat_tmp/               # Temporary stats files
├── pg_logical/                # Logical replication state
├── pg_snapshots/              # Exported snapshots
├── postgresql.conf            # Main configuration
├── pg_hba.conf                # Client authentication
├── pg_ident.conf              # Ident mapping
├── PG_VERSION                 # Major version number
├── postmaster.pid             # PID file (running server)
└── postmaster.opts            # Last startup options
```

| Directory | Contents | Managed By |
|-----------|----------|-----------|
| `base/` | All user database files. Each subdirectory is named by the database OID | PostgreSQL core |
| `global/` | Shared catalogs (`pg_database`, `pg_authid`, `pg_tablespace`) | PostgreSQL core |
| `pg_wal/` | Write-Ahead Log segments | WAL writer, archiver |
| `pg_xact/` | Commit Log (CLOG) — maps XID → committed/aborted/in-progress | Transaction manager |

```sql
-- Find the data directory
SHOW data_directory;

-- Find a database's OID
SELECT oid, datname FROM pg_database;

-- Find a table's filenode (physical file name)
SELECT pg_relation_filepath('orders');
-- 'base/16384/16512'
```

### Page Layout

**Every file** in PostgreSQL (tables, indexes, WAL) is organized into fixed **8 KB pages** (blocks).

```
┌──────────────────────── 8192 bytes (8 KB) ────────────────────────┐
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ PageHeaderData (24 bytes)                                     │  │
│  │  pd_lsn          — LSN of last WAL record affecting this page │  │
│  │  pd_checksum     — Page checksum (if data checksums enabled)  │  │
│  │  pd_lower        — Offset to end of line pointers             │  │
│  │  pd_upper        — Offset to start of free space              │  │
│  │  pd_special      — Offset to start of special space           │  │
│  │  pd_pagesize_version — Page size + layout version             │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Line Pointers (ItemIdData array)                              │  │
│  │  [lp1] → offset, length, flags (points to tuple 1)           │  │
│  │  [lp2] → offset, length, flags (points to tuple 2)           │  │
│  │  [lp3] → offset, length, flags (dead)                        │  │
│  │  ... grows downward →                                         │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │                     FREE SPACE                                │  │
│  │               (between pd_lower and pd_upper)                 │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Tuples (HeapTupleHeaderData + data)                           │  │
│  │  ← grows upward                                               │  │
│  │  [Tuple 2 data]                                               │  │
│  │  [Tuple 1 data]                                               │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Special Space (index-specific metadata, 0 bytes for heap)     │  │
│  └──────────────────────────────────────────────────────────────┘  │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

Key points:
- **Line pointers** grow downward from the header; **tuples** grow upward from the bottom.
- The free space between them is where new tuples are inserted.
- The **ctid** (tuple identifier) = `(page_number, line_pointer_index)` — e.g., `(0, 3)` means page 0, 3rd line pointer.

```sql
-- View the ctid of rows
SELECT ctid, id, name FROM orders LIMIT 5;
--  ctid  | id |  name
-- -------+----+--------
--  (0,1) |  1 | Alice
--  (0,2) |  2 | Bob
--  (0,3) |  3 | Charlie
--  (1,1) |  4 | Diana     ← page 1, first tuple
```

### Heap Files & Tuple Structure

Tables are stored as **heap files** — unordered collections of pages. Each tuple has a header:

```
HeapTupleHeaderData (23 bytes minimum):
┌──────────────────────────────────────────────────────────┐
│ t_xmin (4 bytes)      — XID of inserting transaction      │
│ t_xmax (4 bytes)      — XID of deleting/updating txn     │
│                         (0 = not deleted)                  │
│ t_cid  (4 bytes)      — Command ID within the transaction │
│ t_ctid (6 bytes)      — Current tuple ID (page, offset)  │
│                         (points to self, or to newer       │
│                          version if HOT updated)           │
│ t_infomask  (2 bytes) — Status bits (committed, aborted,  │
│                         null bitmap, etc.)                  │
│ t_infomask2 (2 bytes) — Number of attributes + HOT flag   │
│ t_hoff (1 byte)       — Offset to user data               │
├──────────────────────────────────────────────────────────┤
│ [Null bitmap — 1 bit per column, if any NULLable cols]   │
├──────────────────────────────────────────────────────────┤
│ [User data — actual column values]                        │
└──────────────────────────────────────────────────────────┘
```

**MVCC in the tuple header**: `t_xmin` and `t_xmax` are how PostgreSQL implements visibility — see [10-mvcc.md](10-mvcc.md).

| Field | Purpose |
|-------|---------|
| `t_xmin` | Transaction that created this tuple version |
| `t_xmax` | Transaction that deleted/updated this tuple (0 = still alive) |
| `t_ctid` | Points to itself, or to the next version (for HOT chains) |
| `t_infomask` | Bit flags: is committed? is aborted? has nulls? is frozen? |

**HOT (Heap-Only Tuple) Updates**: When an UPDATE doesn't change any indexed column and the new tuple fits on the same page, PostgreSQL creates a HOT chain — `t_ctid` of the old tuple points to the new one. The index still points to the old tuple's line pointer, which redirects to the new one. This avoids index updates entirely.

```sql
-- Check HOT update ratio for a table
SELECT
    relname,
    n_tup_upd,
    n_tup_hot_upd,
    round(100.0 * n_tup_hot_upd / nullif(n_tup_upd, 0), 2) AS hot_pct
FROM pg_stat_user_tables
WHERE n_tup_upd > 0
ORDER BY n_tup_upd DESC;
-- hot_pct > 90% is excellent
```

### Free Space Map & Visibility Map

Each table has two auxiliary structures stored as separate file forks:

| Structure | File Suffix | Purpose |
|-----------|:-----------:|---------|
| **Free Space Map (FSM)** | `_fsm` | Tracks how much free space each page has. Used during INSERT to find a page with enough room |
| **Visibility Map (VM)** | `_vm` | 2 bits per page: (1) all-visible — all tuples visible to all transactions, (2) all-frozen — all tuples frozen. Enables **index-only scans** and allows VACUUM to skip pages |

```sql
-- Which extension lets you inspect these?
CREATE EXTENSION pg_freespacemap;

SELECT blkno, avail AS free_bytes
FROM pg_freespace('orders')
ORDER BY blkno
LIMIT 10;

-- Visibility map info (via pageinspect extension)
CREATE EXTENSION pageinspect;

SELECT * FROM pg_visibility('orders') LIMIT 10;
-- blkno | all_visible | all_frozen | pd_all_visible
```

### Filenode Mapping

```sql
-- Table OID → physical file path
SELECT
    c.oid,
    c.relname,
    c.relfilenode,
    pg_relation_filepath(c.oid) AS filepath,
    pg_size_pretty(pg_relation_size(c.oid)) AS size,
    c.relpages,       -- number of 8KB pages
    c.reltuples        -- estimated number of rows
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'public' AND c.relkind = 'r'
ORDER BY c.relname;
```

> **Note**: After `VACUUM FULL` or `CLUSTER`, the `relfilenode` changes (the table is rewritten to a new file). The OID stays the same.

### Forks

Each relation (table, index) can have multiple **forks** — separate physical files for different purposes:

| Fork | Suffix | Purpose |
|------|:------:|---------|
| **Main** | (none) | The actual data (heap tuples or index entries) |
| **FSM** | `_fsm` | Free Space Map |
| **VM** | `_vm` | Visibility Map |
| **Init** | `_init` | Initialization fork (for unlogged tables — empty version used after crash) |

Files larger than 1 GB are split into segments:

```
base/16384/16512          # first 1 GB of table
base/16384/16512.1        # next 1 GB
base/16384/16512.2        # next 1 GB
base/16384/16512_fsm      # FSM fork
base/16384/16512_vm       # VM fork
```

---

## 2. WAL (Write-Ahead Logging)

### WAL Principle & Crash Recovery

The fundamental guarantee: **no data page is ever modified on disk until the corresponding WAL record has been flushed to disk.**

```
Transaction: INSERT INTO orders VALUES (1, 'Alice', 100);

Step 1: Write WAL record to WAL buffer
Step 2: On COMMIT → flush WAL buffer to disk (pg_wal/)      ← durability point
Step 3: Modified data page stays in shared buffers (dirty)
Step 4: Eventually, checkpointer writes dirty page to disk

CRASH after step 2, before step 4?
→ On restart, PostgreSQL replays WAL from last checkpoint
→ Reconstructs the dirty pages in shared buffers
→ Data is recovered ✅
```

```
Timeline of a crash recovery:

     Checkpoint         WAL records         CRASH
        │                   │                 │
────────┼───────────────────┼─────────────────┼────►
        │                   │                 │
        │◄── redo starts ──►│◄── replay ─────►│
        │                                     │
        │    Recovery replays all WAL          │
        │    records since the last            │
        │    completed checkpoint              │
```

### WAL Segments & LSN

WAL is a continuous stream divided into **segments** (16 MB files by default):

```
pg_wal/
├── 000000010000000000000001    # Timeline 1, segment 1
├── 000000010000000000000002    # Timeline 1, segment 2
├── 000000010000000000000003    # Timeline 1, segment 3
└── ...

Segment file name: TTTTTTTTSSSSSSSSSSSSSSSS
  T = Timeline ID (8 hex digits)
  S = Segment number (16 hex digits)
```

**LSN (Log Sequence Number)** is the byte offset into the WAL stream:

```sql
-- Current WAL position
SELECT pg_current_wal_lsn();          -- e.g., '0/16B3748'
SELECT pg_current_wal_insert_lsn();   -- insert position (may be ahead of flush)
SELECT pg_current_wal_flush_lsn();    -- flushed to disk

-- Convert LSN to WAL file name
SELECT pg_walfile_name('0/16B3748');
-- '000000010000000000000001'

-- Difference between two LSNs (in bytes)
SELECT pg_wal_lsn_diff('0/16B3748', '0/1000000');
-- 7127880 bytes
```

### Full Page Writes

After each checkpoint, the **first modification** to any page writes the **entire 8 KB page image** to WAL (not just the change). This prevents torn page corruption.

```
                    Checkpoint
                        │
Page first modified ────┼──── FPW (full 8 KB image in WAL record)
Page modified again ────┼──── Only the change (WAL record much smaller)
Page modified again ────┼──── Only the change
                        │
                    Checkpoint
                        │
Page first modified ────┼──── FPW again (new checkpoint cycle)
```

| Setting | Effect |
|---------|--------|
| `full_page_writes = on` (default) | Prevents torn pages. **Never turn off** in production |
| `wal_compression = lz4` | Compress FPW images. Reduces WAL size significantly |

> **Why FPW matters**: OS writes 8 KB pages, but the disk might write 512-byte sectors. A power failure mid-write could result in a half-old, half-new page (torn page). FPW lets recovery restore the complete page from WAL.

### Checkpoints

A checkpoint is a **consistency point** — all dirty buffers are flushed to disk, and a checkpoint record is written to WAL.

```
┌──────────────────────────────────────────────────────────────┐
│                    Checkpoint Process                          │
│                                                               │
│  1. Write checkpoint START record to WAL                      │
│  2. Flush all dirty shared buffers to data files              │
│     (spread over checkpoint_completion_target × interval)     │
│  3. Write checkpoint DONE record to WAL                       │
│  4. Truncate/recycle old WAL segments before this checkpoint  │
│  5. Update pg_control with new checkpoint location            │
└──────────────────────────────────────────────────────────────┘
```

**Checkpoint triggers**:

| Trigger | Condition |
|---------|-----------|
| **Timed** | Every `checkpoint_timeout` (default: 5 min) |
| **WAL volume** | When WAL generated since last checkpoint exceeds `max_wal_size` |
| **Manual** | `CHECKPOINT` command |
| **Shutdown** | During clean server shutdown |

```sql
-- Force a checkpoint
CHECKPOINT;

-- Monitor checkpoint activity
SELECT
    checkpoints_timed,          -- triggered by timeout
    checkpoints_req,            -- triggered by WAL volume (or manual)
    checkpoint_write_time,      -- ms spent writing dirty buffers
    checkpoint_sync_time,       -- ms spent syncing to disk
    buffers_checkpoint           -- buffers written during checkpoints
FROM pg_stat_bgwriter;
```

### WAL Levels

| Level | What's Recorded | Use Case |
|-------|----------------|---------|
| `minimal` | Only enough for crash recovery | Standalone servers with no replication or archiving |
| `replica` (default) | Enough for streaming replication + PITR | ✅ Standard production setting |
| `logical` | Adds logical decoding information | Required for logical replication, CDC |

```sql
SHOW wal_level;  -- 'replica' by default

-- Changing wal_level requires restart
ALTER SYSTEM SET wal_level = 'logical';
-- Then: pg_ctl restart
```

### WAL Internals: Record Format

Each WAL record contains:

```
WAL Record:
┌────────────────────────────────────────────┐
│ xl_tot_len    — Total record length         │
│ xl_xid        — Transaction ID              │
│ xl_prev       — LSN of previous record      │
│ xl_info       — Resource manager specific    │
│ xl_rmid       — Resource manager ID          │
│               (HEAP, BTREE, XACT, etc.)     │
│ xl_crc        — CRC checksum                │
├────────────────────────────────────────────┤
│ Block reference data:                       │
│   relfilenode, fork, block number           │
│   [Full page image if FPW]                  │
├────────────────────────────────────────────┤
│ Record-specific data:                       │
│   e.g., tuple data for INSERT              │
│   e.g., old/new values for UPDATE          │
└────────────────────────────────────────────┘
```

```bash
# Inspect WAL records with pg_waldump
pg_waldump -p pg_wal -s 0/16B3000 -e 0/16B4000

# Output shows:
# rmgr: Heap    len: 59  tx: 739  lsn: 0/16B3748  desc: INSERT off 5
# rmgr: Transaction len: 34  tx: 739  lsn: 0/16B3790  desc: COMMIT 2024-06-01
```

---

## 3. Buffer Manager & Shared Buffers

> For tuning `shared_buffers` and memory configuration, see [25-performance-tuning.md](25-performance-tuning.md). This section covers **how it works internally**.

### Buffer Pool Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Shared Buffer Pool                     │
│                                                          │
│   Buffer[0]  Buffer[1]  Buffer[2]  ...  Buffer[N-1]     │
│   ┌───────┐  ┌───────┐  ┌───────┐      ┌───────┐       │
│   │ 8 KB  │  │ 8 KB  │  │ 8 KB  │      │ 8 KB  │       │
│   │ page  │  │ page  │  │ page  │      │ page  │       │
│   │       │  │       │  │       │      │       │       │
│   │tag:   │  │tag:   │  │tag:   │      │tag:   │       │
│   │(db,rel│  │(db,rel│  │(db,rel│      │(db,rel│       │
│   │,fork, │  │,fork, │  │,fork, │      │,fork, │       │
│   │blkno) │  │blkno) │  │blkno) │      │blkno) │       │
│   │       │  │       │  │       │      │       │       │
│   │refcnt │  │refcnt │  │refcnt │      │refcnt │       │
│   │usage  │  │usage  │  │usage  │      │usage  │       │
│   │dirty? │  │dirty? │  │dirty? │      │dirty? │       │
│   └───────┘  └───────┘  └───────┘      └───────┘       │
│                                                          │
│   N = shared_buffers / 8KB                               │
│   (e.g., 8 GB → 1,048,576 buffers)                      │
└─────────────────────────────────────────────────────────┘
```

### Buffer Tags & Hash Table

Every buffer is identified by a **buffer tag**:

```
Buffer Tag = (RelFileNode, ForkNumber, BlockNumber)

RelFileNode = (tablespace OID, database OID, relfilenode)
ForkNumber  = MAIN_FORKNUM (0), FSM_FORKNUM (1), VM_FORKNUM (2), INIT_FORKNUM (3)
BlockNumber = 0-based page number within the fork
```

PostgreSQL uses a **hash table** to map buffer tags to buffer IDs:

```
Read request for page (db=16384, rel=16512, fork=0, block=5):

1. Compute hash of buffer tag
2. Look up hash table → found in Buffer[42]? → buffer HIT ✅
3. Not found → buffer MISS:
   a. Find a victim buffer (clock sweep)
   b. If victim is dirty → write to disk first
   c. Read page from disk into victim buffer
   d. Update hash table: tag → Buffer[victim_id]
   e. Return buffer
```

### Clock Sweep Eviction

When the buffer pool is full and a new page is needed, PostgreSQL uses **clock sweep** to find a victim:

```
Clock hand rotates through all buffers:

Buffer[0] usage=3  →  decrement → usage=2  →  skip
Buffer[1] usage=0  →  VICTIM FOUND ✅ (evict this buffer)
Buffer[2] usage=1  →  decrement → usage=0  →  skip (next time it'll be evicted)
Buffer[3] usage=5  →  decrement → usage=4  →  skip
...

Rules:
- Each time a buffer is accessed: usage_count++ (max = 5)
- Clock hand sweeps: if usage_count = 0 → evict; else → decrement and move on
- Pinned buffers (refcount > 0) are always skipped
```

This approximates LRU behavior:
- Frequently accessed buffers maintain high usage counts → survive multiple sweeps
- Rarely accessed buffers quickly drop to 0 → evicted soon

### Pin & Lock Protocol

| Mechanism | Purpose | Held By |
|-----------|---------|--------|
| **Pin (refcount)** | Prevents eviction — "I'm using this buffer" | Backend for duration of access |
| **Content lock (shared/exclusive)** | Controls read/write access to the page contents | Shared for reads, exclusive for writes |
| **IO lock** | Prevents concurrent I/O on the same buffer | Backend performing read/write to disk |

```
Read a tuple:
  1. Pin the buffer (refcount++)
  2. Acquire shared content lock
  3. Read the tuple
  4. Release content lock
  5. Unpin (refcount--)

Modify a tuple:
  1. Pin the buffer (refcount++)
  2. Acquire exclusive content lock
  3. Write WAL record
  4. Modify the page in buffer
  5. Mark buffer dirty
  6. Release content lock
  7. Unpin (refcount--)
```

### Background Writer & Checkpointer

Two background processes keep the buffer pool healthy:

| Process | Job | Controlled By |
|---------|-----|:------------:|
| **Background Writer (bgwriter)** | Continuously writes dirty buffers to disk in small batches — keeping free buffers available for backends | `bgwriter_delay`, `bgwriter_lru_maxpages`, `bgwriter_lru_multiplier` |
| **Checkpointer** | Writes ALL dirty buffers at checkpoint time — creates a consistency point for recovery | `checkpoint_timeout`, `max_wal_size`, `checkpoint_completion_target` |

```
Without bgwriter:
  Backend needs buffer → all clean buffers used → must evict dirty buffer →
  WRITE dirty page to disk → READ new page → SLOW (synchronous I/O in hot path)

With bgwriter:
  bgwriter quietly writes dirty buffers in background →
  Clean buffers always available → backend reads are fast
```

### Ring Buffer Strategy

For large sequential scans, PostgreSQL uses a **ring buffer** — a small fixed-size buffer (256 KB = 32 pages) instead of cycling through the entire shared buffer pool. This prevents a single `SELECT * FROM huge_table` from evicting all useful cached pages.

Ring buffers are used for:
- Large sequential scans (> 25% of `shared_buffers`)
- Bulk writes (COPY, CREATE TABLE AS)
- VACUUM

---

## 4. Query Processing Pipeline

### Pipeline Visualization

```
SQL text
    │
    ▼
┌──────────┐     Parse tree      ┌──────────┐     Query tree
│  Parser  │ ──────────────────► │ Analyzer │ ──────────────────►
└──────────┘                     └──────────┘
                                                     │
                                                     ▼
┌──────────┐     Plan tree       ┌──────────┐     Query tree
│ Executor │ ◄────────────────── │ Planner  │ ◄────────────────── 
└──────────┘                     └──────────┘     ┌──────────┐
    │                                              │ Rewriter │
    ▼                                              └──────────┘
 Result set
```

### 1. Parser

Converts SQL text into a **parse tree** (abstract syntax tree). Two sub-stages:

```sql
SELECT name, total FROM orders WHERE status = 'active' ORDER BY total DESC;
```

**Lexer** → Tokens: `SELECT`, `name`, `,`, `total`, `FROM`, `orders`, `WHERE`, `status`, `=`, `'active'`, `ORDER`, `BY`, `total`, `DESC`

**Grammar parser** → Parse tree:

```
SelectStmt
├── targetList: [name, total]
├── fromClause: [orders]
├── whereClause: OpExpr(=, status, 'active')
└── sortClause: [total DESC]
```

At this stage:
- Only syntax is checked — `SELECTT * FROM orders` fails here
- Table and column names are **not** verified yet
- `SELECT * FROM nonexistent_table` passes the parser

### 2. Analyzer (Semantic Analysis)

Resolves names and types against the **system catalog**:

- Looks up `orders` in `pg_class` → finds the table OID
- Looks up `name`, `total`, `status` in `pg_attribute` → finds column types
- Resolves operator `=` for `text = text` in `pg_operator`
- Checks user has `SELECT` privilege on the table

Produces a **query tree** with OIDs, type information, and resolved references.

Errors at this stage: `"column does not exist"`, `"relation does not exist"`, `"permission denied"`

### 3. Rewriter

Applies **rules** (the rule system):

```sql
-- If you query a view:
SELECT * FROM active_orders;

-- The rewriter expands it using the view's stored rule:
SELECT * FROM orders WHERE status = 'active';
```

Rules are also used for:
- Updatable views (INSERT/UPDATE/DELETE rules)
- `INSTEAD OF` transformations
- `CREATE RULE` custom rules

### 4. Planner / Optimizer

The most complex stage. Generates multiple possible **execution plans** and chooses the cheapest one.

#### Cost Model

The planner assigns a **cost** to each plan based on estimated I/O and CPU:

```
Cost = (pages_read × page_cost) + (tuples_processed × cpu_tuple_cost) + ...

page_cost:
  seq_page_cost    = 1.0    (sequential read)
  random_page_cost = 4.0    (random read — lower for SSDs)

cpu_tuple_cost   = 0.01    (processing one tuple)
cpu_index_tuple_cost = 0.005 (processing one index entry)
cpu_operator_cost = 0.0025  (evaluating one operator/function)
```

#### Plan Selection Example

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 42;

-- The planner considers:
-- Plan A: Sequential Scan → scan all pages, filter each tuple
--   cost = (pages × seq_page_cost) + (rows × cpu_tuple_cost)
--   = 1000 × 1.0 + 100000 × 0.01 = 2000
--
-- Plan B: Index Scan on idx_customer_id → look up in index, fetch heap pages
--   cost = (index_pages × random_page_cost) + (matching_rows × cpu_cost)
--   = 3 × 4.0 + 50 × 0.01 = 12.5
--
-- Plan B wins → Index Scan chosen ✅
```

#### Statistics Used

The planner relies on statistics from `ANALYZE` (stored in `pg_statistic` / `pg_stats`):

| Statistic | What | Used For |
|-----------|------|---------|
| `n_distinct` | Estimated distinct values | Estimating selectivity of `=` predicates |
| `most_common_vals` + `most_common_freqs` | Top N values and their frequencies | Skewed distribution estimation |
| `histogram_bounds` | Equal-height histogram buckets | Range query selectivity (`<`, `>`, `BETWEEN`) |
| `correlation` | Physical vs logical order correlation | Deciding between index scan and bitmap scan |
| `null_frac` | Fraction of NULL values | `IS NULL` / `IS NOT NULL` selectivity |

```sql
-- View statistics for a column
SELECT
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

#### Join Optimization

For queries with multiple tables, the planner must choose:
1. **Join order** — Which tables to join first (exhaustive search up to ~12 tables, then GEQO genetic algorithm)
2. **Join method** — Nested Loop, Hash Join, or Merge Join for each pair
3. **Access method** — Seq Scan, Index Scan, Bitmap Scan for each table

### 5. Executor

Runs the chosen plan using a **pull-based iterator model** (Volcano model):

```
Each plan node implements:
  Init()  — Set up state
  Next()  — Return the next tuple (pull from child nodes)
  End()   — Clean up

         Limit (Next → pulls from Sort)
           │
         Sort (Next → pulls all from SeqScan, sorts, returns one by one)
           │
         SeqScan on orders (Next → reads next tuple from heap)

Each call to the top node's Next() cascades down the tree.
```

Plan node types:

| Category | Nodes |
|----------|-------|
| **Scan** | Seq Scan, Index Scan, Index Only Scan, Bitmap Heap Scan, TID Scan, Function Scan, CTE Scan |
| **Join** | Nested Loop, Hash Join, Merge Join |
| **Aggregate** | Aggregate, Group Aggregate, Hash Aggregate |
| **Sort** | Sort, Incremental Sort |
| **Modify** | Insert, Update, Delete |
| **Other** | Limit, Unique, Append, Materialize, Subquery Scan, Gather (parallel) |

---

## 5. Catalog System

The system catalog is PostgreSQL's **metadata database** — it stores everything about databases, tables, columns, types, functions, operators, indexes, and more.

> The catalog is **self-describing**: catalog tables are themselves described in the catalog. `pg_class` has a row for `pg_class` itself.

### Key System Catalogs

| Catalog | Stores | Key Columns |
|---------|--------|-------------|
| `pg_class` | Relations (tables, indexes, views, sequences, matviews) | `oid`, `relname`, `relkind`, `relfilenode`, `relpages`, `reltuples` |
| `pg_attribute` | Columns of all relations | `attrelid`, `attname`, `atttypid`, `attnum`, `attnotnull` |
| `pg_type` | Data types | `oid`, `typname`, `typlen`, `typtype` |
| `pg_index` | Index metadata | `indexrelid`, `indrelid`, `indkey`, `indisunique`, `indisprimary` |
| `pg_namespace` | Schemas | `oid`, `nspname` |
| `pg_proc` | Functions and procedures | `oid`, `proname`, `proargtypes`, `prorettype`, `prosrc` |
| `pg_operator` | Operators | `oid`, `oprname`, `oprleft`, `oprright`, `oprresult` |
| `pg_constraint` | Constraints (PK, FK, CHECK, UNIQUE, EXCLUDE) | `conname`, `contype`, `conrelid`, `confrelid` |
| `pg_trigger` | Triggers | `tgname`, `tgrelid`, `tgfoid`, `tgtype` |
| `pg_database` | Databases (in `global/`) | `oid`, `datname`, `datdba`, `encoding` |
| `pg_authid` | Roles (in `global/`) | `oid`, `rolname`, `rolsuper`, `rolcanlogin` |
| `pg_tablespace` | Tablespaces (in `global/`) | `oid`, `spcname`, `spcowner` |
| `pg_statistic` | Column statistics (from ANALYZE) | `starelid`, `staattnum`, `stadistinct` |
| `pg_depend` | Object dependencies | `classid`, `objid`, `refclassid`, `refobjid`, `deptype` |

### Information Schema vs pg_catalog

| Feature | `pg_catalog` | `information_schema` |
|---------|:------------:|:--------------------:|
| Standard | PostgreSQL-specific | SQL standard |
| Completeness | Everything | Subset |
| OIDs | ✅ Available | ❌ Not exposed |
| Speed | Fast (direct) | Slower (views over pg_catalog) |
| Portability | PostgreSQL only | Cross-database |
| Use for | Admin, extensions, internal queries | Application-level schema introspection |

### Exploring the Catalog

```sql
-- List all tables in the current database
SELECT relname, relkind, relpages, reltuples
FROM pg_class
WHERE relnamespace = 'public'::regnamespace
    AND relkind = 'r'   -- 'r' = ordinary table
ORDER BY relname;

-- relkind values:
-- 'r' = table, 'i' = index, 'v' = view, 'm' = materialized view,
-- 'S' = sequence, 'f' = foreign table, 'p' = partitioned table

-- List columns of a table
SELECT attname, format_type(atttypid, atttypmod) AS data_type,
       attnotnull, atthasdef
FROM pg_attribute
WHERE attrelid = 'orders'::regclass
    AND attnum > 0          -- exclude system columns
    AND NOT attisdropped     -- exclude dropped columns
ORDER BY attnum;

-- List indexes on a table
SELECT
    i.relname AS index_name,
    ix.indisunique,
    ix.indisprimary,
    pg_get_indexdef(i.oid) AS definition
FROM pg_index ix
JOIN pg_class i ON i.oid = ix.indexrelid
WHERE ix.indrelid = 'orders'::regclass;

-- List constraints
SELECT conname, contype, pg_get_constraintdef(oid) AS definition
FROM pg_constraint
WHERE conrelid = 'orders'::regclass;
-- contype: 'p' = PK, 'f' = FK, 'c' = CHECK, 'u' = UNIQUE, 'x' = EXCLUDE

-- Object dependencies (what depends on a table?)
SELECT
    classid::regclass AS dependent_catalog,
    objid,
    deptype,
    refobjid::regclass AS referenced_object
FROM pg_depend
WHERE refobjid = 'orders'::regclass
LIMIT 20;

-- List all functions in a schema
SELECT proname, pg_get_function_arguments(oid) AS args,
       pg_get_function_result(oid) AS returns
FROM pg_proc
WHERE pronamespace = 'public'::regnamespace
ORDER BY proname;
```

**Shortcut: `\d` in psql uses these catalogs under the hood:**

```
\dt        → queries pg_class WHERE relkind = 'r'
\di        → queries pg_class WHERE relkind = 'i'
\d orders  → queries pg_attribute, pg_index, pg_constraint for that table
\df        → queries pg_proc
```

---

## 6. TOAST (The Oversized-Attribute Storage Technique)

### How TOAST Works

PostgreSQL's page size is 8 KB. A single tuple cannot span multiple pages. When a row's data exceeds the **TOAST threshold (~2 KB)**, PostgreSQL automatically:

1. **Compresses** the oversized attribute (pglz or lz4)
2. If still too large, **moves** it to a separate **TOAST table**
3. Stores a **TOAST pointer** (18 bytes) in the original row

```
Main table row:
┌─────────┬─────────┬──────────────────────────┐
│ id = 1  │ name    │ large_text = TOAST PTR   │──────►  TOAST table
│         │ "Kim"   │ (18 bytes: chunk_id,     │         ┌──────────────────┐
│         │         │  chunk_seq, toast_relid)  │         │ chunk_id | seq | data
│         │         │                          │         │ 12345    | 0   | [2000 bytes]
│         │         │                          │         │ 12345    | 1   | [2000 bytes]
│         │         │                          │         │ 12345    | 2   | [1500 bytes]
└─────────┴─────────┴──────────────────────────┘         └──────────────────┘
```

Each TOAST table has its own index on `(chunk_id, chunk_seq)` for fast retrieval.

### TOAST Strategies

Each column has a TOAST strategy that controls how oversized values are handled:

| Strategy | Compression? | Out-of-line? | When Applied | Typical Types |
|----------|:-----------:|:------------:|-------------|--------------|
| **PLAIN** | ❌ | ❌ | Never TOASTed | `integer`, `boolean`, `date` (always fit in-line) |
| **EXTENDED** | ✅ First | ✅ Then if still too large | Default for variable-length types | `text`, `bytea`, `jsonb` |
| **EXTERNAL** | ❌ | ✅ Directly | Move out-of-line without compressing | Useful for pre-compressed data (images, already-compressed files) |
| **MAIN** | ✅ | ⚠️ Only as last resort | Try hard to keep in-line, compress aggressively | `numeric` (compression very effective) |

```sql
-- Check current TOAST strategy for columns
SELECT
    attname,
    attstorage,
    CASE attstorage
        WHEN 'p' THEN 'PLAIN'
        WHEN 'e' THEN 'EXTERNAL'
        WHEN 'x' THEN 'EXTENDED'
        WHEN 'm' THEN 'MAIN'
    END AS strategy
FROM pg_attribute
WHERE attrelid = 'articles'::regclass
    AND attnum > 0
    AND NOT attisdropped;

-- Change TOAST strategy for a column
ALTER TABLE articles ALTER COLUMN body SET STORAGE EXTERNAL;
-- Use when body stores pre-compressed data
```

**TOAST compression algorithms** (PG 14+):

```sql
-- Default compression (pglz — PostgreSQL's built-in)
-- Or use LZ4 (faster, slightly less compression):
ALTER TABLE articles ALTER COLUMN body SET COMPRESSION lz4;

-- Global default:
SET default_toast_compression = 'lz4';
```

### Inspecting TOAST

```sql
-- Find the TOAST table for a relation
SELECT
    c.relname AS table_name,
    c.reltoastrelid::regclass AS toast_table,
    pg_size_pretty(pg_relation_size(c.reltoastrelid)) AS toast_size,
    pg_size_pretty(pg_relation_size(c.oid)) AS main_size
FROM pg_class c
WHERE c.relname = 'articles' AND c.relkind = 'r';

-- Query the TOAST table directly (system table, needs superuser)
SELECT chunk_id, chunk_seq, length(chunk_data) AS chunk_bytes
FROM pg_toast.pg_toast_16512    -- replace with actual TOAST table name
LIMIT 10;

-- Total TOAST overhead for all tables
SELECT
    c.relname,
    pg_size_pretty(pg_relation_size(c.oid)) AS main_size,
    pg_size_pretty(pg_relation_size(c.reltoastrelid)) AS toast_size,
    pg_size_pretty(pg_total_relation_size(c.oid)) AS total_size
FROM pg_class c
WHERE c.relkind = 'r' AND c.reltoastrelid != 0
ORDER BY pg_relation_size(c.reltoastrelid) DESC
LIMIT 20;
```

**TOAST performance implications**:

| Scenario | Behavior | Impact |
|----------|----------|--------|
| `SELECT *` | Fetches and decompresses ALL TOASTed columns | ⚠️ Slow if large columns exist |
| `SELECT id, name` | Only fetches referenced columns — TOAST data for `body` is **never read** | ✅ Fast |
| `WHERE body LIKE ...` | Must fetch and decompress `body` for each row | ⚠️ Slow — use full-text search instead |

> **Key insight**: TOAST is **column-smart** — it only fetches TOASTed data for columns actually referenced in the query. This is why `SELECT *` on tables with large text/JSON columns is significantly slower than selecting only the small columns you need.

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `SELECT *` on tables with TOASTed columns | Fetches and decompresses large columns unnecessarily | Select only needed columns |
| Ignoring buffer hit ratio | Not noticing when working set exceeds `shared_buffers` | Monitor `pg_stat_database` hit ratio; aim for > 99% |
| Not running `ANALYZE` after bulk loads | Planner uses stale statistics → bad plans | `ANALYZE` after significant data changes |
| Manual `CHECKPOINT` in production | Causes sudden I/O spike, stalls writes | Let timed/WAL-triggered checkpoints handle it |
| `full_page_writes = off` | Torn pages after crash = data corruption | **Never** disable in production |
| Ignoring `checkpoints_req` growth | Frequent forced checkpoints → excessive I/O | Increase `max_wal_size` |
| Not understanding line pointer indirection | Confusion when ctid doesn't match expectations (HOT chains) | Study HOT and line pointer redirect mechanisms |

---

## What to Learn Next

1. **MVCC** — How `t_xmin`/`t_xmax` in tuple headers enable concurrent access — see [10-mvcc.md](10-mvcc.md).
2. **Performance Tuning** — Tuning the parameters described here — see [25-performance-tuning.md](25-performance-tuning.md).
3. **Indexes** — EXPLAIN plans and how the planner chooses access methods — see [05-indexes.md](05-indexes.md).
4. **WAL in Practice** — Backup, archiving, and PITR using WAL — see [06-backup-restore.md](06-backup-restore.md).
5. **Monitoring** — Observing buffer, WAL, and checkpoint metrics — see [26-monitoring-diagnostics-bloat.md](26-monitoring-diagnostics-bloat.md).

---

> *Ref: [Docs — Database File Layout](https://www.postgresql.org/docs/18/storage-file-layout.html) · [Docs — Database Page Layout](https://www.postgresql.org/docs/18/storage-page-layout.html) · [Docs — WAL](https://www.postgresql.org/docs/18/wal.html) · [Docs — Query Processing](https://www.postgresql.org/docs/18/overview.html) · [Docs — System Catalogs](https://www.postgresql.org/docs/18/catalogs.html) · [Docs — TOAST](https://www.postgresql.org/docs/18/storage-toast.html) · [The Internals of PostgreSQL](https://www.interdb.jp/pg/)*
