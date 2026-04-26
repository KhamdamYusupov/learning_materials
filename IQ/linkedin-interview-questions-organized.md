# Java Backend Developer — Interview Questions (Curated & Deduplicated)

---

## Core Java

### OOP & Fundamentals
- What are the four pillars of OOP? Explain with examples.
- What are the SOLID principles? Explain with real-world use cases.
- What is the difference between abstraction and encapsulation?
- What is the difference between an abstract class and an interface?
- What is the difference between method overloading and method overriding?
- How does Java handle method overloading vs overriding at runtime?
- What is a constructor in Java and what are its types?
- What is the purpose of the `static` keyword?
- What is the difference between `final`, `finally`, and `finalize()`?
- Why is `finalize()` deprecated and what are its risks?
- What is the difference between `==` and `equals()`?
- What is the `equals()` and `hashCode()` contract? What breaks if it is violated?
- What is immutability? How do you design an immutable class in Java?
- Why is `String` immutable in Java? What are its real advantages?
- What is the difference between `String`, `StringBuilder`, and `StringBuffer`?
- What is the String Pool and how does `intern()` work?
- What is typecasting in Java?
- What is autoboxing and unboxing? What is the cost internally?
- What is serialization and deserialization? Why can serialization be risky?
- What is the difference between `transient` and `volatile`?
- What is the difference between shallow copy and deep copy?
- What is reflection in Java? Why is it used and what are its performance implications?
- Can a static method be overridden? What is method hiding?
- Can the `main()` method be overloaded? What happens when you do?
- What happens if `main()` method is not static?
- What is a marker interface and what is its purpose?
- What are generics in Java and why are they used?
- What is the difference between checked and unchecked exceptions?
- What is the difference between `throw` and `throws`?
- What happens if an exception is thrown inside a `finally` block?
- How do you design custom exceptions in a Spring Boot application?
- Why is catching `Exception` or `Throwable` considered bad practice?
- What is try-with-resources?
- What is exception propagation?
- How does Java achieve platform independence?
- What is the difference between a process and a thread?
- What are design patterns? Name commonly used ones in Java.
- What is the difference between Factory, Abstract Factory, and Builder patterns?
- What is the Strategy pattern?
- What is the Observer pattern?
- What is the Proxy pattern?
- When should you avoid using design patterns?
- What is the Dependency Inversion Principle?
- What is a DTO and why is it used?
- What is association, aggregation, and composition?
- What are the 12-Factor App principles?
- What is the difference between Java IO and NIO?

### Modern Java (8–21)
- What are the key features introduced in Java 8?
- What are functional interfaces? Why are they important?
- What is a lambda expression and why is it useful?
- What is a method reference? How does the JVM resolve it?
- What are default and static methods in interfaces?
- What is the Stream API? How does it work internally?
- What is the difference between `map()` and `flatMap()` in Streams?
- What is the difference between intermediate and terminal operations in Streams?
- What is the difference between `Optional.map()` and `Optional.flatMap()`?
- What is `Optional`? How does it help avoid `NullPointerException`?
- Why is `Optional` not recommended for fields or method parameters?
- When should you avoid using parallel streams?
- What is the difference between `parallelStream()` and traditional multithreading?
- How do Streams impact performance with large datasets?
- Streams vs loops — which is better and when?
- How do lambdas affect debugging and code readability?
- What are the benefits of sealed classes (Java 17)?
- How does pattern matching for `switch` improve code readability (Java 17+)?
- What are text blocks and where are they useful?
- What are virtual threads (Java 21) and how do they improve scalability?
- What is the difference between `record`, sealed class, and traditional POJO?
- How does `var` help with type inference and what are its limitations?
- What is `CompletableFuture` and how does it enable asynchronous programming?
- What is the difference between `CompletableFuture.thenApply()` and `thenCompose()`?
- What are common mistakes with `CompletableFuture` in production?
- Why can `CompletableFuture` silently swallow exceptions?
- What is the difference between `Future` and `CompletableFuture`?
- What are `Collectors` and `Accumulators` in the Stream API?

