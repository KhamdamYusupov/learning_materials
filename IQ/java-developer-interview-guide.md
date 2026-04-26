# Java Developer Interview Preparation Guide

> A comprehensive, difficulty-graded interview preparation handbook for Java Backend Developer roles (2–10+ YOE).
> Covers Core Java through System Design, with emphasis on real-world interview relevance.

---

## 1. Core Java — OOP & Language Fundamentals

### Basic
- What are the four pillars of OOP? Explain each with a one-line real-world analogy.
- What is the difference between an abstract class and an interface? When would you choose one over the other?
- What is the difference between method overloading and method overriding?
- What is a constructor? What are its types and what restrictions does it have?
- What is the purpose of the `static` keyword? Can a top-level class be declared `static`?
- What is the difference between `==` and `equals()`?
- What is the difference between `final`, `finally`, and `finalize()`?
- What is the difference between `String`, `StringBuilder`, and `StringBuffer`?
- What is the String Pool? How does `intern()` work?
- What is autoboxing and unboxing?
- What is the difference between checked and unchecked exceptions?
- What is the difference between `throw` and `throws`?
- What is try-with-resources and why was it introduced?
- What is the difference between a process and a thread?
- What is typecasting? What is the difference between widening and narrowing?

### Intermediate
- What are the SOLID principles? Explain each with a Java code-level example.
- What is the difference between abstraction and encapsulation in terms of design intent?
- How does Java resolve which overloaded method to call at compile time vs which overridden method at runtime?
- What is the `equals()` and `hashCode()` contract? What breaks if you violate it?
- What is immutability? Write the steps to make a class with a mutable `List` field fully immutable.
- Why is `String` immutable? How does immutability help in multithreading, caching, and security?
- What is serialization and deserialization? What risks does Java serialization introduce?
- What is the difference between `transient` and `volatile` keywords?
- What is the difference between shallow copy and deep copy? How do you implement each?
- What is a marker interface? How does `Serializable` work without any methods?
- What are generics? What is type erasure and how does it affect runtime behavior?
- What happens if an exception is thrown inside a `finally` block?
- What is exception propagation? How does it differ for checked vs unchecked exceptions?
- How do you design custom exception hierarchies for a layered Spring Boot application?
- What is association, aggregation, and composition? How are they different at the JVM level?
- Can a static method be overridden? Explain method hiding with an example.
- What is a DTO? Why should you never expose JPA entities directly in REST responses?
- What is the difference between Java IO (blocking) and NIO (non-blocking)?

### Advanced
- Why is catching generic `Exception` or `Throwable` considered harmful? What should you catch instead?
- What is reflection? How does Spring use it internally, and why do performance-critical libraries avoid it?
- Why is `finalize()` deprecated? What are the alternatives (`Cleaner`, `PhantomReference`)?
- What is the cost of autoboxing in tight loops? How can it cause `OutOfMemoryError`?
- What are the 12-Factor App principles? Which ones directly influence how you write Java microservices?
- How does Java achieve platform independence? What are the limitations of "write once, run anywhere"?
- What is the Dependency Inversion Principle? How does Spring's IoC container implement it?
- Explain a scenario where violating the Liskov Substitution Principle caused a real bug.

---

## 2. Core Java — Design Patterns

### Basic
- What is the Singleton pattern? Write a basic implementation.
- What is the Factory pattern? Where have you seen it used in the JDK or Spring?
- What is the Builder pattern? When is it preferred over a constructor?
- Why are design patterns useful? Can you over-apply them?

### Intermediate
- What is the difference between Factory, Abstract Factory, and Builder patterns?
- What is the Strategy pattern? Give an example where you replaced `if-else` chains with it.
- What is the Observer pattern? How does Spring's event system (`ApplicationEvent`) use it?
- What is the Proxy pattern? How does Spring AOP leverage proxies?
- What is the Template Method pattern? Where is it used in the Spring framework?
- What is the Adapter pattern? Give a real-world Java example.
- How do design patterns improve testability?

### Advanced
- How do you make a Singleton thread-safe? Compare eager initialization, double-checked locking, enum, and Bill Pugh approaches.
- How can Singleton be broken via reflection, serialization, or cloning? How do you prevent each?
- Why is Enum Singleton considered the safest approach?
- What is the difference between the Decorator and Proxy patterns?
- When should you deliberately avoid using a design pattern?
- How would you refactor a large `switch` statement using the Strategy + Factory pattern combination?

---

## 3. Core Java — Modern Java (8–21)

### Basic
- What are the key features introduced in Java 8?
- What is a functional interface? Name three from `java.util.function`.
- What is a lambda expression? How does it relate to functional interfaces?
- What is a method reference? What are the four types?
- What are default methods in interfaces? Why were they introduced?
- What is the Stream API? What problem does it solve over traditional loops?
- What is `Optional`? How does it help avoid `NullPointerException`?

### Intermediate
- What is the difference between `map()` and `flatMap()` in Streams?
- What is the difference between intermediate and terminal operations? Give examples of each.
- What is the difference between `Optional.map()` and `Optional.flatMap()`?
- Why is `Optional` not recommended for fields, method parameters, or collections?
- When should you avoid parallel streams? What are the performance pitfalls?
- What is the difference between `parallelStream()` and manual multithreading with `ExecutorService`?
- How do lambdas affect debugging and stack traces?
- What are `Collectors` and how does `Collectors.groupingBy()` work?
- Explain `reduce()` — how does the identity, accumulator, and combiner work in parallel streams?
- What are text blocks (Java 13+)? Where are they most useful?
- How does `var` (Java 10) help with type inference? What are its limitations and when should you avoid it?

