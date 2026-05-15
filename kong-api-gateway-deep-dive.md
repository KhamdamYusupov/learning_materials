# Kong API Gateway — Deep Dive From Beginner to Senior Platform Engineer

> A complete, production-grade guide for Java + Spring Boot engineers who want to master Kong Gateway, understand API gateway architecture, and operate it confidently in Kubernetes.

---

## Table of Contents

1. [API Gateway Fundamentals](#1-api-gateway-fundamentals)
2. [Kong Gateway Fundamentals](#2-kong-gateway-fundamentals)
3. [Kong Core Concepts](#3-kong-core-concepts)
4. [Kong Deployment Modes](#4-kong-deployment-modes)
5. [Installing & Running Kong](#5-installing--running-kong)
6. [Kong with Java + Spring Boot](#6-kong-with-java--spring-boot)
7. [Authentication & Authorization](#7-authentication--authorization)
8. [Rate Limiting & Traffic Control](#8-rate-limiting--traffic-control)
9. [Load Balancing & Service Discovery](#9-load-balancing--service-discovery)
10. [Observability & Monitoring](#10-observability--monitoring)
11. [Kubernetes Integration](#11-kubernetes-integration)
12. [TLS & Security](#12-tls--security)
13. [Advanced Kong Plugins](#13-advanced-kong-plugins)
14. [Performance & Scaling](#14-performance--scaling)
15. [Failure Scenarios & Debugging](#15-failure-scenarios--debugging)
16. [Kong vs Alternatives](#16-kong-vs-alternatives)
17. [API Gateway Design Best Practices](#17-api-gateway-design-best-practices)
18. [Real-World Architectures](#18-real-world-architectures)
19. [Common Edge Cases Senior Engineers Must Know](#19-common-edge-cases-senior-engineers-must-know)
20. [Production Operations](#20-production-operations)
21. [Final Checklist](#21-final-checklist)

---

## 1. API Gateway Fundamentals

### 1.1 What is an API Gateway?

An **API Gateway** is a reverse proxy that sits **in front of your backend services** and acts as the **single entry point** for all client requests. It is the **policy enforcement point** of your distributed system.

```
                        ┌───────────────────────────────────┐
                        │           API Gateway             │
                        │  (routing, auth, rate limit, etc) │
                        └───────────────┬───────────────────┘
                                        │
   ┌─────────┐         ┌────────────────┼────────────────┐
   │ Browser │────────▶│                │                │
   └─────────┘         ▼                ▼                ▼
   ┌─────────┐    ┌──────────┐   ┌──────────┐   ┌──────────────┐
   │ Mobile  │───▶│  User    │   │  Order   │   │   Payment    │
   └─────────┘    │ Service  │   │ Service  │   │   Service    │
   ┌─────────┐    └──────────┘   └──────────┘   └──────────────┘
   │  3rd    │
   │  Party  │
   └─────────┘
```

Conceptually, an API gateway is **a reverse proxy on steroids**: a normal reverse proxy forwards requests; a gateway forwards requests **plus** applies cross-cutting concerns (authentication, rate limiting, transformation, logging, caching, tracing).

### 1.2 Why API Gateways Exist (First Principles)

Without a gateway, **every microservice** must re-implement:

- Authentication / JWT validation
- Rate limiting
- TLS termination
- Logging / metrics / tracing
- CORS handling
- Request/response transformation
- Versioning of APIs
- IP allowlisting / firewall rules

This causes:

- **Duplication** of cross-cutting code in every service
- **Inconsistency** — Service A handles auth differently than Service B
- **Security risks** — one service forgets to validate the token
- **Operational chaos** — every team owns infra concerns

An API Gateway **centralizes** these concerns into one tier so backend services focus only on **business logic**.

### 1.3 Problems API Gateways Solve

| Problem | How a gateway solves it |
|---|---|
| Auth duplicated everywhere | Central JWT/OAuth2/API key validation |
| Inconsistent rate limiting | Global, consumer-based, route-based throttling |
| Many TLS endpoints | TLS termination at one place |
| Untraceable requests | Inject correlation IDs, OpenTelemetry tracing |
| Multiple client types | Request/response transformation per client |
| Internal service URLs leaking | Hide topology; clients only know gateway URL |
| Cross-cutting changes need 20 deploys | One config change at the gateway |
| Public API contracts | Versioning, deprecation, throttling |

### 1.4 API Gateway vs Load Balancer

| | Load Balancer | API Gateway |
|---|---|---|
| OSI layer | L4 (TCP) or L7 (HTTP) | L7 (HTTP, gRPC, WebSockets) |
| Primary job | Distribute traffic across instances | Enforce policy + route by API semantics |
| Awareness | IP / connection level | Headers, paths, JWT claims, methods |
| Auth | No | Yes |
| Rate limiting | Basic | Sophisticated (consumer, route, header) |
| Plugin model | No | Yes |
| Example | AWS NLB, HAProxy, ELB | Kong, Apigee, Tyk |

A load balancer answers: "**Which instance gets this packet?**"
An API gateway answers: "**Should this request happen at all, and if so where does it go?**"

In production they are **complementary**: an L4 load balancer (e.g., AWS NLB, K8s Service) often sits in front of Kong, distributing connections across Kong pods.

### 1.5 API Gateway vs Service Mesh

| | API Gateway | Service Mesh |
|---|---|---|
| Direction | **North–South** (external ↔ cluster) | **East–West** (service ↔ service inside cluster) |
| Deployment | Centralized tier | Sidecar per service (Envoy) |
| Concerns | Auth, versioning, public throttling | mTLS, retries, traffic splitting, internal policy |
| Examples | Kong, Apigee | Istio, Linkerd, Consul Connect |
| Use both? | Yes — common pattern in production |

In modern setups you typically have **Kong at the edge** + **Istio inside the cluster**.

### 1.6 When NOT to Use an API Gateway

- **Single service, single team, internal use** — adds latency and ops cost
- **Ultra-low-latency systems** — added hop (1–3 ms) may be unacceptable
- **Pure backend-to-backend** mesh traffic — use a service mesh instead
- **Static asset delivery** — use a CDN
- **You don't have someone to own it** — gateways become a critical SPOF if neglected

### 1.7 Real-World Architecture Examples

**E-commerce platform:**

```
[Mobile/Web] ──▶ [CDN] ──▶ [WAF] ──▶ [Kong] ──┬─▶ Catalog Service
                                              ├─▶ Cart Service
                                              ├─▶ Checkout Service
                                              └─▶ Payment Service
```

**Fintech platform:**

```
[Partner API] ──▶ [Kong (mTLS, JWT, IP allowlist)] ──▶ [Spring Boot core banking]
                                                  │
                                                  ├─▶ KYC Service
                                                  └─▶ Ledger Service
```

---

## 2. Kong Gateway Fundamentals

### 2.1 What is Kong Gateway?

**Kong Gateway** is an open-source, cloud-native API gateway built on top of **NGINX** and extended with **OpenResty (Lua)**. It is the **most widely deployed open-source API gateway** in the industry.

Two key tracks:
- **Kong OSS / Kong Gateway** — free, open source.
- **Kong Enterprise** — adds RBAC, workspaces, advanced plugins (OIDC, GraphQL rate limit), Dev Portal, Kong Manager UI, FIPS, advanced analytics.

### 2.2 OSS vs Enterprise (Quick Map)

| Feature | OSS | Enterprise |
|---|---|---|
| Routing, plugins, Admin API | ✅ | ✅ |
| JWT, key-auth, basic-auth, rate-limit | ✅ | ✅ |
| OIDC plugin | ❌ | ✅ |
| Dev Portal | ❌ | ✅ |
| Kong Manager (full UI) | Limited | ✅ |
| RBAC / Workspaces | ❌ | ✅ |
| Advanced analytics | ❌ | ✅ |
| Mocking, GraphQL plugins | ❌ | ✅ |

### 2.3 Kong Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                       KONG NODE                             │
│                                                             │
│   ┌─────────────────────────────────────────────────────┐   │
│   │                    NGINX (workers)                  │   │
│   │   ┌─────────────────────────────────────────────┐   │   │
│   │   │    OpenResty (Lua VM inside each worker)    │   │   │
│   │   │   ┌─────────────────────────────────────┐   │   │   │
│   │   │   │           KONG CORE (Lua)           │   │   │   │
│   │   │   │   Router • Plugin runner • Cache    │   │   │   │
│   │   │   └─────────────────────────────────────┘   │   │   │
│   │   └─────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────┘   │
│       ▲                       ▲                             │
│       │ Admin API (8001)      │ Proxy (8000/8443)           │
└───────┼───────────────────────┼─────────────────────────────┘
        │                       │
   ┌────┴────┐             ┌────┴────┐
   │ Config  │             │ Clients │
   │ store   │             └─────────┘
   │ (DB or  │
   │ YAML)   │
   └─────────┘
```

**Key building blocks:**

- **NGINX** — handles raw network I/O, HTTP parsing, TLS termination. Battle-tested, event-driven, multi-worker.
- **OpenResty** — embeds **LuaJIT** into NGINX. Lets Kong run Lua at every NGINX phase (`access`, `header_filter`, `body_filter`, `log`).
- **Kong core** — Lua code that implements routing, plugin lifecycle, caching, clustering.
- **Plugins** — Lua (or now Go/JavaScript/Python via PDK) modules that hook into NGINX phases.
- **Admin API** — REST API to configure Kong (DB mode) or read state (DB-less mode).

### 2.4 Data Plane vs Control Plane

- **Control plane (CP)**: Holds configuration. In hybrid mode, CP doesn't proxy traffic.
- **Data plane (DP)**: Receives configuration and **handles request traffic**.

In a single-node setup, both run in the same process. In hybrid mode (recommended for production), they are **separate clusters**.

### 2.5 Request Lifecycle Inside Kong

Kong processes each request through a series of **phases** (mapped to NGINX phases). Plugins hook into these phases.

```
Client                Kong (per worker)                    Upstream
  │                         │                                 │
  │  ── TCP/TLS ──▶         │                                 │
  │                         │  1. certificate (TLS)           │
  │                         │  2. rewrite     (rewrite URL)   │
  │                         │  3. access      (auth, rate     │
  │                         │     limit, ACL, transform)      │
  │                         │  4. router resolves Service     │
  │                         │  5. load balancer picks target  │
  │                         │  ── HTTP ──▶                    │
  │                         │                                 │
  │                         │  6. header_filter (mutate hdrs) │
  │                         │  7. body_filter   (mutate body) │
  │                         │  ◀── response ──                │
  │  ◀── response ──        │  8. log (HTTP log, prometheus,  │
  │                         │     file-log, tracing)          │
```

**Important properties:**
- Plugins run in **priority order** within each phase.
- The `access` phase is where **authentication and rate limiting** happen.
- The `log` phase runs **after** the response has been sent → log plugins should never block.
- Each NGINX worker has its **own LuaJIT VM**; shared state uses `lua_shared_dict` (process-wide shared memory).

### 2.6 Why Lua + NGINX?

- **NGINX** gives Kong proven networking performance.
- **LuaJIT** is one of the fastest scripting runtimes available.
- **Plugin model** is hot-loadable: enabling a plugin doesn't restart NGINX.
- Newer **plugin servers** (Go/JS/Python) let you write plugins in your favorite language while preserving the NGINX core.

---

## 3. Kong Core Concepts

Mastering Kong = mastering these six primitives. Everything else is plumbing.

### 3.1 Service

**What:** A logical reference to an upstream API (your backend).

**Why:** Decouples *what you call it* from *where it lives*.

**Real example (declarative YAML):**

```yaml
services:
  - name: order-service
    url: http://order-service.default.svc.cluster.local:8080
    retries: 5
    connect_timeout: 2000
    write_timeout: 60000
    read_timeout: 60000
```

**Common mistakes:**
- Hard-coding a pod IP instead of a Kubernetes Service DNS.
- Setting `retries: 0` then complaining about transient failures.
- Forgetting that timeouts are in **milliseconds**, not seconds.

### 3.2 Route

**What:** A rule mapping incoming requests → a Service.

**Why:** A single Service can have many Routes (e.g., `/v1/orders`, `/v2/orders`, internal `/admin/orders`).

```yaml
routes:
  - name: order-public
    service: order-service
    paths: [ "/v1/orders" ]
    methods: [ GET, POST ]
    strip_path: true       # /v1/orders/123 ──▶ /123 to upstream
    preserve_host: false   # don't forward client's Host header
```

**Mental model:** A Route is a **matcher** (host + path + method + headers + SNI) that **selects** a Service.

**Common mistakes:**
- `strip_path: true` when upstream expects the prefix.
- Conflicting routes — Kong picks one but engineers can't predict which (priority rules below).
- Mixing path-based and host-based routing carelessly.

**Route priority (key for debugging "wrong route"):**

1. More specific host wins (`api.example.com` > `*.example.com`).
2. Longer matching path wins.
3. More HTTP methods matching.
4. Explicit `regex_priority` (higher first).
5. Creation timestamp (older route wins as last tiebreaker).

### 3.3 Consumer

**What:** An entity that consumes your API (a user, a partner, an app).

**Why:** Authentication plugins map a credential (JWT, API key, cert) → a Consumer. All downstream policies (rate limit, ACL) can then target a specific Consumer.

```yaml
consumers:
  - username: mobile-app
    custom_id: "app-mobile-prod"
    keyauth_credentials:
      - key: "super-secret-key"
    acls:
      - group: trusted-apps
```

**Mental model:** A Consumer is **the identity** Kong cares about after auth. After authentication, Kong injects `X-Consumer-ID` / `X-Consumer-Username` headers into the upstream request.

**Common mistakes:**
- Creating one Consumer per end-user (don't — that's the app's job; create per-tenant or per-app).
- Sharing one Consumer across many apps → rate limits become useless.

### 3.4 Plugin

**What:** A piece of behavior attached to global / Service / Route / Consumer scope.

**Why:** This is **how Kong does everything beyond raw proxying** — JWT, rate limit, logging, transformation, CORS, ACL, OIDC, etc.

```yaml
plugins:
  - name: rate-limiting
    service: order-service
    config:
      minute: 60
      policy: redis
      redis_host: redis.default.svc
```

**Plugin scopes (most specific wins):**

```
global ◀── service ◀── route ◀── consumer ◀── route+consumer (most specific)
```

**Plugin execution order** is by `priority`. Higher priority runs first in `access` phase. Examples:

| Plugin | Priority |
|---|---|
| pre-function | 1000000 |
| ip-restriction | 990 |
| bot-detection | 2500 |
| key-auth | 1250 |
| jwt | 1450 |
| rate-limiting | 910 |
| request-transformer | 801 |
| cors | 2000 |
| acl | 950 |

**Common mistakes:**
- Putting rate-limit *before* auth at a global level → bots can exhaust your limits.
- Stacking transformers and not realizing the order matters.
- Same plugin attached at both Service and Route scope → only the Route-level one runs (most specific).

### 3.5 Upstream

**What:** A virtual hostname Kong load-balances across multiple **Targets**.

**Why:** Useful when Kong needs to **own** load balancing (instead of Kubernetes Services). Adds active health checks, weighted load balancing, hash-based stickiness.

```yaml
upstreams:
  - name: order-upstream
    algorithm: round-robin   # or least-connections, consistent-hashing
    healthchecks:
      active:
        http_path: /actuator/health
        healthy:   { interval: 5, successes: 2 }
        unhealthy: { interval: 5, http_failures: 3, tcp_failures: 3 }
    targets:
      - target: 10.0.1.5:8080
        weight: 100
      - target: 10.0.1.6:8080
        weight: 100
```

You then point a Service at the Upstream:

```yaml
services:
  - name: order-service
    host: order-upstream     # NOT a real DNS name; Kong resolves it
```

**When to use Upstream vs K8s Service?**

| | Use Kong Upstream | Use K8s Service |
|---|---|---|
| Need active health checks | ✅ | (only readiness probes) |
| Weighted blue/green | ✅ | (needs extra config) |
| Inside Kubernetes cluster | Often unnecessary | ✅ Default |
| Outside Kubernetes (VMs) | ✅ | n/a |

### 3.6 Target

**What:** A backend instance (IP/host:port + weight) inside an Upstream.

**Why:** Lets you add/remove backends dynamically and weight them.

```yaml
targets:
  - target: 10.0.1.5:8080
    weight: 80
  - target: 10.0.1.6:8080
    weight: 20   # canary instance
```

A weight of `0` removes a target from rotation (without deleting it).

---

## 4. Kong Deployment Modes

Kong supports **three** modes. Choosing wrong = pain in production.

### 4.1 DB-backed Mode (Traditional)

Kong nodes share configuration via **PostgreSQL**.

```
   ┌──────────┐  ┌──────────┐  ┌──────────┐
   │ Kong #1  │  │ Kong #2  │  │ Kong #3  │
   └────┬─────┘  └────┬─────┘  └────┬─────┘
        └────────────┬─┴────────────┘
                     ▼
              ┌──────────────┐
              │  PostgreSQL  │
              └──────────────┘
```

**Pros:** Admin API can mutate configuration dynamically. Plugins (e.g., rate-limit with local policy) keep counters per node.

**Cons:** Postgres is a hard dependency — its outage = your gateway can't update config. Not GitOps-friendly.

**Use when:** You have a UI/operator team and want to mutate config via Admin API or Konga.

### 4.2 DB-less Mode (Declarative)

Each Kong node reads a **YAML file** (kong.yml) at startup. No database.

```yaml
_format_version: "3.0"
services:
  - name: order-service
    url: http://order:8080
    routes:
      - name: order-route
        paths: [ "/orders" ]
```

**Pros:**
- Fully **GitOps** compatible (Git is the source of truth).
- No DB dependency = simpler ops and faster boot.
- Immutable, reproducible deployments.

**Cons:**
- Admin API is **read-only**.
- No dynamic mutation (must redeploy or reload).
- Some plugins (e.g., OAuth2 OSS plugin that stores tokens) require DB.

**Use when:** Kubernetes + GitOps + microservices (very common today).

### 4.3 Hybrid Mode (Control Plane + Data Plane)

The **recommended production architecture**.

```
                    ┌──────────────────────┐
                    │   Control Plane      │
                    │   (Kong + Postgres)  │
                    └──────────┬───────────┘
                               │  mTLS, push config (8005)
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
       ┌────────┐         ┌────────┐         ┌────────┐
       │  DP 1  │         │  DP 2  │         │  DP 3  │
       └────────┘         └────────┘         └────────┘
       (proxy traffic only — no DB connection)
```

**Pros:**
- DPs have **no DB**, so they boot fast and survive DB outages.
- Scale DPs independently of CP.
- Strong isolation: tenants/region-specific DPs share one CP.

**Cons:**
- More moving parts. Needs **CP↔DP mTLS** setup.

**Use when:** Any serious production with >2 Kong nodes.

### 4.4 Comparison

| | DB-backed | DB-less | Hybrid |
|---|---|---|---|
| Config source | Postgres | YAML file | CP DB → push to DPs |
| Admin API | Read/Write | Read-only | Read/Write on CP only |
| GitOps friendly | Hard | ✅ Easy | ✅ Easy (via CP) |
| Survives DB outage | ❌ | n/a | ✅ DPs keep serving |
| Simplest | ❌ | ✅ | ❌ |
| Recommended for prod | Maybe | Small/medium | Large/critical |

---

## 5. Installing & Running Kong

### 5.1 Docker (DB-less, fastest to learn)

`kong.yml`:

```yaml
_format_version: "3.0"
services:
  - name: httpbin
    url: http://httpbin.org
    routes:
      - name: httpbin-route
        paths: [ "/httpbin" ]
        strip_path: true
```

`docker-compose.yml`:

```yaml
version: "3.8"
services:
  kong:
    image: kong:3.7
    environment:
      KONG_DATABASE: "off"
      KONG_DECLARATIVE_CONFIG: /kong/declarative/kong.yml
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_ADMIN_LISTEN: "0.0.0.0:8001"
      KONG_PROXY_LISTEN: "0.0.0.0:8000, 0.0.0.0:8443 ssl"
    volumes:
      - ./kong.yml:/kong/declarative/kong.yml:ro
    ports:
      - "8000:8000"   # proxy
      - "8443:8443"   # proxy TLS
      - "8001:8001"   # admin
```

Test:

```bash
curl http://localhost:8000/httpbin/get
curl http://localhost:8001/services    # read-only Admin API
```

### 5.2 Docker (DB-backed with PostgreSQL)

```yaml
version: "3.8"
services:
  kong-db:
    image: postgres:16
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: kongpass
    volumes: [ kong-db-data:/var/lib/postgresql/data ]

  kong-migrations:
    image: kong:3.7
    command: kong migrations bootstrap
    environment: &kong-env
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: kongpass
    depends_on: [ kong-db ]

  kong:
    image: kong:3.7
    environment:
      <<: *kong-env
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_LISTEN: "0.0.0.0:8001"
      KONG_PROXY_LISTEN: "0.0.0.0:8000, 0.0.0.0:8443 ssl"
    depends_on: [ kong-db, kong-migrations ]
    ports: [ "8000:8000", "8443:8443", "8001:8001" ]
volumes:
  kong-db-data: {}
```

Create a Service + Route via Admin API:

```bash
curl -X POST http://localhost:8001/services \
  -d name=order-service -d url=http://host.docker.internal:8080

curl -X POST http://localhost:8001/services/order-service/routes \
  -d name=order-route -d paths[]=/v1/orders -d strip_path=true
```

### 5.3 Kubernetes with Helm + Kong Ingress Controller (KIC)

Install:

```bash
helm repo add kong https://charts.konghq.com
helm repo update

helm install kong kong/kong \
  --namespace kong --create-namespace \
  --set ingressController.installCRDs=false \
  --set proxy.type=LoadBalancer
```

Verify:

```bash
kubectl get pods -n kong
kubectl get svc  -n kong kong-kong-proxy
```

Define an Ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-ingress
  annotations:
    konghq.com/strip-path: "true"
spec:
  ingressClassName: kong
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080
```

### 5.4 Troubleshooting Install

- **`kong: connect: connection refused`** → Admin API listening on `127.0.0.1` only. Set `KONG_ADMIN_LISTEN=0.0.0.0:8001`.
- **`failed to bootstrap`** → migration step failed; check Postgres host/credentials.
- **In K8s, the LoadBalancer has no external IP** → cloud provider not provisioning LB; use a NodePort or MetalLB locally.
- **`502 Bad Gateway`** → upstream is unreachable from Kong's network namespace.

---

## 6. Kong with Java + Spring Boot

This section is the core for a Java engineer: how to design Spring Boot microservices and front them with Kong.

### 6.1 Architecture

```
        ┌──────────────────────────────────────────────┐
        │                  Kong                        │
        │  /v1/users/**    →  user-service             │
        │  /v1/orders/**   →  order-service            │
        │  /v1/payments/** →  payment-service          │
        └────────┬───────────────┬───────────────┬─────┘
                 ▼               ▼               ▼
          user-service     order-service    payment-service
          (Spring Boot)    (Spring Boot)    (Spring Boot)
                 │               │               │
                 ▼               ▼               ▼
              Postgres        Postgres        Postgres
```

### 6.2 Spring Boot Service (Order Service)

`pom.xml` minimal dependencies:

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  <dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
  </dependency>
</dependencies>
```

`application.yml`:

```yaml
server:
  port: 8080
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  endpoint:
    health:
      probes:
        enabled: true
```

Controller:

```java
@RestController
@RequestMapping("/orders")
public class OrderController {

    @GetMapping("/{id}")
    public Order get(@PathVariable Long id,
                     @RequestHeader(value = "X-Consumer-Username", required = false) String consumer,
                     @RequestHeader(value = "X-Request-ID", required = false) String requestId) {
        return new Order(id, "PENDING", consumer);
    }

    @PostMapping
    public ResponseEntity<Order> create(@RequestBody Order order) {
        return ResponseEntity.status(201).body(order);
    }
}
```

**Key insight:** Spring Boot services **never** validate the JWT themselves in this model — Kong does. The service **trusts** the gateway and reads identity from `X-Consumer-Username` and other Kong-injected headers. To be safe, restrict service ingress so **only Kong** can reach it (NetworkPolicy / ClusterIP).

### 6.3 Dockerfile

```dockerfile
FROM eclipse-temurin:21-jre
COPY target/order-service-1.0.0.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app/app.jar"]
```

### 6.4 Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: order-service, namespace: apps }
spec:
  replicas: 3
  selector: { matchLabels: { app: order-service } }
  template:
    metadata: { labels: { app: order-service } }
    spec:
      containers:
        - name: order
          image: ghcr.io/example/order-service:1.0.0
          ports: [ { containerPort: 8080 } ]
          readinessProbe:
            httpGet: { path: /actuator/health/readiness, port: 8080 }
          livenessProbe:
            httpGet: { path: /actuator/health/liveness, port: 8080 }
---
apiVersion: v1
kind: Service
metadata: { name: order-service, namespace: apps }
spec:
  selector: { app: order-service }
  ports: [ { port: 8080, targetPort: 8080 } ]
```

### 6.5 Kong Configuration for Multiple Spring Boot Services (DB-less)

```yaml
_format_version: "3.0"

services:
  - name: user-service
    url: http://user-service.apps.svc.cluster.local:8080
    routes:
      - name: user-route
        paths: [ "/v1/users" ]
        strip_path: true

  - name: order-service
    url: http://order-service.apps.svc.cluster.local:8080
    routes:
      - name: order-route
        paths: [ "/v1/orders" ]
        strip_path: true

  - name: payment-service
    url: http://payment-service.apps.svc.cluster.local:8080
    routes:
      - name: payment-route
        paths: [ "/v1/payments" ]
        strip_path: true

plugins:
  - name: correlation-id
    config: { header_name: X-Request-ID, generator: uuid, echo_downstream: true }
  - name: prometheus
```

### 6.6 Request Flow

```
Client GET /v1/orders/42
   │
   ▼
Kong matches Route "order-route" (path=/v1/orders)
   │
   ▼
Kong runs plugins in `access` phase: auth, rate-limit, transformer
   │
   ▼
Kong resolves Service order-service → http://order-service:8080
   │
   ▼
Kong strips /v1/orders → forwards GET /42 to upstream
   │
   ▼
Spring Boot receives GET /orders/42 (NOTE: strip_path strips only the matched
prefix; /v1/orders/42 → /42, so the controller must map / instead of /orders)
```

> **Pitfall:** With `strip_path: true`, the prefix is removed. Either disable `strip_path` or design your Spring controllers to not include the version prefix.

### 6.7 Spring Cloud Gateway vs Kong?

Many Spring teams use **Spring Cloud Gateway (SCG)** instead. Comparison:

- **SCG**: Same JVM, easy to write Java filters, less network hop, but **the gateway is yours to scale and patch**, and plugins must be written in Java.
- **Kong**: Platform-team gateway, decoupled from app teams, far richer plugin ecosystem, polyglot. Better for organizations with many services in many languages.

A **hybrid** is also valid: Kong at the edge → SCG per business domain → microservices.

---

## 7. Authentication & Authorization

### 7.1 How Kong Authenticates

Auth plugins run during the `access` phase. They:

1. Extract a credential (header, cookie, query param).
2. Validate it (signature, DB lookup, cache).
3. Find / set the **Consumer**.
4. Inject identity headers (`X-Consumer-ID`, `X-Consumer-Username`, `X-Authenticated-Userid`, JWT claims via plugins like `jwt-signer`).
5. Optionally short-circuit with 401 if validation fails.

### 7.2 Key-Auth

Simple shared secret. Great for **machine-to-machine** internal APIs.

```yaml
plugins:
  - name: key-auth
    service: order-service
    config:
      key_names: [ apikey ]
      hide_credentials: true   # strip header before sending upstream

consumers:
  - username: partner-app
    keyauth_credentials:
      - key: "abc-123-very-secret"
```

Client: `curl -H "apikey: abc-123-very-secret" https://api/v1/orders`

**Pitfalls:** Rotating keys requires a strategy (multiple keys per Consumer). Keys leak via logs — set `hide_credentials: true`.

### 7.3 Basic-Auth

Stored hashed. Useful for legacy systems. Not for public APIs.

### 7.4 JWT Plugin

Use when clients **already** receive JWTs from your IdP. Kong validates signature, exp, iss, aud.

```yaml
plugins:
  - name: jwt
    service: order-service
    config:
      claims_to_verify: [ exp ]
      key_claim_name: iss
      run_on_preflight: true

consumers:
  - username: mobile-app
    jwt_secrets:
      - key: "https://idp.example.com/realms/prod"   # iss claim
        algorithm: RS256
        rsa_public_key: |
          -----BEGIN PUBLIC KEY-----
          MIIBIjANBgkqhki...
          -----END PUBLIC KEY-----
```

How Kong matches: the JWT's `iss` claim must equal a Consumer's `jwt_secret.key`. That tells Kong **which Consumer is calling**.

**Common mistake:** Letting clients send unsigned JWTs (alg=none). Always specify `algorithm` and reject mismatches.

### 7.5 OAuth2 (OSS plugin)

Kong's OSS OAuth2 plugin actually **issues** tokens (Kong becomes the authorization server). In most modern setups you do NOT want this — you want Kong to **validate** tokens issued by Keycloak/Auth0/Okta. For that:

- Use the **JWT plugin** (validate signed JWTs locally).
- Or use the **Enterprise OIDC plugin** (full OAuth2/OIDC flow, introspection).
- Or use the community **kong-oidc** plugin (popular OSS alternative — review carefully for production use).

### 7.6 OIDC Pattern with Keycloak (Conceptual)

```
Client ──login──▶ Keycloak ──issues JWT──▶ Client
Client ──GET /v1/orders + Bearer JWT──▶ Kong
   Kong validates JWT (jwt plugin or OIDC plugin)
   Kong injects X-Consumer-Username, X-Authenticated-Userid
   Kong forwards to order-service
order-service reads identity from headers (does NOT re-validate)
```

### 7.7 ACL Plugin (Authorization)

After auth, **authorization** is who-can-do-what. The ACL plugin gates routes by Consumer groups.

```yaml
consumers:
  - username: partner-app
    acls: [ { group: trusted-partners } ]

plugins:
  - name: acl
    route: order-route
    config:
      allow: [ trusted-partners ]
```

### 7.8 Security Best Practices

- **Never** rely on the client to set `X-Consumer-*` headers — strip them at ingress.
- Always pair auth plugins with **rate-limit** and **bot-detection** plugins.
- Use **mTLS** between Kong and upstream for sensitive services.
- Disable the **Admin API** from being publicly reachable. Use a separate ingress + auth.
- Rotate JWT keys via JWKS endpoints (Enterprise OIDC) instead of static keys.
- Use `hide_credentials: true` so plaintext secrets never reach upstream logs.

---

## 8. Rate Limiting & Traffic Control

### 8.1 Why Rate Limit?

- Prevent abuse / DoS
- Enforce paid plan tiers (free vs pro)
- Protect downstream from thundering herds
- Stop bots scraping APIs

### 8.2 Strategies

| Strategy | Description | Use case |
|---|---|---|
| Fixed window | Counter resets every N seconds | Simple, slightly bursty |
| Sliding window | Continuous time window | Smoother throttling |
| Token bucket | Tokens refill at fixed rate | Allows bursts |
| Leaky bucket | Fixed outflow rate | Strict shaping |

Kong's `rate-limiting` plugin uses **fixed-window** counters across multiple time buckets (`second`, `minute`, `hour`, `day`).

### 8.3 Policies (where counters live)

| Policy | Counter store | Pros | Cons |
|---|---|---|---|
| `local` | Per-Kong-node memory | Fastest | Inaccurate when scaled out |
| `cluster` | Postgres (DB-mode only) | Cluster-wide accurate | Slower; DB load |
| `redis` | Redis | Cluster-wide accurate, fast | Needs Redis |

**Production recommendation:** Use `redis` policy with a dedicated Redis cluster.

### 8.4 Examples

**Per-Consumer (per API key):**

```yaml
plugins:
  - name: rate-limiting
    route: order-route
    config:
      minute: 60
      hour: 1000
      policy: redis
      redis_host: redis.kong.svc.cluster.local
      fault_tolerant: true       # if Redis down, allow request
      hide_client_headers: false # send X-RateLimit-* headers
```

**By IP (no Consumer):**

```yaml
plugins:
  - name: rate-limiting
    route: public-route
    config:
      minute: 30
      limit_by: ip
      policy: redis
```

**Advanced (sliding window) – `rate-limiting-advanced` (Enterprise):**

```yaml
plugins:
  - name: rate-limiting-advanced
    config:
      limit: [ 60, 1000 ]
      window_size: [ 60, 3600 ]
      sync_rate: 0.5
      strategy: redis
      window_type: sliding
```

### 8.5 Trade-offs

- **`fault_tolerant: true`** = availability > strict limiting. Wise default.
- **`fault_tolerant: false`** = security > availability. Use for paid quota enforcement.
- **`local` policy** is fine if you have just 1–2 Kong nodes.
- Don't put rate-limit *before* auth — bots will hit your limits anonymously.

### 8.6 Other Traffic Control Plugins

- **`request-size-limiting`** — block requests > N MB.
- **`bot-detection`** — block by User-Agent regex.
- **`ip-restriction`** — allow/deny lists.
- **`proxy-cache`** — short-TTL response caching for idempotent GETs.

---

## 9. Load Balancing & Service Discovery

### 9.1 Upstreams & Targets Recap

Kong can load-balance across **Targets** within an **Upstream**:

```yaml
upstreams:
  - name: order-upstream
    algorithm: least-connections
    healthchecks:
      active:
        type: http
        http_path: /actuator/health
        timeout: 1
        healthy:   { interval: 5, successes: 2, http_statuses: [200,201] }
        unhealthy: { interval: 5, http_failures: 3, tcp_failures: 3, timeouts: 3 }
      passive:
        healthy:   { http_statuses: [200,201,202], successes: 5 }
        unhealthy: { http_statuses: [500,502,503,504], http_failures: 5 }
    targets:
      - target: 10.0.1.5:8080
        weight: 100
      - target: 10.0.1.6:8080
        weight: 100
```

### 9.2 Load Balancing Algorithms

- **round-robin** — default; even distribution.
- **least-connections** — best when request latency varies.
- **consistent-hashing** — sticky by client IP, header, cookie, or consumer. Critical for stateful workloads (e.g., WebSockets).

```yaml
upstreams:
  - name: ws-upstream
    algorithm: consistent-hashing
    hash_on: consumer
```

### 9.3 Active vs Passive Health Checks

| | Active | Passive |
|---|---|---|
| How | Kong actively probes target | Kong observes real traffic |
| Cost | Extra requests | Free |
| Detects pre-prod issues | ✅ | ❌ (requires real traffic) |
| Required for cold start | ✅ | ❌ |

Combine both: active probes for startup + passive for live traffic anomalies.

### 9.4 Retries

```yaml
services:
  - name: order-service
    retries: 5                # max retries on failure
    connect_timeout: 2000
    write_timeout: 60000
    read_timeout: 60000
```

Retries kick in on **TCP connect failures** or non-idempotent 502/503 (depending on plugin config). **Be careful** — auto-retrying POSTs is dangerous if the upstream is non-idempotent.

### 9.5 Inside Kubernetes — Do You Need Upstreams?

If your services are exposed as Kubernetes Services, kube-proxy already round-robins to pods. You typically don't need Kong Upstreams. **You do** need them when:
- You want active L7 health checks on a path other than the readiness probe.
- You want weighted blue/green or canary at the gateway level.
- You're load-balancing to external IPs (VMs, serverless functions).

---

## 10. Observability & Monitoring

A gateway you can't observe is **a black hole** in your production architecture.

### 10.1 Logging

Kong has many log plugins:

- **`file-log`** — JSON to a file.
- **`http-log`** — POST JSON to an HTTP endpoint (Logstash, Loki, Splunk).
- **`tcp-log` / `udp-log`** — raw TCP/UDP sinks.
- **`syslog`** — to syslog facility.

Example:

```yaml
plugins:
  - name: http-log
    config:
      http_endpoint: http://logstash.observability:8080
      method: POST
      timeout: 10000
      keepalive: 60000
      flush_timeout: 2
      queue_size: 1000
```

**Pro tip:** Use `queue_size` and `flush_timeout` to batch logs so the gateway never blocks.

### 10.2 Metrics — Prometheus Plugin

```yaml
plugins:
  - name: prometheus
    config:
      status_code_metrics: true
      latency_metrics: true
      bandwidth_metrics: true
      upstream_health_metrics: true
```

Scrape config:

```yaml
- job_name: kong
  static_configs:
    - targets: [ "kong-kong-admin:8001" ]
  metrics_path: /metrics
```

Key Kong metrics:

| Metric | Meaning |
|---|---|
| `kong_http_status` | Counter by status code |
| `kong_latency_ms_bucket` | Histogram: total proxy latency |
| `kong_upstream_latency_ms_bucket` | Histogram: upstream time only |
| `kong_kong_latency_ms_bucket` | Histogram: time spent in Kong itself |
| `kong_bandwidth_bytes` | Bytes sent/received |
| `kong_nginx_connections_total` | Active / waiting / handled |

**Golden formula for "is Kong slow?":**
`kong_kong_latency` should be < 5 ms p99. If it grows, **Kong itself** is the bottleneck (plugins, DNS, Lua GC). If `kong_upstream_latency` is high → upstream is slow.

### 10.3 Grafana Dashboard Essentials

Display:
- Request rate per Service / Route
- Error rate (4xx, 5xx) per Service
- p50, p95, p99 latency for `kong_latency` and `upstream_latency`
- Connection counts
- Per-Consumer rate limit usage
- Upstream health status (per target)

Kong publishes a public dashboard on Grafana.com (ID 7424 for the classic one).

### 10.4 Distributed Tracing

Kong supports **OpenTelemetry** natively:

```yaml
plugins:
  - name: opentelemetry
    config:
      endpoint: http://otel-collector.observability:4318/v1/traces
      resource_attributes:
        service.name: kong
      headers:
        X-Custom: kong
```

This sends spans to an OTel collector → Jaeger / Tempo / Datadog. Spring Boot services using `micrometer-tracing` join the same trace via the `traceparent` header propagated by Kong.

### 10.5 Real Debugging Workflow

> "API latency p99 jumped from 80ms to 600ms at 14:02 UTC."

1. **Grafana**: Is `kong_kong_latency` or `kong_upstream_latency` the culprit? → if upstream, jump to step 4.
2. If Kong's own latency: check `nginx_connections_active`, plugin error rates, DNS cache hit rate.
3. Check Kong pod CPU; check `lua_shared_dict` saturation.
4. If upstream: find which Service. Cross-check `kong_upstream_latency{service=...}`.
5. Jump to Spring Boot service metrics (HikariCP pool, GC pauses, slow SQL).
6. Use the distributed trace from step 1 to find the slowest span.

---

## 11. Kubernetes Integration

### 11.1 Kong Ingress Controller (KIC)

KIC watches Kubernetes resources (`Ingress`, `Service`, `KongPlugin`, `KongConsumer`) and translates them into Kong configuration.

```
┌────────────────────────────────┐
│   Kong Ingress Controller      │
│   (watches K8s API)            │
└─────────────┬──────────────────┘
              │ updates
              ▼
        ┌──────────┐
        │   Kong   │   ◀── traffic from clients
        └──────────┘
```

### 11.2 Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-ingress
  annotations:
    konghq.com/strip-path: "true"
    konghq.com/plugins: "rate-limit-orders,jwt-auth"
spec:
  ingressClassName: kong
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/orders
            pathType: Prefix
            backend:
              service: { name: order-service, port: { number: 8080 } }
```

### 11.3 KongPlugin CRD

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongPlugin
metadata:
  name: rate-limit-orders
config:
  minute: 60
  policy: redis
  redis_host: redis.kong.svc.cluster.local
plugin: rate-limiting
```

Attach by annotation on Ingress, Service, or KongConsumer.

### 11.4 KongConsumer & KongCredential

```yaml
apiVersion: configuration.konghq.com/v1
kind: KongConsumer
metadata:
  name: mobile-app
  annotations: { kubernetes.io/ingress.class: kong }
username: mobile-app
custom_id: mobile-prod
credentials: [ mobile-app-key ]
---
apiVersion: v1
kind: Secret
metadata: { name: mobile-app-key }
stringData:
  kongCredType: key-auth
  key: super-secret
```

### 11.5 Gateway API (Modern Approach)

KIC supports the **Gateway API** (replacing Ingress long-term):

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata: { name: order-route }
spec:
  parentRefs: [ { name: kong-gateway } ]
  hostnames: [ "api.example.com" ]
  rules:
    - matches: [ { path: { type: PathPrefix, value: /v1/orders } } ]
      backendRefs:
        - name: order-service
          port: 8080
```

Prefer Gateway API for **new** deployments — it's cleaner and standardized across ingress vendors.

### 11.6 Internal Networking

```
Client ──► LB ──► Kong Pod (NGINX) ──► kube-proxy iptables ──► Target Pod
                       │
                       └─ DNS lookup (CoreDNS)
                           cached by Kong (60s)
```

Implications:
- Kong **caches DNS** — a recently-killed pod may still be a target for ~60s. Reduce `KONG_DNS_STALE_TTL` if needed.
- Kubernetes Services give Kong a single virtual IP; **Kong does not see pod-level health**, only the Service. Use Kong Upstreams + KongIngress if you need per-pod control.

### 11.7 TLS Termination

Kong can terminate TLS at the edge using a Kubernetes Secret:

```yaml
spec:
  tls:
    - hosts: [ api.example.com ]
      secretName: api-example-com-tls
```

Use **cert-manager** to provision certs from Let's Encrypt:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata: { name: api-example-com }
spec:
  secretName: api-example-com-tls
  issuerRef: { name: letsencrypt-prod, kind: ClusterIssuer }
  dnsNames: [ api.example.com ]
```

---

## 12. TLS & Security

### 12.1 HTTPS Termination

Kong can:
- **Terminate TLS** at the edge (most common).
- **Passthrough TLS** to upstream (rare; loses L7 visibility).
- **Re-encrypt** to upstream (proxy_ssl). Best of both.

`KONG_PROXY_LISTEN`:

```
0.0.0.0:8000, 0.0.0.0:8443 ssl http2
```

Add a cert via Admin API:

```bash
curl -X POST http://localhost:8001/certificates \
  -F "cert=@/etc/ssl/api.example.com.crt" \
  -F "key=@/etc/ssl/api.example.com.key" \
  -F "snis=api.example.com"
```

### 12.2 Re-encrypt to Upstream

```yaml
services:
  - name: secure-service
    url: https://secure-service.apps.svc.cluster.local:8443
    client_certificate:    # mTLS to upstream
      id: <cert-uuid>
```

### 12.3 mTLS — Client-to-Kong

Use the **`mtls-auth`** plugin (Enterprise) or community alternatives. With OSS, terminate mTLS at an outer NGINX or use `KONG_NGINX_PROXY_SSL_VERIFY_CLIENT=on` for raw mTLS.

```yaml
plugins:
  - name: mtls-auth
    route: partner-route
    config:
      ca_certificates: [ <ca-uuid> ]
      skip_consumer_lookup: false
```

### 12.4 Secure Headers

```yaml
plugins:
  - name: response-transformer
    config:
      add:
        headers:
          - "Strict-Transport-Security:max-age=31536000; includeSubDomains"
          - "X-Content-Type-Options:nosniff"
          - "X-Frame-Options:DENY"
          - "Referrer-Policy:strict-origin-when-cross-origin"
```

### 12.5 Production Security Practices

- TLS 1.2 minimum (`KONG_SSL_PROTOCOLS=TLSv1.2 TLSv1.3`).
- Disable weak ciphers.
- Forbid `HTTP` proxy listen in production (HTTPS only, or redirect to HTTPS).
- Isolate Admin API: bind to a private network or expose only via authenticated ingress.
- Use **secret managers** (Vault, AWS Secrets Manager) via Kong's vault references rather than env vars.
- Enable WAF in front of Kong (AWS WAF, Cloudflare, ModSecurity).
- Rotate certs automatically (cert-manager).
- Network policies that restrict Kong → upstream connections.
- Audit log all Admin API changes.

---

## 13. Advanced Kong Plugins

### 13.1 CORS

```yaml
plugins:
  - name: cors
    config:
      origins: [ "https://app.example.com" ]
      methods: [ GET, POST, PUT, DELETE, OPTIONS ]
      headers: [ Accept, Authorization, Content-Type, X-Request-ID ]
      exposed_headers: [ X-RateLimit-Remaining ]
      credentials: true
      max_age: 3600
      preflight_continue: false
```

**Pitfall:** `origins: ["*"]` + `credentials: true` is **invalid** per CORS spec; browsers will reject. Be explicit.

### 13.2 Request Transformer

```yaml
plugins:
  - name: request-transformer
    route: order-route
    config:
      add:
        headers: [ "X-Tenant: acme-corp" ]
        querystring: [ "trace=true" ]
      remove:
        headers: [ "Cookie" ]
      replace:
        headers: [ "User-Agent: kong-proxy" ]
```

### 13.3 Response Transformer

```yaml
plugins:
  - name: response-transformer
    config:
      remove:
        headers: [ "Server", "X-Powered-By" ]
      add:
        json:
          - "powered_by:kong"
```

### 13.4 ACL

```yaml
plugins:
  - name: acl
    route: admin-route
    config:
      allow: [ admins ]
      hide_groups_header: true
```

### 13.5 Proxy Cache

```yaml
plugins:
  - name: proxy-cache
    route: catalog-route
    config:
      strategy: memory
      content_type: [ "application/json" ]
      cache_ttl: 30
      cache_control: false
      request_method: [ GET, HEAD ]
      response_code: [ 200 ]
```

**Pitfalls:** Cache key includes path + method + query, not headers. If your response varies by Authorization header, caching by default leaks data. Set `vary_headers` carefully or **don't cache authenticated responses**.

### 13.6 Plugin Execution Order Pitfalls

If two plugins of the same type are attached at different scopes (e.g., global + route), **only the most specific runs**. To compose, attach **different** plugin instances.

To override priority globally: set `KONG_PLUGINS_ORDERED=...` (Enterprise) or use the new `ordering.before/after` field (3.x).

```yaml
plugins:
  - name: rate-limiting
    ordering:
      before:
        access: [ key-auth ]   # run rate-limit BEFORE key-auth (unusual)
```

---

## 14. Performance & Scaling

### 14.1 Kong Performance Characteristics

- A well-tuned Kong node handles **20k–50k req/s** of simple proxying on a single 4-core VM.
- Each NGINX worker = 1 CPU core. Set `nginx_worker_processes: auto`.
- Each plugin adds latency. Auth + rate-limit + log → typically 1–2 ms.
- Kong's "warm" steady-state p99 latency overhead is **< 5 ms** for cached routes.

### 14.2 Horizontal Scaling

- **Stateless data plane** — scale by adding more Kong pods.
- Place Kong behind a Kubernetes Service (`type=LoadBalancer`) → cloud LB distributes.
- For very high traffic, use **multiple LBs** in different regions and DNS-based geo routing.

### 14.3 Caching Behavior

Kong caches:
- **Router** (parsed routes) — invalidated on config change.
- **DNS lookups** — TTL configurable (`KONG_DNS_HOSTSFILE`, `KONG_DNS_STALE_TTL`).
- **Plugin config** — cached in `lua_shared_dict kong`.
- **Database entities** (DB mode) — cached in `lua_shared_dict kong_db_cache`.

Tune `mem_cache_size`:

```
KONG_MEM_CACHE_SIZE=512m
```

### 14.4 Worker Processes & Connections

```
KONG_NGINX_WORKER_PROCESSES=auto
KONG_NGINX_WORKER_CONNECTIONS=16384
```

Workers are bound to cores. Set CPU requests/limits on the pod accordingly.

### 14.5 Upstream Keepalive

Keep connections to upstream alive (avoid TCP handshake per request):

```
KONG_NGINX_UPSTREAM_KEEPALIVE=320
KONG_NGINX_UPSTREAM_KEEPALIVE_REQUESTS=10000
KONG_NGINX_UPSTREAM_KEEPALIVE_TIMEOUT=60s
```

### 14.6 NGINX Tuning Cheatsheet

```
KONG_NGINX_HTTP_CLIENT_MAX_BODY_SIZE=10m
KONG_NGINX_HTTP_CLIENT_BODY_BUFFER_SIZE=128k
KONG_NGINX_PROXY_PROXY_BUFFER_SIZE=128k
KONG_NGINX_PROXY_PROXY_BUFFERS="8 128k"
KONG_NGINX_HTTP_LARGE_CLIENT_HEADER_BUFFERS="4 16k"
```

### 14.7 Memory & GC

LuaJIT can suffer GC stutter under heavy load. Solutions:

- Avoid plugins that allocate large tables in `access` phase.
- Set `KONG_LUA_PACKAGE_PATH` to keep modules small.
- Monitor `lua_shared_dict` usage via the Prometheus plugin (`kong_memory_lua_shared_dict_bytes`).

### 14.8 Production Considerations

- Run **3+ Kong pods** for redundancy.
- Spread across AZs (`topologySpreadConstraints`).
- Set `requests` and `limits` equal to avoid CPU throttling on QoS Burstable.
- Use a **dedicated node pool** for Kong if it's a bottleneck.
- Enable HPA on CPU and `kong_nginx_connections_active`.

---

## 15. Failure Scenarios & Debugging

### 15.1 Gateway Timeout (504)

**Symptom:** 504 from Kong, upstream eventually responds (after timeout).

**Causes:**
- Upstream is genuinely slow.
- `read_timeout` too low.
- DB query in upstream blocked.
- NGINX worker exhaustion.

**Investigation:**
1. Check `kong_upstream_latency`; is it spiking?
2. `kubectl logs kong-pod | grep 504` → find upstream path.
3. Inspect upstream service metrics: GC pauses, slow SQL.
4. Hit upstream directly (`kubectl port-forward`) and reproduce.

**Fix:**
- Increase `read_timeout` only as a band-aid.
- Real fix: profile and improve upstream latency.
- Add circuit-breaker / retry caps.

### 15.2 Upstream Unavailable (502 / 503)

**Symptom:** 502/503 from Kong.

**Causes:**
- Pods crashing.
- DNS resolution failure (CoreDNS down).
- Service has 0 endpoints.
- All targets marked unhealthy.

**Investigation:**
1. `kubectl get endpoints order-service` — any IPs?
2. Check Kong error log: `*** upstream connect refused` or `no resolver`.
3. `curl http://localhost:8001/upstreams/order-upstream/health`.

**Fix:**
- Ensure deployment is healthy.
- Check NetworkPolicy isn't blocking Kong → upstream.
- Confirm DNS works inside Kong pod: `kubectl exec kong-pod -- nslookup order-service`.

### 15.3 Authentication Failure (401)

**Symptom:** Client gets 401 even with seemingly correct token.

**Investigation:**
1. Decode JWT — verify `iss`, `exp`, `alg`.
2. Compare JWT `iss` to Consumer's `jwt_secret.key` (must match exactly).
3. Check Kong logs: `*** [jwt] no credential found` or `invalid signature`.
4. If using `key-auth`, header name mismatch is the #1 cause (`apikey` vs `api-key`).

**Fix:**
- Align `key_names` config with what the client sends.
- Refresh JWKS / rotate public key.
- Use a debug Consumer with broad scopes to isolate.

### 15.4 Misconfigured Routes

**Symptom:** Request hits the wrong Service, or returns 404 from Kong.

**Investigation:**
1. `curl http://localhost:8001/routes` — list routes; check priorities.
2. Reproduce with `curl -v http://kong/...` and inspect `Via` / `X-Kong-Route-Id` (if `KONG_HEADERS=server-tokens, latency-tokens, X-Kong-Route-Id`).
3. Conflicting paths between two routes.

**Fix:**
- Use explicit `regex_priority`.
- Tighten paths with `methods` + `hosts` to avoid overlap.
- Avoid sharing the same path across routes without method/host differentiation.

### 15.5 Rate Limit Issues

**Symptom:** Random 429 responses, or rate limit not enforcing.

**Investigation:**
1. Check `policy`: `local` is per-node; with N nodes the effective limit is N×configured.
2. Check Redis connectivity if `policy: redis`.
3. Check `fault_tolerant`: if Redis is down and this is `true`, limits are bypassed.

**Fix:**
- Switch to `redis` for accurate cross-node limits.
- Monitor Redis health with the same alerting as Kong.

### 15.6 Plugin Order Issues

**Symptom:** Auth happens after rate-limit (bots exhaust limits anonymously).

**Investigation:**
1. List plugins by route: `curl http://localhost:8001/routes/<id>/plugins`.
2. Cross-reference priorities.

**Fix:** Use `ordering.before` / `ordering.after` (3.x), or set plugins at the right scope.

### 15.7 General Debugging Toolkit

- `KONG_LOG_LEVEL=debug` (temporarily; very verbose).
- `kong health` — check internal health.
- `kong config parse kong.yml` — validate declarative config.
- `kong config db_export` — dump current config (DB mode).
- `curl http://localhost:8001/status` — connections, workers.
- `kubectl exec kong-pod -- /bin/bash` → `tail -f /usr/local/kong/logs/error.log`.

---

## 16. Kong vs Alternatives

### 16.1 Kong vs NGINX

| | Kong | Plain NGINX |
|---|---|---|
| API-level config | ✅ Services/Routes/Plugins | Hand-written nginx.conf |
| Plugins | Hot-loadable | Reload required |
| Admin API | ✅ | ❌ |
| Auth, rate-limit, etc | Out of the box | Build yourself |
| Footprint | Heavier (Lua, OpenResty) | Tiny |

Kong **is** NGINX underneath, with a control plane and ecosystem.

### 16.2 Kong vs Spring Cloud Gateway

| | Kong | Spring Cloud Gateway |
|---|---|---|
| Runtime | OpenResty (LuaJIT) | JVM (Reactor) |
| Plugin model | Lua / Go / JS | Java filters |
| Language fit | Polyglot | Java teams |
| Ops | Platform team owns it | App team owns it |
| Performance | Higher RPS / lower latency | Good but JVM-bound |
| Maturity / ecosystem | Larger | Spring-only |

Use **SCG** if you're a Spring-only shop and want gateway logic in the same repo as code. Use **Kong** for a true platform gateway across many languages/teams.

### 16.3 Kong vs Traefik

| | Kong | Traefik |
|---|---|---|
| Origin | API gateway | Ingress / edge router |
| Plugin ecosystem | Larger, mature | Smaller (Yaegi-based plugins) |
| Configuration | Declarative + Admin API | Watches K8s/Docker |
| Best at | Heavy API policy, rate limit, auth | Simple ingress + Let's Encrypt |
| Maturity for enterprise | Higher | Good for small-medium |

Traefik is excellent as a **basic ingress + auto-cert**. Kong wins when you need rich policies, consumers, advanced rate limiting.

### 16.4 Kong vs Istio Gateway

Istio Gateway is **part of a service mesh**; it terminates external traffic into the mesh. Comparing:

- Istio Gateway is great at **mesh-aware** north-south (mTLS, traffic shifting).
- Kong is better at **API-level features**: consumers, JWT, rate limit, plugin ecosystem.
- Common pattern: **Kong (edge)** → **Istio (in-cluster mesh)**.

### 16.5 When Kong is the Best Choice

- Polyglot microservices.
- You need a **platform-level gateway** owned by a platform team.
- You need rich plugins (rate limit, auth, transformation, ACL) out of the box.
- Hybrid CP/DP makes sense for compliance/region isolation.

### 16.6 When NOT to Use Kong

- You have **one service** and a small team — overkill.
- You need ultra-low latency where adding 1–3 ms hurts.
- You need mesh features (mTLS everywhere, retries between services) → use Istio.
- You're already deep in cloud-native gateways (AWS API Gateway, Apigee).

---

## 17. API Gateway Design Best Practices

### 17.1 API Versioning

- **Path-based:** `/v1/...`, `/v2/...` — simplest, gateway-friendly.
- **Header-based:** `Accept: application/vnd.example.v2+json` — flexible but harder to cache.
- **Subdomain-based:** `v2.api.example.com` — clean separation but cert complexity.

**Recommendation:** Path-based. Route by `/vN/...` and run different upstream Services per version.

### 17.2 Gateway Responsibilities (Keep It Lean)

The gateway should:
- ✅ Authenticate
- ✅ Rate limit
- ✅ Route
- ✅ Inject correlation IDs / trace headers
- ✅ Terminate TLS
- ✅ Log access

The gateway should **not**:
- ❌ Contain business logic
- ❌ Aggregate responses from many services (use a BFF instead)
- ❌ Store user data
- ❌ Reach into databases

### 17.3 Avoiding Gateway Bloat

- Each plugin adds latency. Audit plugin lists per route.
- Don't add plugins "just in case." Add them when you need them.
- Periodically purge unused Services and Routes (config drift = security risk).

### 17.4 Security Boundaries

- The **gateway is the perimeter**. Treat upstreams as trusted.
- Use **NetworkPolicy / mesh** to forbid upstreams from being reached directly.
- The gateway **strips** sensitive headers from clients (`X-Internal-*`, `X-Consumer-*`).

### 17.5 API Lifecycle

- Use **deprecation headers** (`Sunset: ...`) at the gateway for old versions.
- Use **versioned consumer plans** (free, pro, enterprise) → different rate limits.
- Use **traffic shifting** at the gateway for blue/green.

---

## 18. Real-World Architectures

### 18.1 Microservices Platform (Standard SaaS)

```
    Internet
       │
       ▼
  ┌─────────┐
  │  CDN    │  (static, cache)
  └────┬────┘
       ▼
  ┌─────────┐
  │   WAF   │  (Cloudflare / AWS WAF)
  └────┬────┘
       ▼
  ┌────────────┐
  │  Kong DP   │  (3 replicas, mTLS to CP)
  └────┬───────┘
       │   Kong CP (separate cluster, Postgres)
       ▼
  ┌──────────────────────────────────────────────┐
  │   Kubernetes (apps namespace)                │
  │   user-svc   order-svc   payment-svc   ...   │
  └──────────────────────────────────────────────┘
```

**Trade-offs:** Hybrid mode = max resilience. Multiple plugins (jwt, rate-limit, prometheus, otel). Per-tenant API keys.

### 18.2 Fintech API Platform

```
  Partner Bank  ──mTLS──▶  Kong (mtls-auth + jwt + ip-restriction)
                                │
                                ▼
                       Spring Boot core banking
                                │
                                ├─▶ KYC service
                                ├─▶ Ledger
                                └─▶ Fraud (sync, low-latency)
```

**Special concerns:**
- mTLS to clients (partner banks).
- WAF + IP allowlist before Kong.
- All traffic logged for audit (immutable bucket).
- Strict rate limits per partner Consumer.

### 18.3 Internal Enterprise APIs

```
  Internal apps  ──▶  Kong (SSO via OIDC + ACL by group)  ──▶  HR API
                                                          ──▶  Finance API
                                                          ──▶  Identity API
```

**Trade-offs:**
- OIDC plugin integrates with corporate IdP (Okta/Azure AD).
- ACL by AD group → multiple Consumers mapping to groups.
- No public exposure; private LB only.

### 18.4 Public Developer API Platform

```
  Devs ──▶ Dev Portal (sign up, get API key) ──▶ Kong
                                                 │
                                                 ├─▶ V1 API (deprecated)
                                                 ├─▶ V2 API (current)
                                                 └─▶ V3 API (beta)
```

**Trade-offs:**
- Dev Portal (Enterprise) handles self-service.
- Versioned routes, soft deprecation headers.
- Per-plan rate limits (free / pro / enterprise) using ACL groups.
- Detailed usage analytics (per-Consumer metrics).

### 18.5 Multi-Region with Hybrid Mode

```
    Region A                       Region B
  ┌──────────┐                   ┌──────────┐
  │ Kong DP  │                   │ Kong DP  │
  └────┬─────┘                   └────┬─────┘
       │   mTLS push config            │
       └────────────┬──────────────────┘
                    ▼
              ┌──────────┐
              │ Kong CP  │   (global, with replicated Postgres)
              └──────────┘
```

DPs survive WAN outages because config is cached locally.

---

## 19. Common Edge Cases Senior Engineers Must Know

### 19.1 Header Propagation Issues

**Problem:** Client sends `X-User-Id: admin` and upstream trusts it.

**Cause:** Kong forwards client headers by default.

**Fix:** Strip suspicious headers at the gateway:

```yaml
plugins:
  - name: request-transformer
    config:
      remove:
        headers: [ "X-User-Id", "X-Consumer-Id", "X-Internal-*" ]
```

Auth plugins should **set** trusted headers, not allow them through.

### 19.2 Infinite Redirect Loops

**Problem:** Upstream returns `Location: /v1/orders/42`, client follows, hits Kong again, but path now mismatches.

**Cause:** `strip_path: true` strips on inbound but not in upstream Location header.

**Fix:**
- Disable strip_path (let upstream see `/v1/orders/42`).
- Or use `response-transformer` to rewrite Location headers.

### 19.3 Incorrect Route Matching

**Problem:** `/v1/orders/123` hits the wrong service because two routes overlap.

**Cause:** Kong picks the first route by priority. Two routes with overlapping paths and no explicit priority cause undefined-looking behavior.

**Fix:**
- Add `methods` to disambiguate.
- Add `hosts` to disambiguate.
- Use `regex_priority` to make intent explicit.

### 19.4 Plugin Execution Order

**Problem:** Logging plugin sees redacted body because transformer ran first.

**Cause:** Plugins in the same phase run by priority.

**Fix:** Use the `ordering` field (Kong 3.x):

```yaml
ordering:
  after:
    access: [ request-transformer ]
```

### 19.5 Kubernetes Ingress Conflicts

**Problem:** Two Ingresses define the same host/path; Kong picks one.

**Cause:** Multiple Ingress resources merged by KIC.

**Fix:**
- Enforce naming + path conventions per team.
- Use admission policies (OPA/Kyverno) to prevent overlapping paths.

### 19.6 TLS Handshake Failures

**Problem:** Random `SSL_ERROR_RX_RECORD_TOO_LONG` or `bad record mac`.

**Causes:**
- Mismatched cert chain (missing intermediate).
- TLS version negotiation: client TLS 1.0, Kong TLS 1.2 minimum.
- SNI mismatch.

**Fix:**
- Concatenate full cert chain in the cert file.
- Verify with `openssl s_client -servername api.example.com -connect host:443`.
- Enable `KONG_NGINX_PROXY_SSL_VERIFY=on` for upstream TLS only when CA is known.

### 19.7 WebSocket Proxying Problems

**Problem:** WebSocket disconnects after 60s.

**Cause:** Kong's default `read_timeout` is 60s.

**Fix:**

```yaml
services:
  - name: ws-service
    read_timeout: 3600000     # 1 hour
    write_timeout: 3600000
```

Also use `consistent-hashing` algorithm to keep clients on the same backend.

### 19.8 Authentication Bypass Risks

**Problem:** A new route added without an auth plugin → public exposure.

**Fix:**
- Apply auth plugin **globally** + ACL plugin to allow only specific Consumers.
- Use admission policies to require auth annotations on every Ingress.
- Run a periodic scanner that lists routes without auth plugins.

### 19.9 DNS Caching Stale Targets

**Problem:** A pod was killed but Kong still sends 5xx for 60s.

**Cause:** Kong caches DNS responses.

**Fix:**

```
KONG_DNS_STALE_TTL=2
KONG_DNS_NOT_FOUND_TTL=2
```

Or prefer Kong Upstreams with active health checks.

### 19.10 Large Request Bodies Failing

**Problem:** 413 Payload Too Large.

**Fix:**

```
KONG_NGINX_HTTP_CLIENT_MAX_BODY_SIZE=50m
```

But also enforce upstream `spring.servlet.multipart.max-file-size` to match.

---

## 20. Production Operations

### 20.1 Deployment Strategies

- **Rolling update** (default in K8s): pods replaced one by one. Safe for stateless Kong DPs. Use `maxUnavailable: 0` for zero-downtime.
- **Blue/green**: two full sets of Kong pods, switch traffic at LB. Safer for major upgrades.
- **Canary**: 5% traffic to new version using LB or multiple ingress classes.

### 20.2 Rolling Upgrades

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
```

Ensure liveness probe doesn't restart pods during config reload.

### 20.3 Canary Releases at the Gateway

Split traffic using weighted Upstream:

```yaml
upstreams:
  - name: order-upstream
    targets:
      - target: order-v1:8080
        weight: 90
      - target: order-v2:8080
        weight: 10
```

Bump v2 weight gradually while monitoring error rate.

### 20.4 Backup & Restore

- **DB mode:** Postgres dumps (`pg_dump`). Test restores quarterly.
- **DB-less:** Git repo is the backup.
- **Hybrid:** Back up the CP DB; DPs are stateless.

### 20.5 Config Management

- Keep `kong.yml` in **Git**. PR-review changes.
- CI validates with `kong config parse kong.yml`.
- Deploy with **Argo CD** or **Flux** — declarative GitOps.
- For DB-mode, use **decK** (`deck sync`, `deck diff`) to drive Kong from YAML.

```bash
deck diff --kong-addr http://kong-admin:8001 -s kong.yml
deck sync --kong-addr http://kong-admin:8001 -s kong.yml
```

### 20.6 Observability in Production

- Alert on: `kong_http_status{code=~"5.."}` rate, p99 latency, `lua_shared_dict` saturation, upstream unhealthy targets.
- Track Admin API mutations (audit log).
- Dashboards per Service / Route / Consumer.

### 20.7 Capacity Planning

- Profile: hit Kong with `wrk` / `k6` at expected RPS.
- Watch CPU per worker.
- Plan for **3× peak** capacity to absorb spikes and rolling updates.
- Use HPA on CPU + custom metric (`nginx_connections_active`).

### 20.8 Disaster Recovery

- DR plan: standby Kong cluster in another region, fed by Git or DB replication.
- DNS failover (Route 53 health checks).
- Regularly run **game days** simulating Kong outage.

---

## 21. Final Checklist

> **You are a senior-level Kong Gateway engineer if you can:**

### Architecture & Mental Model
- ✅ Explain the differences between API gateway, load balancer, and service mesh.
- ✅ Draw the request lifecycle inside Kong from network arrival to upstream and back.
- ✅ Explain NGINX phases and how Kong plugins hook into them.
- ✅ Choose between DB-backed, DB-less, and hybrid mode and justify the choice.

### Core Concepts
- ✅ Configure Services, Routes, Consumers, Plugins, Upstreams, Targets from scratch.
- ✅ Predict plugin execution order using scope + priority.
- ✅ Predict route matching given multiple overlapping routes.

### Spring Boot Integration
- ✅ Design a Spring Boot microservice that trusts Kong-injected headers and is unreachable except via Kong.
- ✅ Configure routes for multiple Spring Boot services with versioning.
- ✅ Decide when to use Spring Cloud Gateway vs Kong vs both.

### Security
- ✅ Configure JWT validation against an external IdP, including key rotation.
- ✅ Apply mTLS between clients and Kong, and between Kong and upstreams.
- ✅ Strip dangerous client headers and inject trusted identity headers.
- ✅ Lock down the Admin API.

### Kubernetes
- ✅ Install Kong with Helm, configure Ingress and KongPlugin/KongConsumer CRDs.
- ✅ Use Gateway API resources (HTTPRoute) with Kong.
- ✅ Manage cert-manager + TLS automatically.

### Observability
- ✅ Wire up the Prometheus plugin and read its key metrics fluently.
- ✅ Use OpenTelemetry to trace a request from client to Spring Boot service.
- ✅ Diagnose whether latency lies in Kong, upstream, or DNS.

### Performance & Scaling
- ✅ Tune NGINX workers, connections, and upstream keepalive.
- ✅ Choose appropriate rate-limiting policy and load-balancing algorithm.
- ✅ Scale Kong horizontally with HPA and topology spread.

### Debugging
- ✅ Diagnose 401, 404, 429, 502, 504 with clear methodology.
- ✅ Identify DNS, TLS, header, and route-matching bugs quickly.
- ✅ Use Admin API + logs + metrics together.

### Operations
- ✅ Run GitOps with decK or KIC.
- ✅ Perform safe rolling upgrades and canary releases.
- ✅ Plan for disaster recovery and capacity.

### Judgment
- ✅ Know **when not** to add a plugin.
- ✅ Know **when not** to use Kong at all.
- ✅ Treat the gateway as a critical perimeter — design accordingly.

---

### Closing Thought

A great API gateway is **invisible when it works and obvious when something is wrong**. Master the mental model: every plugin is a deliberate trade-off between security, latency, and operability. Build the smallest gateway that meets your needs, observe it ruthlessly, and let your microservices focus on what only they can do — your business logic.

> Save this file as: `kong-api-gateway-deep-dive.md`
