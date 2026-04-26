# Hibernate 6+ & Spring Data JPA (Spring Boot 3+) — Complete Professional Guide

> **Target audience:** Beginner-to-intermediate Java developers aiming for production-grade mastery.
> **Stack:** Java 17+, Hibernate 6.x, Spring Boot 3.x, Spring Data JPA 3.x, Jakarta Persistence 3.1.

---

## Table of Contents

1. [Hibernate Foundations](#1-hibernate-foundations)
2. [Entity Lifecycle & Object States](#2-entity-lifecycle--object-states)
3. [CRUD Operations](#3-crud-operations)
4. [Querying Mechanisms](#4-querying-mechanisms)
5. [Associations & Relationship Mapping](#5-associations--relationship-mapping)
6. [Inheritance & Composition](#6-inheritance--composition)
7. [Caching](#7-caching)
8. [Fetching & Performance Optimization](#8-fetching--performance-optimization)
9. [Transactions & Concurrency](#9-transactions--concurrency)
10. [Lifecycle Callbacks & Interceptors](#10-lifecycle-callbacks--interceptors)
11. [Advanced Hibernate Features](#11-advanced-hibernate-features)
12. [Spring Data JPA Integration](#12-spring-data-jpa-integration)

---

# 1. Hibernate Foundations

## 1.1 What Is ORM and Why It Matters

**Object-Relational Mapping (ORM)** bridges the structural mismatch between object-oriented Java code and relational databases. Without ORM, you write repetitive JDBC boilerplate: open connections, prepare statements, map `ResultSet` rows to objects, handle exceptions, close resources. ORM automates this.

**Hibernate** is the most widely adopted JPA (Jakarta Persistence API) implementation. It:

- Translates Java objects to SQL transparently
- Manages object identity, dirty checking, and caching
- Provides a powerful query language (HQL) and Criteria API
- Handles schema generation, migrations, and dialect-specific SQL

### Architecture Overview

```
┌─────────────────────────────────────────────┐
│              Application Code               │
├─────────────────────────────────────────────┤
│        Jakarta Persistence API (JPA)        │
├─────────────────────────────────────────────┤
│      Hibernate ORM (JPA Implementation)     │
│  ┌────────────┐  ┌──────────┐  ┌─────────┐ │
│  │  Session /  │  │  Query   │  │  Cache  │ │
│  │EntityManager│  │  Engine  │  │  Layer  │ │
│  └────────────┘  └──────────┘  └─────────┘ │
├─────────────────────────────────────────────┤
│               JDBC / DataSource             │
├─────────────────────────────────────────────┤
│           Relational Database               │
└─────────────────────────────────────────────┘
```

**Key interfaces:**

| JPA Interface      | Hibernate Equivalent | Purpose                              |
|--------------------|----------------------|--------------------------------------|
| `EntityManagerFactory` | `SessionFactory`  | Thread-safe factory, one per DB      |
| `EntityManager`    | `Session`            | Unit-of-work, not thread-safe        |
| `EntityTransaction`| `Transaction`        | Demarcates transaction boundaries    |
| `TypedQuery`       | `Query`              | Execute HQL/JPQL/SQL                 |

## 1.2 Entity Mapping with Annotations

An **entity** is a Java class mapped to a database table. Every entity must have:

1. `@Entity` annotation
2. A no-argument constructor (can be `protected`)
3. An `@Id` field

```java
package com.example.entity;

import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "employees", schema = "hr",
       uniqueConstraints = @UniqueConstraint(columnNames = {"email"}),
       indexes = @Index(name = "idx_emp_dept", columnList = "department_id"))
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false, length = 120)
    private String fullName;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(name = "hire_date")
    private LocalDateTime hireDate;

    @Enumerated(EnumType.STRING)
    @Column(length = 20)
    private EmployeeStatus status;

    @Column(name = "department_id")
    private Long departmentId;

    protected Employee() {} // Required by JPA

    public Employee(String fullName, String email) {
        this.fullName = fullName;
        this.email = email;
        this.hireDate = LocalDateTime.now();
        this.status = EmployeeStatus.ACTIVE;
    }

    // Getters and setters omitted for brevity — always provide them
    public Long getId() { return id; }
    public String getFullName() { return fullName; }
    public void setFullName(String fullName) { this.fullName = fullName; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public LocalDateTime getHireDate() { return hireDate; }
    public EmployeeStatus getStatus() { return status; }
    public void setStatus(EmployeeStatus status) { this.status = status; }
    public Long getDepartmentId() { return departmentId; }
    public void setDepartmentId(Long departmentId) { this.departmentId = departmentId; }
}
```

```java
package com.example.entity;

public enum EmployeeStatus {
    ACTIVE, ON_LEAVE, TERMINATED
}
```

**Generated DDL** (PostgreSQL dialect):

```sql
CREATE TABLE hr.employees (
    id          BIGSERIAL PRIMARY KEY,
    full_name   VARCHAR(120) NOT NULL,
    email       VARCHAR(255) NOT NULL UNIQUE,
    hire_date   TIMESTAMP,
    status      VARCHAR(20),
    department_id BIGINT
);
CREATE INDEX idx_emp_dept ON hr.employees (department_id);
```

### Key Annotation Reference

| Annotation           | Purpose                                        |
|----------------------|------------------------------------------------|
| `@Entity`            | Marks class as a JPA entity                    |
| `@Table`             | Customizes table name, schema, indexes         |
| `@Id`                | Marks primary key field                        |
| `@GeneratedValue`    | Auto-generation strategy for PK                |
| `@Column`            | Customizes column name, length, nullable       |
| `@Enumerated`        | Maps enum (use `STRING`, never `ORDINAL`)      |
| `@Temporal`          | Legacy date mapping (not needed with `java.time`) |
| `@Transient`         | Excludes field from persistence                |
| `@Lob`               | Maps to BLOB/CLOB                              |
| `@Access`            | Field vs property access strategy              |

### ID Generation Strategies

| Strategy         | Description                          | Best For           |
|------------------|--------------------------------------|--------------------|
| `IDENTITY`       | DB auto-increment column             | MySQL, PostgreSQL  |
| `SEQUENCE`       | DB sequence object                   | PostgreSQL, Oracle |
| `TABLE`          | Simulated sequence via a table       | Portable (slow)    |
| `UUID`           | Hibernate generates a UUID           | Distributed systems|
| `AUTO`           | Hibernate picks based on dialect     | Avoid in production|

**Best practice:** Prefer `SEQUENCE` with an allocation size for batch-insert performance:

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "emp_seq")
@SequenceGenerator(name = "emp_seq", sequenceName = "employee_seq", allocationSize = 50)
private Long id;
```

This lets Hibernate pre-allocate 50 IDs at once, reducing round-trips during bulk inserts.

## 1.3 XML Mapping (Legacy Reference)

While annotations are the modern standard, legacy projects may use XML mappings:

```xml
<!-- src/main/resources/mappings/Employee.hbm.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-mapping PUBLIC
    "-//Hibernate/Hibernate Mapping DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-mapping-3.0.dtd">

<hibernate-mapping package="com.example.entity">
    <class name="Employee" table="employees" schema="hr">
        <id name="id" column="id">
            <generator class="identity"/>
        </id>
        <property name="fullName" column="full_name" not-null="true" length="120"/>
        <property name="email" not-null="true" unique="true"/>
        <property name="hireDate" column="hire_date" type="timestamp"/>
    </class>
</hibernate-mapping>
```

**Recommendation:** Always use annotations for new projects. XML mappings are harder to maintain and lack compile-time checks.

## 1.4 Session and SessionFactory

### SessionFactory

- **One per database**, created once at startup
- Thread-safe and immutable after creation
- Expensive to build (parses mappings, builds metamodel)

### Session (EntityManager)

- **One per unit of work** (usually one per request)
- NOT thread-safe — never share across threads
- Wraps a JDBC connection (obtained lazily)
- Contains the **first-level cache** (persistence context)

### Standalone Hibernate Example (No Spring)

```java
package com.example;

import com.example.entity.Employee;
import org.hibernate.Session;
import org.hibernate.SessionFactory;
import org.hibernate.cfg.Configuration;

public class HibernateStandaloneDemo {

    private static final SessionFactory sessionFactory = buildSessionFactory();

    private static SessionFactory buildSessionFactory() {
        return new Configuration()
                .configure("hibernate.cfg.xml") // loads from classpath
                .addAnnotatedClass(Employee.class)
                .buildSessionFactory();
    }

    public static void main(String[] args) {
        // CREATE
        try (Session session = sessionFactory.openSession()) {
            session.beginTransaction();

            Employee emp = new Employee("Alice Johnson", "alice@example.com");
            session.persist(emp);
            System.out.println("Generated ID: " + emp.getId());

            session.getTransaction().commit();
        }

        // READ
        try (Session session = sessionFactory.openSession()) {
            Employee emp = session.find(Employee.class, 1L);
            System.out.println("Found: " + emp.getFullName());
        }

        sessionFactory.close();
    }
}
```

## 1.5 Transaction Management

Every write operation in Hibernate **must** occur within a transaction. Without an active transaction, Hibernate will throw an exception on flush.

```java
Session session = sessionFactory.openSession();
Transaction tx = null;
try {
    tx = session.beginTransaction();

    Employee emp = session.find(Employee.class, 1L);
    emp.setFullName("Alice M. Johnson"); // Dirty checking detects the change

    tx.commit(); // Flushes changes, executes UPDATE
} catch (Exception e) {
    if (tx != null) tx.rollback();
    throw e;
} finally {
    session.close();
}
```

**How dirty checking works internally:**

1. When an entity is loaded, Hibernate takes a **snapshot** of its field values.
2. At flush time, Hibernate compares current values against the snapshot.
3. If any field changed, Hibernate generates and executes an `UPDATE` statement.
4. You never need to call `update()` or `merge()` for managed entities.

## 1.6 Configuration

### hibernate.cfg.xml (Standalone)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE hibernate-configuration PUBLIC
    "-//Hibernate/Hibernate Configuration DTD 3.0//EN"
    "http://www.hibernate.org/dtd/hibernate-configuration-3.0.dtd">

<hibernate-configuration>
    <session-factory>
        <property name="hibernate.connection.driver_class">org.postgresql.Driver</property>
        <property name="hibernate.connection.url">jdbc:postgresql://localhost:5432/mydb</property>
        <property name="hibernate.connection.username">postgres</property>
        <property name="hibernate.connection.password">secret</property>
        <property name="hibernate.dialect">org.hibernate.dialect.PostgreSQLDialect</property>
        <property name="hibernate.hbm2ddl.auto">validate</property>
        <property name="hibernate.show_sql">true</property>
        <property name="hibernate.format_sql">true</property>
        <property name="hibernate.jdbc.batch_size">25</property>

        <mapping class="com.example.entity.Employee"/>
    </session-factory>
</hibernate-configuration>
```

### Spring Boot application.yml (Recommended)

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: postgres
    password: secret
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      idle-timeout: 300000

  jpa:
    hibernate:
      ddl-auto: validate     # NEVER use 'create' or 'update' in production
    open-in-view: false       # Disable OSIV — critical for performance
    properties:
      hibernate:
        dialect: org.hibernate.dialect.PostgreSQLDialect
        format_sql: true
        jdbc:
          batch_size: 25
        order_inserts: true
        order_updates: true
        query:
          in_clause_parameter_padding: true

logging:
  level:
    org.hibernate.SQL: DEBUG                      # Log SQL statements
    org.hibernate.orm.jdbc.bind: TRACE            # Log bind parameters
```

### Critical Configuration Properties

| Property                        | Recommended Value | Why                                      |
|---------------------------------|-------------------|------------------------------------------|
| `ddl-auto`                      | `validate`        | Use Flyway/Liquibase for schema migration|
| `open-in-view`                  | `false`           | Prevents lazy loading in the view layer  |
| `jdbc.batch_size`               | `25-50`           | Enables JDBC batch inserts/updates       |
| `order_inserts` / `order_updates`| `true`           | Groups statements for batching           |
| `in_clause_parameter_padding`   | `true`            | Reduces query plan cache pollution       |

### pom.xml Dependencies (Spring Boot 3+)

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.5</version>
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.flywaydb</groupId>
        <artifactId>flyway-core</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>com.h2database</groupId>
        <artifactId>h2</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

# 2. Entity Lifecycle & Object States

Understanding object states is fundamental to working correctly with Hibernate. Every entity instance exists in exactly one of four states at any given moment.

## 2.1 The Four States

```
                    persist()
   ┌──────────┐  ────────────►  ┌──────────────┐
   │ TRANSIENT │                │  PERSISTENT   │
   └──────────┘  ◄────────────  │ (Managed)     │
        ▲         new entity    └──────┬───────┘
        │         not yet               │
        │         persisted         detach() /
        │                          session.close()
        │                              │
        │                              ▼
        │                       ┌──────────────┐
        │    GC collects        │   DETACHED    │
        │◄──────────────────────│              │
        │                       └──────┬───────┘
        │                              │
        │                          merge()
        │                              │
        │                              ▼
        │                       ┌──────────────┐
        │                       │  PERSISTENT   │
        │                       │  (re-merged)  │
        │                       └──────┬───────┘
        │                              │
        │                          remove()
        │                              │
        │                              ▼
        │                       ┌──────────────┐
        └───────────────────────│   REMOVED     │
              after flush/      └──────────────┘
              commit
```

### State Definitions

| State        | In Persistence Context? | Has DB Row? | Tracked by Hibernate? |
|--------------|-------------------------|-------------|----------------------|
| **Transient**| No                      | No          | No                   |
| **Persistent** (Managed) | Yes          | Yes (or will after flush) | Yes — dirty checking active |
| **Detached** | No                      | Yes         | No — changes are invisible to Hibernate |
| **Removed**  | Yes (scheduled for delete)| Yes (until flush) | Yes          |

## 2.2 Transient State

An object is **transient** when it is created with `new` but not yet associated with any persistence context.

```java
// This object exists only in the JVM heap — Hibernate knows nothing about it
Employee emp = new Employee("Bob Smith", "bob@example.com");
System.out.println(emp.getId()); // null — no ID assigned yet
```

- Hibernate does not track field changes
- No SQL is generated
- If the reference is lost, the object is garbage collected

## 2.3 Persistent (Managed) State

An entity becomes **persistent** when it is associated with an open `Session` / `EntityManager`. This happens via `persist()`, `merge()`, or by loading it from the database.

```java
try (Session session = sessionFactory.openSession()) {
    session.beginTransaction();

    // Transition: Transient → Persistent
    Employee emp = new Employee("Bob Smith", "bob@example.com");
    session.persist(emp);
    // emp is now PERSISTENT. ID is assigned (may require INSERT for IDENTITY strategy).

    // Dirty checking in action:
    emp.setFullName("Robert Smith"); // No explicit save() needed!
    // Hibernate detects the change at flush time.

    // Loading an entity also yields a Persistent object:
    Employee loaded = session.find(Employee.class, 1L);
    // loaded is PERSISTENT

    session.getTransaction().commit();
    // Flush happens here:
    //   INSERT INTO employees (full_name, email, ...) VALUES ('Robert Smith', 'bob@example.com', ...);
    //   (The UPDATE is folded into the INSERT because the entity was new.)
}
```

**Key behaviors of persistent entities:**

1. **Automatic dirty checking** — modified fields generate UPDATE at flush
2. **Identity guarantee** — `session.find(Employee.class, 1L)` always returns the same Java object instance within a session
3. **First-level cache** — repeated `find()` calls for the same ID return the cached instance (no extra SQL)

## 2.4 Detached State

An entity becomes **detached** when the session it was loaded in is closed, or when `detach()` / `clear()` is called.

```java
Employee detached;

// Load in one session
try (Session session = sessionFactory.openSession()) {
    detached = session.find(Employee.class, 1L);
} // session closes → 'detached' is now DETACHED

// Modify outside any session — Hibernate is unaware
detached.setFullName("Bobby Smith");

// Re-attach in a new session using merge()
try (Session session = sessionFactory.openSession()) {
    session.beginTransaction();

    // merge() returns a NEW managed copy; 'detached' itself stays detached
    Employee managed = session.merge(detached);

    // IMPORTANT: use 'managed', not 'detached', for further operations
    System.out.println(managed == detached); // false

    session.getTransaction().commit();
    // Generates: UPDATE employees SET full_name='Bobby Smith' WHERE id=1;
}
```

**Common mistake:** Calling `session.persist(detachedEntity)` — this throws `PersistentObjectException` because the entity already has an ID. Always use `merge()` for detached entities.

## 2.5 Removed State

An entity enters the **removed** state when `remove()` is called on a managed entity.

```java
try (Session session = sessionFactory.openSession()) {
    session.beginTransaction();

    Employee emp = session.find(Employee.class, 1L); // Persistent
    session.remove(emp);                              // Removed (scheduled for DELETE)

    // The entity is still in memory but marked for deletion.
    // Accessing it is legal, but persisting it again would re-insert it.

    session.getTransaction().commit();
    // Generates: DELETE FROM employees WHERE id=1;
}
```

## 2.6 Complete State Transition Demo

```java
package com.example;

import com.example.entity.Employee;
import org.hibernate.Session;
import org.hibernate.SessionFactory;

public class LifecycleDemo {

    public static void demonstrateLifecycle(SessionFactory sf) {

        Long empId;

        // === Phase 1: Transient → Persistent ===
        try (Session session = sf.openSession()) {
            session.beginTransaction();

            Employee emp = new Employee("Carol Lee", "carol@example.com");
            // STATE: Transient (no ID, not managed)

            session.persist(emp);
            // STATE: Persistent (ID assigned, managed, in first-level cache)
            // SQL (IDENTITY): INSERT INTO employees (...) VALUES (...)
            empId = emp.getId();
            System.out.println("After persist, ID = " + empId);

            emp.setFullName("Carol A. Lee");
            // No explicit save — dirty checking handles it

            session.getTransaction().commit();
            // SQL: UPDATE employees SET full_name='Carol A. Lee' WHERE id=?
        }
        // After session closes: emp reference (if held) is now DETACHED

        // === Phase 2: Load (Persistent) → Detach → Merge ===
        Employee snapshot;
        try (Session session = sf.openSession()) {
            snapshot = session.find(Employee.class, empId);
            // STATE: Persistent
        }
        // STATE: Detached (session closed)

        snapshot.setEmail("carol.lee@example.com"); // Hibernate unaware

        try (Session session = sf.openSession()) {
            session.beginTransaction();

            Employee managed = session.merge(snapshot);
            // STATE of 'managed': Persistent
            // STATE of 'snapshot': still Detached
            // SQL: SELECT ... FROM employees WHERE id=?
            //      UPDATE employees SET email='carol.lee@example.com' WHERE id=?

            session.getTransaction().commit();
        }

        // === Phase 3: Load → Remove ===
        try (Session session = sf.openSession()) {
            session.beginTransaction();

            Employee emp = session.find(Employee.class, empId);
            // STATE: Persistent

            session.remove(emp);
            // STATE: Removed

            session.getTransaction().commit();
            // SQL: DELETE FROM employees WHERE id=?
        }
    }
}
```

### State Transition Summary Table

| Operation                  | From State  | To State   | SQL Generated At Flush         |
|----------------------------|-------------|------------|-------------------------------|
| `new Entity()`             | —           | Transient  | None                           |
| `persist(entity)`          | Transient   | Persistent | INSERT                         |
| `find(Class, id)`          | —           | Persistent | SELECT (if not cached)         |
| `merge(detached)`          | Detached    | Persistent (new copy) | SELECT + UPDATE if dirty |
| `remove(entity)`           | Persistent  | Removed    | DELETE                         |
| `session.close()`          | Persistent  | Detached   | None                           |
| `session.detach(entity)`   | Persistent  | Detached   | None                           |
| `session.clear()`          | All managed | Detached   | None                           |

---

# 3. CRUD Operations

## 3.1 Create Operations

### persist() — JPA Standard

```java
session.beginTransaction();

Employee emp = new Employee("Dave Wilson", "dave@example.com");
session.persist(emp);
// For IDENTITY strategy: INSERT is executed immediately to obtain the generated ID
// For SEQUENCE strategy: only a SELECT nextval('employee_seq') runs now; INSERT is deferred to flush

System.out.println("ID: " + emp.getId()); // ID is available immediately

session.getTransaction().commit();
```

**Generated SQL (IDENTITY):**
```sql
INSERT INTO employees (full_name, email, hire_date, status, department_id)
VALUES ('Dave Wilson', 'dave@example.com', '2025-01-15 10:30:00', 'ACTIVE', NULL)
```

### save() — Hibernate-specific (Legacy)

```java
// Hibernate's save() returns the generated identifier
Long id = (Long) session.save(emp);
```

**`persist()` vs `save()`:**

| Aspect          | `persist()`               | `save()`                  |
|-----------------|---------------------------|---------------------------|
| Standard        | JPA                       | Hibernate-only            |
| Return type     | `void`                    | `Serializable` (the ID)  |
| Transient only  | Yes — throws on detached  | Works on transient        |
| Recommended     | **Yes**                   | No — use persist()        |

### Batch Inserts

For inserting hundreds or thousands of rows, clear the session periodically:

```java
session.beginTransaction();

for (int i = 0; i < 10_000; i++) {
    Employee emp = new Employee("Employee " + i, "emp" + i + "@example.com");
    session.persist(emp);

    if (i % 50 == 0) { // Match jdbc.batch_size
        session.flush();  // Execute batched INSERTs
        session.clear();  // Release memory from first-level cache
    }
}

session.getTransaction().commit();
```

**Requirements for JDBC batching to work:**

1. `hibernate.jdbc.batch_size` must be set (e.g., 25 or 50)
2. `hibernate.order_inserts=true` (groups inserts by entity type)
3. ID generation must **not** be `IDENTITY` (breaks batching because each INSERT must execute individually to get the ID). Use `SEQUENCE` with `allocationSize`.

## 3.2 Read Operations

### find() — JPA Standard (Eager Load)

```java
Employee emp = session.find(Employee.class, 1L);
// Returns null if not found
// SQL: SELECT e.id, e.full_name, e.email, ... FROM employees e WHERE e.id=1
```

### getReference() — Returns a Proxy (Lazy Load)

```java
Employee proxy = session.getReference(Employee.class, 1L);
// No SQL executed yet — returns a proxy object
// SQL executes when you access a non-ID field:
String name = proxy.getFullName();
// NOW: SELECT e.id, e.full_name, ... FROM employees e WHERE e.id=1

// Throws EntityNotFoundException if ID doesn't exist (at access time)
```

**When to use `getReference()`:** When you only need to set a foreign key relationship and never actually access the entity's data:

```java
Department dept = session.getReference(Department.class, 5L);
employee.setDepartment(dept); // No SELECT on Department needed
```

### get() vs load() (Hibernate-specific, Legacy)

| Method    | JPA Equivalent     | Returns null? | Lazy?  |
|-----------|--------------------|---------------|--------|
| `get()`   | `find()`           | Yes           | No     |
| `load()`  | `getReference()`   | No (throws)   | Yes    |

**Recommendation:** Use `find()` and `getReference()` — the JPA standard methods.

## 3.3 Update Operations

### Automatic Dirty Checking (Primary Mechanism)

```java
session.beginTransaction();

Employee emp = session.find(Employee.class, 1L);
emp.setFullName("David Wilson");    // Modified in memory
emp.setStatus(EmployeeStatus.ON_LEAVE);

session.getTransaction().commit();
// Hibernate detects changes and generates:
// UPDATE employees SET full_name='David Wilson', status='ON_LEAVE' WHERE id=1
```

No explicit `update()` or `save()` call needed for managed entities.

### merge() — For Detached Entities

```java
// Employee was loaded in a previous session (now detached)
detachedEmp.setFullName("David A. Wilson");

session.beginTransaction();
Employee managed = session.merge(detachedEmp);
// Returns a new managed instance — use this, not the detached one
session.getTransaction().commit();
// SQL: UPDATE employees SET full_name='David A. Wilson' WHERE id=?
```

**How merge() works internally:**

1. Checks if an entity with the same ID is already in the persistence context
2. If yes, copies the state from the detached entity to the managed one
3. If no, loads the entity from the database (SELECT), then copies state
4. Returns the managed copy

### update() — Hibernate-specific (Avoid)

```java
// Reattaches the detached instance directly — error-prone
session.update(detachedEmp);
// Throws NonUniqueObjectException if another instance with same ID is already managed
```

**Recommendation:** Always use `merge()` over `update()`.

## 3.4 Delete Operations

### remove() — JPA Standard

```java
session.beginTransaction();

Employee emp = session.find(Employee.class, 1L);
if (emp != null) {
    session.remove(emp);
}

session.getTransaction().commit();
// SQL: DELETE FROM employees WHERE id=1
```

### Delete Without Loading (Performance Optimization)

Loading an entity just to delete it wastes a SELECT. Use bulk delete instead:

```java
session.beginTransaction();

int deleted = session.createMutationQuery(
        "DELETE FROM Employee e WHERE e.id = :id")
    .setParameter("id", 1L)
    .executeUpdate();
// SQL: DELETE FROM employees WHERE id=1
// No SELECT needed. Returns count of deleted rows.

session.getTransaction().commit();
```

**Warning:** Bulk operations bypass the persistence context and first-level cache. Cached entities may become stale after a bulk delete.

## 3.5 saveOrUpdate() — Hibernate-specific (Legacy)

```java
// If entity is transient (no ID) → INSERT
// If entity is detached (has ID) → UPDATE
session.saveOrUpdate(emp);
```

**Modern replacement:** Use `merge()` which handles both cases and is JPA standard.

## 3.6 CRUD Operations Summary

| Operation | JPA Method | Hibernate Method | SQL | Notes |
|-----------|-----------|-----------------|-----|-------|
| Create    | `persist()` | `save()` | INSERT | Use persist() |
| Read (eager) | `find()` | `get()` | SELECT | Returns null if missing |
| Read (lazy) | `getReference()` | `load()` | SELECT (deferred) | Returns proxy |
| Update (managed) | Automatic dirty checking | Same | UPDATE | No method call needed |
| Update (detached) | `merge()` | `update()` | SELECT + UPDATE | Use merge() |
| Delete    | `remove()` | `delete()` | DELETE | Entity must be managed |
| Upsert    | `merge()` | `saveOrUpdate()` | INSERT or UPDATE | Use merge() |

---

# 4. Querying Mechanisms

## 4.1 HQL (Hibernate Query Language)

HQL operates on **entity classes and their fields**, not database tables and columns. Hibernate translates HQL into SQL based on the entity mappings.

```java
// Basic selection
List<Employee> employees = session.createQuery(
        "FROM Employee e WHERE e.status = :status ORDER BY e.fullName",
        Employee.class)
    .setParameter("status", EmployeeStatus.ACTIVE)
    .getResultList();
// SQL: SELECT e.id, e.full_name, e.email, e.hire_date, e.status, e.department_id
//      FROM employees e
//      WHERE e.status = 'ACTIVE'
//      ORDER BY e.full_name

// Projection (DTO)
List<Object[]> results = session.createQuery(
        "SELECT e.fullName, e.email FROM Employee e WHERE e.hireDate > :date",
        Object[].class)
    .setParameter("date", LocalDateTime.of(2024, 1, 1, 0, 0))
    .getResultList();

// Aggregate
Long count = session.createQuery(
        "SELECT COUNT(e) FROM Employee e WHERE e.status = :status",
        Long.class)
    .setParameter("status", EmployeeStatus.ACTIVE)
    .getSingleResult();

// Update (bulk)
int updated = session.createMutationQuery(
        "UPDATE Employee e SET e.status = :newStatus WHERE e.status = :oldStatus")
    .setParameter("newStatus", EmployeeStatus.TERMINATED)
    .setParameter("oldStatus", EmployeeStatus.ON_LEAVE)
    .executeUpdate();
// SQL: UPDATE employees SET status='TERMINATED' WHERE status='ON_LEAVE'
```

## 4.2 JPQL (Jakarta Persistence Query Language)

JPQL is the JPA-standard subset of HQL. Most HQL queries are valid JPQL. The primary difference: HQL has Hibernate-specific extensions (e.g., some functions, `WITH` clause syntax).

```java
// Using EntityManager (JPA interface)
EntityManager em = entityManagerFactory.createEntityManager();

TypedQuery<Employee> query = em.createQuery(
    "SELECT e FROM Employee e WHERE e.email LIKE :domain", Employee.class);
query.setParameter("domain", "%@example.com");
query.setMaxResults(10);
query.setFirstResult(0); // Pagination offset

List<Employee> results = query.getResultList();
```

### JPQL with DTO Projection (Constructor Expression)

```java
package com.example.dto;

public record EmployeeSummary(String fullName, String email) {}
```

```java
List<EmployeeSummary> summaries = em.createQuery(
    "SELECT new com.example.dto.EmployeeSummary(e.fullName, e.email) " +
    "FROM Employee e WHERE e.status = :status", EmployeeSummary.class)
    .setParameter("status", EmployeeStatus.ACTIVE)
    .getResultList();
// Returns DTOs directly — no entity overhead, no dirty checking
```

## 4.3 Criteria API

The Criteria API provides **type-safe, programmatic query construction** — ideal for dynamic queries where conditions vary at runtime.

```java
import jakarta.persistence.criteria.*;

CriteriaBuilder cb = session.getCriteriaBuilder();
CriteriaQuery<Employee> cq = cb.createQuery(Employee.class);
Root<Employee> root = cq.from(Employee.class);

// Dynamic predicate construction
List<Predicate> predicates = new ArrayList<>();

predicates.add(cb.equal(root.get("status"), EmployeeStatus.ACTIVE));

// Conditionally add filters
String nameFilter = "Alice"; // might come from user input
if (nameFilter != null) {
    predicates.add(cb.like(cb.lower(root.get("fullName")),
                           "%" + nameFilter.toLowerCase() + "%"));
}

LocalDateTime dateFilter = LocalDateTime.of(2024, 1, 1, 0, 0);
if (dateFilter != null) {
    predicates.add(cb.greaterThan(root.get("hireDate"), dateFilter));
}

cq.select(root)
  .where(predicates.toArray(new Predicate[0]))
  .orderBy(cb.asc(root.get("fullName")));

List<Employee> results = session.createQuery(cq)
    .setMaxResults(20)
    .getResultList();
```

**Generated SQL:**
```sql
SELECT e.id, e.full_name, e.email, e.hire_date, e.status, e.department_id
FROM employees e
WHERE e.status = 'ACTIVE'
  AND LOWER(e.full_name) LIKE '%alice%'
  AND e.hire_date > '2024-01-01 00:00:00'
ORDER BY e.full_name ASC
LIMIT 20
```

### Type-Safe Criteria with JPA Metamodel

Generate the metamodel by adding the Hibernate JPA Metamodel Generator to your build:

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jpamodelgen</artifactId>
    <scope>provided</scope>
</dependency>
```

This generates `Employee_` class at compile time:

```java
// Type-safe — compile error if field name changes
predicates.add(cb.equal(root.get(Employee_.status), EmployeeStatus.ACTIVE));
predicates.add(cb.like(root.get(Employee_.fullName), "%Alice%"));
```

## 4.4 Native SQL

When you need database-specific features (window functions, CTEs, full-text search, etc.):

```java
// Native query returning entities
List<Employee> results = session.createNativeQuery(
        "SELECT * FROM employees WHERE full_name ILIKE :pattern",
        Employee.class)
    .setParameter("pattern", "%smith%")
    .getResultList();

// Native query returning scalar values
List<Object[]> stats = session.createNativeQuery(
        """
        SELECT department_id, COUNT(*) as cnt, AVG(EXTRACT(YEAR FROM hire_date)) as avg_year
        FROM employees
        WHERE status = :status
        GROUP BY department_id
        HAVING COUNT(*) > 5
        ORDER BY cnt DESC
        """)
    .setParameter("status", "ACTIVE")
    .getResultList();

// Native query with result mapping to DTO
@SqlResultSetMapping(
    name = "DeptStatsMapping",
    classes = @ConstructorResult(
        targetClass = DeptStats.class,
        columns = {
            @ColumnResult(name = "department_id", type = Long.class),
            @ColumnResult(name = "cnt", type = Long.class)
        }
    )
)
```

## 4.5 Named Queries

Named queries are validated at startup, catching syntax errors early.

```java
@Entity
@NamedQueries({
    @NamedQuery(
        name = "Employee.findByStatus",
        query = "SELECT e FROM Employee e WHERE e.status = :status ORDER BY e.fullName"
    ),
    @NamedQuery(
        name = "Employee.countByDepartment",
        query = "SELECT e.departmentId, COUNT(e) FROM Employee e GROUP BY e.departmentId"
    )
})
public class Employee { /* ... */ }
```

```java
// Usage
List<Employee> active = session.createNamedQuery("Employee.findByStatus", Employee.class)
    .setParameter("status", EmployeeStatus.ACTIVE)
    .getResultList();
```

## 4.6 Parameter Binding

**Always use parameter binding.** Never concatenate user input into queries.

```java
// CORRECT — parameterized (safe from SQL injection, enables query plan caching)
session.createQuery("FROM Employee e WHERE e.email = :email", Employee.class)
    .setParameter("email", userInput)
    .getResultList();

// WRONG — string concatenation (SQL injection vulnerability!)
session.createQuery("FROM Employee e WHERE e.email = '" + userInput + "'", Employee.class)
    .getResultList();
```

| Binding Type       | Syntax               | Notes                         |
|--------------------|----------------------|-------------------------------|
| Named parameter    | `:paramName`         | Recommended — self-documenting|
| Positional parameter| `?1`, `?2`          | JPA standard alternative      |

## 4.7 When to Use Each Query Approach

| Approach      | Use When                                         | Pros                             | Cons                              |
|---------------|--------------------------------------------------|----------------------------------|-----------------------------------|
| **HQL/JPQL**  | Most queries; fixed or semi-dynamic              | Readable, entity-oriented        | String-based, no compile check    |
| **Criteria API** | Dynamic filters, search screens               | Type-safe, programmatic          | Verbose, harder to read           |
| **Native SQL** | DB-specific features, complex analytics          | Full SQL power                   | Not portable, bypasses some ORM   |
| **Named Query** | Frequently used, fixed queries                  | Validated at startup, cacheable  | Less flexible                     |
| **DTO Projection** | Read-only use cases, API responses           | No managed state overhead        | Must define DTO classes/records   |

---

# 5. Associations & Relationship Mapping

Relationship mapping is where Hibernate delivers the most value — and where most mistakes happen. Every association maps a foreign key or join table to a navigable Java reference.

## 5.1 Many-to-One (Most Common)

The owning side of a relationship — the entity that contains the foreign key column.

```java
@Entity
@Table(name = "departments")
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String name;

    protected Department() {}

    public Department(String name) {
        this.name = name;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
}
```

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false)
    private String fullName;

    @ManyToOne(fetch = FetchType.LAZY) // ALWAYS use LAZY for @ManyToOne
    @JoinColumn(name = "department_id", nullable = false,
                foreignKey = @ForeignKey(name = "fk_emp_dept"))
    private Department department;

    protected Employee() {}

    public Employee(String fullName, Department department) {
        this.fullName = fullName;
        this.department = department;
    }

    public Long getId() { return id; }
    public String getFullName() { return fullName; }
    public void setFullName(String fullName) { this.fullName = fullName; }
    public Department getDepartment() { return department; }
    public void setDepartment(Department department) { this.department = department; }
}
```

**Generated DDL:**
```sql
CREATE TABLE departments (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE employees (
    id            BIGSERIAL PRIMARY KEY,
    full_name     VARCHAR(255) NOT NULL,
    department_id BIGINT NOT NULL,
    CONSTRAINT fk_emp_dept FOREIGN KEY (department_id) REFERENCES departments(id)
);
```

**Critical rule:** `@ManyToOne` defaults to `FetchType.EAGER`. **Always override to LAZY.** Eager fetch on `@ManyToOne` is the number one cause of N+1 query problems in real applications.

## 5.2 One-to-Many (Bidirectional)

The inverse (non-owning) side of a Many-to-One relationship. The `mappedBy` attribute tells Hibernate which field in the child entity owns the relationship.

```java
@Entity
@Table(name = "departments")
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 100)
    private String name;

    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();

    protected Department() {}

    public Department(String name) {
        this.name = name;
    }

    // Synchronization helper methods — critical for bidirectional relationships
    public void addEmployee(Employee emp) {
        employees.add(emp);
        emp.setDepartment(this);
    }

    public void removeEmployee(Employee emp) {
        employees.remove(emp);
        emp.setDepartment(null);
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public List<Employee> getEmployees() { return Collections.unmodifiableList(employees); }
}
```

**Why synchronization helpers matter:** In a bidirectional relationship, both sides must be kept in sync in Java memory. If you only set one side, the in-memory model is inconsistent until the session is refreshed:

```java
// WRONG — only sets one side
employee.setDepartment(department);
// department.getEmployees() does NOT contain employee (in memory)

// CORRECT — use the helper
department.addEmployee(employee);
// Both sides are now consistent
```

**SQL for loading employees of a department (lazy):**
```sql
-- When department.getEmployees() is first accessed:
SELECT e.id, e.full_name, e.department_id
FROM employees e
WHERE e.department_id = ?
```

### Common Mistake: Unidirectional @OneToMany (Without @JoinColumn)

```java
// AVOID THIS — generates an unnecessary join table
@OneToMany(cascade = CascadeType.ALL)
private List<Employee> employees = new ArrayList<>();
```

This creates an intermediary table `department_employees(department_id, employees_id)`, which is almost never what you want. Either:
1. Use bidirectional with `mappedBy` (recommended), or
2. Add `@JoinColumn` explicitly for unidirectional

## 5.3 One-to-One

```java
@Entity
@Table(name = "employee_profiles")
public class EmployeeProfile {

    @Id
    private Long id; // Shared PK with Employee

    @OneToOne(fetch = FetchType.LAZY)
    @MapsId // Uses Employee PK as this entity PK
    @JoinColumn(name = "id")
    private Employee employee;

    @Column(length = 1000)
    private String bio;

    @Column(name = "linkedin_url")
    private String linkedInUrl;

    protected EmployeeProfile() {}

    public EmployeeProfile(Employee employee, String bio) {
        this.employee = employee;
        this.bio = bio;
    }

    public Long getId() { return id; }
    public Employee getEmployee() { return employee; }
    public String getBio() { return bio; }
    public void setBio(String bio) { this.bio = bio; }
    public String getLinkedInUrl() { return linkedInUrl; }
    public void setLinkedInUrl(String linkedInUrl) { this.linkedInUrl = linkedInUrl; }
}
```

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false)
    private String fullName;

    @OneToOne(mappedBy = "employee", cascade = CascadeType.ALL,
              fetch = FetchType.LAZY, orphanRemoval = true)
    private EmployeeProfile profile;

    // ... constructors, getters, setters
}
```

**Generated DDL:**
```sql
CREATE TABLE employees (
    id        BIGSERIAL PRIMARY KEY,
    full_name VARCHAR(255) NOT NULL
);

CREATE TABLE employee_profiles (
    id           BIGINT PRIMARY KEY, -- same as employee.id
    bio          VARCHAR(1000),
    linkedin_url VARCHAR(255),
    CONSTRAINT fk_profile_emp FOREIGN KEY (id) REFERENCES employees(id)
);
```

**Key insight:** `@MapsId` shares the primary key, eliminating a separate foreign key column. This is the most efficient One-to-One mapping. Without `@MapsId`, Hibernate may not be able to lazy-load the parent side of a `@OneToOne` because it must query the child table to determine whether the proxy should be null or not.

## 5.4 Many-to-Many

```java
@Entity
@Table(name = "projects")
public class Project {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @ManyToMany(cascade = {CascadeType.PERSIST, CascadeType.MERGE})
    @JoinTable(
        name = "project_assignments",
        joinColumns = @JoinColumn(name = "project_id"),
        inverseJoinColumns = @JoinColumn(name = "employee_id")
    )
    private Set<Employee> members = new HashSet<>();
    // Use Set, not List — List causes DELETE ALL + RE-INSERT on modification

    protected Project() {}

    public Project(String name) {
        this.name = name;
    }

    public void addMember(Employee emp) {
        members.add(emp);
    }

    public void removeMember(Employee emp) {
        members.remove(emp);
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public Set<Employee> getMembers() { return Collections.unmodifiableSet(members); }
}
```

**Generated DDL:**
```sql
CREATE TABLE projects (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE project_assignments (
    project_id  BIGINT NOT NULL REFERENCES projects(id),
    employee_id BIGINT NOT NULL REFERENCES employees(id),
    PRIMARY KEY (project_id, employee_id)
);
```

**Critical Many-to-Many rules:**

1. **Use `Set`, not `List`** — With `List`, removing one element causes Hibernate to delete ALL rows from the join table and re-insert the remaining ones. With `Set`, only the specific row is deleted.
2. **Never use `CascadeType.REMOVE`** on `@ManyToMany` — Deleting one project would delete shared employees.
3. **If the join table needs extra columns** (e.g., `assigned_date`, `role`), you must model the join table as a separate entity.

### Many-to-Many with Extra Columns

```java
@Entity
@Table(name = "project_assignments")
public class ProjectAssignment {

    @EmbeddedId
    private ProjectAssignmentId id;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("projectId")
    @JoinColumn(name = "project_id")
    private Project project;

    @ManyToOne(fetch = FetchType.LAZY)
    @MapsId("employeeId")
    @JoinColumn(name = "employee_id")
    private Employee employee;

    @Column(name = "assigned_date", nullable = false)
    private LocalDate assignedDate;

    @Column(length = 50)
    private String role;

    protected ProjectAssignment() {}

    public ProjectAssignment(Project project, Employee employee, String role) {
        this.project = project;
        this.employee = employee;
        this.role = role;
        this.assignedDate = LocalDate.now();
        this.id = new ProjectAssignmentId(project.getId(), employee.getId());
    }

    public ProjectAssignmentId getId() { return id; }
    public Project getProject() { return project; }
    public Employee getEmployee() { return employee; }
    public LocalDate getAssignedDate() { return assignedDate; }
    public String getRole() { return role; }
    public void setRole(String role) { this.role = role; }
}
```

```java
@Embeddable
public class ProjectAssignmentId implements Serializable {

    @Column(name = "project_id")
    private Long projectId;

    @Column(name = "employee_id")
    private Long employeeId;

    protected ProjectAssignmentId() {}

    public ProjectAssignmentId(Long projectId, Long employeeId) {
        this.projectId = projectId;
        this.employeeId = employeeId;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ProjectAssignmentId that)) return false;
        return Objects.equals(projectId, that.projectId)
            && Objects.equals(employeeId, that.employeeId);
    }

    @Override
    public int hashCode() {
        return Objects.hash(projectId, employeeId);
    }
}
```

## 5.5 Cascade Types

| Cascade Type       | Effect                                           | Use With              |
|--------------------|--------------------------------------------------|-----------------------|
| `PERSIST`          | Persisting parent persists children               | Parent-child          |
| `MERGE`            | Merging parent merges children                    | Parent-child          |
| `REMOVE`           | Removing parent removes children                  | One-to-One, One-to-Many |
| `REFRESH`          | Refreshing parent refreshes children              | Rare                  |
| `DETACH`           | Detaching parent detaches children                | Rare                  |
| `ALL`              | All of the above                                  | Strong parent-child   |

**`orphanRemoval = true`** — Deletes a child entity when it is removed from the parent collection, even without `CascadeType.REMOVE`. Use this for true composition (child cannot exist without parent).

```java
// orphanRemoval example:
department.removeEmployee(emp); // removes from list
// At flush: DELETE FROM employees WHERE id = ?
// The employee is deleted because it was "orphaned"
```

**Anti-pattern:** Using `CascadeType.ALL` on `@ManyToMany` or `@ManyToOne`. Cascades should flow from parent to child, never from child to parent or between peers.

## 5.6 Fetch Strategies

| Association         | JPA Default  | Recommended  | Why                              |
|--------------------|-------------|-------------|----------------------------------|
| `@ManyToOne`       | EAGER       | **LAZY**    | Avoids loading the parent for every child query |
| `@OneToOne`        | EAGER       | **LAZY**    | Same reason                      |
| `@OneToMany`       | LAZY        | LAZY        | Already correct                  |
| `@ManyToMany`      | LAZY        | LAZY        | Already correct                  |

**Rule of thumb:** Make everything LAZY, then use JOIN FETCH in queries where you need related data.

```java
// Fetch employees WITH their department in a single query
List<Employee> employees = session.createQuery(
    "SELECT e FROM Employee e JOIN FETCH e.department WHERE e.status = :status",
    Employee.class)
    .setParameter("status", EmployeeStatus.ACTIVE)
    .getResultList();
// SQL: SELECT e.*, d.* FROM employees e
//      INNER JOIN departments d ON e.department_id = d.id
//      WHERE e.status = 'ACTIVE'
```

## 5.7 Common Relationship Mapping Mistakes

| Mistake                                 | Problem                                  | Fix                                      |
|-----------------------------------------|------------------------------------------|------------------------------------------|
| EAGER fetch on @ManyToOne               | N+1 queries on every collection load     | Use LAZY + JOIN FETCH                    |
| List instead of Set for @ManyToMany     | Full delete + re-insert on changes       | Use Set with proper equals/hashCode      |
| Missing `mappedBy`                      | Creates unwanted join table              | Add `mappedBy` on inverse side           |
| Cascade REMOVE on @ManyToMany           | Deleting one side deletes shared entities| Never cascade REMOVE on Many-to-Many     |
| Missing sync helpers on bidirectional   | Inconsistent in-memory state             | Add addX/removeX methods                 |
| Missing equals/hashCode on entities     | Broken Set operations and detach/merge   | Use business key or ID-based equality    |

### Implementing equals() and hashCode()

```java
@Entity
public class Employee {
    // ... fields ...

    // Use business key (email) — stable across transient/persistent/detached states
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee other)) return false;
        return email != null && email.equals(other.getEmail());
    }

    @Override
    public int hashCode() {
        return getClass().hashCode(); // Constant — safe for all entity states
    }
}
```

**Why constant hashCode?** A newly created (transient) entity has no ID yet. If hashCode uses the ID, adding the entity to a `HashSet` before persist and then looking it up after persist would fail because the hash changed. Using a constant hashCode is safe (degrades to linear scan in worst case, but entities rarely have huge hash collisions per bucket in practice).

---

# 6. Inheritance & Composition

## 6.1 Inheritance Strategies

JPA supports three inheritance mapping strategies. The right choice depends on your query patterns, data volume, and schema constraints.

### 6.1.1 SINGLE_TABLE (Default)

All classes in the hierarchy share one table. A **discriminator column** identifies the type.

```java
@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "payment_type", discriminatorType = DiscriminatorType.STRING)
public abstract class Payment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private BigDecimal amount;

    @Column(name = "payment_date", nullable = false)
    private LocalDateTime paymentDate;

    protected Payment() {}

    protected Payment(BigDecimal amount) {
        this.amount = amount;
        this.paymentDate = LocalDateTime.now();
    }

    public Long getId() { return id; }
    public BigDecimal getAmount() { return amount; }
    public LocalDateTime getPaymentDate() { return paymentDate; }
}
```

```java
@Entity
@DiscriminatorValue("CREDIT_CARD")
public class CreditCardPayment extends Payment {

    @Column(name = "card_number", length = 19)
    private String cardNumber;

    @Column(name = "expiry_month")
    private Integer expiryMonth;

    @Column(name = "expiry_year")
    private Integer expiryYear;

    protected CreditCardPayment() {}

    public CreditCardPayment(BigDecimal amount, String cardNumber, int expiryMonth, int expiryYear) {
        super(amount);
        this.cardNumber = cardNumber;
        this.expiryMonth = expiryMonth;
        this.expiryYear = expiryYear;
    }

    public String getCardNumber() { return cardNumber; }
    public Integer getExpiryMonth() { return expiryMonth; }
    public Integer getExpiryYear() { return expiryYear; }
}
```

```java
@Entity
@DiscriminatorValue("BANK_TRANSFER")
public class BankTransferPayment extends Payment {

    @Column(name = "bank_name", length = 100)
    private String bankName;

    @Column(name = "account_number", length = 34)
    private String accountNumber;

    protected BankTransferPayment() {}

    public BankTransferPayment(BigDecimal amount, String bankName, String accountNumber) {
        super(amount);
        this.bankName = bankName;
        this.accountNumber = accountNumber;
    }

    public String getBankName() { return bankName; }
    public String getAccountNumber() { return accountNumber; }
}
```

**Generated DDL:**
```sql
CREATE TABLE payments (
    id             BIGSERIAL PRIMARY KEY,
    payment_type   VARCHAR(31) NOT NULL,  -- discriminator
    amount         NUMERIC(38,2) NOT NULL,
    payment_date   TIMESTAMP NOT NULL,
    card_number    VARCHAR(19),           -- CreditCard fields (NULL for other types)
    expiry_month   INTEGER,
    expiry_year    INTEGER,
    bank_name      VARCHAR(100),          -- BankTransfer fields (NULL for other types)
    account_number VARCHAR(34)
);
```

**Polymorphic query:**
```java
List<Payment> allPayments = session.createQuery("FROM Payment", Payment.class).getResultList();
// SQL: SELECT * FROM payments
// Hibernate instantiates the correct subclass based on payment_type
```

| Pros | Cons |
|------|------|
| Fastest for polymorphic queries (single table scan) | Subclass columns must be nullable |
| No JOINs needed | Table grows wide with many subclasses |
| Simple schema | No DB-level NOT NULL on subclass columns |

### 6.1.2 JOINED

Each class gets its own table. Subclass tables contain only their specific fields plus a FK to the parent table.

```java
@Entity
@Table(name = "payments")
@Inheritance(strategy = InheritanceType.JOINED)
public abstract class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private BigDecimal amount;

    @Column(name = "payment_date", nullable = false)
    private LocalDateTime paymentDate;

    // ... same constructors and getters
}
```

```java
@Entity
@Table(name = "credit_card_payments")
public class CreditCardPayment extends Payment {
    @Column(name = "card_number", nullable = false, length = 19)
    private String cardNumber;
    // ... subclass-specific fields, NOT NULL is now possible
}
```

**Generated DDL:**
```sql
CREATE TABLE payments (
    id           BIGSERIAL PRIMARY KEY,
    amount       NUMERIC(38,2) NOT NULL,
    payment_date TIMESTAMP NOT NULL
);

CREATE TABLE credit_card_payments (
    id           BIGINT PRIMARY KEY REFERENCES payments(id),
    card_number  VARCHAR(19) NOT NULL,
    expiry_month INTEGER NOT NULL,
    expiry_year  INTEGER NOT NULL
);

CREATE TABLE bank_transfer_payments (
    id             BIGINT PRIMARY KEY REFERENCES payments(id),
    bank_name      VARCHAR(100) NOT NULL,
    account_number VARCHAR(34) NOT NULL
);
```

**Polymorphic query SQL:**
```sql
SELECT p.id, p.amount, p.payment_date,
       cc.card_number, cc.expiry_month, cc.expiry_year,
       bt.bank_name, bt.account_number,
       CASE WHEN cc.id IS NOT NULL THEN 1
            WHEN bt.id IS NOT NULL THEN 2 END AS clazz_
FROM payments p
LEFT JOIN credit_card_payments cc ON p.id = cc.id
LEFT JOIN bank_transfer_payments bt ON p.id = bt.id
```

| Pros | Cons |
|------|------|
| Normalized schema | Polymorphic queries require JOINs |
| Subclass columns can be NOT NULL | Slower polymorphic reads (especially with many subclasses) |
| No wasted space | INSERT requires two statements (parent + child table) |

### 6.1.3 TABLE_PER_CLASS

Each concrete class gets a complete table with all inherited fields duplicated.

```java
@Entity
@Inheritance(strategy = InheritanceType.TABLE_PER_CLASS)
public abstract class Payment {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO) // Cannot use IDENTITY
    private Long id;
    // ...
}
```

**Generated DDL:**
```sql
CREATE TABLE credit_card_payments (
    id           BIGINT PRIMARY KEY,
    amount       NUMERIC(38,2) NOT NULL,   -- duplicated from Payment
    payment_date TIMESTAMP NOT NULL,        -- duplicated from Payment
    card_number  VARCHAR(19) NOT NULL,
    expiry_month INTEGER NOT NULL,
    expiry_year  INTEGER NOT NULL
);

CREATE TABLE bank_transfer_payments (
    id             BIGINT PRIMARY KEY,
    amount         NUMERIC(38,2) NOT NULL,  -- duplicated
    payment_date   TIMESTAMP NOT NULL,       -- duplicated
    bank_name      VARCHAR(100) NOT NULL,
    account_number VARCHAR(34) NOT NULL
);
```

**Polymorphic query SQL:**
```sql
SELECT id, amount, payment_date, card_number, expiry_month, expiry_year,
       NULL, NULL, 1 AS clazz_
FROM credit_card_payments
UNION ALL
SELECT id, amount, payment_date, NULL, NULL, NULL,
       bank_name, account_number, 2 AS clazz_
FROM bank_transfer_payments
```

| Pros | Cons |
|------|------|
| Good for querying single subclass | Polymorphic queries use UNION ALL (slow) |
| Each table is self-contained | Cannot use IDENTITY generation |
| No joins for concrete queries | Column duplication across tables |

### Strategy Comparison

| Criteria                    | SINGLE_TABLE | JOINED     | TABLE_PER_CLASS |
|-----------------------------|-------------|------------|-----------------|
| Polymorphic query speed     | Best        | Moderate   | Worst (UNION)   |
| Concrete class query speed  | Good        | Good       | Best            |
| Schema normalization        | Worst       | Best       | Moderate        |
| NOT NULL on subclass fields | No          | Yes        | Yes             |
| Insert performance          | Best (1 table)| Moderate (2 tables)| Good (1 table)|
| Recommended for             | Few subclasses, polymorphic queries | Many subclasses, data integrity | Rarely used |

**Default recommendation:** Use `SINGLE_TABLE` unless you have strict NOT NULL requirements on subclass columns, in which case use `JOINED`.

## 6.2 @MappedSuperclass

When you want to share fields across entities **without** creating a polymorphic hierarchy (no common table, no polymorphic queries):

```java
@MappedSuperclass
public abstract class BaseEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }

    public Long getId() { return id; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

```java
@Entity
@Table(name = "employees")
public class Employee extends BaseEntity {
    @Column(name = "full_name", nullable = false)
    private String fullName;
    // Inherits id, createdAt, updatedAt — no separate table for BaseEntity
}

@Entity
@Table(name = "departments")
public class Department extends BaseEntity {
    @Column(nullable = false)
    private String name;
    // Also inherits id, createdAt, updatedAt — in departments table
}
```

**Key difference from @Entity inheritance:** `@MappedSuperclass` is NOT an entity. You cannot query `FROM BaseEntity` and it has no table. It only provides shared field definitions.

## 6.3 @Embeddable / @Embedded (Composition)

For value objects that have no identity of their own and are stored in the parent table:

```java
@Embeddable
public class Address {

    @Column(name = "street_address", length = 200)
    private String street;

    @Column(length = 100)
    private String city;

    @Column(length = 50)
    private String state;

    @Column(name = "zip_code", length = 10)
    private String zipCode;

    @Column(length = 2)
    private String country;

    protected Address() {}

    public Address(String street, String city, String state, String zipCode, String country) {
        this.street = street;
        this.city = city;
        this.state = state;
        this.zipCode = zipCode;
        this.country = country;
    }

    public String getStreet() { return street; }
    public String getCity() { return city; }
    public String getState() { return state; }
    public String getZipCode() { return zipCode; }
    public String getCountry() { return country; }
}
```

```java
@Entity
@Table(name = "companies")
public class Company {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Embedded
    private Address headquarters;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "street", column = @Column(name = "billing_street")),
        @AttributeOverride(name = "city",   column = @Column(name = "billing_city")),
        @AttributeOverride(name = "state",  column = @Column(name = "billing_state")),
        @AttributeOverride(name = "zipCode",column = @Column(name = "billing_zip")),
        @AttributeOverride(name = "country",column = @Column(name = "billing_country"))
    })
    private Address billingAddress;

    protected Company() {}

    public Company(String name, Address headquarters) {
        this.name = name;
        this.headquarters = headquarters;
    }
}
```

**Generated DDL:**
```sql
CREATE TABLE companies (
    id               BIGSERIAL PRIMARY KEY,
    name             VARCHAR(255) NOT NULL,
    street_address   VARCHAR(200),    -- headquarters fields
    city             VARCHAR(100),
    state            VARCHAR(50),
    zip_code         VARCHAR(10),
    country          VARCHAR(2),
    billing_street   VARCHAR(255),    -- billingAddress (overridden names)
    billing_city     VARCHAR(255),
    billing_state    VARCHAR(255),
    billing_zip      VARCHAR(255),
    billing_country  VARCHAR(255)
);
```

**When to use @Embeddable vs a separate @Entity:**

| Use @Embeddable when | Use @Entity when |
|---------------------|------------------|
| The object has no identity (value type) | The object has its own lifecycle |
| Always loaded/saved with its parent | Shared across multiple parents |
| No need to query it independently | Queried independently |
| Example: Address, Money, DateRange | Example: Department, User |


---

# 7. Caching

Hibernate provides a multi-level caching architecture. Understanding each level is essential to making caching work for you instead of against you.

## 7.1 First-Level Cache (Persistence Context)

The first-level cache is built into every `Session` / `EntityManager`. It is **always on** and cannot be disabled.

**How it works:**

1. When you load an entity, Hibernate stores it in a `Map<EntityKey, Object>` inside the Session.
2. Subsequent `find()` calls for the same entity type + ID return the cached instance — no SQL.
3. The cache is flushed and cleared when the Session closes.

```java
try (Session session = sf.openSession()) {
    Employee emp1 = session.find(Employee.class, 1L);
    // SQL: SELECT ... FROM employees WHERE id=1

    Employee emp2 = session.find(Employee.class, 1L);
    // NO SQL — returned from first-level cache

    System.out.println(emp1 == emp2); // true — same Java object
}
```

**Identity guarantee:** Within a single Session, there is exactly one Java instance per database row. This is the **Repeatable Read** guarantee at the application level.

**When the first-level cache hurts:**

```java
// Loading 100,000 entities fills the first-level cache with 100K objects
// This causes OutOfMemoryError
try (Session session = sf.openSession()) {
    for (int i = 1; i <= 100_000; i++) {
        Employee emp = session.find(Employee.class, (long) i);
        processEmployee(emp);
    }
    // FIX: Call session.clear() or session.detach(emp) periodically
}
```

**Best practice for bulk reads:**
```java
try (Session session = sf.openSession()) {
    session.setDefaultReadOnly(true); // Disables snapshot storage for dirty checking

    ScrollableResults<Employee> scroll = session.createQuery(
            "FROM Employee", Employee.class)
        .setFetchSize(100) // JDBC fetch size
        .scroll(ScrollMode.FORWARD_ONLY);

    int count = 0;
    while (scroll.next()) {
        Employee emp = scroll.get();
        processEmployee(emp);

        if (++count % 100 == 0) {
            session.clear(); // Evict all entities from first-level cache
        }
    }
}
```

## 7.2 Second-Level Cache (L2)

The second-level cache is a **SessionFactory-scoped** cache shared across all Sessions. It caches entity data between sessions and even between transactions.

### Enabling the Second-Level Cache

**Step 1: Add a cache provider dependency (e.g., Ehcache via JCache):**

```xml
<dependency>
    <groupId>org.hibernate.orm</groupId>
    <artifactId>hibernate-jcache</artifactId>
</dependency>
<dependency>
    <groupId>org.ehcache</groupId>
    <artifactId>ehcache</artifactId>
    <classifier>jakarta</classifier>
</dependency>
```

**Step 2: Configure in application.yml:**

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_second_level_cache: true
          region.factory_class: org.hibernate.cache.jcache.JCacheRegionFactory
        jakarta:
          cache:
            provider: org.ehcache.jsr107.EhcacheCachingProvider
```

**Step 3: Annotate entities to cache:**

```java
@Entity
@Table(name = "departments")
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Department {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @OneToMany(mappedBy = "department")
    @Cache(usage = CacheConcurrencyStrategy.READ_WRITE) // Cache the collection too
    private List<Employee> employees = new ArrayList<>();

    // ...
}
```

**Step 4: Ehcache configuration (ehcache.xml):**

```xml
<config xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xmlns="http://www.ehcache.org/v3"
        xsi:noNamespaceSchemaLocation="http://www.ehcache.org/schema/ehcache-core-3.0.xsd">

    <cache alias="com.example.entity.Department">
        <expiry>
            <ttl unit="minutes">60</ttl>
        </expiry>
        <heap unit="entries">1000</heap>
    </cache>

    <cache-template name="default">
        <expiry>
            <ttl unit="minutes">30</ttl>
        </expiry>
        <heap unit="entries">500</heap>
    </cache-template>
</config>
```

### How the L2 Cache Works Internally

1. **Entity data is stored in a dehydrated form** — not as Java objects, but as arrays of column values: `[1, "Engineering", ...]`
2. When Session A loads `Department#1`, it goes to L1 cache first, then L2, then DB.
3. When Session B later loads `Department#1`, it finds the dehydrated data in L2, hydrates it into a new Java object, and stores it in Session B's L1 cache.
4. Each Session still has its own Java instance — L2 sharing is at the data level.

### Cache Concurrency Strategies

| Strategy              | Description                          | Use Case                    |
|-----------------------|--------------------------------------|-----------------------------|
| `READ_ONLY`           | Cached data never changes            | Reference data, enums, countries |
| `NONSTRICT_READ_WRITE`| Eventual consistency, no locking     | Rarely updated data         |
| `READ_WRITE`          | Soft locks prevent stale reads       | General-purpose, most common|
| `TRANSACTIONAL`       | Full JTA transaction support         | Requires JTA (heavy)        |

## 7.3 Query Cache

The query cache stores **query results** (list of entity IDs or scalar values) keyed by the query string and its parameters.

```yaml
spring:
  jpa:
    properties:
      hibernate:
        cache:
          use_query_cache: true
```

```java
List<Employee> employees = session.createQuery(
        "FROM Employee e WHERE e.status = :status", Employee.class)
    .setParameter("status", EmployeeStatus.ACTIVE)
    .setCacheable(true)               // Enable query cache for this query
    .setHint("jakarta.persistence.cache.storeMode", "USE")
    .getResultList();

// First call: executes SQL, stores result IDs in query cache
// Second call with same parameters: reads IDs from query cache,
//   then loads entities from L2 cache (no SQL at all if entities are cached)
```

**The query cache is invalidated whenever any entity in the queried table is modified.** This makes it useful only for:
- Tables that are read-heavy and write-light (e.g., lookup tables)
- Queries that are executed frequently with the same parameters

**When the query cache hurts:**
- On write-heavy tables: constant invalidation means the cache is always stale
- When used without the second-level entity cache: the query cache stores IDs, and then Hibernate loads each entity individually — causing N+1 queries

## 7.4 When Caching Helps vs Hurts

| Scenario                              | L2 Cache?   | Query Cache? |
|---------------------------------------|------------|-------------|
| Reference/lookup data (countries, currencies) | Yes (READ_ONLY) | Yes |
| Frequently read, rarely written entities | Yes (READ_WRITE) | Maybe |
| User profile loaded on every request | Yes        | No           |
| Search results (dynamic filters)     | No         | No           |
| Write-heavy transactional tables     | No         | No           |
| Aggregation queries                  | No         | Maybe (if stable data) |

**General rules:**
1. Profile before caching — premature caching adds complexity
2. Start with no L2 cache. Add it only when you prove a read bottleneck exists.
3. Always use L2 entity cache if you enable the query cache.
4. Monitor cache hit rates in production (Ehcache exposes JMX metrics).

---

# 8. Fetching & Performance Optimization

Performance problems in Hibernate almost always come from poorly planned fetching strategies. This section covers the most common issues and their solutions.

## 8.1 Lazy vs Eager Loading

**Lazy loading** means associated entities are not loaded from the database until they are first accessed. Hibernate uses **proxy objects** (generated subclasses) to implement this.

```java
Employee emp = session.find(Employee.class, 1L);
// SQL: SELECT e.* FROM employees e WHERE e.id=1
// emp.department is a proxy — NOT yet loaded

String deptName = emp.getDepartment().getName();
// NOW: SELECT d.* FROM departments d WHERE d.id=5
// The proxy triggers a SELECT on first non-ID access
```

**LazyInitializationException** — the most common Hibernate exception:

```java
Employee emp;
try (Session session = sf.openSession()) {
    emp = session.find(Employee.class, 1L);
} // Session closes

// THROWS LazyInitializationException:
String deptName = emp.getDepartment().getName();
// The proxy tries to load data but the session is closed
```

**Solutions:**
1. JOIN FETCH in the query (best)
2. `Hibernate.initialize()` within the session
3. Entity graph (JPA standard)
4. Open Session in View (OSIV) — avoid in production

## 8.2 The N+1 Problem

The most critical performance problem in ORM applications.

**Setup:**
```java
// 1 query to load all departments
List<Department> departments = session.createQuery(
    "FROM Department", Department.class).getResultList();
// SQL: SELECT * FROM departments  (returns 100 departments)

// N queries when accessing each department employees
for (Department dept : departments) {
    System.out.println(dept.getName() + ": " + dept.getEmployees().size());
    // Each iteration triggers: SELECT * FROM employees WHERE department_id = ?
}
// TOTAL: 1 + 100 = 101 queries!
```

### Solution 1: JOIN FETCH (Best for Known Access Patterns)

```java
List<Department> departments = session.createQuery(
    "SELECT DISTINCT d FROM Department d JOIN FETCH d.employees",
    Department.class)
    .getResultList();
// SQL: SELECT d.*, e.* FROM departments d
//      INNER JOIN employees e ON d.id = e.department_id
// TOTAL: 1 query
```

**The DISTINCT keyword** is important here. Without it, you get duplicate Department objects — one per employee row. In Hibernate 6+, `DISTINCT` is automatically applied to the in-memory result deduplication (it does not add `DISTINCT` to the SQL unless needed).

### Solution 2: @BatchSize (Best for Unpredictable Access)

```java
@Entity
public class Department {

    @OneToMany(mappedBy = "department")
    @BatchSize(size = 25)
    private List<Employee> employees = new ArrayList<>();
}
```

Now when you access employees, Hibernate loads up to 25 department employee collections in a single query:

```sql
-- Instead of 100 individual queries, Hibernate generates ~4 queries:
SELECT * FROM employees WHERE department_id IN (?, ?, ?, ..., ?)
-- With up to 25 IDs per batch
```

**Global batch size configuration:**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        default_batch_fetch_size: 16
```

### Solution 3: @Fetch(FetchMode.SUBSELECT)

```java
@Entity
public class Department {

    @OneToMany(mappedBy = "department")
    @Fetch(FetchMode.SUBSELECT)
    private List<Employee> employees = new ArrayList<>();
}
```

When any department employees collection is accessed, Hibernate loads ALL collections at once using a subselect:

```sql
SELECT * FROM employees
WHERE department_id IN (SELECT d.id FROM departments d)
-- All employee collections loaded in 1 query
```

### Solution 4: Entity Graph (JPA Standard)

```java
@Entity
@NamedEntityGraph(name = "Department.withEmployees",
    attributeNodes = @NamedAttributeNode("employees"))
public class Department { /* ... */ }
```

```java
EntityGraph<?> graph = session.getEntityGraph("Department.withEmployees");

List<Department> departments = session.createQuery(
        "FROM Department", Department.class)
    .setHint("jakarta.persistence.fetchgraph", graph)
    .getResultList();
// Hibernate generates a JOIN FETCH based on the graph
```

**Programmatic entity graph:**
```java
EntityGraph<Department> graph = session.createEntityGraph(Department.class);
graph.addAttributeNodes("employees");
graph.addSubgraph("employees").addAttributeNodes("department");

Map<String, Object> hints = Map.of("jakarta.persistence.fetchgraph", graph);
Department dept = session.find(Department.class, 1L, hints);
```

## 8.3 The Cartesian Product Problem

When JOIN FETCH-ing multiple collections simultaneously:

```java
// DO NOT DO THIS — Cartesian product
List<Department> departments = session.createQuery(
    "SELECT d FROM Department d JOIN FETCH d.employees JOIN FETCH d.projects",
    Department.class).getResultList();
// If a department has 10 employees and 5 projects:
// Result set = 10 x 5 = 50 rows per department
// Hibernate warning: "firstResult/maxResults specified with collection fetch; applying in memory!"
```

**Solutions:**

1. **Fetch one collection with JOIN FETCH, use @BatchSize for the other:**
```java
List<Department> departments = session.createQuery(
    "SELECT d FROM Department d JOIN FETCH d.employees",
    Department.class).getResultList();
// d.projects will be batch-loaded when accessed (via @BatchSize)
```

2. **Use separate queries:**
```java
List<Department> departments = session.createQuery(
    "FROM Department d JOIN FETCH d.employees", Department.class).getResultList();

session.createQuery(
    "FROM Department d JOIN FETCH d.projects WHERE d IN :depts", Department.class)
    .setParameter("depts", departments)
    .getResultList();
// Both collections are now initialized — Hibernate merges results into existing objects
```

## 8.4 DTO Projections for Read-Only Use Cases

Loading full entities when you only need a few fields wastes memory and CPU (dirty checking, snapshot storage). Use projections instead:

```java
public record EmployeeListItem(Long id, String fullName, String departmentName) {}
```

```java
List<EmployeeListItem> items = session.createQuery(
    """
    SELECT new com.example.dto.EmployeeListItem(e.id, e.fullName, d.name)
    FROM Employee e JOIN e.department d
    WHERE e.status = :status
    ORDER BY e.fullName
    """, EmployeeListItem.class)
    .setParameter("status", EmployeeStatus.ACTIVE)
    .setMaxResults(50)
    .getResultList();
// Returns lightweight DTOs — no entity management overhead
```

## 8.5 Pagination

### Offset-based Pagination

```java
int page = 2;
int pageSize = 20;

List<Employee> employees = session.createQuery(
        "FROM Employee e ORDER BY e.id", Employee.class)
    .setFirstResult((page - 1) * pageSize) // offset = 20
    .setMaxResults(pageSize)               // limit = 20
    .getResultList();
// SQL: SELECT ... FROM employees ORDER BY id LIMIT 20 OFFSET 20
```

**Problem with offset pagination on large datasets:** The database must still scan and discard `OFFSET` rows. Page 1000 of 20 items requires scanning 20,000 rows.

### Keyset (Cursor) Pagination — Better Performance

```java
Long lastSeenId = 1000L; // ID of the last item from previous page

List<Employee> employees = session.createQuery(
        "FROM Employee e WHERE e.id > :lastId ORDER BY e.id", Employee.class)
    .setParameter("lastId", lastSeenId)
    .setMaxResults(20)
    .getResultList();
// SQL: SELECT ... FROM employees WHERE id > 1000 ORDER BY id LIMIT 20
// Uses the index efficiently — no scanning skipped rows
```

## 8.6 Read-Only Optimization

For queries where you know the results will never be modified:

```java
// Session-level read-only
session.setDefaultReadOnly(true);

// Query-level read-only
List<Employee> employees = session.createQuery(
        "FROM Employee e WHERE e.status = :status", Employee.class)
    .setParameter("status", EmployeeStatus.ACTIVE)
    .setReadOnly(true)  // Hibernate skips snapshot creation
    .getResultList();
```

**Benefit:** Hibernate does not store a snapshot for dirty checking, reducing memory usage by ~50% per entity.

## 8.7 Performance Checklist

| Issue | Detection | Solution |
|-------|-----------|----------|
| N+1 queries | Enable SQL logging; count queries per request | JOIN FETCH, @BatchSize, Entity Graph |
| Cartesian product | Result set row count >> entity count | Separate queries or batch fetch |
| Loading unused fields | Full entity SELECT for list views | DTO projections |
| Slow pagination | Query time increases linearly with page number | Keyset pagination |
| Memory pressure | Heap growth during bulk reads | session.clear(), StatelessSession, read-only |
| Batch insert slowness | INSERT per row instead of batch | SEQUENCE ID + jdbc.batch_size + order_inserts |
| Too many round-trips | Many small queries | Batch fetching, query consolidation |


---

# 9. Transactions & Concurrency

## 9.1 Transaction Boundaries

A **transaction** is a unit of work that either fully completes (commit) or fully rolls back. In Hibernate, all write operations must happen within a transaction.

### Programmatic Transaction Management

```java
Session session = sf.openSession();
Transaction tx = null;
try {
    tx = session.beginTransaction();

    Employee emp = session.find(Employee.class, 1L);
    emp.setFullName("Updated Name");

    session.persist(new Employee("New Employee", "new@example.com"));

    tx.commit();
    // Both the UPDATE and INSERT are committed atomically
} catch (Exception e) {
    if (tx != null && tx.isActive()) {
        tx.rollback();
    }
    throw e;
} finally {
    session.close();
}
```

### Declarative Transaction Management (Spring — Recommended)

```java
@Service
@Transactional(readOnly = true) // Default for the class: read-only
public class EmployeeService {

    private final EmployeeRepository employeeRepository;

    public EmployeeService(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }

    // Inherits class-level @Transactional(readOnly = true)
    public Employee findById(Long id) {
        return employeeRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Employee not found: " + id));
    }

    @Transactional // Override: read-write transaction
    public Employee updateName(Long id, String newName) {
        Employee emp = employeeRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Employee not found: " + id));
        emp.setFullName(newName);
        // No explicit save() needed — dirty checking handles it
        return emp;
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void logAuditEvent(String message) {
        // Runs in a SEPARATE transaction — commits even if the caller rolls back
        auditRepository.save(new AuditEvent(message));
    }
}
```

**Why `readOnly = true` matters:**
1. Hibernate skips dirty checking (no snapshot storage)
2. Some JDBC drivers optimize read-only connections (e.g., PostgreSQL routes to a replica)
3. Signals intent clearly to other developers

### Transaction Propagation

| Propagation       | Behavior                                                  |
|--------------------|----------------------------------------------------------|
| `REQUIRED` (default) | Join existing transaction or create a new one          |
| `REQUIRES_NEW`     | Always create a new transaction; suspend the current one |
| `MANDATORY`        | Must run within an existing transaction; throw if none   |
| `SUPPORTS`         | Join if one exists; run without if none                  |
| `NOT_SUPPORTED`    | Suspend any current transaction; run without             |
| `NEVER`            | Throw if a transaction exists                            |
| `NESTED`           | Create a savepoint within the current transaction        |

**Common pitfall — self-invocation bypasses @Transactional:**

```java
@Service
public class EmployeeService {

    @Transactional
    public void methodA() {
        // This DOES NOT create a new transaction because it is a direct method call
        // within the same object — the Spring proxy is bypassed
        this.methodB();
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void methodB() {
        // ...
    }
}
```

**Fix:** Inject the service into itself (via `ApplicationContext`), or extract `methodB` into a separate `@Service`.

## 9.2 Isolation Levels

| Level              | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|--------------------|-----------|--------------------|--------------|----|
| `READ_UNCOMMITTED` | Possible  | Possible           | Possible     | Fastest |
| `READ_COMMITTED`   | Prevented | Possible           | Possible     | Good (default for most DBs) |
| `REPEATABLE_READ`  | Prevented | Prevented          | Possible     | Moderate |
| `SERIALIZABLE`     | Prevented | Prevented          | Prevented    | Slowest |

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
public void sensitiveOperation() {
    // Runs with REPEATABLE_READ isolation
}
```

**Best practice:** Use the database default (`READ_COMMITTED` for PostgreSQL/Oracle, `REPEATABLE_READ` for MySQL InnoDB). Only increase isolation for specific critical operations.

## 9.3 Optimistic Locking (@Version)

Optimistic locking assumes **conflicts are rare**. It detects concurrent modifications at commit time rather than preventing them with locks.

```java
@Entity
@Table(name = "products")
public class Product {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private BigDecimal price;

    @Column(name = "stock_quantity", nullable = false)
    private Integer stockQuantity;

    @Version
    private Integer version; // Hibernate manages this field automatically

    protected Product() {}

    public Product(String name, BigDecimal price, int stockQuantity) {
        this.name = name;
        this.price = price;
        this.stockQuantity = stockQuantity;
    }

    public Long getId() { return id; }
    public String getName() { return name; }
    public BigDecimal getPrice() { return price; }
    public void setPrice(BigDecimal price) { this.price = price; }
    public Integer getStockQuantity() { return stockQuantity; }
    public void setStockQuantity(Integer stockQuantity) { this.stockQuantity = stockQuantity; }
    public Integer getVersion() { return version; }
}
```

**How it works:**

```sql
-- When Hibernate updates a versioned entity:
UPDATE products
SET name = 'Widget', price = 19.99, stock_quantity = 50, version = 3
WHERE id = 1 AND version = 2
--                 ^^^^^^^^^^^^ This is the key!

-- If another transaction already incremented the version to 3,
-- this UPDATE matches 0 rows, and Hibernate throws OptimisticLockException
```

**Handling the conflict:**

```java
@Service
public class ProductService {

    @Transactional
    public Product updatePrice(Long productId, BigDecimal newPrice) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new EntityNotFoundException("Product not found"));
        product.setPrice(newPrice);
        return product;
        // If another transaction modified this product concurrently,
        // Hibernate throws OptimisticLockException at flush/commit time
    }
}

// In the controller or caller — retry logic
@RestController
public class ProductController {

    @PutMapping("/products/{id}/price")
    public ResponseEntity<Product> updatePrice(@PathVariable Long id,
                                                @RequestBody BigDecimal newPrice) {
        int maxRetries = 3;
        for (int attempt = 0; attempt < maxRetries; attempt++) {
            try {
                Product updated = productService.updatePrice(id, newPrice);
                return ResponseEntity.ok(updated);
            } catch (OptimisticLockException e) {
                if (attempt == maxRetries - 1) {
                    return ResponseEntity.status(HttpStatus.CONFLICT).build();
                }
                // Retry with fresh data
            }
        }
        return ResponseEntity.status(HttpStatus.CONFLICT).build();
    }
}
```

## 9.4 Pessimistic Locking

Pessimistic locking acquires a database-level lock when reading, preventing other transactions from modifying (or sometimes reading) the locked row.

```java
// PESSIMISTIC_WRITE — exclusive lock (SELECT ... FOR UPDATE)
Product product = session.find(Product.class, 1L,
    LockModeType.PESSIMISTIC_WRITE);
// SQL: SELECT * FROM products WHERE id=1 FOR UPDATE
// Other transactions block when trying to read/update this row

product.setStockQuantity(product.getStockQuantity() - 1);
// No conflict possible — we hold the lock
```

```java
// With timeout to avoid deadlocks
Product product = session.find(Product.class, 1L,
    LockModeType.PESSIMISTIC_WRITE,
    Map.of("jakarta.persistence.lock.timeout", 5000)); // 5 second timeout
```

### Lock Modes

| Lock Mode            | SQL                    | Blocks Others From         |
|----------------------|------------------------|----------------------------|
| `PESSIMISTIC_READ`   | `SELECT ... FOR SHARE` | Writing (not reading)      |
| `PESSIMISTIC_WRITE`  | `SELECT ... FOR UPDATE`| Reading and writing        |
| `PESSIMISTIC_FORCE_INCREMENT` | `FOR UPDATE` + version bump | Reading and writing + increments version |

### When to Use Which Locking Strategy

| Strategy     | Best For                                        | Trade-off                   |
|-------------|------------------------------------------------|----------------------------|
| Optimistic  | Most web applications; low contention           | Retry logic needed          |
| Pessimistic | High contention; financial transactions; inventory | Blocks other transactions; risk of deadlocks |

## 9.5 Concurrent Update Example: Inventory Management

```java
@Entity
@Table(name = "products")
public class Product {
    // ... fields with @Version ...

    public void decrementStock(int quantity) {
        if (this.stockQuantity < quantity) {
            throw new InsufficientStockException(
                "Requested " + quantity + " but only " + stockQuantity + " available");
        }
        this.stockQuantity -= quantity;
    }
}
```

**Optimistic approach (recommended for most cases):**

```java
@Service
public class OrderService {

    private final ProductRepository productRepository;
    private final OrderRepository orderRepository;

    @Transactional
    public Order placeOrder(Long productId, int quantity) {
        Product product = productRepository.findById(productId)
            .orElseThrow(() -> new EntityNotFoundException("Product not found"));

        product.decrementStock(quantity); // Throws if insufficient

        Order order = new Order(product, quantity, product.getPrice());
        return orderRepository.save(order);
        // If two users order simultaneously:
        // - First commit succeeds (version 1 -> 2)
        // - Second commit throws OptimisticLockException (expected version 1, found 2)
        // - Caller retries with fresh data
    }
}
```

**Pessimistic approach (for high-contention scenarios):**

```java
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdWithLock(@Param("id") Long id);
}

@Service
public class OrderService {

    @Transactional
    public Order placeOrderPessimistic(Long productId, int quantity) {
        Product product = productRepository.findByIdWithLock(productId)
            .orElseThrow(() -> new EntityNotFoundException("Product not found"));
        // Row is locked — other transactions wait

        product.decrementStock(quantity);

        Order order = new Order(product, quantity, product.getPrice());
        return orderRepository.save(order);
        // Lock released on commit
    }
}
```

---

# 10. Lifecycle Callbacks & Interceptors

## 10.1 JPA Lifecycle Callbacks

JPA defines callback annotations that fire at specific points in an entity lifecycle. These are ideal for auditing, validation, and computed fields.

| Callback        | Fires When                                    |
|----------------|-----------------------------------------------|
| `@PrePersist`  | Before INSERT (after persist() is called)      |
| `@PostPersist` | After INSERT (entity has an ID)               |
| `@PreUpdate`   | Before UPDATE (during flush)                   |
| `@PostUpdate`  | After UPDATE completes                         |
| `@PreRemove`   | Before DELETE                                  |
| `@PostRemove`  | After DELETE completes                         |
| `@PostLoad`    | After entity is loaded from DB or refreshed    |

### Inline Callbacks

```java
@Entity
@Table(name = "articles")
public class Article {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String content;

    @Column(nullable = false, length = 200)
    private String slug;

    @Column(name = "word_count")
    private Integer wordCount;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    protected Article() {}

    public Article(String title, String content) {
        this.title = title;
        this.content = content;
    }

    @PrePersist
    protected void onPrePersist() {
        this.createdAt = LocalDateTime.now();
        this.updatedAt = this.createdAt;
        this.slug = generateSlug(this.title);
        this.wordCount = countWords(this.content);
    }

    @PreUpdate
    protected void onPreUpdate() {
        this.updatedAt = LocalDateTime.now();
        this.slug = generateSlug(this.title);
        this.wordCount = countWords(this.content);
    }

    @PostLoad
    protected void onPostLoad() {
        // Compute transient fields after loading from DB
        // Example: set a transient "readTimeMinutes" field
    }

    private String generateSlug(String title) {
        return title.toLowerCase()
            .replaceAll("[^a-z0-9\\s-]", "")
            .replaceAll("\\s+", "-")
            .replaceAll("-+", "-")
            .replaceAll("^-|-$", "");
    }

    private int countWords(String text) {
        if (text == null || text.isBlank()) return 0;
        return text.trim().split("\\s+").length;
    }

    // Getters and setters
    public Long getId() { return id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
    public String getSlug() { return slug; }
    public Integer getWordCount() { return wordCount; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
}
```

### Entity Listener (Reusable Across Entities)

```java
public class AuditListener {

    @PrePersist
    public void setCreatedFields(Object entity) {
        if (entity instanceof Auditable auditable) {
            LocalDateTime now = LocalDateTime.now();
            auditable.setCreatedAt(now);
            auditable.setUpdatedAt(now);
            auditable.setCreatedBy(getCurrentUser());
        }
    }

    @PreUpdate
    public void setUpdatedFields(Object entity) {
        if (entity instanceof Auditable auditable) {
            auditable.setUpdatedAt(LocalDateTime.now());
            auditable.setUpdatedBy(getCurrentUser());
        }
    }

    private String getCurrentUser() {
        // In a Spring Security context:
        // return SecurityContextHolder.getContext().getAuthentication().getName();
        return "system";
    }
}
```

```java
public interface Auditable {
    void setCreatedAt(LocalDateTime createdAt);
    void setUpdatedAt(LocalDateTime updatedAt);
    void setCreatedBy(String createdBy);
    void setUpdatedBy(String updatedBy);
}
```

```java
@Entity
@Table(name = "employees")
@EntityListeners(AuditListener.class) // Attach the listener
public class Employee implements Auditable {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "full_name", nullable = false)
    private String fullName;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @Column(name = "created_by", length = 100)
    private String createdBy;

    @Column(name = "updated_by", length = 100)
    private String updatedBy;

    // Implement Auditable setters
    @Override
    public void setCreatedAt(LocalDateTime createdAt) { this.createdAt = createdAt; }
    @Override
    public void setUpdatedAt(LocalDateTime updatedAt) { this.updatedAt = updatedAt; }
    @Override
    public void setCreatedBy(String createdBy) { this.createdBy = createdBy; }
    @Override
    public void setUpdatedBy(String updatedBy) { this.updatedBy = updatedBy; }
}
```

### Spring Data JPA Auditing (Simpler Alternative)

```java
@Configuration
@EnableJpaAuditing
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        return () -> Optional.ofNullable(
            SecurityContextHolder.getContext().getAuthentication())
            .map(Authentication::getName);
    }
}
```

```java
@MappedSuperclass
@EntityListeners(AuditingEntityListener.class)
public abstract class AuditableEntity {

    @CreatedDate
    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(name = "updated_at")
    private LocalDateTime updatedAt;

    @CreatedBy
    @Column(name = "created_by", updatable = false)
    private String createdBy;

    @LastModifiedBy
    @Column(name = "updated_by")
    private String updatedBy;

    // Getters
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getUpdatedAt() { return updatedAt; }
    public String getCreatedBy() { return createdBy; }
    public String getUpdatedBy() { return updatedBy; }
}
```

```java
@Entity
@Table(name = "employees")
public class Employee extends AuditableEntity {
    // All audit fields are automatically populated
}
```

## 10.2 Hibernate Interceptors

Interceptors provide lower-level hooks into Hibernate operations. They can modify SQL, inspect entity state changes, and implement cross-cutting concerns.

```java
public class SqlLoggingInterceptor implements Interceptor {

    private static final Logger log = LoggerFactory.getLogger(SqlLoggingInterceptor.class);

    @Override
    public boolean onFlushDirty(Object entity, Object id, Object[] currentState,
                                 Object[] previousState, String[] propertyNames,
                                 Type[] types) {
        if (entity instanceof Employee emp) {
            log.info("Employee {} (id={}) is being updated", emp.getFullName(), id);
            for (int i = 0; i < propertyNames.length; i++) {
                if (!Objects.equals(currentState[i], previousState[i])) {
                    log.info("  Changed {}: {} -> {}",
                        propertyNames[i], previousState[i], currentState[i]);
                }
            }
        }
        return false; // false = no state modification by interceptor
    }

    @Override
    public boolean onSave(Object entity, Object id, Object[] state,
                           String[] propertyNames, Type[] types) {
        log.info("New entity being saved: {} (type={})",
            id, entity.getClass().getSimpleName());
        return false;
    }
}
```

**Registering the interceptor:**

```java
// Per-session
Session session = sf.withOptions()
    .interceptor(new SqlLoggingInterceptor())
    .openSession();

// Global (Spring Boot)
@Configuration
public class HibernateConfig {

    @Bean
    public HibernatePropertiesCustomizer hibernatePropertiesCustomizer() {
        return properties -> properties.put(
            "hibernate.session_factory.interceptor",
            new SqlLoggingInterceptor()
        );
    }
}
```

## 10.3 Hibernate Event Listeners

For more granular control than interceptors, Hibernate provides an event system:

```java
public class SoftDeleteEventListener implements PreDeleteEventListener {

    @Override
    public boolean onPreDelete(PreDeleteEvent event) {
        Object entity = event.getEntity();
        if (entity instanceof SoftDeletable softDeletable) {
            // Instead of deleting, set a flag
            softDeletable.setDeleted(true);
            softDeletable.setDeletedAt(LocalDateTime.now());

            // Cancel the DELETE and trigger an UPDATE instead
            Session session = event.getSession();
            session.merge(entity);

            return true; // true = veto the delete
        }
        return false; // false = proceed with delete normally
    }
}
```

**Registering event listeners (Spring Boot):**

```java
@Component
public class HibernateEventListenerConfig {

    @PersistenceUnit
    private EntityManagerFactory entityManagerFactory;

    @PostConstruct
    public void registerListeners() {
        SessionFactoryImpl sessionFactory = entityManagerFactory
            .unwrap(SessionFactoryImpl.class);
        EventListenerRegistry registry = sessionFactory
            .getServiceRegistry()
            .getService(EventListenerRegistry.class);

        registry.prependListeners(EventType.PRE_DELETE, new SoftDeleteEventListener());
    }
}
```

## 10.4 Use Cases Summary

| Mechanism           | Best For                                    | Complexity |
|---------------------|---------------------------------------------|-----------|
| `@PrePersist` etc.  | Auditing, computed fields, validation       | Low       |
| Entity Listeners    | Reusable callback logic across entities     | Low       |
| Spring `@EntityListeners` | Auditing with Spring Security context | Low       |
| Interceptors        | Cross-cutting concerns, logging, modification | Medium  |
| Event Listeners     | Soft delete, complex business rules at DB level | High  |


---

# 11. Advanced Hibernate Features

## 11.1 StatelessSession

A `StatelessSession` is a stripped-down session with **no persistence context**, **no dirty checking**, **no first-level cache**, and **no cascading**. Every operation is immediately translated to SQL.

```java
StatelessSession stateless = sf.openStatelessSession();
Transaction tx = stateless.beginTransaction();

try {
    // Bulk insert without first-level cache overhead
    for (int i = 0; i < 100_000; i++) {
        Employee emp = new Employee("Employee " + i, "emp" + i + "@example.com");
        stateless.insert(emp);
        // INSERT executes immediately — no flush() or clear() needed
    }

    // Bulk read with ScrollableResults — memory efficient
    ScrollableResults<Employee> scroll = stateless.createQuery(
            "FROM Employee e WHERE e.status = :status", Employee.class)
        .setParameter("status", EmployeeStatus.ACTIVE)
        .scroll(ScrollMode.FORWARD_ONLY);

    while (scroll.next()) {
        Employee emp = scroll.get();
        // Process — entity is NOT managed, no dirty checking
        emp.setStatus(EmployeeStatus.ON_LEAVE);
        stateless.update(emp); // Explicit update required (no dirty checking)
    }

    tx.commit();
} catch (Exception e) {
    tx.rollback();
    throw e;
} finally {
    stateless.close();
}
```

**When to use StatelessSession:**

| Use Case                    | Regular Session | StatelessSession |
|-----------------------------|-----------------|------------------|
| Bulk data import/export     | Slow (cache overhead) | Fast         |
| ETL processing              | Memory-heavy    | Memory-efficient  |
| Report generation           | Snapshot overhead | Lightweight    |
| Standard CRUD operations    | Best choice     | Too low-level     |
| Cascade operations needed   | Supported       | Not supported     |

**Limitations:**
- No lazy loading (all associations must be fetched eagerly or via explicit joins)
- No automatic dirty checking
- No cascading (persist/merge/remove child entities manually)
- No first-level cache (same entity loaded twice = two different Java objects)

## 11.2 Bean Validation Integration

Hibernate Validator (the reference implementation of Jakarta Bean Validation) integrates directly with Hibernate ORM. Constraints are checked automatically before INSERT and UPDATE.

```java
@Entity
@Table(name = "employees")
public class Employee {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotBlank(message = "Name is required")
    @Size(min = 2, max = 120, message = "Name must be between 2 and 120 characters")
    @Column(name = "full_name", nullable = false, length = 120)
    private String fullName;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    @Column(nullable = false, unique = true)
    private String email;

    @NotNull(message = "Salary is required")
    @DecimalMin(value = "0.01", message = "Salary must be positive")
    @Digits(integer = 10, fraction = 2, message = "Invalid salary format")
    private BigDecimal salary;

    @Past(message = "Hire date must be in the past")
    @Column(name = "hire_date")
    private LocalDate hireDate;

    // ...
}
```

**Spring Boot dependency (auto-configured):**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

**Validation happens at two levels:**

1. **JPA pre-persist / pre-update** — Hibernate triggers validation automatically before flushing. A `ConstraintViolationException` is thrown if validation fails.
2. **Spring MVC `@Valid`** — Validation at the controller level before the entity reaches Hibernate.

```java
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    @PostMapping
    public ResponseEntity<Employee> create(@Valid @RequestBody CreateEmployeeRequest request) {
        // @Valid triggers Bean Validation on the request DTO
        // If validation fails, Spring returns 400 Bad Request automatically
        Employee employee = employeeService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(employee);
    }
}
```

## 11.3 Hibernate Search

Hibernate Search bridges Hibernate ORM with full-text search engines (Lucene or Elasticsearch). It automatically indexes entities when they are persisted/updated.

**Dependencies:**
```xml
<dependency>
    <groupId>org.hibernate.search</groupId>
    <artifactId>hibernate-search-mapper-orm</artifactId>
    <version>7.1.0.Final</version>
</dependency>
<dependency>
    <groupId>org.hibernate.search</groupId>
    <artifactId>hibernate-search-backend-lucene</artifactId>
    <version>7.1.0.Final</version>
</dependency>
```

**Configuration:**
```yaml
spring:
  jpa:
    properties:
      hibernate:
        search:
          backend:
            type: lucene
            directory:
              type: local-filesystem
              root: ./data/search-indexes
```

**Indexing an entity:**
```java
@Entity
@Table(name = "articles")
@Indexed
public class Article {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @FullTextField(analyzer = "english")
    @KeywordField(name = "title_sort", sortable = Sortable.YES, normalizer = "lowercase")
    @Column(nullable = false)
    private String title;

    @FullTextField(analyzer = "english")
    @Column(columnDefinition = "TEXT")
    private String content;

    @GenericField
    @Column(name = "published_at")
    private LocalDateTime publishedAt;

    // ...
}
```

**Searching:**
```java
@Service
public class ArticleSearchService {

    private final EntityManager entityManager;

    public ArticleSearchService(EntityManager entityManager) {
        this.entityManager = entityManager;
    }

    public List<Article> search(String query, int maxResults) {
        SearchSession searchSession = Search.session(entityManager);

        return searchSession.search(Article.class)
            .where(f -> f.match()
                .fields("title", "content")
                .matching(query))
            .sort(f -> f.score())
            .fetchHits(maxResults);
    }

    // Rebuild the entire index (e.g., after initial deployment)
    public void reindexAll() throws InterruptedException {
        SearchSession searchSession = Search.session(entityManager);
        searchSession.massIndexer()
            .threadsToLoadObjects(4)
            .startAndWait();
    }
}
```

## 11.4 Native Result Mapping

When native queries return complex result sets that do not map directly to entities:

```java
@SqlResultSetMapping(
    name = "EmployeeStatsMapping",
    classes = @ConstructorResult(
        targetClass = EmployeeStats.class,
        columns = {
            @ColumnResult(name = "dept_name", type = String.class),
            @ColumnResult(name = "emp_count", type = Long.class),
            @ColumnResult(name = "avg_salary", type = BigDecimal.class),
            @ColumnResult(name = "max_salary", type = BigDecimal.class)
        }
    )
)
@Entity
public class Employee { /* ... */ }
```

```java
public record EmployeeStats(String deptName, Long empCount,
                              BigDecimal avgSalary, BigDecimal maxSalary) {}
```

```java
List<EmployeeStats> stats = session.createNativeQuery(
    """
    SELECT d.name AS dept_name,
           COUNT(e.id) AS emp_count,
           AVG(e.salary) AS avg_salary,
           MAX(e.salary) AS max_salary
    FROM employees e
    JOIN departments d ON e.department_id = d.id
    GROUP BY d.name
    ORDER BY emp_count DESC
    """, "EmployeeStatsMapping")
    .getResultList();
```

## 11.5 Multi-Tenancy

Multi-tenancy allows a single application to serve multiple tenants (customers) with data isolation.

### Strategy 1: Separate Databases

```yaml
# Each tenant gets a completely separate database
# Strongest isolation, most resource-intensive
```

```java
@Component
public class DatabaseMultiTenantConnectionProvider implements MultiTenantConnectionProvider<String> {

    private final Map<String, DataSource> dataSources;

    public DatabaseMultiTenantConnectionProvider(Map<String, DataSource> dataSources) {
        this.dataSources = dataSources;
    }

    @Override
    public Connection getAnyConnection() throws SQLException {
        return dataSources.values().iterator().next().getConnection();
    }

    @Override
    public Connection getConnection(String tenantId) throws SQLException {
        DataSource ds = dataSources.get(tenantId);
        if (ds == null) throw new IllegalArgumentException("Unknown tenant: " + tenantId);
        return ds.getConnection();
    }

    @Override
    public void releaseConnection(String tenantId, Connection connection) throws SQLException {
        connection.close();
    }

    // ... other required methods
}
```

### Strategy 2: Separate Schemas (Same Database)

```java
@Component
public class SchemaMultiTenantConnectionProvider implements MultiTenantConnectionProvider<String> {

    private final DataSource dataSource;

    @Override
    public Connection getConnection(String tenantId) throws SQLException {
        Connection connection = dataSource.getConnection();
        connection.setSchema(tenantId); // Switch schema: SET search_path TO 'tenant_abc'
        return connection;
    }
}
```

### Strategy 3: Discriminator Column (Shared Table)

```java
@Entity
@Table(name = "employees")
@FilterDef(name = "tenantFilter", parameters = @ParamDef(name = "tenantId", type = String.class))
@Filter(name = "tenantFilter", condition = "tenant_id = :tenantId")
public class Employee {

    @Column(name = "tenant_id", nullable = false)
    private String tenantId;

    // ... other fields
}
```

```java
// Enable filter for every session
Session session = entityManager.unwrap(Session.class);
session.enableFilter("tenantFilter").setParameter("tenantId", currentTenantId);
```

**Multi-tenancy strategy comparison:**

| Strategy          | Isolation | Complexity | Cost   | Use Case             |
|-------------------|-----------|------------|--------|----------------------|
| Separate Database | Strongest | High       | High   | Enterprise SaaS with strict compliance |
| Separate Schema   | Strong    | Medium     | Medium | Most SaaS applications |
| Discriminator     | Weakest   | Low        | Low    | Simple multi-tenant apps |

---

# 12. Spring Data JPA Integration

## 12.1 How Spring Data JPA Uses Hibernate

Spring Data JPA is an **abstraction layer** on top of JPA (and therefore Hibernate). It does not replace Hibernate — it simplifies common data access patterns by generating repository implementations automatically.

```
┌─────────────────────────────────┐
│        Your Application         │
├─────────────────────────────────┤
│      Spring Data JPA            │
│  (Repository interfaces,        │
│   derived queries, @Query)      │
├─────────────────────────────────┤
│       JPA API (Jakarta)         │
│  (EntityManager, TypedQuery)    │
├─────────────────────────────────┤
│     Hibernate ORM               │
│  (Session, dirty checking,      │
│   caching, SQL generation)      │
├─────────────────────────────────┤
│          JDBC / HikariCP        │
├─────────────────────────────────┤
│       PostgreSQL / MySQL        │
└─────────────────────────────────┘
```

When you call `repository.findById(1L)`, Spring Data JPA delegates to `entityManager.find()`, which Hibernate implements using a Session.

## 12.2 Project Structure

```
src/main/java/com/example/
├── Application.java                    # @SpringBootApplication
├── config/
│   └── JpaConfig.java                 # JPA/Auditing configuration
├── entity/
│   ├── BaseEntity.java                # @MappedSuperclass with audit fields
│   ├── Employee.java
│   ├── Department.java
│   └── enums/
│       └── EmployeeStatus.java
├── repository/
│   ├── EmployeeRepository.java        # Interface — Spring generates impl
│   └── DepartmentRepository.java
├── service/
│   ├── EmployeeService.java           # Business logic + @Transactional
│   └── DepartmentService.java
├── dto/
│   ├── CreateEmployeeRequest.java
│   ├── UpdateEmployeeRequest.java
│   └── EmployeeResponse.java
├── controller/
│   └── EmployeeController.java        # REST endpoints
└── exception/
    └── ResourceNotFoundException.java

src/main/resources/
├── application.yml
└── db/migration/                       # Flyway migrations
    ├── V1__create_departments.sql
    └── V2__create_employees.sql
```

## 12.3 Repository Interfaces

### JpaRepository (Recommended Starting Point)

```java
package com.example.repository;

import com.example.entity.Employee;
import com.example.entity.enums.EmployeeStatus;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // Derived query methods — Spring generates the JPQL and SQL

    Optional<Employee> findByEmail(String email);
    // SQL: SELECT * FROM employees WHERE email = ?

    List<Employee> findByStatus(EmployeeStatus status);
    // SQL: SELECT * FROM employees WHERE status = ?

    List<Employee> findByDepartmentIdAndStatus(Long departmentId, EmployeeStatus status);
    // SQL: SELECT * FROM employees WHERE department_id = ? AND status = ?

    List<Employee> findByFullNameContainingIgnoreCase(String namePart);
    // SQL: SELECT * FROM employees WHERE LOWER(full_name) LIKE LOWER('%' || ? || '%')

    List<Employee> findByHireDateAfterOrderByHireDateDesc(LocalDateTime date);
    // SQL: SELECT * FROM employees WHERE hire_date > ? ORDER BY hire_date DESC

    long countByStatus(EmployeeStatus status);
    // SQL: SELECT COUNT(*) FROM employees WHERE status = ?

    boolean existsByEmail(String email);
    // SQL: SELECT CASE WHEN COUNT(*) > 0 THEN TRUE ELSE FALSE END FROM employees WHERE email = ?

    void deleteByStatus(EmployeeStatus status);
    // SQL: DELETE FROM employees WHERE status = ?
}
```

### Repository Hierarchy

| Interface            | Extends            | Adds                                    |
|----------------------|--------------------|-----------------------------------------|
| `Repository<T, ID>` | —                  | Marker interface only                   |
| `CrudRepository`    | Repository         | save, findById, findAll, delete, count  |
| `ListCrudRepository`| CrudRepository     | findAll returns List instead of Iterable|
| `PagingAndSortingRepository` | CrudRepository | findAll(Pageable), findAll(Sort) |
| `JpaRepository`     | ListCrudRepository + PagingAndSortingRepository | flush, saveAndFlush, deleteInBatch, findAll(Example) |

## 12.4 @Query Annotation

For queries that cannot be expressed as derived query methods:

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // JPQL query
    @Query("SELECT e FROM Employee e JOIN FETCH e.department WHERE e.status = :status")
    List<Employee> findActiveWithDepartment(@Param("status") EmployeeStatus status);

    // JPQL with DTO projection
    @Query("""
        SELECT new com.example.dto.EmployeeResponse(
            e.id, e.fullName, e.email, d.name, e.status
        )
        FROM Employee e JOIN e.department d
        WHERE e.status = :status
        ORDER BY e.fullName
        """)
    List<EmployeeResponse> findEmployeeResponses(@Param("status") EmployeeStatus status);

    // Native SQL query
    @Query(value = """
        SELECT e.* FROM employees e
        WHERE e.full_name ILIKE :pattern
        ORDER BY e.full_name
        LIMIT :limit
        """, nativeQuery = true)
    List<Employee> searchByName(@Param("pattern") String pattern, @Param("limit") int limit);

    // Modifying query (UPDATE/DELETE)
    @Modifying
    @Query("UPDATE Employee e SET e.status = :newStatus WHERE e.status = :oldStatus")
    int bulkUpdateStatus(@Param("oldStatus") EmployeeStatus oldStatus,
                         @Param("newStatus") EmployeeStatus newStatus);
    // @Modifying is required for UPDATE/DELETE queries
    // Returns the count of affected rows
    // Must be called within a @Transactional method
}
```

### Interface-Based Projections

```java
// Projection interface — only selected columns are fetched
public interface EmployeeSummaryProjection {
    Long getId();
    String getFullName();
    String getEmail();
}
```

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    List<EmployeeSummaryProjection> findByStatus(EmployeeStatus status);
    // SQL: SELECT e.id, e.full_name, e.email FROM employees e WHERE e.status = ?
    // Only selected columns — more efficient than loading full entities
}
```

## 12.5 Pagination & Sorting

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    Page<Employee> findByStatus(EmployeeStatus status, Pageable pageable);

    // With @Query
    @Query("SELECT e FROM Employee e JOIN FETCH e.department WHERE e.status = :status")
    Page<Employee> findActiveWithDepartment(@Param("status") EmployeeStatus status,
                                             Pageable pageable);
}
```

```java
@Service
@Transactional(readOnly = true)
public class EmployeeService {

    private final EmployeeRepository employeeRepository;

    public EmployeeService(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }

    public Page<Employee> listActive(int page, int size, String sortBy) {
        Pageable pageable = PageRequest.of(page, size, Sort.by(Sort.Direction.ASC, sortBy));
        return employeeRepository.findByStatus(EmployeeStatus.ACTIVE, pageable);
    }
}
```

```java
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    private final EmployeeService employeeService;

    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }

    @GetMapping
    public Page<EmployeeResponse> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "fullName") String sortBy) {

        return employeeService.listActive(page, size, sortBy)
            .map(EmployeeResponse::fromEntity);
    }
}
```

**Response JSON:**
```json
{
    "content": [
        {"id": 1, "fullName": "Alice Johnson", "email": "alice@example.com", ...},
        {"id": 2, "fullName": "Bob Smith", "email": "bob@example.com", ...}
    ],
    "pageable": {
        "pageNumber": 0,
        "pageSize": 20,
        "sort": {"sorted": true, "orders": [{"property": "fullName", "direction": "ASC"}]}
    },
    "totalElements": 150,
    "totalPages": 8,
    "last": false
}
```

## 12.6 Transaction Management in Spring

### Standard Pattern

```java
@Service
public class EmployeeService {

    private final EmployeeRepository employeeRepository;
    private final DepartmentRepository departmentRepository;

    public EmployeeService(EmployeeRepository employeeRepository,
                           DepartmentRepository departmentRepository) {
        this.employeeRepository = employeeRepository;
        this.departmentRepository = departmentRepository;
    }

    @Transactional(readOnly = true)
    public EmployeeResponse findById(Long id) {
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Employee", id));
        return EmployeeResponse.fromEntity(employee);
    }

    @Transactional
    public EmployeeResponse create(CreateEmployeeRequest request) {
        // Validate business rules
        if (employeeRepository.existsByEmail(request.email())) {
            throw new DuplicateResourceException("Email already in use: " + request.email());
        }

        Department department = departmentRepository.findById(request.departmentId())
            .orElseThrow(() -> new ResourceNotFoundException("Department", request.departmentId()));

        Employee employee = new Employee(request.fullName(), department);
        employee.setEmail(request.email());

        Employee saved = employeeRepository.save(employee);
        return EmployeeResponse.fromEntity(saved);
    }

    @Transactional
    public EmployeeResponse update(Long id, UpdateEmployeeRequest request) {
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Employee", id));

        if (request.fullName() != null) {
            employee.setFullName(request.fullName());
        }
        if (request.email() != null && !request.email().equals(employee.getEmail())) {
            if (employeeRepository.existsByEmail(request.email())) {
                throw new DuplicateResourceException("Email already in use");
            }
            employee.setEmail(request.email());
        }
        // No explicit save — dirty checking handles the UPDATE

        return EmployeeResponse.fromEntity(employee);
    }

    @Transactional
    public void delete(Long id) {
        Employee employee = employeeRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Employee", id));
        employeeRepository.delete(employee);
    }
}
```

### DTOs

```java
package com.example.dto;

import com.example.entity.Employee;
import com.example.entity.enums.EmployeeStatus;

public record EmployeeResponse(
    Long id,
    String fullName,
    String email,
    String departmentName,
    EmployeeStatus status
) {
    public static EmployeeResponse fromEntity(Employee e) {
        return new EmployeeResponse(
            e.getId(),
            e.getFullName(),
            e.getEmail(),
            e.getDepartment() != null ? e.getDepartment().getName() : null,
            e.getStatus()
        );
    }
}

public record CreateEmployeeRequest(
    @NotBlank String fullName,
    @NotBlank @Email String email,
    @NotNull Long departmentId
) {}

public record UpdateEmployeeRequest(
    String fullName,
    @Email String email
) {}
```

## 12.7 Complete Spring Boot Application Example

### Application Entry Point

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

### Configuration

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.domain.AuditorAware;
import org.springframework.data.jpa.repository.config.EnableJpaAuditing;

import java.util.Optional;

@Configuration
@EnableJpaAuditing
public class JpaConfig {

    @Bean
    public AuditorAware<String> auditorProvider() {
        // Replace with SecurityContextHolder in a real application
        return () -> Optional.of("system");
    }
}
```

### application.yml

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp
    username: postgres
    password: secret
    hikari:
      maximum-pool-size: 20

  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false
    properties:
      hibernate:
        format_sql: true
        jdbc:
          batch_size: 25
        order_inserts: true
        order_updates: true
        default_batch_fetch_size: 16
        query:
          in_clause_parameter_padding: true

  flyway:
    enabled: true
    locations: classpath:db/migration

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.orm.jdbc.bind: TRACE
```

### Flyway Migration

```sql
-- V1__create_departments.sql
CREATE TABLE departments (
    id         BIGSERIAL PRIMARY KEY,
    name       VARCHAR(100) NOT NULL UNIQUE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP
);

-- V2__create_employees.sql
CREATE TABLE employees (
    id            BIGSERIAL PRIMARY KEY,
    full_name     VARCHAR(120) NOT NULL,
    email         VARCHAR(255) NOT NULL UNIQUE,
    status        VARCHAR(20) NOT NULL DEFAULT 'ACTIVE',
    department_id BIGINT NOT NULL REFERENCES departments(id),
    hire_date     TIMESTAMP,
    version       INTEGER NOT NULL DEFAULT 0,
    created_at    TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMP,
    created_by    VARCHAR(100),
    updated_by    VARCHAR(100)
);

CREATE INDEX idx_emp_dept ON employees(department_id);
CREATE INDEX idx_emp_status ON employees(status);
CREATE INDEX idx_emp_email ON employees(email);
```

### REST Controller

```java
package com.example.controller;

import com.example.dto.*;
import com.example.service.EmployeeService;
import jakarta.validation.Valid;
import org.springframework.data.domain.Page;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    private final EmployeeService employeeService;

    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }

    @GetMapping
    public Page<EmployeeResponse> list(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "fullName") String sortBy) {
        return employeeService.listActive(page, size, sortBy);
    }

    @GetMapping("/{id}")
    public EmployeeResponse findById(@PathVariable Long id) {
        return employeeService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public EmployeeResponse create(@Valid @RequestBody CreateEmployeeRequest request) {
        return employeeService.create(request);
    }

    @PutMapping("/{id}")
    public EmployeeResponse update(@PathVariable Long id,
                                    @Valid @RequestBody UpdateEmployeeRequest request) {
        return employeeService.update(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        employeeService.delete(id);
    }
}
```

### Global Exception Handler

```java
package com.example.exception;

import jakarta.persistence.EntityNotFoundException;
import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ProblemDetail handleDuplicate(DuplicateResourceException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidation(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatus(HttpStatus.BAD_REQUEST);
        problem.setTitle("Validation failed");
        problem.setProperty("errors", ex.getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList());
        return problem;
    }

    @ExceptionHandler(org.hibernate.StaleObjectStateException.class)
    public ProblemDetail handleOptimisticLock(org.hibernate.StaleObjectStateException ex) {
        return ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT,
            "The resource was modified by another user. Please refresh and try again.");
    }
}
```

## 12.8 Best Practices for Combining Spring Data JPA and Hibernate

### 1. Use Spring Data JPA repositories for standard operations

```java
// Let Spring Data handle the boilerplate
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    List<Employee> findByDepartmentId(Long departmentId);
}
```

### 2. Drop down to Hibernate when Spring Data is not enough

```java
@Repository
public class EmployeeCustomRepository {

