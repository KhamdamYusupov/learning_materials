# Senior Backend Interview Guide (Java / Spring Boot)

> A production-grade technical handbook for Java backend engineers transitioning from mid-level to senior-level thinking. Every concept is framed through the lens of real systems, failure modes, and trade-offs — the way a senior engineer actually reasons.

---

## 1. Executive Summary

This guide is not a tutorial. It is a structured reference for how senior backend engineers think about systems in production.

**Who this is for:** A Java/Spring Boot engineer with ~2.5 years of experience who understands REST APIs, JPA, and basic networking, but wants to reason about systems the way a Staff or Senior engineer does — in terms of failure modes, resource contention, latency budgets, and operational cost.

**What separates a Mid-Level from a Senior engineer:**

| Dimension | Mid-Level Thinking | Senior-Level Thinking |
|-----------|-------------------|----------------------|
| API Design | "It returns 200 OK" | "What happens at p99? Under contention? During deployment?" |
| Error Handling | try-catch blocks | Circuit breakers, retry budgets, idempotency, partial failure |
| Database | "Use an index" | "What's the lock contention pattern? Read/write ratio? Connection pool sizing?" |
| Concurrency | "Use a thread pool" | "What's the optimal pool size given I/O wait ratio? What happens when the pool is saturated?" |
| Scaling | "Add more instances" | "Where's the bottleneck? Is it CPU, memory, I/O, or coordination?" |
| Debugging | "Check the logs" | "Correlate traces, check thread dumps, analyze GC logs, inspect TCP state" |
| Architecture | "Use microservices" | "What's the coordination cost? What consistency model do we need?" |

**Core themes throughout this guide:**
1. **Latency is a distribution, not a number.** p50 means nothing without p99.
2. **Every abstraction leaks.** Know what's underneath Spring Boot, Tomcat, JDBC, and the JVM.
3. **Failure is the default state.** Design for partial failure, not success.
4. **Resources are finite.** Threads, connections, memory, file descriptors — everything has a limit.
5. **Trade-offs are the job.** There is no "best" solution — only the best solution given constraints.

---

## 2. Mental Models for Senior Backend Engineers

### 2.1 The Resource Budget Model

Every request consumes a budget of finite resources:

```
Request Budget:
├── Thread (1 of N from pool)
├── DB Connection (1 of M from pool)
├── Memory (heap allocation)
├── Network socket (file descriptor)
├── CPU time (context switches)
└── Time (SLA budget, typically 200-500ms)
```

A senior engineer reasons about which resource will be exhausted first under load. That's your bottleneck.

**Example:** A Spring Boot service with `server.tomcat.threads.max=200` and `spring.datasource.hikari.maximum-pool-size=10` — under load, all 200 threads will compete for 10 DB connections. The DB pool is the bottleneck. 190 threads will be blocked, holding memory, doing nothing. This is thread starvation caused by pool misconfiguration.

### 2.2 The Latency Decomposition Model

Never say "the API is slow." Decompose:

```
Total Latency (p99: 850ms)
├── DNS resolution:           ~1ms (cached)
├── TCP handshake:            ~15ms
├── TLS handshake:            ~30ms (TLS 1.2, 2-RTT)
├── Request serialization:    ~2ms
├── Network transit:          ~10ms
├── Server queuing:           ~50ms (thread pool wait)
├── Business logic:           ~5ms
├── DB query:                 ~20ms
├── DB connection wait:       ~300ms  ← BOTTLENECK
├── Response serialization:   ~3ms
├── Network transit:          ~10ms
└── GC pause (occasional):   ~400ms  ← TAIL LATENCY CAUSE
```

The senior engineer identifies that p99 latency is dominated by DB connection pool contention and occasional GC pauses — not the query itself.

### 2.3 The Failure Propagation Model

```
Service A → Service B → Service C → Database

If Database is slow (not down, SLOW):
  → Service C threads block waiting
  → Service C thread pool saturates
  → Service B requests to C start timing out
  → Service B thread pool saturates
  → Service A requests to B start timing out
  → Service A thread pool saturates
  → ALL services are now down

This is cascading failure. It started with a slow database.
```

**Key insight:** A slow dependency is worse than a dead one. A dead dependency fails fast. A slow one holds threads hostage.

**Prevention:** Circuit breakers, timeouts, bulkheads, and backpressure.

### 2.4 The Consistency-Availability Spectrum

Stop thinking in binary (CP vs AP). Think in spectrum:

```
Strong Consistency ←──────────────────→ Eventual Consistency
     │                                        │
  Single-leader DB              Async replication + events
  Synchronous replication       Read-your-writes (session)
  2PC / Saga                    CRDTs
  Linearizable reads            Stale reads OK
     │                                        │
  Lower throughput                    Higher throughput
  Higher latency                      Lower latency
  Simpler reasoning                   Complex reasoning
```

**Senior-level framing in interviews:** "For this use case, we need read-after-write consistency for the user who made the change, but we can tolerate eventual consistency for other users observing it. This lets us use async replication with session affinity."

### 2.5 The Backpressure Model

```
Producer (1000 RPS) → Queue/Buffer → Consumer (500 RPS)

Without backpressure:
  Queue grows unbounded → OOM → crash

With backpressure options:
  1. Drop: Discard excess (metrics, logs)
  2. Sample: Process every Nth item
  3. Buffer with bound: Reject when full (HTTP 429)
  4. Slow the producer: TCP flow control, reactive streams
  5. Scale consumers: Auto-scaling
```

**Production rule:** If your producer can outpace your consumer, you MUST have a backpressure strategy. There are no exceptions.

### 2.6 The Blast Radius Model

Every change, every failure, every deployment has a blast radius. Senior engineers minimize it:

```
Blast Radius Reduction:
├── Feature flags (limit who sees new code)
├── Canary deployments (limit how many instances)
├── Bulkheads (limit which threads/pools are affected)
├── Circuit breakers (limit failure propagation)
├── Rate limiting (limit how many requests)
├── Timeouts (limit how long we wait)
└── Graceful degradation (limit feature scope)
```

---

## 3. Backend Communication Design Patterns

### 3.1 Request-Response

**Concept:** The most fundamental pattern. Client sends a request, blocks (or awaits), and receives a response. Synchronous by default in Spring MVC, but the underlying TCP connection is always full-duplex.

```
Client                    Server
  │                         │
  │──── HTTP Request ──────→│
  │                         │ (processing)
  │←── HTTP Response ───────│
  │                         │
```

**Spring Boot Implementation:**

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final OrderService orderService;

    // Constructor injection — always preferred over field injection
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<OrderResponse> createOrder(
            @Valid @RequestBody CreateOrderRequest request,
            @RequestHeader("X-Idempotency-Key") String idempotencyKey,
            @RequestHeader("X-Request-Id") String requestId) {

        MDC.put("requestId", requestId); // Distributed tracing context

        OrderResponse response = orderService.createOrder(request, idempotencyKey);

        return ResponseEntity
                .status(HttpStatus.CREATED)
                .header("X-Request-Id", requestId)
                .body(response);
    }
}
```

**Scaling implications:**
- Each in-flight request holds a Tomcat thread. Default max is 200. At 200ms avg response time, max throughput = 200 / 0.2 = 1000 RPS.
- If downstream calls add latency, throughput drops proportionally.
- Scaling horizontally (more instances) is the typical solution, but it shifts the bottleneck to the database.

**Performance characteristics:**

| Metric | Typical Range | Concern Threshold |
|--------|--------------|-------------------|
| p50 latency | 5-50ms | >200ms |
| p99 latency | 50-200ms | >1s |
| Thread pool utilization | 20-60% | >80% |
| Error rate | <0.1% | >1% |

**Failure scenarios:**
1. **Thread pool exhaustion:** All 200 threads are blocked on slow downstream calls. New requests get queued, then rejected.
2. **Timeout misconfiguration:** Client timeout is 5s, but server-side processing takes 6s. The client retries, but the server is still processing the first request — doubling load.
3. **Missing idempotency:** Client retries after a network timeout. The server already processed the first request but the response was lost. Without idempotency keys, the order is created twice.

**Common mistakes:**
- Returning 200 OK for everything (even errors) — breaks client retry logic.
- Not setting explicit timeouts on RestTemplate/WebClient calls — defaults to "infinite."
- Ignoring request/response size — a 50MB JSON payload will blow up memory.

**What weak candidates miss:** They describe the happy path but cannot explain what happens when the response is lost mid-flight, or when the server processes the request but the client times out before receiving the response. This is the **"gray failure"** zone — the request might or might not have succeeded, and the client cannot know.

---

### 3.2 Synchronous vs Asynchronous Communication

**The real distinction is not about threads — it's about coupling.**

```
Synchronous (Temporal Coupling):
  Service A ──sync call──→ Service B
  A MUST wait for B. If B is down, A is down.
  A and B must be alive at the same time.

Asynchronous (Temporal Decoupling):
  Service A ──message──→ Queue ──message──→ Service B
  A fires and forgets. B processes later.
  A and B do NOT need to be alive at the same time.
```

**Java implementation — sync call with timeout:**

```java
@Configuration
public class RestClientConfig {

    @Bean
    public RestTemplate restTemplate() {
        var factory = new SimpleClientHttpRequestFactory();
        factory.setConnectTimeout(Duration.ofMillis(500));  // TCP connect
        factory.setReadTimeout(Duration.ofMillis(2000));     // Response wait
        return new RestTemplate(factory);
    }
}

@Service
public class PaymentService {

    private final RestTemplate restTemplate;

    public PaymentResult processPayment(PaymentRequest request) {
        try {
            ResponseEntity<PaymentResult> response = restTemplate.postForEntity(
                "https://payment-gateway/api/charge",
                request,
                PaymentResult.class
            );
            return response.getBody();
        } catch (ResourceAccessException e) {
            // Timeout or connection refused
            throw new PaymentUnavailableException("Payment gateway unreachable", e);
        }
    }
}
```

**Java implementation — async with message queue:**

```java
@Service
public class OrderService {

    private final RabbitTemplate rabbitTemplate;
    private final OrderRepository orderRepository;

    @Transactional
    public OrderResponse createOrder(CreateOrderRequest request, String idempotencyKey) {
        // 1. Persist order in PENDING state
        Order order = orderRepository.save(Order.pending(request, idempotencyKey));

        // 2. Publish event (NOT in the same transaction — use Transactional Outbox)
        rabbitTemplate.convertAndSend(
            "orders.exchange",
            "order.created",
            new OrderCreatedEvent(order.getId(), order.getItems())
        );

        // 3. Return immediately — payment processing happens async
        return OrderResponse.fromEntity(order);
    }
}

@Component
public class PaymentEventHandler {

    @RabbitListener(queues = "payment.process")
    public void handleOrderCreated(OrderCreatedEvent event) {
        // Process payment asynchronously
        // Update order status to PAID or FAILED
    }
}
```

**Trade-off table:**

| Dimension | Synchronous | Asynchronous |
|-----------|-------------|-------------|
| Latency | Immediate (if fast) | Delayed |
| Coupling | Temporal + spatial | Decoupled |
| Consistency | Strong (read-after-write) | Eventual |
| Error handling | Immediate feedback | Compensating transactions |
| Debugging | Easier (stack trace) | Harder (distributed trace) |
| Throughput ceiling | Limited by slowest call | Much higher |
| Data loss risk | Lower (fail fast) | Possible (queue loss) |

**Senior-level insight:** The choice is not sync vs async globally. It's per-interaction. User-facing request validation should be synchronous (immediate feedback). Payment processing can be async (user gets "processing" status). Email sending must be async (never block a request on SMTP).

---

### 3.3 Push vs Polling vs Long Polling

```
Polling:
Client ──GET──→ Server (no data)
Client ──GET──→ Server (no data)
Client ──GET──→ Server (data!)
Problem: Wasted requests, delayed data.

Long Polling:
Client ──GET──→ Server (holds connection open)
                Server (waits for data...)
                Server ──Response──→ Client (data!)
Client ──GET──→ Server (new long poll)
Better: Less waste, near real-time.

Push (SSE / WebSocket):
Server ──push──→ Client (data!)
Server ──push──→ Client (data!)
Best: Real-time, efficient.
```

**Long Polling in Spring Boot (DeferredResult):**

```java
@RestController
@RequestMapping("/api/v1/notifications")
public class NotificationController {

    // Thread-safe map of pending long-poll requests
    private final ConcurrentHashMap<String, DeferredResult<List<Notification>>>
        pendingRequests = new ConcurrentHashMap<>();

    @GetMapping("/{userId}")
    public DeferredResult<List<Notification>> pollNotifications(
            @PathVariable String userId) {

        // 30-second timeout — does NOT hold a Tomcat thread
        DeferredResult<List<Notification>> result = new DeferredResult<>(30_000L);

        result.onTimeout(() -> {
            pendingRequests.remove(userId);
            result.setResult(Collections.emptyList()); // Return empty on timeout
        });

        result.onCompletion(() -> pendingRequests.remove(userId));

        pendingRequests.put(userId, result);
        return result;
    }

    // Called internally when new data arrives
    public void pushNotification(String userId, Notification notification) {
        DeferredResult<List<Notification>> pending = pendingRequests.remove(userId);
        if (pending != null) {
            pending.setResult(List.of(notification));
        }
    }
}
```

**Why DeferredResult matters:** It releases the Tomcat thread back to the pool while the request stays open. Without it, long polling would hold threads hostage — 10,000 connected clients would need 10,000 threads.

**Comparison:**

| Aspect | Polling | Long Polling | SSE | WebSocket |
|--------|---------|-------------|-----|-----------|
| Latency | Polling interval | Near real-time | Real-time | Real-time |
| Server resources | Low per request | Medium (open conns) | Medium | High (stateful) |
| Scalability | Easy (stateless) | Moderate | Good | Hard (stateful) |
| Bi-directional | No | No | No (server→client) | Yes |
| Proxy/LB friendly | Yes | Mostly | Yes | Needs upgrade |
| Connection overhead | High (repeated TCP) | Medium | Low (reuse) | Low (persistent) |

---

### 3.4 Server-Sent Events (SSE)

**Concept:** Unidirectional server-to-client streaming over HTTP. The connection stays open, and the server pushes events as they occur. Built on standard HTTP — works through proxies and load balancers without special configuration (unlike WebSockets).

```
Client ──GET /stream──→ Server
         Content-Type: text/event-stream

Server ──event: update──→ Client
Server ──event: update──→ Client
Server ──event: update──→ Client
         (connection stays open)
```

**Spring Boot implementation:**

```java
@RestController
@RequestMapping("/api/v1/stream")
public class LiveUpdateController {

    private final SseEmitterRegistry emitterRegistry;

    @GetMapping(produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public SseEmitter streamUpdates(@RequestParam String userId) {
        // Timeout after 5 minutes — client should reconnect
        SseEmitter emitter = new SseEmitter(300_000L);

        emitterRegistry.register(userId, emitter);

        emitter.onTimeout(() -> emitterRegistry.remove(userId));
        emitter.onCompletion(() -> emitterRegistry.remove(userId));
        emitter.onError(e -> emitterRegistry.remove(userId));

        return emitter;
    }
}

@Component
public class SseEmitterRegistry {

    private final ConcurrentHashMap<String, SseEmitter> emitters = new ConcurrentHashMap<>();

    public void register(String userId, SseEmitter emitter) {
        emitters.put(userId, emitter);
    }

    public void remove(String userId) {
        emitters.remove(userId);
    }

