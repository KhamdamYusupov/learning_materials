# Hibernate Senior Engineer Interview Preparation Guide

> **Target:** Senior Java Engineer role — Hibernate deep-dive
> **Approach:** Internal mechanics, performance trade-offs, production war stories
> **Philosophy:** If you understand *why* Hibernate does something, you can answer *any* question about it.

---

# A. Core Fundamentals

---

## A.1 What Is Hibernate?

**Interview Question:** *"Explain what Hibernate is and why it exists. Don't give me the textbook answer — tell me what problem it actually solves at the architectural level."*

### Ideal Senior-Level Answer

Hibernate is an ORM framework that implements the JPA specification and acts as a **persistence abstraction layer** between your Java domain model and the relational database. But at its core, it solves the **object-relational impedance mismatch** — the fundamental incompatibility between object graphs (identity, inheritance, polymorphism, encapsulation, bidirectional references) and relational tables (rows, foreign keys, joins, normalization).

What it actually does under the hood:

- **Maintains a first-level cache (Persistence Context)** — an identity map that guarantees `a == b` for two references to the same database row within a session
- **Implements Unit of Work pattern** — tracks dirty state and batches SQL writes at flush time rather than executing immediately
- **Generates SQL dynamically** — based on mapping metadata, dialect configuration, and the current state of the persistence context
- **Manages object lifecycle** — transitions entities through transient → persistent → detached → removed states with well-defined semantics at each transition

It is **not** a SQL replacement. It is a **state synchronization engine** between object heap memory and relational storage.

### Common Mistakes

- Describing Hibernate as "a tool that writes SQL for you" — this misses the state management aspect entirely
- Conflating Hibernate with JPA — Hibernate predates JPA and has features beyond the spec
- Claiming Hibernate eliminates the need to understand SQL — in production, you must understand the SQL it generates

### Follow-Up Questions

- "What is the impedance mismatch specifically? Give me three concrete examples."
- "When would you choose not to use Hibernate?"
- "How does Hibernate differ from a query mapper like MyBatis?"

---

## A.2 ORM Concepts

**Interview Question:** *"Walk me through the object-relational impedance mismatch. Why can't we just map objects to tables 1:1?"*

### Ideal Senior-Level Answer

The impedance mismatch manifests in five key areas:

| Problem | Object World | Relational World |
|---------|-------------|-----------------|
| **Identity** | `==` (reference) and `.equals()` (value) | Primary key |
| **Inheritance** | Class hierarchy, polymorphism | No native concept — must be simulated |
| **Associations** | Object references, bidirectional | Foreign keys, always unidirectional (FK → PK) |
| **Data navigation** | `order.getCustomer().getAddress()` | JOINs, set-based operations |
| **Granularity** | Fine-grained value objects | Flat column structures |

**Why this matters in practice:**

The navigation problem is the most performance-critical. In Java, you traverse an object graph imperceptibly. In SQL, each traversal is a potential JOIN or additional query. Hibernate must decide **when to load** (lazy vs eager), **how to load** (join, subselect, batch), and **what to load** (projections vs full entities). Getting these decisions wrong causes N+1 queries, Cartesian products, or OutOfMemoryErrors.

### Common Mistakes

- Only mentioning the "mapping columns to fields" aspect
- Ignoring the navigation/data access pattern mismatch — this is where most production issues originate
- Not understanding that bidirectional associations in Java have no relational counterpart — they're a convenience maintained by application code

---

## A.3 JPA vs Hibernate

**Interview Question:** *"What exactly is the relationship between JPA and Hibernate? When do you use JPA annotations and when do you use Hibernate-specific ones?"*

### Ideal Senior-Level Answer

**JPA** (`jakarta.persistence` / `javax.persistence`) is a **specification** — a set of interfaces, annotations, and behavioral contracts. It defines no implementation.

**Hibernate** is the most widely used **implementation** of that specification, but it also provides:

- Features beyond the spec: `@NaturalId`, `@Formula`, `@Where`, `@Filter`, `@BatchSize`, `@Fetch`, second-level cache SPI, multi-tenancy, Envers (auditing), custom types, soft deletes
- A native API (`Session`, `SessionFactory`) alongside the JPA API (`EntityManager`, `EntityManagerFactory`)
- Its own query language extensions (HQL is a superset of JPQL)

**Decision framework:**

```
Use JPA annotations when:
  → The feature exists in the JPA spec
  → You want implementation portability (rare in practice)

Use Hibernate annotations when:
  → You need performance tuning (@BatchSize, @Fetch)
  → You need features not in JPA (@NaturalId, @Formula, @Where)
  → You need advanced caching configuration
```

**Production reality:** In most Spring Boot projects, you use JPA annotations for entity mappings and Hibernate-specific features for performance tuning. Full portability away from Hibernate is a theoretical concern — switching ORM implementations is a rewrite, not a configuration change.

The key internal distinction:

```
EntityManagerFactory wraps SessionFactory
EntityManager wraps Session
Both share the same persistence context
```

### Common Mistakes

- Saying "JPA is the interface, Hibernate is the implementation" without understanding the implications — e.g., `EntityManager.unwrap(Session.class)` gives you full Hibernate API access
- Believing JPA portability is a practical goal — it almost never is
- Not knowing which annotations are JPA vs Hibernate-specific

### Follow-Up Questions

- "Name three Hibernate features that have no JPA equivalent."
- "If I call `entityManager.find()` vs `session.get()`, what's the difference?"
- "How does Hibernate implement the JPA Criteria API internally?"

---

## A.4 Hibernate Architecture

**Interview Question:** *"Draw me the Hibernate architecture. What are the major components and how do they interact?"*

### Ideal Senior-Level Answer

```
┌─────────────────────────────────────────────────┐
│                  Application Code                │
└──────────────────────┬──────────────────────────┘
                       │
         ┌─────────────▼─────────────┐
         │    SessionFactory          │ (thread-safe, heavyweight, one per DB)
         │  ┌─────────────────────┐  │
         │  │  Mapping Metadata   │  │ (compiled at startup)
         │  │  Query Plan Cache   │  │
         │  │  2nd Level Cache    │  │
         │  │  Connection Pool    │  │ (via DataSource)
         │  └─────────────────────┘  │
         └─────────────┬─────────────┘
                       │ openSession() / getCurrentSession()
         ┌─────────────▼─────────────┐
         │    Session (EntityManager) │ (NOT thread-safe, one per request/unit-of-work)
         │  ┌─────────────────────┐  │
         │  │ Persistence Context │  │ (1st level cache = identity map)
         │  │   Action Queue      │  │ (pending INSERT/UPDATE/DELETE)
         │  │   Snapshot Array    │  │ (original state for dirty checking)
         │  └─────────────────────┘  │
         └─────────────┬─────────────┘
                       │
         ┌─────────────▼─────────────┐
         │    Transaction (JTA/JDBC)  │
         └─────────────┬─────────────┘
                       │
         ┌─────────────▼─────────────┐
         │    JDBC / DataSource       │
         └─────────────┬─────────────┘
                       │
         ┌─────────────▼─────────────┐
         │    Database                │
         └───────────────────────────┘
```

**Key internals:**

- **SessionFactory**: Built once from `Configuration` / `MetadataSources`. Compiles all mapping metadata into internal `EntityPersister` and `CollectionPersister` objects. Immutable after creation. Holds the second-level cache regions.
- **Session**: Wraps a JDBC connection (acquired lazily). Contains the persistence context (identity map + snapshot array). Not thread-safe — binding it to a thread or request scope is mandatory.
- **Persistence Context**: Two main data structures — (1) a `Map<EntityKey, Object>` for identity mapping, (2) a `Map<EntityKey, Object[]>` holding the "loaded state" snapshot used for dirty checking at flush time.
- **Action Queue**: Collects `EntityInsertAction`, `EntityUpdateAction`, `EntityDeleteAction`, `CollectionUpdateAction`, etc. These are sorted and executed in a specific order at flush time to respect FK constraints.

### Common Mistakes

- Not knowing that `SessionFactory` is heavyweight and must be created once
- Not understanding that the persistence context is the first-level cache
- Confusing the action queue with the persistence context

---

## A.5 Session vs SessionFactory

**Interview Question:** *"Explain the lifecycle and threading model of Session and SessionFactory. Why does this matter in production?"*

### Ideal Senior-Level Answer

| Aspect | SessionFactory | Session |
|--------|---------------|---------|
| **Creation cost** | Expensive (seconds) — parses mappings, builds metamodel | Cheap (microseconds) |
| **Lifecycle** | Application-scoped (singleton) | Request/transaction-scoped |
| **Thread safety** | Thread-safe | **NOT** thread-safe |
| **Memory** | Holds compiled metadata, 2L cache references | Holds persistence context (grows with loaded entities) |
| **Typical count** | 1 per database | 1 per HTTP request or unit of work |

**Why this matters in production:**

1. **Memory leaks**: If you keep a Session open too long (e.g., across multiple requests), the persistence context accumulates entities that can never be garbage collected. This is one of the most common Hibernate memory leaks.
2. **Thread safety violations**: Sharing a Session across threads causes ConcurrentModificationException on the internal maps or, worse, silent data corruption.
3. **Connection management**: Each Session eventually acquires a JDBC connection. Long-lived sessions hold connections from the pool, starving other threads.

**Spring Boot context:** Spring binds one Session per thread via `ThreadLocal` through the `OpenSessionInViewInterceptor` or `@Transactional` proxy. The Session is created when the transaction begins and closed when it commits/rolls back.

```java
// Anti-pattern: Session leaking across requests
@Component
public class BadService {
    @Autowired
    private EntityManager em; // This is a proxy! Safe.

    // The actual Session behind the proxy is scoped to the
    // current transaction. Spring handles this.
}
```

### Common Mistakes

