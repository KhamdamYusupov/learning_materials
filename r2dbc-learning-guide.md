# R2DBC: The Production-Grade Engineering Guide

> A deep, opinionated, real-world guide to Reactive Relational Database Connectivity for senior backend engineers.
> Written for developers fluent in Java 17/21, Spring Boot, PostgreSQL, JPA/Hibernate, Docker, and Kubernetes — with basic Spring WebFlux exposure but no prior R2DBC experience.

---

## Table of Contents

1. [Reactive Database Fundamentals](#1-reactive-database-fundamentals)
2. [R2DBC Fundamentals](#2-r2dbc-fundamentals)
3. [R2DBC Architecture & How It Works](#3-r2dbc-architecture--how-it-works)
4. [R2DBC vs JPA/JDBC](#4-r2dbc-vs-jpajdbc)
5. [Spring Data R2DBC](#5-spring-data-r2dbc)
6. [Reactive Query Patterns](#6-reactive-query-patterns)
7. [Transactions in R2DBC](#7-transactions-in-r2dbc)
8. [R2DBC + WebFlux Integration](#8-r2dbc--webflux-integration)
9. [Performance & Tuning](#9-performance--tuning)
10. [Production Best Practices](#10-production-best-practices)
11. [R2DBC in System Design](#11-r2dbc-in-system-design)
12. [Real-World Scenarios](#12-real-world-scenarios)
13. [Comparisons & Trade-offs](#13-comparisons--trade-offs)
14. [Final Checklist](#14-final-checklist)

---

## 1. Reactive Database Fundamentals

### 1.1 Why Traditional JDBC Is Blocking

JDBC, designed in 1997, is built on a **synchronous, thread-per-request** model:

```java
Connection conn = dataSource.getConnection();         // blocks until a connection is available
PreparedStatement ps = conn.prepareStatement("...");  // blocks
ResultSet rs = ps.executeQuery();                     // blocks the thread until the DB returns
while (rs.next()) { ... }                             // each next() can block on network I/O
```

Every blocking call parks the carrier thread on a socket `read()` syscall. The thread cannot do other work — it is *owned* by that single query for its entire duration.

The JDBC SPI (`java.sql.*`) was never designed for non-blocking I/O. There is no callback, no `Publisher`, no signal of "more data when ready." The API fundamentally assumes the caller will block and wait.

### 1.2 Problems With Blocking I/O at High Concurrency

Under the Tomcat / Thread-per-request model:

| Concurrency | Threads Needed | Memory (1 MB stack each) | Context-Switch Cost |
|---|---|---|---|
| 100 reqs/s, 100 ms each | ~10 | ~10 MB | Trivial |
| 10,000 reqs/s, 100 ms each | ~1,000 | ~1 GB | Substantial |
| 100,000 connections idle on slow queries | ~100,000 | ~100 GB | Catastrophic |

**Symptoms** you will see:
- Thread pool saturation → requests queue → p99 latency spikes.
- "Connection pool exhausted" exceptions even when the DB is idle.
- A single slow query (e.g., missing index) monopolizes threads and takes the whole service down.
- Horizontal scaling is the only lever — and it's expensive.

JDK 21's **virtual threads (Project Loom)** actually mitigate most of this — see §4.5 — but for now, classical Tomcat + JDBC stacks pay the thread tax.

### 1.3 What Reactive Database Access Means

**Reactive** means:
1. **Non-blocking I/O**: a query is submitted; the calling thread is released immediately. When the DB replies, an event loop notifies a subscriber.
2. **Asynchronous by default**: results flow as signals (`onNext`, `onComplete`, `onError`) rather than being pulled synchronously.
3. **Backpressure-aware**: the consumer tells the producer how much it can handle via `request(n)`.

Think of it as **Netty ↔ DB**: one thread handles thousands of in-flight queries because each is just an open socket waiting for bytes. When bytes arrive, a handler is dispatched; when the app is slow, upstream is told to slow down.

### 1.4 Backpressure in Database Interactions

Backpressure is the mechanism that prevents fast producers from overwhelming slow consumers.

In Reactor terms:

```
DB Publisher  ──onNext(row)──▶ ... ──▶  Subscriber
              ◀──request(n)──
```

- The subscriber calls `request(n)`.
- The publisher emits **up to n** items.
- When the subscriber is ready, it requests more.

With a streaming query like `SELECT * FROM big_table`, R2DBC drivers translate this into protocol-level flow control — e.g., Postgres cursors or PostgreSQL's `Execute(n)` limits, or incremental fetching. This means you can `Flux<Row>` over 100M rows without materializing them in memory.

**Without backpressure** (classic JDBC `ResultSet`), the driver buffers, or you manually set `fetchSize`. With R2DBC, flow control is first-class.

### 1.5 The Reactive Streams Contract

R2DBC APIs return `Publisher<T>` (from `org.reactivestreams`). Reactor wraps these as `Mono<T>` (0..1) and `Flux<T>` (0..N).

```
Publisher.subscribe(Subscriber)
  → Subscriber.onSubscribe(Subscription)
  → Subscription.request(n)
  → Publisher.onNext(item) * N
  → Publisher.onComplete() | Publisher.onError(t)
```

Rules that matter for DB code:
- Nothing happens until you subscribe. A `Mono` that wasn't subscribed runs **no SQL**.
- Cancellation is cooperative — `dispose()` / cancel signals propagate back; drivers should cancel the in-flight query.
- Signals are serialized — a subscriber sees events one at a time.

---

## 2. R2DBC Fundamentals

### 2.1 What R2DBC Is

**R2DBC (Reactive Relational Database Connectivity)** is a specification — an SPI, not a library — that defines a fully non-blocking API for relational databases in Java. Version 1.0 of the spec was released in 2021.

Key facts:
- It is **not a JDBC replacement** in the "runtime" sense — they can coexist.
- It is an **SPI**: drivers implement it (e.g., `r2dbc-postgresql`), applications consume it (usually via Spring Data R2DBC).
- It is built on **Reactive Streams** (`org.reactivestreams.Publisher`).
- Maintained under the **R2DBC GitHub org** (originally incubated at Pivotal, now community + VMware).

### 2.2 How It Differs From JDBC

| Dimension | JDBC | R2DBC |
|---|---|---|
| Year | 1997 | 2018 (spec), 2021 (1.0) |
| Model | Blocking, thread-per-query | Non-blocking, event-driven |
| Return types | `ResultSet`, `int`, `boolean` | `Publisher<T>`, `Mono<T>`, `Flux<T>` |
| Transactions | `Connection.commit()` | Reactive `TransactionalOperator` / `@Transactional` |
| ORM support | JPA/Hibernate (mature) | Spring Data R2DBC (lightweight mapper, no ORM) |
| Lazy loading | JPA-level | Does not exist |
| Driver ecosystem | Every DB | Postgres, MySQL/MariaDB, MSSQL, Oracle, H2 |
| Vendor extensions | Rich | Varies by driver |

### 2.3 Supported Databases

Official / widely-used R2DBC drivers:

| Database | Driver artifact | Status |
|---|---|---|
| PostgreSQL | `io.r2dbc:r2dbc-postgresql` | First-class, most mature |
| MySQL | `io.asyncer:r2dbc-mysql` (fork of dev.miku) | Mature |
| MariaDB | `org.mariadb:r2dbc-mariadb` | Mature |
| MS SQL Server | `io.r2dbc:r2dbc-mssql` | Mature |
| Oracle | `com.oracle.database.r2dbc:oracle-r2dbc` | Maintained by Oracle |
| H2 | `io.r2dbc:r2dbc-h2` | Testing / dev only |
| Google Cloud Spanner | `com.google.cloud:spring-cloud-gcp-data-r2dbc-spanner` | GCP-specific |
| CockroachDB | Uses `r2dbc-postgresql` | Wire-compatible |
| YugabyteDB | Uses `r2dbc-postgresql` | Wire-compatible |

**No SQLite** (SQLite is fundamentally blocking). **No Cassandra / Mongo** — those are non-relational and have their own reactive drivers.

### 2.4 When to Use vs NOT Use R2DBC

**Use R2DBC when:**
- You are already on **Spring WebFlux** end-to-end and want a non-blocking bottom.
- You need **high-concurrency**, I/O-bound APIs with long-lived DB calls (streaming APIs, server-sent events, websockets, SSE dashboards).
- You need **streaming large result sets** with backpressure (exports, reports, ETL).
- You want to **minimize threads** (small container footprints, edge deployments).
- Your workload is genuinely I/O-bound and your team is comfortable with Reactor.

**Avoid R2DBC when:**
- Your team is primarily familiar with JPA and your data model is relational-rich (joins, graphs, inheritance).
- You rely on **JPA features**: lazy loading, dirty checking, cascade persistence, entity graphs, `@ManyToMany` join tables auto-managed.
- You need **complex mappings**: embeddables, inheritance, secondary tables, converters per column with bi-directional relationships.
- You run on **JDK 21+ with virtual threads** and your concurrency problem is already solved by Loom + JDBC.
- Your app is CRUD-heavy with moderate load — JPA productivity >> R2DBC savings.
- You need second-level caching — R2DBC has no equivalent.

**Rule of thumb:** R2DBC pays off when your system's bottleneck is **threads waiting on DB**, not **DB throughput itself**. If the DB is saturated, R2DBC won't save you.

### 2.5 Typical Use Cases

- High-concurrency REST or GraphQL APIs on WebFlux.
- Server-Sent Events / WebSocket feeds backed by change streams or `LISTEN/NOTIFY`.
- Data pipelines that stream rows through transformations.
- BFF (Backend-for-Frontend) layers that fan out to multiple downstreams.
- Edge services with tight memory budgets (k8s sidecars, serverless-like containers).

---

## 3. R2DBC Architecture & How It Works

### 3.1 The R2DBC SPI

The spec (package `io.r2dbc.spi`) defines a minimal contract:

```java
public interface ConnectionFactory {
    Publisher<? extends Connection> create();
    ConnectionFactoryMetadata getMetadata();
}

public interface Connection {
    Publisher<Void> beginTransaction();
    Publisher<Void> commitTransaction();
    Publisher<Void> rollbackTransaction();
    Statement createStatement(String sql);
    Batch createBatch();
    Publisher<Void> close();
    // ...
}

public interface Statement {
    Statement bind(int index, Object value);
    Statement bind(String name, Object value);
    Publisher<? extends Result> execute();
    // ...
}

public interface Result {
    Publisher<Long> getRowsUpdated();
    <T> Publisher<T> map(Function<Readable, T> mappingFunction);
}
```

Every return type is a `Publisher`. Nothing blocks.

### 3.2 Drivers

A driver (e.g., `r2dbc-postgresql`) implements the SPI on top of a **non-blocking network library**, usually **Netty**. Concretely:

- An `EventLoopGroup` owns a small number of threads (typically `2 * cores`).
- Each connection is a `Channel` registered with an event loop.
- The Postgres wire protocol is parsed by Netty handlers.
- Query results are emitted as Reactive Streams signals.

This is fundamentally different from JDBC, where the thread *is* the execution context.

### 3.3 Connection Lifecycle

```
app code
  │  ConnectionFactory.create()
  ▼
pool (r2dbc-pool)
  │  ── checkout ──▶  Connection (Netty Channel)
  ▼
Connection.createStatement("SELECT ...")
  │
  ▼
Statement.execute()  ──▶ Publisher<Result>
  │
  ▼
Result.map(row -> ...)  ──▶ Publisher<T>
  │  subscribe / request(n)
  ▼
Rows flow as onNext signals; onComplete terminates the stream
  │
  ▼
Connection returned to the pool (auto-closed when the Flux completes)
```

Key points:
- **Connections are not cheap** — you still need a pool (see §3.5).
- **Statement execution is lazy**: nothing hits the wire until `subscribe()`.
- **Cancellation**: unsubscribing mid-stream sends a cancel signal; the driver tries to stop the query.
- A connection is **returned to the pool** when the `Publisher<Result>` terminates (or cancels).

### 3.4 Reactive Query Execution Flow (PostgreSQL)

```
Spring Data R2DBC
    │  "SELECT * FROM users WHERE id = $1"
    ▼
DatabaseClient → Statement → bind($1, 42)
    │
    ▼
r2dbc-postgresql driver
    │  encodes Parse/Bind/Execute extended-protocol messages
    ▼
Netty Channel (TCP to Postgres)
    │  non-blocking write
    ▼
Postgres server processes query, streams DataRow frames
    │
    ▼
Netty read loop parses frames → onNext(Row)
    │
    ▼
Spring Data maps Row → Entity
    │
    ▼
Flux<User> emitted to application code
```

No thread is blocked anywhere. A handful of Netty threads can service tens of thousands of in-flight queries.

### 3.5 Connection Pooling (`r2dbc-pool`)

Even though R2DBC is non-blocking, **connections remain a limited resource**:
- The database has a max-connections limit (PostgreSQL default: 100).
- TCP handshakes and TLS are expensive.
- Authentication is per-connection.

`r2dbc-pool` provides a reactive connection pool:

```java
ConnectionFactory base = ConnectionFactories.get(
    ConnectionFactoryOptions.builder()
        .option(DRIVER, "postgresql")
        .option(HOST, "db")
        .option(PORT, 5432)
        .option(USER, "app")
        .option(PASSWORD, "secret")
        .option(DATABASE, "app")
        .build()
);

ConnectionPoolConfiguration poolConf = ConnectionPoolConfiguration.builder(base)
    .initialSize(5)
    .maxSize(20)
    .maxIdleTime(Duration.ofMinutes(10))
    .maxLifeTime(Duration.ofMinutes(30))
    .acquireRetry(2)
    .validationQuery("SELECT 1")
    .build();

ConnectionFactory pool = new ConnectionPool(poolConf);
```

Pool size guidance: start at `CPU_cores * 2` to `DB_max_connections / replicas`. Do **not** set it as high as JDBC pools — with R2DBC, concurrency is not bounded by pool size; the pool is just protecting the DB.

### 3.6 How Non-Blocking DB Access Is Actually Achieved

1. The **TCP socket** is set to non-blocking mode.
2. **Netty** registers the socket with an epoll/kqueue selector.
3. When `SELECT` is sent, the write completes without blocking.
4. The calling thread returns to the event loop.
5. When the DB replies, the OS notifies the selector.
6. Netty dispatches the data; the R2DBC driver decodes rows.
7. Each row triggers `onNext` on the subscriber chain.
8. Backpressure: if the subscriber hasn't requested more, the driver stops reading from the socket → TCP window fills → Postgres stops sending → true end-to-end backpressure.

This last point is the real power: **pressure propagates all the way back to the database**.

---

## 4. R2DBC vs JPA/JDBC

### 4.1 Blocking vs Non-Blocking

| Scenario | JDBC (Tomcat) | R2DBC (Netty) |
|---|---|---|
| 1,000 concurrent slow queries (1s each) | 1,000 blocked threads, ~1 GB memory | ~10 threads, <100 MB memory |
| Thread-pool saturation | Likely | Unlikely |
| Query cancellation | Not cooperative | Cooperative via cancel signal |
| CPU efficiency | Low when I/O-bound | High |
| CPU efficiency when CPU-bound | Similar | Similar |

### 4.2 Transactions

- **JDBC / JPA**: thread-local transaction context (`TransactionSynchronizationManager`), nested calls implicitly join the same TX.
- **R2DBC**: no thread-locals — a transaction is bound to a **subscriber context** (Reactor's `Context`) and a specific `Connection`. See §7.

### 4.3 Lazy Loading

JPA's lazy loading is based on proxies and the Hibernate session. **R2DBC has no session, no proxies, no lazy loading.** You explicitly query related data as additional `Mono`/`Flux` calls.

```java
// JPA
@OneToMany(fetch = LAZY)
List<OrderLine> lines;  // loaded on first access

// R2DBC
Flux<Order> orders = orderRepo.findAll();
Flux<OrderWithLines> enriched = orders.flatMap(o ->
    lineRepo.findByOrderId(o.getId()).collectList().map(lines -> new OrderWithLines(o, lines))
);
```

This is explicit — which is actually good for reasoning about query counts.

### 4.4 ORM Capabilities

Spring Data R2DBC is **not an ORM**. It is a **data-mapper** (row ↔ entity) with a query DSL. Specifically, it does **not** provide:

- Entity graph / navigation by reference
- Cascading persistence
- Dirty checking
- Automatic join-table management for `@ManyToMany`
- First- or second-level caching
- JPQL
- `@Inheritance` strategies
- `@EntityListeners` lifecycle hooks (limited replacement via auditing)

It **does** provide:
- Annotation-based mapping (`@Table`, `@Column`, `@Id`)
- Repositories (`ReactiveCrudRepository`, `R2dbcRepository`)
- Derived query methods (`findByEmail`)
- `@Query` native queries
- Pagination, sorting
- Reactive auditing (`@CreatedDate`, `@LastModifiedDate`)

### 4.5 Trade-offs: Simplicity vs Control vs Performance

| Dimension | JDBC + JPA | JDBC + plain SQL | R2DBC (+ Spring Data R2DBC) |
|---|---|---|---|
| Productivity for CRUD | Highest | Low | Medium |
| Control over SQL | Low (abstracted) | Highest | High |
| Non-blocking | No | No | Yes |
| Lazy loading | Yes | N/A | No |
| Streaming large result sets | Awkward | Awkward | Native |
| Team familiarity | High | High | Lower |
| Ecosystem maturity | Highest | Highest | Growing |
| **Virtual threads (JDK 21+)** | Solves most concurrency issues | Same | Still wins for streaming/backpressure |

**The virtual-threads caveat:** JDK 21+ with virtual threads + JDBC gives you most of R2DBC's concurrency benefit with none of its cost in API complexity. After Loom, R2DBC's unique value narrows to:
- True **end-to-end backpressure** from DB to HTTP.
- Deep integration with existing WebFlux / Netty stacks.
- Use cases where you're building a **streaming data pipeline** (`Flux` all the way down).

If you're starting a new project in 2026 and your main driver was "fewer threads," **consider virtual threads first.**

---

## 5. Spring Data R2DBC

### 5.1 Setup

#### 5.1.1 Dependencies (Gradle, Spring Boot 3.x)

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-r2dbc'
    implementation 'org.springframework.boot:spring-boot-starter-webflux'

    // PostgreSQL R2DBC driver
    runtimeOnly 'org.postgresql:r2dbc-postgresql'

    // Optional: keep JDBC for Flyway/Liquibase migrations
    runtimeOnly 'org.postgresql:postgresql'
    implementation 'org.flywaydb:flyway-core'
    implementation 'org.flywaydb:flyway-database-postgresql'

    // Validation, testing
    implementation 'org.springframework.boot:spring-boot-starter-validation'
    testImplementation 'io.projectreactor:reactor-test'
    testImplementation 'org.testcontainers:postgresql'
    testImplementation 'org.testcontainers:r2dbc'
}
```

**Why Flyway/Liquibase use JDBC:** migration tools are schema-evolution tools that run at startup; they don't need to be reactive. Run migrations with JDBC, run the app with R2DBC.

#### 5.1.2 Configuration (`application.yml`)

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://db:5432/appdb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    pool:
      enabled: true
      initial-size: 5
      max-size: 20
      max-idle-time: 10m
      max-life-time: 30m
      validation-query: SELECT 1

  # JDBC URL only for Flyway
  flyway:
    url: jdbc:postgresql://db:5432/appdb
    user: ${DB_USER}
    password: ${DB_PASSWORD}
    locations: classpath:db/migration

  sql:
    init:
      mode: never    # never let Spring auto-run schema.sql in prod

logging:
  level:
    io.r2dbc.postgresql.QUERY: DEBUG        # logs each query
    io.r2dbc.postgresql.PARAM: DEBUG        # logs parameter bindings (careful: PII)
```

#### 5.1.3 Custom Connection Factory (if needed)

```java
@Configuration
@EnableR2dbcRepositories
@EnableR2dbcAuditing
public class R2dbcConfig extends AbstractR2dbcConfiguration {

    @Override
    @Bean
    public ConnectionFactory connectionFactory() {
        PostgresqlConnectionConfiguration conf = PostgresqlConnectionConfiguration.builder()
            .host("db")
            .port(5432)
            .database("appdb")
            .username(System.getenv("DB_USER"))
            .password(System.getenv("DB_PASSWORD"))
            .sslMode(SSLMode.REQUIRE)
            .preparedStatementCacheQueries(256)
            .build();

        ConnectionPoolConfiguration pool = ConnectionPoolConfiguration.builder(
                new PostgresqlConnectionFactory(conf))
            .maxSize(20)
            .initialSize(5)
            .maxIdleTime(Duration.ofMinutes(10))
            .maxLifeTime(Duration.ofMinutes(30))
            .validationQuery("SELECT 1")
            .build();

        return new ConnectionPool(pool);
    }

    @Bean
    public ReactiveAuditorAware<String> auditorAware() {
        return () -> ReactiveSecurityContextHolder.getContext()
            .map(ctx -> ctx.getAuthentication().getName())
            .defaultIfEmpty("system");
    }
}
```

### 5.2 Example 1 — Basic CRUD

**Entity:**

```java
@Table("users")
public class User {

    @Id
    private Long id;

    @Column("email")
    private String email;

    @Column("full_name")
    private String fullName;

    @CreatedDate
    @Column("created_at")
    private Instant createdAt;

    @LastModifiedDate
    @Column("updated_at")
    private Instant updatedAt;

    // getters / setters / constructors
}
```

**Important:** R2DBC uses `@Table` / `@Column` from `org.springframework.data.relational.core.mapping.*`, **not** `javax.persistence.*`. Do not mix JPA annotations — they will be silently ignored.

**Schema (`V1__init.sql`):**

```sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    email       VARCHAR(255) NOT NULL UNIQUE,
    full_name   VARCHAR(255) NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL,
    updated_at  TIMESTAMPTZ NOT NULL
);
CREATE INDEX idx_users_email ON users(email);
```

**Repository:**

```java
public interface UserRepository extends R2dbcRepository<User, Long> {
    Mono<User> findByEmail(String email);
    Flux<User> findByFullNameContainingIgnoreCase(String q);
}
```

**Service:**

```java
@Service
public class UserService {

    private final UserRepository users;

    public UserService(UserRepository users) { this.users = users; }

    public Mono<User> create(User u) {
        return users.save(u);   // INSERT because id == null
    }

    public Mono<User> getById(Long id) {
        return users.findById(id)
            .switchIfEmpty(Mono.error(new ResponseStatusException(NOT_FOUND, "user")));
    }

    public Mono<User> update(Long id, User patch) {
        return users.findById(id)
            .switchIfEmpty(Mono.error(new ResponseStatusException(NOT_FOUND, "user")))
            .flatMap(existing -> {
                existing.setFullName(patch.getFullName());
                return users.save(existing);
            });
    }

    public Mono<Void> delete(Long id) {
        return users.deleteById(id);
    }
}
```

**How it works:**
- `save()` executes `INSERT ... RETURNING *` if the `@Id` is null; else `UPDATE`.
- The `Mono<User>` does **nothing** until subscribed — WebFlux subscribes when the response is written.
- `findById` returns an **empty** `Mono` if no row; `switchIfEmpty` converts that into a 404.

### 5.3 Example 2 — Reactive Repository Extras

```java
public interface UserRepository extends R2dbcRepository<User, Long> {

    Flux<User> findAllByOrderByCreatedAtDesc();

    Mono<Long> countByCreatedAtAfter(Instant since);

    @Query("SELECT * FROM users WHERE email ILIKE :pattern")
    Flux<User> search(@Param("pattern") String pattern);

    @Modifying
    @Query("UPDATE users SET full_name = :name WHERE id = :id")
    Mono<Integer> renameById(Long id, String name);
}
```

- **Derived queries** behave like Spring Data JPA — parsed from the method name.
- **`@Query`** is **native SQL** — R2DBC has no JPQL.
- **`@Modifying`** is required for `UPDATE`/`DELETE`.
- **Return types**: `Mono<T>` for 0..1, `Flux<T>` for 0..N, `Mono<Long>` or `Mono<Integer>` for counts and affected rows.

### 5.4 Example 3 — Custom Queries with `DatabaseClient`

For dynamic or multi-statement queries, use the lower-level `DatabaseClient`:

```java
@Repository
public class OrderSearchDao {

    private final DatabaseClient db;

    public OrderSearchDao(DatabaseClient db) { this.db = db; }

    public Flux<OrderSummary> searchOrders(String status, Instant since, int limit) {
        String sql = """
            SELECT o.id, o.status, o.placed_at, u.email
            FROM orders o JOIN users u ON u.id = o.user_id
            WHERE (:status IS NULL OR o.status = :status)
              AND o.placed_at >= :since
            ORDER BY o.placed_at DESC
            LIMIT :limit
        """;

        return db.sql(sql)
            .bind("status", status == null ? Parameter.empty(String.class) : Parameter.from(status))
            .bind("since", since)
            .bind("limit", limit)
            .map((row, meta) -> new OrderSummary(
                row.get("id", Long.class),
                row.get("status", String.class),
                row.get("placed_at", Instant.class),
                row.get("email", String.class)))
            .all();
    }
}
```

`DatabaseClient` is the idiomatic way to write complex SQL. It is lower-friction than `R2dbcEntityTemplate` and avoids the overhead of framework entity mapping for projections.

### 5.5 Example 4 — Mapping Entities

Spring Data R2DBC's mapper:
- Uses **constructor injection** if a matching constructor exists; otherwise setters.
- Uses `@Column("...")` for snake_case / custom names.
- Supports `@Embedded` for flat embeddables.
- Does **not** support `@OneToMany`, `@ManyToOne`, or `@ManyToMany`.
- Supports **custom converters** for types the driver doesn't map.

Example converter:

```java
@ReadingConverter
public class JsonToAddressConverter implements Converter<Json, Address> {
    private final ObjectMapper om;
    public JsonToAddressConverter(ObjectMapper om) { this.om = om; }
    public Address convert(Json src) {
        try { return om.readValue(src.asString(), Address.class); }
        catch (Exception e) { throw new IllegalStateException(e); }
    }
}

@Configuration
public class ConvertersConfig {
    @Bean
    public R2dbcCustomConversions conversions(ObjectMapper om) {
        return R2dbcCustomConversions.of(PostgresDialect.INSTANCE,
            new JsonToAddressConverter(om),
            new AddressToJsonConverter(om));
    }
}
```

For **aggregates with children**, model them explicitly — fetch children in a second query and combine with `flatMap`.

### 5.6 Example 5 — Pagination & Sorting

```java
public interface ProductRepository extends R2dbcRepository<Product, Long> {
    Flux<Product> findByCategoryId(Long categoryId, Pageable pageable);
    Mono<Long> countByCategoryId(Long categoryId);
}
```

```java
public Mono<PageResult<Product>> list(Long categoryId, int page, int size) {
    Pageable p = PageRequest.of(page, size, Sort.by(Sort.Direction.DESC, "createdAt"));
    return repo.findByCategoryId(categoryId, p).collectList()
        .zipWith(repo.countByCategoryId(categoryId))
        .map(t -> new PageResult<>(t.getT1(), page, size, t.getT2()));
}
```

**Gotchas:**
- Spring Data R2DBC does **not** return `Page<T>` directly from repository methods because producing a `Page` requires a blocking `count` — you must run `count()` separately and zip results.
- Use `zipWith` to run them in parallel on the same connection pool.

### 5.7 Example 6 — Reactive Transactions

```java
@Service
public class MoneyTransferService {

    private final AccountRepository accounts;
    private final TransactionalOperator tx;

    public MoneyTransferService(AccountRepository accounts, TransactionalOperator tx) {
        this.accounts = accounts;
        this.tx = tx;
    }

    public Mono<Void> transfer(Long fromId, Long toId, BigDecimal amount) {
        Mono<Void> op = accounts.findById(fromId)
            .switchIfEmpty(Mono.error(new IllegalStateException("from missing")))
            .flatMap(from -> {
                if (from.getBalance().compareTo(amount) < 0)
                    return Mono.error(new IllegalStateException("insufficient funds"));
                from.setBalance(from.getBalance().subtract(amount));
                return accounts.save(from);
            })
            .then(accounts.findById(toId)
                .switchIfEmpty(Mono.error(new IllegalStateException("to missing")))
                .flatMap(to -> {
                    to.setBalance(to.getBalance().add(amount));
                    return accounts.save(to);
                }))
            .then();

        return tx.transactional(op);
    }
}
```

Or declaratively:

```java
@Transactional
public Mono<Void> transfer(Long from, Long to, BigDecimal amount) { ... }
```

**Crucial:** the `@Transactional` annotation must be on a method returning `Mono`/`Flux`. Spring binds the transaction to the **Reactor Context**, not the current thread. See §7 for the mechanics.

---

## 6. Reactive Query Patterns

### 6.1 Streaming Large Datasets

```java
@GetMapping(value = "/export/users", produces = MediaType.APPLICATION_NDJSON_VALUE)
public Flux<User> export() {
    return userRepository.findAll();   // streams row-by-row
}
```

- The response body is written as **NDJSON** as rows arrive.
- Backpressure: the HTTP client's read pace throttles DB reads.
- Memory is bounded — you never hold 100M rows.
- The connection to the DB is held open for the duration; plan for long-running queries (statement timeout).

### 6.2 Handling Backpressure

By default, Reactor operators request `Long.MAX_VALUE`. To introduce real backpressure:

```java
userRepository.findAll()
    .limitRate(256)          // request 256 at a time, then request 64 more when 192 consumed
    .buffer(Duration.ofSeconds(1), 500)
    .concatMap(batch -> processor.handle(batch))
    .then();
```

- `limitRate(n)` caps how much the upstream will prefetch.
- `buffer(time, size)` groups rows into micro-batches to amortize downstream cost.
- `concatMap` preserves ordering and processes one batch at a time — avoid `flatMap` unless you want parallelism.

### 6.3 Combining DB Calls

**Sequential (dependent):**

```java
userRepo.findByEmail(email)
    .flatMap(u -> orderRepo.findByUserId(u.getId()).collectList());
```

**Parallel (independent):**

```java
Mono<User> userM = userRepo.findById(id);
Mono<List<Order>> ordersM = orderRepo.findByUserId(id).collectList();

Mono<Profile> profileM = Mono.zip(userM, ordersM)
    .map(t -> new Profile(t.getT1(), t.getT2()));
```

**Fire-and-forget subscribe?** Don't. In a request-scoped flow, never call `.subscribe()` manually — return the `Mono`/`Flux` and let WebFlux subscribe. Calling `.subscribe()` detaches the operation from the request's lifecycle and loses error handling.

### 6.4 Parallel vs Sequential

```java
// concatMap: sequential, preserves order, bounded to 1 in-flight
Flux.fromIterable(ids).concatMap(repo::findById);

// flatMap: concurrent, unbounded (up to 256 default), unordered
Flux.fromIterable(ids).flatMap(repo::findById);

// flatMapSequential: concurrent, ordered
Flux.fromIterable(ids).flatMapSequential(repo::findById);

// flatMap with concurrency bound
Flux.fromIterable(ids).flatMap(repo::findById, 8);
```

**Rules of thumb:**
- Use `concatMap` when order matters or you want zero parallelism.
- Use `flatMap(fn, concurrency)` with an **explicit concurrency bound** to avoid saturating the pool.
- Default `flatMap` concurrency is 256 — enough to drain a pool of 20 connections many times over.

### 6.5 N+1 Awareness

R2DBC does not have lazy loading, which is a blessing (no hidden queries) but means you must design for N+1 manually:

```java
// BAD — N+1
Flux<Order> orders = orderRepo.findAll();
orders.flatMap(o -> lineRepo.findByOrderId(o.getId()).collectList()
    .map(lines -> new OrderDto(o, lines)));

// BETTER — bulk-fetch children
orderRepo.findAll().collectList()
    .flatMap(orders -> {
        List<Long> ids = orders.stream().map(Order::getId).toList();
        return lineRepo.findByOrderIdIn(ids).collectMultimap(OrderLine::getOrderId)
            .map(map -> orders.stream()
                .map(o -> new OrderDto(o, map.getOrDefault(o.getId(), List.of())))
                .toList());
    });
```

---

## 7. Transactions in R2DBC

### 7.1 How Reactive Transactions Work

In JDBC, a transaction is bound to a `ThreadLocal`. In R2DBC, Spring binds it to the **Reactor Context** — a key-value map that flows along the subscriber chain.

When `@Transactional` wraps a reactive pipeline:

1. `TransactionalOperator` (or `@Transactional` proxy) intercepts the `Mono`/`Flux`.
2. It obtains a connection from the pool.
3. It issues `BEGIN`.
4. The connection is stored in the subscriber context.
5. All downstream operations **that look up a connection from the context** run on the same connection.
6. On `onComplete` → `COMMIT`; on `onError` → `ROLLBACK`.
7. The connection is released.

### 7.2 Differences From Traditional Transactions

| Aspect | JDBC/JPA | R2DBC |
|---|---|---|
| Storage | `ThreadLocal` | Reactor `Context` |
| Propagation | REQUIRED / REQUIRES_NEW / NESTED etc. | Limited — effectively REQUIRED; NESTED/REQUIRES_NEW support is thin |
| Savepoints | Well supported | Supported at SPI level; driver-dependent |
| Timeout | `@Transactional(timeout=...)` | Supported; behavior varies |
| Read-only hints | Yes | Yes (`@Transactional(readOnly = true)`) |
| Thread-hopping | N/A | Safe — context follows the subscriber |

### 7.3 Limitations

- **No mixing** of blocking and reactive transactions in a single logical operation. If you must call JDBC mid-flow, use `TransactionSynchronizationManager` awareness — but generally, don't.
- **Propagation**: `REQUIRES_NEW` is implemented but some drivers handle it imperfectly. Test.
- **Nested savepoints**: functional but not always as polished as JPA.
- **Cross-service transactions (XA)**: not practical. Use sagas.

### 7.4 Best Practices

1. **Declare transactions at the service layer**, not in controllers or repos.
2. **Read-only transactions** for pure read paths (`@Transactional(readOnly = true)`):
   ```java
   @Transactional(readOnly = true)
   public Mono<Dashboard> loadDashboard(Long userId) { ... }
   ```
3. **Keep transactions short.** Every TX pins a connection for its duration.
4. **Avoid mixing `.subscribe()` inside a transactional method** — you'll lose the context.
5. **Never call `.block()`** from a reactive transaction.
6. **Validate before writing.** Re-querying inside a transaction is fine, but do it upfront to avoid long rollbacks.
7. **Idempotency keys** beat distributed transactions in microservice flows.

### 7.5 Explicit `TransactionalOperator`

When you need fine-grained control:

```java
public Mono<Receipt> checkout(Cart cart) {
    return tx.execute(status ->
        reserveStock(cart)
            .then(createOrder(cart))
            .flatMap(order -> chargePayment(order)
                .doOnError(e -> status.setRollbackOnly())
                .thenReturn(order))
            .flatMap(this::emitOrderPlacedEvent)
    ).single();
}
```

`TransactionalOperator` is preferred over `@Transactional` when you're composing operators dynamically or need programmatic rollback.

---

## 8. R2DBC + WebFlux Integration

### 8.1 End-to-End Reactive Flow

```
HTTP request
   │   Netty event loop
   ▼
@RestController (returns Mono/Flux)
   │
   ▼
Service (composes Monos/Fluxes)
   │
   ▼
R2DBC Repository (returns Mono/Flux)
   │
   ▼
r2dbc-postgresql driver (Netty Channel)
   │
   ▼
PostgreSQL
```

**No thread is ever blocked on I/O.**

### 8.2 End-to-End Example

**Controller:**

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;
    public UserController(UserService svc) { this.userService = svc; }

    @PostMapping
    public Mono<ResponseEntity<User>> create(@Valid @RequestBody Mono<CreateUserRequest> body) {
        return body
            .map(req -> new User(null, req.email(), req.fullName(), null, null))
            .flatMap(userService::create)
            .map(u -> ResponseEntity.created(URI.create("/api/users/" + u.getId())).body(u));
    }

    @GetMapping("/{id}")
    public Mono<User> get(@PathVariable Long id) {
        return userService.getById(id);
    }

    @GetMapping(produces = MediaType.APPLICATION_NDJSON_VALUE)
    public Flux<User> stream() {
        return userService.streamAll();
    }
}
```

**Service:**

```java
@Service
public class UserService {

    private final UserRepository repo;
    private final EventPublisher events;

    public UserService(UserRepository repo, EventPublisher events) {
        this.repo = repo;
        this.events = events;
    }

    @Transactional
    public Mono<User> create(User u) {
        return repo.findByEmail(u.getEmail())
            .flatMap(existing -> Mono.<User>error(new DuplicateEmailException(u.getEmail())))
            .switchIfEmpty(Mono.defer(() -> repo.save(u)))
            .flatMap(saved -> events.publish(new UserCreated(saved.getId())).thenReturn(saved));
    }

    @Transactional(readOnly = true)
    public Mono<User> getById(Long id) {
        return repo.findById(id)
            .switchIfEmpty(Mono.error(new ResponseStatusException(HttpStatus.NOT_FOUND)));
    }

    public Flux<User> streamAll() {
        return repo.findAll();
    }
}
```

### 8.3 Error Handling Properly

**Global handler:**

```java
@ControllerAdvice
public class GlobalErrorHandler {

    @ExceptionHandler(DuplicateEmailException.class)
    public ResponseEntity<ErrorBody> handleDup(DuplicateEmailException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorBody("duplicate_email", e.getMessage()));
    }

    @ExceptionHandler(DataIntegrityViolationException.class)
    public ResponseEntity<ErrorBody> handleIntegrity(DataIntegrityViolationException e) {
        return ResponseEntity.status(HttpStatus.BAD_REQUEST)
            .body(new ErrorBody("integrity_violation", "Constraint violation"));
    }
}
```

**In-flow error handling:**

```java
userRepository.save(u)
    .onErrorMap(DuplicateKeyException.class, e -> new DuplicateEmailException(u.getEmail()))
    .onErrorResume(TransientDataAccessException.class, e -> Mono.empty());
```

**Timeouts:**

```java
userRepository.findById(id)
    .timeout(Duration.ofSeconds(2))
    .onErrorMap(TimeoutException.class, e -> new ResponseStatusException(GATEWAY_TIMEOUT));
```

### 8.4 Avoiding Blocking Calls

The cardinal sin in WebFlux is blocking the event loop. Detection:

```java
// Add BlockHound in dev/test
dependencies {
    testImplementation 'io.projectreactor.tools:blockhound:1.0.8.RELEASE'
}
```

```java
@BeforeAll
static void setup() { BlockHound.install(); }
```

BlockHound throws if any Reactor thread calls a known blocking method.

**Common offenders:**
- `RestTemplate` → use `WebClient`.
- `JdbcTemplate` → use `DatabaseClient`.
- `Files.readAllBytes` → use `DataBufferUtils` with async channels.
- `Thread.sleep` → use `Mono.delay`.
- `.block()` anywhere → never.

**Legitimate blocking?** Offload:

```java
Mono.fromCallable(() -> legacyJdbc.query(...))
    .subscribeOn(Schedulers.boundedElastic());
```

---

## 9. Performance & Tuning

### 9.1 Connection Pooling

With R2DBC, concurrency is not limited by your pool — it's limited by the DB. Sizing philosophy:

```
max_pool_size = min(db_max_connections / replicas, app_cpu * 4)
```

**PostgreSQL default:** 100 connections. If you run 10 replicas, cap each pool at ~8. Add **PgBouncer** in front for real fan-out.

```yaml
spring:
  r2dbc:
    pool:
      initial-size: 5
      max-size: 20
      max-create-connection-time: 5s
      max-acquire-time: 10s
      max-idle-time: 10m
      max-life-time: 30m
      validation-query: SELECT 1
```

### 9.2 Throughput vs Latency

- R2DBC wins on **throughput under concurrency** — more requests per CPU.
- R2DBC does **not** make a single query faster. A 100 ms query is still 100 ms.
- For **latency-sensitive single-user paths**, JDBC with a small pool is fine — even preferable.

### 9.3 When R2DBC Improves Performance

| Workload | R2DBC benefit |
|---|---|
| High fan-out (BFF calling many services + DB) | ✅ large |
| Streaming large result sets | ✅ large |
| Many concurrent slow queries (e.g., 300 ms+) | ✅ large |
| Sparse, low-QPS admin APIs | ❌ negligible |
| CPU-bound computations over DB data | ❌ negligible |
| Large transactional writes | ❌ or neutral |

### 9.4 When R2DBC Does NOT Improve Performance

- Your DB is the bottleneck. Reactive clients can't make Postgres faster.
- Queries themselves are slow. Optimize indexes, not the client.
- Your app is CPU-bound. Reactor adds overhead.
- Your concurrency is low (<100 concurrent requests).

### 9.5 Benchmark Considerations

- **Measure under realistic concurrency** (1k+ concurrent requests). Low-concurrency benchmarks often favor JDBC because of Reactor overhead.
- **Warm the JVM** (30+ seconds) before measuring.
- **Measure the 99th percentile**, not the mean.
- **Match pool sizes fairly** — don't pit R2DBC with pool=10 against HikariCP with pool=200.
- **Measure end-to-end** including network + DB — client-side benchmarks lie.
- **Test both JDK 17 Tomcat** and **JDK 21 virtual threads** — Loom changes the comparison significantly.

### 9.6 Driver-Level Tuning (PostgreSQL)

```java
PostgresqlConnectionConfiguration.builder()
    .fetchSize(500)                           // rows per backend fetch
    .preparedStatementCacheQueries(256)        // server-side prepared stmts
    .tcpKeepAlive(true)
    .tcpNoDelay(true)
    .lockWaitTimeout(Duration.ofSeconds(5))
    .statementTimeout(Duration.ofSeconds(30))
    .build();
```

**`fetchSize`** matters for streaming — it's the unit of back-and-forth.

---

## 10. Production Best Practices

### 10.1 Avoiding Blocking Calls

- Install **BlockHound in tests** — fail fast.
- Code review rule: **no `.block()`, ever**, outside `main` or tests.
- Wrap any unavoidable blocking call with `subscribeOn(Schedulers.boundedElastic())`.
- Never call JDBC inside a reactive flow unless scheduled on `boundedElastic`.

### 10.2 Debugging Reactive DB Issues

- Enable R2DBC query logging:
  ```yaml
  logging:
    level:
      io.r2dbc.postgresql.QUERY: DEBUG
  ```
- Use **Reactor debug agent** in dev:
  ```java
  ReactorDebugAgent.init();
  ```
  Gives you assembly-time stack traces — invaluable when errors are thrown from deep chains.
- Use `checkpoint("name")` operators to attach location info:
  ```java
  repo.findById(id).checkpoint("find user by id");
  ```
- `Hooks.onOperatorDebug()` in dev only (expensive in prod).
- Micrometer metrics: `r2dbc.pool.acquired`, `r2dbc.pool.pending`, `r2dbc.pool.idle`.

### 10.3 Logging Strategies

- Use **MDC via Reactor Context**:
  ```java
  Mono.deferContextual(ctx -> {
      String reqId = ctx.get("requestId");
      MDC.put("requestId", reqId);
      try { /* work */ } finally { MDC.remove("requestId"); }
      return repo.findById(id);
  });
  ```
- Prefer **structured (JSON) logs**.
- Log **query parameters only in dev** — production param logging is a PII leak.
- Log the **correlation ID** on every DB call in dev; sample in prod.

### 10.4 Error Handling

- Don't swallow `Mono.onErrorResume(e -> Mono.empty())` without logging — silent failures are nightmares in reactive code.
- Map driver-specific exceptions to domain exceptions.
- Use `retryWhen` with backoff for transient errors:
  ```java
  .retryWhen(Retry.backoff(3, Duration.ofMillis(100))
      .filter(TransientDataAccessException.class::isInstance));
  ```

### 10.5 Resource Management

- Don't hand-manage connections. Let Spring's `DatabaseClient` or repositories do it.
- If using raw `ConnectionFactory`: always `usingWhen` to guarantee cleanup:
  ```java
  Flux.usingWhen(
      connectionFactory.create(),
      conn -> Flux.from(conn.createStatement("SELECT 1").execute())
                  .flatMap(r -> r.map((row, m) -> row.get(0, Integer.class))),
      Connection::close
  );
  ```
- Always set **statement timeouts** at the driver or SQL level.

### 10.6 Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Mixing JPA annotations with R2DBC | Fields silently not mapped | Use `org.springframework.data.relational.*` |
| Calling `.block()` in a controller | Blocked event loop, cascading failures | Return `Mono`/`Flux` |
| Forgetting `@Modifying` on update `@Query` | Query silently returns nothing | Add `@Modifying` |
| Expecting `@ManyToOne` to work | Field is null | Fetch relation explicitly |
| Running `count()` inline | Extra round-trip on every request | Use `zipWith` and cache where possible |
| Using `flatMap` unbounded | Pool exhaustion, DB overload | Pass concurrency argument |
| Subscribing manually in controllers | Lost errors, disconnected lifecycle | Return publisher from controller |
| Oversized pool | DB max_connections exceeded | Size for DB, not for app |
| Using `findAll().collectList()` on 10M rows | OOM | Stream with `findAll()` + NDJSON |
| Not enabling statement timeout | Runaway queries pin connections forever | Set `statementTimeout` |

---

## 11. R2DBC in System Design

### 11.1 High-Concurrency APIs

**Scenario:** 50k concurrent SSE subscribers per node receiving live updates from Postgres `LISTEN/NOTIFY`.

```
Browser ────SSE────▶ WebFlux app ────R2DBC LISTEN────▶ PostgreSQL
                                     ▲
                                     └─ Netty event loop (4 threads)
```

With JDBC this is infeasible — 50k blocking threads. With R2DBC, a handful of threads handle all subscribers.

### 11.2 Streaming / Data-Intensive Systems

**Scenario:** export 500M rows as NDJSON to S3.

```
R2DBC Flux<Row>
   │  limitRate(1000)
   ▼
Mapper Flux<Json>
   │  buffer(5000)
   ▼
S3 multipart upload (non-blocking)
```

End-to-end backpressure: S3 uploader throttles buffer → throttles mapper → throttles DB reads.

### 11.3 Microservices With Reactive DB Access

```
[API Gateway]
     │
     ├─▶ [catalog-svc] ─ R2DBC ─▶ Postgres (catalog)
     ├─▶ [order-svc]   ─ R2DBC ─▶ Postgres (orders)
     └─▶ [inventory]   ─ R2DBC ─▶ Postgres (inventory)
                     └─ Kafka (reactive) ─▶ order-svc
```

**Benefits:**
- Each service scales on threads cheaply.
- Event-driven pipelines compose naturally with `Flux`.
- A slow downstream does not saturate the upstream's thread pool — it just backpressures.

**Failure modes:**
- **Cascading backpressure**: a slow DB causes HTTP response latency to rise; the client sees timeouts. Add circuit breakers (`Resilience4j` reactive).
- **Connection-pool contention**: under partial DB outage, pool acquire waits may exceed request timeout. Set aggressive `maxAcquireTime`.
- **Silent subscription leaks**: forgotten `subscribe()` calls with errors get swallowed. Use `Hooks.onErrorDropped` to log them.

### 11.4 Trade-offs at the Architecture Level

- You **commit your entire stack** to reactive. Every library in the request path must be non-blocking.
- **Debugging is harder** — stack traces span operators, not methods.
- **Hiring / onboarding is harder** — Reactor fluency takes months.
- **Observability tooling** is less mature for reactive than for classical MVC.

---

## 12. Real-World Scenarios

### 12.1 High-Load Public API

**Context:** public REST API serving 30k RPS, 95% reads, p99 < 100 ms.

- Spring WebFlux + Spring Data R2DBC + Postgres (read replicas).
- Pool size 20 per replica; 5 app instances; PgBouncer in transaction mode.
- Redis reactive cache in front.
- Micrometer → Prometheus; Grafana dashboards for `r2dbc.pool.*`.

**Why R2DBC:**
- Read traffic is bursty — hundreds of concurrent slow queries during spikes.
- A JDBC deployment was hitting 1k-thread pools and running out of memory.
- After migration, thread count dropped 10× and p99 improved under load.

**Lessons learned:**
- Writes were rewritten carefully; `@Transactional` needed explicit context propagation for the audit trail.
- Had to replace an old JPA soft-delete feature with explicit partial-unique indexes.

### 12.2 Streaming Analytics Service

**Context:** internal service exporting 100M–1B row reports to Parquet.

- `Flux<Row>` from R2DBC → Arrow batches → Parquet writer → S3.
- Pool size 4 (only 4 concurrent reports at a time).
- Statement timeouts set to 30 min; monitored via Prometheus.

**Why R2DBC:**
- End-to-end backpressure prevents OOM on very wide rows.
- `fetchSize(1000)` ensures DB streams efficiently.
- Single process handles large jobs with <2 GB heap.

### 12.3 Event-Driven Microservices Platform

**Context:** e-commerce platform, 12 services, all on WebFlux + R2DBC + Kafka (reactive).

- Each service has its own Postgres.
- `LISTEN/NOTIFY` used internally for cache invalidation.
- `ReactiveKafkaConsumer` + R2DBC for idempotent event processing.

**Why R2DBC:**
- Uniform reactive stack reduces cognitive load across services.
- Backpressure from DB naturally throttles Kafka consumption.
- No thread explosion during traffic spikes.

### 12.4 BFF (Backend-for-Frontend) for a Dashboard

**Context:** fan-out aggregation — each request hits the DB 4–10 times + 2–3 downstreams.

```java
public Mono<Dashboard> load(Long userId) {
    Mono<User> u = userRepo.findById(userId);
    Mono<List<Order>> o = orderRepo.findByUserId(userId).collectList();
    Mono<Metrics> m = metricsClient.get(userId);
    Mono<List<Alert>> a = alertsRepo.findActiveFor(userId).collectList();
    return Mono.zip(u, o, m, a)
        .map(t -> new Dashboard(t.getT1(), t.getT2(), t.getT3(), t.getT4()));
}
```

**Why R2DBC:**
- Parallel fan-out is idiomatic.
- Without R2DBC, each `zip` branch would pin a thread.

### 12.5 Real-Time Notification Service

**Context:** push notifications + in-app feeds for 10M DAU.

- WebSocket channels per user → reactive repositories for feed state.
- Postgres `LISTEN` used for server-to-server cache invalidation.
- 20k concurrent websockets per node on a 2-core container.

**Why R2DBC:** feasible on a small footprint. JDBC would require ~100× the threads.

---

## 13. Comparisons & Trade-offs

### 13.1 R2DBC vs JPA/Hibernate

| Dimension | R2DBC (+ Spring Data R2DBC) | JPA/Hibernate |
|---|---|---|
| Non-blocking | Yes | No |
| ORM | Mapper only | Full ORM |
| Relationships | Manual | `@OneToMany` etc. |
| Lazy loading | No | Yes |
| Query language | Native SQL | JPQL + native |
| Caching | None | 1st + 2nd-level |
| Dirty checking | No | Yes |
| Schema evolution | Flyway/Liquibase (JDBC) | Flyway/Liquibase or `ddl-auto` |
| Complex queries | `DatabaseClient` | Criteria API / JPQL |
| Learning curve | Medium + Reactor | Medium |
| Maturity | Growing | Entrenched |

### 13.2 R2DBC vs JDBC

| Dimension | R2DBC | JDBC |
|---|---|---|
| API | Reactive | Imperative |
| Thread model | Event-loop | Thread-per-query |
| Virtual threads (JDK 21+) | N/A (already non-blocking) | JDBC + VT competes closely |
| Streaming | First-class | Requires `fetchSize` tuning |
| Ecosystem | Smaller | Universal |
| Cancellation | Cooperative | Non-cooperative / `Statement.cancel()` |
| Transactions | Context-bound | Thread-local |

### 13.3 When R2DBC Is a Bad Choice

- Your team lives in JPA. The retraining cost alone may exceed the performance gains.
- You need multi-DB transactions (XA) — not supported.
- You need mature ORM features: graphs, inheritance, complex mappings.
- You are on **JDK 21+ with virtual threads** and most pain is already solved.
- Your load is low (<100 concurrent requests); Reactor overhead will hurt you.
- Your DB is the bottleneck; the client choice is irrelevant.
- You rely on MyBatis / jOOQ / Hibernate-specific features with no reactive equivalent.
- You need SQLite (no R2DBC driver) or a DB without a mature driver.

---

## 14. Final Checklist

### You are production-ready with R2DBC if you can:

- [ ] Explain **why JDBC blocks** and **how R2DBC doesn't**, including the Netty event loop model.
- [ ] Justify choosing R2DBC over JDBC for a given workload — and honestly say "use JDBC + virtual threads" when that's the right call.
- [ ] Describe the **R2DBC SPI** and the role of drivers, pools, and Spring Data R2DBC.
- [ ] Write a full CRUD service in **Spring Data R2DBC** with derived queries, `@Query`, and `DatabaseClient`.
- [ ] Implement **reactive transactions** using `@Transactional` and `TransactionalOperator`, and explain context propagation.
- [ ] Stream **millions of rows** with **end-to-end backpressure** from DB through HTTP.
- [ ] Use `zipWith`, `flatMap(fn, concurrency)`, and `concatMap` correctly, and know the default `flatMap` concurrency pitfall.
- [ ] Avoid **N+1** by bulk-fetching children and combining in memory.
- [ ] Prevent **blocking calls** in a WebFlux pipeline using **BlockHound** and `Schedulers.boundedElastic()`.
- [ ] Size a **connection pool** correctly for R2DBC — based on DB limits, not app concurrency.
- [ ] Configure **statement timeouts, TLS, prepared-statement caches**, and driver-level tuning for the PostgreSQL R2DBC driver.
- [ ] Run **Flyway** for schema migrations alongside a pure-R2DBC runtime.
- [ ] Debug with **Reactor's debug agent**, `checkpoint()`, and structured logs propagated via the Reactor `Context`.
- [ ] Identify workloads where R2DBC will **not** help (low-concurrency, CPU-bound, DB-bottlenecked).
- [ ] Articulate the **trade-offs** of committing an entire system to a reactive stack to a skeptical architect in under 5 minutes.

---

*End of guide. Save as `r2dbc-learning-guide.md`.*