### JVM Internals & Memory
- What is the difference between JDK, JRE, and JVM?
- Explain JVM architecture.
- What is the Java Memory Model (Heap, Stack, Metaspace)?
- What is the difference between Metaspace and PermGen?
- What is a ClassLoader? Explain the ClassLoader hierarchy.
- When is a custom ClassLoader needed?
- What is the difference between `ClassNotFoundException` and `NoClassDefFoundError`?
- How does garbage collection work in Java?
- What are the different types of garbage collectors (Serial, Parallel, G1, ZGC, CMS)?
- What is the difference between Minor GC and Major GC?
- What is GC tuning and when is it needed?
- What causes `OutOfMemoryError` even when GC is running?
- What is the difference between `OutOfMemoryError` and `StackOverflowError`?
- What causes memory leaks in Java even with garbage collection?
- How do you identify and fix memory leaks in a running application?
- What is the difference between a memory leak and memory overflow?
- What are strong, weak, soft, and phantom references? What are their real use cases?
- What is JIT compilation and how does JVM optimize code at runtime?
- What is escape analysis and how does JVM use it to optimize memory?
- When does JVM allocate objects on the stack instead of the heap?
- What are safepoints and stop-the-world pauses?
- What JVM flags have you used in production and why?
- How do you tune JVM for high-throughput vs low-latency systems?
- What JVM tools have you used for profiling (JFR, JMC, jstack, jmap)?
- What is false sharing and how does `@Contended` fix it?
- What happens internally when you create a Java object?

---

## Collections

- What is the difference between `List`, `Set`, and `Map`?
- What is the difference between `ArrayList` and `LinkedList`? When would you choose each?
- What is the difference between `HashSet` and `TreeSet`?
- Why is `HashSet` faster than `ArrayList` for search operations?
- What is the difference between `Iterator` and `ListIterator`?
- What is the difference between fail-fast and fail-safe iterators?
- How do you avoid `ConcurrentModificationException`?
- How does `HashMap` work internally? What changes were introduced in Java 8 (treeification)?
- What happens when a `HashMap` reaches threshold capacity (resizing)?
- Why does `HashMap` allow one null key but `Hashtable` does not?
- Why is `HashMap` not thread-safe even for read operations?
- What is the difference between `HashMap`, `LinkedHashMap`, and `TreeMap`?
- How does `TreeMap` maintain its order?
- How does `LinkedHashMap` maintain insertion order?
- What is the difference between `HashMap`, `Hashtable`, and `ConcurrentHashMap`?
- How does `ConcurrentHashMap` achieve thread safety without locking the entire map?
- Why does `ConcurrentHashMap` not allow null keys or values?
- What is the difference between `ConcurrentHashMap` and `Collections.synchronizedMap()`?
- What is the difference between `CopyOnWriteArrayList` and `ArrayList`?
- When should you use `CopyOnWriteArrayList` over a synchronized list?
- How does `PriorityQueue` work internally?
- What is the difference between `Comparable` and `Comparator`?
- How do you implement custom sorting using `Comparator` and `Comparable`?
- What happens if you override `equals()` but not `hashCode()`?
- When would you choose `TreeMap` over `HashMap`?
- How does a `Set` prevent duplicate elements?
- What is the `Iterable` interface?
- How is `hashCode()` generated?
- What is `WeakHashMap` and how does it decide key eviction?

---

## Multithreading & Concurrency

- What problem does multithreading solve in Java?
- What is the thread lifecycle? Explain the difference between `Thread.start()` and `run()`.
- What is the difference between `Runnable` and `Callable`?
- What is `ExecutorService`? How does a thread pool work internally?
- What are the different types of thread pools?
- What is `ForkJoinPool` and when should you use it?
- How does `ForkJoinPool` steal tasks internally?
- What is the difference between `ExecutorService` and `ForkJoinPool`?
- What is the `synchronized` keyword? How does it work internally (monitor, object header)?
- What is the difference between a synchronized method and a synchronized block?
- What is object-level lock vs class-level lock?
- What is `volatile`? What problem does it solve?
- Why does `volatile` not guarantee atomicity?
- When should `volatile` be preferred over `synchronized`?
- What is the difference between `synchronized` and `Lock`?
- What is `ReentrantLock`?
- What is a `Semaphore`?
- What is `CountDownLatch`?
- What is `CyclicBarrier`? How does it differ from `CountDownLatch`?
- What are atomic variables? What is Compare-And-Swap (CAS)?
- What is `ThreadLocal`? How can it cause memory leaks?
- What is `BlockingQueue`?
- What is a deadlock? What are the necessary conditions for deadlock?
- How do you prevent or detect deadlocks in Java?
- What is a race condition? Give a real production example.
- What is thread starvation? How does it differ from deadlock?
- What is thread safety? How can it be achieved in Java?
- How does immutability reduce synchronization needs?
- Why is Double-Checked Locking broken without `volatile`?
- What is the happens-before relationship in the Java Memory Model?
- What is the difference between `wait()` and `sleep()`?
- Why must `wait()` be called inside a synchronized block?
- Does `Thread.sleep()` release a lock?
- Why are `wait()`, `notify()`, and `notifyAll()` defined in the `Object` class?
- What is the difference between `yield()` and `join()`?
- What is the difference between `park/unpark` and `wait/notify`?
- What is the difference between concurrency and parallelism?
- What is over-synchronization and why is it harmful?
- What is instruction reordering and why is it dangerous?
- What is the cost of context switching in multithreading?
- How do you safely stop a running thread?
- What happens when a thread pool queue is full?
- What happens when a thread dies while holding a lock?
- How does Java handle biased, lightweight, and heavyweight locks?
- How do you design a high-performance, thread-safe counter?
- How do you ensure safe publication of shared objects?
- What happens if a thread throws an exception inside a `synchronized` block?