    public void broadcast(String eventName, Object data) {
        emitters.forEach((userId, emitter) -> {
            try {
                emitter.send(SseEmitter.event()
                        .name(eventName)
                        .data(data, MediaType.APPLICATION_JSON)
                        .id(UUID.randomUUID().toString())
                        .reconnectTime(3000));
            } catch (IOException e) {
                emitter.completeWithError(e);
                emitters.remove(userId);
            }
        });
    }
}
```

**Scaling challenge:** SSE connections are stateful — each connection is tied to a specific server instance. With 10 instances behind a load balancer, a client's SSE connection goes to instance 3, but the event that should trigger a push happens on instance 7. Solution: Use Redis Pub/Sub or Kafka to fan out events to all instances.

```
                          ┌─── Instance 1 ──→ Client A
Event Source → Redis ─────┼─── Instance 2 ──→ Client B
              Pub/Sub     └─── Instance 3 ──→ Client C
```

**Failure scenarios:**
- **Proxy timeout:** Many reverse proxies (nginx default: 60s) close idle connections. Fix: send periodic heartbeat comments (`:\n\n`).
- **Memory leak:** Forgetting to remove dead emitters from the registry. Always handle `onTimeout`, `onCompletion`, and `onError`.
- **File descriptor exhaustion:** Each SSE connection = 1 socket = 1 file descriptor. With `ulimit -n 1024`, you max out at ~1000 concurrent connections per process.

---

### 3.5 WebSockets

**Concept:** Full-duplex, bidirectional communication over a single TCP connection. Starts as an HTTP request, then "upgrades" to the WebSocket protocol. Unlike SSE, both client and server can send messages at any time.

```
Client ──HTTP Upgrade──→ Server
         Connection: Upgrade
         Upgrade: websocket

Server ──101 Switching──→ Client

Client ←──────────────→ Server (full-duplex)
```

**Spring Boot implementation:**

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    private final ChatWebSocketHandler chatHandler;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(chatHandler, "/ws/chat")
                .setAllowedOrigins("https://yourdomain.com") // Never use "*" in production
                .addInterceptors(new AuthHandshakeInterceptor());
    }
}

@Component
public class ChatWebSocketHandler extends TextWebSocketHandler {

    private final ConcurrentHashMap<String, WebSocketSession> sessions =
        new ConcurrentHashMap<>();

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String userId = extractUserId(session);
        sessions.put(userId, session);
        log.info("WebSocket connected: userId={}, sessionId={}", userId, session.getId());
    }

    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        String userId = extractUserId(session);
        ChatMessage chatMessage = objectMapper.readValue(message.getPayload(), ChatMessage.class);

        // Process and broadcast to target user
        WebSocketSession targetSession = sessions.get(chatMessage.getTargetUserId());
        if (targetSession != null && targetSession.isOpen()) {
            targetSession.sendMessage(new TextMessage(
                objectMapper.writeValueAsString(chatMessage)
            ));
        }
    }

    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        String userId = extractUserId(session);
        sessions.remove(userId);
        log.info("WebSocket closed: userId={}, status={}", userId, status);
    }

    @Override
    public void handleTransportError(WebSocketSession session, Throwable exception) {
        log.error("WebSocket error: sessionId={}", session.getId(), exception);
        sessions.remove(extractUserId(session));
    }

    private String extractUserId(WebSocketSession session) {
        return (String) session.getAttributes().get("userId");
    }
}
```

**When NOT to use WebSockets:**
- If data flows only server→client: use SSE (simpler, auto-reconnect built into browsers).
- If you need request-response semantics: use HTTP (WebSocket has no built-in request/response correlation).
- If clients are behind corporate proxies: many proxies don't support WebSocket upgrade.
- If your infrastructure is serverless: WebSocket connections are stateful and long-lived — the opposite of serverless design.

**Trade-offs vs SSE:**

| Dimension | WebSocket | SSE |
|-----------|-----------|-----|
| Direction | Bidirectional | Server → Client |
| Protocol | Custom (ws://) | Standard HTTP |
| Auto-reconnect | Manual | Built-in (EventSource API) |
| Binary support | Yes | No (text only) |
| Load balancer | Needs sticky sessions | Standard HTTP routing |
| Scalability | Harder (stateful) | Easier |
| Use case | Chat, gaming, collaboration | Notifications, feeds, dashboards |

---

### 3.6 Publish-Subscribe

**Concept:** Producers publish messages to a topic. Consumers subscribe to topics and receive messages independently. The producer does not know (or care) who consumes the message.

```
                         ┌──→ Consumer A (Email Service)
Producer ──→ Topic ──────┼──→ Consumer B (Analytics Service)
                         └──→ Consumer C (Notification Service)
```

**Spring Boot with Kafka:**

```java
// Producer
@Service
public class OrderEventPublisher {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public CompletableFuture<SendResult<String, OrderEvent>> publishOrderCreated(Order order) {
        OrderEvent event = new OrderEvent(
            UUID.randomUUID().toString(),
            "ORDER_CREATED",
            order.getId(),
            Instant.now()
        );

        // Use order ID as key — ensures ordering per order
        return kafkaTemplate.send("orders", order.getId(), event)
                .whenComplete((result, ex) -> {
                    if (ex != null) {
                        log.error("Failed to publish event: orderId={}", order.getId(), ex);
                        // Store in outbox table for retry
                    } else {
                        log.info("Published event: topic={}, partition={}, offset={}",
                            result.getRecordMetadata().topic(),
                            result.getRecordMetadata().partition(),
                            result.getRecordMetadata().offset());
                    }
                });
    }
}

// Consumer
@Component
public class InventoryEventConsumer {

    @KafkaListener(
        topics = "orders",
        groupId = "inventory-service",
        concurrency = "3"  // 3 consumer threads
    )
    public void handleOrderEvent(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment ack) {

        log.info("Received: event={}, partition={}, offset={}", event.getType(), partition, offset);

        try {
            inventoryService.reserveStock(event.getOrderId());
            ack.acknowledge(); // Manual commit — only after successful processing
        } catch (Exception e) {
            log.error("Failed to process event: orderId={}", event.getOrderId(), e);
            // Don't acknowledge — message will be redelivered
            // But beware: this can cause the consumer to get stuck on a poison pill
        }
    }
}
```

**Scaling implications:**
- Kafka partitions determine max consumer parallelism. 6 partitions = max 6 consumers in a group.
- Message ordering is only guaranteed within a partition. Use the entity ID as the partition key to ensure ordered processing per entity.
- Consumer lag (offset lag) is the key metric. Growing lag = consumers can't keep up = you need more partitions or faster consumers.

**Failure scenarios:**
1. **Poison pill message:** A malformed message causes the consumer to throw an exception repeatedly. The consumer gets stuck retrying the same message. Fix: dead letter queue (DLQ) after N retries.
2. **Consumer rebalancing storm:** A consumer crashes → Kafka rebalances partitions across remaining consumers → rebalance takes time → other consumers stop processing → if it takes too long, more consumers miss heartbeats → cascade.
3. **Duplicate processing:** Consumer processes message, crashes before committing offset. On restart, it re-processes the message. Your consumer MUST be idempotent.

---

### 3.7 Sidecar Pattern

**Concept:** Deploy a companion process (sidecar) alongside your main application to handle cross-cutting concerns: networking, observability, security, configuration. The sidecar intercepts all inbound and outbound traffic.

```
┌─────────────── Pod ────────────────┐
│                                    │
│  ┌──────────┐   ┌──────────────┐  │
│  │  Your     │   │   Sidecar    │  │
│  │  Spring   │←─→│   (Envoy)    │←──── Inbound traffic
│  │  Boot App │   │              │───→  Outbound traffic
│  └──────────┘   └──────────────┘  │
│   localhost:8080  localhost:15001  │
│                                   │
└───────────────────────────────────┘
```

**What the sidecar handles (so your app doesn't have to):**
- mTLS termination and certificate rotation
- Circuit breaking, retries, timeouts
- Load balancing (client-side)
- Observability (distributed tracing, metrics)
- Rate limiting
- Traffic shaping (canary deployments)

**Why this matters for Java engineers:** Without a service mesh, your Spring Boot app needs Resilience4j for circuit breaking, Spring Cloud LoadBalancer for client-side LB, Micrometer for metrics, and Sleuth for tracing. With a sidecar like Envoy (Istio), you get all of this at the infrastructure layer — your app just makes plain HTTP calls to `localhost`.

**Trade-off:** Sidecar adds ~1-3ms latency per hop (due to the extra localhost network hop). For latency-sensitive systems (<10ms p99), this is significant. For typical services (50-200ms), it's negligible.

---

### 3.8 Stateful vs Stateless

**Stateless service:** No request depends on any previous request. Any instance can handle any request. Scale horizontally trivially.

**Stateful service:** The server maintains state (session, WebSocket, in-memory cache) that is specific to a client or entity. Requires sticky sessions, careful failover, and complicates scaling.

```java
// STATELESS — any instance can handle this
@RestController
public class UserController {

    @GetMapping("/api/users/{id}")
    public User getUser(@PathVariable Long id,
                        @RequestHeader("Authorization") String token) {
        // Token contains all auth info — no server-side session
        Claims claims = jwtParser.parseSignedClaims(token).getPayload();
        return userService.findById(id);
    }
}

// STATEFUL — this is tied to a specific instance
@Component
public class WebSocketHandler extends TextWebSocketHandler {
    // In-memory session map — lost on restart, not shared across instances
    private final Map<String, WebSocketSession> sessions = new ConcurrentHashMap<>();
}
```

**Production rule:** Default to stateless. When state is unavoidable (WebSocket, SSE, caching), externalize it (Redis, database) or accept the operational complexity of stateful services (sticky sessions, graceful drain on deploy).

---

### 3.9 Idempotency

**Concept:** An operation is idempotent if performing it multiple times produces the same result as performing it once. This is critical because networks are unreliable — clients WILL retry, and you MUST handle duplicate requests safely.

```
Non-idempotent (dangerous):
  POST /transfer {from: A, to: B, amount: 100}
  Client timeout → retry → $200 transferred instead of $100

Idempotent (safe):
  POST /transfer {from: A, to: B, amount: 100}
  Header: Idempotency-Key: txn-abc-123
  Client timeout → retry with same key → server returns cached result
```

**Production implementation:**

```java
@Service
public class IdempotentOperationService {

    private final IdempotencyKeyRepository keyRepository;
    private final TransactionTemplate txTemplate;

    public <T> T executeIdempotent(String idempotencyKey, Supplier<T> operation) {
        // Check if we've already processed this key
        Optional<IdempotencyRecord> existing = keyRepository.findByKey(idempotencyKey);

        if (existing.isPresent()) {
            IdempotencyRecord record = existing.get();
            if (record.getStatus() == Status.COMPLETED) {
                // Already processed — return cached result
                return deserialize(record.getResponseBody());
            }
            if (record.getStatus() == Status.IN_PROGRESS) {
                // Another request is processing — conflict
                throw new ConflictException("Request is being processed");
            }
        }

        return txTemplate.execute(status -> {
            // Insert with IN_PROGRESS status (unique constraint on key)
            try {
                keyRepository.save(new IdempotencyRecord(idempotencyKey, Status.IN_PROGRESS));
            } catch (DataIntegrityViolationException e) {
                // Race condition — another thread inserted first
                throw new ConflictException("Duplicate request");
            }

            // Execute the actual operation
            T result = operation.get();

            // Mark as completed with cached response
            keyRepository.updateStatus(idempotencyKey, Status.COMPLETED, serialize(result));

            return result;
        });
    }
}
```

**What HTTP methods are naturally idempotent:**

| Method | Idempotent | Safe | Notes |
|--------|-----------|------|-------|
| GET | Yes | Yes | Must not have side effects |
| PUT | Yes | No | Full replacement — same PUT = same result |
| DELETE | Yes | No | Deleting twice = same end state |
| POST | **No** | No | Requires explicit idempotency key |
| PATCH | **Depends** | No | Idempotent if using absolute values, not if using deltas |

---

### 3.10 Multiplexing vs Connection Pooling

**Connection Pooling:** Maintain a pool of reusable TCP connections. Each request borrows a connection, uses it exclusively, then returns it. Avoids the cost of TCP/TLS handshake per request.

```
Thread 1 ──→ Connection A ──→ Server
Thread 2 ──→ Connection B ──→ Server
Thread 3 ──→ (waiting for a connection from pool)

Pool size = 2, Thread 3 must wait.
```

**Multiplexing (HTTP/2):** Multiple requests share a single TCP connection simultaneously via streams. No head-of-line blocking at the HTTP layer (but still at the TCP layer).

```
Thread 1 ──→ Stream 1 ─┐
Thread 2 ──→ Stream 2 ──┼──→ Single TCP Connection ──→ Server
Thread 3 ──→ Stream 3 ─┘

All three requests use one connection, concurrently.
```

**HikariCP (DB connection pool) configuration:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10         # Max connections
      minimum-idle: 5               # Min idle connections
      connection-timeout: 30000     # Max wait for connection (ms)
      idle-timeout: 600000          # Max idle time before eviction
      max-lifetime: 1800000         # Max connection lifetime (30 min)
      leak-detection-threshold: 60000  # Log warning if connection held >60s
```

**Pool sizing formula (from HikariCP wiki):**
```
Pool Size = Tn × (Cm - 1) + 1

Where:
  Tn = max number of threads that can access the database simultaneously
  Cm = max number of simultaneous connections each thread may hold

For a simple web app: connections ≈ (2 × CPU cores) + effective_spindle_count
Example: 4 cores, SSD → pool size ≈ 9-10
```

**Why 10 connections can outperform 100:** A database has limited resources (CPU, I/O, lock manager). 100 connections mean 100 concurrent queries competing for the same B-tree locks, the same buffer pool pages, and the same I/O bandwidth. At high concurrency, most time is spent waiting for locks, not doing useful work. A smaller pool with queuing often has higher throughput because it reduces contention.

**Senior-level interview question:** "Your Spring Boot app has 200 Tomcat threads and 10 HikariCP connections. Under load, p99 latency spikes to 5 seconds. What's happening?"

**Answer:** 200 threads are competing for 10 connections. 190 threads are blocked on `HikariPool.getConnection()`. The thread dump will show most threads in `WAITING` state on the Hikari pool semaphore. Solutions: (1) increase pool size (but watch DB max connections), (2) reduce query execution time, (3) use read replicas to distribute load, (4) implement caching to reduce DB hits.

---

## 4. OS & Kernel Fundamentals (From Java POV)

Understanding what happens below the JVM is what separates senior engineers from mid-level engineers. When your Spring Boot app is "slow," the root cause is often in the OS layer — thread scheduling, TCP buffer pressure, or file descriptor limits.

### 4.1 System Calls: accept(), read(), write()

Every network operation in Java eventually becomes a system call. The JVM does not talk to the network — the kernel does.

```
Your Spring Boot App
       │
   JVM (Java code)
       │
   JNI → Native code
       │
   System call (syscall)
       │
   Kernel (TCP/IP stack)
       │
   NIC (network card)
```

**accept() — accepting incoming connections:**

When Tomcat calls `ServerSocket.accept()`, the JVM issues the `accept()` syscall. The kernel removes a connection from the **accept queue** (also called the complete connection queue) and returns a new file descriptor representing that connection.

```
SYN Queue (half-open):    Client sent SYN, server sent SYN-ACK, waiting for ACK
Accept Queue (complete):  Three-way handshake done, waiting for accept()

If accept queue is full → kernel drops new connections (or sends RST)
Controlled by: net.core.somaxconn (default 128 in Linux, often too low)
```

**read() — reading data from a socket:**

When your `InputStream.read()` is called, the kernel copies data from the socket's **receive buffer** (kernel space) to the application's buffer (user space). If the receive buffer is empty, the thread blocks (in blocking I/O mode).

**write() — writing data to a socket:**

When your `OutputStream.write()` is called, the kernel copies data from user space to the socket's **send buffer**. If the send buffer is full (receiver is slow — backpressure via TCP flow control), the thread blocks.

```
Application (User Space)          Kernel Space
┌─────────────┐                 ┌──────────────────┐
│ byte[] buf  │──write()──→     │ Socket Send Buffer│──→ Network
│             │←──read()──      │ Socket Recv Buffer│←── Network
└─────────────┘                 └──────────────────┘

Buffer sizes:
  net.core.rmem_default = 212992 (208 KB receive buffer)
  net.core.wmem_default = 212992 (208 KB send buffer)
```

### 4.2 Process vs Thread

| Dimension | Process | Thread |
|-----------|---------|--------|
| Memory | Separate address space | Shared address space |
| Creation cost | Heavy (~10ms, fork()) | Light (~1ms, clone()) |
| Communication | IPC (pipes, sockets, shared memory) | Direct (shared heap) |
| Isolation | Strong (crash doesn't affect others) | Weak (one bad thread can crash the JVM) |
| Context switch | Expensive (~5-10μs, TLB flush) | Cheaper (~1-3μs, no TLB flush) |

**JVM context:** A Java application is one process with many threads. The JVM manages:
- Application threads (your code)
- GC threads (parallel, G1, ZGC)
- JIT compiler threads (C1, C2)
- Finalizer thread
- Signal handler thread

**Production insight:** On a 4-core machine running a Spring Boot app with 200 Tomcat threads, only 4 threads can truly run simultaneously. The other 196 are either blocked (waiting for I/O) or in the OS run queue waiting for a CPU time slice. If most threads are CPU-bound (not I/O-bound), you have massive context switching overhead — each switch costs ~1-3μs plus cache invalidation.

### 4.3 Context Switching

```
Thread A running on Core 0:
  1. Timer interrupt fires (time slice expired, ~4ms)
  2. Kernel saves Thread A's registers (PC, SP, general registers)
  3. Kernel loads Thread B's registers
  4. Thread B resumes on Core 0

Cost:
  Direct: ~1-3μs (register save/restore)
  Indirect: ~5-10μs (CPU cache pollution — L1/L2 cache now has Thread A's data,
            Thread B must refill it)
```

**How to measure context switches:**

```bash
# Linux: context switches per second
vmstat 1
# cs column shows context switches

# Per-process: voluntary (I/O wait) vs involuntary (time slice expired)
cat /proc/<pid>/status | grep ctxt
# voluntary_ctxt_switches: Thread chose to wait (good — I/O)
# nonvoluntary_ctxt_switches: OS preempted thread (bad — CPU contention)
```

**Senior insight:** High involuntary context switches on a Java server means your thread pool is too large for the available CPU cores. You're wasting time switching instead of doing work.

### 4.4 Blocking vs Non-blocking I/O

```
Blocking I/O (BIO):
  Thread calls read()
  No data available
  Thread BLOCKS (goes to WAITING state)
  OS parks the thread
  When data arrives, OS wakes the thread
  Thread resumes

  Problem: 1 thread per connection. 10,000 connections = 10,000 threads
           Each thread ≈ 1MB stack = 10GB just for stacks.

Non-blocking I/O (NIO):
  Thread calls read()
  No data available
  read() returns immediately with 0 bytes
  Thread can do other work
  Thread polls or registers for notification

  Advantage: 1 thread can handle many connections.
```

**The evolution:**

```
Level 0: Blocking I/O + Thread-per-connection
  → Simple, correct, doesn't scale past ~1000 connections
  → Old Tomcat BIO connector

Level 1: Non-blocking I/O + select()/poll()
  → Single thread checks all connections
  → O(n) per check — slow with many connections
  → Java: Selector with SelectionKey

Level 2: Non-blocking I/O + epoll (Linux) / kqueue (macOS)
  → Kernel notifies which connections are ready
  → O(1) per event — scales to millions
  → Java NIO with native epoll transport (Netty)

Level 3: io_uring (Linux 5.1+)
  → True async I/O — kernel does the I/O, notifies on completion
  → No syscalls in the hot path — shared ring buffer
  → Not yet mainstream in Java ecosystem
```

### 4.5 Epoll and Java NIO

**Epoll** is the Linux kernel's event notification mechanism. Instead of asking the kernel "which of my 10,000 sockets have data?" repeatedly, you register interest with epoll once, and the kernel tells you when events happen.

```java
// Java NIO — what Netty/Tomcat NIO does under the hood
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.configureBlocking(false);
serverChannel.bind(new InetSocketAddress(8080));
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    int readyCount = selector.select(); // Blocks until events are ready (epoll_wait)

    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    for (SelectionKey key : selectedKeys) {
        if (key.isAcceptable()) {
            // New connection
            SocketChannel client = serverChannel.accept();
            client.configureBlocking(false);
            client.register(selector, SelectionKey.OP_READ);
        } else if (key.isReadable()) {
            // Data available to read
            SocketChannel client = (SocketChannel) key.channel();
            ByteBuffer buffer = ByteBuffer.allocate(1024);
            int bytesRead = client.read(buffer);
            // Process data...
        }
    }
    selectedKeys.clear();
}
```

**Netty event loop — the production version of the above:**

```
┌─────────────────────────────────────────┐
│            Netty Event Loop             │
│                                         │
│  1. selector.select() — wait for I/O    │
│  2. Process all ready I/O events        │
│  3. Run queued tasks (scheduled work)   │
│  4. Go back to step 1                   │
│                                         │
│  CRITICAL RULE: Never block this thread!│
│  No Thread.sleep(), no blocking DB call,│
│  no synchronized blocks.                │
└─────────────────────────────────────────┘
```

**Netty thread model:**

```
                         ┌─── Channel A ──┐
Boss Event Loop ────→    ├─── Channel B ──┤──→ Worker Event Loop 1
(accepts connections)    └─── Channel C ──┘   (handles I/O for A, B, C)

                         ┌─── Channel D ──┐
                         ├─── Channel E ──┤──→ Worker Event Loop 2
                         └─── Channel F ──┘   (handles I/O for D, E, F)

Worker count default = 2 × CPU cores
```

### 4.6 Thread Pools and CPU Competition

**Tomcat thread pool tuning:**

```yaml
server:
  tomcat:
    threads:
      max: 200          # Max worker threads
      min-spare: 10     # Min idle threads kept alive
    max-connections: 8192  # Max simultaneous connections (NIO)
    accept-count: 100    # OS accept queue size when max-connections reached
    connection-timeout: 20000  # TCP connection timeout (ms)
```

**Thread pool sizing — the key formula:**

```
For I/O-bound work (most web apps):
  Optimal threads = N_cpu × (1 + W/C)

  N_cpu = number of CPU cores
  W = wait time (I/O, network, DB)
  C = compute time (CPU processing)

Example: 4 cores, avg request spends 100ms waiting for DB and 10ms computing
  Optimal = 4 × (1 + 100/10) = 4 × 11 = 44 threads

Example: 4 cores, CPU-intensive work (image processing), W≈0
  Optimal = 4 × (1 + 0) = 4 threads (≈ CPU count)
```

**ExecutorService examples:**

```java
@Configuration
public class AsyncConfig {

    // For I/O-bound work (calling external APIs, DB queries)
    @Bean("ioExecutor")
    public ExecutorService ioExecutor() {
        int cpuCores = Runtime.getRuntime().availableProcessors();
        return new ThreadPoolExecutor(
            cpuCores,                              // core pool
            cpuCores * 10,                         // max pool
            60L, TimeUnit.SECONDS,                 // idle thread keepalive
            new LinkedBlockingQueue<>(1000),        // bounded queue
            new ThreadFactoryBuilder()
                .setNameFormat("io-worker-%d")
                .setDaemon(true)
                .build(),
            new ThreadPoolExecutor.CallerRunsPolicy() // backpressure: caller thread does work
        );
    }

    // For CPU-bound work (computation, serialization)
    @Bean("cpuExecutor")
    public ExecutorService cpuExecutor() {
        int cpuCores = Runtime.getRuntime().availableProcessors();
        return Executors.newFixedThreadPool(cpuCores, new ThreadFactoryBuilder()
            .setNameFormat("cpu-worker-%d")
            .build());
    }
}
```

**CompletableFuture — async composition:**

```java
@Service
public class OrderFulfillmentService {

    @Autowired @Qualifier("ioExecutor")
    private ExecutorService ioExecutor;

    public CompletableFuture<FulfillmentResult> fulfillOrder(String orderId) {
        // Run three I/O calls in parallel
        CompletableFuture<InventoryResult> inventoryFuture =
            CompletableFuture.supplyAsync(
                () -> inventoryClient.checkStock(orderId), ioExecutor);

        CompletableFuture<PaymentResult> paymentFuture =
            CompletableFuture.supplyAsync(
                () -> paymentClient.authorize(orderId), ioExecutor);

        CompletableFuture<ShippingEstimate> shippingFuture =
            CompletableFuture.supplyAsync(
                () -> shippingClient.estimate(orderId), ioExecutor);

        // Combine results when all complete
        return CompletableFuture.allOf(inventoryFuture, paymentFuture, shippingFuture)
            .thenApply(v -> new FulfillmentResult(
                inventoryFuture.join(),
                paymentFuture.join(),
                shippingFuture.join()
            ))
            .orTimeout(5, TimeUnit.SECONDS)  // Fail if any call exceeds 5s
            .exceptionally(ex -> {
                log.error("Fulfillment failed: orderId={}", orderId, ex);
                return FulfillmentResult.failed(ex.getMessage());
            });
    }
}
```

### 4.7 JVM Thread ↔ OS Thread Mapping

**Before Virtual Threads (Java < 21):**

```
Java Thread 1:1 OS Thread (platform thread)

Each Thread.start() → pthread_create() → kernel thread

Consequence:
  - 1 Java thread = 1 OS thread = ~1MB stack (default)
  - 10,000 threads = 10GB just for stacks
  - OS scheduler manages all threads
  - Context switch cost is real (~1-3μs + cache effects)
```

**With Virtual Threads (Java 21+, Project Loom):**

```
Virtual Thread M:N OS Thread (carrier thread)

Many virtual threads share a small number of carrier threads (platform threads)
When a virtual thread blocks (I/O, sleep), it's unmounted from the carrier
The carrier runs another virtual thread immediately

┌──────────────────────────────────────┐
│   Virtual Thread Pool (millions)     │
│  VT1  VT2  VT3  VT4  VT5  ...  VTn │
└────────────┬─────────────────────────┘
             │ (mount/unmount)
┌────────────┴─────────────────────────┐
│   Carrier Threads (= CPU cores)      │
│  CT1    CT2    CT3    CT4            │
└──────────────────────────────────────┘

VT1 runs on CT1 → VT1 blocks on DB call → VT1 unmounted
VT2 mounts on CT1 → VT2 runs → VT2 blocks → VT2 unmounted
VT3 mounts on CT1 → ...
```

**Spring Boot with Virtual Threads:**

```yaml
# application.yml — Spring Boot 3.2+
spring:
  threads:
    virtual:
      enabled: true  # Tomcat uses virtual threads instead of platform threads
```

```java
// Or programmatic configuration
@Bean
public TomcatProtocolHandlerCustomizer<?> protocolHandlerVirtualThreadExecutorCustomizer() {
    return protocolHandler -> {
        protocolHandler.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
    };
}
```

**What Virtual Threads solve:** The thread-per-request model is conceptually simple and easy to debug (stack traces make sense). The only problem was that OS threads are expensive. Virtual threads remove that constraint — you can have millions of virtual threads, each representing one request, with the simplicity of synchronous code.

**What Virtual Threads do NOT solve:**
- CPU-bound work (you still only have N cores)
- `synchronized` blocks (virtual thread gets pinned to the carrier — use `ReentrantLock` instead)
- Native method calls (also cause pinning)
- ThreadLocal abuse (virtual threads are cheap to create, but ThreadLocal can leak memory if you create millions)

### 4.8 The C10K Problem

**Problem:** How do you handle 10,000 concurrent connections on a single server?

**Why it was hard (circa 2000):**
- Thread-per-connection model: 10,000 threads × 1MB stack = 10GB RAM
- `select()`/`poll()` system calls: O(n) scanning — checking 10,000 file descriptors every time

**How it was solved:**
1. **epoll/kqueue:** O(1) event notification — kernel tells you which FDs are ready
2. **Non-blocking I/O:** Threads handle multiple connections
3. **Event-driven architecture:** Netty, Nginx, Node.js

**Modern C10M challenge:** 10 million concurrent connections requires:
- Kernel bypass (DPDK, io_uring)
- Zero-copy networking
- Lock-free data structures
- Memory-mapped I/O

### 4.9 TCP Buffers, SYN Backlog, TIME_WAIT, Ephemeral Port Exhaustion

**SYN backlog — the two queues:**

```
Client ──SYN──→ Server

Server has TWO queues:
  1. SYN Queue (half-open): SYN received, SYN-ACK sent, waiting for ACK
     Size: net.ipv4.tcp_max_syn_backlog (default 128-1024)

  2. Accept Queue (complete): Handshake done, waiting for accept()
     Size: min(net.core.somaxconn, backlog param in listen())
     Default somaxconn: 128

If Accept Queue is full:
  → Kernel drops SYN (client retries with exponential backoff)
  → Or sends RST (depends on tcp_abort_on_overflow)

Production tuning:
  sysctl -w net.core.somaxconn=65535
  sysctl -w net.ipv4.tcp_max_syn_backlog=65535
```

**TIME_WAIT — the lingering connection state:**

```
When a TCP connection closes, the initiator enters TIME_WAIT for 2×MSL (60 seconds on Linux).

Why: To handle delayed packets from the old connection that might confuse a new connection
     using the same port.

Problem: A server making many outbound connections (e.g., calling microservices) accumulates
         TIME_WAIT sockets. Each holds a (src_ip, src_port, dst_ip, dst_port) tuple.

Check: ss -s | grep TIME-WAIT
       If you see >10,000 TIME_WAIT sockets, you have a problem.

Mitigation:
  sysctl -w net.ipv4.tcp_tw_reuse=1       # Reuse TIME_WAIT sockets for new connections
  sysctl -w net.ipv4.tcp_fin_timeout=15   # Reduce FIN_WAIT_2 timeout
  Use connection pooling (don't create new connections per request!)
```

**Ephemeral port exhaustion:**

```
Outbound connections use ephemeral ports (default range: 32768-60999).
That's ~28,000 ports per destination IP.

If your service makes many short-lived connections to one destination
(e.g., calling a single microservice without connection pooling):
  28,000 connections / 60s TIME_WAIT ≈ 467 new connections/second MAX

Exceed this? You get: "Cannot assign requested address" (EADDRNOTAVAIL)

Fix: Connection pooling (reuse connections instead of creating new ones).
```

**WebFlux example for contrast with blocking model:**

```java
@RestController
@RequestMapping("/api/v1/reactive")
public class ReactiveOrderController {

    private final WebClient webClient;
    private final ReactiveOrderRepository orderRepository; // R2DBC

    @GetMapping("/orders/{id}/enriched")
    public Mono<EnrichedOrder> getEnrichedOrder(@PathVariable String id) {
        // Non-blocking: no thread is blocked at any point
        return orderRepository.findById(id)                       // R2DBC — non-blocking DB
            .flatMap(order ->
                Mono.zip(
                    webClient.get()                                // Non-blocking HTTP
                        .uri("/api/users/{id}", order.getUserId())
                        .retrieve()
                        .bodyToMono(User.class),
                    webClient.get()                                // Parallel non-blocking HTTP
                        .uri("/api/inventory/{sku}", order.getSku())
                        .retrieve()
                        .bodyToMono(InventoryStatus.class)
                )
                .map(tuple -> new EnrichedOrder(order, tuple.getT1(), tuple.getT2()))
            )
            .timeout(Duration.ofSeconds(3))                       // Global timeout
            .onErrorReturn(EnrichedOrder.fallback(id));            // Graceful degradation
    }
}
```

---

## 5. Networking & Protocols (Java Ecosystem View)

### 5.1 TCP Three-Way Handshake

```
Client                          Server
  │                               │
  │────── SYN (seq=x) ──────────→│  Client picks initial sequence number
  │                               │  Server receives, allocates resources
  │←───── SYN-ACK (seq=y,ack=x+1)│  Server picks its sequence number
  │                               │
  │────── ACK (ack=y+1) ────────→│  Connection established
  │                               │

Time cost: 1 RTT (Round Trip Time)
  - Same datacenter: ~0.5ms
  - Same region: ~1-5ms
  - Cross-region: ~50-150ms
  - Cross-continent: ~100-300ms

EVERY new TCP connection pays this cost.
That's why connection pooling exists.
```

**What the JVM handles vs what the Kernel handles:**

```
┌──────────────── JVM ──────────────────┐
│  Socket API (java.net.Socket)         │
│  Channel API (java.nio.channels)      │
│  SSLContext / SSLEngine               │
│  HttpClient (Java 11+)               │
│  Application protocol (HTTP, gRPC)    │
└───────────────────┬───────────────────┘
                    │ System calls
┌───────────────────┴───────────────────┐
│              Kernel                    │
│  TCP/IP stack (segmentation,          │
│    sequencing, flow control,          │
│    congestion control, retransmit)    │
│  Socket buffers (send/receive)        │
│  Connection state machine             │
│  epoll / kqueue notification          │
│  IP routing                           │
│  ARP resolution                       │
└───────────────────┬───────────────────┘
                    │
┌───────────────────┴───────────────────┐
│          Network Interface Card (NIC) │
│  Ethernet framing, checksums          │
│  DMA to/from kernel buffers           │
└───────────────────────────────────────┘
```

### 5.2 TCP Flow Control vs Congestion Control

**Flow control** — prevents the sender from overwhelming the receiver:

```
Receiver says: "My receive window is 64KB"
Sender: "OK, I'll send at most 64KB before waiting for ACK"

If receiver is slow (app not calling read() fast enough):
  → Receive buffer fills up
  → Receiver advertises smaller window
  → Sender slows down
  → Eventually: zero window → sender stops completely

This is TCP-level backpressure.
```

**Congestion control** — prevents the sender from overwhelming the network:

```
Algorithms: Slow Start → Congestion Avoidance → Fast Recovery

Slow Start:
  Start with cwnd = 1 MSS (1460 bytes typically)
  Double cwnd every RTT until packet loss or ssthresh

  Time to ramp up on a new connection (100ms RTT, 10MB/s target):
    RTT 0: cwnd=1 (1.4KB)
    RTT 1: cwnd=2 (2.9KB)
    RTT 2: cwnd=4 (5.8KB)
    ...
    RTT 13: cwnd=8192 (~12MB) — finally at target

  That's 13 RTTs × 100ms = 1.3 seconds to reach full throughput!

  This is why long-lived connections outperform short-lived ones.
  This is another reason connection pooling matters.
```

**Production impact — the First Request Problem:**

When a Spring Boot service starts and makes its first HTTP call to another service, the TCP connection goes through slow start. The first request on a new connection is often slower than subsequent ones. Connection pool warming on startup mitigates this.

### 5.3 TLS 1.2 vs TLS 1.3

```
TLS 1.2 Handshake (2 RTT):
  Client ──ClientHello──→ Server
  Client ←──ServerHello, Cert, KeyExchange──
  Client ──ClientKeyExchange, Finished──→
  Client ←──Finished──
  [Application Data]

TLS 1.3 Handshake (1 RTT):
  Client ──ClientHello + KeyShare──→ Server
  Client ←──ServerHello + KeyShare + Cert + Finished──
  Client ──Finished──→
  [Application Data]
```

| Feature | TLS 1.2 | TLS 1.3 |
|---------|---------|---------|
| Handshake RTTs | 2 | 1 |
| 0-RTT resumption | No | Yes (with risk) |
| Forward secrecy | Optional | Mandatory |
| Cipher suites | Many (some weak) | 5 strong suites only |
| RSA key exchange | Supported | Removed |
| Latency (100ms RTT) | +200ms | +100ms |

**0-RTT risks:** TLS 1.3 allows 0-RTT resumption — the client sends application data in the first message. However, 0-RTT data is **replayable**. An attacker can capture and resend the 0-RTT data. Never use 0-RTT for non-idempotent operations (POST requests that create resources).

**SSLContext configuration in Spring Boot:**

```yaml
server:
  ssl:
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    enabled-protocols: TLSv1.3           # TLS 1.3 only
    ciphers:
      - TLS_AES_256_GCM_SHA384
      - TLS_AES_128_GCM_SHA256
```

```java
// Programmatic SSLContext for outbound calls
@Bean
public WebClient secureWebClient() throws Exception {
    SslContext sslContext = SslContextBuilder.forClient()
        .trustManager(InsecureTrustManagerFactory.INSTANCE) // NEVER in production
        .protocols("TLSv1.3")
        .build();

    HttpClient httpClient = HttpClient.create()
        .secure(spec -> spec.sslContext(sslContext));

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .build();
}
```

### 5.4 QUIC Protocol

```
QUIC = UDP + TLS 1.3 + Streams (HTTP/3 transport)

TCP problems QUIC solves:
  1. Head-of-line blocking: TCP delivers bytes in order.
     One lost packet blocks ALL streams.
     QUIC: Each stream is independent. Loss in stream A doesn't block stream B.

  2. Connection setup latency: TCP (1 RTT) + TLS (1-2 RTT) = 2-3 RTT
     QUIC: 1 RTT (combined transport + crypto handshake)
     QUIC resumption: 0 RTT

  3. Connection migration: TCP connection = (src_ip, src_port, dst_ip, dst_port)
     Phone switches from WiFi to cellular → new IP → TCP connection dies.
     QUIC uses connection IDs → survives IP changes.
```

**Java ecosystem support:** As of Java 21, QUIC is not natively supported in the JDK. Netty has experimental QUIC support via `netty-incubator-codec-quic`. For most Spring Boot services (server-to-server communication within a datacenter), QUIC adds complexity without significant benefit — TCP works fine at sub-1ms RTT. QUIC matters most for mobile clients and cross-region communication.

### 5.5 Latency Cost Per Layer

```
Operation                        Typical Latency
─────────────────────────────────────────────────
L1 cache reference                     0.5 ns
L2 cache reference                       7 ns
Main memory reference                  100 ns
SSD random read                     16,000 ns  (16 μs)
HDD random read                  4,000,000 ns  (4 ms)
Send 1KB over 1 Gbps network        10,000 ns  (10 μs)
TCP handshake (same DC)             500,000 ns  (0.5 ms)
TLS handshake (TLS 1.3, same DC)   500,000 ns  (0.5 ms)
TCP handshake (cross-region)     50,000,000 ns  (50 ms)
TLS handshake (cross-region)    100,000,000 ns  (100 ms)
DNS lookup (uncached)            10,000,000 ns  (10 ms)

Total cost of a NEW HTTPS connection (cross-region):
  DNS + TCP + TLS 1.3 = 10 + 50 + 100 = 160ms
  With connection reuse: 0ms (already established)
```

**Key takeaway:** Connection reuse (via pooling) is one of the single most impactful performance optimizations in backend systems. A reused connection saves 100-300ms per request for cross-region calls.

### 5.6 How HTTPS Works in Spring Boot — Full Request Lifecycle

```
1. Client sends HTTPS request to https://api.example.com/orders

2. DNS resolution: api.example.com → 10.0.1.50 (~1ms cached, ~10ms uncached)

3. TCP handshake: SYN → SYN-ACK → ACK (1 RTT)

4. TLS handshake:
   a. ClientHello (supported cipher suites, TLS version)
   b. ServerHello + Certificate + KeyShare
   c. Client verifies certificate chain (trust store)
   d. Key derivation → symmetric encryption keys established
   e. Finished messages exchanged

5. HTTP request (encrypted):
   POST /orders HTTP/1.1
   Content-Type: application/json
   Authorization: Bearer <jwt>

   {"item": "widget", "quantity": 5}

6. Tomcat NIO:
   a. Acceptor thread: Selector detects new connection (epoll_wait)
   b. NIO thread: Reads bytes from socket, SSL unwrap
   c. Worker thread (from pool): Handles request

7. Spring DispatcherServlet:
   a. HandlerMapping → finds @PostMapping("/orders") method
   b. Filter chain: SecurityFilter → validate JWT
   c. HandlerAdapter → invokes controller method
   d. @RequestBody → Jackson deserializes JSON
   e. Service layer → business logic
   f. Repository → JPA/JDBC → HikariCP → database
   g. Response → Jackson serializes → OutputStream → SSL wrap → socket

8. Response bytes travel back through TLS → TCP → network → client
```

---

## 6. HTTP/1.1 vs HTTP/2 vs HTTP/3 (Java View)

### 6.1 Comparison Overview

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Transport | TCP | TCP | QUIC (UDP) |
| Multiplexing | No (1 req/conn) | Yes (streams) | Yes (streams) |
| Head-of-line blocking | Yes (HTTP + TCP) | Partial (TCP only) | No |
| Header compression | No | HPACK | QPACK |
| Server push | No | Yes | Yes (rarely used) |
| Connection setup | TCP + TLS (2-3 RTT) | TCP + TLS + ALPN (2-3 RTT) | 1 RTT (0 RTT resume) |
| Binary protocol | No (text) | Yes | Yes |
| Request prioritization | No | Yes (stream weights) | Yes (priority frames) |
| Spring Boot support | Full | Full (Tomcat 9+/Jetty) | Experimental |
| Typical use | Legacy, simple APIs | Modern APIs, microservices | Mobile, edge |

### 6.2 HTTP/1.1 — The Workhorse

```
Connection 1: GET /api/users    → Response → GET /api/orders → Response
Connection 2: GET /api/products → Response → GET /api/cart   → Response
Connection 3: GET /api/reviews  → Response

Problem: Each connection handles one request at a time (sequential).
Browsers open 6-8 parallel connections per domain to compensate.
Servers see 6-8× the connection count.
```

**Head-of-line (HOL) blocking:** If request A takes 5 seconds, request B queued behind it on the same connection waits 5 seconds before it even starts processing. Pipelining was an HTTP/1.1 feature to address this, but it was never widely adopted because responses still had to be returned in order.

**Keep-Alive:** HTTP/1.1 defaults to persistent connections (`Connection: keep-alive`). The TCP connection stays open for reuse. Without it, every request would pay the full TCP+TLS handshake cost.

### 6.3 HTTP/2 — Multiplexing and Binary Framing

```
Single TCP Connection:
  ┌─────────────────────────────┐
  │  Stream 1: GET /api/users   │──→ HEADERS frame, DATA frame
  │  Stream 3: GET /api/orders  │──→ HEADERS frame, DATA frame
  │  Stream 5: POST /api/cart   │──→ HEADERS frame, DATA frame
  │                             │
  │  All interleaved on ONE     │
  │  TCP connection             │
  └─────────────────────────────┘
```

**Binary framing layer:**

```
HTTP/1.1 (text):
  GET /api/users HTTP/1.1\r\n
  Host: example.com\r\n
  Accept: application/json\r\n
  \r\n

HTTP/2 (binary frames):
  ┌──────────┬──────┬───────┬────────────────┐
  │ Length(3) │Type(1)│Flags(1)│ Stream ID (4)  │
  ├──────────┴──────┴───────┴────────────────┤
  │            Frame Payload                  │
  └───────────────────────────────────────────┘

  Frame types: HEADERS, DATA, SETTINGS, PUSH_PROMISE,
               RST_STREAM, PING, GOAWAY, WINDOW_UPDATE
```

**HPACK header compression:** HTTP headers are repetitive across requests (same Host, Authorization, Accept on every call). HPACK compresses headers using:
1. Static table: 61 common header name-value pairs
2. Dynamic table: Previously seen headers (per connection)
3. Huffman encoding: For values not in tables

Typical compression: 85-90% reduction in header size.

**Remaining HOL problem:** HTTP/2 solves HOL at the HTTP layer (multiple streams), but TCP still delivers bytes in order. If a TCP packet is lost, ALL streams on that connection are blocked until retransmission. This is why HTTP/3 uses QUIC (UDP) — each stream is independent at the transport layer.

**Spring Boot HTTP/2 configuration:**

```yaml
server:
  http2:
    enabled: true    # Enable HTTP/2
  ssl:
    enabled: true    # HTTP/2 requires TLS in practice (h2)
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_PASSWORD}
```

```java
// HTTP/2 client with Java HttpClient
@Bean
public java.net.http.HttpClient http2Client() {
    return java.net.http.HttpClient.newBuilder()
        .version(java.net.http.HttpClient.Version.HTTP_2)
        .connectTimeout(Duration.ofSeconds(5))
        .build();
}
```

### 6.4 HTTP/3 — QUIC-Based Transport

```
HTTP/3 Stack:
  ┌──────────────┐
  │   HTTP/3     │  (Semantics: methods, headers, status codes — same)
  ├──────────────┤
  │   QUIC       │  (Streams, reliability, flow/congestion control, encryption)
  ├──────────────┤
  │   UDP        │  (Just the transport — no connection state in kernel)
  └──────────────┘

vs HTTP/2 Stack:
  ┌──────────────┐
  │   HTTP/2     │
  ├──────────────┤
  │   TLS 1.2/3  │
  ├──────────────┤
  │   TCP        │
  └──────────────┘
```

**When NOT to use HTTP/2 or HTTP/3:**
- Internal microservice-to-microservice on the same rack (<0.1ms RTT): HTTP/1.1 with connection pooling is simpler and sufficient. The overhead of stream management isn't worth it.
- gRPC already handles multiplexing over HTTP/2 for you — don't add another layer.
- If your reverse proxy doesn't support it, you gain nothing (nginx supports HTTP/2 backend since 1.25.1).

### 6.5 Spring Boot Protocol Examples

**REST Controller (works with HTTP/1.1 and HTTP/2 transparently):**

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    private final ProductService productService;

    @GetMapping("/{id}")
    public ResponseEntity<Product> getProduct(@PathVariable Long id) {
        return productService.findById(id)
            .map(product -> ResponseEntity.ok()
                .cacheControl(CacheControl.maxAge(60, TimeUnit.SECONDS))
                .eTag(product.getVersion().toString())
                .body(product))
            .orElse(ResponseEntity.notFound().build());
    }

    @GetMapping
    public ResponseEntity<Page<Product>> listProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {

        Page<Product> products = productService.findAll(PageRequest.of(page, size));

        return ResponseEntity.ok()
            .header("X-Total-Count", String.valueOf(products.getTotalElements()))
            .body(products);
    }
}
```

**WebFlux reactive controller:**

```java
@RestController
@RequestMapping("/api/v1/reactive/products")
public class ReactiveProductController {

    private final ReactiveProductRepository repository;

    @GetMapping(produces = MediaType.APPLICATION_NDJSON_VALUE)
    public Flux<Product> streamProducts() {
        // Streams products as newline-delimited JSON
        // Client receives products as they're fetched, not all at once
        return repository.findAll()
            .delayElements(Duration.ofMillis(100)); // Simulated backpressure
    }

    @GetMapping("/{id}")
    public Mono<ResponseEntity<Product>> getProduct(@PathVariable String id) {
        return repository.findById(id)
            .map(ResponseEntity::ok)
            .defaultIfEmpty(ResponseEntity.notFound().build());
    }
}
```

**SSE Emitter for real-time streaming:**

```java
@RestController
@RequestMapping("/api/v1/events")
public class EventStreamController {

    private final ApplicationEventPublisher eventPublisher;

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<ServerSentEvent<String>> streamEvents() {
        return Flux.interval(Duration.ofSeconds(1))
            .map(seq -> ServerSentEvent.<String>builder()
                .id(String.valueOf(seq))
                .event("heartbeat")
                .data("{\"timestamp\":\"" + Instant.now() + "\"}")
                .retry(Duration.ofSeconds(5))
                .build());
    }
}
```

**gRPC service definition and implementation:**

```protobuf
// order_service.proto
syntax = "proto3";
package com.example.order;

service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
  rpc StreamOrderUpdates (OrderQuery) returns (stream OrderUpdate); // Server streaming
}

message CreateOrderRequest {
  string user_id = 1;
  repeated OrderItem items = 2;
}
```

```java
@GrpcService
public class OrderGrpcService extends OrderServiceGrpc.OrderServiceImplBase {

    private final OrderService orderService;

    @Override
    public void createOrder(CreateOrderRequest request,
                           StreamObserver<OrderResponse> responseObserver) {
        try {
            Order order = orderService.create(fromProto(request));
            responseObserver.onNext(toProto(order));
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL.withDescription(e.getMessage()).asRuntimeException()
            );
        }
    }

    @Override
    public void streamOrderUpdates(OrderQuery request,
                                   StreamObserver<OrderUpdate> responseObserver) {
        // Server-streaming RPC — push updates as they happen
        orderService.watchUpdates(request.getOrderId(), update -> {
            responseObserver.onNext(toProtoUpdate(update));
        }, () -> {
            responseObserver.onCompleted();
        });
    }
}
```

**WebSocket configuration:**

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketStompConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic", "/queue"); // In-memory broker
        // For production: config.enableStompBrokerRelay("/topic", "/queue")
        //   .setRelayHost("rabbitmq-host")
        //   .setRelayPort(61613);
        config.setApplicationDestinationPrefixes("/app");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws")
                .setAllowedOrigins("https://yourdomain.com")
                .withSockJS(); // Fallback for older browsers
    }
}

