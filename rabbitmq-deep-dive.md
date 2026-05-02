# RabbitMQ Deep Dive

> A senior-level guide for Java backend engineers who already know Spring Boot and Kafka, and want to design, build, debug, and operate RabbitMQ in production.

---

## Table of Contents

1. [Messaging Fundamentals](#1-messaging-fundamentals)
2. [RabbitMQ Fundamentals](#2-rabbitmq-fundamentals)
3. [AMQP Protocol Deep Dive](#3-amqp-protocol-deep-dive)
4. [Message Lifecycle](#4-message-lifecycle)
5. [Message Delivery Guarantees](#5-message-delivery-guarantees)
6. [Exchanges and Routing](#6-exchanges-and-routing)
7. [Spring Boot Integration](#7-spring-boot-integration)
8. [Advanced Features](#8-advanced-features)
9. [Error Handling and Reliability](#9-error-handling-and-reliability)
10. [Performance and Scaling](#10-performance-and-scaling)
11. [RabbitMQ in System Design](#11-rabbitmq-in-system-design)
12. [Monitoring and Observability](#12-monitoring-and-observability)
13. [Security](#13-security)
14. [Common Pitfalls](#14-common-pitfalls)
15. [RabbitMQ vs Kafka vs Other Systems](#15-rabbitmq-vs-kafka-vs-other-systems)
16. [Real-World Scenarios](#16-real-world-scenarios)
17. [Edge Cases Senior Developers Must Know](#17-edge-cases-senior-developers-must-know)
18. [Final Checklist](#18-final-checklist)

---

## 1. Messaging Fundamentals

### Why messaging systems exist

In a synchronous world, service A calls service B over HTTP and **waits**. If B is slow, A is slow. If B is down, A fails. If B's throughput is 100/s and traffic spikes to 1,000/s, requests time out and the system collapses.

A messaging system inserts a **buffer with rules** between services. The producer drops a message into a queue and moves on. The consumer pulls messages at its own pace. The broker absorbs spikes, retries failures, and decouples lifetimes.

You introduce a broker because you need at least one of these properties:

- **Decoupling** — producer doesn't know who consumes, or even if anyone is listening yet.
- **Buffering** — handle bursts that exceed the consumer's steady-state capacity.
- **Reliability** — survive consumer crashes; redeliver after restart.
- **Fan-out** — one event, many independent reactions.
- **Asynchrony** — caller doesn't need the result right now.
- **Workload smoothing** — long-running jobs don't block user requests.

### Synchronous vs asynchronous communication

| | Synchronous (HTTP/RPC) | Asynchronous (messaging) |
|---|---|---|
| Caller waits for result | Yes | No |
| Failure model | Caller sees the error | Broker holds the message; retried later |
| Coupling | Tight (caller must know callee) | Loose (producer doesn't know consumer) |
| Latency | Low end-to-end | Higher, but bounded by broker |
| Backpressure | Caller blocks or fails | Queue grows; consumer paces itself |
| Best for | Reads, user-blocking actions | Background work, events, fan-out |

The right choice is rarely "all sync" or "all async." Most production systems use sync for the user-blocking path and async for everything that can happen *eventually*.

### Queue vs stream vs pub/sub

| Model | Mental picture | Reading semantics | Examples |
|---|---|---|---|
| **Queue** | Work to-do list | Each message consumed once by one worker | RabbitMQ classic queues, SQS |
| **Pub/Sub** | Broadcast to subscribers | Each subscriber gets a copy | RabbitMQ fanout, Redis pub/sub |
| **Stream / log** | Append-only file with cursors | Multiple consumers, each tracks its own offset; messages retained for time/size | Kafka, RabbitMQ Streams |

RabbitMQ classic queues are **work distribution**: one message → one consumer. RabbitMQ exchanges turn that into **pub/sub** (one event → many queues, each with its own consumers). RabbitMQ Streams (since 3.9) bring Kafka-style log semantics into the same broker.

### Message delivery guarantees (preview)

Three theoretical levels — covered fully in §5:

- **At-most-once** — fast, possibly lossy. Fire and forget.
- **At-least-once** — durable, may duplicate. Default for production.
- **Exactly-once** — only achievable end-to-end via deduplication or idempotent consumers; no broker truly guarantees it across the network.

### When messaging is necessary

- Long-running tasks (image processing, PDF generation, billing runs).
- Spiky traffic where consumers can't scale instantly.
- Fan-out: one event triggers many independent side-effects (email + analytics + audit + cache invalidation).
- Cross-service communication where you must survive a downstream outage.
- Workflows with retries and DLQs.

### When messaging is overkill

- Simple synchronous request/response (use HTTP).
- Strongly consistent transactions across services (use a database, sagas, or outbox pattern, but not naïve messaging).
- Tiny systems where a cron job or a thread pool would do.
- Adding a broker just because "everyone uses one" — every broker is operational debt.

---

## 2. RabbitMQ Fundamentals

### What RabbitMQ is

RabbitMQ is a **general-purpose message broker** written in Erlang/OTP, originally implementing **AMQP 0-9-1**. It also speaks STOMP, MQTT, and (since 3.9) its own Streams protocol.

It is optimized for **flexible routing** and **per-message delivery semantics** — not for log retention. It excels at task queues, event distribution, and request/response over messaging.

### Core concepts

```
[Producer] ──publish──▶ [Exchange] ──bindings──▶ [Queue(s)] ──deliver──▶ [Consumer(s)]
```

**Producer.** Anything that publishes a message. Producers publish to an **exchange**, never directly to a queue.

**Exchange.** A routing engine. It receives messages and decides which queue(s) to put them in (or none) based on **bindings** and the message's **routing key** / **headers**.

**Binding.** A rule connecting an exchange to a queue. May include a routing key (direct, topic) or header arguments (headers exchange).

**Queue.** A named buffer that holds messages until consumed. Queues store messages on disk (if durable) and in memory.

**Consumer.** A subscriber that receives messages from a queue. Consumers acknowledge (ack) successful processing; the broker removes acked messages and may redeliver unacked ones.

**Message.** A binary payload plus headers and properties (content-type, persistence flag, priority, expiration, correlation-id, etc.).

**Connection.** A long-lived TCP connection (with TLS) from a client to the broker. Expensive to create.

**Channel.** A lightweight virtual connection multiplexed inside a TCP connection. Cheap. Most operations happen on a channel.

**Vhost (virtual host).** A logical isolation boundary — its own exchanges, queues, users, and permissions. Like a namespace.

### How a message flows

1. Producer opens a connection, opens a channel.
2. Producer publishes a message to **an exchange** with a routing key (or headers).
3. Exchange evaluates bindings. Each match results in a copy delivered to that queue.
4. Queue holds the message (in memory and/or on disk).
5. Consumer (subscribed to the queue) receives the message via a channel.
6. Consumer processes it, then sends an **ack** (success) or **nack/reject** (failure).
7. Acked → broker forgets the message. Nacked → requeued or dead-lettered.

Key mental model: **producers don't know about queues.** They publish to exchanges. Queues subscribe to exchanges. This decoupling is the source of RabbitMQ's flexibility.

---

## 3. AMQP Protocol Deep Dive

### AMQP 0-9-1 in one paragraph

AMQP 0-9-1 is a **binary, framed, asynchronous protocol** over TCP. A client opens a connection, then opens one or more channels. Communication uses small **methods** (e.g., `basic.publish`, `basic.consume`, `queue.declare`) wrapped in frames. Most operations are non-blocking; success or failure comes back asynchronously on the channel.

### Connections vs channels

| | Connection | Channel |
|---|---|---|
| What it is | TCP socket (with TLS) | Logical session inside a connection |
| Cost to open | Expensive (handshake, TLS, auth) | Cheap |
| Lifetime | Long (hours to days) | Short to medium (per consumer or worker) |
| Concurrency | One TCP socket | Many channels per connection (default 2047) |
| Errors | Killing the connection kills all channels | A channel can be closed without killing the connection |
| Threading rule | Share connection across threads | **Never share a channel across threads** |

**Rule of thumb:** one connection per process (or per producer/consumer pool); one channel per thread or per consumer.

### Frames

Every AMQP message on the wire is composed of frames:

- `method` frame — the operation (e.g., `basic.publish`).
- `header` frame — content properties + body size.
- `body` frame(s) — the payload, possibly split if larger than the frame size.
- `heartbeat` frame — keepalive.

You rarely deal with frames directly — clients abstract them.

### Exchange types

There are four built-in exchange types. They differ only in **how they decide which queues to deliver to.**

#### Direct exchange

Routes a message to queues whose **binding key exactly equals** the message's routing key.

```
exchange "orders.direct"
  binding key "paid"     → queue "billing"
  binding key "shipped"  → queue "logistics"

publish(routing key = "paid")    → billing
publish(routing key = "shipped") → logistics
publish(routing key = "refunded")→ (dropped, no binding)
```

**Use:** classic point-to-point routing where the routing key is a known enum-like value.

#### Topic exchange

Routes by matching the routing key against **wildcard patterns** in binding keys. Words are dot-separated.

- `*` matches exactly one word.
- `#` matches zero or more words.

```
exchange "events.topic"
  binding "order.*.eu"     → queue "eu-orders"
  binding "order.paid.#"   → queue "billing"
  binding "#"              → queue "audit"

publish "order.paid.eu"   → eu-orders, billing, audit
publish "order.shipped.us"→ audit
publish "user.created"    → audit
```

**Use:** event distribution where consumers want flexible subsets of an event stream.

#### Fanout exchange

Ignores routing keys. **Every bound queue gets a copy.**

```
exchange "broadcast.fanout"
  binding → queue "metrics"
  binding → queue "audit"
  binding → queue "search-indexer"

publish (any routing key) → metrics, audit, search-indexer
```

**Use:** pure pub/sub broadcast. Lowest routing cost.

#### Headers exchange

Routes based on **message headers**, not the routing key. Bindings specify a set of header key/value pairs and either `x-match: all` or `x-match: any`.

```
binding { x-match: all, format: "pdf", region: "EU" } → queue "eu-pdf-pipeline"
publish headers { format: "pdf", region: "EU", priority: 5 } → matches
publish headers { format: "pdf", region: "US" }              → doesn't match
```

**Use:** routing on multiple criteria when topic patterns can't express it. Slower than direct/topic; rarely needed.

### Default exchange

Every vhost has a nameless `""` exchange — a special **direct exchange** where every queue is implicitly bound with its own name as the routing key. Publishing to `""` with `routingKey="my.queue"` goes straight to `my.queue`. Convenient for quick scripts; **don't rely on it in production code** — it ties producers to specific queue names and bypasses your routing topology.

### Routing logic example end-to-end

```
events.topic (topic exchange)
├── binding "user.created.#"  → q.welcome-emails
├── binding "user.*.eu"       → q.gdpr-audit
├── binding "user.deleted.#"  → q.account-cleanup
└── binding "#"               → q.analytics

publish routing key "user.created.eu":
  → q.welcome-emails (matches user.created.#)
  → q.gdpr-audit     (matches user.*.eu)
  → q.analytics      (matches #)
```

Three queues, three independent consumer apps, one producer that just publishes one message.

---

## 4. Message Lifecycle

This section walks the **happy path** and **every common failure** so you can reason about reliability concretely.

### 4.1 Producer publishes

1. Producer serializes payload (JSON, Protobuf, …).
2. Producer sets properties:
   - `delivery_mode = 2` (persistent) — **required** for durability.
   - `content-type`, `correlation-id`, `headers`, `expiration`, `priority`.
3. Producer calls `basic.publish(exchange, routingKey, properties, body)` on a channel.
4. By default, this is **fire-and-forget** — no broker confirmation. To know the broker accepted it, enable **publisher confirms** (see §5 and §9).

**Failure modes here:**

- **TCP dropped before publish reached the broker** → message lost unless using publisher confirms + transactional outbox.
- **Exchange doesn't exist** → channel is closed with an error.
- **No queue is bound to match the routing key** → message is silently discarded unless the producer set the `mandatory` flag (then it's returned).

### 4.2 Exchange routes

1. Exchange evaluates bindings.
2. For each match, the broker creates an internal reference to the message and delivers it to that queue's data structure.
3. If `mandatory=true` and **no** queue matched, the broker returns the message to the producer via `basic.return`.

**Failure modes:**

- **No matching binding + no mandatory flag** → silent drop. This is the #1 production surprise.
- **Alternate exchange configured on the exchange** → unrouted messages go there instead of being dropped.

### 4.3 Queue stores

1. Queue appends the message to its internal queue index.
2. If `delivery_mode=2` **and** the queue is `durable=true`, the message is fsynced to disk before the broker acknowledges to the publisher (with confirms enabled).
3. Memory pressure may force the queue to **page** messages to disk regardless of persistence (see "lazy queues" / classic queues v2 in §10).

**Failure modes:**

- **Queue not durable + broker restart** → queue and messages disappear.
- **Message not persistent + broker restart** → message disappears even from a durable queue.
- **Disk full** → broker enters memory/disk alarm; producers are blocked.

### 4.4 Consumer receives

1. Consumer calls `basic.consume(queue, consumerTag, autoAck)` on a channel.
2. Broker delivers messages up to the channel's **prefetch** count without waiting for acks.
3. Each delivery comes with a **delivery tag** unique to that channel.

**Failure modes:**

- **`autoAck=true`** → broker considers the message delivered the moment it puts it on the wire. Consumer crash = message loss. **Avoid in production.**
- **Prefetch too high** → one slow consumer hoards messages while others sit idle.

### 4.5 Consumer processes and acks

1. Consumer does the work (write to DB, call API, etc.).
2. Consumer calls one of:
   - `basic.ack(deliveryTag)` — success. Broker drops the message.
   - `basic.nack(deliveryTag, requeue=true|false)` — failure. Requeue or send to DLX.
   - `basic.reject(deliveryTag, requeue=true|false)` — like nack but for a single tag.

**Failure modes:**

- **Consumer crashes before ack** → broker eventually times out the channel (or sees the connection drop) and **redelivers** the message to another consumer with `redelivered=true`.
- **Consumer acks before doing the work** → on crash, the work is lost.
- **Consumer rejects with `requeue=true` repeatedly** → poison message storms the queue. Use DLX.

### 4.6 End-to-end flow with failures

```
producer ─publish─▶ exchange ─route─▶ queue ─deliver─▶ consumer ─ack─▶ broker drops
                       │                                  │
                       │                                  └─crash before ack
                       │                                       └─▶ redelivered to another consumer
                       │
                       └─no binding match
                            └─▶ alternate exchange OR dropped
```

### 4.7 Retry behavior

RabbitMQ has **no built-in retry-with-delay**. Native options:

- **`requeue=true`** — instant requeue. Re-attempted immediately, almost always by the same consumer; tight loop on poison messages.
- **DLX + TTL** — reject without requeue → message goes to a Dead Letter Exchange → routed to a "retry" queue with a TTL → expired messages get dead-lettered back to the original queue. This is how you implement **delayed retries** (see §8).
- **Delayed Message Plugin** — `x-delayed-message` exchange type; first-class delays.

Idempotency on the consumer side is non-negotiable: redelivery can happen at any time.

---

## 5. Message Delivery Guarantees

### At-most-once

Each message is delivered **zero or one** times. Fast, lossy.

- Producer doesn't enable confirms.
- Consumer uses `autoAck=true` or acks before doing the work.
- Queue is non-durable, message is non-persistent.

**Use when:** loss is cheap (telemetry samples, click counters where you accept ~1% loss).

### At-least-once (the production default)

Each message is delivered **one or more** times. No loss, but duplicates possible.

How RabbitMQ implements it:

- **Producer** uses **publisher confirms** (`channel.confirmSelect()`); only considers a message successfully published when the broker acks it.
- **Producer** sets `delivery_mode = 2` (persistent).
- **Queue** is `durable=true`.
- **Exchange** is `durable=true`.
- **Consumer** uses **manual ack** (`autoAck=false`) and acks **after** processing.
- **Consumer** is **idempotent** to handle redeliveries.

This is what you should default to.

### Exactly-once (and why it doesn't really exist over the network)

Theoretically: each message is processed exactly one time. In practice, no broker can give you this *end-to-end* without help, because:

- The consumer can crash *between* doing the work and sending the ack. The broker will redeliver.
- The producer can crash *between* sending and receiving the confirm. It may resend.

You achieve **effectively-once** by combining:

1. **At-least-once delivery** at the broker layer.
2. **Idempotent consumers** (deduplication by message ID, or naturally idempotent operations like `INSERT ... ON CONFLICT DO NOTHING` keyed by an event ID).
3. **Outbox pattern** on the producer side to atomically commit DB state and enqueue the message.

### Trade-offs

| Guarantee | Throughput | Complexity | When to use |
|---|---|---|---|
| At-most-once | Highest | Lowest | Lossy telemetry, samples |
| At-least-once | Medium | Medium (idempotency) | **99% of production** |
| Effectively-once | Lower (dedup overhead) | High (outbox + dedup store) | Money, billing, irreversible actions |

---

## 6. Exchanges and Routing

### Direct exchange — deep dive

**How it works.** Hash table from binding key to bound queues. O(1) routing.

**Example.** Order events with type as routing key:
```
exchange order.events.direct
  "order.created"   → q.welcome
  "order.paid"      → q.fulfillment
  "order.refunded"  → q.support
```

**Use cases.** RPC-style routing; logs by severity (`info`, `warn`, `error`); finite, well-known event types.

**Pitfalls.**
- Adding a new event type means adding a new binding everywhere — no wildcards.
- Multiple queues bound with the same key all get a copy. Surprising for newcomers.
- Typo in routing key = silent drop unless `mandatory` flag is set.

### Topic exchange — deep dive

**How it works.** Trie of dot-separated tokens. Each binding becomes a path with literals and `*`/`#` wildcards.

**Example.** Multi-tenant SaaS:
```
exchange events.topic
  "tenant.acme.order.#"  → q.acme
  "tenant.*.order.paid"  → q.global-billing
  "#"                    → q.audit-log
```

**Use cases.** Event-driven systems with many event types and consumers wanting subsets; multi-tenant routing; metric/log collection where consumers filter by hierarchy.

**Pitfalls.**
- `#` matches zero or more words — `"#"` binding catches everything, including future events you didn't design for.
- Overlapping bindings deliver the same message multiple times **per matching binding** to the same queue? No — the broker deduplicates per (queue, message): a queue gets at most one copy regardless of how many bindings match. But **different queues** all get their own copy.
- Routing key design becomes a contract; renaming hierarchies is a breaking change.

### Fanout exchange — deep dive

**How it works.** Send to every bound queue. No matching cost.

**Example.** Cache invalidation:
```
exchange cache.invalidate.fanout
  → q.user-cache
  → q.product-cache
  → q.search-cache
publish (any) → all three queues
```

**Use cases.** Broadcast (notifications to all consumer groups), cache invalidation, event mirroring across regions.

**Pitfalls.**
- Adding a new consumer = a new queue + binding. Easy to forget.
- Combined with persistent messages and many queues, fanout multiplies disk writes.
- Don't bind a single shared queue with many fanout exchanges expecting deduplication — you'll get one message per exchange.

### Headers exchange — deep dive

**How it works.** Linear scan of bindings; for each, evaluate header match (`all` or `any`).

**Example.** Document pipeline:
```
binding {x-match: all, format: pdf, lang: en} → q.en-pdf
binding {x-match: any, urgent: true, vip: true} → q.priority
```

**Use cases.** When routing depends on **multiple orthogonal attributes** that don't fit one routing key.

**Pitfalls.**
- Slower than direct/topic.
- Most teams underuse it; you can usually express the same logic with topic + a hierarchical key.
- Easy to misconfigure `x-match` and produce silent mismatches.

### Choosing an exchange type

| Need | Use |
|---|---|
| Single, exact key match | Direct |
| Wildcards, hierarchies | Topic |
| Broadcast to all subscribers | Fanout |
| Multi-attribute routing | Headers |
| Delayed delivery | Delayed-message plugin (special) |

---

## 7. Spring Boot Integration

### 7.1 Setup

`pom.xml`:
```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

`application.yml`:
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: app
    password: ${RABBITMQ_PASSWORD}
    virtual-host: /prod
    publisher-confirm-type: correlated   # enables async publisher confirms
    publisher-returns: true              # enables 'mandatory' returns
    listener:
      simple:
        acknowledge-mode: manual         # manual ack — never use auto in production
        prefetch: 20
        concurrency: 4                   # min consumer threads per listener
        max-concurrency: 16
        retry:
          enabled: false                 # we handle retry via DLX, not in-memory
```

### 7.2 Connection and template configuration

```java
@Configuration
public class RabbitConfig {

    @Bean
    public Jackson2JsonMessageConverter jsonConverter(ObjectMapper mapper) {
        return new Jackson2JsonMessageConverter(mapper);
    }

    @Bean
    public RabbitTemplate rabbitTemplate(ConnectionFactory cf,
                                         Jackson2JsonMessageConverter conv) {
        RabbitTemplate t = new RabbitTemplate(cf);
        t.setMessageConverter(conv);
        t.setMandatory(true); // returned-message callback fires for unroutable
        t.setConfirmCallback((corr, ack, cause) -> {
            if (!ack) {
                // log + persist for replay; corr.getId() is your correlation token
            }
        });
        t.setReturnsCallback(returned -> {
            // unroutable message: log loudly, persist, alert
        });
        return t;
    }
}
```

### 7.3 Declaring topology (queues, exchanges, bindings)

Topology is **declarative** — Spring auto-declares these on startup if they don't exist with matching arguments.

```java
@Configuration
public class OrderTopology {

    public static final String EXCHANGE = "orders.topic";
    public static final String QUEUE_PAID = "q.orders.paid";
    public static final String DLX = "orders.dlx";
    public static final String DLQ_PAID = "q.orders.paid.dlq";

    @Bean TopicExchange ordersExchange() {
        return ExchangeBuilder.topicExchange(EXCHANGE).durable(true).build();
    }

    @Bean DirectExchange dlx() {
        return ExchangeBuilder.directExchange(DLX).durable(true).build();
    }

    @Bean Queue paidQueue() {
        return QueueBuilder.durable(QUEUE_PAID)
            .withArgument("x-dead-letter-exchange", DLX)
            .withArgument("x-dead-letter-routing-key", "orders.paid.dlq")
            .build();
    }

    @Bean Queue paidDlq() {
        return QueueBuilder.durable(DLQ_PAID).build();
    }

    @Bean Binding bindPaid() {
        return BindingBuilder.bind(paidQueue()).to(ordersExchange()).with("order.paid.#");
    }

    @Bean Binding bindPaidDlq() {
        return BindingBuilder.bind(paidDlq()).to(dlx()).with("orders.paid.dlq");
    }
}
```

**Important:** changing arguments on an already-declared queue requires deleting and recreating it — Spring will fail to start with a 406 PRECONDITION_FAILED.

### 7.4 Producer

```java
@Component
public class OrderEventPublisher {

    private final RabbitTemplate template;

    public OrderEventPublisher(RabbitTemplate template) {
        this.template = template;
    }

    public void publishPaid(OrderPaidEvent event) {
        CorrelationData corr = new CorrelationData(event.orderId());
        template.convertAndSend(
            OrderTopology.EXCHANGE,
            "order.paid." + event.region(),
            event,
            msg -> {
                msg.getMessageProperties().setMessageId(event.eventId());
                msg.getMessageProperties().setContentType("application/json");
                msg.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                msg.getMessageProperties().setHeader("traceparent", currentTraceparent());
                return msg;
            },
            corr
        );
    }

    private String currentTraceparent() { /* fetch from MDC */ return ""; }
}
```

The `CorrelationData` lets the confirm callback identify which publish succeeded or failed.

### 7.5 Consumer

```java
@Component
public class OrderPaidListener {

    private static final Logger log = LoggerFactory.getLogger(OrderPaidListener.class);
    private final FulfillmentService fulfillment;
    private final ProcessedEventStore processed;

    public OrderPaidListener(FulfillmentService f, ProcessedEventStore p) {
        this.fulfillment = f;
        this.processed = p;
    }

    @RabbitListener(queues = OrderTopology.QUEUE_PAID, concurrency = "4-16")
    public void onPaid(OrderPaidEvent event,
                       @Header(AmqpHeaders.MESSAGE_ID) String messageId,
                       @Header(name = "x-death", required = false) List<Map<String,?>> deaths,
                       Channel channel,
                       @Header(AmqpHeaders.DELIVERY_TAG) long tag) throws IOException {
        try {
            if (processed.alreadySeen(messageId)) {
                channel.basicAck(tag, false);
                return;
            }
            fulfillment.fulfill(event);
            processed.markSeen(messageId);
            channel.basicAck(tag, false);
        } catch (TransientException e) {
            // retryable: nack without requeue → DLX → retry queue with TTL → back here
            log.warn("transient failure for {}", messageId, e);
            channel.basicNack(tag, false, false);
        } catch (Exception e) {
            // poisonous: same path, but the retry queue eventually parks it
            log.error("permanent failure for {}", messageId, e);
            channel.basicNack(tag, false, false);
        }
    }
}
```

Why this shape:
- **Manual ack** lets you ack only after success.
- **`x-death` header** tells you how many times the message has bounced through the DLX.
- **Idempotency** via `ProcessedEventStore` (Redis SET with TTL or a unique-constraint table).
- **`channel.basicNack(..., false)`** → no requeue; goes to the DLX. We never `requeue=true` for failures — that's a tight loop.

### 7.6 Sending and receiving JSON

`Jackson2JsonMessageConverter` (registered in §7.2) converts payloads automatically. The `__TypeId__` header is added by default — you can use type mapping to avoid coupling consumer classes to producer FQCNs:

```java
@Bean
public Jackson2JsonMessageConverter jsonConverter() {
    Jackson2JsonMessageConverter c = new Jackson2JsonMessageConverter();
    DefaultClassMapper mapper = new DefaultClassMapper();
    mapper.setIdClassMapping(Map.of(
        "OrderPaidEvent", OrderPaidEvent.class,
        "OrderRefundedEvent", OrderRefundedEvent.class
    ));
    mapper.setTrustedPackages("com.example.orders.events");
    c.setClassMapper(mapper);
    return c;
}
```

This decouples the wire `__TypeId__` from your Java packaging.

### 7.7 Routing keys in Spring

Routing keys are **first-class** in Spring AMQP — pass them via `convertAndSend(exchange, routingKey, payload)`. For topic exchanges, build them deterministically from event metadata (`order.paid.eu.acme`), not from free-form strings.

---

## 8. Advanced Features

### 8.1 Dead Letter Exchanges (DLX)

A queue can be configured with `x-dead-letter-exchange` (and optionally `x-dead-letter-routing-key`). Messages are dead-lettered when:

- They are **rejected/nacked with `requeue=false`**.
- Their **TTL expires** in the queue.
- The queue's **length limit** is exceeded (and the overflow policy says drop-head).

Dead-lettered messages get an `x-death` header recording every dead-letter event with reason, exchange, queue, and count. Use it to detect retry loops.

### 8.2 TTL

Two flavors:

- **Per-queue TTL** — `x-message-ttl` queue argument. All messages in that queue expire after N ms.
- **Per-message TTL** — set `expiration` property on publish. Expiration is only checked when the message is at the **head** of the queue.

Both push expired messages to the DLX (if configured) or drop them.

### 8.3 Delayed retries via DLX + TTL (the classic pattern)

```
work queue (q.orders.paid)
  ── nack(requeue=false) ──▶ retry exchange (orders.dlx)
                                       │
                                       ▼
                          retry queue (q.orders.paid.retry)
                            x-message-ttl: 30000
                            x-dead-letter-exchange: orders.topic
                            x-dead-letter-routing-key: order.paid.<...>
                                       │
                                       ▼
                          (after 30s, dead-letters back to original)
```

Multiple retry queues with increasing TTLs (1s, 5s, 30s, 300s) give exponential-backoff retries without any in-memory loops.

### 8.4 Delayed messages (plugin)

`rabbitmq_delayed_message_exchange` plugin adds an `x-delayed-message` exchange type. Set `x-delay` header on a publish; the broker holds it for the requested ms before routing as if it were a normal message.

```java
template.convertAndSend("delayed.x", "user.followup", payload, msg -> {
    msg.getMessageProperties().setHeader("x-delay", 60_000); // 1 minute
    return msg;
});
```

**Caveat:** delayed messages live in Mnesia until due — heavy use bloats memory and slows broker startup. For large-scale delayed work (millions of pending messages), use a database-backed scheduler (Quartz, Temporal) instead.

### 8.5 Message priority

Declare a queue with `x-max-priority: N` (1–10 typical). Producers set `priority` on the message. The queue is internally a priority structure (multiple sub-queues).

```java
@Bean Queue prioritized() {
    return QueueBuilder.durable("q.work")
        .withArgument("x-max-priority", 10)
        .build();
}
```

**Caveats:**
- Priority queues are more expensive than plain ones.
- Once a message is delivered to a consumer, prefetched messages aren't re-prioritized — keep prefetch low if priority matters.
- If most traffic is the same priority, you're paying for nothing.

### 8.6 Retry patterns

| Pattern | Mechanism | Pros | Cons |
|---|---|---|---|
| **Instant requeue** | `nack(requeue=true)` | Trivial | Tight loop on poison |
| **In-memory retry** | Spring Retry interceptor | Simple | Blocks consumer; lost on crash |
| **DLX + TTL ladder** | Multiple retry queues | Durable, scales | More topology |
| **Delayed plugin** | `x-delay` | Cleanest topology | Memory load on broker |
| **Outbox + scheduler** | DB table + worker | Full control | Most code |

**Production default:** DLX + TTL ladder, with a final **parking lot DLQ** that humans inspect.

---

## 9. Error Handling and Reliability

### 9.1 Acknowledgments

| Mode | Behavior | Use |
|---|---|---|
| `auto` | Broker considers delivered the moment it sends | Avoid in production |
| `manual` | Consumer calls `basicAck` / `basicNack` | **Default** |
| `none` | Same as auto, no acks at all | Pure telemetry |

In Spring AMQP, set `acknowledge-mode: manual` and inject `Channel`/`deliveryTag` into the listener.

### 9.2 ACK / NACK / Reject

- `basic.ack(tag, multiple)` — tag is the delivery tag; `multiple=true` acks all up to and including this tag. Useful for batched processing.
- `basic.nack(tag, multiple, requeue)` — negative ack, optionally for many.
- `basic.reject(tag, requeue)` — rejects a single message.

Rule: **ack after success; nack without requeue on failure.**

### 9.3 Requeueing

`requeue=true` puts the message back at the **head** (or near it) of the same queue. The same consumer almost always picks it up again immediately.

When to use:
- **Transient infrastructure errors** that you have already verified will not repeat in microseconds (e.g., consumer detected a dependency outage and is shutting itself down).

When **not** to use:
- Code bugs (every consumer will fail the same way).
- DB constraint violations.
- Anything you'd retry "in a few seconds."

### 9.4 Retry strategies

Layer them:

1. **In-process retry** for genuinely transient blips (e.g., 1 retry on transient timeout) using Spring Retry **only** if the work is fast and idempotent.
2. **DLX + TTL ladder** for delayed retries (preferred default).
3. **Parking lot queue** (terminal DLQ) for messages that exceeded retry attempts. A human inspects, fixes, and replays.

Stop retrying after a fixed `x-death` count:

```java
int attempts = (deaths == null) ? 0 : ((Number) deaths.get(0).get("count")).intValue();
if (attempts >= 5) {
    // route to parking lot — manual ack to remove from cycle
    parkingLot.publish(event);
    channel.basicAck(tag, false);
    return;
}
```

### 9.5 Poison messages

A **poison message** is one that always fails (bad payload, missing reference, schema mismatch). Without a cap, it cycles forever and pegs your consumers.

Detection:
- High `x-death` count.
- Same message ID repeatedly in error logs.
- DLQ depth growing.

Mitigation:
- Cap retries (see above).
- Alert on parking lot growth.
- Capture the payload + headers in the alert so the on-call engineer can replay or discard.

### 9.6 Best practices

- **Always** publisher confirms + persistent messages + durable exchanges + durable queues for important traffic.
- **Always** manual ack on consumers; ack after success.
- **Always** idempotent consumers.
- Cap retries; have a parking lot.
- Set `mandatory=true` and a returns callback so unroutable messages don't disappear.
- Use `messageId` + dedup store for exactly-once semantics.

### 9.7 Anti-patterns

- `autoAck=true` in production.
- `nack(requeue=true)` on every error.
- No DLX, no parking lot.
- Acking before doing the work ("optimistic ack").
- Sharing a channel across threads.
- Producing without confirms in critical paths.

---

## 10. Performance and Scaling

### 10.1 Consumer scaling

RabbitMQ classic queues are **single-leader** — one node owns the queue and all consumers attach through it. Scaling is **horizontal across consumers**, not across queue leaders.

Options:
- **More consumers on one queue** — work-stealing across processes/pods.
- **Sharded queues** — N queues, producers hash by key → queue. Horizontal at the cost of per-key ordering only.
- **Quorum queues** (since 3.8) — Raft-replicated; better failover but throughput per queue is lower than classic.
- **RabbitMQ Streams** — log-style, partition-able like Kafka.

### 10.2 Prefetch count

Prefetch = how many unacked messages the broker will let one channel hold concurrently.

- **Low (1–10):** even distribution; one slow consumer doesn't starve others. Lower throughput.
- **High (100–1000):** one consumer batches a lot; high throughput; risk of one consumer hoarding.
- **Per-channel vs per-consumer:** in AMQP 0-9-1 it's per-channel by default. Spring AMQP uses one channel per consumer thread, so it's effectively per-consumer.

**Rule of thumb:** start at `prefetch = 2 * average concurrency per consumer`. Tune from metrics.

### 10.3 Queue performance

- **Classic queues v2** (default since 3.10) — large queues page to disk efficiently; replaces "lazy queues."
- **Quorum queues** — durable, replicated; ideal for critical data; slower for very high message rates per queue.
- **Streams** — append-only logs; high throughput; consumer-tracked offsets.

Avoid:
- **Very long queues** (millions of messages) — broker memory and disk pressure, slow recovery on restart. Treat queue depth as a smoke alarm.
- **Many short-lived queues** (per-request RPC queues) — control-plane overhead. Reuse reply-to queues or use direct reply-to.

### 10.4 Horizontal scaling

- **Cluster** — multiple RabbitMQ nodes share metadata via Erlang clustering.
- **Quorum queues** replicate across nodes for HA.
- **Federation** — link brokers across regions/datacenters loosely.
- **Shovels** — move messages between brokers/queues, useful for migrations.

### 10.5 Tuning checklist

- Prefetch tuned per workload.
- Consumer concurrency matched to downstream capacity (DB, APIs).
- Persistent + durable only where you need it; turn it off for ephemeral data.
- TLS adds CPU cost — measure.
- File descriptor and Erlang process limits raised on the host.
- `vm_memory_high_watermark` and `disk_free_limit` set; alerts on them.
- Producers spread across many connections (not all on one).
- Avoid `ttl=0` on the head of long queues; the broker re-checks TTLs on head.

---

## 11. RabbitMQ in System Design

### 11.1 Microservices communication

**Async events.** Service A publishes `order.paid`; services B, C, D each consume. A doesn't know about them. Adding service E is one binding, not a code change in A.

**Trade-off:** debugging cross-service flows requires tracing (OpenTelemetry, correlation IDs in headers).

### 11.2 Event-driven architecture

Choreography over orchestration: services react to events instead of being commanded. RabbitMQ topic exchanges are a natural fit for hierarchical event taxonomies.

**Trade-off:** beware of "event spaghetti" — without a registry/catalog, the implicit graph of producers/consumers becomes invisible. Maintain an event catalog (a doc, a schema registry, or both).

### 11.3 Task queues

Long-running, CPU- or IO-bound work moved off the request path: image processing, PDF generation, email sending, ML inference. RabbitMQ's per-message routing and acknowledgments fit well.

**Trade-off:** for very high throughput streaming work, Kafka or Streams is a better tool.

### 11.4 RPC over messaging

`reply-to` + `correlation-id` lets you do request/response over RabbitMQ. Use cases: queue-mediated RPC where you want backpressure or rate limiting. Use **direct reply-to** (`amq.rabbitmq.reply-to`) to avoid creating a per-request queue.

**Trade-off:** mostly synchronous over an async substrate; HTTP/gRPC is usually simpler unless you specifically want broker-level queueing.

### 11.5 Outbox / inbox patterns

To get effectively-once across DB + broker:
- **Outbox:** in the same DB transaction, write the domain change and an outbox row. A separate process reads outbox rows and publishes to RabbitMQ, marking them sent.
- **Inbox:** consumer records seen `messageId`s in a table to dedupe redeliveries.

This bridges the gap that simple "publish from a service method" cannot.

---

## 12. Monitoring and Observability

### 12.1 What to monitor

**Per queue:**
- `messages_ready` — waiting for a consumer. Growing = slow consumers or none.
- `messages_unacknowledged` — in flight. Stuck-growing = consumers timing out.
- `messages_persistent` — disk-backed.
- `message_stats.publish_details.rate` — publish rate.
- `message_stats.deliver_get_details.rate` — delivery rate.
- `consumer_count` — running consumers. 0 = nobody is listening.

**Per node:**
- `mem_used` vs `mem_limit` — memory alarm imminent if close.
- `disk_free` vs `disk_free_limit` — disk alarm imminent if close.
- `fd_used` / `fd_total` — file descriptor exhaustion blocks new connections.
- `socket_used` / `socket_total`.
- Erlang process count.

**Per connection / channel:**
- Long-lived connection count — leaked connections are a common bug.
- Channel count per connection — leaked channels too.

### 12.2 Management UI

Built-in at `http://broker:15672`. Shows queues, exchanges, bindings, connections, channels, and live message rates. Indispensable for ad-hoc debugging — but **don't make production changes through it** without tracking them in code.

### 12.3 Prometheus integration

Enable the plugin: `rabbitmq-plugins enable rabbitmq_prometheus`. Metrics exposed at `:15692/metrics`. Scrape with Prometheus, dashboard with Grafana (the official RabbitMQ Grafana dashboards are excellent starting points).

Key alerts to set up:
- `rabbitmq_queue_messages_ready > X` for Y minutes (queue buildup).
- Consumer count == 0 on a critical queue.
- Memory alarm or disk alarm raised.
- Publisher confirm failures spiking.
- DLQ depth growing.
- Parking lot non-empty.

### 12.4 Debugging

Common diagnostic moves:
- `rabbitmqctl list_queues name messages_ready messages_unacknowledged consumers`
- `rabbitmqctl list_consumers`
- `rabbitmqctl list_connections`
- Get a stuck message: drain it from the management UI or use a one-off consumer.
- Inspect `x-death` headers to see retry history.
- Check broker logs at `/var/log/rabbitmq/rabbit@hostname.log`.
- Capture a network trace with `tcpdump` if you suspect protocol issues.

### 12.5 Tracing

Propagate `traceparent` (W3C trace context) in message headers. On consume, restore the trace context before processing. Most OpenTelemetry instrumentations do this automatically for Spring AMQP.

---

## 13. Security

### 13.1 Authentication

- **Username / password** (default). Stored hashed (SHA-256). Avoid weak passwords.
- **X.509 certificates** (mTLS).
- **OAuth 2.0** via `rabbitmq_auth_backend_oauth2` — broker validates JWTs from your IdP.
- **LDAP** plugin for enterprise directories.

Disable the default `guest`/`guest` user on anything reachable from outside `localhost`.

### 13.2 Authorization

Permissions are per **vhost** and use three regex patterns: `configure`, `write`, `read`. Set them on each user.

```bash
rabbitmqctl set_permissions -p /prod app_orders \
  '^orders\..*$' '^orders\..*$' '^orders\..*$'
```

That user can only configure, write to, and read from resources matching `orders.*`. Apply least privilege per service: a service should only see its own queues/exchanges.

Topic permissions add a finer-grained layer for topic exchanges (which routing keys a user may publish/consume).

### 13.3 TLS

- Use TLS for AMQP (5671) and management (15671) in any non-trivial deployment.
- Pin server certs on clients where possible.
- For internal mesh deployments, mTLS is preferable to passwords.

### 13.4 Operational hygiene

- Vhost-per-tenant or vhost-per-environment to enforce isolation.
- Audit logs (`rabbit_auth_backend_*` logs auth attempts).
- Network policies / security groups to restrict who can reach 5672 / 15672.
- Rotate passwords and certs regularly.
- Don't put secrets in queue names or routing keys (they appear in logs and the management UI).

---

## 14. Common Pitfalls

### 14.1 Message loss

**Why it happens:**
- Non-persistent messages on a durable queue + broker restart.
- Non-durable queue + broker restart.
- No publisher confirms — TCP buffer dropped.
- `autoAck=true` + consumer crash.
- `mandatory=false` + no matching binding → silent drop.

**Fix:** persistent + durable + confirms + mandatory + manual ack. The full reliability stack.

### 14.2 Duplicate processing

**Why it happens:**
- Consumer crashes after work, before ack → redelivery.
- Producer retries after partial confirm failure.
- Multiple consumers on the same queue without idempotency.

**Fix:** assign a stable `messageId` (UUID per logical event), store seen IDs server-side, check before processing. Or rely on naturally idempotent operations (UPSERT, set semantics).

### 14.3 Queue buildup

**Why it happens:**
- Consumers slower than producers.
- Consumer crash loop.
- Downstream dependency outage (DB, API) blocking consumers.
- Prefetch too low or consumer pool too small.

**Fix:** alerts on `messages_ready`. Auto-scale consumers. Backpressure from producers when queue depth exceeds a threshold (or apply per-queue length limits with overflow policy).

### 14.4 Poor routing design

**Why it happens:**
- "Quick" direct exchanges that grow into 50+ bindings nobody understands.
- Routing keys with implicit semantics never written down.
- One queue bound to everything via `#`.

**Fix:** treat routing topology as **API design**. Document it. Version it. Use a topic exchange with an explicit hierarchy. Avoid `#` catch-alls except for audit/log queues.

### 14.5 Channel leaks

**Why it happens:**
- Opening a channel per request and not closing on error.
- Sharing channels across threads (they break and the recovery doesn't always happen).

**Fix:** one channel per thread or use a pooled `RabbitTemplate`. Spring AMQP handles this if you don't fight it.

### 14.6 Connection storms

**Why it happens:**
- N pods restart simultaneously and all reconnect with no backoff.
- Misconfigured client retry that hammers the broker.

**Fix:** jittered exponential backoff in the client. Use Spring AMQP's built-in `ConnectionFactory` recovery; don't disable it.

### 14.7 Memory and disk alarms

**Why it happens:**
- Queue grew to millions of messages.
- A burst of large persistent messages.
- Slow disks (network-attached storage, throttled cloud volumes).

**Effect:** the broker **blocks producers** until pressure drops. Looks like a hard outage from the producer side.

**Fix:** alert before alarms fire. Use queue length limits with `overflow: drop-head` for non-critical queues. Use proper local SSDs.

### 14.8 Schema changes break consumers

**Why it happens:**
- Producer adds a required field.
- Consumer deserializes strictly.

**Fix:** treat events as a published schema. Use additive evolution. For breaking changes, version the routing key (`order.paid.v2`) and run consumers in parallel during migration.

---

## 15. RabbitMQ vs Kafka vs Other Systems

### 15.1 Comparison table

| Dimension | RabbitMQ | Kafka | Redis Streams | SQS |
|---|---|---|---|---|
| Model | Queue + flexible routing | Distributed log | Lightweight log | Managed queue |
| Retention | Until consumed | Time/size based | Time/size based | Up to 14 days |
| Replay | No (consume = remove) | Yes (offset rewind) | Yes | No |
| Routing | Direct/topic/fanout/headers | Per-topic, partition keys | None | None |
| Ordering | Per-queue (per consumer if single) | Per-partition | Per-stream | Best-effort, FIFO option |
| Throughput | High (10k–100k msg/s/queue) | Very high (millions/s) | High | Medium |
| Latency | Low (sub-ms locally) | Low–medium | Low | Higher (managed) |
| Delivery | At-most / at-least-once | At-least-once / effectively-once with idempotency + transactions | At-least-once | At-least-once (FIFO: exactly-once) |
| Operational complexity | Medium | High | Low (if you have Redis) | None (managed) |
| Sweet spot | Task queues, flexible routing, RPC | Event streaming, replay, analytics | Lightweight pub/sub | Cloud-native simple queues |

### 15.2 Queue vs stream systems mental model

- **Queue (RabbitMQ classic):** message disappears once acked. The broker holds *unprocessed* work.
- **Stream (Kafka, RabbitMQ Streams):** messages persist for a retention window. Consumers maintain offsets and can replay.

If you ever say "I want to add a new consumer next week and have it process the last month of events," you want a stream. If you say "this message is a unit of work; once done, it's gone," you want a queue.

### 15.3 When to use RabbitMQ

- Per-message routing rules matter (topic/headers).
- Task queues with retry/DLQ semantics.
- Lower volume, latency-sensitive, complex routing.
- RPC over messaging.
- Multi-protocol needs (MQTT, STOMP).

### 15.4 When NOT to use RabbitMQ

- High-volume event streams that need replay → Kafka (or RabbitMQ Streams if you want one broker).
- Long-term retention / analytics → Kafka.
- Pure managed simplicity in AWS → SQS/SNS.
- You only need lightweight pub/sub inside one team's app → Redis Streams may suffice.
- Strong cross-partition ordering across millions of events → Kafka with sensible keying.

---

## 16. Real-World Scenarios

### 16.1 Image processing pipeline (task queue)

**Architecture.**
```
upload service ──publish──▶ images.x (direct)
                              "image.process" ──▶ q.image.process
q.image.process ──▶ N worker pods
workers nack(no-requeue) on transient → DLX → retry queues (5s, 30s, 5m)
```

**RabbitMQ usage.**
- Direct exchange because routing key is a fixed action.
- Quorum queue for durability across broker node failures.
- Prefetch = 2 because per-message work takes minutes.
- DLX ladder for retries; parking lot DLQ at end.

**Trade-offs.**
- Long-running tasks tie up channels → keep prefetch low; consider chunking.
- For bursts >10k images, batch upload acks (`multiple=true`).

### 16.2 Event-driven order system

**Architecture.**
```
order-service ──▶ orders.topic
  bindings:
    "order.paid.#"     → q.fulfillment, q.billing, q.analytics
    "order.refunded.#" → q.support, q.analytics
    "order.*.eu"       → q.gdpr-audit
    "#"                → q.audit (long-retention via streams plugin)
```

**RabbitMQ usage.**
- Topic exchange for hierarchical events.
- Each consumer service owns its queue and binds for what it needs.
- Outbox pattern in `order-service` for atomic DB+publish.

**Trade-offs.**
- Need an event catalog so the implicit graph stays known.
- Schema versioning for events; new fields additive only.
- Consumers must be idempotent — at-least-once delivery.

### 16.3 Notification system (email/SMS/push)

**Architecture.**
```
producer ──▶ notify.topic
  "notify.email.#" → q.email
  "notify.sms.#"   → q.sms
  "notify.push.#"  → q.push

each queue has DLX → retry-with-backoff queues → parking lot
```

**RabbitMQ usage.**
- Topic exchange routing by channel.
- Priority queue for urgent (`priority=9`) vs marketing (`priority=1`).
- Per-channel rate limits via consumer concurrency caps (provider rate limits).

**Trade-offs.**
- Idempotency is critical — duplicate emails are user-visible.
- Use `messageId` = `(userId, notificationId, channel)` for dedup.

### 16.4 RPC over messaging

**Architecture.**
```
client ──publish (reply-to=amq.rabbitmq.reply-to, correlation-id=X)──▶ rpc.requests
worker ──publish to reply-to with correlation-id=X──▶ client
```

**RabbitMQ usage.**
- Direct reply-to (no queue creation per call).
- Correlation IDs tie responses to in-flight requests.
- Timeouts on the client side; orphan responses logged.

**Trade-offs.**
- Slower than HTTP/gRPC for low-latency calls.
- Useful when you need queue-based backpressure to a rate-limited backend.

### 16.5 Multi-region event mirroring

**Architecture.**
```
region A broker ──federation/shovel──▶ region B broker
  same topic exchange replicated; consumers in B consume mirrored events
```

**RabbitMQ usage.**
- Federation for cross-region pub/sub with loose coupling.
- Shovels for one-shot migrations.

**Trade-offs.**
- WAN latency makes confirms slow — don't use for hot paths.
- Conflict resolution if both regions can produce; usually mirror is one-directional per event class.

---

## 17. Edge Cases Senior Developers Must Know

### 17.1 Message duplication

**Behavior.** RabbitMQ at-least-once means duplicates can appear after consumer crashes, producer retries on confirm failure, or network blips.

**Fix.**
- Stable `messageId` per logical event.
- Dedup store (Redis SET with TTL or a unique-constraint table).
- Naturally idempotent operations where possible.

### 17.2 Consumer crashes mid-processing

**Behavior.** Unacked deliveries are returned to the queue when the channel/connection closes. The next consumer sees `redelivered=true`. Work may have been partially done.

**Fix.**
- Treat partial state as the default. Always design for "this might be a redelivery."
- Use idempotency keys.
- Avoid non-idempotent side effects without dedup.

### 17.3 Network partitions in a cluster

**Behavior.** Classic mirrored queues used to have several partition handling modes (`pause_minority`, `autoheal`, `ignore`), each with edge-case data loss. Quorum queues (Raft) tolerate partitions natively but require a majority of nodes available to make progress.

**Fix.**
- Use **quorum queues** for important data.
- Plan for a node loss in a 3-node cluster: 2 healthy = still working.
- Don't run a 2-node cluster — it cannot tolerate any partition.

### 17.4 Unacked messages forever

**Behavior.** A consumer holding an unacked message and never crashing also never acking is invisible to RabbitMQ. The message appears as "unacked" indefinitely until the channel is closed.

**Fix.**
- Set **consumer timeouts** at the client (or use `consumer_timeout` broker setting since 3.12 — defaults to 30 minutes).
- Time-bound your processing; if it must take longer, design batched ack or split the work.
- Alert on growing `messages_unacknowledged` that doesn't drain.

### 17.5 Infinite retry loops

**Behavior.** A poison message + `nack(requeue=true)` (or a misconfigured DLX that loops back without TTL/cap) → CPU pegged, queue stuck.

**Fix.**
- Cap retries via `x-death` count check.
- Always have a parking lot.
- Alert on retry count spikes per message ID.

### 17.6 Broker memory / disk alarms

**Behavior.** When `mem_used > vm_memory_high_watermark` or disk falls below `disk_free_limit`, the broker **blocks publishers** (`flow` state). Consumers continue. If consumers are also slow, the system stalls.

**Fix.**
- Alert before alarms fire (e.g., at 70% of the watermark).
- Set sensible queue length limits with `overflow: drop-head` for non-critical streams.
- Provision local SSDs; avoid throttled storage.

### 17.7 Channel closed unexpectedly (PRECONDITION_FAILED, NOT_FOUND, ACCESS_REFUSED)

**Behavior.** Any error on a channel closes it. Clients need to reopen. If the application doesn't recover correctly, all consumers on that channel stop.

**Fix.**
- Use Spring AMQP's auto-recovery (default on).
- Don't catch and ignore channel errors.
- Watch `channel.shutdownListener` events.

### 17.8 Reconnection storms after a broker failover

**Behavior.** All clients reconnect simultaneously, each opens many channels, broker CPU spikes, more clients time out, retry storm.

**Fix.**
- Jittered backoff on reconnection.
- Spread clients across multiple connections rather than one fat connection per pod.
- Use a load balancer in front for cluster nodes.

### 17.9 Queue declaration drift

**Behavior.** Service A declares `q.work` with `x-message-ttl=60000`. Service B declares the same queue without TTL. Whichever runs second fails with PRECONDITION_FAILED. Or worse: arguments diverge across environments.

**Fix.**
- One owner per queue/exchange. Other services bind/consume but don't redeclare with different args.
- Treat topology as code; review changes.

### 17.10 Mirrored queue (legacy) deprecation

**Behavior.** Classic mirrored queues are deprecated. Migrate to quorum queues.

**Fix.**
- Plan a migration: quorum queue + consumer cutover + drain old queue + delete.
- Quorum queues lack some classic features (priority, per-message TTL has caveats) — verify your features are supported.

### 17.11 Direct reply-to misuse

**Behavior.** Direct reply-to (`amq.rabbitmq.reply-to`) is a pseudo-queue for RPC responses. It works **only on the same channel that consumed it**, and only with `autoAck`.

**Fix.**
- Don't try to use it like a normal queue.
- Use a single dedicated channel for RPC responses per client process.

---

## 18. Final Checklist

**You are a senior-level RabbitMQ engineer if you can:**

- Explain in one paragraph how a message flows from producer to consumer including exchange routing, persistence, and acknowledgment, and identify every point at which it can be lost or duplicated.
- Choose between direct, topic, fanout, and headers exchanges for a given problem and justify the choice with concrete trade-offs.
- Configure a Spring Boot service with publisher confirms, mandatory routing, manual ack, prefetch, and JSON message conversion — from memory, well enough to get a working pom.xml + config + producer + consumer running.
- Design a delayed-retry topology using DLX + TTL ladders, explain when to use the delayed-message plugin instead, and add a parking lot DLQ at the end.
- Distinguish at-most-once, at-least-once, and effectively-once delivery — and implement effectively-once via outbox + idempotent consumer.
- Explain why exactly-once delivery is not actually achievable end-to-end without consumer-side deduplication, and what role `messageId` plays in dedup.
- Tune prefetch and consumer concurrency from queue metrics; describe the failure mode of "prefetch too high" vs "prefetch too low."
- Read a queue's runtime metrics (`messages_ready`, `messages_unacknowledged`, `consumer_count`) and diagnose the most likely cause of each anomaly.
- Set up Prometheus + Grafana monitoring and define the top five alerts for a production RabbitMQ.
- Recognize all the situations in which a message can be silently dropped (no binding match without `mandatory`, non-persistent on durable queue restart, autoAck + crash) and prevent each one.
- Compare RabbitMQ to Kafka, Streams, and SQS, picking the right tool for a given use case (task queue, event stream, replay, managed simplicity).
- Identify and break a poison message retry loop using `x-death` count caps and a parking lot.
- Explain the difference between classic queues, quorum queues, and streams; choose one per use case and justify it.
- Design a multi-service event-driven system with a clear topic taxonomy, an event catalog, additive schema evolution, and idempotent consumers.
- Operate a cluster: handle network partitions, broker memory/disk alarms, reconnection storms, and queue declaration drift.
- Audit a security configuration: vhost isolation, per-service permissions, TLS, secrets handling, default `guest` user disabled.
- Walk a junior engineer through every common pitfall — message loss, duplicates, queue buildup, poor routing, infinite retries — and explain both *why* it happens and *how* to fix it without hand-waving.

If you can do all of the above without looking anything up, you are ready to design, ship, and run RabbitMQ in production at a senior level.