---

## Spring Boot

### Core & Internals
- What is Spring Boot and why is it preferred over Spring?
- What is the difference between Spring and Spring Boot?
- What is `@SpringBootApplication`? What does it do internally?
- What is the difference between `@ComponentScan` and `@EnableAutoConfiguration`?
- How does Spring Boot auto-configuration work internally?
- What is the role of `spring.factories` / `AutoConfiguration.imports`?
- What is the role of `SpringFactoriesLoader`?
- What happens internally when a Spring Boot application starts?
- How does Spring Boot reduce boilerplate/XML configuration?
- What is convention over configuration in Spring Boot?
- What happens when you add `spring-boot-starter-web`?
- What are Spring Boot Starters and what is their purpose?
- How do you create custom starter dependencies?

### Dependency Injection & Beans
- What is Dependency Injection? What are the types of DI?
- How does Dependency Injection work internally in Spring (using reflection)?
- What is the Spring IoC Container?
- What is the difference between `ApplicationContext` and `BeanFactory`?
- What is `@Component`? What is `@Service`? What is `@Repository`?
- What is the difference between `@Component`, `@Service`, `@Repository`, and `@Controller`?
- What is the difference between `@Bean` and `@Component`?
- What is the difference between `@Configuration` and `@Component`?
- What is `@Qualifier` and when do you use it?
- What is `@Primary` and how does it differ from `@Qualifier`?
- What is the difference between `@Autowired` and `@Inject`?
- What is the difference between constructor injection and field injection?
- What are bean scopes in Spring (singleton, prototype, request, session)?
- What is the difference between Singleton and Prototype beans? How do you inject a Prototype into a Singleton?
- What is the lifecycle of a Spring Bean?
- What is lazy initialization of beans and when should you use it?
- How does Spring handle circular dependencies?
- How does Spring decide bean initialization order when no `@DependsOn` is present?
- What happens if two beans of the same type exist and no `@Qualifier` is specified?
- What happens if a bean fails during startup?
- What is `@PostConstruct` and `@PreDestroy`?
- What happens if `@PostConstruct` throws an exception?
- What is the use of `CommandLineRunner` and `ApplicationRunner`?
- What is `@Conditional` and how does it enable flexible configuration?
- What is AOP in Spring? Explain with a real use case (logging, security, auditing).
- What is a Spring proxy and how does it work?
- What is `@Value` and `@ConfigurationProperties`? When to use each?
- How does Spring Boot handle classpath scanning?

### Configuration & Profiles
- How do you externalize configuration in Spring Boot?
- What is `application.properties` vs `application.yml`?
- What happens if both `application.yml` and `application.properties` exist?
- What is the priority order of configuration sources (properties, YAML, env vars, CLI args)?
- How does Spring Boot load configuration files internally?
- How does Spring Boot handle profile-specific configuration?
- How do you manage multiple environments (dev, test, prod)?
- How do you safely reload configs without restarting?

### REST & MVC
- What is `@RestController` vs `@Controller`?
- What is `DispatcherServlet` and how does the Spring MVC flow work?
- What happens internally when you hit a REST endpoint?
- What is `ResponseEntity` and how do you return proper HTTP status codes?
- What is the difference between `PUT` and `PATCH`?
- What is the difference between `@RequestParam`, `@RequestBody`, and `@PathVariable`?
- How do you validate incoming request data?
- How do you implement global exception handling (`@ControllerAdvice`, `@ExceptionHandler`)?
- What is the difference between `@ControllerAdvice` and `@RestControllerAdvice`?
- What is the difference between Filters and Interceptors?
- What is `OncePerRequestFilter`?
- How do you handle CORS in Spring Boot?
- How does file upload and multipart request handling work?
- How do you implement pagination and sorting in Spring Boot?
- How do you design a REST API in Spring Boot?
- How does Spring Boot handle exception translation?
- What is `@RequestScope` vs `@SessionScope`?