@Controller
public class StompMessageController {

    @MessageMapping("/chat.send")
    @SendTo("/topic/chat")
    public ChatMessage sendMessage(@Payload ChatMessage message,
                                   SimpMessageHeaderAccessor headerAccessor) {
        message.setTimestamp(Instant.now());
        return message; // Broadcast to all subscribers of /topic/chat
    }
}
```

### 6.6 Reverse Proxy Implications

```
Client ──HTTP/2──→ Nginx ──HTTP/1.1──→ Spring Boot (common setup)
                   │
                   ├─ Terminates TLS
                   ├─ Multiplexes client HTTP/2 into backend HTTP/1.1
                   ├─ Connection pooling to backend
                   └─ Load balancing

Client ──HTTP/2──→ Nginx ──HTTP/2──→ Spring Boot (requires nginx 1.25.1+)
                   │
                   └─ End-to-end HTTP/2 — better for gRPC passthrough

Client ──HTTP/3──→ CDN/Edge ──HTTP/2──→ Origin (typical for public APIs)
```

**Senior insight:** In most microservice architectures, HTTP/2 benefits are mainly between the edge (load balancer/API gateway) and the client. Internal service-to-service communication uses gRPC (which runs on HTTP/2) or HTTP/1.1 with connection pooling. The protocol choice depends on where the latency budget is spent.

---

## 7. Backend Execution & Concurrency Models

### 7.1 Thread-Per-Request Model

The dominant model in the Java ecosystem (Spring MVC + Tomcat).

```
Request arrives → Tomcat assigns worker thread → Thread handles ENTIRE request
                                                 (including I/O waits)
                                                 → Response sent → Thread returned to pool