### Advanced
- What are sealed classes (Java 17)? How do they enable exhaustive pattern matching?
- How does pattern matching for `switch` (Java 17+) improve code over traditional `instanceof` chains?
- What are virtual threads (Java 21)? How do they differ from platform threads architecturally?
- When would virtual threads NOT help? What are their limitations with `synchronized` and native calls?
- What is the difference between `record`, sealed class, and traditional POJO? When would you use each?
- What is `CompletableFuture`? How does it differ from `Future`?
- What is the difference between `thenApply()`, `thenCompose()`, and `thenCombine()`?
- Why can `CompletableFuture` silently swallow exceptions? How do you handle errors properly?
- What are common production mistakes with `CompletableFuture` (wrong thread pool, blocking calls)?
- How does the common `ForkJoinPool` impact async APIs when used with `CompletableFuture`?
- How do Streams work internally (lazy evaluation, spliterator, short-circuiting)?
- What is the performance impact of Streams vs loops for small collections?

---

## 4. Core Java — JVM Internals & Memory Management

### Basic
- What is the difference between JDK, JRE, and JVM?
- What are the main memory areas in JVM (Heap, Stack, Metaspace, Code Cache)?
- What is garbage collection? Why is manual memory management unnecessary in Java?
- What is the difference between `OutOfMemoryError` and `StackOverflowError`?
- What is a ClassLoader? Name the three built-in ClassLoaders.

### Intermediate
- Explain JVM architecture: ClassLoader subsystem, runtime data areas, and execution engine.
- What is the difference between Metaspace and PermGen? Why was PermGen removed?
- What is the difference between Minor GC (Young Gen) and Major/Full GC (Old Gen)?
- What are the different garbage collectors (Serial, Parallel, G1, ZGC)? When would you choose each?
- What causes `OutOfMemoryError` even when heap size appears sufficient?
- What causes memory leaks in Java despite garbage collection? Give three real examples.
- How do you identify a memory leak in a running application (heap dump, MAT, VisualVM)?
- What are strong, weak, soft, and phantom references? Give a real use case for each.
- What is JIT compilation? How does the JVM decide which methods to compile?
- What is the difference between `ClassNotFoundException` and `NoClassDefFoundError`?
- What happens internally when you create a Java object (`new MyClass()`)?

### Advanced
- What is escape analysis? How does the JVM use it to allocate objects on the stack?
- What are safepoints? How do they cause stop-the-world pauses?
- What is false sharing? How does `@Contended` mitigate it?
- What JVM flags have you used in production (`-Xms`, `-Xmx`, `-XX:+UseG1GC`, `-XX:MaxGCPauseMillis`)?
- How do you tune JVM for high-throughput vs low-latency applications? What are the trade-offs?
- What JVM profiling tools have you used (JFR, JMC, jstack, jmap, async-profiler)?
- When does JVM allocate objects on the stack instead of the heap?
- How does G1 GC decide which regions to collect first?
- What is biased locking? When is it revoked? (Note: removed in Java 15+)
- How do ClassLoader memory leaks happen in application servers?
- What is string deduplication in G1 GC? Does it help speed or only memory?
- How does JVM warm-up affect benchmark results? What is the JMH framework?

---

## 5. Collections Framework

### Basic
- What is the difference between `List`, `Set`, and `Map`? When would you use each?
- What is the difference between `ArrayList` and `LinkedList`?
- What is the difference between `HashSet` and `TreeSet`?
- What is the difference between `HashMap` and `Hashtable`?
- What is the difference between `Comparable` and `Comparator`?
- What is the difference between `Iterator` and `ListIterator`?
- What is the `Iterable` interface and why is it important?

### Intermediate
- How does `HashMap` work internally? Explain hashing, buckets, and what changed in Java 8 (treeification at threshold 8).
- What happens when a `HashMap` reaches its load factor threshold? Describe the resizing process.
- Why does `HashMap` allow one null key but `ConcurrentHashMap` does not?
- Why is `HashMap` not thread-safe even for read-only operations?
- What is the difference between `HashMap`, `LinkedHashMap`, and `TreeMap`? When would you use each?
- How does `LinkedHashMap` maintain insertion order? How does access-order mode work?
- How does `TreeMap` maintain sorted order internally (Red-Black tree)?
- What is the difference between `ConcurrentHashMap` and `Collections.synchronizedMap()`?
- How does `ConcurrentHashMap` achieve thread safety without locking the entire map (segment locking vs CAS)?
- What is the difference between fail-fast and fail-safe iterators? Give examples of each.
- How do you avoid `ConcurrentModificationException`?
- What happens if you override `equals()` but not `hashCode()`? Demonstrate the bug.
- How does a `Set` prevent duplicate elements internally?
- How does `PriorityQueue` work internally (min-heap)?
- What is `CopyOnWriteArrayList`? When is it appropriate and when is it a performance trap?
- How do you implement custom sorting using `Comparator` with lambda expressions?

### Advanced
- Why is `HashSet` faster than `ArrayList` for `contains()` but slower for iteration?
- What is `WeakHashMap`? How does it decide when to evict keys? Give a real caching use case.
- When would you choose `TreeMap` over `HashMap` in a production system?
- How is `hashCode()` generated for `String`? What makes a good hash function?
- How does `ConcurrentHashMap.size()` work? Why is it not constant-time?
- How does `CopyOnWriteArrayList` perform under heavy write load? What is the alternative?
- What is the `NavigableMap` interface? What operations does it add over `SortedMap`?
- How would you implement an LRU cache using only JDK collections?
- Sorting millions of objects — `Comparable` or `Comparator`? What are the performance considerations?

---

## 6. Multithreading & Concurrency

### Basic
- What problem does multithreading solve? When can it make things worse?
- What is the thread lifecycle in Java? Explain the difference between `start()` and `run()`.
- What is the difference between `Runnable` and `Callable`?
- What is the difference between `wait()` and `sleep()`?
- What is the `synchronized` keyword? What does it guarantee?
- What is `volatile`? What problem does it solve?
- What is a deadlock? What are the four necessary conditions?
- What is a race condition?
- What is the difference between concurrency and parallelism?