### Embedded Server & Deployment
- How does Spring Boot detect and configure an embedded server (Tomcat/Jetty/Netty)?
- How do you customize embedded server configuration?
- What is the difference between a fat JAR and a normal JAR?
- How does Spring Boot decide server port priority?
- What are the deployment options (JAR, WAR, Docker)?
- How do you reduce the startup time of a Spring Boot application?
- What is graceful shutdown and how do you implement it?

### Actuator & Monitoring
- What is Spring Boot Actuator? What endpoints are commonly used?
- How does Spring Boot Actuator help in production monitoring and metrics?
- What are the risks of enabling too many Actuator endpoints?
- How do you secure Actuator endpoints?

### Miscellaneous
- How do you create custom annotations in Spring Boot?
- How do you implement scheduling with `@Scheduled`?
- How do you enable asynchronous method execution with `@Async`?
- Why does `@Async` sometimes not run asynchronously?
- Why does `@Transactional` not work for private methods or self-invocation?
- Why does a Spring Boot app sometimes consume more memory over time?
- How does Spring Boot integrate with cloud services (AWS S3/RDS)?
- What are common Spring Boot performance mistakes?
- How do you handle feature toggles in Spring Boot?
- How do you implement caching in Spring Boot (Redis, Caffeine)?
- Why does `@Cacheable` sometimes not cache?
- How do you create a bean from a third-party JAR?
- How do you implement logging in Spring Boot?
- What is WebClient vs RestTemplate vs FeignClient? When to use each?
- What is reactive programming? What is WebClient?
- What is reactive backpressure strategy?
- How do you manage database migrations with Flyway or Liquibase?
- How do you configure multiple data sources?

---

## Spring Security

- What is Spring Security and why is it used?
- What are the core components of Spring Security (SecurityFilterChain, AuthenticationManager, UserDetailsService, etc.)?
- What is the difference between authentication and authorization?
- What is `SecurityContext` and how is it maintained?
- What is `UserDetailsService` and its role?
- What is `PasswordEncoder`? How does password encoding work?
- What is the difference between hashing and encryption?
- What is hashing and salting in the context of password security?
- How do you configure Spring Security using a component-based approach (post `WebSecurityConfigurerAdapter` deprecation)?
- What is `SecurityFilterChain`?
- How does the Spring Security filter chain work internally?
- What is CSRF protection and why is it needed?
- What is CORS? What is the difference between CORS and CSRF?
- How do you configure CORS in API Gateway?
- What is JWT? How does it work? What information is inside a JWT?
- How does JWT authentication flow work in Spring Boot?
- What is the difference between Access Token and Refresh Token?
- How do you handle JWT expiration?
- Where should JWT tokens be stored?
- How do you validate a JWT token in Spring Boot?
- What is stateless vs stateful authentication?
- What is OAuth 2.0? What are its grant types?
- How does Authorization Code Grant flow work?
- What is the difference between JWT and OAuth2?
- What is OpenID Connect (OIDC) and how does it differ from OAuth 2.0?
- How do you implement role-based access control?
- What is method-level security? How do `@PreAuthorize` and `@PostAuthorize` work?
- How does session management work in Spring Security?
- What is session fixation protection?
- How does Spring Security handle concurrent logins?
- What is `@Lazy` in the context of security beans?
- How do you secure REST APIs in production?

---

## Hibernate / JPA

### Core Fundamentals
- What is JPA and how is it different from Hibernate?
- What is ORM?
- What are the main components of JPA architecture?
- What is an Entity? What makes a class a JPA entity?
- What is the role of `@Entity`, `@Table`, `@Id`, `@Column`?
- What is `@GeneratedValue`? What are the ID generation strategies?
- What is the difference between `AUTO` and `IDENTITY` strategies?
- What is `@Transient`?
- What is `persistence.xml`? Is it still required in Spring Boot?
- What is an `EntityManager`? What is the difference between `EntityManager` and Hibernate `Session`?
- What is a Persistence Context?
- What is the entity lifecycle (Transient, Persistent/Managed, Detached, Removed)?
- What is dirty checking in JPA/Hibernate?
- What is flushing? What is the difference between `flush()` and `commit()`?

### Relationships & Mappings
- What is the difference between `@OneToOne`, `@OneToMany`, `@ManyToOne`, and `@ManyToMany`?
- What is the owning side of a relationship?
- What is the difference between `mappedBy` and `@JoinColumn`?
- How do you map a bidirectional relationship?
- How do you avoid infinite recursion in bidirectional mappings?
- What is cascading? What are the types of cascade operations?
- What is the difference between `CascadeType.ALL` and `orphanRemoval`?
- How do you map composite primary keys?
- What is `@Embeddable` and `@Embedded`?
- How do you map inheritance in JPA? What are the strategies and their trade-offs?

