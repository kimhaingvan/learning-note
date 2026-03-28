# Replication & High Availability

> **Level**: Advanced | **Tone**: Technical / Teacher-like  
> **Roadmap**: Phase 10.1 — Streaming Replication / Phase 10.2 — Logical Replication / Phase 10.3 — Failover & Switchover  
> How PostgreSQL replicates data across servers — streaming (physical) replication, logical replication, failover, switchover, and high availability patterns.

---

## Table of Contents

1. [Replication Overview](#1-replication-overview)
2. [Streaming Replication (Physical)](#2-streaming-replication-physical)
   - [How It Works](#how-it-works)
   - [Setting Up a Standby](#setting-up-a-standby)
   - [Synchronous vs Asynchronous](#synchronous-vs-asynchronous)
   - [Hot Standby (Read Replicas)](#hot-standby-read-replicas)
   - [Replication Slots](#replication-slots)
   - [Monitoring Streaming Replication](#monitoring-streaming-replication)
   - [Cascading Replication](#cascading-replication)
3. [Logical Replication](#3-logical-replication)
   - [How It Works](#how-it-works-1)
   - [Setting Up Publication & Subscription](#setting-up-publication--subscription)
   - [Conflict Handling](#conflict-handling)
   - [Use Cases](#use-cases)
   - [Limitations](#limitations)
4. [Streaming vs Logical Replication](#4-streaming-vs-logical-replication)
5. [Failover & Switchover](#5-failover--switchover)
   - [Manual Failover](#manual-failover)
   - [Automatic Failover — Patroni](#automatic-failover--patroni)
   - [Switchover (Planned)](#switchover-planned)
   - [Split Brain Prevention](#split-brain-prevention)
6. [High Availability Architectures](#6-high-availability-architectures)

---

## 1. Replication Overview

```
                    ┌──────────────┐
                    │   Primary    │
                    │ (read-write) │
                    └──────┬───────┘
                           │ WAL stream
              ┌────────────┼────────────┐
              ▼            ▼            ▼
     ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
     │  Standby 1   │ │  Standby 2   │ │  Standby 3   │
     │ (read-only)  │ │ (read-only)  │ │ (read-only)  │
     │  sync        │ │  async       │ │  async        │
     └──────────────┘ └──────────────┘ └──────────────┘
```

| Type | What It Replicates | Granularity | Cross-Version? |
|------|-------------------|-------------|:--------------:|
| **Streaming (Physical)** | Entire WAL byte stream | Whole cluster | ❌ Must match major version |
| **Logical** | Row-level changes via decode | Per-table (publication) | ✅ Yes |

---

## 2. Streaming Replication (Physical)

### How It Works

The primary sends WAL records to standbys in real-time over a TCP connection. Standbys replay them to stay in sync.

```
Primary                          Standby
   │                                │
   │── WAL records (streaming) ───► │
   │                                │ apply/replay WAL
   │                                │
   │◄── LSN acknowledgment ──────── │  (sync mode only)
```

Key components:

| Component | Role |
|-----------|------|
| **WAL sender** (`walsender`) | Primary-side process that streams WAL to a standby |
| **WAL receiver** (`walreceiver`) | Standby-side process that receives and writes WAL |
| **`standby.signal`** | Signal file that tells PostgreSQL to start in standby mode |
| **`primary_conninfo`** | Connection string the standby uses to connect to the primary |

### Setting Up a Standby

#### 1. Primary Configuration

```ini
# postgresql.conf on the primary
wal_level = replica                  # required (default in PG 10+)
max_wal_senders = 10                 # max concurrent streaming connections
wal_keep_size = '1GB'                # retain WAL for slow standbys (or use replication slots)
```

```ini
# pg_hba.conf on the primary — allow replication connections
host    replication    repl_user    10.0.0.0/8    scram-sha-256
```

```sql
-- Create a replication user
CREATE ROLE repl_user LOGIN REPLICATION PASSWORD 'strong_password';
```

#### 2. Base Backup on the Standby

```bash
# Take a base backup from the primary directly into the standby's data dir
pg_basebackup -h primary_host -U repl_user -D /var/lib/postgresql/18/main \
    -Fp -Xs -P -R
#  -Fp : plain format (files, not tar)
#  -Xs : stream WAL during backup
#  -P  : show progress
#  -R  : auto-create standby.signal + write primary_conninfo to postgresql.auto.conf
```

The `-R` flag is critical — it automatically creates:
- `standby.signal` — tells PostgreSQL to start as a standby
- `postgresql.auto.conf` with `primary_conninfo` — connection to the primary

#### 3. Start the Standby

```bash
pg_ctl start -D /var/lib/postgresql/18/main
```

The standby connects to the primary and begins streaming WAL.

#### 4. Verify

```sql
-- On the primary: check connected standbys
SELECT pid, usename, application_name, client_addr,
       state, sync_state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;

-- On the standby: confirm it's in recovery
SELECT pg_is_in_recovery();  -- true
```

### Synchronous vs Asynchronous

| Mode | Primary Behavior | Data Safety | Performance |
|------|-----------------|:-----------:|:-----------:|
| **Asynchronous** (default) | Commits immediately, doesn't wait for standby | Risk of data loss on failover | ✅ Best |
| **Synchronous** | Waits for standby to confirm WAL receipt/flush/apply | ✅ Zero data loss | Slower (network latency) |

#### Configuring Synchronous Replication

```ini
# postgresql.conf on the primary
synchronous_standby_names = 'standby1'   # match application_name on the standby

# Sync levels (choose one):
synchronous_commit = 'on'               # wait for standby to flush WAL to disk (default when sync enabled)
synchronous_commit = 'remote_write'     # wait for standby to write to OS cache (faster, less safe)
synchronous_commit = 'remote_apply'     # wait for standby to replay (strongest — queries on standby see the data)
```

```ini
# On the standby — set the application name
primary_conninfo = 'host=primary_host user=repl_user password=... application_name=standby1'
```

#### Multiple Sync Standbys

```ini
# ANY 1: at least 1 of the listed standbys must confirm
synchronous_standby_names = 'ANY 1 (standby1, standby2, standby3)'

# FIRST 2: the first 2 in list order must confirm
synchronous_standby_names = 'FIRST 2 (standby1, standby2, standby3)'
```

### Hot Standby (Read Replicas)

A standby that accepts **read-only queries** while replaying WAL.

```ini
# postgresql.conf on the standby
hot_standby = on    # default in PG 10+
```

```sql
-- On the standby: read queries work
SELECT * FROM orders WHERE status = 'pending';   -- ✅ works

-- Write queries fail
INSERT INTO orders ...;   -- ERROR: cannot execute INSERT in a read-only transaction
```

#### Replication Conflict

Queries on the hot standby can conflict with WAL replay:

| Conflict | Cause | Resolution |
|----------|-------|-----------|
| Lock conflict | Standby query holds a lock that replay needs | `max_standby_streaming_delay` — how long to wait before cancelling the query |
| Snapshot conflict | Standby query needs rows that replay is about to clean up (VACUUM) | `hot_standby_feedback = on` — standby sends feedback to delay cleanup on primary |

```ini
# postgresql.conf on the standby
max_standby_streaming_delay = 30s      # wait up to 30s before cancelling conflicting queries
hot_standby_feedback = on               # tell primary which rows standby queries still need
```

### Replication Slots

Without slots, the primary may recycle WAL segments before the standby has received them. Replication slots **guarantee WAL retention**.

```sql
-- Create a physical replication slot on the primary
SELECT pg_create_physical_replication_slot('standby1_slot');

-- View slots
SELECT slot_name, slot_type, active, restart_lsn
FROM pg_replication_slots;
```

```ini
# On the standby — use the slot
primary_slot_name = 'standby1_slot'
```

> **Warning**: A replication slot for a dead standby retains WAL **indefinitely**, eventually filling the disk. Monitor slots and drop unused ones:
> ```sql
> SELECT pg_drop_replication_slot('standby1_slot');
> ```

### Monitoring Streaming Replication

```sql
-- On the primary: replication lag
SELECT
    client_addr,
    application_name,
    state,
    sync_state,
    pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_lag_bytes,
    replay_lag
FROM pg_stat_replication;

-- On the standby: how far behind
SELECT
    now() - pg_last_xact_replay_timestamp() AS replay_lag_time,
    pg_last_wal_receive_lsn(),
    pg_last_wal_replay_lsn(),
    pg_wal_lsn_diff(pg_last_wal_receive_lsn(), pg_last_wal_replay_lsn()) AS apply_lag_bytes;
```

| Metric | Where | What It Shows |
|--------|-------|--------------|
| `pg_stat_replication` | Primary | All connected standbys, LSN positions, lag |
| `replay_lag` | Primary | Time-based lag estimate per standby |
| `pg_last_xact_replay_timestamp()` | Standby | When the last replayed transaction was committed on primary |
| `pg_wal_lsn_diff()` | Either | Byte difference between two LSN positions |

### Cascading Replication

A standby can stream to **its own standbys**, reducing load on the primary.

```
Primary → Standby A → Standby B → Standby C
                    ↘ Standby D
```

```ini
# On Standby A — allow downstream connections
max_wal_senders = 5

# On Standby B — connect to Standby A, not the primary
primary_conninfo = 'host=standby_a_host user=repl_user ...'
```

---

## 3. Logical Replication

### How It Works

Logical replication decodes WAL into row-level changes and applies them via the **publish/subscribe** model.

```
Publisher (source)                    Subscriber (target)
┌──────────────────┐                  ┌──────────────────┐
│  Table: orders   │                  │  Table: orders   │
│                  │                  │  (pre-created)   │
│  WAL → logical   │ ── changes ──►  │  apply worker    │
│  decoding        │                  │  applies rows    │
└──────────────────┘                  └──────────────────┘
```

| Component | Role |
|-----------|------|
| **Publication** | Defines which tables to replicate from the publisher |
| **Subscription** | Connects to the publisher and receives changes |
| **Logical decoding** | Converts WAL into row-level change events |
| **Replication slot** | Automatically created for each subscription; tracks progress |

### Setting Up Publication & Subscription

#### Publisher (Source)

```ini
# postgresql.conf on the publisher
wal_level = logical    # required (higher than 'replica')
max_replication_slots = 10
max_wal_senders = 10
```

```sql
-- Create a publication for specific tables
CREATE PUBLICATION my_pub FOR TABLE orders, customers;

-- Or publish all tables
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- Publish only specific operations
CREATE PUBLICATION orders_inserts FOR TABLE orders
    WITH (publish = 'insert');   -- only INSERT, not UPDATE/DELETE

-- Publish with row filter (PG 15+)
CREATE PUBLICATION active_orders FOR TABLE orders
    WHERE (status != 'archived');

-- Publish specific columns (PG 15+)
CREATE PUBLICATION order_basics FOR TABLE orders (id, customer_id, total, status);
```

#### Subscriber (Target)

```sql
-- Tables must exist on the subscriber with compatible schema
CREATE TABLE orders (LIKE orders_on_publisher INCLUDING ALL);
CREATE TABLE customers (LIKE customers_on_publisher INCLUDING ALL);

-- Create a subscription
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=publisher_host dbname=mydb user=repl_user password=...'
    PUBLICATION my_pub;

-- The subscriber:
-- 1. Creates a replication slot on the publisher
-- 2. Copies initial data (initial table sync)
-- 3. Starts streaming ongoing changes
```

```sql
-- Monitor subscription status
SELECT subname, pid, received_lsn, latest_end_lsn,
       latest_end_time
FROM pg_stat_subscription;

-- Check initial sync progress
SELECT relname, srsubstate
FROM pg_subscription_rel
JOIN pg_class ON pg_class.oid = srrelid;
-- srsubstate: 'i' = initializing, 'd' = data copy, 's' = synced, 'r' = ready
```

### Conflict Handling

Logical replication can encounter conflicts (e.g., unique key violation on the subscriber):

```sql
-- On subscriber: check for replication errors
SELECT * FROM pg_stat_subscription;
-- If the apply worker has stopped, check PostgreSQL logs for the error

-- Fix the conflict (e.g., delete the conflicting row)
DELETE FROM orders WHERE id = 123;  -- remove the conflicting row

-- Or skip the conflicting transaction (PG 15+)
ALTER SUBSCRIPTION my_sub SKIP (lsn = '0/12345678');

-- The apply worker will restart automatically
```

| Conflict Type | Common Cause | Resolution |
|---------------|-------------|-----------|
| Unique violation | Row already exists on subscriber | Delete conflicting row or skip transaction |
| Missing FK target | Referenced row doesn't exist on subscriber | Insert the missing referenced row |
| Apply worker crash | Repeated error | Fix data, then `ALTER SUBSCRIPTION ... ENABLE` |

### Use Cases

| Use Case | Why Logical Replication |
|----------|----------------------|
| **Cross-version upgrade** | Replicate from PG 16 → PG 18, then switch apps to subscriber |
| **Selective replication** | Replicate only specific tables to an analytics database |
| **Multi-datacenter** | Replicate subsets of data to regional databases |
| **Data integration** | Feed changes to external systems (combine with logical decoding plugins) |
| **Multi-master (limited)** | Bi-directional replication with careful conflict handling |

### Limitations

| Limitation | Detail |
|-----------|--------|
| **DDL not replicated** | Schema changes must be applied manually on both sides |
| **Sequences not replicated** | Sequence values on subscriber may diverge; sync manually |
| **Large objects** | Not supported |
| **TRUNCATE** | Supported (PG 11+) but must be explicitly included in publication |
| **Initial data copy** | Can be slow for large tables; happens over a single connection |
| **Conflicts** | No built-in auto-resolution; manual intervention needed |

### Managing Publications & Subscriptions

```sql
-- Add tables to an existing publication
ALTER PUBLICATION my_pub ADD TABLE new_table;

-- Remove tables
ALTER PUBLICATION my_pub DROP TABLE old_table;

-- Refresh subscription (pick up new tables)
ALTER SUBSCRIPTION my_sub REFRESH PUBLICATION;

-- Disable/enable subscription
ALTER SUBSCRIPTION my_sub DISABLE;
ALTER SUBSCRIPTION my_sub ENABLE;

-- Drop (also drops the replication slot on publisher)
DROP SUBSCRIPTION my_sub;

-- View publications
SELECT * FROM pg_publication;
SELECT * FROM pg_publication_tables;

-- View subscriptions
SELECT * FROM pg_subscription;
```

---

## 4. Streaming vs Logical Replication

| Feature | Streaming (Physical) | Logical |
|---------|:-------------------:|:-------:|
| **Replicates** | Entire WAL (all databases) | Selected tables |
| **Cross-version** | ❌ Same major version | ✅ Different versions |
| **Cross-platform** | ❌ Same architecture | ✅ Different architectures |
| **DDL replication** | ✅ Automatic (part of WAL) | ❌ Manual |
| **Selective tables** | ❌ All or nothing | ✅ Per-table control |
| **Standby queries** | ✅ Hot standby (read-only) | ✅ Full read-write on subscriber |
| **Failover** | ✅ Promote standby | ⚠️ Not designed for failover |
| **Performance overhead** | Low (byte streaming) | Higher (logical decoding) |
| **Conflict handling** | N/A (standby is read-only) | Manual resolution needed |
| **Setup complexity** | Low | Medium |
| **Best for** | HA, disaster recovery, read scaling | Migration, selective sync, integration |

---

## 5. Failover & Switchover

### Manual Failover

When the primary goes down and you need to promote a standby.

```sql
-- On the standby: promote to primary
SELECT pg_promote();
-- Or from command line:
-- pg_ctl promote -D /var/lib/postgresql/18/main
```

After promotion:
1. The standby stops replay, removes `standby.signal`, and opens for read-write.
2. A new **timeline** is created (prevents mixing old and new WAL).
3. Other standbys must be reconfigured to follow the new primary.

```sql
-- Reconfigure other standbys to follow the new primary
-- On each remaining standby:
ALTER SYSTEM SET primary_conninfo = 'host=new_primary_host user=repl_user ...';
SELECT pg_reload_conf();
```

### Manual Failover Checklist

```
1. ❌ Confirm primary is truly down (not just slow)
2. ✅ Choose the most up-to-date standby (lowest replay lag)
3. ✅ Promote: SELECT pg_promote()
4. ✅ Update DNS/VIP to point to new primary
5. ✅ Reconfigure remaining standbys to follow new primary
6. ✅ Verify replication is flowing to all standbys
7. ✅ Investigate and repair the old primary
8. ✅ Optionally rebuild old primary as a new standby
```

### Automatic Failover — Patroni

**Patroni** is the most popular open-source HA solution for PostgreSQL.

```
┌──────────────────────────────────────────────────┐
│                  Patroni Cluster                  │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Node 1   │  │ Node 2   │  │ Node 3   │       │
│  │ Patroni  │  │ Patroni  │  │ Patroni  │       │
│  │ PG (pri) │  │ PG (stby)│  │ PG (stby)│       │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘       │
│       │              │              │             │
│       └──────────────┼──────────────┘             │
│                      │                            │
│              ┌───────┴──────┐                     │
│              │  DCS (etcd / │                     │
│              │  Consul /    │                     │
│              │  ZooKeeper)  │                     │
│              └──────────────┘                     │
└──────────────────────────────────────────────────┘
```

| Component | Role |
|-----------|------|
| **Patroni agent** | Runs alongside PostgreSQL on each node; manages lifecycle |
| **DCS** (Distributed Configuration Store) | etcd, Consul, or ZooKeeper — provides leader election and cluster state |
| **Leader key** | The node holding the DCS leader lock is the primary |
| **Health checks** | Patroni monitors PG health, triggers failover when primary fails |

Key Patroni features:
- **Automatic failover** — promotes the best standby within seconds
- **Fencing** — ensures old primary cannot accept writes (prevents split brain)
- **Switchover** — planned primary swap via REST API or `patronictl`
- **Bootstrap** — initializes new cluster members automatically
- **Configuration management** — manages `postgresql.conf` and replication setup

```bash
# Patroni commands
patronictl list           # show cluster state
patronictl switchover     # planned switchover (interactive)
patronictl failover       # manual failover (when auto didn't trigger)
patronictl restart node1  # restart PostgreSQL on a node
```

Other HA solutions:

| Tool | Approach |
|------|---------|
| **Patroni** | Agent-based, DCS for leader election. Most popular. |
| **pg_auto_failover** | Citus extension, monitor node for failover decisions. Simpler setup. |
| **Stolon** | Agent-based with proxy. Less common now. |
| **Corosync/Pacemaker** | Traditional Linux HA stack. Complex but proven. |
| **Cloud-managed** | AWS RDS, GCP Cloud SQL, Azure — fully managed HA. |

### Switchover (Planned)

A controlled swap of primary and standby roles — for maintenance, patching, or hardware migration.

```bash
# With Patroni
patronictl switchover --leader node1 --candidate node2

# The sequence:
# 1. Patroni ensures node2 is in sync
# 2. node1 is demoted to standby
# 3. node2 is promoted to primary
# 4. Other standbys follow node2
# 5. DNS/VIP updated
```

> **Key difference**: Failover is **unplanned** (primary died). Switchover is **planned** (zero data loss, coordinated swap).

### Split Brain Prevention

Split brain = two nodes both think they're the primary. This causes data divergence and corruption.

| Prevention Method | How |
|------------------|-----|
| **DCS leader lock** (Patroni) | Only the lock holder can be primary |
| **Fencing / STONITH** | Power off or isolate the old primary before promoting new one |
| **`primary_conninfo` verification** | Standbys only follow a verified primary |
| **Replication slot monitoring** | Detect if a former primary is still accepting writes |
| **Network partitioning aware** | Quorum-based decisions (majority must agree) |

---

## 6. High Availability Architectures

### Single Primary + Async Standbys (Most Common)

```
Primary ──async──► Standby 1 (hot standby for reads)
        ──async──► Standby 2 (hot standby for reads)
```

- Simple to set up
- Slight risk of data loss on failover (async gap)
- Good for read scaling

### Single Primary + Sync Standby (Zero Data Loss)

```
Primary ──sync───► Standby 1 (guarantees data safety)
        ──async──► Standby 2 (extra read replica)
```

- Zero data loss on failover to sync standby
- Higher write latency (network round-trip)
- Synchronous standby failure stalls writes — use `ANY 1 (s1, s2)` for availability

### Multi-Region with Logical Replication

```
Region A (primary)                Region B (subscriber)
┌──────────────────┐              ┌──────────────────┐
│ Full read-write  │ ──logical──► │ Selected tables   │
│                  │              │ Read-write local  │
└──────────────────┘              └──────────────────┘
```

- Each region has its own read-write database
- Selective data sharing via publications
- Conflict handling needed for bi-directional

### Decision Guide

| Requirement | Architecture |
|------------|-------------|
| Simple HA + read scaling | Primary + async standbys + Patroni |
| Zero data loss | Primary + sync standby + async standby |
| Cross-version migration | Logical replication (temporary) |
| Multi-region presence | Logical replication per region |
| Maximum simplicity | Cloud-managed (RDS, Cloud SQL) |

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| No replication slots | WAL recycled before standby catches up | Create slots, but monitor for dead slots |
| Dead replication slot left active | WAL retained forever, disk fills | Monitor `pg_replication_slots`, drop unused |
| `synchronous_commit = on` with unreachable standby | Primary writes stall indefinitely | Use `ANY 1 (s1, s2)` or fallback to async |
| Promoting without checking lag | Data loss — standby missed recent transactions | Check `replay_lag` before promoting |
| Not reconfiguring remaining standbys after failover | Standbys try to connect to dead old primary | Update `primary_conninfo` on all standbys |
| Using logical replication for HA failover | Not designed for it — no automatic failover | Use streaming replication for HA |
| Forgetting DDL on logical subscriber | Schema drift causes apply errors | Apply DDL manually on both sides |

---

## What to Learn Next

1. **Backup & PITR** — Physical backups as the foundation for standbys — see [06-backup-restore.md](06-backup-restore.md).
2. **Connection Pooling** — Manage connections across primary and replicas — see [24-connection-pooling-load-balancing.md](24-connection-pooling-load-balancing.md).
3. **WAL Internals** — Understanding what WAL records are streamed — see [06-backup-restore.md § WAL Archiving](06-backup-restore.md).
4. **Authentication & SSL** — Secure replication connections — see [21-authentication-ssl.md](21-authentication-ssl.md).
5. **Performance Tuning** — Optimizing replication performance — Phase 11.

---

> *Ref: [Docs — Streaming Replication](https://www.postgresql.org/docs/18/warm-standby.html) · [Docs — Logical Replication](https://www.postgresql.org/docs/18/logical-replication.html) · [Docs — High Availability](https://www.postgresql.org/docs/18/high-availability.html) · [Patroni Docs](https://patroni.readthedocs.io/) · [Neon — PostgreSQL Administration](https://neon.com/postgresql/postgresql-administration)*