┌─────── Tomcat Thread Pool (200 threads) ───────┐
│ [Thread-1: processing] [Thread-2: waiting DB]  │
│ [Thread-3: waiting HTTP] [Thread-4: idle]      │
│ [Thread-5: processing] [Thread-6: waiting DB]  │
│ ...                                            │
│ [Thread-200: idle]                             │
└────────────────────────────────────────────────┘

Throughput = PoolSize / AvgResponseTime
  200 threads / 0.1s = 2,000 RPS (fast responses)
  200 threads / 1.0s = 200 RPS (slow responses)
  200 threads / 5.0s = 40 RPS (very slow — you're in trouble)
```

**Advantages:**
- Simple mental model — one thread, one request, one stack trace
- Easy debugging — `jstack` shows what each request is doing
- Works well when I/O wait is moderate (10-100ms)
- Natural integration with blocking JDBC, JPA, Spring Security

**Disadvantages:**
- Each thread costs ~1MB stack + OS scheduling overhead
- Thread count limits concurrency
- Slow downstream calls hold threads hostage
- Under high concurrency with slow calls: thread starvation

### 7.2 Thread Pool Sizing Deep Dive

```java
@Configuration
public class ThreadPoolConfiguration {

    /**
     * Sizing formula: N_cpu × (1 + W/C)
     * Where W = wait time, C = compute time
     *
     * Measurement approach:
     * 1. Profile typical request with async-profiler or JFR
     * 2. Measure wall time (total) and CPU time
     * 3. W = wall_time - cpu_time, C = cpu_time
     */

    @Bean
    public ThreadPoolTaskExecutor applicationTaskExecutor() {
        int cpuCores = Runtime.getRuntime().availableProcessors();

        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(cpuCores * 2);         // For mixed I/O + CPU
        executor.setMaxPoolSize(cpuCores * 10);          // Burst capacity
        executor.setQueueCapacity(500);                  // Bounded queue
        executor.setKeepAliveSeconds(60);
        executor.setThreadNamePrefix("app-worker-");

        // CRITICAL: rejection policy when pool + queue are full
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // CallerRunsPolicy: the calling thread (Tomcat thread) does the work
        // This provides natural backpressure — Tomcat thread is busy, can't accept new requests

        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);

        return executor;
    }
}
```

**Common thread pool sizing mistakes:**

| Mistake | Why It's Wrong | What Happens |
|---------|---------------|--------------|
| `max-pool-size: 1000` | Way too many threads | Massive context switching, GC pressure, CPU thrashing |
| `max-pool-size: 2` | Way too few | Requests queue up, throughput collapses |
| Unbounded queue `(Integer.MAX_VALUE)` | Memory leak under load | OOM when requests pile up faster than processing |
| Same pool for CPU and I/O work | CPU work starves when I/O threads block | Unpredictable latency |

### 7.3 Blocking JDBC vs R2DBC

```
Blocking JDBC (thread-per-request):
  Thread blocks on socket.read() while waiting for DB response
  Thread is parked by OS, consumes no CPU, but holds 1MB stack

  Tomcat Thread → HikariCP → JDBC Driver → socket.read() → BLOCKED