### Fetching & Performance
- What is lazy loading vs eager loading?
- What are the default fetch types for `@OneToMany` and `@ManyToOne`?
- What is `LazyInitializationException` and how do you fix it?
- What is the N+1 select problem? How do you solve it?
- What is the difference between `JOIN` and `JOIN FETCH`?
- What is `@EntityGraph` and when should you use it?
- What is the difference between entity graphs and fetch joins for optimizing queries?
- How do fetch strategies impact performance?
- What is the first-level cache? Is it enabled by default?
- What is the second-level cache? When should you disable it?
- What is the difference between first-level and second-level cache?
- How does second-level caching work and which providers have you used?
- What is query cache?
- What is batch processing/fetching in Hibernate?
- How do you optimize large batch inserts/updates?
- How do you handle large data volumes and result sets efficiently?
- What is fetch size and batch size?

### Queries
- What is HQL/JPQL? How is it different from SQL?
- What is the difference between JPQL and native queries?
- What is the Criteria API? What are its pros and cons?
- What is `@Query` annotation?
- How do derived query methods work in Spring Data JPA?
- How do projections work in JPQL/Spring Data JPA?
- How do you write dynamic queries?
- What is a named query?
- How do you fetch only selected columns?

### Spring Data JPA
- What is Spring Data JPA?
- What is the difference between `CrudRepository`, `JpaRepository`, and `PagingAndSortingRepository`?
- What is the difference between `findById()` and `getReferenceById()` (`getOne()`)?
- What is the difference between `save()`, `persist()`, and `merge()`?
- What is the difference between `save()` and `saveAndFlush()`?
- What is the difference between `update()` and `merge()`?
- What is the difference between `get()` and `load()`?
- What happens internally when you call `save()`?
- What is `@Modifying` and when is it used?
- How do you implement pagination and sorting in JPA?
- How do you write custom repository implementations?
- How does Hibernate generate SQL?

### Transactions & Concurrency
- What is transaction management in JPA?
- What is `@Transactional`? How does it work internally?
- What are transaction propagation types? Explain `REQUIRED` vs `REQUIRES_NEW`.
- What are transaction isolation levels?
- What is optimistic locking? What is `@Version`?
- What is pessimistic locking?
- What is the difference between optimistic and pessimistic locking?
- How do you handle concurrent updates to the same database record?
- How do you handle optimistic locking failures gracefully?
- How do you handle deadlocks in JPA?

### Advanced
- How do you implement auditing (`createdDate`, `updatedDate`)?
- How do you handle soft deletes?
- What is multi-tenancy in Hibernate?
- How do you handle schema evolution and database migrations?
- How do you map JSON columns?
- How do you handle large object fields (LOB)?
- How do you tune SQL queries generated by JPA for performance?
- What are common JPA anti-patterns and mistakes in production?
- When should you NOT use JPA/Hibernate?
- Explain Hibernate architecture.

---

## Microservices

### Architecture & Design
- What are microservices? How do they differ from monolithic architecture?
- What are the key benefits and challenges of microservices?
- When should you NOT use microservices?
- How do you decompose a monolithic application into microservices (DDD, bounded contexts)?
- What is Domain-Driven Design (DDD) and how does it apply to microservices?
- What is a bounded context?
- How do you decide service boundaries and service granularity?
- What issues arise when services are split incorrectly?
- What is the Strangler Fig pattern for migration?
- What is a distributed monolith and why is it bad?
- What is the Single Responsibility Principle in microservices?
- What is the database-per-service pattern? Why is a shared database a bad idea?
- What is polyglot persistence?

### Communication
- How do microservices communicate (synchronous vs asynchronous)?
- When would you use REST vs messaging (Kafka/RabbitMQ) vs gRPC?
- What is the difference between synchronous REST calls and asynchronous messaging?
- What problems arise from synchronous service-to-service calls at scale?
- What is event-driven architecture and when is it useful?
- What is the difference between Feign Client, WebClient, and RestTemplate?
- How do you prevent chatty microservices?

### Resilience & Fault Tolerance
- What is the Circuit Breaker pattern? Explain the three states.
- How does Resilience4j implement Circuit Breaker?
- What is the Bulkhead pattern?
- What is the difference between Retry and Circuit Breaker? When can retries make things worse?
- How do you implement retry with exponential backoff?
- How do you handle timeouts between services?
- What is graceful degradation and fallback mechanisms?
- How do you prevent cascading failures?
- How do you handle partial failures in a microservice chain?
- What happens when a downstream service times out or becomes slow?
- How do you design fault-tolerant microservices?