- Storing a Session in a singleton bean field (bypassing Spring's proxy mechanism)
- Not calling `session.clear()` in batch processing loops, causing the persistence context to grow unbounded
- Creating multiple SessionFactory instances for the same database

---

## A.6 Persistence Lifecycle & Entity States

**Interview Question:** *"Walk me through the Hibernate entity lifecycle. For each state transition, tell me what SQL Hibernate executes and what happens in the persistence context."*

### Ideal Senior-Level Answer

```
                    new()
                      │
                      ▼
               ┌─────────────┐
               │  TRANSIENT   │  Not in persistence context
               │              │  No database representation
               └──────┬──────┘
                      │ persist() / save()
                      ▼
               ┌─────────────┐
               │  PERSISTENT  │  In persistence context (identity map)
               │              │  Tracked for dirty checking
               │              │  Has database representation (or will at flush)
               └──┬───┬───┬──┘
                  │   │   │
    detach()/     │   │   │  remove()
    close()/      │   │   │
    clear()       │   │   ▼
                  │   │  ┌─────────────┐
                  │   │  │   REMOVED    │  Scheduled for DELETE at flush
                  │   │  │             │  Still in persistence context
                  │   │  └─────────────┘
                  │   │
                  ▼   │
           ┌──────────┴──┐
           │  DETACHED    │  Not in any persistence context
           │              │  Has database representation
           │              │  Changes NOT tracked
           └──────┬──────┘
                  │ merge() / update() / lock()
                  ▼
           Back to PERSISTENT
```

**Detailed state transitions:**

| Transition | Method | SQL Timing | Persistence Context Effect |
|-----------|--------|------------|---------------------------|
| Transient → Persistent | `persist()` | INSERT at flush (or immediately if `IDENTITY` generator) | Entity added to identity map + snapshot |
| Persistent → Detached | `detach()` / `close()` / `clear()` | None | Entity removed from identity map |
| Detached → Persistent | `merge()` | SELECT (to load current state) + UPDATE at flush | **New copy** added to context; original remains detached |
| Persistent → Removed | `remove()` | DELETE at flush | Entity scheduled for deletion |
| Removed → Persistent | `persist()` on removed entity | Cancels the DELETE | Re-added to context |

**Critical detail about `merge()` vs `update()`:**

```java
// merge() — JPA standard
Detached d = ...;
Managed m = em.merge(d);
// d is STILL detached! m is the managed copy.
// d != m (different references)

// update() — Hibernate-specific (deprecated)
session.update(d);
// d itself becomes managed
// Throws if another instance with same ID is already in the context
```

**The `IDENTITY` generator exception:** With `GenerationType.IDENTITY`, Hibernate **must** execute the INSERT immediately on `persist()`, because the ID is assigned by the database. This breaks JDBC batching for inserts — a significant performance implication.

```java
@Id
@GeneratedValue(strategy = GenerationType.IDENTITY) // INSERT happens immediately
private Long id;

@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE)  // INSERT deferred to flush
private Long id;
```

### Common Mistakes

- Thinking `merge()` makes the passed instance managed — it returns a **new** managed copy
- Not knowing that `IDENTITY` strategy forces immediate INSERT
- Forgetting that detached entities throw `LazyInitializationException` when accessing uninitialized lazy collections
- Believing `remove()` executes DELETE immediately — it doesn't until flush

### Follow-Up Questions

- "What happens if you call `persist()` on a detached entity?"
- "What's the difference between `merge()` and `saveOrUpdate()`?"
- "How does `IDENTITY` generator impact batch insert performance?"
- "Can a removed entity become persistent again?"

---

# B. Advanced Mapping

---

## B.1 Association Mappings

### B.1.1 @ManyToOne / @OneToMany

**Interview Question:** *"Explain the difference between unidirectional and bidirectional @OneToMany. What SQL does each generate? Which should you prefer and why?"*

### Ideal Senior-Level Answer

**Unidirectional @OneToMany (no @ManyToOne on child):**

```java
@Entity
public class Parent {
    @OneToMany
    @JoinColumn(name = "parent_id") // Without this: junction table!
    private List<Child> children;
}

@Entity
public class Child {
    // No reference to Parent
}
```

Without `@JoinColumn`, Hibernate creates a **junction table** (`parent_child`) — almost never what you want. With `@JoinColumn`, it uses the FK on the child table, but Hibernate still generates inefficient SQL:

```sql
-- On adding a child:
INSERT INTO child (id, name) VALUES (?, ?)        -- child inserted with NULL parent_id
UPDATE child SET parent_id = ? WHERE id = ?        -- then updated separately
```

**Bidirectional @OneToMany + @ManyToOne:**

```java
@Entity
public class Parent {
    @OneToMany(mappedBy = "parent")  // Inverse side
    private List<Child> children = new ArrayList<>();

    public void addChild(Child child) {
        children.add(child);
        child.setParent(this);  // Synchronize both sides!
    }
}

@Entity
public class Child {
    @ManyToOne(fetch = FetchType.LAZY)  // Owning side (has the FK)
    @JoinColumn(name = "parent_id")
    private Parent parent;
}
```

SQL generated:
```sql
-- On adding a child:
INSERT INTO child (id, name, parent_id) VALUES (?, ?, ?)  -- single statement, FK set immediately
```

**Rule:** Always prefer bidirectional with `@ManyToOne` as the owning side. The `@ManyToOne` side controls the FK column, producing cleaner SQL.

**Performance implication of collection type:**

| Collection Type | Behavior |
|----------------|----------|
| `List` (without `@OrderColumn`) | Bag semantics — duplicates allowed, **removes all + re-inserts** on element removal |
| `Set` | Proper set semantics — individual INSERT/DELETE per element change |
| `List` with `@OrderColumn` | Indexed list — maintains order column, UPDATE on reorder |

**Always use `Set` for `@OneToMany` unless you need ordered duplicates.** The `List` (bag) removal behavior is a common hidden performance killer.

### B.1.2 @OneToOne

**Interview Question:** *"What's the performance trap with @OneToOne? Why can't Hibernate lazy-load the non-owning side?"*

```java
@Entity
public class User {
    @OneToOne(mappedBy = "user", fetch = FetchType.LAZY)  // LAZY is IGNORED here!
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @OneToOne
    @JoinColumn(name = "user_id")
    private User user;  // Owning side — LAZY works here
}
```

**Why LAZY is ignored on the non-owning (mappedBy) side:**

Hibernate needs to decide: should it create a proxy or set `null`? To determine if the associated entity exists, it must query the FK — but the FK is on the **other** table. So Hibernate must execute a SELECT to check if a row exists in `user_profile` for this user. At that point, it might as well load the entire entity.

**Solutions:**

1. **`@MapsId`** — share the primary key. Hibernate knows the profile exists if the user exists.
   ```java
   @Entity
   public class UserProfile {
       @Id  // Same PK as User
       private Long id;

       @OneToOne
       @MapsId
       private User user;
   }
   ```
2. **Bytecode enhancement** — Hibernate instruments the entity class to intercept field access, enabling true lazy loading without proxies.
3. **Redesign** — use `@ManyToOne` with a unique constraint, or embed the data.

### B.1.3 @ManyToMany

**Interview Question:** *"When should you use @ManyToMany and when should you model the junction table explicitly?"*

**Short answer:** Almost always model it explicitly.

```java
// Simple @ManyToMany — only works if junction table has NO extra columns
@Entity
public class Student {
    @ManyToMany
    @JoinTable(name = "enrollment",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();
}

// Explicit junction entity — use when junction has extra data
@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("studentId")
    private Student student;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("courseId")
    private Course course;

    private LocalDate enrolledDate;  // Extra column on junction
    private String grade;
}
```

**Use explicit junction entity when:**
- The relationship has attributes (date, status, role)
- You need to query the relationship independently
- You need lifecycle control over the relationship
- Performance matters — `@ManyToMany` with `List` has the same bag removal problem

---

## B.2 Owning Side vs Inverse Side

**Interview Question:** *"What does 'owning side' mean in Hibernate? What happens if you only set the inverse side?"*

### Ideal Senior-Level Answer

The **owning side** is the side that controls the foreign key column in the database. Hibernate **only** inspects the owning side when generating SQL for the relationship.

**Rule:** The side **without** `mappedBy` is the owning side.

```java
// CRITICAL: This code does NOT persist the relationship
parent.getChildren().add(child);  // Inverse side only
// At flush: no UPDATE or INSERT for the FK

// This DOES persist it:
child.setParent(parent);  // Owning side

// Best practice: synchronize both sides
parent.addChild(child);  // Helper method sets both
```

**Why this trips people up:** In Java, both sides "look" connected. But Hibernate ignores the inverse side when determining FK values. If you only set the inverse side, the FK column stays `NULL` in the database.

---

## B.3 Cascade Types

**Interview Question:** *"Explain cascade types and their pitfalls. When does CascadeType.ALL become dangerous?"*

| Cascade Type | Propagated Operation | Danger Level |
|-------------|---------------------|-------------|
| `PERSIST` | `persist()` → children | Low |
| `MERGE` | `merge()` → children | Medium — may trigger unexpected SELECTs |
| `REMOVE` | `remove()` → children | **High** — deletes children |
| `REFRESH` | `refresh()` → children | Low |
| `DETACH` | `detach()` → children | Low |
| `ALL` | All of the above | **Dangerous** — includes REMOVE |

**`CascadeType.REMOVE` on `@ManyToMany`:** This is a **classic bug**. Removing a Student cascades REMOVE to Courses, which deletes courses that other students are enrolled in.

```java
// BUG: Deleting a student deletes their courses for everyone
@ManyToMany(cascade = CascadeType.ALL)
private Set<Course> courses;

// CORRECT: Only cascade PERSIST and MERGE
@ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
private Set<Course> courses;
```

**`orphanRemoval = true`:** Removes child when it's removed from the parent's collection. Only valid on `@OneToMany` and `@OneToOne`. Different from `CascadeType.REMOVE` — orphanRemoval triggers on **collection dereference**, not parent deletion.

```java
@OneToMany(mappedBy = "parent", orphanRemoval = true)
private List<Child> children;

// This deletes child from DB:
parent.getChildren().remove(child);

// This also deletes ALL children from DB:
parent.setChildren(new ArrayList<>());  // Dangerous!
```

---

## B.4 Fetch Types: LAZY vs EAGER

**Interview Question:** *"What are the defaults for fetch types? Why does Hibernate use those defaults? When would you override them?"*

### Ideal Senior-Level Answer

**Defaults:**

| Mapping | Default Fetch | Why |
|---------|--------------|-----|
| `@ManyToOne` | `EAGER` | Single FK lookup — cheap |
| `@OneToOne` | `EAGER` | Single FK lookup |
| `@OneToMany` | `LAZY` | Collection — potentially expensive |
| `@ManyToMany` | `LAZY` | Collection — potentially expensive |
| Basic fields | `EAGER` | Already in the same row |

**Critical point:** The JPA defaults for `@ManyToOne` and `@OneToOne` are `EAGER`. This is universally considered a **design mistake**. In production, you should **always** override to `LAZY`:

```java
@ManyToOne(fetch = FetchType.LAZY)  // Always do this
@JoinColumn(name = "parent_id")
private Parent parent;
```

**Why EAGER is problematic:**
- It's a **compile-time decision** that cannot be overridden at query time. You can make LAZY into EAGER per-query (join fetch), but you **cannot** make EAGER into LAZY per-query.
- It causes implicit JOINs or additional SELECTs on every query touching that entity.
- In entity graphs, it cascades — loading one entity pulls its entire eager graph, which pulls their eager graphs, etc.

**LAZY loading mechanics:** Hibernate creates a **proxy** (via bytecode generation using ByteBuddy by default) that extends your entity class. The proxy intercepts getter calls and triggers a SELECT on first access. This is why:
- Entity classes must not be `final`
- At minimum one constructor must be non-private
- Getter methods must not be `final`

### Common Mistakes

- Leaving `@ManyToOne` at its default `EAGER`
- Thinking you can override `EAGER` to `LAZY` at query time — you can't
- Not understanding that `LAZY` is a **hint**, not a guarantee (provider may still load eagerly)

---

## B.5 Embedded & Embeddable

**Interview Question:** *"When would you use @Embeddable vs a separate entity? What's the impact on the database schema?"*

```java
@Embeddable
public class Address {
    private String street;
    private String city;
    private String zipCode;
}

@Entity
public class Customer {
    @Embedded
    private Address billingAddress;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "ship_street")),
        @AttributeOverride(name = "city", column = @Column(name = "ship_city")),
        @AttributeOverride(name = "zipCode", column = @Column(name = "ship_zip"))
    })
    private Address shippingAddress;
}
```

**Database impact:** Embeddable columns are **inlined** into the owning entity's table. No separate table, no FK, no JOIN needed.

**Use embeddable when:**
- The value has no independent identity (no own primary key)
- It's always loaded with the parent
- It represents a value type concept (Address, Money, DateRange)

**Use entity when:**
- It has its own lifecycle
- It's shared/referenced by multiple entities
- You need to query it independently

---

## B.6 Inheritance Strategies

**Interview Question:** *"Compare the three JPA inheritance strategies. For each, give me the SQL schema, query behavior, and when you'd use it."*

### Single Table (SINGLE_TABLE) — Default

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "type")
public abstract class Payment { ... }