### Intermediate
- What is `ExecutorService`? What are the different types of thread pools (`fixed`, `cached`, `scheduled`, `single`)?
- What is `ForkJoinPool`? How does work-stealing improve performance?
- What is the difference between `ExecutorService` and `ForkJoinPool`?
- What is the difference between a synchronized method and a synchronized block?
- What is object-level lock vs class-level lock?
- Why does `volatile` not guarantee atomicity? Give an example with `count++`.
- When should you prefer `volatile` over `synchronized`?
- What is the difference between `synchronized` and `Lock` (`ReentrantLock`)?
- What is `ReentrantLock`? What advantages does it have over `synchronized`?
- What are atomic variables (`AtomicInteger`, `AtomicReference`)? What is CAS?
- What is `CountDownLatch`? Give a practical example (waiting for services to initialize).
- What is `CyclicBarrier`? How does it differ from `CountDownLatch`?
- What is `Semaphore`? When would you use it?
- What is `BlockingQueue`? How is it used in the Producer-Consumer pattern?
- What is `ThreadLocal`? Give a real use case (request-scoped data, transaction context).
- What is thread starvation? How does it differ from deadlock?
- What is thread safety? List five ways to achieve it in Java.
- How does immutability eliminate the need for synchronization?
- Does `Thread.sleep()` release a lock? Does `wait()` release a lock?
- Why must `wait()` be called inside a `synchronized` block?
- Why are `wait()`, `notify()`, and `notifyAll()` defined in `Object`, not `Thread`?
- What is the difference between `yield()` and `join()`?
- How do you safely stop a running thread (interrupt + flag pattern)?

### Advanced
- What is the happens-before relationship in the Java Memory Model? Give three examples.
- Why is Double-Checked Locking broken without `volatile`? Explain the memory visibility issue.
- What is instruction reordering? How does the JVM/CPU reorder instructions and why is it dangerous?
- How does Java handle biased, lightweight, and heavyweight locks internally?
- What is the cost of context switching? How does it affect throughput?
- What is `ThreadLocal`? How can it cause memory leaks in thread pools and servlet containers?
- What happens when a thread pool queue is full? How do rejection policies (`AbortPolicy`, `CallerRunsPolicy`) work?
- What happens if a thread dies while holding a lock?
- What happens if a thread throws an uncaught exception inside a `synchronized` block?
- What is the difference between `LockSupport.park()/unpark()` and `wait()/notify()`?
- What is over-synchronization? How does it reduce throughput?
- How do you detect deadlocks in production (jstack, `ThreadMXBean`)?
- How do you design a high-performance, thread-safe counter without `synchronized`?
- How do you ensure safe publication of shared objects?
- What is false sharing and how does it impact multithreaded performance on multicore CPUs?
- How do virtual threads (Java 21) change the concurrency model? Do they eliminate the need for `synchronized`?
- How do you size a thread pool? Explain the difference between CPU-bound and IO-bound tuning formulas.

---

## 7. Spring Boot

### Basic
- What is Spring Boot? Why is it preferred over the traditional Spring framework?
- What is `@SpringBootApplication`? What three annotations does it combine?
- What is Dependency Injection? What are the types (constructor, setter, field)?
- What is the Spring IoC Container?
- What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?
- What is `@Autowired`? What is the difference between constructor injection and field injection?
- What is `@Qualifier` and when do you need it?
- What are bean scopes? Name the five scopes in Spring.
- What is `application.properties` / `application.yml`? What is it used for?
- What is `@RestController` vs `@Controller`?
- What is `ResponseEntity`?
- What is the difference between `@RequestParam`, `@RequestBody`, and `@PathVariable`?

### Intermediate
- How does Spring Boot auto-configuration work internally? What role does `@Conditional` play?
- What is the role of `spring.factories` / `AutoConfiguration.imports` in auto-configuration?
- What happens internally when a Spring Boot application starts (bootstrap sequence)?
- What is the difference between `@Bean` and `@Component`?
- What is the difference between `@Configuration` and `@Component` (full vs lite mode)?
- What is the difference between `ApplicationContext` and `BeanFactory`?
- What is the lifecycle of a Spring Bean (`@PostConstruct` -> initialization -> `@PreDestroy`)?
- What is the difference between Singleton and Prototype beans? What happens when you inject a Prototype into a Singleton?
- How does Spring handle circular dependencies? What are the strategies to resolve them?
- What happens if two beans of the same type exist and no `@Qualifier` is specified?
- What is `@Value` vs `@ConfigurationProperties`? When should you use each?
- What is `DispatcherServlet`? Describe the full request lifecycle in Spring MVC.
- How do you implement global exception handling with `@ControllerAdvice` and `@ExceptionHandler`?
- What is the difference between `@ControllerAdvice` and `@RestControllerAdvice`?
- What is the difference between Filters and Interceptors? When do you use each?
- How does Spring Boot handle profile-specific configuration?
- What is the priority order of configuration sources (CLI args > env vars > YAML > properties)?
- How does Spring Boot detect and configure an embedded server (Tomcat/Jetty)?
- What is the difference between a fat JAR and a normal JAR?
- What is Spring Boot Actuator? Name five useful endpoints.
- How do you implement pagination and sorting using Spring Data?
- What are Spring Boot Starters? What happens when you add `spring-boot-starter-web`?
- What is AOP in Spring? Explain `@Aspect`, `@Around`, `@Before`, `@After` with a logging example.
- How do you validate request bodies using `@Valid`, `@NotNull`, `@Size`?
- What is `CommandLineRunner` vs `ApplicationRunner`?