### Data Consistency & Transactions
- What is eventual consistency and how do you handle it?
- What is the Saga pattern? Explain choreography vs orchestration.
- How do you handle failures in a Saga flow?
- What is the Outbox pattern?
- What is the difference between distributed transactions and eventual consistency?
- What is two-phase commit (2PC)? Why is it often avoided?
- Why are distributed transactions discouraged in microservices?
- What are XA transactions and why are they often avoided?

### Service Discovery & Gateway
- What is service discovery? How do Eureka/Consul/Zookeeper handle it?
- What is the difference between client-side and server-side discovery?
- What is an API Gateway? What is its role?
- What is the difference between an API Gateway and a load balancer?
- What is the difference between an API Gateway and a Service Mesh?
- What is a reverse proxy vs a forward proxy?
- How do you implement authentication, rate limiting, and caching at the gateway?
- What is the role of a load balancer in microservices?

### Configuration & Deployment
- What is Spring Cloud Config Server and centralized configuration?
- How do you handle configuration changes without redeploying?
- How do you manage secrets in microservices (Vault, AWS Secrets Manager)?
- What is blue-green deployment?
- What is canary deployment?
- What is rolling deployment?
- How do you achieve zero-downtime deployments?
- How do you handle rollbacks during production failures?

### Observability & Monitoring
- What is distributed tracing? How do you implement it (Zipkin, Jaeger, Sleuth, OpenTelemetry)?
- How do you implement centralized logging (ELK stack, Splunk)?
- How do you monitor microservices health (Prometheus, Grafana)?
- How do you trace a single request across multiple microservices?
- How do you debug production issues in microservices?
- What observability tools/metrics are commonly used?

### Security
- How do you secure microservices (OAuth2, JWT, Keycloak)?
- How do you secure inter-service communication (mTLS, service mesh)?
- What is zero-trust architecture?
- How do you handle token propagation across services?

### Patterns & Advanced
- What is CQRS pattern?
- What is event sourcing?
- What is idempotency and why is it important in distributed architectures?
- What is API versioning? How do you version APIs without breaking clients?
- How do you handle backward compatibility in APIs?
- What is the sidecar pattern?
- What is a dead-letter queue (DLQ)?
- What are common microservice design patterns (Circuit Breaker, Saga, Bulkhead, Strangler, etc.)?
- What is the difference between exactly-once and at-least-once delivery?
- How do you design stateless services?
- How do you design for high availability?
- How do you handle high traffic spikes?
- How do you scale services independently (horizontal vs vertical scaling)?
- How do you handle multi-region deployment?
- What is chaos engineering?
- What are the risks of sharing libraries between microservices?

---

## SQL / Database

### Fundamentals
- What is the difference between `WHERE` and `HAVING`?
- What is the difference between `INNER JOIN`, `LEFT JOIN`, and `RIGHT JOIN`?
- What is a self join? Where is it used?
- Can a table have multiple unique keys but only one primary key? Why?
- What is a foreign key and how does it improve data integrity?
- What is the difference between `DELETE`, `TRUNCATE`, and `DROP`?
- What is the difference between `CHAR` and `VARCHAR`?
- What is the difference between `UNION` and `UNION ALL`?
- What is the difference between `BETWEEN` and `IN`?
- What is `COALESCE` vs `NVL` vs `IFNULL`?
- What is the difference between `NOT IN` and `NOT EXISTS`?
- How does SQL handle `NULL` in comparisons?
- Why does `GROUP BY` sometimes fail without aggregate functions?
- What is the query execution order in SQL?

### Indexing & Optimization
- What is an index? What are the types (clustered, non-clustered)?
- How does indexing improve query performance? What are the drawbacks?
- What is a composite index? Does column order matter?
- What is a covering index?
- When should you NOT use an index?
- What is an execution plan and how do you read it?
- How do you optimize a slow SQL query?
- How do you find duplicate records in a table?

### Transactions & Locking
- What are ACID properties?
- What is a transaction isolation level? What is the default?
- Explain isolation levels (Read Uncommitted through Serializable).
- What is the difference between dirty read and phantom read?
- What is a database deadlock? How do databases resolve it?
- What is the difference between shared and exclusive locks?
- What is the difference between optimistic and pessimistic locking at the database level?
- What are transaction control commands (COMMIT, ROLLBACK, SAVEPOINT)?

