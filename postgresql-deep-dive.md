# PostgreSQL Deep Dive — From Java Developer to Senior PostgreSQL Engineer

> A production-grade reference for understanding how PostgreSQL works *inside*, debugging real incidents, and designing systems that don't melt under load.
>
> **Audience:** Backend engineers (Java/Spring) with working PostgreSQL experience who want the mental models, internals, and diagnostic skills of a senior database engineer.

---

## Table of Contents

1. [PostgreSQL Architecture](#1-postgresql-architecture)
2. [MVCC — The Core Concept](#2-mvcc--the-core-concept)
3. [WAL — Write-Ahead Logging](#3-wal--write-ahead-logging)
4. [Vacuum & Autovacuum](#4-vacuum--autovacuum)
5. [Query Execution Engine](#5-query-execution-engine)
6. [EXPLAIN / EXPLAIN ANALYZE](#6-explain--explain-analyze)
7. [Indexing Deep Dive](#7-indexing-deep-dive)
8. [Transactions & Isolation Levels](#8-transactions--isolation-levels)
9. [Locking & Concurrency](#9-locking--concurrency)
10. [Performance Tuning](#10-performance-tuning)
11. [Partitioning & Scaling](#11-partitioning--scaling)
12. [Replication & High Availability](#12-replication--high-availability)
13. [Backup & Recovery](#13-backup--recovery)
14. [Monitoring & Debugging](#14-monitoring--debugging)
15. [Advanced PostgreSQL Features](#15-advanced-postgresql-features)
16. [Real-World Debugging Scenarios](#16-real-world-debugging-scenarios)
17. [Production Best Practices](#17-production-best-practices)
18. [When PostgreSQL Becomes a Bottleneck](#18-when-postgresql-becomes-a-bottleneck)
19. [Final Checklist](#19-final-checklist)

---

## 1. PostgreSQL Architecture

### 1.1 The Mental Model

PostgreSQL is a **multi-process**, **shared-memory** database. Not threaded. Each client connection is a separate OS process. This one fact explains most of its operational behavior: why connection pooling is mandatory, why per-connection memory adds up, why `max_connections` is a real ceiling.

```
                            ┌───────────────────────────┐
                            │        Postmaster         │
                            │  (parent, listens on 5432)│
                            └─────────────┬─────────────┘
                                          │ fork()
           ┌──────────────────────────────┼──────────────────────────────┐
           ▼                              ▼                              ▼
    ┌───────────┐                 ┌───────────┐                 ┌───────────┐
    │ Backend 1 │  …              │ Backend N │                 │Background │
    │ (client)  │                 │ (client)  │                 │ workers   │
    └─────┬─────┘                 └─────┬─────┘                 └─────┬─────┘
          │                             │                             │
          └─────────────────────────────┼─────────────────────────────┘
                                        ▼
                          ╔══════════════════════════╗
                          ║     Shared Memory         ║
                          ║  shared_buffers (pages)   ║
                          ║  WAL buffers              ║
                          ║  Lock table, CLOG, procs  ║
                          ╚═════════════╤═════════════╝
                                        │
      ┌──────────────┬──────────────────┼────────────────┬──────────────┐
      ▼              ▼                  ▼                ▼              ▼
 ┌──────────┐  ┌───────────┐      ┌─────────────┐  ┌──────────┐  ┌─────────────┐
 │ WAL      │  │ Bgwriter  │      │ Checkpointer│  │Autovacuum│  │ Stats       │
 │ writer   │  │           │      │             │  │ launcher │  │ collector   │
 └──────────┘  └───────────┘      └─────────────┘  └──────────┘  └─────────────┘
                                        │
                                        ▼
                                  ┌──────────┐
                                  │   Disk   │
                                  │ base/    │
                                  │ pg_wal/  │
                                  └──────────┘
```

### 1.2 Process Architecture

- **Postmaster** — the parent. Listens on TCP, authenticates, and `fork()`s a backend per connection.
- **Backend process** — handles one client's queries. Gets its own memory (work_mem, maintenance_work_mem, catalog cache) plus access to shared memory.
- **Background writer** — trickles dirty pages from shared_buffers to disk so checkpoints aren't a spike.
- **Checkpointer** — runs periodic checkpoints: flushes all dirty pages and writes a checkpoint record to WAL.
- **WAL writer** — flushes WAL buffers to disk asynchronously; `COMMIT` forces a synchronous flush.
- **Autovacuum launcher + workers** — vacuums and analyzes tables based on thresholds.
- **Stats collector / cumulative statistics** — feeds `pg_stat_*` views.
- **Replication processes** — walsender, walreceiver, logical replication workers (when replication is on).
- **Parallel workers** — spawned per query when the planner chooses parallelism.

A connection = a process ≈ 2–10 MB baseline RAM. 500 connections = several GB before a single query runs. This is the #1 reason people add PgBouncer.

### 1.3 Memory Architecture

| Area | Scope | Default | What it holds |
|---|---|---|---|
| `shared_buffers` | Shared | 128 MB | Cached pages (the buffer pool) |
| `wal_buffers` | Shared | -1 (auto, ~16 MB) | WAL not yet written to disk |
| `work_mem` | Per operation per session | 4 MB | Sorts, hash tables, per node in a plan |
| `maintenance_work_mem` | Per maintenance op | 64 MB | VACUUM, CREATE INDEX, ALTER TABLE |
| `temp_buffers` | Per session | 8 MB | Temp-table pages |
| OS page cache | Kernel | All free RAM | Also caches PG data files |

**Critical detail — `work_mem` multiplies.** It's *per sort/hash node* in a plan, *per concurrent query*. A plan with 3 sort nodes × 200 concurrent queries × 64 MB `work_mem` = 38 GB allocated at peak. Tune conservatively, then override per-session for heavy reports:

```sql
SET work_mem = '256MB';
```

**Postgres relies on the OS page cache.** That's why `effective_cache_size` is a *hint* to the planner about total available cache (shared_buffers + OS cache), not an allocation.

### 1.4 Disk Layout

```
$PGDATA/
├── base/               # one subdir per database, one file per table/index (1GB segments)
│   └── 16384/
│       ├── 24576         # table/index data file (oid)
│       ├── 24576_fsm     # free space map
│       ├── 24576_vm      # visibility map
│       └── 24576.1       # next 1GB segment of same relation
├── global/             # cluster-wide catalogs (pg_database, pg_authid)
├── pg_wal/             # WAL segments (16MB each by default)
├── pg_xact/            # transaction commit log (CLOG)
├── pg_multixact/       # multi-transaction metadata (row locking)
├── pg_stat/, pg_stat_tmp/
├── pg_tblspc/          # symlinks to tablespaces
├── postgresql.conf
├── pg_hba.conf         # client auth
└── postmaster.pid
```

- **Page** (8 KB) — the atomic unit of I/O.
- **Relation** — table, index, sequence, toast table. Each has a file; once it exceeds 1 GB a new segment is created (`.1`, `.2`, …).
- **TOAST** — The Oversized Attribute Storage Technique. Large column values (>~2KB) are compressed and moved to a side table (`pg_toast.*`). Transparent to queries but measurable in EXPLAIN (BUFFERS).
- **Free Space Map (`_fsm`)** — where new tuples can go.
- **Visibility Map (`_vm`)** — which pages are all-visible (used by index-only scans and vacuum).
- **Tablespaces** — let you move a table/index to a different filesystem. Rarely needed today — cloud block devices are already fast; volume management outside Postgres is simpler.

### 1.5 How a Query Flows Through PostgreSQL

```
SQL string
   │
   ▼
[1] Parser               → syntactic tree
   │
   ▼
[2] Rewriter             → apply views, rules
   │
   ▼
[3] Planner/Optimizer    → enumerate plans, cost them, pick cheapest
   │
   ▼
[4] Executor             → pull-based Volcano model; each node emits tuples
   │        │
   │        ├── touches shared_buffers → OS cache → disk
   │        ├── acquires locks (shared/exclusive)
   │        └── writes WAL for modifications
   ▼
Result sent to client
```

### 1.6 What Happens Internally for SELECT / INSERT / UPDATE / DELETE

**SELECT:**
1. Parse + plan.
2. Executor acquires an `AccessShareLock` on each table (lightweight, co-exists with everything except `AccessExclusiveLock`).
3. Reads pages: shared_buffers → OS cache → disk.
4. For each tuple, MVCC visibility check against the current snapshot.
5. Emits visible tuples upward.

**INSERT:**
1. Find a page with free space (Free Space Map).
2. Write the tuple with `xmin = current_txid`, `xmax = 0`.
3. Emit WAL record *before* the page is considered dirty-durable.
4. Update indexes (each index write is a separate WAL record).
5. Mark page dirty; checkpoint will flush it later.

**UPDATE (this is the twist):**
1. Mark old tuple's `xmax = current_txid` (logical delete).
2. Insert a **new tuple** with the updated values.
3. If the new tuple fits on the same page **and** no indexed column changed, HOT (Heap-Only Tuple) update avoids updating indexes — a significant optimization.
4. WAL records for old-tuple mark + new-tuple insert + index updates (if any).

So `UPDATE` is really `delete-then-insert` at the tuple level. This is the seed of MVCC and of bloat.

**DELETE:**
1. Mark the tuple's `xmax = current_txid`.
2. WAL record.
3. The row is not actually removed from disk. `VACUUM` will reclaim it later.

---

## 2. MVCC — The Core Concept

**Multi-Version Concurrency Control** is the single most important PostgreSQL concept. Internalize it and 80% of PG's behavior becomes obvious.

### 2.1 The Problem MVCC Solves

Classic locking: readers take shared locks, writers take exclusive locks. Readers block writers; writers block readers. Under contention, throughput collapses.

MVCC's answer: **writers never overwrite data in place; they create new versions.** Readers see a consistent snapshot of data "as of" their transaction's start (or statement start). No read locks needed.

Slogan: **readers don't block writers; writers don't block readers.**

### 2.2 Tuple Versioning

Every row on disk has system columns (invisible in `SELECT *` but queryable):

| Column | Meaning |
|---|---|
| `xmin` | Transaction ID that **created** this version |
| `xmax` | Transaction ID that **invalidated** this version (0 if still live) |
| `cmin` / `cmax` | Command IDs within the transaction (rare to care) |
| `ctid` | Physical tuple location `(block, offset)` |

Inspect:

```sql
SELECT xmin, xmax, ctid, id, name FROM customers WHERE id = 1;
```

### 2.3 Life of a Row — Step by Step

Two concurrent transactions; tuple has id=1, name='Alice'.

```
Initial state (committed row):
  (xmin=100, xmax=0, name='Alice')

T200 starts:
  BEGIN;
  -- sees xmin=100 committed, xmax=0 → visible

T201 starts, updates:
  BEGIN;
  UPDATE customers SET name='Alex' WHERE id=1;
  -- old tuple now: (xmin=100, xmax=201, name='Alice')
  -- new tuple:    (xmin=201, xmax=0,   name='Alex')
  -- both exist on disk simultaneously

T200 reads:
  SELECT name FROM customers WHERE id=1;
  -- old tuple: xmin=100 committed < T200's snapshot, xmax=201 not committed → VISIBLE
  -- new tuple: xmin=201 not yet committed → INVISIBLE
  -- result: 'Alice' ✓

T201 commits.

T200 reads again (READ COMMITTED):
  -- new statement, new snapshot
  -- old tuple: xmax=201 now committed → INVISIBLE
  -- new tuple: xmin=201 committed → VISIBLE
  -- result: 'Alex'

T202 starts:
  -- old tuple is dead for everyone now.
  -- VACUUM will reclaim its space eventually.
```

### 2.4 Visibility Rules (Simplified)

A tuple is visible to transaction T with snapshot S when:

```
xmin is committed AND xmin < S.xmax AND xmin NOT IN S.xip_list
AND (xmax is 0
     OR xmax is aborted
     OR xmax >= S.xmax
     OR xmax IN S.xip_list)
```

`S.xip_list` is the set of transactions in progress at snapshot time. Reads: "I see you if you were committed before my snapshot, and I don't see deletions made by transactions still running when my snapshot was taken."


### 2.5 Snapshot Isolation

A *snapshot* records:
- `xmin` — oldest still-running transaction
- `xmax` — next transaction ID to be assigned
- `xip_list` — in-progress transactions at snapshot time

A tuple is visible if its creator committed before `xmax` and isn't in the in-progress list, and its deleter either hasn't committed or is in the in-progress list.

- **READ COMMITTED** — new snapshot per statement. This is the default.
- **REPEATABLE READ** — one snapshot for the whole transaction. Postgres's RR is *snapshot isolation*, stronger than the SQL standard: no phantom reads, because a single snapshot is used.
- **SERIALIZABLE** — SSI (Serializable Snapshot Isolation). Tracks read/write dependencies; aborts transactions that would produce a non-serializable schedule.

### 2.6 Consequences You Must Know

1. **`UPDATE` is `INSERT`-heavy.** Every update creates a new version. Heavy update workloads = high bloat risk.
2. **Long-running transactions are poison.** A transaction started an hour ago is still holding a snapshot → vacuum cannot remove *any* dead tuples newer than that snapshot → bloat across the whole DB.
3. **`COUNT(*)` is slow.** Postgres must check visibility per row; there is no single cached count. (Use estimates from `pg_class.reltuples` if exact isn't needed.)
4. **Readers never block writers, but writers **can** block writers.** Two sessions updating the same row: the second waits on a tuple lock (Postgres stores this in the tuple header + in `pg_multixact` for shared locks).
5. **Index entries point to tuples, not rows.** Each tuple version has its own index entry; HOT updates avoid this when possible.
6. **HOT (Heap-Only Tuple).** If the new version fits on the same page and no indexed columns changed, the indexes continue pointing at the old tuple via a chain; massive write optimization.

### 2.7 Common Misconceptions

| Myth | Reality |
|---|---|
| "`DELETE` frees space" | It marks dead; VACUUM frees. |
| "Index rebuild fixes bloat" | Only index bloat. Heap bloat needs `VACUUM FULL` or repack. |
| "Higher isolation = safer always" | Also more retries (serialization failures). Design for them. |
| "`SELECT` is free" | Under snapshot isolation it blocks vacuum from reclaiming. |
| "`VACUUM FULL` is just a bigger `VACUUM`" | Totally different: rewrites the table, takes AccessExclusiveLock. |

---

## 3. WAL — Write-Ahead Logging

### 3.1 Why WAL Exists

Durability requires that committed data survives crashes. But flushing each 8 KB page to disk on every `COMMIT` would be slow and random. WAL inverts the problem:

> **Rule: log the change before changing the page.** Flush WAL (sequential, small, fast) on commit. Flush data pages lazily, at checkpoints.

Benefits:
- Commits are fast (sequential fsync of WAL, not random page writes).
- Crash recovery = "replay WAL from last checkpoint."
- Replication is a byproduct: stream WAL to replicas.
- PITR is a byproduct: base backup + archived WAL = recover to any point in time.

### 3.2 How WAL Works

1. Backend modifies a page in shared_buffers.
2. Before marking the page dirty, it writes a **WAL record** into WAL buffers describing the change (logical + physical enough to redo).
3. At `COMMIT`, the WAL up to that point is flushed (`fsync`) to `pg_wal/` on disk.
4. Only then does the client receive "commit OK."
5. The dirty data page sits in shared_buffers until:
   - Background writer trickles it out, OR
   - The checkpointer flushes it at the next checkpoint.

Each WAL file is 16 MB (configurable). Files are numbered and keep advancing. When replication or archiving are on, old WAL must be retained until consumers catch up.

### 3.3 Checkpoints

A checkpoint is the moment where **all dirty pages are flushed**, and a marker is written to WAL. After a checkpoint, WAL before it is no longer needed for crash recovery.

Triggered by:
- `checkpoint_timeout` elapsed (default 5 min, tune to 15 min for less I/O noise)
- `max_wal_size` exceeded
- Manual `CHECKPOINT;` command
- Shutdown

Parameters:
- `checkpoint_timeout` — time-based trigger
- `max_wal_size` — size-based trigger (2–8 GB typical)
- `checkpoint_completion_target` — spread writes across this fraction of the interval (0.9 default — smooths I/O)

A checkpoint that can't finish before the next one triggers = "checkpoints occurring too frequently" warning — raise `max_wal_size`.

### 3.4 Crash Recovery

On startup after a crash:

1. Find the last checkpoint record in WAL.
2. Replay every WAL record after it, in order, against the data files.
3. Discard any uncommitted transactions (their changes exist in WAL but without a commit record).
4. Open for connections.

This is why committed transactions survive: the commit record is in WAL, so replay reconstructs them.

### 3.5 Performance Implications

- WAL is sequential — keep it on fast disk. On cloud, a dedicated EBS volume or fast local NVMe.
- `synchronous_commit = off` — **dangerous trade-off.** Commit returns before WAL is fsynced. On crash you can lose the last ~3× wal_writer_delay of commits. Tables stay consistent; just committed data can disappear. Use only for non-critical data (analytics, idempotent retriable work).
- `full_page_writes = on` (default) — after a checkpoint, the first modification to a page writes the full page image into WAL. Protects against torn writes; significant WAL size contributor.
- `wal_compression = on` — compresses full-page images; trades CPU for WAL bytes; usually a win.
- Long-running transactions + heavy writes → WAL piles up if archiving or replicas lag.

### 3.6 Real Scenarios

**Scenario:** Replica lags 30 minutes under load.
- WAL keeps accumulating on primary (can't be recycled until replica consumes it).
- Disk on primary fills up.
- Fix: `max_wal_size`, a replication slot with safeguards (`max_slot_wal_keep_size`), monitor `pg_stat_replication.write_lsn` vs `flush_lsn`.

**Scenario:** Checkpoints cause periodic latency spikes.
- Symptom: p99 query time spikes every N minutes.
- Investigate `log_checkpoints = on` output.
- Fixes: raise `checkpoint_timeout` and `max_wal_size` so checkpoints are less frequent; raise `checkpoint_completion_target` closer to 0.9 to spread I/O.

---

## 4. Vacuum & Autovacuum

MVCC leaves dead tuples. Somebody must clean up. That's VACUUM's job. Failure to vacuum well = table bloat = slow queries = midnight pages.

### 4.1 What Dead Tuples Are

Any tuple whose `xmax` is committed and older than the oldest running transaction is "dead." Its space can be reclaimed.

But space is only reclaimable if **no transaction could still need to see it.** This is why a long-running transaction (even if it's reading an unrelated table) blocks cleanup globally.

### 4.2 What VACUUM Does

1. Scans the table.
2. Finds dead tuples.
3. Marks their space as reusable in the Free Space Map.
4. Updates the Visibility Map (all-visible pages).
5. Cleans up related index entries (in default mode).
6. Updates table statistics (if combined with ANALYZE).

It does **not**:
- Compact the table (heap keeps its size; future inserts reuse the freed slots)
- Return disk to the OS (truncation happens only if the tail of the table is fully empty)

Variants:
- `VACUUM` — standard; online; takes weak locks.
- `VACUUM FULL` — rewrites the table into a new file, compact. **Takes `AccessExclusiveLock` — blocks everything on the table.** Do NOT run casually.
- `VACUUM FREEZE` — aggressive: freezes old tuples to prevent transaction ID wraparound.
- `VACUUM (ANALYZE)` — also refresh statistics.
- `pg_repack` / `pg_squeeze` — online table compaction without exclusive lock. Use in production instead of `VACUUM FULL`.

### 4.3 Autovacuum

A background process that decides when each table needs vacuum / analyze based on thresholds:

```
dead_tuples_threshold = vacuum_scale_factor * n_live_tup + vacuum_threshold
```

Defaults: `scale_factor = 0.2`, `threshold = 50`. For a 10M-row table that means waiting for ~2M dead tuples before vacuuming. On hot tables that's way too late.

**Tune per-table:**

```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor  = 0.02,   -- 2%
  autovacuum_analyze_scale_factor = 0.01,
  autovacuum_vacuum_cost_limit    = 2000,   -- let it work faster
  autovacuum_vacuum_cost_delay    = 10ms
);
```

Autovacuum cost-based throttling (`vacuum_cost_limit`, `vacuum_cost_delay`) exists so vacuum doesn't swamp the I/O subsystem. On modern SSDs these limits are too conservative — raise them for hot tables.

### 4.4 Transaction ID Wraparound (Know This or Suffer)

Transaction IDs are 32-bit. Postgres uses a modular comparison that assumes no live xid is more than 2^31 apart from another. If you never vacuum, you eventually approach wraparound; Postgres will refuse writes to protect data integrity:

```
ERROR: database is not accepting commands to avoid wraparound data loss in database "app"
HINT:  Stop the postmaster and vacuum that database in single-user mode.
```

This is catastrophic — downtime to recover. Autovacuum automatically escalates to a `FREEZE` vacuum when approaching the limit. If autovacuum is disabled or starved, you get wraparound. **Never disable autovacuum globally.**

Monitor with:

```sql
SELECT datname, age(datfrozenxid) AS xid_age
FROM pg_database ORDER BY xid_age DESC;
```

Keep the max well under `autovacuum_freeze_max_age` (default 200M; hard wall at 2B).

### 4.5 Bloat — Real-World Problem

Bloat = physical table/index size much larger than the live data.

Causes:
- Autovacuum can't keep up (scale factor too high, cost limits too low)
- Long-running transactions hold snapshots
- Abandoned replication slots pin WAL (indirect: the WAL-based feedback loop)
- Very high update rate to a few hot pages

Detect:

```sql
SELECT relname,
       n_live_tup, n_dead_tup,
       CASE WHEN n_live_tup > 0
            THEN round(100.0 * n_dead_tup / n_live_tup, 2) END AS dead_pct,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 20;
```

For physical bloat estimates, use the `pgstattuple` extension or community scripts (check-postgres, pgexperts bloat query).

Fix:
- Tune autovacuum more aggressively per-table
- `pg_repack` during off-hours for big tables
- Kill the long-running transaction that's blocking cleanup
- Drop unused indexes (index bloat adds up too)

### 4.6 Debugging Vacuum Issues

```sql
-- Is autovacuum running? On which tables?
SELECT pid, datname, relid::regclass, phase, heap_blks_scanned, heap_blks_total
FROM pg_stat_progress_vacuum;

-- Which table hasn't been vacuumed recently?
SELECT relname, last_vacuum, last_autovacuum, n_dead_tup
FROM pg_stat_user_tables
WHERE n_dead_tup > 10000
ORDER BY last_autovacuum NULLS FIRST;

-- Who's holding the oldest snapshot (blocking vacuum)?
SELECT pid, backend_xid, backend_xmin, state,
       now() - xact_start AS tx_age,
       query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin LIMIT 10;
```

---

## 5. Query Execution Engine

### 5.1 Planner vs Executor

- **Planner** turns the parsed query into a physical plan: a tree of operators (scans, joins, aggregates).
- **Executor** runs that tree, Volcano-style: each node's `Next()` pulls tuples from its children, transforms them, emits upward.

### 5.2 Cost-Based Optimization

The planner enumerates possible plans (join orderings, scan methods) and picks the cheapest. Cost is an abstract number; two inputs matter:

- **Statistics** — `pg_statistic`, refreshed by ANALYZE: row count, nulls, most common values, histograms, correlation.
- **Cost parameters:**
  - `seq_page_cost = 1.0`
  - `random_page_cost = 4.0` (default assumes spinning disk — **set 1.1 on SSD**)
  - `cpu_tuple_cost = 0.01`
  - `cpu_index_tuple_cost = 0.005`
  - `cpu_operator_cost = 0.0025`

Query cost ≈ `seq_pages × seq_page_cost + random_pages × random_page_cost + rows × cpu_tuple_cost + …`.

### 5.3 Scan Methods

| Scan | When chosen | Cost model |
|---|---|---|
| **Seq Scan** | Small table or many rows match | Pages × seq_page_cost |
| **Index Scan** | Few rows match; ordered output needed | Index pages (random) + heap lookups (random) |
| **Index-Only Scan** | All needed columns in index, VM says page all-visible | Index pages only |
| **Bitmap Heap Scan** | Medium selectivity; combines multiple indexes | Index pages + sorted heap pages (sequential-ish) |
| **TID Scan** | WHERE ctid = ... | Single page fetch |

Bitmap Heap Scan is the planner's answer to "this index returns lots of rows in random order; let me group them by page first." That's why you see it for medium-selectivity predicates.

### 5.4 Join Algorithms

#### Nested Loop

```
for each row in outer:
  for each matching row in inner:
    emit(outer, inner)
```

- Optimal when outer is small and inner has an index on join key.
- O(outer × inner) without index; O(outer × log inner) with index.
- `loops=N` in EXPLAIN ANALYZE tells you N inner executions.

#### Hash Join

```
1. Build hash table on smaller side (in work_mem)
2. Stream the larger side, probe the hash
```

- Optimal when no useful index, equality join, one side fits in memory.
- If hash doesn't fit: batched hash join, spills to disk (slow).

#### Merge Join

```
Both inputs sorted on the join key → zip them together
```

- Optimal when both inputs are already sorted (e.g., two index scans).
- Rarely the cheapest unless sort is already free.

### 5.5 Parallel Query

Postgres can use multiple workers for:
- Parallel seq scan
- Parallel index scan
- Parallel hash join (build phase sequential, probe parallel since PG11)
- Parallel aggregate

Controlled by `max_parallel_workers_per_gather`, `parallel_setup_cost`, `parallel_tuple_cost`. Works best for analytics queries over large tables; overhead defeats it for short queries.

### 5.6 Why Wrong Plans Happen

1. **Stale statistics.** Big data change without ANALYZE → planner thinks the table is smaller/less skewed than it is. Fix: `ANALYZE tablename`. For very skewed data, raise `default_statistics_target` (e.g., 500) for that column:
   ```sql
   ALTER TABLE events ALTER COLUMN category SET STATISTICS 1000;
   ```
2. **Correlated columns.** Planner assumes predicates on different columns are independent. They often aren't (city + country). Fix: extended statistics:
   ```sql
   CREATE STATISTICS events_city_country_stats (dependencies)
     ON city, country FROM events;
   ```
3. **Cost parameters wrong for hardware.** `random_page_cost = 4` on SSD biases toward seq scans. Set 1.1.
4. **Parameter sniffing / generic plans.** For prepared statements, PG may choose a generic plan that's bad for some parameter values. Diagnose via `pg_stat_statements` + `EXPLAIN` for specific parameter sets. Workaround: `plan_cache_mode = force_custom_plan` per session/query.
5. **work_mem too small.** Forces on-disk hash/sort spills and changes plan choice.
6. **Bad estimates from functions.** `SELECT * FROM t WHERE f(x) = ?` — planner doesn't know `f`'s selectivity. Consider an expression index, or rewrite.

---

## 6. EXPLAIN / EXPLAIN ANALYZE

This is the skill that separates seniors from juniors. Read plans like code.

### 6.1 The Commands

```sql
EXPLAIN SELECT ...;                       -- plan only; no execution
EXPLAIN ANALYZE SELECT ...;               -- actually runs; real times/rows
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;    -- add buffer/cache info
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS, WAL) SELECT ...;  -- everything
EXPLAIN (ANALYZE, FORMAT JSON) SELECT ...;  -- for tools (explain.depesz.com)
```

For DML:
```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS) UPDATE ...;
ROLLBACK;   -- so you don't actually change data
```

### 6.2 Anatomy of a Plan

```
 Hash Join  (cost=105.50..923.40 rows=2500 width=128)
            (actual time=2.34..12.67 rows=2473 loops=1)
   Hash Cond: (o.customer_id = c.id)
   Buffers: shared hit=412 read=8
   ->  Seq Scan on orders o (cost=0.00..710.00 rows=10000 width=64)
                            (actual time=0.01..5.12 rows=10000 loops=1)
   ->  Hash  (cost=80.00..80.00 rows=2000 width=64)
              (actual time=2.20..2.20 rows=2000 loops=1)
         Buckets: 4096 Batches: 1 Memory Usage: 264kB
         ->  Index Scan using customers_active_idx on customers c
               (cost=0.42..80.00 rows=2000 width=64)
               (actual time=0.02..1.80 rows=2000 loops=1)
 Planning Time: 0.41 ms
 Execution Time: 13.12 ms
```

- **cost=start..total** — planner's abstract cost.
- **rows=** — planner estimate.
- **actual time=start..end** — from node's first tuple to last. In ms.
- **rows=** (actual) — real row count.
- **loops=N** — the node ran N times (inner side of nested loop × outer row count).
- **Buffers: shared hit=X read=Y** — `hit` is in shared_buffers; `read` is OS cache / disk.

### 6.3 How to Read a Plan Like a Senior

1. **Identify the biggest `actual time` contributor.** Time accumulates bottom-up: parent includes children. Find the leaf/intermediate node where the clock goes from small to large.
2. **Compare `rows (estimated)` vs `actual rows`.** Ratio > 10× = stats problem. Planner's choice above this node might be wrong.
3. **Multiply `actual time × loops` for nested-loop inner nodes.** If you see `actual time=0.5, loops=50000`, that's 25 seconds, not 0.5 ms.
4. **Watch for seq scans on big tables.** Fine for small tables or most rows; terrible for needle-in-haystack.
5. **`Rows Removed by Filter: N`** — the scan read N rows just to discard them. Index candidate.
6. **`Sort Method: external merge Disk: XXXkB`** — sort spilled to disk. Raise work_mem.
7. **`Hash Batches > 1`** — hash didn't fit in memory. Raise work_mem.
8. **`Buffers: read=` high** — cold cache. After a warmup run the plan should show `hit` instead.

### 6.4 Step-by-Step Optimization Example

Query: list the 50 most recent PAID orders for a customer with customer name.

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.*, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.customer_id = 42 AND o.status = 'PAID'
ORDER BY o.created_at DESC
LIMIT 50;
```

Plan:
```
Limit (...)
  -> Sort (actual rows=50 loops=1)
       Sort Key: o.created_at DESC
       Sort Method: top-N heapsort Memory: 35kB
       -> Nested Loop
            -> Seq Scan on orders o (rows=23000, filter: customer_id=42 AND status='PAID')
                 Rows Removed by Filter: 9,977,000
            -> Index Scan on customers_pkey (rows=1 loops=23000)
 Execution Time: 3400 ms
```

Diagnosis:
- Seq Scan on 10M orders; 9.97M rows removed by filter → missing index.

Fix:
```sql
CREATE INDEX idx_orders_customer_status_created
  ON orders (customer_id, status, created_at DESC);
```

Replan:
```
Limit (actual rows=50 loops=1)
  -> Nested Loop
       -> Index Scan using idx_orders_customer_status_created on orders
            Index Cond: (customer_id=42 AND status='PAID')
            (actual rows=50 loops=1)
       -> Index Scan on customers_pkey (loops=50)
 Execution Time: 1.8 ms
```

1800× faster. Because:
- Index matches predicates `(customer_id, status)`
- Its ordering matches `ORDER BY created_at DESC` → no sort
- `LIMIT` stops after 50 fetches

Lesson: column order in a composite index follows `WHERE` equality → `WHERE` range / `ORDER BY`.

### 6.5 Tools

- **explain.depesz.com** — paste plan, get colored heatmap
- **explain.dalibo.com** — tree visualization
- **pgMustard** — advice engine
- **Auto-explain** extension — log slow plans automatically:
  ```
  shared_preload_libraries = 'auto_explain'
  auto_explain.log_min_duration = '500ms'
  auto_explain.log_analyze = true
  auto_explain.log_buffers = true
  ```

---

## 7. Indexing Deep Dive

### 7.1 Mental Model

An index is a **separate data structure** that answers "which tuples (ctids) match this condition?" without scanning the heap. Every index has a cost:
- Slower writes (each INSERT/UPDATE updates indexes)
- Disk space (often ~10–30% of table size)
- Maintenance overhead (vacuum, bloat)

So: index aggressively for *read* patterns you actually have, and drop unused indexes.

### 7.2 B-tree

Postgres's default. A self-balancing tree sorted by key. Supports:
- Equality: `=`, `IN`
- Range: `<`, `<=`, `>`, `>=`, `BETWEEN`
- Sort: `ORDER BY col` / `ORDER BY col DESC` (if `DESC` or `NULLS FIRST/LAST` matches index)
- Prefix pattern: `LIKE 'abc%'` (with C locale or `text_pattern_ops`)

Composite index `(a, b, c)`:
- Uses: `WHERE a=?`, `WHERE a=? AND b=?`, `WHERE a=? AND b=? AND c=?`, `WHERE a=? AND b>?`
- Doesn't use: `WHERE b=?` alone (no leading column)

**Column order rule:**
1. Equality predicates first
2. Then the column for range or ORDER BY
3. Most selective equality first among several

Covering (`INCLUDE`):
```sql
CREATE INDEX ON orders (customer_id, status) INCLUDE (total, created_at);
```
Columns in `INCLUDE` are stored in the index leaf but don't participate in sort/search. Enables index-only scans without changing the sort key.

### 7.3 Hash Index

Equality only. Historically less useful; became crash-safe and WAL-logged in Postgres 10. Can be slightly smaller/faster than B-tree for huge single-column equality workloads, but the ecosystem defaults to B-tree.

Rarely worth it.

### 7.4 GIN (Generalized Inverted Index)

An inverted index: maps each "key piece" → list of tuples containing it.

Use for:
- `jsonb` containment: `WHERE doc @> '{"type":"order"}'`
- Arrays: `WHERE tags @> ARRAY['priority']`
- Full-text search: `WHERE tsv @@ plainto_tsquery('english', 'postgres deep dive')`
- Trigrams with `pg_trgm`: `WHERE name ILIKE '%widget%'`

```sql
CREATE INDEX ON events USING GIN (payload jsonb_path_ops);
CREATE INDEX ON products USING GIN (name gin_trgm_ops);  -- pg_trgm
```

Trade-off: slower writes (GIN is write-amplifying); `gin_pending_list_limit` defers index merges for throughput.

### 7.5 GiST (Generalized Search Tree)

Framework for non-standard search problems. Use for:
- Geometric types (PostGIS)
- Ranges: `tstzrange`, `int4range` — e.g., overlap `&&`
- Nearest-neighbor: `ORDER BY location <-> point LIMIT 5`
- Exclusion constraints: ensure no two reservations overlap:
  ```sql
  ALTER TABLE bookings ADD EXCLUDE USING gist (room WITH =, during WITH &&);
  ```

### 7.6 BRIN (Block Range Index)

Tiny index storing min/max values per range of blocks. Works only when data has strong physical correlation with the indexed column (e.g., append-only time series).

```sql
CREATE INDEX ON metrics USING BRIN (recorded_at) WITH (pages_per_range = 32);
```

Pros: microscopic size; cheap to maintain. Cons: useless if the column isn't physically clustered.

### 7.7 Partial Indexes

Index only rows matching a condition.

```sql
-- Only index active rows (most queries only care about them)
CREATE INDEX ON orders (customer_id, created_at DESC) WHERE status IN ('PENDING','PAID');

-- Nullable foreign key, most rows NULL
CREATE INDEX ON invoices (customer_id) WHERE customer_id IS NOT NULL;

-- Unique constraint that applies only when active
CREATE UNIQUE INDEX ON users (email) WHERE deleted_at IS NULL;
```

Huge size wins on skewed data. The planner uses the partial index only when it can prove the query's WHERE implies the index's predicate.

### 7.8 Expression Indexes

Index on a function:

```sql
CREATE INDEX ON users ((lower(email)));
-- now usable:
SELECT * FROM users WHERE lower(email) = lower(?);
```

### 7.9 Composite Indexes: Real Examples

```sql
-- Query: WHERE customer_id=? AND status IN ('A','B') ORDER BY created_at DESC LIMIT 20
CREATE INDEX ON orders (customer_id, status, created_at DESC);

-- Query: range on date + equality on tenant
CREATE INDEX ON events (tenant_id, occurred_at);  -- equality first, range second

-- Query: exact-match on email (always low cardinality per email)
CREATE UNIQUE INDEX ON users (email);
```

### 7.10 Anti-Patterns

- Indexing every column "just in case" → write amplification, bloat, planner confusion
- Duplicating `(a)` when `(a, b)` exists — the composite covers the single-column prefix
- Indexing low-cardinality booleans alone (`status = 'X'`) — usually a partial index is better
- Relying on index for `LIKE '%foo%'` — needs trigram GIN
- `CREATE INDEX` on big tables without `CONCURRENTLY` → takes `ShareLock`, blocks writes for the whole build
- Indexing columns never used in predicates (check `pg_stat_user_indexes.idx_scan = 0`)
- Forgetting indexes on FK columns — causes slow cascades and `DELETE`s on parent

### 7.11 CONCURRENTLY

```sql
CREATE INDEX CONCURRENTLY idx_name ON t (col);
DROP INDEX CONCURRENTLY idx_name;
REINDEX INDEX CONCURRENTLY idx_name;   -- PG 12+
```

Doesn't block writes. Cost: slower, uses two passes, can leave an invalid index on error — always check `pg_index.indisvalid` after and drop/retry if invalid.

---

## 8. Transactions & Isolation Levels

### 8.1 The Anomalies

| Anomaly | Description |
|---|---|
| **Dirty read** | See uncommitted changes of another tx |
| **Non-repeatable read** | Same row re-read returns different value |
| **Phantom read** | Same WHERE returns different rows on re-read |
| **Write skew** | Two txs read overlapping data, each writes based on stale view |
| **Lost update** | Concurrent updates; one's change is overwritten silently |

### 8.2 PostgreSQL's Levels

Postgres has three usable levels (it treats READ UNCOMMITTED as READ COMMITTED).

| | Dirty | Non-repeatable | Phantom | Write skew |
|---|---|---|---|---|
| **Read Committed (default)** | no | yes | yes | yes |
| **Repeatable Read (Snapshot)** | no | no | no | yes |
| **Serializable (SSI)** | no | no | no | no |

Postgres's RR is **snapshot isolation**, stronger than the SQL standard (no phantoms).

### 8.3 Serializable Snapshot Isolation (SSI)

Postgres's SERIALIZABLE isn't 2PL. It runs transactions under snapshot isolation **and** tracks read/write dependencies ("predicate locks"). If it detects a dependency cycle that would produce a non-serializable outcome, it aborts one transaction with:

```
ERROR: could not serialize access due to read/write dependencies among transactions
```

Your app **must retry** these errors with the same logic. Design transactions to be idempotent or retry-safe.

### 8.4 When to Use Which

| Scenario | Level |
|---|---|
| Most OLTP | Read Committed |
| Reports that read many tables and must agree | Repeatable Read |
| Financial operations without explicit locking | Serializable (+ retry) |
| You're already using `FOR UPDATE` carefully | Read Committed is fine |

### 8.5 Write Skew — The Classic Example

Constraint: "at least one doctor on call."

```
T1: read "doctors on call" → 2
    UPDATE doctors SET on_call=false WHERE id=1;
T2: read "doctors on call" → 2 (same snapshot as T1? No, different tx, but snapshot semantics)
    UPDATE doctors SET on_call=false WHERE id=2;
Both commit → zero doctors on call. Constraint broken.
```

Read Committed or Repeatable Read both allow this. Serializable prevents it — one of them aborts.

Alternative without SERIALIZABLE: `SELECT ... FOR UPDATE` to lock rows you read and base decisions on.

### 8.6 Practical Rules

- Default to Read Committed.
- Keep transactions **short**. Never wait for network/user inside a tx.
- Make mutations **idempotent** (idempotency keys) so retries are safe.
- Be explicit about locking when correctness depends on it — don't rely on isolation level magic.

---

## 9. Locking & Concurrency

### 9.1 Lock Model

Postgres has **table-level** locks and **row-level** locks, plus advisory locks. Every statement acquires some lock. The beauty of MVCC: SELECT takes only the lightest table lock (`AccessShareLock`) and no row locks by default.

### 9.2 Table-Level Lock Modes

From weakest to strongest; each mode conflicts with specific others:

| Mode | Acquired by | Conflicts with |
|---|---|---|
| `ACCESS SHARE` | SELECT | ACCESS EXCLUSIVE |
| `ROW SHARE` | SELECT FOR UPDATE/SHARE | EXCLUSIVE, ACCESS EXCLUSIVE |
| `ROW EXCLUSIVE` | INSERT/UPDATE/DELETE | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |
| `SHARE UPDATE EXCLUSIVE` | VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY | Another of same mode, SHARE, … |
| `SHARE` | CREATE INDEX (non-concurrent) | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE |
| `SHARE ROW EXCLUSIVE` | CREATE TRIGGER, some ALTER TABLE | most others |
| `EXCLUSIVE` | REFRESH MATERIALIZED VIEW | most |
| `ACCESS EXCLUSIVE` | DROP TABLE, TRUNCATE, VACUUM FULL, most ALTER TABLE | everything |

**Key insight:** many `ALTER TABLE` variants need `ACCESS EXCLUSIVE` — they block all reads and writes. Plan schema changes carefully (§17).

### 9.3 Row-Level Locks

Stored in the tuple header (the `xmax` field + a bit) and, for shared locks, in `pg_multixact`. Modes:

| Mode | Acquired by | Conflicts |
|---|---|---|
| `FOR KEY SHARE` | Weakest; protects PK | Only with FOR UPDATE |
| `FOR SHARE` | SELECT FOR SHARE | FOR UPDATE, FOR NO KEY UPDATE |
| `FOR NO KEY UPDATE` | UPDATE of non-key columns | FOR UPDATE, FOR NO KEY UPDATE, FOR SHARE |
| `FOR UPDATE` | SELECT FOR UPDATE, DELETE, UPDATE of key | All row modes |

### 9.4 `FOR UPDATE SKIP LOCKED` — The SQL Job Queue

```sql
-- Worker picks the next N available jobs without waiting on busy ones.
SELECT id, payload FROM jobs
WHERE status = 'READY'
ORDER BY scheduled_at
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

This single feature replaces RabbitMQ for many use cases when you already have Postgres.

### 9.5 Advisory Locks

Application-level mutexes, stored in shared memory:

```sql
SELECT pg_advisory_xact_lock(12345);  -- held till tx end
-- critical section
```

- Transactional variant (`_xact_`) releases on commit/rollback — the safe choice.
- Two forms: single 64-bit key or (int, int) pair.
- Great for "only one instance at a time" jobs across a cluster.

### 9.6 How Locks Work Internally

- Shared lock table in shared memory. Each entry: `(locktag, mode, pid, granted)`.
- Heavy-contention fast path avoids the shared table for common modes.
- Row locks live in tuple headers, not the shared table — avoids a lock row per update.

### 9.7 Deadlocks

Two sessions hold locks that the other wants:

```
T1: UPDATE accounts WHERE id=1;   -- locks row 1
T2: UPDATE accounts WHERE id=2;   -- locks row 2
T1: UPDATE accounts WHERE id=2;   -- waits on T2
T2: UPDATE accounts WHERE id=1;   -- waits on T1 → cycle
```

Postgres detects cycles after `deadlock_timeout` (default 1s) and aborts one transaction:

```
ERROR: deadlock detected
DETAIL: Process 1234 waits for ShareLock on transaction 5001; blocked by 1235.
```

**Prevention:**
- Always acquire locks **in a consistent order** (e.g., by primary key ascending)
- Keep transactions short; no network I/O inside
- Use `SKIP LOCKED` semantics when applicable
- Take row locks with a single `SELECT ... FOR UPDATE` covering all rows you'll touch

**Debugging:**
- `log_lock_waits = on` — logs waits that exceed deadlock_timeout
- `deadlock_timeout = 1s` — keep it as-is; too low = noisy; too high = slow detection
- Inspect the app log: both conflicting queries appear in the error detail

### 9.8 Detecting Lock Contention

```sql
-- Who's blocking whom right now
SELECT blocked.pid AS blocked_pid,
       blocked_activity.query AS blocked_query,
       blocking.pid AS blocking_pid,
       blocking_activity.query AS blocking_query,
       blocking_activity.state AS blocking_state,
       now() - blocked_activity.xact_start AS blocked_tx_age
FROM pg_stat_activity AS blocked_activity
JOIN pg_stat_activity AS blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked_activity.pid))
JOIN pg_stat_activity AS blocking_activity
  ON blocking_activity.pid = blocking.pid
JOIN pg_stat_activity AS blocked
  ON blocked.pid = blocked_activity.pid;
```

---

## 10. Performance Tuning

### 10.1 The Tuning Workflow

1. **Measure** — `pg_stat_statements`, logs, APM. Don't tune on hunches.
2. **Categorize** — is it CPU, I/O, lock wait, or plan quality?
3. **Fix the worst offender** — usually 1–2 queries account for most pain.
4. **Verify** — re-measure. Plans change under load.
5. **Then tune memory/config.** Parameters matter less than queries.

### 10.2 Query Optimization Habits

1. Always `EXPLAIN (ANALYZE, BUFFERS)` before changing anything.
2. Keep the query the same; change only indexes until you understand the plan.
3. Project only needed columns (`SELECT id, total` not `SELECT *`).
4. Replace `OFFSET` pagination with keyset when offset grows:
   ```sql
   WHERE id > :last_id ORDER BY id LIMIT 20;
   ```
5. Replace correlated subqueries with joins or `LATERAL`.
6. For "top-N per group":
   ```sql
   SELECT * FROM customers c
   JOIN LATERAL (
     SELECT * FROM orders WHERE customer_id = c.id ORDER BY created_at DESC LIMIT 5
   ) recent ON true;
   ```
7. Use `WITH ... AS MATERIALIZED` / `NOT MATERIALIZED` to control CTE fencing (PG12+ CTEs are inlined by default — good).
8. Avoid implicit casts: `WHERE varchar_col = 123` casts the column → no index use.
9. Replace `count(*)` with cached counts or estimates when exact isn't required.

### 10.3 Index Tuning

- Find missing indexes: look for Seq Scan with high "Rows Removed by Filter" on large tables.
- Find unused indexes:
  ```sql
  SELECT schemaname, relname, indexrelname, idx_scan,
         pg_size_pretty(pg_relation_size(indexrelid)) size
  FROM pg_stat_user_indexes
  WHERE idx_scan = 0 AND indexrelname NOT LIKE '%_pkey';
  ```
- Detect redundant indexes: if `(a, b)` exists, `(a)` alone is usually redundant.
- Fix bloated indexes: `REINDEX INDEX CONCURRENTLY`.

### 10.4 Memory Tuning

| Parameter | Starting point |
|---|---|
| `shared_buffers` | 25% of RAM |
| `effective_cache_size` | 50–75% of RAM |
| `work_mem` | 16–64 MB; raise per-session for heavy reports |
| `maintenance_work_mem` | 512 MB – 2 GB |
| `wal_buffers` | -1 (auto) |

For big reporting queries:
```sql
SET LOCAL work_mem = '512MB';
```

### 10.5 Connection Tuning

- Keep `max_connections` modest (100–300).
- Pool aggressively: HikariCP (in JVM) + PgBouncer (outside) for many clients.
- Set `idle_in_transaction_session_timeout = '30s'` — kill apps that "BEGIN, then go idle."
- Set `statement_timeout` per role/app — protects from runaway queries.
- `lock_timeout = '2s'` for migrations to fail fast instead of blocking.

### 10.6 Disk / I/O Tuning

- SSD: `random_page_cost = 1.1`, `effective_io_concurrency = 200` (for RAID/SSD).
- `checkpoint_timeout = 15min`, `max_wal_size = 8GB`, `checkpoint_completion_target = 0.9` — smooth out I/O.
- `wal_compression = on`.
- Put `pg_wal` on a separate fast volume if your workload is WAL-heavy.

### 10.7 Autovacuum Tuning (Summary)

Defaults are for small databases. For any serious workload:

```
autovacuum_max_workers = 5
autovacuum_naptime = 10s
autovacuum_vacuum_cost_limit = 2000
autovacuum_vacuum_cost_delay = 2ms
```

Per hot table: lower `vacuum_scale_factor` to 0.02–0.05.

---

## 11. Partitioning & Scaling

### 11.1 When to Partition

Consider partitioning when:
- A table exceeds ~50 GB **and** queries naturally filter by a common key (time, tenant).
- You need to **delete old data** cheaply (drop partition vs delete millions of rows).
- Vacuum of the whole table takes too long.

Don't partition when:
- Table is small (<10 GB) — overhead > benefit.
- No natural partition key matches your queries — queries touch all partitions = worse.

### 11.2 Native Declarative Partitioning (PG 10+)

```sql
CREATE TABLE events (
    id          BIGSERIAL,
    tenant_id   BIGINT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL,
    payload     JSONB,
    PRIMARY KEY (id, occurred_at)     -- partition key must be in PK
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2025_01 PARTITION OF events
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE events_2025_02 PARTITION OF events
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- indexes defined on parent propagate to partitions
CREATE INDEX ON events (tenant_id, occurred_at DESC);
```

Strategies:
- **RANGE** — most common; time-series, append-heavy.
- **LIST** — categorical (country, tenant bucket).
- **HASH** — even spread; no natural range.

### 11.3 Partition Pruning

When the planner can prove a query touches only some partitions, it skips the rest:

```sql
-- Prunes to events_2025_01 only
SELECT * FROM events WHERE occurred_at >= '2025-01-10' AND occurred_at < '2025-01-20';
```

Pruning happens at:
- **Plan time** — from literal predicates
- **Execution time** — from parameters (PG11+)

Verify with `EXPLAIN`: only relevant partitions appear.

### 11.4 Operational Patterns

- Automate partition creation (`pg_partman` extension, or a scheduled job).
- Pre-create future partitions a week/month ahead — inserts into non-existent partitions fail otherwise.
- Drop old partitions: `DROP TABLE events_2023_01` — instant, frees space immediately.
- Detach before drop to avoid locks: `ALTER TABLE events DETACH PARTITION events_2023_01 CONCURRENTLY;`
- Keep partition count sensible (tens to hundreds; PG 14+ handles thousands).

### 11.5 Sharding (Conceptual)

Horizontal split **across Postgres instances**. Each shard is independent; app or middleware routes queries.

- **App-level sharding** — app computes `shard = hash(tenant_id) % N`, connects to the right DB.
- **Citus** — Postgres extension that makes a cluster of nodes look like one database (distributed tables, reference tables, distributed SQL).
- **Cloud-native distributed SQL** — YugabyteDB, CockroachDB (PostgreSQL-compatible wire protocol, different storage, different consistency).

Sharding gives you write scalability beyond a single node but imposes real costs:
- Cross-shard joins: slow or forbidden.
- Distributed transactions: complex; usually replaced by sagas.
- Rebalancing when shards grow: painful without good tooling.
- Schema migrations across shards: must be coordinated.

**Shard only after you've exhausted partitioning, read replicas, archiving, and query tuning.**

---

## 12. Replication & High Availability

### 12.1 Streaming Replication (Physical)

Primary ships WAL to replicas; replicas replay it into identical data files. Binary-identical clones.

```
[Primary]
  writes → local WAL → walsender → network → walreceiver → replay → [Replica data files]
```

Modes:
- **Asynchronous** (default) — primary doesn't wait. Low latency; small risk of losing recent commits on failover.
- **Synchronous** — primary waits for at least one replica to confirm WAL flush before commit returns. Zero data loss on failover; higher commit latency.
- **Quorum sync** — wait for N of M replicas (PG10+).

Configuration (primary):
```
wal_level = replica
max_wal_senders = 10
synchronous_standby_names = 'ANY 1 (replica1, replica2)'
```

Replica's `recovery.conf` / `standby.signal` + `primary_conninfo`.

### 12.2 Replication Slots

A slot is a registered consumer of WAL. The primary refuses to recycle WAL past the slot's position. This protects the replica but **can fill the disk** if the replica disappears.

- Always set `max_slot_wal_keep_size` (PG13+) to cap the risk.
- Clean up abandoned slots promptly.

### 12.3 Read Replicas

- Hot standby: a physical replica that accepts read-only queries.
- Use for read scaling, reporting, analytics.
- Replay **can pause** if a query on the replica conflicts with applied WAL (`max_standby_streaming_delay`). Options:
  - Cancel replica queries (default, small delay)
  - Let replay wait (replica falls behind)
  - Use `hot_standby_feedback = on` — replica tells primary "don't vacuum tuples I still need" (prevents cancels but increases primary bloat)

### 12.4 Logical Replication

Replicates **row changes** (not WAL binary), via publications/subscriptions. Can replicate:
- A subset of tables
- Across major versions (great for upgrades)
- To non-Postgres consumers via logical decoding (Debezium → Kafka)

```sql
-- Publisher
CREATE PUBLICATION app_pub FOR TABLE orders, customers;

-- Subscriber
CREATE SUBSCRIPTION app_sub
  CONNECTION 'host=primary dbname=app user=repl'
  PUBLICATION app_pub;
```

Limitations: DDL not replicated; primary keys required on large tables; can't replicate sequences.

### 12.5 Failover

When the primary dies, one replica is promoted. Needs a management layer — Postgres doesn't do it alone:

- **Patroni** — the de facto standard. Uses etcd/Consul/ZooKeeper for leader election.
- **repmgr** — simpler, older.
- **Managed** — RDS / Aurora / Cloud SQL handle failover themselves.

Concerns on failover:
- **Data divergence** — async replica might miss the last few commits. Embrace it or use synchronous.
- **Split-brain** — two primaries thinking they're live. Fencing (STONITH) prevents it.
- **Connection draining** — clients need to reconnect; use a virtual IP, DNS, or HAProxy/pgpool.

### 12.6 Trade-offs

| Approach | Consistency | Latency | Ops cost | Fits |
|---|---|---|---|---|
| Async streaming | Eventual | Low | Low | Most apps |
| Sync streaming | Zero data loss | Higher commit latency | Medium | Financial |
| Logical replication | Selective | Async | Medium | Upgrades, CDC |
| Multi-master (BDR/Citus-distributed) | Conflict resolution | Variable | High | Global apps |

---

## 13. Backup & Recovery

Backups are worthless if you've never restored them. **Test your recovery regularly.**

### 13.1 pg_dump — Logical Backup

Produces SQL or custom-format dumps.

```bash
pg_dump -Fc -d app -f app.dump              # custom format, compressed
pg_restore -d app_restored app.dump         # restore
pg_dump -d app -t orders -f orders.sql       # one table
```

Pros: portable across PG versions and architectures; selective (per table/schema).
Cons: slow on huge DBs; no PITR; consistent snapshot only at dump start (uses a transaction).

**Use for:** small to medium databases, dev refreshes, migrating between servers.

### 13.2 Base Backup — Physical Backup

Copies the entire data directory while the cluster is running.

```bash
pg_basebackup -D /backup/base -Fp -Xs -P
```

Pros: fast for large DBs; required for streaming replication setup; required for PITR.
Cons: binary — specific to Postgres version and architecture; no selective restore.

### 13.3 PITR — Point-in-Time Recovery

The gold standard. Combines:
1. A recent base backup
2. Archived WAL files since the backup

Restore to any target time:

```bash
# Restore base backup
# Set recovery target
echo "restore_command = 'cp /wal_archive/%f %p'" >> postgresql.conf
echo "recovery_target_time = '2026-04-22 09:15:00+00'" >> postgresql.conf
touch recovery.signal
# Start Postgres; it replays WAL up to the target
```

Enable by setting `archive_mode = on`, `archive_command` (cp / S3 push / pgBackRest).

### 13.4 Tools

- **pgBackRest** — the leading production backup tool. Parallel, compressed, incremental, encrypted, S3 support.
- **Barman** — similar category.
- **WAL-G** — cloud-native, simpler for S3.
- Managed services (RDS, Cloud SQL) — handle all of this for you; still *test* recovery.

### 13.5 Recovery Strategies

Define your RPO and RTO before choosing:

- **RPO (Recovery Point Objective)** — how much data can you afford to lose? 0 seconds? 15 minutes? 24 hours?
- **RTO (Recovery Time Objective)** — how quickly must you be back up? 1 min? 1 hour?

Rough matching:
- RPO 24h, RTO hours → nightly `pg_dump`.
- RPO minutes, RTO minutes → PITR with archived WAL.
- RPO ~0, RTO seconds → synchronous replication + managed failover.

### 13.6 Backup Rules

- 3-2-1: three copies, two media, one offsite.
- Encrypt backups.
- **Test restoration monthly.** Untested backups are fiction.
- Keep at least one backup older than your longest "oh no" detection window (e.g., 30 days).

---

## 14. Monitoring & Debugging

### 14.1 The Essential Views

#### `pg_stat_activity` — who's doing what right now

```sql
SELECT pid, usename, application_name, state,
       now() - xact_start AS tx_age,
       now() - query_start AS query_age,
       wait_event_type, wait_event,
       query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;
```

Columns to know:
- `state` — `active`, `idle`, `idle in transaction` (**this one kills you** — it holds snapshots and locks)
- `wait_event_type` / `wait_event` — what it's waiting on (Lock, IO, Client, LWLock…)
- `backend_xmin` — the oldest row version this connection still needs. If it's way behind, autovacuum is blocked.
- `query` — current (or most recent) statement

#### `pg_stat_statements` — which queries hurt most

Enable:
```
shared_preload_libraries = 'pg_stat_statements'
-- then restart, then:
CREATE EXTENSION pg_stat_statements;
```

Query:
```sql
SELECT queryid, calls, total_exec_time/1000 AS total_s,
       mean_exec_time AS mean_ms,
       rows,
       100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_ratio
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

This is your starting point whenever someone says "the DB is slow."

#### Other high-value views

- `pg_stat_user_tables` — per-table stats: scans, tuples, vacuum times, dead rows.
- `pg_stat_user_indexes` — per-index usage.
- `pg_stat_io` (PG 16+) — per-backend-type I/O counters (replaces a lot of legacy metrics).
- `pg_stat_database` — per-DB aggregates.
- `pg_stat_replication` — replica lag.
- `pg_locks` — all current locks.
- `pg_stat_progress_vacuum` / `_cluster` / `_create_index` — live progress for long ops.

### 14.2 Logs

Essential log settings:

```
log_min_duration_statement = 500        # log queries > 500ms
log_lock_waits = on
log_temp_files = 10MB
log_checkpoints = on
log_autovacuum_min_duration = 0         # log all autovacuum runs
log_connections = on
log_disconnections = on
log_line_prefix = '%m [%p] %u@%d app=%a '
```

Analyze with **pgBadger** to produce reports (top queries, error distribution, checkpoint stats).

### 14.3 Debugging Slow Queries

1. Find it in `pg_stat_statements` (by `total_exec_time` or `mean_exec_time`).
2. Get a concrete parameter set (from logs or APM).
3. `EXPLAIN (ANALYZE, BUFFERS)` on the live data.
4. Identify the bottleneck node: seq scan, bad rows estimate, spill to disk, too many loops.
5. Fix: index, rewrite, add stats, raise work_mem.
6. Re-run ANALYZE; verify new plan.
7. Deploy; verify in `pg_stat_statements` that mean time dropped.

### 14.4 Debugging Lock Contention

1. See §9.8 blocking query.
2. Identify the blocker — is it a long `idle in transaction`?
3. Terminate it: `SELECT pg_terminate_backend(pid);` (only after you understand what it's doing).
4. Root-cause the app: missing commit, long network call inside tx, wrong isolation.

### 14.5 Debugging High CPU

- `pg_stat_statements` → top queries by `total_exec_time`.
- Many short queries? Connection storm or poor caching — look at app-side.
- Few big queries? Optimize them (indexes, rewrites, parallelism).
- `pg_stat_activity.state = 'active'` count — is CPU spent or waiting?
- `top` / `perf` on the DB host to confirm it's Postgres burning CPU.

### 14.6 Debugging High I/O

- `pg_stat_io` (PG16+) by backend type.
- Checkpoints? `log_checkpoints` output and tune.
- Autovacuum? `pg_stat_progress_vacuum`.
- Cold queries? `Buffers: read` dominating over `hit` in EXPLAIN — too-small `shared_buffers` or first-time warmup.

### 14.7 Observability Stack (Production)

- **Prometheus + `postgres_exporter`** — metrics
- **Grafana dashboards** — visualization
- **pgBadger** — log reports
- **pganalyze / Datadog DB Monitoring / RDS Performance Insights** — commercial deep insight
- **Auto-explain** — slow plan logs

---

## 15. Advanced PostgreSQL Features

### 15.1 JSONB

Binary JSON. Indexable, efficient to query.

```sql
CREATE TABLE events (id BIGSERIAL PRIMARY KEY, data JSONB);
INSERT INTO events (data) VALUES ('{"type":"order","total":99,"tags":["priority"]}');

-- Access
SELECT data->>'type', (data->>'total')::numeric FROM events;
SELECT data #>> '{customer,address,city}' FROM events;

-- Containment / existence
SELECT * FROM events WHERE data @> '{"type":"order"}';
SELECT * FROM events WHERE data ? 'promoCode';

-- Indexing
CREATE INDEX ON events USING GIN (data jsonb_path_ops);          -- smaller, containment-only
CREATE INDEX ON events USING GIN (data);                          -- full ops
CREATE INDEX ON events ((data->>'type'));                         -- expression index for equality
```

**When to use:** flexible-schema payloads, API blobs, event data, config.
**When NOT:** structured data that maps cleanly to columns — use real columns (faster, constraints, smaller).

### 15.2 Full-Text Search

```sql
ALTER TABLE articles ADD COLUMN tsv tsvector
  GENERATED ALWAYS AS (to_tsvector('english', title || ' ' || body)) STORED;

CREATE INDEX ON articles USING GIN (tsv);

SELECT id, ts_rank(tsv, q) AS rank, ts_headline('english', body, q) AS snippet
FROM articles, plainto_tsquery('english', 'postgres deep dive') q
WHERE tsv @@ q
ORDER BY rank DESC LIMIT 20;
```

For fuzzy/typo matches, add `pg_trgm`:

```sql
CREATE EXTENSION pg_trgm;
CREATE INDEX ON products USING GIN (name gin_trgm_ops);

SELECT * FROM products WHERE name ILIKE '%widgt%';               -- works
SELECT * FROM products ORDER BY similarity(name, 'widgt') DESC;   -- ranking
```

For more complex search needs (faceting, typo-tolerance at scale, multi-field scoring), Elasticsearch / OpenSearch still win — but for many apps, Postgres FTS is enough.

### 15.3 Window Functions

Compute per-row aggregates without collapsing rows.

```sql
-- Rank each order within its customer's history
SELECT id, customer_id, total,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at) AS n,
       RANK()       OVER (PARTITION BY customer_id ORDER BY total DESC) AS spend_rank
FROM orders;

-- Running totals
SELECT id, total,
       SUM(total) OVER (PARTITION BY customer_id ORDER BY created_at
                        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running
FROM orders;

-- 7-day moving average
SELECT day, amount,
       avg(amount) OVER (ORDER BY day ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM daily_sales;
```

### 15.4 CTEs (Common Table Expressions)

Named subqueries. In PG 12+ they're **inlined by default** (no optimization fence). Use `MATERIALIZED` to force materialization when you want it.

```sql
WITH recent AS (
  SELECT id, customer_id, total
  FROM orders WHERE created_at > now() - interval '7 days'
),
top_customers AS (
  SELECT customer_id, sum(total) AS spend
  FROM recent GROUP BY customer_id ORDER BY spend DESC LIMIT 100
)
SELECT c.name, t.spend
FROM top_customers t JOIN customers c ON c.id = t.customer_id;
```

Recursive CTE — hierarchical data:

```sql
WITH RECURSIVE tree AS (
  SELECT id, parent_id, name, 1 AS depth FROM categories WHERE parent_id IS NULL
  UNION ALL
  SELECT c.id, c.parent_id, c.name, t.depth + 1
  FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree ORDER BY depth, name;
```

### 15.5 Other Worth-Knowing

- **`DISTINCT ON`** — fastest "one row per group" syntax; paired with `ORDER BY`.
- **`INSERT ... ON CONFLICT`** — upserts.
- **`RETURNING`** — get generated keys / updated rows back in the same statement.
- **Generated columns** (`GENERATED ALWAYS AS (...) STORED`) — derived columns, no triggers.
- **Materialized views** — precomputed queries; refresh on demand.
- **LISTEN/NOTIFY** — pub/sub from DB to app connections (small messages, not a Kafka replacement).
- **Range types + exclusion constraints** — enforce "no overlapping bookings" at the DB level.
- **Foreign Data Wrappers (FDW)** — query remote Postgres, MySQL, file, or API as if it were a table.

---

## 16. Real-World Debugging Scenarios

### 16.1 Scenario — Slow Query

**Symptom:** user-facing endpoint timing out; `pg_stat_statements` shows one query with `mean_exec_time = 4500ms`, 50k calls/day.

**Query:**
```sql
SELECT * FROM orders
WHERE customer_email = ? AND status = 'PAID'
ORDER BY created_at DESC LIMIT 20;
```

**Investigation:**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE customer_email = 'x@y.com' AND status='PAID'
ORDER BY created_at DESC LIMIT 20;
```
```
Limit  (actual time=4490..4491 rows=20 loops=1)
  ->  Sort  (rows=15830)
       Sort Method: top-N heapsort
       ->  Seq Scan on orders  (rows=15830, removed by filter: 9,984,170)
            Filter: (customer_email='x@y.com' AND status='PAID')
 Buffers: shared read=148000
```

**Diagnosis:** Seq Scan on 10M rows; no usable index.

**Fix:**
```sql
CREATE INDEX CONCURRENTLY idx_orders_email_status_created
  ON orders (customer_email, status, created_at DESC);
ANALYZE orders;
```

**Result:** plan flips to Index Scan, exec time 2 ms. `pg_stat_statements` confirms after a reset.

**Lesson:** composite index order = equality predicates first, then the ordering column.

---

### 16.2 Scenario — Deadlock

**Symptom:** sporadic app errors `deadlock detected`; financial transfer endpoint.

**Logs:**
```
ERROR: deadlock detected
DETAIL: Process 1001 waits for ShareLock on transaction 5002; blocked by 1002.
        Process 1002 waits for ShareLock on transaction 5001; blocked by 1001.
Statement: UPDATE accounts SET balance = balance - 100 WHERE id = 7;
```

**Code:**
```java
void transfer(Long from, Long to, BigDecimal amount) {
  accountRepo.debit(from, amount);
  accountRepo.credit(to, amount);
}
```

**Diagnosis:** two concurrent transfers with opposite direction (A→B and B→A) lock rows in opposite order.

**Fix:** lock in consistent order (smallest id first):

```java
@Transactional
void transfer(Long from, Long to, BigDecimal amount) {
  Long first = Math.min(from, to);
  Long second = Math.max(from, to);
  accountRepo.lockById(first);
  accountRepo.lockById(second);
  accountRepo.debit(from, amount);
  accountRepo.credit(to, amount);
}
```

**Lesson:** deadlocks are about ordering. Always acquire locks in a deterministic order.

---

### 16.3 Scenario — High CPU

**Symptom:** DB CPU at 95% at peak; replicas lag.

**Investigation:**
```sql
SELECT queryid, calls, mean_exec_time, total_exec_time/1000 AS total_s
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

Top: `SELECT count(*) FROM products WHERE category_id = $1` — 2M calls/day, 60ms each. 33h of CPU/day.

**Diagnosis:** Widgets dashboard polls category counts on every page view. Count is expensive; MVCC can't fast-path it.

**Fix (layered):**
1. Cache at app level (Redis, 30s TTL).
2. Precompute via trigger-maintained table or materialized view:
   ```sql
   CREATE MATERIALIZED VIEW category_counts AS
   SELECT category_id, count(*) AS n FROM products GROUP BY category_id;
   CREATE UNIQUE INDEX ON category_counts (category_id);
   REFRESH MATERIALIZED VIEW CONCURRENTLY category_counts;
   ```
3. Expose an approximate count where exact isn't needed.

**Result:** DB CPU drops to 40%.

**Lesson:** when a query runs often enough, making it cheaper beats making the DB bigger.

---

### 16.4 Scenario — Table Bloat

**Symptom:** `orders` table on disk is 180 GB; live rows ~30 GB worth. Queries slowing.

**Investigation:**
```sql
SELECT relname, n_live_tup, n_dead_tup,
       pg_size_pretty(pg_total_relation_size(relid)) AS total,
       last_autovacuum
FROM pg_stat_user_tables WHERE relname='orders';
```
→ 500M dead tuples; last autovacuum 3 weeks ago.

**Why autovacuum stuck?**
```sql
SELECT pid, backend_xmin, now() - xact_start AS tx_age, state, query
FROM pg_stat_activity
WHERE backend_xmin IS NOT NULL
ORDER BY backend_xmin LIMIT 5;
```
→ one connection from a reporting job has been in `idle in transaction` for 18 days. `backend_xmin` is pinning every dead tuple.

**Fix:**
1. Terminate the culprit: `SELECT pg_terminate_backend(pid);`
2. Investigate the code: reports were opening a tx and never committing when the worker crashed.
3. Add `idle_in_transaction_session_timeout = '30s'`.
4. Force autovacuum: `VACUUM (VERBOSE, ANALYZE) orders;`
5. Shrink: `pg_repack` during a maintenance window (online; no exclusive lock).
6. Tune per-table autovacuum so it doesn't happen again.

**Lesson:** long-running / `idle in transaction` sessions are the #1 cause of Postgres bloat incidents.

---

### 16.5 Scenario — Connection Exhaustion

**Symptom:** `FATAL: sorry, too many clients already`; app 500s.

**Investigation:**
```sql
SELECT state, count(*) FROM pg_stat_activity GROUP BY state;
```
→ 300 connections, `max_connections=300`. 210 `idle`, 50 `idle in transaction`.

**Diagnosis:** app has 15 pods × pool size 20 = 300 max, no headroom; several pods have leaked transactions.

**Fix:**
1. Immediate: terminate idle-in-tx, raise `max_connections` temporarily.
2. Mid-term: put PgBouncer in front (transaction pooling) so pod count decouples from DB connections.
3. Reduce pod pool size to 5; DB sees far fewer connections.
4. Enable `idle_in_transaction_session_timeout`.
5. Add `HikariCP.leakDetectionThreshold` in the app.

**Lesson:** connection counts scale with pods × pool size, not with load. Pool correctly.

---

## 17. Production Best Practices

### 17.1 Schema Design

- Use `BIGINT` / `BIGSERIAL` for surrogate keys. `INT` overflow is a real incident.
- Prefer UUIDv7 (time-ordered) over UUIDv4 if you want UUIDs without index locality disaster.
- `TIMESTAMPTZ` always; never `TIMESTAMP WITHOUT TIME ZONE` for events.
- Constraints **in the DB**: `NOT NULL`, `CHECK`, `FOREIGN KEY`, `UNIQUE`. App code lies; constraints don't.
- Name things consistently: `orders_pkey`, `orders_customer_id_fkey`, `orders_status_idx`.
- Use Postgres types where they help: `INET`, `CIDR`, `UUID`, `ENUM`, `JSONB`, range types.
- Avoid `varchar(n)` unless you mean the cap; just use `TEXT`.
- Soft deletes with `deleted_at TIMESTAMPTZ`; add partial unique indexes `WHERE deleted_at IS NULL`.

### 17.2 Indexing Strategy

- One PK per table, no exceptions.
- Index **every** FK column (Postgres doesn't do it automatically).
- Composite indexes following actual query patterns — observe queries, don't guess.
- Use partial indexes for skewed data.
- `CREATE INDEX CONCURRENTLY` for any table >100 MB.
- Review `pg_stat_user_indexes` quarterly; drop unused ones.

### 17.3 Safe Schema Migrations

Rolling deploy = old code + new code both run for a while. Migrations must be **backwards-compatible** through that window.

**Pattern: expand → migrate → contract**

Add a new column:
1. Migration A: add column nullable with default. Deploy.
2. Backfill in batches (or via background job).
3. Migration B: set NOT NULL. Deploy.

Rename a column:
1. Add new column, dual-write.
2. Backfill.
3. Switch readers.
4. Drop old column.

Never: rename in one migration. Old pods break instantly.

**Safe-by-default practices:**
- Set `lock_timeout` at the start of migration SQL:
  ```sql
  SET lock_timeout = '2s';
  ```
  If `ALTER TABLE` can't acquire the lock, fail fast rather than block the app.
- Use `ALTER TABLE ... ADD COLUMN x INT NOT NULL DEFAULT 0` — PG11+ does this without rewriting the table.
- For big indexes, always CONCURRENTLY.
- For `ALTER TABLE ... ADD CONSTRAINT ... NOT VALID` then `VALIDATE CONSTRAINT` separately — avoids long exclusive locks.

### 17.4 Avoiding Common Mistakes

- `SELECT *` in application code — breaks index-only scans, transfers noise.
- JPA `spring.jpa.open-in-view: true` — causes lazy loads outside transactions. **Always disable.**
- Letting `ddl-auto = update` near production.
- Trusting defaults (`shared_buffers=128MB`, `random_page_cost=4`, `autovacuum_scale_factor=0.2` on a hot table).
- No `statement_timeout` / `idle_in_transaction_session_timeout`.
- No monitoring on replication slots.
- Backups never tested.

### 17.5 Capacity Planning

Track growth of:
- Table sizes (per day)
- WAL generation rate (per day)
- Dead-tuple rate (per table per day)
- Lock wait events
- Connections

Pick thresholds with headroom: plan to scale at 70% utilization, not 100%. Cloud disk growth is easy; moving to bigger instances is a restart (managed services) or a migration.

---

## 18. When PostgreSQL Becomes a Bottleneck

Most "we're hitting Postgres limits" stories end with "we added an index." Before believing the DB is the bottleneck, verify.

### 18.1 Scale Vertically First

A single Postgres can handle:
- Tens of thousands of simple queries/sec
- Terabytes of data (well-partitioned)
- Thousands of concurrent clients (via PgBouncer)

Vertical scale (bigger instance) is the cheapest path. On AWS you can go from db.r6i.xlarge → db.r6i.32xlarge without touching a line of code.

Escalation ladder:

1. Tune the top queries (95% of cases)
2. Add indexes / fix plans
3. Bigger instance
4. Read replicas for read scaling
5. Archive / partition for storage
6. App-level caching (Caffeine / Redis)
7. Shard writes (last resort)

### 18.2 Signs You're Really at a Limit

- Write TPS sustained near the WAL throughput ceiling of your disk.
- CPU pinned at 100% with top queries already optimized.
- `pg_stat_activity` overwhelmed by lock waits on a single hot row (not fixable by indexes).
- Table > 1 TB with queries that must touch all of it.
- Replicas can't keep up even with fast network and modern hardware.

### 18.3 When to Shard

- A single primary can't absorb writes despite vertical scaling.
- Multi-tenant SaaS where tenant data is naturally isolated and you have thousands of big tenants.
- Global workloads that need regional write locality.

Approaches:
- **Citus** — distributed Postgres as an extension. Preserves SQL and transactions (with caveats).
- **App-level sharding** — route by tenant or shard key.
- **Distributed SQL** (Cockroach/Yugabyte) — Postgres-compatible, very different under the hood.

### 18.4 When to Introduce Caching

- Read:write ratio > 10:1
- The cached data tolerates staleness (seconds to minutes)
- Cache key cardinality is bounded (hit rate > 60%)
- Computation is expensive (joins, aggregations, third-party calls)

Don't cache:
- Data that must be fresh (inventory, ledgers)
- Queries already sub-millisecond
- Per-request unique results (no hits)

### 18.5 When to Add Another Store

- **Full-text search at scale / faceting** → Elasticsearch / OpenSearch
- **OLAP / huge analytics** → ClickHouse / BigQuery / Snowflake
- **Time series at massive scale** → TimescaleDB (stays in Postgres) or InfluxDB
- **Global low-latency KV** → Redis / DynamoDB
- **Graph traversal** → Neo4j (most graph use cases in Postgres are fine; consider Neo4j only when traversals dominate)

Stay in Postgres as long as reasonable. Every additional datastore doubles operational complexity.

---

## 19. Final Checklist

**You are a senior-level PostgreSQL engineer if you can:**

- [ ] Explain the process architecture and why `max_connections` matters at both RAM and contention levels
- [ ] Describe `shared_buffers`, `work_mem`, `maintenance_work_mem`, and reason about their multiplication
- [ ] Explain MVCC using `xmin`/`xmax`, predict what a concurrent reader sees, and spot the bloat implications
- [ ] Explain WAL, checkpoints, and how crash recovery works
- [ ] Describe why long-running transactions cause bloat across the whole database
- [ ] Tune autovacuum per-table for a hot write workload and avoid transaction ID wraparound
- [ ] Read `EXPLAIN (ANALYZE, BUFFERS)` fluently — identify stats errors, sorts on disk, bad nested loops
- [ ] Design a composite index given a query and justify column order and direction
- [ ] Use partial, expression, covering, and GIN/GiST indexes appropriately
- [ ] Pick the right isolation level for a workload and design retry logic for serialization failures
- [ ] Prevent and debug deadlocks from `pg_locks` / `pg_blocking_pids`
- [ ] Use `FOR UPDATE SKIP LOCKED` to build a SQL-based work queue
- [ ] Partition a 100M+ row table by range and explain partition pruning
- [ ] Set up streaming replication, understand replication slots, and design for failover
- [ ] Design a PITR backup strategy with defined RPO/RTO and actually restore it
- [ ] Use `pg_stat_activity`, `pg_stat_statements`, `pg_locks`, and `pg_stat_progress_*` to debug production
- [ ] Use JSONB with GIN indexes and know when plain columns are better
- [ ] Write window functions and recursive CTEs without pausing
- [ ] Diagnose slow queries, deadlocks, high CPU, bloat, and connection exhaustion with a clear playbook
- [ ] Run a zero-downtime schema migration on a multi-pod production deploy
- [ ] Recognize when a "scaling problem" is really a missing index, a leaked transaction, or a bad plan
- [ ] Decide, with evidence, between vertical scaling, read replicas, partitioning, caching, and sharding

If you can tick every box, you can hold the pager for a PostgreSQL-backed production system — and when the pager goes off, you'll know *where to look first*, *why*, and *what to change*.