### Advanced
- How does Spring perform Dependency Injection internally (reflection, CGLIB proxies)?
- What is lazy initialization? When is it useful and what are the risks?
- How does Spring decide bean initialization order without `@DependsOn`?
- What happens if `@PostConstruct` throws an exception? Does the application start?
- Why does `@Transactional` not work for private methods or self-invocation? Explain the proxy mechanism.
- Why does `@Async` sometimes not run asynchronously? What common mistakes cause this?
- Why does `@Cacheable` sometimes not cache? What are the proxy-related gotchas?
- What is `OncePerRequestFilter`? When would you write a custom one?
- How do you implement graceful shutdown in Spring Boot?
- How do you reduce Spring Boot startup time (lazy init, class indexing, GraalVM native images)?
- What is the difference between WebClient, RestTemplate, and FeignClient? Which is recommended and why?
- What is reactive programming in Spring (WebFlux)? When would you use it over MVC?
- What is reactive backpressure? How does Project Reactor handle it?
- How do you configure multiple data sources in a single Spring Boot application?
- How do you manage database migrations with Flyway or Liquibase?
- How do you implement feature toggles safely in Spring Boot?
- How do you handle logging levels and configuration across environments?
- What are common Spring Boot performance mistakes (eager fetching, missing connection pool tuning, synchronous calls)?
- How does Spring Boot integrate with cloud services (AWS S3, RDS, SQS)?
- How do you create a custom Spring Boot Starter?
- How does Spring convert low-level database exceptions into consistent runtime exceptions (exception translation)?
- How does `@Conditional` family (`@ConditionalOnProperty`, `@ConditionalOnClass`, `@ConditionalOnMissingBean`) work?
- What is the difference between `@Import` and `@EnableAutoConfiguration`?

---

## 8. Spring Security

### Basic
- What is Spring Security? What problems does it solve?
- What is the difference between authentication and authorization?
- What is `UserDetailsService`? What is its role in authentication?
- What is `PasswordEncoder`? Why should passwords never be stored in plain text?
- What is the difference between hashing and encryption?
- What is CSRF? Why is CSRF protection important for web applications?
- What is CORS? How is it different from CSRF?
- What is JWT? What are its three parts (header, payload, signature)?

### Intermediate
- What are the core components of Spring Security (SecurityFilterChain, AuthenticationManager, AuthenticationProvider, GrantedAuthority)?
- How does the Spring Security filter chain work internally?
- How do you configure Spring Security using a component-based approach (post `WebSecurityConfigurerAdapter` deprecation)?
- What is `SecurityFilterChain`? How do you define one as a bean?
- How does JWT authentication flow work in Spring Boot (filter -> validate -> set SecurityContext)?
- What is the difference between Access Token and Refresh Token? How do you implement the refresh flow?
- How do you handle JWT expiration? Where should tokens be stored (HttpOnly cookie vs localStorage)?
- What is stateless vs stateful authentication? Why is stateless preferred in microservices?
- What is OAuth 2.0? Name the four grant types and when to use each.
- What is the difference between JWT and OAuth2? (Hint: they solve different problems)
- What is OpenID Connect (OIDC)? How does it extend OAuth 2.0?
- How do you implement role-based access control (RBAC)?
- What is method-level security? How do `@PreAuthorize` and `@PostAuthorize` work?
- How do you configure CORS in a Spring Boot application?
- What is hashing and salting? Why is BCrypt preferred over SHA-256 for passwords?

### Advanced
- What is `SecurityContext` and `SecurityContextHolder`? How is the context propagated across async threads?
- What is session fixation? How does Spring Security protect against it?
- How does Spring Security handle concurrent session management?
- What is zero-trust architecture? How do you apply it to microservices?
- How do you secure inter-service communication (mTLS, service mesh, JWT propagation)?
- How do you handle token propagation across microservices in an OAuth2 system?
- How do you secure Spring Boot Actuator endpoints?
- How does the Authorization Code Grant flow with PKCE work? Why is the implicit flow deprecated?
- What is a `DelegatingPasswordEncoder`? How does it support password migration?
- How do you implement custom authentication providers for non-standard auth mechanisms?

---

## 9. Hibernate / JPA

### Basic
- What is JPA? How is it different from Hibernate?
- What is ORM? What problem does it solve?
- What is an Entity? What annotations make a POJO a JPA entity?
- What is the role of `@Entity`, `@Table`, `@Id`, `@Column`?
- What is `@GeneratedValue`? What are the ID generation strategies (AUTO, IDENTITY, SEQUENCE, TABLE)?
- What is an `EntityManager`?
- What is a Persistence Context?
- What is lazy loading vs eager loading?
- What is the difference between `CrudRepository` and `JpaRepository`?
- What is `@Transactional`?

### Intermediate
- What is the entity lifecycle (Transient, Managed, Detached, Removed)? Draw the state transitions.
- What is dirty checking? How does Hibernate detect changes without explicit `update()` calls?
- What is flushing? What is the difference between `flush()` and `commit()`?
- What is the difference between `save()`, `persist()`, and `merge()`?
- What is the difference between `save()` and `saveAndFlush()`?
- What is the difference between `@OneToOne`, `@OneToMany`, `@ManyToOne`, and `@ManyToMany`?
- What is the owning side of a relationship? What is `mappedBy`?
- What is cascading? What is the difference between `CascadeType.ALL` and `orphanRemoval`?
- How do you avoid infinite recursion in bidirectional mappings (`@JsonIgnore`, `@JsonManagedReference`)?
- What is the N+1 select problem? Demonstrate it and show two solutions.
- What is the difference between `JOIN` and `JOIN FETCH` in JPQL?
- What is `@EntityGraph`? How does it differ from fetch joins?
- What is the first-level cache (session cache)? Is it enabled by default?
- What is the second-level cache? Which providers are commonly used (EhCache, Hazelcast)?
- What is `LazyInitializationException`? What are the proper fixes (not `OpenSessionInView`)?
- What are the default fetch types for `@OneToMany` (LAZY) and `@ManyToOne` (EAGER)?
- What is HQL/JPQL? How is it different from native SQL?
- What is `@Query`? How do derived query methods work (`findByNameAndAge`)?
- How do you implement pagination and sorting in Spring Data JPA?
- What are transaction propagation types? Explain `REQUIRED` vs `REQUIRES_NEW` with a real scenario.
- What are transaction isolation levels? What is the default for most databases?
- What is optimistic locking? How does `@Version` work?
- What is pessimistic locking? When is it appropriate?