@Entity @DiscriminatorValue("CC")
public class CreditCardPayment extends Payment { private String cardNumber; }

@Entity @DiscriminatorValue("BANK")
public class BankTransfer extends Payment { private String iban; }
```

| Aspect | Behavior |
|--------|----------|
| Schema | One table, all columns, discriminator column |
| NULLs | Subclass columns are NULL for other types |
| Polymorphic query | Single SELECT, no JOIN — **fastest** |
| Insert | Single INSERT |
| Constraint | Cannot use NOT NULL on subclass columns |

**Best for:** Hierarchies where subclasses share most columns and you query polymorphically often.

### Joined Table (JOINED)

```java
@Entity
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment { ... }
```

| Aspect | Behavior |
|--------|----------|
| Schema | One table per class, joined by PK/FK |
| NULLs | Clean — no wasted columns |
| Polymorphic query | JOIN across all subclass tables — **slow** |
| Insert | INSERT into base + subclass table |
| Constraint | NOT NULL works on subclass columns |

**Best for:** When subclasses have many distinct columns and you rarely query polymorphically.

### Table Per Class (TABLE_PER_CLASS)

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment { ... }
```

| Aspect | Behavior |
|--------|----------|
| Schema | One complete table per concrete class |
| Polymorphic query | UNION ALL across all tables — **slowest** |
| Insert | Single INSERT |
| ID generation | Cannot use IDENTITY (must share sequence) |

**Best for:** Almost never. Avoid unless you never query the base type and want completely independent tables.

**Senior recommendation:** Default to `SINGLE_TABLE`. Switch to `JOINED` only when column waste is extreme. Avoid `TABLE_PER_CLASS` in production.

---

## B.7 Composite Keys

**Interview Question:** *"Compare @IdClass and @EmbeddedId. Which do you prefer and why?"*

```java
// @EmbeddedId approach (preferred)
@Embeddable
public class OrderLineId implements Serializable {
    private Long orderId;
    private Long productId;
    // equals() and hashCode() MANDATORY
}

@Entity
public class OrderLine {
    @EmbeddedId
    private OrderLineId id;

    @ManyToOne
    @MapsId("orderId")
    private Order order;

    @ManyToOne
    @MapsId("productId")
    private Product product;
}
```

```java
// @IdClass approach
@Entity
@IdClass(OrderLineId.class)
public class OrderLine {
    @Id private Long orderId;
    @Id private Long productId;
}
```

**Preference: `@EmbeddedId`** because:
- The composite key is a first-class object you can pass around
- Works naturally with `@MapsId`
- JPQL: `SELECT o FROM OrderLine o WHERE o.id.orderId = :id`
- Cleaner semantics — the key is explicitly a value type

**Critical:** Both approaches require `equals()` and `hashCode()` based on the key fields. Without these, Hibernate's identity map breaks — you get duplicate entries for the same database row.

---

## B.8 Natural ID

**Interview Question:** *"What is @NaturalId and when would you use it over a surrogate key lookup?"*

```java
@Entity
public class Book {
    @Id @GeneratedValue
    private Long id;

    @NaturalId
    @Column(nullable = false, unique = true)
    private String isbn;
}

// Optimized lookup — hits first-level cache by natural ID
Book book = session.byNaturalId(Book.class)
    .using("isbn", "978-3-16-148410-0")
    .load();
```

Hibernate maintains a **natural ID → PK resolution cache** in the persistence context and optionally in the second-level cache. This means `byNaturalId()` first resolves the natural ID to a PK (possibly cached), then uses the PK to load from the identity map or L2 cache — avoiding a database hit entirely.

**Use when:** You have a business key (ISBN, email, SSN) that is immutable and uniquely identifies the entity, and you frequently look up by that key.

---

## B.9 Value Types vs Entity Types

| Aspect | Value Type | Entity Type |
|--------|-----------|-------------|
| Identity | Defined by owning entity | Has own identity (`@Id`) |
| Lifecycle | Bound to owner | Independent |
| Shared | Cannot be shared | Can be referenced by multiple entities |
| Table | Embedded or collection table | Own table |
| Examples | `@Embeddable`, basic types, `@ElementCollection` | `@Entity` |

**Key insight:** `@ElementCollection` creates a **value type collection** — the elements have no identity and are stored in a separate table, but their lifecycle is entirely controlled by the parent. Deleting the parent deletes all elements. Hibernate handles removal by deleting **all rows** and re-inserting the remaining ones — which is very expensive for large collections.

---

# C. Performance & Optimization

---

## C.1 N+1 Problem

**Interview Question:** *"You're seeing 1001 queries for a list of 1000 orders with their customers. Walk me through the root cause, how you'd detect it, and all the ways to fix it."*

### Root Cause

```java
List<Order> orders = em.createQuery("SELECT o FROM Order o", Order.class)
    .getResultList();  // 1 query: SELECT * FROM orders

for (Order o : orders) {
    o.getCustomer().getName();  // N queries: SELECT * FROM customer WHERE id = ?
}
```

The initial query loads orders without their customers. Each `getCustomer()` call triggers a lazy-load SELECT. With 1000 orders → 1 + 1000 = 1001 queries.

### Detection Methods

1. **Hibernate statistics:** `hibernate.generate_statistics=true` → log shows query count per session
2. **p6spy / datasource-proxy:** SQL logging interceptor that shows actual executed queries
3. **Hibernate `org.hibernate.SQL` logger at DEBUG** — shows every SQL statement
4. **Integration tests asserting query count** — use `com.vladmihalcea:db-util` or `datasource-proxy`

```java
// Asserting query count in tests
@Test
void shouldLoadOrdersInOneQuery() {
    SQLStatementCountValidator.reset();

    List<Order> orders = orderRepository.findAllWithCustomers();

    SQLStatementCountValidator.assertSelectCount(1);
}
```

### Solutions

**1. JOIN FETCH (most common)**
```java
SELECT o FROM Order o JOIN FETCH o.customer
// SQL: SELECT o.*, c.* FROM orders o JOIN customer c ON o.customer_id = c.id
```
**Limitation:** Cannot JOIN FETCH multiple collections (Cartesian product). Can JOIN FETCH one collection + multiple `@ManyToOne`/`@OneToOne`.

**2. Entity Graph**
```java
@EntityGraph(attributePaths = {"customer", "orderLines"})
List<Order> findAll();
```

**3. @BatchSize (on the association)**
```java
@Entity
public class Customer {
    // When any Customer proxy is initialized, Hibernate loads up to 25 at once
    @BatchSize(size = 25)
    @OneToMany(mappedBy = "customer")
    private Set<Order> orders;
}
```
SQL: `SELECT * FROM orders WHERE customer_id IN (?, ?, ?, ..., ?)` — loads in batches.

**4. Subselect Fetch**
```java
@Fetch(FetchMode.SUBSELECT)
@OneToMany(mappedBy = "customer")
private Set<Order> orders;
```
SQL: `SELECT * FROM orders WHERE customer_id IN (SELECT id FROM customer WHERE ...)` — uses the original query as a subselect.

**5. DTO Projection (best for read-only)**
```java
SELECT new com.example.OrderSummaryDTO(o.id, o.date, c.name)
FROM Order o JOIN o.customer c
```
No entity management overhead, no persistence context, no dirty checking. Ideal for read-heavy endpoints.

### Comparison Matrix

| Approach | Queries | Memory | Flexibility | Use When |
|----------|---------|--------|-------------|----------|
| JOIN FETCH | 1 | High (full entities) | Per-query | Known access pattern |
| Entity Graph | 1 | High | Declarative | Spring Data repositories |
| @BatchSize | N/batch_size | Medium | Global | Unpredictable access patterns |
| Subselect | 2 | Medium | Global | Loading all children of a query result |
| DTO Projection | 1 | Low | Per-query | Read-only use cases |

---

## C.2 Batch Fetching & JDBC Batching

**Interview Question:** *"Explain the difference between @BatchSize and JDBC batching. How do you configure each and what's the performance impact?"*

### @BatchSize — Collection/Entity Batch Loading

Controls how many **uninitialized proxies or collections** are loaded in a single SELECT:

```properties
# Global default
hibernate.default_batch_fetch_size=25
```

Or per-entity/collection:
```java
@Entity
@BatchSize(size = 25)
public class Product { ... }
```

When Hibernate needs to initialize a lazy proxy and finds other uninitialized proxies of the same type in the persistence context, it loads them all in one `WHERE id IN (...)` query.

### JDBC Batching — DML Statement Batching

Controls how many INSERT/UPDATE/DELETE statements are sent to the database in a single JDBC batch:

```properties
hibernate.jdbc.batch_size=50
hibernate.order_inserts=true
hibernate.order_updates=true
hibernate.jdbc.batch_versioned_data=true
```

**Critical requirements:**
- `order_inserts=true` is essential — Hibernate must group INSERT statements by entity type for JDBC batching to work
- `IDENTITY` ID generation **disables** insert batching (Hibernate needs the generated ID immediately)
- Use `SEQUENCE` with `@SequenceGenerator(allocationSize = 50)` for efficient batching

```java
// Batch insert pattern
for (int i = 0; i < 100_000; i++) {
    em.persist(new Item("item-" + i));

    if (i % 50 == 0) {
        em.flush();  // Execute batched SQL
        em.clear();  // Release persistence context memory
    }
}
```

**Without `flush()/clear()` in the loop:** The persistence context grows to 100K entities, dirty checking compares 100K snapshots, and you likely run out of memory.

---

## C.3 Second-Level Cache

**Interview Question:** *"Explain the Hibernate second-level cache architecture. When does it help and when does it hurt?"*

### Architecture

```
Session.find(id)
  → 1st Level Cache (persistence context) — HIT? Return.
  → 2nd Level Cache (SessionFactory-scoped) — HIT? Return.
  → Database SELECT
```

The L2 cache stores **dehydrated state** (not entity instances) — essentially `Object[]` arrays of column values, keyed by entity type + ID. On cache hit, Hibernate reconstructs the entity from the cached state array.

### Configuration

```properties
hibernate.cache.use_second_level_cache=true
hibernate.cache.region.factory_class=org.hibernate.cache.jcache.JCacheRegionFactory
```

```java
@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Country {
    // Cached
}

@Entity
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Customer {
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
    @OneToMany(mappedBy = "customer")
    private Set<Order> orders;  // Collection cache — separate region
}
```

### Cache Concurrency Strategies

| Strategy | Consistency | Performance | Use Case |
|----------|-----------|-------------|----------|
| `READ_ONLY` | Perfect | Best | Reference data (countries, currencies) |
| `NONSTRICT_READ_WRITE` | Eventual | Good | Data that rarely changes, stale reads acceptable |
| `READ_WRITE` | Strong (soft locks) | Medium | General mutable data |
| `TRANSACTIONAL` | Full (JTA) | Lowest | When you need strict transactional cache guarantees |

### When L2 Cache Hurts

- **High-write entities:** Cache invalidation on every write; you pay cache overhead with few hits
- **Entities always loaded with custom queries:** L2 cache is keyed by ID; JPQL/HQL results don't hit it (that's the query cache)
- **Large entities with volatile fields:** Entire cached state is invalidated when any field changes
- **Clustered environments without proper cache synchronization:** Stale data across nodes

