# JSON-RPC: A Production-Grade Learning Guide

> A complete, example-driven guide for backend engineers who already know REST and want to design, implement, and ship JSON-RPC services in Java + Spring Boot.

---

## Table of Contents

1. [RPC Fundamentals](#1-rpc-fundamentals)
2. [JSON-RPC Fundamentals](#2-json-rpc-fundamentals)
3. [JSON-RPC Specification Basics](#3-json-rpc-specification-basics)
4. [Core Concepts](#4-core-concepts)
5. [Error Handling](#5-error-handling)
6. [JSON-RPC over HTTP](#6-json-rpc-over-http)
7. [JSON-RPC vs REST vs gRPC](#7-json-rpc-vs-rest-vs-grpc)
8. [API Design Best Practices](#8-api-design-best-practices)
9. [Java + Spring Boot Implementation](#9-java--spring-boot-implementation)
10. [JSON-RPC Client in Java](#10-json-rpc-client-in-java)
11. [Security](#11-security)
12. [Observability](#12-observability)
13. [Performance Considerations](#13-performance-considerations)
14. [Real-World Use Cases](#14-real-world-use-cases)
15. [Common Pitfalls](#15-common-pitfalls)
16. [Final Checklist](#16-final-checklist)

---

## 1. RPC Fundamentals

### What is RPC?

**Remote Procedure Call (RPC)** is a communication style in which a client invokes a function on a remote server **as if it were a local function call**. The client says *"do this action"*, and the server runs it and returns a result.

```
client.transferMoney(fromAccount, toAccount, 100.00)
        │
        ▼  (over the network)
server.transferMoney(fromAccount, toAccount, 100.00)
```

The mental model is **action-oriented**: you call a *verb* (`transferMoney`, `createInvoice`, `cancelOrder`).
This contrasts with REST, which is **resource-oriented**: you manipulate *nouns* (`/accounts/123`, `/invoices`, `/orders/42`).

### Why RPC exists

RPC predates REST by decades (Sun RPC, CORBA, XML-RPC, DCOM, Thrift, gRPC). It exists because:

- **Many real-world domains are naturally action-oriented.** "Refund this payment", "rebalance this portfolio", "trigger fraud check" — these aren't easily mapped to CRUD on a resource.
- **Developers think in functions.** RPC matches how programmers already think about behavior.
- **Tooling can hide the network.** Stubs/clients make remote calls look local, reducing boilerplate.
- **It's transport-agnostic.** RPC payloads can flow over HTTP, TCP, WebSocket, message queues, IPC pipes — wherever you need.

### RPC vs REST (high level)

| Dimension | REST | RPC |
|---|---|---|
| Mental model | Resources + uniform verbs | Functions / actions |
| URL semantics | URL = resource identity | URL = single endpoint, method in body |
| HTTP verbs | Required (`GET/POST/PUT/DELETE`) | Almost always `POST` |
| Discoverability | Hypermedia, OpenAPI | Method registry / IDL |
| Best fit | CRUD, public APIs, browsers | Internal services, action-heavy domains |

### Request-Response model

RPC is fundamentally a **request → response** model:

1. Client serializes the call: method name + parameters.
2. Sends it over a transport (usually HTTP).
3. Server deserializes, dispatches to a handler, executes.
4. Server serializes the result (or an error).
5. Client deserializes and returns to the caller.

A subtle but important variant is the **notification** — a request that intentionally has *no* response (fire-and-forget).

### Pros and cons of RPC

**Pros**
- Natural fit for verb-heavy domains (workflows, payments, blockchains, control planes).
- Compact: one endpoint, one envelope, one mental model.
- Easy to add new methods without designing new URLs.
- Strong client codegen ecosystems (especially in gRPC; lighter in JSON-RPC).
- Pairs well with batching and streaming.

**Cons**
- Less HTTP-native: caching, status codes, and content negotiation are weak or unused.
- Browsers and generic HTTP tooling expect REST conventions.
- Discoverability is weaker without an IDL or schema.
- Risk of leaking internal function names into your public contract.
- Harder for cross-team API governance unless you enforce conventions.

### When RPC shines

- Internal microservices where teams own both ends.
- Action-heavy domains (workflow engines, ledger systems, blockchain nodes).
- Long-tail of methods that don't map cleanly to resources.

### When RPC hurts

- Public APIs consumed by many third parties expecting REST conventions.
- CDN/proxy caching matters and you're avoiding `GET`.
- Browser-driven apps where developers expect resource URLs.

---

## 2. JSON-RPC Fundamentals

### What is JSON-RPC?

**JSON-RPC** is a lightweight, transport-agnostic RPC protocol that encodes calls and responses as JSON. The current standard is **JSON-RPC 2.0** (2010), which is short, stable, and intentionally minimal — the entire spec fits on a few pages.

You can think of JSON-RPC as:

> *"RPC with the smallest possible JSON envelope, no IDL, no codegen required."*

### JSON-RPC 2.0 overview

A JSON-RPC interaction is just two JSON objects:

- A **Request** identifies a method, optional parameters, and an `id`.
- A **Response** returns either a `result` or an `error`, with the same `id`.

There is no schema language, no enforced type system, no service registry. The entire contract is *"agree on method names and param shapes."*

### Key characteristics

- **Transport-independent.** HTTP, WebSocket, TCP, stdin/stdout, message queues — anything that moves bytes.
- **Stateless.** Each request stands alone (state lives in your app, not the protocol).
- **Symmetric.** Server and client roles can swap (especially over WebSocket).
- **Bidirectional-friendly.** Useful for push/notifications when on a duplex transport.
- **Batch-aware.** Multiple requests can be packed into a single array.
- **Notification-aware.** Requests without `id` are fire-and-forget.

### Transport independence

This is one of JSON-RPC's biggest advantages. The same envelope works over:

| Transport | Typical usage |
|---|---|
| HTTP | Most server-to-server APIs (Ethereum, Solana, internal microservices). |
| WebSocket | Live data, server push, IDE protocols (LSP). |
| TCP raw | Embedded systems, legacy infra. |
| stdin/stdout | Editor extensions (LSP, DAP), CLI tools. |
| Kafka / NATS | Async command-style RPC over message buses. |

### Simple request/response example

**Request**
```json
{
  "jsonrpc": "2.0",
  "method": "user.getById",
  "params": { "id": 42 },
  "id": 1
}
```

**Successful response**
```json
{
  "jsonrpc": "2.0",
  "result": { "id": 42, "name": "Ada Lovelace", "email": "ada@example.com" },
  "id": 1
}
```

**Error response**
```json
{
  "jsonrpc": "2.0",
  "error": { "code": -32602, "message": "Invalid params: id must be positive" },
  "id": 1
}
```

That's the whole protocol surface. Everything else in this guide is *how to use it well*.

---

## 3. JSON-RPC Specification Basics

### The Request object

| Field | Type | Required | Purpose |
|---|---|---|---|
| `jsonrpc` | string | yes | Must be exactly `"2.0"`. Identifies the protocol version. |
| `method` | string | yes | Name of the method to invoke. SHOULD NOT begin with `rpc.` (reserved). |
| `params` | array or object | no | Positional (array) or named (object) arguments. |
| `id` | string, number, or null | no | Correlation ID. Omit it to make the call a **notification**. |

#### `jsonrpc`
Always `"2.0"`. If absent, the message is treated as JSON-RPC 1.0 (legacy) or invalid.

#### `method`
A string identifying the procedure. Common conventions:
- Dot-namespaced: `wallet.sendTransaction`, `user.create`, `order.cancel`.
- Verb-first or resource-first — pick one and stick to it.
- Avoid names starting with `rpc.` — those are reserved for future extensions and introspection.

#### `params`
Either:
- **By position** (`[arg1, arg2, ...]`) — terse, brittle when arguments evolve.
- **By name** (`{ "name": "...", "amount": 100 }`) — verbose, robust to reordering and additions. **Prefer this in production.**

#### `id`
- **Present (string/number)** → server MUST reply with the same `id`.
- **`null`** → technically valid but discouraged (ambiguous: is it a notification or a missing id?).
- **Absent** → this is a **notification**; server MUST NOT reply.

### The Response object

A response always contains `jsonrpc: "2.0"` and `id`, plus **exactly one** of `result` or `error` — never both, never neither.

| Field | Type | Required | Purpose |
|---|---|---|---|
| `jsonrpc` | string | yes | Always `"2.0"`. |
| `result` | any | only on success | Method's return value. |
| `error` | object | only on failure | Error envelope (see below). |
| `id` | same as request | yes | Echoes the request's `id`. If it couldn't be parsed, must be `null`. |

### The Error object

| Field | Type | Required | Purpose |
|---|---|---|---|
| `code` | integer | yes | Machine-readable error code. |
| `message` | string | yes | Short, human-readable summary. |
| `data` | any | no | Optional structured details (validation errors, trace IDs, etc.). |

### Valid examples

**Valid request (named params)**
```json
{
  "jsonrpc": "2.0",
  "method": "transfer.execute",
  "params": { "from": "A1", "to": "B2", "amountCents": 5000 },
  "id": "tx-9f3"
}
```

**Valid notification (no id)**
```json
{
  "jsonrpc": "2.0",
  "method": "metrics.recordHit",
  "params": { "endpoint": "/checkout" }
}
```

**Valid success response**
```json
{ "jsonrpc": "2.0", "result": { "txId": "0xabc..." }, "id": "tx-9f3" }
```

**Valid error response**
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": { "field": "amountCents", "reason": "must be > 0" }
  },
  "id": "tx-9f3"
}
```

### Invalid examples (and why)

```json
{ "method": "x", "id": 1 }
```
❌ Missing `jsonrpc`.

```json
{ "jsonrpc": "2.0", "method": "x", "result": null, "error": null, "id": 1 }
```
❌ Both `result` and `error` present (mutually exclusive).

```json
{ "jsonrpc": "2.0", "method": "rpc.internal", "id": 1 }
```
❌ `rpc.` prefix is reserved.

```json
{ "jsonrpc": "2.0", "result": "ok", "id": 1 }
```
❌ Response missing the discriminator: it's a response shape but `error`/`result` rules apply only on the response side; without context, this is fine, but having no `id` echo when the server couldn't parse the request would require `id: null`.

---

## 4. Core Concepts

### 4.1 Methods

A **method** is a named server-side procedure. It is the JSON-RPC equivalent of an endpoint.

**Why it exists:** RPC is action-oriented; the method name *is* the action.

**When to use:** Every JSON-RPC call. There is no alternative.

**When not to use:** N/A — but choose names carefully (see §8).

**Example**
```json
{ "jsonrpc": "2.0", "method": "invoice.markPaid", "params": { "invoiceId": "INV-100" }, "id": 1 }
```

### 4.2 Params: positional vs named

**Positional**
```json
{ "jsonrpc": "2.0", "method": "math.add", "params": [2, 3], "id": 1 }
```

**Named**
```json
{ "jsonrpc": "2.0", "method": "math.add", "params": { "a": 2, "b": 3 }, "id": 1 }
```

| | Positional | Named |
|---|---|---|
| Verbosity | Lowest | Slightly higher |
| Refactor safety | Poor (order matters) | Good (add/reorder freely) |
| Optional args | Awkward | Natural |
| Best for | Tiny stable methods, math-style APIs | **Everything else** |

**Rule of thumb:** Default to **named params** in production. Reserve positional for trivial, frozen methods.

### 4.3 IDs

The `id` correlates a response with its request. It must be unique *within an in-flight conversation*.

**Why it exists:** Over duplex transports (WebSocket) or batched requests, responses can arrive out of order. The `id` lets the client match them up.

**When to use:** Every non-notification request.

**When not to use:** Notifications (omit `id` entirely — don't send `id: null`).

**Best practices**
- Use a UUID, ULID, or monotonically increasing counter.
- Don't reuse IDs for different in-flight requests.
- Don't leak meaning into the ID (it's a correlation token, not a business key).

### 4.4 Notifications

A **notification** is a request with **no `id` field**. The server processes it and **must not** respond.

**Why it exists:** Some calls are fire-and-forget (telemetry, logs, push events). Skipping the response saves a round trip and lets the server batch/queue work.

**When to use:**
- Telemetry (`log.event`, `metrics.increment`)
- Push notifications from server → client over WebSocket
- Async commands where the caller doesn't need the result

**When not to use:**
- Anything where you need confirmation, a return value, or error reporting.
- Anything safety-critical (payments, mutations) — the client cannot tell whether the server processed it.

**Example**
```json
{ "jsonrpc": "2.0", "method": "telemetry.pageView", "params": { "url": "/home" } }
```

The server returns nothing — not even an empty body in HTTP terms (typically `204 No Content`).

### 4.5 Batch requests

A **batch** is a JSON array of requests sent in one transport call. The server responds with an array of responses (in any order; clients use `id` to correlate).

**Request**
```json
[
  { "jsonrpc": "2.0", "method": "user.getById", "params": { "id": 1 }, "id": "a" },
  { "jsonrpc": "2.0", "method": "user.getById", "params": { "id": 2 }, "id": "b" },
  { "jsonrpc": "2.0", "method": "metrics.hit", "params": { "page": "/x" } }
]
```

**Response** (only the two non-notifications produce entries)
```json
[
  { "jsonrpc": "2.0", "result": { "id": 1, "name": "Ada" }, "id": "a" },
  { "jsonrpc": "2.0", "result": { "id": 2, "name": "Linus" }, "id": "b" }
]
```

**Why it exists:** Reduces network overhead when many small calls happen together.

**When to use:**
- Many small, independent reads on a chatty transport.
- Initial page loads that need several pieces of data.

**When not to use:**
- Calls that depend on each other's results (no ordering or data flow guarantees).
- Calls with very different latencies (the slow one blocks the response).
- Mutations where partial success is hard to reason about.

**Edge cases**
- Empty batch `[]` → Invalid Request (`-32600`), single response.
- All-notifications batch → server returns nothing (HTTP 204 typically).
- Mixed valid/invalid → server returns a response array containing per-element errors.

---

## 5. Error Handling

JSON-RPC errors are **structured**. They are not HTTP status codes, not stack traces, not strings — they are objects with `code`, `message`, and optional `data`.

### 5.1 Standard error codes

The spec reserves codes from **-32768 to -32000** for protocol-level errors:

| Code | Meaning | When |
|---|---|---|
| -32700 | Parse error | Malformed JSON received. |
| -32600 | Invalid Request | JSON is valid but not a valid Request object. |
| -32601 | Method not found | Method name does not exist. |
| -32602 | Invalid params | Params are wrong type, missing, or out of range. |
| -32603 | Internal error | Unexpected server-side failure. |
| -32000 to -32099 | Server error | Implementation-defined server errors. |

Anything outside `-32768..-32000` is yours to use for **application-level** errors.

### 5.2 Custom (application) errors

Define a **stable, documented** range for your domain. Common conventions:

```
1000–1999  Validation
2000–2999  Authentication / authorization
3000–3999  Business rules (insufficient funds, duplicate, not found)
4000–4999  External dependency failures
```

**Example — business error**
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": 3001,
    "message": "Insufficient funds",
    "data": { "accountId": "A1", "available": 1500, "requested": 5000 }
  },
  "id": "tx-9f3"
}
```

### 5.3 Validation errors

Use `-32602` (Invalid params) for type/shape problems and your own range for **semantic** validation. Put the field-level details in `data`.

```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params",
    "data": {
      "violations": [
        { "field": "email", "reason": "must be a valid email" },
        { "field": "age", "reason": "must be >= 18" }
      ]
    }
  },
  "id": 7
}
```

### 5.4 Best practices

- **Pick error codes once and never reuse them.** They become part of your contract.
- **Document every code** alongside the method.
- **Never leak stack traces or internal class names** in `message` or `data`.
- **Use `data` for machine-readable details** (field paths, retry hints, trace IDs).
- **Never put a result inside an error or vice versa.** They are mutually exclusive.
- **Always echo the request `id`.** If you couldn't parse it, return `id: null`.
- **Distinguish retryable vs non-retryable** errors (often via a `data.retryable: true` flag).
- **Map exceptions centrally** — don't sprinkle `try/catch` in every handler.

---

## 6. JSON-RPC over HTTP

JSON-RPC doesn't mandate HTTP, but HTTP is by far the most common transport.

### How JSON-RPC uses HTTP

- **One endpoint** for everything (e.g., `POST /rpc`).
- **HTTP method:** almost always `POST` (rarely `GET` for cacheable reads, but uncommon).
- **Content-Type:** `application/json`.
- **Body:** the JSON-RPC request (or batch array).
- **Response body:** the JSON-RPC response (or batch array).

### Endpoint design

| Pattern | Example | Notes |
|---|---|---|
| Single endpoint | `POST /rpc` | The classic. Works fine for most cases. |
| Versioned | `POST /rpc/v1` | Use when you need parallel major versions. |
| Namespaced | `POST /rpc/wallet`, `POST /rpc/admin` | Useful for routing/auth boundaries. |

### HTTP status codes

This is the most misunderstood part. **JSON-RPC carries its own errors in the body.** HTTP status codes should reflect the *transport*, not the application outcome.

| Situation | HTTP status |
|---|---|
| Request was processed (success or business error) | **200 OK** |
| Notification accepted, no body | **204 No Content** |
| Malformed JSON (parse error) | **200 OK** with `-32700` body, or **400 Bad Request** |
| Invalid Request shape | **200 OK** with `-32600` body, or **400 Bad Request** |
| Auth required / failed | **401 / 403** (auth lives at the transport layer) |
| Method exists but server is overloaded | **429 / 503** |

> **Rule:** Application-level errors are **always** in the JSON-RPC body with HTTP 200. Transport/auth errors use HTTP status codes.

### Request example
```http
POST /rpc HTTP/1.1
Host: api.example.com
Content-Type: application/json
Authorization: Bearer eyJhbGciOi...

{ "jsonrpc": "2.0", "method": "user.create", "params": { "name": "Ada" }, "id": 1 }
```

### Response example
```http
HTTP/1.1 200 OK
Content-Type: application/json

{ "jsonrpc": "2.0", "result": { "id": 42 }, "id": 1 }
```

---

## 7. JSON-RPC vs REST vs gRPC

### Comparison table

| Dimension | REST | JSON-RPC | gRPC |
|---|---|---|---|
| Style | Resource-oriented | Action-oriented | Action-oriented |
| Encoding | JSON (usually) | JSON | Protocol Buffers (binary) |
| Schema/IDL | Optional (OpenAPI) | None by default | Mandatory (`.proto`) |
| Transport | HTTP/1.1, HTTP/2 | HTTP, WebSocket, TCP, anything | HTTP/2 only |
| Browser-native | Yes | Yes (over HTTP) | No (needs gRPC-Web) |
| Streaming | Limited (SSE) | Via WebSocket | First-class (4 modes) |
| Performance | Medium | Medium (JSON cost) | High (binary + HTTP/2) |
| Tooling/codegen | OpenAPI generators | Sparse | Excellent (multi-language) |
| Caching (CDN/proxy) | Strong (`GET`) | Weak (everything `POST`) | None |
| Discoverability | Strong (OpenAPI/HATEOAS) | Weak | Strong (proto reflection) |
| Complexity | Low | **Lowest** | Higher |
| Best fit | Public CRUD APIs | Internal action APIs, blockchain, JSON-native consumers | High-perf microservices, polyglot stacks |

### When to use JSON-RPC

- **Internal microservices** where teams agree on conventions.
- **Action-heavy domains** that don't fit CRUD (workflow engines, ledger ops, blockchain RPC).
- **JSON-native clients** (browsers, scripting languages) where Protobuf adds friction.
- **Bidirectional or push** scenarios over WebSocket (LSP, trading feeds).
- **Quick to ship** — no IDL, no codegen, no infra dependencies.

### When NOT to use JSON-RPC

- **Public APIs consumed by many third parties** — they expect REST conventions.
- **High-throughput, low-latency** internal traffic — gRPC's binary format and HTTP/2 multiplexing win.
- **CDN/edge caching is critical** — REST `GET` is far better.
- **You need first-class streaming** beyond simple notifications — use gRPC.
- **Strong typing across many languages** is a hard requirement — gRPC's `.proto` ecosystem is unmatched.

---

## 8. API Design Best Practices

### 8.1 Method naming

Pick **one** convention and apply it everywhere.

**`namespace.action`** (most common)
```
user.create
user.getById
order.cancel
wallet.sendTransaction
```

**`namespace_action`**
```
user_create
order_cancel
```

Guidelines:
- **Verbs for actions**, not generic CRUD-only names.
- **Nouns first** when grouping; this enables namespace-based routing and authz.
- **Avoid `rpc.` prefix** — reserved by the spec.
- **Be consistent with case** — `camelCase` is most common in JSON-RPC ecosystems.
- **Don't expose internal class/method names.** Method names are part of your contract; they outlive the code.

### 8.2 Parameter design

- **Use named params (objects), not positional arrays.** Adding/reordering fields stays backward-compatible.
- **Make optional fields truly optional** — handlers must tolerate missing keys.
- **Keep the param shape stable** — don't change a field's type or meaning silently.
- **Validate at the edge** with declarative rules (Bean Validation, JSON Schema).
- **Prefer flat objects** over deep nesting unless the domain truly is nested.
- **Use clear, domain-language field names** — `amountCents`, not `amt`.

### 8.3 Versioning

JSON-RPC has no built-in versioning, so you must add one. Three patterns:

| Pattern | Example | Pros | Cons |
|---|---|---|---|
| **Method prefix** | `v1.user.create` | Per-method versioning, fine-grained | Many duplicated names |
| **Endpoint path** | `POST /rpc/v2` | Clean routing, clear boundary | Coarse-grained |
| **Namespace** | `userV2.create` | Per-domain | Same drawbacks as method prefix |

**Recommended:** Endpoint path versioning (`/rpc/v1`, `/rpc/v2`) for major versions, plus additive evolution within a version.

### 8.4 Backward compatibility

The cardinal rule: **never break existing clients silently.**

**Safe (additive) changes**
- Adding new methods.
- Adding new optional fields to params or result.
- Adding new error codes (clients should handle unknown codes gracefully).

**Breaking changes — require a new version**
- Removing methods, params, or result fields.
- Renaming methods or fields.
- Changing field types or semantics.
- Tightening validation (e.g., new required fields).

**Deprecation playbook**
1. Document the method as deprecated and announce the timeline.
2. Add a sibling/replacement method.
3. Log every call to the deprecated method with caller identity.
4. Set and communicate a removal date; remove only in a new major version.

---

## 9. Java + Spring Boot Implementation

This section builds a minimal but production-shaped JSON-RPC server in Spring Boot. We deliberately use a **single endpoint + dispatcher** design rather than a third-party JSON-RPC library, because the protocol is small enough that you usually want full control.

### 9.1 Project setup

`pom.xml` (key dependencies):
```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
  </dependency>
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

### 9.2 Protocol DTOs

```java
package com.example.rpc.protocol;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.databind.JsonNode;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record JsonRpcRequest(
    String jsonrpc,
    String method,
    JsonNode params,   // can be object or array; resolve later
    JsonNode id        // string, number, or null
) {}
```

```java
package com.example.rpc.protocol;

import com.fasterxml.jackson.annotation.JsonInclude;
import com.fasterxml.jackson.databind.JsonNode;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record JsonRpcResponse(
    String jsonrpc,
    Object result,
    JsonRpcError error,
    JsonNode id
) {
    public static JsonRpcResponse ok(Object result, JsonNode id) {
        return new JsonRpcResponse("2.0", result, null, id);
    }
    public static JsonRpcResponse fail(JsonRpcError error, JsonNode id) {
        return new JsonRpcResponse("2.0", null, error, id);
    }
}
```

```java
package com.example.rpc.protocol;

import com.fasterxml.jackson.annotation.JsonInclude;

@JsonInclude(JsonInclude.Include.NON_NULL)
public record JsonRpcError(int code, String message, Object data) {
    public static JsonRpcError parseError()           { return new JsonRpcError(-32700, "Parse error", null); }
    public static JsonRpcError invalidRequest()       { return new JsonRpcError(-32600, "Invalid Request", null); }
    public static JsonRpcError methodNotFound(String m){ return new JsonRpcError(-32601, "Method not found: " + m, null); }
    public static JsonRpcError invalidParams(Object d){ return new JsonRpcError(-32602, "Invalid params", d); }
    public static JsonRpcError internalError(String m){ return new JsonRpcError(-32603, "Internal error", m); }
}
```

### 9.3 A typed exception for business errors

```java
package com.example.rpc.protocol;

public class JsonRpcException extends RuntimeException {
    private final int code;
    private final transient Object data;

    public JsonRpcException(int code, String message, Object data) {
        super(message);
        this.code = code;
        this.data = data;
    }
    public int code() { return code; }
    public Object data() { return data; }
}
```

### 9.4 The handler interface and a sample implementation

```java
package com.example.rpc.dispatch;

import com.fasterxml.jackson.databind.JsonNode;

public interface JsonRpcHandler {
    String method();          // e.g. "user.getById"
    Object handle(JsonNode params) throws Exception;
}
```

```java
package com.example.rpc.user;

import com.example.rpc.dispatch.JsonRpcHandler;
import com.example.rpc.protocol.JsonRpcException;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.stereotype.Component;

@Component
public class UserGetByIdHandler implements JsonRpcHandler {

    private final UserService users;
    private final ObjectMapper mapper;

    public UserGetByIdHandler(UserService users, ObjectMapper mapper) {
        this.users = users;
        this.mapper = mapper;
    }

    @Override public String method() { return "user.getById"; }

    public record Params(Long id) {}

    @Override
    public Object handle(JsonNode params) throws Exception {
        Params p = mapper.treeToValue(params, Params.class);
        if (p.id() == null || p.id() <= 0) {
            throw new JsonRpcException(-32602, "Invalid params",
                java.util.Map.of("field", "id", "reason", "must be > 0"));
        }
        return users.findById(p.id())
            .orElseThrow(() -> new JsonRpcException(3001, "User not found",
                java.util.Map.of("id", p.id())));
    }
}
```

### 9.5 The dispatcher

The dispatcher owns: routing, request validation, error mapping, and batch handling.

```java
package com.example.rpc.dispatch;

import com.example.rpc.protocol.*;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.stream.Collectors;

@Component
public class JsonRpcDispatcher {

    private final Map<String, JsonRpcHandler> handlers;
    private final ObjectMapper mapper;

    public JsonRpcDispatcher(List<JsonRpcHandler> handlers, ObjectMapper mapper) {
        this.handlers = handlers.stream()
            .collect(Collectors.toMap(JsonRpcHandler::method, h -> h));
        this.mapper = mapper;
    }

    /** Dispatch a parsed JSON node (object or array). Returns null if all entries were notifications. */
    public Object dispatch(JsonNode root) {
        if (root.isArray()) {
            ArrayNode arr = (ArrayNode) root;
            if (arr.isEmpty()) {
                return JsonRpcResponse.fail(JsonRpcError.invalidRequest(), null);
            }
            List<JsonRpcResponse> out = new ArrayList<>();
            for (JsonNode item : arr) {
                JsonRpcResponse r = dispatchSingle(item);
                if (r != null) out.add(r);
            }
            return out.isEmpty() ? null : out;
        }
        return dispatchSingle(root);
    }

    private JsonRpcResponse dispatchSingle(JsonNode node) {
        if (!node.isObject()) {
            return JsonRpcResponse.fail(JsonRpcError.invalidRequest(), null);
        }
        JsonNode id = node.get("id");
        boolean isNotification = (id == null);

        if (!"2.0".equals(text(node, "jsonrpc")) || text(node, "method") == null) {
            return isNotification ? null
                : JsonRpcResponse.fail(JsonRpcError.invalidRequest(), id);
        }

        String method = node.get("method").asText();
        JsonRpcHandler handler = handlers.get(method);
        if (handler == null) {
            return isNotification ? null
                : JsonRpcResponse.fail(JsonRpcError.methodNotFound(method), id);
        }

        try {
            Object result = handler.handle(node.get("params"));
            return isNotification ? null : JsonRpcResponse.ok(result, id);
        } catch (JsonRpcException e) {
            return isNotification ? null
                : JsonRpcResponse.fail(new JsonRpcError(e.code(), e.getMessage(), e.data()), id);
        } catch (IllegalArgumentException e) {
            return isNotification ? null
                : JsonRpcResponse.fail(JsonRpcError.invalidParams(e.getMessage()), id);
        } catch (Exception e) {
            return isNotification ? null
                : JsonRpcResponse.fail(JsonRpcError.internalError(e.getClass().getSimpleName()), id);
        }
    }

    private static String text(JsonNode n, String f) {
        JsonNode v = n.get(f);
        return v == null || v.isNull() ? null : v.asText();
    }
}
```

### 9.6 The single HTTP endpoint

```java
package com.example.rpc.web;

import com.example.rpc.dispatch.JsonRpcDispatcher;
import com.example.rpc.protocol.*;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.http.MediaType;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping(value = "/rpc", consumes = MediaType.APPLICATION_JSON_VALUE)
public class JsonRpcController {

    private final JsonRpcDispatcher dispatcher;
    private final ObjectMapper mapper;

    public JsonRpcController(JsonRpcDispatcher dispatcher, ObjectMapper mapper) {
        this.dispatcher = dispatcher;
        this.mapper = mapper;
    }

    @PostMapping
    public ResponseEntity<?> handle(@RequestBody String rawBody) {
        JsonNode root;
        try {
            root = mapper.readTree(rawBody);
        } catch (Exception e) {
            return ResponseEntity.ok(JsonRpcResponse.fail(JsonRpcError.parseError(), null));
        }
        Object result = dispatcher.dispatch(root);
        if (result == null) {
            return ResponseEntity.noContent().build(); // 204 for all-notification batches
        }
        return ResponseEntity.ok(result);
    }
}
```

### 9.7 What this gives you

- **One endpoint** (`POST /rpc`) handles everything.
- **Auto-registration** of handlers via Spring's `List<JsonRpcHandler>` injection.
- **Batch + notification** handling out of the box.
- **Centralized error mapping** — handlers throw `JsonRpcException` with a code/message/data.
- **No third-party JSON-RPC library** — easy to audit, easy to extend.

### 9.8 End-to-end test

```bash
curl -s http://localhost:8080/rpc \
  -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"user.getById","params":{"id":42},"id":1}'
```

```json
{ "jsonrpc": "2.0", "result": { "id": 42, "name": "Ada" }, "id": 1 }
```

---

## 10. JSON-RPC Client in Java

### 10.1 With `RestClient` (Spring 6.1+, recommended for sync)

```java
package com.example.rpc.client;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.web.client.RestClient;

import java.util.Map;
import java.util.UUID;

public class JsonRpcClient {

    private final RestClient http;
    private final ObjectMapper mapper;

    public JsonRpcClient(String baseUrl, ObjectMapper mapper) {
        this.http = RestClient.builder().baseUrl(baseUrl).build();
        this.mapper = mapper;
    }

    public <T> T call(String method, Object params, Class<T> resultType) {
        Map<String, Object> body = Map.of(
            "jsonrpc", "2.0",
            "method", method,
            "params", params,
            "id", UUID.randomUUID().toString()
        );

        JsonNode response = http.post()
            .uri("/rpc")
            .body(body)
            .retrieve()
            .body(JsonNode.class);

        if (response.has("error") && !response.get("error").isNull()) {
            JsonNode err = response.get("error");
            throw new JsonRpcClientException(
                err.get("code").asInt(),
                err.get("message").asText(),
                err.get("data")
            );
        }
        try {
            return mapper.treeToValue(response.get("result"), resultType);
        } catch (Exception e) {
            throw new RuntimeException("Failed to deserialize result", e);
        }
    }

    public void notify(String method, Object params) {
        Map<String, Object> body = Map.of(
            "jsonrpc", "2.0",
            "method", method,
            "params", params
        );
        http.post().uri("/rpc").body(body).retrieve().toBodilessEntity();
    }
}
```

```java
public class JsonRpcClientException extends RuntimeException {
    private final int code;
    private final transient Object data;
    public JsonRpcClientException(int code, String message, Object data) {
        super(message);
        this.code = code;
        this.data = data;
    }
    public int code() { return code; }
    public Object data() { return data; }
}
```

**Usage**
```java
JsonRpcClient client = new JsonRpcClient("https://api.example.com", new ObjectMapper());
User u = client.call("user.getById", Map.of("id", 42), User.class);
client.notify("metrics.hit", Map.of("page", "/home"));
```

### 10.2 With `WebClient` (reactive)

```java
package com.example.rpc.client;

import com.fasterxml.jackson.databind.JsonNode;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.core.publisher.Mono;

import java.util.Map;
import java.util.UUID;

public class ReactiveJsonRpcClient {

    private final WebClient web;

    public ReactiveJsonRpcClient(String baseUrl) {
        this.web = WebClient.builder().baseUrl(baseUrl).build();
    }

    public Mono<JsonNode> call(String method, Object params) {
        return web.post()
            .uri("/rpc")
            .bodyValue(Map.of(
                "jsonrpc", "2.0",
                "method", method,
                "params", params,
                "id", UUID.randomUUID().toString()))
            .retrieve()
            .bodyToMono(JsonNode.class);
    }
}
```

### 10.3 Batch calls

```java
public JsonNode batch(List<Map<String, Object>> requests) {
    return http.post().uri("/rpc").body(requests)
        .retrieve().body(JsonNode.class);
}
```

The response is a JSON array; correlate by `id`.

### 10.4 Client-side concerns

- **Timeouts** — always set a connection + read timeout.
- **Retries** — only for idempotent methods or notifications.
- **Circuit breakers** — Resilience4j fits well at the client layer.
- **Auth headers** — attach `Authorization` once via an interceptor.
- **Tracing** — propagate `traceparent` and any tenant headers.

---

## 11. Security

JSON-RPC has **no built-in security**. Everything below lives at the transport/HTTP layer or your application layer.

### 11.1 Authentication

| Method | When to use | Notes |
|---|---|---|
| **JWT (Bearer)** | User-facing services, microservices with a shared identity provider | Validate signature + claims (`exp`, `aud`, `iss`). |
| **API keys** | Server-to-server, third-party integrations | Rotate; never log full keys; hash at rest. |
| **mTLS** | Internal service mesh, zero-trust networks | Strongest; requires PKI infrastructure. |
| **HMAC signatures** | Webhooks, untrusted networks | Sign body + timestamp; reject stale requests. |

**Spring Security filter sketch**
```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    private final JwtVerifier verifier;
    public JwtAuthFilter(JwtVerifier v) { this.verifier = v; }

    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws IOException, ServletException {
        String header = req.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            try {
                var principal = verifier.verify(header.substring(7));
                SecurityContextHolder.getContext().setAuthentication(
                    new UsernamePasswordAuthenticationToken(principal, null, principal.authorities()));
            } catch (Exception e) {
                res.sendError(HttpServletResponse.SC_UNAUTHORIZED);
                return;
            }
        }
        chain.doFilter(req, res);
    }
}
```

### 11.2 Authorization

- **Method-level checks.** Map each method (or namespace) to required roles/scopes:
  ```java
  Map.of(
    "admin.*",   "ROLE_ADMIN",
    "wallet.*",  "SCOPE_wallet:write"
  )
  ```
- **Object-level checks.** Inside the handler, verify the caller can act on this entity (tenant ID match, ownership).
- **Deny by default.** Unknown methods or missing scopes → reject; never silently allow.

### 11.3 Input validation

- **Validate at the dispatcher boundary** — before any business logic runs.
- **Use Bean Validation** (`@NotNull`, `@Min`, `@Email`) on param records when you map them.
- **Never trust client-provided IDs** — always re-check tenancy and ownership.
- **Reject oversized payloads** at the HTTP layer (Spring's `spring.servlet.multipart.max-request-size` or a filter).
- **Beware deeply nested JSON** — set a max depth in Jackson to prevent stack-bomb attacks.

### 11.4 Other essentials

- **TLS everywhere.** No JSON-RPC traffic over plain HTTP, even internally.
- **Rate limiting per identity**, not per IP, when behind proxies.
- **Audit logging** for all mutating methods (who, when, what, result).
- **Don't echo secrets** in `error.data`.
- **CORS** if browser clients call the endpoint — restrict origins.

---

## 12. Observability

### 12.1 Logging

Log the **method**, **id**, **caller**, **duration**, and **outcome** — not the full payload (PII risk).

```java
@Component
public class LoggingDispatcherWrapper {
    private static final Logger log = LoggerFactory.getLogger(LoggingDispatcherWrapper.class);

    public Object dispatch(JsonNode req, JsonRpcDispatcher inner) {
        long start = System.nanoTime();
        String method = req.path("method").asText("?");
        String id = req.path("id").asText("notif");
        try {
            Object out = inner.dispatch(req);
            log.info("rpc method={} id={} status=ok durMs={}",
                method, id, (System.nanoTime() - start) / 1_000_000);
            return out;
        } catch (Exception e) {
            log.error("rpc method={} id={} status=err err={}", method, id, e.toString());
            throw e;
        }
    }
}
```

Use **structured logging** (JSON logs) so you can filter by `method` and `status`.

### 12.2 Metrics

With Micrometer:

```java
Timer.builder("jsonrpc.requests")
     .tag("method", method)
     .tag("status", status)  // ok | error
     .register(meterRegistry)
     .record(Duration.ofNanos(elapsed));
```

Track at minimum:
- `jsonrpc.requests.count` by `method` and `status`.
- `jsonrpc.requests.duration` (histogram) by `method`.
- `jsonrpc.errors.count` by `code`.
- `jsonrpc.batch.size` (histogram).

### 12.3 Tracing

- Use **OpenTelemetry**: each RPC call is a span named after the method (`rpc.user.getById`).
- Propagate `traceparent` through clients.
- Attach attributes: `rpc.system=jsonrpc`, `rpc.method`, `rpc.jsonrpc.error_code`.

### 12.4 Debugging

- **Echo correlation IDs** in `error.data.traceId` so a user can quote it back.
- **Provide a `rpc.ping` method** for liveness from clients.
- **Dump method registry** in a non-prod admin endpoint for discovery.
- **Replay-friendly logs**: store the raw request (PII-redacted) for the last N failures.

---

## 13. Performance Considerations

### 13.1 Payload size

- **JSON is verbose** — field names repeat in every object. Trim only when measured.
- **Don't return huge collections by default.** Add pagination params (`limit`, `cursor`).
- **Compression matters.** Enable HTTP gzip/br for response bodies (`server.compression.enabled=true` in Spring Boot).
- **Stream large results** if you can — but JSON-RPC over HTTP isn't great for streaming; consider WebSocket or a separate streaming endpoint.

### 13.2 JSON parsing cost

- **Reuse `ObjectMapper`** — it's thread-safe and expensive to construct.
- **Use `JsonNode` for routing** and only deserialize into typed records inside the matching handler.
- **Avoid double parsing** — pass the parsed tree through, don't re-stringify.
- **Consider afterburner** modules or `jackson-jr` for hot paths if profiling shows JSON cost dominating.
- **Limit input size and depth** to prevent CPU/memory abuse.

### 13.3 Batching

- **Good:** N small reads done together (latency dominated by RTT).
- **Bad:** mixing fast and slow calls — the slowest blocks the response.
- **Bad:** mutations in a batch — partial success is hard to reason about.
- **Cap batch size** server-side (e.g., 50). Reject larger batches with `-32600`.
- **Don't batch by default** in your client — only when you have a real reason.

### 13.4 Other levers

- **Connection reuse** (HTTP/1.1 keep-alive or HTTP/2) on the client.
- **Async handlers** with `CompletableFuture` for I/O-bound methods.
- **Cache idempotent reads** at the application layer (Redis/Caffeine), keyed by `method + canonical(params)`.
- **Avoid unbounded recursion** in JSON params; reject deep payloads at the parser.

---

## 14. Real-World Use Cases

### 14.1 Blockchain node APIs (Ethereum, Solana, Bitcoin)

The **canonical** JSON-RPC use case. Every Ethereum node (Geth, Erigon, Nethermind) exposes JSON-RPC:

```json
{ "jsonrpc": "2.0", "method": "eth_getBalance",
  "params": ["0xabc...", "latest"], "id": 1 }
```

- Hundreds of methods (`eth_*`, `net_*`, `web3_*`, `debug_*`, `trace_*`).
- Identical wire format works over HTTP, WebSocket (for subscriptions), and IPC.
- Notifications (`eth_subscribe`/`newHeads`) push new blocks via WebSocket.
- A poor fit for REST: the domain is verbs (`call`, `sendRawTransaction`, `estimateGas`).

### 14.2 Internal microservices

A workflow service exposing dozens of actions:

```
workflow.start
workflow.cancel
workflow.signal
workflow.queryState
workflow.scheduleRetry
```

- One endpoint per service is easier to govern than 30 REST routes.
- Method-level authz (`workflow.cancel` requires `workflow:write`) maps cleanly.
- Adding a new action is one method, not a new URL + verb decision.

### 14.3 Editor protocols (LSP, DAP)

The **Language Server Protocol** (used by VS Code, JetBrains, Neovim) is JSON-RPC over stdio.

- Bidirectional: server pushes `textDocument/publishDiagnostics`; client requests `textDocument/completion`.
- Notifications dominate (`textDocument/didChange`).
- The transport (stdio) is what matters — JSON-RPC just works there.

### 14.4 Trading and market data

A WebSocket-based market data service:

```json
// client → server
{ "jsonrpc": "2.0", "method": "subscribe",
  "params": { "channel": "trades", "symbol": "BTC-USD" }, "id": 1 }

// server → client (notification)
{ "jsonrpc": "2.0", "method": "trade",
  "params": { "symbol": "BTC-USD", "price": 60100, "size": 0.5 } }
```

- Subscribe/unsubscribe via request/response.
- Pushes via notifications.
- Same envelope both directions — clean, symmetric design.

### 14.5 Admin / control planes

Internal tools (admin consoles, ops scripts) where every operation is an action:

```
ops.drainNode
ops.reroute
ops.snapshotDatabase
ops.rotateKey
```

JSON-RPC keeps the surface small, the call shape uniform, and the auth boundary obvious (`/rpc/admin` behind extra auth).

---

## 15. Common Pitfalls

### 15.1 Misusing HTTP status codes
**Mistake:** Returning 400/404/500 for application-level JSON-RPC errors.
**Why it's wrong:** Clients then have *two* error channels (HTTP + JSON-RPC), which they handle inconsistently.
**Fix:** Return **HTTP 200** for any request the server processed (success or business error). Use HTTP status only for transport/auth issues.

### 15.2 Both `result` and `error` (or neither)
**Mistake:** Setting both fields, or returning a response with neither.
**Why it's wrong:** Spec violation; clients can't tell what happened.
**Fix:** Exactly one. Use a sealed response type or factory methods.

### 15.3 Forgetting to echo `id`
**Mistake:** Server returns `id: null` for valid requests.
**Why it's wrong:** Clients can't correlate batched/concurrent calls.
**Fix:** Always echo the request `id`. Only use `id: null` when the server couldn't parse the request.

### 15.4 Treating notifications as guaranteed
**Mistake:** Using a notification for a payment or other critical mutation.
**Why it's wrong:** No response means no confirmation. If the call fails (network, server crash), neither side knows.
**Fix:** Notifications are for telemetry and idempotent fire-and-forget signals only.

### 15.5 Reusing or omitting unique IDs
**Mistake:** Hardcoding `"id": 1` everywhere or reusing IDs in batches.
**Why it's wrong:** Correlation breaks; the wrong response gets matched to the wrong caller.
**Fix:** UUID/ULID per call.

### 15.6 Method names that leak internals
**Mistake:** `UserServiceImpl.findByIdInternal` as a method name.
**Why it's wrong:** Method names are part of the public contract; refactors now require breaking changes.
**Fix:** Use stable, domain-language names: `user.getById`.

### 15.7 Positional params in production
**Mistake:** `"params": [42, "USD", true]`.
**Why it's wrong:** Adding/reordering arguments silently breaks every client.
**Fix:** Named params `{"userId": 42, "currency": "USD", "active": true}`.

### 15.8 Overusing batch requests
**Mistake:** Wrapping every page load's calls into a single 100-element batch.
**Why it's wrong:** The slowest call defines the response latency; partial failures are hard to handle; server resource spikes.
**Fix:** Cap batches (e.g., 20–50). Use parallel HTTP/2 requests instead when calls are independent and have different SLAs.

### 15.9 Leaky error messages
**Mistake:** `"message": "java.sql.SQLException at org.postgresql..."`.
**Why it's wrong:** Information disclosure; brittle clients matching on text.
**Fix:** Stable error codes + curated messages. Stack traces go to logs, not responses.

### 15.10 Skipping versioning
**Mistake:** Renaming a method or changing a param shape without a new version.
**Why it's wrong:** Silent client breakage; impossible rollbacks.
**Fix:** Endpoint or namespace versioning + additive evolution within a version.

### 15.11 No rate limiting / no payload caps
**Mistake:** Trusting clients to behave.
**Why it's wrong:** A single client (or attacker) can exhaust your JSON parser, DB, or thread pool.
**Fix:** Per-identity rate limits, max body size, max JSON depth, max batch size.

### 15.12 Choosing JSON-RPC for a public API
**Mistake:** Exposing a JSON-RPC API to thousands of unknown third-party developers.
**Why it's wrong:** They expect REST conventions, OpenAPI, browser-friendly URLs.
**Fix:** Use REST for public APIs; reserve JSON-RPC for internal or specialized (blockchain, RPC-native) consumers.

---

## 16. Final Checklist

**You are confident using JSON-RPC if you can:**

- Explain in one sentence what RPC is and how it differs from REST in mental model.
- Describe every field of a JSON-RPC 2.0 request, response, and error object — and what's allowed/forbidden in each.
- Distinguish a notification from a regular request and explain when each is appropriate (and when not).
- Decide between positional and named params with a clear rationale, and default to named in production.
- Use the standard error codes correctly and design a stable, documented application-level error code range.
- Explain why JSON-RPC errors live in the response body at HTTP 200, and when HTTP status codes apply instead.
- Build a JSON-RPC server in Spring Boot with one endpoint, a dispatcher, auto-registered handlers, and centralized error mapping.
- Build a JSON-RPC client (sync with `RestClient`, reactive with `WebClient`) that handles success, business errors, and notifications.
- Design and version a JSON-RPC API with backward-compatible evolution rules.
- Choose JSON-RPC vs REST vs gRPC for a given service, citing concrete trade-offs.
- Secure a JSON-RPC service with TLS, JWT/API keys, method-level authorization, input validation, and payload limits.
- Instrument a JSON-RPC service with structured logs, Micrometer metrics, and OpenTelemetry traces tagged by method and code.
- Use batching responsibly — knowing exactly when it helps and when it hurts.
- Identify the common pitfalls (HTTP status misuse, positional params, leaky errors, missing versioning) and avoid them by reflex.
- Point at three real-world JSON-RPC systems (Ethereum nodes, LSP, internal control planes) and explain why JSON-RPC fits each.

If you can do all of the above without looking anything up, you are ready to design, implement, and run JSON-RPC services in production.