### Advanced
- How does Hibernate's second-level cache work internally? What are cache concurrency strategies?
- How do you optimize large batch inserts (batch sizing, `hibernate.jdbc.batch_size`, clearing session)?
- What is the Criteria API? When would you use it over JPQL?
- How do you write dynamic queries safely (Criteria API vs `Specification`)?
- How do you map inheritance in JPA (SINGLE_TABLE, TABLE_PER_CLASS, JOINED)? What are the trade-offs?
- How do you map composite primary keys (`@IdClass` vs `@EmbeddedId`)?
- What is `@Embeddable` and `@Embedded`? Give a real use case (Address, Money).
- How do you implement auditing (`@CreatedDate`, `@LastModifiedDate`, `@CreatedBy`)?
- How do you implement soft deletes (`@Where`, `@SQLDelete`)?
- What is multi-tenancy in Hibernate? What are the strategies (separate DB, separate schema, discriminator)?
- How do you map JSON columns in JPA?
- What happens if you update an entity without calling `save()`? (Hint: dirty checking in managed state)
- How does Hibernate decide when to flush (FlushMode.AUTO vs COMMIT)?
- What are common JPA anti-patterns (N+1, `OpenSessionInView`, entity as DTO, wrong fetch type)?
- When should you NOT use JPA/Hibernate and prefer plain JDBC or jOOQ instead?
- How do you tune queries generated by JPA (`EXPLAIN ANALYZE`, slow query log, Hibernate statistics)?
- How do you handle schema evolution safely (Flyway, Liquibase)?
- What is the difference between `EntityManager` and Hibernate `Session`?
- What are projections in Spring Data JPA (interface-based, class-based, dynamic)?
- How do you handle concurrent updates and optimistic locking failures gracefully in a REST API?

---

## 10. Microservices Architecture

### Basic
- What are microservices? How do they differ from monolithic architecture?
- What are the key benefits of microservices?
- What are the key challenges of microservices?
- When should you NOT use microservices?
- What is service-to-service communication?
- What is an API Gateway? Why is it needed?
- What is service discovery?
- What is the Circuit Breaker pattern?
- What is the database-per-service pattern?

### Intermediate
- How do you decompose a monolith into microservices (DDD, bounded contexts)?
- What is Domain-Driven Design (DDD)? What is a bounded context?
- How do you decide service boundaries? What happens when services are split incorrectly?
- What is the Strangler Fig pattern? How do you migrate without downtime?
- What is a distributed monolith? How do you avoid creating one?
- How do microservices communicate (sync vs async)? When would you choose REST vs messaging vs gRPC?
- What problems arise from synchronous service-to-service calls at scale?
- What is event-driven architecture? When is it useful?
- What is the difference between Feign Client, WebClient, and RestTemplate?
- What is the Circuit Breaker pattern? Explain the three states (Closed, Open, Half-Open).
- How does Resilience4j implement Circuit Breaker, Retry, and Bulkhead?
- What is the difference between Retry and Circuit Breaker? When can retries make things worse (retry storm)?
- How do you handle timeouts between services?
- What is graceful degradation? What is a fallback mechanism?
- How do you prevent cascading failures?
- What is eventual consistency? Give a real-world example.
- What is the Saga pattern? Explain choreography vs orchestration with trade-offs.
- What is two-phase commit (2PC)? Why is it often avoided in microservices?
- What is service discovery? How do Eureka/Consul handle it?
- What is the difference between client-side and server-side discovery?
- What is the difference between an API Gateway and a load balancer?
- What is Spring Cloud Config Server? How does centralized configuration work?
- How do you manage secrets in microservices (Vault, AWS Secrets Manager)?
- What is distributed tracing? How do you implement it (Zipkin, Jaeger, OpenTelemetry)?
- How do you implement centralized logging (ELK stack)?
- How do you monitor microservices health (Prometheus, Grafana, Spring Boot Actuator)?

### Advanced
- What is the Outbox pattern? How does it solve the dual-write problem?
- What is CQRS? When should you use it and when is it overkill?
- What is event sourcing? How does it differ from traditional CRUD?
- What is the Saga pattern at scale? How do you handle compensation and partial failure rollback?
- What is the difference between an API Gateway and a Service Mesh? When would you use Istio vs Spring Cloud Gateway?
- What is idempotency? How do you design idempotent APIs in practice (idempotency key pattern)?
- What is API versioning? Compare URI versioning, header versioning, and content negotiation.
- How do you handle backward compatibility when evolving APIs?
- What is the sidecar pattern? How does it relate to service mesh?
- What is zero-trust architecture applied to microservices?
- How do you secure inter-service communication (mTLS, JWT propagation)?
- What is the difference between exactly-once and at-least-once delivery semantics?
- What is a dead-letter queue (DLQ)? When and how should it be used?
- How do you design stateless services?
- How do you handle multi-region deployment?
- What is chaos engineering? How would you implement it (Chaos Monkey)?
- What are the risks of sharing libraries between microservices?
- How do you handle data consistency across services without distributed transactions?
- What is blue-green deployment? How does it differ from canary and rolling deployments?
- How do you achieve zero-downtime deployments?
- How do you handle rollbacks safely during production failures?
- How do you trace a single request across 10+ microservices?
- How do you debug a production issue when logs are scattered across services?
- What metrics matter most for microservices in production (latency, error rate, throughput, saturation)?

---

## 11. SQL & Database

