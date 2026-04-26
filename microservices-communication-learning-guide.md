# Network Communication in Spring Boot Microservices — Production-Grade Learning Guide

> A senior-engineer's reference for designing, implementing, and reasoning about service-to-service communication in real distributed systems.
>
> **Audience:** Java backend engineers with ~3 years of Spring Boot experience who are comfortable with REST but want to reason about protocols, patterns, and trade-offs at the architect level.

---

## Table of Contents

1. [Microservices Communication Fundamentals](#1-microservices-communication-fundamentals)
2. [Protocols Overview](#2-protocols-overview)
3. [HTTP / REST](#3-http--rest)
4. [gRPC](#4-grpc)
5. [GraphQL](#5-graphql)
6. [WebSocket](#6-websocket)
7. [Server-Sent Events (SSE)](#7-server-sent-events-sse)
8. [RSocket](#8-rsocket)
9. [Messaging Systems](#9-messaging-systems)
10. [Spring Integration](#10-spring-integration)
11. [Service Discovery & Routing](#11-service-discovery--routing)
12. [Config & Coordination](#12-config--coordination)
13. [Communication Patterns](#13-communication-patterns)
14. [Comparison Table](#14-comparison-table)
15. [Production Best Practices](#15-production-best-practices)
16. [Real-World Architectures](#16-real-world-architectures)
17. [When NOT to Use Certain Tools](#17-when-not-to-use-certain-tools)
18. [Final Checklist](#18-final-checklist)

---

## 1. Microservices Communication Fundamentals

### 1.1 Why Communication Is Hard in Distributed Systems

In a monolith, a method call is:
- Atomic
- Deterministic
- In-memory (nanoseconds)
- Either succeeds or throws

In a distributed system, a "method call" across services becomes a network operation:
- **Latency** can range from 1ms (same AZ) to 200ms+ (cross-region)
- **Partial failures** — the remote service may have processed the request but your response was lost
- **Nondeterminism** — same request can succeed once, fail next time
- **Unknown state** — after a timeout, *you don't know* if the work was done

The **Fallacies of Distributed Computing** (Peter Deutsch) are the bedrock:

1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology doesn't change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

Every communication decision is ultimately a reaction to one or more of these being false.

### 1.2 Synchronous vs Asynchronous Communication

| Aspect | Synchronous | Asynchronous |
|---|---|---|
| Caller waits? | Yes | No |
| Coupling | Temporal — both services must be up | Decoupled — broker buffers |
| Failure mode | Caller fails if callee fails | Caller is safe; message sits in queue |
| Latency visibility | Immediate | Deferred / eventual |
| Example | REST, gRPC, GraphQL | Kafka, RabbitMQ, SQS |

**Rule of thumb:** if the user is waiting on the result *right now*, it's synchronous. If the work can be done later without blocking a user response, make it asynchronous.

### 1.3 Request-Response vs Event-Driven

- **Request-Response:** "Do X and tell me the result." Caller knows the callee. Tight contract.
- **Event-Driven:** "X happened." Publisher does not know who consumes. Loose coupling, inversion of control at the system level.

Event-driven flips the direction of knowledge. `OrderCreated` doesn't know about `InventoryService` or `EmailService` — those services *subscribe*. This is how you scale teams: new consumers can be added without touching the producer.

### 1.4 Latency, Reliability, Coupling

These three form the **CAP-adjacent triangle** of communication design:

- **Low latency** often means synchronous and in-memory — which couples services temporally.
- **High reliability** often means asynchronous with durable queues — which adds latency.
- **Low coupling** often means event-driven — which sacrifices strong consistency.

You rarely get all three. Pick based on business requirements.

### 1.5 The Partial Failure Problem

The defining problem of distributed systems. Consider:

```
Service A ---HTTP POST---> Service B
                            [processes, writes DB]
                            [tries to return 200]
                            [network drops the response]
Service A: timeout. Did B process it or not?
```

Options:
1. **Retry** — may cause duplicate processing (needs idempotency)
2. **Give up** — may lose work
3. **Query B's state** — needs B to expose a way to ask "did you see request X?"

There is no "correct" answer without business context. This is why idempotency keys, transactional outboxes, and sagas exist.

### 1.6 Retries, Timeouts, Circuit Breakers (Conceptual)

**Timeout:** Upper bound on how long you wait. Never make a network call without one. Default HTTP client timeouts are often infinite or minutes long — cascade failures start here.

```
Rule: every network call has:
  - connect timeout (e.g., 2s)
  - read/request timeout (e.g., 5s)
  - total/overall timeout (SLA budget minus headroom)
```

**Retry:** Re-attempt on transient failure. Must be paired with:
- **Backoff** (exponential with jitter) — prevent thundering herd
- **Budget** (max attempts) — don't retry forever
- **Idempotency** — avoid duplicate side effects

**Circuit Breaker:** Fail fast when a downstream is unhealthy. States:
- `CLOSED` — requests flow normally
- `OPEN` — fail immediately, don't even try
- `HALF_OPEN` — let a trickle through to test recovery

Without circuit breakers, one slow dependency takes down your entire thread pool. Resilience4j is the standard in Spring.

---

## 2. Protocols Overview

### 2.1 HTTP/1.1 vs HTTP/2

**HTTP/1.1:**
- Text-based, human-readable
- One request per connection at a time (head-of-line blocking)
- Workaround: multiple TCP connections per host (browsers use ~6)
- Massive ecosystem, universal support

**HTTP/2:**
- Binary framing (not text)
- Multiplexing — multiple parallel streams over one TCP connection
- Header compression (HPACK)
- Server push (rarely used now)
- Still suffers TCP-level head-of-line blocking (fixed in HTTP/3/QUIC)

**Implication for microservices:** HTTP/2 is massively better for high-RPS service-to-service traffic. It's what gRPC requires.

### 2.2 WebSocket

- Upgrades an HTTP connection to a persistent, full-duplex TCP channel
- Both client and server can send messages anytime
- No request/response semantics built in — you define your own protocol on top (e.g., STOMP)

**Use when:** chat, collaborative editing, live dashboards, games.

### 2.3 TCP vs UDP

- **TCP:** reliable, ordered, connection-oriented. Retransmits lost packets. Almost all business traffic.
- **UDP:** fire-and-forget, no delivery guarantees. Low latency. Used by DNS, video/voice (RTP), QUIC/HTTP3.

Most Spring Boot apps never see raw TCP/UDP — you use protocols built on TCP (HTTP, AMQP, Kafka).

### 2.4 AMQP (Advanced Message Queuing Protocol)

- Protocol used by RabbitMQ (and others)
- Explicit concepts: exchanges, queues, bindings, routing keys
- Per-message acknowledgments
- Strong at *complex routing* and *work queues*

### 2.5 Kafka Protocol

- Custom binary protocol over TCP
- Not "messaging" in the AMQP sense — it's a **distributed commit log**
- Messages persist on disk; consumers track their own offset
- Strong at *high-throughput*, *replay*, *event sourcing*

### 2.6 MQTT

- Pub/sub over TCP, designed for constrained devices (IoT)
- Tiny header, QoS levels (0/1/2)
- Widely used: sensors, vehicles, home automation

### 2.7 Protobuf

- Not a transport — a **serialization format**
- Binary, schema-first (`.proto` files)
- Smaller and faster than JSON
- Used by gRPC, but also standalone in Kafka messages, config, etc.

### 2.8 Protocol Quick-Pick Matrix

| Protocol | Transport | Pattern | When to use |
|---|---|---|---|
| HTTP/1.1 | TCP | Req/Resp | Public APIs, low RPS internal, legacy |
| HTTP/2 | TCP | Req/Resp, streaming | Internal high-RPS, gRPC |
| WebSocket | TCP (upgraded HTTP) | Full-duplex | Chat, live UIs |
| AMQP | TCP | Messaging | Work queues, complex routing |
| Kafka | TCP (custom) | Log / pub-sub | Event streaming, high throughput |
| MQTT | TCP | Pub/sub | IoT, constrained networks |
| Protobuf | (any) | Serialization | Efficient wire format |

---

## 3. HTTP / REST

REST over HTTP is the default. It should remain the default unless you have a reason to pick something else.

### 3.1 The Four Spring HTTP Clients

| Client | Blocking? | Reactive? | Status |
|---|---|---|---|
| `RestTemplate` | Yes | No | Maintenance mode since Spring 5. Avoid for new code. |
| `WebClient` | No | Yes (WebFlux) | Modern non-blocking default |
| `RestClient` | Yes | No | Modern blocking default (Spring 6.1+) |
| OpenFeign (`@FeignClient`) | Both | Both (with reactor) | Declarative abstraction |

### 3.2 RestTemplate (Legacy)

```java
@Service
public class LegacyOrderClient {
    private final RestTemplate rt = new RestTemplate();

    public Order fetch(Long id) {
        return rt.getForObject("http://orders/api/v1/orders/{id}", Order.class, id);
    }
}
```

**When to use:** existing codebases only. **Do not start new projects with it.**

### 3.3 WebClient (Non-Blocking)

```java
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient ordersClient(WebClient.Builder builder) {
        return builder
            .baseUrl("http://orders")
            .defaultHeader(HttpHeaders.ACCEPT, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}

@Service
public class OrderClient {
    private final WebClient client;

    public Mono<Order> fetch(Long id) {
        return client.get()
            .uri("/api/v1/orders/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError,
                      r -> Mono.error(new OrderNotFoundException(id)))
            .bodyToMono(Order.class)
            .timeout(Duration.ofSeconds(3))
            .retryWhen(Retry.backoff(3, Duration.ofMillis(200))
                           .filter(ex -> ex instanceof IOException));
    }
}
```

**When to use:** WebFlux apps, high-concurrency clients, streaming responses, any service where thread-per-request is limiting.

### 3.4 RestClient (Modern Blocking)

```java
@Bean
public RestClient ordersRestClient() {
    return RestClient.builder()
        .baseUrl("http://orders")
        .defaultStatusHandler(HttpStatusCode::is5xxServerError,
            (req, res) -> { throw new DownstreamUnavailable(); })
        .build();
}

public Order fetch(Long id) {
    return restClient.get()
        .uri("/api/v1/orders/{id}", id)
        .retrieve()
        .body(Order.class);
}
```

**When to use:** new MVC (blocking) codebases. This is the direct replacement for `RestTemplate` with a fluent, modern API.

### 3.5 OpenFeign — Declarative

```java
@FeignClient(name = "orders", url = "${orders.url}")
public interface OrdersApi {
    @GetMapping("/api/v1/orders/{id}")
    Order getOrder(@PathVariable Long id);

    @PostMapping("/api/v1/orders")
    Order createOrder(@RequestBody CreateOrderRequest req);
}
```

```yaml
feign:
  client:
    config:
      orders:
        connectTimeout: 2000
        readTimeout: 5000
```

**When to use:** you value readability and consistency across many clients; you're already on Spring Cloud; you want interface-driven integration tests. **When not to use:** you need fine-grained streaming/reactive control.

### 3.6 Choosing a Client — Decision Tree

```
Is the app reactive (WebFlux)?
  yes → WebClient
  no  → Do you want declarative interfaces?
         yes → OpenFeign
         no  → RestClient
```

### 3.7 HTTP/1.1 vs HTTP/2 in Practice

- Enable HTTP/2 on the server: `server.http2.enabled=true` (needs TLS for most clients).
- HTTP/2 shines for *many concurrent requests to the same host* — exactly the microservices pattern.
- WebClient (Netty/Reactor Netty) supports HTTP/2 trivially. RestTemplate does not.

### 3.8 REST Best Practices (Condensed)

- Resource-oriented URIs: `/orders/42`, not `/getOrder?id=42`
- Use HTTP verbs correctly: GET (safe, idempotent), PUT/DELETE (idempotent), POST (not)
- Version via URL (`/v1/`) or header — pick one and stick with it
- Use proper status codes: `201 Created`, `204 No Content`, `409 Conflict`, `422 Unprocessable Entity`
- Return **RFC 7807 Problem Details** for errors, not free-form JSON
- Support pagination (`?page=`, `?cursor=`) and never return unbounded lists
- `ETag` + `If-None-Match` for caching
- `Idempotency-Key` header for non-idempotent `POST` in financial/critical flows

### 3.9 Error Handling

```java
@RestControllerAdvice
public class GlobalErrors extends ResponseEntityExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ProblemDetail handleNotFound(OrderNotFoundException e) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(
            HttpStatus.NOT_FOUND, e.getMessage());
        pd.setType(URI.create("https://errors.example.com/order-not-found"));
        return pd;
    }
}
```

---

## 4. gRPC

gRPC is **HTTP/2 + Protobuf + code generation**. It's the go-to for internal, high-performance service-to-service calls.

### 4.1 Why gRPC Exists

REST+JSON is great for browsers and external APIs, but for internal service-to-service traffic:
- JSON parsing is slow (CPU-heavy)
- No strong schema → drift between services
- No bidirectional streaming
- HTTP/1.1 overhead at high RPS

gRPC fixes all four.

### 4.2 Starter

```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
```

### 4.3 Proto Definition

```proto
syntax = "proto3";

package orders.v1;
option java_package = "com.example.orders.grpc";
option java_multiple_files = true;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);
  rpc CreateOrders(stream CreateOrderRequest) returns (CreateOrdersSummary);
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
}

message GetOrderRequest { int64 id = 1; }
message Order {
  int64 id = 1;
  string customer_id = 2;
  double total = 3;
  Status status = 4;
  enum Status { PENDING = 0; PAID = 1; SHIPPED = 2; }
}
```

### 4.4 Server Implementation

```java
@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    private final OrderRepository repo;

    @Override
    public void getOrder(GetOrderRequest req, StreamObserver<Order> res) {
        repo.findById(req.getId())
            .map(this::toProto)
            .ifPresentOrElse(
                o -> { res.onNext(o); res.onCompleted(); },
                () -> res.onError(Status.NOT_FOUND
                        .withDescription("order " + req.getId())
                        .asRuntimeException())
            );
    }
}
```

### 4.5 Client Implementation

```java
@Service
public class OrderGrpcClient {

    @GrpcClient("orders")
    private OrderServiceGrpc.OrderServiceBlockingStub stub;

    public Order get(long id) {
        return stub.withDeadlineAfter(2, TimeUnit.SECONDS)
                   .getOrder(GetOrderRequest.newBuilder().setId(id).build());
    }
}
```

```yaml
grpc:
  client:
    orders:
      address: 'static://orders-service:9090'
      negotiationType: plaintext
```

### 4.6 gRPC vs REST

| Dimension | REST/JSON | gRPC |
|---|---|---|
| Wire format | Text (JSON) | Binary (Protobuf) |
| Transport | HTTP/1.1 or 2 | HTTP/2 only |
| Schema | Optional (OpenAPI) | Mandatory (.proto) |
| Streaming | Limited (SSE) | First-class, bidirectional |
| Browser support | Native | Needs gRPC-Web proxy |
| Debug tooling | curl, Postman | grpcurl, BloomRPC |
| Payload size | 3–10x larger | Smallest practical |
| CPU | Higher (parse JSON) | Lower |

### 4.7 When to Use gRPC

- Internal service-to-service with clear schemas
- High RPS / low-latency paths
- Polyglot stacks (Go, Java, Python speaking same contract)
- Streaming: telemetry, live data, chat, AI token streaming

### 4.8 When NOT to Use gRPC

- Public APIs for browsers or third parties (use REST or GraphQL)
- Teams unfamiliar with Protobuf tooling
- Environments without HTTP/2 (old proxies, some PaaS)
- Simple CRUD with 10 RPS — the complexity isn't worth it

---

## 5. GraphQL

GraphQL is a **query language** for APIs. The client specifies exactly what it needs; the server resolves it.

### 5.1 Starter

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
```

### 5.2 Schema (`src/main/resources/graphql/schema.graphqls`)

```graphql
type Query {
  order(id: ID!): Order
  orders(status: Status, first: Int = 20, after: String): OrderConnection!
}

type Mutation {
  createOrder(input: CreateOrderInput!): Order!
}

type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}

type Order {
  id: ID!
  customer: Customer!
  items: [LineItem!]!
  total: Float!
  status: Status!
}

enum Status { PENDING PAID SHIPPED }
```

### 5.3 Resolvers

```java
@Controller
public class OrderGraphqlController {

    @QueryMapping
    public Order order(@Argument Long id) {
        return orderService.get(id);
    }

    @SchemaMapping(typeName = "Order", field = "customer")
    public Customer customer(Order order) {
        return customerService.get(order.getCustomerId());  // batch via DataLoader
    }

    @MutationMapping
    public Order createOrder(@Argument CreateOrderInput input) {
        return orderService.create(input);
    }

    @SubscriptionMapping
    public Flux<Order> orderStatusChanged(@Argument Long orderId) {
        return orderService.streamChanges(orderId);
    }
}
```

### 5.4 HTTP vs WebSocket

- **Queries + Mutations:** plain HTTP POST to `/graphql`
- **Subscriptions:** WebSocket (GraphQL over WebSocket protocol, `/graphql`)

### 5.5 When to Use GraphQL

- Frontends need flexible, over-fetch-free queries (mobile, multi-team UIs)
- Aggregating multiple backend services in a single call (BFF)
- Rapidly changing UI requirements

### 5.6 When NOT to Use GraphQL

- Simple server-to-server (gRPC/REST fit better)
- You can't control query complexity (risk of expensive queries)
- You need strong HTTP caching semantics (GraphQL POST defeats it)
- Your team has no GraphQL experience and deadlines are tight

---

## 6. WebSocket

### 6.1 What It Is

A persistent, full-duplex TCP connection between client and server, upgraded from HTTP. Once open, either side can send frames freely.

### 6.2 Spring Setup (STOMP)

```java
@Configuration
@EnableWebSocketMessageBroker
public class WsConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry reg) {
        reg.addEndpoint("/ws").setAllowedOriginPatterns("*").withSockJS();
    }

    @Override
    public void configureMessageBroker(MessageBrokerRegistry reg) {
        reg.enableSimpleBroker("/topic", "/queue");
        reg.setApplicationDestinationPrefixes("/app");
    }
}
```

### 6.3 Chat Example

```java
@Controller
public class ChatController {

    @MessageMapping("/chat.send/{room}")
    @SendTo("/topic/room.{room}")
    public ChatMessage send(@DestinationVariable String room, ChatMessage msg, Principal user) {
        msg.setSender(user.getName());
        msg.setTimestamp(Instant.now());
        return msg;
    }
}
```

Client (JS):

```js
const client = new StompJs.Client({ brokerURL: 'ws://host/ws' });
client.onConnect = () => {
  client.subscribe('/topic/room.lobby', m => render(JSON.parse(m.body)));
  client.publish({ destination: '/app/chat.send/lobby',
                   body: JSON.stringify({ text: 'hi' }) });
};
client.activate();
```

### 6.4 WebSocket vs REST

| | REST | WebSocket |
|---|---|---|
| Direction | Client → Server | Bidirectional |
| Connection | Per-request | Long-lived |
| Overhead per msg | Full HTTP headers | Few bytes |
| Load balancing | Stateless, trivial | Sticky/sharded sessions needed |
| Use case | CRUD, most APIs | Live chat, live feeds |

### 6.5 When to Use

- Chat, collaboration, multiplayer
- Live trading/betting dashboards
- Real-time notifications with frequent client→server interaction

### 6.6 When NOT to Use

- One-way server push only → prefer SSE (simpler, HTTP-native)
- Simple polling every few minutes → plain REST
- Stateless, cache-friendly APIs

---

## 7. Server-Sent Events (SSE)

One-way streaming from server to client over a plain HTTP connection. Each event is a line of text.

### 7.1 WebFlux SSE

```java
@GetMapping(value = "/stocks/{symbol}", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<ServerSentEvent<Quote>> stream(@PathVariable String symbol) {
    return quoteService.stream(symbol)
        .map(q -> ServerSentEvent.<Quote>builder()
            .id(q.id())
            .event("quote")
            .data(q)
            .build());
}
```

### 7.2 MVC `SseEmitter`

```java
@GetMapping("/notifications")
public SseEmitter subscribe(Principal user) {
    SseEmitter emitter = new SseEmitter(Duration.ofMinutes(30).toMillis());
    notificationBus.register(user.getName(), event -> {
        try { emitter.send(SseEvent.builder().name(event.type()).data(event).build()); }
        catch (IOException e) { emitter.completeWithError(e); }
    });
    return emitter;
}
```

### 7.3 SSE vs WebSocket

| | SSE | WebSocket |
|---|---|---|
| Direction | Server → Client | Bidirectional |
| Protocol | Plain HTTP | Upgraded, custom |
| Proxy/firewall friendly | Very | Sometimes blocked |
| Auto-reconnect | Built-in in browsers | DIY |
| Binary | No (text only) | Yes |

### 7.4 When to Use SSE

- Live feeds, progress updates, AI token streaming, log tailing
- When the client only consumes

### 7.5 When NOT to Use SSE

- Need client-to-server messages on the same channel → WebSocket
- Binary payloads

---

## 8. RSocket

RSocket is a **reactive, message-oriented binary protocol** — designed from scratch for microservices on top of Reactive Streams. Ships with Spring Boot.

### 8.1 The Four Interaction Models

| Mode | Direction | Example |
|---|---|---|
| Request/Response | 1 → 1 | Like REST call, but reactive |
| Fire-and-Forget | 1 → 0 | Metric emit, log event |
| Request/Stream | 1 → N | "Give me all events since X" |
| Channel | N ↔ N | Bidirectional streams (like gRPC bidi) |

### 8.2 Starter

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-rsocket</artifactId>
</dependency>
```

### 8.3 Server

```java
@Controller
public class QuoteRSocketController {

    @MessageMapping("quote.get")
    public Mono<Quote> get(String symbol) {
        return quoteService.current(symbol);
    }

    @MessageMapping("quote.stream")
    public Flux<Quote> stream(String symbol) {
        return quoteService.stream(symbol);
    }

    @MessageMapping("quote.fire")
    public Mono<Void> logClientTick(ClientTick t) {
        metrics.record(t);
        return Mono.empty();
    }

    @MessageMapping("quote.channel")
    public Flux<Quote> channel(Flux<String> symbols) {
        return symbols.flatMap(quoteService::stream);
    }
}
```

```yaml
spring.rsocket.server.port: 7000
spring.rsocket.server.transport: tcp
```

### 8.4 Client

```java
@Bean
public RSocketRequester quoteRequester(RSocketRequester.Builder b) {
    return b.tcp("quote-service", 7000);
}

public Flux<Quote> stream(String symbol) {
    return requester.route("quote.stream").data(symbol).retrieveFlux(Quote.class);
}
```

### 8.5 When to Use RSocket

- All-reactive stacks
- Need multiple interaction models on one connection
- Back-pressure matters (Reactive Streams is the protocol, not an afterthought)
- Mobile/IoT backends — survives transport changes (TCP, WebSocket)

### 8.6 When NOT to Use RSocket

- Teams unfamiliar with reactive programming
- Ecosystem tooling is thinner than REST/gRPC
- Public APIs — no standard browser support

---

## 9. Messaging Systems

Asynchronous, decoupled communication. The big lever for scaling teams and systems.

### 9.1 Apache Kafka

A **distributed, partitioned, replicated commit log**. Not a queue — messages are retained (days to forever) and consumers track their own position.

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

#### Producer

```java
@Service
@RequiredArgsConstructor
public class OrderEventPublisher {
    private final KafkaTemplate<String, OrderCreated> template;

    public void publish(OrderCreated evt) {
        template.send("orders.v1.created", evt.orderId().toString(), evt);
    }
}
```

#### Consumer

```java
@Component
public class InventoryListener {

    @KafkaListener(topics = "orders.v1.created", groupId = "inventory")
    public void onOrderCreated(OrderCreated evt, Acknowledgment ack) {
        inventory.reserve(evt);
        ack.acknowledge();
    }
}
```

```yaml
spring:
  kafka:
    bootstrap-servers: kafka:9092
    consumer:
      enable-auto-commit: false
      group-id: inventory
    listener:
      ack-mode: manual
```

#### Architecture

- **Topic** — named log, split into **partitions**
- **Partition** — ordered, append-only sequence; one consumer per partition per group
- **Consumer group** — parallel consumers sharing the workload
- **Offset** — consumer's position in the partition

**Use when:**
- High throughput (100k+ msg/s)
- Event sourcing, replay, audit
- Fan-out to many consumers

**Don't use when:**
- You need complex routing by message content (RabbitMQ fits better)
- You need per-message delivery retry with DLQ and priorities — Kafka can do it but it's more work than AMQP
- Low throughput, simple work queue — operational overhead not worth it

### 9.2 RabbitMQ (AMQP)

Traditional message broker with rich routing.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

#### Concepts

- **Exchange** — receives messages, routes to queues
- **Queue** — consumers pull from here
- **Binding** — rule connecting exchange to queue (with routing key / pattern)
- **Routing key** — message attribute used by exchange to decide

Exchange types: `direct`, `topic`, `fanout`, `headers`.

#### Example

```java
@Configuration
public class RabbitConfig {
    @Bean TopicExchange orders() { return new TopicExchange("orders"); }
    @Bean Queue inventoryQueue() { return new Queue("inventory.orders"); }
    @Bean Binding bind(Queue inventoryQueue, TopicExchange orders) {
        return BindingBuilder.bind(inventoryQueue).to(orders).with("order.created.#");
    }
}

@Service
@RequiredArgsConstructor
public class OrderPublisher {
    private final RabbitTemplate template;
    public void send(OrderCreated e) {
        template.convertAndSend("orders", "order.created.eu", e);
    }
}

@RabbitListener(queues = "inventory.orders")
public void handle(OrderCreated e) { inventory.reserve(e); }
```

**Use when:**
- Work queues with retries + DLQ
- Complex routing (by type, region, priority)
- Moderate throughput

**Don't use when:**
- You need replay / long retention — Kafka does this natively
- Extreme throughput — RabbitMQ caps lower than Kafka

### 9.3 JMS (ActiveMQ / IBM MQ)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
```

```java
@JmsListener(destination = "orders.created")
public void on(OrderCreated e) { ... }
```

**Use when:** legacy enterprise integration (IBM MQ, TIBCO), regulated environments, XA transactions needed.

**Don't use when:** cloud-native greenfield — Kafka or a managed SQS is usually better.

### 9.4 MQTT

```xml
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-mqtt</artifactId>
</dependency>
```

```java
@Bean
public IntegrationFlow mqttIn(MqttPahoClientFactory f) {
    return IntegrationFlow.from(new MqttPahoMessageDrivenChannelAdapter(
                "ingest", f, "sensors/+/temp"))
            .handle(msg -> ingest.store((String) msg.getPayload()))
            .get();
}
```

**Use when:** IoT devices, unreliable networks, battery-constrained clients.

**Don't use when:** pure backend traffic — it offers nothing over Kafka/AMQP there.

### 9.5 AWS Messaging — SQS & SNS

```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sqs</artifactId>
</dependency>
```

```java
@SqsListener("orders-queue")
public void on(OrderCreated e) { ... }
```

- **SQS** — managed queue (standard + FIFO)
- **SNS** — pub/sub fan-out (HTTP, SQS, Lambda, email)
- **Classic pattern:** SNS → multiple SQS queues (one per consumer group)

**Use when:** AWS-native, want zero ops, moderate throughput.

**Don't use when:** you need replay, ordering across all messages (FIFO is per-group), or very high throughput at very low cost — Kafka wins at scale.

### 9.6 Spring Cloud Stream

A thin abstraction over brokers (Kafka, RabbitMQ, SQS, Kinesis…). You write functions; bindings route them.

```java
@Bean
public Function<OrderCreated, ShipmentRequested> ship() {
    return e -> new ShipmentRequested(e.orderId(), e.address());
}
```

```yaml
spring:
  cloud:
    stream:
      bindings:
        ship-in-0:
          destination: orders.created
        ship-out-0:
          destination: shipments.requested
      kafka:
        binder:
          brokers: kafka:9092
```

**Use when:** you want broker portability, standardized event handling across many services.

**Don't use when:** you need broker-specific features (Kafka Streams, RabbitMQ dead-lettering policies) — drop to the native client.

---

## 10. Spring Integration

Enterprise Integration Patterns (Hohpe/Woolf) realized in Spring. Think: pipelines of *channels*, *endpoints*, *transformers*, *adapters*.

```java
@Bean
public IntegrationFlow ftpToKafka(FtpInboundFileSynchronizer sync,
                                  KafkaTemplate<String, String> kt) {
    return IntegrationFlow
        .from(Ftp.inboundAdapter(sessionFactory).remoteDirectory("/drop"),
              c -> c.poller(Pollers.fixedRate(30, TimeUnit.SECONDS)))
        .transform(File.class, f -> Files.readString(f.toPath()))
        .filter((String s) -> s.contains("ORDER"))
        .handle(Kafka.outboundChannelAdapter(kt).topic("orders.raw"))
        .get();
}
```

**Use when:** legacy systems, file/FTP/SFTP ingestion, complex routing across heterogeneous endpoints.

**Don't use when:** modern event-driven Kafka-first architecture — Spring Cloud Stream is leaner. Also: the learning curve is steep.

---

## 11. Service Discovery & Routing

### 11.1 Why

In dynamic environments (Kubernetes, ECS, Cloud Foundry) instances come and go. You cannot hard-code IPs.

### 11.2 Eureka (Netflix)

```yaml
eureka:
  client:
    service-url:
      defaultZone: http://eureka:8761/eureka/
```

Clients resolve `http://orders/...` via Eureka.

### 11.3 Consul

Consul adds KV store, health checks, and DNS-based discovery. Used where service mesh is absent.

### 11.4 Spring Cloud Gateway

API gateway: routing, filtering, auth, rate limiting, observability entry point.

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: orders
          uri: lb://orders
          predicates:
            - Path=/api/orders/**
          filters:
            - RewritePath=/api/(?<seg>.*), /$\{seg}
            - name: CircuitBreaker
              args:
                name: ordersCB
                fallbackUri: forward:/fallback/orders
```

### 11.5 Spring Cloud LoadBalancer

Replacement for Ribbon. Client-side load balancing; integrates with discovery clients. Usually transparent — just use the logical service name.

### 11.6 Kubernetes-Native Alternative

In K8s, you often skip Eureka entirely: K8s Services provide DNS + load balancing. A service mesh (Istio, Linkerd) handles retries, mTLS, observability at the infra level, leaving the app code simpler.

---

## 12. Config & Coordination

### 12.1 Spring Cloud Config

Centralized configuration backed by Git, Vault, or JDBC.

```yaml
spring:
  config:
    import: "configserver:http://config:8888"
  application:
    name: orders
  profiles:
    active: prod
```

Server serves `orders-prod.yml` from a Git repo. Clients pull at startup.

### 12.2 Spring Cloud Bus

Broadcasts config refresh events across services (over Kafka or RabbitMQ):

```
POST /actuator/busrefresh   → all instances refresh @RefreshScope beans
```

**Alternative in K8s:** ConfigMaps + reloader sidecars + a GitOps workflow (ArgoCD). Spring Cloud Config is less essential when K8s is already solving this.

---

## 13. Communication Patterns

### 13.1 Request-Response

Caller waits for a response. Default synchronous pattern (REST, gRPC, GraphQL).
- **Pros:** simple to reason about, immediate feedback
- **Cons:** temporal coupling, cascading failures

### 13.2 Event-Driven

Publisher emits events; consumers subscribe. No knowledge of who listens.
- **Pros:** decoupling, extensibility, natural async
- **Cons:** eventual consistency, harder to debug end-to-end

### 13.3 Pub/Sub

Concrete shape of event-driven: topics + subscriptions. Kafka, SNS, RabbitMQ fanout exchange.

### 13.4 Streaming

Continuous flow of data, not discrete request/response. gRPC streams, Kafka consumers, RSocket streams, SSE.

- **Use:** telemetry, live feeds, log pipelines, LLM token streams

### 13.5 Saga Pattern (Overview)

Distributed transactions without 2PC. A saga is a **sequence of local transactions**, each with a **compensating action**.

Two styles:
- **Choreography** — services react to each other's events. No central brain. Hard to visualize.
- **Orchestration** — a saga orchestrator tells each service what to do next. Easier to reason about; central point.

Example: `CreateOrder` saga
```
1. OrderService.create()         → OrderCreated
2. PaymentService.authorize()    → PaymentAuthorized | PaymentFailed
3. InventoryService.reserve()    → InventoryReserved | InventoryFailed
4. ShippingService.schedule()    → ShipmentScheduled
   Compensations (reverse order) on any failure.
```

Use sagas instead of distributed transactions when crossing service boundaries. Pair with the **transactional outbox** pattern to reliably publish events.

---

## 14. Comparison Table

| Method | Protocol | Sync/Async | Perf | Complexity | Primary Use Cases |
|---|---|---|---|---|---|
| REST | HTTP/1.1–2 | Sync | Medium | Low | Public APIs, CRUD, broad compat |
| gRPC | HTTP/2 + Protobuf | Sync + streaming | Very high | Medium | Internal high-RPS, polyglot, streaming |
| GraphQL | HTTP(+WS) | Sync + sub | Medium | Medium–High | Flexible client queries, BFF |
| WebSocket | WS (TCP) | Async, duplex | High | Medium | Chat, real-time UI, collaboration |
| SSE | HTTP | Async, server→client | High | Low | Live feeds, LLM streams, progress |
| RSocket | TCP/WS | Async, 4 modes | Very high | Medium | Reactive stacks, back-pressure, mobile |
| Kafka | Custom/TCP | Async | Extreme | Medium–High | Event streams, replay, high throughput |
| RabbitMQ | AMQP | Async | High | Medium | Work queues, complex routing |
| SQS/SNS | HTTPS (AWS) | Async | High (managed) | Low | AWS-native fan-out / queues |
| JMS | Broker-specific | Async | Medium–High | Medium | Enterprise/legacy, XA transactions |
| MQTT | MQTT (TCP) | Async | High (tiny msgs) | Low–Medium | IoT, constrained devices |

---

## 15. Production Best Practices

### 15.1 Timeouts (Non-Negotiable)

- Always set **connect** and **read** timeouts
- Timeout budgets should cascade: if your SLA is 1s and you call 3 services, no call should allow more than ~300ms each
- Document timeouts next to SLAs

### 15.2 Retries

- Only retry **idempotent** operations, OR use an idempotency key
- Exponential backoff with **jitter** (prevent retry storms)
- Cap attempts (e.g., 3)
- Retry only on transient errors (5xx, connection reset, timeouts) — not on 4xx

```java
Retry retry = Retry.ofDefaults("orders");
CircuitBreaker cb = CircuitBreaker.ofDefaults("orders");
var decorated = Decorators.ofSupplier(() -> client.fetch(id))
    .withCircuitBreaker(cb)
    .withRetry(retry)
    .withFallback(List.of(Exception.class), ex -> Order.empty(id))
    .decorate();
```

### 15.3 Idempotency

- Expose an `Idempotency-Key` header on mutating endpoints
- Store key + result; return cached result on duplicate
- Essential for payments, orders, any external write with retries

### 15.4 Circuit Breakers

Resilience4j, wired via Spring Cloud Circuit Breaker or `@CircuitBreaker` annotation. Tune:
- `slidingWindowSize`
- `failureRateThreshold`
- `waitDurationInOpenState`
- `permittedNumberOfCallsInHalfOpenState`

### 15.5 Observability

The three pillars:
- **Logs** — structured JSON, include trace IDs
- **Metrics** — Micrometer → Prometheus (latency histograms, error rates, saturation)
- **Traces** — Micrometer Tracing + OpenTelemetry → Jaeger/Tempo/Zipkin

Propagate `traceparent` / `b3` headers through every client. Without distributed tracing you cannot debug production microservices.

### 15.6 Security

- **TLS everywhere** — even internal. Service mesh mTLS is ideal.
- **AuthN:** OAuth2/OIDC, JWTs between services, mTLS for infra-level identity
- **AuthZ:** deny-by-default; per-endpoint policies (Spring Security, OPA, JWT scopes)
- Never log tokens, PII, or full request bodies
- Validate at the gateway AND at the service — defense in depth

### 15.7 Contract Testing

- **Consumer-Driven Contracts** (Spring Cloud Contract, Pact) — prevent schema drift before deploy
- For gRPC, the `.proto` file is the contract; run breaking-change detection (e.g., Buf)

### 15.8 Schema Evolution

- Never remove fields; mark deprecated
- Never repurpose field numbers (Protobuf) or names (JSON)
- Use additive changes only; bump major version on breaking changes

---

## 16. Real-World Architectures

### 16.1 High-Load Microservices API (E-Commerce)

**Scenario:** 50+ services, 100k RPS at peak, mobile + web clients.

```
[ Mobile / Web ]
      │ HTTPS (REST or GraphQL)
      ▼
[ API Gateway (Spring Cloud Gateway) ]
      │
      ├── REST → User Service     (HTTP/2, JSON)
      ├── REST → Catalog Service  (HTTP/2, JSON)
      └── REST → Order Service    (HTTP/2, JSON)
                      │ gRPC (HTTP/2, Protobuf)
                      ▼
               Pricing Service  — internal, hot path
                      │
                      ▼
               Kafka: "orders.created"
               ├─► Inventory (reserve)
               ├─► Fulfillment (ship)
               ├─► Analytics (stream)
               └─► Notifications (email/SMS)
```

**Why:**
- REST/GraphQL at the edge — broad client compatibility
- gRPC internally for low-latency critical paths
- Kafka for fan-out, replay, analytics pipeline
- Circuit breakers at the gateway protect from cascade failures

### 16.2 Event-Driven System (Banking / Payments)

**Scenario:** strong audit requirements, eventual consistency acceptable between domains.

```
[ REST ] → Account Service ── writes DB
                          └─ outbox event ─► Kafka
                                              │
                                              ├─► Ledger (double-entry)
                                              ├─► Fraud Detection
                                              ├─► Notifications
                                              └─► Reporting Warehouse
```

**Why:**
- Transactional outbox ensures exactly-once event publication
- Kafka's retention is the audit log
- Sagas orchestrate multi-step flows (transfer = debit + credit + notify)

### 16.3 Real-Time System (Chat / Notifications)

```
[ Clients ] ←─ WebSocket ──► Gateway (Spring Cloud Gateway + STOMP)
                                │
                                ▼
                        Chat Service (WS + Redis pub/sub)
                                │
                                ▼ Kafka: "messages.sent"
                        ┌───────┴───────────┐
                        ▼                   ▼
                   Push Service      Archive Service
                 (FCM/APNs SSE)       (store to DB)
```

**Why:**
- WebSocket/STOMP for bidirectional chat
- Redis pub/sub for fanout between gateway instances (sticky sessions + cross-node delivery)
- Kafka decouples message archiving and push notifications from the hot path
- SSE for server-only push to web clients not needing bidirectional

### 16.4 IoT Telemetry Pipeline

```
[ Devices ] ── MQTT ──► MQTT Broker (HiveMQ / EMQX)
                                 │
                                 ▼
                     Spring Integration MQTT adapter
                                 │
                                 ▼
                           Kafka: "telemetry.raw"
                                 │
                    ┌────────────┼────────────┐
                    ▼            ▼            ▼
                 Stream         Alerts      Storage
               (Kafka Streams)  (Flink)     (TSDB)
```

### 16.5 Reactive Streaming Backend (Trading / LLM)

```
[ Clients ] ←─ RSocket (request/stream) ─► Quote Service (WebFlux + RSocket)
                                            │
                                            ▼ back-pressure aware
                                   Pricing engine (Reactor)
                                            │
                                            ▼
                                 Kafka: "prices.ticks"
```

**Why:**
- RSocket gives first-class streaming + back-pressure
- WebFlux avoids thread-per-request at 10k+ concurrent streams
- Kafka downstream for durability / replay

---

## 17. When NOT to Use Certain Tools

### 17.1 When REST Is Enough

- < 1000 RPS per service
- Public API or browser client
- Simple CRUD
- Team unfamiliar with HTTP/2, streaming, Protobuf

**Don't switch to gRPC "for performance" without measuring.** The cost of gRPC is real: tooling, debugging, browser bridge, HTTP/2 proxies. JSON parsing is rarely the bottleneck at typical business RPS.

### 17.2 When Kafka Is Overkill

- < 100 msg/s
- Simple "send and forget" queue with one producer / one consumer
- No replay requirement
- Team has no Kafka operational experience

Use SQS, RabbitMQ, or even a DB outbox + poll. Kafka has real operational cost (ZooKeeper/KRaft, partitions, consumer offsets, rebalance storms).

### 17.3 When gRPC Is Unnecessary

- External / public API
- Browser clients (needs gRPC-Web proxy)
- Human-debuggable APIs are a priority
- Schemas rarely change AND performance is not critical

### 17.4 When GraphQL Is Wrong

- Simple server-to-server calls
- You can't constrain query cost (exposed to untrusted clients without depth/complexity limits)
- HTTP caching matters (GET-based REST is cacheable; GraphQL POST is not)

### 17.5 When WebSocket Is Wrong

- One-way server push → SSE is simpler
- Infrequent updates (every few minutes) → polling is fine
- Stateless horizontal scaling is critical and you can't handle sticky sessions

### 17.6 When RSocket Is Wrong

- Non-reactive team
- You need broad client support (browsers, partners)
- Small ecosystem is a risk

### 17.7 When Service Mesh / Eureka Is Overkill

- Monolith or 2–3 services in Kubernetes — K8s DNS is sufficient
- No cross-cluster traffic, no fine-grained policy

### 17.8 The Meta-Rule

**Boring technology is a feature.** The operational, staffing, and debugging cost of every new protocol/broker/library is higher than it looks. Adopt novelty only when the pain of *not* having it is larger than the cost of running it.

---

## 18. Final Checklist

**You are production-ready in microservices communication if you can:**

- [ ] Explain why the network is unreliable and name three specific failure modes you'd design for
- [ ] Decide between synchronous and asynchronous for a given flow and defend the choice
- [ ] Choose between REST, gRPC, GraphQL, WebSocket, SSE, and RSocket based on concrete trade-offs
- [ ] Implement HTTP clients with `WebClient`, `RestClient`, and `@FeignClient` with proper timeouts, retries, and error handling
- [ ] Write a `.proto` file, generate server + client stubs, and explain why HTTP/2 matters
- [ ] Design a GraphQL schema for a BFF and know when NOT to expose GraphQL publicly
- [ ] Implement a STOMP-over-WebSocket chat and know when SSE is the better choice
- [ ] Explain all four RSocket interaction models and when each applies
- [ ] Design an event-driven flow with Kafka, including partitioning strategy and consumer groups
- [ ] Contrast Kafka vs RabbitMQ vs SQS and pick one based on traffic shape
- [ ] Implement the transactional outbox pattern and explain why it exists
- [ ] Describe a saga (choreography and orchestration) and when to use each
- [ ] Configure timeouts that respect an SLA budget across multiple hops
- [ ] Apply retries with backoff + jitter + idempotency safely
- [ ] Wire a circuit breaker with Resilience4j and reason about its thresholds
- [ ] Propagate trace IDs end-to-end and debug a latency spike via distributed tracing
- [ ] Enforce mTLS or JWT-based service identity and understand what each protects against
- [ ] Contract-test services to catch schema drift before deploy
- [ ] Draw the communication topology of a real system and justify every arrow
- [ ] Recognize when a "boring" REST + Postgres approach beats an event-driven rewrite

If you can tick every box above, you can hold your own in any microservices communication design review.
