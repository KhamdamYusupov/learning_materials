# gRPC, Protocol Buffers, and Spring Boot — A Senior-Level Deep Dive

> A production-grade guide for Java backend engineers (Spring Boot, Postgres, Kafka, Kubernetes) who already know REST and want to design, implement, debug, and operate gRPC services like a senior engineer.

---

## Table of Contents

1. [RPC Fundamentals](#1-rpc-fundamentals)
2. [gRPC Fundamentals](#2-grpc-fundamentals)
3. [Protocol Buffers Deep Dive](#3-protocol-buffers-deep-dive)
4. [gRPC Communication Models](#4-grpc-communication-models)
5. [gRPC Architecture and Internals](#5-grpc-architecture-and-internals)
6. [gRPC vs REST vs JSON-RPC vs GraphQL](#6-grpc-vs-rest-vs-json-rpc-vs-graphql)
7. [Java + Spring Boot Integration](#7-java--spring-boot-integration)
8. [Error Handling and Status Codes](#8-error-handling-and-status-codes)
9. [Security](#9-security)
10. [Performance and Optimization](#10-performance-and-optimization)
11. [Observability and Debugging](#11-observability-and-debugging)
12. [Production Best Practices](#12-production-best-practices)
13. [Real-World Scenarios](#13-real-world-scenarios)
14. [Common Pitfalls and Edge Cases](#14-common-pitfalls-and-edge-cases)
15. [When NOT to Use gRPC](#15-when-not-to-use-grpc)
16. [Final Checklist](#16-final-checklist)

---

## 1. RPC Fundamentals

### What RPC is

**Remote Procedure Call (RPC)** is a communication style where a client invokes a function on a remote machine **as if it were a local method**. The wire protocol, serialization, transport, and error model are abstracted away by stubs generated from a contract.

```
client.transferMoney(from, to, amount)
        │
        ▼  (network)
server.transferMoney(from, to, amount)
```

The mental model is **action-oriented**: you call a verb (`transferMoney`, `cancelOrder`, `streamPrices`). REST, by contrast, is **resource-oriented**: you manipulate nouns through uniform verbs (`POST /payments`, `DELETE /orders/42`).

### Why RPC exists

- **Many domains are naturally action-oriented.** "Refund this charge", "trigger fraud check", "reroute traffic" don't map cleanly to CRUD.
- **Developers think in functions.** RPC matches how programmers already model behavior.
- **Stubs hide the network.** Generated code reduces boilerplate; you call a method and get a typed result or a typed error.
- **Strong contracts via IDL.** Interface Description Languages (Protobuf, Thrift, Avro IDL) give you typed schemas, codegen, and forward/backward compatibility rules.

### RPC vs REST in one table

| Dimension | REST | RPC |
|---|---|---|
| Mental model | Resources + uniform verbs | Functions / actions |
| URL semantics | URL = resource identity | Endpoint = service.method |
| Verbs | `GET/POST/PUT/PATCH/DELETE` | One transport call per method |
| Discoverability | Hypermedia, OpenAPI | IDL (proto), reflection |
| Best fit | Public APIs, CRUD, browsers | Internal services, action-heavy domains |
| Caching | First-class (HTTP `GET`) | Weak (everything is a request) |
| Streaming | Limited (SSE, WebSocket) | First-class (gRPC) |

### Network reality vs local-method illusion

This is the most important fallacy a junior engineer can absorb from RPC: **a remote call is not a local call.** The "Eight Fallacies of Distributed Computing" apply:

- The network **is not** reliable.
- Latency **is not** zero.
- Bandwidth **is not** infinite.
- The network **is not** secure.
- Topology changes.
- There **is** more than one administrator.
- Transport cost **is not** zero.
- The network **is not** homogeneous.

A method that takes 100 µs locally takes 0.5–50 ms remotely, may fail, may partially succeed, may be retried. RPC's job is to make this **manageable**, not invisible. Senior engineers use RPC frameworks **knowing** the network is there — they don't pretend it isn't.

### Synchronous vs asynchronous RPC

| | Synchronous RPC | Asynchronous RPC |
|---|---|---|
| Caller waits | Yes — blocks until response | No — gets a future / callback / stream |
| Best for | Read paths, request/reply | Long-running calls, streaming, high concurrency |
| Failure model | Call returns/throws | Future resolves/fails |
| Threading | Often one thread per call | Event-loop or coroutine-based |

gRPC supports **both**. You typically write blocking client stubs for simple cases and async (`StreamObserver`, `ListenableFuture`, or coroutines/reactive) for streaming and high-concurrency paths.

### When RPC is useful

- Internal microservice-to-microservice traffic.
- Action-heavy domains (workflow engines, control planes, ledgers, ML inference).
- High-throughput, low-latency communication where REST overhead matters.
- Streaming workloads (price feeds, telemetry, model output).
- Polyglot stacks needing strong typing across languages.

### When RPC is a bad choice

- Public APIs consumed by many third parties expecting REST conventions.
- Browser-facing APIs (gRPC needs gRPC-Web; standard gRPC needs HTTP/2 trailers, which browsers don't expose).
- CDN/edge caching is critical (REST `GET` wins).
- A team that doesn't have the operational maturity to run a binary protocol with codegen.

---

## 2. gRPC Fundamentals

### What gRPC is

**gRPC** is a high-performance, open-source RPC framework originally built at Google, now a CNCF project. It standardizes:

- **Contract:** Protocol Buffers (proto3) `.proto` files.
- **Transport:** HTTP/2 (always — no HTTP/1.1).
- **Serialization:** Protocol Buffers binary (default), with pluggable codecs (JSON, FlatBuffers).
- **Codegen:** First-class clients and servers in C++, Java, Go, Python, Kotlin, Rust, Node, Ruby, C#, Swift, Dart, PHP, etc.
- **Status model:** Standardized status codes and error details.
- **Patterns:** Unary, server-streaming, client-streaming, bidirectional-streaming.

### Key features

- **HTTP/2 multiplexing.** Many concurrent calls on a single TCP connection — no head-of-line blocking at the application layer.
- **Binary serialization.** Compact and fast; typically 3–10× smaller than JSON, 5–20× faster to encode/decode.
- **Strong typing.** Schema-first; clients and servers cannot drift accidentally.
- **First-class streaming.** Four call patterns covering everything from request/reply to long-lived bidirectional channels.
- **Pluggable auth, compression, deadlines, metadata, interceptors.**
- **Built-in deadlines.** Every call carries a deadline that propagates through the call chain.

### Core components

```
┌──────────────┐                                ┌──────────────┐
│   Client     │ ── HTTP/2 over TCP/TLS ──▶     │   Server     │
│              │                                │              │
│ Generated    │  proto-defined service.method   │ Generated    │
│ stub         │  ◀── status + response/stream ─ │ skeleton     │
└──────────────┘                                └──────────────┘
       ▲                                                │
       │                                                │
       └──────── service.proto (shared contract) ───────┘
```

- **Service definition** — a `.proto` file declaring services, RPC methods, and message types.
- **Stub** (client) — generated code that turns method calls into HTTP/2 requests.
- **Skeleton / `Bind*ServiceImplBase`** (server) — generated abstract class your implementation extends.
- **Channel** — a managed gRPC connection (typically backed by one or more HTTP/2 connections to a target).
- **Server** — runs your service implementations; binds interceptors, auth, compression.

The **single source of truth** is the `.proto` file. Servers and clients are generated; if they go out of sync, the build fails.

---

## 3. Protocol Buffers Deep Dive

Protocol Buffers (Protobuf) is the IDL **and** wire format. You design with it; you debug with it; you live and die by its evolution rules.

### 3.1 A first `.proto` file

```proto
syntax = "proto3";

package payments.v1;
option java_package = "com.example.payments.v1";
option java_multiple_files = true;

import "google/protobuf/timestamp.proto";

service PaymentService {
  rpc Charge (ChargeRequest) returns (ChargeResponse);
  rpc StreamSettlements (StreamSettlementsRequest) returns (stream Settlement);
}

message ChargeRequest {
  string idempotency_key = 1;
  string customer_id = 2;
  Money amount = 3;
  string description = 4;
}

message ChargeResponse {
  string charge_id = 1;
  ChargeStatus status = 2;
  google.protobuf.Timestamp created_at = 3;
}

message Money {
  string currency = 1;     // ISO-4217
  int64 amount_minor = 2;  // e.g. cents
}

enum ChargeStatus {
  CHARGE_STATUS_UNSPECIFIED = 0;
  CHARGE_STATUS_AUTHORIZED  = 1;
  CHARGE_STATUS_CAPTURED    = 2;
  CHARGE_STATUS_DECLINED    = 3;
}
```

Conventions worth absorbing immediately:
- **Versioned package** (`payments.v1`) — the package is part of your contract; bumping it is an explicit major-version event.
- **`UNSPECIFIED = 0`** for every enum — there's no nullable enum in proto3; the zero value must mean "not set."
- **`java_multiple_files`** so each message becomes its own `.java` file — easier diffs.
- **Snake_case fields** in proto, camelCase getters/setters in generated Java.

### 3.2 Field numbering

Field numbers are part of the wire format. They identify fields in encoded messages.

- **1–15** use 1-byte tags. Reserve them for **frequently-used** fields.
- **16–2047** use 2-byte tags.
- **19000–19999** are reserved by Protobuf. Don't use them.
- **Maximum** is 536,870,911 — but you should never need more than ~50.

**Once you assign a field number and ship code, never reuse it.** Even if you delete the field. Reserve the old number:

```proto
message UserProfile {
  reserved 4, 6 to 8;
  reserved "old_email";
  string email = 1;
  // ... etc
}
```

### 3.3 Field types

| proto3 type | Java type | Notes |
|---|---|---|
| `int32`, `int64` | `int`, `long` | Variable-width encoding (varint). |
| `sint32`, `sint64` | same as above | Zig-zag encoded — better for negatives. |
| `uint32`, `uint64` | `int`, `long` | No native unsigned in Java; treat with care. |
| `fixed32`, `fixed64` | `int`, `long` | Fixed 4/8-byte encoding. |
| `bool` | `boolean` | |
| `float`, `double` | `float`, `double` | |
| `string` | `String` | UTF-8. |
| `bytes` | `ByteString` | Avoid huge payloads (see §14). |
| `repeated T` | `List<T>` | Maps to immutable lists. |
| `map<K, V>` | `Map<K, V>` | Equivalent to `repeated MapEntry`. |
| `oneof` | tag-discriminated union | At most one field set. |
| `enum` | Java enum | Always have `_UNSPECIFIED = 0`. |
| `message` | nested type | Recursion allowed. |

For numbers, prefer `int64` over `int32` unless you're certain of the range — schema evolution is harder if you outgrow `int32`.

### 3.4 Optional vs required (and why proto3 dropped `required`)

`required` existed in proto2 and is **gone** in proto3 — because once a field is required, removing it breaks every consumer forever. The proto3 compromise:

- **All scalars are implicitly "optional"** but their **default value is the zero value** (`0`, `""`, `false`).
- You can't tell "field not set" from "field set to default" for scalars, **unless** you mark the field `optional`:

```proto
message Patch {
  optional string nickname = 1;  // Java: hasNickname() / getNickname()
}
```

In Java, an `optional` scalar gets a `hasX()` accessor. Without `optional`, you only get `getX()` returning the default.

- **Message fields** are always nullable (`null` or set). Use them for fields that need a tri-state.
- **Repeated fields** can't distinguish "not set" from "empty list" — that's by design.

### 3.5 Schema evolution rules (the most important section)

Treat your `.proto` files as **a published API**. The wire format is forward- and backward-compatible *only if* you follow these rules.

#### Safe (additive) changes

- **Add a new field** with a new field number.
- **Add a new RPC method.**
- **Add a new enum value** — but old clients see the wire value; their generated enum has `UNRECOGNIZED`.
- **Rename a field** — wire format only sees numbers. Renames are source-incompatible (clients using the old name won't compile against new generated code) but wire-compatible.
- **Add an `optional` modifier** on an existing scalar — wire-compatible.

#### Unsafe (breaking) changes

- **Reuse a field number** for a different field.
- **Change a field's wire type** (`int32` → `string`, `int32` → `int64` is **mostly** safe but signed/unsigned can sneak issues).
- **Change `repeated` to single or vice versa.**
- **Remove a field without reserving its number.**
- **Rename or change semantics of an enum value.**
- **Change a `oneof` membership.**

#### Standard practice

- Use `reserved` to mark deleted field numbers and names.
- Bump the package version (`v1` → `v2`) for major breaking changes.
- Run buf or `protolock` in CI to detect breaking changes against the previous schema.

### 3.6 Best practices

- **Always include an `idempotency_key`** in mutating RPCs.
- **Always have `_UNSPECIFIED = 0`** in enums.
- **Use `Timestamp` and `Duration`** from `google/protobuf/timestamp.proto` and `duration.proto` — never invent your own time format.
- **Use `Money`-style `int64 amount_minor + currency`** for currency. Never use `double`.
- **Don't put PII in proto comments**; comments end up in generated docs and may leak.
- **Don't put a `bytes` field of unbounded size** in a unary RPC — stream it.
- **Keep messages flat**ish; deep nesting compresses fine but is harder to evolve.
- **One `service` per logical bounded context**, not one giant `KitchenSinkService`.

---

## 4. gRPC Communication Models

gRPC offers four call patterns. The choice shapes everything: client semantics, server-side concurrency, error handling, deadline propagation.

### 4.1 Unary RPC

**One request → one response.** Like a typed REST POST.

```proto
rpc Charge (ChargeRequest) returns (ChargeResponse);
```

- The default and most common pattern.
- Backed by a single HTTP/2 stream that opens, sends the request, receives the response, and closes.
- All gRPC's per-call features (deadline, auth metadata, retry, status) work most cleanly here.

**Use when:** request/reply semantics. CRUD, queries, commands.

**Pitfalls:**
- Sending a multi-MB payload in one unary call — use streaming or pagination.
- Forgetting deadlines — see §10.

### 4.2 Server streaming

**One request → stream of responses.**

```proto
rpc StreamSettlements (StreamSettlementsRequest) returns (stream Settlement);
```

- Server keeps a single HTTP/2 stream open; pushes messages until it half-closes.
- Client iterates: blocking iterator (Java), `Flow<T>` (Kotlin), `Mono<Flux<T>>`-style (Reactor), or `StreamObserver` callbacks.

**Use when:**
- Pagination over a long result set.
- Server push (live tail of logs, prices, telemetry).
- Long-running progressive responses (ML inference token streaming).

**Pitfalls:**
- Forgetting backpressure → server outpaces client → memory bloat.
- Holding open thousands of long-lived streams → resource exhaustion.
- No idempotency guarantees on reconnect — clients must handle resume.

### 4.3 Client streaming

**Stream of requests → one response.**

```proto
rpc UploadEvents (stream Event) returns (UploadAck);
```

- Client keeps the stream open and pushes messages.
- Server reads them, optionally aggregates, returns a single response when the client half-closes.

**Use when:**
- Bulk uploads of records.
- Streaming bytes for large file uploads (with chunking).
- Aggregation operations.

**Pitfalls:**
- Server can't validate the request as a whole until you finish — design for incremental validation.
- Errors mid-stream should fail fast (server cancels the call).

### 4.4 Bidirectional streaming

**Stream of requests ↔ stream of responses.**

```proto
rpc Chat (stream ChatMessage) returns (stream ChatMessage);
```

- Both sides send and receive concurrently on the same stream.
- Most complex pattern; matches WebSocket semantics for typed protocols.

**Use when:**
- Chat / realtime messaging.
- Trading systems (subscribe + send orders + receive fills on one channel).
- Bidirectional control planes (request/response pipelining without ordering constraints).

**Pitfalls:**
- Concurrency between read and write halves — you must handle them as **independent** event loops.
- Half-close semantics — one side can close its send half while still receiving.
- Hardest to debug: the wire trace is interleaved.

### 4.5 Choosing the model

| Scenario | Model |
|---|---|
| Get user by ID | Unary |
| Tail logs | Server streaming |
| Bulk-import 1M rows | Client streaming |
| Live trading session | Bidirectional |
| Large file upload | Client streaming (chunked) |
| Subscribe to events | Server streaming |

When in doubt, start with **unary**. Add streaming when there's a real reason — they introduce concurrency, lifecycle, and operational complexity.

---

## 5. gRPC Architecture and Internals

This is the section that distinguishes gRPC users from gRPC engineers. Understanding the HTTP/2 mechanics under the hood explains every operational quirk you'll hit.

### 5.1 The transport: HTTP/2

gRPC **requires HTTP/2** (or HTTP/3 in some implementations). HTTP/2 was designed for exactly the workload gRPC has:

- **Single TCP connection** carries many concurrent streams.
- **Streams are independent**, identified by stream IDs.
- **Frames** (HEADERS, DATA, RST_STREAM, SETTINGS, PING, GOAWAY, WINDOW_UPDATE) are the wire unit.
- **HPACK** header compression — repeated headers (auth tokens, content-type) are sent as small references.
- **Trailers** — HTTP/2 supports headers at the end of a response. gRPC uses trailers to carry the final status (`grpc-status`, `grpc-message`).

### 5.2 Multiplexing

A single HTTP/2 connection can carry hundreds or thousands of concurrent streams:

```
TCP connection
├── stream 1: unary call to GetUser
├── stream 3: server-streaming Subscribe
├── stream 5: bidi Chat
└── stream 7: unary to Charge (in flight at the same time)
```

Stream IDs are **odd for client-initiated** and **even for server-initiated** (gRPC only uses client-initiated).

This is why gRPC can saturate a single TCP connection in ways HTTP/1.1 cannot: with HTTP/1.1, browsers cap at ~6 connections per host with one in-flight request each; gRPC over HTTP/2 needs only one TCP socket for thousands of in-flight calls.

### 5.3 Streams and frames

A unary call's wire trace looks like:

```
Client → Server  HEADERS    (stream=1, :method=POST :path=/payments.v1.PaymentService/Charge,
                              content-type=application/grpc, te=trailers,
                              authority=..., authorization=Bearer ...,
                              grpc-timeout=5000m, grpc-encoding=gzip)
Client → Server  DATA       (stream=1, length-prefixed proto bytes)
Client → Server  END_STREAM (or DATA with END_STREAM flag)

Server → Client  HEADERS    (stream=1, :status=200 grpc-encoding=gzip)
Server → Client  DATA       (stream=1, length-prefixed proto bytes)
Server → Client  HEADERS    (stream=1, END_STREAM, grpc-status=0 grpc-message=OK)  ← trailers
```

Notice: HTTP **`:status=200`** is almost always 200 for gRPC. The real outcome is in the **trailers** as `grpc-status`. This is why gRPC needs HTTP/2 trailers — and why browsers (which don't expose trailers via fetch) can't speak native gRPC.

### 5.4 Flow control

HTTP/2 has **stream-level** and **connection-level** flow control via `WINDOW_UPDATE` frames:

- Each side advertises a window size — how many bytes it's willing to accept.
- Sender decrements the window with every DATA frame.
- When the window hits zero, sender stops sending until receiver issues `WINDOW_UPDATE`.

This is your **backpressure** mechanism. In Java, `StreamObserver.isReady()` and `setOnReadyHandler` expose it.

If your server publishes faster than the client reads, the client's window fills, the server's send buffer fills, then the server's `onNext()` calls block (in async mode they're queued in memory — leak risk). Always honor `isReady()` for any non-trivial streaming work.

### 5.5 Connection lifecycle

#### Client side

1. **Channel** created via `ManagedChannelBuilder` — a pool/manager of underlying HTTP/2 connections.
2. **Name resolver** maps the target (`dns:///service:9090`, `xds:///service`, etc.) to a list of addresses.
3. **Load balancer** picks one (round-robin, pick-first, ring-hash, custom).
4. **Subchannel** is the actual HTTP/2 connection.
5. **PING** frames keep it alive (`keepAlive` config).
6. On idle, **GOAWAY** from server → channel disconnects gracefully; new calls trigger reconnect.

#### Server side

- Listens on a port, accepts TCP+TLS, upgrades to HTTP/2.
- Each accepted connection has a `MAX_CONCURRENT_STREAMS` setting (default 100 in many implementations; increase for high-fan-in services).
- On shutdown: emits `GOAWAY` so in-flight calls finish; refuses new streams.

### 5.6 Step-by-step: a unary call

1. Client app calls `stub.charge(req)`.
2. Stub asks the channel for a free stream on a healthy subchannel; load balancer picks one.
3. Stub serializes `req` with Protobuf → bytes.
4. Channel sends HEADERS (with method path, deadline, auth, encoding) + DATA frames.
5. Server's HTTP/2 layer routes the stream to the gRPC stack based on `:path`.
6. gRPC runs interceptors (auth, logging, tracing) → invokes the service implementation.
7. Service produces a response object → serialized → DATA frames sent.
8. Server emits trailers with `grpc-status: 0` (or non-zero on error).
9. Client receives trailers, sees `grpc-status`, completes the future / unblocks the call.
10. Either side can `RST_STREAM` to cancel; deadline expiration emits `DEADLINE_EXCEEDED` on the client side and (eventually) cancels the server work via the deadline-aware Context.

### 5.7 Deadlines and cancellation propagation

Deadlines are first-class. The client sets one (`stub.withDeadlineAfter(...)`) and the framework propagates it as the `grpc-timeout` HTTP/2 header. The server's `Context` carries the deadline; when it expires, the framework cancels in-flight DB queries, downstream RPC calls, and any other Context-aware operation.

**Always set deadlines on every RPC.** Without one, a hung server pegs your client threads forever.

When a service A calls service B which calls service C, and A's deadline is 200ms, B should propagate the *remaining* deadline to C. The Java stub does this automatically when you use `Context`-aware code paths.

### 5.8 Cancellation

Either side can cancel:

- Client: closes its end → `RST_STREAM` to server → server's `Context.isCancelled() == true`.
- Server: returns an error early → trailers go out; the rest of the stream is RST'd.

A canceled context means **stop work immediately** — it's how you avoid wasting CPU on a request the client no longer wants.

---

## 6. gRPC vs REST vs JSON-RPC vs GraphQL

### 6.1 Comparison matrix

| Dimension | gRPC | REST (JSON) | JSON-RPC | GraphQL |
|---|---|---|---|---|
| Style | Action-oriented RPC | Resource-oriented | Action-oriented RPC | Query language |
| Wire format | Protobuf binary | JSON | JSON | JSON (+ binary opt) |
| Transport | HTTP/2 only | HTTP/1.1, HTTP/2 | Anything (HTTP/WS/TCP) | HTTP, WS |
| Schema | `.proto` (mandatory) | OpenAPI (optional) | None | SDL (mandatory) |
| Codegen | Excellent, polyglot | OpenAPI generators | Sparse | Excellent |
| Streaming | First-class (4 modes) | SSE only | Via WebSocket | Subscriptions over WS |
| Browser support | gRPC-Web only | Native | Native | Native |
| Caching (CDN) | Weak | Strong (`GET`) | Weak | Weak |
| Performance | **Highest** | Medium | Medium | Medium |
| Payload size | Smallest | Medium | Medium | Medium |
| Tooling maturity | Strong | **Strongest** | Weak | Strong |
| Discoverability | Reflection / IDL | OpenAPI | None | Introspection |
| Backward compat | Good (Protobuf rules) | Manual | Manual | Strong (deprecation) |
| Operational complexity | Higher | Lowest | Low | Higher |

### 6.2 When to use gRPC

- **Internal microservice traffic** with control over both ends.
- **Polyglot stacks** (Go ↔ Java ↔ Python) where typed contracts and codegen are essential.
- **High throughput / low latency** workloads where JSON parsing dominates.
- **Streaming use cases** beyond what SSE can do.
- **Strong contracts and CI-enforced compatibility** are organizational requirements.

### 6.3 When NOT to use gRPC

- **Public APIs** consumed by unknown third parties — REST is the lingua franca.
- **Browser-direct clients** without a proxy — native gRPC isn't browser-friendly; gRPC-Web requires Envoy/NGINX translation.
- **Caching is critical** at the CDN/edge layer — REST `GET` is irreplaceable here.
- **Simple internal tools** where REST + Spring Boot is faster to ship and your team already knows it.
- **Highly dynamic / ad-hoc queries** — GraphQL fits better; gRPC is rigid by design.

---

## 7. Java + Spring Boot Integration

This section uses the de facto community starter `net.devh:grpc-spring-boot-starter`, which integrates gRPC with Spring Boot's lifecycle, configuration, security, and Actuator. (The official `grpc-java` library works fine standalone too — the starter just wires it into Spring.)

### 7.1 Project setup

`pom.xml`:

```xml
<properties>
  <grpc.version>1.62.2</grpc.version>
  <protobuf.version>3.25.3</protobuf.version>
  <netdevh.version>3.0.0.RELEASE</netdevh.version>
</properties>

<dependencies>
  <dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>${netdevh.version}</version>
  </dependency>
  <dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-client-spring-boot-starter</artifactId>
    <version>${netdevh.version}</version>
  </dependency>
  <dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-netty-shaded</artifactId>
    <version>${grpc.version}</version>
  </dependency>
  <dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>${grpc.version}</version>
  </dependency>
  <dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>${grpc.version}</version>
  </dependency>
  <dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-services</artifactId>
    <version>${grpc.version}</version>
  </dependency>
</dependencies>

<build>
  <extensions>
    <extension>
      <groupId>kr.motd.maven</groupId>
      <artifactId>os-maven-plugin</artifactId>
      <version>1.7.1</version>
    </extension>
  </extensions>
  <plugins>
    <plugin>
      <groupId>org.xolstice.maven.plugins</groupId>
      <artifactId>protobuf-maven-plugin</artifactId>
      <version>0.6.1</version>
      <configuration>
        <protocArtifact>com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}</protocArtifact>
        <pluginId>grpc-java</pluginId>
        <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
      </configuration>
      <executions>
        <execution><goals><goal>compile</goal><goal>compile-custom</goal></goals></execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

`src/main/proto/payments.proto`:

```proto
syntax = "proto3";
package payments.v1;
option java_package = "com.example.payments.v1";
option java_multiple_files = true;

service PaymentService {
  rpc Charge (ChargeRequest) returns (ChargeResponse);
  rpc StreamSettlements (StreamSettlementsRequest) returns (stream Settlement);
}

message ChargeRequest {
  string idempotency_key = 1;
  string customer_id = 2;
  Money amount = 3;
}

message ChargeResponse {
  string charge_id = 1;
  ChargeStatus status = 2;
}

message Money { string currency = 1; int64 amount_minor = 2; }

enum ChargeStatus {
  CHARGE_STATUS_UNSPECIFIED = 0;
  CHARGE_STATUS_AUTHORIZED  = 1;
  CHARGE_STATUS_DECLINED    = 2;
}

message StreamSettlementsRequest { string merchant_id = 1; }
message Settlement { string id = 1; int64 amount_minor = 2; }
```

`application.yml`:

```yaml
grpc:
  server:
    port: 9090
    security:
      enabled: false   # dev only; see §9 for TLS
    max-inbound-message-size: 4MB
    keep-alive-time: 30s
    keep-alive-timeout: 5s
    permit-keep-alive-without-calls: true
  client:
    payments:
      address: "static://payments-service:9090"
      negotiation-type: plaintext   # dev only
      enable-keep-alive: true
      keep-alive-without-calls: true
```

### 7.2 Server: a unary service

```java
package com.example.payments;

import com.example.payments.v1.*;
import io.grpc.Status;
import io.grpc.stub.StreamObserver;
import net.devh.boot.grpc.server.service.GrpcService;
import org.springframework.beans.factory.annotation.Autowired;

@GrpcService
public class PaymentServiceImpl extends PaymentServiceGrpc.PaymentServiceImplBase {

    private final PaymentProcessor processor;
    private final IdempotencyStore idempotency;

    public PaymentServiceImpl(PaymentProcessor processor, IdempotencyStore idempotency) {
        this.processor = processor;
        this.idempotency = idempotency;
    }

    @Override
    public void charge(ChargeRequest req, StreamObserver<ChargeResponse> response) {
        if (req.getIdempotencyKey().isBlank()) {
            response.onError(Status.INVALID_ARGUMENT
                .withDescription("idempotency_key is required")
                .asRuntimeException());
            return;
        }

        ChargeResponse cached = idempotency.lookup(req.getIdempotencyKey());
        if (cached != null) {
            response.onNext(cached);
            response.onCompleted();
            return;
        }

        try {
            ChargeResponse out = processor.charge(req);
            idempotency.store(req.getIdempotencyKey(), out);
            response.onNext(out);
            response.onCompleted();
        } catch (CardDeclinedException e) {
            response.onError(Status.FAILED_PRECONDITION
                .withDescription("declined: " + e.reasonCode())
                .asRuntimeException());
        } catch (Exception e) {
            response.onError(Status.INTERNAL.withCause(e).asRuntimeException());
        }
    }
}
```

What's happening:
- `@GrpcService` registers the service with the embedded gRPC server.
- The signature is **callback-based** (`StreamObserver`) — gRPC's idiomatic async style.
- For a unary call: exactly one `onNext` then `onCompleted`. Or `onError`. Never both.
- Errors are **status codes**, not Java exceptions on the wire — see §8.

### 7.3 Server: a server-streaming method

```java
@Override
public void streamSettlements(StreamSettlementsRequest req,
                              StreamObserver<Settlement> response) {
    var serverObserver = (ServerCallStreamObserver<Settlement>) response;
    serverObserver.setOnCancelHandler(() -> log.info("client canceled"));

    Iterator<Settlement> source = repo.iterateSettlements(req.getMerchantId());

    Runnable drain = () -> {
        while (serverObserver.isReady() && source.hasNext()) {
            response.onNext(source.next());
        }
        if (!source.hasNext()) {
            response.onCompleted();
        }
    };

    serverObserver.setOnReadyHandler(drain);
    drain.run();
}
```

Key points:
- Cast to `ServerCallStreamObserver` to access `isReady` and `setOnReadyHandler` — that's how you honor backpressure.
- Without those handlers, a slow client → unbounded memory growth on the server.
- `setOnCancelHandler` lets you stop work when the client goes away.

### 7.4 Server: a client-streaming method

```proto
rpc UploadEvents (stream Event) returns (UploadAck);
```

```java
@Override
public StreamObserver<Event> uploadEvents(StreamObserver<UploadAck> response) {
    final long[] count = {0};
    return new StreamObserver<>() {
        @Override public void onNext(Event e)        { count[0]++; sink.append(e); }
        @Override public void onError(Throwable t)   { log.warn("upload aborted", t); }
        @Override public void onCompleted()          {
            response.onNext(UploadAck.newBuilder().setCount(count[0]).build());
            response.onCompleted();
        }
    };
}
```

The framework calls *your* `StreamObserver` for each incoming message. Reply once with `response.onNext` + `onCompleted`.

### 7.5 Server: a bidirectional method

```proto
rpc Chat (stream ChatMessage) returns (stream ChatMessage);
```

```java
@Override
public StreamObserver<ChatMessage> chat(StreamObserver<ChatMessage> outbound) {
    var serverObs = (ServerCallStreamObserver<ChatMessage>) outbound;
    return new StreamObserver<>() {
        @Override public void onNext(ChatMessage in) {
            if (serverObs.isReady()) {
                outbound.onNext(transform(in));
            } else {
                buffer.add(transform(in));
            }
        }
        @Override public void onError(Throwable t) { /* log + cleanup */ }
        @Override public void onCompleted()        { outbound.onCompleted(); }
    };
}
```

Bidi requires you to think about **two** independent streams: inbound and outbound. They share a Context, but their lifecycles are separate.

### 7.6 Client: blocking and async stubs

```java
@Component
public class PaymentClient {

    @GrpcClient("payments")
    private PaymentServiceGrpc.PaymentServiceBlockingStub blocking;

    @GrpcClient("payments")
    private PaymentServiceGrpc.PaymentServiceStub async;

    public ChargeResponse charge(ChargeRequest req) {
        return blocking
            .withDeadlineAfter(2, TimeUnit.SECONDS)
            .charge(req);
    }

    public void streamSettlements(String merchantId, Consumer<Settlement> onItem) {
        var req = StreamSettlementsRequest.newBuilder().setMerchantId(merchantId).build();
        async.withDeadlineAfter(30, TimeUnit.SECONDS)
             .streamSettlements(req, new StreamObserver<>() {
                 @Override public void onNext(Settlement s)  { onItem.accept(s); }
                 @Override public void onError(Throwable t)  { /* log */ }
                 @Override public void onCompleted()         { /* done */ }
             });
    }
}
```

- **Blocking stub** for unary calls and simple server-streaming (returns an `Iterator`).
- **Async stub** for streaming, multiple in-flight calls, or non-blocking.
- **Always** add `withDeadlineAfter(...)`. Forgetting deadlines is the #1 gRPC operational bug.

### 7.7 Interceptors

Interceptors are gRPC's middleware. Use them for cross-cutting: logging, tracing, auth, metrics, retry.

```java
@GrpcGlobalServerInterceptor
public class LoggingServerInterceptor implements ServerInterceptor {
    private static final Logger log = LoggerFactory.getLogger(LoggingServerInterceptor.class);

    @Override
    public <Req, Res> ServerCall.Listener<Req> interceptCall(
            ServerCall<Req, Res> call, Metadata headers, ServerCallHandler<Req, Res> next) {
        long start = System.nanoTime();
        String method = call.getMethodDescriptor().getFullMethodName();
        ServerCall<Req, Res> wrapped = new SimpleForwardingServerCall<>(call) {
            @Override public void close(Status status, Metadata trailers) {
                long ms = (System.nanoTime() - start) / 1_000_000;
                log.info("rpc method={} status={} durMs={}", method, status.getCode(), ms);
                super.close(status, trailers);
            }
        };
        return next.startCall(wrapped, headers);
    }
}
```

Client-side (e.g., attach JWT):

```java
@GrpcGlobalClientInterceptor
public class AuthClientInterceptor implements ClientInterceptor {
    private static final Metadata.Key<String> AUTH =
        Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
    private final TokenSupplier tokens;
    public AuthClientInterceptor(TokenSupplier t) { this.tokens = t; }
    @Override
    public <Req, Res> ClientCall<Req, Res> interceptCall(
            MethodDescriptor<Req, Res> m, CallOptions opts, Channel next) {
        return new SimpleForwardingClientCall<>(next.newCall(m, opts)) {
            @Override public void start(Listener<Res> l, Metadata headers) {
                headers.put(AUTH, "Bearer " + tokens.current());
                super.start(l, headers);
            }
        };
    }
}
```

Order of interceptors matters: trace before auth before metrics, etc. Keep it explicit.

---

## 8. Error Handling and Status Codes

### 8.1 Status codes (the canonical list)

gRPC defines a fixed set of status codes — your error contract on the wire.

| Code | Numeric | When to use |
|---|---|---|
| `OK` | 0 | Success. |
| `CANCELLED` | 1 | Client canceled or context canceled. |
| `UNKNOWN` | 2 | Default for unclassified errors. Avoid. |
| `INVALID_ARGUMENT` | 3 | Bad input regardless of system state. |
| `DEADLINE_EXCEEDED` | 4 | Operation timed out. |
| `NOT_FOUND` | 5 | Entity not found. |
| `ALREADY_EXISTS` | 6 | Conflict on create. |
| `PERMISSION_DENIED` | 7 | Authenticated but not authorized. |
| `RESOURCE_EXHAUSTED` | 8 | Quota / rate-limit / memory pressure. |
| `FAILED_PRECONDITION` | 9 | State doesn't allow this operation (e.g., locked, declined). |
| `ABORTED` | 10 | Concurrency conflict, retry possible. |
| `OUT_OF_RANGE` | 11 | Argument out of range (offsets, sizes). |
| `UNIMPLEMENTED` | 12 | Method not implemented or disabled. |
| `INTERNAL` | 13 | Server-side bug. |
| `UNAVAILABLE` | 14 | Service unavailable (transient). Retryable. |
| `DATA_LOSS` | 15 | Unrecoverable data corruption. |
| `UNAUTHENTICATED` | 16 | No or invalid credentials. |

Two important distinctions:

- **`INVALID_ARGUMENT`** — bad input regardless of state. Don't retry.
- **`FAILED_PRECONDITION`** — input was syntactically fine, but state forbids the op. Don't retry without changing state.
- **`ABORTED`** — concurrency / optimistic-lock conflict. Retry usually works.
- **`UNAVAILABLE`** — transient infra. Retry with backoff.
- **`DEADLINE_EXCEEDED`** — caller's clock; may or may not be retryable depending on whether the work is idempotent.

### 8.2 Returning errors

```java
response.onError(
    Status.INVALID_ARGUMENT
        .withDescription("idempotency_key required")
        .asRuntimeException()
);
```

For richer detail, use **error details** (`com.google.rpc.Status` with `Any`-packed messages) via `StatusProto`:

```java
import com.google.rpc.BadRequest;
import com.google.rpc.Code;
import io.grpc.protobuf.StatusProto;

var fieldViolation = BadRequest.FieldViolation.newBuilder()
    .setField("amount")
    .setDescription("must be > 0")
    .build();
var details = BadRequest.newBuilder().addFieldViolations(fieldViolation).build();
var status = com.google.rpc.Status.newBuilder()
    .setCode(Code.INVALID_ARGUMENT_VALUE)
    .setMessage("Invalid charge amount")
    .addDetails(com.google.protobuf.Any.pack(details))
    .build();
response.onError(StatusProto.toStatusRuntimeException(status));
```

This gives clients **structured, typed** error details — the gRPC equivalent of the JSON error envelope.

### 8.3 Centralized exception mapping

Don't sprinkle `Status` translation in every service. Use a server interceptor:

```java
@GrpcGlobalServerInterceptor
public class ExceptionMapping implements ServerInterceptor {
    @Override public <Req, Res> ServerCall.Listener<Req> interceptCall(
            ServerCall<Req, Res> call, Metadata headers, ServerCallHandler<Req, Res> next) {
        var listener = next.startCall(call, headers);
        return new SimpleForwardingServerCallListener<>(listener) {
            @Override public void onHalfClose() {
                try { super.onHalfClose(); }
                catch (NotFoundException e)        { close(call, Status.NOT_FOUND, e); }
                catch (ConflictException e)        { close(call, Status.ALREADY_EXISTS, e); }
                catch (ValidationException e)      { close(call, Status.INVALID_ARGUMENT, e); }
                catch (UnauthorizedException e)    { close(call, Status.PERMISSION_DENIED, e); }
                catch (RuntimeException e)         { close(call, Status.INTERNAL, e); }
            }
        };
    }
    private static void close(ServerCall<?,?> call, Status base, Throwable t) {
        call.close(base.withDescription(t.getMessage()), new Metadata());
    }
}
```

Now your services throw plain Java exceptions; the interceptor translates them. Less boilerplate, consistent statuses.

### 8.4 Client-side error handling

```java
try {
    ChargeResponse r = blocking.withDeadlineAfter(2, SECONDS).charge(req);
} catch (StatusRuntimeException e) {
    Status.Code code = e.getStatus().getCode();
    switch (code) {
        case UNAVAILABLE, DEADLINE_EXCEEDED -> retryWithBackoff(req);
        case INVALID_ARGUMENT, FAILED_PRECONDITION, NOT_FOUND -> surfaceToCaller(e);
        case UNAUTHENTICATED -> refreshTokenAndRetryOnce(req);
        default -> log.error("unexpected gRPC status {}", code, e);
    }
}
```

### 8.5 Retry strategies

gRPC Java supports declarative retries via service config:

```json
{
  "methodConfig": [{
    "name": [{ "service": "payments.v1.PaymentService", "method": "Charge" }],
    "retryPolicy": {
      "maxAttempts": 4,
      "initialBackoff": "0.1s",
      "maxBackoff": "1s",
      "backoffMultiplier": 2.0,
      "retryableStatusCodes": ["UNAVAILABLE"]
    }
  }]
}
```

Apply with `ManagedChannelBuilder.defaultServiceConfig(...)` and `enableRetry()`.

**Rules:**
- Retry only **idempotent** methods, or methods with idempotency keys.
- Retry only `UNAVAILABLE` and `DEADLINE_EXCEEDED` (sometimes `ABORTED`).
- Cap attempts (3–5) and total budget (e.g., the call's own deadline).
- Exponential backoff with **jitter** to avoid thundering herds.

Don't retry `INVALID_ARGUMENT`, `NOT_FOUND`, `FAILED_PRECONDITION`, `PERMISSION_DENIED`, or `INTERNAL` (without investigation).

---

## 9. Security

### 9.1 TLS

gRPC defaults to plaintext in dev, **never** in production. Enable TLS at the server:

```yaml
grpc:
  server:
    security:
      enabled: true
      certificate-chain: file:/etc/tls/server.crt
      private-key: file:/etc/tls/server.key
      trust-cert-collection: file:/etc/tls/ca.crt
      client-auth: require   # for mTLS; otherwise NONE / OPTIONAL
```

Client:

```yaml
grpc:
  client:
    payments:
      address: "static://payments-service:9090"
      negotiation-type: tls
      security:
        client-auth-enabled: true
        certificate-chain: file:/etc/tls/client.crt
        private-key: file:/etc/tls/client.key
        trust-cert-collection: file:/etc/tls/ca.crt
```

ALPN selects HTTP/2; gRPC verifies the negotiation was actually for `h2`. Older JDKs needed Jetty ALPN agents — JDK 11+ supports ALPN natively.

### 9.2 mTLS for service-to-service

In a service mesh (Istio, Linkerd) or a zero-trust internal network, mTLS is the right default. Each service has a workload identity (cert from a CA you control), and authorization is based on the cert's identity.

If you run in Kubernetes with Istio/Linkerd, **let the mesh do mTLS** — it's transparent to your code. You write plaintext gRPC; the sidecar adds TLS.

### 9.3 Authentication beyond mTLS

For end-user identity (not service identity):

- **JWT** in the `authorization` metadata: `Authorization: Bearer <jwt>`.
- Validate signature, issuer, audience, expiry on the server (server interceptor).
- Pass the verified principal via gRPC `Context` so handlers can read it.

```java
@GrpcGlobalServerInterceptor
public class JwtAuthInterceptor implements ServerInterceptor {
    public static final Context.Key<Principal> PRINCIPAL = Context.key("principal");
    private final JwtVerifier verifier;
    public JwtAuthInterceptor(JwtVerifier v) { this.verifier = v; }

    @Override
    public <Req, Res> ServerCall.Listener<Req> interceptCall(
            ServerCall<Req, Res> call, Metadata headers, ServerCallHandler<Req, Res> next) {
        var auth = headers.get(Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER));
        if (auth == null || !auth.startsWith("Bearer ")) {
            call.close(Status.UNAUTHENTICATED.withDescription("missing token"), new Metadata());
            return new ServerCall.Listener<>() {};
        }
        try {
            Principal principal = verifier.verify(auth.substring(7));
            Context ctx = Context.current().withValue(PRINCIPAL, principal);
            return Contexts.interceptCall(ctx, call, headers, next);
        } catch (Exception e) {
            call.close(Status.UNAUTHENTICATED.withDescription("invalid token"), new Metadata());
            return new ServerCall.Listener<>() {};
        }
    }
}
```

### 9.4 Authorization

Two layers:

- **Method-level.** Map service/method names to required scopes/roles. Reject in an interceptor.
- **Object-level.** Inside the handler, verify the principal can act on this entity (tenant ID match, ownership).

Using Spring Security with gRPC is possible (the starter integrates it) but a focused interceptor is often simpler and more transparent.

### 9.5 Other security essentials

- **Don't expose reflection** in production unless you gate it behind admin auth.
- **Limit message sizes** — `maxInboundMessageSize` (default 4 MB). A 50 MB message is an attack surface.
- **Rate-limit per identity**, not per IP, when behind proxies (use the verified JWT subject).
- **Audit-log mutating calls** with the principal, idempotency key, status, and timing.
- **Don't echo secrets in error messages or `data` fields.**

---

## 10. Performance and Optimization

### 10.1 Connection reuse

A `ManagedChannel` is **not** disposable — it's a long-lived multiplexed connection. Build one channel per target service and reuse it for the lifetime of the process.

Anti-pattern: building a new `ManagedChannel` per request. You'll exhaust file descriptors and ephemeral ports.

### 10.2 Threading model

gRPC Java with `grpc-netty` uses Netty's event loops:

- A small **boss group** accepts connections.
- A larger **worker group** drives I/O.
- Your **service implementations** run on a separate executor (`ServerBuilder.executor`). Default is a cached thread pool.

**Never block** in service code without thinking. Long blocking calls (DB queries, external HTTP) on the default executor are usually fine because it grows. But if you're using a small bounded executor, blocking calls will starve it.

For high-concurrency, IO-bound services, consider:

- **Reactive (Reactor) gRPC** — `salesforce/reactor-grpc` — async stubs returning `Mono`/`Flux`.
- **Coroutines (Kotlin)** — `grpc-kotlin` — suspend functions over gRPC.
- **Project Loom virtual threads** (Java 21) — set the server executor to `Executors.newVirtualThreadPerTaskExecutor()` for cheap blocking I/O.

### 10.3 Serialization efficiency

- Protobuf is already 5–20× faster to serialize than JSON. Don't worry about it until profiling says you should.
- `repeated` of small messages is fine; deeply nested messages are fine.
- `bytes` fields with multi-MB blobs **are not fine** in unary RPCs. Stream them.
- Reuse builders if you create many messages in a tight loop (Protobuf builders are mutable, deliberately).

### 10.4 Compression

```java
// Server-side
serverBuilder.intercept(new GzipCompressorInterceptor());
// or per-call: ServerCallStreamObserver.setCompression("gzip")

// Client-side
stub.withCompression("gzip").charge(req);
```

When to enable:
- Payloads > a few KB with redundancy (text-heavy fields).
- Cross-region or low-bandwidth links.

When not to:
- Already-compressed payloads (images, video, encrypted).
- Tiny messages (< 500 bytes) — overhead exceeds savings.
- Internal-LAN, high-bandwidth, low-CPU-budget services.

Default to **off** and turn on after measuring.

### 10.5 Channel and call configuration

| Knob | Effect |
|---|---|
| `keepAliveTime` / `keepAliveTimeout` | PING frames to detect dead connections. |
| `maxInboundMessageSize` | Cap incoming message bytes. |
| `maxConcurrentCallsPerConnection` (server) | Prevent one client from monopolizing a connection. |
| `MAX_CONCURRENT_STREAMS` (HTTP/2) | Per-connection concurrent streams; raise for high fan-in. |
| `flowControlWindow` | HTTP/2 stream window — affects throughput on high-BDP links. |
| `idleTimeout` | When idle channels disconnect; reconnect cost on next call. |
| Deadlines | Always set; cap server work. |

### 10.6 Load balancing

#### Client-side LB (default for gRPC)

gRPC's preferred model is **client-side load balancing**: the client resolves the target to multiple endpoints, picks one per call.

```java
ManagedChannelBuilder.forTarget("dns:///payments-service:9090")
    .defaultLoadBalancingPolicy("round_robin")
    .build();
```

In Kubernetes, a normal `ClusterIP` Service load-balances at L4 — but with gRPC's single long-lived HTTP/2 connection, you'd pin to one pod. Workarounds:

- **Headless service** + DNS round-robin + client-side LB (the gRPC-native way).
- **Service mesh** (Istio/Linkerd) handles L7 LB transparently.
- **xDS** with gRPC-xDS for advanced routing.

This is the #1 surprise when running gRPC on Kubernetes. Plan for it.

#### Server-side LB

A dedicated gRPC-aware LB (Envoy, NGINX, Linkerd, ALB with HTTP/2) terminates the connection and round-robins **per stream**, not per connection. Works fine with naive clients.

---

## 11. Observability and Debugging

### 11.1 Logging

Log per-call: method, status code, duration, and trace IDs. Don't log full payloads (PII risk + size).

Use the interceptor in §7.7 plus structured logging (JSON logs) and you have queryable per-call telemetry.

### 11.2 Metrics

`grpc-spring-boot-starter` integrates with Micrometer. Out of the box, you get:

- `grpc.server.calls` (counter) — by method and status.
- `grpc.server.processing.duration` (timer) — by method and status.
- Client equivalents.

Useful additional dashboards:

- p50/p95/p99 latency per method.
- Status-code distribution per method.
- Active streams.
- Connection count (per channel).

### 11.3 Tracing

OpenTelemetry has first-class gRPC instrumentation. Spans are created per call with `rpc.system=grpc`, `rpc.service`, `rpc.method`, `rpc.grpc.status_code`. The auto-instrumentation handles trace context propagation through gRPC metadata.

In a service-mesh setup, sidecars also emit spans; deduplicate carefully.

### 11.4 Debugging

#### grpcurl

The `curl` of gRPC:

```bash
grpcurl -plaintext payments-service:9090 list
grpcurl -plaintext payments-service:9090 describe payments.v1.PaymentService
grpcurl -plaintext -d '{"customerId":"c1","amount":{"currency":"USD","amountMinor":1000}}' \
        payments-service:9090 payments.v1.PaymentService/Charge
```

Requires **server reflection** (`grpc-services` artifact + `addService(ProtoReflectionService.newInstance())`). Enable in dev/staging; gate behind auth in prod.

#### Channelz

`grpc-services` ships **channelz** — an admin gRPC service exposing connection/channel/server state.

```java
serverBuilder.addService(ChannelzService.newInstance(100));
```

Then query with `grpcurl ... grpc.channelz.v1.Channelz/...`. Indispensable for debugging "is the channel actually connected?" in production.

#### Protobuf tracing

When the wire contains binary, you can't `tcpdump | strings`. Either:
- Disable TLS in a dev cluster and use Wireshark (it has gRPC dissector).
- Add a logging interceptor that decodes messages with `JsonFormat.printer()`.

Don't run production traces over plaintext to save debugging time — that's how secrets leak.

#### Flame graphs and JFR

For server CPU/lock issues, use async-profiler and JFR (see your JVM guide). gRPC's bottlenecks are usually in your code, not in gRPC itself.

### 11.5 Common debugging scenarios

| Symptom | First check |
|---|---|
| All calls hang | Channel state via channelz; DNS resolution; firewall on port. |
| `UNAVAILABLE` storms | Server up? Sidecar up? GOAWAY in logs? Keepalive misconfigured? |
| `DEADLINE_EXCEEDED` on healthy server | Client deadline too tight; downstream slow; thread starvation. |
| Single-pod traffic in K8s | Headless service / mesh issue (§10.6). |
| Streaming OOM | Backpressure not honored (`isReady`/`onReady`). |
| Message size errors | Hit `maxInboundMessageSize`; raise or paginate. |

---

## 12. Production Best Practices

### 12.1 Service versioning

- **Package versioning** in `.proto`: `payments.v1`, `payments.v2`. The package becomes the gRPC service path component.
- **Run v1 and v2 in parallel** during migrations. Same process can serve both.
- **Never rename a `service` or `method`** within a version; use a new method or a new version.
- **CI gate on schema breaks.** Tools: `buf breaking`, `protolock`. They diff your proto against the previous commit and fail on incompatible changes.

### 12.2 Backward and forward compatibility

Follow the Protobuf evolution rules religiously (§3.5). The most common production breakages:

- Renamed enum values (source-incompatible; old clients see `UNRECOGNIZED`).
- Reused field numbers after deletion (data corruption — old clients write garbage to the new field).
- Tightened validation rules with no version bump.
- Required fields turned into oneof.

Process:

1. Add a new field/method (additive).
2. Deploy server.
3. Deploy clients that use it.
4. Eventually deprecate the old field; mark with `[deprecated = true]`.
5. Remove only after telemetry shows zero traffic on the old field for a long enough quiet period.

### 12.3 Load balancing in K8s

Pick one and document it:

- **gRPC client-side LB** with headless services. Lightweight, no extra infra, but every client needs the right config.
- **Service mesh** (Istio/Linkerd). Centralized; gRPC traffic balanced per request via sidecars; transparent to clients.
- **L7 proxy** (Envoy front, ALB with HTTP/2). Centralized; visible single point.

Whatever you pick, **test it at scale**. Pinned-to-one-pod gRPC on K8s is one of the most common production bugs.

### 12.4 Service discovery

- **DNS** is the simplest target (`dns:///service:port`). Combine with periodic re-resolution to handle pod churn.
- **xDS** (`xds:///service`) for full Envoy-style discovery if your platform supports it.
- **Consul / etcd** via custom name resolvers if you must.

### 12.5 Rolling deployments

- Honor **graceful shutdown**: send GOAWAY (`server.shutdown()`), drain in-flight calls, then exit.
- Spring Boot lifecycle integration handles this with `grpc-spring-boot-starter`.
- Ensure the readiness probe goes red before SIGTERM is acted upon, so the LB stops sending new traffic.

### 12.6 Quotas and rate limiting

- Per-identity rate limits in an interceptor (Bucket4j, Redis-based limiter).
- Server-level concurrency caps (`maxConcurrentCallsPerConnection`).
- Deadline enforcement — long deadlines + many slow clients = DOS.

### 12.7 Documentation

- Treat `.proto` as your API doc.
- Add comments on every message, field, and method. They appear in generated Java JavaDoc and in Buf-generated docs.
- Maintain an **error catalog**: which methods return which status codes and why.
- Maintain a **deprecation calendar** for fields/methods.

---

## 13. Real-World Scenarios

### 13.1 Internal microservice mesh

**Architecture.**
```
api-gateway (REST) ──► order-service (gRPC) ──► payment-service (gRPC)
                                            └──► inventory-service (gRPC)
                                            └──► fulfillment-service (gRPC)
```

**Why gRPC.**
- High-throughput, low-latency internal traffic.
- Strong typing across teams writing in Java, Go, and Python.
- mTLS via service mesh.
- Streaming where useful (e.g., subscribe-to-events APIs).

**Trade-offs.**
- Public edge stays REST/JSON (gateway translates).
- Schema discipline required across teams (CI breaking-change gates).
- Browser-based admin tools need gRPC-Web or REST equivalents.

### 13.2 Real-time price streaming

**Architecture.** Trading service exposes:

```proto
rpc Subscribe (SubscribeRequest) returns (stream Tick);
rpc PlaceOrder (PlaceOrderRequest) returns (OrderAck);
```

A single bidirectional alternative:

```proto
rpc Session (stream ClientMessage) returns (stream ServerMessage);
```

**Why gRPC.**
- Server-streaming gives sub-millisecond push without HTTP polling.
- Backpressure via HTTP/2 flow control prevents memory blow-up.
- Bidi works well for "command + market-data" on one channel.
- Compression for high-volume tick streams.

**Trade-offs.**
- Connection lifecycle complexity (reconnection, resume tokens).
- Browser clients need gRPC-Web (server-streaming only — no client streaming).

### 13.3 ML model serving

**Architecture.** A model server (Triton, custom Python service) exposes:

```proto
rpc Predict (PredictRequest) returns (PredictResponse);
rpc PredictStream (PredictRequest) returns (stream Token);  // for LLM token streaming
```

**Why gRPC.**
- Tight binary serialization for tensor inputs (smaller than JSON).
- Streaming tokens for LLM responses.
- Polyglot — Python servers, Java/Go clients.

**Trade-offs.**
- Large tensors push you toward chunked streaming.
- Browser clients via gRPC-Web again.

### 13.4 IoT control plane

**Architecture.** Devices maintain a long-lived bidi stream to a control service:

```proto
rpc Connect (stream DeviceEvent) returns (stream Command);
```

**Why gRPC.**
- One connection per device; bidi for telemetry up + commands down.
- mTLS with per-device certificates.
- Built-in keepalive detects dropped devices.
- Protobuf compactness matters on cellular/constrained links.

**Trade-offs.**
- Many long-lived streams stress server connection limits.
- HTTP/2 flow control and keepalive tuning become operational concerns.

### 13.5 Bulk import pipeline

**Architecture.** Client streams records into a server endpoint:

```proto
rpc Import (stream Record) returns (ImportSummary);
```

**Why gRPC.**
- Client-streaming with backpressure naturally handles "send as fast as the server allows."
- Single response with summary statistics.
- Idempotency by chunk ID.

**Trade-offs.**
- Mid-stream errors abort the call; design for resumability if you need it.
- Idempotent re-imports require server-side dedup.

---

## 14. Common Pitfalls and Edge Cases

### 14.1 Missing deadlines

**Symptom.** A downstream service hangs; client thread pools exhaust; cascading failure.

**Why.** Default gRPC has no deadline. A hung server holds your client forever.

**Fix.** **Always** `withDeadlineAfter`. Configure default deadlines via service config. Propagate remaining deadline downstream (the framework does this for you if you use `Context`).

### 14.2 K8s single-pod traffic

**Symptom.** N pods of the server, but one of them takes all traffic and the others sit idle.

**Why.** Standard `ClusterIP` Service load-balances at L4 (per TCP connection). gRPC's one long-lived HTTP/2 connection pins to one pod.

**Fix.** Headless service + client-side LB, or service mesh (Istio/Linkerd), or L7 proxy.

### 14.3 Reusing field numbers

**Symptom.** Old clients write garbage into the new field; new servers read the old field as a different type.

**Fix.** Always `reserved` deleted field numbers. CI breaking-change gate (`buf breaking`).

### 14.4 Streaming without backpressure

**Symptom.** Server OOMs under a slow client. Heap fills with queued messages.

**Why.** You're calling `responseObserver.onNext(...)` faster than the client can read.

**Fix.** Cast to `ServerCallStreamObserver`, use `isReady()` and `setOnReadyHandler`. Don't accumulate unbounded buffers on your side.

### 14.5 Large unary payloads

**Symptom.** `RESOURCE_EXHAUSTED: gRPC message too large`.

**Why.** Default `maxInboundMessageSize` is 4 MB. Sending a 50 MB blob is bad for memory pressure even if you raise it.

**Fix.** Stream large payloads in chunks. Raise the limit only when justified, and keep it bounded.

### 14.6 Forgetting to call `onCompleted` or `onError`

**Symptom.** Calls hang client-side until deadline.

**Why.** A service implementation that returns from a Java method without calling `onCompleted` doesn't end the gRPC stream.

**Fix.** Code review and tests. Every code path **must** call exactly one of `onError` / (zero-or-more `onNext` then `onCompleted`).

### 14.7 Mixing blocking and async

**Symptom.** Mysterious deadlocks or starvation.

**Why.** Calling a blocking stub from within a gRPC service handler that's running on a small bounded executor → all threads blocked on each other.

**Fix.** Use async stubs in service handlers, or move blocking calls to a dedicated executor. Project Loom virtual threads largely solve this for blocking I/O paths.

### 14.8 GOAWAY storms during deploys

**Symptom.** Brief spike of `UNAVAILABLE` errors during rolling deployments.

**Why.** Server sends GOAWAY when shutting down; clients reconnect. If many pods rotate at once and clients retry without jitter, brief pile-ups happen.

**Fix.** Surge control on deploys. Client retries with jittered backoff. PreStop hook on the pod that delays SIGTERM until the LB stops sending traffic.

### 14.9 Reflection enabled in production

**Symptom.** Anyone with network access can list your services and their schemas.

**Fix.** Disable reflection in prod, or require auth on the reflection service. Or generate static `.proto` files for clients separately.

### 14.10 `Any` and `oneof` overuse

**Symptom.** Schemas are technically valid, but consumers can't tell what's in them; runtime type errors.

**Fix.** Prefer typed messages over `Any`. Use `oneof` for genuine sum types, not as a generic "I'll figure out later" container.

### 14.11 Forgetting `_UNSPECIFIED = 0` on enums

**Symptom.** Field default is the first non-unspecified value, leading to silent misclassification when the field isn't set.

**Fix.** Always `XXX_UNSPECIFIED = 0` as the first enum value. Lint with buf or protolint.

### 14.12 Channel shutdown leaks

**Symptom.** File descriptors slowly leak; eventually `Too many open files`.

**Why.** `ManagedChannel` not shut down on app close; or new channels created per request.

**Fix.** One channel per target; shut down on bean destruction; use the Spring Boot starter which manages this for you.

---

## 15. When NOT to Use gRPC

### 15.1 Public APIs

If your consumers are external developers, REST is the cultural default. They expect:
- Browseable URLs.
- `curl` and Postman support.
- OpenAPI documentation.
- Well-known semantics for caching.

You can offer gRPC alongside REST for power users — many infrastructure APIs (etcd, Kubernetes, Cilium) do — but never **only** gRPC for a public API.

### 15.2 Browser clients

Native gRPC requires HTTP/2 trailers, which browsers don't expose to JavaScript. To use gRPC from a browser:

- **gRPC-Web** with an Envoy or NGINX proxy that translates between gRPC-Web and gRPC.
- Limitations: no client streaming and no bidi (pre-Connect protocol). Server streaming works.
- **Connect** (from Buf) — a newer protocol speaking gRPC-Web semantics over HTTP/1.1 too.

If your frontend is the primary client, REST or GraphQL is usually less friction.

### 15.3 Edge / CDN caching

REST `GET` is cacheable at every layer of the internet. gRPC isn't. If your workload benefits from edge caching (public APIs, content delivery), don't fight it.

### 15.4 Tiny systems

For a small monolith with two internal services, gRPC adds:
- A build-time codegen step.
- Operational learning curve for the team.
- New tooling (grpcurl, channelz, mTLS).

If REST + Spring Boot is what your team already knows and your perf needs are modest, the simpler path wins.

### 15.5 Highly dynamic queries

If clients need to specify which fields they want, with varying joins and filters, GraphQL is a better fit. gRPC schemas are rigid by design — you'd end up with sprawling RPC method counts.

### 15.6 You don't have CI breaking-change gates

Without buf or equivalent enforcement, gRPC's strong-contract advantage becomes a liability — someone *will* reuse a field number eventually. If the team isn't ready to invest in schema discipline, REST's tolerance for drift may serve you better.

---

## 16. Final Checklist

**You are a senior-level gRPC engineer if you can:**

- Explain in one paragraph what gRPC is, why it requires HTTP/2, and how a unary call flows from stub to skeleton including HEADERS, DATA, trailers, and the `grpc-status` trailer.
- Walk through all four communication modes (unary, server-streaming, client-streaming, bidirectional) and pick the right one for a given scenario, with concrete trade-offs.
- Read a `.proto` file and instantly tell whether a change to it is wire-safe, source-safe, or breaking — including the rules for adding fields, deprecating fields, reusing field numbers, and evolving enums.
- Design a versioning strategy (`v1`, `v2`) with parallel deployment and a CI breaking-change gate.
- Configure a production Spring Boot gRPC server with TLS, deadlines, message-size caps, keepalive, exception mapping, JWT auth, structured logging, and Micrometer metrics.
- Implement a server-streaming method that honors backpressure via `ServerCallStreamObserver.isReady()` / `setOnReadyHandler`, and explain why naive implementations OOM under slow clients.
- Map domain exceptions to the right gRPC status codes and explain the difference between `INVALID_ARGUMENT`, `FAILED_PRECONDITION`, `ABORTED`, and `UNAVAILABLE` — and which of those should be retried.
- Configure declarative retries with bounded attempts, exponential backoff, and jitter — and articulate which methods are safe to retry.
- Diagnose the K8s single-pod gRPC issue and pick a remedy (headless service + client-side LB, or service mesh, or L7 proxy).
- Use grpcurl, channelz, server reflection, JFR, and async-profiler to debug a real production incident — and know which to reach for first based on the symptom.
- Distinguish service-to-service mTLS from end-user JWT auth, and implement both correctly.
- Choose gRPC vs REST vs JSON-RPC vs GraphQL for a given problem and defend the choice with the actual operational and developer-experience trade-offs.
- Refuse to use gRPC for a public, browser-driven, CDN-cached API and explain why — without hand-waving.
- Walk a junior engineer through every common pitfall (missing deadlines, K8s single-pod traffic, reused field numbers, streaming without backpressure, large payloads, GOAWAY storms) and explain both *why* it happens and *how* to prevent it.
- Treat `.proto` files as a first-class API contract — versioned, reviewed, lint-gated, deprecation-tracked — not as boilerplate.

If you can do all of the above without notes, you can confidently design, ship, and operate gRPC services in production at a senior level.