### Basic
- What is the difference between `WHERE` and `HAVING`?
- What is the difference between `INNER JOIN`, `LEFT JOIN`, and `RIGHT JOIN`?
- What is a primary key? Can a table have multiple primary keys?
- What is a foreign key? Why is it important?
- What is the difference between `DELETE`, `TRUNCATE`, and `DROP`?
- What is the difference between `CHAR` and `VARCHAR`?
- What is the difference between `UNION` and `UNION ALL`?
- What are ACID properties? Explain each.
- What is normalization? Why is 3NF commonly used?

### Intermediate
- What is a self join? Give a real-world use case (employee-manager hierarchy).
- What is the difference between correlated and non-correlated subqueries?
- What is the difference between `NOT IN` and `NOT EXISTS`? Which is usually faster?
- What are window functions? Explain `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()`.
- What is a CTE (Common Table Expression)? When would you use it over a subquery?
- What is an index? What are clustered vs non-clustered indexes?
- What is a composite index? Does column order matter? (Yes, leftmost prefix rule)
- What is a covering index? How does it eliminate table lookups?
- When should you NOT create an index?
- How does indexing improve query performance? What are the write-performance drawbacks?
- What is an execution plan? How do you read it (`EXPLAIN ANALYZE`)?
- What is a transaction isolation level? What is the default for MySQL/PostgreSQL?
- What is the difference between dirty read, non-repeatable read, and phantom read?
- What is a database deadlock? How do databases detect and resolve them?
- What is the difference between shared and exclusive locks?
- What is denormalization? When is it better than normalization?
- How does SQL handle `NULL` in comparisons, `WHERE`, and aggregations?
- What is the query execution order (FROM -> WHERE -> GROUP BY -> HAVING -> SELECT -> ORDER BY -> LIMIT)?
- How do you find duplicate records using `GROUP BY` and `HAVING`?
- How do you optimize a slow SQL query? List five techniques.

### Advanced
- What is the difference between optimistic and pessimistic locking at the database level?
- What is a materialized view? How does it differ from a regular view?
- How does pagination work with `LIMIT` and `OFFSET`? Why is it slow for large tables? What are the alternatives (keyset pagination)?
- What is database sharding? How do you decide the shard key?
- What is a read replica? What problems does replication lag introduce?
- What is the difference between ACID and BASE consistency models?
- What is the difference between OLTP and OLAP databases?
- What is write amplification? How does it affect SSD-based databases?
- What is SQL injection? How do you prevent it in Java (parameterized queries, prepared statements)?
- What is a stored procedure? When should you use it vs querying from Java?
- What is a trigger? What are the dangers of overusing triggers?
- How do you handle transactions across multiple databases (distributed transactions, Saga)?
- How do you design pagination for very large datasets (cursor-based pagination)?
- What is cardinality? How does it affect index effectiveness?
- What is referential integrity? What happens with cascade delete/update?

### SQL Coding Questions
- Write a query to find the Nth highest salary per department using window functions.
- Write a query to find the 2nd highest salary without using `LIMIT` or window functions.
- Write a query to find duplicate records and their count.
- Write a query to calculate a running total using window functions.
- Write a query to find the average salary per department, filtered to departments with avg > 50000.
- Write a query to fetch paginated records (e.g., page 3, 20 records per page).
- Write a query to find employees who earn more than their manager (self join).

---

## 12. REST API Design

### Basic
- What is a REST API? What are its key constraints?
- What is the difference between REST and SOAP?
- What is the difference between `GET`, `POST`, `PUT`, `PATCH`, and `DELETE`?
- What is the difference between `PUT` and `PATCH`?
- What are HTTP status codes? Categorize 2xx, 3xx, 4xx, and 5xx.
- What does "stateless" mean in REST?

### Intermediate
- What is idempotency? Which HTTP methods are idempotent?
- How do you design idempotent APIs in practice (idempotency key)?
- How do you implement API versioning (URI path, query param, header, content type)?
- How do you implement pagination, sorting, and filtering in REST APIs?
- How do you validate request payloads in Spring Boot (`@Valid`, Bean Validation)?
- What is content negotiation? How does Spring Boot handle it?
- How do you handle backward compatibility when evolving APIs?
- How do you implement rate limiting for APIs (token bucket vs leaky bucket)?
- What is API throttling? How does it differ from rate limiting?
- What is HATEOAS? When is it worth implementing?
- What is the difference between HTTP/1.1, HTTP/2, and HTTP/3?
- How do you design REST APIs for bulk operations?
- What is the difference between 400, 401, 403, and 404? Give scenarios for each.
- How do you handle long-running operations in REST APIs (async patterns, polling, webhooks)?

### Advanced
- How do you design a REST API that is both backward-compatible and extensible?
- What are the trade-offs between REST and gRPC? When would you choose gRPC?
- How do you handle partial failures in a REST API call chain?
- How do you implement a custom rate limiter using Redis and the sliding window algorithm?
- What is API Gateway pattern vs BFF (Backend for Frontend) pattern?
- How do you design APIs for file upload with progress tracking and resumable uploads?

---

## 13. Kafka & Messaging

### Basic
- What is Apache Kafka? What problems does it solve?
- What is the difference between a message queue and an event stream?
- What are Kafka's core concepts (producer, consumer, broker, topic, partition, offset)?
- What is a consumer group?
- What is the difference between Kafka and RabbitMQ at a high level?

### Intermediate
- How do Kafka partitions enable horizontal scalability?
- How do Kafka partitions and replication factor ensure fault tolerance?
- What delivery guarantees does Kafka provide (at-most-once, at-least-once, exactly-once)?
- How do you ensure message ordering in Kafka? (Hint: same key -> same partition)
- How do you handle message duplication (idempotent consumers)?
- What happens if a consumer crashes after reading but before committing an offset?
- What is consumer lag? What causes it and how do you resolve it?
- What is a dead-letter queue (DLQ)? How do you implement it in Kafka?
- How do you handle poison messages (messages that always fail processing)?
- How do you integrate Kafka with Spring Boot (`@KafkaListener`, `KafkaTemplate`)?
- What is the difference between Kafka and RabbitMQ? When would you choose each?