### Query Cache

```properties
hibernate.cache.use_query_cache=true
```

```java
List<Country> countries = em.createQuery("SELECT c FROM Country c", Country.class)
    .setHint("org.hibernate.cacheable", true)
    .getResultList();
```

**Query cache stores:** query string + parameters → list of entity IDs. On cache hit, it then fetches each entity by ID from the L2 cache.

**Query cache invalidation:** Invalidated when **any** entity in the cached result's table space is modified. This makes it only useful for near-static reference data. For anything that changes frequently, the query cache has a near-zero hit rate and adds overhead.

---

## C.4 Dirty Checking

**Interview Question:** *"How does Hibernate's dirty checking work internally? What's the performance cost for large persistence contexts?"*

### Internal Mechanism

1. When an entity is loaded, Hibernate stores a **snapshot** — a deep copy of the entity's property values as an `Object[]`.
2. At flush time, Hibernate iterates every entity in the persistence context and compares its current state (`Object[]` from current field values) against the stored snapshot, element by element.
3. If any element differs, the entity is scheduled for UPDATE.

```
Persistence Context:
  Entity Map: { EntityKey(User, 42) → User@abc }
  Snapshot Map: { EntityKey(User, 42) → ["John", "john@email.com", ...] }

At flush:
  currentState = [user.getName(), user.getEmail(), ...]  // Reflect current values
  snapshot     = ["John", "john@email.com", ...]          // Loaded state

  Compare element by element → detect changes
```

### Performance Cost

- **O(n × m)** where n = number of managed entities, m = number of properties per entity
- For 10,000 entities with 20 fields each: 200,000 comparisons per flush
- This is why long-running sessions with many entities cause performance degradation

### Mitigation Strategies

1. **Read-only entities:** `@Immutable` or `session.setDefaultReadOnly(true)` — Hibernate skips dirty checking entirely
2. **Read-only transactions:** `@Transactional(readOnly = true)` — Spring hints to Hibernate to skip dirty checking
3. **DTO projections:** No entities in the persistence context, no dirty checking
4. **`session.clear()`:** In batch processing, periodically clear the persistence context
5. **Bytecode enhancement for dirty tracking:** Instead of comparing snapshots, Hibernate instruments setter methods to flag the entity as dirty at modification time — O(1) instead of O(n×m)

```properties
# Enable bytecode-enhanced dirty tracking
hibernate.enhancer.enableDirtyTracking=true
```

---

## C.5 Flush Modes

**Interview Question:** *"What are the flush modes and when would you change the default?"*

| Mode | Behavior | SQL Timing |
|------|----------|-----------|
| `AUTO` (default) | Flush before queries that would be affected by pending changes | Before each relevant query |
| `COMMIT` | Flush only at transaction commit | At commit |
| `ALWAYS` | Flush before every query (even unrelated ones) | Before every query |
| `MANUAL` | Never auto-flush — you control when | Only on explicit `flush()` |

**Why change from AUTO:**

- **`COMMIT` mode** in read-heavy transactions: Avoids unnecessary flushes before read queries. If you know your reads don't depend on unflushed writes, `COMMIT` reduces flush overhead.
- **`MANUAL` mode** for batch processing: Full control over when SQL is generated. Combine with periodic `flush()/clear()`.

**Danger of `COMMIT`:** If you have unflushed changes and then run a query, the query results won't reflect your pending changes. This can cause subtle bugs.

```java
user.setEmail("new@example.com");  // Not flushed yet

// With FlushMode.COMMIT, this query returns the OLD email
User found = em.createQuery("SELECT u FROM User u WHERE u.email = :e")
    .setParameter("e", "new@example.com")
    .getSingleResult();  // Returns null or wrong result!
```

---

## C.6 Pagination Pitfalls

**Interview Question:** *"What is the 'HHH90003004: firstResult/maxResults specified with collection fetch' warning? How do you handle pagination correctly?"*

### The Problem

```java
// WARNING: in-memory pagination!
SELECT o FROM Order o JOIN FETCH o.items
  → setFirstResult(0).setMaxResults(10)
```

When you combine `JOIN FETCH` on a collection with `setFirstResult/setMaxResults`, Hibernate **cannot apply LIMIT/OFFSET in SQL** because the join multiplies the rows. A single Order with 5 items produces 5 rows. LIMIT 10 would cut across entity boundaries.

Hibernate falls back to **in-memory pagination**: it loads ALL results, then slices in Java. With 1M rows, this means loading 1M rows into memory.

### Solutions

**1. Two-query approach (Window Query):**
```java
// Query 1: Get IDs with pagination (no collection fetch)
List<Long> ids = em.createQuery(
    "SELECT o.id FROM Order o ORDER BY o.date DESC", Long.class)
    .setFirstResult(0).setMaxResults(10)
    .getResultList();

// Query 2: Fetch full entities with collections for those IDs
List<Order> orders = em.createQuery(
    "SELECT DISTINCT o FROM Order o JOIN FETCH o.items WHERE o.id IN :ids", Order.class)
    .setParameter("ids", ids)
    .getResultList();
```

**2. @BatchSize on the collection:**
```java
// Paginate orders normally (no join fetch), let @BatchSize handle items
@BatchSize(size = 25)
@OneToMany(mappedBy = "order")
private Set<OrderItem> items;
```

**3. Keyset pagination (for very large datasets):**
```java
// No OFFSET — use WHERE clause on the last seen key
SELECT o FROM Order o WHERE o.date < :lastDate ORDER BY o.date DESC
```

Keyset avoids OFFSET entirely (OFFSET scans and discards rows). For deep pagination, this is orders of magnitude faster.

---

## C.7 Transaction Isolation Impact

**Interview Question:** *"How does transaction isolation level affect Hibernate's behavior?"*

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read | Hibernate Impact |
|----------------|-----------|-------------------|-------------|-----------------|
| READ_UNCOMMITTED | Yes | Yes | Yes | Rarely used. L1 cache may mask dirty reads within a session |
| READ_COMMITTED | No | Yes | Yes | Most common default. L1 cache provides repeatable reads **within a session** |
| REPEATABLE_READ | No | No | Yes | Database provides guarantees similar to Hibernate's L1 cache |
| SERIALIZABLE | No | No | No | Maximum consistency, lowest throughput |

**Key insight:** Hibernate's first-level cache provides **application-level repeatable read** within a session regardless of the database isolation level. If you load entity A, then load it again, the second load returns the same Java instance from the identity map — even if the database row changed.

This means at `READ_COMMITTED`, two `em.find(User.class, 42)` calls in the same session always return the same object. But a JPQL query might return different result sets between calls (phantom reads).

---

# D. Hibernate Internals

---

## D.1 First-Level Cache (Persistence Context) Deep Dive

**Interview Question:** *"What data structures does Hibernate use for the first-level cache? What guarantees does it provide?"*

### Internal Data Structures

```
PersistenceContext (StatefulPersistenceContext):
├── entitiesByKey: Map<EntityKey, Object>          // Identity map
├── entitySnapshotsByKey: Map<EntityKey, Object[]>  // Dirty checking snapshots
├── entityEntryContext: Map<Object, EntityEntry>    // Entity metadata (status, dirty flags, etc.)
├── collectionsByKey: Map<CollectionKey, PersistentCollection>
├── collectionEntries: Map<PersistentCollection, CollectionEntry>
├── arrayHolders: Map                               // For persistent arrays
├── nullifiableEntityKeys: Set<EntityKey>            // Entities that could be nullified
└── proxiesByKey: Map<EntityKey, Object>             // Uninitialized proxies
```

### Guarantees

1. **Identity guarantee:** `em.find(User.class, 1) == em.find(User.class, 1)` — same Java reference
2. **Repeatable read:** Within a session, the same entity always returns the same state
3. **Automatic dirty detection:** Changes to managed entities are detected and synchronized

### Memory Implications

Each managed entity consumes ~2x memory: the entity itself + the snapshot array. For a session managing 10,000 entities with 10 fields each, that's 10,000 extra `Object[10]` arrays. This is why `session.clear()` is critical in batch processing and why read-only entities (`@Immutable`, `readOnly = true`) should skip snapshot creation.

---

## D.2 Proxy & Bytecode Enhancement

**Interview Question:** *"How does lazy loading work at the bytecode level? What's the difference between proxies and bytecode enhancement?"*

### Proxy-Based Lazy Loading (Default)

Hibernate generates a **runtime subclass** of your entity using ByteBuddy:

```java
// What Hibernate creates at runtime:
class User$HibernateProxy extends User implements HibernateProxy {
    private LazyInitializer handler;

    @Override
    public String getName() {
        handler.initialize();  // Triggers SELECT if not loaded
        return handler.getImplementation().getName();
    }
}
```

**Implications:**
- Entity classes cannot be `final`
- `instanceof` works (the proxy IS-A User)
- `getClass()` returns the proxy class, not the entity class — use `Hibernate.getClass()` instead
- `equals()`/`hashCode()` must use getters, not field access (fields on the proxy are null until initialized)

**The `equals()` trap:**
```java
// BROKEN with proxies — direct field access bypasses the proxy
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User)) return false;  // Use instanceof, not getClass()
    User user = (User) o;
    return Objects.equals(id, user.id);  // WRONG: user.id is null on proxy!
}

// CORRECT — use getters
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof User)) return false;
    User user = (User) o;
    return Objects.equals(getId(), user.getId());  // Triggers initialization
}
```

### Bytecode Enhancement

An alternative approach where Hibernate modifies entity class bytecode at build time:

```xml
<!-- Maven plugin -->
<plugin>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-enhance-maven-plugin</artifactId>
    <executions>
        <execution>
            <configuration>
                <enableLazyInitialization>true</enableLazyInitialization>
                <enableDirtyTracking>true</enableDirtyTracking>
            </configuration>
        </execution>
    </executions>
</plugin>
```

**Advantages over proxies:**
- Enables true lazy loading on `@OneToOne` non-owning side
- Enables lazy loading of basic fields (e.g., large BLOB/CLOB)
- Dirty tracking via field interception (O(1) vs O(n) snapshot comparison)
- No proxy issues with `getClass()`, `final` classes, etc.

---

## D.3 Write-Behind & Action Queue

**Interview Question:** *"What is the write-behind mechanism and in what order does Hibernate execute SQL statements?"*

### Write-Behind

Hibernate does **not** execute SQL immediately when you call `persist()`, `remove()`, or modify an entity. Instead, it queues **actions** and executes them at flush time.

### SQL Execution Order

Hibernate flushes SQL in this specific order to avoid FK constraint violations:

```
1. OrphanRemovalAction      (delete orphaned children)
2. EntityInsertAction        (INSERT parent entities)
3. EntityUpdateAction        (UPDATE entities)
4. CollectionRemoveAction    (remove collection elements)
5. CollectionUpdateAction    (update collection elements)
6. CollectionRecreateAction  (recreate collections)
7. EntityDeleteAction        (DELETE entities)
```

**Why this order?** Inserts happen before deletes because:
- A new child may reference a new parent (parent INSERT must come first)
- Deleting a parent before its children violates FK constraints
- Within inserts, Hibernate topologically sorts by dependency