R2DBC (reactive, non-blocking):
  Event loop thread registers interest in socket read
  When data arrives, callback fires on event loop thread
  No thread is blocked

  EventLoop → R2DBC Driver → Netty Channel → epoll → callback
```

```java
// Blocking JDBC (traditional)
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    @Query("SELECT o FROM Order o WHERE o.userId = :userId AND o.status = :status")
    List<Order> findByUserIdAndStatus(@Param("userId") Long userId,
                                     @Param("status") OrderStatus status);
}

// R2DBC (reactive, non-blocking)
@Repository
public interface ReactiveOrderRepository extends ReactiveCrudRepository<Order, Long> {
    Flux<Order> findByUserIdAndStatus(Long userId, OrderStatus status);
}
```

**When R2DBC helps vs when it doesn't:**

| Scenario | JDBC | R2DBC |
|----------|------|-------|
| Simple CRUD, low concurrency | Better (simpler) | Overkill |
| High concurrency, many I/O-bound calls | Thread starvation risk | Better (fewer threads needed) |
| Complex transactions, JPA features | Full support | Limited (no lazy loading, no JPA) |
| Team familiarity | High | Low (steep learning curve) |
| Debugging | Easy (stack traces) | Hard (reactive chains) |
| Throughput at 10K concurrent | Pool exhaustion | Handles well |

**Senior-level insight:** Virtual Threads (Java 21) largely eliminate the need for R2DBC. You get the simplicity of blocking JDBC with the scalability of non-blocking I/O. The virtual thread blocks, but the carrier thread is released. Unless you specifically need reactive backpressure semantics (Flux/Mono operators), prefer Virtual Threads + JDBC over R2DBC for new projects.

### 7.4 Netty Event Loop Model

```
┌─────────────────────────────────────────────────────┐
│                 Netty Event Loop                     │
│                                                     │
│  while (!shutdown) {                                │
│    1. select(timeout)     // epoll_wait — get I/O   │
│    2. processSelectedKeys() // handle I/O events    │
│    3. runAllTasks()       // run queued tasks        │
│  }                                                  │
│                                                     │
│  GOLDEN RULE: Never block this thread.              │
│  If you block, ALL connections on this loop stall.  │
└─────────────────────────────────────────────────────┘

Thread count: 2 × CPU cores (default)
Each event loop handles thousands of connections.

4-core machine → 8 event loops → can handle 100K+ connections
```

**What counts as "blocking" in an event loop thread:**
- `Thread.sleep()`
- Blocking JDBC calls
- `synchronized` blocks under contention
- `Future.get()` without timeout
- File I/O (disk reads)
- DNS resolution (if not async)

**If you must do blocking work from a Netty-based app (WebFlux):**

```java
@Service
public class HybridService {

    // Offload blocking work to a bounded elastic scheduler
    public Mono<Result> processWithLegacyLibrary(Request request) {
        return Mono.fromCallable(() -> {
                // This blocking call runs on a separate thread pool, NOT the event loop
                return legacyBlockingLibrary.process(request);
            })
            .subscribeOn(Schedulers.boundedElastic()) // Dedicated pool for blocking
            .timeout(Duration.ofSeconds(5));
    }
}
```

### 7.5 ForkJoinPool

The work-stealing thread pool used by `CompletableFuture`, parallel streams, and Virtual Threads.

```
Traditional ThreadPool:
  Thread 1: [Task A] [Task B] [Task C]  ← heavy queue
  Thread 2: [idle]                       ← wasted

ForkJoinPool (work-stealing):
  Thread 1: [Task A] [Task B]
  Thread 2: [Task C] ← stole Task C from Thread 1's queue

  Each thread has a double-ended queue (deque).
  Idle threads steal from the tail of busy threads' queues.
```

```java
// Common ForkJoinPool — shared across ALL CompletableFuture.supplyAsync() calls
// and ALL parallel streams in the JVM
ForkJoinPool.commonPool(); // Size = CPU cores - 1

// DANGER: If one module uses parallel streams for CPU work and another
// uses CompletableFuture for I/O calls, they compete for the same pool.
// The I/O calls can block common pool threads, starving CPU work.

// Solution: Use dedicated pools
ForkJoinPool customPool = new ForkJoinPool(16);
customPool.submit(() -> {
    myList.parallelStream()
        .map(this::processItem)
        .collect(Collectors.toList());
}).get();
```

### 7.6 Backpressure Strategies

```
Fast Producer (10,000 events/sec) → Slow Consumer (1,000 events/sec)

Without backpressure:
  Buffer grows: 9,000 events/sec accumulate
  After 10 seconds: 90,000 events buffered
  After 1 minute: OOM → crash

With backpressure:
  Strategy 1 — DROP: Discard excess events (acceptable for metrics)
  Strategy 2 — LATEST: Keep only most recent N (dashboards)
  Strategy 3 — BUFFER_BOUNDED: Fixed buffer, reject when full (HTTP 429)
  Strategy 4 — ERROR: Signal error upstream (reactive streams)
  Strategy 5 — SLOW_PRODUCER: Flow control (TCP window, reactive request(n))
```

**Reactive backpressure in WebFlux:**

```java
@Service
public class StreamingService {

    public Flux<DataChunk> processLargeDataset() {
        return Flux.create(sink -> {
            dataSource.forEachChunk(chunk -> {
                // Respect backpressure: only emit when downstream can handle it
                if (!sink.isCancelled()) {
                    sink.next(chunk);
                }
            });
            sink.complete();
        }, FluxSink.OverflowStrategy.ERROR)  // Error if consumer can't keep up
        .onBackpressureBuffer(1000,          // Buffer up to 1000 items
            dropped -> log.warn("Dropped chunk: {}", dropped.getId()),
            BufferOverflowStrategy.DROP_OLDEST
        );
    }
}
```

### 7.7 L4 vs L7 Load Balancers

```
L4 (Transport Layer — TCP):
  ┌────────┐    ┌──────────────┐    ┌──────────┐
  │ Client ├───→│  L4 LB       ├───→│ Server A │
  │        │    │ (TCP forward) │───→│ Server B │
  └────────┘    └──────────────┘    └──────────┘

  - Sees: IP, port, TCP flags
  - Cannot see: URL, headers, cookies, body
  - Decision: Hash(src_ip, src_port) → backend
  - Speed: Very fast (~microseconds)
  - Use case: Database load balancing, raw TCP services