    @PersistenceContext
    private EntityManager entityManager;

    public List<Employee> dynamicSearch(EmployeeSearchCriteria criteria) {
        CriteriaBuilder cb = entityManager.getCriteriaBuilder();
        CriteriaQuery<Employee> cq = cb.createQuery(Employee.class);
        Root<Employee> root = cq.from(Employee.class);

        List<Predicate> predicates = new ArrayList<>();

        if (criteria.name() != null) {
            predicates.add(cb.like(cb.lower(root.get("fullName")),
                "%" + criteria.name().toLowerCase() + "%"));
        }
        if (criteria.status() != null) {
            predicates.add(cb.equal(root.get("status"), criteria.status()));
        }
        if (criteria.departmentId() != null) {
            predicates.add(cb.equal(root.get("department").get("id"),
                criteria.departmentId()));
        }

        cq.where(predicates.toArray(new Predicate[0]));
        cq.orderBy(cb.asc(root.get("fullName")));

        return entityManager.createQuery(cq)
            .setMaxResults(criteria.maxResults())
            .getResultList();
    }
}
```

### 3. Never expose entities directly in API responses

```java
// BAD — entity leaks internal structure, lazy-loading issues, circular references
@GetMapping("/{id}")
public Employee getEmployee(@PathVariable Long id) {
    return employeeRepository.findById(id).orElseThrow();
}