**When flush happens (AUTO mode):**
1. Before query execution (if the query's entity space overlaps with pending changes)
2. Before transaction commit
3. On explicit `em.flush()` call

---

## D.4 SQL Generation Process

**Interview Question:** *"How does Hibernate generate SQL from a JPQL query?"*

### Pipeline

```
JPQL String
  → ANTLR Parser → AST (Abstract Syntax Tree)
  → Semantic Analysis (resolve entity names, validate paths)
  → HQL AST → SQL AST transformation
  → SQL AST → SQL String (dialect-specific)
  → PreparedStatement
```

**Query Plan Cache:** Hibernate caches the compiled query plan (JPQL → SQL AST) to avoid re-parsing. This is keyed by the query string. Using inline parameters instead of bind parameters bypasses the cache:

```java
// BAD: Creates new query plan for each value, fills the plan cache
em.createQuery("SELECT u FROM User u WHERE u.id = " + id);

// GOOD: Parameterized — single cached plan
em.createQuery("SELECT u FROM User u WHERE u.id = :id")
  .setParameter("id", id);
```

The query plan cache has a finite size (`hibernate.query.plan_cache_max_size`, default 2048). If overwhelmed by dynamic queries, it churns, causing constant re-parsing. This shows up as high CPU usage with no obvious cause.

---

# E. Transactions & Concurrency

---

## E.1 Optimistic Locking

**Interview Question:** *"Explain optimistic locking with @Version. What SQL does Hibernate generate and what happens on a conflict?"*

### Mechanism

```java
@Entity
public class Account {
    @Id private Long id;
    @Version private int version;  // Can be int, Integer, long, Long, short, Timestamp
    private BigDecimal balance;
}
```

### SQL Generated

```sql
-- Update includes version check in WHERE clause
UPDATE account
SET balance = ?, version = 2
WHERE id = ? AND version = 1

-- If 0 rows updated → someone else changed it → OptimisticLockException
```

### Conflict Scenario

```
Thread A: loads Account (version=1, balance=1000)
Thread B: loads Account (version=1, balance=1000)

Thread A: sets balance=900, commits
  → UPDATE ... SET balance=900, version=2 WHERE id=1 AND version=1
  → 1 row updated ✓

Thread B: sets balance=800, commits
  → UPDATE ... SET balance=800, version=2 WHERE id=1 AND version=1
  → 0 rows updated → OptimisticLockException ✗
```

### Handling the Exception

```java
try {
    accountService.transfer(fromId, toId, amount);
} catch (OptimisticLockException e) {
    // Reload and retry — or inform the user
    // DO NOT catch silently — that hides data conflicts
}
```

### Versionless Optimistic Locking

For legacy schemas without a version column:

```java
@Entity
@OptimisticLocking(type = OptimisticLockType.DIRTY)
@DynamicUpdate  // Required — only include changed columns in WHERE
public class LegacyEntity { ... }
```

SQL: `UPDATE ... SET name = ? WHERE id = ? AND name = 'old_value'`

**Trade-off:** More columns in WHERE clause, less reliable than a version column.

---

## E.2 Pessimistic Locking

**Interview Question:** *"When would you choose pessimistic over optimistic locking? Show me the SQL difference."*

```java
// Pessimistic read lock — blocks writes, allows reads
Account a = em.find(Account.class, id, LockModeType.PESSIMISTIC_READ);
// SQL: SELECT ... FROM account WHERE id = ? FOR SHARE

// Pessimistic write lock — blocks reads and writes
Account a = em.find(Account.class, id, LockModeType.PESSIMISTIC_WRITE);
// SQL: SELECT ... FROM account WHERE id = ? FOR UPDATE

// With timeout (avoid infinite waits)
Map<String, Object> hints = new HashMap<>();
hints.put("jakarta.persistence.lock.timeout", 3000);  // 3 seconds
Account a = em.find(Account.class, id, LockModeType.PESSIMISTIC_WRITE, hints);
```

### Decision Framework

| Factor | Optimistic | Pessimistic |
|--------|-----------|-------------|
| Conflict frequency | Low | High |
| Read/write ratio | Read-heavy | Write-heavy |
| Transaction duration | Short | Short (MUST be short) |
| User experience | Retry on conflict | Wait for lock |
| Scalability | Better | Worse (locks reduce concurrency) |
| Deadlock risk | None | Real |

**Senior insight:** In a typical web application, optimistic locking is the default choice. Pessimistic locking is reserved for:
- Financial transactions where conflicts are guaranteed (account balance updates)
- Inventory systems with high contention (concert ticket reservations)
- Scenarios where retry cost is higher than wait cost

---

## E.3 Lost Update Problem

**Interview Question:** *"Describe the lost update problem and how Hibernate prevents it at each isolation level."*

### Scenario Without Protection

```
Thread A: SELECT balance FROM account WHERE id = 1  → 1000
Thread B: SELECT balance FROM account WHERE id = 1  → 1000

Thread A: UPDATE balance = 1000 - 100 = 900  → commits
Thread B: UPDATE balance = 1000 - 200 = 800  → commits  // Thread A's update is LOST
```

### Prevention Strategies

1. **@Version (optimistic):** Thread B's commit fails because version mismatch
2. **PESSIMISTIC_WRITE:** Thread B blocks until Thread A commits, then sees balance = 900
3. **REPEATABLE_READ isolation:** Database detects the conflict (behavior varies by DB)
4. **SERIALIZABLE:** Full serialization of transactions — guaranteed no conflicts

**In practice:** Use `@Version`. It's simple, scalable, and works across all databases consistently.

---

## E.4 Transaction Propagation (Spring Integration)

**Interview Question:** *"Explain REQUIRED vs REQUIRES_NEW. What happens to the Hibernate session in each case?"*

| Propagation | Behavior | Hibernate Session |
|------------|----------|------------------|
| `REQUIRED` (default) | Join existing tx or create new | Same session if tx exists |
| `REQUIRES_NEW` | Suspend current tx, create new | **New session** — separate persistence context |
| `NESTED` | Savepoint within current tx | Same session, savepoint |
| `SUPPORTS` | Use tx if exists, else non-tx | Session may or may not be transactional |
| `NOT_SUPPORTED` | Suspend current tx, run non-tx | Session without transaction |

**`REQUIRES_NEW` danger:**

```java
@Service
public class OrderService {
    @Transactional
    public void createOrder(Order order) {
        em.persist(order);
        auditService.logCreation(order);  // REQUIRES_NEW
        // If this method fails, order creation ALSO needs to be rolled back?
        // With REQUIRES_NEW, audit runs in a SEPARATE tx
        // Order creation commits even if audit fails
    }
}

@Service
public class AuditService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logCreation(Order order) {
        // Runs in new tx, new session
        // Cannot see the unpersisted order from the outer tx!
        // em.find(Order.class, order.getId()) → NULL (if outer tx hasn't flushed/committed)
    }
}
```

**Critical trap:** `REQUIRES_NEW` creates a new persistence context. It cannot see uncommitted changes from the outer transaction. The new session will read from the database, not from the outer session's persistence context.

---

# F. Querying

---

## F.1 HQL vs JPQL

**Interview Question:** *"What can HQL do that JPQL cannot?"*

JPQL is a subset of HQL. HQL adds:

- **List parameter support (older versions):** `WHERE id IN (:ids)` with `List` parameter
- **`INSERT ... SELECT`:** `INSERT INTO Archive(id, data) SELECT id, data FROM Entity WHERE ...`
- **Hibernate-specific functions:** `str()`, `year()`, `month()`, etc.
- **Polymorphic queries on non-mapped types:** `FROM java.lang.Object` (loads all entities)
- **`WITH` clause for join conditions** (now also in JPQL with `ON`)
- **`@Formula` column access**

**In practice:** The differences are minor. Use JPQL for portability, HQL when you need Hibernate-specific features.

---

## F.2 Criteria API

**Interview Question:** *"When would you use the Criteria API over JPQL? What are the drawbacks?"*

### When to Use

- **Dynamic queries:** When WHERE clauses are built at runtime based on user input (search filters, faceted search)
- **Type safety:** Compile-time checking with the metamodel

```java
CriteriaBuilder cb = em.getCriteriaBuilder();
CriteriaQuery<Order> cq = cb.createQuery(Order.class);
Root<Order> root = cq.from(Order.class);

List<Predicate> predicates = new ArrayList<>();
if (status != null) {
    predicates.add(cb.equal(root.get(Order_.status), status));
}
if (minAmount != null) {
    predicates.add(cb.greaterThan(root.get(Order_.amount), minAmount));
}
if (customerId != null) {
    predicates.add(cb.equal(root.get(Order_.customer).get(Customer_.id), customerId));
}

cq.where(predicates.toArray(new Predicate[0]));
return em.createQuery(cq).getResultList();
```

### Drawbacks

- **Verbosity:** Simple queries are 5-10x more code than JPQL
- **Readability:** Hard to scan and understand
- **Query plan cache:** Dynamic criteria queries may produce unique query strings, reducing plan cache efficiency
- **Debugging:** Harder to log and trace than a string query

**Senior recommendation:** Use JPQL for static queries, Criteria API only for genuinely dynamic queries. For very complex dynamic queries, consider a query builder library like QueryDSL (generates Criteria API calls with much better readability).

---

## F.3 Native Queries

**Interview Question:** *"When do you resort to native SQL in a Hibernate application? What do you lose?"*

### When to Use

- Database-specific features (window functions, CTEs, JSON operators, full-text search)
- Existing optimized SQL that cannot be expressed in JPQL
- Bulk operations that bypass Hibernate's entity lifecycle

### What You Lose

- **Portability** across databases
- **Entity state management** (for non-entity results)
- **First-level cache integration** (partially — `@SqlResultSetMapping` helps)
- **Automatic dirty checking** on non-managed results

```java
// Native query with entity mapping (entities ARE managed)
List<Order> orders = em.createNativeQuery(
    "SELECT * FROM orders WHERE status = :status", Order.class)
    .setParameter("status", "PENDING")
    .getResultList();  // These ARE managed entities

// Native query with DTO mapping (not managed)
List<Object[]> results = em.createNativeQuery(
    "SELECT o.id, c.name FROM orders o JOIN customer c ON o.customer_id = c.id")
    .getResultList();
```

**Danger of bulk native queries:**

```java
em.createNativeQuery("UPDATE orders SET status = 'ARCHIVED' WHERE created < :date")
    .setParameter("date", cutoff)
    .executeUpdate();
// WARNING: This bypasses the persistence context entirely
// Any cached Order entities in the session now have STALE data
// You MUST call em.clear() after bulk native operations
```

---

## F.4 DTO Projections

**Interview Question:** *"Why are DTO projections better for read-only queries? What are the options?"*

### Why DTOs Are Superior for Reads

| Aspect | Entity Query | DTO Projection |
|--------|-------------|---------------|
| Persistence context | Entity managed, snapshot stored | Nothing stored |
| Dirty checking | Yes (at flush time) | N/A |
| Memory | 2x (entity + snapshot) | 1x (just the DTO) |
| Columns fetched | All columns (SELECT *) | Only needed columns |
| Post-processing | Full entity hydration | Direct constructor call |

### Approaches

```java
// 1. Constructor expression (JPQL)
@Query("SELECT new com.example.OrderSummary(o.id, o.date, c.name) " +
       "FROM Order o JOIN o.customer c")
List<OrderSummary> findSummaries();

// 2. Interface projection (Spring Data)
public interface OrderSummary {
    Long getId();
    LocalDate getDate();
    String getCustomerName();  // SpEL or @Value for nested
}
@Query("SELECT o.id AS id, o.date AS date, c.name AS customerName ...")
List<OrderSummary> findSummaries();

// 3. Tuple query
List<Tuple> results = em.createQuery(
    "SELECT o.id, o.date, c.name FROM Order o JOIN o.customer c", Tuple.class)
    .getResultList();
results.forEach(t -> {
    Long id = t.get(0, Long.class);
    String name = t.get(2, String.class);
});
```

**Senior insight:** For read-heavy APIs (dashboards, reports, lists), always use DTO projections. Reserve entity queries for operations that need to modify data.

---

# G. Spring Boot + Hibernate Integration

---

## G.1 How Spring Manages Hibernate Sessions

**Interview Question:** *"Walk me through what happens from when a Spring @Transactional method is called to when the SQL hits the database."*

### Execution Flow

```
1. Client calls @Transactional method
2. Spring AOP proxy intercepts
3. TransactionInterceptor.invoke()
4. PlatformTransactionManager.getTransaction()
5. → HibernateTransactionManager (or JpaTransactionManager)
   a. Gets SessionFactory → opens new Session
   b. Binds Session to ThreadLocal (TransactionSynchronizationManager)
   c. Begins JDBC transaction on the Session's connection
6. Target method executes
   a. EntityManager operations use the bound Session
   b. Changes accumulate in the persistence context
7. Method returns normally → TransactionManager.commit()
   a. Session.flush() → dirty checking → SQL generation → JDBC execution
   b. Connection.commit()
   c. Session.close()
   d. Session unbound from ThreadLocal

   OR method throws → TransactionManager.rollback()
   a. Connection.rollback()
   b. Session.close() (persistence context is discarded)
   c. Session unbound from ThreadLocal
```

**Key detail:** The `EntityManager` injected via `@PersistenceContext` is a **shared proxy**. It delegates to the actual Session bound to the current thread. Multiple beans in the same transaction share the same Session.

---

## G.2 Open Session in View (OSIV)

**Interview Question:** *"What is the Open Session in View pattern? Why does Spring enable it by default and why should you disable it?"*

### What It Does

`spring.jpa.open-in-view=true` (default) registers an `OpenEntityManagerInViewInterceptor` that opens a Session at the start of the HTTP request and closes it after the view is rendered — **even outside @Transactional boundaries**.

```
HTTP Request arrives
  → OSIV opens Session (persistence context)
    → Controller executes
      → @Transactional service executes (uses the same Session)
        → Transaction commits
      → Controller accesses lazy collections (Session still open!)
    → View renders (Session still open for lazy loading)
  → OSIV closes Session
```

### Why It's Dangerous in Production

1. **Database connection held for entire request:** From request start to response end, including view rendering. Under load, this exhausts the connection pool.
2. **Hides N+1 problems:** Lazy loading "just works" in the controller/view, so developers don't optimize queries.
3. **Non-deterministic query execution:** SQL fires unpredictably based on what the view accesses.
4. **Mixes concerns:** The web layer triggers database queries, violating clean architecture.

### Recommendation

```properties
spring.jpa.open-in-view=false
```

Then handle lazy loading explicitly:
- Use JOIN FETCH in service layer queries
- Use DTO projections for API responses
- Use `@EntityGraph` on repository methods
- Accept `LazyInitializationException` as a signal that your query is incomplete

---

## G.3 @Transactional Best Practices

**Interview Question:** *"What are the common @Transactional pitfalls in Spring Boot?"*

### Pitfall 1: Self-Invocation Bypass

```java
@Service
public class OrderService {
    public void process(Long id) {
        validate(id);
        createOrder(id);  // @Transactional is IGNORED — self-invocation
    }

    @Transactional
    public void createOrder(Long id) { ... }
}
```

AOP proxy is bypassed on self-invocation because `this.createOrder()` calls the actual method, not the proxy. Solutions:
- Inject self: `@Autowired OrderService self; self.createOrder(id);`
- Extract to a separate service
- Use AspectJ weaving (compile-time or load-time)

### Pitfall 2: Catching Exceptions Silently

```java
@Transactional
public void transfer() {
    try {
        debit();
        credit();
    } catch (Exception e) {
        log.error("Failed", e);
        // Transaction is NOT rolled back! Spring only rolls back on uncaught exceptions
    }
}
```

Fix: Re-throw the exception, or use `@Transactional(rollbackFor = Exception.class)`.

### Pitfall 3: readOnly Misunderstanding

```java
@Transactional(readOnly = true)
public void updateUser(User user) {
    user.setName("new name");
    // In some configurations, this change IS persisted!
    // readOnly=true is a hint, not enforcement
}
```

`readOnly = true` does:
- Tells Hibernate to set `FlushMode.MANUAL` (in Spring) — suppresses auto-flush
- Tells JDBC driver to potentially use read-replica
- Tells Hibernate to skip dirty checking snapshot storage (optimization)

It does **not** make the database transaction read-only in all databases.

### Pitfall 4: Wrong Exception Rollback

```java
@Transactional  // Default: rollback only on RuntimeException and Error
public void process() throws BusinessException {
    throw new BusinessException("...");  // Checked exception — NO rollback!
}

@Transactional(rollbackFor = Exception.class)  // Fix: rollback on all exceptions
public void process() throws BusinessException { ... }
```

---

# H. Real-World Scenario-Based Questions

---

## H.1 High-Traffic Performance Issue

**Interview Question:** *"Your API endpoint that lists orders is responding in 5 seconds under load. Investigation shows 200+ SQL queries per request. How do you diagnose and fix this?"*

### Ideal Senior-Level Answer

**Step 1: Enable SQL logging and statistics**
```properties
hibernate.generate_statistics=true
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.orm.jdbc.bind=TRACE
```

**Step 2: Identify the N+1 queries**
Look for patterns like:
```sql
SELECT * FROM orders WHERE ...          -- 1 query
SELECT * FROM customer WHERE id = ?     -- repeated N times
SELECT * FROM order_item WHERE order_id = ?  -- repeated N times
```

**Step 3: Analyze the entity graph being loaded**
Map which associations are being accessed in the controller/serializer. Common culprit: Jackson serialization triggering lazy loads.

**Step 4: Apply fixes (in priority order)**
1. **DTO projection** for the listing endpoint — don't load entities at all
2. **JOIN FETCH** for necessary associations
3. **`@BatchSize(size = 25)`** on frequently accessed collections
4. **Disable OSIV** to force explicit fetching
5. **Second-level cache** for reference data (countries, categories, statuses)
6. **Database indexes** on FK columns and commonly filtered fields

**Step 5: Validate with query count assertions in tests**

### Follow-Up Questions
- "What if the endpoint also writes data?"
- "How would you handle this differently with pagination?"
- "What monitoring would you add to prevent regression?"

---

## H.2 Hibernate Memory Leak

**Interview Question:** *"Your application's heap keeps growing and eventually OOMs. Heap dump shows millions of entity instances held by Hibernate. What happened and how do you fix it?"*

### Ideal Senior-Level Answer

**Most likely causes:**

1. **Long-running Session accumulating entities:**
   ```java
   // Reading a CSV and persisting 10M rows in one transaction
   @Transactional
   public void importData(File csv) {
       for (Row row : readCsv(csv)) {
           em.persist(mapToEntity(row));
           // Persistence context grows unbounded!
       }
   }
   ```
   **Fix:** Periodic `flush()/clear()` with `@Modifying(clearAutomatically = true)` or manual batching.

2. **OSIV holding Session across request processing:**
   A slow downstream service call while OSIV holds the Session open → entities can't be GC'd.

3. **Second-level cache without eviction policy:**
   Unbounded cache region fills memory.

4. **Stateful session bean holding entity references:**
   HTTP session or singleton bean holding detached entities that reference lazy collections backed by the session.

**Diagnostic approach:**
- Heap dump → find `StatefulPersistenceContext` instances → check `entitiesByKey` map size
- Check `Session` lifecycle — is it being closed properly?
- Look for `ThreadLocal` leaks in thread pools (Session bound to worker thread that's recycled)

### Fix for Batch Processing

```java
@Transactional
public void importData(File csv) {
    int batch = 0;
    for (Row row : readCsv(csv)) {
        em.persist(mapToEntity(row));
        if (++batch % 50 == 0) {
            em.flush();
            em.clear();  // Release persistence context
        }
    }
}
```

Or better — use `StatelessSession`:
```java
StatelessSession session = sessionFactory.openStatelessSession();
Transaction tx = session.beginTransaction();
for (Row row : readCsv(csv)) {
    session.insert(mapToEntity(row));  // No persistence context at all
}
tx.commit();
session.close();
```

---

## H.3 Cache Inconsistency

**Interview Question:** *"In a clustered environment, users sometimes see stale data even after updating. Your application uses Hibernate's second-level cache. What's happening?"*

### Ideal Senior-Level Answer

**Root cause:** The L2 cache is **local to each JVM** by default (e.g., EhCache in local mode). When Node A updates an entity, it invalidates its local cache, but Node B's cache still holds the old version.

**Solutions:**

1. **Distributed cache provider:** Use a cache that supports distributed invalidation.
   - **Hazelcast**, **Infinispan**, or **Redis** (via Redisson Hibernate integration)
   - These either replicate or invalidate cache entries across nodes

2. **Cache-aside pattern:** Don't use L2 cache for frequently-updated entities. Use it only for reference data that changes rarely and where brief staleness is acceptable.

3. **TTL-based expiration:** Set short time-to-live on cache regions:
   ```xml
   <cache name="com.example.Product" maxEntriesLocalHeap="10000"
          timeToLiveSeconds="300" />
   ```

4. **Explicit cache eviction on write:**
   ```java
   sessionFactory.getCache().evictEntityData(Product.class, productId);
   ```

**Key insight:** The L2 cache trades consistency for performance. In a clustered environment, you must choose: distributed cache (more complexity, network overhead) or accept eventual consistency (TTL-based). For most applications, the simplest correct approach is to not use L2 cache for mutable entities in clustered deployments.

---

## H.4 Deadlocks

**Interview Question:** *"Your application logs show database deadlocks during order processing. How do you diagnose and resolve this?"*

### Ideal Senior-Level Answer

**Common causes in Hibernate applications:**

1. **Inconsistent entity access ordering:**
   ```
   Thread A: UPDATE order SET ... WHERE id = 1 (locks row 1)
   Thread A: UPDATE order_item SET ... WHERE id = 10 (waits for row 10)

   Thread B: UPDATE order_item SET ... WHERE id = 10 (locks row 10)
   Thread B: UPDATE order SET ... WHERE id = 1 (waits for row 1)
   → DEADLOCK
   ```

2. **Hibernate's flush ordering not matching expectations:**
   Hibernate sorts SQL by type (INSERT → UPDATE → DELETE), not by business logic. This can cause lock acquisition in an unexpected order.

**Diagnostic steps:**
- Check database deadlock logs (MySQL: `SHOW ENGINE INNODB STATUS`, PostgreSQL: `log_lock_waits = on`)
- Identify which SQL statements are involved
- Trace back to the Hibernate operations that generated them

**Solutions:**

1. **Consistent ordering:** Always process entities in the same order (e.g., by ID)
   ```java
   List<Long> sortedIds = ids.stream().sorted().collect(toList());
   for (Long id : sortedIds) {
       // Process in consistent order across all threads
   }
   ```

2. **Pessimistic locking with timeout:**
   ```java
   em.find(Order.class, id, LockModeType.PESSIMISTIC_WRITE,
       Map.of("jakarta.persistence.lock.timeout", 3000));
   ```

3. **Retry on deadlock:**
   ```java
   @Retryable(value = DeadlockLoserDataAccessException.class, maxAttempts = 3)
   @Transactional
   public void processOrder(Long orderId) { ... }
   ```

4. **Reduce transaction scope:** Shorter transactions hold locks for less time, reducing deadlock windows.

---

## H.5 Multi-Tenancy

**Interview Question:** *"How would you implement multi-tenancy with Hibernate? What are the trade-offs of each approach?"*

### Approaches

| Strategy | Isolation | Complexity | Resource Usage |
|----------|----------|-----------|---------------|
| **Separate database** per tenant | Highest | High (connection management) | Highest |
| **Separate schema** per tenant | High | Medium | Medium |
| **Discriminator column** (shared tables) | Lowest | Lowest | Lowest |

### Hibernate Implementation

```java
// Discriminator-based (Hibernate 6+)
@Entity
@TenantId("tenantId")  // Hibernate 6 annotation
public class Order {
    private String tenantId;  // Automatically filtered
}

// Tenant resolver
@Component
public class TenantResolver implements CurrentTenantIdentifierResolver {
    @Override
    public String resolveCurrentTenantIdentifier() {
        return TenantContext.getCurrentTenant();  // ThreadLocal
    }

    @Override
    public boolean validateExistingCurrentSessions() {
        return true;
    }
}
```

**For schema/database strategy:**
```java
@Component
public class TenantConnectionProvider implements MultiTenantConnectionProvider {
    @Override
    public Connection getConnection(String tenantIdentifier) {
        Connection conn = dataSource.getConnection();
        conn.setSchema(tenantIdentifier);  // Schema strategy
        return conn;
    }
}
```

**Discriminator column pitfalls:**
- Every query gets `AND tenant_id = ?` appended — index the column
- JOIN queries must filter both sides
- Native queries bypass the filter — security risk
- Bulk operations must include the tenant filter manually

---

## H.6 Large Dataset Processing

**Interview Question:** *"You need to process 10 million records from a Hibernate-managed table. How do you do it without running out of memory?"*

### Approaches

**1. ScrollableResults (cursor-based):**
```java
StatelessSession session = sessionFactory.openStatelessSession();
Transaction tx = session.beginTransaction();

try (ScrollableResults<Order> scroll = session.createQuery(
        "FROM Order WHERE status = :status", Order.class)
        .setParameter("status", "PENDING")
        .setFetchSize(1000)          // JDBC fetch size
        .scroll(ScrollMode.FORWARD_ONLY)) {

    while (scroll.next()) {
        Order order = scroll.get();
        processOrder(order);
        // No persistence context — no memory accumulation
    }
}
tx.commit();
session.close();
```

**2. Stream API (Hibernate 5.2+):**
```java
@Transactional(readOnly = true)
public void processAll() {
    try (Stream<Order> stream = em.createQuery(
            "FROM Order WHERE status = :status", Order.class)
            .setParameter("status", "PENDING")
            .setHint(QueryHints.HINT_FETCH_SIZE, 1000)
            .getResultStream()) {

        stream.forEach(this::processOrder);
    }
    // WARNING: With a standard Session, persistence context still grows!
    // Use readOnly=true to at least skip snapshots
}
```

**3. Chunk-based with ID pagination:**
```java
Long lastId = 0L;
int chunkSize = 1000;

while (true) {
    List<Order> chunk = em.createQuery(
        "FROM Order WHERE id > :lastId ORDER BY id", Order.class)
        .setParameter("lastId", lastId)
        .setMaxResults(chunkSize)
        .getResultList();

    if (chunk.isEmpty()) break;

    chunk.forEach(this::processOrder);
    lastId = chunk.get(chunk.size() - 1).getId();

    em.flush();
    em.clear();  // Release persistence context
}
```

**Key decisions:**
- **StatelessSession** for pure reads — no persistence context overhead at all
- **Chunk + clear** for read-modify-write — controlled memory with dirty checking
- **Stream** for functional pipeline processing — elegant but watch persistence context size
- **JDBC fetch size** is critical — default (often 10 or all rows) can cause either too many round-trips or loading all rows at once

---

# Bonus Sections

---

## Most Common Tricky Interview Questions

### 1. "What happens if you call `equals()` on a proxy?"

The proxy's `equals()` delegates to the underlying entity's `equals()` — but **only if `equals()` is overridden**. Default `Object.equals()` compares references, and the proxy is a different reference than any previously loaded instance. This is why you **must** override `equals()/hashCode()` for entities used in collections.

### 2. "Can you explain what `select n+1` means if I have 3 levels of nesting?"

With Order → Customer → Address, loading 100 orders:
- 1 query for orders
- Up to 100 queries for customers (if not batched)
- Up to 100 queries for addresses (if not batched)
- Total: 1 + 100 + 100 = 201 queries

With `@BatchSize(size = 25)` on both associations: 1 + 4 + 4 = 9 queries.

### 3. "What's wrong with using Lombok's @Data on entities?"

- `@Data` generates `equals()/hashCode()` using all fields — including relationships, which can cause infinite loops and lazy loading triggers
- `@ToString` includes all fields — triggers lazy loading on collections
- `@EqualsAndHashCode` uses all fields — violates the contract when entity is transient (id is null) then persistent

**Use instead:** `@Getter @Setter` manually, implement `equals()/hashCode()` based on business key or use ID-based equality with `instanceof` check.

### 4. "What's the difference between `em.getReference()` and `em.find()`?"

- `find()` → Hits L1 cache, then L2 cache, then database. Returns the actual entity or `null`.
- `getReference()` → Returns a **proxy** without hitting the database. Throws `EntityNotFoundException` only when you access a non-ID property. Useful for setting FK relationships without loading the parent:

```java
// No SELECT for Department — just creates a proxy
Department dept = em.getReference(Department.class, deptId);
employee.setDepartment(dept);  // Only needs the ID for the FK
```

### 5. "Explain `MultipleBagFetchException`."

Hibernate throws this when you try to JOIN FETCH two `List` (bag) collections simultaneously. A bag has no ordering or uniqueness, so joining two bags creates a Cartesian product with no way to de-duplicate.

**Fix:** Change one (or both) collections to `Set`, or use two separate queries.

```java
// FAILS: MultipleBagFetchException
SELECT p FROM Post p JOIN FETCH p.comments JOIN FETCH p.tags
// Both comments and tags are List → Cartesian product

// FIX 1: Use Set instead of List
@OneToMany Set<Comment> comments;
@ManyToMany Set<Tag> tags;

// FIX 2: Two queries
List<Post> posts = em.createQuery("SELECT p FROM Post p JOIN FETCH p.comments").getResultList();
posts = em.createQuery("SELECT p FROM Post p JOIN FETCH p.tags WHERE p IN :posts")
    .setParameter("posts", posts).getResultList();
```

### 6. "Why should you avoid `TABLE` ID generation strategy?"

`GenerationType.TABLE` uses a separate database table to simulate sequences. It requires pessimistic locking on that table for each ID allocation, creating a serialization bottleneck. Under high concurrency, this table becomes a severe contention point.

**Prefer:** `SEQUENCE` with `allocationSize > 1` (e.g., 50). Hibernate pre-allocates IDs from the sequence, reducing database round-trips.

### 7. "How does Hibernate handle `hashCode()` for new (transient) entities?"

If `hashCode()` is based on `@Id` and the ID is `null` (not yet assigned), all transient entities have the same hash → hash collisions → `HashSet` degrades to a linked list.

**Best practice:** Use a natural business key for `equals()/hashCode()`, or use a UUID assigned at construction time.

```java
@Entity
public class Order {
    @Id @GeneratedValue
    private Long id;

    // Assigned at construction — stable across all entity states
    @Column(unique = true, updatable = false)
    private UUID uuid = UUID.randomUUID();

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        return uuid.equals(((Order) o).getUuid());
    }

    @Override
    public int hashCode() {
        return uuid.hashCode();
    }
}
```

---

## Hibernate vs Spring Data JPA

| Aspect | Hibernate (direct) | Spring Data JPA |
|--------|-------------------|----------------|
| API | `Session` / `EntityManager` | `JpaRepository` interface |
| Query definition | JPQL/HQL strings, Criteria API | Derived query methods, `@Query` |
| Boilerplate | Manual repository implementation | Zero — interface-only |
| Control | Full | Abstracted (can break through via `@Query(nativeQuery)` or custom implementations) |
| Performance tuning | Direct access to all Hibernate features | Some features require workarounds (e.g., `@QueryHints`, `@EntityGraph`) |
| Custom queries | Full flexibility | Good with `@Query`, limited with derived methods |
| Pagination | Manual | Built-in (`Pageable`, `Page<T>`) |
| Auditing | Manual or Envers | `@CreatedDate`, `@LastModifiedDate` via Spring Data auditing |

**Senior perspective:** Spring Data JPA is the right choice for 90% of CRUD operations. For complex queries, batch processing, or performance-critical paths, drop down to the `EntityManager` / `Session` API directly.

```java
// Custom implementation for complex cases
public interface OrderRepositoryCustom {
    List<Order> findOrdersWithComplexLogic(SearchCriteria criteria);
}

public class OrderRepositoryCustomImpl implements OrderRepositoryCustom {
    @PersistenceContext
    private EntityManager em;

    @Override
    public List<Order> findOrdersWithComplexLogic(SearchCriteria criteria) {
        // Full EntityManager access — Criteria API, native queries, etc.
    }
}

public interface OrderRepository extends JpaRepository<Order, Long>, OrderRepositoryCustom {
    // Spring Data methods + custom methods
}
```

---

## Hibernate vs MyBatis

| Aspect | Hibernate | MyBatis |
|--------|----------|---------|
| **Paradigm** | ORM (object-relational mapping) | SQL Mapper (SQL-centric) |
| **SQL Control** | Generated (can be overridden) | Full manual control |
| **Caching** | L1 + L2 + query cache | Simple local cache |
| **Lazy loading** | Built-in proxy mechanism | Limited, manual |
| **Dirty checking** | Automatic | None — you call update explicitly |
| **Schema evolution** | HBM2DDL / Flyway+Liquibase | Manual / Flyway+Liquibase |
| **Learning curve** | Steeper (must understand internals) | Simpler (just SQL + mapping XML/annotations) |
| **Best for** | Domain-rich apps, CRUD-heavy, standard schemas | Report-heavy, complex SQL, legacy schemas, DBA-controlled environments |
| **Performance floor** | Higher (more overhead) | Lower (thin layer over JDBC) |
| **Performance ceiling** | Higher (caching, batch optimization) | Depends on SQL quality |

**When to choose MyBatis:**
- DBA team writes and owns all SQL
- Complex reporting queries are the majority of operations
- Legacy database with non-standard schemas that fight ORM mapping
- Team has strong SQL skills but limited ORM experience

---

## Production Best Practices Checklist

```
□ All @ManyToOne and @OneToOne set to FetchType.LAZY
□ OSIV disabled (spring.jpa.open-in-view=false)
□ @Version on all mutable entities
□ equals()/hashCode() implemented correctly (business key or UUID)
□ SEQUENCE ID strategy with allocationSize > 1
□ JDBC batch size configured (hibernate.jdbc.batch_size=25-50)
□ order_inserts and order_updates enabled
□ hibernate.default_batch_fetch_size configured (16-32)
□ DTO projections for read-only API endpoints
□ Query count assertions in integration tests
□ No List bags on @OneToMany — use Set
□ Connection pool properly sized (HikariCP, min=max for predictability)
□ Slow query logging enabled at database level
□ hibernate.generate_statistics=true in staging/test
□ @Transactional(readOnly = true) on read-only service methods
□ No cascade=ALL on @ManyToMany
□ Flyway/Liquibase for schema migrations (never hbm2ddl in prod)
□ Second-level cache only for reference/read-heavy data
□ Monitor persistence context size in long-running operations
```

---

## Red Flags Interviewers Look For

1. **Cannot explain N+1** — The #1 disqualifier for senior roles
2. **Doesn't know LAZY vs EAGER defaults** — suggests lack of hands-on experience
3. **Never used DTO projections** — loading full entities for every API endpoint
4. **Cannot explain the persistence context** — doesn't understand Hibernate's core mechanism
5. **Uses `CascadeType.ALL` everywhere** — hasn't been burned by unintended deletes
6. **Doesn't understand owning side** — leads to FK values not being persisted
7. **Cannot discuss transaction isolation** — critical for correctness under concurrency
8. **Has never profiled Hibernate SQL** — has never worked on a system under load
9. **Confuses merge() with update()** — doesn't understand entity state transitions
10. **Uses `hbm2ddl.auto=update` in production** — immediate red flag for production awareness

---

## Whiteboard Explanation Examples

### "Draw how a @Transactional service call works"

```
Client → Spring AOP Proxy → TransactionInterceptor
  → Begin TX (get Connection from pool, set autocommit=false)
    → Bind Session to ThreadLocal
      → Execute service method
        → em.persist(entity) → add to persistence context + action queue
        → em.find(id) → check L1 cache → check L2 cache → SELECT
      → Return from method
    → Session.flush() → dirty check → generate SQL → execute batch
  → Connection.commit()
  → Session.close() → unbind from ThreadLocal → return Connection to pool
→ Return result to client
```

### "Draw the persistence context during a typical operation"

```
em.find(User.class, 1)
  Persistence Context:
  ┌─────────────────────────────────────────────────┐
  │ Identity Map: {User#1 → User@a1b2}             │
  │ Snapshots:    {User#1 → ["Alice", "alice@..."]} │
  └─────────────────────────────────────────────────┘

user.setName("Bob")  // Just changes the Java object

em.flush()  // Now Hibernate kicks in
  Current state: ["Bob", "alice@..."]
  Snapshot:      ["Alice", "alice@..."]
  → name differs → schedule UPDATE
  → SQL: UPDATE user SET name='Bob' WHERE id=1 AND version=1
```

---

# Mock Interview

---

## 20 Rapid-Fire Senior Questions

1. **What's the default fetch type for `@ManyToOne`?**
   `EAGER`. Always override to `LAZY`.

2. **What does `mappedBy` indicate?**
   The inverse (non-owning) side of a bidirectional relationship. The owning side controls the FK.

3. **What's the difference between `persist()` and `merge()`?**
   `persist()` makes a transient entity persistent (INSERT). `merge()` copies a detached entity's state into a new managed instance (SELECT + UPDATE).

4. **What causes `LazyInitializationException`?**
   Accessing an uninitialized lazy association after the Session is closed.

5. **How do you prevent N+1 queries?**
   JOIN FETCH, Entity Graphs, `@BatchSize`, DTO projections, or Subselect fetching.

6. **What SQL does `@Version` generate?**
   `UPDATE ... SET version = version + 1 WHERE id = ? AND version = ?`. Zero rows updated → `OptimisticLockException`.

7. **Why is `IDENTITY` strategy bad for batch inserts?**
   Hibernate must INSERT immediately to get the generated ID, preventing JDBC batching.

8. **What's the first-level cache?**
   The persistence context — an identity map within the Session. Guarantees one Java instance per database row per Session.

9. **What does `session.clear()` do?**
   Detaches all entities, clears snapshots, empties the identity map. Essential in batch processing.

10. **What's the difference between `@Embeddable` and `@Entity`?**
    Embeddable has no identity, no own table, lifecycle bound to owning entity. Entity has own identity, own table, independent lifecycle.

11. **When does Hibernate flush?**
    In AUTO mode: before relevant queries and at transaction commit. In COMMIT mode: only at commit.

12. **What's `orphanRemoval`?**
    Deletes child entity when removed from parent's collection. Only on `@OneToMany`/`@OneToOne`.

13. **How does dirty checking work?**
    At flush, Hibernate compares each entity's current state against its loaded snapshot. Differences trigger UPDATE.

14. **What's the query plan cache?**
    Caches parsed JPQL/HQL → SQL translation. String concatenation in queries bypasses it, causing re-parsing.

15. **What's a Hibernate proxy?**
    A runtime-generated subclass (ByteBuddy) that intercepts method calls to implement lazy loading.

16. **Why use `Set` instead of `List` for `@OneToMany`?**
    `List` (bag semantics) removes all elements and re-inserts on modification. `Set` does individual INSERT/DELETE.

17. **What does `@Transactional(readOnly = true)` do?**
    Tells Spring/Hibernate to skip dirty checking snapshots and set FlushMode.MANUAL.

18. **What's `MultipleBagFetchException`?**
    Thrown when fetching two or more `List` collections simultaneously. Fix: use `Set` or separate queries.

19. **What's the difference between `em.find()` and `em.getReference()`?**
    `find()` loads the entity immediately (or returns null). `getReference()` returns a proxy that loads on first non-ID access.

20. **What does `@DynamicUpdate` do?**
    Generates UPDATE with only changed columns instead of all columns. Useful with versionless optimistic locking or wide tables.

---

## 5 Deep-Dive Architecture Questions

### 1. "Design the data access layer for a high-throughput e-commerce order processing system."

**Expected discussion points:**
- Entity model: Order → OrderItems → Product, Customer, Payment
- All `@ManyToOne` as LAZY, `@Version` on Order
- SEQUENCE ID strategy with allocationSize=50
- DTO projections for order listing API
- JOIN FETCH for order detail API
- `@BatchSize` on OrderItems collection
- JDBC batching for order creation (multiple items)
- Separate read-only replicas for listing queries
- Second-level cache for Product catalog (read-heavy, rarely changes)
- No OSIV, explicit fetch strategy per use case
- Keyset pagination for order history

### 2. "Your team wants to migrate from raw JDBC to Hibernate for 200+ tables. What's your strategy?"

**Expected discussion points:**
- Phase 1: Set up Hibernate alongside existing JDBC (dual-write or read-through)
- Start with read-only entities (`@Immutable`) to reduce risk
- Map entities incrementally — high-value, CRUD-heavy tables first
- Generate initial mappings from schema, then refine
- Flyway for schema migrations from day 1
- Extensive integration testing — query count assertions
- Performance benchmarking: compare JDBC vs Hibernate query latency
- Team training on N+1, fetching strategies, persistence context lifecycle
- Legacy stored procedures: keep as native queries initially
- Do NOT use `hbm2ddl.auto` — schema ownership stays with migration tool

### 3. "How would you handle Hibernate in a microservices architecture?"

**Expected discussion points:**
- Each microservice owns its database — no shared Hibernate sessions
- Entity models are service-local — no shared entity JARs
- Cross-service references use IDs, not JPA relationships
- Events for cross-service consistency (outbox pattern)
- Each service has its own SessionFactory, cache regions, and connection pool
- Beware of distributed transaction pitfalls — prefer saga pattern
- L2 cache per-service, not shared (no cross-service cache coherence needed)

### 4. "You're seeing intermittent data corruption in production. Describe your debugging approach with Hibernate."

**Expected discussion points:**
- Enable SQL logging with parameters (`org.hibernate.orm.jdbc.bind`)
- Check `@Version` fields — are optimistic lock exceptions being swallowed?
- Check for missing `synchronized` keyword or `@Transactional` boundary issues
- Look for OSIV allowing lazy loads outside transaction (no atomicity guarantee)
- Check for self-invocation bypassing `@Transactional` proxy
- Verify `CascadeType` settings — unintended cascades can corrupt related data
- Check flush ordering — are there FK constraint violations being silently caught?
- Review thread safety — is a Session being shared across threads?
- Audit `@Transactional(propagation = REQUIRES_NEW)` — are nested transactions seeing uncommitted data?

### 5. "Design a multi-tenant SaaS platform's data layer using Hibernate."

**Expected discussion points:**
- Evaluate tenant isolation requirements (regulatory, performance, cost)
- Discriminator column for most tenants (cost-effective, shared tables)
- Separate schema for premium/regulated tenants (higher isolation)
- Hibernate 6 `@TenantId` for automatic filtering
- `CurrentTenantIdentifierResolver` backed by JWT claims or request headers
- Connection routing for schema-per-tenant (MultiTenantConnectionProvider)
- Index strategy: composite indexes including tenant_id
- L2 cache regions must be tenant-aware (cache key includes tenant)
- Native queries must include tenant filter — security review mandatory
- Migration tooling: Flyway multi-tenant support for schema-per-tenant

---

## 3 System Design Questions Involving Hibernate

### 1. "Design a real-time analytics dashboard that reads from the same database as the transactional OLTP system."

**Key Hibernate considerations:**
- Read replicas for analytics queries — separate DataSource with `@Transactional(readOnly = true)` routed to replica
- `StatelessSession` for large analytical queries — no persistence context overhead
- DTO projections exclusively — no entity management for read-only dashboards
- Native queries for complex aggregations (window functions, CTEs)
- Consider materialzed views managed outside Hibernate for pre-aggregated data
- JDBC fetch size tuned for large result sets (10,000+)
- Connection pool segregation — analytics queries should not starve OLTP connections

### 2. "Design an event sourcing system where Hibernate manages the command side (write model)."

**Key Hibernate considerations:**
- Write model: Standard Hibernate entities with `@Version` for consistency
- Event store: Append-only table, possibly with native INSERT for performance
- Projections (read model): Built from events, possibly in separate tables managed by Hibernate or plain JDBC
- Transaction boundary: Command + event publishing in same transaction (outbox pattern)
- `@DomainEvents` (Spring Data) for publishing events after successful commit
- No L2 cache on write model — events are the source of truth
- Read model can be heavily cached — rebuilt from events on invalidation

### 3. "Design a document management system that stores large files with metadata in a relational database."

**Key Hibernate considerations:**
- Metadata entity mapped normally with Hibernate
- File content stored as `@Lob` with `FetchType.LAZY` (requires bytecode enhancement for true lazy loading on basic fields)
- Or better: store file content in object storage (S3), store only the reference URL in the entity
- If using `@Lob`: streaming via `session.getLobHelper().createBlob(inputStream, length)` — don't load entire file into memory
- Pagination for document listing — DTO projection (never load the LOB for list views)
- Full-text search: Use PostgreSQL `tsvector` or Elasticsearch, not Hibernate Search (for production scale)
- Version history: Hibernate Envers for metadata audit trail, or a custom version table
- Multi-tenancy: Discriminator column for metadata, tenant-prefixed S3 buckets for files

---

*End of guide. Study the sections in order, then use the mock interview to test recall. Focus on being able to explain the "why" behind every answer — that's what separates senior engineers from mid-level ones.*
