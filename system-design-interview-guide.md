# The Complete System Design Interview Guide

> A production-grade preparation guide for mid-level Java backend developers aiming for Big Tech (Google, Amazon, Meta, Netflix, Uber, Stripe).
>
> This guide teaches you **how to think, how to communicate, and how to design** — not just facts.

---

## Table of Contents

1. [What System Design Interviews Really Test](#1-what-system-design-interviews-really-test)
2. [The Step-by-Step Framework](#2-the-step-by-step-framework)
3. [Core Building Blocks](#3-core-building-blocks)
4. [Scalability Concepts](#4-scalability-concepts)
5. [Data Design & Storage](#5-data-design--storage)
6. [Caching Strategies](#6-caching-strategies)
7. [Communication Patterns](#7-communication-patterns)
8. [Reliability & Fault Tolerance](#8-reliability--fault-tolerance)
9. [Consistency & Distributed Systems](#9-consistency--distributed-systems)
10. [Performance Optimization](#10-performance-optimization)
11. [Real System Design Problems](#11-real-system-design-problems)
12. [Interview Communication Skills](#12-interview-communication-skills)
13. [Common Trade-offs](#13-common-trade-offs)
14. [Red Flags in Interviews](#14-red-flags-in-interviews)
15. [Preparation Strategy](#15-preparation-strategy)
16. [Final Checklist](#16-final-checklist)

---

# 1. What System Design Interviews Really Test

A system design interview is **not** a test of how many technologies you have memorized. It is a test of **engineering judgment under ambiguity**.

## 1.1 What Interviewers Actually Evaluate

| Dimension | What They're Looking For | How It Shows Up |
|-----------|--------------------------|-----------------|
| **Problem Solving** | Can you decompose a vague problem into solvable pieces? | You ask the right clarifying questions, not random ones. |
| **Trade-off Reasoning** | Can you explain *why* you chose X over Y? | You don't say "Kafka is best" — you say "Kafka because we need durable, replayable event logs at millions of msgs/sec; the trade-off is operational complexity." |
| **Scalability Thinking** | Do you think in orders of magnitude? | You convert 100M DAU into QPS, storage, and bandwidth. |
| **Communication** | Can a teammate follow your thought process? | You think aloud, draw as you talk, and summarize before deep-diving. |
| **Pragmatism** | Do you know when to stop designing? | You meet the requirements, not an imaginary future. |
| **Depth in One Area** | Do you own a component end-to-end? | When asked "how do you shard this DB?" you don't hand-wave. |

## 1.2 The Four Signals Interviewers Score

Senior interviewers at Big Tech typically score on:

1. **Requirements Gathering** — did you understand the problem before solving it?
2. **High-Level Design** — is your architecture reasonable and justified?
3. **Deep Dive** — can you design a component rigorously?
4. **Scale & Trade-offs** — do you understand the cost of your choices?

> **Key insight:** You can miss on any single dimension and still pass, but you **cannot** pass if you skip requirements gathering. It's the most common failure mode.

## 1.3 Common Mistakes Candidates Make

- **Jumping into architecture in the first 2 minutes** — you haven't defined the problem yet.
- **Name-dropping tech** — saying "I'd use Cassandra and Kafka and Kubernetes" without justifying why.
- **Ignoring scale** — designing a system as if it serves 10 users when the prompt implies 100M.
- **One-way communication** — not pausing, not checking in, not adjusting based on hints.
- **Over-engineering** — designing for 1B users when the requirement is 1M.
- **Under-engineering** — single DB, no caching, no replication, then saying "we'd scale later."
- **Refusing to commit** — "it depends" with no recommendation. Interviewers want a decision *and* its reasoning.
- **Silence while thinking** — the interviewer cannot score what they cannot hear.

## 1.4 How Senior Engineers Think Differently

| Junior Thinking | Senior Thinking |
|-----------------|-----------------|
| "What technology should I use?" | "What's the actual bottleneck I'm solving?" |
| "I'll use microservices because they scale." | "A modular monolith is fine for 1M users — microservices add complexity we don't need yet." |
| "I'll add Redis everywhere." | "I'll cache this specific hot read path because it's 80% of traffic and tolerates staleness." |
| "CAP theorem says pick 2." | "In a network partition, this system needs availability over strict consistency because users tolerate stale feeds but not downtime." |
| "I'll shard by user ID." | "I'll shard by user ID because 95% of queries are user-scoped, and this avoids cross-shard joins — the trade-off is hot-shard risk for power users, which I'd mitigate with X." |

**The senior mindset:** every decision has a cost. State the cost. Own the decision.

---

# 2. The Step-by-Step Framework

A **repeatable 7-step framework**. Use it every time. It signals structured thinking.

```
┌─────────────────────────────────────────────────────────────┐
│  1. Clarify Requirements     (5 min)                        │
│  2. Functional vs Non-Functional   (3 min)                  │
│  3. Back-of-Envelope Estimates    (5 min)                   │
│  4. High-Level Design             (10 min)                  │
│  5. Deep Dive into Components     (15 min)                  │
│  6. Identify Bottlenecks          (5 min)                   │
│  7. Trade-offs & Wrap-up          (2 min)                   │
└─────────────────────────────────────────────────────────────┘
```

Total: ~45 minutes. Leave 5 for Q&A.

---

## Step 1: Clarify Requirements (5 min)

**Goal:** Reduce ambiguity. Define scope. Surface constraints.

### What to think internally
> "What is the *core* use case? What am I explicitly *not* building? Who are the users? What's the expected scale?"

### What to say in the interview
- "Before I design anything, I want to clarify a few things. Is it OK if I take a few minutes to scope the problem?"
- "Who are the primary users? End users? Internal services?"
- "What are the core features? Can I assume [X] is out of scope for this interview?"
- "Are there specific SLA or latency requirements?"
- "Is this a read-heavy or write-heavy system?"

### Example phrases
- *"Let me make sure I understand — you want a system that does X, Y, Z. Is there anything else I'm missing?"*
- *"I'll assume we're building for web and mobile clients. Let me know if that's wrong."*
- *"I'm going to scope out analytics and admin dashboards for now. Fair?"*

### Checklist
- [ ] Core features listed
- [ ] Users identified
- [ ] Out-of-scope items stated
- [ ] Read/write ratio understood

---

## Step 2: Functional vs Non-Functional Requirements (3 min)

### Functional (what the system does)
- User can shorten a URL
- User can view a shortened URL
- URLs expire after N years

### Non-functional (how well it does it)
- **Availability:** 99.99% uptime
- **Latency:** p99 < 100ms for reads
- **Scalability:** 100M DAU, 1000:1 read:write ratio
- **Consistency:** eventual consistency acceptable for analytics, strong for URL mapping
- **Durability:** no URL data loss
- **Security:** rate limiting, abuse prevention

### Example phrase
> *"Let me separate functional from non-functional. Functionally we need A, B, C. Non-functionally, given the scale, I'll target 99.99% availability and sub-100ms p99 reads. Does that match your expectations?"*

---

## Step 3: Back-of-Envelope Estimates (5 min) — **VERY IMPORTANT**

**This is the most-skipped step and the most-scored step.** Numbers drive design.

### Numbers to memorize

| Item | Value |
|------|-------|
| Seconds in a day | ~86,400 (~100K for easy math) |
| Seconds in a month | ~2.5M |
| 1 byte int | 4 bytes |
| UUID | 16 bytes |
| Typical DB row | 100 bytes – 1 KB |
| Small image | 100 KB – 1 MB |
| HD video minute | ~50 MB |
| L1 cache | ~1 ns |
| Main memory | ~100 ns |
| SSD random read | ~100 μs |
| Network round trip (datacenter) | ~500 μs |
| Network round trip (cross-region) | ~50–150 ms |
| HDD seek | ~10 ms |

### Estimation template

```
DAU:              100M
Writes per user:  1/day  → 100M writes/day  → ~1,200 writes/sec
Reads per user:   100/day → 10B reads/day   → ~120K reads/sec
Storage/write:    500 bytes
Storage/day:      50 GB/day
Storage/5 years:  ~90 TB
Bandwidth/read:   500 bytes × 120K/s = 60 MB/s
Cache size (80/20): ~20% of daily reads × 500B ≈ 1 GB hot set
```

### Example phrase
> *"Let me estimate scale. 100M DAU, avg 100 reads per user per day gives ~10B reads/day, which is roughly 120K QPS. With 500-byte payloads, that's ~60 MB/s of read bandwidth. This tells me single-node won't cut it — we need horizontal scaling and caching."*

**The numbers justify every later design decision.**

---

## Step 4: High-Level Design (10 min)

Draw the **happy path** end-to-end. Keep it simple. No details yet.

### Standard template

```
[Client] → [CDN] → [Load Balancer] → [API Gateway]
                                          │
                                          ▼
                                  [Application Servers]
                                   │              │
                                   ▼              ▼
                              [Cache]        [Database(s)]
                                   │
                                   ▼
                              [Message Queue] → [Workers] → [Storage/Analytics]
```

### What to say
- "I'll start with a high-level architecture and we can zoom into any component."
- "Here's the request flow: client hits CDN for static assets, API goes through LB to app servers, which check cache first, fall back to DB."
- "For writes, I'm using async processing via a queue to keep p99 write latency low."

### Rules
- Label every arrow with the *operation* (read, write, publish).
- Don't name specific vendors yet ("cache" not "Redis" — unless asked).
- Show the async path if writes are heavy.

---

## Step 5: Deep Dive into Components (15 min)

Pick 1–2 components the interviewer cares about. Common deep-dive targets:

- **Database schema + partitioning strategy**
- **Cache design + invalidation**
- **API design** (endpoints, request/response shapes)
- **Specific algorithm** (rate limiter, feed ranking, consistent hashing)
- **Failure handling**

### Template for deep dive

1. **State the component's job** in one sentence.
2. **Define its API** (inputs, outputs).
3. **Define its data model** (schema, indexes).
4. **Define its scaling strategy** (sharding, replication).
5. **Define its failure behavior**.

### Example phrase
> *"Let me deep-dive on the URL-mapping store. Its job is: given a short code, return the long URL in under 10ms at 120K QPS. I'll use a key-value store, sharded by short code hash, with read replicas per shard. Schema is `(short_code PK, long_url, created_at, expires_at)`. On failure of the primary shard, reads fall back to replicas; writes fail fast and are retried by the client."*

---

## Step 6: Identify Bottlenecks (5 min)

Walk through your design and ask: **where does it break first?**

### Checklist
- [ ] Single point of failure? (DB? LB? Any singleton?)
- [ ] Hot partitions? (celebrity users, trending content)
- [ ] Write amplification? (one user action → 100 writes)
- [ ] Fan-out on reads? (feed that queries 100 services)
- [ ] Cache stampede risk?
- [ ] Thundering herd on retries?
- [ ] Cross-region latency?

### Example phrase
> *"The bottleneck here is the write path to the mapping DB — at 1000 writes/sec with replication, we could hit I/O limits. My mitigation is batching writes and using a write-through cache, but if it grows past 10K/sec I'd shard the primary by short-code range."*

---

## Step 7: Trade-offs & Wrap-up (2 min)

Summarize: what you chose, what you gave up, what you'd do next.

### Example phrase
> *"To recap: I prioritized availability over strict consistency because stale reads are acceptable for this use case. The main trade-off is eventual consistency on analytics counters — we accept ~1 minute lag. If I had more time, I'd go deeper into abuse prevention and geo-replication."*

---

# 3. Core Building Blocks

## 3.1 Load Balancers

**Job:** distribute incoming traffic across backend servers.

### Types
| Type | Layer | Use When |
|------|-------|----------|
| DNS LB | — | Global traffic steering (GeoDNS) |
| L4 (TCP) | Transport | High throughput, low overhead, long-lived connections |
| L7 (HTTP) | Application | Routing by path/header, TLS termination, observability |

### Algorithms
- **Round-robin** — simple, no state
- **Least connections** — good for variable-cost requests
- **Consistent hashing** — good for cache affinity / sticky sessions
- **Weighted** — for mixed-capacity fleets

### When to use
- Always, in production. Single server = single point of failure.

### Trade-offs
- Adds one hop (~1–5ms).
- L7 LBs are CPU-heavier than L4.
- Sticky sessions reduce elasticity — prefer stateless services.

### Real examples
- AWS ALB/NLB, GCP Cloud LB, HAProxy, Nginx, Envoy.

---

## 3.2 API Gateway

**Job:** single entry point for clients; handles cross-cutting concerns.

### Responsibilities
- Authentication / authorization
- Rate limiting
- Request routing to internal services
- Protocol translation (REST ↔ gRPC)
- Response aggregation (BFF pattern)
- Caching, logging, metrics

### When to use
- Microservices architectures with multiple client types.
- Public APIs (auth, rate limit, quota).

### When NOT to use
- Simple monoliths — the gateway is overkill.
- Internal service-to-service traffic (use a service mesh instead).

### Trade-offs
- Another hop, another thing to operate.
- Can become a bottleneck or SPOF if not redundant.

### Real examples
- Kong, AWS API Gateway, Zuul, Spring Cloud Gateway, Envoy.

---

## 3.3 Databases: SQL vs NoSQL

### SQL (Relational)

**Examples:** PostgreSQL, MySQL, Aurora, Spanner.

**Strengths**
- ACID transactions
- Strong consistency
- Complex queries, joins
- Mature tooling, well-understood

**Weaknesses**
- Harder to scale writes horizontally (solvable with sharding, Vitess, Citus)
- Schema migrations at scale require care
- Joins become painful across shards

**When to use**
- Financial data, user accounts, order systems — anything needing transactions.
- Complex relational data with query flexibility.
- When in doubt, start with PostgreSQL.

---

### NoSQL

**Families:**

| Family | Examples | Good For |
|--------|----------|----------|
| Key-Value | Redis, DynamoDB, Riak | Session stores, caches, simple lookups |
| Document | MongoDB, Couchbase | Flexible schemas, JSON-heavy workloads |
| Wide-column | Cassandra, ScyllaDB, HBase | Time-series, write-heavy, massive scale |
| Graph | Neo4j, Neptune | Social graphs, recommendation engines |
| Search | Elasticsearch, OpenSearch | Full-text search, log analytics |

**When to use NoSQL**
- Scale exceeds what a single SQL instance + read replicas can handle.
- Access pattern is well-known and simple (e.g., key lookup).
- Schema is flexible or evolving.
- Write throughput is very high (Cassandra excels).

**When NOT to use NoSQL**
- You need multi-row transactions.
- Access patterns are unpredictable — NoSQL punishes ad-hoc queries.
- You're choosing it for "cool factor" without a real scale problem.

### Cheat Sheet: Which DB?

| Workload | Start With |
|----------|-----------|
| User accounts, orders | PostgreSQL |
| Session/cache | Redis |
| Product catalog / CMS | PostgreSQL or MongoDB |
| Social feed write path | Cassandra |
| Event log | Kafka + long-term in S3/parquet |
| Search | Elasticsearch |
| Real-time leaderboard | Redis sorted sets |
| Graph (friends, followers) | Neo4j or native graph on top of KV |

---

## 3.4 Caching (Redis / Memcached)

**Job:** serve hot data from memory to reduce latency and DB load.

### Levels
```
Client → Browser cache → CDN → LB cache → App-layer cache → Distributed cache (Redis) → DB
```

### When to use
- Read-heavy workloads with a hot set (80/20 rule).
- Expensive computations you can memoize.
- Session state for horizontally-scaled app servers.

### When NOT to use
- Write-heavy, rarely-read data.
- Data where staleness is unacceptable *and* you can't invalidate fast.
- Small datasets that fit in DB memory anyway.

### Trade-offs
- Adds a dependency + failure mode.
- Invalidation is hard.
- Cache stampede risk on expiry.

### Real examples
- Redis: session stores, leaderboards, rate limiters, pub/sub.
- Memcached: pure LRU cache, no persistence, simpler.

*See [Section 6](#6-caching-strategies) for patterns.*

---

## 3.5 Message Queues (Kafka, RabbitMQ, SQS)

**Job:** decouple producers from consumers via durable async messaging.

### Kafka vs RabbitMQ vs SQS

| Feature | Kafka | RabbitMQ | SQS |
|---------|-------|----------|-----|
| Model | Log (append-only) | Broker (queue) | Managed queue |
| Throughput | Very high (millions/sec) | High | High |
| Ordering | Per partition | Per queue | FIFO queues only |
| Replay | Yes | No (messages consumed) | No |
| Use case | Event sourcing, streaming, logs | Task queues, RPC | Simple decoupling on AWS |

### When to use a queue
- Decouple slow work from fast requests (email, thumbnailing, analytics).
- Smooth out traffic spikes (buffer).
- Enable multiple consumers of the same event stream.
- Cross-service events (order-placed → notify, bill, ship, log).

### When NOT to use
- Synchronous request/response — use RPC.
- You need transactions across the queue and DB (hard; use outbox pattern).

### Real examples
- LinkedIn: Kafka for activity streams.
- Uber: Kafka for trip events.
- Netflix: Kafka for observability pipeline.

---

## 3.6 CDN (Content Delivery Network)

**Job:** serve static (and sometimes dynamic) content from edge servers close to users.

### When to use
- Static assets: images, JS, CSS, videos.
- Geographically distributed users.
- Protection against traffic spikes (edge absorbs load).

### When NOT to use
- Internal APIs.
- Highly personalized content (low hit rate).

### Trade-offs
- Cache invalidation latency (minutes to propagate).
- Cost scales with bandwidth.

### Real examples
- CloudFront, Cloudflare, Akamai, Fastly.

---

## 3.7 Microservices vs Monolith

| Dimension | Monolith | Microservices |
|-----------|----------|---------------|
| Initial velocity | Fast | Slow |
| Long-term velocity at scale | Slow (coupling) | Fast (teams own pieces) |
| Deployment complexity | Low | High |
| Observability | Simple | Requires tracing, metrics |
| Debugging | Easy (stack traces) | Hard (distributed) |
| Team structure fit | Small teams | Multiple independent teams |

### Recommendation
- Start with a **modular monolith**.
- Extract services only when you have a real reason: team autonomy, independent scaling, technology mismatch.
- **"Microservices" is not free scalability.** Network calls replace method calls — now you have to deal with partial failure, retries, and serialization overhead.

### Interview phrase
> *"I'd start as a modular monolith. At our scale tier, the extraction cost isn't worth it yet. I'd extract the [X] service first because it has independent scaling requirements."*

---

# 4. Scalability Concepts

## 4.1 Horizontal vs Vertical Scaling

| | Vertical (scale up) | Horizontal (scale out) |
|-|-|-|
| Approach | Bigger machine | More machines |
| Ceiling | Hardware limit | Near-unlimited |
| Cost curve | Exponential | Linear-ish |
| Complexity | Low | High (distribution) |
| Failure mode | Single box down = outage | Individual failures tolerated |

**Rule:** Vertical first (it's cheap and simple). Horizontal when you must (stateful systems, large scale).

---

## 4.2 Partitioning / Sharding

**Why:** one node can't hold all the data or handle all the traffic.

### Strategies

**1. Range-based**
- Shard by key ranges (e.g., `user_id 0–1M → shard 0`).
- Pro: range scans fast. Con: hot shards.

**2. Hash-based**
- `shard = hash(key) % N`.
- Pro: even distribution. Con: resharding moves most data.

**3. Consistent hashing**
- Keys and nodes on a ring; each key goes to next node clockwise.
- Pro: adding/removing a node only moves `1/N` of keys. Con: more complex.

**4. Directory-based**
- Lookup service maps keys → shards.
- Pro: full flexibility. Con: directory is a SPOF.

### Choosing a shard key
- High cardinality (avoid hot keys).
- Matches query patterns (avoid cross-shard queries).
- Distributes write load evenly.

### Example
- Twitter shards tweets by `tweet_id` using snowflake IDs.
- Instagram shards by `user_id` — 95% of queries are user-scoped.

---

## 4.3 Replication

**Why:** availability + read scaling + disaster recovery.

### Patterns

**Primary-Replica (Leader-Follower)**
- All writes → primary; reads from replicas.
- Pro: simple, read-scalable.
- Con: replica lag, failover complexity.

**Multi-Primary (Multi-Leader)**
- Writes to any node; replicated to others.
- Pro: lower write latency globally.
- Con: conflict resolution is hard.

**Leaderless (Dynamo-style)**
- Clients write to `W` nodes, read from `R` nodes (quorum: `R + W > N`).
- Pro: no failover needed.
- Con: last-write-wins edge cases.

### Trade-offs
- Sync replication: strong consistency, higher latency.
- Async replication: fast writes, possible data loss on primary crash.

---

## 4.4 Consistency Models

From strongest to weakest:

1. **Linearizability** — every operation appears to happen atomically at some point between invocation and response. Gold standard, expensive.
2. **Sequential consistency** — all ops appear in some total order, same for all observers.
3. **Causal consistency** — operations causally related are seen in order.
4. **Read-your-writes** — a user always sees their own updates.
5. **Monotonic reads** — you don't go back in time.
6. **Eventual consistency** — if no new writes, all replicas converge.

### When each matters
- Financial transactions → linearizable.
- Comment threads → causal (replies after parents).
- Social feeds → eventual (nobody notices 500ms lag).
- User profile edits → read-your-writes (you see your change immediately).

---

## 4.5 CAP Theorem — The Real Interpretation

**Popular (wrong) version:** "Pick 2 of Consistency, Availability, Partition tolerance."

**Real version:** Partition tolerance is non-negotiable in distributed systems — networks fail. **When a partition occurs**, you choose between:
- **CP:** reject requests (give up availability) to maintain consistency.
- **AP:** serve possibly-stale data (give up consistency) to maintain availability.

During normal operation, good systems are both consistent and available. CAP only kicks in during partitions.

### Examples
- **CP:** etcd, ZooKeeper, MongoDB (default), HBase.
- **AP:** Cassandra, DynamoDB (configurable), Riak.

### PACELC (more accurate)
- **If Partition:** choose Availability or Consistency.
- **Else (normal operation):** choose Latency or Consistency.

---

## 4.6 Eventual Consistency

**Definition:** given enough time with no updates, all replicas converge.

### Where it's acceptable
- Like counts on social media.
- Analytics dashboards.
- Search indexes.
- CDN content.

### Where it's NOT
- Account balances (could double-spend).
- Inventory at checkout (could oversell).
- Authentication tokens.

### Techniques to make it safe
- Read-your-writes via sticky sessions or client-side caching.
- Version vectors or CRDTs for conflict resolution.
- Idempotent operations so duplicates don't corrupt state.

---

# 5. Data Design & Storage

## 5.1 Choosing a Database

### Decision flowchart
```
Is data relational with transactions?
├── Yes → PostgreSQL / MySQL
└── No
    ├── Pure key lookup at huge scale?   → Redis / DynamoDB
    ├── Write-heavy, time-series?        → Cassandra / InfluxDB
    ├── Flexible schema, document-y?     → MongoDB
    ├── Full-text search?                → Elasticsearch
    ├── Graph traversal?                 → Neo4j
    └── Blob storage?                    → S3 / GCS
```

## 5.2 Schema Design Principles

- **Normalize first**, denormalize for performance when profiled.
- **Denormalize for read-heavy access** — duplicate data to avoid joins.
- **Immutable event logs** are easier to scale than mutable state.
- **Soft delete (`deleted_at`)** is almost always better than hard delete.
- **Design for the query** — start from access patterns, derive schema.

### Example: URL shortener schema

```sql
CREATE TABLE url_mapping (
  short_code VARCHAR(10) PRIMARY KEY,
  long_url   TEXT NOT NULL,
  user_id    BIGINT,
  created_at TIMESTAMP NOT NULL,
  expires_at TIMESTAMP,
  INDEX idx_user (user_id, created_at)
);
```

## 5.3 Indexing Strategies

| Index Type | Use Case |
|------------|----------|
| B-tree | Range queries, ordered data (default) |
| Hash | Exact lookup (Postgres hash index, in-memory KV) |
| Composite | Multi-column filters (`WHERE a=? AND b=?`) |
| Covering | Query satisfied entirely from index |
| Partial | Index only rows matching a predicate |
| Full-text | `LIKE '%x%'` or search queries (use Elasticsearch) |

### Rules
- Every index slows writes — don't over-index.
- Index selectivity matters: boolean columns = bad index.
- The leading column of a composite index is used for range filtering; order matters.

## 5.4 Handling Large Datasets

- **Archive cold data** to cheap storage (S3, Glacier).
- **Partition tables** by time or tenant (Postgres declarative partitioning).
- **Use TTL** on ephemeral data (sessions, tokens).
- **Batch deletes** to avoid long-running locks.
- **Read replicas** for analytical queries to keep OLTP fast.
- **Materialized views** for expensive aggregations.

---

# 6. Caching Strategies

## 6.1 Cache-Aside (Lazy Loading) — Most Common

```
read(key):
  v = cache.get(key)
  if v: return v
  v = db.get(key)
  cache.set(key, v, ttl)
  return v

write(key, value):
  db.write(key, value)
  cache.delete(key)   // or cache.set
```

- **Pros:** only caches what's used; resilient if cache fails.
- **Cons:** first request is a miss; brief staleness window.

**Use for:** most read-heavy workloads. Default choice.

---

## 6.2 Write-Through

```
write(key, value):
  cache.set(key, value)
  db.write(key, value)
```

- **Pros:** cache always fresh.
- **Cons:** write latency includes both writes; cache holds data that may never be read.

**Use for:** data that's both written and read frequently.

---

## 6.3 Write-Back (Write-Behind)

```
write(key, value):
  cache.set(key, value)
  queueDbWrite(key, value)   // async
```

- **Pros:** fastest writes; batches DB ops.
- **Cons:** risk of data loss if cache crashes before flush.

**Use for:** high-volume non-critical writes (analytics counters).

---

## 6.4 Read-Through

Cache itself loads from DB on miss (transparent to app).

**Use for:** when using managed caches that support it (DAX, some Redis setups).

---

## 6.5 Cache Invalidation — *"the hardest problem"*

Techniques, in order of simplicity:

1. **TTL** — set a time-to-live. Simple but stale reads possible. Great default.
2. **Explicit invalidation** — on write, delete cache key. Works for single-key updates.
3. **Versioned keys** — embed version in key; bump version on update (`user:42:v7`). Eliminates stale reads.
4. **Change Data Capture (CDC)** — listen to DB changes (Debezium → Kafka) and invalidate asynchronously.
5. **Event-driven** — publish invalidation events on write; subscribers invalidate.

### Cache stampede
When a hot key expires and 10K requests all miss the cache simultaneously, hammering the DB.

**Mitigations:**
- **Request coalescing** — first miss populates; others wait.
- **Probabilistic early expiration** — refresh before TTL at increasing probability.
- **Pre-warming** — refresh popular keys proactively.

---

## 6.6 Distributed Caching

- **Redis Cluster:** native sharding with consistent hashing.
- **Redis Sentinel:** HA with failover (no sharding).
- **Client-side consistent hashing:** for simple horizontal scaling.
- **Replication:** primary-replica per shard for read scaling and HA.

### Eviction policies
- `LRU` (least recently used) — default.
- `LFU` (least frequently used) — for items with steady popularity.
- `TTL-based` — when data has natural expiry.

---

# 7. Communication Patterns

## 7.1 REST

**When:** public APIs, CRUD, human-debuggable, broad client compatibility.

**Pros:** ubiquitous, cacheable, stateless, human-readable.

**Cons:** verbose (JSON), no native streaming, no schema enforcement by default.

**In interview:** default for client-facing APIs.

---

## 7.2 gRPC

**When:** internal service-to-service, low latency, high throughput.

**Pros:** HTTP/2 multiplexing, protobuf (compact, typed), streaming, cross-language codegen.

**Cons:** not browser-friendly without proxy, harder to debug (binary).

**In interview:** internal microservices at Big Tech are almost always gRPC.

---

## 7.3 GraphQL

**When:** clients need flexible, nested data; multiple client types with different needs.

**Pros:** clients ask for exactly what they need; single round trip.

**Cons:** complex server implementation, caching is harder, N+1 query risk.

---

## 7.4 Sync vs Async

| | Sync (RPC) | Async (Queue/Event) |
|-|-|-|
| Latency model | Request waits | Fire and forget |
| Coupling | Tight | Loose |
| Failure blast radius | Caller sees errors | Isolated |
| Use when | You need the answer now | You can defer work |

### Rule of thumb
- User-facing read path → sync.
- Background work (emails, analytics, indexing) → async.
- Cross-service side effects → async events.

---

## 7.5 Event-Driven Systems

### Patterns
- **Event notification** — "Something happened." Consumer fetches details.
- **Event-carried state transfer** — event carries enough data to act on.
- **Event sourcing** — the log of events *is* the source of truth.

### When to use
- Loose coupling between services.
- Multiple consumers of the same event.
- Audit / replay requirements.

### Costs
- Eventual consistency everywhere.
- Debugging distributed flows is hard — need distributed tracing.
- Schema evolution discipline required.

---

# 8. Reliability & Fault Tolerance

## 8.1 Retries

### Rules
- **Only retry idempotent operations** (or use idempotency keys).
- **Exponential backoff** with **jitter** to avoid thundering herd.
- **Cap retries** (typically 3–5).
- **Retry budget** — stop retries if error rate is high (system is clearly down).

```java
// Pseudo
for (int i = 0; i < maxRetries; i++) {
  try { return call(); }
  catch (TransientException e) {
    sleep(baseDelay * (2^i) + random(0, jitter));
  }
}
throw new FailedAfterRetriesException();
```

## 8.2 Timeouts

- **Always set timeouts.** A hanging call blocks threads, cascades into outages.
- Layered timeouts: client < service < DB.
- Typical: p99 × 2 as an upper bound.

## 8.3 Circuit Breakers

**Pattern:** if error rate exceeds threshold, fail fast instead of hammering a dying service.

### States
- **Closed:** normal; requests pass through.
- **Open:** failing; requests rejected immediately.
- **Half-open:** test with a few requests; close if they succeed.

### Library examples
- Java: Resilience4j, Hystrix (legacy).
- Service mesh: Istio/Envoy handle it at infra level.

## 8.4 Idempotency

**Definition:** performing the same operation multiple times has the same effect as once.

### How to achieve
- Assign an **idempotency key** per request; server stores outcomes keyed by it.
- Use `PUT` / `DELETE` semantics where possible.
- Upserts instead of inserts when retry-safe.
- Deduplicate with a TTL cache of seen keys.

```
POST /payments
Idempotency-Key: abc-123
```

If the client retries with the same key, server returns the same result rather than charging twice.

## 8.5 Handling Failures

- **Bulkheads:** isolate thread pools per downstream dependency.
- **Graceful degradation:** return partial/cached data if a non-critical dep is down.
- **Fallbacks:** default values, stale cache, queue for later processing.
- **Dead-letter queues:** messages that repeatedly fail go to a DLQ for manual inspection.
- **Health checks:** liveness (am I running?) vs readiness (can I serve traffic?).
- **Chaos engineering:** intentionally inject failures to test (Netflix Chaos Monkey).

---

# 9. Consistency & Distributed Systems

## 9.1 Strong vs Eventual Consistency — Summary

| | Strong | Eventual |
|-|-|-|
| Latency | Higher | Lower |
| Availability during partition | Lower | Higher |
| Developer model | Simple | Complex (must handle staleness) |
| Cost | Higher (coordination) | Lower |

## 9.2 Distributed Transactions

### Two-Phase Commit (2PC)
1. **Prepare:** coordinator asks all nodes if they can commit.
2. **Commit:** if all say yes, coordinator tells everyone to commit.

**Problems:** blocking if coordinator crashes; slow; assumes reliable network.

**In practice:** avoided at scale. Instead, use:

### Saga Pattern

Long-running transaction decomposed into a series of local transactions, each with a compensating action.

```
Order Saga:
  1. Reserve inventory       (compensate: release)
  2. Charge payment          (compensate: refund)
  3. Create shipment         (compensate: cancel)
  
If step 3 fails, run compensations for 2, 1 in reverse.
```

### Two styles
- **Choreography:** services listen to events and react (decentralized).
- **Orchestration:** a saga orchestrator drives the steps (centralized).

### When to use orchestration vs choreography
- **Orchestration:** complex flows, need observability.
- **Choreography:** simple flows, fewer services.

## 9.3 Consensus

**Problem:** get N nodes to agree on a single value despite failures.

**Algorithms:** Paxos, Raft, Zab.

**Used by:** etcd, ZooKeeper, Consul, CockroachDB, Spanner.

**In interviews:** you don't need to implement Raft — but recognize that *leader election*, *config management*, and *distributed locks* all rely on consensus, and these are the foundations of "CP" systems.

---

# 10. Performance Optimization

## 10.1 Latency vs Throughput

- **Latency:** time for one request. Measured per request. p50/p95/p99.
- **Throughput:** requests per second. Measured in aggregate.

They trade off: batching improves throughput but worsens latency. Parallelism improves throughput but can worsen tail latency (wait for slowest).

## 10.2 Key Metrics to Track

- **p50, p95, p99, p99.9 latency** — averages lie; tails matter.
- **QPS / RPS** — throughput.
- **Error rate** — % of 5xx / failures.
- **Saturation** — CPU, memory, disk I/O, network.
- **Queue depth** — backlog indicates bottleneck.

## 10.3 Bottleneck Analysis

Work outward from the user:
1. **Client** — rendering, too many requests?
2. **Network** — DNS, TLS, distance (use CDN, HTTP/2).
3. **Load balancer** — saturation, connection limits.
4. **App server** — CPU, memory, GC pauses, thread pool.
5. **Cache** — miss rate, eviction rate.
6. **Database** — slow queries, lock contention, index misses, I/O.
7. **Downstream services** — chain of RPC calls adds latency.

## 10.4 Common Optimizations

| Problem | Fix |
|---------|-----|
| DB CPU saturated | Add read replicas, cache hot reads |
| Slow queries | Add index, denormalize, materialized views |
| Hot key in cache | Client-side caching, request coalescing |
| Fan-out on read | Precompute (fan-out on write) |
| Fan-out on write | Lazy compute (fan-out on read) |
| Network round trips | Batch, pipeline, colocate |
| Serialization cost | Use protobuf / flatbuffers |
| Thread contention | Async I/O, virtual threads (Java 21) |
| GC pauses | Tune GC, reduce allocations, use off-heap |

## 10.5 Read vs Write Optimization

- **Read-heavy:** cache, replicas, CDN, denormalize.
- **Write-heavy:** partition, append-only logs, batch writes, use write-optimized DBs (Cassandra, LSM-tree stores).

---

# 11. Real System Design Problems

Each problem is worked through the 7-step framework.

---

## 11.1 Design a URL Shortener (bit.ly)

### 1. Requirements
**Functional**
- Create a short URL from a long URL.
- Redirect short URL → long URL.
- Optional: custom aliases, analytics, expiration.

**Non-functional**
- 99.99% availability (redirect must work).
- p99 redirect latency < 50ms.
- Read-heavy: 100:1 read:write.

### 2. Scale estimates
- 100M new URLs / month ≈ 40 writes/sec.
- 10B redirects / month ≈ 4K reads/sec (avg), 40K peak.
- Storage: 500 bytes × 100M × 12 × 5yr = ~3 TB.

### 3. High-level design
```
Client → CDN → LB → API Gateway → URL Service
                                    │
                        ┌───────────┴───────────┐
                        ▼                       ▼
                  Redis Cache              URL Database
                                          (sharded KV store)
```

### 4. Deep dive: short code generation

**Options**
- **Hash(long_url)[:7]** — duplicates possible, collisions, non-idempotent for same URL across users.
- **Counter + Base62** — simple, predictable (security risk), single counter = bottleneck.
- **Distributed counter (snowflake-like)** — scalable, unordered, safe.
- **Pre-generated range** — each server gets a block of IDs from a coordinator (e.g., ZooKeeper), converts to Base62.

**Choice:** pre-generated ID ranges from a centralized ticket server, then Base62-encoded. Gives 7-character codes, 3.5 trillion possibilities.

### 5. Data model

```
Table: url_mapping
  short_code   VARCHAR(10) PRIMARY KEY
  long_url     TEXT
  user_id      BIGINT
  created_at   TIMESTAMP
  expires_at   TIMESTAMP
```

Sharded by `hash(short_code)`. Replicated per shard.

### 6. Read path
1. Request hits CDN (miss).
2. LB → URL Service.
3. Check Redis cache by `short_code`. Hit? return 301/302.
4. Miss? Query DB shard. Populate cache with TTL.
5. Return redirect.

### 7. Trade-offs
- 301 (permanent) is cacheable but hides analytics; use 302 for analytics.
- Cache hit rate > 95% because distribution is long-tailed (few URLs get most clicks).
- Analytics via async event → Kafka → warehouse.

---

## 11.2 Design a Rate Limiter

### 1. Requirements
- Limit requests per user/API key per time window.
- Support distributed deployment.
- Low latency: < 1ms overhead per request.
- Handle edge cases: clock skew, burst traffic.

### 2. Algorithms

**Token Bucket**
- Bucket holds `N` tokens; refills at rate `R`/sec.
- Each request consumes one token.
- Allows bursts up to bucket size.
- **Use when:** you want to allow bursts.

**Leaky Bucket**
- Requests enter a queue; processed at constant rate.
- Smooths out spikes.
- **Use when:** you want uniform outflow.

**Fixed Window**
- Count requests per wall-clock window (e.g., per minute).
- **Problem:** edge bursts at window boundaries.

**Sliding Window Log**
- Store timestamps of each request; count within window.
- Accurate but memory-heavy.

**Sliding Window Counter**
- Weighted combination of current + previous fixed windows.
- Approximate; very cheap.

### 3. Distributed design

```
Request → Rate Limiter Middleware → Redis (atomic INCR + EXPIRE)
               │
               ├─ Allowed → forward
               └─ Denied → 429
```

- Use `INCR` + `EXPIRE` or Lua script for atomicity.
- Key: `rate_limit:{user_id}:{window}`.
- TTL = window duration.

### 4. Trade-offs
- **Centralized Redis:** accurate, but Redis becomes a dependency on every request.
- **Local + sync:** fast but inaccurate across nodes.
- **Approximation:** use local counters with periodic sync. Good for high-throughput, rough limits.

### 5. Edge cases
- Fail-open (allow on Redis outage) vs fail-closed (deny).
- Exponential backoff response header: `Retry-After`.

---

## 11.3 Design a Chat System (WhatsApp / Slack)

### 1. Requirements
**Functional**
- 1:1 chat, group chat.
- Online presence.
- Read receipts, typing indicators.
- Push notifications for offline users.
- Message history.

**Non-functional**
- 500M DAU, 100B messages/day.
- p99 send latency < 100ms.
- Durable — no message loss.

### 2. Scale
- 100B messages/day ≈ 1.2M messages/sec avg, 5M peak.
- Storage: 100 bytes × 100B × 365 = ~3.6 PB/year (cold storage for old).

### 3. High-level design

```
Mobile Client  ←WebSocket→  Chat Gateway  ←→  Chat Service
                                               │
                                    ┌──────────┼──────────┐
                                    ▼          ▼          ▼
                               Message DB   Kafka      Presence
                               (Cassandra)  (fan-out)  (Redis)
                                              │
                                              ▼
                                       Notification Service
                                        (APNs / FCM)
```

### 4. Deep dive: message delivery

- Client connects via persistent **WebSocket** to gateway.
- Gateway registers `user_id → gateway_instance` in Redis.
- Sender → Chat Service → writes to Cassandra (append-only, partition by `chat_id`).
- Publishes to Kafka topic for fan-out.
- Fan-out worker looks up each recipient's gateway; delivers over WebSocket.
- Offline? Queue push notification.

### 5. Data model

```
Cassandra table: messages
  PK: (chat_id, created_at DESC, message_id)
  Columns: sender_id, content, seen_by
```

Partition key = `chat_id` gives fast time-range queries for chat history.

### 6. Group chat
- Group membership in separate table: `(group_id → [user_ids])`.
- Fan-out to all members (store-and-forward if offline).
- Very large groups (>1000): use fan-out on read instead.

### 7. Trade-offs
- WebSocket vs long-polling: WebSocket lower latency, more complex reconnection.
- Cassandra chosen for write-heavy, time-series-shaped data.
- Presence via Redis with TTL-based heartbeats — eventual consistency is fine.
- End-to-end encryption changes design: server stores ciphertext only.

---

## 11.4 Design a News Feed System (Twitter / Facebook)

### 1. Requirements
- User posts a tweet/status.
- User sees a feed sorted by time (or relevance).
- Followers see new posts from people they follow.

### 2. Scale
- 500M DAU, avg 200 followers.
- 50M posts/day, 10B feed reads/day.

### 3. The core trade-off: fan-out on write vs fan-out on read

| | Fan-out on Write (push) | Fan-out on Read (pull) |
|-|-|-|
| Write cost | High (write to all followers' timelines) | Cheap (single write) |
| Read cost | Cheap (read precomputed timeline) | High (query all followed users) |
| Good for | Users with few followers | Users with many followers (celebrities) |

### 4. Hybrid model (Twitter's actual approach)
- **Fan-out on write** for most users.
- **Fan-out on read** for celebrity users (>1M followers) — fetch their posts at read time to avoid writing to 100M timelines.

### 5. High-level design

```
Post Service → Kafka → Fan-out Worker → User Timeline Cache (Redis)
                                              │
                                              ▼
                                         Feed Service
                                              │
                                              ▼
                                           Client
```

### 6. Data

- Posts in Cassandra, partitioned by `user_id`.
- Timelines in Redis sorted sets per user: `ZADD timeline:{user_id} {timestamp} {post_id}`.
- Capped at ~1000 entries; older fetched from DB on deep scroll.

### 7. Trade-offs
- Eventual consistency in feed — OK; users don't expect instant delivery.
- Cold cache for infrequent users — regenerate on login.
- Ranking (ML-based) adds a scoring layer; pre-computed features + online ranking.

---

## 11.5 Design a File Storage System (Dropbox / Google Drive)

### 1. Requirements
**Functional**
- Upload / download files.
- Sync across devices.
- Share files / folders.
- Version history.

**Non-functional**
- Durable: 11 nines (object storage level).
- Efficient: only sync changes, not whole files.
- Support files up to 10 GB.

### 2. Scale
- 500M users, avg 10 GB each = 5 EB.
- Dedupe + compression typically cuts ~30%.

### 3. High-level design

```
Client (with sync agent) ←→ API Gateway ←→ Metadata Service ←→ Metadata DB
                                    │
                                    ↓
                              Block Service ←→ Object Store (S3)
                                    │
                                    ↓
                                 Notification (WebSocket)
```

### 4. Deep dive: chunked uploads

- Split files into **fixed-size chunks** (e.g., 4 MB).
- Hash each chunk (SHA-256).
- Only upload chunks that don't already exist (dedup).
- Store chunk → object mapping in metadata DB.
- **Benefits:** resumable uploads, dedup across users, efficient diffs.

### 5. Data model

```
files:
  file_id, user_id, name, parent_folder_id, size, updated_at

file_versions:
  file_id, version_id, chunk_hashes[], created_at

chunks:
  chunk_hash PK, s3_key, size, ref_count
```

### 6. Sync

- Client polls (or subscribes) for changes since last sync cursor.
- Server returns delta: added/removed/modified files.
- Conflict resolution: last-write-wins or keep both as versions.

### 7. Trade-offs
- Client-side encryption (E2E) vs server-side (enables sharing, dedup).
- Strong vs eventual consistency for metadata — need read-your-writes.
- Cold storage tiering for old versions.

---

## Bonus: 11.6 Design a Distributed Key-Value Store (DynamoDB-style)

### Core techniques
- **Consistent hashing** for partitioning.
- **Virtual nodes** to balance load.
- **Vector clocks** or **version numbers** for conflict resolution.
- **Quorum reads/writes:** `R + W > N`.
- **Merkle trees** for anti-entropy (replica sync).
- **Gossip protocol** for membership.

This is a favorite for senior+ interviews — mastering the Dynamo paper is high ROI.

---

# 12. Interview Communication Skills

## 12.1 How to Speak Clearly

- **One idea per sentence.** Pause. Then next.
- **Signpost.** "Three things I want to cover: X, Y, Z. Let me start with X."
- **State-then-explain.** "I'll use Cassandra. Here's why: ..."
- **Think aloud.** Silence is unscorable. "I'm weighing A vs B — A's strength is..."

## 12.2 How to Structure Answers

Use the 7-step framework visibly. At each transition, tell the interviewer:
- "I'm done with requirements. Moving to estimates."
- "High level done. Which component should I deep dive first?"

Checking in hands control to the interviewer — they'll steer you where they want to score.

## 12.3 How to Handle Feedback / Hints

**Interviewer hints are gold.** They're nudging you because they want you to succeed.

- **"Are you sure about that?"** → Re-examine that decision. They spotted a flaw.
- **"What if the load was 10x?"** → They want you to discuss scaling.
- **"How does this work if X fails?"** → They want failure modes.

Never get defensive. Say: *"Good point — let me reconsider."*

## 12.4 How to Recover If Stuck

- **Verbalize what you're stuck on.** "I'm torn between X and Y because..."
- **Ask the interviewer.** "Is latency or throughput more important here?"
- **Simplify.** Reduce scale or remove a constraint. Solve the easier version. Then reintroduce.
- **Switch components.** "Let me come back to this; can I deep-dive on [Z] first?"

## 12.5 Pacing

| Stage | Target time |
|-------|-------------|
| Requirements | 5 min |
| Estimates | 5 min |
| High-level | 10 min |
| Deep dive | 15 min |
| Bottlenecks & wrap-up | 5 min |
| Q&A | 5 min |

If you're 15 minutes in and still on requirements, that's a red flag. Move.

---

# 13. Common Trade-offs

Every system design decision is a trade-off. Master these.

## 13.1 Consistency vs Availability
- **Payments, inventory** → consistency.
- **Social feed, search** → availability.

## 13.2 Latency vs Throughput
- **Real-time chat** → latency.
- **Batch analytics** → throughput.

## 13.3 Latency vs Cost
- Single region = cheap but far from global users.
- Multi-region = low latency but costly and consistency-complex.

## 13.4 Read vs Write Performance
- Indexes speed reads, slow writes.
- Denormalization speeds reads, costs write amplification and storage.

## 13.5 Simplicity vs Scalability
- Modular monolith = simple, scales to millions of users.
- Microservices = complex, scales to thousands of engineers.
- **Choose simplicity until you can't.**

## 13.6 Strong Consistency vs Partition Tolerance
- See CAP. State the choice explicitly.

## 13.7 Freshness vs Cost
- Cache TTL controls staleness and hit rate.
- Real-time invalidation is expensive; TTL is cheap.

## 13.8 Fan-out on Write vs Read
- Write-heavy precompute = fast reads, expensive writes.
- Read-time compute = cheap writes, expensive reads.

## Interview phrase template
> *"I chose X. The trade-off is I'm giving up Y for Z. If [condition] changes, I'd revisit and pick differently."*

---

# 14. Red Flags in Interviews

## Design-Level Red Flags
- Single database, no caching, no replication — no consideration of failure.
- "We'd add Redis later" without saying *why* or *where*.
- Every service has its own database but data flows aren't defined.
- No discussion of how data is sharded at scale.
- Unbounded queues ("Kafka handles it").

## Reasoning Red Flags
- **Buzzword mode.** "Kubernetes microservices serverless event-driven…" — empty calories.
- **No numbers.** Design has no relation to the scale you estimated.
- **Silence when challenged.** Treating the interviewer's push as an attack.
- **Stubbornness.** Refusing to update a design after a good counter-argument.
- **Missing trade-offs.** Every choice is "just better."

## Process Red Flags
- Skipping requirements gathering.
- Never drawing anything.
- Not checking in with the interviewer.
- Running out of time on setup, never reaching deep dive.

## Over-engineering Red Flags
- Multi-region active-active for a prototype.
- ML-based ranking for 100 users.
- Service mesh for 2 services.

**Rule:** design to the stated requirements. Nothing more.

---

# 15. Preparation Strategy

## 15.1 4-Week Plan (for someone with your background)

### Week 1: Fundamentals
- Read this guide end-to-end.
- Memorize the latency/storage numbers.
- Learn the 7-step framework by heart.
- Practice estimation: given DAU, derive QPS, storage, bandwidth.

### Week 2: Building Blocks
- Deep-read on: consistent hashing, CAP, quorum, Kafka, Cassandra, DynamoDB paper.
- Implement a toy: rate limiter (token bucket in Java + Redis) and consistent hashing in Java.
- Draw 5 architectures from memory: URL shortener, rate limiter, feed, chat, file storage.

### Week 3: Mock Interviews
- 3–5 mock interviews (peers, Pramp, Hello Interview, Exponent).
- Record yourself — rewatch to spot filler words, pacing, unclear moments.
- After each mock: write down what you'd do differently.

### Week 4: Polish
- Focus on weak components from your mocks.
- Do 2 mocks with harder problems (payments, ad auction, distributed cache).
- Prepare your "default stack" story — what you'd reach for and why.

## 15.2 How to Practice

### Solo practice
1. Pick a problem (see list below).
2. Set a 45-minute timer.
3. Design it out loud — record yourself.
4. Review: did you hit all 7 steps? Where did you stall?
5. Read a solution (e.g., *Grokking the System Design Interview*, *Designing Data-Intensive Applications*).
6. Redo next day from scratch.

### Problem list (ordered by typical difficulty)
1. URL Shortener
2. Rate Limiter
3. Key-Value Store
4. Web Crawler
5. Pastebin
6. News Feed
7. Chat / Messenger
8. Uber / ride matching
9. YouTube / video streaming
10. Google Drive / Dropbox
11. Instagram
12. Twitter / timelines
13. Distributed Job Scheduler
14. Payment System
15. Ad Click Aggregation
16. Distributed Cache
17. Notification System
18. Search / Typeahead
19. Stock Exchange / Order Matching

## 15.3 Resources

- **Books:** *Designing Data-Intensive Applications* (Kleppmann) — the single highest-ROI book.
- **Courses:** *Grokking the Modern System Design Interview*, ByteByteGo, Hello Interview.
- **Papers:** Dynamo (2007), Google Bigtable, MapReduce, Spanner.
- **Real architectures:** engineering blogs from Netflix, Uber, Discord, Airbnb, Stripe.

## 15.4 How to Improve Over Time

- After every practice session, write down **one specific thing to improve next time**.
- Keep a running "mistakes log" — you'll see patterns.
- Revisit your own old designs a month later and critique them as if they weren't yours.
- Pair with other candidates — teaching accelerates learning.

---

# 16. Final Checklist

## You are ready for system design interviews if you can:

### Framework
- [ ] Explain the 7-step framework without looking at notes.
- [ ] Drive any interview through the framework without being prompted.
- [ ] Pace a 45-minute interview comfortably.

### Estimation
- [ ] Convert DAU → QPS → storage → bandwidth in under 3 minutes.
- [ ] Recall key latency numbers (network, disk, memory) from memory.
- [ ] Justify your architecture using the numbers you estimated.

### Building blocks
- [ ] Explain when to use SQL vs NoSQL, and which flavor of NoSQL.
- [ ] Explain three sharding strategies and their trade-offs.
- [ ] Describe cache-aside and when it breaks.
- [ ] Explain leader-follower replication and the lag problem.
- [ ] Explain consistent hashing and why it's useful.

### Concepts
- [ ] State the CAP theorem accurately, not the common oversimplification.
- [ ] Explain eventual consistency with a concrete example.
- [ ] Explain idempotency with a payment example.
- [ ] Describe the saga pattern and when to use it.
- [ ] Explain a circuit breaker and why it's needed.

### Design
- [ ] Design a URL shortener in 45 minutes from scratch.
- [ ] Design a rate limiter with distributed correctness.
- [ ] Design a chat system with offline delivery.
- [ ] Design a news feed with fan-out trade-offs.
- [ ] Design a file storage system with chunked uploads.

### Communication
- [ ] Speak in complete, structured sentences while thinking.
- [ ] Ask clarifying questions without sounding unsure.
- [ ] Receive pushback as collaboration, not attack.
- [ ] Summarize your design in under 2 minutes at the end.

### Judgment
- [ ] Name at least one trade-off for every design decision you make.
- [ ] Refuse to over-engineer for imaginary scale.
- [ ] State what you're explicitly *not* building.
- [ ] Commit to a choice and own it, even under scrutiny.

---

## Closing Note

System design interviews are not about knowing everything — they're about **thinking clearly under uncertainty**. The candidates who pass are the ones who communicate like engineers their interviewers would want to work with.

Draw. Talk. Estimate. Decide. Trade off. Repeat.

You've got this. Go ship.
