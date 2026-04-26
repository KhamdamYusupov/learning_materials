# Prometheus: Complete Production Guide for Java/Spring Boot Engineers

> For backend developers with solid Java experience building production-grade observability systems from scratch.

---

## Table of Contents

1. [Observability Fundamentals](#1-observability-fundamentals)
2. [Prometheus Fundamentals](#2-prometheus-fundamentals)
3. [Prometheus Architecture & How It Works](#3-prometheus-architecture--how-it-works)
4. [Metrics & Data Model](#4-metrics--data-model)
5. [PromQL — Complete Guide](#5-promql--complete-guide)
6. [Prometheus Configuration](#6-prometheus-configuration)
7. [Prometheus + Spring Boot](#7-prometheus--spring-boot)
8. [Alerting](#8-alerting)
9. [Visualization — Grafana Integration](#9-visualization--grafana-integration)
10. [Production Best Practices](#10-production-best-practices)
11. [Prometheus in System Design](#11-prometheus-in-system-design)
12. [Real-World Scenarios](#12-real-world-scenarios)
13. [Comparisons & Trade-offs](#13-comparisons--trade-offs)
14. [Production-Ready Checklist](#14-production-ready-checklist)

---

# 1. Observability Fundamentals

## What Observability Is

Observability is the ability to understand the **internal state of a system by examining its external outputs**. Coined from control theory, it answers: "Can I understand what's happening inside my system without having to ship new code?"

In distributed systems, things fail in novel ways. Observability is not about predicting failure modes — it's about building systems where you can answer **any question** about their behavior after the fact, even questions you didn't think to ask during development.

**Monitoring** is a subset: you monitor for known failure modes. Observability is broader: you can investigate unknown failures.

The practical difference:
- **Monitoring**: "Alert me when CPU > 80%" (you knew to ask this)
- **Observability**: "Why did this user's request fail at 14:32:17 on a Tuesday?" (you didn't predict this question)

## Metrics vs Logs vs Traces

These are the three pillars of observability. They answer different questions.

| Pillar | What it is | What it answers | Cost |
|---|---|---|---|
| **Metrics** | Numerical measurements aggregated over time | "What is happening?" "Is the system healthy?" | Low (aggregated) |
| **Logs** | Timestamped records of discrete events | "What happened specifically?" "What was the context?" | High (all events) |
| **Traces** | End-to-end request journey across services | "Where did this request spend its time?" "Which service was the bottleneck?" | Medium (sampled) |

### Metrics
Aggregated numerical data: counters, gauges, histograms. Efficient to store (one data point per interval, not per event). Great for alerting, dashboards, trends. Bad for debugging specific incidents without context.

### Logs
Structured or unstructured text events. Rich context per event. Bad for aggregation across millions of events ("how many errors per minute?") — you need dedicated tools (ELK, Loki) for that.

### Traces
Distributed request tracing (Jaeger, Zipkin, AWS X-Ray). Follows a request across service boundaries. Shows exactly where latency was introduced. Expensive to store for every request — typically sampled.

**In practice**: You need all three. Metrics alert you something is wrong. Logs tell you what happened. Traces show you where.

## Why Monitoring Is Critical in Distributed Systems

A monolith fails in one place. A microservices system of 20 services has 20 failure points that interact in unpredictable ways. Without monitoring:

- **Silent failures**: Service B calls Service C which is down. Service B returns defaults. Users see wrong data. No alert fires. Goes undetected for hours.
- **Cascading failures**: Slow database → connection pool exhaustion → all services timing out → entire platform down. Without metrics, you can't tell where it started.
- **Performance regressions**: A deploy slows the 99th percentile by 3x. Users complain. Without historical metrics, you can't tell which deploy caused it or when it started.
- **Capacity planning**: Without trends, you can't predict when to scale before you hit limits.

## White-Box vs Black-Box Monitoring

### Black-Box Monitoring
Tests the system from the **outside**, as a user would see it. Doesn't require instrumentation of the application.

Examples:
- HTTP health check: `GET /health` → expect 200
- Synthetic transactions: "can I log in and complete a purchase?"
- External uptime checkers (Pingdom, UptimeRobot)

**Pros**: Detects end-user visible failures. Works for any system.
**Cons**: Detects failures only after they affect users. No insight into *why* something failed.

### White-Box Monitoring
Instruments the system **from the inside**. The application explicitly reports its own metrics: JVM memory, request durations, DB connection pool usage, business metrics.

Examples:
- JVM heap usage, GC pause time
- HTTP request duration by endpoint, status code
- Queue depth, processing lag

**Pros**: Detects problems before they're user-visible. Gives rich context for debugging.
**Cons**: Requires instrumentation. Requires knowledge of what to measure.

**Production strategy**: You need both. Black-box confirms user-visible SLAs. White-box enables root cause analysis and early warning.

## Pull vs Push Monitoring Model

This is a fundamental architectural choice that defines how metrics get from your application into the monitoring system.

### Push Model
Application **sends** metrics to the monitoring system on a schedule.

```
Application ──────────► Monitoring System
           "here are my metrics"
```

Examples: StatsD, InfluxDB (via Telegraf), AWS CloudWatch agent, Datadog agent.

**Pros**:
- Works naturally behind firewalls (outbound traffic only)
- Short-lived jobs can push metrics before they die
- Simple to implement for fire-and-forget scenarios

**Cons**:
- If the monitoring system is overloaded, pushes succeed but data is lost (no feedback)
- Hard to know if an application is down vs just not pushing (is silence "no data" or "service down"?)
- Monitoring system must handle the write load from all agents simultaneously
- Distributed: config changes for metric endpoints must be deployed to each agent

### Pull Model (Prometheus's Model)
The monitoring system **scrapes** metrics from applications at a defined interval.

```
Monitoring System ────────► Application
                  "give me your metrics"
             ◄────────────────────────
                  "here they are"
```

**Pros**:
- Prometheus knows exactly which targets it's scraping — if a scrape fails, Prometheus knows the target is down (vs just not pushing)
- Centralized configuration: all scrape targets defined in Prometheus config
- Rate limiting is natural: Prometheus controls scrape frequency
- Easy to test: you can manually `curl` any target's `/metrics` endpoint
- Service discovery: Prometheus can dynamically find targets

**Cons**:
- Targets must be reachable from Prometheus (may require firewall rules, VPN)
- Short-lived jobs (batch jobs) die before Prometheus scrapes them → use **Pushgateway** as intermediary
- Large-scale deployments (millions of targets) require federation or sharding

**Why Prometheus chose pull**: Simpler operations. The monitoring system is in control. Target health is directly observable. The `/metrics` endpoint is a standard contract any tool can consume.

---

# 2. Prometheus Fundamentals

## What Prometheus Is

Prometheus is an **open-source monitoring and alerting toolkit** originally built at SoundCloud in 2012, now a CNCF graduated project. It is the de facto standard for Kubernetes and cloud-native monitoring.

At its core, Prometheus is:
1. A **time-series database** (TSDB) optimized for metrics storage
2. A **scraping engine** that pulls metrics from HTTP endpoints
3. A **query engine** (PromQL) for analyzing metric data
4. An **alerting engine** that evaluates rules and fires alerts

What Prometheus is NOT:
- A log aggregator (use Loki, ELK)
- A distributed tracing system (use Jaeger, Zipkin)
- A long-term metrics storage system (use Thanos, Cortex, VictoriaMetrics for that)
- An event streaming platform

## Core Features

### Time-Series Database
Prometheus stores metrics as time series: a sequence of (timestamp, value) pairs identified by a metric name and a set of key-value labels.

```
http_requests_total{method="GET", status="200", service="order-api"} @ 1705312800  → 15234
http_requests_total{method="GET", status="200", service="order-api"} @ 1705312860  → 15289
http_requests_total{method="POST", status="500", service="order-api"} @ 1705312860 → 12
```

Each unique combination of metric name + labels is one **time series**.

### Pull-Based Scraping
Every `scrape_interval` (default 15s), Prometheus sends an HTTP GET to `<target>/metrics`. The target responds with all its current metric values in the Prometheus exposition format.

### Multi-Dimensional Data Model (Labels)
Labels are key-value pairs that add dimensions to metrics. They enable powerful filtering and aggregation in PromQL without defining separate metric names.

```promql
# Total requests across all services
sum(http_requests_total)

# Requests per service
sum(http_requests_total) by (service)

# Error rate for the order service only
rate(http_requests_total{service="order-api", status=~"5.."}[5m])
```

## When to Use vs NOT Use Prometheus

### Use Prometheus When:
- Running Kubernetes workloads (Prometheus is the standard)
- Need infrastructure + application metrics in one system
- Want open-source, no vendor lock-in
- Time-series metrics (performance, utilization, error rates)
- Need flexible querying with PromQL
- Integrating with Grafana for dashboards

### Do NOT Use Prometheus When:
- **Long-term storage (>2 years)**: Prometheus's local TSDB is designed for ~15 days by default. For long-term, use Thanos or Cortex as a remote storage backend.
- **High-cardinality event data**: User IDs, request IDs, session IDs as labels create millions of time series — Prometheus's TSDB is not designed for this.
- **Log-like data**: Prometheus metrics are aggregated numbers. If you need "which user got this error," use logs.
- **Global multi-region aggregation as primary design**: Prometheus is single-server by design. Multi-region requires federation or a purpose-built system.
- **Financial audit trails**: Metrics can be aggregated and may lose individual event precision. Not suitable for "count every transaction exactly."

---

# 3. Prometheus Architecture & How It Works

## High-Level Architecture

```
                         ┌──────────────────────────────────────────────────┐
                         │              Prometheus Server                    │
                         │                                                  │
  ┌─────────────┐        │  ┌───────────────┐    ┌────────────────────┐    │
  │  Service    │◄───────┼──│  Scrape       │    │  TSDB              │    │
  │  /metrics   │        │  │  Engine       │───►│  (Local Storage)   │    │
  └─────────────┘        │  └───────────────┘    └──────┬─────────────┘    │
                         │         ▲                     │                  │
  ┌─────────────┐        │         │             ┌───────▼─────────────┐   │
  │  Exporter   │◄───────┼─────────┤             │  Query Engine       │   │
  │  /metrics   │        │         │             │  (PromQL)           │   │
  └─────────────┘        │  ┌──────┴──────┐      └───────┬─────────────┘   │
                         │  │  Service    │              │                  │
  ┌─────────────┐        │  │  Discovery  │      ┌───────▼─────────────┐   │
  │  Pushgateway│◄───────┼──│  (K8s, DNS, │      │  Alert Rules        │   │
  │  /metrics   │        │  │   static)   │      │  Evaluator          │   │
  └─────────────┘        │  └─────────────┘      └───────┬─────────────┘   │
                         └──────────────────────────────┬─┘                 │
                                                        │                   │
                    ┌───────────────┐    ┌──────────────▼────────┐          │
                    │  Grafana      │◄───│  HTTP API              │          │
                    │  Dashboards   │    └───────────────────────┘          │
                    └───────────────┘                                        │
                                         ┌──────────────────────┐           │
                                         │  Alertmanager         │◄──────────┘
                                         │  (routing, silencing) │
                                         └──────────┬───────────┘
                                                    │
                               ┌────────────────────┼────────────────────┐
                               ▼                    ▼                    ▼
                          Slack/Email          PagerDuty             Webhook
```

## Prometheus Server

The Prometheus server is a single binary that contains:
- **Scrape engine**: Fetches metrics from targets on schedule
- **TSDB**: Stores time series locally on disk
- **Rule evaluator**: Evaluates recording and alerting rules
- **PromQL engine**: Processes queries from API/UI
- **HTTP API**: Used by Grafana, CLI, and web UI

Prometheus is intentionally **single-node**. No built-in clustering. This keeps it simple and operationally reliable. For scale, use federation or remote write to Thanos/Cortex.

## Targets & Scraping

A **target** is an HTTP endpoint that Prometheus scrapes. Every target exposes metrics at `/metrics` (by default) in the **Prometheus text exposition format**.

```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 15234
http_requests_total{method="POST",status="500"} 12
# HELP process_resident_memory_bytes Resident memory size in bytes
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 1.35e+08
```

Scrape lifecycle:
1. Prometheus selects a target based on scrape config
2. Sends `GET /metrics` (or configured path) with configured timeout
3. Parses the response
4. Stores each time series with the scrape timestamp
5. Records `up{job="...", instance="..."}` = 1 (success) or 0 (failure)
6. Waits for next `scrape_interval`

## Exporters

An **exporter** is a process that translates non-Prometheus metrics into the Prometheus exposition format. They bridge the gap between systems that don't expose `/metrics` natively.

| Exporter | What it monitors |
|---|---|
| `node_exporter` | Linux host metrics (CPU, memory, disk, network) |
| `blackbox_exporter` | HTTP/TCP/DNS probes (black-box monitoring) |
| `postgres_exporter` | PostgreSQL metrics |
| `redis_exporter` | Redis metrics |
| `kafka_exporter` | Kafka metrics |
| `jmx_exporter` | JVM/Java application metrics via JMX |
| `kube-state-metrics` | Kubernetes object state (pods, deployments) |

Exporters run as separate processes alongside the monitored system. Prometheus scrapes the exporter, the exporter queries the monitored system internally.

## Service Discovery

In dynamic environments (Kubernetes), targets come and go. Static configuration can't keep up. Prometheus supports:

### Kubernetes Service Discovery

```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Use custom port from annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${1}
```

Prometheus watches the Kubernetes API for pod changes. When a new pod starts with the right annotation, it's automatically added as a scrape target. When it dies, it's removed.

**Other discovery mechanisms**: DNS, Consul, AWS EC2, Azure, GCE, static file (for semi-dynamic configs).

## TSDB Internals

Prometheus's time-series database (written from scratch in Go) is optimized for write-heavy workloads with sequential reads.

### Storage Layout

```
prometheus/
└── data/
    ├── 01BKGTZQ1SYQJTR4PB43C8PD98/   ← completed block (2h of data)
    │   ├── chunks/
    │   │   ├── 000001               ← raw sample data (compressed)
    │   ├── index                    ← inverted index for labels
    │   ├── meta.json                ← block metadata
    │   └── tombstones               ← deleted series markers
    ├── 01BKGV7JC0RY8A6LMJTEX3W2GY/   ← another completed block
    ├── chunks_head/                 ← in-memory write buffer (current 2h)
    └── wal/                         ← Write-Ahead Log (crash recovery)
        ├── 00000001
        └── 00000002
```

### How Data is Stored

1. **WAL (Write-Ahead Log)**: Every sample is first written to the WAL for crash safety. If Prometheus crashes, it replays the WAL on restart.

2. **In-memory head block**: The most recent ~2 hours of data lives in memory for fast writes and reads. Samples are compressed using XOR encoding (delta-of-delta for timestamps, XOR for values) — achieving ~1.37 bytes per sample.

3. **Completed blocks**: Every 2 hours, the head block is persisted to disk as a new block. Blocks are immutable once written.

4. **Compaction**: Periodically, smaller blocks are merged into larger ones (up to the configured max block duration, default 25% of retention). Compaction reduces files, improves query performance, and removes deleted/expired data.

### Retention

```yaml
# In prometheus configuration or CLI flags
--storage.tsdb.retention.time=15d    # Delete blocks older than 15 days
--storage.tsdb.retention.size=50GB   # Delete oldest blocks when total exceeds 50GB
```

Default retention is **15 days**. For longer retention, configure `remote_write` to Thanos, Cortex, or VictoriaMetrics.

---

# 4. Metrics & Data Model

## The Data Model

Every metric in Prometheus is identified by:
1. **Metric name**: describes what is measured (`http_requests_total`)
2. **Labels**: key-value pairs that add dimensions (`{method="GET", status="200"}`)
3. **Timestamp**: when the sample was collected
4. **Value**: a 64-bit float

A **time series** is a unique combination of metric name + label set. Each time series has its own stream of (timestamp, value) pairs.

```
<metric_name>{<label_name>=<label_value>, ...} <value> [<timestamp>]

# Example:
http_requests_total{service="order-api", method="GET", status="200"} 15234 1705312800000
```

## Counter

### What it is
A **monotonically increasing value** that only goes up (or resets to 0 on restart). Represents a count of events that have occurred.

**Real meaning**: Cumulative total since the process started. You never query the raw value — you always use `rate()` or `increase()` to get the rate of change.

### When to use
- HTTP request count
- Error count
- Number of jobs processed
- Bytes sent/received
- Any event that happens and you want to count occurrences

### Example

```java
// Micrometer
Counter requestCounter = Counter.builder("http.requests.total")
    .tag("method", "GET")
    .tag("status", "200")
    .description("Total HTTP requests")
    .register(meterRegistry);

requestCounter.increment();
```

Exposed as:
```
http_requests_total{method="GET",status="200"} 15234.0
```

PromQL usage:
```promql
# Requests per second over last 5 minutes
rate(http_requests_total[5m])

# Total requests in the last hour
increase(http_requests_total[1h])
```

### Pitfalls
- **Never use a gauge for counters**: If a value can only go up, use Counter — Prometheus can detect resets (process restart) and handle them in `rate()`.
- **Querying raw value is meaningless**: `http_requests_total = 152340` tells you nothing useful without knowing the timeframe. Always use `rate()`.
- **Counter reset handling**: `rate()` automatically handles counter resets (process restarts). If the counter drops, `rate()` detects the reset and calculates correctly.

---

## Gauge

### What it is
A **value that can go up or down**. Represents a current state or measurement at a point in time.

### When to use
- Current memory usage
- Active connections
- Queue depth (current number of messages waiting)
- Temperature
- Number of currently running goroutines/threads
- CPU utilization at this moment

### Example

```java
// Micrometer
Gauge.builder("jvm.memory.used", runtime, r -> r.totalMemory() - r.freeMemory())
    .tag("area", "heap")
    .description("JVM heap memory used")
    .register(meterRegistry);

// Or a tracked object
AtomicInteger activeConnections = new AtomicInteger(0);
Gauge.builder("db.connections.active", activeConnections, AtomicInteger::get)
    .register(meterRegistry);
```

Exposed as:
```
jvm_memory_used_bytes{area="heap"} 1.35e+08
```

PromQL usage:
```promql
# Current value
jvm_memory_used_bytes{area="heap"}

# Percentage of heap used
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100
```

### Pitfalls
- **Don't use gauge for counters**: A gauge for "total requests" will look like it spikes and drops — meaningless.
- **Point-in-time sampling**: Gauge value between scrapes is unknown. If the queue depth spikes and drops between scrapes, you'll miss it. For such cases, consider tracking a counter of enqueues and dequeues separately.

---

## Histogram

### What it is
A **distribution of observations** across configurable buckets. Measures the distribution of values like request durations, response sizes.

A histogram with name `<metric>` exposes three time series:
- `<metric>_bucket{le="<bound>"}`: cumulative count of observations ≤ bound
- `<metric>_sum`: sum of all observed values
- `<metric>_count`: total number of observations

### When to use
- Request latency distribution
- Response sizes
- Database query durations
- Any measurement where you need percentiles

### Example

```java
// Micrometer Timer (backed by Histogram)
Timer requestTimer = Timer.builder("http.server.requests")
    .tag("method", "GET")
    .tag("uri", "/api/orders")
    .publishPercentileHistogram()          // enables histogram buckets
    .slo(                                  // custom SLO buckets
        Duration.ofMillis(10),
        Duration.ofMillis(50),
        Duration.ofMillis(100),
        Duration.ofMillis(500),
        Duration.ofSeconds(1))
    .register(meterRegistry);

requestTimer.record(() -> processRequest());
```

Exposed as:
```
http_server_requests_seconds_bucket{le="0.01"} 1520
http_server_requests_seconds_bucket{le="0.05"} 4320
http_server_requests_seconds_bucket{le="0.1"}  5890
http_server_requests_seconds_bucket{le="0.5"}  6120
http_server_requests_seconds_bucket{le="1.0"}  6134
http_server_requests_seconds_bucket{le="+Inf"} 6141
http_server_requests_seconds_sum   45.23
http_server_requests_seconds_count 6141
```

PromQL usage:
```promql
# 95th percentile latency over last 5 minutes
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))

# Average latency
rate(http_server_requests_seconds_sum[5m]) / rate(http_server_requests_seconds_count[5m])
```

### Pitfalls
- **Bucket configuration is critical**: Default buckets may not match your use case. A default bucket of 10 seconds is useless for an API with 99th percentile < 500ms. Define buckets that bracket your expected SLO range.
- **Histograms are cumulative**: `le="+Inf"` is always equal to `_count`. Each bucket includes all observations up to that bound.
- **Aggregating histograms across instances**: `histogram_quantile` can be applied across multiple instances because the underlying buckets are counters that can be summed. This is a major advantage over Summary.
- **Memory cost**: Each bucket is a separate time series. 10 buckets × 100 histogram metrics = 1000+ time series per instance.

---

## Summary

### What it is
Similar to Histogram, but calculates **quantiles on the client side** (in the application). Exposes pre-computed quantiles.

A summary with name `<metric>` exposes:
- `<metric>{quantile="0.5"}`: 50th percentile
- `<metric>{quantile="0.95"}`: 95th percentile
- `<metric>_sum`: total sum
- `<metric>_count`: total count

### When to use
Summary is now largely superseded by Histogram for most use cases. Use Summary only when:
- You need accurate quantiles at a specific instance level
- Your SLO quantile thresholds are fixed and you don't need to aggregate across instances

### Histogram vs Summary

| Aspect | Histogram | Summary |
|---|---|---|
| Quantile calculation | Server-side (PromQL) | Client-side (in app) |
| Aggregation across instances | **Possible** — sum buckets, then calculate | **Not possible** — can't sum pre-calculated quantiles |
| Configuration | Bucket bounds at instrument time | Quantiles at instrument time |
| Accuracy | Approximate (within bucket bounds) | Configurable accuracy (costs memory) |
| PromQL flexibility | Any quantile, any time window | Fixed quantiles only |

**In 2024, prefer Histogram over Summary** for almost all use cases.

---

## Labels: The Multi-Dimensional Model

Labels transform a metric from a single number into a queryable dataset.

```
Without labels: http_requests_total = 45230
With labels:    http_requests_total{service="order", method="POST", status="500"} = 12
                http_requests_total{service="order", method="GET",  status="200"} = 15234
                http_requests_total{service="user",  method="GET",  status="200"} = 8901
```

With labels, one metric covers all dimensions. PromQL can slice and dice:
```promql
# All errors for the order service
sum(http_requests_total{service="order", status=~"5.."})

# Error rate per service
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
```

### The Cardinality Problem

**Cardinality** is the number of unique time series created by a metric. It equals the product of unique values across all label dimensions.

```
http_requests_total with:
  service: 10 unique values
  method: 5 unique values (GET, POST, PUT, DELETE, PATCH)
  status: 20 unique values (200, 201, 400, 401, 403, 404, 500, 502, 503...)
  
Cardinality = 10 × 5 × 20 = 1,000 time series
```

**High cardinality kills Prometheus.** Each unique time series consumes memory in the head block, CPU for indexing, and disk for storage. At millions of time series, Prometheus becomes unusable.

### What Causes High Cardinality

```java
// CATASTROPHIC: userId as a label — millions of unique users = millions of time series
Counter.builder("api.calls")
    .tag("userId", userId.toString())   // ← NEVER DO THIS
    .register(meterRegistry);

// CATASTROPHIC: full URL path with IDs
Counter.builder("http.requests")
    .tag("path", "/orders/12345/items") // ← path contains entity ID
    .register(meterRegistry);

// CATASTROPHIC: request IDs, session IDs, trace IDs as labels
Counter.builder("requests")
    .tag("requestId", requestId)        // ← unique per request
    .register(meterRegistry);
```

### Label Best Practices

```java
// CORRECT: low-cardinality labels only
Counter.builder("api.calls")
    .tag("service", "order-service")      // ~10 unique values
    .tag("endpoint", "GET /orders")        // ~50 unique values (normalized)
    .tag("status", String.valueOf(status.value()))  // ~20 unique values
    .register(meterRegistry);

// CORRECT: normalize path parameters
// /orders/12345 → /orders/{id}
// /users/abc/orders → /users/{userId}/orders
```

**Label cardinality guidelines**:
- Target total time series per Prometheus instance: <10 million
- Labels should have <100 unique values ideally
- Never use: user IDs, session IDs, request IDs, email addresses, free-form text
- Good labels: service name, endpoint template, HTTP method, status code, region, environment, pod name

### Automatic Labels Added by Prometheus

When Prometheus scrapes a target, it adds labels from the scrape config:
- `job`: the scrape job name
- `instance`: the `host:port` of the target

These are always available without instrumentation.

---

# 5. PromQL — Complete Guide

## What PromQL Is

PromQL (Prometheus Query Language) is a functional query language for selecting and aggregating time-series data. It is expression-based — every query returns one of:

- **Instant vector**: Set of time series, one value per series at a single point in time
- **Range vector**: Set of time series, multiple values per series over a time range
- **Scalar**: A single numeric value
- **String**: A string (rare)

## Basic Queries

### Selecting Metrics

```promql
# All time series for a metric
http_requests_total

# Exact label match
http_requests_total{job="order-service"}

# Multiple label matchers
http_requests_total{job="order-service", method="GET"}

# Regex match (~=)
http_requests_total{status=~"5.."}       # 5xx errors

# Negative match (!=)
http_requests_total{status!="200"}

# Negative regex (!~)
http_requests_total{method!~"GET|POST"}

# Range vector: values over last 5 minutes
http_requests_total[5m]

# Offset: query from 1 hour ago
http_requests_total offset 1h
```

### Label Filtering

```promql
# Contains a label (any value)
http_requests_total{region=~".+"}

# Does NOT contain a label / has empty value
http_requests_total{region=""}

# Specific values for multiple labels (OR logic within same label, use regex)
http_requests_total{status=~"200|201|202"}
```

### Aggregation Operators

```promql
# Sum all time series into one
sum(http_requests_total)

# Sum grouped by a label
sum(http_requests_total) by (service)

# Sum everything EXCEPT specified labels
sum(http_requests_total) without (instance, pod)

# Average
avg(http_server_requests_seconds)

# Maximum value across instances
max(jvm_memory_used_bytes) by (service)

# Count number of time series
count(up{job="kubernetes-pods"})

# Count distinct values of a label (approximation)
count(count by (instance)(up))
```

---

## Functions — Deep Dive

### `rate()`

**What it does**: Calculates the **per-second average rate of increase** of a counter over a time window. Handles counter resets (process restarts).

**Signature**: `rate(v range-vector)`

```promql
# Requests per second over last 5 minutes
rate(http_requests_total[5m])

# Error rate per service
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
```

**How it works**: Takes the first and last samples in the window, calculates (last - first) / window_seconds. If a reset is detected (value drops), it adds the pre-reset value.

**Choosing the window `[5m]`**:
- Too short (e.g., `[1m]`): noisy, spiky graphs. With 15s scrape interval and a 1m window, you only have 4 samples — high variance.
- Too long (e.g., `[1h]`): smooths out spikes you need to see, slow to react to changes
- **Rule**: window should be at least 4× the scrape interval. 15s scrape → `[1m]` minimum. Common choice: `[5m]` for dashboards, `[15m]` for slower trend graphs.

**Real-world usage**:
```promql
# HTTP request rate by method and status
sum(rate(http_requests_total[5m])) by (method, status)

# Error percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m])) * 100
```

**Pitfalls**:
- Only valid for counters (monotonically increasing values). Using `rate()` on a gauge is meaningless.
- `rate()` returns per-second rate, not total count. For total in window, use `increase()`.
- If the scrape interval is larger than the rate window, you get zero results or inaccurate data.

---

### `irate()`

**What it does**: Calculates the **instantaneous rate** using only the **last two data points** in the range vector. More reactive to sudden spikes than `rate()`.

**Signature**: `irate(v range-vector)`

```promql
irate(http_requests_total[5m])
```

**`rate` vs `irate`**:
| | `rate()` | `irate()` |
|---|---|---|
| Calculation | avg over entire window | last 2 data points only |
| Smoothing | High — averages out spikes | None — shows instantaneous changes |
| Use for | Dashboards, alerts | Detecting sudden spikes, volatile data |
| Alerting | Preferred | Use carefully |

**Real-world usage**: Detecting sudden traffic spikes in real-time dashboards. Not recommended for alerting rules (too noisy — single-sample spikes fire alerts).

**Pitfall**: The range vector `[5m]` in `irate()` is only used to find the last two samples — all other samples in the window are ignored. The window size affects staleness detection but not the calculation.

---

### `increase()`

**What it does**: Returns the **total increase** of a counter over a time range. Equivalent to `rate() * duration_in_seconds`.

**Signature**: `increase(v range-vector)`

```promql
# Total HTTP requests in the last hour
increase(http_requests_total[1h])

# New errors in the last 15 minutes
increase(http_requests_total{status=~"5.."}[15m])
```

**`rate()` vs `increase()`**:
- `rate(x[5m])` = requests per second (average over 5 minutes)
- `increase(x[5m])` = total requests in 5 minutes = `rate(x[5m]) * 300`

**Real-world usage**: "How many orders were placed in the last hour?" "How many errors occurred today?"

**Pitfall**: `increase()` extrapolates. If a counter was scraped twice in a 5-minute window with values 100 and 115, `increase([5m])` doesn't just return 15 — it extrapolates to fill the full window. This can lead to non-integer results even for integer counters.

---

### `sum()`

**What it does**: Aggregates multiple time series into one by summing their values.

```promql
# Total requests across all instances
sum(http_requests_total)

# Total requests per service (group by service, sum across instances)
sum(http_requests_total) by (service)

# Sum with rate (order matters: rate first, then sum)
sum(rate(http_requests_total[5m])) by (service)
```

**Pitfall — aggregating rates**: Always apply `rate()` before `sum()`, not after:

```promql
# CORRECT: rate() on raw counter, then sum
sum(rate(http_requests_total[5m])) by (service)

# WRONG: sum raw counters across instances, then rate()
# This adds counters from different instances which may have different start times
rate(sum(http_requests_total)[5m])  # ← produces wrong results
```

---

### `avg()`

**What it does**: Calculates the average value across multiple time series.

```promql
# Average CPU usage across all nodes
avg(cpu_usage_percent) by (region)

# Average request latency per service
avg(rate(http_server_requests_seconds_sum[5m]) / rate(http_server_requests_seconds_count[5m])) by (service)
```

**Pitfall**: `avg()` of CPU across nodes hides outliers. A cluster where 4 nodes are at 20% and 1 node is at 100% shows average 36% — healthy-looking but one node is at capacity. Use `max()` for capacity alerts.

---

### `max()` / `min()`

**What they do**: Return the maximum/minimum value across multiple time series.

```promql
# Worst-case memory usage across all JVM instances
max(jvm_memory_used_bytes{area="heap"}) by (service)

# Node with highest CPU
max(cpu_usage_percent) by (instance)

# Minimum available disk space across all nodes
min(disk_available_bytes) by (node)
```

**Real-world usage**: Capacity alerts should use `max()`, not `avg()`. One node at 100% CPU is a problem even if average is 30%.

---

### `histogram_quantile()`

**What it does**: Calculates a quantile (percentile) from a histogram metric. This is the **standard way to get p50/p95/p99 latency**.

**Signature**: `histogram_quantile(φ scalar, b instant-vector)`

```promql
# 95th percentile request latency
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket[5m]))

# 99th percentile per service
histogram_quantile(0.99,
    sum(rate(http_server_requests_seconds_bucket[5m])) by (le, service)
)

# 50th percentile (median)
histogram_quantile(0.5, rate(http_server_requests_seconds_bucket[5m]))
```

**How it works**: The `le` (less-than-or-equal) label is special — it's the bucket boundary. `histogram_quantile` finds the bucket where `φ` fraction of observations fall, then linearly interpolates within that bucket.

**The `le` label MUST be present** in the aggregation. When using `sum() by (...)`, always include `le`:

```promql
# CORRECT: le is included
sum(rate(http_server_requests_seconds_bucket[5m])) by (le, service)

# WRONG: le is lost, histogram_quantile gets wrong data
sum(rate(http_server_requests_seconds_bucket[5m])) by (service)
```

**Pitfall — bucket boundary accuracy**: The result is only as accurate as your bucket configuration. If 95% of requests fall in the 100ms–500ms bucket, you know p95 is in that range but not exactly where. Configure buckets to bracket your SLO thresholds tightly.

```promql
# p99 across all services combined (requires le in sum)
histogram_quantile(0.99,
    sum(rate(http_server_requests_seconds_bucket[5m])) by (le)
)
```

---

### `count()`

**What it does**: Counts the number of time series in the input vector.

```promql
# Number of running pods per deployment
count(kube_pod_info) by (namespace, deployment)

# Number of service instances
count(up{job="order-service"}) 

# How many instances are down
count(up == 0)
```

**Real-world usage**: Fleet size tracking, ensuring minimum replica counts.

---

### `topk()` / `bottomk()`

**What they do**: Return the K time series with the highest/lowest values.

**Signature**: `topk(k scalar, instant-vector)` / `bottomk(k scalar, instant-vector)`

```promql
# Top 5 endpoints by request rate
topk(5, sum(rate(http_requests_total[5m])) by (endpoint))

# Top 10 services by error rate
topk(10, sum(rate(http_requests_total{status=~"5.."}[5m])) by (service))

# 3 pods with highest memory usage
topk(3, jvm_memory_used_bytes{area="heap"})

# Slowest 5 endpoints by p99 latency
topk(5,
    histogram_quantile(0.99,
        sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri)
    )
)
```

**Pitfall**: `topk/bottomk` return different series at different timestamps — they're not stable over time. Don't use them in alerting rules if you need stable label sets.

---

## Advanced PromQL

### Aggregation with `by` and `without`

```promql
# Sum by specific labels (keep only these)
sum(http_requests_total) by (service, status)

# Sum, removing specific labels (keep everything else)
sum(http_requests_total) without (instance, pod, namespace)
```

`by` and `without` are complementary. Use `by` when you know exactly what dimensions you need. Use `without` when you want to aggregate away noisy dimensions (like `instance`) while keeping everything else.

### Joins Between Metrics

PromQL supports binary operations between instant vectors. For two vectors to match element-by-element, they must have the same label set (or you specify matching with `on`/`ignoring`).

```promql
# Error rate (fraction of requests that error)
rate(http_requests_total{status=~"5.."}[5m])
/
rate(http_requests_total[5m])

# Memory saturation: used / max
jvm_memory_used_bytes / jvm_memory_max_bytes

# Multi-dimensional join: CPU usage per core, keeping only "mode" label to match
sum without(cpu)(rate(node_cpu_seconds_total{mode!="idle"}[5m]))
/ ignoring(mode)
sum without(cpu)(rate(node_cpu_seconds_total[5m]))
```

**Vector matching modifiers**:
- `on(labels)`: only match on specified labels
- `ignoring(labels)`: ignore specified labels when matching
- `group_left` / `group_right`: many-to-one joins

```promql
# Join pod info with request metrics (pod labels on both sides)
http_requests_total
* on(pod) group_left(node)
kube_pod_info
```

### Recording Rules

Recording rules pre-compute expensive PromQL expressions and store results as new time series. This speeds up dashboards and reduces query load.

```yaml
# prometheus/rules/recording_rules.yml
groups:
  - name: http_aggregations
    interval: 30s  # evaluate every 30s (default: global evaluation_interval)
    rules:
      # Pre-compute per-service request rate
      - record: job:http_requests:rate5m
        expr: sum(rate(http_requests_total[5m])) by (job)

      # Pre-compute per-service error rate
      - record: job:http_errors:rate5m
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) by (job)

      # Pre-compute p99 latency per service
      - record: job:http_request_duration_seconds:p99_rate5m
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le, job)
          )
```

**Naming convention for recording rules**: `level:metric:operations`
- `level`: aggregation level (job, instance, cluster)
- `metric`: the metric name
- `operations`: operations applied (rate5m, p99, etc.)

**When to use recording rules**:
- Dashboards that run the same expensive query repeatedly
- Alerting rules that require complex PromQL (recording rules make alert evaluation faster)
- Cross-metric ratios that are frequently queried

### Alerting Expressions

```promql
# High error rate: > 1% of requests are errors for more than 5 minutes
sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
/ sum(rate(http_requests_total[5m])) by (service) > 0.01

# High latency: p99 > 500ms
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, service))
> 0.5

# Instance down
up{job="order-service"} == 0

# High memory: heap > 90%
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} > 0.9
```

---

# 6. Prometheus Configuration

## prometheus.yml — Complete Example

```yaml
# prometheus.yml

# Global defaults: apply to all scrape configs unless overridden
global:
  scrape_interval: 15s          # How often to scrape targets
  scrape_timeout: 10s           # Timeout for each scrape
  evaluation_interval: 15s      # How often to evaluate alert/recording rules
  
  # Labels added to all time series and alerts from this Prometheus instance
  external_labels:
    cluster: production-us-east-1
    environment: production

# Rules files: alert and recording rules
rule_files:
  - "rules/recording_rules.yml"
  - "rules/alert_rules.yml"
  - "rules/*.yml"              # Glob supported

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093
      timeout: 10s

# Scrape configurations
scrape_configs:

  # Scrape Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Static targets: known, fixed hosts
  - job_name: 'node-exporters'
    static_configs:
      - targets:
          - 'node1.example.com:9100'
          - 'node2.example.com:9100'
          - 'node3.example.com:9100'
        labels:
          region: us-east-1
          env: production

  # Spring Boot application
  - job_name: 'order-service'
    metrics_path: '/actuator/prometheus'  # Spring Boot exposes here
    scrape_interval: 15s
    static_configs:
      - targets:
          - 'order-service:8080'
    # Relabeling: modify labels before storage
    relabel_configs:
      - source_labels: [__address__]
        target_label: instance
        regex: '(.+):.*'
        replacement: '$1'    # Remove port from instance label

  # PostgreSQL via exporter
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
    relabel_configs:
      - target_label: service
        replacement: postgres

  # HTTP blackbox probing
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]      # Use http_2xx module from blackbox config
    static_configs:
      - targets:
          - https://api.example.com/health
          - https://api.example.com/api/v1/products
    relabel_configs:
      # Move target URL to 'instance' label
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      # Point scrape at blackbox exporter
      - target_label: __address__
        replacement: blackbox-exporter:9115

  # Remote write to long-term storage (Thanos/VictoriaMetrics)
remote_write:
  - url: "http://thanos-receive:10908/api/v1/receive"
    queue_config:
      max_samples_per_send: 10000
      max_shards: 200
      capacity: 2500
```

## Kubernetes Service Discovery

```yaml
scrape_configs:

  # Scrape all pods with annotation prometheus.io/scrape: "true"
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names:
            - production
            - staging

    relabel_configs:
      # Only scrape pods with the annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: 'true'

      # Use custom path from annotation (default /metrics)
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

      # Use custom port from annotation
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

      # Add Kubernetes labels as Prometheus labels
      - action: labelmap
        regex: __meta_kubernetes_pod_label_(.+)

      # Add namespace label
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace

      # Add pod name label
      - source_labels: [__meta_kubernetes_pod_name]
        target_label: pod

      # Add container name label
      - source_labels: [__meta_kubernetes_pod_container_name]
        target_label: container

  # Scrape Kubernetes API server
  - job_name: 'kubernetes-apiservers'
    kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names: [default]
    scheme: https
    tls_config:
      ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
        action: keep
        regex: kubernetes;https

  # Scrape kube-state-metrics
  - job_name: 'kube-state-metrics'
    static_configs:
      - targets: ['kube-state-metrics.monitoring.svc.cluster.local:8080']
```

## Pod Annotation for Auto-Discovery

Add these annotations to your Kubernetes deployments:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      containers:
        - name: order-service
          image: order-service:latest
          ports:
            - containerPort: 8080
```

---

# 7. Prometheus + Spring Boot

## Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <!-- Spring Boot Actuator: exposes /actuator/* endpoints -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <!-- Micrometer Prometheus registry: formats metrics for Prometheus -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <!-- Micrometer core (usually pulled transitively) -->
    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-core</artifactId>
    </dependency>
</dependencies>
```

## application.yml Configuration

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics  # expose these actuator endpoints
  endpoint:
    prometheus:
      enabled: true
    health:
      show-details: always
  metrics:
    tags:
      # Global tags added to ALL metrics — critical for multi-service environments
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:development}
      region: ${AWS_REGION:us-east-1}
    distribution:
      # Configure histogram buckets for timers
      percentiles-histogram:
        http.server.requests: true          # enable histogram for HTTP requests
      slo:
        http.server.requests: 10ms,50ms,100ms,200ms,500ms,1s,2s  # SLO buckets
      percentiles:
        http.server.requests: 0.5,0.95,0.99  # publish these percentiles (Summary mode)
  
spring:
  application:
    name: order-service
```

**Important**: After configuration, Spring Boot exposes metrics at `/actuator/prometheus`.

```bash
# Test it
curl http://localhost:8080/actuator/prometheus | head -50
```

---

## Example 1: Default Auto-Configured Metrics

Spring Boot with Micrometer auto-configures dozens of metrics out of the box:

```
# JVM memory
jvm_memory_used_bytes{area="heap", id="G1 Eden Space"}
jvm_memory_max_bytes{area="heap"}
jvm_gc_pause_seconds_bucket  (histogram)

# JVM threads
jvm_threads_live_threads
jvm_threads_daemon_threads

# HTTP server requests (auto-instrumented)
http_server_requests_seconds_bucket{method="GET",status="200",uri="/api/orders"}
http_server_requests_seconds_count
http_server_requests_seconds_sum

# Database connection pool (if using HikariCP)
hikaricp_connections_active{pool="HikariPool-1"}
hikaricp_connections_idle
hikaricp_connections_pending
hikaricp_connections_max

# Process
process_cpu_usage
process_uptime_seconds

# System
system_cpu_usage
system_load_average_1m

# Logback
logback_events_total{level="warn"}
logback_events_total{level="error"}
```

These metrics require zero code — just the dependency and configuration.

**Critical PromQL queries for these defaults**:
```promql
# Error rate
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (uri)
/ sum(rate(http_server_requests_seconds_count[5m])) by (uri)

# p99 latency per endpoint
histogram_quantile(0.99,
    sum(rate(http_server_requests_seconds_bucket[5m])) by (le, uri, method)
)

# DB connection pool saturation
hikaricp_connections_active / hikaricp_connections_max

# JVM heap utilization
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"}
```

---

## Example 2: Custom Counter

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final MeterRegistry meterRegistry;
    private final OrderRepository orderRepository;
    
    // Register counter once at startup (efficient — don't create per-request)
    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Counter ordersCancelled;
    
    public OrderService(MeterRegistry meterRegistry, OrderRepository orderRepository) {
        this.meterRegistry = meterRegistry;
        this.orderRepository = orderRepository;
        
        // Counter: number of orders created (ever increasing)
        this.ordersCreated = Counter.builder("orders.created.total")
            .description("Total number of orders successfully created")
            .tag("service", "order-service")
            .register(meterRegistry);
        
        this.ordersFailed = Counter.builder("orders.failed.total")
            .description("Total number of order creation failures")
            .tag("service", "order-service")
            .register(meterRegistry);
        
        this.ordersCancelled = Counter.builder("orders.cancelled.total")
            .description("Total number of cancelled orders")
            .tag("service", "order-service")
            .register(meterRegistry);
    }
    
    public Order createOrder(CreateOrderRequest request) {
        try {
            Order order = orderRepository.save(Order.from(request));
            ordersCreated.increment();  // increment on success
            return order;
        } catch (Exception e) {
            ordersFailed.increment();  // increment on failure
            throw e;
        }
    }
    
    public void cancelOrder(Long orderId, String reason) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.cancel(reason);
        orderRepository.save(order);
        
        // Counter with dynamic tags: low-cardinality tags only
        meterRegistry.counter("orders.cancelled.total",
            "reason", reason.length() > 50 ? "other" : reason  // normalize
        ).increment();
    }
}
```

**PromQL for these counters**:
```promql
# Order creation rate (per minute)
rate(orders_created_total[5m]) * 60

# Order failure rate
rate(orders_failed_total[5m]) / rate(orders_created_total[5m]) * 100

# Cancellation trend
increase(orders_cancelled_total[1h])
```

---

## Example 3: Custom Gauge

```java
@Service
@RequiredArgsConstructor
public class QueueMonitoringService {

    private final MeterRegistry meterRegistry;
    private final OrderQueueService orderQueueService;
    private final InventoryService inventoryService;
    
    @PostConstruct
    public void registerGauges() {
        
        // Gauge: points to a live object — value is fetched at scrape time
        Gauge.builder("orders.queue.depth", orderQueueService, svc -> svc.getPendingCount())
            .description("Number of orders waiting to be processed")
            .tag("queue", "pending-orders")
            .register(meterRegistry);
        
        Gauge.builder("orders.queue.depth", orderQueueService, svc -> svc.getProcessingCount())
            .description("Number of orders currently being processed")
            .tag("queue", "processing")
            .register(meterRegistry);
        
        // Gauge tracking an AtomicInteger directly
        AtomicInteger activeWorkers = new AtomicInteger(0);
        Gauge.builder("workers.active", activeWorkers, AtomicInteger::doubleValue)
            .description("Number of active worker threads")
            .register(meterRegistry);
        
        // Functional gauge: computed from an external service
        Gauge.builder("inventory.low.stock.items", inventoryService, 
                       svc -> svc.countItemsBelowReorderPoint())
            .description("Number of products below reorder threshold")
            .register(meterRegistry);
    }
}
```

**PromQL for gauges**:
```promql
# Current queue depth
orders_queue_depth{queue="pending-orders"}

# Alert: queue growing (derivative over 10 min)
deriv(orders_queue_depth{queue="pending-orders"}[10m]) > 5

# Alert: queue depth too high
orders_queue_depth{queue="pending-orders"} > 1000
```

---

## Example 4: Custom Timer (Histogram)

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    private final MeterRegistry meterRegistry;
    private final PaymentGateway paymentGateway;
    
    // Timer is a Histogram + Counter combined
    private final Timer paymentProcessingTimer;
    private final Timer paymentGatewayTimer;
    
    public PaymentService(MeterRegistry meterRegistry, PaymentGateway paymentGateway) {
        this.meterRegistry = meterRegistry;
        this.paymentGateway = paymentGateway;
        
        this.paymentProcessingTimer = Timer.builder("payment.processing.duration")
            .description("Time to process a complete payment")
            .tag("service", "payment-service")
            .publishPercentileHistogram()   // enable histogram buckets for p99 in PromQL
            .slo(                            // service-level objective buckets
                Duration.ofMillis(100),
                Duration.ofMillis(250),
                Duration.ofMillis(500),
                Duration.ofSeconds(1),
                Duration.ofSeconds(2),
                Duration.ofSeconds(5))
            .register(meterRegistry);
        
        this.paymentGatewayTimer = Timer.builder("payment.gateway.duration")
            .description("Time for payment gateway response")
            .tag("gateway", "stripe")
            .publishPercentileHistogram()
            .register(meterRegistry);
    }
    
    public PaymentResult processPayment(PaymentRequest request) {
        // Method 1: record(() -> ...)
        return paymentProcessingTimer.record(() -> {
            PaymentResult gatewayResult = paymentGatewayTimer.record(() ->
                paymentGateway.charge(request)
            );
            return processGatewayResult(gatewayResult, request);
        });
    }
    
    // Method 2: manual start/stop
    public PaymentResult processPaymentManual(PaymentRequest request) {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            PaymentResult result = paymentGateway.charge(request);
            sample.stop(meterRegistry.timer("payment.processing.duration",
                "status", "success",
                "gateway", "stripe"));
            return result;
        } catch (Exception e) {
            sample.stop(meterRegistry.timer("payment.processing.duration",
                "status", "error",
                "error_type", e.getClass().getSimpleName()));
            throw e;
        }
    }
}
```

**PromQL for timers**:
```promql
# p99 payment processing latency
histogram_quantile(0.99,
    sum(rate(payment_processing_duration_seconds_bucket[5m])) by (le)
)

# Average payment duration
rate(payment_processing_duration_seconds_sum[5m])
/ rate(payment_processing_duration_seconds_count[5m])

# SLO compliance: % of payments completing within 1 second
sum(rate(payment_processing_duration_seconds_bucket{le="1.0"}[5m]))
/ sum(rate(payment_processing_duration_seconds_count[5m])) * 100
```

---

## Example 5: Business Metrics with AOP

```java
// Custom annotation for metric tracking
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface Tracked {
    String value();                    // metric name
    String[] tags() default {};        // additional static tags
}

// AOP aspect: automatically records timer and counter for @Tracked methods
@Aspect
@Component
@RequiredArgsConstructor
@Slf4j
public class MetricsAspect {

    private final MeterRegistry meterRegistry;

    @Around("@annotation(tracked)")
    public Object trackMethod(ProceedingJoinPoint joinPoint, Tracked tracked) throws Throwable {
        String metricName = tracked.value();
        String[] tags = tracked.tags();
        
        Timer.Sample sample = Timer.start(meterRegistry);
        String status = "success";
        
        try {
            Object result = joinPoint.proceed();
            return result;
        } catch (Exception e) {
            status = "error";
            meterRegistry.counter(metricName + ".errors",
                "exception", e.getClass().getSimpleName())
                .increment();
            throw e;
        } finally {
            List<String> tagList = new ArrayList<>(Arrays.asList(tags));
            tagList.add("status");
            tagList.add(status);
            
            sample.stop(meterRegistry.timer(metricName + ".duration",
                tagList.toArray(new String[0])));
        }
    }
}

// Usage: clean business code with automatic metrics
@Service
public class OrderService {

    @Tracked(value = "order.create", tags = {"component", "order-service"})
    public Order createOrder(CreateOrderRequest request) {
        return processOrder(request);
    }
    
    @Tracked(value = "order.fulfillment", tags = {"component", "fulfillment"})
    public void fulfillOrder(Long orderId) {
        processFullfillment(orderId);
    }
}
```

---

## Example 6: Tagging Metrics Properly

```java
@Configuration
public class MetricsConfig {

    // Common tags applied to ALL metrics from this service
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags(
            @Value("${spring.application.name}") String appName,
            @Value("${app.version:unknown}") String version,
            @Value("${ENVIRONMENT:dev}") String environment) {
        
        return registry -> registry.config()
            .commonTags(
                "application", appName,
                "version", version,
                "environment", environment
            );
    }
    
    // Rename default metrics to match conventions
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> renameMetrics() {
        return registry -> registry.config()
            .meterFilter(MeterFilter.renameTag("http.server.requests",
                "exception", "error_type"));
    }
    
    // Deny high-cardinality metrics that could explode label count
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> denyHighCardinalityMetrics() {
        return registry -> registry.config()
            .meterFilter(new MeterFilter() {
                @Override
                public MeterFilterReply accept(Meter.Id id) {
                    // Deny any metric with a 'userId' tag (too high cardinality)
                    if (id.getTag("userId") != null) {
                        return MeterFilterReply.DENY;
                    }
                    return MeterFilterReply.NEUTRAL;
                }
            });
    }
    
    // Normalize URI patterns to prevent high cardinality from path variables
    @Bean
    public WebMvcTagsContributor uriNormalizationTagsContributor() {
        return (request, response, ex) -> {
            // Normalize: /orders/12345 → /orders/{id}
            String uri = (String) request.getAttribute(HandlerMapping.BEST_MATCHING_PATTERN_ATTRIBUTE);
            return Tags.of("uri.normalized", uri != null ? uri : "UNKNOWN");
        };
    }
}
```

---

# 8. Alerting

## Alertmanager Overview

Alertmanager is a **separate component** that receives alerts from Prometheus and handles:
- **Routing**: Direct different alerts to different teams/channels
- **Grouping**: Batch related alerts into single notifications
- **Inhibition**: Suppress certain alerts when others are firing (e.g., suppress service alerts when the entire host is down)
- **Silencing**: Temporarily mute alerts (during maintenance)
- **Deduplication**: Don't spam the same alert repeatedly

The flow:
```
Prometheus evaluates alert rule → fires alert to Alertmanager → Alertmanager routes → Slack/PagerDuty/Email
```

## Alert Rules

```yaml
# rules/alert_rules.yml
groups:
  - name: availability
    rules:

      # Service down: target unreachable for 1 minute
      - alert: ServiceDown
        expr: up == 0
        for: 1m          # Must be true for 1 minute before firing (reduces false positives)
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "Service {{ $labels.job }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."
          runbook_url: "https://runbooks.example.com/service-down"

      # High error rate: > 1% of requests are 5xx
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (application)
          / sum(rate(http_server_requests_seconds_count[5m])) by (application)
          > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on {{ $labels.application }}"
          description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.application }}"

      # Critical error rate: > 5%
      - alert: CriticalErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (application)
          / sum(rate(http_server_requests_seconds_count[5m])) by (application)
          > 0.05
        for: 2m
        labels:
          severity: critical
          pagerduty: "true"
        annotations:
          summary: "Critical error rate on {{ $labels.application }}"
          description: "Error rate is {{ $value | humanizePercentage }} — immediate action required"

  - name: latency
    rules:

      # High p99 latency: > 500ms
      - alert: HighRequestLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application, uri)
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High p99 latency for {{ $labels.application }} {{ $labels.uri }}"
          description: "p99 latency is {{ $value | humanizeDuration }} (threshold: 500ms)"

      # SLO breach: < 99.9% of requests completing within 1s
      - alert: SLOBreach
        expr: |
          sum(rate(http_server_requests_seconds_bucket{le="1.0"}[30m])) by (application)
          / sum(rate(http_server_requests_seconds_count[30m])) by (application)
          < 0.999
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "SLO breach: {{ $labels.application }} — < 99.9% requests within 1s"

  - name: resources
    rules:

      # High JVM heap usage
      - alert: HighJvmHeapUsage
        expr: |
          jvm_memory_used_bytes{area="heap"}
          / jvm_memory_max_bytes{area="heap"} > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High JVM heap usage on {{ $labels.application }}"
          description: "Heap is {{ $value | humanizePercentage }} full. GC pressure may increase."

      # DB connection pool saturation
      - alert: DatabaseConnectionPoolSaturation
        expr: |
          hikaricp_connections_active
          / hikaricp_connections_max > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "DB connection pool nearly exhausted on {{ $labels.application }}"
          description: "Pool is {{ $value | humanizePercentage }} utilized. New requests may fail."

      # Node disk almost full
      - alert: DiskAlmostFull
        expr: |
          (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Disk almost full on {{ $labels.instance }}"
          description: "Only {{ $value | humanizePercentage }} disk space remaining on {{ $labels.mountpoint }}"
```

## Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK'
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@example.com'
  smtp_auth_username: 'alerts@example.com'
  smtp_auth_password: 'your-password'

# Templates for notification messages
templates:
  - '/etc/alertmanager/templates/*.tmpl'

# Routing tree: determines where each alert goes
route:
  # Default: group alerts by alertname and application
  group_by: ['alertname', 'application']
  group_wait: 30s        # Wait 30s before sending first notification (allow grouping)
  group_interval: 5m     # Wait 5m before sending new notifications for same group
  repeat_interval: 4h    # Repeat notification every 4h if alert is still firing
  receiver: 'default-slack'  # Default receiver

  routes:
    # Critical alerts → PagerDuty + Slack
    - match:
        severity: critical
      receiver: 'pagerduty-and-slack'
      repeat_interval: 1h

    # Platform team alerts
    - match:
        team: platform
      receiver: 'platform-team-slack'

    # Database alerts → DBA team
    - match_re:
        alertname: 'Database.*'
      receiver: 'dba-team'

    # Silenced during maintenance window
    - match:
        maintenance: "true"
      receiver: 'null-receiver'  # Drop the alert

# Inhibition rules: suppress certain alerts when others are firing
inhibit_rules:
  # If a service is down, don't also alert about its high error rate
  - source_match:
      alertname: 'ServiceDown'
    target_match_re:
      alertname: 'HighErrorRate|HighRequestLatency'
    equal: ['application']  # Only suppress if 'application' label matches

# Receivers: notification channels
receivers:
  - name: 'default-slack'
    slack_configs:
      - channel: '#monitoring-alerts'
        title: '{{ template "slack.default.title" . }}'
        text: '{{ template "slack.default.text" . }}'
        send_resolved: true  # Notify when alert resolves

  - name: 'pagerduty-and-slack'
    pagerduty_configs:
      - routing_key: 'YOUR_PAGERDUTY_KEY'
        description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'
    slack_configs:
      - channel: '#incidents'
        title: ':fire: CRITICAL: {{ .GroupLabels.alertname }}'
        text: '{{ .CommonAnnotations.description }}'

  - name: 'platform-team-slack'
    slack_configs:
      - channel: '#platform-alerts'
        send_resolved: true

  - name: 'dba-team'
    email_configs:
      - to: 'dba-team@example.com'
        subject: 'DB Alert: {{ .GroupLabels.alertname }}'
        body: '{{ range .Alerts }}{{ .Annotations.description }}\n{{ end }}'

  - name: 'null-receiver'
    # No receivers: silently drops alerts
```

---

# 9. Visualization — Grafana Integration

## Why Grafana

Prometheus's built-in web UI is good for ad-hoc PromQL queries but lacks:
- Multi-panel dashboards
- Variables and template dashboards
- Annotations (mark deploys on graphs)
- Team sharing and permissions
- Alerting with richer visualization

Grafana is the standard companion to Prometheus.

## Grafana + Prometheus Setup

```yaml
# docker-compose.yml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:v2.47.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./rules:/etc/prometheus/rules
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'    # Allows hot-reload via POST /-/reload
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.2.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
      GF_USERS_ALLOW_SIGN_UP: false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

  alertmanager:
    image: prom/alertmanager:v0.26.0
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"

volumes:
  prometheus_data:
  grafana_data:
```

## Provisioning Grafana (as Code)

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST
      timeInterval: 15s  # must match prometheus scrape_interval
```

## Dashboard Best Practices

### The Four Golden Signals (Google SRE)

Every service should have these four panels at minimum:

1. **Latency**: Time to serve a request (p50, p95, p99)
2. **Traffic**: Number of requests per second
3. **Errors**: Rate of failed requests (5xx, exceptions)
4. **Saturation**: How "full" the service is (CPU, memory, connection pools)

```promql
# 1. Latency panels
# p50, p95, p99 request latency
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))
histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))
histogram_quantile(0.50, sum(rate(http_server_requests_seconds_bucket[5m])) by (le))

# 2. Traffic panel
sum(rate(http_server_requests_seconds_count[5m]))

# 3. Errors panel
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count[5m])) * 100

# 4. Saturation panels
# CPU
process_cpu_usage * 100
# Memory
jvm_memory_used_bytes{area="heap"} / jvm_memory_max_bytes{area="heap"} * 100
# Thread pool
hikaricp_connections_active / hikaricp_connections_max * 100
```

### Dashboard Design Principles

1. **USE Method** (for resources): Utilization, Saturation, Errors
2. **RED Method** (for services): Rate, Errors, Duration
3. **Start broad, drill down**: Top-level service health → per-endpoint → per-instance
4. **Time range consistency**: All panels on a dashboard should show the same time range
5. **Use variables**: Template dashboards with `$service`, `$namespace`, `$instance` variables for reuse
6. **Annotations**: Mark deployments on your graphs — invaluable for correlating changes with metric spikes

### Important Grafana Panels for Spring Boot

```promql
# Panel: Service Overview
# Request rate
sum(rate(http_server_requests_seconds_count{application="$application"}[5m]))

# Panel: Endpoint breakdown
# Request rate by endpoint
sum(rate(http_server_requests_seconds_count{application="$application"}[5m])) by (uri, method)

# Panel: JVM Health
# Heap usage trend
jvm_memory_used_bytes{application="$application", area="heap"}
/ jvm_memory_max_bytes{application="$application", area="heap"}

# GC pause time
rate(jvm_gc_pause_seconds_sum{application="$application"}[5m])

# Panel: Database
# Connection pool utilization
hikaricp_connections_active{application="$application"}
/ hikaricp_connections_max{application="$application"}

# Slow queries (if using custom metrics)
histogram_quantile(0.99, rate(db_query_duration_seconds_bucket{application="$application"}[5m]))
```

---

# 10. Production Best Practices

## Avoiding High Cardinality

High cardinality is the single most common cause of Prometheus performance problems.

### Rules for Labels

```
DO use:
  - service name (10-50 unique values)
  - HTTP method (5 values: GET, POST, PUT, DELETE, PATCH)
  - HTTP status code (20-30 values)
  - endpoint/route template (50-200 values: /api/orders, /api/orders/{id})
  - region (5-10 values)
  - environment (3-5 values: prod, staging, dev)
  - error type/exception class (50-100 values)
  - queue name (10-50 values)

DO NOT use:
  - user ID, customer ID, account ID
  - request ID, trace ID, correlation ID
  - email address
  - IP address (only if you have very few trusted IPs)
  - free-form text fields
  - product ID, order ID, item ID
  - any field with unbounded unique values
```

### Detecting Cardinality Issues

```promql
# Find metrics with high cardinality
topk(20, count by (__name__)({__name__=~".+"}))

# Find label values causing explosion
count by (uri)(http_server_requests_seconds_count)

# Total time series count
count({__name__=~".+"})
```

```bash
# Check Prometheus's own metrics
curl -s http://localhost:9090/metrics | grep prometheus_tsdb_head_series
# prometheus_tsdb_head_series 1234567
```

**Alert thresholds**:
- <1M series: healthy
- 1M–5M series: monitor closely, optimize labels
- 5M–10M series: performance impact likely, action needed
- 10M+: Prometheus will struggle, consider sharding

## Metric Naming Conventions

Follow the Prometheus naming conventions:

```
# Pattern: <namespace>_<subsystem>_<name>_<unit>

# Good examples:
http_server_requests_seconds_total     # namespace=http, subsystem=server, name=requests, unit=seconds
jvm_memory_used_bytes                  # namespace=jvm, name=memory_used, unit=bytes
database_query_duration_seconds        # namespace=database, name=query_duration, unit=seconds
orders_created_total                   # namespace=orders, name=created, total suffix for counters

# Rules:
# - snake_case only
# - Include unit suffix: _seconds, _bytes, _total, _ratio
# - Counters: _total suffix
# - No service name in metric (use labels for that)
# - Plural for things that are counted: requests_total, not request_total
```

## Retention Tuning

```bash
# Prometheus CLI flags
--storage.tsdb.retention.time=30d     # how long to keep data
--storage.tsdb.retention.size=100GB   # max disk usage
--storage.tsdb.min-block-duration=2h  # minimum block size (default, don't change)
--storage.tsdb.max-block-duration=36h # maximum block after compaction

# Storage estimation:
# ~1.37 bytes per sample (compressed)
# 10,000 time series × 4 samples/min × 1440 min/day × 30 days = 1.75 billion samples
# 1.75B samples × 1.37 bytes = ~2.4 GB for 30 days of 10k series
```

**For retention beyond 30 days**: Use remote write to Thanos, Cortex, or VictoriaMetrics. These are designed for long-term, multi-region metrics storage.

## Scaling Prometheus

### Problem: Single Prometheus Has Limits

A single Prometheus instance handles:
- ~1M active time series comfortably
- ~5M with careful tuning
- Beyond that: memory pressure, slow queries, scrape lag

### Solution 1: Sharding

Run multiple Prometheus instances, each scraping a subset of targets:

```yaml
# prometheus-shard-1.yml: scrapes services A-M
scrape_configs:
  - job_name: 'services'
    static_configs:
      - targets: ['service-a:8080', 'service-b:8080', ..., 'service-m:8080']

# prometheus-shard-2.yml: scrapes services N-Z
scrape_configs:
  - job_name: 'services'
    static_configs:
      - targets: ['service-n:8080', ..., 'service-z:8080']
```

### Solution 2: Federation

Prometheus can scrape other Prometheus instances. Lower-level Prometheus instances scrape targets; an aggregating instance scrapes the lower-level ones for high-level metrics.

```yaml
# federate-prometheus.yml: aggregating instance
scrape_configs:
  - job_name: 'federate'
    scrape_interval: 30s
    honor_labels: true
    metrics_path: '/federate'
    params:
      match[]:
        # Only pull pre-computed recording rules, not raw metrics
        - '{__name__=~"job:.*"}'
        - 'up'
    static_configs:
      - targets:
          - 'prometheus-shard-1:9090'
          - 'prometheus-shard-2:9090'
```

**Best practice**: Use recording rules in the sharded instances to pre-aggregate. The federated instance only pulls aggregated metrics.

### Solution 3: Thanos / VictoriaMetrics

For true horizontal scale:
- **Thanos**: Open-source, adds global querying, deduplication, long-term storage via object storage (S3/GCS)
- **VictoriaMetrics**: Drop-in replacement for Prometheus with better performance and cluster mode
- **Cortex / Grafana Mimir**: Multi-tenant, horizontally scalable Prometheus

## Debugging Issues

```bash
# 1. Check Prometheus status and active targets
curl http://localhost:9090/api/v1/targets

# 2. Check what's happening to a specific series
curl 'http://localhost:9090/api/v1/query?query=up{job="order-service"}'

# 3. Check TSDB stats (cardinality, memory usage)
curl http://localhost:9090/api/v1/status/tsdb

# 4. Check rule evaluation status
curl http://localhost:9090/api/v1/rules

# 5. Check active alerts
curl http://localhost:9090/api/v1/alerts

# 6. Reload configuration (without restart)
curl -X POST http://localhost:9090/-/reload

# 7. Check Prometheus's own performance metrics
prometheus_engine_query_duration_seconds  # how long queries take
prometheus_tsdb_head_series              # current active series
prometheus_rule_evaluation_duration_seconds  # rule evaluation time
prometheus_target_scrape_duration_seconds   # scrape duration
```

**Signs of trouble**:
- `prometheus_tsdb_head_series` > 5M: high cardinality
- `prometheus_rule_evaluation_duration_seconds` > 5s: rules too complex
- `prometheus_target_scrape_duration_seconds{quantile="0.9"}` close to `scrape_timeout`: targets are slow
- Memory > 8GB: likely cardinality issues

---

# 11. Prometheus in System Design

## Monitoring Microservices

In a microservices architecture, each service is independently deployable and monitored. The challenge: correlating metrics across services to understand system-wide health.

### RED Method Per Service

Every microservice should expose these three families of metrics:

```promql
# Rate: requests per second
sum(rate(http_server_requests_seconds_count{application="$service"}[5m]))

# Errors: 5xx error rate
sum(rate(http_server_requests_seconds_count{application="$service",status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count{application="$service"}[5m]))

# Duration: p99 latency
histogram_quantile(0.99,
    sum(rate(http_server_requests_seconds_bucket{application="$service"}[5m])) by (le)
)
```

### Service Dependency Mapping

Use client-side metrics to track calls to downstream services:

```java
// WebClient with Micrometer instrumentation (auto-configured in Spring Boot)
// Exposes: http_client_requests_seconds_bucket{uri="/api/payments", method="POST", ...}

// Manual instrumentation for specific downstream calls
Timer.Sample sample = Timer.start(meterRegistry);
try {
    result = paymentClient.charge(request);
    sample.stop(meterRegistry.timer("downstream.calls.duration",
        "service", "payment-service",
        "endpoint", "charge",
        "status", "success"));
} catch (Exception e) {
    sample.stop(meterRegistry.timer("downstream.calls.duration",
        "service", "payment-service",
        "endpoint", "charge",
        "status", "error",
        "error", e.getClass().getSimpleName()));
}
```

## Kubernetes Monitoring

Full Kubernetes monitoring requires:

| Component | What it provides |
|---|---|
| `kube-state-metrics` | Kubernetes object state: pod status, deployment replicas, node conditions |
| `node_exporter` | Host-level: CPU, memory, disk, network |
| `cAdvisor` (built into kubelet) | Container-level: CPU/memory per container |
| Prometheus with K8s SD | Scrapes all pods with annotations |
| `kube-apiserver` metrics | API server latency, request rate |

```promql
# Pods not running (crashloop, pending, etc.)
kube_pod_status_phase{phase!="Running"} == 1

# Deployment rollout stuck (desired != ready)
kube_deployment_status_replicas_ready != kube_deployment_spec_replicas

# Node memory pressure
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.1

# Container CPU throttling (high = CPU limit too low)
rate(container_cpu_cfs_throttled_seconds_total[5m])
/ rate(container_cpu_cfs_periods_total[5m]) > 0.25

# Pod restarts in the last hour
increase(kube_pod_container_status_restarts_total[1h]) > 0
```

## SLO/SLI Design

**SLI** (Service Level Indicator): A metric that measures a specific aspect of service reliability.
**SLO** (Service Level Objective): A target value for an SLI.
**SLA** (Service Level Agreement): A contract based on SLOs.

```promql
# SLI: Availability (% of successful requests)
# SLO: 99.9% availability
sum(rate(http_server_requests_seconds_count{status!~"5.."}[30d]))
/ sum(rate(http_server_requests_seconds_count[30d]))

# SLI: Latency (% of requests < 1s)
# SLO: 99% of requests complete within 1 second
sum(rate(http_server_requests_seconds_bucket{le="1.0"}[30d]))
/ sum(rate(http_server_requests_seconds_count[30d]))

# Error budget remaining
# (1 - error_rate) / (1 - SLO_target)
# If SLO = 99.9% availability:
(sum(rate(http_server_requests_seconds_count{status!~"5.."}[30d]))
/ sum(rate(http_server_requests_seconds_count[30d]))
- 0.999) / (1 - 0.999)
```

### Multi-Window, Multi-Burn-Rate Alerts (Google SRE Model)

```yaml
# Alert when burning through error budget too fast
# Fast burn (1h window): 14x burn rate for 1h = 1% budget gone in 1h
- alert: ErrorBudgetBurnRateFast
  expr: |
    (
      sum(rate(http_server_requests_seconds_count{status=~"5.."}[1h])) / 
      sum(rate(http_server_requests_seconds_count[1h]))
    ) > (14 * 0.001)  # 14× the 0.1% error budget rate
  for: 2m
  labels:
    severity: critical  # page immediately

# Slow burn (6h window): 6x burn rate for 6h
- alert: ErrorBudgetBurnRateSlow
  expr: |
    (
      sum(rate(http_server_requests_seconds_count{status=~"5.."}[6h])) / 
      sum(rate(http_server_requests_seconds_count[6h]))
    ) > (6 * 0.001)
  for: 30m
  labels:
    severity: warning   # ticket, not page
```

---

# 12. Real-World Scenarios

## Scenario 1: High-Load API Monitoring

### Context
E-commerce platform: 500 req/sec, 20 microservices, peak traffic during sales events.

### Architecture
```
Load Balancer
    │
    ├── Order Service (10 pods) ──── Prometheus ──── Grafana
    ├── Inventory Service (5 pods)        │
    ├── Payment Service (5 pods)     Alertmanager
    └── User Service (8 pods)             │
                                      PagerDuty + Slack
```

### Key Metrics

```promql
# Real-time request rate (dashboard: is traffic normal?)
sum(rate(http_server_requests_seconds_count[1m])) by (application)

# Error rate alert: fire if any service >2% error rate for 3 minutes
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (application)
/ sum(rate(http_server_requests_seconds_count[5m])) by (application) > 0.02

# Business metric: orders per minute (custom counter)
rate(orders_created_total[1m]) * 60

# Revenue rate: sum of order values per minute
rate(orders_total_value_dollars_sum[1m]) * 60

# Payment success rate
rate(payment_processing_duration_seconds_count{status="success"}[5m])
/ rate(payment_processing_duration_seconds_count[5m])
```

### Alerting Strategy
- **P1 (page immediately)**: Payment service down, checkout error rate >5%, complete outage
- **P2 (notify Slack, ticket)**: Single service high latency, DB pool saturation >90%, pod restarts
- **P3 (business hours)**: Slow degradation trends, SLO burn rate slow warnings

---

## Scenario 2: Kubernetes Cluster Monitoring

### Architecture

```
Prometheus (deployed via kube-prometheus-stack Helm chart)
    │
    ├── Scrapes: all pods with annotations
    ├── Scrapes: node_exporter (DaemonSet on every node)
    ├── Scrapes: kube-state-metrics
    ├── Scrapes: kube-apiserver, etcd, scheduler
    └── Scrapes: kubelet/cAdvisor
```

### Install via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set prometheus.prometheusSpec.retention=30d \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi \
  --set grafana.adminPassword=your-secure-password
```

### Critical K8s Alerts

```yaml
- alert: PodCrashLooping
  expr: |
    rate(kube_pod_container_status_restarts_total[15m]) > 0
  for: 5m
  annotations:
    summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"

- alert: NodeMemoryPressure
  expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes < 0.15
  for: 5m
  annotations:
    summary: "Node {{ $labels.instance }} memory < 15% available"

- alert: PersistentVolumeAlmostFull
  expr: |
    kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes < 0.10
  for: 5m
  annotations:
    summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} is 90%+ full"

- alert: HorizontalPodAutoscalerMaxReached
  expr: |
    kube_horizontalpodautoscaler_status_current_replicas
    == kube_horizontalpodautoscaler_spec_max_replicas
  for: 15m
  annotations:
    summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} at max replicas"
```

---

## Scenario 3: Distributed Microservices Observability

### Context
Financial services platform: 40 microservices, strict SLOs (99.95% availability), compliance requirements.

### Observability Stack

```
Metrics:   Prometheus + Thanos (global view, 1 year retention in S3)
Logs:      Loki + Grafana (structured JSON logs)
Traces:    Jaeger (distributed tracing, 1% sample rate)
Dashboards: Grafana (single pane of glass)
Alerts:    Alertmanager → PagerDuty → On-call rotation
```

### Service Mesh Metrics (Istio/Linkerd)

Service meshes automatically generate metrics for every service-to-service call:

```promql
# Istio: request success rate per destination service
sum(rate(istio_requests_total{destination_service=~"payment.*", response_code!~"5.."}[5m]))
/ sum(rate(istio_requests_total{destination_service=~"payment.*"}[5m]))

# Service-to-service latency
histogram_quantile(0.99,
    sum(rate(istio_request_duration_milliseconds_bucket{destination_service="payment.prod.svc"}[5m]))
    by (le, source_workload)
)
```

### Correlation IDs and Context Propagation

```java
// All metrics tagged with trace context for correlation with logs and traces
@Component
public class MetricsContextFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        HttpServletRequest request = (HttpServletRequest) req;
        String traceId = request.getHeader("X-Trace-Id");
        
        // Add trace context to MDC for log correlation
        MDC.put("traceId", traceId);
        
        // Custom gauge: track active requests per trace context (low cardinality!)
        // Note: DON'T use traceId as label — use request type
        meterRegistry.gauge("requests.active", 
            Tags.of("type", resolveRequestType(request)),
            activeRequests, AtomicInteger::get);
        
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("traceId");
        }
    }
}
```

---

# 13. Comparisons & Trade-offs

## Prometheus vs Traditional Monitoring (Nagios, Zabbix)

| Aspect | Traditional (Nagios/Zabbix) | Prometheus |
|---|---|---|
| Data model | Check-based (pass/fail) | Multi-dimensional metrics |
| Query language | Limited filters | Full PromQL |
| Alerting | Threshold-based on checks | Flexible PromQL expressions |
| Dynamic environments | Poor (manual config) | Excellent (service discovery) |
| Kubernetes native | No | Yes (designed for K8s) |
| Ecosystem | Plugins for everything | Rich exporter ecosystem |
| Setup complexity | High (agents everywhere) | Medium (pull-based, single binary) |
| Scalability | Limited | Moderate (use Thanos for scale) |

**Traditional monitoring** is still used for infrastructure monitoring in non-containerized environments. Prometheus dominates for cloud-native.

## Prometheus vs Push-Based Systems (Graphite, InfluxDB, StatsD)

| Aspect | Prometheus (Pull) | Push-Based (Graphite/StatsD) |
|---|---|---|
| Target discovery | Automatic (service discovery) | Must configure each agent |
| Target health | Directly observable (up/down) | Ambiguous: silence = down or just not pushing |
| Firewall requirements | Prometheus → targets | Targets → central server |
| Short-lived jobs | Awkward (use Pushgateway) | Natural (push before exit) |
| Config management | Centralized in Prometheus | Distributed across agents |
| Backpressure | Built-in (Prometheus controls rate) | None (agents push as fast as they can) |
| Data format | Text (human-readable) | Varies (usually binary/compact) |

## When Prometheus Is a Bad Choice

1. **Events, not metrics**: Tracking individual user actions, purchases, audit logs. Prometheus aggregates — you lose individual records. Use a database or log system.

2. **Very long retention (years)**: Prometheus local TSDB is not designed for multi-year retention. Use Thanos/Cortex with object storage.

3. **Multi-tenancy with strict isolation**: Prometheus has no built-in multi-tenant isolation. One tenant could query another's metrics. Use Cortex or Grafana Mimir for multi-tenancy.

4. **Extremely high cardinality time series**: If your use case inherently requires high-cardinality labels (e.g., per-user metrics with millions of users), Prometheus will struggle. Consider ClickHouse or a purpose-built analytics database.

5. **Push-only environments**: If your targets are behind strict firewalls with no inbound connections allowed, pull is impossible. Pushgateway partially addresses this but adds complexity and a single point of failure.

6. **Sub-second precision**: Prometheus scrapes every 15s by default. If you need millisecond-granularity time series, consider a dedicated time-series DB (InfluxDB, TimescaleDB).

7. **Already on a fully managed observability platform**: If your team uses Datadog, New Relic, or AWS CloudWatch and it meets your needs, adding Prometheus creates duplicate infrastructure without benefit.

---

# 14. Production-Ready Checklist

**You are production-ready with Prometheus if you can:**

## Fundamentals
- [ ] Explain the pull model and why Prometheus chose it over push
- [ ] Distinguish between the four metric types (Counter, Gauge, Histogram, Summary) and know when to use each
- [ ] Explain why you use `rate()` on a Counter rather than querying its raw value
- [ ] Explain the cardinality problem and why high-cardinality labels destroy Prometheus performance
- [ ] Describe how Prometheus's TSDB stores data (WAL, head block, compaction, blocks)

## PromQL
- [ ] Write a query for requests per second using `rate()`
- [ ] Calculate an error rate percentage (errors / total requests)
- [ ] Compute p99 latency using `histogram_quantile()`
- [ ] Aggregate across instances using `sum() by (label)`
- [ ] Explain why `sum(rate(...))` is correct and `rate(sum(...))` is wrong
- [ ] Use `topk()` to find the slowest endpoints
- [ ] Write a recording rule to pre-compute expensive queries
- [ ] Write an alert expression with a `for` duration to avoid flapping

## Spring Boot Integration
- [ ] Add Micrometer + Prometheus dependencies and expose `/actuator/prometheus`
- [ ] Configure common tags to apply to all metrics
- [ ] Create a custom Counter and increment it correctly
- [ ] Create a custom Gauge that tracks a live value
- [ ] Create a Timer (Histogram) with custom SLO buckets and measure method duration
- [ ] Configure HTTP endpoint metrics with normalized URI templates (prevent `{id}` cardinality explosion)
- [ ] Deny high-cardinality labels via `MeterFilter`

## Alerting
- [ ] Write alert rules for the four golden signals (latency, traffic, errors, saturation)
- [ ] Configure `for` duration appropriately to avoid alert flapping
- [ ] Configure Alertmanager routing (critical → PagerDuty, warning → Slack)
- [ ] Implement inhibition rules (suppress secondary alerts when primary fires)
- [ ] Write SLO burn rate alerts using multi-window approach

## Production Operations
- [ ] Detect high cardinality using `count by (__name__)({__name__=~".+"})`
- [ ] Identify and fix metrics with path variables creating cardinality explosion
- [ ] Configure appropriate retention (`--storage.tsdb.retention.time`)
- [ ] Set up remote_write to Thanos/VictoriaMetrics for long-term storage
- [ ] Monitor Prometheus itself: `prometheus_tsdb_head_series`, query duration, scrape lag
- [ ] Use `relabel_configs` in Kubernetes service discovery to filter and transform labels
- [ ] Hot-reload Prometheus configuration without restart via `POST /-/reload`

## Failure Scenarios — Know What Goes Wrong
- [ ] **Cardinality explosion**: New label with high unique values added → memory spikes → OOM. Fix: `MeterFilter.deny()`, normalize labels.
- [ ] **Counter resets**: Process restarts reset counters. `rate()` handles this automatically. Never alert on counter values directly.
- [ ] **Scrape failures**: Target down → `up == 0`. Always have a `ServiceDown` alert.
- [ ] **Slow rule evaluation**: Complex PromQL in alert rules takes >5s to evaluate → misses evaluation cycles. Fix: recording rules to pre-compute.
- [ ] **Alert flapping**: Alert fires and resolves rapidly. Fix: appropriate `for` duration (5m minimum for non-critical).
- [ ] **Prometheus TSDB full**: No `--storage.tsdb.retention.size` set + unlimited cardinality = disk fills up. Fix: set both time and size retention.
- [ ] **Missing `le` label in histogram aggregation**: `histogram_quantile` returns wrong results or NaN. Fix: always include `le` in `sum() by (le, ...)`.
- [ ] **Pushgateway stale metrics**: Job finishes, Pushgateway keeps serving last metrics forever. Fix: delete metrics from Pushgateway after job completes, or use timestamps.
- [ ] **Clock skew**: Prometheus and target clocks differ → samples ingested with wrong timestamps. Fix: NTP synchronization on all nodes.
- [ ] **Alertmanager split brain**: Two Alertmanager instances without clustering → duplicate notifications. Fix: configure Alertmanager clustering or use single instance with HA.

---

*Last reviewed: 2024. Prometheus 2.47.x, Grafana 10.x, Spring Boot 3.x.*