// GOOD — DTO decouples internal model from API contract
@GetMapping("/{id}")
public EmployeeResponse getEmployee(@PathVariable Long id) {
    return employeeService.findById(id); // Returns DTO
}
```

### 4. Always disable Open Session in View (OSIV)

```yaml
spring:
  jpa:
    open-in-view: false  # Critical for production
```

OSIV keeps the Hibernate session open throughout the HTTP request, allowing lazy loading in controllers and views. This hides N+1 problems and keeps database connections checked out for the entire request duration.

### 5. Use Flyway or Liquibase for schema management

Never use `ddl-auto: create` or `ddl-auto: update` in production. They can silently drop columns, lose data, or create incorrect indexes.

### 6. Summary of Key Rules

| Rule | Why |
|------|-----|
| All `@ManyToOne` and `@OneToOne`: `fetch = LAZY` | Prevent N+1 queries |
| Use `@Version` on all modifiable entities | Prevent lost updates |
| `open-in-view: false` | No lazy loading surprises outside transactions |
| `ddl-auto: validate` in production | Flyway/Liquibase manages schema |
| `readOnly = true` on read methods | Performance + intent clarity |
| DTO projections for list endpoints | Avoid loading unused data |
| `Set` (not `List`) for `@ManyToMany` | Avoid delete-all-reinsert |
| Bidirectional sync helpers | Keep in-memory state consistent |
| `default_batch_fetch_size: 16` | Mitigate N+1 globally |
| Test with real database (Testcontainers) | H2 behaves differently than PostgreSQL |

---

# Appendix: Quick Reference

## Entity Annotation Cheat Sheet

```java
@Entity                          // Marks as JPA entity
@Table(name = "t", schema = "s") // Customize table
@Id                              // Primary key
@GeneratedValue(strategy = ...)  // Auto-generation
@Column(name, nullable, unique, length, precision, scale)
@Enumerated(EnumType.STRING)     // Map enum as string
@Transient                       // Exclude from persistence
@Version                         // Optimistic locking
@ManyToOne(fetch = LAZY)         // FK relationship
@OneToMany(mappedBy = "field")   // Inverse side
@OneToOne(mappedBy = "field")    // 1:1 inverse
@ManyToMany                      // Join table
@JoinColumn(name = "fk_col")    // FK column name
@JoinTable(name = "join_tbl")   // Join table name
@Embedded / @Embeddable          // Value object composition
@MappedSuperclass               // Shared fields, no table
@Inheritance(strategy = ...)     // Inheritance mapping
@NamedQuery(name, query)        // Pre-defined query
@Cache(usage = ...)             // Second-level cache
@BatchSize(size = N)            // Batch fetch size
@PrePersist / @PostLoad / etc.  // Lifecycle callbacks
@EntityListeners(Class.class)   // External callback class
```

## Spring Data JPA Derived Query Keywords

| Keyword            | Example                          | JPQL Equivalent                |
|--------------------|----------------------------------|--------------------------------|
| `findBy`           | `findByName(String)`             | `WHERE name = ?`               |
| `And`              | `findByNameAndStatus(...)`       | `WHERE name = ? AND status = ?`|
| `Or`               | `findByNameOrEmail(...)`         | `WHERE name = ? OR email = ?`  |
| `Between`          | `findByAgeBetween(int, int)`     | `WHERE age BETWEEN ? AND ?`    |
| `LessThan`         | `findByAgeLessThan(int)`         | `WHERE age < ?`                |
| `GreaterThan`      | `findByAgeGreaterThan(int)`      | `WHERE age > ?`                |
| `IsNull`           | `findByNameIsNull()`             | `WHERE name IS NULL`           |
| `Like`             | `findByNameLike(String)`         | `WHERE name LIKE ?`            |
| `Containing`       | `findByNameContaining(String)`   | `WHERE name LIKE '%' + ? + '%'`|
| `StartingWith`     | `findByNameStartingWith(String)` | `WHERE name LIKE ? + '%'`      |
| `OrderBy`          | `findByStatusOrderByNameAsc()`   | `ORDER BY name ASC`            |
| `In`               | `findByStatusIn(Collection)`     | `WHERE status IN (?)`          |
| `Not`              | `findByStatusNot(Status)`        | `WHERE status <> ?`            |
| `True` / `False`   | `findByActiveTrue()`             | `WHERE active = TRUE`          |
| `IgnoreCase`       | `findByNameIgnoreCase(String)`   | `WHERE LOWER(name) = LOWER(?)` |
| `Top` / `First`    | `findTop5ByStatus(Status)`       | `LIMIT 5`                      |
| `Distinct`         | `findDistinctByStatus(Status)`   | `SELECT DISTINCT ...`          |
| `countBy`          | `countByStatus(Status)`          | `SELECT COUNT(*) WHERE ...`    |
| `existsBy`         | `existsByEmail(String)`          | `SELECT CASE WHEN EXISTS ...`  |
| `deleteBy`         | `deleteByStatus(Status)`         | `DELETE WHERE ...`             |
