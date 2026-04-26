# Apache Kafka with Java & Spring Boot: The Production-Grade Engineering Guide

> A deep, opinionated, real-world guide for backend engineers who want to understand Kafka both **internally** and **operationally**.
> Written for developers fluent in Java 17/21, Spring Boot, PostgreSQL, AWS, Docker, and Kubernetes with no prior deep Kafka experience.

---

## Table of Contents

1. [Event-Driven Architecture Fundamentals](#1-event-driven-architecture-fundamentals)
2. [Apache Kafka Fundamentals](#2-apache-kafka-fundamentals)
3. [Kafka Architecture & Internals](#3-kafka-architecture--internals)
4. [Message Lifecycle](#4-message-lifecycle)
5. [Kafka in System Design](#5-kafka-in-system-design)
6. [Kafka + Spring Boot](#6-kafka--spring-boot)
7. [Serialization & Data Formats](#7-serialization--data-formats)
8. [Delivery Semantics](#8-delivery-semantics)
9. [Advanced Kafka Concepts](#9-advanced-kafka-concepts)
10. [Error Handling & Reliability](#10-error-handling--reliability)
11. [Performance & Scaling](#11-performance--scaling)
12. [Kafka Streams](#12-kafka-streams)
13. [Monitoring & Observability](#13-monitoring--observability)
14. [Production Best Practices](#14-production-best-practices)
15. [Real-World Scenarios](#15-real-world-scenarios)
16. [Comparisons & Trade-offs](#16-comparisons--trade-offs)
17. [Final Checklist](#17-final-checklist)

---

## 1. Event-Driven Architecture Fundamentals

### 1.1 What Event-Driven Architecture Is

An **event-driven architecture (EDA)** is a style in which components communicate primarily by **producing and reacting to events** — *immutable records of something that has happened*.

In contrast to request/response (HTTP, gRPC), where service A **asks** service B to do something and waits, EDA has service A **announcing** that something occurred; whoever cares subscribes.

Mental model:
- Request/response: "Please ship order 42."
- Event-driven: "Order 42 has been placed." — shipping, billing, analytics all react independently.

Key properties:
- **Temporal decoupling**: producer and consumer need not be online at the same time.
- **Loose coupling**: the producer doesn't know (or care) who consumes.
- **Extensibility**: new consumers can subscribe without touching the producer.

### 1.2 When to Use EDA — and When Not To

**Use EDA when:**
- Multiple consumers react to the same occurrence (fan-out).
- Producer and consumer have different availability or throughput profiles.
- You need an auditable history of what happened.
- The workflow is naturally asynchronous (billing, notifications, analytics).
- You want to decouple deployment lifecycles across teams/services.

**Avoid EDA when:**
- The caller genuinely needs a **synchronous answer** (e.g., auth checks, price lookups).
- Strong **transactional consistency** across components is required.
- Your system is small and request/response is sufficient — EDA adds operational complexity.
- Debuggability is a priority and your team has no experience with async flows.

**Rule of thumb:** reach for events when *more than one thing* should happen in response to a business fact.

### 1.3 Events vs Messages

Though often used interchangeably, the distinction matters:

| | Event | Message / Command |
|---|---|---|
| Intent | A **fact**: "something happened" | An **instruction**: "please do X" |
| Direction | Fan-out (many consumers possible) | Usually one consumer |
| Tense | Past (`OrderPlaced`) | Imperative (`PlaceOrder`) |
| Ownership | Producer describes their own reality | Consumer is the authority |
| Example | `PaymentSucceeded` | `ChargeCustomer` |

Kafka handles both well, but **events dominate** — Kafka's append-only log is a natural fit for facts.

### 1.4 Asynchronous Communication

Async means the sender is **not blocked** waiting for the receiver. The sender writes to a log; a consumer processes later. The consequences:

- **Latency** in the pipeline is bounded by the slowest consumer, not the fastest.
- **Backpressure** must be handled explicitly — consumers can fall behind.
- **Order** must be considered — in what order do consumers see events?
- **Idempotency** is a must — events can be redelivered.

### 1.5 Benefits and Trade-offs

| Benefit | Trade-off |
|---|---|
| Services can evolve independently | Harder to reason about end-to-end behavior |
| Fan-out is cheap | Debugging requires distributed tracing |
| Failure isolation (a slow consumer doesn't block producers) | Eventual consistency is the default |
| Natural audit log | Schema evolution across many consumers is hard |
| Replay/reprocessing is possible | Delivery guarantees require care |
| Asymmetric scaling of producer/consumer | Operational complexity (broker ops, monitoring) |

---

## 2. Apache Kafka Fundamentals

### 2.1 What Kafka Is

Apache Kafka is a **distributed, partitioned, replicated commit log**, exposed through producer, consumer, and stream-processing APIs. Originally built at LinkedIn (2010) to solve log aggregation at massive scale, it is now the de-facto backbone of most streaming platforms.

Kafka's defining traits:
- **Append-only log per partition** — messages are immutable, kept in order.
- **Durable by design** — replicated across brokers before being acknowledged.
- **High throughput** — millions of events/sec on modest hardware, thanks to sequential I/O and zero-copy.
- **Consumer-driven** — consumers track their own position (offset); Kafka doesn't push.
- **Horizontally scalable** — add brokers; add partitions; add consumer instances.

Kafka is **not** a queue in the RabbitMQ/ActiveMQ sense — messages are not deleted on consumption. Multiple consumer groups read the same data independently, each maintaining its own offset.

### 2.2 Core Use Cases

| Use Case | Why Kafka Fits |
|---|---|
| **Messaging** between services | Durable, ordered per partition, multi-consumer |
| **Event streaming** (pipeline of events) | Long retention, replay, exactly-once via transactions |
| **Log aggregation** (app/server logs → storage) | Massive throughput, cheap durability |
| **Activity/metric ingestion** | Clickstreams, sensor data, telemetry |
| **CDC (change data capture)** | Debezium captures DB changes → Kafka → consumers |
| **Stream processing** | Kafka Streams / Flink process in-flight |
| **Event sourcing** | Kafka is the append-only store |

### 2.3 Kafka vs Traditional Message Brokers

| Aspect | Traditional Broker (RabbitMQ / ActiveMQ) | Kafka |
|---|---|---|
| Storage | Temporary; deleted on ack | Durable log; retention-based |
| Consumer model | Push; broker routes | Pull; consumer seeks offsets |
| Ordering | Per-queue (limited) | Per-partition, strong |
| Throughput | Tens–hundreds of k msg/s | Millions/s |
| Replay | Rare / not native | First-class |
| Routing | Rich (exchanges, headers) | Simple (topic + key) |
| Latency | Very low (sub-ms) | Low (single-digit ms) |
| Fan-out | Per-queue consumer | Each consumer group reads the full log independently |
| Best for | Command dispatch, RPC-like flows | Event logs, streaming pipelines |

### 2.4 When to Use vs NOT Use Kafka

**Use Kafka when:**
- You need **durable, replayable event streams**.
- **Multiple consumers** independently consume the same data.
- Throughput is or will be high (10k+ msg/s sustained).
- You're building data pipelines, analytics, or event-sourced systems.
- Consumers can fall behind without losing data.

**Avoid Kafka when:**
- You need **low-latency, low-volume task queues** (RabbitMQ is simpler).
- You need **per-message routing rules** or complex topologies.
- You want **priority queues** — Kafka has no native priority.
- You need **short-lived, point-to-point RPC** — use HTTP/gRPC.
- Your team cannot operate distributed infrastructure and managed Kafka (MSK, Confluent Cloud) isn't an option.
- Your volume is low (< a few hundred msg/s) and you'd be adding Kafka purely for buzzword-compliance.

---

## 3. Kafka Architecture & Internals

### 3.1 Topics

A **topic** is a named stream of records — like a table, but append-only.

```
Topic: orders
├── Partition 0: [msg0, msg1, msg2, ...]
├── Partition 1: [msg0, msg1, ...]
└── Partition 2: [msg0, msg1, msg2, msg3, ...]
```

Topics are logical; partitions are the unit of storage, ordering, and parallelism.

### 3.2 Partitions

A **partition** is an ordered, immutable log. It is the **unit of parallelism** in Kafka.

- Each partition lives on **one leader broker** plus replicas.
- Messages have a 64-bit offset assigned at append time.
- Order is guaranteed **within a partition**, **not across partitions**.
- A consumer instance reads from one or more partitions; a partition is read by at most one consumer in a group.

**Why partitions matter:**
- Scaling: more partitions → more parallel consumers.
- Ordering: a key routes related events to the same partition, preserving order for that key.
- Retention: old segments of each partition are reaped based on time/size.

### 3.3 Brokers

A **broker** is a Kafka server process. A cluster is a set of brokers.

- Each partition replica lives on one broker.
- One broker is elected **controller** (coordinates metadata operations).
- Brokers form a cluster via **KRaft** (modern, Raft-based) or older ZooKeeper (deprecated since Kafka 3.x, removed in 4.x).

Broker responsibilities:
- Host partition replicas.
- Handle produce/fetch requests.
- Replicate data.
- Participate in leader election.

### 3.4 Producers

A **producer** is a client that publishes records to topics. Producers:
- Decide the partition (by key hash, round-robin, custom).
- Buffer and batch records for throughput.
- Send asynchronously with optional `acks` (`0`, `1`, `all`).
- Can be **idempotent** (no duplicates within a session) and/or **transactional** (atomic multi-partition writes).

### 3.5 Consumers

A **consumer** pulls records from partitions. Consumers:
- Belong to a **consumer group**.
- Are assigned a subset of partitions by the broker coordinator.
- Commit **offsets** (positions) to track progress.
- Periodically send heartbeats; a slow or dead consumer triggers **rebalance**.

### 3.6 Consumer Groups

A **consumer group** is a set of consumer instances coordinating to read a topic.

- Each partition is read by **exactly one** consumer in a group at a time.
- Adding consumers up to the partition count scales read throughput.
- Adding more consumers than partitions leaves some idle.
- Each group maintains its own offset — multiple groups read the same topic independently.

```
Topic: orders (4 partitions)
Group A (shipping):
  consumer-1 → P0, P1
  consumer-2 → P2, P3
Group B (analytics):
  consumer-1 → P0, P1, P2, P3
```

### 3.7 Offsets

An **offset** is a record's position in its partition.

- Produced-side: assigned when the leader appends.
- Consumer-side: the position of the *next* record to read.
- Consumers commit offsets via `__consumer_offsets` (an internal topic).
- **Auto-commit** is convenient but risky; manual commit is production-grade.

### 3.8 Replication: Leaders & Followers

Each partition has:
- **1 leader** that accepts writes and reads.
- **N-1 followers** that replicate from the leader.
- An **ISR** (in-sync replicas) set — replicas caught up within a threshold.

On leader failure, the controller elects a new leader from the ISR. If no ISR is available and `unclean.leader.election.enable=true`, a lagging replica can be elected — this risks data loss; typically **disabled in prod**.

Replication factor (`RF`):
- RF=1: no redundancy.
- RF=3: standard — tolerates 1 broker failure safely, 2 with `min.insync.replicas=2`.
- `min.insync.replicas` controls how many replicas must acknowledge before `acks=all` returns.

### 3.9 How Data Flows (High Level)

```
Producer ──(batch, key, value)──▶ Leader broker for partition P
                                       │
                                       ├── writes to local log (page cache)
                                       ├── replicates to followers
                                       │
                                       ▼
                                  acks to producer
                                       │
Consumer ──(fetch offset N)──▶ Leader broker for partition P
                                       │
                                       ▼
                                  returns batch starting at N
```

### 3.10 Scalability and Durability Mechanisms

- **Scalability**: partitioning + consumer groups + horizontal broker scaling.
- **Durability**: replication + `acks=all` + `min.insync.replicas` + fsync behavior (tunable).
- **Performance**: sequential disk I/O, OS page cache, `sendfile()` zero-copy, batching, compression (gzip/lz4/snappy/zstd).

---

## 4. Message Lifecycle

### 4.1 Step 1 — Producing

1. Producer creates a `ProducerRecord(topic, key, value, headers)`.
2. Serializers convert key and value to bytes.
3. The **partitioner** picks a partition:
   - If `partition` is set explicitly → use it.
   - Else if key is set → `hash(key) % numPartitions` (sticky, consistent for the key).
   - Else → **sticky partitioner** batches to the same partition until full, then rotates (better than pure round-robin for throughput).
4. Record is appended to an internal **batch** for that partition.
5. Once batch fills (`batch.size`) or `linger.ms` elapses, the batch is sent to the leader.
6. Producer waits for the configured `acks`.

### 4.2 Step 2 — Partition Assignment

Given key `K`, target partition `P = hash(K) % numPartitions`.

**Consequences:**
- Adding partitions later remaps keys — old keys may split across partitions; ordering guarantees break for historical data.
- Choose partition count carefully up front.

### 4.3 Step 3 — Replication

1. Leader writes to its local log.
2. Followers fetch from leader continuously.
3. Once followers acknowledge, message is marked as committed.
4. Only **committed** messages are visible to consumers.
5. With `acks=all`, the producer waits until `min.insync.replicas` have replicated.

### 4.4 Step 4 — Consuming

1. Consumer subscribes to topic(s).
2. Coordinator assigns partitions (group rebalance).
3. Consumer fetches batches from leaders for its partitions.
4. Deserializers convert bytes to objects.
5. Application processes records.
6. Consumer commits offsets (auto or manual).

### 4.5 Step 5 — Offset Commit

- **Auto-commit** (`enable.auto.commit=true`) commits every `auto.commit.interval.ms`. Can lose or duplicate records on crash.
- **Manual commit** (`commitSync` / `commitAsync`):
  - Commit **after** successful processing → at-least-once.
  - Commit **before** processing → at-most-once.
- For at-least-once: always commit after successful processing; design for idempotency.

### 4.6 Failure Scenarios

| Failure | Effect | Mitigation |
|---|---|---|
| Producer crashes mid-send | Unacked records lost (unless retried) | `acks=all`, `enable.idempotence=true`, retries |
| Leader broker dies | Followers elect new leader | `min.insync.replicas=2`; disable unclean leader election |
| Consumer crashes after processing, before commit | Record re-delivered | Idempotent consumers |
| Consumer crashes after commit, before processing | Record lost | Commit after, not before |
| Slow consumer | Lag grows; eventually missed by retention | Scale consumers, alert on lag |
| Network partition | Followers fall out of ISR; writes may block | `acks=all` with proper RF handles this |
| Disk full on broker | Partition goes offline | Monitoring + quotas |
| Poison message (unparseable) | Loop; blocks partition | Dead Letter Topic |

---

## 5. Kafka in System Design

### 5.1 Decoupling Services

```
[Order Service] ──OrderPlaced──▶  Kafka  ──▶ [Shipping]
                                         ──▶ [Billing]
                                         ──▶ [Analytics]
                                         ──▶ [Notifications]
```

The order service has **no knowledge** of downstream consumers. Adding Notifications doesn't require touching Order.

### 5.2 Event Sourcing (Overview)

Event sourcing: the **source of truth** is a log of events. Current state is a projection.

- Every state change is an event, appended to Kafka.
- Projections (materialized views) are rebuilt by replaying events.
- Kafka's durable log + compacted topics is a natural fit.

**Pros:** full audit, temporal queries, rebuildable views, "why did this happen?" is always answerable.
**Cons:** schema evolution is hard, replay costs grow, querying requires materialized views.

**When:** strongly auditable domains (finance, healthcare), complex derivable views, versioned state history.
**When NOT:** simple CRUD — it's massive overkill.

### 5.3 CQRS (Overview)

**Command Query Responsibility Segregation:** split the model.

- **Command side**: writes → emit events to Kafka.
- **Query side**: consumers build read-optimized stores (Elasticsearch, Redis, denormalized SQL).

Kafka connects the two.

**Pros:** read side can be reshaped freely; each side scales independently; natural async.
**Cons:** eventual consistency, more infra, higher cognitive load.

### 5.4 Async Processing

Offload slow, bursty, or non-critical work:

```
[Web API] → accepts request → writes Kafka event → returns 202
                                                     │
                                                     ▼
                                        [Worker] processes event
```

Good for: sending emails, generating reports, image processing, integrating with 3rd-party APIs, webhooks.

### 5.5 Failure Handling

| Design consideration | Approach |
|---|---|
| Consumer can't process | Retry → DLT |
| Producer outage mid-transaction | Outbox pattern (write DB + event in one TX) |
| Ordering required across aggregates | Key by aggregate ID → single partition |
| Replay needed | Long retention; or compacted topic for "latest" state |
| Cross-service consistency | Sagas over events |

---

## 6. Kafka + Spring Boot

### 6.1 Running a Kafka Broker Locally (Docker)

`docker-compose.yml` — **KRaft** mode (no ZooKeeper):

```yaml
version: "3.9"
services:
  kafka:
    image: bitnami/kafka:3.7
    ports:
      - "9092:9092"
      - "9094:9094"
    environment:
      KAFKA_CFG_NODE_ID: 1
      KAFKA_CFG_PROCESS_ROLES: broker,controller
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094
      KAFKA_CFG_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094
      KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,EXTERNAL:PLAINTEXT
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE: "false"
      KAFKA_KRAFT_CLUSTER_ID: abcdefghijk1234567890
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    ports: ["8080:8080"]
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:9092
```

```bash
docker compose up -d
```

### 6.2 Dependencies

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'org.springframework.kafka:spring-kafka'
    // Optional
    implementation 'org.apache.kafka:kafka-streams'
    testImplementation 'org.springframework.kafka:spring-kafka-test'
    testImplementation 'org.testcontainers:kafka'
}
```

### 6.3 Configuration (`application.yml`)

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9094
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all
      properties:
        enable.idempotence: true
        max.in.flight.requests.per.connection: 5
        compression.type: lz4
        linger.ms: 20
        batch.size: 65536
    consumer:
      group-id: order-service
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      enable-auto-commit: false
      auto-offset-reset: earliest
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: com.acme.events
        isolation.level: read_committed
    listener:
      ack-mode: manual
      concurrency: 3
      missing-topics-fatal: false
```

**Key knobs explained:**
- `acks=all` + `enable.idempotence=true` → strong durability, no duplicates within a producer session.
- `compression.type=lz4` → excellent compression/CPU ratio.
- `linger.ms=20` → small latency trade for much bigger batches = higher throughput.
- `ErrorHandlingDeserializer` → deserialization errors don't crash the consumer.
- `ack-mode: manual` → you commit offsets explicitly.
- `concurrency: 3` → 3 listener threads per `@KafkaListener` (up to partition count).
- `isolation.level: read_committed` → consumers only see committed transactional messages.

### 6.4 Declaring Topics

```java
@Configuration
public class KafkaTopics {

    @Bean
    NewTopic ordersTopic() {
        return TopicBuilder.name("orders.v1")
            .partitions(12)
            .replicas(3)
            .config(TopicConfig.MIN_IN_SYNC_REPLICAS_CONFIG, "2")
            .config(TopicConfig.RETENTION_MS_CONFIG, String.valueOf(Duration.ofDays(7).toMillis()))
            .config(TopicConfig.CLEANUP_POLICY_CONFIG, TopicConfig.CLEANUP_POLICY_DELETE)
            .build();
    }

    @Bean
    NewTopic ordersDlt() {
        return TopicBuilder.name("orders.v1.DLT")
            .partitions(12)
            .replicas(3)
            .config(TopicConfig.RETENTION_MS_CONFIG, String.valueOf(Duration.ofDays(14).toMillis()))
            .build();
    }
}
```

### 6.5 Example 1 — Simple Producer

```java
public record OrderPlaced(String orderId, String customerId, BigDecimal total, Instant placedAt) {}

@Service
public class OrderEventPublisher {

    private final KafkaTemplate<String, Object> kafka;

    public OrderEventPublisher(KafkaTemplate<String, Object> kafka) {
        this.kafka = kafka;
    }

    public CompletableFuture<SendResult<String, Object>> publish(OrderPlaced event) {
        // Key by orderId → all events for this order land on the same partition
        ProducerRecord<String, Object> record = new ProducerRecord<>(
            "orders.v1", event.orderId(), event);
        record.headers().add("event-type", "OrderPlaced".getBytes(StandardCharsets.UTF_8));
        record.headers().add("event-version", "1".getBytes(StandardCharsets.UTF_8));

        return kafka.send(record).whenComplete((res, ex) -> {
            if (ex != null) {
                log.error("Failed to publish OrderPlaced for {}", event.orderId(), ex);
            } else {
                log.info("Published OrderPlaced order={} partition={} offset={}",
                    event.orderId(),
                    res.getRecordMetadata().partition(),
                    res.getRecordMetadata().offset());
            }
        });
    }
}
```

**Key decisions:**
- **Key on business identity** (`orderId`) — preserves per-order ordering.
- **Headers** carry metadata (event type, version, correlation ID) — avoids polluting the payload.
- `KafkaTemplate.send()` returns a `CompletableFuture<SendResult>` — handle success/failure.

### 6.6 Example 2 — Simple Consumer

```java
@Component
public class OrderEventConsumer {

    @KafkaListener(topics = "orders.v1", groupId = "shipping-service")
    public void onOrderPlaced(
            @Payload OrderPlaced event,
            @Header(KafkaHeaders.RECEIVED_KEY) String key,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        log.info("Received order={} partition={} offset={}", key, partition, offset);
        try {
            shippingService.schedule(event);
            ack.acknowledge();
        } catch (TransientException e) {
            // Don't ack — let the error handler retry
            throw e;
        } catch (PermanentException e) {
            // Send to DLT via error handler
            throw e;
        }
    }
}
```

**Manual ack** is the production default. Auto-commit loses or duplicates records on crashes.

### 6.7 Example 3 — Consumer Groups

Two Spring Boot apps, both consuming `orders.v1`, but with **different group IDs**:

- `shipping-service` → ships orders.
- `analytics-service` → tracks metrics.

Each group reads the full stream independently. Within a group, partitions are split among instances.

```java
// In shipping-service
@KafkaListener(topics = "orders.v1", groupId = "shipping-service", concurrency = "6")
public void handle(OrderPlaced e, Acknowledgment ack) { ... }

// In analytics-service
@KafkaListener(topics = "orders.v1", groupId = "analytics-service", concurrency = "3")
public void handle(OrderPlaced e, Acknowledgment ack) { ... }
```

`concurrency` = number of listener threads in this JVM. To truly scale, run multiple instances of the service.

### 6.8 Example 4 — JSON Messages

```yaml
spring:
  kafka:
    producer:
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
    consumer:
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
      properties:
        spring.deserializer.value.delegate.class: org.springframework.kafka.support.serializer.JsonDeserializer
        spring.json.trusted.packages: com.acme.events
        spring.json.value.default.type: com.acme.events.OrderPlaced
        spring.json.use.type.headers: false
```

**Gotchas:**
- `JsonDeserializer` adds `__TypeId__` headers by default — avoid in cross-language setups (`use.type.headers=false`).
- Always set `spring.json.trusted.packages` in production — security hardening.
- Consider Avro + Schema Registry (see §7) for stronger schema evolution.

### 6.9 Example 5 — Error Handling

```java
@Configuration
public class KafkaErrorHandlingConfig {

    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, Object> template) {
        DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(
            template,
            (rec, ex) -> new TopicPartition(rec.topic() + ".DLT", rec.partition())
        );

        ExponentialBackOffWithMaxRetries backOff = new ExponentialBackOffWithMaxRetries(5);
        backOff.setInitialInterval(500L);
        backOff.setMultiplier(2.0);
        backOff.setMaxInterval(10_000L);

        DefaultErrorHandler handler = new DefaultErrorHandler(recoverer, backOff);

        // Don't retry these — go to DLT immediately
        handler.addNotRetryableExceptions(
            IllegalArgumentException.class,
            DeserializationException.class,
            MethodArgumentResolutionException.class
        );

        return handler;
    }

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, Object> factory(
            ConsumerFactory<String, Object> cf,
            DefaultErrorHandler errorHandler) {
        ConcurrentKafkaListenerContainerFactory<String, Object> f =
            new ConcurrentKafkaListenerContainerFactory<>();
        f.setConsumerFactory(cf);
        f.setCommonErrorHandler(errorHandler);
        f.getContainerProperties().setAckMode(ContainerProperties.AckMode.MANUAL);
        return f;
    }
}
```

**Behavior:**
- Transient exceptions → retry with exponential backoff (5 attempts).
- After exhaustion → publish to `orders.v1.DLT`.
- `DeserializationException` → straight to DLT (no point retrying unparseable bytes).

### 6.10 Example 6 — Retry Mechanisms

Two main patterns:

**(a) In-process retry** (blocking — `DefaultErrorHandler` as above). Simple; blocks the partition during retry.

**(b) Non-blocking retry topics** (`@RetryableTopic` — Spring Kafka 2.7+):

```java
@RetryableTopic(
    attempts = "5",
    backoff = @Backoff(delay = 1000, multiplier = 2.0, maxDelay = 30_000),
    autoCreateTopics = "true",
    topicSuffixingStrategy = SUFFIX_WITH_INDEX_VALUE,
    dltStrategy = FAIL_ON_ERROR,
    include = { TransientException.class }
)
@KafkaListener(topics = "orders.v1", groupId = "shipping-service")
public void handle(OrderPlaced event, Acknowledgment ack) {
    shippingService.schedule(event);
    ack.acknowledge();
}

@DltHandler
public void handleDlt(OrderPlaced failed,
                      @Header(KafkaHeaders.EXCEPTION_MESSAGE) String reason) {
    alerts.notify("Failed order " + failed.orderId() + ": " + reason);
}
```

Spring creates topics `orders.v1-retry-0`, `orders.v1-retry-1`, ..., `orders.v1-dlt`. Failed messages move forward through retry topics with delays — the main consumer is not blocked.

**Choose non-blocking retries when:** partition head-of-line blocking is unacceptable and ordering across keys is not strictly required per-retry-stage.

### 6.11 Example 7 — Dead Letter Topics

A **DLT** is a normal topic for messages that couldn't be processed.

- **What to put there**: the failed payload + headers with the original topic, partition, offset, exception class and message, timestamp.
- **What to do with it**: alert, inspect, replay after fix, or discard with reason.

`DeadLetterPublishingRecoverer` attaches those headers automatically. A dedicated DLT consumer can:
- Alert on volume spikes.
- Display in an ops dashboard.
- Allow manual replay (copy record back to the main topic).

```java
@KafkaListener(topics = "orders.v1.DLT", groupId = "dlt-monitor")
public void onDlt(OrderPlaced failed,
                  @Header(name = KafkaHeaders.DLT_ORIGINAL_TOPIC) String origTopic,
                  @Header(name = KafkaHeaders.DLT_EXCEPTION_MESSAGE) String cause) {
    dltStore.record(origTopic, failed, cause);
    alertService.notifyOps(failed.orderId(), cause);
}
```

---

## 7. Serialization & Data Formats

### 7.1 String / Plain Text

- **Pros:** simplest; always readable.
- **Cons:** no schema; parsing pushed to the consumer; no evolution story.
- **Use for:** logs, one-off events, quick prototypes.

### 7.2 JSON

- **Pros:** human-readable, ubiquitous, Jackson is fast.
- **Cons:** no schema enforcement, larger on wire, ambiguous types (numbers, dates).
- **Use for:** internal events with moderate volume and a small number of producers/consumers.

Production tip: include an explicit `event-type` and `event-version` header; never rely on polymorphic type hints in the payload.

### 7.3 Avro (with Schema Registry)

- **Pros:** compact binary, strong schema, **backward/forward compatibility rules**, multi-language.
- **Cons:** requires Schema Registry; schema authoring is a team process.
- **Use for:** inter-service contracts with stable evolution needs.

Example producer config:
```yaml
spring:
  kafka:
    producer:
      value-serializer: io.confluent.kafka.serializers.KafkaAvroSerializer
      properties:
        schema.registry.url: http://schema-registry:8081
```

### 7.4 Protobuf

- **Pros:** compact, strict typing, excellent tooling, widely supported.
- **Cons:** schema management workflow needed; less common in Kafka ecosystems than Avro.

### 7.5 Schema Registry (Overview)

A **Schema Registry** (Confluent, Apicurio) stores schemas and versions them per **subject** (typically `topic-value`, `topic-key`).

It enforces compatibility rules on write:
- **BACKWARD**: new schema can read old data (default; safe for consumers).
- **FORWARD**: old schema can read new data (safe for producers).
- **FULL**: both.
- **NONE**: no checks.

Why it matters: in a large org with many services, breaking schema changes slip in unnoticed. The registry rejects them at deploy.

### 7.6 Choosing

| Format | When |
|---|---|
| String | Logs, plaintext events |
| JSON | Small teams, human-friendly dev ops, low overhead |
| Avro | Multi-team, schema evolution matters, volume is meaningful |
| Protobuf | gRPC-adjacent orgs, existing Protobuf investment |

---

## 8. Delivery Semantics

### 8.1 At-Most-Once

Each message is delivered 0 or 1 times. Data loss possible.

Achieved by:
- Committing offset **before** processing.
- Not retrying on producer failure.

**Use when:** metrics, telemetry where occasional loss is acceptable and duplicates are worse.

### 8.2 At-Least-Once (Default)

Each message is delivered ≥1 time. Duplicates possible.

Achieved by:
- Producer retries on send failure.
- Consumer commits offset **after** successful processing.

**Use when:** most business workloads — pair with idempotent consumers.

### 8.3 Exactly-Once

Each message is effectively processed once. Achieved via:

**Producer side:**
- `enable.idempotence=true` — no duplicates within a single producer session (handled via sequence numbers per partition).

**End-to-end (Kafka-to-Kafka):**
- **Transactional producers**: `transactional.id` set, `initTransactions()`, `beginTransaction()`, send + `sendOffsetsToTransaction()` + `commitTransaction()`.
- Consumers with `isolation.level=read_committed` see only committed transactional messages.

**Kafka-to-External (e.g., DB):**
- Kafka's EOS only guarantees within Kafka. To extend it to an external system, either:
  - **Idempotent writes** on the external side (use event ID as dedupe key).
  - **Outbox pattern**: write to DB and outbox table in one transaction; a relay publishes to Kafka.
  - **Transactional sinks** (e.g., Kafka Connect sinks that support EOS).

### 8.4 Trade-offs

| Semantic | Throughput | Complexity | Use Case |
|---|---|---|---|
| At-most-once | Highest | Low | Metrics, telemetry |
| At-least-once | High | Low–Medium | Most business logic |
| Exactly-once (Kafka-to-Kafka) | Medium (20–30% lower) | Medium | Stream processing |
| End-to-end EOS | Medium | High | Financial transactions, billing |

**Practical advice:** pursue at-least-once + idempotent consumers. "True" exactly-once across arbitrary systems is usually engineered through idempotency, not protocols.

---

## 9. Advanced Kafka Concepts

### 9.1 Idempotent Producers

`enable.idempotence=true` makes the producer:
- Assign a **producer ID** (PID) and monotonic sequence numbers per partition.
- Brokers detect duplicates and drop them.
- Retries no longer cause duplicates **within the session**.

Requirements (auto-set when enabled): `acks=all`, `retries>0`, `max.in.flight.requests.per.connection≤5`.

### 9.2 Transactions

```java
@Bean
public KafkaTransactionManager<String, Object> txManager(ProducerFactory<String, Object> pf) {
    return new KafkaTransactionManager<>(pf);
}

@Transactional("txManager")
public void placeOrderAndEmit(Order o) {
    orderRepo.save(o);                          // DB write
    kafka.send("orders.v1", o.getId(), event);  // Kafka publish
    // Atomic: both commit or both rollback (via ChainedTransactionManager)
}
```

**Limitations:**
- DB + Kafka atomicity requires `ChainedKafkaTransactionManager` — but not truly XA; it's a best-effort chain. For guaranteed atomicity, **use the outbox pattern**.
- Transactions have a throughput cost.

### 9.3 Partitioning Strategies

| Strategy | When |
|---|---|
| **Key-hash** (default) | Preserve order per key |
| **Round-robin / sticky** (no key) | Maximize throughput, no ordering need |
| **Custom partitioner** | Skew control, tenant isolation, geography |

**Skew:** if one key dominates (one huge customer), its partition becomes a hotspot. Mitigations:
- Add a sub-key salt.
- Split logically (per-region topics).
- Use custom partitioner with balanced buckets.

### 9.4 Rebalancing

When a consumer joins/leaves a group, the coordinator reassigns partitions. Issues:
- **Stop-the-world**: all consumers pause during classic rebalance.
- Long rebalances during deploys cause lag.

Modern strategies (configure via `partition.assignment.strategy`):
- **RangeAssignor** — default, simple but can be uneven.
- **RoundRobinAssignor** — balanced.
- **StickyAssignor** — minimizes churn.
- **CooperativeStickyAssignor** — **incremental rebalance**, no stop-the-world; strongly recommended.

Tune:
- `session.timeout.ms` (default 45s) — consumer considered dead.
- `max.poll.interval.ms` (default 5m) — max time between polls; long processing can trigger rebalance.

### 9.5 Compaction vs Retention

**Retention policies:**
- `delete` (default): remove messages older than `retention.ms` / larger than `retention.bytes`.
- `compact`: keep at least the latest value for each key.
- `compact,delete`: both.

**Compaction** enables Kafka as a source of truth for latest state:
```
Key=user-42 → v1, v2, v3  →  after compaction → v3
```

**Use compaction when:** topic represents the latest state per key (e.g., a CDC topic of user profiles). Combined with a tombstone (null value) to "delete" a key.

**Don't compact when:** topic is a pure event log — you lose history.

---

## 10. Error Handling & Reliability

### 10.1 Classify Errors

| Class | Example | Action |
|---|---|---|
| Transient | Timeout, 503 from downstream | Retry with backoff |
| Permanent | Validation error, schema mismatch | To DLT immediately |
| Poison | Unparseable payload | To DLT, don't loop |
| Business | "Order already shipped" | Discard or to DLT — not really an error |

### 10.2 Retry Strategies

- **In-process retry** (blocking): simple, but blocks the partition. Good for short transient failures.
- **Non-blocking retry topics** (`@RetryableTopic`): great for longer waits or variable downstream SLAs.
- **Circuit breaker** (Resilience4j) around downstream calls: short-circuits when downstream is down — keep Kafka consumers alive.

### 10.3 Backoff

Always use **exponential backoff with jitter** for retries:
```java
new ExponentialBackOffWithMaxRetries(5) {{
    setInitialInterval(500);
    setMultiplier(2.0);
    setMaxInterval(10_000);
}};
```

### 10.4 Dead Letter Topics

Already covered in §6. Key operational practices:
- Monitor DLT volume — sudden spike → incident.
- Automate DLT replay after deploys fixing the cause.
- Retain DLT longer than the main topic (~2–4× retention).
- Include correlation IDs and cause headers.

### 10.5 Poison Messages

A poison message is one the consumer **cannot make progress on**. Without DLT, it loops forever, blocks the partition, and lag explodes.

Always route deserialization failures to DLT:
```yaml
spring:
  kafka:
    consumer:
      value-deserializer: org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
```

### 10.6 Idempotency on the Consumer Side

Because at-least-once is the norm, consumers must be idempotent. Techniques:
- **Use event ID as primary/unique key** on a processed-events table.
- **Upsert semantics**: "set status=PAID where order=42" rather than "increment paid_count".
- **Conditional updates**: version columns, optimistic locking.

---

## 11. Performance & Scaling

### 11.1 Throughput Tuning

**Producer:**
- `linger.ms=10–50` → better batching.
- `batch.size=64k–128k`.
- `compression.type=lz4` or `zstd` → both CPU-friendly, 3–5× compression on JSON.
- `acks=all` is required for durability; expect a small hit.
- Share a single `KafkaProducer` across threads (it's thread-safe).

**Broker:**
- SSDs, plenty of page cache, 10 Gb networking.
- `num.io.threads`, `num.network.threads`, `socket.send.buffer.bytes` — tune for your hardware.
- Separate log dirs per disk.

**Consumer:**
- `max.poll.records=500+` for higher throughput.
- `fetch.min.bytes`, `fetch.max.wait.ms` — tune for latency vs throughput.
- Scale horizontally — add instances up to partition count.

### 11.2 Partitioning Strategies for Scale

- Choose **partition count** to support your **target parallelism**. 2–3× expected peak consumer count is a reasonable ballpark.
- Avoid very high partition counts (> a few thousand per broker) — metadata overhead grows.
- Choose **keys** that distribute evenly.

Partition count can be increased but **not decreased**, and increasing remaps keys. Pick wisely up front.

### 11.3 Consumer Scaling

- One consumer per partition is the max useful concurrency **within a group**.
- `concurrency=N` in `@KafkaListener` creates N consumers in one JVM.
- For true horizontal scale, deploy multiple instances.
- Don't exceed partition count — extras sit idle.

### 11.4 Producer Batching

```
Without batching:  |R1| |R2| |R3| |R4| ...   (many small requests)
With batching:     |R1 R2 R3 R4 ...|         (one large request)
```

Batching increases latency slightly (`linger.ms`) and drastically increases throughput. Always batch in production.

### 11.5 When Kafka Is Too Slow

Usually you're not using it right:
- Too many small partitions → broker overhead.
- `acks=all` with no `min.insync.replicas` tuning → waits unnecessarily.
- No compression.
- Synchronous per-message `send().get()` instead of async.
- Overly small batches.
- Tiny consumer `max.poll.records`.

---

## 12. Kafka Streams

### 12.1 What It Is

**Kafka Streams** is a Java library (not a separate cluster) for stream processing. It reads from Kafka topics, transforms, and writes back to Kafka.

Key properties:
- Runs inside your app — no separate compute cluster.
- Exactly-once processing within Kafka (when configured).
- Stateful operations (aggregations, joins) via local RocksDB state stores, backed by compacted changelog topics.

### 12.2 Basic Processing

```java
@Configuration
@EnableKafkaStreams
public class StreamsConfig {

    @Bean
    public KStream<String, OrderPlaced> pipeline(StreamsBuilder builder) {
        KStream<String, OrderPlaced> orders = builder.stream("orders.v1",
            Consumed.with(Serdes.String(), new JsonSerde<>(OrderPlaced.class)));

        KStream<String, HighValueOrder> highValue = orders
            .filter((k, v) -> v.total().compareTo(new BigDecimal("1000")) > 0)
            .mapValues(v -> new HighValueOrder(v.orderId(), v.total()));

        highValue.to("orders.high-value.v1",
            Produced.with(Serdes.String(), new JsonSerde<>(HighValueOrder.class)));

        return orders;
    }
}
```

### 12.3 Stateful Aggregation

```java
KTable<String, Long> ordersPerCustomer = builder.stream("orders.v1", ...)
    .groupBy((k, v) -> v.customerId())
    .count(Materialized.as("orders-per-customer"));

ordersPerCustomer.toStream().to("orders.count-by-customer");
```

State is kept locally in RocksDB and replicated via a compacted changelog topic — fault-tolerant across restarts.

### 12.4 Real-World Use Cases

- **Fraud detection**: join transactions with customer profiles in real time.
- **Enrichment**: take raw events + reference data → enriched stream.
- **Windowed aggregations**: rolling counts, moving averages.
- **Joining streams**: orders + payments → payment-matched-orders.
- **CDC pipelines**: Debezium → Kafka → Streams filters/projects → sinks.

**When NOT to use Kafka Streams:**
- You need global windowing over unbounded joins across massive state — Flink handles this better.
- Non-JVM stack.
- Streams you don't own — Kafka Streams is an "app library", not a centralized engine.

---

## 13. Monitoring & Observability

### 13.1 Key Metrics

**Broker:**
- `UnderReplicatedPartitions` — > 0 means degraded; > 0 for long means trouble.
- `OfflinePartitionsCount` — should always be 0.
- `ActiveControllerCount` — should always be 1 across the cluster.
- Request latency percentiles (produce/fetch).
- Disk usage per log dir.

**Producer:**
- `record-error-rate`, `record-retry-rate`.
- `batch-size-avg`, `compression-rate-avg`.
- `request-latency-avg`, `request-latency-max`.

**Consumer:**
- **Consumer lag** (records-lag-max / per-partition lag) — most important.
- `fetch-latency-avg`.
- `commit-latency-avg`.
- Rebalance count.

### 13.2 Prometheus Integration

Expose JMX metrics via JMX Exporter or Micrometer.

```groovy
implementation 'io.micrometer:micrometer-registry-prometheus'
```

Spring Boot auto-registers Kafka client metrics to Micrometer:
```
kafka_consumer_fetch_manager_records_lag{topic="orders.v1", partition="0"} 0
kafka_producer_outgoing_byte_total{client_id="..."} 1.23e8
```

For broker metrics, run **Kafka Exporter** or **jmx_exporter** as a sidecar.

**Dashboards to build:**
- Per-consumer-group lag.
- Produce/Fetch request rates and latencies.
- Under-replicated partitions over time.
- Rebalance counts.
- DLT publish rate.

### 13.3 Logging & Debugging

- Correlate events with a **correlation ID header** propagated through your services.
- Use **OpenTelemetry** (`spring-kafka` integrates): producer/consumer spans are linked.
- Log on failure with `topic/partition/offset/key` — without these, you can't reproduce.
- Use **kafka-ui**, **Conduktor**, or `kafka-console-consumer` to inspect data.

---

## 14. Production Best Practices

### 14.1 Topic Design

- **Name topics with intent + version**: `orders.placed.v1`, `payments.completed.v2`.
- **One event type per topic** is cleaner; mixing types is possible but complicates consumers.
- **Pick partition count carefully** — changing it remaps keys. Size for 2–3 year growth.
- **RF=3, min.insync.replicas=2** in prod. Anything less is not production.
- **Retention**: default 7d; increase if consumers need replay; use compaction for state topics.

### 14.2 Key Selection

- **Key = aggregate ID**: `orderId`, `userId`, `tenantId` — something whose events belong in order.
- Avoid **unbounded keys** (e.g., request IDs) with no reuse — destroys batching.
- Avoid **null keys** unless you truly don't care about ordering; they cause round-robin fan-out.

### 14.3 Avoiding Large Messages

Kafka is designed for **small** messages (< 1 MB). Large messages:
- Inflate replication cost.
- Saturate network.
- Stall consumers.

**Rules:**
- Target < 100 KB per message.
- For larger payloads (images, documents): store in S3, send the URL.
- If you must send large: increase `message.max.bytes`, `replica.fetch.max.bytes`, `max.request.size` carefully.

### 14.4 Schema Evolution

- **Never remove or rename fields** without a major version bump.
- **New fields**: add as optional with defaults (backward-compatible).
- **Versioning**: `event-version` header + optionally new topic `.v2` for breaking changes.
- Run Schema Registry in **BACKWARD** compatibility by default.
- Keep old schema versions available for consumers that haven't caught up.

### 14.5 Security

- **TLS** on all brokers (client-broker, broker-broker).
- **SASL** for authentication: `SASL_SSL` with SCRAM-SHA-512 or mTLS.
- **ACLs** per topic / consumer group.
- Use **MSK IAM** on AWS MSK.
- Rotate credentials; store in Vault / Secrets Manager.
- Network isolation: brokers in private subnets.

```yaml
spring:
  kafka:
    properties:
      security.protocol: SASL_SSL
      sasl.mechanism: SCRAM-SHA-512
      sasl.jaas.config: >-
        org.apache.kafka.common.security.scram.ScramLoginModule required
        username="app"
        password="${KAFKA_PASSWORD}";
```

### 14.6 The Outbox Pattern

For DB + Kafka atomicity (most critical workloads), the **outbox pattern** beats Kafka transactions:

1. In the same DB transaction, write business data *and* an outbox row.
2. A relay (Debezium / scheduled job) publishes outbox rows to Kafka.
3. Mark outbox row as published (or rely on CDC).

This gives **at-least-once** end-to-end with DB-level atomicity, which is usually what you actually want.

---

## 15. Real-World Scenarios

### 15.1 Order Processing System

```
[Web API]
   │ places order (HTTP)
   ▼
[Order Service] ──(tx: DB + outbox)──▶ Postgres
                                        │
                                        ▼ Debezium
                            Kafka: orders.placed.v1
                                        │
         ┌──────────────┬───────────────┼──────────────┐
         ▼              ▼               ▼              ▼
   [Inventory]    [Payment]     [Notifications]    [Analytics]
```

- Key: `orderId` → per-order ordering preserved.
- Each consumer group independent.
- Payment emits `payment.succeeded.v1` or `payment.failed.v1` → Order service updates state.
- DLT per consumer; retries non-blocking.
- Throughput: ~20k orders/minute; partitions=24; RF=3.

### 15.2 Event-Driven Microservices Platform

Large marketplace:
- 30+ services, each owning a bounded context.
- Kafka is the central nervous system.
- Avro + Schema Registry, BACKWARD compatibility enforced.
- Every service emits domain events; any service may subscribe.
- Sagas implemented via orchestrator services reading/emitting events.
- OpenTelemetry tracing across Kafka boundaries.

**Why Kafka:** services evolve independently; adding a new consumer doesn't require coordination.

### 15.3 Log Processing Pipeline

Logs from 500 Kubernetes pods:

```
[Pod logs] → Fluent Bit → Kafka (topic=logs.raw) → Kafka Streams (parse + enrich)
                                                        │
                                                        ▼
                                                 Kafka (topic=logs.structured)
                                                        │
                                                        ├──▶ ClickHouse (analytics)
                                                        └──▶ S3 (archive)
```

- Compression: zstd (logs compress 10×+).
- RF=3; retention=3 days; raw → delete; structured → delete.
- Streams processor enriches with pod metadata.

### 15.4 Change Data Capture (CDC)

```
[PostgreSQL] ──WAL──▶ Debezium ──▶ Kafka (topic=db.public.orders)
                                      │
                                      ▼
                       compacted; keyed by PK
                                      │
                      ┌───────────────┴───────────────┐
                      ▼                               ▼
                Elasticsearch (search)         Cache (Redis)
```

- Compacted topic keeps the latest row per PK.
- Consumers rebuild derived state at will.

### 15.5 Fraud Detection (Real-Time)

```
transactions.v1 ─┐
                  ├─▶ Kafka Streams ──▶ alerts.v1 ──▶ Ops dashboard
customer-profile ─┘      (join + window)
```

- Streams app joins transactions with customer profile (KTable).
- Windowed aggregates detect rapid transaction bursts.
- Alerts published to `alerts.v1`; Ops service consumes.

---

## 16. Comparisons & Trade-offs

### 16.1 Kafka vs RabbitMQ

| Aspect | Kafka | RabbitMQ |
|---|---|---|
| Model | Log | Queue |
| Retention | Days/weeks/forever | Until ack |
| Throughput | Millions/s | 10k–100k/s |
| Routing | Topic + key | Exchanges, bindings, headers |
| Fan-out | Multiple consumer groups | Multiple bindings |
| Priority queues | No | Yes |
| Delay queues | Plugin / retry topics | Native plugin |
| Ordering | Per-partition | Per-queue (limited) |
| Ops complexity | High | Medium |
| Best for | Event streams, pipelines, replay | Task queues, RPC-like patterns |

**Use RabbitMQ when:** job queues, priority/delay workloads, complex routing, smaller volume.
**Use Kafka when:** event logs, high throughput, multiple independent consumers, replay.

### 16.2 Kafka vs REST/gRPC

| Aspect | REST/gRPC | Kafka |
|---|---|---|
| Communication | Sync request/response | Async, fire-and-forget |
| Coupling | Tight (caller knows callee) | Loose |
| Latency | Lowest (single hop) | Higher (log + consumer lag) |
| Durability | None by default | Built-in |
| Fan-out | Manual, 1 call per consumer | Native |
| Use for | Queries, commands with immediate response | Events, async workflows |

**They're complementary.** Most real systems use both: REST for user-facing queries, Kafka for cross-service events.

### 16.3 When Kafka Is a Bad Choice

- **Low volume + low team maturity**: the operational cost dwarfs the benefit.
- **Priority queues, delayed jobs**: RabbitMQ/SQS fit better.
- **RPC-style synchronous calls**: Kafka forces async thinking where you don't need it.
- **Tiny messages in real time at microsecond scale**: Kafka latency is ms, not μs.
- **No team to own it**: managed options (MSK, Confluent Cloud) help — use them.
- **Strong global ordering across keys**: Kafka gives per-partition ordering only.
- **You just need a task queue**: use SQS, RabbitMQ, or a DB-backed queue.

---

## 17. Final Checklist

### You are production-ready with Apache Kafka if you can:

- [ ] Explain the difference between **events and messages**, and why Kafka is a log, not a queue.
- [ ] Describe Kafka's internals — **topics, partitions, brokers, replication, ISR, leaders, followers, offsets, consumer groups** — and how they combine to achieve scalability and durability.
- [ ] Walk through a **message's complete lifecycle**, including failure modes at each step.
- [ ] Configure a Spring Boot producer with **`acks=all`, `enable.idempotence=true`**, compression, and sensible batching.
- [ ] Configure a Spring Boot consumer with **manual ack**, `ErrorHandlingDeserializer`, `CooperativeStickyAssignor`, and tuned `max.poll.records`.
- [ ] Declare topics via `TopicBuilder` with **RF=3, `min.insync.replicas=2`**, proper retention, and partition counts sized for growth.
- [ ] Implement **retries + DLT** with `DefaultErrorHandler` or `@RetryableTopic`, and explain when each is appropriate.
- [ ] Choose **at-most-once / at-least-once / exactly-once** correctly, and extend EOS to external systems via **idempotency** or the **outbox pattern**.
- [ ] Choose **JSON vs Avro vs Protobuf** and manage schema evolution with a registry.
- [ ] Design **keys and partitions** to preserve order and avoid hotspots.
- [ ] Explain **retention vs compaction** and pick the right cleanup policy per topic.
- [ ] Reason about **rebalancing**, pick the right assignor, and tune `session.timeout.ms` and `max.poll.interval.ms`.
- [ ] Use **Kafka Streams** for filters, joins, and windowed aggregations, and know when to reach for Flink instead.
- [ ] Monitor **consumer lag, under-replicated partitions, rebalance counts, DLT volume**, and wire them into Prometheus + Grafana.
- [ ] Secure a cluster with **TLS + SASL + ACLs**.
- [ ] Debug production issues from `topic/partition/offset` + headers + tracing context.
- [ ] Articulate to a skeptical architect **when Kafka is wrong for the job** (low volume, RPC, priority queues) and propose a better fit.

---

*End of guide. Save as `kafka-learning-guide.md`.*