### Advanced
- What is exactly-once semantics in Kafka? How does the transactional API enable it?
- What challenges occur during Kafka partition rebalancing? How does the cooperative rebalance protocol help?
- How do you handle schema evolution without breaking consumers (Schema Registry, Avro)?
- What is event versioning strategy?
- How do you design event schemas for forward and backward compatibility?
- What factors lead to consumer lag and how do you diagnose it (consumer group lag monitoring)?
- In which scenarios should Kafka NOT be used?
- How does Kafka's log compaction work? When would you use it?
- How do you achieve cross-datacenter Kafka replication (MirrorMaker)?
- What is the Outbox pattern with Kafka? How does it solve the dual-write problem?
- How does Kafka's ISR (In-Sync Replicas) mechanism work?

---

## 14. System Design

### Basic
- What is the CAP theorem? Can you have all three?
- What is the difference between horizontal and vertical scaling?
- What is a load balancer? What are the common algorithms (round-robin, least connections, IP hash)?
- What is caching? At which levels can you cache (client, CDN, API gateway, application, database)?
- What is a CDN and how does it improve performance?

### Intermediate
- How do you design a URL shortener (hash generation, collision handling, redirection, analytics)?
- How do you design an LRU cache? What data structures do you use?
- How do you design a rate limiter (token bucket, sliding window)?
- What is connection pooling? How does HikariCP work?
- What is database sharding? How do you choose a shard key?
- What is a read replica? What problems does replication lag introduce?
- How do you design stateless services?
- What are caching strategies (Cache Aside, Write Through, Write Behind, Read Through)?
- What is cache invalidation? Why is it considered one of the hardest problems?
- How do you handle high-throughput APIs in Java?

### Advanced
- How do you design a distributed chat system (WebSocket, presence, message ordering)?
- How do you design a scalable payment system (idempotency, reconciliation, eventual consistency)?
- How do you design microservices for an e-commerce checkout flow (inventory, cart, payment, order)?
- How do you design a distributed tracing system?
- How do you design a notification system that sends alerts to millions of users in real time?
- How do you design a system for high availability (redundancy, failover, health checks)?
- How do you identify and fix performance bottlenecks in a Java backend system?
- What is consistent hashing? How does it help in distributed caching?
- What is the difference between leader-follower and leader-leader replication?
- How do you design a search system similar to Elasticsearch?
- How do you design a cache system with TTL, eviction policies (LRU, LFU), and distributed invalidation?
- How do you handle thundering herd / cache stampede problems?

---

## 15. DevOps & Cloud

### Basic
- What is Docker? What is the difference between a Docker image and a container?
- What is the difference between Docker and a virtual machine?
- What is Kubernetes? Why is it used?
- What is a CI/CD pipeline?
- What is the difference between blue-green and canary deployments?

### Intermediate
- How do you Dockerize a Spring Boot application? Write a sample `Dockerfile`.
- What is a multi-stage Docker build? What are its advantages?
- What is the difference between Docker volumes and bind mounts?
- What is the difference between Kubernetes Deployments and StatefulSets?
- What are liveness and readiness probes? How do they differ?
- How do rolling updates work in Kubernetes?
- How does Kubernetes handle auto-scaling (HPA, VPA)?
- How do you integrate microservices in a CI/CD pipeline (Jenkins, GitHub Actions)?
- How do you achieve zero-downtime deployments?
- How do you implement logging, tracing, and monitoring in the cloud (CloudWatch, ELK, Prometheus)?

### Advanced
- How do you secure secrets and credentials in cloud environments (AWS Secrets Manager, Vault, K8s Secrets)?
- What is the difference between AWS EC2, ECS, EKS, and Lambda? When would you choose each?
- How do you handle multi-region deployment for disaster recovery?
- How do you scale using Auto Scaling Groups vs Kubernetes HPA?
- What is Infrastructure as Code? How do Terraform and CloudFormation compare?
- How do you implement GitOps with ArgoCD or Flux?
- What is service mesh (Istio, Linkerd)? When is it worth the complexity?
- How do you design a cost-effective cloud architecture?

---

## 16. Testing

### Basic
- What is the difference between unit tests and integration tests?
- What is Test-Driven Development (TDD)?
- What is Mockito? What is the difference between `@Mock` and `@InjectMocks`?
- What is the difference between `@MockBean` and `@Mock`?

### Intermediate
- How do you write integration tests with Spring Boot (`@SpringBootTest`, `@DataJpaTest`)?
- What is Testcontainers? When would you use it over H2 for testing?
- What is the test pyramid? How do you balance unit, integration, and E2E tests?
- How do you test REST APIs (MockMvc, RestAssured)?
- How do you mock external service calls in tests (WireMock)?
- What is the difference between `@WebMvcTest` and `@SpringBootTest`?
- How do you test `@Transactional` behavior and rollback in tests?
- How do you test asynchronous code (`@Async`, `CompletableFuture`)?

### Advanced
- What is contract testing (Pact, Spring Cloud Contract)? How does it prevent integration failures?
- How do you test microservices end-to-end without deploying the full system?
- How do you test Kafka producers and consumers (`@EmbeddedKafka`)?
- How do you measure and enforce code coverage without gaming the metric?
- What is mutation testing? How does it improve test quality beyond code coverage?
- How do you performance-test a Spring Boot application (Gatling, JMeter)?
- What are the best practices for writing maintainable test suites?

---

## 17. Coding & Problem Solving

