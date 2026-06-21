# Mastering REST, HTTP, Postman, and Web API Debugging with Spring Boot

> A complete learning path from networking fundamentals to production-grade REST API design, debugging, and troubleshooting.
>
> **Audience:** A Java Spring Boot developer with ~3 years of experience who can build REST controllers, knows HTTP methods and status codes at a basic level, uses Postman, debugs Spring Boot apps, and works in microservices — and who now wants *deep, senior-level mastery*.

---

## How to read this guide

This is **not** a cheat sheet and **not** API documentation. Every major concept follows a six-step learning structure so that you build *mental models*, not just memorize facts:

1. **Problem Statement** — why the concept exists, what existed before.
2. **Intuition** — the simple mental model.
3. **Real-World Analogy** — banking, e-commerce, payments, enterprise systems.
4. **Internal Mechanics** — components, data flow, processing steps.
5. **Practical Example** — HTTP, Spring Boot, Postman, debugging.
6. **Common Mistakes** — wrong approaches, misconceptions, production issues.

Read it front to back once. Then keep it as a reference. By the end you should be able to design professional REST APIs, debug *any* API communication problem, and explain HTTP internals confidently in a senior interview.

---

## Table of Contents

1. [Part I — What Happens When a Browser Calls an API](#part-i)
2. [Part II — TCP, TLS, and the HTTP Connection](#part-ii)
3. [Part III — HTTP Internals: The Anatomy of a Request and Response](#part-iii)
4. [Part IV — REST: What It Really Is](#part-iv)
5. [Part V — HTTP Methods Deep Dive](#part-v)
6. [Part VI — HTTP Headers Deep Dive](#part-vi)
7. [Part VII — HTTP Status Codes Deep Dive](#part-vii)
8. [Part VIII — The Spring Boot Request Lifecycle](#part-viii)
9. [Part IX — REST API Design Masterclass](#part-ix)
10. [Part X — Postman Complete Guide](#part-x)
11. [Part XI — Chrome DevTools Masterclass](#part-xi)
12. [Part XII — The API Debugging Playbook](#part-xii)
13. [Part XIII — Microservices HTTP Communication](#part-xiii)
14. [Part XIV — Production Debugging](#part-xiv)
15. [Part XV — Senior Interview Preparation](#part-xv)

---

<a name="part-i"></a>
# Part I — What Happens When a Browser Calls an API

This is the single most important question in this entire guide, and one of the most common senior interview questions. We will return to it many times with increasing depth. Let's start with the full picture.

### Step 1 — Problem Statement

When you type `https://api.company.com/users` into a browser, or when JavaScript runs `fetch('/api/users')`, a *staggering* amount of machinery activates. Most developers treat this as magic. A senior engineer treats it as a sequence of well-understood, debuggable steps.

The reason you must understand this: **every API bug lives somewhere in this chain.** If you don't know the chain, you guess. If you know the chain, you bisect.

### Step 2 — Intuition

Think of it as a relay race. The request is a baton passed through many runners:

```
Browser → DNS → TCP → TLS → HTTP request → Load Balancer/Proxy →
Web Server (Tomcat) → Servlet Filters → DispatcherServlet →
Controller → Service → Repository → Database → ... and all the way back
```

Each runner can drop the baton. Debugging is figuring out *which runner* dropped it.

### Step 3 — Real-World Analogy

Imagine sending a registered letter from your office in New York to a bank branch in Tokyo:

- **DNS** = looking up the branch's street address from its name.
- **TCP** = establishing a reliable courier route and confirming the courier on both ends are ready.
- **TLS** = sealing the letter in a tamper-proof, encrypted envelope only the recipient can open.
- **HTTP** = the actual letter, written in a format the recipient understands ("Please give me the account list").
- **Load balancer** = the bank's mailroom that decides which clerk handles it.
- **Spring Boot** = the clerk who reads the letter, pulls records, and writes a reply.

### Step 4 — Internal Mechanics: The Full Journey

Here is the complete sequence when a browser requests `https://api.company.com/users`:

**1. URL parsing.** The browser breaks the URL into scheme (`https`), host (`api.company.com`), port (implicit `443`), path (`/users`), query string, and fragment.

**2. HSTS / cache checks.** The browser checks its HSTS list (should this host *always* be HTTPS?), its own cache (do I already have a fresh copy?), and cookies to attach.

**3. DNS resolution.** The browser needs an IP address for `api.company.com`. It checks (in order): browser DNS cache → OS cache → `hosts` file → the configured DNS resolver (often your router → ISP → recursive resolver). The resolver walks the DNS hierarchy: root servers → `.com` TLD servers → `company.com` authoritative servers → returns e.g. `203.0.113.10`.

**4. TCP connection.** The browser opens a TCP socket to `203.0.113.10:443` via the three-way handshake (SYN → SYN-ACK → ACK). Now there is a reliable byte pipe.

**5. TLS handshake.** Over that pipe, the browser and server negotiate encryption: agree on TLS version and cipher, the server presents its certificate, the browser validates it against trusted Certificate Authorities, and they derive shared session keys. After this, everything is encrypted.

**6. HTTP request sent.** The browser writes an HTTP request (request line + headers + optional body) into the encrypted tunnel.

**7. Infrastructure hops.** The request often passes through a CDN, a load balancer, a reverse proxy (NGINX), and an API gateway before it ever reaches your app. Each may add/strip headers, terminate TLS, route, rate-limit, or authenticate.

**8. Embedded Tomcat.** Your Spring Boot app's embedded servlet container (Tomcat by default) accepts the connection, parses the raw bytes into an `HttpServletRequest`.

**9. Servlet filter chain.** Filters run (security, CORS, logging, encoding).

**10. DispatcherServlet.** Spring's front controller receives the request, finds the matching handler (your controller method), runs interceptors, resolves arguments, deserializes JSON via Jackson.

**11. Controller → Service → Repository → Database.** Your business logic runs; the repository queries the DB.

**12. Response built.** The return value is serialized to JSON, headers and status set, and the response travels back out through the same chain — repository to controller to converter to DispatcherServlet to filters to Tomcat to the encrypted TLS tunnel to TCP packets across the network to the browser.

**13. Browser processing.** The browser reads the status, parses the body, runs your `fetch().then()` callback, and (for documents) renders.

### Step 5 — Practical Example

In Chrome DevTools → Network tab, click any request and look at the **Timing** tab. You'll literally see this chain as phases:

```
Queued / Stalled
DNS Lookup          ← step 3
Initial connection  ← step 4 (TCP)
SSL                 ← step 5 (TLS)
Request sent        ← step 6
Waiting (TTFB)      ← steps 7–12 (server processing)
Content Download    ← step 13
```

When an API is "slow," this breakdown tells you *which step* is slow. Slow DNS, slow TLS, or slow TTFB are three completely different problems with three different fixes.

### Step 6 — Common Mistakes

- **Assuming "the API is slow" means the server code is slow.** It might be DNS, TLS renegotiation, or a saturated load balancer. Read the timing breakdown first.
- **Forgetting infrastructure exists.** "It works on my machine" often means your machine skips the CDN/gateway/proxy that production has. Many bugs (stripped headers, CORS, timeouts) live in that layer.
- **Believing the browser and Postman do the same thing.** They don't — the browser enforces CORS, sends cookies/origin automatically, and respects HSTS. (Whole section on this later.)

---

<a name="part-ii"></a>
# Part II — TCP, TLS, and the HTTP Connection

### Step 1 — Problem Statement

HTTP is a set of *rules for a conversation*, but it needs a reliable channel to talk over. Raw networks lose packets, reorder them, and corrupt them. And the internet is hostile — anyone on the path can read or tamper with plaintext. We need (a) reliability and (b) confidentiality + integrity + authenticity. TCP solves the first; TLS solves the second.

### Step 2 — Intuition

- **IP** delivers individual packets, best-effort, no guarantees.
- **TCP** sits on top of IP and turns unreliable packets into a reliable, ordered byte stream (like a phone call where nothing is dropped and words arrive in order).
- **TLS** sits on top of TCP and wraps that stream in encryption.
- **HTTP** sits on top of TLS and is the actual language spoken.

```
HTTP   (the language)
 TLS   (the sealed envelope)
 TCP   (the reliable pipe)
 IP    (the postal packets)
```

### Step 3 — Real-World Analogy

A secure phone call to your bank:
- **TCP** = dialing and both sides confirming "Can you hear me? Yes." before talking.
- **TLS** = both of you switching to a scrambler device only you two can unscramble, after the bank proves its identity with a verified caller-ID.
- **HTTP** = the actual conversation: "I'd like my balance." "Here it is."

### Step 4 — Internal Mechanics

#### The TCP three-way handshake

Before any HTTP byte is sent, TCP establishes the connection:

```
Client → Server:  SYN          (I want to talk; here's my sequence number)
Server → Client:  SYN-ACK      (OK; I acknowledge yours; here's mine)
Client → Server:  ACK          (I acknowledge yours; let's go)
```

Three messages = one round trip before data. This is why opening many connections is expensive, and why connection reuse (keep-alive) matters.

#### The TLS handshake (TLS 1.2 simplified)

```
Client → Server:  ClientHello   (TLS versions I support, cipher suites, random)
Server → Client:  ServerHello   (chosen version + cipher), Certificate,
                                 ServerKeyExchange, ServerHelloDone
Client → Server:  ClientKeyExchange (key material), ChangeCipherSpec, Finished
Server → Client:  ChangeCipherSpec, Finished
```

Key points:
- The server's **certificate** contains its public key and is signed by a **Certificate Authority (CA)**.
- The client validates the certificate: Is it signed by a CA I trust? Is the hostname correct? Is it expired? Is it revoked?
- Both sides derive a shared **session key** used for fast symmetric encryption of the actual data.
- **TLS 1.3** reduced this to effectively *one* round trip (and 0-RTT for resumption), which is a major latency win.

#### Certificates and validation — what actually gets checked

1. **Chain of trust:** server cert → intermediate CA → root CA in the OS/browser trust store.
2. **Hostname match:** the cert's Subject Alternative Names must include the hostname you connected to.
3. **Validity period:** `notBefore`/`notAfter` dates.
4. **Revocation:** via CRL or OCSP.

When any of these fail, the browser shows `NET::ERR_CERT_*` and **refuses to connect**. This is the single most common cause of "Postman works but browser fails" being *inverted* — sometimes Postman fails because of strict SSL while the browser has the corporate root CA installed, or vice versa.

#### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| Transport | TCP | TCP | **QUIC over UDP** |
| Format | Text | Binary frames | Binary frames |
| Multiplexing | No (one request per connection at a time) | **Yes** (many streams over one connection) | Yes |
| Head-of-line blocking | At HTTP level | At TCP level (one lost packet stalls all streams) | **Eliminated** (per-stream) |
| Header compression | None | HPACK | QPACK |
| Connection setup | TCP + TLS | TCP + TLS | **0–1 RTT (QUIC integrates TLS 1.3)** |

- **HTTP/1.1** opens multiple TCP connections to parallelize and suffers *head-of-line blocking*: request 2 waits for request 1 on the same connection.
- **HTTP/2** multiplexes many logical streams over one TCP connection — huge win — but a single lost TCP packet still stalls all streams (TCP-level HOL blocking).
- **HTTP/3** moves to QUIC (over UDP), giving independent streams so one lost packet only stalls its own stream, plus faster handshakes.

### Step 5 — Practical Example

What happens internally when `https://api.company.com/users` is called (connection layer):

```
1. DNS:  api.company.com → 203.0.113.10
2. TCP:  SYN / SYN-ACK / ACK to 203.0.113.10:443
3. TLS:  ClientHello → ServerHello + Certificate → key exchange → Finished
         (browser validates cert chain, hostname, expiry)
4. HTTP: encrypted "GET /users HTTP/1.1 ... " sent through the tunnel
```

Inspect it yourself:

```bash
# See the full TLS handshake and certificate chain:
openssl s_client -connect api.company.com:443 -servername api.company.com

# See timing breakdown of each phase with curl:
curl -w "dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" \
     -o /dev/null -s https://api.company.com/users
```

### Step 6 — Common Mistakes

- **Thinking HTTPS only "encrypts."** It also *authenticates* the server (you're really talking to the bank) and ensures *integrity* (nobody altered the bytes).
- **Disabling certificate validation** ("trust all certs") in production code to "fix" an SSL error. This silently destroys the security guarantee and is a classic audit finding. Fix the trust store instead.
- **Ignoring keep-alive.** Re-doing TCP+TLS per request is brutal for latency. Reuse connections (it's automatic in browsers and in Spring's `WebClient`/pooled `RestTemplate`).
- **Mixed content:** an HTTPS page calling an HTTP API. The browser blocks it. (More in DevTools Security section.)

---

<a name="part-iii"></a>
# Part III — HTTP Internals: The Anatomy of a Request and Response

### Step 1 — Problem Statement

To debug APIs you must be able to read *raw* HTTP, not just the pretty version Postman shows you. Every header, every byte, has meaning. When something breaks, the raw message is the source of truth.

### Step 2 — Intuition

An HTTP message is just **text** (in HTTP/1.1) with a strict structure:

```
<start-line>
<headers>
<blank line>
<optional body>
```

The blank line is sacred — it separates headers from body. That's the whole format.

### Step 3 — Real-World Analogy

A request is a formal business letter:
- **Request line** = the subject line ("Re: Request for user list").
- **Headers** = the metadata at the top (To, From, Date, "Confidential", preferred reply language).
- **Blank line** = the visual gap before the letter body.
- **Body** = the actual content/attachment.

### Step 4 — Internal Mechanics

#### A raw HTTP request, dissected

```http
GET /api/users HTTP/1.1
Host: example.com
Authorization: Bearer eyJhbGciOiJIUzI1Ni9...
Accept: application/json
```

Line by line:

- **`GET`** — the **method** (verb). What action to perform.
- **`/api/users`** — the **request target** (path + optional query string). Which resource.
- **`HTTP/1.1`** — the **protocol version**. Together these three form the **request line**.
- **`Host: example.com`** — mandatory in HTTP/1.1. One IP can host many sites; this says *which* virtual host you mean.
- **`Authorization: Bearer <token>`** — credentials proving who you are.
- **`Accept: application/json`** — content negotiation: "I want the response as JSON."
- The **blank line** after the headers (here implied) ends the header section. A GET has no body.

A request *with* a body (POST):

```http
POST /api/users HTTP/1.1
Host: example.com
Content-Type: application/json
Content-Length: 27

{"name":"John","age":30}
```

Note `Content-Type` (what the body *is*) and `Content-Length` (how many bytes), then the blank line, then the body.

#### A raw HTTP response, dissected

```http
HTTP/1.1 201 Created
Date: Sat, 20 Jun 2026 12:00:00 GMT
Content-Type: application/json
Content-Length: 52
Location: /api/users/42

{"id":42,"name":"John","age":30,"status":"ACTIVE"}
```

- **`HTTP/1.1 201 Created`** — the **status line**: protocol version, status code, reason phrase.
- **Response headers** — metadata about the response (`Date`, `Content-Type`, `Location`).
- **Blank line** then **body** — the representation of the resource.

### Step 5 — Practical Example

See raw HTTP yourself:

```bash
# -v shows the request and response lines + headers (> is sent, < is received)
curl -v https://example.com/api/users

# Even rawer: send bytes by hand
printf 'GET /api/users HTTP/1.1\r\nHost: example.com\r\nAccept: application/json\r\nConnection: close\r\n\r\n' \
  | openssl s_client -quiet -connect example.com:443
```

In Spring Boot, you can log raw requests/responses with a filter:

```java
@Bean
public CommonsRequestLoggingFilter requestLoggingFilter() {
    var filter = new CommonsRequestLoggingFilter();
    filter.setIncludeQueryString(true);
    filter.setIncludeHeaders(true);
    filter.setIncludePayload(true);
    filter.setMaxPayloadLength(10000);
    return filter;
}
```

### Step 6 — Common Mistakes

- **Sending a body with GET.** Allowed by the spec to exist but semantically meaningless and dropped/ignored by many proxies and the Spring stack. Use query params for GET filtering.
- **Wrong line endings.** HTTP uses CRLF (`\r\n`), not just `\n`. Rarely an issue with tools, but matters when hand-crafting.
- **Forgetting `Content-Length` / chunked encoding** when writing low-level clients — the server can't tell where the body ends.
- **Confusing path vs query vs fragment.** The fragment (`#section`) is *never* sent to the server — it's browser-only.

---

<a name="part-iv"></a>
# Part IV — REST: What It Really Is

### Step 1 — Problem Statement

Before REST, distributed systems often used **SOAP**: heavyweight XML envelopes, rigid WSDL contracts, tunneling everything through `POST`, and ignoring HTTP's own semantics. It worked but was verbose, tightly coupled, and hard to cache or scale.

In 2000, Roy Fielding's doctoral dissertation described **REST (Representational State Transfer)** — an *architectural style* that describes *why the web itself scaled to planetary size*. The insight: stop fighting HTTP, and *use HTTP as designed*.

### Step 2 — Intuition

REST is not a protocol or a library. It's a **style** — a set of constraints. The core idea:

> Model your system as **resources** (nouns), each addressable by a **URL**, manipulated through a **uniform set of methods** (HTTP verbs), exchanging **representations** (JSON/XML), with **stateless** communication.

You don't invent `getUser`, `deleteUser`, `updateUserStatus` operations. You have *one* resource `/users/42` and you `GET`, `PUT`, `PATCH`, `DELETE` it. The verbs are already standardized by HTTP.

### Step 3 — Real-World Analogy

Think of a library:
- Each **book** is a resource with a permanent shelf location (URL): `/books/978-0-13-468599-1`.
- The **actions** are universal and limited: borrow (read), return, reserve, remove. You don't need a different *kind of action* per book.
- A **representation** is a photocopy or summary of the book — not the book itself. You get the *state*, transferred to you. Hence "Representational State Transfer."

### Step 4 — Internal Mechanics: The REST Constraints

Fielding defined six constraints. A system is "RESTful" to the degree it honors them:

1. **Client–Server** — separate the UI (client) from data storage (server). They evolve independently.
2. **Stateless** — *each request contains everything needed to process it.* The server stores no client session state between requests. Authentication travels on every request (e.g., a Bearer token). This is what lets you run 50 identical backend instances behind a load balancer.
3. **Cacheable** — responses declare whether they can be cached (`Cache-Control`, `ETag`). The web's scale depends on caching.
4. **Uniform Interface** — the heart of REST: resources identified by URLs, manipulated through representations, self-descriptive messages, and HATEOAS (links in responses). This uniformity decouples client and server.
5. **Layered System** — clients can't tell if they're talking to the real server or a proxy/cache/gateway. Enables load balancers, CDNs, gateways.
6. **Code on Demand** (optional) — servers can send executable code (e.g., JavaScript).

#### Resource-oriented design

The shift in thinking: **nouns, not verbs**.

| Operation | RPC style (wrong for REST) | REST style |
|---|---|---|
| List users | `GET /getAllUsers` | `GET /users` |
| Get one | `GET /getUser?id=42` | `GET /users/42` |
| Create | `POST /createUser` | `POST /users` |
| Update | `POST /updateUser` | `PUT /users/42` |
| Delete | `POST /deleteUser?id=42` | `DELETE /users/42` |

The URL identifies the *resource*; the *method* identifies the action.

#### Statelessness in practice

```
Request 1: GET /cart       Authorization: Bearer abc   → server reads token, knows user
Request 2: POST /cart/items Authorization: Bearer abc   → server reads token AGAIN
```

The server doesn't "remember" you from request 1. Each request re-establishes identity. This is *why* tokens exist and *why* you send them every time.

#### Idempotency

An operation is **idempotent** if performing it once or many times yields the same *server state*. Critical for retries in unreliable networks.

- `GET`, `PUT`, `DELETE`, `HEAD`, `OPTIONS` → idempotent.
- `POST`, `PATCH` → generally *not* idempotent.

Why it matters: if a client sends `DELETE /users/42` and the network drops the response, the client can safely retry — deleting an already-deleted user just stays deleted. But retrying `POST /users` might create *two* users.

### Step 5 — Practical Example

A well-designed REST resource in Spring Boot:

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @GetMapping                 GET    /api/v1/users        → list
    @GetMapping("/{id}")        GET    /api/v1/users/42     → one
    @PostMapping                POST   /api/v1/users        → create
    @PutMapping("/{id}")        PUT    /api/v1/users/42     → full replace
    @PatchMapping("/{id}")      PATCH  /api/v1/users/42     → partial update
    @DeleteMapping("/{id}")     DELETE /api/v1/users/42     → delete
}
```

#### REST vs SOAP at a glance

| Aspect | SOAP | REST |
|---|---|---|
| Type | Protocol | Architectural style |
| Format | XML only | JSON, XML, anything |
| Transport | HTTP, SMTP, TCP | HTTP only |
| Contract | Rigid (WSDL) | Loose (OpenAPI optional) |
| Uses HTTP verbs | No (everything via POST) | Yes (GET/POST/PUT/...) |
| Caching | Hard | Native (HTTP caching) |
| Overhead | Heavy envelopes | Lightweight |
| State | Can be stateful | Stateless |
| Best for | Formal enterprise, WS-Security, transactions | Web/mobile APIs, microservices |

### Step 6 — Common Mistakes

- **RPC-over-HTTP disguised as REST.** `POST /api/doUserUpdate` with a giant action body is not REST. It's RPC wearing a REST costume.
- **Stateful "REST" APIs** that rely on server-side session affinity (sticky sessions). This breaks horizontal scaling and is fragile.
- **Verbs in URLs** (`/users/42/activate`). Tolerated for genuine *actions* that don't map to CRUD, but overusing it means you've stopped thinking in resources.
- **Treating REST as "JSON over HTTP."** That's necessary but not sufficient. The constraints (statelessness, uniform interface, caching) are what make it REST.
- **Ignoring HTTP semantics** — returning `200 OK` with `{"error": true}` in the body. Use the status codes; they exist for this.

---

<a name="part-v"></a>
# Part V — HTTP Methods Deep Dive

### Step 1 — Problem Statement

HTTP methods are the **verbs** of the uniform interface. Using the wrong one breaks caching, retries, idempotency, and clients' expectations. "It works" is not enough — using `POST` for everything makes your API un-cacheable, un-retryable, and unintelligible to proxies and tooling.

### Step 2 — Intuition

Two properties classify every method:

- **Safe** — does it *modify* server state? Safe methods (`GET`, `HEAD`, `OPTIONS`) only read.
- **Idempotent** — does repeating it change the outcome beyond the first call?

| Method | Safe | Idempotent | Cacheable | Has request body |
|---|---|---|---|---|
| GET | ✅ | ✅ | ✅ | No |
| HEAD | ✅ | ✅ | ✅ | No |
| OPTIONS | ✅ | ✅ | No | No |
| POST | ❌ | ❌ | Rarely | Yes |
| PUT | ❌ | ✅ | No | Yes |
| PATCH | ❌ | ❌ | No | Yes |
| DELETE | ❌ | ✅ | No | Optional |

### Step 3 — Real-World Analogy (banking)

- **GET** = checking your balance. Read-only, repeatable, safe to cache for a moment.
- **POST** = depositing cash at a teller. Do it twice = two deposits. Not idempotent.
- **PUT** = setting your profile to an exact new state ("my address is now X"). Do it twice = same final address. Idempotent.
- **PATCH** = "add $50 to my limit." Twice = +$100. *Not* idempotent (depends on whether the patch is absolute or relative).
- **DELETE** = closing an account. Closing twice = still closed. Idempotent.

---

### GET

- **Purpose:** retrieve a representation of a resource. Read-only.
- **When to use:** fetching data; the request should be expressible entirely as a URL.
- **When NOT to use:** anything that changes state, or anything with sensitive data in the URL (URLs land in logs, browser history, proxies).
- **Idempotency:** idempotent and safe.
- **Cache:** highly cacheable.
- **Real-world example:** `GET /api/v1/orders/123` to view an order.
- **Common mistakes:** putting state-changing logic behind GET (a prefetcher or crawler will trigger it!); cramming secrets into query strings; relying on a request body.

```java
@GetMapping("/{id}")
public OrderDto getOrder(@PathVariable Long id) {
    return orderService.findById(id);   // never mutate here
}
```

### POST

- **Purpose:** create a new resource, or trigger a non-idempotent process.
- **When to use:** creating entities (`POST /users`), submitting forms, operations that aren't safely repeatable.
- **When NOT to use:** as a lazy default for reads or updates.
- **Idempotency:** *not* idempotent. Retrying may create duplicates.
- **Cache:** generally not cached.
- **Real-world example:** `POST /api/v1/payments` to charge a card.
- **Common mistakes:** double-charging because the client retried a POST after a timeout. Fix with an **idempotency key** (client sends `Idempotency-Key: <uuid>`; server deduplicates). Also: returning `200` instead of `201 Created` with a `Location` header.

```java
@PostMapping
public ResponseEntity<UserDto> create(@RequestBody @Valid CreateUserRequest req) {
    UserDto created = userService.create(req);
    return ResponseEntity
        .created(URI.create("/api/v1/users/" + created.id()))  // 201 + Location
        .body(created);
}
```

### PUT

- **Purpose:** create-or-replace a resource at a *known* URL with the *full* representation.
- **When to use:** when the client controls the ID and sends the complete new state.
- **When NOT to use:** partial updates (use PATCH) or when the server assigns IDs (use POST).
- **Idempotency:** idempotent — replacing with the same body repeatedly yields the same state.
- **Cache:** not cacheable.
- **Real-world example:** `PUT /api/v1/users/42` with the entire updated user object.
- **Common mistakes:** treating PUT as partial update — fields omitted from the body should be *cleared/reset* under strict PUT semantics, which surprises clients who expected them preserved.

```java
@PutMapping("/{id}")
public UserDto replace(@PathVariable Long id, @RequestBody @Valid UserDto full) {
    return userService.replace(id, full);  // entire resource replaced
}
```

### PATCH

- **Purpose:** apply a *partial* modification.
- **When to use:** updating a subset of fields (`{"status":"INACTIVE"}`).
- **When NOT to use:** when you intend a full replacement (PUT) or creation (POST).
- **Idempotency:** *not guaranteed*. A patch like `{"op":"increment","value":1}` is not idempotent; a patch like `{"status":"INACTIVE"}` happens to be. Don't rely on it.
- **Cache:** not cacheable.
- **Real-world example:** `PATCH /api/v1/users/42` with just `{"email":"new@x.com"}`.
- **Common mistakes:** ambiguity between "field is null" (set to null) and "field is absent" (leave unchanged). With Jackson, distinguish using `JsonNullable`/`Optional` or a `Map<String,Object>` so absent fields aren't accidentally nulled.

### DELETE

- **Purpose:** remove a resource.
- **When to use:** deleting an entity.
- **Idempotency:** idempotent — deleting twice leaves it deleted. The *second* call commonly returns `404` or `204`; both are acceptable, just be consistent.
- **Cache:** not cacheable.
- **Real-world example:** `DELETE /api/v1/users/42`.
- **Common mistakes:** returning `200` with a body when `204 No Content` is cleaner; or hard-deleting when the business needs *soft delete* (audit trails). In enterprises, "delete" is often a status change.

### OPTIONS

- **Purpose:** ask what's allowed on a resource; **the engine of CORS preflight**.
- **When to use:** mostly automatic — browsers send it before "non-simple" cross-origin requests.
- **Idempotency:** idempotent and safe.
- **Real-world example:** before `PUT /api/v1/users/42` from a different origin, the browser sends `OPTIONS /api/v1/users/42` to check permissions.
- **Common mistakes:** your backend not handling OPTIONS (returns 401/404), so the *preflight* fails and the browser never sends the real request — a classic "works in Postman, fails in browser" bug.

### HEAD

- **Purpose:** like GET but returns *only headers, no body*.
- **When to use:** checking existence, size (`Content-Length`), or freshness (`ETag`/`Last-Modified`) cheaply; checking if a large download is worth starting.
- **Idempotency:** idempotent and safe.
- **Real-world example:** `HEAD /api/v1/files/big.zip` to read `Content-Length` before downloading.
- **Common mistakes:** assuming HEAD is rarely used — health checks, link checkers, and caches use it constantly. Spring maps HEAD to your `@GetMapping` automatically and strips the body.

### Step 6 — Method-level Common Mistakes (summary)

- Using `POST` for reads (un-cacheable, breaks GET semantics).
- Using `GET` for writes (crawlers/prefetch trigger side effects).
- Assuming `PATCH` is idempotent.
- Not supporting `OPTIONS` → broken CORS.
- Returning `200` for creates and deletes instead of `201`/`204`.

---

<a name="part-vi"></a>
# Part VI — HTTP Headers Deep Dive

### Step 1 — Problem Statement

The request line says *what* and *where*. But you also need to communicate *how*: what format, who you are, how to cache, what language, security policies. That metadata is **headers**. Most "weird" API behavior — caching, auth, CORS, encoding — is governed by headers most developers never look at.

### Step 2 — Intuition

Headers are **key–value metadata** attached to every request and response. They don't carry the business payload; they carry instructions *about* the payload and the conversation. Categories:

- **General** — apply to the whole message (`Date`, `Connection`).
- **Request** — info about the client/request (`Host`, `User-Agent`, `Accept`).
- **Response** — info about the server/response (`Server`, `Set-Cookie`).
- **Representation/Entity** — describe the body (`Content-Type`, `Content-Length`, `ETag`).
- **Security** — policies (`Strict-Transport-Security`, `Content-Security-Policy`).

### Step 3 — Real-World Analogy

Headers are the **shipping label and customs form** on a package. The box (body) holds the product. The label says: who it's for (`Host`), what's inside (`Content-Type`), how heavy (`Content-Length`), fragile/handling instructions (`Cache-Control`), sender ID (`Authorization`), and language of the manual (`Accept-Language`).

### Step 4 — Internal Mechanics + Step 5 — Practical Examples

#### General headers

- **`Host: api.company.com`** — *mandatory* in HTTP/1.1. Selects the virtual host. Without it, a server hosting many domains can't route you.
- **`Connection: keep-alive`** — reuse the TCP connection for more requests (default in HTTP/1.1). `close` ends it after this response.
- **`Date: Sat, 20 Jun 2026 12:00:00 GMT`** — when the message was generated.
- **`Cache-Control: no-cache, max-age=3600`** — the master caching directive (details below).
- **`Pragma: no-cache`** — legacy HTTP/1.0 caching control, kept for backward compatibility; `Cache-Control` supersedes it.

#### Authentication headers

- **`Authorization: Bearer <token>`** — the dominant scheme. The "bearer" of the token is granted access — so protect it. Typically a **JWT**.
- **`Authorization: Basic <base64(user:pass)>`** — Basic auth. Base64 is *encoding, not encryption* — trivially reversible. Only safe over HTTPS.
- **API keys** — often `Authorization: ApiKey <key>` or a custom header like `X-API-Key: <key>`. Identifies the *application*, not necessarily a user.

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiI0MiIsInJvbGUiOiJBRE1JTiJ9.sig
```

A JWT has three dot-separated parts: header, payload (claims like `sub`, `role`, `exp`), and signature. The server verifies the signature — it doesn't need to call a database to trust it (that's why JWT scales statelessly).

#### Content headers (content negotiation)

- **`Content-Type: application/json; charset=utf-8`** — what the body *is*. Drives Spring's choice of message converter. Send JSON but label it `text/plain` → Spring won't deserialize it (`415 Unsupported Media Type`).
- **`Content-Length: 348`** — body size in bytes. Lets the receiver know when the body ends.
- **`Accept: application/json`** — what the client *wants back*. Server picks a representation accordingly.
- **`Accept-Encoding: gzip, br`** — compression the client supports. Server may respond `Content-Encoding: gzip`.
- **`Accept-Language: en-US,en;q=0.9`** — preferred languages, with quality weights (`q`).

#### Browser-related headers (set automatically by browsers)

- **`Origin: https://app.company.com`** — scheme+host+port of the page making the request. Central to CORS. *Cannot be spoofed by JS* — that's the point.
- **`Referer: https://app.company.com/dashboard`** — the page the request came from (misspelled in the spec forever).
- **`User-Agent: Mozilla/5.0 ...`** — client software identity. Used for analytics, feature detection, sometimes (poorly) for blocking.
- **`Cookie: sessionId=abc; theme=dark`** — sends stored cookies back to the server.
- **`Set-Cookie: sessionId=abc; HttpOnly; Secure; SameSite=Strict`** — *response* header instructing the browser to store a cookie. `HttpOnly` hides it from JS (XSS defense); `Secure` = HTTPS only; `SameSite` controls cross-site sending (CSRF defense).

#### Caching headers

- **`Cache-Control: max-age=3600, public`** — cache for 3600s; `public` = any cache, `private` = browser only, `no-store` = never cache.
- **`ETag: "v23-abc"`** — an opaque version fingerprint of the resource.
- **`If-None-Match: "v23-abc"`** — client sends its ETag; if unchanged, server returns **`304 Not Modified`** with no body (bandwidth saved).
- **`Last-Modified: Sat, 20 Jun 2026 10:00:00 GMT`** + **`If-Modified-Since`** — time-based version of the same dance.
- **`Expires: Sat, 20 Jun 2026 13:00:00 GMT`** — absolute expiry (legacy; `Cache-Control: max-age` wins if both present).

**Conditional request flow:**
```
1. GET /users/42            → 200, ETag: "v23"
2. (later) GET /users/42    with If-None-Match: "v23"
3a. unchanged → 304 Not Modified (empty body, client reuses cached copy)
3b. changed   → 200 + new body + new ETag
```

#### Security headers (response)

- **`Strict-Transport-Security: max-age=31536000`** — force HTTPS for this host (HSTS).
- **`Content-Security-Policy: default-src 'self'`** — restrict where resources can load from (XSS mitigation).
- **`X-Content-Type-Options: nosniff`** — don't let the browser guess content types.
- **`X-Frame-Options: DENY`** — prevent clickjacking via iframes.

#### CORS headers — and *why browsers block requests*

This deserves its own focus because it's the #1 "works in Postman, fails in browser" cause.

**Why CORS exists (Step 1 — Problem):** The browser enforces the **Same-Origin Policy**: by default, JavaScript on `https://app.company.com` may **not** read responses from a *different* origin (`https://api.other.com`). Why? Because your browser is logged into your bank. If any website's JS could silently call `bank.com/transfer` *with your cookies* and read the result, the web would be unusable. SOP is a sandbox.

**Intuition (Step 2):** CORS is the server's way of saying *"I explicitly allow this other origin to read my responses."* The browser is the enforcer; the server is the policy author. CORS *loosens* SOP in a controlled way.

**Mechanics (Step 4):**

The key response headers:
- **`Access-Control-Allow-Origin: https://app.company.com`** — which origin(s) may read responses. `*` allows all (but not with credentials).
- **`Access-Control-Allow-Methods: GET, POST, PUT, DELETE`** — allowed methods for cross-origin calls.
- **`Access-Control-Allow-Headers: Authorization, Content-Type`** — which custom request headers are permitted.
- **`Access-Control-Allow-Credentials: true`** — allow cookies/Authorization to be sent cross-origin. When `true`, `Allow-Origin` **cannot** be `*` — it must name an exact origin.

**Simple vs preflighted requests:**
- A **simple request** (GET/POST/HEAD with only "simple" headers and simple content types) is sent directly; the browser checks `Allow-Origin` on the response and *blocks JS from reading it* if it doesn't match.
- A **preflighted request** (PUT/PATCH/DELETE, or custom headers like `Authorization`, or `Content-Type: application/json`) triggers an automatic **`OPTIONS` preflight** *first*:

```http
OPTIONS /api/users HTTP/1.1
Origin: https://app.company.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: authorization, content-type
```
Server must answer:
```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.company.com
Access-Control-Allow-Methods: POST
Access-Control-Allow-Headers: authorization, content-type
Access-Control-Allow-Credentials: true
```
Only if the preflight passes does the browser send the real POST.

**Spring Boot CORS config:**

```java
@Configuration
public class CorsConfig implements WebMvcConfigurer {
    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("https://app.company.com")
            .allowedMethods("GET","POST","PUT","PATCH","DELETE","OPTIONS")
            .allowedHeaders("Authorization","Content-Type")
            .allowCredentials(true)
            .maxAge(3600);   // cache preflight result for 1h
    }
}
```

With Spring Security, also enable `http.cors()` or the filter order will let Security reject the preflight before CORS runs.

### Step 6 — Common Mistakes

- **Mislabeling `Content-Type`** → `415` or silent deserialization failure.
- **Sending Basic auth over HTTP** → credentials in plaintext.
- **`Access-Control-Allow-Origin: *` with credentials** → browser rejects; must name the origin.
- **Forgetting the preflight needs to bypass auth.** If Spring Security 401s the `OPTIONS` request, CORS dies. Permit OPTIONS.
- **Logging `Authorization` headers** → tokens leak into log aggregators. Mask them.
- **Assuming the browser "blocks the request."** Often the request *is sent and processed by the server*; the browser just blocks *JavaScript from reading the response*. Check your server logs — the call may have actually mutated data.

---

<a name="part-vii"></a>
# Part VII — HTTP Status Codes Deep Dive

### Step 1 — Problem Statement

The status code is the *first thing* a client and every piece of infrastructure reads. It's a contract: it tells caches whether to store, clients whether to retry, and engineers where to look. Returning `200 OK` for an error (or `500` for a client mistake) corrupts that contract and misleads everyone downstream.

### Step 2 — Intuition

The first digit is the *category*:

- **1xx** — Informational (rare): "continue, I'm still here."
- **2xx** — Success: "it worked."
- **3xx** — Redirection: "look elsewhere / use your cache."
- **4xx** — Client error: "*you* made a mistake — don't retry unchanged."
- **5xx** — Server error: "*I* failed — retrying might help."

The 4xx/5xx split is the most important distinction in all of API debugging: **whose fault is it?**

### Step 3 — Real-World Analogy (ordering at a restaurant)

- **200** — here's your meal.
- **201** — your reservation is booked (something was created).
- **202** — order received, kitchen will start (async).
- **204** — done, nothing to hand back (you cleared your plate).
- **301/302** — "that counter moved; go to the new one."
- **304** — "your order hasn't changed since last time; reuse it."
- **400** — your order made no sense.
- **401** — you haven't identified yourself.
- **403** — we know who you are, but you're not allowed in the kitchen.
- **404** — there's no such dish.
- **409** — someone took the last table you tried to book.
- **429** — you're ordering too fast; slow down.
- **500** — the kitchen caught fire.
- **503** — kitchen is temporarily closed.
- **504** — the kitchen sent your order to the supplier, who never replied.

### Step 4/5 — Mechanics, Examples, Spring Boot, Debugging

#### 1xx — Informational

- **100 Continue** — client sent `Expect: 100-continue` with a big upload; server says "go ahead, send the body." Saves bandwidth if the server would reject it anyway.

#### 2xx — Success

| Code | Meaning | When | Spring Boot |
|---|---|---|---|
| **200 OK** | Generic success with body | GET, successful PUT/PATCH | default for `@RestController` returns |
| **201 Created** | Resource created | POST that creates | `ResponseEntity.created(uri).body(...)` |
| **202 Accepted** | Accepted, processing async | queued jobs | `ResponseEntity.accepted().body(...)` |
| **204 No Content** | Success, no body | DELETE, PUT with nothing to return | `ResponseEntity.noContent().build()` |

- **201** should include a `Location` header pointing to the new resource.
- **202** is for *asynchronous* work — you accepted the task but haven't finished it. Return a status URL.
- **Debugging:** a POST returning `200` instead of `201`, or a DELETE returning a body instead of `204`, signals sloppy semantics — not fatal, but a smell.

#### 3xx — Redirection

| Code | Meaning | Note |
|---|---|---|
| **301 Moved Permanently** | Resource has a new URL forever | clients/SEO update bookmarks; method may change to GET |
| **302 Found** | Temporary redirect | original URL still valid |
| **304 Not Modified** | Cached copy is still fresh | returned to a conditional request; **no body** |

- **304 mechanics:** the server compares `If-None-Match`/`If-Modified-Since` to the current `ETag`/`Last-Modified`; if equal, returns 304 and the client reuses its cache. Saves bandwidth.
- **Debugging:** unexpected 301→ HTTPS upgrades can turn a `POST` into a `GET` and drop the body. If a POST "loses its body," check for a redirect in the chain.

#### 4xx — Client errors

| Code | Meaning | Real-world example | Spring Boot trigger |
|---|---|---|---|
| **400 Bad Request** | Malformed/invalid request | bad JSON, failed validation | `@Valid` failure, `HttpMessageNotReadableException` |
| **401 Unauthorized** | *Unauthenticated* — who are you? | missing/invalid/expired token | Spring Security entry point |
| **403 Forbidden** | *Authenticated but not allowed* | user lacks role | `@PreAuthorize` denial, `AccessDeniedException` |
| **404 Not Found** | No such resource (or hidden) | wrong URL, wrong context path, deleted entity | no handler / empty repository result |
| **405 Method Not Allowed** | Method not supported on this URL | `DELETE` on a read-only resource | `HttpRequestMethodNotSupportedException` |
| **409 Conflict** | State conflict | duplicate email, optimistic lock | `OptimisticLockException`, unique constraint |
| **415 Unsupported Media Type** | Server can't read the `Content-Type` | sending XML to a JSON-only endpoint | `HttpMediaTypeNotSupportedException` |
| **429 Too Many Requests** | Rate limit exceeded | client hammering API | gateway/bucket limiter; include `Retry-After` |

**401 vs 403 — the classic interview trap:**
- **401** = "I don't know who you are." → *authentication* problem. The fix is to provide valid credentials. Often paired with a `WWW-Authenticate` header.
- **403** = "I know exactly who you are, and you're not allowed." → *authorization* problem. Providing the same credentials again won't help; you need different permissions.

**Spring Boot — clean 400 from validation:**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiError onValidation(MethodArgumentNotValidException ex) {
        var fields = ex.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .toList();
        return new ApiError(Instant.now(), 400, "VALIDATION_ERROR",
                            "Request validation failed", fields, MDC.get("traceId"));
    }
}
```

- **Debugging 400:** read the response body — Spring usually names the offending field. Check JSON validity, required fields, and type mismatches (`"age":"thirty"`).
- **Debugging 401:** is a token present? Inspect it at jwt.io — is `exp` in the past? Is the signing key right? Is the header literally `Authorization: Bearer <token>` (no typos, no "Bearer Bearer")?
- **Debugging 403:** the token is valid but the user's roles/scopes don't grant access. Decode the JWT claims; check `@PreAuthorize` rules.
- **Debugging 404:** verify the *exact* path including `server.servlet.context-path`, trailing slashes, and method. A 404 can also be a deliberately hidden 403 (don't reveal existence).

#### 5xx — Server errors

| Code | Meaning | Real-world example | Where it comes from |
|---|---|---|---|
| **500 Internal Server Error** | Unhandled server exception | NPE, uncaught bug | your code throws something unmapped |
| **502 Bad Gateway** | Upstream returned garbage | proxy got an invalid response from app | load balancer / gateway |
| **503 Service Unavailable** | Temporarily down/overloaded | deploy, no healthy instances, circuit open | LB / app shedding load |
| **504 Gateway Timeout** | Upstream didn't respond in time | downstream service/DB too slow | gateway/proxy timeout |

- **500** is *your* bug. Don't leak stack traces to clients; log them with a `traceId` and return a clean error body.
- **502/503/504** usually originate in **infrastructure**, not your controller. If you see these but your app logs are clean, look at the load balancer, gateway, and downstream dependencies.
- **504 vs 500:** 504 means something *upstream of the responder* timed out waiting (e.g., gateway → your slow service, or your service → a slow DB). The chain matters.

**Spring Boot — mapping a domain exception to a status:**

```java
@ResponseStatus(HttpStatus.CONFLICT)
public class DuplicateEmailException extends RuntimeException { ... }
```

### Step 6 — Common Mistakes

- **`200 OK` with `{"success": false}`.** Breaks caches, monitoring, and clients that check status. Use the right code.
- **`500` for client mistakes.** Validation failures are `400`, not `500`. (And don't let a `NullPointerException` be your validation.)
- **Confusing 401 and 403.** Costs you interviews and confuses clients.
- **Leaking stack traces in 500 bodies.** Security risk + ugly. Return a sanitized body, log the detail with a trace ID.
- **Treating 4xx as retryable.** Retrying a `400`/`401`/`403` unchanged just repeats the failure. Only `429` (with backoff) and most `5xx` are retry candidates.

---

<a name="part-viii"></a>
# Part VIII — The Spring Boot Request Lifecycle

### Step 1 — Problem Statement

You write `@GetMapping` and return an object, and JSON appears. But *how*? When a request 404s, 415s, or your JSON won't bind, the cause lives in this pipeline. To debug Spring APIs you must know the path a request takes from socket to controller and back.

### Step 2 — Intuition

Spring Boot embeds a servlet container (Tomcat). Tomcat hands every request to **one** servlet — Spring's `DispatcherServlet` — which orchestrates everything: finding your method, converting JSON, calling you, converting the result, handling errors.

```
Client → Network → Embedded Tomcat → Servlet Filters → DispatcherServlet
       → HandlerMapping → HandlerInterceptors → ArgumentResolvers (+ Jackson)
       → Your Controller → Service → Repository → Database
       → return value → Message Converters (Jackson) → Interceptors → Filters → Tomcat → Client
```

### Step 3 — Real-World Analogy

`DispatcherServlet` is the **front desk of a large enterprise office**:
- **Filters** = building security at the door (badge check, metal detector) — they see *everyone*, even visitors who'll be turned away.
- **DispatcherServlet** = the receptionist who reads your request and routes you.
- **HandlerMapping** = the directory that finds the right department.
- **Interceptors** = the department's own check-in desk.
- **ArgumentResolvers + Jackson** = a translator who turns your foreign-language form (JSON) into the internal form (Java object) the staff use.
- **Controller/Service/Repository** = the staff who actually do the work.

### Step 4 — Internal Mechanics

#### Filters vs Interceptors — know the difference

- **Filters** (Servlet API) run *outside* Spring MVC, wrapping the entire `DispatcherServlet`. They see the raw `HttpServletRequest`/`Response`. Use for: CORS, security (Spring Security is a filter chain!), request logging, compression, encoding. They run even for requests that never reach a controller.
- **Interceptors** (`HandlerInterceptor`) run *inside* Spring MVC, around the handler. They know *which controller method* will run (the handler object). Use for: auth checks tied to handlers, timing, populating `MDC` for logging. Hooks: `preHandle`, `postHandle`, `afterCompletion`.

```
Filter.doFilter (before)
  └─ DispatcherServlet
       └─ Interceptor.preHandle
            └─ Controller method
       └─ Interceptor.postHandle
  └─ Interceptor.afterCompletion
Filter.doFilter (after)
```

#### The dispatch sequence inside DispatcherServlet

1. **HandlerMapping** matches the request (method + path + headers) to a handler method. No match → `404` (or `405` if path matches but method doesn't).
2. **HandlerAdapter** invokes the handler.
3. **HandlerInterceptor.preHandle** runs; returning false short-circuits.
4. **ArgumentResolvers** build each method parameter: `@PathVariable`, `@RequestParam`, `@RequestHeader`, and `@RequestBody` (which invokes a **message converter**).
5. **Your controller** runs → service → repository → DB.
6. The **return value** is handled: a `@ResponseBody`/`@RestController` return goes to a **message converter** for serialization.
7. **postHandle**, then the response flushes; **afterCompletion** runs last.
8. Any exception is routed to **HandlerExceptionResolver** (e.g., your `@RestControllerAdvice`).

#### How JSON becomes a Java object (and back)

This is the **message converter** layer. For JSON, Spring uses `MappingJackson2HttpMessageConverter`, which wraps Jackson's **`ObjectMapper`**.

**Deserialization (request body → object):**

```
HTTP body:  {"name":"John"}
            │
   Content-Type: application/json  → Spring picks the Jackson converter
            │
   ObjectMapper.readValue(json, User.class)
            │
            ▼
   User user = new User(); user.setName("John");
```

```java
@PostMapping
public UserDto create(@RequestBody CreateUserRequest req) { ... }
// @RequestBody → Jackson reads the body into CreateUserRequest
```

**Serialization (object → response body):**

```
return userDto;
   │
   Accept: application/json → Jackson converter chosen
   │
   ObjectMapper.writeValueAsString(userDto)
   │
   ▼
{"id":42,"name":"John"}
```

**Content negotiation** decides *which* converter: the request's `Accept` header (and the `Content-Type` for the body) selects JSON vs XML vs others.

**Jackson essentials you'll actually hit:**
- `@JsonProperty("user_name")` — map a JSON field name to a Java field.
- `@JsonIgnore` — never serialize a field (e.g., password).
- `@JsonInclude(NON_NULL)` — omit null fields.
- `@JsonFormat` — date/time formatting.
- Unknown JSON fields → by default Jackson *fails* (`FAIL_ON_UNKNOWN_PROPERTIES`); often disabled for tolerant APIs.

### Step 5 — Practical Example: tracing a full request

```java
// 1. Filter sets a correlation ID for every request
@Component
public class TraceIdFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
                                    FilterChain chain) throws ... {
        String traceId = Optional.ofNullable(req.getHeader("X-Trace-Id"))
                                  .orElse(UUID.randomUUID().toString());
        MDC.put("traceId", traceId);
        try { chain.doFilter(req, res); } finally { MDC.clear(); }
    }
}

// 2. Controller — Jackson binds the body, @Valid validates
@PostMapping("/api/v1/users")
public ResponseEntity<UserDto> create(@RequestBody @Valid CreateUserRequest req) {
    UserDto dto = service.create(req);             // 3. service → repository → DB
    return ResponseEntity.created(URI.create("/api/v1/users/" + dto.id())).body(dto);
}

// 4. Errors converted centrally
@RestControllerAdvice
class Handler { /* maps exceptions to clean JSON + status, see Part VII */ }
```

### Step 6 — Common Mistakes

- **Confusing filter and interceptor order**, especially with Spring Security. If Security (a filter) rejects a request before your CORS filter runs, you get mysterious CORS errors on the *preflight*.
- **Doing auth in a controller instead of a filter/interceptor** — easy to forget on a new endpoint.
- **`FAIL_ON_UNKNOWN_PROPERTIES` surprises** — a client adds a field and every request 400s. Decide your tolerance explicitly.
- **Expecting `@RequestBody` to work without `Content-Type: application/json`** — wrong/missing content type → `415` and Jackson never runs.
- **Putting heavy logic in `postHandle`** when the response is already partly committed.

---

<a name="part-ix"></a>
# Part IX — REST API Design Masterclass

### Step 1 — Problem Statement

A poorly designed API leaks internal structure, breaks on every change, and forces clients into brittle workarounds. Good design is what separates an API that survives five years and a hundred clients from one that needs a rewrite every quarter.

### Step 2 — Intuition

Design around **resources and representations**, keep URLs about *nouns*, use HTTP semantics for behavior, version deliberately, and make errors machine-readable. Consistency beats cleverness.

### Step 3 — Real-World Analogy

An e-commerce catalog: products, categories, orders, customers are **resources** with stable addresses. You don't invent a new "operation" for every UI button; the UI composes standard resource operations. A new mobile app should be buildable against the *same* API without server changes.

### Step 4/5 — Mechanics and Examples

#### URL design

```
Bad:   GET  /getUsers
       POST /createUser
       GET  /user-list-active

Good:  GET    /users
       POST   /users
       GET    /users?status=ACTIVE
```

Rules:
- **Nouns, plural, lowercase, hyphenated:** `/purchase-orders`, not `/PurchaseOrder` or `/getPurchaseOrders`.
- **The method is the verb.** The URL names the *thing*.
- **Hierarchy via nesting** for true ownership.

#### Resources, collections, nested resources

```
/users                     collection
/users/42                  single resource
/users/42/orders           orders belonging to user 42 (nested)
/users/42/orders/1001      a specific nested resource
```

Don't nest more than ~2 levels — `/users/42/orders/1001/items/5/discounts` is a sign you should expose `/order-items/...` directly.

#### Versioning

| Strategy | Example | Notes |
|---|---|---|
| URI versioning | `/api/v1/users` | Most common, explicit, cache-friendly |
| Header versioning | `Accept: application/vnd.company.v1+json` | Cleaner URLs, harder to test |
| Param versioning | `/users?version=1` | Discouraged |

Pick one and be consistent. URI versioning (`/v1`) is the pragmatic default for enterprises.

#### Pagination

**Offset pagination:**
```
GET /users?page=2&size=20      (or ?offset=40&limit=20)
```
- Simple, supports "jump to page N."
- **Problem:** unstable on changing data (items shift between pages); slow on deep offsets (`OFFSET 1000000` scans a million rows).

**Cursor (keyset) pagination:**
```
GET /users?limit=20
→ returns items + "nextCursor":"eyJpZCI6NjB9"
GET /users?limit=20&cursor=eyJpZCI6NjB9
```
- Stable and fast even on huge datasets (uses `WHERE id > :cursor`).
- **Problem:** no random page jumps; cursor is opaque.

> Rule of thumb: offset for small/admin datasets and "page N" UIs; **cursor for large, fast-changing, or infinite-scroll feeds.**

Return pagination metadata:
```json
{
  "data": [ ... ],
  "page": { "size": 20, "totalElements": 1043, "nextCursor": "..." }
}
```

#### Filtering and sorting

```
GET /users?status=ACTIVE&role=ADMIN     filtering (AND of conditions)
GET /users?sort=name,asc&sort=createdAt,desc   sorting (multi-field)
GET /users?fields=id,name,email         sparse fieldsets (return only some fields)
```

Spring supports this cleanly with `Pageable`:

```java
@GetMapping("/users")
public Page<UserDto> list(@RequestParam(required=false) Status status,
                          Pageable pageable) {  // ?page=&size=&sort=
    return service.search(status, pageable);
}
```

#### Enterprise error handling

A consistent, machine-readable error envelope across *all* endpoints:

```json
{
  "timestamp": "2026-06-20T12:00:00Z",
  "status": 400,
  "error": "VALIDATION_ERROR",
  "message": "Request validation failed",
  "details": [
    { "field": "email", "issue": "must be a valid email" }
  ],
  "path": "/api/v1/users",
  "traceId": "a1b2c3d4-..."
}
```

Categorize errors:
- **Validation errors (400):** field-level problems the client can fix.
- **Business errors (409/422):** rules violated (insufficient funds, duplicate email) — valid request, invalid in context.
- **Technical errors (500/502/503):** infrastructure/code failures the client can't fix; expose a `traceId`, hide internals.

Consider the standard **RFC 7807 `application/problem+json`** format for interoperability:
```json
{ "type":"https://errors.company.com/insufficient-funds",
  "title":"Insufficient funds", "status":409, "detail":"Balance 20 < 50",
  "instance":"/api/v1/accounts/7/withdrawals", "traceId":"..." }
```

### Step 6 — Common Mistakes

- **Verbs in URLs / RPC-style endpoints.** `/users/42/activate` occasionally OK; `/doStuff` never.
- **Exposing DB IDs/schema** directly when it couples clients to internals; consider public IDs.
- **Inconsistent error shapes** — every endpoint inventing its own — making clients write per-endpoint parsing.
- **Breaking changes without versioning** — renaming/removing fields, tightening validation. Add, don't remove; deprecate, then version.
- **Deep offset pagination on huge tables** — performance cliff. Switch to cursor.
- **Returning everything** — no pagination on `/users` that has 2M rows.

---

<a name="part-x"></a>
# Part X — Postman Complete Guide

### Step 1 — Problem Statement

You need to call APIs without writing a client, reproduce bugs exactly, automate regression checks, and share a working contract with teammates. Postman is the standard tool for all of this — but most developers use 10% of it.

### Step 2 — Intuition

Postman is a *programmable HTTP client + test runner + collaboration tool*. A **request** is a saved HTTP call; a **collection** groups them; **environments** swap values (dev/staging/prod) without editing each request; **scripts** add logic before and after.

### Step 3 — Real-World Analogy

Postman is a **flight simulator for your API**. You can fly any route (request), save flight plans (collections), switch airports (environments), run pre-flight checklists (pre-request scripts) and post-flight inspections (tests), and run the whole schedule automatically (collection runner / Newman).

### Step 4 — Core Concepts

- **Requests** — method, URL, params, headers, body, auth.
- **Collections** — folders of requests; the unit you share, version, and run.
- **Environments** — named sets of variables: `{{baseUrl}}` = `https://dev.api...` in Dev, `https://api...` in Prod.
- **Variables** — scopes: global → collection → environment → local/data. Reference with `{{name}}`.
- **Workspaces** — team collaboration boundaries (personal/team/public).

### Step 5 — Practical: The request tabs

| Tab | What it does | Debug use |
|---|---|---|
| **Params** | Query (`?a=b`) and path variables | Verify filters/pagination are actually being sent |
| **Authorization** | Auth schemes (see below) | Wrong/missing token → 401 |
| **Headers** | Request headers (auto + custom) | Wrong `Content-Type` → 415; missing `Accept` |
| **Body** | raw/JSON, form-data, x-www-form-urlencoded, binary | Malformed JSON → 400; wrong type → 415 |
| **Pre-request Script** | JS before sending | Generate timestamps, sign requests, fetch tokens |
| **Tests** | JS after response | Assertions, extract values into variables |
| **Settings** | Redirects, SSL verification, encoding | Toggle SSL verify to diagnose cert issues |

#### Authentication in Postman

- **Basic Auth** — username/password → Postman builds `Authorization: Basic <base64>`.
- **Bearer Token** — paste/store a token → `Authorization: Bearer <token>`.
- **OAuth 2.0** — Postman runs the flow, gets an access token, and attaches it. Configure auth URL, token URL, client ID/secret, scopes.
- **API Key** — add as header (`X-API-Key`) or query param.

#### Pre-request scripts (run *before* the request)

```javascript
// Auto-attach a fresh timestamp and a generated idempotency key
pm.request.headers.add({ key: 'Idempotency-Key', value: pm.variables.replaceIn('{{$guid}}') });
pm.environment.set('now', Date.now());
```

#### Tests / assertions (run *after* the response)

```javascript
pm.test("status is 201", () => pm.response.to.have.status(201));
pm.test("returns an id", () => {
  const body = pm.response.json();
  pm.expect(body.id).to.be.a('number');
  pm.environment.set('userId', body.id);   // chain into the next request
});
pm.test("responds under 500ms", () => pm.expect(pm.response.responseTime).to.be.below(500));
```

This is how you build **request chains**: create a user, capture its `id`, then `GET /users/{{userId}}`.

#### Advanced features

- **Collection Runner** — run a whole collection in order, optionally driven by a CSV/JSON **data file** (data-driven testing).
- **Newman** — Postman's CLI runner; run collections in **CI/CD**:
  ```bash
  newman run users.postman_collection.json -e dev.postman_environment.json --reporters cli,junit
  ```
- **Mock Servers** — Postman returns example responses so frontend devs can build against an API before the backend exists.
- **API Documentation** — auto-generated, shareable docs from your collection.
- **Monitors** — scheduled runs that alert on failures (uptime/contract checks).

### Postman as a debugging tool

| Problem | How Postman reveals it |
|---|---|
| Wrong headers | Headers tab + "Postman Console" (View → Show Postman Console) shows the *actual* bytes sent |
| Wrong body | Body tab; Console shows the serialized payload |
| Auth failures | Swap tokens in Authorization tab; decode JWT; compare 401 vs 403 |
| Wrong content type | Set `Content-Type` explicitly; reproduce 415 |
| Timeouts | Console shows where time went; adjust Settings → Request timeout |
| SSL issues | Settings → turn off "SSL certificate verification" to confirm a cert problem (then fix the cert, don't ship with it off) |

> **The Postman Console** (separate from the test console) is your best friend: it shows the *raw* request and response including auto-added headers, redirects, and cookies. When "it should work," open the Console and read what was *actually* sent.

### Step 6 — Common Mistakes

- **Hardcoding URLs/tokens** instead of using environment variables → can't switch dev/prod, secrets get committed.
- **Forgetting Postman doesn't enforce CORS.** It happily makes any cross-origin call — so "works in Postman, fails in browser" is *expected* for CORS issues, not a contradiction.
- **Committing real secrets** in shared collections. Use environment variables and `secret` variable type.
- **Not reading the Postman Console** and guessing what was sent.
- **Leaving SSL verification off** permanently to "make it work."

---

<a name="part-xi"></a>
# Part XI — Chrome DevTools Masterclass

### Step 1 — Problem Statement

When a frontend reports "the API doesn't work," the truth is *in the browser*, not your IDE. DevTools shows exactly what the browser sent, what came back, how long it took, and why it was blocked. A backend engineer who can read DevTools resolves frontend/backend disputes in minutes.

### Step 2 — Intuition

DevTools is an **x-ray of the browser**. The Network tab is the most important: it records every HTTP exchange with full headers, payload, timing, and errors.

### Step 3 — Real-World Analogy

DevTools is the **flight data recorder + control tower view** of the browser: every request that took off, where it went, how long each leg took, and whether the tower (browser) refused clearance (CORS, mixed content).

### Step 4/5 — The tabs that matter

#### Network tab (your primary tool)

Open DevTools (F12) → **Network**. Refresh. Each row is a request. Click one:

- **Headers** — General (URL, method, **status code**, remote address), Request headers, Response headers. *First stop for any API bug.*
- **Payload** — what was actually sent (query params, JSON body, form data).
- **Preview** — formatted (pretty) response — handy for JSON/HTML.
- **Response** — the raw response body bytes.
- **Timing** — the phase breakdown: Queueing, Stalled, DNS, Initial connection, SSL, **Waiting (TTFB)**, Content Download. (This is the Part I chain, visualized.)
- **Initiator** — what code triggered the request (which JS line/stack).
- **Cookies** — cookies sent and received for this request.

**Debugging with the Network tab:**

| Symptom | What to check |
|---|---|
| **API not called** | Is there *any* row? If not, the bug is in frontend JS — the request never fired. |
| **Wrong URL** | The General → Request URL. Typos, missing `/api`, wrong env base URL. |
| **Wrong method** | General → Request Method. Frontend sending GET where POST is expected (or a redirect changed it). |
| **CORS error** | Console shows `blocked by CORS policy`; Network shows the **OPTIONS preflight** failing or missing `Access-Control-Allow-*` response headers. |
| **Slow API** | Timing tab — is it DNS? TLS? Or **TTFB** (server slow)? Different fixes. |
| **Failed auth** | Status 401/403; Request Headers → is `Authorization` present and correct? |
| **Wrong payload** | Payload tab — frontend sent the wrong field names or empty body. |

Pro tips: enable **Preserve log** (survive redirects/navigations), **Disable cache** (while DevTools open), and filter by `Fetch/XHR` to see only API calls. Right-click a request → **Copy as cURL** to reproduce it in a terminal or share it.

#### Console tab

- **JavaScript errors** — a thrown error may abort the code *before* the API call fires.
- **Network errors** — `Failed to load resource: net::ERR_*`.
- **CORS messages** — the exact, explicit reason: *"No 'Access-Control-Allow-Origin' header is present"* or *"...does not allow credentials."* Read it literally; it tells you which header to fix.
- **Mixed content warnings** — HTTPS page calling HTTP.

#### Application tab

- **Cookies** — inspect/edit/delete; check `HttpOnly`, `Secure`, `SameSite`, expiry, domain. A wrong cookie `domain`/`SameSite` is a common "logged out unexpectedly" cause.
- **Local Storage / Session Storage** — often where SPA tokens live. Verify the token the app *thinks* it has.
- **Cache Storage / Service Workers** — a stale **service worker** can serve old responses or intercept/rewrite API calls. If "my fix isn't showing," check for a service worker and "Update on reload" / Unregister.

#### Security tab

- **HTTPS / certificate** — view the cert chain, issuer, validity. Diagnose `NET::ERR_CERT_*`.
- **Mixed content** — flags HTTP subresources on an HTTPS page (browser blocks them).

#### Performance tab

- Record a session to find **slow requests**, long tasks, and **rendering** bottlenecks. For API work, correlate a slow request with what the UI was doing.

### Step 6 — Common Mistakes

- **Not enabling "Preserve log"** → the evidence vanishes on redirect/navigation.
- **Confusing a CORS block with a server failure.** The server often *did* respond (check your server logs); the *browser* blocked JS from reading it. The Network row may even show the response.
- **Ignoring the Console's exact CORS text** and guessing. It names the missing header.
- **Forgetting service workers** caching old responses.
- **Reading TTFB as "network slow"** when it's the *server* thinking (TTFB = server processing time).

---

<a name="part-xii"></a>
# Part XII — The API Debugging Playbook

> A symptom → causes → investigation → fix runbook. This is the section to keep open during an incident.

### General method (memorize this)

1. **Reproduce** with `curl`/Postman to remove the frontend as a variable.
2. **Read the status code** — it tells you *whose* fault (4xx = client, 5xx = server).
3. **Read the response body** — Spring usually explains 4xx.
4. **Bisect the chain** (Part I): browser → gateway → app → DB. Where does it break?
5. **Check logs by traceId** (Part XIV).

---

### API returns 400 (Bad Request)

- **Symptoms:** request rejected immediately; body often names a field.
- **Possible causes:** invalid/malformed JSON; failed `@Valid` validation; missing required fields; type mismatch (`"age":"thirty"`); wrong date format.
- **Investigate:** read the response body; validate the JSON (jsonlint); check the DTO's constraints; confirm `Content-Type: application/json`; reproduce in Postman Console to see exact bytes.
- **Fix:** correct the payload; align field names with Jackson mappings; relax/clarify validation messages.

### API returns 401 (Unauthorized)

- **Symptoms:** rejected before business logic; maybe a `WWW-Authenticate` header.
- **Possible causes:** missing token; malformed header (`Bearer Bearer ...`, extra spaces); invalid signature; **expired token** (`exp` in the past); wrong audience/issuer; clock skew.
- **Investigate:** is `Authorization` present (DevTools Request Headers)? Decode the JWT at jwt.io — check `exp`, `iss`, `aud`, signature. Compare with a known-good token.
- **Fix:** obtain a fresh token; correct the header; sync clocks (NTP); fix issuer/audience config.

### API returns 403 (Forbidden)

- **Symptoms:** authenticated but denied.
- **Possible causes:** user lacks required role/scope; method-level security (`@PreAuthorize`) denies; resource-level ownership check fails; CSRF token missing (for cookie-based flows).
- **Investigate:** decode JWT claims — what roles/scopes does this user *actually* have vs. what the endpoint requires? Check `@PreAuthorize` expressions and security config.
- **Fix:** grant the role/scope; correct the authorization rule; for CSRF, include the token.

### API returns 404 (Not Found)

- **Symptoms:** "no such endpoint" or "no such resource."
- **Possible causes:** wrong URL/typo; missing/extra `server.servlet.context-path`; wrong version prefix (`/v1` vs `/v2`); trailing slash mismatch; the entity genuinely doesn't exist; a hidden 403 (server returns 404 to avoid leaking existence).
- **Investigate:** print the full effective URL; check `RequestMappingHandlerMapping` logs (`logging.level.org.springframework.web=DEBUG`) to see registered mappings; confirm the gateway routes the path to this service.
- **Fix:** correct the path/context; ensure routing; handle missing entities with a clean 404 body.

### CORS errors

- **Symptoms:** Console: *"blocked by CORS policy."* Works in Postman/curl, fails in browser.
- **Why Postman works but browser fails:** Postman is not a browser — it doesn't enforce the Same-Origin Policy and doesn't send `Origin` the same way. CORS is a *browser* security mechanism. So a CORS failure is invisible to Postman by design.
- **Possible causes:** missing `Access-Control-Allow-Origin`; origin not in allow-list; **preflight OPTIONS** failing (Spring Security 401s it); `Allow-Credentials: true` with `Allow-Origin: *`; missing `Allow-Headers` for a custom header like `Authorization`.
- **Investigate:** in Network, find the **OPTIONS** request — what status and response headers? Read the Console's exact text (it names the missing header). Confirm the server returns the right `Access-Control-Allow-*`.
- **Fix:** configure CORS on the server (Part VI); permit OPTIONS in security config; name the exact origin when using credentials; add the needed headers to `Allow-Headers`.

### Preflight OPTIONS requests

- **What:** for non-simple cross-origin calls, the browser sends `OPTIONS` *first* to ask permission. If it fails, the real request never goes.
- **Investigate:** is there an OPTIONS row before your PUT/POST? Does it return 200/204 with `Access-Control-Allow-*`? If it 401s, Security is intercepting it.
- **Fix:** allow unauthenticated OPTIONS; return correct CORS headers; set `maxAge` to cache preflights.

### Timeout problems

- **Symptoms:** request hangs then fails; 504 from a gateway.
- **Types:**
  - **Network/connection timeout** — can't even establish TCP (firewall, wrong host, service down).
  - **Read/response (server) timeout** — connected, but the server didn't respond in time (slow code, blocked thread, deadlock).
  - **Database timeout** — a slow query or exhausted connection pool stalls the request.
- **Investigate:** DevTools Timing (stuck in "Connecting" vs "Waiting"?). Check app thread dumps, slow-query logs, connection pool metrics (HikariCP), and gateway timeout settings.
- **Fix:** add timeouts everywhere (never infinite); optimize/queries/indexes; size the connection pool; add circuit breakers; scale.

### Large request problems

- **Symptoms:** 413 Payload Too Large, 431 Header Fields Too Large, truncated uploads, or a silent reset.
- **Causes:**
  - **Header size limits** — too many/large headers (huge JWT, many cookies) exceed server/proxy limits (`server.max-http-request-header-size`, NGINX `large_client_header_buffers`).
  - **Payload limits** — body exceeds `spring.servlet.multipart.max-request-size` or proxy `client_max_body_size`.
  - **Multipart upload limits** — `max-file-size` / `max-request-size` for file uploads.
- **Investigate:** check the *exact* status; check both app and proxy limits (a 413 may come from NGINX before Spring sees it).
- **Fix:** raise the relevant limits intentionally; shrink tokens/cookies; stream large uploads; use presigned URLs for big files instead of routing bytes through the app.

### Serialization problems

- **Symptoms:** 400 on a request that "looks fine," 500 on a response, mangled fields, wrong date formats.
- **Causes:**
  - **JSON parsing errors** — malformed JSON, trailing commas, wrong types → `HttpMessageNotReadableException` (400).
  - **Jackson issues** — unknown fields with `FAIL_ON_UNKNOWN_PROPERTIES`; missing no-arg constructor; field name mismatch; `LocalDateTime` without the JSR-310 module; infinite recursion on bidirectional JPA relations (`StackOverflowError` → 500).
  - **Serialization of lazy JPA proxies** outside a transaction.
- **Investigate:** read the exception in logs; reproduce the exact body; check the DTO/entity annotations.
- **Fix:** use DTOs (not entities) for the API surface; configure the `ObjectMapper` (JSR-310, `NON_NULL`, unknown-property tolerance); break recursion with `@JsonManagedReference`/`@JsonBackReference` or DTOs; standardize date formats.

---

<a name="part-xiii"></a>
# Part XIII — Microservices HTTP Communication

### Step 1 — Problem Statement

In a monolith, a "service call" is a method call — fast and reliable. In microservices, it's a *network call* over HTTP: slow, fallible, and partially-failing. Everything you assumed (it always returns, instantly, correctly) is now false. You must design for failure.

### Step 2 — Intuition

When Service A calls Service B, you've replaced a function call with the entire Part I chain (DNS → TCP → TLS → HTTP → ... → back). Networks drop, services restart, and B might be slow. So service-to-service calls need **timeouts, retries, circuit breakers, load balancing, and discovery**.

### Step 3 — Real-World Analogy

Calling another microservice is like an e-commerce **checkout** calling a separate **payment** provider: it might be slow, temporarily down, or rate-limit you. You don't freeze the whole checkout forever — you set a deadline (timeout), retry briefly, stop hammering a dead provider (circuit breaker), and have a fallback ("payment is busy, try again").

### Step 4 — Internal Mechanics

#### The clients

- **`RestTemplate`** — classic, synchronous, blocking. Simple, still common, in maintenance mode (no new features).
- **`WebClient`** — modern, reactive, non-blocking; works for both reactive and (with `.block()`) blocking use. Preferred for new code and for high-concurrency calls.
- **OpenFeign** — declarative: define an interface, Spring generates the HTTP client. Cleanest for service-to-service in Spring Cloud.

```java
// Feign — declarative
@FeignClient(name = "payment-service")
public interface PaymentClient {
    @PostMapping("/api/v1/charges")
    ChargeResult charge(@RequestBody ChargeRequest req);
}
```

```java
// WebClient — modern
ChargeResult r = webClient.post()
    .uri("http://payment-service/api/v1/charges")
    .bodyValue(req)
    .retrieve()
    .bodyToMono(ChargeResult.class)
    .timeout(Duration.ofSeconds(2))
    .block();
```

#### The resilience problems and their patterns

- **Timeout** — *always* bound every call. A missing read timeout means one slow dependency can exhaust your thread pool and cascade an outage. Set connect *and* read timeouts.
- **Retry** — retry only **idempotent**, **transient** failures (network blips, 503), with **exponential backoff + jitter**, and a small max attempts. Never blindly retry a non-idempotent POST (double charge!) — use idempotency keys.
- **Circuit breaker** (Resilience4j) — after N failures, "open" the circuit and fail fast for a cooldown, sparing the struggling service and your threads. Then "half-open" to test recovery. Provide a **fallback**.
- **Load balancing** — spread calls across B's instances. Spring Cloud LoadBalancer resolves `http://payment-service` to a healthy instance.
- **Service discovery** — instances register with a registry (Eureka/Consul/Kubernetes DNS); callers look up *current* addresses instead of hardcoding IPs (which change as pods scale).

```java
@CircuitBreaker(name = "payment", fallbackMethod = "fallback")
@Retry(name = "payment")
public ChargeResult charge(ChargeRequest req) { return paymentClient.charge(req); }

private ChargeResult fallback(ChargeRequest req, Throwable t) {
    return ChargeResult.deferred();   // graceful degradation
}
```

### Step 5 — Practical: What happens when Service A calls Service B

```
A: orderService.charge()
 → Feign/WebClient builds POST http://payment-service/api/v1/charges
 → Service discovery resolves "payment-service" → 10.0.3.7:8080 (a healthy instance)
 → Client-side load balancer picks the instance
 → Circuit breaker checks: is the circuit closed? (yes → proceed)
 → DNS/TCP/TLS/HTTP to B  (the whole Part I chain, intra-cluster)
 → B's Tomcat → filters → DispatcherServlet → controller → ... → response
 → A reads response within the 2s timeout
   ├─ success → continue
   ├─ timeout/5xx → Retry (backoff) → maybe circuit opens → fallback
 → Correlation/trace ID propagated in headers the whole way (Part XIV)
```

### Step 6 — Common Mistakes

- **No timeouts** → one slow service freezes threads everywhere → cascading failure. The #1 microservices outage cause.
- **Retrying non-idempotent calls** → duplicate side effects (double orders/charges).
- **Retry storms** — everyone retries a struggling service at once and finishes it off. Use backoff + jitter + circuit breakers.
- **Hardcoding instance IPs** instead of using discovery.
- **Not propagating trace/correlation IDs** → un-debuggable distributed flows.
- **Synchronous chains** (A→B→C→D) multiplying latency and failure probability; consider async/events where appropriate.

---

<a name="part-xiv"></a>
# Part XIV — Production Debugging

### Step 1 — Problem Statement

In production you can't attach a debugger to a request that already failed for a user in another country an hour ago. You have *logs, traces, and metrics*. Senior engineers debug by correlating these across services.

### Step 2 — Intuition

Every request gets a unique **ID** that travels with it across every service and appears in every log line. To debug, you find that one ID and read the whole story — across machines — in order.

### Step 3 — Real-World Analogy

A **package tracking number**. One ID follows your parcel through every warehouse and truck. When it's lost, support types the number and sees every scan. Trace IDs are tracking numbers for requests.

### Step 4 — Internal Mechanics

#### Correlation IDs, trace IDs, request IDs

- **Request/Correlation ID** — one ID per incoming request, generated at the edge (gateway) or first service.
- **Trace ID** — the ID for an entire distributed transaction across services (OpenTelemetry/Micrometer Tracing).
- **Span ID** — one unit of work *within* a trace (one service's portion). A trace is a tree of spans.

Propagation: each service reads the trace ID from inbound headers (e.g., W3C `traceparent`) and forwards it on outbound calls. Put it in the logging **MDC** so every log line carries it.

```java
// Edge filter: ensure every request has a trace id, in logs and forwarded
@Component
public class TraceFilter extends OncePerRequestFilter {
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain c) throws ... {
        String id = Optional.ofNullable(req.getHeader("X-Trace-Id")).orElse(UUID.randomUUID().toString());
        MDC.put("traceId", id);
        res.setHeader("X-Trace-Id", id);     // return to client for support tickets
        try { c.doFilter(req, res); } finally { MDC.clear(); }
    }
}
```

```
# logback pattern includes the trace id
%d{ISO8601} %-5level [%X{traceId}] %logger{36} - %msg%n
```

#### Distributed tracing

Tools: **Micrometer Tracing + OpenTelemetry**, exported to **Zipkin/Jaeger/Tempo**. You get a flame-graph of a request across services: which span was slow, which call failed, with the trace ID linking back to logs.

#### Structured logging

Log JSON with consistent fields (`traceId`, `userId`, `path`, `status`, `latencyMs`) so log aggregators (ELK/Loki/Splunk/Datadog) can filter and chart. Search `traceId:"abc"` to reconstruct one request across all services.

### Step 5 — Practical: the production toolkit

| Tool | Use |
|---|---|
| **Logs (by traceId)** | Reconstruct one request end-to-end across services |
| **Distributed tracing (Jaeger/Zipkin)** | See span timings; find the slow/failing hop |
| **Spring Boot Actuator** | `/actuator/health`, `/metrics`, `/httpexchanges` (recent HTTP calls), `/loggers` (change log level at runtime) |
| **curl** | Reproduce from a server/pod with exact headers; bypass the frontend |
| **Postman / DevTools** | Reproduce client-side; capture exact request (Copy as cURL) |
| **Metrics/dashboards** | Error rate, p99 latency, throughput, pool saturation — *trends*, not single requests |

```bash
# Change a logger to DEBUG at runtime without redeploying:
curl -X POST localhost:8080/actuator/loggers/com.company.payment \
  -H 'Content-Type: application/json' -d '{"configuredLevel":"DEBUG"}'

# See recent HTTP exchanges this instance handled:
curl localhost:8080/actuator/httpexchanges
```

#### A production debugging workflow

1. **Get the trace ID** from the user's error (you returned it in the body/`X-Trace-Id`).
2. **Search logs** for that ID → see every service it touched and where it failed.
3. **Open the trace** in Jaeger → which span errored or was slow?
4. **Check metrics** around that time → was it isolated or systemic (deploy, DB saturation)?
5. **Reproduce** with curl/Postman using the same inputs.
6. **Confirm the fix** with a metric/log query, not a hunch.

### Step 6 — Common Mistakes

- **No correlation/trace IDs** → impossible to follow a request across services; you grep blindly.
- **Logging secrets** (`Authorization`, card numbers, PII) → compliance breach. Mask them.
- **Only logging errors** → you can't see the *normal* path that led to the error.
- **Returning no traceId to clients** → support can't give you a needle for the haystack.
- **Debugging from single requests instead of metrics** for systemic issues (and vice versa).
- **Exposing Actuator publicly** without securing it → info leak. Restrict it.

---

<a name="part-xv"></a>
# Part XV — Senior Interview Preparation

Detailed answers to the questions a senior backend interview will probe. Practice saying these out loud.

### Q1. Explain what happens when a browser calls a REST API.

The browser parses the URL and checks HSTS/cache/cookies. It resolves the host via **DNS** (browser → OS → resolver → root → TLD → authoritative). It opens a **TCP** connection (three-way handshake). For HTTPS it performs a **TLS** handshake: negotiate cipher, validate the server's certificate (chain, hostname, expiry), derive session keys. It sends an **HTTP request** (request line + headers + optional body) through the encrypted tunnel. The request traverses **CDN/load balancer/gateway/reverse proxy**, then reaches **embedded Tomcat**, passes **servlet filters**, hits the **DispatcherServlet**, which maps it to a controller, runs interceptors, and uses **Jackson** to deserialize the body. The **controller → service → repository → DB** produces a result, serialized back to JSON, with status and headers set, and the response travels back through the same chain. The browser reads the status, parses the body, and runs the JS callback (or renders). *Crucially, the browser also enforces the Same-Origin Policy/CORS — which a tool like Postman does not.*

### Q2. Difference between PUT and PATCH.

Both modify an existing resource. **PUT replaces the entire resource** with the representation you send — omitted fields are conceptually reset; it is **idempotent** (same body twice = same state). **PATCH applies a partial update** — you send only changed fields; it is **not guaranteed idempotent**. Use PUT when the client sends the full object and controls the ID; use PATCH for targeted field updates. A PATCH gotcha: distinguishing "field absent" (leave alone) from "field = null" (clear it).

### Q3. Why does CORS happen?

Browsers enforce the **Same-Origin Policy**: JS on origin A can't *read* responses from origin B by default. This stops a malicious site from silently calling your bank (where your browser holds cookies) and reading the result. **CORS** lets a server *opt in* to cross-origin access via `Access-Control-Allow-*` headers. For non-simple requests the browser first sends an **OPTIONS preflight** asking permission; only if the server approves does the real request proceed. CORS is enforced by the browser, authored by the server.

### Q4. Why does Postman work but the browser fails?

Because **Postman is not a browser**. It doesn't enforce the Same-Origin Policy, doesn't send an `Origin` header the same way, doesn't run preflights, and doesn't auto-attach cookies/HSTS the way a page does. So CORS failures are invisible in Postman. Other reasons: the browser may send cookies/session that Postman lacks (or vice versa), different `Content-Type`, HSTS upgrades, mixed-content blocking, or a service worker rewriting requests. The fix is to reproduce in the browser's DevTools and read the exact Console error — usually CORS or a missing header/cookie.

### Q5. How does Spring convert JSON into Java objects?

When a request has `Content-Type: application/json` and the handler has `@RequestBody`, Spring selects the `MappingJackson2HttpMessageConverter`, which uses Jackson's `ObjectMapper.readValue()` to **deserialize** the body into your DTO (binding field names, with annotations like `@JsonProperty`). On the way out, the return value is **serialized** back to JSON by the same converter (`writeValue`), chosen via content negotiation (`Accept` header). Without the right `Content-Type`, you get **415** and Jackson never runs.

### Q6. What happens inside DispatcherServlet?

It's Spring MVC's front controller — the single servlet Tomcat routes to. It: (1) uses **HandlerMapping** to find the controller method for the request (404/405 if none); (2) invokes it via a **HandlerAdapter**; (3) runs **HandlerInterceptors** (`preHandle`); (4) resolves arguments with **ArgumentResolvers** (`@PathVariable`, `@RequestParam`, `@RequestBody` → message converter); (5) calls your method; (6) serializes the return value via a **message converter**; (7) runs `postHandle`/`afterCompletion`; (8) routes any exception to **HandlerExceptionResolvers** (your `@RestControllerAdvice`).

### Q7. Difference between 401 and 403.

**401 Unauthorized** = *not authenticated* — "I don't know who you are." Missing/invalid/expired credentials; providing valid ones can fix it; often accompanied by `WWW-Authenticate`. **403 Forbidden** = *authenticated but not authorized* — "I know who you are, and you may not." The same credentials won't help; you need different permissions/roles. Mnemonic: 401 = *who are you?*, 403 = *you're not allowed.*

### Q8. Explain HTTP caching.

Servers tell clients/proxies whether and how long to reuse a response. **`Cache-Control`** (`max-age`, `public/private`, `no-store/no-cache`) is the master switch. **Validation** uses **`ETag`** (a version fingerprint) or **`Last-Modified`**: the client later sends **`If-None-Match`**/**`If-Modified-Since`**; if unchanged, the server returns **`304 Not Modified`** with no body, saving bandwidth. **`Expires`** is the legacy absolute-time form. Good caching is a core reason the web scales (a REST constraint).

### Q9. How does HTTPS work?

HTTPS is HTTP over **TLS**. After the TCP handshake, the TLS handshake runs: client and server agree on a TLS version and cipher; the server presents a **certificate** (its public key, signed by a CA); the client validates it (trusted CA chain, hostname match, not expired/revoked); they derive a shared **session key** (via asymmetric crypto / Diffie-Hellman) and switch to fast **symmetric** encryption for the data. This gives **confidentiality** (encrypted), **integrity** (tamper-evident), and **authenticity** (you're really talking to the named server). TLS 1.3 cut the handshake to ~1 round trip.

### Q10. How would you debug an intermittent 504 in production?

A 504 means something *upstream of the responder* timed out waiting. I'd: check **which hop** emits it (gateway→service or service→DB) via traces; look at **TTFB**/span timings in distributed tracing; check **DB slow-query logs** and **connection pool** saturation (HikariCP) around the time; correlate with **metrics** (latency p99, deploys, traffic spikes); verify **timeouts** are set sanely at each layer and that **circuit breakers/retries** aren't amplifying load. Fixes typically: optimize/queries+indexes, right-size pools and timeouts, add caching, or scale.

### Rapid-fire facts to have ready

- Idempotent: GET, PUT, DELETE, HEAD, OPTIONS. Not: POST, PATCH.
- Safe: GET, HEAD, OPTIONS.
- 201 + `Location` for creates; 204 for empty success; 202 for async.
- `Allow-Credentials: true` ⇒ `Allow-Origin` can't be `*`.
- The fragment (`#...`) never reaches the server.
- Basic auth is base64 (encoding, not encryption) — HTTPS only.
- HTTP/2 multiplexes over one TCP connection; HTTP/3 uses QUIC/UDP to kill TCP head-of-line blocking.
- A preflight is an automatic `OPTIONS` for non-simple cross-origin requests.

---

## Appendix A — A senior's debugging command kit

```bash
# Full timing breakdown of a call
curl -w "dns:%{time_namelookup} conn:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} total:%{time_total}\n" \
     -o /dev/null -s https://api.company.com/users

# Show request + response headers
curl -v https://api.company.com/users

# Inspect the TLS certificate chain
openssl s_client -connect api.company.com:443 -servername api.company.com </dev/null

# Send a JSON POST with auth
curl -X POST https://api.company.com/v1/users \
  -H 'Authorization: Bearer TOKEN' -H 'Content-Type: application/json' \
  -d '{"name":"John"}'

# Reproduce a browser request: DevTools → right-click request → Copy as cURL
```

```properties
# Spring Boot knobs that come up constantly
server.servlet.context-path=/api
server.tomcat.max-http-form-post-size=2MB
server.max-http-request-header-size=16KB
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
logging.level.org.springframework.web=DEBUG
spring.jackson.deserialization.fail-on-unknown-properties=false
spring.jackson.default-property-inclusion=non_null
management.endpoints.web.exposure.include=health,metrics,httpexchanges,loggers
```

## Appendix B — Mental checklists

**Designing an endpoint:** noun-based URL? right method? right status codes (201/204/4xx/5xx)? validation + consistent error envelope? pagination/filtering/sorting? versioned? idempotency where needed? auth + CORS? trace ID?

**"The API is broken":** reproduce in curl/Postman → read status (4xx vs 5xx) → read body → DevTools Network (URL/method/headers/CORS/timing) → server logs by traceId → distributed trace → fix → verify with a query, not a hope.

---

### Final word

Master this chain end to end — DNS, TCP, TLS, HTTP, REST semantics, the Spring lifecycle, and the tools that x-ray each layer — and "the API doesn't work" stops being a mystery and becomes a methodical bisection. That calm, systematic confidence is exactly what distinguishes a senior backend engineer.