### Advanced
- What is normalization? Why is 3NF preferred?
- What is denormalization and when is it better?
- What is a VIEW? When should you avoid it?
- What is a materialized view?
- What is a stored procedure? When should you use it vs querying from Java?
- What is a trigger? Give a real use case.
- What is referential integrity?
- What is cardinality?
- What is a composite key?
- What is cascade delete/update?
- What is the difference between correlated and non-correlated subqueries?
- What are window functions (`ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`)?
- What is a CTE (WITH clause)?
- What is the difference between OLTP and OLAP databases?
- What is data consistency vs data integrity?
- What is database sharding and data partitioning?
- What is a read replica? What problems does it introduce?
- What is the difference between ACID and BASE consistency models?
- How does pagination work internally with `LIMIT` and `OFFSET`? Why does it become slow for large tables?
- What is SQL injection? How do you prevent it at the database and Java level?
- What is write amplification in databases?
- How do you handle transactions across multiple databases?

### Common SQL Coding Questions
- Write a query to find the 2nd (or Nth) highest salary per department.
- Write a query to fetch records with pagination (e.g., 20 records starting from the 5th row).
- Write an `INNER JOIN` query between two tables.
- Write a query to find the average salary per department.
- Write a query to find duplicate records.

---

## REST APIs

- What is a REST API?
- What is the difference between REST and SOAP?
- What is the difference between `PUT` and `POST`?
- What is the difference between `PUT` and `PATCH`?
- What is idempotency? Is `PUT` idempotent?
- What does a stateless API mean?
- What are HTTP status codes? Explain commonly used ones (200, 201, 204, 301, 400, 401, 403, 404, 429, 500, 503).
- How do you design idempotent REST APIs?
- How do you implement API versioning (URI, header-based)?
- How do you handle backward compatibility in APIs?
- How do you implement rate limiting for APIs?
- What is the difference between token bucket and leaky bucket algorithms?
- How do you implement pagination, sorting, and filtering in REST APIs?
- How do you validate REST API request payloads?
- What is HATEOAS?
- What is the difference between HTTP/2 and HTTP/3?
- What is API throttling?

---

## Kafka & Messaging

- What is Kafka? Explain its architecture (topics, partitions, offsets).
- What is a consumer group in Kafka?
- What is consumer lag in Kafka? How do you resolve it?
- What is the difference between Kafka topics and partitions?
- How do Kafka partitions help with scalability?
- How do Kafka partitions and replication ensure fault tolerance?
- What delivery guarantees does Kafka provide (at-least-once, exactly-once)?
- How do you ensure message ordering in Kafka?
- How do you handle message duplication in Kafka?
- What happens if a consumer crashes after consuming a message?
- What is a dead-letter queue (DLQ) in Kafka?
- How do you handle poison messages?
- What is the difference between Kafka and RabbitMQ?
- How do you integrate Kafka with Spring Boot?
- What factors lead to consumer lag in Kafka?
- What challenges occur during Kafka partition rebalancing?
- How do you handle schema changes without breaking consumers?
- How do you design event schemas?
- What is event versioning strategy?
- In which scenarios should Kafka NOT be used?

---

## System Design

- How do you design a URL shortener?
- How do you design a chat system?
- How do you design an LRU cache?
- How do you design a distributed tracing system?
- How do you design a rate limiter?
- How do you design a cache system with expiration and eviction policies?
- How do you design a scalable payment system?
- How do you design microservices for a retail checkout (inventory, cart, payment)?
- How do you design a system to send alerts to millions in real time?
- What is the CAP theorem?
- What is the difference between horizontal and vertical scaling?
- What are caching strategies? At which level — API, DB, or Gateway?
- What is connection pooling? How does HikariCP work internally?
- What is database sharding?
- What is a read replica?
- What are load balancer types?
- How do you design stateless services?
- How do you design for high availability?
- How do you handle high-throughput APIs in Java?
- How do you identify and fix performance bottlenecks?

---

## DevOps & Cloud

### Docker & Kubernetes
- What is Docker? What is the difference between a Docker image and a container?
- What is the difference between Docker and a VM?
- What is the difference between Docker volumes and bind mounts?
- What is a multi-stage Docker build?
- How do you Dockerize a Spring Boot application?
- What is Kubernetes and why is it used?
- What is the difference between Kubernetes Deployments and StatefulSets?
- What are liveness and readiness probes in Kubernetes?
- How does Kubernetes handle auto-scaling?
- How do rolling updates work in Kubernetes?
- What is the difference between Docker and Kubernetes?

### CI/CD & Deployment
- What is a CI/CD pipeline?
- How do you integrate microservices in a CI/CD pipeline?
- What is blue-green deployment?
- What is canary deployment?
- What is rolling deployment?
- How do you achieve zero-downtime deployments?
- What is the difference between AWS EC2, ECS, EKS, and Lambda?
- What is AWS Lambda? How does it integrate with other services?
- How do you implement logging, tracing, and monitoring in the cloud (CloudWatch, X-Ray, ELK)?
- How do you secure secrets and credentials in the cloud (Parameter Store, Secrets Manager)?
- How do you scale using Auto Scaling Groups vs Kubernetes HPA?

