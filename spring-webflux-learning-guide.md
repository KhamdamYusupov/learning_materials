# Spring WebFlux: Complete Production Guide for Java/Spring Boot Engineers

> For backend developers strong in imperative Spring MVC transitioning to reactive systems.

---

## Table of Contents

1. [Reactive Programming Fundamentals](#1-reactive-programming-fundamentals)
2. [Spring WebFlux Overview](#2-spring-webflux-overview)
3. [Project Reactor Core Concepts](#3-project-reactor-core-concepts)
4. [Mono — Complete Method Guide](#4-mono--complete-method-guide)
5. [Flux — Complete Method Guide](#5-flux--complete-method-guide)
6. [WebClient — Complete Guide](#6-webclient--complete-guide)
7. [Functional Endpoints](#7-functional-endpoints-serverresponse)
8. [Schedulers & Threading](#8-schedulers--threading)
9. [WebFlux in System Design](#9-webflux-in-system-design)
10. [Production Best Practices](#10-production-best-practices)
11. [Real-World Scenarios](#11-real-world-scenarios)
12. [Comparisons & Trade-offs](#12-comparisons--trade-offs)
13. [Production-Ready Checklist](#13-production-ready-checklist)

---

# 1. Reactive Programming Fundamentals

## What Reactive Programming Is

Reactive programming is a programming paradigm built around **data streams and the propagation of change**. Instead of asking "give me the data now" (pull model), reactive systems say "notify me when data is available" (push model).

The core shift: **you describe a pipeline of transformations, and the system executes them when data flows through**.

```java
// Imperative: pull model
User user = userRepository.findById(id);   // Block until DB responds
String email = user.getEmail();            // Then do the next thing

// Reactive: push model — describe the pipeline
Mono<String> email = userRepository.findById(id)  // Describe: when found...
    .map(User::getEmail);                          // ...then transform
// Nothing executes yet. Only on subscribe().
```

## Why It Exists: The Problem with Blocking I/O

### The Traditional Blocking Model

In Spring MVC, each HTTP request occupies a thread for its entire duration:

```
Thread 1: [accept request]──[DB query 50ms wait]──[respond]
Thread 2: [accept request]──[HTTP call 100ms wait]──[respond]
Thread 3: [accept request]──[DB query 50ms wait]──[respond]
...
Thread N: [accept request]──[waiting...]─────────────────────
```

During "waiting" the thread is **parked** — it consumes ~1MB of stack memory but does zero useful work. The CPU is idle.

With a default Tomcat pool of 200 threads:
- Each request that waits 100ms for I/O ties up a thread
- At 2000 concurrent users, 1800 requests queue up waiting for a free thread
- Latency explodes, not because the CPU is busy, but because threads are all asleep

**The math**: 200 threads × 100ms I/O wait = 2000 req/sec theoretical max. But with reactive and async I/O: 1 thread can handle thousands of concurrent I/O-bound requests.

### What Non-Blocking I/O Changes

Non-blocking I/O (NIO) means: "start this I/O operation and call me back when it's done." The calling thread is **not blocked** — it returns immediately and can handle other work.

```
Event Loop Thread:
├── Accept request 1 → start DB query (non-blocking) → move on
├── Accept request 2 → start HTTP call (non-blocking) → move on
├── Accept request 3 → start DB query (non-blocking) → move on
│
├── DB result for request 1 arrives → complete request 1 pipeline
├── HTTP result for request 2 arrives → complete request 2 pipeline
└── ...
```

One thread handles thousands of concurrent requests. The trade-off: **you must never block this thread**.

## Synchronous vs Asynchronous vs Non-Blocking

| Model | Thread behavior | Example |
|---|---|---|
| **Synchronous blocking** | Thread waits for result | `JDBC`, `RestTemplate` |
| **Synchronous non-blocking** | Thread polls for result | Busy-wait loop (bad) |
| **Asynchronous blocking** | Separate thread waits | `ExecutorService.submit()` |
| **Asynchronous non-blocking** | Thread registered for callback | `WebClient`, `R2DBC` |

WebFlux uses **asynchronous non-blocking**: register a callback, thread is freed, callback fires when data arrives.

## Event Loop Model

Netty (the default server under WebFlux) uses an event loop architecture:

```
                     ┌─────────────────────────────────────────┐
                     │           Netty Event Loop               │
                     │                                         │
Incoming             │  ┌──────────────────────────────────┐   │
Connections ─────────┼──► Boss Group (1-2 threads)         │   │
                     │  │  Accepts connections              │   │
                     │  └──────────────┬───────────────────┘   │
                     │                 │ register channel        │
                     │  ┌──────────────▼───────────────────┐   │
                     │  │ Worker Group (N threads = CPUs×2) │   │
                     │  │  • Reads data from socket         │   │
                     │  │  • Executes pipeline              │   │
                     │  │  • Writes response                │   │
                     │  │  • Handles timer events           │   │
                     │  └──────────────────────────────────┘   │
                     └─────────────────────────────────────────┘
```

Each worker thread runs an **event loop** — a tight loop that:
1. Checks for new I/O events (incoming data, connection ready)
2. Dispatches events to registered handlers
3. Processes callbacks from completed operations
4. Loops back

**Critical rule**: Never block an event loop thread. A single blocking call (like `Thread.sleep()` or JDBC) freezes the loop and starves all other connections on that thread.

## Backpressure

Backpressure is the mechanism by which a **slow consumer can signal to a fast producer to slow down**. Without it, the producer overwhelms the consumer's memory.

**The problem without backpressure**:
```
Producer (1M events/sec) ──► Consumer (1K events/sec)
                                  ↑
                             Buffer fills up
                             OutOfMemoryError
```

**The Reactive Streams solution**:
```
Consumer: "I can handle 10 items"        → request(10)
Producer: sends 10 items
Consumer: processes 10 items
Consumer: "Ready for 10 more"            → request(10)
Producer: sends 10 items
...
```

The subscriber controls the flow via `request(n)`. Operators like `buffer`, `onBackpressureDrop`, and `onBackpressureBuffer` provide strategies for when the producer is faster than the consumer.

```java
// Explicit backpressure strategy
Flux.range(1, 1_000_000)
    .onBackpressureBuffer(1000)          // Buffer up to 1000 items
    .onBackpressureDrop(item ->          // Drop and log if buffer full
        log.warn("Dropped: {}", item))
    .subscribe(this::processItem);
```

## Publisher / Subscriber Model

The Reactive Streams specification (adopted by Java 9 as `java.util.concurrent.Flow`) defines four interfaces:

```java
// Publisher: produces data
public interface Publisher<T> {
    void subscribe(Subscriber<? super T> subscriber);
}

// Subscriber: consumes data
public interface Subscriber<T> {
    void onSubscribe(Subscription subscription);  // Called first, always
    void onNext(T item);                          // Each item
    void onError(Throwable throwable);            // Terminal: error
    void onComplete();                            // Terminal: done
}

// Subscription: controls the flow
public interface Subscription {
    void request(long n);    // Ask for n items (backpressure)
    void cancel();           // Stop receiving
}

// Processor: both publisher and subscriber
public interface Processor<T, R> extends Subscriber<T>, Publisher<R> {}
```

**The contract**:
1. `subscribe()` → `onSubscribe()` is called with a `Subscription`
2. `Subscription.request(n)` → up to n `onNext()` calls
3. `onError()` or `onComplete()` terminates the stream (mutually exclusive)
4. After terminal signal, no more signals are sent

## Reactive Streams Specification

The Reactive Streams spec (reactive-streams.org) defines a standard for async stream processing with non-blocking backpressure. Key points:

- **TCK (Technology Compatibility Kit)**: Test suite to verify implementations comply
- **Implementations**: Project Reactor (`Flux`/`Mono`), RxJava (`Observable`/`Flowable`), Akka Streams
- **Java 9 adoption**: `java.util.concurrent.Flow` mirrors the spec interfaces exactly
- **Interoperability**: Reactor types can be converted to/from RxJava, Java 9 Flow types

---

# 2. Spring WebFlux Overview

## What Spring WebFlux Is

Spring WebFlux is Spring's reactive web framework, introduced in Spring 5. It provides:

- **Reactive HTTP layer**: Non-blocking request handling built on Project Reactor
- **Two programming models**: Annotation-based (similar to MVC) and functional (RouterFunction/HandlerFunction)
- **Reactive server adapters**: Netty (default), Undertow, Tomcat (with async servlet support)
- **Reactive client**: `WebClient` — non-blocking HTTP client
- **Reactive test support**: `WebTestClient`

It is **not** a replacement for Spring MVC in all cases. It's a different tool for different problems.

## When to Use vs NOT Use WebFlux

### Use WebFlux When:
- **High concurrency with I/O-bound workloads**: Many concurrent requests where most time is spent waiting for DB/HTTP/cache responses
- **Streaming responses**: Server-Sent Events, real-time data feeds, large file streaming
- **Event-driven architectures**: Processing Kafka/RabbitMQ streams reactively
- **API gateway / aggregator**: Calling multiple upstream services concurrently and merging results
- **Microservices calling microservices**: Long chains of service calls where non-blocking is crucial
- **WebSockets**: Long-lived connections at scale

### Do NOT Use WebFlux When:
- **CPU-bound workloads**: Reactive brings no benefit if your bottleneck is computation, not I/O
- **Blocking dependencies you can't replace**: If you must use JDBC, Hibernate, or a blocking SDK and can't switch to R2DBC/reactive alternatives, WebFlux's benefits are negated (you'll still need to offload to threads)
- **Simple CRUD apps with low concurrency**: Spring MVC is simpler, easier to debug, and performs identically at low concurrency
- **Team unfamiliar with reactive**: The mental model shift is significant. Reactive bugs (missed subscriptions, thread context loss, accidental blocking) are hard to debug
- **Existing MVC app**: Migrating a large MVC codebase to WebFlux is a massive undertaking rarely justified by performance alone

## WebFlux vs Spring MVC

| Aspect | Spring MVC | Spring WebFlux |
|---|---|---|
| Thread model | Thread-per-request | Event loop + small thread pool |
| I/O model | Blocking | Non-blocking |
| Scalability | ~200 concurrent (default pool) | Thousands concurrent |
| Programming model | Imperative, sequential | Declarative pipelines |
| Debugging | Stack traces are clear | Stack traces are complex (async) |
| Ecosystem | Mature, wide support | Growing, some gaps |
| Blocking libraries | Fine | Fatal to performance |
| CPU-bound tasks | Fine | No advantage |
| Learning curve | Low | High |
| DB layer | JPA/JDBC | R2DBC (reactive), MongoDB reactive |

**The nuanced truth**: Spring MVC with async servlet support (`DeferredResult`, `Callable`) can achieve similar I/O concurrency benefits in many cases. The real advantage of WebFlux is end-to-end non-blocking — from HTTP accept to DB query to HTTP response — with backpressure.

## Netty vs Servlet Model

### Servlet Model (Spring MVC on Tomcat)
```
Request arrives → assign a thread from pool → thread executes entire request → return thread to pool
```
- Thread-per-request: simple mental model
- 1MB+ stack memory per thread
- Context switches when many threads are active
- Max concurrency = pool size

### Netty Model (Spring WebFlux)
```
Request arrives → event loop thread reads data → register callbacks → thread handles next request
              → when DB responds → event loop thread continues pipeline → sends response
```
- Few threads, many concurrent operations
- No context switch overhead between requests
- Max concurrency = limited by memory (futures/callbacks), not threads

## How WebFlux Handles Requests Internally

```
1. Netty accept thread: accept TCP connection

2. Worker thread picks up the connection:
   a. Reads HTTP request bytes
   b. Decodes into ServerHttpRequest
   c. Looks up handler (DispatcherHandler)

3. Handler method executes:
   - Returns Mono<ServerResponse> or Mono<ResponseEntity<T>>
   - This is just a description — not yet executed

4. The Mono pipeline is subscribed to:
   - Each operator in the pipeline registers a callback
   - Non-blocking I/O operations (R2DBC query, WebClient call) register with Netty

5. Worker thread is RELEASED — it handles other requests

6. When I/O completes (DB responds, HTTP response arrives):
   - An event arrives on the event loop
   - Worker thread picks up, continues the Mono pipeline
   - Sends HTTP response

7. Connection closed or kept alive (HTTP/2)
```

The key insight: steps 1–4 and 6–7 are fast (microseconds). The waiting (step 5) happens with no thread held. This is the efficiency gain.

---

# 3. Project Reactor Core Concepts

## Mono vs Flux

```java
// Mono<T>: 0 or 1 item, then onComplete or onError
Mono<User> user = userRepository.findById(userId);  // 0 or 1 user
Mono<Void> delete = userRepository.deleteById(id);  // 0 items (just completion)

// Flux<T>: 0 to N items, then onComplete or onError
Flux<User> users = userRepository.findAll();         // 0 to many users
Flux<String> stream = Flux.interval(Duration.ofSeconds(1)).map(i -> "event-" + i); // infinite
```

**Mono is NOT just "Flux with one element."** Mono has different operators optimized for the 0-or-1 case. `Mono.zipWith` combines two Monos into a tuple. `Flux.zipWith` zips element-by-element.

## Cold vs Hot Publishers

### Cold Publisher
Every subscriber gets its **own independent execution** of the pipeline. Like a DVD — each viewer gets their own private copy.

```java
Mono<User> cold = Mono.fromCallable(() -> {
    System.out.println("Querying DB...");
    return userRepository.findById(1L);
});

cold.subscribe(u -> System.out.println("Subscriber 1: " + u));
cold.subscribe(u -> System.out.println("Subscriber 2: " + u));
// Prints "Querying DB..." TWICE — two independent DB queries
```

**Most Reactor operators produce cold publishers**. This is the default.

### Hot Publisher
Produces data regardless of subscribers. Subscribers join an **existing stream**. Like a TV broadcast — you see only what's airing now.

```java
// Sinks are hot: data published to them is not replayed for late subscribers
Sinks.Many<String> sink = Sinks.many().multicast().onBackpressureBuffer();
Flux<String> hot = sink.asFlux();

hot.subscribe(s -> System.out.println("Sub1: " + s));
sink.tryEmitNext("A");  // Sub1 gets "A"
sink.tryEmitNext("B");  // Sub1 gets "B"

hot.subscribe(s -> System.out.println("Sub2: " + s));  // Sub2 joins late
sink.tryEmitNext("C");  // Both Sub1 and Sub2 get "C"
// Sub2 never receives "A" or "B"
```

**When to use hot publishers**: Server-Sent Events (broadcast to multiple clients), Kafka consumer (events happen once), WebSocket broadcasts.

## Lazy Execution

**Nothing executes until you subscribe.** This is the most important concept for developers coming from imperative code.

```java
// This does NOTHING — just builds a description
Mono<String> pipeline = Mono.fromCallable(() -> {
        System.out.println("This never prints");
        return expensiveComputation();
    })
    .map(String::toUpperCase)
    .doOnNext(s -> System.out.println("Got: " + s));

// Subscribe triggers execution
pipeline.subscribe();  // NOW it executes
```

**The practical implication**: In Spring WebFlux, your controller method returning `Mono<T>` means "here is the recipe." WebFlux subscribes to it when handling the request. If you call `.block()` inside a controller, you're blocking the event loop — catastrophic.

## Subscription Lifecycle

```
Publisher.subscribe(Subscriber)
    │
    ▼
Subscriber.onSubscribe(Subscription)   ← Subscriber receives flow control handle
    │
    ▼
Subscription.request(Long.MAX_VALUE)   ← Subscriber asks for items (unbounded by default)
    │
    ▼
Subscriber.onNext(item1)               ← Items flow
Subscriber.onNext(item2)
...
    │
    ▼
Subscriber.onComplete()                ← OR Subscriber.onError(t) — mutually exclusive
```

After `onComplete()` or `onError()`, no more signals. The subscription is terminated.

**`Long.MAX_VALUE`**: Passing this to `request()` effectively disables backpressure — request all items as fast as possible. Most `subscribe()` overloads do this internally. For true backpressure, implement `BaseSubscriber` and control `request(n)` manually.

## Signal Types

Every Reactor pipeline deals with four signal types:

| Signal | When | Carries |
|---|---|---|
| `onSubscribe` | Once at start | `Subscription` object |
| `onNext` | For each item | The item `T` |
| `onError` | Terminal: failure | `Throwable` |
| `onComplete` | Terminal: success | Nothing |

Operators like `doOnNext`, `doOnError`, `doOnComplete` let you **observe** signals without transforming them (for logging, metrics, side effects). Operators like `map`, `flatMap` **transform** signals.

---

# 4. Mono — Complete Method Guide

## `Mono.just`

**What it does**: Creates a Mono that immediately emits a single given item and completes.

**Signature**: `static <T> Mono<T> just(T data)`

```java
Mono<String> mono = Mono.just("Hello");
mono.subscribe(System.out::println);  // prints: Hello
```

**Real-world use case**: Returning a known value from a cache hit, wrapping an already-resolved value in a reactive pipeline.

```java
public Mono<UserDto> getUser(Long id) {
    UserDto cached = localCache.get(id);
    if (cached != null) {
        return Mono.just(cached);  // Already have it, wrap it
    }
    return userRepository.findById(id).map(this::toDto);
}
```

**Execution**: The value is captured at creation time. Every subscriber receives the same already-captured value.

**Pitfall**: `Mono.just(expensiveMethod())` evaluates `expensiveMethod()` **immediately**, not lazily. Use `Mono.fromCallable()` for lazy evaluation.

```java
// WRONG: calls expensiveMethod() right now, even if nobody subscribes
Mono<Result> eager = Mono.just(expensiveMethod());

// CORRECT: only calls expensiveMethod() when subscribed
Mono<Result> lazy = Mono.fromCallable(() -> expensiveMethod());
```

**When NOT to use**: When the value computation is expensive or has side effects — use `fromCallable` or `defer` instead.

---

## `Mono.empty`

**What it does**: Creates a Mono that completes immediately without emitting any item. Represents "no result, and that's OK."

**Signature**: `static <T> Mono<T> empty()`

```java
Mono<String> empty = Mono.empty();
empty.subscribe(
    item -> System.out.println("Got: " + item),   // never called
    err  -> System.out.println("Error: " + err),  // never called
    ()   -> System.out.println("Completed!")       // called
);
// prints: Completed!
```

**Real-world use case**: Returning from a delete operation (no value to return), or from a method that may find nothing.

```java
// Repository returning empty if not found
public Mono<User> findById(Long id) {
    User user = cache.get(id);
    return user != null ? Mono.just(user) : Mono.empty();
}

// Caller handles empty with switchIfEmpty
findById(999L)
    .switchIfEmpty(Mono.error(new UserNotFoundException(999L)))
    .subscribe(...);
```

**Pitfall**: Returning `Mono.empty()` when the caller expects a value and doesn't handle empty will silently do nothing. Always chain `switchIfEmpty` or `defaultIfEmpty` at consumption points.

---

## `Mono.error`

**What it does**: Creates a Mono that immediately signals an error without emitting any item.

**Signature**: `static <T> Mono<T> error(Throwable error)`

```java
Mono<User> error = Mono.error(new UserNotFoundException("User not found"));
error.subscribe(
    item -> System.out.println("Got: " + item),          // never called
    err  -> System.out.println("Error: " + err.getMessage())  // called
);
```

**Real-world use case**: Translating empty results into domain exceptions, validation errors in pipelines.

```java
public Mono<Order> getOrder(Long orderId, String userId) {
    return orderRepository.findById(orderId)
        .switchIfEmpty(Mono.error(new OrderNotFoundException(orderId)))
        .flatMap(order -> {
            if (!order.getUserId().equals(userId)) {
                return Mono.error(new AccessDeniedException("Not your order"));
            }
            return Mono.just(order);
        });
}
```

**Execution**: Error is created lazily only when subscribed (in the `Mono.error(Throwable)` form). For lazy error creation: `Mono.error(() -> new RuntimeException("message"))`.

**Pitfall**: `Mono.error(new RuntimeException())` creates the exception (and its stack trace) **immediately**. Prefer the `Supplier<Throwable>` overload in hot paths: `Mono.error(() -> new RuntimeException())`.

---

## `Mono.fromCallable`

**What it does**: Creates a Mono from a `Callable` (a supplier that can throw). The callable is executed **lazily** — only when a subscriber subscribes.

**Signature**: `static <T> Mono<T> fromCallable(Callable<? extends T> supplier)`

```java
Mono<User> mono = Mono.fromCallable(() -> {
    // This executes only on subscribe
    return jdbcTemplate.queryForObject("SELECT * FROM users WHERE id=?", User.class, 1L);
});
```

**Real-world use case**: Wrapping a blocking call to be executed on a bounded thread pool. This is the **correct way to integrate blocking I/O** with WebFlux.

```java
public Mono<User> findUserBlocking(Long id) {
    return Mono.fromCallable(() -> jdbcTemplate.findById(id))  // blocking
        .subscribeOn(Schedulers.boundedElastic());              // run on thread pool, not event loop
}
```

**Execution**: Each subscriber triggers a new execution of the callable. It's a cold publisher.

**Pitfall**: If you use `fromCallable` without `subscribeOn(Schedulers.boundedElastic())`, the callable runs on the event loop thread — blocking it. **Always pair `fromCallable` with `subscribeOn(Schedulers.boundedElastic())`** for blocking operations.

---

## `Mono.defer`

**What it does**: Creates a Mono factory — the actual Mono is created fresh **per subscription**. Used when the Mono itself needs to be created lazily (not just the value).

**Signature**: `static <T> Mono<T> defer(Supplier<? extends Mono<? extends T>> supplier)`

```java
// Without defer: timestamp captured once at creation
Mono<Long> eager = Mono.just(System.currentTimeMillis());

// With defer: timestamp captured fresh per subscription
Mono<Long> lazy = Mono.defer(() -> Mono.just(System.currentTimeMillis()));
```

**Real-world use case**: Creating Monos that depend on state evaluated at subscription time (not creation time), or retrying with fresh state.

```java
// Retry with fresh token — without defer, the old token would be reused
public Mono<Response> callApi() {
    return Mono.defer(() -> {
        String freshToken = tokenCache.getCurrentToken();  // evaluated fresh each time
        return webClient.get()
            .uri("/api/data")
            .header("Authorization", "Bearer " + freshToken)
            .retrieve()
            .bodyToMono(Response.class);
    }).retry(3);  // each retry gets a fresh token
}
```

**`defer` vs `fromCallable`**:
- `fromCallable`: wraps a blocking value computation
- `defer`: wraps the creation of a Mono (for Mono-returning operations that need lazy initialization)

**Pitfall**: Not using `defer` when your Mono creation has side effects (timestamp, random, state-dependent headers). Without defer, the side effect happens at assembly time, not subscription time.

---

## `Mono.map`

**What it does**: Transforms the item emitted by a Mono using a **synchronous, non-blocking** function. 1-to-1 transformation.

**Signature**: `<R> Mono<R> map(Function<? super T, ? extends R> mapper)`

```java
Mono<User> user = findUser(1L);
Mono<String> email = user.map(User::getEmail);
Mono<UserDto> dto = user.map(u -> new UserDto(u.getId(), u.getName(), u.getEmail()));
```

**Real-world use case**: Converting domain objects to DTOs, extracting fields, applying transformations.

```java
return userRepository.findById(id)
    .map(user -> UserDto.builder()
        .id(user.getId())
        .name(user.getName())
        .email(user.getEmail())
        .build());
```

**Execution**: The mapper function runs synchronously on the thread that emits the item. It must be fast and non-blocking.

**`map` vs `flatMap`**:
- `map`: mapper returns `R` (plain value) — use for synchronous transformations
- `flatMap`: mapper returns `Mono<R>` — use when the transformation is async (DB call, HTTP call)

**Pitfall**: Doing blocking work in `map` — calling a REST service, doing a DB query. Use `flatMap` for async operations.

```java
// WRONG: blocking HTTP call inside map
.map(id -> restTemplate.getForObject("/users/" + id, User.class))  // blocks event loop

// CORRECT: async operation in flatMap
.flatMap(id -> webClient.get().uri("/users/" + id).retrieve().bodyToMono(User.class))
```

---

## `Mono.flatMap`

**What it does**: Transforms the item by applying a function that returns a **Mono** (async operation), then flattens the result. The most important Mono operator.

**Signature**: `<R> Mono<R> flatMap(Function<? super T, ? extends Mono<? extends R>> transformer)`

```java
Mono<Order> order = userRepository.findById(userId)     // Mono<User>
    .flatMap(user -> orderRepository.findLatest(user));  // User → Mono<Order>
                                                         // Result: Mono<Order>
```

**Real-world use case**: Chaining async operations — find user, then find their orders, then fetch order details.

```java
public Mono<OrderDetailsDto> getOrderDetails(Long orderId) {
    return orderRepository.findById(orderId)
        .flatMap(order ->
            Mono.zip(
                productRepository.findById(order.getProductId()),  // parallel
                userRepository.findById(order.getUserId())          // parallel
            ).map(tuple -> OrderDetailsDto.of(order, tuple.getT1(), tuple.getT2()))
        );
}
```

**Execution**: The outer Mono completes → the transformer function runs → produces an inner Mono → that inner Mono is subscribed to → its item flows downstream. The outer and inner executions are sequential.

**Pitfall 1**: Nesting `flatMap` deeply creates callback hell (the same problem callbacks had before Promises). Use method chaining, not nesting.

```java
// BAD: callback hell
return mono1.flatMap(a ->
    mono2.flatMap(b ->
        mono3.flatMap(c ->
            Mono.just(combine(a, b, c)))));

// GOOD: use Mono.zip for parallel, chain for sequential
return Mono.zip(mono1, mono2, mono3)
    .map(tuple -> combine(tuple.getT1(), tuple.getT2(), tuple.getT3()));
```

**Pitfall 2**: Returning `null` from flatMap — prohibited. Return `Mono.empty()` instead.

---

## `Mono.zipWith`

**What it does**: Combines this Mono with another Mono. Waits for **both** to complete, then combines their results using a combinator function.

**Signature**: `<T2, O> Mono<O> zipWith(Mono<? extends T2> other, BiFunction<? super T, ? super T2, ? extends O> combinator)`

```java
Mono<String> name = Mono.just("Alice");
Mono<Integer> age = Mono.just(30);

Mono<String> combined = name.zipWith(age, (n, a) -> n + " is " + a + " years old");
// "Alice is 30 years old"
```

**Real-world use case**: Fetching two independent pieces of data concurrently and combining them.

```java
public Mono<UserProfileDto> getUserProfile(Long userId) {
    Mono<User> user = userRepository.findById(userId);
    Mono<List<Order>> orders = orderRepository.findByUserId(userId).collectList();
    
    return user.zipWith(orders, (u, o) -> 
        new UserProfileDto(u.getName(), u.getEmail(), o.size())
    );
    // Both queries execute in PARALLEL (subscribed simultaneously)
}
```

**Execution**: Both Monos are subscribed to at the same time (in parallel). When BOTH complete, the combinator runs. If either errors, the result errors immediately.

**`zipWith` vs `flatMap` for combining**:
- `zipWith`: both run in **parallel** — use when they're independent
- `flatMap`: sequential — use when the second depends on the first

**Pitfall**: If one Mono is empty, `zipWith` produces `Mono.empty()`. Use `Mono.zip` with defaults if you want to handle the empty case.

---

## `Mono.switchIfEmpty`

**What it does**: Falls back to another Mono if the upstream Mono completes empty (no item emitted).

**Signature**: `Mono<T> switchIfEmpty(Mono<? extends T> alternate)`

```java
Mono<User> result = userRepository.findById(id)
    .switchIfEmpty(Mono.error(new UserNotFoundException(id)));
```

**Real-world use case**: Converting "not found" (empty Mono) into a domain exception, or providing a fallback value.

```java
public Mono<User> getOrCreate(String email) {
    return userRepository.findByEmail(email)
        .switchIfEmpty(Mono.defer(() -> {
            User newUser = User.builder().email(email).build();
            return userRepository.save(newUser);  // create if not found
        }));
}
```

**Execution**: The upstream Mono is subscribed to. If it completes without emitting an item (`onComplete` without any `onNext`), the alternate Mono is subscribed to instead.

**`switchIfEmpty` vs `defaultIfEmpty`**:
- `switchIfEmpty`: fallback is another Mono (can be async, can error)
- `defaultIfEmpty`: fallback is a plain value (synchronous, never errors)

**Pitfall**: The alternate Mono is subscribed to **eagerly** — it's created when `switchIfEmpty` is called. To avoid eager evaluation of the fallback, wrap it in `Mono.defer`:

```java
// WRONG: fallback DB query runs immediately even if it won't be needed
.switchIfEmpty(userRepository.createDefault(email))  // executes now

// CORRECT: deferred — only runs if upstream is empty
.switchIfEmpty(Mono.defer(() -> userRepository.createDefault(email)))
```

---

## `Mono.defaultIfEmpty`

**What it does**: Emits a default value if the upstream Mono is empty.

**Signature**: `Mono<T> defaultIfEmpty(T defaultValue)`

```java
Mono<String> result = Mono.<String>empty()
    .defaultIfEmpty("No value found");
// emits: "No value found"
```

**Real-world use case**: Providing fallback values for optional lookups.

```java
public Mono<ConfigValue> getConfig(String key) {
    return configRepository.findByKey(key)
        .defaultIfEmpty(ConfigValue.defaultFor(key));  // synchronous default
}
```

**Execution**: Synchronous — the default value is evaluated at assembly time (when `defaultIfEmpty` is called), not subscription time. For lazy defaults, use `switchIfEmpty(Mono.fromCallable(...))`.

**Pitfall**: Creating expensive objects as defaults (the default is constructed at assembly time):
```java
// WRONG: creates BigObject immediately, even if result is non-empty
.defaultIfEmpty(new BigObject(expensiveInit()))

// CORRECT for expensive defaults: use switchIfEmpty + defer
.switchIfEmpty(Mono.defer(() -> Mono.just(new BigObject(expensiveInit()))))
```

---

## `Mono.doOnNext`

**What it does**: Registers a side-effect callback that runs when the Mono emits an item. Does not modify the item. Returns the same Mono.

**Signature**: `Mono<T> doOnNext(Consumer<? super T> onNext)`

```java
return userRepository.findById(id)
    .doOnNext(user -> log.info("Found user: {}", user.getId()))
    .doOnNext(user -> metricsService.incrementCacheHit())
    .map(this::toDto);
```

**Real-world use case**: Logging, metrics, cache population, audit trails — all without changing the data flow.

```java
public Mono<User> findUser(Long id) {
    return cache.get(id)
        .doOnNext(u -> metrics.record("cache.hit"))
        .switchIfEmpty(
            userRepository.findById(id)
                .doOnNext(u -> cache.put(id, u))  // populate cache on miss
                .doOnNext(u -> metrics.record("cache.miss"))
        );
}
```

**Execution**: Runs synchronously on the thread that emits the item, between the upstream signal and the downstream. Must be fast.

**Pitfall**: Blocking operations in `doOnNext` — logging frameworks that do synchronous file I/O in hot paths can be a problem. Use async logging appenders.

**When NOT to use**: When you need to modify the item (use `map`), handle errors (use `doOnError`), or do async operations as part of the pipeline logic (use `flatMap`).

---

## `Mono.doOnError`

**What it does**: Registers a side-effect callback for error signals. Does not change the error — it still propagates downstream.

**Signature**: `Mono<T> doOnError(Consumer<? super Throwable> onError)`

```java
return userRepository.findById(id)
    .doOnError(e -> log.error("Failed to find user {}: {}", id, e.getMessage()))
    .doOnError(DatabaseException.class, e -> metrics.increment("db.errors"))
    .map(this::toDto);
```

**Real-world use case**: Logging errors, recording metrics on failures, alerting.

```java
return paymentService.process(request)
    .doOnError(PaymentDeclinedException.class, 
        e -> auditLog.recordDecline(request.getOrderId(), e.getReason()))
    .doOnError(TimeoutException.class,
        e -> alerting.trigger("Payment service timeout", request.getOrderId()));
```

**Execution**: Called synchronously when the error signal arrives. Error still propagates — `doOnError` is for observation, not handling.

**`doOnError` vs `onErrorResume`**:
- `doOnError`: observe the error, does not handle it — still propagates
- `onErrorResume`: handle the error, replace it with a fallback Mono

**Pitfall**: Throwing from `doOnError` replaces the original exception — confusing and not recommended.

---

## `Mono.onErrorReturn`

**What it does**: Provides a fallback value when the Mono errors. The error is consumed; the fallback value is emitted instead.

**Signature**: `Mono<T> onErrorReturn(T fallbackValue)`

Also: `Mono<T> onErrorReturn(Class<E> type, T fallbackValue)` — only catches specific error types.

```java
return userRepository.findById(id)
    .onErrorReturn(new User("anonymous"));  // return anonymous user on any error
```

**Real-world use case**: Providing degraded but functional responses when dependencies fail.

```java
public Mono<PricingDto> getPrice(Long productId) {
    return pricingService.getPrice(productId)
        .onErrorReturn(TimeoutException.class, PricingDto.defaultPrice())  // timeout → default
        .onErrorReturn(ServiceUnavailableException.class, PricingDto.cachedPrice(productId));
}
```

**Execution**: When error signal arrives, the fallback value is emitted and `onComplete` follows. The pipeline continues as if nothing went wrong.

**`onErrorReturn` vs `onErrorResume`**:
- `onErrorReturn`: fallback is a **value** (synchronous, pre-determined)
- `onErrorResume`: fallback is a **Mono** (can be async, can do a DB lookup, etc.)

**Pitfall**: Using `onErrorReturn` when you need async fallback logic (use `onErrorResume`). Also, catching too broadly — not specifying an error type catches everything, including `NullPointerException` and programming errors that should propagate.

---

## `Mono.onErrorResume`

**What it does**: Handles an error by providing an alternative Mono as a fallback. The most flexible error handling operator.

**Signature**: `Mono<T> onErrorResume(Function<? super Throwable, ? extends Mono<? extends T>> fallback)`

```java
return userRepository.findById(id)
    .onErrorResume(DatabaseException.class, e -> 
        backupRepository.findById(id)  // fallback to backup DB
    )
    .onErrorResume(e -> Mono.just(User.anonymous()));  // catch-all fallback
```

**Real-world use case**: Circuit breaker pattern, fallback to cache on service failure, retry from different source.

```java
public Mono<ProductDetails> getProductDetails(Long id) {
    return primaryCatalogService.getProduct(id)
        .onErrorResume(ServiceUnavailableException.class, e -> {
            log.warn("Primary catalog down, using cache: {}", e.getMessage());
            return cacheService.getProduct(id)  // async fallback
                .switchIfEmpty(Mono.error(new ProductNotFoundException(id)));
        });
}
```

**Execution**: When error arrives, the error is passed to the fallback function. The returned Mono is subscribed to. Its output becomes the downstream signal.

**Pitfall**: Order matters when chaining multiple `onErrorResume` — only the first matching one is used. More specific error types should come before broader ones.

---

## `Mono.timeout`

**What it does**: Signals a `TimeoutException` if the Mono doesn't emit within the specified duration.

**Signature**: `Mono<T> timeout(Duration timeout)`

Also: `Mono<T> timeout(Duration timeout, Mono<? extends T> fallback)` — fallback on timeout.

```java
return externalService.getData()
    .timeout(Duration.ofSeconds(5))
    .onErrorReturn(TimeoutException.class, Data.empty());
```

**Real-world use case**: Preventing indefinite waits on external services, enforcing SLAs.

```java
public Mono<PaymentResult> processPayment(PaymentRequest request) {
    return paymentGateway.submit(request)
        .timeout(
            Duration.ofSeconds(10),
            Mono.defer(() -> {
                // Timeout fallback: check async result later
                asyncResultChecker.scheduleCheck(request.getTransactionId());
                return Mono.just(PaymentResult.pending(request.getTransactionId()));
            })
        );
}
```

**Execution**: A timer starts when subscribed. If `onNext` or `onComplete` arrives before timeout — normal flow. If timer fires first — `TimeoutException` is emitted.

**Pitfall**: Setting timeout too short in a slow test environment causes flaky tests. Use different timeout values per environment.

**When NOT to use**: For operations that should run indefinitely (e.g., Kafka consumers, long-lived WebSocket connections). For those, use heartbeat/idle detection instead.

---

## `Mono.block`

**What it does**: Subscribes to the Mono and **blocks the calling thread** until the Mono completes, returning the result.

**Signature**: `T block()` and `T block(Duration timeout)`

```java
User user = userRepository.findById(1L).block();  // blocks until result
```

**When to use**: In tests, in `main()` methods, in non-reactive code that must call reactive code. Essentially: **only outside reactive pipelines**.

```java
// Test
@Test
void testFindUser() {
    User user = userRepository.findById(1L).block(Duration.ofSeconds(5));
    assertThat(user).isNotNull();
}

// Non-reactive integration point (migration scenario)
public User getLegacyUser(Long id) {
    return reactiveUserService.findById(id)
        .subscribeOn(Schedulers.boundedElastic())
        .block(Duration.ofSeconds(5));
}
```

**Pitfall — THE MOST DANGEROUS MISTAKE IN WEBFLUX**: Calling `.block()` inside a reactive pipeline or on a Netty event loop thread causes a deadlock:

```java
// CATASTROPHIC: calling block() inside a @GetMapping method
@GetMapping("/users/{id}")
public Mono<UserDto> getUser(@PathVariable Long id) {
    User user = userRepository.findById(id).block();  // DEADLOCK on Netty thread
    return Mono.just(toDto(user));
}
```

Reactor detects this and throws: `IllegalStateException: block()/blockFirst()/blockLast() are blocking, which is not supported in thread reactor-http-nio-X`

**When NOT to use**: Inside any reactive pipeline, inside any `@GetMapping`/`@PostMapping` etc., inside any `flatMap`/`map` that runs on a Netty thread. This is an absolute rule.

---

# 5. Flux — Complete Method Guide

## `Flux.just`

**What it does**: Creates a Flux that emits the given items in order, then completes.

**Signature**: `static <T> Flux<T> just(T... data)`

```java
Flux<String> flux = Flux.just("A", "B", "C");
flux.subscribe(System.out::println);
// A, B, C
```

**Real-world use case**: Static lists of items, test data, configuration values.

**Pitfall**: Like `Mono.just`, values are captured at creation time. For lazy evaluation: `Flux.defer(() -> Flux.just(dynamicValues()))`.

---

## `Flux.fromIterable`

**What it does**: Creates a Flux from an `Iterable` (List, Set, etc.).

**Signature**: `static <T> Flux<T> fromIterable(Iterable<? extends T> it)`

```java
List<User> users = List.of(user1, user2, user3);
Flux<User> flux = Flux.fromIterable(users);
```

**Real-world use case**: Converting in-memory lists to reactive streams for processing, wrapping the result of blocking queries.

```java
public Flux<UserDto> getAllUsers() {
    return Mono.fromCallable(() -> jdbcTemplate.query("SELECT * FROM users", userMapper))
        .subscribeOn(Schedulers.boundedElastic())
        .flatMapMany(Flux::fromIterable)  // List<User> → Flux<User>
        .map(this::toDto);
}
```

**Pitfall**: The Iterable is consumed **once**. If you subscribe twice to a Flux from a one-time Iterable (like a Stream), the second subscription may get no items. Use `Flux.defer(() -> Flux.fromIterable(getItems()))` for fresh items per subscription.

---

## `Flux.fromStream`

**What it does**: Creates a Flux from a Java `Stream`.

**Signature**: `static <T> Flux<T> fromStream(Stream<? extends T> s)`

```java
Flux<String> flux = Flux.fromStream(Stream.of("a", "b", "c"));
```

**Critical pitfall**: Java Streams are **single-use**. Subscribing twice to `Flux.fromStream(stream)` will throw `IllegalStateException` on the second subscription because the stream is already consumed.

```java
Stream<String> stream = Stream.of("a", "b", "c");
Flux<String> flux = Flux.fromStream(stream);
flux.subscribe(System.out::println);  // works: a, b, c
flux.subscribe(System.out::println);  // THROWS: stream already operated upon

// CORRECT: use the Supplier overload
Flux<String> safeFlux = Flux.fromStream(() -> Stream.of("a", "b", "c"));  // fresh stream per sub
```

**When to use**: Wrapping a Stream you receive from a library, processing large in-memory streams with backpressure.

---

## `Flux.range`

**What it does**: Emits a sequence of integers from `start` to `start + count - 1`.

**Signature**: `static Flux<Integer> range(int start, int count)`

```java
Flux.range(1, 5).subscribe(System.out::println);
// 1, 2, 3, 4, 5
```

**Real-world use case**: Pagination, generating test data, offset-based batch processing.

```java
int pageSize = 100;
int totalPages = 10;

Flux.range(0, totalPages)
    .flatMap(page -> repository.findPage(page, pageSize), 3)  // 3 pages concurrent
    .subscribe(this::processRecord);
```

---

## `Flux.interval`

**What it does**: Emits an incrementing `Long` (0, 1, 2...) at the given time interval, starting after an optional initial delay.

**Signature**: `static Flux<Long> interval(Duration period)`

```java
Flux.interval(Duration.ofSeconds(1))
    .take(5)
    .subscribe(i -> System.out.println("Tick: " + i));
// Tick: 0, Tick: 1, ..., Tick: 4  (one per second)
```

**Real-world use case**: Periodic polling, heartbeat signals, scheduled streaming.

```java
// Server-Sent Events: push stock prices every second
@GetMapping(value = "/prices/{symbol}", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<StockPrice> streamPrices(@PathVariable String symbol) {
    return Flux.interval(Duration.ofSeconds(1))
        .flatMap(tick -> stockService.getPrice(symbol))
        .distinctUntilChanged();
}
```

**Pitfall**: `Flux.interval` runs on `Schedulers.parallel()` by default — a non-blocking scheduler. If the subscriber can't keep up, backpressure handling may drop ticks (`onBackpressureDrop`). Be aware of the scheduling context.

---

## `Flux.empty`

**What it does**: Creates a Flux that completes immediately without emitting any items.

**Signature**: `static <T> Flux<T> empty()`

**Real-world use case**: No-op return, conditional empty streams.

```java
public Flux<Notification> getNotifications(Long userId) {
    if (!notificationsEnabled) return Flux.empty();
    return notificationRepository.findByUserId(userId);
}
```

---

## `Flux.map`

**What it does**: Applies a synchronous function to each item, transforming it. 1-to-1 element mapping.

**Signature**: `<V> Flux<V> map(Function<? super T, ? extends V> mapper)`

```java
Flux<UserDto> dtos = userFlux.map(user -> new UserDto(user.getId(), user.getName()));
```

**Real-world use case**: DTO conversion, field extraction, data transformation.

**Execution**: For each `onNext(item)`, the mapper runs synchronously and the result flows downstream immediately.

**Pitfall**: Same as Mono.map — no blocking, no async operations inside `map`.

---

## `Flux.flatMap`

**What it does**: Maps each item to a Publisher (Mono or Flux), then **merges** the inner publishers' outputs. Inner publishers run **concurrently** by default.

**Signature**: `<R> Flux<R> flatMap(Function<? super T, ? extends Publisher<? extends R>> mapper)`

Also: `flatMap(mapper, concurrency)` — limit concurrent inner subscriptions.

```java
Flux<UserDto> result = userIds
    .flatMap(id -> userRepository.findById(id))  // concurrent DB lookups
    .map(this::toDto);
```

**Real-world use case**: Parallel enrichment of items, fetching related data for each item in a collection.

```java
// Enrich each order with product details — up to 5 concurrent DB calls
return orderRepository.findByUserId(userId)
    .flatMap(order -> productRepository.findById(order.getProductId())
        .map(product -> OrderDto.of(order, product)), 5);  // concurrency = 5
```

**Execution**: Each item from upstream is mapped to an inner publisher. All inner publishers are subscribed to (up to `concurrency` limit). Their items are merged — output order may differ from input order.

**`flatMap` vs `concatMap`**:
- `flatMap`: concurrent, output order not guaranteed — use for throughput
- `concatMap`: sequential, preserves order — use when order matters or when inner publishers must not overlap

**Pitfall**: Unbounded concurrency (default is 256) can overwhelm downstream services. Always set a concurrency limit when calling external services:

```java
.flatMap(id -> externalService.call(id), 10)  // max 10 concurrent calls
```

---

## `Flux.concatMap`

**What it does**: Like `flatMap` but subscribes to inner publishers **one at a time, in order**. Only subscribes to the next inner publisher when the previous completes.

**Signature**: `<R> Flux<R> concatMap(Function<? super T, ? extends Publisher<? extends R>> mapper)`

```java
Flux.just(1, 2, 3)
    .concatMap(i -> Flux.just(i * 10, i * 100).delayElements(Duration.ofMillis(100)))
    .subscribe(System.out::println);
// 10, 100, 20, 200, 30, 300 — each group completes before the next starts
```

**Real-world use case**: Database migrations that must run in order, sequential API calls where each depends on the previous, processing ordered event streams.

```java
// Apply database migrations in sequence — order is critical
return Flux.fromIterable(pendingMigrations)
    .concatMap(migration -> migrationExecutor.apply(migration)
        .doOnSuccess(v -> log.info("Applied: {}", migration.getName())));
```

**When `concatMap` over `flatMap`**:
- Output must preserve input order
- Inner publishers must not overlap (e.g., step 2 must complete before step 3 starts)
- Simpler rate limiting (naturally sequential)

**Pitfall**: Much slower than `flatMap` for I/O-bound work — no parallelism. Don't use `concatMap` when you just want order-preserving output — use `flatMapSequential` which parallelizes but reorders output.

---

## `Flux.filter`

**What it does**: Emits only items that match the predicate.

**Signature**: `Flux<T> filter(Predicate<? super T> p)`

```java
Flux<User> activeUsers = allUsers.filter(User::isActive);
Flux<Integer> evens = Flux.range(1, 10).filter(n -> n % 2 == 0);
```

**Real-world use case**: Filtering results in the stream rather than in the query (when DB filtering is impractical), stream partitioning.

**Pitfall**: Filtering large streams wastes processing on items that won't be used. Filter at the source (DB query predicate) when possible. `filter` is appropriate when the source can't be filtered (e.g., Kafka topic, binary protocol).

---

## `Flux.take`

**What it does**: Emits at most the specified number of items, then cancels the upstream subscription.

**Signature**: `Flux<T> take(long n)` and `Flux<T> take(Duration timespan)`

```java
Flux.range(1, 1000).take(5).subscribe(System.out::println);
// 1, 2, 3, 4, 5  — then upstream is cancelled
```

**Real-world use case**: Pagination, limiting results, stopping an infinite stream after N items.

```java
// Process at most 100 events, then stop
kafkaFlux
    .take(100)
    .doOnComplete(() -> log.info("Processed 100 events"))
    .subscribe(this::processEvent);
```

**Execution**: After N items, `cancel()` is sent upstream — the producer stops producing. This is backpressure in action.

---

## `Flux.skip`

**What it does**: Ignores the first N items, then passes the rest downstream.

**Signature**: `Flux<T> skip(long skipped)` and `Flux<T> skip(Duration timespan)`

```java
Flux.range(1, 10).skip(3).subscribe(System.out::println);
// 4, 5, 6, 7, 8, 9, 10
```

**Real-world use case**: Offset-based pagination.

```java
public Flux<Item> getPage(int page, int size) {
    return itemRepository.findAll()
        .skip((long) page * size)
        .take(size);
}
```

**Pitfall**: `skip` still processes and discards the skipped elements — they are fetched from upstream but thrown away. For large offsets, this wastes resources. Prefer database-level pagination (OFFSET/LIMIT or cursor-based).

---

## `Flux.distinct`

**What it does**: Filters out duplicate items based on identity or a key extractor.

**Signature**: `Flux<T> distinct()` and `Flux<T> distinct(Function<? super T, ? extends V> keySelector)`

```java
Flux.just("a", "b", "a", "c", "b")
    .distinct()
    .subscribe(System.out::println);
// a, b, c

Flux.fromIterable(users)
    .distinct(User::getEmail)  // unique by email
    .subscribe(...);
```

**Pitfall**: `distinct()` keeps an internal `HashSet` of all seen items. For infinite or very large streams, this is a **memory leak**. Use `distinctUntilChanged()` for streams where duplicates only need to be suppressed when consecutive, or add a size limit.

---

## `Flux.sort`

**What it does**: Collects all items and emits them in sorted order. The entire Flux must complete before any item is emitted downstream.

**Signature**: `Flux<T> sort()` and `Flux<T> sort(Comparator<? super T> sortFunction)`

```java
Flux.just(3, 1, 4, 1, 5, 9)
    .sort()
    .subscribe(System.out::println);
// 1, 1, 3, 4, 5, 9
```

**Pitfall**: **Cannot be used on infinite streams** — `sort` must buffer everything. On large finite streams, it buffers all items in memory. Prefer sorting at the database level. Only use `sort` when sorting cannot be done at the source.

---

## `Flux.reduce`

**What it does**: Aggregates all items into a single value using an accumulator function. Emits the final accumulated value as a `Mono`.

**Signature**: `<A> Mono<A> reduce(A initial, BiFunction<A, ? super T, A> accumulator)`

```java
Mono<Integer> sum = Flux.range(1, 10)
    .reduce(0, Integer::sum);  // 55
```

**Real-world use case**: Aggregating stream results (totals, building collections).

```java
// Calculate total order value
Mono<BigDecimal> total = orderFlux
    .reduce(BigDecimal.ZERO, (acc, order) -> acc.add(order.getAmount()));
```

**Pitfall**: Like `sort`, `reduce` must process the entire stream before emitting. Not suitable for infinite streams.

---

## `Flux.collectList`

**What it does**: Collects all emitted items into a `List<T>` and emits it as a single `Mono<List<T>>`.

**Signature**: `Mono<List<T>> collectList()`

```java
Mono<List<User>> allUsers = userFlux.collectList();
```

**Real-world use case**: When downstream code expects a List, when you need the full collection before processing.

```java
return userRepository.findAll()
    .collectList()
    .map(users -> new UserBatch(users, users.size()));
```

**Pitfall**: Loads everything into memory. For large datasets, this negates the memory benefits of streaming. Prefer streaming operators when possible. Also: `collectList()` on an infinite Flux will never complete (waits forever).

---

## `Flux.collectMap`

**What it does**: Collects items into a `Map` keyed by a key extractor function.

**Signature**: `<K> Mono<Map<K, T>> collectMap(Function<? super T, ? extends K> keyExtractor)`

```java
Mono<Map<Long, User>> userMap = userFlux
    .collectMap(User::getId);
```

**Real-world use case**: Building lookup maps from streams, indexing results.

```java
// Build a product lookup map for an order
return productRepository.findAllById(order.getProductIds())
    .collectMap(Product::getId)
    .map(productMap -> order.getItems().stream()
        .map(item -> ItemDto.of(item, productMap.get(item.getProductId())))
        .toList());
```

**Pitfall**: Duplicate keys — by default, last value wins. Provide a merge function if duplicates are possible: `collectMap(keyFn, valueFn, mergeFn)`.

---

## `Flux.zipWith`

**What it does**: Combines this Flux with another Publisher element-by-element. Emits a pair (or combined value) for each matching pair of elements.

**Signature**: `<T2> Flux<Tuple2<T, T2>> zipWith(Publisher<? extends T2> source2)`

```java
Flux<String> names = Flux.just("Alice", "Bob", "Carol");
Flux<Integer> scores = Flux.just(95, 87, 92);

Flux<String> result = names.zipWith(scores, (name, score) -> name + ": " + score);
// "Alice: 95", "Bob: 87", "Carol: 92"
```

**Real-world use case**: Pairing two related streams, combining paginated results.

**Execution**: Waits for one item from each publisher, then combines them. Terminates when the shorter publisher completes.

**Pitfall**: If publishers emit at different rates, the faster one must buffer its items waiting for the slower one — memory implications.

---

## `Flux.merge`

**What it does**: Merges multiple publishers into one Flux. Items from all publishers are interleaved as they arrive. All publishers are subscribed to simultaneously.

**Signature**: `static <T> Flux<T> merge(Publisher<? extends T>... sources)`

```java
Flux<String> stream1 = Flux.just("A", "B").delayElements(Duration.ofMillis(100));
Flux<String> stream2 = Flux.just("1", "2").delayElements(Duration.ofMillis(150));

Flux.merge(stream1, stream2).subscribe(System.out::println);
// Output order depends on timing: A, 1, B, 2  (approximately)
```

**Real-world use case**: Combining results from multiple parallel data sources, merging notification streams.

```java
// Merge alerts from multiple monitoring systems
Flux<Alert> allAlerts = Flux.merge(
    prometheusAlerts.stream(),
    datadogAlerts.stream(),
    customMetricsAlerts.stream()
);
```

**`merge` vs `concat` vs `zip`**:
- `merge`: all publishers active simultaneously, items interleaved — use for parallel, order unimportant
- `concat`: publishers sequential, one after another — use when order matters
- `zip`: element-by-element pairing — use to combine paired data

---

## `Flux.buffer`

**What it does**: Groups items into lists (buffers) of the specified size, emitting each buffer as a `List<T>`.

**Signature**: `Flux<List<T>> buffer(int maxSize)` and `Flux<List<T>> buffer(Duration timespan)`

```java
Flux.range(1, 10).buffer(3).subscribe(System.out::println);
// [1, 2, 3], [4, 5, 6], [7, 8, 9], [10]
```

**Real-world use case**: Batch database inserts, batching API calls to reduce overhead.

```java
// Batch insert events in groups of 100
eventFlux
    .buffer(100)
    .flatMap(batch -> eventRepository.saveAll(batch), 5)  // 5 concurrent batches
    .subscribe();

// Time-based batching: flush every 500ms
sensorDataFlux
    .buffer(Duration.ofMillis(500))
    .filter(batch -> !batch.isEmpty())
    .flatMap(batch -> influxdb.writeBatch(batch))
    .subscribe();
```

**Pitfall**: If the upstream completes while the buffer is partially filled, the partial buffer is emitted. If a partial buffer is unacceptable, add validation. Also, if processing is slow and upstream is fast, buffers accumulate in memory.

---

## `Flux.window`

**What it does**: Like `buffer`, but each window is a `Flux<T>` rather than a `List<T>`. Windows can be processed reactively as they arrive.

**Signature**: `Flux<Flux<T>> window(int maxSize)` and `Flux<Flux<T>> window(Duration timespan)`

```java
Flux.range(1, 10)
    .window(3)
    .flatMap(window -> window.collectList())
    .subscribe(System.out::println);
// [1, 2, 3], [4, 5, 6], [7, 8, 9], [10]
```

**`window` vs `buffer`**:
- `buffer`: emits completed `List<T>` — simpler, requires all items in buffer before emitting
- `window`: emits `Flux<T>` windows — each window can be processed reactively as items arrive within the window

**Real-world use case**: Real-time aggregations over sliding or tumbling time windows.

```java
// Aggregate sensor readings in 1-second windows
sensorFlux
    .window(Duration.ofSeconds(1))
    .flatMap(window -> window.reduce(AggregatedReading::merge))
    .subscribe(this::storeAggregation);
```

---

## `Flux.doOnNext`

Same semantics as `Mono.doOnNext` but for each item in the Flux.

```java
return productRepository.findAll()
    .doOnNext(p -> log.debug("Processing product: {}", p.getId()))
    .doOnNext(p -> metricsService.incrementProcessed())
    .map(this::toDto);
```

---

## `Flux.doOnComplete`

**What it does**: Registers a `Runnable` that runs when the Flux completes normally (after the last `onNext`, on `onComplete`).

**Signature**: `Flux<T> doOnComplete(Runnable onComplete)`

```java
return eventFlux
    .doOnComplete(() -> log.info("All events processed"))
    .doOnComplete(() -> metrics.record("stream.completed"));
```

**Real-world use case**: Cleanup actions, logging stream completion, triggering downstream processes.

**Pitfall**: `doOnComplete` does NOT run on error. Use `doFinally` for guaranteed cleanup:

```java
.doFinally(signalType -> {
    if (signalType == SignalType.ON_COMPLETE) cleanup();
    else if (signalType == SignalType.ON_ERROR) handleError();
    else if (signalType == SignalType.CANCEL) handleCancel();
})
```

---

## `Flux.onErrorResume`

Same semantics as `Mono.onErrorResume` but for Flux. Replaces the errored Flux with a fallback Publisher.

```java
return userRepository.findAll()
    .onErrorResume(DatabaseException.class, e -> {
        log.error("DB unavailable, returning empty", e);
        return Flux.empty();
    });
```

**Pitfall**: After an error, the entire Flux terminates. `onErrorResume` provides a replacement stream from that point — you don't recover items emitted before the error from a different source. Use `onErrorContinue` if you want to skip errored items and continue:

```java
// onErrorContinue: skip bad items, continue with rest
Flux.just("1", "bad", "3")
    .map(Integer::parseInt)
    .onErrorContinue((err, item) -> log.warn("Skipping bad item: {}", item))
    .subscribe(System.out::println);
// 1, 3  (skips "bad")
```

---

# 6. WebClient — Complete Guide

## Setup

```java
// Minimal WebClient
WebClient client = WebClient.create("https://api.example.com");

// Production WebClient with full config
@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
        .responseTimeout(Duration.ofSeconds(10))
        .doOnConnected(conn ->
            conn.addHandlerLast(new ReadTimeoutHandler(10, TimeUnit.SECONDS))
                .addHandlerLast(new WriteTimeoutHandler(10, TimeUnit.SECONDS))
        );
    
    return WebClient.builder()
        .baseUrl("https://api.example.com")
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
        .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
        .codecs(configurer -> configurer
            .defaultCodecs()
            .maxInMemorySize(2 * 1024 * 1024))  // 2MB response buffer
        .filter(logRequest())
        .filter(logResponse())
        .build();
}

// Request/response logging filter
private ExchangeFilterFunction logRequest() {
    return ExchangeFilterFunction.ofRequestProcessor(request -> {
        log.debug("Request: {} {}", request.method(), request.url());
        return Mono.just(request);
    });
}
```

---

## `WebClient.create` / `WebClient.builder`

**Purpose**: Creates a WebClient instance. `create()` for simple cases, `builder()` for production configuration.

```java
// Simple
WebClient simple = WebClient.create("https://api.example.com");

// With builder (prefer this in production)
WebClient configured = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader("X-Api-Key", apiKey)
    .build();
```

**When to use builder**: Always in production. Set timeouts, response size limits, default headers, and filters.

---

## `get`, `post`, `put`, `delete`

**Purpose**: Start building an HTTP request of the given method.

```java
webClient.get()     // HTTP GET
webClient.post()    // HTTP POST
webClient.put()     // HTTP PUT
webClient.delete()  // HTTP DELETE
webClient.patch()   // HTTP PATCH
webClient.head()    // HTTP HEAD
```

Each returns a `RequestHeadersUriSpec` (for GET/DELETE) or `RequestBodyUriSpec` (for POST/PUT) for further chaining.

---

## `.uri`

**Purpose**: Specifies the URI to call. Supports template variables, query params, URI builders.

```java
// Static URI
webClient.get().uri("/users/123")

// Template variable
webClient.get().uri("/users/{id}", userId)

// Multiple variables
webClient.get().uri("/realms/{realm}/users/{id}", realm, userId)

// Query parameters via UriBuilder
webClient.get()
    .uri(uriBuilder -> uriBuilder
        .path("/users")
        .queryParam("page", page)
        .queryParam("size", size)
        .queryParam("active", true)
        .build()
    )

// Query params from a map
webClient.get()
    .uri(uriBuilder -> uriBuilder
        .path("/search")
        .queryParams(params)  // MultiValueMap
        .build()
    )
```

**Pitfall**: Never build the URI by string concatenation (`"/users/" + userId`) — use template variables. String concatenation is vulnerable to path traversal and doesn't handle encoding.

---

## `.header`

**Purpose**: Adds a header to the request.

```java
webClient.get()
    .uri("/protected")
    .header(HttpHeaders.AUTHORIZATION, "Bearer " + token)
    .header("X-Correlation-Id", correlationId)
    .header("X-Tenant-Id", tenantId)
```

**`headers()` for multiple**:
```java
.headers(h -> {
    h.setBearerAuth(token);
    h.set("X-Correlation-Id", correlationId);
    h.setContentType(MediaType.APPLICATION_JSON);
})
```

**Real-world pattern — propagate request context**:
```java
// Extract correlation ID from incoming request and propagate to outbound
@Component
public class CorrelationIdFilter implements WebFilter {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, WebFilterChain chain) {
        String correlationId = exchange.getRequest().getHeaders()
            .getFirst("X-Correlation-Id");
        return chain.filter(exchange)
            .contextWrite(Context.of("correlationId", correlationId));
    }
}

// In WebClient filter, read from context and add to request
private ExchangeFilterFunction correlationFilter() {
    return ExchangeFilterFunction.ofRequestProcessor(request ->
        Mono.deferContextual(ctx -> {
            String id = ctx.getOrDefault("correlationId", UUID.randomUUID().toString());
            return Mono.just(ClientRequest.from(request)
                .header("X-Correlation-Id", id)
                .build());
        })
    );
}
```

---

## `.bodyValue`

**Purpose**: Sets the request body from a synchronous value. The value is serialized (JSON by default) and sent as the request body.

**Signature**: `RequestHeadersSpec<?> bodyValue(Object body)`

```java
CreateUserRequest request = new CreateUserRequest("alice", "alice@example.com");

webClient.post()
    .uri("/users")
    .bodyValue(request)      // serialized to JSON automatically
    .retrieve()
    .bodyToMono(UserDto.class)
```

**For reactive body (Mono or Flux)**:
```java
.body(Mono.just(request), CreateUserRequest.class)
// or
.body(BodyInserters.fromValue(request))
```

**For form data**:
```java
MultiValueMap<String, String> formData = new LinkedMultiValueMap<>();
formData.add("grant_type", "client_credentials");
formData.add("client_id", clientId);

webClient.post()
    .uri("/token")
    .contentType(MediaType.APPLICATION_FORM_URLENCODED)
    .body(BodyInserters.fromFormData(formData))
    .retrieve()
    .bodyToMono(TokenResponse.class)
```

---

## `.retrieve`

**Purpose**: Executes the request and provides access to the response body. The simplest way to get a response.

```java
Mono<UserDto> user = webClient.get()
    .uri("/users/{id}", id)
    .retrieve()
    .bodyToMono(UserDto.class);
```

**Error handling with `retrieve`**:
```java
webClient.get()
    .uri("/users/{id}", id)
    .retrieve()
    .onStatus(HttpStatusCode::is4xxClientError, response ->
        response.bodyToMono(ErrorDto.class)
            .flatMap(error -> Mono.error(new ApiException(error.message())))
    )
    .onStatus(HttpStatusCode::is5xxServerError, response ->
        Mono.error(new ServiceUnavailableException("User service error"))
    )
    .bodyToMono(UserDto.class);
```

**`retrieve` vs `exchangeToMono`**: See dedicated section below.

---

## `.bodyToMono`

**Purpose**: Deserializes the response body into a `Mono<T>`. For responses with a single object.

**Signature**: `<T> Mono<T> bodyToMono(Class<T> elementClass)`

```java
Mono<UserDto> user = ...retrieve().bodyToMono(UserDto.class);
Mono<String> rawJson = ...retrieve().bodyToMono(String.class);
Mono<Void> noBody = ...retrieve().bodyToMono(Void.class);  // ignore response body
```

**For generic types**:
```java
Mono<List<UserDto>> users = ...retrieve()
    .bodyToMono(new ParameterizedTypeReference<List<UserDto>>() {});
```

**Pitfall**: `bodyToMono` buffers the entire response body in memory before deserializing. For large responses (>a few MB), configure `maxInMemorySize` in the codecs or use `bodyToFlux` for streaming.

---

## `.bodyToFlux`

**Purpose**: Deserializes a streaming response body into a `Flux<T>`. Each JSON array element (or newline-delimited JSON) becomes an item.

```java
Flux<UserDto> users = webClient.get()
    .uri("/users")
    .retrieve()
    .bodyToFlux(UserDto.class);
```

**Real-world use case**: Consuming streaming APIs, Server-Sent Events, large paginated data.

```java
// Server-Sent Events consumer
Flux<StockPrice> prices = webClient.get()
    .uri("/prices/stream")
    .accept(MediaType.TEXT_EVENT_STREAM)
    .retrieve()
    .bodyToFlux(StockPrice.class);
```

---

## `.exchangeToMono`

**Purpose**: Gives full access to the `ClientResponse` object (status, headers, body). More control than `retrieve`, but requires manual body consumption.

**Signature**: `<V> Mono<V> exchangeToMono(Function<ClientResponse, Mono<V>> responseHandler)`

```java
Mono<UserDto> result = webClient.get()
    .uri("/users/{id}", id)
    .exchangeToMono(response -> {
        if (response.statusCode().equals(HttpStatus.OK)) {
            return response.bodyToMono(UserDto.class);
        } else if (response.statusCode().equals(HttpStatus.NOT_FOUND)) {
            return Mono.empty();
        } else {
            return response.bodyToMono(ErrorDto.class)
                .flatMap(err -> Mono.error(new ApiException(err.message())));
        }
    });
```

### `retrieve` vs `exchangeToMono` — When to Use Which

| Factor | `retrieve` | `exchangeToMono` |
|---|---|---|
| Simplicity | Simpler | More verbose |
| Access to headers | No (body only) | Yes — full ClientResponse |
| Status code handling | Via `onStatus` callbacks | Direct in lambda |
| Response body must be consumed | Automatic | **MANUAL — must consume or release** |
| When to use | Standard cases | Custom status handling, need response headers, conditional body reading |

**Critical pitfall with `exchangeToMono`**: You **must** consume or release the response body in every code path. Not doing so leaks connections:

```java
// WRONG: if condition fails, body is never consumed — connection leak
.exchangeToMono(response -> {
    if (response.statusCode().is2xxSuccessful()) {
        return response.bodyToMono(UserDto.class);
    }
    return Mono.empty();  // body NOT consumed — memory/connection leak!
})

// CORRECT: always release the body
.exchangeToMono(response -> {
    if (response.statusCode().is2xxSuccessful()) {
        return response.bodyToMono(UserDto.class);
    }
    return response.releaseBody()  // explicitly release
        .then(Mono.empty());
})
```

---

## `.onStatus`

**Purpose**: Handles specific HTTP status codes when using `retrieve`. Converts error status codes into exceptions.

**Signature**: `ResponseSpec onStatus(Predicate<HttpStatusCode> statusPredicate, Function<ClientResponse, Mono<? extends Throwable>> exceptionFunction)`

```java
webClient.get()
    .uri("/users/{id}", id)
    .retrieve()
    .onStatus(status -> status.value() == 404,
        response -> Mono.error(new UserNotFoundException(id)))
    .onStatus(status -> status.value() == 403,
        response -> Mono.error(new AccessDeniedException("Insufficient permissions")))
    .onStatus(HttpStatusCode::is5xxServerError,
        response -> response.bodyToMono(String.class)
            .flatMap(body -> Mono.error(new ServiceException("Service error: " + body))))
    .bodyToMono(UserDto.class);
```

**Default behavior**: By default, `retrieve` automatically throws `WebClientResponseException` for 4xx and 5xx. Use `onStatus` to replace this with domain-specific exceptions.

**Pitfall**: When using `onStatus`, the error response body is consumed by your function — you can't read it again. Make sure to extract needed information inside the `onStatus` handler.

---

## Full WebClient Example — Service with Retry and Fallback

```java
@Service
@RequiredArgsConstructor
public class UserApiClient {

    private final WebClient webClient;

    public Mono<UserDto> getUser(Long userId) {
        return webClient.get()
            .uri("/users/{id}", userId)
            .header("X-Request-Id", UUID.randomUUID().toString())
            .retrieve()
            .onStatus(status -> status.value() == 404,
                resp -> Mono.error(new UserNotFoundException(userId)))
            .onStatus(HttpStatusCode::is5xxServerError,
                resp -> resp.bodyToMono(String.class)
                    .flatMap(body -> Mono.error(new ServiceException(body))))
            .bodyToMono(UserDto.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(500))
                .filter(ex -> ex instanceof ServiceException)  // retry only on server errors
                .onRetryExhaustedThrow((spec, signal) -> signal.failure()))
            .doOnError(e -> log.error("Failed to fetch user {}: {}", userId, e.getMessage()))
            .onErrorResume(ServiceException.class,
                e -> cacheService.getCachedUser(userId));  // fallback to cache
    }
    
    public Flux<UserDto> getAllUsers(int page, int size) {
        return webClient.get()
            .uri(builder -> builder
                .path("/users")
                .queryParam("page", page)
                .queryParam("size", size)
                .build())
            .retrieve()
            .bodyToFlux(UserDto.class)
            .timeout(Duration.ofSeconds(15))
            .doOnError(e -> log.error("Failed to fetch users page {}: {}", page, e.getMessage()));
    }

    public Mono<UserDto> createUser(CreateUserRequest request) {
        return webClient.post()
            .uri("/users")
            .bodyValue(request)
            .retrieve()
            .onStatus(status -> status.value() == 409,
                resp -> Mono.error(new UserAlreadyExistsException(request.email())))
            .bodyToMono(UserDto.class);
    }
}
```

---

# 7. Functional Endpoints (ServerResponse)

## Why Functional Endpoints Exist

Annotation-based controllers (`@GetMapping`, `@RestController`) are declarative and familiar, but the routing and handling are coupled to Spring's MVC dispatcher machinery. Functional endpoints separate routing from handling: routes are explicit `RouterFunction` beans, handlers are plain functions.

**Use annotation-based when**: Standard CRUD API, team familiar with Spring MVC, simple request/response.
**Use functional when**: Complex routing logic, handler composition, testing handlers in isolation without Spring context, more explicit control over request/response.

## Core Components

```java
// RouterFunction: maps requests to handlers
RouterFunction<ServerResponse> router = RouterFunctions
    .route(GET("/users/{id}"), handler::getUser)
    .andRoute(POST("/users"), handler::createUser);

// HandlerFunction: processes a request and returns a response
// ServerRequest  → Mono<ServerResponse>
public Mono<ServerResponse> getUser(ServerRequest request) { ... }
```

## `ServerResponse.ok`

**What it does**: Creates a 200 OK response builder.

```java
public Mono<ServerResponse> getUser(ServerRequest request) {
    Long id = Long.parseLong(request.pathVariable("id"));
    return userService.findById(id)
        .flatMap(user -> ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(user));
}
```

---

## `ServerResponse.created`

**What it does**: Creates a 201 Created response builder. Requires a `Location` URI.

```java
public Mono<ServerResponse> createUser(ServerRequest request) {
    return request.bodyToMono(CreateUserRequest.class)
        .flatMap(userService::createUser)
        .flatMap(user -> ServerResponse.created(
                URI.create("/users/" + user.getId()))
            .bodyValue(user));
}
```

---

## `ServerResponse.notFound`

**What it does**: Creates a 404 Not Found response builder.

```java
public Mono<ServerResponse> getUser(ServerRequest request) {
    Long id = Long.parseLong(request.pathVariable("id"));
    return userService.findById(id)
        .flatMap(user -> ServerResponse.ok().bodyValue(user))
        .switchIfEmpty(ServerResponse.notFound().build());
}
```

---

## `ServerResponse.badRequest`

**What it does**: Creates a 400 Bad Request response builder.

```java
public Mono<ServerResponse> createUser(ServerRequest request) {
    return request.bodyToMono(CreateUserRequest.class)
        .flatMap(req -> {
            if (req.email() == null) {
                return ServerResponse.badRequest()
                    .bodyValue(Map.of("error", "Email is required"));
            }
            return userService.create(req)
                .flatMap(user -> ServerResponse.created(
                    URI.create("/users/" + user.getId())).bodyValue(user));
        });
}
```

---

## `.bodyValue`

**What it does**: Sets the response body from a synchronous value (auto-serialized to JSON/XML).

```java
ServerResponse.ok().bodyValue(userDto)
ServerResponse.ok().bodyValue(List.of(dto1, dto2))
```

---

## `.body`

**What it does**: Sets the response body from a reactive publisher (Mono or Flux).

```java
// From Mono
ServerResponse.ok().body(userMono, UserDto.class)

// From Flux (streaming)
ServerResponse.ok()
    .contentType(MediaType.APPLICATION_NDJSON)  // newline-delimited JSON
    .body(userFlux, UserDto.class)
```

---

## `.contentType`

**What it does**: Sets the `Content-Type` response header.

```java
ServerResponse.ok()
    .contentType(MediaType.APPLICATION_JSON)
    .bodyValue(data)

// Server-Sent Events
ServerResponse.ok()
    .contentType(MediaType.TEXT_EVENT_STREAM)
    .body(eventFlux, String.class)
```

---

## `.build`

**What it does**: Builds a response with no body (used for 204 No Content, 404, etc.).

```java
ServerResponse.noContent().build()   // 204
ServerResponse.notFound().build()    // 404
ServerResponse.accepted().build()    // 202
```

---

## Complete Functional Endpoint Example

```java
@Configuration
public class UserRouter {

    @Bean
    public RouterFunction<ServerResponse> userRoutes(UserHandler handler) {
        return RouterFunctions
            .route(GET("/api/v1/users"), handler::listUsers)
            .andRoute(GET("/api/v1/users/{id}"), handler::getUser)
            .andRoute(POST("/api/v1/users"), handler::createUser)
            .andRoute(PUT("/api/v1/users/{id}"), handler::updateUser)
            .andRoute(DELETE("/api/v1/users/{id}"), handler::deleteUser)
            // Nested routes
            .andRoute(GET("/api/v1/users/{id}/orders"), handler::getUserOrders);
    }
}

@Component
@RequiredArgsConstructor
public class UserHandler {

    private final UserService userService;
    private final Validator validator;

    public Mono<ServerResponse> listUsers(ServerRequest request) {
        int page = Integer.parseInt(request.queryParam("page").orElse("0"));
        int size = Integer.parseInt(request.queryParam("size").orElse("20"));
        
        return ServerResponse.ok()
            .contentType(MediaType.APPLICATION_JSON)
            .body(userService.findAll(page, size), UserDto.class);
    }
    
    public Mono<ServerResponse> getUser(ServerRequest request) {
        Long id = Long.parseLong(request.pathVariable("id"));
        
        return userService.findById(id)
            .flatMap(user -> ServerResponse.ok().bodyValue(user))
            .switchIfEmpty(ServerResponse.notFound().build());
    }
    
    public Mono<ServerResponse> createUser(ServerRequest request) {
        return request.bodyToMono(CreateUserRequest.class)
            .doOnNext(this::validate)
            .flatMap(userService::create)
            .flatMap(user -> ServerResponse
                .created(request.uriBuilder().path("/{id}").build(user.getId()))
                .bodyValue(user))
            .onErrorResume(ValidationException.class, e ->
                ServerResponse.badRequest().bodyValue(Map.of("error", e.getMessage())));
    }
    
    private void validate(CreateUserRequest req) {
        Set<ConstraintViolation<CreateUserRequest>> violations = validator.validate(req);
        if (!violations.isEmpty()) {
            throw new ValidationException(violations.iterator().next().getMessage());
        }
    }
}
```

---

# 8. Schedulers & Threading

## The Critical Mental Model

In WebFlux, operators don't have a fixed thread — they execute on **whatever thread signals them**. The thread that calls `onNext` on an operator is the thread that operator runs on, unless you change it with `publishOn` or `subscribeOn`.

## `subscribeOn`

**What it does**: Specifies the scheduler for the **subscription** (source) side. It affects where the source publisher's `subscribe()` method runs — typically where blocking I/O happens.

**Rule**: `subscribeOn` affects upstream execution regardless of where in the chain it appears. Only the **first** `subscribeOn` in a chain has effect (subsequent ones are ignored for upstream).

```java
Mono.fromCallable(() -> blockingDb.findById(id))   // blocking call
    .subscribeOn(Schedulers.boundedElastic())        // run subscription on boundedElastic
    .map(this::toDto)                               // runs on boundedElastic too (upstream thread)
    .subscribe();
```

**When to use**: Wrapping blocking calls. `subscribeOn(Schedulers.boundedElastic())` tells Reactor to start the pipeline on a thread that's safe to block.

**Execution flow**:
```
subscribe() called on main thread
    → subscribeOn routes to boundedElastic thread
    → fromCallable runs on boundedElastic (blocking OK here)
    → map runs on boundedElastic (continues on same thread)
    → downstream receives result
```

---

## `publishOn`

**What it does**: Switches the thread for **downstream** operators. Everything **after** `publishOn` runs on the specified scheduler. Everything before it is unchanged.

```java
Flux.fromIterable(ids)                              // main thread
    .subscribeOn(Schedulers.boundedElastic())        // subscription on boundedElastic
    .map(this::loadFromDb)                          // runs on boundedElastic (blocking OK)
    .publishOn(Schedulers.parallel())               // ── switch thread ──
    .map(this::cpuIntensiveTransform)               // runs on parallel thread
    .subscribe();
```

**When to use**: After a blocking upstream, switch to a non-blocking thread for CPU-bound downstream processing. For thread hop between different workload types.

**`subscribeOn` vs `publishOn`**:

| | `subscribeOn` | `publishOn` |
|---|---|---|
| Affects | Upstream (source) | Downstream (after it) |
| Placement | Anywhere, first wins | Placement matters |
| Use for | Source subscription (blocking I/O) | Switching thread mid-pipeline |

```java
// Typical pattern: blocking source → boundedElastic → parallel for CPU work
Mono.fromCallable(() -> jdbcQuery())               // blocking
    .subscribeOn(Schedulers.boundedElastic())       // run blocking on elastic thread
    .publishOn(Schedulers.parallel())              // switch to parallel for CPU work
    .map(this::heavyTransformation)                // CPU-bound work on parallel
    .subscribe();
```

---

## `Schedulers.boundedElastic`

**What it is**: A thread pool for **blocking** operations. Dynamically grows up to a bound (default: 10 × CPU cores), with queuing for excess tasks. Threads are reused and idle threads expire.

**Use for**: Wrapping blocking calls — JDBC, blocking HTTP clients, file I/O, any legacy blocking code.

```java
Mono.fromCallable(() -> Files.readString(Path.of(filePath)))
    .subscribeOn(Schedulers.boundedElastic())
```

**Why "bounded"**: The old `Schedulers.elastic()` was unbounded — could create unlimited threads, causing resource exhaustion under load. `boundedElastic` adds a ceiling and a work queue.

**Pitfall**: `boundedElastic` is not free — creating/managing threads has overhead. Don't use it to make blocking code "reactive by default." Use it as an isolation layer while you migrate to truly non-blocking drivers (R2DBC, reactive HTTP clients).

---

## `Schedulers.parallel`

**What it is**: A fixed-size thread pool with `N = CPU cores` threads. Designed for **CPU-bound, non-blocking** work.

**Use for**: CPU-intensive computations, data transformations, in-memory operations.

```java
Flux.range(1, 1000)
    .parallel()                           // split into parallel rails
    .runOn(Schedulers.parallel())         // each rail on parallel thread
    .map(this::cpuIntensiveTransform)
    .sequential()                         // merge back to single flux
    .subscribe();
```

**Pitfall**: **Never block a parallel thread.** It only has N=CPUs threads — blocking one reduces parallelism. A blocked parallel thread = performance degradation for all CPU work.

---

## `Schedulers.single`

**What it is**: A single reusable thread for all tasks. Sequential execution, no parallelism.

**Use for**: Operations that must be single-threaded (non-thread-safe resources), ordered processing guarantees.

```java
Flux.interval(Duration.ofMillis(100), Schedulers.single())  // timer on dedicated thread
```

**Pitfall**: A bottleneck by design. Don't use for parallel work or anywhere latency matters.

---

## Thread Switching Example — Real-World Pattern

```java
@Service
public class ReportService {

    public Mono<Report> generateReport(Long userId) {
        return Mono.fromCallable(() -> jdbcTemplate.query(...))  // 1. blocking DB on boundedElastic
            .subscribeOn(Schedulers.boundedElastic())
            .flatMap(rawData ->                                  // 2. still on boundedElastic
                Mono.fromCallable(() -> parseRawData(rawData))   //    another blocking call
                    .subscribeOn(Schedulers.boundedElastic())
            )
            .publishOn(Schedulers.parallel())                    // 3. switch to parallel
            .map(data -> computeAggregations(data))              //    CPU-bound: parallel thread
            .map(data -> formatReport(data))                     //    CPU-bound: parallel thread
            .publishOn(Schedulers.boundedElastic())              // 4. switch back for I/O
            .flatMap(report ->                                   //    write to disk: boundedElastic
                Mono.fromCallable(() -> writeToFile(report))
                    .subscribeOn(Schedulers.boundedElastic())
            );
    }
}
```

---

## Common Threading Mistakes

### Mistake 1: Blocking on the Event Loop

```java
// FATAL: blocks Netty's event loop thread
@GetMapping("/users/{id}")
public Mono<UserDto> getUser(@PathVariable Long id) {
    User user = jdbcRepository.findById(id);  // BLOCKS event loop
    return Mono.just(toDto(user));
}

// CORRECT
@GetMapping("/users/{id}")
public Mono<UserDto> getUser(@PathVariable Long id) {
    return Mono.fromCallable(() -> jdbcRepository.findById(id))
        .subscribeOn(Schedulers.boundedElastic())
        .map(this::toDto);
}
```

### Mistake 2: Not Using subscribeOn with fromCallable

```java
// WRONG: fromCallable runs on whatever thread subscribes — could be event loop
Mono.fromCallable(() -> blockingQuery());

// CORRECT: always specify where it runs
Mono.fromCallable(() -> blockingQuery())
    .subscribeOn(Schedulers.boundedElastic());
```

### Mistake 3: Assuming Thread Identity

```java
// WRONG: assuming the same thread throughout
@GetMapping("/")
public Mono<String> handler() {
    ThreadLocal<String> tl = new ThreadLocal<>();
    tl.set("value");
    
    return Mono.fromCallable(() -> tl.get())  // NULL! Different thread after subscribeOn
        .subscribeOn(Schedulers.boundedElastic());
}

// CORRECT: use Reactor Context, not ThreadLocal
return Mono.deferContextual(ctx ->
    Mono.just(ctx.get("myKey"))
).contextWrite(Context.of("myKey", "value"));
```

---

# 9. WebFlux in System Design

## When to Use WebFlux in Real Systems

**The right question isn't "should we use WebFlux?" but "what is our bottleneck?"**

If your service:
- Handles >100 concurrent I/O-bound requests
- Makes multiple parallel outbound HTTP/DB calls per request
- Serves streaming data (SSE, WebSocket, large files)
- Is an aggregation/BFF layer calling multiple services

→ WebFlux provides measurable benefits.

If your service:
- Has moderate load (<50 concurrent requests)
- Is CPU-bound (heavy computation per request)
- Uses blocking dependencies you can't replace
- Has a small team unfamiliar with reactive

→ Spring MVC is the better choice.

## High-Concurrency APIs

```
WebFlux + R2DBC: handles 50,000+ concurrent connections on 16 CPU cores
Spring MVC + JDBC: handles ~2,000 concurrent connections (200-thread Tomcat pool × 10x headroom)
```

The advantage is not raw throughput for simple requests — it's concurrency under load. When each request waits 50–200ms for I/O, WebFlux multiplexes thousands of such waits on a handful of threads.

## Streaming APIs

WebFlux is purpose-built for streaming:

```java
// Server-Sent Events — push data to browser in real-time
@GetMapping(value = "/notifications", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<NotificationDto>> streamNotifications(
        @RequestParam String userId) {
    
    return notificationService.streamForUser(userId)
        .map(notification -> ServerSentEvent.<NotificationDto>builder()
            .id(notification.getId().toString())
            .event("notification")
            .data(notification)
            .build())
        .doOnCancel(() -> log.info("Client disconnected: {}", userId));
}
```

Streaming with MVC would require `SseEmitter` — more boilerplate, less composable.

## Integration with Blocking Systems

The hardest part of adopting WebFlux in existing systems is interacting with blocking dependencies.

### Pattern: Isolate Blocking with boundedElastic

```java
@Repository
public class BlockingUserRepository {

    private final JdbcTemplate jdbcTemplate;
    
    // Wrap blocking JDBC in reactive, isolated on boundedElastic
    public Mono<User> findById(Long id) {
        return Mono.fromCallable(() ->
                jdbcTemplate.queryForObject(
                    "SELECT * FROM users WHERE id = ?",
                    userRowMapper, id))
            .subscribeOn(Schedulers.boundedElastic());
    }
    
    public Flux<User> findAll() {
        return Mono.fromCallable(() ->
                jdbcTemplate.query("SELECT * FROM users", userRowMapper))
            .subscribeOn(Schedulers.boundedElastic())
            .flatMapMany(Flux::fromIterable);
    }
}
```

### The Honest Assessment

Wrapping blocking JDBC in `fromCallable + subscribeOn(boundedElastic)` is **not true reactive**. You still have:
- One thread per concurrent request in boundedElastic (just like Tomcat, but with more overhead)
- No backpressure from the DB driver
- Context switching between event loop and elastic threads

For true reactive benefits with a relational DB: use **R2DBC** (Reactive Relational Database Connectivity).

```java
// True reactive with R2DBC
@Repository
public interface UserRepository extends ReactiveCrudRepository<User, Long> {
    Flux<User> findByActiveTrue();
    Mono<User> findByEmail(String email);
}
```

---

# 10. Production Best Practices

## Avoiding Blocking Calls

This is rule #1. A single blocking call on the event loop thread degrades performance for all requests on that thread.

**Detection tools**:
```java
// BlockHound: development-time tool that throws on blocking calls in reactive threads
// Add to build, enable in tests:
BlockHound.install();

// In tests:
BlockHound.install(
    builder -> builder
        .allowBlockingCallsInside("com.example.MyLegacyClass", "allowedMethod")
);
```

```xml
<!-- Maven -->
<dependency>
    <groupId>io.projectreactor.tools</groupId>
    <artifactId>blockhound</artifactId>
    <version>1.0.8.RELEASE</version>
    <scope>test</scope>
</dependency>
```

**Common hidden blocking calls**:
- `SLF4J` with synchronous appenders — use async appenders in production
- `@Transactional` with JPA — JPA is blocking; use R2DBC + `@Transactional` from `spring-data-r2dbc`
- `SecurityContextHolder.getContext()` — use Reactor Context instead
- `RestTemplate` — use `WebClient`
- Any `Thread.sleep()` — use `Mono.delay()`

## Debugging Reactive Pipelines

Reactive stack traces are notoriously unhelpful:

```
// Typical useless stack trace:
java.lang.NullPointerException: null
    at reactor.core.publisher.MonoMap.subscribe(MonoMap.java:58)
    at reactor.core.publisher.MonoFlatMap.subscribe(MonoFlatMap.java:66)
    ...
    at reactor.core.publisher.MonoSubscribeOn.subscribe(MonoSubscribeOn.java:54)
```

### Hook-Based Debugging

```java
// Enable in development — captures assembly stack traces (expensive!)
Hooks.onOperatorDebug();

// Or per-pipeline (less overhead):
Mono<User> pipeline = userService.findById(id)
    .checkpoint("After findById")
    .map(this::toDto)
    .checkpoint("After toDto");
```

### `log()` Operator

```java
return userRepository.findById(id)
    .log("UserService.findById", Level.INFO)  // logs all signals with prefix
    .map(this::toDto)
    .log("UserService.toDto");

// Output:
// INFO  UserService.findById: | onSubscribe([MonoNext])
// INFO  UserService.findById: | request(unbounded)
// INFO  UserService.findById: | onNext(User{id=1})
// INFO  UserService.findById: | onComplete()
```

### Reactor Tools — Better Stack Traces

```xml
<dependency>
    <groupId>io.projectreactor</groupId>
    <artifactId>reactor-tools</artifactId>
</dependency>
```

```java
ReactorDebugAgent.init();  // production-safe, uses bytecode instrumentation
```

## Logging Strategies

Traditional logging is tricky in reactive because a single request may span multiple threads. `MDC` (Mapped Diagnostic Context) is thread-local — it doesn't survive thread switches.

```java
// WRONG: MDC set on one thread, lost after subscribeOn/publishOn
MDC.put("userId", userId);
return Mono.fromCallable(() -> db.find(userId))
    .subscribeOn(Schedulers.boundedElastic());  // MDC is GONE here

// CORRECT: propagate context through Reactor Context, restore MDC per operator
public static <T> Mono<T> withMdc(String key, String value, Mono<T> mono) {
    return mono.contextWrite(Context.of(key, value))
        .doOnEach(signal -> {
            if (!signal.isOnComplete()) {
                ContextView ctx = signal.getContextView();
                ctx.getOrEmpty(key).ifPresent(v -> MDC.put(key, v.toString()));
            }
        });
}

// Usage
return withMdc("userId", userId.toString(), 
    userService.findById(userId).map(this::toDto));
```

**Structured logging approach**: Use `doOnEach` with `signal.getContextView()` to restore MDC before each log statement.

## Backpressure Handling Strategies

```java
// Strategy 1: Buffer (risky for large bursts)
fastProducer
    .onBackpressureBuffer(1000,
        dropped -> log.warn("Buffer full, dropped: {}", dropped),
        BufferOverflowStrategy.DROP_OLDEST)

// Strategy 2: Drop (lose data, but don't crash)
fastProducer
    .onBackpressureDrop(item -> log.warn("Dropped due to backpressure: {}", item))

// Strategy 3: Error (fail fast)
fastProducer
    .onBackpressureError()  // throws OverflowException when buffer full

// Strategy 4: Latest (keep only the most recent)
fastProducer
    .onBackpressureLatest()  // always keeps latest value, drops older ones
```

**Choose based on domain**:
- Financial transactions: `onBackpressureBuffer` with error (never drop)
- Metrics/telemetry: `onBackpressureDrop` or `onBackpressureLatest` (losing some is OK)
- Real-time prices: `onBackpressureLatest` (stale prices are useless)

## Testing Reactive Code

```java
// StepVerifier: the primary testing tool
@Test
void testFindUser() {
    StepVerifier.create(userService.findById(1L))
        .expectNextMatches(user -> user.getId().equals(1L))
        .expectComplete()
        .verify(Duration.ofSeconds(5));
}

// Test Flux
@Test
void testFindAll() {
    StepVerifier.create(userService.findAll())
        .expectNextCount(3)
        .expectComplete()
        .verify();
}

// Test error case
@Test
void testUserNotFound() {
    StepVerifier.create(userService.findById(999L))
        .expectError(UserNotFoundException.class)
        .verify();
}

// Virtual time for time-based operators (instant testing)
@Test
void testRetryWithDelay() {
    StepVerifier.withVirtualTime(() ->
            userService.findWithRetry(1L))  // has .retryWhen with 1-second delay
        .expectSubscription()
        .thenAwait(Duration.ofSeconds(3))   // advance virtual time 3 seconds
        .expectNextCount(1)
        .expectComplete()
        .verify();
}

// WebTestClient for integration tests
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerTest {

    @Autowired
    private WebTestClient webTestClient;
    
    @Test
    void getUser() {
        webTestClient.get()
            .uri("/api/v1/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(UserDto.class)
            .value(user -> assertThat(user.id()).isEqualTo(1L));
    }
    
    @Test
    void getUserNotFound() {
        webTestClient.get()
            .uri("/api/v1/users/999")
            .exchange()
            .expectStatus().isNotFound();
    }
}
```

---

# 11. Real-World Scenarios

## Scenario 1: High-Throughput API Gateway / BFF

### Context
A Backend For Frontend (BFF) service aggregates data from 4 microservices to serve a dashboard. Each page load requires data from: User Service, Order Service, Product Service, and Recommendation Service.

### Why WebFlux
Without WebFlux: 4 sequential calls × 50ms each = 200ms minimum. With WebFlux: all 4 calls run in parallel = ~50ms (slowest service).

```java
@RestController
@RequestMapping("/api/v1/dashboard")
@RequiredArgsConstructor
public class DashboardController {

    private final UserApiClient userClient;
    private final OrderApiClient orderClient;
    private final ProductApiClient productClient;
    private final RecommendationApiClient recoClient;

    @GetMapping("/{userId}")
    public Mono<DashboardDto> getDashboard(@PathVariable Long userId) {
        // All 4 calls start simultaneously
        Mono<UserDto> user = userClient.getUser(userId).cache();
        Mono<List<OrderDto>> orders = orderClient.getRecentOrders(userId, 5)
            .collectList()
            .onErrorReturn(List.of());  // degrade gracefully
        Mono<List<RecommendationDto>> recommendations = recoClient.getFor(userId)
            .collectList()
            .timeout(Duration.ofSeconds(1))  // don't let slow reco service affect dashboard
            .onErrorReturn(List.of());
        
        // Products depend on orders — sequential dependency
        Mono<Map<Long, ProductDto>> products = orders
            .flatMap(orderList -> {
                List<Long> productIds = orderList.stream()
                    .map(OrderDto::productId).distinct().toList();
                return productClient.getByIds(productIds)
                    .collectMap(ProductDto::id);
            })
            .onErrorReturn(Map.of());
        
        // Combine all results
        return Mono.zip(user, orders, products, recommendations)
            .map(tuple -> DashboardDto.of(
                tuple.getT1(), tuple.getT2(), tuple.getT3(), tuple.getT4()));
    }
}
```

### Architecture
```
Browser ──► BFF (WebFlux) ──┬──► User Service
                             ├──► Order Service   (all parallel)
                             ├──► Product Service
                             └──► Recommendation Service
```

### Result
Response time ≈ max(service latencies) instead of sum. With timeouts and `onErrorReturn`, failures in non-critical services don't break the dashboard.

---

## Scenario 2: Real-Time Streaming Service

### Context
A monitoring platform that streams live application metrics to operator dashboards via Server-Sent Events.

```java
@RestController
@RequiredArgsConstructor
public class MetricsController {

    private final MetricsService metricsService;

    @GetMapping(value = "/stream/metrics/{appId}",
                produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<MetricSnapshot>> streamMetrics(
            @PathVariable String appId,
            @RequestParam(defaultValue = "1000") long intervalMs) {
        
        return Flux.interval(Duration.ofMillis(intervalMs))
            .onBackpressureDrop()  // drop ticks if consumer is slow
            .flatMap(tick -> metricsService.collectSnapshot(appId), 1)  // one at a time
            .distinctUntilChanged(MetricSnapshot::getChecksum)  // skip unchanged snapshots
            .map(snapshot -> ServerSentEvent.<MetricSnapshot>builder()
                .id(String.valueOf(snapshot.getTimestamp()))
                .event("metrics")
                .data(snapshot)
                .comment("app: " + appId)
                .build())
            .doOnCancel(() -> metricsService.unregisterConsumer(appId))
            .doOnError(e -> log.error("Stream error for {}: {}", appId, e.getMessage()));
    }
    
    // Also support WebSocket for bidirectional
    @MessageMapping("/metrics/{appId}")
    public Flux<MetricSnapshot> wsMetrics(@DestinationVariable String appId) {
        return metricsService.streamForApp(appId)
            .share();  // hot publisher: share stream among multiple WebSocket subscribers
    }
}
```

### Why WebFlux
Each SSE connection is long-lived (minutes to hours). With 10,000 concurrent operator dashboards, MVC would need 10,000 threads. WebFlux handles this with ~32 threads.

---

## Scenario 3: Event-Driven Order Processing

### Context
An e-commerce order processing pipeline consuming from Kafka, enriching each event with data from multiple services, and persisting results.

```java
@Service
@RequiredArgsConstructor
public class OrderProcessor {

    private final ReactiveKafkaConsumer<String, OrderEvent> consumer;
    private final ProductService productService;
    private final UserService userService;
    private final OrderRepository orderRepository;

    @PostConstruct
    public void startProcessing() {
        consumer.receive()
            .doOnNext(record -> log.debug("Received order: {}", record.value().orderId()))
            .flatMap(this::processOrderRecord, 20)  // 20 concurrent orders
            .doOnError(e -> log.error("Processing error", e))
            .retryWhen(Retry.backoff(Long.MAX_VALUE, Duration.ofSeconds(1))
                .maxBackoff(Duration.ofMinutes(5)))
            .subscribe();
    }

    private Mono<Void> processOrderRecord(ReceiverRecord<String, OrderEvent> record) {
        OrderEvent event = record.value();
        
        return Mono.zip(
                productService.getProduct(event.productId()),
                userService.getUser(event.userId())
            )
            .flatMap(tuple -> {
                Order order = Order.builder()
                    .id(event.orderId())
                    .product(tuple.getT1())
                    .user(tuple.getT2())
                    .quantity(event.quantity())
                    .status(OrderStatus.CREATED)
                    .build();
                return orderRepository.save(order);
            })
            .then(Mono.fromRunnable(record.receiverOffset()::acknowledge))
            .timeout(Duration.ofSeconds(30))
            .onErrorResume(e -> {
                log.error("Failed to process order {}: {}", event.orderId(), e.getMessage());
                return record.receiverOffset().acknowledge();  // commit offset to avoid reprocessing
            });
    }
}
```

---

# 12. Comparisons & Trade-offs

## WebFlux vs Spring MVC

| Aspect | Spring MVC | Spring WebFlux |
|---|---|---|
| **Best for** | CRUD, moderate load | High concurrency, streaming, microservice aggregation |
| **Thread model** | Thread-per-request | Event loop + worker threads |
| **Max concurrency** | ~200–500 (pool size) | Thousands (memory limited) |
| **Stack traces** | Clear, sequential | Complex, async chains |
| **Debugging** | Easy | Hard (requires Hooks.onOperatorDebug) |
| **Ecosystem** | Complete | Growing (most popular libs support it) |
| **JPA/Hibernate** | First-class | Not supported — use R2DBC |
| **Testing** | MockMvc | WebTestClient + StepVerifier |
| **Team ramp-up** | Days | Weeks to months |
| **Error handling** | Try/catch | `onErrorResume`, `onErrorReturn` — different mental model |
| **Transaction support** | `@Transactional` everywhere | R2DBC `@Transactional` — not all cases supported |

## Reactive vs Imperative

| Aspect | Imperative | Reactive |
|---|---|---|
| Code style | Sequential, easy to read | Declarative pipelines — harder to read for beginners |
| Mental model | "Do this, then do that" | "When this happens, then transform" |
| Debugging | Line-by-line in IDE | Requires specialized tools |
| Error handling | Try/catch — familiar | Operator-based — powerful but different |
| Backpressure | Manual (threads/queues) | Built-in, first-class |
| Testing | Unit test methods | StepVerifier, virtual time |

## When WebFlux Is a Bad Choice

1. **CPU-bound workloads**: Reactive adds overhead without benefit. If your request handler does heavy computation (image processing, PDF generation, ML inference), threads are busy anyway. Use Spring MVC with async processing.

2. **Simple CRUD with PostgreSQL + JPA**: If you're using JPA/Hibernate, you can't be fully reactive. Wrapping JDBC in `boundedElastic` gives you reactive syntax with imperative reality — the worst of both worlds.

3. **Small teams or startup speed**: The reactive learning curve steals weeks from feature development. Correctness bugs (missed subscriptions, blocking calls) are subtle and costly to debug.

4. **Existing large MVC codebase**: Migrating requires changing every layer — controllers, services, repositories, tests. The ROI rarely justifies it unless you have a specific measured performance problem.

5. **Libraries without reactive support**: If a key dependency (payment SDK, internal library, analytics framework) is blocking-only, you'll be fighting the paradigm. The `boundedElastic` workaround works but negates reactive benefits.

6. **When you need strong transactional consistency across multiple operations**: Reactive transactions with R2DBC are limited compared to JPA. Complex transaction patterns are harder to express reactively.

---

# 13. Production-Ready Checklist

**You are production-ready with Spring WebFlux if you can:**

## Reactive Fundamentals
- [ ] Explain why reactive programming exists and what problem it solves (blocking I/O at scale)
- [ ] Explain the Publisher/Subscriber contract and what `request(n)` means for backpressure
- [ ] Distinguish cold vs hot publishers and know when each is used
- [ ] Explain why nothing executes until `subscribe()` is called
- [ ] Draw the event loop model and explain why blocking it is catastrophic

## Project Reactor
- [ ] Know when to use `Mono` vs `Flux`
- [ ] Know when to use `map` vs `flatMap` (sync vs async transformation)
- [ ] Know when to use `flatMap` vs `concatMap` (concurrent vs ordered sequential)
- [ ] Use `switchIfEmpty` and `defaultIfEmpty` correctly for empty stream handling
- [ ] Implement proper error handling with `onErrorReturn`, `onErrorResume`, `doOnError`
- [ ] Use `Mono.zip` / `Flux.zip` for parallel independent operations
- [ ] Use `timeout` with fallbacks for external calls
- [ ] Never call `.block()` inside a reactive pipeline
- [ ] Use `Mono.defer` when the Mono creation itself must be lazy

## Threading
- [ ] Explain the difference between `subscribeOn` and `publishOn`
- [ ] Use `subscribeOn(Schedulers.boundedElastic())` for blocking calls
- [ ] Never block a `Schedulers.parallel()` thread
- [ ] Propagate request context via Reactor `Context`, not `ThreadLocal`
- [ ] Detect blocking calls with `BlockHound` in tests

## WebClient
- [ ] Build a production `WebClient` with timeouts, connection pool, and error handling
- [ ] Know when to use `retrieve` vs `exchangeToMono`
- [ ] Always consume the response body when using `exchangeToMono`
- [ ] Implement retry with exponential backoff using `retryWhen`
- [ ] Map HTTP error status codes to domain exceptions via `onStatus`
- [ ] Propagate correlation IDs via `ExchangeFilterFunction`

## System Design & Production
- [ ] Identify whether a use case actually benefits from WebFlux (I/O-bound, high concurrency)
- [ ] Integrate blocking dependencies correctly (R2DBC preferred, `boundedElastic` fallback)
- [ ] Implement streaming endpoints with Server-Sent Events
- [ ] Handle backpressure explicitly for fast producers with `onBackpressureDrop`, `onBackpressureBuffer`
- [ ] Debug reactive pipelines with `checkpoint()`, `log()`, and `Hooks.onOperatorDebug()`
- [ ] Test reactive code with `StepVerifier` including virtual time for time-based operators
- [ ] Write `WebTestClient` integration tests

## Failure Scenarios — Know What Goes Wrong
- [ ] **Blocking on event loop**: Any blocking call on Netty thread freezes the loop. Symptoms: latency spikes, thread starvation. Fix: `subscribeOn(boundedElastic)`.
- [ ] **Missing subscription**: Returning `Mono<Void>` from a method but never subscribing — the pipeline never executes. In Spring WebFlux, the framework subscribes for you when you return a `Mono` from a handler. Outside the framework, you must subscribe.
- [ ] **Not consuming response body (WebClient)**: `exchangeToMono` without releasing body — connection pool exhaustion over time.
- [ ] **Unbounded flatMap concurrency**: `flatMap` with unlimited concurrency on 1000 items fires 1000 concurrent DB/HTTP calls — overwhelming downstream services.
- [ ] **No timeout on external calls**: Without `timeout()`, a slow external service causes your futures to pile up indefinitely — eventual OOM.
- [ ] **ThreadLocal loss after thread switch**: MDC, Spring Security context, etc. don't survive `subscribeOn`/`publishOn`. Use Reactor Context.
- [ ] **Infinite stream without backpressure**: `Flux.interval` producing faster than the subscriber can consume — eventual buffer overflow.
- [ ] **Cold vs hot confusion**: Publishing to a `Sinks.Many` before any subscriber — events lost. Use `replay` or ensure subscribers are connected before publishing.

---

*Last reviewed: 2024. Spring Boot 3.x, Project Reactor 3.6.x, Java 21.*
