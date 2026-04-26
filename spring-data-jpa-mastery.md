# Spring Data JPA Mastery — From Mid-Level to Senior

> A production-grade guide for Java backend developers who already use Spring Data JPA but want to understand it the way an architect does: what happens behind the scenes, why it breaks under load, and when to reach for something else.

---

## Table of Contents

1. [JPA & Hibernate Fundamentals](#1-jpa--hibernate-fundamentals)
2. [Persistence Context](#2-persistence-context)
3. [Entity Mapping Deep Dive](#3-entity-mapping-deep-dive)
4. [Fetching Strategies](#4-fetching-strategies)
5. [Queries](#5-queries)
6. [Spring Data JPA Repositories](#6-spring-data-jpa-repositories)
7. [Transactions](#7-transactions)
8. [Performance Optimization](#8-performance-optimization)
9. [Edge Cases & Pitfalls](#9-edge-cases--pitfalls)
10. [Concurrency & Locking](#10-concurrency--locking)
11. [Pagination & Sorting](#11-pagination--sorting)
12. [DTO Projections](#12-dto-projections)
13. [Advanced Hibernate Features](#13-advanced-hibernate-features)
14. [Debugging & Monitoring](#14-debugging--monitoring)
15. [When NOT to Use JPA](#15-when-not-to-use-jpa)
16. [Real-World Scenarios](#16-real-world-scenarios)
17. [Final Checklist](#17-final-checklist)

---

# 1. JPA & Hibernate Fundamentals

## 1.1 JPA vs Hibernate

**JPA (Jakarta Persistence API)** is a *specification* — a set of interfaces, annotations, and behavior contracts (`EntityManager`, `@Entity`, `@OneToMany`, etc.). JPA doesn't execute anything; it defines what a compliant ORM must do.

**Hibernate** is an *implementation* of that specification (the dominant one). It also offers its own non-standard extensions (`@BatchSize`, `@Filter`, `Session`, etc.) that JPA doesn't define.

**Spring Data JPA** is *not* an ORM. It's an abstraction layer on top of JPA that generates repository implementations, derives queries from method names, and integrates with Spring's transaction management.

```
Your code
   │
   ▼
Spring Data JPA  (repositories, query methods, paging)
   │
   ▼
JPA API          (EntityManager, @Entity — the specification)
   │
   ▼
Hibernate        (the actual ORM engine, session, caches)
   │
   ▼
JDBC driver      (SQL over the wire)
   │
   ▼
Database         (Postgres/MySQL/etc.)
```

**Why it matters:** understanding the layers tells you where a problem lives. A lazy-loading exception is a Hibernate + persistence-context issue, not a Spring Data issue. A derived query limitation is a Spring Data issue, not a JPA limitation.

## 1.2 What ORM Actually Does

Object-Relational Mapping solves the **impedance mismatch** between:

| Object world | Relational world |
|--------------|------------------|
| Objects with identity by reference | Rows with identity by PK |
| Inheritance, polymorphism | Flat tables |
| Navigable graphs (`order.getCustomer().getAddress()`) | Joins |
| Mutable state + encapsulation | Set-based operations |

ORM's job is to let you manipulate objects and transparently translate that into SQL. Internally, Hibernate does three things:

1. **Tracks** which entities you've loaded (the persistence context).
2. **Detects** changes you make to those entities (dirty checking).
3. **Flushes** those changes to SQL at the right moment.

This is powerful — and dangerous. Every major JPA bug you'll hit comes from misunderstanding one of these three mechanisms.

## 1.3 When NOT to Use JPA

JPA is optimized for **OLTP workloads with a domain model** — short transactions, graphs of entities, dozens to hundreds of rows per request.

**Avoid JPA when:**

- **Bulk data processing.** Inserting 10M rows through the persistence context is catastrophic. Use JDBC batch, COPY, or Spring Batch.
- **Complex analytical queries.** Window functions, CTEs, `LATERAL JOIN`, `UNION`, pivots — JPQL struggles or can't. Use native SQL, jOOQ, or views.
- **Write-heavy systems with predictable SQL.** When you know your SQL exactly, ORM overhead is pure cost.
- **Read models that don't need object graphs.** Reporting, dashboards, search — DTO projections or raw SQL win.
- **Highly denormalized NoSQL-like access.** ORMs don't help when there are no relations.

**Rule of thumb:** JPA for the write side and domain logic; lean SQL for the read side when performance matters. Many mature backends run a hybrid: JPA for mutations, jOOQ/Spring JDBC for reports.

---

# 2. Persistence Context

The persistence context is the **single most important concept** in JPA. Miss it and nothing makes sense.

## 2.1 What Is the Persistence Context?

It's an **in-memory map** (`Map<EntityKey, Object>`) managed by `EntityManager`, scoped to a transaction by default.

Two guarantees it provides:

1. **Identity guarantee:** within one persistence context, `em.find(User.class, 1L) == em.find(User.class, 1L)` — always the same object reference. This is the *first-level cache*.
2. **Change tracking:** any entity loaded through it is watched for changes.

At transaction commit (or explicit flush), Hibernate computes the diff between current state and the snapshot it took when the entity was loaded, and emits `UPDATE` statements for what changed. This is **automatic dirty checking** — you never call `save()` on a managed entity.

```java
@Transactional
public void renameUser(Long id, String newName) {
    User u = userRepo.findById(id).orElseThrow();
    u.setName(newName);
    // No save() call. At commit, Hibernate issues UPDATE user SET name=? WHERE id=?
}
```

If you find yourself calling `save()` on an entity you just loaded, that's a sign you don't yet trust (or understand) the persistence context.

## 2.2 Entity States

Every entity exists in exactly one of four states:

```
                    persist()
   ┌───────────┐ ───────────────▶ ┌───────────┐
   │ TRANSIENT │                  │ PERSISTENT│ ◀──┐
   └───────────┘                  └───────────┘    │
         ▲                              │          │ merge()
         │                      detach()│ remove() │
         │                              ▼          │
         │                        ┌───────────┐    │
         │                        │  DETACHED │────┘
         │                        └───────────┘
         │                              │
         │                              ▼
         │                        ┌───────────┐
         └──────────────────────  │  REMOVED  │
                                  └───────────┘
```

### TRANSIENT (new)
- Just instantiated with `new`. Not known to any persistence context. No ID (usually).
- Not tracked. Changes do nothing.

### PERSISTENT (managed)
- In the persistence context. Tracked. Has an ID.
- Dirty checking applies. At flush, changes are written.

### DETACHED
- Was persistent, but the persistence context closed (transaction ended) or it was explicitly detached.
- Has an ID but is no longer tracked.
- **This is the source of most production JPA bugs.**

### REMOVED
- `entityManager.remove()` or Spring's `delete()` was called.
- Still in the persistence context; will be `DELETE`d on flush.

### Lifecycle example

```java
@Transactional
public void demo() {
    User u = new User("Alice");          // TRANSIENT
    em.persist(u);                        // PERSISTENT (INSERT queued)
    u.setName("Alicia");                  // dirty; will UPDATE at flush
    em.flush();                           // INSERT then UPDATE sent to DB
    em.detach(u);                         // DETACHED
    u.setName("ignored");                 // no effect — not tracked
    User managed = em.merge(u);           // returns a PERSISTENT copy
    em.remove(managed);                   // REMOVED
}                                         // commit: DELETE flushed, tx closed
```

## 2.3 Dirty Checking — How It Works Internally

When an entity becomes managed, Hibernate takes a **snapshot** of all its persistent fields (primitive values, references, collections). At flush, for every managed entity:

1. Compare current field values to the snapshot.
2. Build a column-level diff.
3. Emit `UPDATE ... SET <changed columns> WHERE id=? [AND version=?]`.

**Implications:**

- It's O(N × fields) per flush where N = managed entities. Huge persistence contexts (thousands of entities) = slow flushes.
- Even reads with mutations allocate snapshot memory.
- Changes to **transient fields** (`@Transient`) are ignored.
- Changes to **non-mapped fields** are ignored.
- `hibernate.jdbc.batch_size` controls batching of these UPDATEs.

## 2.4 Flush Mechanism

Flush = "sync in-memory state to the database" — push SQL, don't commit yet.

### When flush happens
1. **Before query execution** (default: `FlushModeType.AUTO`) — because the query might otherwise return stale data.
2. **On `em.flush()`** — explicit.
3. **On transaction commit.**

### Order of operations at flush
Hibernate orders SQL internally for FK consistency:
1. `INSERT`s (in dependency order)
2. `UPDATE`s
3. Collection element deletes
4. Collection element inserts
5. `DELETE`s

This is why you occasionally see odd ordering vs the order of your Java calls.

### Flush modes
- `AUTO` (default) — flush before queries that might read modified state. Safe.
- `COMMIT` — flush only at commit. Faster but queries may return stale data.
- `MANUAL` — flush only on explicit call. Advanced use cases.

## 2.5 First-Level Cache (Not a Performance Feature)

The persistence context *is* the first-level cache. It's **not** there to make your system faster — it's there to maintain identity and enable dirty checking.

**Consequences:**
- Within a transaction, repeated `findById(1L)` hits memory, not DB. Good.
- **Across transactions, nothing is cached** by default. Each transaction gets its own persistence context. Good — no stale data between requests.
- Memory grows with loaded entities. A transaction that loads 100K entities holds them all in memory until commit.

**Common pitfall:** treating the first-level cache as an app-level cache. It's not. Close the transaction → gone.

## 2.6 Common Pitfalls

- **Modifying a detached entity and expecting updates.** Detached = not tracked. Nothing happens. Use `merge()` to reattach.
- **Open Session In View (OSIV).** Spring Boot enables `spring.jpa.open-in-view=true` by default. It keeps the persistence context alive through view rendering, hiding lazy-loading bugs in development and blowing up in production. **Turn it off.**
- **Large persistence contexts.** Loading 100K entities in a loop without clearing → OutOfMemoryError + slow flush. Use `em.clear()` periodically or stream-based processing.
- **Calling `save()` on a managed entity.** Unnecessary, redundant, and occasionally masks bugs.

```yaml
# application.yml — turn this OFF explicitly
spring:
  jpa:
    open-in-view: false
```

---

# 3. Entity Mapping Deep Dive

## 3.1 @Entity, @Table

```java
@Entity
@Table(name = "users",
       indexes = @Index(columnList = "email", unique = true))
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 255)
    private String email;
}
```

### Non-obvious rules
- Entity classes need a **no-arg constructor** (can be `protected`; Hibernate uses reflection).
- Entities must not be `final`; methods shouldn't be `final` (Hibernate subclasses for proxies).
- Avoid `record` — immutable, no setters, no proxying. Use only for DTOs, not entities.
- Annotating `@Entity` without `@Table` defaults to table name = class name (some vendors lowercase it; don't rely on defaults).

## 3.2 @Id Strategies

| Strategy | How it works | When to use | Pitfalls |
|----------|--------------|-------------|----------|
| `IDENTITY` | Uses DB auto-increment. PK assigned on INSERT. | MySQL, SQL Server | **Disables JDBC batch inserts.** Each INSERT is a round trip. |
| `SEQUENCE` | Hibernate calls `nextval`. PK assigned before INSERT. | PostgreSQL, Oracle | Best for batch inserts. Requires a sequence. |
| `TABLE` | Uses a separate ID table with row locking. | Portability only | Slow, avoid. |
| `AUTO` | Hibernate picks. | Prototypes | Opaque; don't use in prod. |
| `UUID` / assigned | You set the ID. | Distributed systems, UUIDs | Can hurt BTREE index locality — use UUIDv7 or ULID. |

### Postgres best practice

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "user_seq")
@SequenceGenerator(name = "user_seq", sequenceName = "user_seq", allocationSize = 50)
private Long id;
```

`allocationSize > 1` (pooled sequence) lets Hibernate pre-allocate IDs and batch inserts. Must match the `INCREMENT BY` of the sequence.

## 3.3 @Column

```java
@Column(
    name = "full_name",
    nullable = false,
    length = 200,
    unique = false,
    updatable = true,
    insertable = true,
    columnDefinition = "varchar(200)"
)
private String fullName;
```

- `@Column` is metadata for DDL generation AND for JPA behavior (`updatable=false` means "never UPDATE this column" — useful for created_at).
- **Don't rely on Hibernate DDL in prod.** Use Flyway or Liquibase. Set `spring.jpa.hibernate.ddl-auto=validate` (or `none`).

## 3.4 Relationships: Owning vs Inverse Side

The **single most important rule** of JPA relationships:

> **The owning side is the one with the foreign key column. Only changes on the owning side are persisted.**

For a `User (1) — (N) Order`, the `order.user_id` column lives on the `orders` table — so `Order` owns the relationship.

```java
@Entity
class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;      // owning side (has the FK)
}

@Entity
class User {
    @OneToMany(mappedBy = "user")   // inverse side — points back
    private List<Order> orders = new ArrayList<>();
}
```

If you only do `user.getOrders().add(order)` without `order.setUser(user)`, **nothing gets persisted** on the relationship — because the owning side (`Order.user`) wasn't set.

### Helper methods — the safe pattern

```java
public void addOrder(Order o) {
    orders.add(o);
    o.setUser(this);   // critical — set both sides
}

public void removeOrder(Order o) {
    orders.remove(o);
    o.setUser(null);
}
```

## 3.5 @OneToOne

```java
@OneToOne(fetch = FetchType.LAZY, cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "profile_id")
private Profile profile;
```

### Gotchas
- **`@OneToOne` is eagerly loaded by default**, regardless of `fetch = LAZY`, when the field is on the non-owning side (no FK). Hibernate can't know if the associated row exists without a query.
- Workaround: map it as optional owning side, or use `@OneToOne(optional = false)` + `@MapsId`, or simply avoid bidirectional `@OneToOne` — usually you can model it as `@ManyToOne` on one side.

### @MapsId — shared primary key

```java
@OneToOne
@MapsId
@JoinColumn(name = "id")
private User user;
```

Makes the profile's PK equal the user's PK. Clean, no surrogate FK. Only works for true 1:1 relationships.

## 3.6 @OneToMany / @ManyToOne

The 90% case. Always make `@ManyToOne` the owning side — it has the FK naturally.

```java
@Entity
class Order {
    @ManyToOne(fetch = FetchType.LAZY, optional = false)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
}
```

**Always `fetch = LAZY` on `@ManyToOne`.** The default is EAGER, which is a performance trap — every query for an Order silently joins or re-queries the user.

### @OneToMany without @ManyToOne (unidirectional)

Don't do this:

```java
@OneToMany
@JoinColumn(name = "user_id")     // unidirectional
private List<Order> orders;
```

It generates **extra UPDATE statements** to set FK after insert, because Hibernate doesn't know about the `@ManyToOne` side. Always prefer bidirectional with `mappedBy`.

## 3.7 @ManyToMany

`@ManyToMany` requires a join table.

```java
@ManyToMany
@JoinTable(
    name = "user_role",
    joinColumns = @JoinColumn(name = "user_id"),
    inverseJoinColumns = @JoinColumn(name = "role_id")
)
private Set<Role> roles = new HashSet<>();
```

### Rules
- **Use `Set`, not `List`.** `List` forces Hibernate to DELETE and re-INSERT the entire join table on any change. `Set` does diff updates.
- Avoid `@ManyToMany` entirely if the join table has any attributes (timestamps, roles, metadata). Promote it to a real `@Entity` with two `@ManyToOne` relationships — you'll need to eventually anyway.

```java
// Instead of @ManyToMany, model the join as an entity:
@Entity
class UserRole {
    @EmbeddedId private UserRoleId id;
    @ManyToOne @MapsId("userId") private User user;
    @ManyToOne @MapsId("roleId") private Role role;
    private Instant assignedAt;
}
```

## 3.8 Cascade Types

Cascade lets you propagate operations from parent to children.

| Cascade | Triggers |
|---------|----------|
| `PERSIST` | `em.persist(parent)` → persists children |
| `MERGE` | `em.merge(parent)` → merges children |
| `REMOVE` | `em.remove(parent)` → removes children |
| `REFRESH` | `em.refresh(parent)` → refreshes children |
| `DETACH` | `em.detach(parent)` → detaches children |
| `ALL` | all of the above |

### When to use cascade
- **Strong composition** (parent owns children, children can't live alone): order + order_items. Use `CascadeType.ALL` + `orphanRemoval = true`.
- **Shared references** (user + order — an order does not own a user): **never cascade**. Deleting an order should not delete a user.

**Rule:** cascade only from aggregate root to child entities it owns.

## 3.9 Orphan Removal

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
private List<OrderItem> items = new ArrayList<>();
```

`orphanRemoval = true` means: if a child is removed from the collection, delete it from the DB.

```java
order.getItems().remove(item);   // deletes item row at flush
```

Without it, the item's FK would simply be set to null (if nullable) or throw.

## 3.10 @Embeddable

For value objects — things without identity.

```java
@Embeddable
class Money {
    private BigDecimal amount;
    private String currency;
}

@Entity
class Order {
    @Embedded
    private Money total;
}
```

- No separate table; columns live on the owner.
- Reusable across entities.
- Preferred over small `@Entity` types for value objects.

## 3.11 Inheritance Strategies

```java
@Inheritance(strategy = InheritanceType.XXX)
```

| Strategy | Layout | Pro | Con |
|----------|--------|-----|-----|
| `SINGLE_TABLE` (default) | One table for the hierarchy with discriminator | Fast queries, simple | Many nullable columns |
| `JOINED` | Parent table + child tables joined | Normalized | Multi-join queries |
| `TABLE_PER_CLASS` | Each concrete class own table | Simple per-class queries | Polymorphic queries do UNION |

**Default choice:** `SINGLE_TABLE` unless you have a specific reason. Polymorphism in relational DBs is awkward; consider avoiding it if you can.

---

# 4. Fetching Strategies

This is where 80% of production JPA problems live.

## 4.1 LAZY vs EAGER

Lazy loading means: when you load an entity, its associations are not loaded until you touch them.

```java
User u = userRepo.findById(1L).get();
// SELECT * FROM users WHERE id = 1  ← only this

u.getOrders().size();
// SELECT * FROM orders WHERE user_id = 1  ← triggered here
```

EAGER means associations load with the parent — always, unconditionally.

### The default fetch types (memorize)

| Annotation | Default |
|------------|---------|
| `@ManyToOne` | **EAGER** |
| `@OneToOne` | **EAGER** |
| `@OneToMany` | LAZY |
| `@ManyToMany` | LAZY |

**Rule:** override every `@ManyToOne` and `@OneToOne` to `LAZY`. EAGER is rarely what you want and makes it impossible to opt out later.

### Why EAGER is dangerous
- Every query for the entity unconditionally joins or re-queries associations.
- You can't turn it off per query — it's baked into the mapping.
- It hides the cost of your queries.

## 4.2 The N+1 Problem

The single most common JPA performance bug.

### How it happens

```java
List<Order> orders = orderRepo.findAll();   // 1 query
for (Order o : orders) {
    System.out.println(o.getUser().getName()); // 1 query per order
}
```

Result: 1 + N queries. For 1000 orders, that's 1001 SQL round trips.

### Why it happens internally

When Hibernate loads the `orders`, it sees `@ManyToOne(fetch = LAZY)` and substitutes each `user` field with a **proxy** (a Hibernate-generated subclass). The proxy stores only the ID. The moment you call any method on the proxy (`getName()`), Hibernate executes:

```sql
SELECT * FROM users WHERE id = ?
```

Once per unique user ID.

### How to detect it

1. **Enable SQL logging** in dev:
   ```yaml
   spring:
     jpa:
       properties:
         hibernate:
           show_sql: true
           format_sql: true
   logging:
     level:
       org.hibernate.SQL: DEBUG
       org.hibernate.orm.jdbc.bind: TRACE  # see bound parameters
   ```

2. **Use Hibernate statistics** (see Section 14).

3. **Integration tests with query count assertions:**
   ```java
   @Test
   @Sql(...)
   void noNPlusOne() {
       SQLStatementCountValidator.reset();
       service.listOrdersWithUsers();
       assertSelectCount(1);   // from db-util library
   }
   ```

4. **Production APM** (Datadog, New Relic, Glowroot) — watch for bursts of identical queries.

## 4.3 Fixing N+1

### Option A: JOIN FETCH (ad hoc)

```java
@Query("SELECT o FROM Order o JOIN FETCH o.user WHERE o.status = :status")
List<Order> findByStatusWithUser(@Param("status") String status);
```

- One SQL with a JOIN.
- Good for single collections / single `@ManyToOne`.
- **Cannot JOIN FETCH two collections** at once without `MultipleBagFetchException` (Hibernate produces a Cartesian product). Workaround: use `Set` instead of `List`, or split into multiple queries, or use `@EntityGraph`.
- **Cannot paginate** a JOIN FETCH on a collection correctly — you'll get a warning and Hibernate paginates in memory after fetching everything.

### Option B: Entity Graphs (reusable)

```java
@Entity
@NamedEntityGraph(
    name = "Order.withUserAndItems",
    attributeNodes = {
        @NamedAttributeNode("user"),
        @NamedAttributeNode("items")
    }
)
public class Order { ... }

// In repository:
@EntityGraph("Order.withUserAndItems")
List<Order> findByStatus(String status);
```

- Declarative, reusable across methods.
- Can be dynamic: `@EntityGraph(attributePaths = {"user", "items"})`.
- Same collection caveats as JOIN FETCH.

### Option C: Batch Fetching (defaulted lazy loading)

```java
@ManyToOne(fetch = FetchType.LAZY)
@BatchSize(size = 50)
private User user;
```

Or globally:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 50
```

Changes the N+1 pattern into `1 + ceil(N/50)` queries using `WHERE id IN (?, ?, ...)`. Much better, and lazy-friendly (no over-fetching when you don't use the relation).

**Recommendation:** set `default_batch_fetch_size: 50` globally. It's a massive safety net.

### Which to pick?

| Scenario | Use |
|----------|-----|
| You always need the relation for this endpoint | JOIN FETCH / EntityGraph |
| You sometimes need it, sometimes not | Lazy + BatchSize |
| You're paginating + joining collections | DTO projection or 2 queries |
| You're exporting large data | Plain SQL, not JPA |

---

# 5. Queries

## 5.1 JPQL (Java Persistence Query Language)

Object-oriented. You query **entities and their fields**, not tables and columns.

```java
@Query("SELECT u FROM User u WHERE u.email = :email AND u.active = true")
Optional<User> findActiveByEmail(@Param("email") String email);
```

- Portable across DBs.
- Type-checked at startup (Spring validates `@Query` strings on app start if `hibernate.query.startup-validation=true`, default `false` — enable it).
- Supports joins, subqueries, aggregations, `CASE`.
- **Limits:** no window functions, no CTEs, no lateral joins, no DB-specific functions without custom dialect extension.

## 5.2 Native Queries

Plain SQL.

```java
@Query(value = """
    SELECT * FROM users u
    WHERE u.email ~ :pattern
    """, nativeQuery = true)
List<User> findByEmailRegex(@Param("pattern") String pattern);
```

### When to use
- DB-specific features (Postgres `~`, window functions, `LATERAL`).
- Performance tuning the ORM can't express.
- Bulk operations.

### Caveats
- Not portable across DBs.
- Returns entities only if the columns match the mapping exactly (or use `@SqlResultSetMapping`).
- Bypasses some JPA features (e.g., not cached by L2).
- `nativeQuery = true` + pagination: add `countQuery` because Spring can't auto-derive count queries.

```java
@Query(value = "SELECT * FROM orders WHERE created_at > :since",
       countQuery = "SELECT count(*) FROM orders WHERE created_at > :since",
       nativeQuery = true)
Page<Order> findRecent(@Param("since") Instant since, Pageable pageable);
```

## 5.3 Criteria API

Type-safe, programmatic query construction. Verbose, but unbeatable for dynamic queries (search forms with optional filters).

```java
public List<User> search(String nameLike, Boolean active, String roleName) {
    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<User> q = cb.createQuery(User.class);
    Root<User> u = q.from(User.class);

    List<Predicate> where = new ArrayList<>();
    if (nameLike != null) where.add(cb.like(u.get("name"), "%" + nameLike + "%"));
    if (active != null) where.add(cb.equal(u.get("active"), active));
    if (roleName != null) {
        Join<User, Role> r = u.join("roles");
        where.add(cb.equal(r.get("name"), roleName));
    }
    q.where(cb.and(where.toArray(Predicate[]::new)));
    return em.createQuery(q).getResultList();
}
```

### Use when
- Dynamic queries where filters are optional at runtime.
- You want compile-time type safety (via the Criteria metamodel: `User_.name`).

### Avoid when
- Query shape is static — JPQL is cleaner.
- You need joins in complex shapes — becomes unreadable fast.

### Alternatives
- **Spring Data JPA Specifications** — lightweight wrapper around Criteria. Good for composable filters.
- **QueryDSL** — a third-party DSL that's much nicer than raw Criteria (recommended if you do a lot of dynamic queries).

## 5.4 Query Comparison Table

| Dimension | JPQL | Native | Criteria |
|-----------|------|--------|----------|
| Portability | ✅ | ❌ | ✅ |
| Type safety | partial | ❌ | ✅ (with metamodel) |
| Dynamic queries | awkward | awkward | ✅ |
| DB-specific features | ❌ | ✅ | ❌ |
| Readability | ✅ | ✅ | ❌ |
| Performance transparency | medium | ✅ | medium |

**Default choice:** JPQL. Native when you need DB features. Criteria/Specifications/QueryDSL for dynamic filters.

---

# 6. Spring Data JPA Repositories

## 6.1 Repository Hierarchy

```
Repository<T, ID>           // marker
  └── CrudRepository<T, ID>           // save, findById, findAll, delete
        └── PagingAndSortingRepository // paging + sorting
              └── JpaRepository       // JPA-specific (flush, saveAll, etc.)
```

Use `JpaRepository<T, ID>` by default. Prefer composition over deep hierarchies — don't create a custom base repository unless needed.

## 6.2 How Spring Generates Queries

Three mechanisms, in priority order:

1. **Named queries** — `@NamedQuery` on the entity or `@Query` on the method.
2. **Derived queries** — method name parsed into JPQL.
3. **Manual `@Query`**.

### Derived query example

```java
List<User> findByLastNameAndActiveTrueOrderByCreatedAtDesc(String lastName);
```

Spring parses the method name and generates JPQL:

```
SELECT u FROM User u
WHERE u.lastName = :lastName AND u.active = true
ORDER BY u.createdAt DESC
```

### Supported keywords (subset)
- `findBy`, `readBy`, `queryBy`, `getBy` — read
- `countBy`, `existsBy`, `deleteBy` — operations
- `And`, `Or`, `Between`, `LessThan`, `GreaterThan`, `Like`, `NotLike`, `In`, `NotIn`, `StartingWith`, `EndingWith`, `Containing`, `IsNull`, `IsNotNull`
- `OrderBy...Asc/Desc`
- `First`, `Top<N>` — limit

### Limitations
- Method names become unreadable past 3–4 conditions. Switch to `@Query`.
- No way to express joins well.
- No way to project to DTOs without additional annotations.
- No dynamic filtering — always all conditions are applied.

## 6.3 @Query

```java
@Query("SELECT u FROM User u WHERE u.lastLogin < :before")
List<User> findStaleUsers(@Param("before") Instant before);
```

### Modifying queries

```java
@Modifying(clearAutomatically = true, flushAutomatically = true)
@Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :before")
int deactivateStaleUsers(@Param("before") Instant before);
```

- `@Modifying` is required for UPDATE/DELETE — tells Spring this isn't a `SELECT`.
- `flushAutomatically` flushes the persistence context before the query, so your in-memory changes are applied.
- `clearAutomatically` clears the persistence context after — because **bulk updates bypass the persistence context**. Entities in memory become stale.

**Pitfall:** bulk updates do NOT trigger entity callbacks (`@PreUpdate`, auditing) and don't touch version columns. Plan accordingly.

## 6.4 Custom Repository Methods

For logic that doesn't fit `@Query`, add a custom implementation:

```java
public interface UserRepository
        extends JpaRepository<User, Long>, UserRepositoryCustom {}

public interface UserRepositoryCustom {
    List<User> search(UserFilter filter);
}

public class UserRepositoryCustomImpl implements UserRepositoryCustom {
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<User> search(UserFilter filter) {
        // Criteria, QueryDSL, or native SQL
    }
}
```

The naming convention `<RepoName>Impl` is mandatory — Spring wires it automatically.

## 6.5 save() — What It Actually Does

```java
// From SimpleJpaRepository
public <S extends T> S save(S entity) {
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
        return entity;
    } else {
        return em.merge(entity);   // ← dangerous
    }
}
```

- For a new entity: `persist()`.
- For an existing one (has an ID): `merge()`.

### Why `merge()` is dangerous
`merge()` copies the state of a detached entity into a new managed entity. If your detached entity has `null` fields that are meant to be "unchanged," merge will set them to null in the database. This is how "I only updated one field but everything else got wiped" happens.

**Recommended pattern:** avoid `save()` for updates. Instead:
```java
@Transactional
public void updateName(Long id, String newName) {
    User u = userRepo.findById(id).orElseThrow();
    u.setName(newName);  // dirty checking does the UPDATE — no save() needed
}
```

---

# 7. Transactions

## 7.1 @Transactional: What Happens Internally

Spring's `@Transactional` is implemented via an **AOP proxy**. When you call `userService.doSomething()`:

1. A proxy around `UserService` intercepts the call.
2. It asks the transaction manager to open (or join) a transaction.
3. It invokes your real method.
4. On return: commit.
5. On runtime exception: rollback.

This is why:

```java
public class UserService {
    @Transactional public void outer() { inner(); }            // proxied ✅
    @Transactional public void inner() { /* ... */ }            // NOT proxied ❌
}
```

Self-invocation skips the proxy. `inner()`'s `@Transactional` is ignored because you're calling `this.inner()`, not `proxy.inner()`.

### Workaround
- Move `inner()` to another bean, or
- Inject `self`: `@Autowired private UserService self;` then call `self.inner()`.

## 7.2 Propagation

Defines what happens when a transactional method is called from within another transactional method.

| Propagation | Behavior |
|-------------|----------|
| `REQUIRED` (default) | Join existing or create new |
| `REQUIRES_NEW` | Suspend existing, create new (physical) |
| `NESTED` | Savepoint within existing (DB must support) |
| `SUPPORTS` | Use if exists, run non-tx otherwise |
| `NOT_SUPPORTED` | Suspend existing, run non-tx |
| `MANDATORY` | Fail if no existing |
| `NEVER` | Fail if existing |

### REQUIRES_NEW — the "audit log" pattern

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void recordAudit(Audit a) {
    auditRepo.save(a);
}
```

Even if the outer transaction rolls back, the audit row persists. Useful for logging, metrics, error tracking.

**Caveat:** uses a second DB connection while the outer is suspended. Watch pool sizing.

## 7.3 Isolation Levels

| Level | Dirty read | Non-repeatable read | Phantom read |
|-------|-----------|---------------------|--------------|
| `READ_UNCOMMITTED` | possible | possible | possible |
| `READ_COMMITTED` (Postgres default) | no | possible | possible |
| `REPEATABLE_READ` (MySQL default) | no | no | possible* |
| `SERIALIZABLE` | no | no | no |

Postgres `REPEATABLE_READ` actually also prevents phantom reads.

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
public void transferFunds(...) { ... }
```

**Practical guidance:** stick with the DB default. Reach for `SERIALIZABLE` only when business logic demands it and you're ready to handle serialization errors (`can't serialize access`) — your code must retry.

## 7.4 Rollback Rules

By default, `@Transactional` rolls back on:
- Unchecked exceptions (`RuntimeException`, `Error`).
- **NOT** on checked exceptions. A checked exception commits.

To roll back on checked exceptions:

```java
@Transactional(rollbackFor = Exception.class)
```

### Common pitfall
```java
@Transactional
public void foo() {
    try {
        bar();
    } catch (RuntimeException e) {
        log.error(...);
        // swallowed — but transaction is ALREADY marked rollback-only
    }
}
```
Once Spring sees a RuntimeException cross a proxy boundary (or a nested `@Transactional` call throws), the outer transaction is marked `rollbackOnly`. Swallowing the exception doesn't save you. At commit, you'll get `UnexpectedRollbackException`.

**Rule:** don't swallow exceptions inside a transactional method unless you intentionally want the rollback.

## 7.5 @Transactional on Read Methods

```java
@Transactional(readOnly = true)
public List<UserDto> listUsers() { ... }
```

### Why readOnly matters
- Hibernate **skips the dirty-check snapshot** — less memory, faster flush (no flush actually, as there's nothing to flush).
- Some DB drivers route to read replicas when `readOnly` is set.
- Signals intent to your team.

**Apply `@Transactional(readOnly = true)` at the service class level for read-heavy services, then override with `@Transactional` on write methods.**

## 7.6 Common Transaction Mistakes

1. **No `@Transactional` on service method** → each repo call opens/closes its own tx; lazy loading throws.
2. **`@Transactional` on controller** → ties tx to HTTP lifecycle; bad separation.
3. **Long transactions** → DB connection held for seconds, locks accumulate, pool exhaustion. Keep tx tight.
4. **External calls inside tx** → HTTP call inside a tx means DB connection held during HTTP round trip. Move external calls outside the tx.
5. **`@Transactional` on private method** — silently ignored (proxy can't intercept private).
6. **`@Transactional` on method called from same class** — self-invocation bypass.
7. **Calling `save()` + `throw`** — thinking the save persisted. It didn't — rollback.

---

# 8. Performance Optimization

## 8.1 The N+1 Problem (again, with rigor)

See Section 4.2–4.3. Summary of fixes by scenario:

| Symptom | Fix |
|---------|-----|
| N+1 on `@ManyToOne` | JOIN FETCH, EntityGraph, or BatchSize |
| N+1 on single `@OneToMany` | JOIN FETCH (+ `DISTINCT` on the JPQL) |
| Need two collections | BatchSize or two separate queries |
| Pagination + collection | DTO projection, or "fetch IDs then fetch entities" pattern |

### The "fetch IDs then entities" pattern

```java
// 1. Paginate IDs only (no join risk)
Page<Long> ids = repo.findIdPage(pageable);

// 2. Fetch full entities with joins
List<Order> orders = repo.findByIdIn(ids.getContent(), "Order.withItems");
```

## 8.2 Batch Inserts / Updates

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc.batch_size: 50
        order_inserts: true
        order_updates: true
        batch_versioned_data: true
```

### Rules for batching to actually work
- **Don't use `GenerationType.IDENTITY`.** It disables JDBC batch inserts because Hibernate needs to round-trip for each generated ID. Use `SEQUENCE` with pooled allocation.
- `order_inserts` / `order_updates` groups SQL by table so batch sizes aren't broken by interleaving.
- `batch_versioned_data` allows batching of UPDATEs on `@Version`ed entities.

### Large batch loop — clear periodically

```java
@Transactional
public void importUsers(List<UserDto> rows) {
    for (int i = 0; i < rows.size(); i++) {
        em.persist(new User(rows.get(i)));
        if (i % 50 == 0) {
            em.flush();
            em.clear();  // detach everything to free memory
        }
    }
}
```

For truly large imports (>10K rows), **don't use JPA** — use `jdbcTemplate.batchUpdate` or Postgres `COPY`.

## 8.3 Query Optimization Checklist

- [ ] Every query explained in prod with `EXPLAIN ANALYZE`.
- [ ] Every FK column has an index.
- [ ] Every `WHERE` column in filter queries has an appropriate index (single or composite).
- [ ] `ORDER BY` + `LIMIT` supported by an index — or you're paying a sort.
- [ ] No `SELECT *` with wide rows for list endpoints — use projections.
- [ ] No `LIKE '%xxx%'` in hot paths without full-text indexes.
- [ ] No unbounded results (always paginate).
- [ ] No lazy collection in response serialization — always fetch deliberately or map to DTO.

## 8.4 Index Usage

JPA doesn't help you index. But you can declare indexes in mappings:

```java
@Entity
@Table(name = "orders", indexes = {
    @Index(name = "idx_orders_user_created", columnList = "user_id, created_at"),
    @Index(name = "idx_orders_status", columnList = "status")
})
public class Order { ... }
```

### Composite index ordering
- Leading column must match `WHERE` or range.
- `WHERE status = ? AND user_id = ?` with index on `(user_id, status)` doesn't help unless you also have `user_id` in the predicate.
- Postgres B-tree supports range on trailing column only; `user_id = ? AND created_at > ?` is perfect for `(user_id, created_at)`.

## 8.5 Read/Write Separation

For high-read apps:
- `@Transactional(readOnly = true)` on read methods.
- Configure AbstractRoutingDataSource to route read-only tx to replicas.
- Caveat: replication lag. Read-your-write gets broken — handle with sticky primary for user's session if needed.

## 8.6 DTO Projections for Reads

See Section 12. Single biggest win for read performance: **stop loading entities when you only need 5 columns**.

## 8.7 Avoid Cartesian Explosion

```sql
-- Bad: 1000 orders × 5 items × 3 payments = 15,000 rows
SELECT o.*, i.*, p.* FROM orders o
JOIN items i ON ...
JOIN payments p ON ...
```

Fixes:
- Never `JOIN FETCH` two independent collections in one query.
- Use two queries and merge in memory.
- Use `DISTINCT` in JPQL (Hibernate applies it both in SQL and in the Java result).

---

# 9. Edge Cases & Pitfalls

## 9.1 LazyInitializationException

```
org.hibernate.LazyInitializationException: could not initialize proxy - no Session
```

**Cause:** you tried to access a lazy association after the transaction closed (persistence context gone).

### Scenarios
- Returning an entity from a service to a controller, and the controller serializes a lazy collection.
- Detaching an entity and accessing its relations later.

### Real fix (not a workaround)
- **Fetch what you need within the transaction.** Use `JOIN FETCH` or EntityGraph.
- **Return DTOs, not entities.** Map inside the transaction.
- **Turn off OSIV.** OSIV (`spring.jpa.open-in-view=true`) hides this bug by keeping the session open during view rendering. It makes dev "work" but causes production issues: long-held DB connections, hidden N+1, confusing behavior.

**Don't use `Hibernate.initialize()`** as a fix — it masks the real problem and encourages lazy loading outside the service layer.

## 9.2 Detached Entity Pitfalls

```java
@Transactional
public User load(Long id) { return userRepo.findById(id).get(); }  // tx closes

public void update(User u) {
    u.setName("new");
    userRepo.save(u);    // merge() — but what if u has null fields?
}
```

Callers mutate detached entities and pass them back → `merge()` wipes out fields.

**Safer pattern:** accept an update *command*, not an entity.

```java
@Transactional
public void renameUser(Long id, String newName) {
    User u = userRepo.findById(id).orElseThrow();
    u.setName(newName);
}
```

## 9.3 Unexpected Updates

Scenario: you load an entity to read a field, but Hibernate issues an UPDATE at commit.

**Why?** Any code path that mutates a managed entity — even through a setter that looks harmless — triggers dirty checking. Common culprits:
- Defensive copy setters that re-create collections.
- `@PreUpdate` callbacks that modify timestamps.
- JPA callbacks that set version or audit fields.

**Debug:** enable `org.hibernate.SQL=DEBUG` and look for unexpected UPDATEs. Compare field-by-field what Hibernate thinks changed vs what you changed.

## 9.4 Infinite Loops in Relationships

Bidirectional `@ManyToMany` or `@OneToMany` in Lombok's `@ToString` / `@EqualsAndHashCode`:

```java
@Entity
@ToString   // includes relations → infinite recursion at toString()
public class User {
    @OneToMany(mappedBy = "user") private List<Order> orders;
}
```

Same with Jackson serialization: `user.orders[0].user.orders[0]...`.

### Fixes
- `@ToString(exclude = "orders")` or use `@ToString.Exclude` on the relation field.
- Jackson: `@JsonManagedReference` / `@JsonBackReference`, or better, **never serialize entities to JSON — use DTOs**.

## 9.5 Cascade Issues

```java
@OneToMany(cascade = CascadeType.ALL)
private List<Order> orders;
```

If you remove an order from the collection without `orphanRemoval`, the FK is set to null (or exception). If you set cascade inconsistently between sides, behavior diverges.

**Rule:** declare cascade on the parent side (`@OneToMany`) only. Never on child (`@ManyToOne`). Cascade **down**, not up.

## 9.6 equals / hashCode on Entities

This is the subtlest bug category.

### The problem
If you use generated IDs:
- An entity before `persist()` has `id = null`.
- After `persist()`, it has an ID.
- If `hashCode()` is based on `id`, its hash code just changed. If it was in a `HashSet`, it's now unreachable.

### Three acceptable patterns

**Pattern A — business key**
```java
@Override public boolean equals(Object o) {
    if (!(o instanceof User u)) return false;
    return Objects.equals(email, u.email);
}
@Override public int hashCode() { return Objects.hash(email); }
```
Only if you have a stable, immutable business key.

**Pattern B — ID-only with null-safe equals**
```java
@Override public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User u) || id == null || u.id == null) return false;
    return id.equals(u.id);
}
@Override public int hashCode() { return getClass().hashCode(); }  // constant
```
Constant hashCode sacrifices HashMap performance but keeps correctness. This is often the best compromise.

**Pattern C — assigned UUID**
```java
@Id
private UUID id = UUID.randomUUID();   // set at construction
```
Now ID is stable from the start. `equals`/`hashCode` on ID works correctly.

**Never:** use Lombok's default `@EqualsAndHashCode` on entities with generated IDs, which includes all fields.

## 9.7 DTO vs Entity Misuse

### Don't
- Return entities from controllers.
- Accept entities as `@RequestBody` on POST/PUT.
- Serialize entities with lazy fields.
- Put business logic that spans multiple aggregates on entity methods.

### Do
- Have separate DTOs (request / response / internal commands).
- Map inside services inside transactions.
- Use MapStruct, manual builders, or records for DTOs.

### Why entities-as-DTOs breaks
- Adds serialization concerns to domain model.
- Triggers lazy loading during serialization.
- Couples API shape to DB schema (refactors become breaking API changes).
- Leaks internal fields (passwords, internal flags).

## 9.8 Other Gotchas

- **`saveAll()` is not a batch insert** unless batching is configured (Section 8.2) and you don't use IDENTITY IDs.
- **`findAll()` loads everything.** Always paginate in production.
- **`@Transactional(readOnly = true)` + dirty entity** — Hibernate *may* still flush. It's "read-only" as a hint, not a guarantee, depending on provider config.
- **`Optional<Entity>` pitfall:** `repo.findById(id).get()` → NoSuchElementException if missing. Use `.orElseThrow(...)` always.
- **`@Enumerated(EnumType.ORDINAL)` (default)** — adding a new enum value mid-list shifts all values in DB. Always use `@Enumerated(EnumType.STRING)`.

---

# 10. Concurrency & Locking

## 10.1 Why Locking?

Two transactions update the same row:

```
T1 reads balance = 100
T2 reads balance = 100
T1 writes 100 - 30 = 70
T2 writes 100 - 50 = 50   ← last write wins; T1's debit is lost
```

This is a **lost update**. JPA provides two ways to prevent it.

## 10.2 Optimistic Locking (@Version)

```java
@Entity
class Account {
    @Id private Long id;
    private BigDecimal balance;
    @Version private long version;
}
```

When you UPDATE:
```sql
UPDATE account SET balance = ?, version = version + 1
WHERE id = ? AND version = ?   -- ← version check
```

If the row's version doesn't match, 0 rows affected → Hibernate throws `OptimisticLockException`.

### When to use
- Low contention workloads (most updates don't conflict).
- Read-modify-write patterns.
- Offline editing (user opens form, edits, submits).

### How to handle failures
- Retry with backoff (up to N times).
- Or surface to user: "Someone else edited this record — reload."

```java
@Retryable(retryFor = OptimisticLockException.class, maxAttempts = 3)
@Transactional
public void debit(Long accountId, BigDecimal amount) {
    Account a = accountRepo.findById(accountId).orElseThrow();
    a.setBalance(a.getBalance().subtract(amount));
}
```

### @Version supported types
- `int`, `Integer`, `long`, `Long`, `short`, `Short`, `Timestamp`, `Instant` (with extension).

## 10.3 Pessimistic Locking

```java
@Query("SELECT a FROM Account a WHERE a.id = :id")
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Account> findByIdForUpdate(@Param("id") Long id);
```

Translates to `SELECT ... FOR UPDATE`. Row is locked until transaction commits. Other transactions attempting the same lock wait (or fail with timeout).

### Lock modes
- `PESSIMISTIC_READ` — shared lock (others can read, not write).
- `PESSIMISTIC_WRITE` — exclusive lock.
- `PESSIMISTIC_FORCE_INCREMENT` — write lock + version bump.

### When to use
- High-contention hot rows where optimistic retries would thrash.
- Critical financial operations.
- Short transactions (lock held briefly).

### Pitfalls
- **Deadlocks.** If T1 locks A then B, and T2 locks B then A — deadlock. Always lock rows in the same order across the codebase.
- **Long transactions → long locks → pool starvation.** Keep locked transactions short.
- **`SELECT FOR UPDATE` + NOT FOUND** — returns no rows without locking; handle the null case.

### Lock timeout

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "3000"))
Optional<Account> findByIdForUpdateWithTimeout(@Param("id") Long id);
```

## 10.4 Which to Choose?

| Situation | Use |
|-----------|-----|
| Rare conflicts | Optimistic |
| Hot row, frequent conflicts | Pessimistic |
| User-editable forms | Optimistic |
| Inventory decrement at checkout | Pessimistic (or atomic SQL) |
| Distributed locking | Neither — use Redis/Redlock or advisory locks |

### Atomic SQL alternative
Sometimes the right move is to skip locking entirely with a single atomic statement:

```java
@Modifying
@Query("UPDATE Account a SET a.balance = a.balance - :amount " +
       "WHERE a.id = :id AND a.balance >= :amount")
int debitIfSufficient(@Param("id") Long id, @Param("amount") BigDecimal amount);
```

Check the returned row count. If 0, the debit failed (either missing or insufficient). No lock, no race.

---

# 11. Pagination & Sorting

## 11.1 Pageable

```java
Page<User> findByActive(boolean active, Pageable pageable);
```

```java
Page<User> page = repo.findByActive(true, PageRequest.of(0, 20, Sort.by("createdAt").descending()));
page.getContent();
page.getTotalElements();
page.getTotalPages();
```

### What Spring does
- Adds `LIMIT 20 OFFSET 0`.
- Executes a **second** `COUNT(*)` query for total.

## 11.2 Slice vs Page

| | `Page<T>` | `Slice<T>` |
|---|---|---|
| Count query | Yes | No |
| `hasNext()` | Yes | Yes |
| Total count available | Yes | No |

**Use `Slice` when you only need "is there a next page?"** — saves a full count query, which can be expensive on large tables.

## 11.3 Offset Pagination Problems

`LIMIT 100 OFFSET 10000` forces the DB to scan and discard 10,000 rows before returning results. Gets slower as offset grows.

### Keyset pagination (cursor-based)

```java
@Query("""
       SELECT u FROM User u
       WHERE u.createdAt < :cursor
       ORDER BY u.createdAt DESC
       """)
List<User> findNextPage(@Param("cursor") Instant cursor, Pageable limit);
```

- Uses index directly.
- Performance is constant regardless of depth.
- No "jump to page 500" support — only next/prev.

**For deep pagination, use keyset.** Spring Data JPA has built-in support via `KeysetScrollPosition` in newer versions.

## 11.4 Pagination + JOIN FETCH

**Don't.** Hibernate will log:

```
HHH000104: firstResult/maxResults specified with collection fetch; applying in memory
```

It fetches ALL rows and paginates in memory — catastrophic at scale.

### Solution: "fetch IDs, then entities"

```java
// 1. Paginate IDs only
Page<Long> idPage = repo.findIdPage(pageable);

// 2. Load entities with joins
@EntityGraph(attributePaths = {"items"})
List<Order> findByIdIn(List<Long> ids);
```

## 11.5 Sort on Derived Column

`@Query` + sort on non-entity column:

```java
@Query("SELECT u, COUNT(o) as orderCount FROM User u LEFT JOIN u.orders o GROUP BY u")
Page<Object[]> findWithOrderCount(Pageable pageable);
```

Sorting on `orderCount` via `Pageable` won't work directly — Spring can't map the alias to the JPQL. Use a DTO projection instead (Section 12).

---

# 12. DTO Projections

## 12.1 Why Projections Matter

Loading an entity is expensive:
- All columns loaded (even TEXT/BLOB you don't need).
- Persistence context tracks it (memory + dirty-check snapshot).
- Associations might lazy-load.
- Entities are mutable — you can accidentally mutate them.

A projection gives you exactly the columns you need, as an immutable DTO, with none of the overhead.

## 12.2 Interface-Based Projections

```java
public interface UserSummary {
    Long getId();
    String getEmail();
    String getFullName();
}

public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findByActive(boolean active);
}
```

Spring auto-implements the interface with a proxy backed by the query result. Works for simple cases.

### Nested projections

```java
public interface OrderSummary {
    Long getId();
    BigDecimal getTotal();
    UserSummary getUser();   // nested projection
}
```

### Open projections (SpEL)

```java
public interface UserSummary {
    @Value("#{target.firstName + ' ' + target.lastName}")
    String getFullName();
}
```

**Caveat:** open projections **load the full entity** (Hibernate can't predict what SpEL needs). Only closed projections (pure getters) allow column-level optimization.

## 12.3 Class-Based Projections (DTOs)

```java
public record UserSummaryDto(Long id, String email, String fullName) {}

@Query("""
       SELECT new com.example.dto.UserSummaryDto(u.id, u.email, u.firstName)
       FROM User u WHERE u.active = true
       """)
List<UserSummaryDto> findActiveSummaries();
```

### Advantages
- Type safety.
- No reflection overhead.
- Java `record` works beautifully.
- Columns selected in SQL match DTO — optimal.

### Caveats
- Full class name required in JPQL.
- Can't use constructor expression across joined collections in a single query.

## 12.4 Dynamic Projections

```java
<T> List<T> findByActive(boolean active, Class<T> type);

List<UserSummary> a = repo.findByActive(true, UserSummary.class);
List<UserDetail>  b = repo.findByActive(true, UserDetail.class);
```

Same method, multiple projections.

## 12.5 Performance Comparison

| | Entity | Interface projection | DTO constructor |
|-|-|-|-|
| Columns fetched | all | only referenced (closed) | only in constructor |
| Persistence context | tracked | not tracked | not tracked |
| Memory overhead | high | low | low |
| Mutation risk | yes | no | no |
| Lazy loading | yes | no | no |

**Rule:** for read endpoints, project. Only use entities when you intend to mutate.

---

# 13. Advanced Hibernate Features

## 13.1 Second-Level Cache

First-level cache = per transaction.
Second-level cache = per `SessionFactory`, across transactions.

### Setup (Hibernate + Ehcache / Caffeine)

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
```

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country { ... }
```

### When to use
- **Reference data:** countries, currencies, lookup tables.
- **Rarely-changing, frequently-read entities.**

### When NOT to use
- Write-heavy entities (invalidation thrashing).
- Multi-instance apps without a distributed cache (stale data across nodes).
- Anything where staleness matters more than latency.

### Concurrency strategies
- `READ_ONLY` — never changes. Fastest.
- `NONSTRICT_READ_WRITE` — occasional staleness OK.
- `READ_WRITE` — synchronous writes; consistent but slower.
- `TRANSACTIONAL` — requires JTA.

## 13.2 Query Cache

Caches the **result IDs** of a query, not the entity data.

```java
@QueryHints(@QueryHint(name = "org.hibernate.cacheable", value = "true"))
@Query("SELECT u FROM User u WHERE u.active = true")
List<User> findActive();
```

- Only useful if the query is hit repeatedly with identical params.
- **Every write to a cached entity invalidates every query involving that entity type** — can thrash badly.
- Measure before enabling.

## 13.3 Interceptors & Event Listeners

For cross-cutting concerns (audit, encryption, soft delete):

### Entity lifecycle callbacks
```java
@Entity
@EntityListeners(AuditingEntityListener.class)
public class Order {
    @CreatedDate   private Instant createdAt;
    @LastModifiedDate private Instant updatedAt;
    @CreatedBy     private String createdBy;
}
```

Enable with `@EnableJpaAuditing` on a config class.

### Global listeners via `@EntityListeners`

```java
public class AuditListener {
    @PrePersist @PreUpdate
    public void beforeChange(Object entity) { ... }
}
```

### Hibernate Interceptor / Integrator (advanced)
- `Interceptor` — intercept entity state changes pre-flush.
- Useful for global read-side effects, schema-level transforms.
- Rarely needed in application code.

## 13.4 @Filter — Soft Delete Example

```java
@Entity
@FilterDef(name = "notDeleted", defaultCondition = "deleted = false")
@Filter(name = "notDeleted")
public class User {
    private boolean deleted;
}

// Enable per session:
em.unwrap(Session.class).enableFilter("notDeleted");
```

Applies a WHERE clause to every query for the entity. Useful for multi-tenancy or soft-delete.

## 13.5 Stored Procedures

```java
@NamedStoredProcedureQuery(
    name = "User.archive",
    procedureName = "archive_user",
    parameters = {
        @StoredProcedureParameter(mode = ParameterMode.IN, name = "user_id", type = Long.class)
    }
)
```

Use when business logic genuinely lives in the DB (generally a smell in modern apps).

---

# 14. Debugging & Monitoring

## 14.1 SQL Logging

```yaml
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
    org.hibernate.engine.transaction.internal.TransactionImpl: DEBUG
spring:
  jpa:
    properties:
      hibernate:
        format_sql: true
        use_sql_comments: true
```

- `org.hibernate.SQL` → logs all SQL.
- `org.hibernate.orm.jdbc.bind` → shows bound parameters (use in dev only — leaks data).
- `use_sql_comments` → adds JPQL comment on top of SQL, so you can trace origin.

## 14.2 Datasource Proxy (Better Logging)

In dev, use `datasource-proxy` or `p6spy` — logs actual SQL with inlined parameters, execution time, and stack traces.

```xml
<dependency>
    <groupId>net.ttddyy</groupId>
    <artifactId>datasource-proxy</artifactId>
</dependency>
```

## 14.3 Hibernate Statistics

```yaml
spring:
  jpa:
    properties:
      hibernate:
        generate_statistics: true
```

```java
Statistics stats = em.getEntityManagerFactory()
    .unwrap(SessionFactory.class).getStatistics();
System.out.println("Queries: " + stats.getQueryExecutionCount());
System.out.println("Slowest: " + stats.getQueryExecutionMaxTimeQueryString());
System.out.println("Cache hit ratio: " + stats.getSecondLevelCacheHitCount());
```

### Useful counters
- `queryExecutionCount` — total SQL issued.
- `entityLoadCount` — how many entities hydrated.
- `collectionFetchCount` — how many lazy collections triggered.
- `sessionOpenCount` — sanity check on session lifecycle.

## 14.4 Query Count Testing

```xml
<dependency>
    <groupId>com.vladmihalcea</groupId>
    <artifactId>db-util</artifactId>
</dependency>
```

```java
@Test
void noNPlusOne() {
    SQLStatementCountValidator.reset();
    service.listOrders();
    SQLStatementCountValidator.assertSelectCount(1);
}
```

Put this in CI for hot paths — catches regressions early.

## 14.5 Production Monitoring

- **APM tools** (Datadog, New Relic, Elastic APM): show slow queries, query counts per endpoint.
- **pg_stat_statements** (Postgres): top queries by total time.
- **HikariCP metrics**: connection pool saturation is an early leading indicator of JPA issues.

Key alerts to set:
- Pool usage > 80% sustained.
- p99 query time > threshold.
- Transaction duration > threshold.
- OptimisticLockException rate.

## 14.6 Diagnosing a Slow Endpoint

1. **Count SQL statements** — is it an N+1?
2. **Run `EXPLAIN ANALYZE`** on the hottest query. Indexes? Sequential scan?
3. **Check transaction length** — are you holding a connection during external I/O?
4. **Check persistence context size** — too many entities loaded?
5. **Check if you're loading what you need** — projection vs entity?

---

# 15. When NOT to Use JPA

JPA is a great fit for ~80% of OLTP workloads. The other 20% is where senior engineers earn their money by choosing differently.

## 15.1 Complex Queries

JPQL can't express:
- Window functions (`ROW_NUMBER() OVER`, `LAG`, `LEAD`).
- Recursive CTEs.
- `LATERAL JOIN`.
- `GROUPING SETS`, `ROLLUP`, `CUBE`.
- DB-specific full-text search.
- Complex pivots.

Native queries can — but you lose type safety and the object graph benefits.

## 15.2 High-Performance Systems

- **Every OR abstraction has overhead.** Dirty checking, proxy generation, snapshot memory.
- For latency-sensitive paths (sub-5ms), a raw query with direct DTO mapping can be 2–5x faster.
- For write-heavy batch work, a JDBC batch is orders of magnitude faster than JPA.

## 15.3 Alternatives

### jOOQ

- Generates a **type-safe SQL DSL** from your schema.
- You write SQL in Java that compiles → exactly the SQL you expect.
- Supports every SQL feature of your target DB.
- No ORM, no dirty checking, no persistence context.

```java
List<UserRecord> users = dsl.selectFrom(USER)
    .where(USER.ACTIVE.eq(true))
    .orderBy(USER.CREATED_AT.desc())
    .limit(20)
    .fetch();
```

**Strengths:** control, type safety, SQL expressivity, performance.
**Weaknesses:** no relationship graph management; DDL coupling.

### MyBatis

- XML or annotation-based SQL → Java object mapping.
- You write SQL; MyBatis handles parameter binding and result mapping.
- No persistence context, no dirty checking.

**Strengths:** familiar to DBA-centric teams, full SQL control.
**Weaknesses:** less type-safe than jOOQ, XML-heavy.

### Spring JDBC (JdbcTemplate / JdbcClient)

Lowest-level, perfectly fine for simple things:

```java
jdbc.query(
    "SELECT id, email FROM users WHERE active = ?",
    new Object[]{true},
    (rs, i) -> new UserDto(rs.getLong("id"), rs.getString("email"))
);
```

**Strengths:** zero magic, fastest iteration on schema.
**Weaknesses:** verbose for complex mappings.

## 15.4 Comparison Table

| | JPA/Hibernate | jOOQ | MyBatis | Spring JDBC |
|-|-|-|-|-|
| Domain model + graphs | ✅ | ❌ | partial | ❌ |
| Dirty checking | ✅ | ❌ | ❌ | ❌ |
| Full SQL power | ⚠️ | ✅ | ✅ | ✅ |
| Type safety | partial | ✅ | partial | ❌ |
| Performance ceiling | medium | high | high | highest |
| Learning curve | high | medium | medium | low |
| Good for reports | ❌ | ✅ | ✅ | ✅ |
| Good for bulk writes | ❌ | ✅ | ✅ | ✅ |

## 15.5 Hybrid Architecture (Recommended at Scale)

```
┌─────────────────────────────────────────────────┐
│  Command side (writes, domain logic)            │
│    → JPA / Hibernate                            │
│    Entities + persistence context + dirty check │
└─────────────────────────────────────────────────┘
┌─────────────────────────────────────────────────┐
│  Query side (reads, reports, dashboards)        │
│    → jOOQ or Spring JDBC                        │
│    DTO-first, SQL-first                         │
└─────────────────────────────────────────────────┘
```

This is how teams at scale pragmatically use JPA without letting it strangle performance-critical reads.

---

# 16. Real-World Scenarios

## 16.1 High-Load Product Catalog API

### Problem
- 10K QPS read, 100 QPS write.
- Endpoint: "list products, 20 per page, with category name, price, thumbnail URL."
- Current code: `productRepo.findAll(pageable)` → N+1 on category, entity serialization.

### Diagnosis
- Sampling shows 41 SQL statements per request (1 products + 20 categories + 20 images).
- p99 latency 680ms; DB CPU at 70%.

### Fix

**Step 1: Projection DTO**
```java
public record ProductListDto(
    Long id, String name, BigDecimal price,
    String categoryName, String thumbUrl
) {}

@Query("""
    SELECT new com.example.ProductListDto(
        p.id, p.name, p.price, c.name, i.url)
    FROM Product p
    JOIN p.category c
    LEFT JOIN p.thumbnail i
    WHERE p.active = true
    """)
Page<ProductListDto> findListPage(Pageable pageable);
```

**Step 2: Add index on `(active, created_at)`** for the ORDER BY.

**Step 3: Cache paginated results at the HTTP layer** (5-second TTL — acceptable staleness for a catalog).

### Result
- 41 SQLs → 2 (data + count).
- p99: 680ms → 30ms.
- DB CPU: 70% → 15%.

### Takeaways
- Never use entities for high-volume list endpoints.
- Count queries matter — cache them or switch to `Slice`.
- Often, the first optimization is just "stop using JPA the wrong way."

---

## 16.2 Complex Domain Model — E-Commerce Order

### Problem
- Order aggregate: Order → OrderItems → Product (ref), Shipping, Payments, Address.
- Adding, removing, modifying items; cascading changes; audit trail needed.

### JPA usage

```java
@Entity
class Order {
    @Id @GeneratedValue(strategy = SEQUENCE, generator = "order_seq")
    @SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
    private Long id;

    @ManyToOne(fetch = LAZY, optional = false)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderItem> items = new ArrayList<>();

    @Embedded private Address shippingAddress;

    @Version private long version;

    public void addItem(Product p, int qty) {
        OrderItem item = new OrderItem(this, p, qty);
        items.add(item);
    }

    public void removeItem(Long itemId) {
        items.removeIf(i -> i.getId().equals(itemId));  // orphanRemoval triggers DELETE
    }
}
```

### Optimizations applied
- `@Version` for optimistic locking against concurrent edits.
- `LAZY` everywhere.
- `default_batch_fetch_size: 50`.
- Write-heavy: batched inserts via sequence IDs.
- Read-heavy paths (e.g., customer history) use DTO projections — not the Order entity.

### When the model fights you
- Reporting: "top customers by revenue last quarter" — **don't** do it in JPA. Write a native query or use a reporting DB.
- Bulk import of legacy orders — **don't** use `saveAll()`. Stream through JDBC batch.

---

## 16.3 Reporting Endpoint — Daily Revenue

### Problem
- GET `/api/reports/revenue-daily?from=...&to=...`
- Returns per-day totals, growth %, top 5 products per day.
- Uses window functions and `GROUP BY`.

### Wrong approach
Try to express it in JPQL → fail on window functions → give up.

### Right approach

```java
@Repository
public class RevenueReportRepository {

    private final JdbcTemplate jdbc;

    public List<RevenueDayDto> dailyRevenue(LocalDate from, LocalDate to) {
        String sql = """
            SELECT d::date                                      AS day,
                   COALESCE(SUM(o.total), 0)                    AS revenue,
                   LAG(SUM(o.total)) OVER (ORDER BY d::date)    AS prev_revenue
            FROM   generate_series(?::date, ?::date, '1 day') d
            LEFT JOIN orders o ON o.created_at::date = d::date AND o.status = 'PAID'
            GROUP BY d::date
            ORDER BY d::date
            """;
        return jdbc.query(sql, new Object[]{from, to},
                (rs, i) -> new RevenueDayDto(
                        rs.getObject("day", LocalDate.class),
                        rs.getBigDecimal("revenue"),
                        rs.getBigDecimal("prev_revenue")));
    }
}
```

### Why this is the right call
- Window functions aren't JPQL.
- DTO is read-only — no need for entity hydration.
- No transaction needed (or use `@Transactional(readOnly = true)`).
- Easily routed to a read replica.

### Takeaway
**Reports belong outside JPA.** Many teams keep a separate reporting module with JdbcTemplate or jOOQ, isolated from the domain model. No shame in mixed approaches.

---

## 16.4 Batch Import — Nightly Data Sync

### Problem
- Sync 2M rows from external system into `imported_records`.
- JPA `saveAll()` takes 45 minutes; memory blows up.

### Wrong approach
```java
List<ImportedRecord> rows = fetchAll();  // 2M objects in memory
repo.saveAll(rows);                       // one by one, IDENTITY IDs
```

### Right approach — JDBC batch

```java
@Transactional
public void importBatch(List<ImportDto> dtos) {
    String sql = "INSERT INTO imported_records (id, payload, imported_at) VALUES (?, ?, ?)";
    jdbc.batchUpdate(sql, dtos, 1000, (ps, dto) -> {
        ps.setLong(1, dto.id());
        ps.setString(2, dto.payload());
        ps.setTimestamp(3, Timestamp.from(Instant.now()));
    });
}
```

Stream the dtos, process 1000 at a time. 45 min → 90 sec.

### If you must use JPA for this
- Sequence IDs with `allocationSize = 1000`.
- `hibernate.jdbc.batch_size = 50`.
- `em.flush() + em.clear()` every 50 rows.
- Read-only transactions can't insert — use explicit `@Transactional`.

Even with all tuning, JPA will be 2–5x slower than JDBC batch for pure inserts.

---

## 16.5 Hot-Key Inventory Decrement at Checkout

### Problem
- Checkout must decrement `product.stock`. Hundreds of concurrent checkouts on hot items cause lost updates or deadlocks.

### Option A: Optimistic + Retry
```java
@Retryable(retryFor = OptimisticLockException.class, maxAttempts = 5)
@Transactional
public void reserve(Long productId, int qty) {
    Product p = productRepo.findById(productId).orElseThrow();
    if (p.getStock() < qty) throw new OutOfStockException();
    p.setStock(p.getStock() - qty);
}
```
**Problem:** high contention → many retries, latency spikes.

### Option B: Atomic SQL (recommended)
```java
@Modifying
@Query("""
    UPDATE Product p SET p.stock = p.stock - :qty
    WHERE p.id = :id AND p.stock >= :qty
    """)
int tryReserve(@Param("id") Long id, @Param("qty") int qty);
```
```java
if (repo.tryReserve(productId, qty) == 0) {
    throw new OutOfStockException();
}
```
- Single statement — no lost updates.
- No read-modify-write round trip.
- No retry needed.

### Option C: Pessimistic Lock
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT p FROM Product p WHERE p.id = :id")
Product findForUpdate(@Param("id") Long id);
```
Use when the decrement is part of complex logic that needs a consistent view across multiple rows.

### Takeaway
**Pick the right concurrency primitive for the contention profile.** Senior engineers know when to skip ORM features in favor of a single SQL statement.

---

# 17. Final Checklist

## You are a senior-level JPA developer if you can:

### Core mental model
- [ ] Explain the difference between JPA, Hibernate, and Spring Data JPA.
- [ ] Describe the persistence context and the four entity states.
- [ ] Explain dirty checking step by step.
- [ ] Know when flush happens (and how to control it).
- [ ] Know exactly what `save()` does internally.

### Mappings
- [ ] Distinguish owning vs inverse side and set both sides correctly.
- [ ] Pick the right `@Id` strategy for your DB + workload.
- [ ] Avoid `@ManyToMany` when attributes are coming.
- [ ] Use `Set` not `List` for `@ManyToMany`.
- [ ] Override `@ManyToOne` and `@OneToOne` to `LAZY`.

### Fetching
- [ ] Detect N+1 from logs or stats.
- [ ] Apply JOIN FETCH, EntityGraph, or BatchSize correctly per scenario.
- [ ] Know when pagination + JOIN FETCH is a trap and how to work around it.
- [ ] Set `default_batch_fetch_size` globally.

### Queries
- [ ] Choose JPQL, native, Criteria, or DTO based on the situation.
- [ ] Write `@Modifying` queries correctly (flush + clear).
- [ ] Use projections for read-heavy endpoints.

### Transactions
- [ ] Know why self-invocation bypasses `@Transactional`.
- [ ] Understand rollback rules and handle exceptions accordingly.
- [ ] Keep transactions short; avoid I/O inside them.
- [ ] Use `@Transactional(readOnly = true)` on read services.

### Performance
- [ ] Diagnose a slow endpoint in under 15 minutes (count SQL, check indexes, check tx length).
- [ ] Configure batch inserts correctly with SEQUENCE IDs.
- [ ] Know when to drop out of JPA for raw JDBC / jOOQ.

### Concurrency
- [ ] Pick optimistic vs pessimistic based on contention.
- [ ] Prefer atomic SQL for simple hot-row updates.
- [ ] Handle `OptimisticLockException` with retries or user-facing conflict UI.

### Pitfalls
- [ ] Turn off OSIV in production.
- [ ] Never return entities from controllers.
- [ ] Never use entities as API request bodies.
- [ ] Use `@Enumerated(EnumType.STRING)`.
- [ ] Implement `equals`/`hashCode` correctly for entities.
- [ ] Avoid Lombok's `@EqualsAndHashCode` / `@ToString` on entities.

### Production readiness
- [ ] Enable Hibernate statistics in staging.
- [ ] Track pool saturation and slow queries in APM.
- [ ] Test for query count in CI for hot paths.
- [ ] Use Flyway/Liquibase instead of `ddl-auto`.
- [ ] Know the business impact of your DB design decisions.

### Judgment
- [ ] Know when JPA is the wrong tool.
- [ ] Architect a hybrid approach (JPA for writes, SQL-first for reads).
- [ ] Explain every trade-off you're making.

---

## Closing Note

JPA is a powerful abstraction — but a leaky one. The engineers who suffer with it are the ones who treat it as magic. The engineers who master it treat it as **a wrapper over SQL + a tracking state machine**, and always know which they're working against at any moment.

When in doubt, turn on SQL logging. The truth is always in the SQL.