---

## Testing

- What is the difference between unit tests and integration tests?
- What is Test-Driven Development (TDD)?
- What is Mockito? How do you use `@Mock` and `@InjectMocks`?
- How do you write integration tests with Spring Boot (`@SpringBootTest`, `@DataJpaTest`)?
- What is Testcontainers and when do you use it?
- What is contract testing?
- How do you test microservices end-to-end?
- How do you mock dependent services?

---

## Coding / Problem Solving

### Arrays & Strings
- Find the second highest number in an array.
- Find the first non-repeating character in a string.
- Count the frequency of each character in a string.
- Reverse a string without using built-in methods.
- Find the longest substring without repeating characters.
- Check if a number is a palindrome without converting to a String.
- Find the maximum sum subarray (Kadane's Algorithm).
- Find the maximum sum of B consecutive elements in an array (sliding window).
- Find all pairs/triplets whose sum equals a given target.
- Find duplicate elements in a list.
- Detect duplicate words in a string using Stream API.

### Linked Lists
- Reverse a linked list (iteratively and recursively).
- Detect a cycle in a linked list.

### Stacks & Queues
- Sort a stack using another stack.
- Implement a custom Queue in Java.

### Trees
- Check if a binary tree is height-balanced.

### Design & Implementation
- Implement an LRU cache (thread-safe version).
- Implement the Producer-Consumer problem using `wait()` and `notify()`.
- Implement rate limiting for API calls in Java.
- Design a search engine (like Elasticsearch).
- Merge two sorted arrays.
- Binary search implementation.
- Find the top K frequent elements.

### Stream API Coding
- Find the second highest salary from a list of Employee objects using Streams.
- Group employees by department.
- Find the employee with the highest salary in each department.
- Filter top N highest salaries by department using Streams.
- Count occurrences of each word in a sentence using Streams.
- Find the longest string in a list using Stream API.
- Calculate average/min/max salary using Streams.
- Partition a list into even and odd numbers using `Collectors.partitioningBy()`.
- Sum values using `reduce()`.
- Flatten a nested `List<List<String>>` using `flatMap()`.
- Convert `List<Employee>` to `Map<Department, List<Employee>>`.
- Convert a list of objects into a map (e.g., id to name).
- Filter and map combinations.
- Remove null values from a list using Streams.
- Sort a list of custom objects by multiple fields.
- Remove duplicates using `distinct()`.
- Convert a list of strings to uppercase using `map()`.
- Reverse elements using Streams.
- Sort a `HashMap` by key and by value.
- Print index positions of all vowels in a string using Streams.

### SQL Coding
- Write a query to find the 2nd (or Nth) highest salary.
- Write a query to find the 3rd highest salary without using `LIMIT`.
- Write a query to find the average salary per department.
- Write a query to fetch paginated records.
- Write an `INNER JOIN` query between two tables.

---

## Scenario-Based / Behavioral

### Production Debugging
- Your Spring Boot app works locally but fails after deployment. What do you check first?
- APIs are fast locally but slow in production. How do you debug?
- You updated `application.properties` but changes didn't reflect. Why?
- CPU usage is low but requests are timing out. What could be blocking?
- Application memory usage keeps increasing over time. What do you suspect?
- `@Transactional` is present but rollback doesn't happen. Why?
- Database connection pool gets exhausted under load. How do you identify and fix it?
- A downstream service is slow and your service also starts failing. How do you protect it?
- Scheduled background jobs start affecting API latency. How do you isolate them?
- Application behaves differently in Docker vs locally. Why?
- Logs are missing in production but fine locally. Where do you look?
- A REST API returns correct data but response time is inconsistent. Why?
- Service restarts automatically without clear errors. What could trigger this?
- Scaling instances didn't improve performance. Why?
- Lazy loading works locally but fails in production. What's missing?
- Circuit breaker is open even when the service is healthy. What's wrong?
- A cache improves performance initially, then degrades it. Why?
- After enabling parallel streams, response time increased. Why?
- Excessive logging was added for debugging and production went down. How?
- A retry mechanism caused system overload. What design mistake?

### Behavioral
- Tell me about yourself and your project/role.
- Describe a microservices project you architected. What were the trade-offs?
- Walk through a production issue and how you debugged it.
- Describe a situation where you showed leadership.
- How do you handle conflicts in a team?
- How do you prioritize tasks with tight deadlines?
- How do you mentor junior developers?
- Describe a time you disagreed with your manager.
- Describe a time when you improved a process or system.
- Why are you leaving your current company?
- Why do you want to join this company?