### Data Structures & Algorithms
- Find the second highest number in an array (single pass).
- Find the first non-repeating character in a string.
- Find the longest substring without repeating characters.
- Check if a number is a palindrome without converting to String.
- Find the maximum sum subarray (Kadane's Algorithm).
- Find the maximum sum of K consecutive elements (sliding window).
- Find all pairs whose sum equals a target (Two Sum — HashMap approach).
- Find all triplets whose sum equals a target.
- Reverse a linked list iteratively and recursively.
- Detect a cycle in a linked list (Floyd's algorithm).
- Sort a stack using another stack.
- Check if a binary tree is height-balanced.
- Merge two sorted arrays in-place.
- Implement binary search.
- Find the top K frequent elements.
- Implement an LRU cache using `LinkedHashMap` or `HashMap` + doubly linked list.
- Implement the Producer-Consumer problem using `BlockingQueue`.
- Implement a thread-safe counter using `AtomicInteger`.
- Design a custom `Queue` using two stacks.
- Count the frequency of each character in a string.

### Java Stream API Coding
- Find the second highest salary from `List<Employee>` using Streams.
- Group employees by department: `Map<Department, List<Employee>>`.
- Find the highest-paid employee in each department.
- Filter the top 3 highest salaries per department.
- Count word occurrences in a sentence using Streams.
- Find the longest string in a list.
- Calculate average, min, and max salary using `IntSummaryStatistics`.
- Partition a list into even and odd numbers using `Collectors.partitioningBy()`.
- Sum values using `reduce()` with an identity.
- Flatten a `List<List<String>>` into `List<String>` using `flatMap()`.
- Convert `List<Employee>` to `Map<Integer, String>` (id -> name) handling duplicates.
- Remove null values from a list.
- Sort employees by salary descending, then by name ascending.
- Remove duplicates from a list while preserving order.
- Count characters in a string and return `Map<Character, Long>`.
- Print index positions of all vowels in a string using Streams and `IntStream`.

### SQL Coding
- Find the Nth highest salary per department using `DENSE_RANK()`.
- Find the 2nd highest salary without using `LIMIT` or window functions.
- Find employees earning more than their manager (self join).
- Calculate a running total of sales by date.
- Find duplicate records and their count.
- Fetch paginated records (page 3, 20 records per page) using offset and keyset approaches.
- Find the average salary per department where avg salary exceeds 50,000.

---

## 18. Scenario-Based & Production Debugging

### Application-Level Scenarios
- Your Spring Boot app works locally but fails after deployment. What are the first five things you check?
- APIs are fast locally but slow only in production. How do you debug this systematically?
- You updated `application.properties` but changes are not reflecting. What are the possible causes?
- CPU usage is low but requests are timing out. What could be blocking threads?
- Application memory usage keeps increasing over time. How do you investigate?
- `@Transactional` is present but rollback doesn't happen. List all possible causes.
- Database connection pool gets exhausted under load. How do you identify and fix it?
- Scheduled background jobs start affecting API latency. How do you isolate them?
- Application behaves differently in Docker vs locally. What are the common causes?
- Logs are missing in production but fine locally. Where do you look?
- A REST API returns correct data but response time is inconsistent. What are the possible causes?
- Service restarts automatically without clear errors. What could trigger this?
- After enabling parallel streams, response time increased. Why?
- Excessive logging was added for debugging and production performance degraded severely. Why?
- Lazy loading works locally but fails in production with `LazyInitializationException`. What is missing?

### Microservices-Level Scenarios
- A downstream service is slow and your service also starts failing. How do you protect it?
- Scaling instances didn't improve performance. What are the possible reasons?
- Circuit breaker is open even when the downstream service is healthy. What could be wrong?
- A cache improves performance initially, then degrades it. What could cause this?
- A retry mechanism caused system overload (retry storm). What design mistake was made?
- One microservice deployment causes system-wide latency spike. How do you investigate?
- Two services update data that must stay consistent but you cannot use distributed transactions. How do you solve it?
- Autoscaling is configured but performance does not improve under load. Why?
- Your service works fine standalone but fails behind an API Gateway. What do you check?
- A Kafka consumer is falling behind (consumer lag is growing). How do you diagnose and fix it?

---

## 19. Behavioral & Leadership Questions

### Self & Experience
- Tell me about yourself and your current project/role.
- Walk through a production issue you debugged. What was the root cause and how did you fix it?
- Describe a microservices project you designed. What trade-offs did you make?
- What is the most impactful technical improvement you made at your current company?
- Describe a time when you chose a new technology or approach over the traditional one. What was the outcome?

### Teamwork & Leadership
- Describe a situation where you showed technical leadership.
- How do you mentor junior developers?
- How do you handle conflicts within a team?
- Describe a time you disagreed with your manager. How did you handle it?
- How do you handle critical feedback?
- How do you conduct code reviews? What do you look for?

### Decision Making & Planning
- How do you prioritize tasks under tight deadlines?
- How do you estimate technical tasks?
- How do you decide when to take on technical debt vs paying it off?
- How do you use AI tools in your development and testing workflow?
- Why are you leaving your current company?
- Why do you want to join this company?
- Where do you see yourself in 3–5 years?

---

## Missing Topics Added

The following sections were **not present or severely underrepresented** in the original file and have been added:

| New Section / Topic | Reason for Addition |
|---|---|
| **Design Patterns** (dedicated section) | Was scattered across Core Java with no structure or difficulty grading |
| **REST API Design** (expanded) | Original had only 17 flat questions; expanded with difficulty levels, bulk operations, long-running ops, gRPC comparison |
| **Testing** (expanded significantly) | Original had only 8 questions; added MockMvc, WireMock, contract testing, mutation testing, performance testing |
| **Kafka & Messaging** (restructured) | Was flat; added difficulty levels, exactly-once semantics, Schema Registry, log compaction, Outbox pattern |
| **System Design** (expanded) | Added consistent hashing, cache stampede, CDN, replication patterns, test pyramid |
| **Behavioral & Leadership** (restructured) | Original was minimal; added decision-making, code review, AI tools, estimation questions |
| **Spring Security** (restructured) | Added PKCE flow, DelegatingPasswordEncoder, SecurityContext propagation in async threads |
| **Production Debugging** (microservices scenarios) | Added Kafka lag diagnosis, API Gateway debugging, retry storm, autoscaling failures |

---