# Databases in Java + Spring Boot + PostgreSQL — Production-Grade Learning Guide

> A senior-engineer's reference for designing, optimizing, and operating relational data systems in real production.
>
> **Audience:** Java backend engineers with ~3 years of Spring Boot + JPA experience who want to level up from "I can write entities" to "I can own a PostgreSQL system in production."

---

## Table of Contents

1. [Database Fundamentals](#1-database-fundamentals)
2. [PostgreSQL Deep Dive](#2-postgresql-deep-dive)
3. [SQL Performance & Query Optimization](#3-sql-performance--query-optimization)
4. [Connection Management](#4-connection-management)
5. [Data Access Strategies in Java](#5-data-access-strategies-in-java)
6. [Schema Management](#6-schema-management)
7. [Mapping & DTOs](#7-mapping--dtos)
8. [Transactions & Concurrency](#8-transactions--concurrency)
9. [Designing Reliable Systems](#9-designing-reliable-systems)
10. [Scaling the Database](#10-scaling-the-database)
11. [Caching Strategies](#11-caching-strategies)
12. [Database Tools](#12-database-tools)
13. [Performance & Tuning](#13-performance--tuning)
14. [Spring Boot Integration](#14-spring-boot-integration)
15. [Real-World Scenarios](#15-real-world-scenarios)
16. [Comparisons & Trade-offs](#16-comparisons--trade-offs)
17. [When NOT to Use Certain Approaches](#17-when-not-to-use-certain-approaches)
18. [Final Checklist](#18-final-checklist)

---

## 1. Database Fundamentals

### 1.1 How a Relational Database Works Internally

A relational database is, at its core, four layers:

1. **Parser / Planner / Optimizer** — turns SQL into an execution plan
2. **Executor** — runs the plan: scans, joins, filters, aggregates
3. **Storage engine** — reads/writes **pages** (8 KB in Postgres) to/from disk
4. **Transaction manager** — enforces ACID across concurrent sessions via locks, MVCC, and WAL

Data lives in **pages** on disk; the database keeps a **buffer pool** (Postgres: `shared_buffers`) in RAM to avoid disk I/O. Reading from RAM is ~100 ns; reading from SSD is ~100 µs; reading from a remote disk can be milliseconds. All performance work is fundamentally about keeping the hot working set in RAM.

### 1.2 ACID

| Property | Meaning | How Postgres enforces it |
|---|---|---|
| **A**tomicity | All-or-nothing per transaction | WAL + transaction IDs; rollback discards in-memory changes and WAL entries |
| **C**onsistency | DB moves from one valid state to another | Constraints (PK, FK, CHECK, NOT NULL), triggers |
| **I**solation | Concurrent txns don't corrupt each other | MVCC + locks + isolation levels |
| **D**urability | Committed data survives crashes | WAL flushed (fsync) to disk before COMMIT returns |

### 1.3 Transactions & Isolation Levels

SQL defines four standard isolation levels. Each prevents specific anomalies:

| Anomaly | RC | RR | Serializable |
|---|---|---|---|
| Dirty read | prevented | prevented | prevented |
| Non-repeatable read | possible | prevented | prevented |
| Phantom read | possible | possible (standard) — prevented in Postgres RR | prevented |
| Write skew / serialization anomaly | possible | possible | prevented |

**Postgres specifics:**
- Default is **READ COMMITTED**. Each statement sees a fresh snapshot.
- **REPEATABLE READ** in Postgres is *snapshot isolation* — stronger than the SQL standard; prevents phantoms but not write skew.
- **SERIALIZABLE** uses SSI (Serializable Snapshot Isolation) — detects conflicts and aborts one of the conflicting transactions.

**Rule of thumb:**
- READ COMMITTED for almost everything.
- REPEATABLE READ when a single transaction runs multiple reads that must agree (reports, consistent exports).
- SERIALIZABLE when correctness depends on there being no write skew and you prefer retries over manual locking.

### 1.4 Indexes (From First Principles)

An index is a **separate data structure that maps keys → row locations** so you don't have to scan the whole table.

- **B-tree** — balanced tree; the default. Supports equality, range, sort, prefix LIKE. >95% of your indexes.
- **Hash** — equality only. Rarely better than B-tree; usable in Postgres 10+.
- **GIN** — generalized inverted index. For multi-valued types: `jsonb`, `tsvector` (full-text), arrays.
- **GiST** — generalized search tree. For geometric types, ranges, nearest-neighbor.
- **BRIN** — block-range index. Tiny; useful for very large, naturally-ordered tables (e.g., append-only time series).
- **SP-GiST** — space-partitioned; niche (IP ranges, phone numbers).

A **primary key** is a unique B-tree index. A **foreign key is NOT an index** — Postgres does not auto-create an index on FK columns, a frequent cause of slow deletes/updates.

#### Composite indexes

```sql
CREATE INDEX ON orders (customer_id, created_at DESC);
```

- Usable for: `WHERE customer_id = ?`, `WHERE customer_id = ? AND created_at > ?`, `ORDER BY customer_id, created_at DESC`
- NOT usable efficiently for: `WHERE created_at > ?` alone (leftmost column rule)

#### Covering / Index-Only Scans

```sql
CREATE INDEX ON orders (customer_id) INCLUDE (total, status);
```

If all columns the query needs are in the index, Postgres can serve it **without touching the heap** — a huge win.

### 1.5 Query Execution Basics

When you send a query:

1. **Parse** — syntax check, build AST
2. **Rewrite** — apply views, rules
3. **Plan** — optimizer enumerates plans; chooses lowest **cost** based on statistics (`pg_statistic`, refreshed by `ANALYZE`)
4. **Execute** — runs the chosen plan

Three join algorithms, each optimal in different regimes:

| Join | Best when |
|---|---|
| **Nested Loop** | Outer relation is small; inner has an index |
| **Hash Join** | One side fits in work_mem; no useful index |
| **Merge Join** | Both inputs are already sorted on the join key |

Scan types:

| Scan | When |
|---|---|
| **Seq Scan** | Small table OR most rows match OR no usable index |
| **Index Scan** | Small fraction of rows; need heap columns not in index |
| **Index-Only Scan** | All needed columns in index; visibility map clean |
| **Bitmap Index Scan** | Medium selectivity; combines multiple indexes |

---

## 2. PostgreSQL Deep Dive

### 2.1 Architecture Overview

Postgres is **process-based**, not thread-based. Every connection spawns a dedicated OS process (a "backend"). That design is why connection pooling is not optional.

```
 ┌─────────────┐       ┌──────────────────────┐
 │ Client      │──────▶│ Postmaster (listener)│
 └─────────────┘       └───────────┬──────────┘
                                   │ fork()
                                   ▼
                         ┌─────────────────┐
                         │ Backend process │  ← one per connection
                         └─────────────────┘
                                   │
 Shared memory (shared_buffers, WAL buffers, locks, clog)
                                   │
       ┌───────────────┬───────────┼──────────────┬────────────┐
       ▼               ▼           ▼              ▼            ▼
  WAL writer       Bgwriter     Checkpointer   Autovacuum   Stats collector
```

### 2.2 MVCC — Multi-Version Concurrency Control

**The single most important Postgres concept.**

Postgres never overwrites a row in place. Every `UPDATE` creates a new row version; `DELETE` marks the old one dead. Each row carries two system columns:

- `xmin` — the transaction ID that *created* this version
- `xmax` — the transaction ID that *invalidated* this version (0 if still live)

A transaction sees only rows visible to its snapshot. Consequences:

- **Readers don't block writers; writers don't block readers.** No shared read locks.
- `UPDATE` is effectively `INSERT new version + mark old dead`.
- Dead rows accumulate. **VACUUM** removes them.
- `COUNT(*)` is slow — Postgres must check visibility per row. No magic cached count.
- Long-running transactions hold old snapshots → block vacuum → table bloat.

### 2.3 WAL — Write-Ahead Logging

The durability mechanism. Rule: **every change is written to WAL and fsynced before the data page is flushed.**

Flow:
1. Transaction changes a page in shared buffers.
2. Change appended to WAL buffer, then to `pg_wal/` files on disk.
3. `COMMIT` forces fsync of WAL up to this point.
4. Much later, a **checkpoint** flushes dirty data pages to their real heap files.
5. On crash recovery, WAL is replayed from the last checkpoint.

WAL is also:
- The basis of **streaming replication** (stream WAL to replicas)
- The basis of **PITR** (point-in-time recovery: base backup + WAL archive)
- The basis of **logical replication** (decoded WAL → row changes)

### 2.4 Vacuum & Autovacuum

Since MVCC leaves dead rows, **something must reclaim them.**

- `VACUUM` — marks dead space as reusable; does NOT usually return disk to the OS
- `VACUUM FULL` — rewrites table; returns space; takes **AccessExclusiveLock** — blocks everything
- `ANALYZE` — refreshes planner statistics
- `VACUUM (VERBOSE, ANALYZE)` — both

**Autovacuum** runs automatically based on thresholds:

```
autovacuum_vacuum_scale_factor = 0.2     -- vacuum when 20% of rows are dead
autovacuum_vacuum_threshold    = 50      -- plus this many rows
autovacuum_analyze_scale_factor= 0.1
```

For large, high-churn tables (millions of rows, thousands of updates/sec) the defaults are **too lax**. Tune per-table:

```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_analyze_scale_factor = 0.01,
  autovacuum_vacuum_cost_limit = 2000
);
```

**Bloat symptoms:** tables/indexes growing faster than logical row count; sequential scans slowing down; slow updates. Check with `pg_stat_user_tables.n_dead_tup` and `pgstattuple`.

### 2.5 Connection Handling

- Each connection = one OS process + ~10 MB baseline RAM
- `max_connections` default is 100 — realistic limit on small-to-medium hardware is a few hundred
- Idle connections still consume RAM and hold snapshots

Hence: **always pool**. See §4.

---

## 3. SQL Performance & Query Optimization

### 3.1 EXPLAIN vs EXPLAIN ANALYZE

- `EXPLAIN` — shows the *estimated* plan. Fast. Read-only.
- `EXPLAIN ANALYZE` — **actually runs the query** and shows real times + row counts. Read this before you trust anything.
- `EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)` — the golden command.

> ⚠️ `EXPLAIN ANALYZE` runs destructive statements too. For `UPDATE`/`DELETE`, wrap it in `BEGIN; EXPLAIN ANALYZE ...; ROLLBACK;`.

### 3.2 Reading a Plan

```
 Nested Loop  (cost=0.43..1234.56 rows=50 width=64)
              (actual time=0.12..2.34 rows=47 loops=1)
   Buffers: shared hit=214 read=3
   ->  Index Scan using idx_orders_customer on orders
         (cost=... rows=50) (actual ... rows=47 loops=1)
         Index Cond: (customer_id = 42)
   ->  Index Scan using customers_pkey on customers
         (cost=... rows=1) (actual ... rows=1 loops=47)
         Index Cond: (id = orders.customer_id)
 Planning Time: 0.4 ms
 Execution Time: 2.8 ms
```

Key signals:
- **`rows=` (estimated) vs `actual rows=`** — if they diverge by 10x+, statistics are stale → `ANALYZE` the table.
- **`Buffers: shared hit=X read=Y`** — `read` means miss (disk). High `read` = cold cache or poor index.
- **`Seq Scan` on a large table with a WHERE clause** — missing or unusable index.
- **`loops=N` on inner side of nested loop** — N index lookups; fine if small, disaster if large.

### 3.3 Seq Scan vs Index Scan

A sequential scan is **not always bad**. For small tables (< a few thousand rows) or queries returning >10–20% of rows, it beats an index scan because:
- Sequential I/O is cheap
- Index scan randomly jumps between index and heap
- Postgres estimates this; tunes via `random_page_cost` (lower on SSD — set to 1.1)

### 3.4 Real Optimization Examples

**1. Classic missing index**

```sql
SELECT * FROM orders WHERE customer_id = 42 AND status = 'PAID';
-- Seq Scan on 10M rows → ~3s

CREATE INDEX ON orders (customer_id, status);
-- Index Scan → ~1ms
```

**2. Function on indexed column kills the index**

```sql
-- BAD: cannot use index on created_at
SELECT * FROM orders WHERE DATE(created_at) = '2025-01-15';

-- GOOD: range, uses index
SELECT * FROM orders
WHERE created_at >= '2025-01-15' AND created_at < '2025-01-16';

-- OR: expression index
CREATE INDEX ON orders ((DATE(created_at)));
```

**3. `ILIKE '%foo%'` — leading wildcard**

```sql
-- Cannot use B-tree
SELECT * FROM products WHERE name ILIKE '%widget%';

-- Use GIN with pg_trgm
CREATE EXTENSION pg_trgm;
CREATE INDEX ON products USING GIN (name gin_trgm_ops);
```

**4. `OR` that stops the planner**

```sql
-- Often seq-scans
SELECT * FROM users WHERE email = ? OR phone = ?;

-- Rewrite as UNION ALL — each branch uses its index
SELECT * FROM users WHERE email = ?
UNION ALL
SELECT * FROM users WHERE phone = ? AND email IS DISTINCT FROM ?;
```

**5. OFFSET pagination is O(offset)**

```sql
-- Slow at page 1000
SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 20000;

-- Keyset pagination — O(log n) per page
SELECT * FROM orders WHERE id > :last_seen_id ORDER BY id LIMIT 20;
```

**6. `SELECT *`**

Pulls wide columns (JSON, bytea), defeats index-only scans, transfers more over the wire. Project only what you need.

### 3.5 Optimization Strategies (Checklist)

1. Measure first (`EXPLAIN ANALYZE`, `pg_stat_statements`)
2. Fix row estimate errors (`ANALYZE`, raise `default_statistics_target`)
3. Add indexes that match real query predicates + sort orders
4. Prefer composite over multiple single-column indexes
5. Use partial indexes for skewed data: `WHERE status = 'ACTIVE'`
6. Use covering (`INCLUDE`) indexes to enable index-only scans
7. Eliminate N+1 — fetch joined data in one query
8. Avoid `DISTINCT` when `GROUP BY` or proper joins suffice
9. Rewrite correlated subqueries into joins or `LATERAL`
10. Consider materialized views for heavy aggregations

---

## 4. Connection Management

### 4.1 Why Pooling Matters

- Postgres backend startup is expensive (fork + auth + catalog load)
- Each connection holds RAM whether or not it's doing work
- TCP + TLS handshake cost is per connection
- `max_connections` is a hard wall

A pool keeps N warm connections; threads borrow/return them in microseconds.

### 4.2 HikariCP (In-Process)

Spring Boot's default. Fast, small, correct.

```yaml
spring:
  datasource:
    url: jdbc:postgresql://db:5432/app
    username: app
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 2000     # ms to wait for a connection
      idle-timeout: 600000
      max-lifetime: 1800000        # < Postgres wal_sender_timeout, < DB-side idle kill
      leak-detection-threshold: 30000
      pool-name: app-hikari
```

### 4.3 PgBouncer (External)

A separate process in front of Postgres. Three modes:

| Mode | When it returns the connection to the pool | Safe with |
|---|---|---|
| **session** | On client disconnect | Everything |
| **transaction** | On COMMIT/ROLLBACK | Most apps; no session state (no `SET`, no prepared statements with names, no `LISTEN`) |
| **statement** | Per statement | Rarely used; breaks transactions |

**Transaction pooling** is the typical production choice — lets you have 10,000 client connections sharing 200 real Postgres connections.

### 4.4 HikariCP vs PgBouncer

| | HikariCP | PgBouncer |
|---|---|---|
| Where | In-process (JVM) | Separate process/container |
| Scope | One app | Many apps / many instances |
| Protocol-aware | Yes (JDBC) | Yes (Postgres wire) |
| Useful when | One app, bounded instances | Many instances, serverless, many languages |

**You can (and often should) have both**: each JVM has HikariCP (fast borrow/return), and they all point at PgBouncer (share the finite Postgres pool). With PgBouncer in transaction mode, disable client-side prepared statements (or use `prepareThreshold=0`) unless using Postgres 14+ PgBouncer with protocol-level prepared statement support.

### 4.5 Pool Sizing

The famous formula (from HikariCP docs):

```
connections = ((core_count * 2) + effective_spindle_count)
```

For SSD, spindle_count ≈ 0–1. For a 4-core DB, pool ≈ 10. Counterintuitively, **smaller pools often give higher throughput** — less contention for DB locks and CPU. If you see your pool maxed out, the answer is usually to make queries faster, not to grow the pool.

Plan for: `app_instances × max_pool_size < max_connections − headroom`.

---

## 5. Data Access Strategies in Java

### 5.1 JPA / Hibernate

ORM: map objects to tables. Lazy loading, caching, dirty checking.

```java
@Entity
@Table(name = "orders")
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @Enumerated(EnumType.STRING)
    private Status status;

    @Version
    private Long version;

    private BigDecimal total;
    private OffsetDateTime createdAt;
}

public interface OrderRepository extends JpaRepository<Order, Long> {
    @EntityGraph(attributePaths = "customer")
    List<Order> findTop50ByStatusOrderByCreatedAtDesc(Status status);
}
```

**Pros:**
- Fast development for CRUD and simple domain models
- Change tracking, lifecycle, cascading
- Portable across databases (sort of)

**Cons:**
- Hides SQL → lazy-loading surprises, N+1, accidental `SELECT *`, big in-memory graphs
- Complex queries become awkward; JPQL is weaker than SQL
- Bulk operations bypass the persistence context (stale state)
- Hard to tune without understanding what Hibernate generates (turn on `hibernate.show_sql` / use p6spy / Datasource Proxy)

**Use when:** CRUD-dominant, domain-driven apps with moderate query complexity.
**Don't use when:** reporting, analytics, bulk ETL, apps whose performance depends on exact SQL.

### 5.2 jOOQ

Typed SQL DSL. Generates Java classes from your schema; you write queries in Java that mirror SQL 1:1.

```java
List<OrderSummary> hot = dsl
    .select(ORDERS.ID, ORDERS.TOTAL, CUSTOMERS.NAME)
    .from(ORDERS)
    .join(CUSTOMERS).on(CUSTOMERS.ID.eq(ORDERS.CUSTOMER_ID))
    .where(ORDERS.STATUS.eq("PAID"))
    .and(ORDERS.CREATED_AT.ge(since))
    .orderBy(ORDERS.CREATED_AT.desc())
    .limit(50)
    .fetchInto(OrderSummary.class);
```

**Pros:**
- Exact SQL control; compile-time safety; refactor-safe column names
- No hidden N+1, no lazy loading
- Superb for complex queries, window functions, CTEs, Postgres-specific features (JSONB, arrays, `DISTINCT ON`)

**Cons:**
- Commercial license for proprietary DBs (Postgres is free tier)
- No dirty checking / lifecycle — you manage persistence manually

**Use when:** SQL-heavy workloads, analytics, reporting, domains where the SQL *is* the logic.
**Don't use when:** trivial CRUD where JPA would be faster to write.

### 5.3 MyBatis

SQL in XML (or annotations); maps results to Java objects.

```xml
<select id="topOrders" resultType="com.example.OrderDto">
  SELECT o.id, o.total, c.name
  FROM orders o JOIN customers c ON c.id = o.customer_id
  WHERE o.status = #{status}
  ORDER BY o.created_at DESC
  LIMIT #{limit}
</select>
```

**Pros:**
- Write SQL exactly; DBA-friendly
- Simpler than jOOQ for teams already comfortable with SQL
- Lightweight, fast

**Cons:**
- No compile-time SQL validation
- Verbose mappers; refactoring column names is error-prone
- Weaker domain modeling than JPA

**Use when:** legacy SQL-heavy apps, strict DBA control of SQL, Chinese/Japanese tech stacks where MyBatis is culturally dominant.
**Don't use when:** greenfield Spring Boot where jOOQ gives similar control with type safety.

### 5.4 Mixed Strategy (Pragmatic Default)

In real systems: **JPA for write-side entity models; jOOQ (or native queries) for reads and reports.** Best of both: lifecycle + explicit SQL where it matters.

```java
@Repository
@RequiredArgsConstructor
public class OrderQueryService {
    private final DSLContext dsl;                      // jOOQ: reads
    private final OrderRepository repo;                // JPA: writes
}
```

---

## 6. Schema Management

Schemas must be versioned like code. Never let developers run ad-hoc `ALTER TABLE` in production.

### 6.1 Flyway

Plain SQL, linear versioning.

```
src/main/resources/db/migration/
  V1__init.sql
  V2__add_orders_status_index.sql
  V3__add_idempotency_keys.sql
  R__refresh_reporting_views.sql         # Repeatable — re-run on checksum change
```

```sql
-- V2__add_orders_status_index.sql
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_status
  ON orders (status)
  WHERE status IN ('PENDING','PAID');
```

```yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
    out-of-order: false
```

> ⚠️ Flyway runs migrations in a transaction by default. `CREATE INDEX CONCURRENTLY` **cannot** run in a transaction. Add the header `-- noTransaction` or disable via `flyway.postgresql.transactional-lock=false` and ensure the file uses a single non-transactional statement. Many teams keep index creation in a separate migration for this reason.

### 6.2 Liquibase

XML/YAML/JSON/SQL changelogs, supports rollback and DB-agnostic syntax.

```yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: kirill
      changes:
        - createTable:
            tableName: customers
            columns:
              - column: { name: id, type: BIGSERIAL, constraints: { primaryKey: true } }
              - column: { name: email, type: VARCHAR(255), constraints: { nullable: false, unique: true } }
```

### 6.3 Flyway vs Liquibase

| | Flyway | Liquibase |
|---|---|---|
| Syntax | Plain SQL | Abstracted changesets |
| Rollback | Paid tier | Built-in |
| DB-agnostic | Less | More |
| Learning curve | Low | Medium |

Most teams pick **Flyway + SQL** for Postgres. SQL is the real interface to the database; abstraction rarely pays.

### 6.4 Migration Best Practices

- **Never** edit a committed migration — add a new one
- Migrations must be **backwards-compatible** for the duration of a deploy window (rolling deploys = old + new code both run)
- **Expand → migrate data → contract** for breaking changes (see §9)
- `CREATE INDEX CONCURRENTLY` on big tables; `SET lock_timeout` / `statement_timeout` at start
- Never rename columns in a single migration — use a new column, backfill, switch reads, drop old
- Add NOT NULL in stages: add column nullable + default → backfill → set NOT NULL
- Keep migrations idempotent where possible (`IF NOT EXISTS`)
- Review migrations like code. They run on production at 3am.

---

## 7. Mapping & DTOs

### 7.1 Why Mapping Matters

Entities encode persistence concerns (lazy refs, cycles, `@Version`). DTOs encode API/contract concerns. Mixing them leaks database details to clients and vice versa. Manual mapping is error-prone and verbose.

### 7.2 MapStruct

Compile-time mappers via annotation processor. Zero reflection.

```java
@Mapper(componentModel = "spring")
public interface OrderMapper {
    @Mapping(source = "customer.name", target = "customerName")
    @Mapping(source = "status",        target = "status", qualifiedByName = "toDto")
    OrderDto toDto(Order entity);

    List<OrderDto> toDtoList(List<Order> entities);

    @Named("toDto")
    default String statusToDto(Status s) { return s == null ? null : s.name(); }
}
```

```xml
<annotationProcessorPaths>
  <path>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct-processor</artifactId>
    <version>1.6.3</version>
  </path>
</annotationProcessorPaths>
```

### 7.3 Performance

- Generated code is plain Java — as fast as hand-written
- **Avoid** ModelMapper / Dozer in hot paths — reflection-based, 10–100x slower
- For truly hot transformation: write it manually; it's a few lines

### 7.4 Patterns

- One DTO per API shape (create, update, response). Don't share DTOs across directions.
- Keep MapStruct mappers in one place, not scattered
- Don't fetch the entity just to map it to a DTO — use a projection (JPA `interface` projection, jOOQ, or native query) that reads only needed columns

---

## 8. Transactions & Concurrency

### 8.1 Optimistic Locking

"I assume no one else is modifying this; detect at commit time."

```java
@Entity
public class Order {
    @Id Long id;
    @Version Long version;   // Hibernate increments automatically
}
```

On update, Hibernate issues:

```sql
UPDATE orders SET ..., version = 3 WHERE id = ? AND version = 2;
```

If zero rows updated → `OptimisticLockException`. You retry.

**Use when:** low write contention, user-facing edits, REST `PUT` with an ETag.
**Don't use when:** a single hot row is updated by many writers (every retry fails) — use pessimistic or refactor.

### 8.2 Pessimistic Locking

"I'm going to modify this; keep everyone else out."

Backed by Postgres row locks.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select a from Account a where a.id = :id")
Account lockById(@Param("id") Long id);
```

Emits `SELECT ... FOR UPDATE`. Holds the lock till transaction end.

Variants:
- `FOR UPDATE` — full row lock (others can't even `SELECT FOR UPDATE`)
- `FOR NO KEY UPDATE` — weaker, allows FK inserts
- `FOR SHARE` — multiple readers, no writers
- `FOR UPDATE SKIP LOCKED` — **the key to job queues in SQL** — skip rows another worker already has

```sql
-- Classic SQL-based work queue
SELECT id FROM jobs
WHERE status = 'READY'
ORDER BY scheduled_at
FOR UPDATE SKIP LOCKED
LIMIT 10;
```

### 8.3 Postgres Row-Level Locks Internally

There is no table of locks for row-level locks. Postgres stores the lock info **in the row itself** — `xmax` plus a bit. This is why row locks scale to billions of rows but can show up as tuple-level contention under heavy update.

Table-level locks live in the shared lock table (`pg_locks`). You can always inspect what's locked:

```sql
SELECT pid, locktype, relation::regclass, mode, granted, query
FROM pg_locks l
JOIN pg_stat_activity a USING (pid)
WHERE NOT granted;
```

### 8.4 Deadlocks

Happen when two transactions hold locks the other wants, in opposite order:

```
T1: lock row A, wants row B
T2: lock row B, wants row A
```

Postgres detects deadlocks (after `deadlock_timeout`, default 1s) and aborts one with:

```
ERROR: deadlock detected
```

**Prevention:**
- Lock resources in a **consistent order** (e.g., always by ascending `account_id`)
- Keep transactions **short** — don't call external APIs inside them
- Use `SELECT FOR UPDATE SKIP LOCKED` where semantics allow
- Avoid application-level locks that span DB transactions

**Debugging:**
- `log_lock_waits = on`, `deadlock_timeout = 1s`
- `pg_stat_activity`, `pg_locks` joins
- Capture the deadlock error; it includes both queries

### 8.5 Spring Transaction Management

```java
@Service
@RequiredArgsConstructor
public class Transfers {
    private final AccountRepo repo;

    @Transactional(isolation = Isolation.READ_COMMITTED,
                   timeout = 5,
                   rollbackFor = Exception.class)
    public void transfer(Long from, Long to, BigDecimal amount) {
        // lock in id-order to avoid deadlocks
        Long a = Math.min(from, to), b = Math.max(from, to);
        Account ax = repo.lockById(a);
        Account bx = repo.lockById(b);
        Account src = a.equals(from) ? ax : bx;
        Account dst = a.equals(from) ? bx : ax;
        src.debit(amount);
        dst.credit(amount);
    }
}
```

Propagation gotchas:
- `REQUIRED` (default) — join outer or start new
- `REQUIRES_NEW` — suspend outer, new tx (new physical connection under the hood → watch for pool exhaustion)
- `NESTED` — savepoint inside same tx
- `SUPPORTS` / `NOT_SUPPORTED` — rarely needed

**Self-invocation gotcha:** `@Transactional` is proxy-based. Calling `this.otherMethod()` from within the same bean bypasses the proxy and the annotation is ignored. Move it to another bean, or inject self, or use AspectJ weaving.

---

## 9. Designing Reliable Systems

### 9.1 Idempotent Operations

An operation is **idempotent** when applying it N times ≥ 1 yields the same state. Essential because networks retry.

Patterns:
- **Natural idempotency:** `PUT /resource/123` with full state is idempotent; `POST /resource` is not.
- **Idempotency key:** client sends `Idempotency-Key: uuid` header; server stores key + result; second call returns cached result.

```sql
CREATE TABLE idempotency_keys (
  key            UUID PRIMARY KEY,
  request_hash   BYTEA NOT NULL,
  response_body  JSONB,
  response_code  INT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 9.2 Retry-Safe Logic

- Make the *effect* idempotent, not the *code path*
- Use database unique constraints as a safety net: dedupe by key inserted in the same transaction as the side effect
- For external side effects, check-and-execute with a "claim" row

### 9.3 The Transactional Outbox (The Pattern You Must Know)

Problem: you want to commit to DB *and* publish to Kafka. You can't do both atomically.

Solution:

```sql
CREATE TABLE outbox (
  id          BIGSERIAL PRIMARY KEY,
  aggregate   TEXT NOT NULL,
  event_type  TEXT NOT NULL,
  payload     JSONB NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at TIMESTAMPTZ
);
```

In the same transaction as the business write, insert a row into `outbox`. A separate poller (or CDC via Debezium) reads unpublished rows and emits them to Kafka. Atomicity of DB write + event intention is guaranteed by the single transaction.

### 9.4 Distributed Transactions & Sagas

Classical 2PC is **not used in modern microservices** — it blocks resources and doesn't scale. Instead:

**Saga:** a sequence of local transactions, each with a compensating action. If step 3 fails, compensations run for steps 1 and 2.

Two flavors:
- **Choreography** — services react to each other's events (no central brain)
- **Orchestration** — a saga orchestrator drives the sequence (easier to reason about)

Trade-off: sagas give you eventual consistency, not isolation. A user can observe intermediate states. Design your APIs (and UI) for that.

### 9.5 Other Reliability Patterns

- **Write-path idempotency:** `INSERT ... ON CONFLICT DO NOTHING` for natural keys
- **Optimistic concurrency with version:** prevents lost updates
- **Advisory locks** (`pg_advisory_xact_lock`): application-level mutexes backed by Postgres — cleaner than app-side locks in a multi-instance deploy
- **Dead-letter tables** for messages/jobs that exhaust retries

---

## 10. Scaling the Database

Order of escalation:
1. Tune queries (90% of problems)
2. Add read replicas
3. Partition large tables
4. Archive cold data
5. Shard (last resort)

### 10.1 PostgreSQL Native Partitioning

Split one logical table into many physical tables by a key. Postgres 11+ supports declarative partitioning.

```sql
CREATE TABLE events (
    id          BIGSERIAL,
    tenant_id   BIGINT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL,
    payload     JSONB
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2025_01 PARTITION OF events
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE INDEX ON events_2025_01 (tenant_id, created_at DESC);
```

**Partitioning strategies:**
- **Range** — time-series, append-only logs
- **List** — categorical (country, tenant group)
- **Hash** — spread load evenly, no natural range

**Wins:**
- **Partition pruning** — queries with the partition key only scan relevant partitions
- Drop old data in O(1): `DROP TABLE events_2024_01` (vs months-long DELETE + VACUUM)
- Per-partition vacuum/analyze
- Per-partition indexes (can differ by partition)

**Costs:**
- Global uniqueness across partitions requires the partition key in the PK
- Planner overhead grows with partition count (keep < ~1000; pg 14+ handles more)
- Foreign keys to partitioned tables have restrictions

**Use when:** tables >50 GB, clear partition key, workload naturally scoped (time, tenant).
**Don't use when:** table is small; no clear partition key; queries span all partitions (you just added overhead).

### 10.2 Sharding (Conceptual)

Horizontal split **across machines**. Each shard is an independent Postgres instance; a routing layer decides which shard owns a given row.

Approaches:
- **Application-level sharding** — app computes `shard = hash(tenant_id) % N` and routes
- **Middleware** — Citus (Postgres extension turning it into distributed Postgres), Vitess (MySQL-native)
- **DBaaS** — Aurora, Yugabyte, Cockroach (different consistency trade-offs)

**Hard parts of sharding:**
- Cross-shard joins → slow or impossible
- Distributed transactions → 2PC with all its problems, or sagas
- Rebalancing when a shard grows — operational nightmare without good tooling
- Schema migrations across shards
- Query planner cannot see all shards

**Rule:** don't shard until you've exhausted partitioning, replicas, and archiving. Most "we need to shard" conversations end after someone adds a proper index.

### 10.3 Read Replicas

Streaming replication → async or sync replicas. Route read-only queries to replicas; writes always go to primary.

```java
@Transactional(readOnly = true)
public Order fetch(Long id) { ... }   // routed to replica
```

Caveat: **replication lag**. A replica may be seconds behind. If the user just wrote and immediately reads, route that read to primary or tolerate eventual consistency.

---

## 11. Caching Strategies

### 11.1 Why Cache

- Read is cheap from RAM (~100 ns) vs disk (100 µs) vs network DB (ms)
- Offload expensive computations / joins / aggregations
- Absorb bursty traffic

### 11.2 Where to Cache

| Layer | Example | Scope | Consistency |
|---|---|---|---|
| Client / CDN | HTTP cache, Cloudflare | Public | TTL-based |
| Application in-memory | Caffeine | Single JVM | Per-instance |
| Distributed cache | Redis, Memcached | Cluster-wide | Centralized |
| Database | Postgres buffer cache | Implicit | Managed by PG |

### 11.3 Caffeine (In-Process)

```java
@Configuration
@EnableCaching
public class CacheConfig {
    @Bean
    public CacheManager caches() {
        CaffeineCacheManager m = new CaffeineCacheManager("products", "categories");
        m.setCaffeine(Caffeine.newBuilder()
            .maximumSize(10_000)
            .expireAfterWrite(Duration.ofMinutes(10))
            .recordStats());
        return m;
    }
}

@Cacheable(cacheNames = "products", key = "#id")
public Product get(Long id) { ... }
```

**Pros:** nanosecond access, no network, no serialization.
**Cons:** per-instance — 10 app replicas = 10 caches; invalidations are tricky.

**Use when:** immutable or slowly-changing reference data; expensive computation; per-request memoization.

### 11.4 Redis

Shared cache/store across instances.

```java
@Bean
public RedisCacheConfiguration redisConfig() {
    return RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .serializeValuesWith(SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));
}

@Cacheable(cacheNames = "orders", key = "#id")
public OrderDto get(Long id) { ... }
```

**Pros:** shared state, pub/sub for invalidation, rich data types (sorted sets for leaderboards, streams for events).
**Cons:** network hop (~0.5 ms), another system to run, serialization overhead.

### 11.5 Patterns

#### Cache-aside (Lazy Loading) — Default

```
1. App reads cache.
2. Miss? Read DB.
3. Write to cache.
```

Trade-off: first request pays DB cost. Stale data until TTL. **Works for most workloads.**

#### Write-through

```
Every write → DB AND cache (synchronously).
```

Trade-off: slower writes. Cache always fresh.

#### Write-behind (Write-back)

```
Writes to cache; cache asynchronously flushes to DB.
```

Trade-off: risk of loss on cache failure. Use only for tolerant workloads (analytics counters).

#### Read-through

Cache library fetches from DB on miss (Caffeine `LoadingCache`). Clean API; same semantics as cache-aside.

### 11.6 Invalidation

> "There are only two hard things in computer science: cache invalidation and naming things."

Options:
- **TTL** — simplest; accept staleness up to TTL
- **Explicit eviction** on write: `@CacheEvict`
- **Version-in-key:** include a version in the cache key; bump it to invalidate all (avoids stale reads during writes)
- **Pub/sub invalidation:** Redis pub/sub to notify replicas to drop local Caffeine entries

```java
@CacheEvict(cacheNames = "products", key = "#p.id")
public Product update(Product p) { return repo.save(p); }
```

### 11.7 When Caching Hurts

- Highly dynamic data with low read multiplicity (cache miss rate ~100%)
- Writes far more frequent than reads (cache is useless)
- Strict consistency requirements (finance, inventory) — staleness is a bug
- Small datasets that already fit in Postgres shared_buffers — you're adding a hop

---

## 12. Database Tools

### 12.1 DBeaver

Universal SQL client. Essential capabilities:

- **Explain plan visualization:** Ctrl+Shift+E on a query → graphical plan tree. Red nodes = expensive steps.
- **Schema inspection:** ERD view, constraints, triggers, partitions, sizes
- **Data export:** CSV, SQL inserts, clipboard (great for bug repros)
- **Session manager:** see active sessions, their queries, lock waits — like `htop` for the database
- **Query history + bookmarks**

### 12.2 Diagnostic SQL You Should Memorize

```sql
-- Top 20 slowest queries (needs pg_stat_statements)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 20;

-- Currently running queries
SELECT pid, now() - query_start AS runtime, state, query
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY runtime DESC;

-- Blocking / blocked
SELECT blocked.pid AS blocked_pid, blocking.pid AS blocking_pid,
       blocked.query AS blocked_q, blocking.query AS blocking_q
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));

-- Table / index sizes
SELECT relname, pg_size_pretty(pg_total_relation_size(relid)) AS total
FROM pg_catalog.pg_statio_user_tables
ORDER BY pg_total_relation_size(relid) DESC LIMIT 20;

-- Unused indexes (candidates to drop)
SELECT schemaname, relname, indexrelname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;

-- Bloat (rough): dead tuple ratio
SELECT relname,
       n_dead_tup,
       n_live_tup,
       CASE WHEN n_live_tup > 0 THEN round(100.0 * n_dead_tup / n_live_tup, 2) END AS dead_pct
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 20;
```

Enable `pg_stat_statements`:

```sql
-- in postgresql.conf
shared_preload_libraries = 'pg_stat_statements'
-- then restart, then:
CREATE EXTENSION pg_stat_statements;
```

### 12.3 Complementary Tools

- **pgAdmin** — Postgres-specific; heavier UI
- **psql** — CLI; indispensable for scripting and emergencies
- **pgbadger** — log analyzer; generates reports from `log_min_duration_statement`
- **pg_top / pg_activity** — `top`-style live view
- **Datadog / pganalyze / RDS Performance Insights** — production-grade observability

---

## 13. Performance & Tuning

### 13.1 Index Design Rules

1. Index predicates (`WHERE`), join columns, and sort orders
2. Order columns in a composite index by **selectivity and usage** (leftmost = most commonly filtered)
3. Partial indexes for lopsided data: `WHERE deleted = false`
4. Covering indexes for hot read paths (`INCLUDE` clause)
5. Don't index low-cardinality booleans alone — often just use partial
6. Drop unused indexes — they slow writes and bloat
7. Rebuild bloated indexes: `REINDEX CONCURRENTLY`

### 13.2 Query Optimization Habits

- Measure with `EXPLAIN (ANALYZE, BUFFERS)` before and after
- Use `pg_stat_statements` to find real pain, not assumed pain
- Avoid `SELECT *`
- Push filters as close to the source as possible
- Prefer joins over correlated subqueries
- Use `LATERAL` for "top-N per group"
- Beware of implicit casts killing index usage (`WHERE varchar_col = 123` — cast side you don't expect)

### 13.3 Connection Tuning

- `max_connections` = as low as you can get away with
- Always pool (HikariCP + optionally PgBouncer)
- `idle_in_transaction_session_timeout = 30s` — kill apps that leave transactions open
- `statement_timeout` — per-statement ceiling, protects against runaway queries

### 13.4 Memory Settings (Postgres)

Starting points (tune from monitoring):

| Param | Guideline |
|---|---|
| `shared_buffers` | 25% of RAM |
| `effective_cache_size` | 50–75% of RAM (planner hint, not allocation) |
| `work_mem` | 4–32 MB — **per sort/hash per query**; beware of multiplication by concurrency |
| `maintenance_work_mem` | 512 MB – 2 GB (vacuum, index build) |
| `wal_buffers` | -1 (auto) |
| `checkpoint_timeout` | 15 min |
| `max_wal_size` | 2–8 GB |
| `random_page_cost` | 1.1 on SSD (default 4 assumes HDD) |
| `default_statistics_target` | 100–500 (higher = better plans, slower ANALYZE) |

### 13.5 Avoiding N+1

Classic offender:

```java
List<Order> orders = orderRepo.findByStatus(PAID);   // 1 query
orders.forEach(o -> o.getCustomer().getName());      // N queries — lazy load
```

Fixes, in order of preference:

1. **JOIN FETCH** / `@EntityGraph`:
   ```java
   @EntityGraph(attributePaths = "customer")
   List<Order> findByStatus(Status s);
   ```
2. **Batch fetching:**
   ```java
   @BatchSize(size = 50)  // on the @ManyToOne or on the entity class
   ```
3. **DTO projection** — skip entities entirely for read paths
4. **jOOQ / native query** — write the exact join

Always watch the SQL log during dev. If you count more queries than entities, you have N+1.

### 13.6 Bulk Operations

- JPA's `saveAll()` is still one INSERT per row unless you configure batching:
  ```properties
  spring.jpa.properties.hibernate.jdbc.batch_size=50
  spring.jpa.properties.hibernate.order_inserts=true
  spring.jpa.properties.hibernate.order_updates=true
  ```
- For >10k rows: `COPY` via JDBC (`org.postgresql.copy.CopyManager`) — orders of magnitude faster
- For bulk updates: `UPDATE ... FROM VALUES (...)` or `jdbcTemplate.batchUpdate`

---

## 14. Spring Boot Integration

### 14.1 Datasource Configuration

```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST}:5432/${DB_NAME}?reWriteBatchedInserts=true
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 2000
      max-lifetime: 1800000
      leak-detection-threshold: 30000

  jpa:
    open-in-view: false                # ALWAYS — prevents lazy-load surprises
    properties:
      hibernate:
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
        default_batch_fetch_size: 50
        generate_statistics: false
    hibernate:
      ddl-auto: validate               # never `update` in prod

  flyway:
    enabled: true
    locations: classpath:db/migration

logging:
  level:
    org.hibernate.SQL: INFO
    org.hibernate.orm.jdbc.bind: TRACE   # see bound parameters (dev only)
```

> ⚠️ Turn off `open-in-view` in every real project. With it on, the Hibernate session stays open through controller rendering — entity lazy loads happen *outside* transactions, silently. Disabling forces you to load what you need, when you need it.

### 14.2 Transaction Management

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orders;
    private final OutboxRepository outbox;

    @Transactional
    public Order place(PlaceOrder cmd) {
        Order o = new Order(cmd);
        orders.save(o);
        outbox.save(new OutboxEvent("OrderCreated", o));
        return o;
    }

    @Transactional(readOnly = true)
    public Page<Order> list(Status s, Pageable p) {
        return orders.findByStatus(s, p);
    }
}
```

- Keep transactions short — no HTTP calls, no `Thread.sleep`, no big in-memory joins
- Separate read-only methods with `readOnly = true` — enables optimizer and replica routing
- Put `@Transactional` on the service layer, not the repository — the service is the business boundary

### 14.3 JPA + jOOQ in the Same App

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-jooq</artifactId>
</dependency>
```

Both share the same `DataSource`, same transactions. Mix freely.

```java
@Repository
@RequiredArgsConstructor
public class OrderReadRepository {
    private final DSLContext dsl;

    public List<OrderDashboardRow> dashboard(LocalDate from) {
        return dsl.select(
                    ORDERS.CUSTOMER_ID,
                    count().as("orders"),
                    sum(ORDERS.TOTAL).as("revenue"))
                .from(ORDERS)
                .where(ORDERS.CREATED_AT.ge(from.atStartOfDay().toOffsetDateTime(ZoneOffset.UTC)))
                .groupBy(ORDERS.CUSTOMER_ID)
                .orderBy(field("revenue").desc())
                .limit(100)
                .fetchInto(OrderDashboardRow.class);
    }
}
```

### 14.4 MyBatis Wired Into Spring Boot

```xml
<dependency>
  <groupId>org.mybatis.spring.boot</groupId>
  <artifactId>mybatis-spring-boot-starter</artifactId>
  <version>3.0.4</version>
</dependency>
```

```java
@Mapper
public interface OrderMapper {
    @Select("SELECT id, total, status FROM orders WHERE id = #{id}")
    OrderRow findById(@Param("id") Long id);
}
```

### 14.5 Flyway Wired In

Already shown in §6. Key detail: Flyway runs **before** Hibernate validates the schema. If you use `ddl-auto: validate`, the migration must bring the schema to the state the entities expect.

### 14.6 Health & Observability

```yaml
management:
  endpoints.web.exposure.include: health, metrics, prometheus
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
        hikaricp.connections.usage: true
```

Key metrics to watch:
- `hikaricp_connections_pending` — pool under pressure
- `hikaricp_connections_usage` — borrow duration (P99 should be < a few ms)
- DB side: `pg_stat_activity` count, `pg_stat_database.xact_commit`, WAL rate

---

## 15. Real-World Scenarios

### 15.1 High-Load Transactional System (Payments)

**Profile:** 5k writes/sec, strict consistency, auditable.

- **Schema:** accounts, ledger entries (double-entry), idempotency keys, outbox
- **Isolation:** READ COMMITTED; row-level locks with `FOR UPDATE` on account rows; lock in id order
- **Data access:** JPA for entities; jOOQ for reports; `COPY` for bulk reconciliation
- **Concurrency:** optimistic `@Version` for user-editable entities; pessimistic for financial mutation
- **Reliability:** idempotency keys on every mutating endpoint; transactional outbox → Kafka; sagas for multi-service flows
- **Scaling:** partition ledger by month; archive >2 years to cold storage; read replica for reporting
- **Monitoring:** alert on `pg_stat_activity` > 80% of `max_connections`, on lock wait > 5s, on WAL rate anomalies

### 15.2 Read-Heavy System (Catalog / E-commerce)

**Profile:** 50:1 reads:writes, latency matters, some staleness OK.

- **Caching:** Caffeine for categories + product metadata (in-process); Redis for cart/session
- **DB:** strong indexes for filter/sort (category + price); covering indexes on listing queries
- **Search:** full-text via `tsvector` + GIN, or Elasticsearch if query complexity warrants
- **Replicas:** reads to async replicas; writes to primary
- **Data access:** JPA for admin CRUD; jOOQ for search and recommendation queries
- **Materialized views** for nightly top-selling lists; `REFRESH MATERIALIZED VIEW CONCURRENTLY`

### 15.3 Microservices — Shared DB vs Separate DB

**Shared DB (antipattern in most cases):**
- Pros: simple joins, one source of truth, cheap ops
- Cons: coupling at the schema level, migrations become a coordination nightmare, isolation goes out the window, can't scale independently

**Database-per-service (default for microservices):**
- Each service owns its schema (sometimes its own Postgres instance)
- Cross-service data fetched via APIs or events
- Consistency between services via sagas + outbox

**Pragmatic middle ground:** shared Postgres cluster, separate schemas per service, strict "no cross-schema reads" policy enforced in CI.

### 15.4 Event-Driven Analytics Pipeline

- App writes to DB **and** outbox in same tx
- Debezium reads Postgres WAL, publishes to Kafka
- Stream processor (Kafka Streams / Flink) aggregates into ClickHouse / BigQuery
- Primary Postgres stays lean — OLTP only

### 15.5 Multi-Tenant SaaS

Three common models:

| Model | Isolation | Ops cost | Fits |
|---|---|---|---|
| Shared schema, `tenant_id` column | Logical | Low | Most SaaS; millions of small tenants |
| Schema per tenant | Stronger | Medium (migrations × N) | Hundreds of enterprise tenants |
| DB per tenant | Strongest | High | Regulated / huge customers |

For "shared schema," partition large tables by `tenant_id` (hash or list). Always include `tenant_id` in PK and indexes.

---

## 16. Comparisons & Trade-offs

### 16.1 JPA vs jOOQ vs MyBatis

| Dimension | JPA/Hibernate | jOOQ | MyBatis |
|---|---|---|---|
| Abstraction level | High (ORM) | Low (typed SQL) | Low (SQL + mapper) |
| Compile-time safety | Entity mappings | Full SQL | Minimal |
| Complex SQL (CTEs, windows) | Painful | Native | Native |
| Change tracking / dirty checks | Yes | No | No |
| N+1 risk | High | None | None |
| Learning curve | High (magic) | Medium | Low |
| Refactor safety | Entity refactor | Full (generated) | Low |
| License | Free | Commercial for non-open-source DBs | Free |
| Best at | CRUD, domain models | Reads, reports, Postgres features | Legacy / SQL-dominant teams |

### 16.2 Redis vs DB Queries

| | DB query | Redis |
|---|---|---|
| Latency | 1–50 ms | 0.3–1 ms |
| Consistency | Strong | Eventual |
| Query flexibility | SQL | KV + data structures |
| Durability | Strong (WAL) | Weaker (AOF / RDB) |
| Cost | CPU + IO on DB | Extra infra + network |

Cache for reads that are **expensive to compute** and **tolerant of staleness**. Don't cache for primitive lookups that are already sub-ms in Postgres — you'll add latency and complexity for no gain.

### 16.3 Partitioning vs Sharding

| | Partitioning | Sharding |
|---|---|---|
| Scope | Single DB instance | Multiple DB instances |
| Ops complexity | Low | Very high |
| Joins across partitions | Yes (planner handles) | Cross-shard = manual |
| Transactions | Standard | Distributed (hard) |
| Scaling writes | Limited (one machine) | Yes (many machines) |
| Good for | >50 GB tables, natural key | 100 TB+, >10k w/s sustained |

Partition **first**. Shard only when a single primary cannot be scaled vertically (and you've tried read replicas, archiving, and query tuning).

---

## 17. When NOT to Use Certain Approaches

### 17.1 When NOT to Use JPA

- **Bulk ETL** — Hibernate's persistence context isn't designed for millions of rows. Use JDBC / `COPY`.
- **Reporting / analytics** — Complex joins, windows, CTEs. Use jOOQ or native SQL.
- **Performance-critical hot paths** — every `@Entity` has overhead; projections help, but jOOQ is often cleaner.
- **CQRS read side** — DTO projections beat entity loading.
- **Non-relational data models** — JSONB-heavy, graph-like, or highly denormalized.

### 17.2 When NOT to Cache

- **Inconsistency is unacceptable** — ledgers, inventory with race conditions
- **High write:read ratio** — cache misses dominate
- **Small dataset, fits in DB buffer cache** — you're adding a hop
- **Cache key cardinality is huge** (unique per request) — hit rate ≈ 0

### 17.3 When NOT to Shard

- Table is <100 GB and growing slowly
- A bigger instance (vertical scaling) buys you years
- Your real bottleneck is a missing index
- You don't have a dedicated team to run distributed databases
- Your cross-entity queries span many shards

### 17.4 When NOT to Use Stored Procedures / Triggers

- Business logic scattered across DB and app layers
- Hard to version, test, deploy
- Acceptable for strict DB-level invariants (audit triggers, cascading soft deletes) — keep it minimal

### 17.5 When NOT to Partition

- Small table
- No clear partition key
- Most queries don't filter by the partition key (no pruning = pure overhead)

### 17.6 When NOT to Use `ddl-auto=update`

**Always.** In prod. Use migrations.

---

## 18. Final Checklist

**You are production-ready in database engineering if you can:**

- [ ] Explain ACID, MVCC, and WAL in a way a junior understands
- [ ] Read `EXPLAIN (ANALYZE, BUFFERS)` output and spot the actual problem
- [ ] Design a composite index given a query and justify the column order
- [ ] Recognize and fix an N+1 query in a JPA codebase
- [ ] Choose isolation level based on business semantics, not out of habit
- [ ] Implement optimistic locking with `@Version` and pessimistic with `FOR UPDATE`, and justify each
- [ ] Diagnose a deadlock from `pg_locks` and rewrite code to prevent it
- [ ] Configure HikariCP sensibly and explain when PgBouncer helps
- [ ] Write a Flyway migration that adds an index to a 500M-row table **without locking it**
- [ ] Design a transactional outbox and explain why it's needed
- [ ] Explain the difference between partitioning and sharding and when each applies
- [ ] Decide between JPA, jOOQ, and MyBatis for a given workload and defend the choice
- [ ] Implement cache-aside with Caffeine or Redis and reason about invalidation
- [ ] Configure `statement_timeout`, `idle_in_transaction_session_timeout`, and explain why
- [ ] Tune `shared_buffers`, `work_mem`, and `effective_cache_size` with justification
- [ ] Identify bloated tables and know when to `VACUUM`, `REINDEX`, or repartition
- [ ] Use `pg_stat_statements` to find real production pain, not speculation
- [ ] Design a schema change for a rolling deploy (expand → migrate → contract)
- [ ] Spot when a "scaling problem" is actually a missing index
- [ ] Argue convincingly for the **simplest** solution that meets the requirement

If you can tick every box, you can own a PostgreSQL-backed service in production without waking up your team at 3am — or at least, when you do, you'll know where to look first.