L7 (Application Layer — HTTP):
  ┌────────┐    ┌──────────────┐    ┌──────────┐
  │ Client ├───→│  L7 LB       ├───→│ Server A │ (/api/*)
  │        │    │ (HTTP proxy)  │───→│ Server B │ (/web/*)
  └────────┘    └──────────────┘    └──────────┘

  - Sees: URL, headers, cookies, body (after TLS termination)
  - Can: Route by path, add/remove headers, rate limit, cache
  - Speed: Slower (~milliseconds, must parse HTTP)
  - Use case: Microservice routing, A/B testing, canary deployments
```

| Feature | L4 (NLB/HAProxy TCP) | L7 (ALB/Nginx/Envoy) |
|---------|---------------------|---------------------|
| TLS termination | Pass-through or terminate | Terminate (required to read HTTP) |
| Path-based routing | No | Yes |
| WebSocket support | Yes (transparent) | Yes (upgrade aware) |
| Sticky sessions | IP hash only | Cookie-based |
| Health checks | TCP connect | HTTP endpoint |
| Latency added | ~5-50μs | ~0.5-2ms |
| Cost | Lower | Higher |
| HTTP/2 awareness | No | Yes |

---

## 8. Performance Engineering & Debugging

### 8.1 Full Request Lifecycle — Where Time Goes

```
Client Request → CDN/Edge → Load Balancer → Spring Boot App → Database → Response

Latency Budget Example (target p99 < 500ms):

┌─────────────────────────────────────────────────────────────────┐
│ CDN           │ LB    │ App Queuing │ Auth │ Logic │ DB │ Resp │
│ 5ms           │ 1ms   │ 50ms        │ 2ms  │ 10ms  │ 30ms│ 5ms │
└─────────────────────────────────────────────────────────────────┘
  Total p50: ~103ms ✓

  p99 adds:
    + GC pause: 200ms (occasional full GC)
    + DB connection wait: 150ms (pool contention)
    + Lock contention: 50ms (synchronized block)
  Total p99: ~503ms ✗ (just over budget)
```

### 8.2 Latency Decomposition in Practice

```java
@Component
public class LatencyTracker {

    private final MeterRegistry meterRegistry;

    public <T> T trackPhase(String phase, Supplier<T> operation) {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            return operation.get();
        } finally {
            sample.stop(Timer.builder("request.phase.duration")
                .tag("phase", phase)
                .publishPercentiles(0.5, 0.95, 0.99, 0.999)
                .register(meterRegistry));
        }
    }
}

@Service
public class OrderService {

    private final LatencyTracker tracker;

    public OrderResponse processOrder(CreateOrderRequest request) {
        // Track each phase independently
        User user = tracker.trackPhase("auth",
            () -> authService.validateAndGetUser(request.getToken()));

        InventoryResult inventory = tracker.trackPhase("inventory_check",
            () -> inventoryService.checkAvailability(request.getItems()));

        PaymentResult payment = tracker.trackPhase("payment",
            () -> paymentService.charge(user, request.getTotal()));

        Order order = tracker.trackPhase("db_write",
            () -> orderRepository.save(Order.from(request, user, payment)));

        return OrderResponse.from(order);
    }
}
```

### 8.3 GC Pauses — The Silent Killer

```
GC Impact on Latency:

Normal request: 10ms
During Minor GC (G1 Young): 10ms + 5-20ms pause = 15-30ms
During Mixed GC (G1): 10ms + 50-200ms pause = 60-210ms
During Full GC: 10ms + 500ms-5s pause = everything is broken

┌──────────────────────────────────────────┐
│  Request Timeline with GC Pauses         │
│                                          │
│  ────────┤▓▓▓▓├──────────────────── p50  │
│           10ms                           │
│                                          │
│  ────────┤▓▓▓▓▓▓▓▓▓▓├──────────── p99   │
│           10ms + 20ms GC                 │
│                                          │
│  ────────┤▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓├──── p999  │
│           10ms + 500ms Full GC           │
└──────────────────────────────────────────┘
```

**GC tuning for Spring Boot services:**

```bash
# G1GC (default since Java 9) — balanced latency and throughput
java -XX:+UseG1GC \
     -XX:MaxGCPauseMillis=100 \        # Target max pause time
     -XX:G1HeapRegionSize=16m \        # Region size (adjust for heap size)
     -XX:InitiatingHeapOccupancyPercent=45 \  # Start mixed GC earlier
     -Xms4g -Xmx4g \                  # Fixed heap (avoid resize pauses)
     -XX:+ParallelRefProcEnabled \     # Parallel reference processing
     -Xlog:gc*:file=gc.log:time,uptime,level,tags  # GC logging
     -jar app.jar

# ZGC (Java 17+) — ultra-low latency (sub-millisecond pauses)
java -XX:+UseZGC \
     -XX:+ZGenerational \              # Generational ZGC (Java 21+)
     -Xms4g -Xmx4g \
     -jar app.jar
# ZGC pauses: <1ms regardless of heap size
# Trade-off: ~10-15% throughput reduction vs G1

# Shenandoah — similar to ZGC, available in OpenJDK
java -XX:+UseShenandoahGC \
     -Xms4g -Xmx4g \
     -jar app.jar
```

### 8.4 Heap vs Stack

```
Stack (per thread):
  - Default size: 1MB (-Xss)
  - Stores: local variables, method call frames, return addresses
  - Allocation: instantaneous (move stack pointer)
  - Deallocation: automatic (method return)
  - Thread-local: no sharing, no contention
  - Overflow: StackOverflowError (deep recursion)

Heap (shared):
  - Size: -Xms / -Xmx
  - Stores: objects, arrays, class instances
  - Allocation: TLAB (thread-local allocation buffer) — fast, no contention
  - Deallocation: GC (the source of pauses)
  - Shared: requires synchronization for mutable shared objects
  - Overflow: OutOfMemoryError

Key insight: Reducing heap allocation reduces GC pressure.
  - Reuse objects (object pooling for expensive objects)
  - Use primitives instead of wrappers (int vs Integer)
  - Avoid unnecessary autoboxing
  - Use StringBuilder instead of String concatenation in loops
  - Stream.toList() creates less garbage than Collectors.toList() (Java 16+)
```

### 8.5 Database Connection Pool Exhaustion

**Symptoms:**
- p99 latency spikes
- Thread dumps show many threads `WAITING` on `HikariPool-1.getConnection()`
- Hikari logs: `Connection is not available, request timed out after 30000ms`
- Application appears hung but CPU is idle

**Root causes:**

```
Cause 1: Pool too small
  200 Tomcat threads, 10 DB connections
  Fix: Increase pool size (but not beyond what DB can handle)

Cause 2: Long-running transactions
  @Transactional on a method that calls an external API
  DB connection is held for the entire API call duration
  Fix: Don't hold transactions open during I/O

Cause 3: Connection leak
  Connection borrowed but never returned (missing close, exception in middle)
  Fix: Enable leak detection:
    spring.datasource.hikari.leak-detection-threshold=60000

Cause 4: Slow queries
  A query takes 10 seconds, holding a connection
  Fix: Query optimization, read replicas, caching
```

```java
// BAD — holds DB connection while calling external API
@Transactional
public OrderResponse createOrder(CreateOrderRequest request) {
    Order order = orderRepository.save(new Order(request));     // uses DB connection
    PaymentResult payment = paymentClient.charge(order);         // HTTP call (2 seconds!)
    order.setPaymentId(payment.getId());
    orderRepository.save(order);                                 // still same connection
    return OrderResponse.from(order);
}
// Connection held for: DB write + HTTP call + DB write = ~2.05 seconds

// GOOD — minimize transaction scope
public OrderResponse createOrder(CreateOrderRequest request) {
    Order order = saveOrder(request);                            // Short transaction
    PaymentResult payment = paymentClient.charge(order);         // No DB connection held
    updateOrderWithPayment(order.getId(), payment.getId());      // Short transaction
    return OrderResponse.from(order);
}

@Transactional
protected Order saveOrder(CreateOrderRequest request) {
    return orderRepository.save(new Order(request));             // ~50ms
}

@Transactional
protected void updateOrderWithPayment(Long orderId, String paymentId) {
    orderRepository.updatePaymentId(orderId, paymentId);         // ~50ms
}
```

### 8.6 Slow Consumer Problem

```
Scenario: Service A sends data to Service B
  Service A writes 1000 RPS to B
  Service B processes 500 RPS

  TCP layer: B's receive buffer fills → B advertises zero window → A blocks on write()
  HTTP layer: B responds slowly → A's thread pool fills with waiting threads
  Kafka: B's consumer lag grows → messages age out → data loss

  The slow consumer doesn't just affect itself — it applies backpressure
  that can cascade to producers and their producers.
```

### 8.7 Debugging Tools

**jstack — thread dump analysis:**

```bash
# Take a thread dump of a running JVM
jstack <pid> > thread_dump.txt

# What to look for:
# 1. Many threads in BLOCKED state — lock contention
# 2. Many threads in WAITING on same monitor — resource starvation
# 3. Many threads in TIMED_WAITING at same location — pool exhaustion
# 4. Deadlock detection (jstack reports these automatically)

# Example output showing pool exhaustion:
# "http-nio-8080-exec-150" WAITING
#   at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:186)
# "http-nio-8080-exec-151" WAITING
#   at com.zaxxer.hikari.pool.HikariPool.getConnection(HikariPool.java:186)
# ... (repeated 190 times — 190 of 200 threads waiting for DB connection)
```

**Spring Actuator + Micrometer + Prometheus:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus,threaddump,heapdump
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true   # Full histogram for latency analysis
      percentiles:
        http.server.requests: 0.5,0.95,0.99,0.999
      slo:
        http.server.requests: 100ms,200ms,500ms,1s  # SLO buckets
    tags:
      application: order-service
```

```java
@Component
public class CustomMetrics {

    private final MeterRegistry registry;

    @PostConstruct
    public void registerMetrics() {
        // Gauge: current value (connection pool, queue size)
        Gauge.builder("hikaricp.connections.active", dataSource,
                ds -> ((HikariDataSource)ds).getHikariPoolMXBean().getActiveConnections())
            .register(registry);

        // Counter: cumulative count (orders processed, errors)
        Counter.builder("orders.processed")
            .tag("status", "success")
            .register(registry);

        // Timer: latency distribution
        Timer.builder("external.api.duration")
            .tag("service", "payment-gateway")
            .publishPercentiles(0.5, 0.95, 0.99)
            .register(registry);

        // Distribution summary: request/response sizes
        DistributionSummary.builder("http.response.size")
            .baseUnit("bytes")
            .publishPercentiles(0.5, 0.95)
            .register(registry);
    }
}
```

### 8.8 How a Senior Reasons During a Production Incident

```
ALERT: p99 latency spike from 200ms to 5 seconds, 2% error rate

Senior's mental checklist (ordered by likelihood):

1. RECENT CHANGE? Was anything deployed in the last hour?
   → git log, deployment dashboard
   → If yes: rollback first, investigate later

2. DOWNSTREAM DEPENDENCY? Is a service we depend on degraded?
   → Check dependency health dashboards
   → Check circuit breaker state
   → Check external API status pages

3. DATABASE? Connection pool exhaustion, slow queries, locks?
   → HikariCP metrics (active connections, pending threads)
   → Slow query log
   → SELECT * FROM pg_stat_activity WHERE state = 'active'

4. JVM? GC pauses, heap pressure, thread issues?
   → GC logs (look for Full GC)
   → Heap utilization trend
   → Thread dump (jstack) — look for BLOCKED/WAITING patterns

5. INFRASTRUCTURE? CPU, memory, disk, network?
   → CPU: >90% → CPU-bound, check for hot loops
   → Memory: swapping → OOM pressure
   → Disk: iowait high → disk-bound
   → Network: packet drops, TCP retransmits

6. TRAFFIC? Sudden spike? DDoS? Retry storm?
   → RPS chart — did traffic jump?
   → Are clients retrying failed requests? (check for exponential growth)

7. RESOURCE LEAK? Growing over time?
   → Thread count growing? (thread leak)
   → Connection count growing? (connection leak)
   → Memory growing without GC recovery? (memory leak)

Action: Mitigate first (scale up, shed load, circuit-break), then root cause.
Never spend 30 minutes debugging while users are impacted.
```

---

## 9. Security & Reliability

### 9.1 JWT Trade-offs

```
JWT Architecture:
  Auth Server issues JWT → Client stores JWT → Client sends JWT with requests
  API Server validates JWT locally (no network call to Auth Server)

Token Structure:
  Header.Payload.Signature (Base64URL encoded)

  Header:  {"alg": "RS256", "typ": "JWT", "kid": "key-2024-01"}
  Payload: {"sub": "user-123", "roles": ["ADMIN"], "exp": 1700000000, "iat": 1699996400}
  Signature: RSASHA256(base64(header) + "." + base64(payload), privateKey)
```

| Advantage | Disadvantage |
|-----------|-------------|
| Stateless — no session store needed | Cannot revoke individual tokens before expiry |
| Scales horizontally — any instance can validate | Token size grows with claims (1KB+ common) |
| No DB lookup for auth | Sensitive data in payload (base64 is NOT encryption) |
| Works across services | Clock skew can cause premature rejection |
| Offline validation | Key rotation requires coordination |

**Spring Security JWT implementation:**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable()) // Stateless API — CSRF not applicable
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/**").authenticated()
                .anyRequest().denyAll()  // Deny by default
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .decoder(jwtDecoder())
                    .jwtAuthenticationConverter(jwtAuthConverter())
                )
            )
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((req, res, e) -> {
                    res.setStatus(401);
                    res.setContentType("application/json");
                    res.getWriter().write("{\"error\":\"unauthorized\"}");
                })
            )
            .build();
    }

    @Bean
    public JwtDecoder jwtDecoder() {
        // Validate against JWKS endpoint — supports key rotation
        return NimbusJwtDecoder
            .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
            .build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthorities = new JwtGrantedAuthoritiesConverter();
        grantedAuthorities.setAuthoritiesClaimName("roles");
        grantedAuthorities.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(grantedAuthorities);
        return converter;
    }
}
```

### 9.2 Token Revocation Strategies

Since JWTs are stateless, you cannot "delete" a token. Revocation strategies:

| Strategy | How It Works | Trade-off |
|----------|-------------|-----------|
| Short-lived tokens (5-15 min) | Token expires quickly, use refresh tokens | Good security, more complexity |
| Token blocklist (Redis) | Check revoked token IDs on every request | Adds stateful check, defeats purpose |
| Token versioning | Store `tokenVersion` per user in DB, reject older versions | Revokes ALL user tokens, not one |
| Refresh token rotation | Issue new access + refresh on each refresh | Stolen refresh token detected on reuse |

```java
// Token versioning approach
@Service
public class TokenRevocationService {

    private final RedisTemplate<String, String> redis;

    public void revokeAllTokensForUser(String userId) {
        // Increment version — all existing tokens become invalid
        redis.opsForValue().increment("token_version:" + userId);
    }

    public boolean isTokenRevoked(String userId, int tokenVersion) {
        String currentVersion = redis.opsForValue().get("token_version:" + userId);
        return currentVersion != null && tokenVersion < Integer.parseInt(currentVersion);
    }
}
```

### 9.3 Replay Attacks

```
Attacker intercepts a valid request:
  POST /api/transfer
  Authorization: Bearer <valid_jwt>
  Body: {"to": "attacker", "amount": 1000}

Replays the same request 10 times → $10,000 transferred

Prevention:
  1. TLS (prevents interception in transit)
  2. Short-lived tokens (limits replay window)
  3. Nonce/Idempotency key (prevents duplicate processing)
  4. Timestamp validation (reject requests older than 5 minutes)
  5. Request signing (HMAC of body + timestamp + nonce)
```

### 9.4 Rate Limiting

```java
@Configuration
public class RateLimitConfig {

    @Bean
    public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
        FilterRegistrationBean<RateLimitFilter> registration = new FilterRegistrationBean<>();
        registration.setFilter(new RateLimitFilter(rateLimiter()));
        registration.addUrlPatterns("/api/*");
        registration.setOrder(1); // Before auth filter
        return registration;
    }
}

// Token bucket rate limiter using Resilience4j
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    private final LoadingCache<String, RateLimiter> limiters;

    public RateLimitFilter() {
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitForPeriod(100)              // 100 requests
            .limitRefreshPeriod(Duration.ofMinutes(1))  // per minute
            .timeoutDuration(Duration.ZERO)    // Don't wait, reject immediately
            .build();

        this.limiters = Caffeine.newBuilder()
            .expireAfterAccess(Duration.ofMinutes(10))
            .maximumSize(100_000)
            .build(key -> RateLimiter.of("rl-" + key, config));
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws ServletException, IOException {
        String clientId = extractClientIdentifier(request); // IP, API key, or user ID

        RateLimiter limiter = limiters.get(clientId);

        if (limiter.acquirePermission()) {
            response.setHeader("X-RateLimit-Remaining",
                String.valueOf(limiter.getMetrics().getAvailablePermissions()));
            chain.doFilter(request, response);
        } else {
            response.setStatus(429);
            response.setHeader("Retry-After", "60");
            response.getWriter().write("{\"error\":\"rate_limit_exceeded\"}");
        }
    }

    private String extractClientIdentifier(HttpServletRequest request) {
        // Prefer API key > authenticated user > IP
        String apiKey = request.getHeader("X-API-Key");
        if (apiKey != null) return "api:" + apiKey;

        String userId = (String) request.getAttribute("userId");
        if (userId != null) return "user:" + userId;

        return "ip:" + request.getRemoteAddr();
    }
}
```

### 9.5 Circuit Breakers (Resilience4j)

```
Circuit Breaker States:

  CLOSED (normal) ──failure rate > threshold──→ OPEN (failing)
       ↑                                           │
       │                                      wait duration
       │                                           │
       └──── success ← ── HALF_OPEN (testing) ←────┘
                          (permits N test requests)
```

```java
@Configuration
public class CircuitBreakerConfig {

    @Bean
    public CircuitBreaker paymentCircuitBreaker() {
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)              // Open if 50% of calls fail
            .slowCallRateThreshold(80)             // Open if 80% of calls are slow
            .slowCallDurationThreshold(Duration.ofSeconds(3))  // "slow" = >3s
            .slidingWindowSize(10)                 // Evaluate last 10 calls
            .slidingWindowType(SlidingWindowType.COUNT_BASED)
            .minimumNumberOfCalls(5)               // Need 5 calls before evaluating
            .waitDurationInOpenState(Duration.ofSeconds(30))  // Wait 30s before half-open
            .permittedNumberOfCallsInHalfOpenState(3)  // Allow 3 test calls
            .recordExceptions(IOException.class, TimeoutException.class)
            .ignoreExceptions(BusinessException.class)  // Don't count business errors
            .build();

        return CircuitBreaker.of("payment-service", config);
    }
}

@Service
public class PaymentService {

    private final CircuitBreaker circuitBreaker;
    private final WebClient paymentClient;

    public PaymentResult processPayment(PaymentRequest request) {
        return CircuitBreaker.decorateSupplier(circuitBreaker,
            () -> paymentClient.post()
                .uri("/api/charge")
                .bodyValue(request)
                .retrieve()
                .bodyToMono(PaymentResult.class)
                .block(Duration.ofSeconds(5))
        ).get();
    }

    // With fallback
    public PaymentResult processPaymentWithFallback(PaymentRequest request) {
        try {
            return circuitBreaker.executeSupplier(() -> callPaymentGateway(request));
        } catch (CallNotPermittedException e) {
            // Circuit is OPEN — don't even try
            log.warn("Payment circuit open, using fallback");
            return PaymentResult.deferred("Payment will be processed when service recovers");
        }
    }
}
```

### 9.6 Retry Storms

```
Scenario:
  Service A → Service B (experiencing errors)

  Without jitter:
    T=0s:  1000 clients retry simultaneously → B gets 1000 requests → B crashes
    T=1s:  1000 clients retry again → B crashes again
    T=2s:  1000 clients retry again → B never recovers

  With exponential backoff + jitter:
    T=0s:  1000 clients retry
    T=1-2s: ~300 clients retry (random jitter spreads them)
    T=2-4s: ~100 clients retry (exponential backoff)
    T=4-8s: ~30 clients retry
    → B has time to recover between waves
```

```java
// Retry with exponential backoff and jitter
@Bean
public Retry paymentRetry() {
    RetryConfig config = RetryConfig.custom()
        .maxAttempts(3)
        .waitDuration(Duration.ofMillis(500))
        .intervalFunction(IntervalFunction.ofExponentialRandomBackoff(
            500,       // initial wait (ms)
            2.0,       // multiplier
            0.5,       // randomization factor (±50%)
            30_000     // max wait (ms)
        ))
        .retryOnException(e -> e instanceof IOException || e instanceof TimeoutException)
        .retryOnResult(response -> response == null)  // Retry on null response
        .ignoreExceptions(BusinessValidationException.class)  // Don't retry business errors
        .failAfterMaxAttempts(true)
        .build();

    return Retry.of("payment-retry", config);
}
```

**Retry budget pattern:** Instead of each client deciding independently how much to retry, set a global retry budget: "retry at most 10% additional traffic." If the error rate is 50%, that means only 20% of failed requests get retried. This prevents retry amplification.

### 9.7 mTLS (Mutual TLS)

```
Standard TLS:
  Client verifies server's certificate (one-way)
  Server doesn't know who the client is

mTLS:
  Client verifies server's certificate
  Server verifies client's certificate (two-way)
  Both sides are authenticated at the transport layer

Use case: Service-to-service authentication in microservices
  - No need for API keys or JWTs between internal services
  - Certificate rotation handled by service mesh (Istio/Linkerd)
```

```java
// Spring Boot mTLS server configuration
@Configuration
public class MtlsConfig {

    @Bean
    public WebServerFactoryCustomizer<TomcatServletWebServerFactory> tlsCustomizer() {
        return factory -> {
            factory.setSsl(createSsl());
        };
    }

    private Ssl createSsl() {
        Ssl ssl = new Ssl();
        ssl.setKeyStore("classpath:server-keystore.p12");
        ssl.setKeyStorePassword("changeit");
        ssl.setKeyStoreType("PKCS12");
        ssl.setTrustStore("classpath:truststore.p12");  // Contains CA certs
        ssl.setTrustStorePassword("changeit");
        ssl.setClientAuth(Ssl.ClientAuth.NEED);  // Require client certificate
        return ssl;
    }
}
```

### 9.8 OWASP Backend Risks (Top Concerns)

| Risk | Java/Spring Mitigation |
|------|----------------------|
| Injection (SQL, LDAP, OS) | Parameterized queries (JPA/JDBC), input validation |
| Broken Authentication | Spring Security, short-lived JWTs, MFA |
| Sensitive Data Exposure | Encrypt at rest (JCE), mask in logs, don't return in APIs |
| Broken Access Control | Method-level `@PreAuthorize`, RBAC, principle of least privilege |
| Security Misconfiguration | Actuator secured, error messages sanitized, headers (HSTS, CSP) |
| Mass Assignment | Use DTOs, never bind directly to entities, `@JsonIgnore` |
| SSRF | Whitelist allowed hostnames, don't accept URLs from users |
| Insecure Deserialization | Avoid Java serialization, use JSON with Jackson type validation |

```java
// Mass assignment prevention
// BAD — attacker can set isAdmin=true in JSON body
@PostMapping("/users")
public User createUser(@RequestBody User user) { // Entity directly from request
    return userRepository.save(user);
}

// GOOD — DTO limits what can be set
@PostMapping("/users")
public UserResponse createUser(@Valid @RequestBody CreateUserRequest request) {
    User user = new User();
    user.setName(request.getName());
    user.setEmail(request.getEmail());
    // isAdmin is NOT settable from the request
    user.setAdmin(false);
    return UserResponse.from(userRepository.save(user));
}
```

---

## 10. Failure Case Studies

### Case Study 1: The Cascading Timeout

**What happened:** An e-commerce checkout service experienced a complete outage during a flash sale. All endpoints returned 503 errors.

**Root cause:** The payment gateway became slow (not down — p99 went from 200ms to 8 seconds) due to its own database load. The checkout service had default RestTemplate timeouts (infinite). All 200 Tomcat threads became blocked waiting for the payment gateway. New requests queued up, then timed out at the load balancer (30s). The retry logic on the frontend made things worse — each user retry doubled the load.

**System behavior:**
- Tomcat thread pool: 200/200 active (100% utilization)
- All threads blocked at `SocketInputStream.read()`
- Incoming requests queued → LB timeout → 503
- Error rate jumped from 0.1% to 100% in 3 minutes

**Metrics observed:**
- Thread pool utilization: 20% → 100% in 90 seconds
- p99 latency: 200ms → 30,000ms (LB timeout)
- Error rate: 0.1% → 100%
- Payment gateway p99: 200ms → 8,000ms
- CPU utilization: 40% → 5% (threads blocked, no CPU work happening)

**Prevention:**
1. Explicit timeouts on all outbound calls: connect=500ms, read=2s
2. Circuit breaker on payment calls: open at 50% failure rate
3. Bulkhead: Separate thread pool for payment calls (isolate from other endpoints)
4. Retry budget with exponential backoff and jitter

---

### Case Study 2: The Connection Pool Leak

**What happened:** A Spring Boot service slowly degraded over 3 days after deployment. Restart fixed it temporarily, but latency crept up again.

**Root cause:** A code path in error handling opened a manual JDBC connection (bypassing HikariCP) and didn't close it in the `finally` block. Under normal operation, this path was rarely hit. But a partner API started returning occasional errors, triggering the path more frequently.

**System behavior:**
- HikariCP active connections grew: 5 → 6 → 7 → ... → 10 (max) over 3 days
- Once pool was exhausted, all requests waiting for connections
- Leak detection log: "Connection leak detection triggered for connection-7, stack trace..."

**Metrics observed:**
- Active connections: linear growth over days (not correlated to traffic)
- Pending threads: 0 → 5 → 50 → 190 (once pool hit max)
- p99: stable at 50ms for 2 days, then suddenly 30,000ms

**Prevention:**
1. Enable HikariCP leak detection: `leak-detection-threshold: 60000`
2. Use try-with-resources for ALL connection handling
3. Code review: search for manual `DataSource.getConnection()` calls
4. Alert on connection pool active count growing without traffic growth

---

### Case Study 3: The GC Death Spiral

**What happened:** A product catalog service had intermittent latency spikes every 30-60 minutes. p99 went from 100ms to 15 seconds during spikes.

**Root cause:** The service loaded a large product catalog into an in-memory HashMap on startup (800MB). With a 2GB heap, this left only 1.2GB for transient objects. Under moderate load, the young generation filled quickly, promoting short-lived objects to the old generation. G1GC mixed collections became frequent, and occasionally a Full GC was triggered, pausing all threads for 10-15 seconds.

**Metrics observed:**
- Heap: oscillating between 1.5GB and 1.95GB
- GC pause time: Young GC 20ms (every 2s), Mixed GC 200ms (every 30s), Full GC 12s (every 60 min)
- p99: 100ms normally, 15,000ms during Full GC
- CPU: 30% normally, 100% during GC (GC threads consuming all cores)

**Prevention:**
1. Move catalog to external cache (Redis) — reduce heap pressure
2. Increase heap size: `-Xms8g -Xmx8g` (4× headroom)
3. Switch to ZGC: max pause <1ms regardless of heap size
4. If keeping in-memory: use off-heap storage (Chronicle Map, MapDB)

---

### Case Study 4: The Retry Storm That Took Down Three Services

**What happened:** A microservices platform experienced cascading failures across three services during a routine database maintenance window.

**Root cause:** Service C's database was undergoing a 5-minute maintenance window. Service B called Service C, got timeouts, and retried 3 times with no backoff. Service A called Service B, got timeouts (because B was waiting for retries to C), and retried 3 times. Normal traffic: 1000 RPS. During the incident: 1000 × 3 (B retries) × 3 (A retries) = 27,000 RPS hitting Service C's database. When the database came back, it was immediately overwhelmed.

**Metrics observed:**
- Service C: 1000 RPS → 9000 RPS (3x retries from B)
- Service B: 1000 RPS → 3000 RPS (3x retries from A)
- Total load amplification: 27×
- Recovery time after DB maintenance: 45 minutes (should have been 0)

**Prevention:**
1. Exponential backoff with jitter on all retries
2. Retry budget: max 10% additional traffic from retries
3. Circuit breakers at each service boundary
4. Coordinated maintenance windows with dependency awareness

---

### Case Study 5: The Ephemeral Port Exhaustion

**What happened:** A notification service started failing intermittently with `java.net.BindException: Address already in use` when calling an SMS API. Restarts didn't help. Only ~40% of SMS messages were being sent.

**Root cause:** The service used `new URL(smsApiUrl).openConnection()` for each SMS — creating a new TCP connection per request. Each connection, after closing, entered TIME_WAIT for 60 seconds. At 500 SMS/second, that's 30,000 connections in TIME_WAIT. The ephemeral port range (32768-60999) = 28,231 ports. They ran out.

**Metrics observed:**
- `ss -s | grep TIME-WAIT`: 28,000+ TIME_WAIT sockets
- `dmesg | grep "port range"`: kernel logs about exhaustion
- SMS delivery rate: 500/s → fluctuating 200-400/s
- Error rate: 0% → 60% (correlated with TIME_WAIT count)

**Prevention:**
1. Use connection pooling (Apache HttpClient, OkHttp, or WebClient) — connections are reused, not recreated
2. Tune kernel: `net.ipv4.tcp_tw_reuse=1`
3. Monitor TIME_WAIT socket count — alert if >10,000
4. Code review: ban `URL.openConnection()` and `new HttpURLConnection()` — enforce pooled clients

---

## 11. Senior-Level Interview Question Bank

### 11.1 Communication Patterns

**Q: "You're designing a notification system. When would you use polling vs WebSocket vs SSE?"**

Weak answer: "WebSocket is real-time so I'd use that."

Senior answer: "It depends on the notification requirements. For email-style notifications where users check periodically, simple polling with caching is sufficient — it's stateless and scales trivially behind a CDN. For a live dashboard showing order status updates, SSE is ideal — it's server-to-client, works through proxies, and the browser's EventSource API handles reconnection automatically. WebSocket is warranted only when we need bidirectional real-time communication — like a collaborative editing feature or chat — because it adds operational complexity: sticky sessions, connection state management across instances, and graceful drain during deployments. The key trade-off is operational complexity vs capability."

**Deep follow-ups:**
- How do you handle SSE across multiple server instances? (Redis Pub/Sub fan-out)
- What happens to in-flight WebSocket messages during a rolling deployment? (graceful drain, client reconnection with message replay)
- How do you handle 100K concurrent SSE connections? (file descriptor limits, kernel tuning, consider a dedicated push service)

---

### 11.2 Concurrency and Threading

**Q: "Your Spring Boot service has 200 threads and 10 DB connections. Under load, latency spikes. What's happening and how do you fix it?"**

What weak candidates miss: They jump to "increase the pool size" without explaining the mechanics — that 190 threads are parked waiting on the HikariCP semaphore, each consuming 1MB of stack memory, and that the real fix might be reducing the time each connection is held (shorter transactions, query optimization) rather than adding more connections.

Senior answer: "The thread pool and connection pool are mismatched. At 200 concurrent requests, all threads attempt to acquire a DB connection, but only 10 can proceed. The other 190 threads are in WAITING state on HikariCP's semaphore. I'd first check: Are connections being held too long? Look for `@Transactional` on methods that include external API calls — that holds a connection for the entire duration. Then check slow queries with Hikari leak detection. The fix depends on the bottleneck: if queries are fast but we genuinely need more parallelism, increase pool size (but the DB also has connection limits — PostgreSQL defaults to 100). If queries are slow, optimize them. If we're holding connections unnecessarily during non-DB work, restructure the code to minimize transaction scope."

---

### 11.3 Performance

**Q: "How do you diagnose a p99 latency spike that doesn't show up in p50?"**

What weak candidates miss: They don't understand that p50 and p99 can have completely different root causes. p50 reflects the typical case. p99 reflects tail latency — GC pauses, lock contention, cold caches, connection pool waits.

Senior answer: "A p99 spike with stable p50 means the typical request is fine but 1% of requests hit a slow path. Common causes: (1) GC pauses — check GC logs for pauses correlating with the latency spikes. (2) Connection pool contention — 99% of requests get a connection immediately, but 1% wait because the pool is briefly exhausted during traffic bursts. (3) Lock contention — a synchronized block that's usually fast but occasionally blocks when multiple threads hit it simultaneously. (4) Cold cache — most requests hit cache, but 1% miss and hit the database. (5) Background tasks — a scheduled job that runs periodically and competes for resources. I'd correlate the timing of p99 spikes with GC logs, thread dumps, and connection pool metrics to identify which factor dominates."

---

### 11.4 System Design

**Q: "How would you design idempotency for a payment API?"**

What weak candidates miss: They describe the idempotency key concept but don't address concurrent requests with the same key, what happens during network partitions between the API and the database, or how to handle idempotency key expiration.

Senior answer: "I'd implement a database-backed idempotency layer. The client provides an Idempotency-Key header (UUID). On receiving a request: (1) Attempt to INSERT a row with the key and IN_PROGRESS status using a unique constraint — this handles concurrent duplicate requests via database-level locking. (2) If INSERT succeeds, process the payment, then UPDATE the row to COMPLETED with the serialized response. (3) If INSERT fails (duplicate key), check the status: if COMPLETED, return the cached response; if IN_PROGRESS, return 409 Conflict (another request is processing). (4) Idempotency keys expire after 24-48 hours to prevent table growth. Edge case: if the process crashes after payment but before marking COMPLETED, the key is stuck IN_PROGRESS. I'd add a timeout — if IN_PROGRESS for >5 minutes, allow a new attempt with the same key. The payment processor should also be queried to check if the charge actually went through."

---

### 11.5 Security

**Q: "What are the security trade-offs of JWT vs session-based auth?"**

Senior answer: "JWTs are stateless — the server doesn't need a session store, which simplifies horizontal scaling. But the trade-off is revocation. You can't invalidate a JWT before expiry without adding state back (a blocklist). Session-based auth is stateful — the server can immediately invalidate a session by deleting it — but requires a shared session store (Redis) across instances. For our use case, I'd choose JWTs with short expiry (15 min) and refresh tokens (stored server-side with revocation). This gives us the scaling benefit of JWTs for the 99% case (valid tokens validated without any external call) and the revocation capability for the 1% case (user changes password, logout). The refresh token endpoint is the only stateful call."

---

### 11.6 Reliability

**Q: "A downstream service is returning errors intermittently. How do you prevent it from taking down your service?"**

Senior answer: "I'd apply defense in depth. Layer 1: Timeouts — set explicit connect (500ms) and read (2s) timeouts so we fail fast instead of hanging. Layer 2: Circuit breaker — using Resilience4j, open the circuit if failure rate exceeds 50% over the last 10 calls. When open, return a degraded response immediately without calling the downstream at all. Layer 3: Bulkhead — run downstream calls in a separate thread pool (e.g., 20 threads) so if they all block, they don't consume Tomcat threads. Layer 4: Retry with exponential backoff and jitter — max 2 retries, 500ms → 1s → 2s with ±50% jitter. Layer 5: Graceful degradation — define what the service does when the downstream is unavailable. Can we serve cached data? Can we queue the request for later? Can we return a partial response?"

---

## 12. System Design Practice Questions

### 12.1 Design a Rate-Limited API Gateway

**Requirements:**
- Handle 100K RPS across all tenants
- Per-tenant rate limits (configurable: 100-10,000 RPS)
- Burst allowance (2× sustained rate for 10 seconds)
- Return 429 with Retry-After header when limited
- Distributed across multiple gateway instances

**Constraints:**
- p99 overhead: <2ms per request
- Rate limit accuracy: ±5% acceptable
- Must work across multiple gateway instances (shared state)

**Scaling challenges:**
- Centralized counter (Redis) becomes a bottleneck at 100K RPS
- Local counters drift apart, allowing 10× the intended rate across 10 instances
- Token bucket vs sliding window vs fixed window trade-offs

**Evaluation rubric:**
- Does the candidate discuss distributed rate limiting (not just single-node)?
- Do they address clock synchronization across instances?
- Do they consider the latency cost of a Redis round-trip on every request?
- Do they propose local rate limiting with periodic sync as an optimization?
- Do they handle race conditions (check-and-increment must be atomic)?

---

### 12.2 Design an Event-Driven Order Processing System

**Requirements:**
- Accept orders via REST API (synchronous acknowledgment)
- Process payment, inventory, and shipping asynchronously
- Handle partial failures (payment succeeds, inventory fails)
- Support order status tracking in real-time
- Process 5,000 orders/minute at peak

**Constraints:**
- Orders must not be lost (durability)
- Each step must be idempotent (at-least-once delivery)
- Eventual consistency is acceptable (but users should see updates within 5 seconds)

**Scaling challenges:**
- Saga orchestration vs choreography trade-offs
- Dead letter queues and poison pill handling
- Compensating transactions (reverse payment if inventory fails)
- Event ordering guarantees (Kafka partition key strategy)

**Evaluation rubric:**
- Does the candidate separate the synchronous API from async processing?
- Do they address the dual-write problem (DB + queue)?
- Do they propose the Transactional Outbox pattern?
- Do they handle compensation for partial failures?
- Do they design for idempotency at each processing stage?

---

### 12.3 Design a Real-Time Notification System

**Requirements:**
- Support push notifications to 1M concurrent users
- Multiple channels: in-app (WebSocket/SSE), email, SMS
- Priority levels: critical (immediate), normal (batched), low (digest)
- User preferences: opt-in/opt-out per channel
- Delivery guarantees: at-least-once for critical, best-effort for low

**Constraints:**
- In-app notification delivery: <1 second for critical
- System must handle 50K notifications/second at peak
- Users may have multiple active sessions (phone + desktop)

**Scaling challenges:**
- Fan-out at scale (1 event → 1M users)
- Connection management for 1M WebSocket/SSE connections
- Session routing (which server holds which user's connection?)
- Graceful degradation when notification volume spikes

**Evaluation rubric:**
- Do they separate notification ingestion from delivery?
- Do they use a connection registry (Redis) for WebSocket routing?
- Do they batch and coalesce notifications to reduce fan-out?
- Do they distinguish infrastructure (connection management) from business logic (routing rules)?

---

### 12.4 Design a Distributed Job Scheduler

**Requirements:**
- Schedule one-time and recurring jobs (cron-like)
- Exactly-once execution guarantee
- Support job priorities and dependencies
- Handle worker failures (job reassignment)
- Scale to 100K jobs/day

**Constraints:**
- Job execution must start within 1 second of scheduled time
- No duplicate execution (critical for payment jobs)
- Job state must survive worker crashes
- Support delayed jobs (execute 30 days from now)

**Scaling challenges:**
- Distributed locking for job claims (pessimistic vs optimistic)
- Clock skew across instances
- Hot partitions (many jobs scheduled at midnight)
- Long-running jobs (heartbeat and lease extension)

**Evaluation rubric:**
- Do they use database-based locking (SELECT FOR UPDATE SKIP LOCKED) vs distributed locks (Redis/ZooKeeper)?
- Do they handle the thundering herd problem at popular schedule times?
- Do they separate the scheduler (decides when) from the executor (does the work)?
- Do they implement heartbeats and lease-based ownership?

---

### 12.5 Design a Multi-Tenant SaaS Backend

**Requirements:**
- Support 1000 tenants with isolated data
- Per-tenant resource limits (API rate, storage, compute)
- Tenant-specific configuration (features, integrations)
- Admin API for tenant management
- Support both small (10 users) and large (10K users) tenants

**Constraints:**
- Data isolation: tenants must never see each other's data
- Performance isolation: one tenant's heavy usage shouldn't affect others
- Cost efficiency: can't run dedicated infrastructure per tenant at small scale

**Scaling challenges:**
- Database strategy: shared DB with tenant_id column vs schema-per-tenant vs DB-per-tenant
- Noisy neighbor problem: large tenant consuming disproportionate resources
- Migration: moving a growing tenant from shared to dedicated infrastructure
- Cross-tenant queries for analytics (billing, usage reporting)

**Evaluation rubric:**
- Do they discuss the trade-off spectrum of shared vs isolated infrastructure?
- Do they propose a tiered approach (shared for small, dedicated for enterprise)?
- Do they address the noisy neighbor problem with bulkheads?
- Do they consider tenant-aware connection pooling and caching?

---

## 13. 30-Day Upgrade Plan (Mid → Senior)

### Week 1: Foundations — Threading, Memory, and the JVM

**Focus:** Understand what happens below your Spring Boot code.

**Day 1-2: Thread Model Deep Dive**
- Read: Java Concurrency in Practice (chapters 1-5)
- Lab: Write a thread-per-request HTTP server using raw `ServerSocket` (no frameworks). Handle 1000 concurrent connections. Observe thread count, memory usage, and at what point it breaks.
- Lab: Rewrite using `Selector` (Java NIO). Compare thread count and throughput.
- Exercise: Take a thread dump of a running Spring Boot app under load. Identify which threads are working vs waiting.

**Day 3-4: Memory and GC**
- Lab: Create a Spring Boot app that allocates objects aggressively. Profile with VisualVM. Observe Young GC, Old GC, and Full GC patterns.
- Lab: Tune GC parameters: try G1, ZGC, and Shenandoah. Measure p99 latency under load with each.
- Exercise: Analyze a heap dump. Find the biggest object trees. Identify potential memory leaks.

**Day 5: Connection Pooling**
- Lab: Configure HikariCP with intentionally small pool (2 connections). Load test with 50 concurrent users. Observe thread dump, connection wait times. Increase pool to 10, then 50. Graph the throughput curve.
- Exercise: Cause and diagnose a connection leak. Enable leak detection, find the stack trace.

**Deliverable:** Write a 1-page analysis: "What happens in my Spring Boot app from HTTP request to database query and back — every thread, every buffer, every syscall."

---

### Week 2: Networking and Protocols

**Day 6-7: TCP Deep Dive**
- Lab: Use `tcpdump` or Wireshark to capture a full HTTP request lifecycle. Identify: SYN, SYN-ACK, ACK, HTTP request, HTTP response, FIN sequence.
- Lab: Measure the cost of connection pooling vs no pooling. Use `curl` with `--connect-time` to measure TCP handshake cost to different destinations.
- Exercise: Intentionally fill up TIME_WAIT sockets. Monitor with `ss -s`. Understand why it happens.

**Day 8-9: TLS and HTTP/2**
- Lab: Configure Spring Boot with HTTP/2. Use `curl --http2 -v` to observe the connection setup and stream IDs.
- Lab: Set up mTLS between two Spring Boot services. Understand the certificate chain.
- Exercise: Compare response times: HTTP/1.1 (6 sequential calls) vs HTTP/2 (6 multiplexed calls). Measure the difference.

**Day 10: DNS and Service Discovery**
- Lab: Trace a DNS resolution in Java. Understand JVM DNS caching (`networkaddress.cache.ttl`). Observe what happens when a DNS entry changes and the JVM still has the old IP cached.
- Exercise: Configure Spring Cloud LoadBalancer. Understand client-side vs server-side load balancing.

**Deliverable:** Create a latency decomposition chart for a typical request in your application. Identify the biggest contributors.

---

### Week 3: Reliability Patterns and Failure Engineering

**Day 11-12: Circuit Breakers and Retries**
- Lab: Implement Resilience4j circuit breaker around a flaky downstream service (use WireMock to simulate failures). Observe state transitions: CLOSED → OPEN → HALF_OPEN → CLOSED.
- Lab: Implement retry with exponential backoff. Measure the traffic amplification factor.
- Exercise: Set up a retry storm scenario: Service A → B → C, each with 3 retries. Calculate total traffic amplification (27×). Then fix it with retry budgets.

**Day 13-14: Idempotency and Exactly-Once Semantics**
- Lab: Implement an idempotency layer using PostgreSQL with a unique constraint on the idempotency key. Handle concurrent duplicate requests.
- Lab: Implement the Transactional Outbox pattern: write to the outbox table in the same transaction as the business operation, then publish events from the outbox.
- Exercise: Write a test that sends the same request 10 times concurrently. Verify only one side effect occurs.

**Day 15: Chaos Engineering**
- Lab: Use Toxiproxy or tc (traffic control) to add latency and packet loss to a downstream dependency. Observe how your circuit breaker and timeouts behave.
- Exercise: Inject a 5-second pause into your database. Observe the cascading effect on your application's thread pool, error rate, and latency.

**Deliverable:** Document a "Resilience Runbook" for your service: what happens when each dependency is slow, down, or returning errors, and how your service handles each case.

---

### Week 4: System Design and Interview Preparation

**Day 16-18: System Design Practice**
- Practice: Design 3 of the 5 systems from Section 12. Write up your design with:
  - Component diagram
  - Data flow for key operations
  - Scaling strategy
  - Failure handling
  - Key trade-off decisions and alternatives considered
- Mock: Present each design verbally in 35 minutes (as you would in an interview). Record yourself. Identify where you're vague or hand-wavy.

**Day 19-20: Production Debugging Practice**
- Lab: Deploy your Spring Boot app with Prometheus + Grafana. Create dashboards for: request rate, error rate, latency percentiles, thread pool utilization, connection pool metrics, JVM heap, GC pauses.
- Lab: Introduce a bug (e.g., connection leak, memory leak, slow query) and diagnose it using only metrics and thread dumps — without reading the code.
- Exercise: Practice the incident response mental checklist from Section 8.8 against each scenario.

**Day 21: Mock Interviews**
- Conduct 2 mock interviews with peers or use a service:
  1. System design (45 min): Pick a problem from Section 12
  2. Deep dive (30 min): Pick questions from Section 11

**Model answers for practice:**

**Q: "Tell me about a time you debugged a production issue."**

Structure (STAR with technical depth):
"Our order service p99 latency jumped from 200ms to 8 seconds during peak traffic. I started by checking recent deployments — none. Then I checked downstream dependencies — the payment gateway's status page showed degradation. I pulled thread dumps from three instances — 180 of 200 threads were WAITING on `SocketInputStream.read()` for the payment gateway. We had no explicit timeouts configured (defaulting to infinite). I immediately deployed a configuration change: 2-second read timeout + circuit breaker (open at 50% failure rate). p99 dropped to 2.5 seconds (for requests that timed out). Then I added a fallback: accept orders with deferred payment processing. That brought p99 back to 200ms. Post-incident, we added timeouts and circuit breakers to all outbound calls, and added thread pool utilization alerts."

**Deliverable:** Complete 2 full mock interviews. Review recordings. Identify your top 3 knowledge gaps and study them.

---

### Daily Habits Throughout the 30 Days

| Time | Activity | Purpose |
|------|----------|---------|
| Morning (30 min) | Read one section of this guide | Build theoretical foundation |
| Midday (1 hour) | Hands-on lab | Build practical experience |
| Evening (30 min) | Answer 2 interview questions (write answers, don't just think) | Build articulation skill |
| Weekly | 1 mock interview with a peer | Build interview muscle memory |

### Key Mindset Shifts

1. **From "it works" to "it works under these conditions."** Always state your assumptions.
2. **From "use X technology" to "X technology because Y trade-off."** Always justify.
3. **From describing WHAT to explaining WHY.** Interviewers want to hear your reasoning.
4. **From single-server thinking to distributed thinking.** Default to multi-instance, multi-region.
5. **From happy path to failure path.** Start every design with "what happens when..."

---

*End of Senior Backend Interview Guide*
