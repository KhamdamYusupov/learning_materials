# Grafana: Complete Production Guide for Java/Spring Boot Engineers

> For backend developers building production-grade observability systems with real dashboards, alerts, and operational workflows.

---

## Table of Contents

1. [Observability & Visualization Fundamentals](#1-observability--visualization-fundamentals)
2. [Grafana Fundamentals](#2-grafana-fundamentals)
3. [Grafana Architecture & How It Works](#3-grafana-architecture--how-it-works)
4. [Data Sources](#4-data-sources)
5. [Dashboards & Panels — Complete Guide](#5-dashboards--panels--complete-guide)
6. [Queries in Grafana](#6-queries-in-grafana)
7. [Variables & Templating](#7-variables--templating)
8. [Alerts in Grafana](#8-alerts-in-grafana)
9. [Grafana + Prometheus Integration](#9-grafana--prometheus-integration)
10. [Production Best Practices](#10-production-best-practices)
11. [Grafana in System Design](#11-grafana-in-system-design)
12. [Real-World Scenarios](#12-real-world-scenarios)
13. [Comparisons & Trade-offs](#13-comparisons--trade-offs)
14. [Production-Ready Checklist](#14-production-ready-checklist)

---

# 1. Observability & Visualization Fundamentals

## The Role of Visualization in Observability

Raw metrics are numbers. Dashboards turn numbers into understanding.

Consider this metric: `http_server_requests_seconds_count{status="500"} = 847`. What does that tell you? Almost nothing without context:
- Is 847 errors high or normal?
- Is it trending up or down?
- Which service, endpoint, or time period?
- Does it correlate with a recent deploy?

Visualization answers these questions by:
1. **Showing trends over time** — is error count increasing or stable?
2. **Providing context** — compare against historical baseline
3. **Enabling correlation** — overlay deploy events with error spikes
4. **Communicating status** — instant red/green understanding for on-call engineers
5. **Supporting investigation** — drill down from "something is wrong" to "this specific endpoint on these pods"

Without visualization, monitoring is archaeology — you dig through raw data after the fact. With good dashboards, monitoring is reconnaissance — you see problems forming before they become outages.

## Metrics vs Logs vs Traces (Quick Recap)

| Pillar | What it answers | Tool | In Grafana |
|---|---|---|---|
| **Metrics** | "What is happening?" trends, rates, utilization | Prometheus, InfluxDB | Time series, stat, gauge panels |
| **Logs** | "What happened exactly?" per-event detail | Loki, Elasticsearch | Logs panel |
| **Traces** | "Where did the request go?" latency breakdown | Jaeger, Tempo | Trace panel |

Grafana's power is **correlating all three in one interface**. You see a latency spike in a metric → click through to logs for that time range → see the slow queries → link to a trace showing exactly which service caused it.

## Why Dashboards Matter in Production

### The On-Call Engineer Scenario

At 2 AM, an alert fires: "High error rate on order-service." Without a dashboard:
- Open terminal, run PromQL queries manually
- Check logs, search through noise
- 30 minutes to find the cause

With a well-designed dashboard:
- Click the alert link → lands on the service dashboard
- Immediately see: error rate spiking on `/api/orders/checkout`, started 8 minutes ago
- DB connection pool panel is red: 98% utilized
- Correlate with deploy annotation: deploy happened 10 minutes ago
- Root cause identified in under 2 minutes

Good dashboards encode institutional knowledge. Every panel answers a question an engineer has asked before.

## Common Monitoring Anti-Patterns

### Anti-Pattern 1: The Everything Dashboard
One dashboard with 50+ panels covering all services, all metrics. Nobody uses it because it's overwhelming. **Fix**: One focused dashboard per service, linked to sub-dashboards.

### Anti-Pattern 2: Vanity Metrics
Panels showing metrics that look good but don't drive decisions: total requests since launch, uptime since last restart. **Fix**: Show only actionable metrics — things you can and would act on.

### Anti-Pattern 3: Missing Baselines
Showing current CPU usage without historical context. Is 70% normal or alarming? **Fix**: Use time ranges that show historical comparison, overlay previous week.

### Anti-Pattern 4: Alert Fatigue Through Dashboards
Too many panels in warning/red state that nobody addresses because they're "always like that." **Fix**: Every red panel should be actionable. If it's always red, fix the threshold or the underlying issue.

### Anti-Pattern 5: Aggregate-Only Dashboards
Only showing total/average metrics. Average latency of 50ms looks fine when p99 is 5 seconds. **Fix**: Always show percentiles (p50, p95, p99), never just averages.

### Anti-Pattern 6: No Annotations
Graphs showing spikes with no indication of what changed. **Fix**: Mark deployments, configuration changes, incidents on time series graphs.

---

# 2. Grafana Fundamentals

## What Grafana Is

Grafana is an **open-source analytics and observability platform** for querying, visualizing, and alerting on metrics, logs, and traces from any data source.

Key insight: Grafana is **not a database**. It is a visualization and query layer. All data stays in your data sources (Prometheus, Loki, PostgreSQL, etc.). Grafana connects to them, sends queries, and renders results.

This matters because:
- Grafana going down doesn't affect your monitoring data — Prometheus keeps collecting
- You can query the same data with different tools
- No data migration when upgrading Grafana

## Core Features

### Dashboards
The primary unit of Grafana. A dashboard is a collection of panels arranged on a grid. Dashboards can be:
- Shared across teams
- Version-controlled as JSON
- Templated with variables (one dashboard for all services)
- Provisioned automatically via code

### Panels
The visual building blocks within a dashboard. Each panel runs a query against a data source and visualizes the result. Panel types: Time series, Stat, Gauge, Table, Bar chart, Pie chart, Heatmap, Logs, Traces, etc.

### Data Sources
Connections to external systems where data lives. Grafana ships with ~30 built-in data sources and hundreds via plugins. A single dashboard can query multiple data sources simultaneously.

### Alerts
Grafana Alerting (unified alerting since Grafana 8) evaluates queries against thresholds and routes notifications. It replaced the older per-panel alerting with a centralized system.

## When to Use Grafana vs Other Tools

| Scenario | Use Grafana | Use Alternative |
|---|---|---|
| Prometheus metrics visualization | ✓ | Prometheus UI (quick ad-hoc queries only) |
| Multi-source dashboard (metrics + logs) | ✓ | — |
| Kubernetes cluster monitoring | ✓ (kube-prometheus-stack) | Datadog, New Relic (managed) |
| Log analytics with full-text search | ✓ with Loki/Elasticsearch | Kibana (richer log analysis) |
| Business intelligence, SQL reporting | Partial | Metabase, Tableau |
| Already on AWS | ✓ self-hosted or Grafana Cloud | CloudWatch (if AWS-only) |
| Fully managed, no ops | Grafana Cloud | Datadog, New Relic |

---

# 3. Grafana Architecture & How It Works

## High-Level Architecture

```
Browser (Grafana UI)
        │
        │ HTTP/WebSocket
        ▼
┌─────────────────────────────────────────┐
│          Grafana Server (Go binary)      │
│                                         │
│  ┌──────────────┐  ┌──────────────────┐ │
│  │  HTTP API    │  │  Alert Engine    │ │
│  │  /api/*      │  │  (evaluates      │ │
│  └──────┬───────┘  │   alert rules)   │ │
│         │          └────────┬─────────┘ │
│  ┌──────▼───────────────────▼─────────┐ │
│  │         Data Source Manager        │ │
│  │  (proxies queries to data sources) │ │
│  └──────────────────┬─────────────────┘ │
│                     │                   │
│  ┌──────────────────▼─────────────────┐ │
│  │         Grafana Database (SQLite/   │ │
│  │         PostgreSQL/MySQL)           │ │
│  │  Stores: dashboards, users, alerts, │ │
│  │          data source configs        │ │
│  └────────────────────────────────────┘ │
└──────────────┬──────────────────────────┘
               │
     ┌─────────┼──────────────┐
     ▼         ▼              ▼
Prometheus   Loki         PostgreSQL
(metrics)    (logs)         (SQL data)
```

## Query Flow

When you load a Grafana dashboard panel:

```
1. Browser loads dashboard JSON from Grafana DB
2. Browser sends query request to Grafana backend API
   POST /api/ds/query
   { "queries": [{ "expr": "rate(http_requests_total[5m])", "datasourceId": 1 }] }

3. Grafana backend proxies query to the data source
   → Prometheus: GET /api/v1/query_range?query=rate(...)&start=...&end=...&step=...

4. Data source returns raw data (JSON)

5. Grafana backend transforms data into Grafana's internal data frame format

6. Browser receives data frames, renders visualization using React + D3.js
```

**Why Grafana proxies queries instead of the browser querying directly**:
- Security: data source credentials never reach the browser
- CORS: avoids cross-origin request issues
- Centralized caching and rate limiting

## Dashboard JSON Model

Every Grafana dashboard is a JSON document. Understanding this is critical for version control and automation.

```json
{
  "id": null,
  "uid": "order-service-dashboard",
  "title": "Order Service",
  "description": "RED metrics for the order service",
  "tags": ["order-service", "production"],
  "timezone": "browser",
  "refresh": "30s",
  "time": {
    "from": "now-1h",
    "to": "now"
  },
  "templating": {
    "list": [
      {
        "name": "environment",
        "type": "query",
        "datasource": "Prometheus",
        "query": "label_values(up, environment)",
        "current": { "value": "production" }
      }
    ]
  },
  "annotations": {
    "list": [
      {
        "name": "Deployments",
        "datasource": "Prometheus",
        "expr": "changes(process_start_time_seconds{application=\"order-service\"}[5m]) > 0",
        "titleFormat": "Deploy",
        "iconColor": "blue"
      }
    ]
  },
  "panels": [
    {
      "id": 1,
      "title": "Request Rate",
      "type": "timeseries",
      "gridPos": { "x": 0, "y": 0, "w": 12, "h": 8 },
      "datasource": "Prometheus",
      "targets": [
        {
          "expr": "sum(rate(http_server_requests_seconds_count{application=\"$application\"}[5m]))",
          "legendFormat": "req/s"
        }
      ]
    }
  ]
}
```

## Plugin System

Grafana's plugin system enables extensions without modifying core:

- **Data source plugins**: Connect to any system (AWS CloudWatch, Splunk, Oracle DB)
- **Panel plugins**: New visualization types (treemaps, network graphs, 3D charts)
- **App plugins**: Full applications within Grafana (Kubernetes app, OnCall, Incident)

Install plugins:
```bash
grafana-cli plugins install grafana-piechart-panel
grafana-cli plugins install yesoreyeram-boomtable-panel

# Via environment variable (Docker)
GF_INSTALL_PLUGINS=grafana-clock-panel,grafana-simple-json-datasource
```

---

# 4. Data Sources

## Prometheus (Most Important)

### What it is and when to use
Prometheus is the primary metrics data source for Grafana in cloud-native environments. Use it for all time-series metrics: request rates, latency, error rates, resource utilization.

### Configuration

```yaml
# grafana/provisioning/datasources/prometheus.yml
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy        # Grafana backend proxies requests (recommended)
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      httpMethod: POST   # POST is more reliable for long queries
      timeInterval: 15s  # Must match Prometheus scrape_interval
      queryTimeout: 60s
      exemplarTraceIdDestinations:
        - name: traceID
          datasourceUid: jaeger   # Link trace IDs in metrics to Jaeger
    version: 1
    editable: false
```

**`access: proxy` vs `access: direct`**:
- `proxy` (recommended): Grafana backend sends the query. Credentials are secure, CORS is handled.
- `direct`: Browser sends the query directly. Only use for public, unauthenticated endpoints or development.

### Query Examples in Grafana

```promql
# Request rate — basic
rate(http_server_requests_seconds_count[5m])

# Error rate with variable substitution
sum(rate(http_server_requests_seconds_count{application="$application", status=~"5.."}[$__rate_interval]))
/ sum(rate(http_server_requests_seconds_count{application="$application"}[$__rate_interval]))

# p99 latency
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket{application="$application"}[$__rate_interval])) by (le, uri)
)
```

**`$__rate_interval`**: Grafana's built-in variable. It automatically calculates the appropriate rate interval based on your dashboard's time range and the data source's scrape interval. Use this instead of hardcoded `[5m]` — it adapts as you zoom in/out on the time range.

**`$__interval`**: The auto-calculated step interval for the current time range and panel width. Use in functions like `avg_over_time(metric[$__interval])`.

---

## Loki (Logs)

### What it is and when to use
Loki is Grafana's log aggregation system (like Elasticsearch but cheaper). Use it for structured log querying when you need to see raw log lines, not aggregated metrics. Grafana's Logs panel is the primary visualization for Loki data.

### Configuration

```yaml
# grafana/provisioning/datasources/loki.yml
apiVersion: 1
datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      derivedFields:
        # Auto-link trace IDs in logs to Jaeger/Tempo
        - datasourceUid: jaeger
          matcherRegex: "traceId=(\\w+)"
          name: TraceID
          url: "$${__value.raw}"
```

### LogQL Query Examples

```logql
# All logs from order-service
{application="order-service"}

# Filter to errors only
{application="order-service"} |= "ERROR"

# Structured JSON logs: filter by field value
{application="order-service"} | json | level="ERROR"

# Parse and extract fields
{application="order-service"} | json | line_format "{{.level}}: {{.message}}"

# Log rate as metrics (count errors per minute)
sum(rate({application="order-service"} |= "ERROR" [1m])) by (pod)

# Correlation: find logs around a metric spike
# Set time range in Grafana to match when metric spiked, then query:
{application="order-service"} | json | statusCode >= 500
```

### Correlating Logs with Metrics

In Grafana, set up **data links** on metric panels to jump to Loki:

```json
{
  "links": [
    {
      "title": "View logs",
      "url": "/explore?left={\"datasource\":\"Loki\",\"queries\":[{\"expr\":\"{application=\\\"${__field.labels.application}\\\"}|=\\\"ERROR\\\"\"}],\"range\":{\"from\":\"${__from}\",\"to\":\"${__to}\"}}"
    }
  ]
}
```

---

## Elasticsearch

### What it is and when to use
Use Elasticsearch as a Grafana data source when:
- You already have an ELK stack
- You need full-text search in logs
- You have existing Elasticsearch indices for metrics or events

### Configuration

```yaml
apiVersion: 1
datasources:
  - name: Elasticsearch
    type: elasticsearch
    access: proxy
    url: http://elasticsearch:9200
    database: "[logstash-]YYYY.MM.DD"  # Index pattern
    jsonData:
      interval: Daily
      timeField: "@timestamp"
      esVersion: "8.0.0"
      logMessageField: message
      logLevelField: level
```

### Query Example in Grafana

Elasticsearch uses Lucene query syntax in the "Query" field:

```
# All errors
level:ERROR

# Errors for specific service
level:ERROR AND application:order-service

# Complex query
level:ERROR AND (application:order-service OR application:payment-service) AND NOT message:"expected"
```

For metrics-style panels with Elasticsearch, use the Metrics section to aggregate (count, avg, sum) and group by terms or date histogram.

---

## PostgreSQL

### What it is and when to use
Use PostgreSQL as a data source for:
- Business metrics stored in your application database (revenue, user signups, order counts)
- Custom reports that are too complex for Prometheus metrics
- Historical data that predates your monitoring stack
- SLO tracking stored in a database

### Configuration

```yaml
apiVersion: 1
datasources:
  - name: PostgreSQL
    type: postgres
    url: postgres:5432
    database: myapp_production
    user: grafana_readonly
    secureJsonData:
      password: "${POSTGRES_GRAFANA_PASSWORD}"
    jsonData:
      sslmode: require
      maxOpenConns: 5
      maxIdleConns: 2
      connMaxLifetime: 14400
      postgresVersion: 1500
      timescaledb: false
```

**Security critical**: Create a dedicated read-only user for Grafana. Never use your application's DB user.

```sql
-- Create read-only Grafana user
CREATE USER grafana_readonly WITH PASSWORD 'secure_password';
GRANT CONNECT ON DATABASE myapp_production TO grafana_readonly;
GRANT USAGE ON SCHEMA public TO grafana_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO grafana_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO grafana_readonly;
```

### Query Examples

```sql
-- Orders per hour (time series panel)
SELECT
  date_trunc('hour', created_at) AS time,
  COUNT(*) AS order_count,
  SUM(total_amount) AS revenue
FROM orders
WHERE created_at BETWEEN $__timeFrom() AND $__timeTo()
  AND status = 'completed'
GROUP BY date_trunc('hour', created_at)
ORDER BY time;

-- Revenue by product category (bar chart)
SELECT
  p.category,
  SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
JOIN products p ON oi.product_id = p.id
JOIN orders o ON oi.order_id = o.id
WHERE o.created_at BETWEEN $__timeFrom() AND $__timeTo()
GROUP BY p.category
ORDER BY revenue DESC;
```

**`$__timeFrom()` and `$__timeTo()`**: Grafana macros that inject the dashboard's current time range as SQL timestamps. This makes queries respect the dashboard time picker.

Other useful macros:
- `$__timeFilter(column)`: Generates `column BETWEEN ... AND ...` from time range
- `$__interval`: The auto-calculated time bucket size
- `$__unixEpochFrom()` / `$__unixEpochTo()`: Unix timestamps

---

# 5. Dashboards & Panels — Complete Guide

## Dashboard Structure

### Rows
Rows are collapsible horizontal sections that group related panels. Use them to organize a dashboard into logical sections.

```
Dashboard: Order Service
├── Row: Overview (collapsed by default for busy engineers)
│   ├── Stat: Current RPS
│   ├── Stat: Error Rate
│   └── Stat: p99 Latency
├── Row: RED Metrics (expanded by default)
│   ├── Time series: Request Rate
│   ├── Time series: Error Rate
│   └── Time series: Latency (p50/p95/p99)
├── Row: JVM Health
│   ├── Gauge: Heap Usage %
│   ├── Time series: GC Pause Duration
│   └── Time series: Thread Count
└── Row: Dependencies
    ├── Time series: DB Connection Pool
    └── Time series: Downstream Service Latency
```

### Grid Layout

Grafana uses a 24-column grid. Each panel has `x, y, w, h` (position and size in grid units).

Common panel sizing patterns:
- Full width: `w: 24, h: 8` — important time series
- Half width: `w: 12, h: 8` — two panels side by side
- Third width: `w: 8, h: 6` — three stat panels in a row
- Quarter width: `w: 6, h: 4` — compact stat display

### Dashboard-Level Settings

```
Time range: "Last 1 hour" (default — good for ops dashboards)
Refresh: 30s (auto-refresh for live monitoring)
Time zone: Browser (respects engineer's local time)
Tags: ["production", "order-service", "team-backend"]
```

---

## Variables (Templating)

Variables are the most powerful Grafana feature for reusable dashboards. Instead of a separate dashboard per service, one dashboard with a `$service` variable serves all services.

See Section 7 for the complete variable guide.

---

## Panel Types

### Time Series Panel

**What it is**: The primary panel for metrics over time. Shows one or more lines, bars, or areas plotted against a time axis.

**When to use**:
- Any metric that changes over time: request rate, latency, error rate, CPU usage
- Comparing multiple metrics on the same time axis
- Showing trends and correlating changes

**When NOT to use**:
- Single current values (use Stat panel)
- Proportional comparisons (use Pie chart)
- Ranking (use Bar chart)

**Key configuration options**:

```
Graph styles:
  - Lines (default): best for continuous metrics
  - Bars: good for discrete periods (requests per minute as discrete bars)
  - Area: good for stacked metrics (memory breakdown by area)

Axis:
  - Y-axis unit: CRITICAL — always set correct unit (ms, bytes, req/s, %)
  - Right Y-axis: when combining metrics with different scales (RPS + latency)

Legend:
  - Show values: avg, min, max, last — useful for quick reference
  - Position: bottom (space efficient) or right (wide dashboards)

Thresholds:
  - Add horizontal threshold lines at SLO boundaries
  - Example: add red line at p99 = 500ms

Overrides:
  - Color specific series: make "error rate" always red
  - Style specific series: dashed line for last week's baseline
```

**Example configuration (JSON)**:
```json
{
  "type": "timeseries",
  "title": "Request Latency",
  "fieldConfig": {
    "defaults": {
      "unit": "s",
      "custom": {
        "lineWidth": 2,
        "fillOpacity": 10
      },
      "thresholds": {
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 0.2 },
          { "color": "red", "value": 0.5 }
        ]
      }
    }
  },
  "options": {
    "legend": {
      "showLegend": true,
      "displayMode": "table",
      "calcs": ["lastNotNull", "max", "mean"]
    }
  }
}
```

---

### Stat Panel

**What it is**: Displays a single large number with optional sparkline and background color based on thresholds. Designed for instant status recognition.

**When to use**:
- Current value of a KPI: current error rate, uptime, active users
- Status indicators: is the service up/down?
- Key numbers at the top of a dashboard for quick overview

**When NOT to use**:
- When the trend over time matters (use Time series)
- When you need to compare across multiple series (use Table)

**Example configuration**:
```json
{
  "type": "stat",
  "title": "Error Rate",
  "options": {
    "reduceOptions": {
      "calcs": ["lastNotNull"]
    },
    "orientation": "horizontal",
    "colorMode": "background",
    "graphMode": "area"
  },
  "fieldConfig": {
    "defaults": {
      "unit": "percentunit",
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 0.01 },
          { "color": "red", "value": 0.05 }
        ]
      }
    }
  }
}
```

**Color modes**:
- `background`: entire panel background changes color — most visible
- `value`: only the number changes color
- `none`: no color changes

---

### Gauge Panel

**What it is**: A semicircular or circular gauge showing a value within a defined range. Good for utilization percentages.

**When to use**:
- Resource utilization percentages: heap usage %, CPU %, disk %, connection pool %
- Any value with a clear min (0) and max (100%) where being near the max is bad

**When NOT to use**:
- Values without a meaningful maximum (e.g., total request count)
- When you need historical context (use Time series)
- More than 4-5 gauges in a row (wastes space)

**Example configuration**:
```json
{
  "type": "gauge",
  "title": "JVM Heap Usage",
  "options": {
    "reduceOptions": {
      "calcs": ["lastNotNull"]
    },
    "showThresholdLabels": true,
    "showThresholdMarkers": true
  },
  "fieldConfig": {
    "defaults": {
      "unit": "percentunit",
      "min": 0,
      "max": 1,
      "thresholds": {
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 0.7 },
          { "color": "orange", "value": 0.85 },
          { "color": "red", "value": 0.95 }
        ]
      }
    }
  }
}
```

---

### Table Panel

**What it is**: Tabular display of data. Shows multiple series/values in rows and columns.

**When to use**:
- Comparing current values across multiple instances/services/endpoints
- "Top N slowest endpoints" view
- Showing multiple metrics for the same entity side by side
- Data that doesn't benefit from a time axis

**When NOT to use**:
- Time-series data (use Time series panel)
- Simple single-value displays (use Stat)

**Real-world example — Top 10 slowest endpoints**:

```promql
# Query A: p99 latency per endpoint
histogram_quantile(0.99,
  sum(rate(http_server_requests_seconds_bucket{application="$application"}[5m])) by (le, uri, method)
)

# Query B: request rate per endpoint
sum(rate(http_server_requests_seconds_count{application="$application"}[5m])) by (uri, method)

# Query C: error rate per endpoint
sum(rate(http_server_requests_seconds_count{application="$application", status=~"5.."}[5m])) by (uri, method)
/ sum(rate(http_server_requests_seconds_count{application="$application"}[5m])) by (uri, method)
```

Then in Transformations:
1. **Merge**: Combine queries A, B, C on `uri` and `method` labels
2. **Sort by**: p99 latency descending
3. **Limit**: Show top 10 rows

**Column configuration**: Set units per column (A=ms, B=req/s, C=percent), set thresholds, add color mapping.

---

### Heatmap Panel

**What it is**: Shows a two-dimensional density map. X-axis = time, Y-axis = value buckets, color intensity = count/frequency.

**When to use**:
- Latency distribution over time — shows how the distribution shifts (perfect for Prometheus histograms)
- Understanding request distribution patterns
- Identifying bi-modal distributions (two peaks = two distinct user behaviors)

**When NOT to use**:
- Simple single-value metrics (use Time series or Gauge)
- Non-numeric categorical data

**Example — Latency heatmap from Prometheus histogram**:

```promql
# Query
sum(rate(http_server_requests_seconds_bucket{application="$application"}[$__rate_interval])) by (le)
```

Panel settings:
- **Data format**: Time series buckets (for Prometheus histograms)
- **Y-axis**: `Bucket bounds` from the `le` label
- **Color scheme**: Scheme from green (low count) to red (high count)
- **Calculate**: from rate buckets

This shows you exactly how latency distributes across time — you can see if the p99 is stable or if there are periodic slow bursts.

---

# 6. Queries in Grafana

## Writing PromQL Queries in the Grafana UI

### Query Editor Modes

**Builder mode** (for beginners):
- Point-and-click metric selection
- Label filter dropdowns
- Auto-suggests available metrics and labels
- Good for exploration, not for complex queries

**Code mode** (for production):
- Direct PromQL text input
- Full syntax support
- Use this for all non-trivial queries

Switch between modes with the toggle in the top-right of the query editor.

### The Time Range and Step

Grafana automatically passes `start`, `end`, and `step` parameters to Prometheus based on the dashboard time range and panel width. Understanding this prevents common bugs:

```
Panel width: 800px
Time range: 1 hour (3600 seconds)
Step: 3600 / 800 = ~4.5s → rounded to Prometheus's evaluation interval
```

**Why `$__rate_interval` matters**: If you use `rate(metric[5m])` but your step is 30 seconds, you're calculating rates over a longer window than your display resolution. `$__rate_interval` adapts to the display, ensuring the rate window is always ≥ 4× the step.

```promql
# Use this (adapts to time range and panel width)
rate(http_requests_total[$__rate_interval])

# NOT this (static, may be wrong for zoomed-in/out views)
rate(http_requests_total[5m])
```

### Using Variables in Queries

Variables (prefixed with `$`) replace text in queries at render time:

```promql
# $application = "order-service" (from variable)
sum(rate(http_server_requests_seconds_count{application="$application"}[$__rate_interval]))

# Multiple selection variable ($namespace = "production|staging")
# Use =~ for regex matching when multi-value is enabled:
sum(rate(http_server_requests_seconds_count{namespace=~"$namespace"}[$__rate_interval]))

# All-value variable ($pod = ".*" when "All" selected)
sum(rate(http_server_requests_seconds_count{pod=~"$pod"}[$__rate_interval])) by (pod)
```

**Important**: When a variable allows multiple selections, use `=~` (regex match) not `=` (exact match). Grafana formats multi-value variables as `value1|value2|value3` for regex.

---

## Transformations

Transformations process query results in the browser before rendering. They're powerful for reshaping data without writing complex queries.

### Most Important Transformations

**Merge**: Combine results from multiple queries into one table.
```
Query A: p99 latency by service
Query B: request rate by service
→ Merge on "service" label → table with both columns
```

**Filter by value**: Remove rows below/above a threshold.
```
Show only endpoints where error rate > 0.01
```

**Sort by**: Order rows (useful for "top N" tables).
```
Sort by p99 latency descending
```

**Add field from calculation**: Create a new column computed from existing columns.
```
New column "Error %" = errors / total * 100
```

**Rename by regex**: Clean up messy label names.
```
Rename "http_server_requests_seconds_count" → "Request Count"
```

**Group by**: Aggregate across rows.
```
Group by "service", calculate sum of "requests"
```

**Labels to fields**: Convert Prometheus label values into table columns.
```
{ service="order", pod="order-1" } → columns: service, pod
```

**Example: Building a service health table**

```
Step 1: Query A — request rate
  sum(rate(http_requests_total[$__rate_interval])) by (application)
  Legend: {{application}}

Step 2: Query B — error rate
  sum(rate(http_requests_total{status=~"5.."}[$__rate_interval])) by (application)
  / sum(rate(http_requests_total[$__rate_interval])) by (application)
  Legend: {{application}}

Step 3: Query C — p99 latency
  histogram_quantile(0.99, sum(rate(http_request_duration_bucket[$__rate_interval])) by (le, application))
  Legend: {{application}}

Transformations:
1. Labels to fields (for each query)
2. Merge (join on "application")
3. Organize fields (rename, reorder)
4. Sort by: error rate, descending
```

---

## Combining Multiple Queries

Use multiple query letters (A, B, C...) in one panel for overlay and comparison:

```promql
# Query A: current week p99
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[$__rate_interval])) by (le))
# Legend: "p99 this week"

# Query B: last week p99 (for comparison)  
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[$__rate_interval] offset 1w)) by (le))
# Legend: "p99 last week"
# Style: dashed line, lower opacity
```

This creates a "week over week" comparison — invaluable for spotting regressions.

---

## Debugging Queries

### Step 1: Use Explore Mode
`Explore` (`Shift+X` shortcut) provides a free-form query interface without dashboard complexity. Use it to:
- Test queries before adding to dashboards
- Investigate incidents
- Check what labels are available

### Step 2: Check the Inspector
Every panel has an **Inspector** (`Inspect > Data`):
- Shows raw data returned by the query
- Shows the actual query sent to the data source
- Shows query execution time
- Use `Inspect > Query` to see the exact HTTP request to Prometheus

### Step 3: Check Prometheus Directly
```bash
# Run the same query in Prometheus UI or curl
curl 'http://prometheus:9090/api/v1/query_range' \
  --data-urlencode 'query=rate(http_requests_total[5m])' \
  --data-urlencode 'start=2024-01-15T10:00:00Z' \
  --data-urlencode 'end=2024-01-15T11:00:00Z' \
  --data-urlencode 'step=15s'
```

### Common Query Problems

| Problem | Symptom | Fix |
|---|---|---|
| No data | Empty panel | Check time range, check metric name, check labels |
| "No data points" | Error in panel | Metric may not exist, check with `{__name__=~"your_metric"}` |
| Wrong units | Values look huge or tiny | Set correct unit in field config (bytes, seconds, etc.) |
| Noisy legend | 50 lines, unreadable | Add `by (relevant_label)` or improve legend format |
| Inconsistent data | Gaps in time series | Scrape failures — check `up` metric for the target |
| NaN values | `histogram_quantile` returns NaN | Missing `le` label in aggregation |

---

# 7. Variables & Templating

## What Variables Are

Variables are dynamic inputs to dashboards. They appear as dropdowns at the top of the dashboard, and their selected values replace `$variable_name` placeholders in all queries on the dashboard.

**Why they're essential**: Without variables, you need one dashboard per service, per environment, per region. With variables, one dashboard works for all of them.

## Variable Types

### Query Variable

Dynamically populated from a data source query. The most common and most powerful type.

```json
{
  "name": "application",
  "type": "query",
  "datasource": "Prometheus",
  "query": "label_values(up{environment=\"$environment\"}, application)",
  "refresh": 2,          // 2 = refresh on time range change
  "includeAll": true,    // Add "All" option
  "multi": true,         // Allow multiple selection
  "sort": 1,             // Alphabetical ascending
  "regex": "",           // Optional filter regex
  "hide": 0              // 0=show, 1=label only, 2=hidden
}
```

**Common query patterns**:
```promql
# All values of a label
label_values(http_requests_total, application)

# Label values filtered by another variable
label_values(up{environment="$environment"}, application)

# All unique namespaces from kube-state-metrics
label_values(kube_namespace_labels, namespace)

# All pods for a selected deployment
label_values(kube_pod_info{namespace="$namespace"}, pod)
```

### Constant Variable

A fixed value that doesn't change. Used for thresholds or base URLs that you want configurable without editing JSON.

```json
{
  "name": "slo_threshold",
  "type": "constant",
  "query": "0.99",
  "hide": 2    // hidden — used in queries but not shown as dropdown
}
```

Usage:
```promql
# Use constant in alert expression
sum(rate(http_requests_total{status!~"5.."}[5m])) / sum(rate(http_requests_total[5m])) < $slo_threshold
```

### Custom Variable

A static list of options you define manually.

```json
{
  "name": "percentile",
  "type": "custom",
  "query": "p50 : 0.5,p95 : 0.95,p99 : 0.99",
  "current": { "value": "0.99", "text": "p99" }
}
```

Usage:
```promql
histogram_quantile($percentile, sum(rate(http_server_requests_seconds_bucket[$__rate_interval])) by (le))
```

This creates a dropdown where engineers can switch between p50/p95/p99 without changing the query.

### Interval Variable

For time grouping intervals in SQL or rate intervals.

```json
{
  "name": "interval",
  "type": "interval",
  "query": "1m,5m,15m,1h",
  "auto": true,
  "auto_count": 30
}
```

Usage:
```sql
SELECT date_trunc('$interval', created_at) AS time, count(*) FROM orders
GROUP BY time ORDER BY time
```

---

## Dynamic Dashboards — Real Example

**Goal**: One dashboard for all microservices, all environments, with pod-level drill-down.

### Variable Chain

```
1. $environment (Query)
   label_values(up, environment)
   → "production", "staging", "development"

2. $application (Query) — filtered by $environment
   label_values(up{environment="$environment"}, application)
   → "order-service", "payment-service", "user-service"

3. $pod (Query) — filtered by $application and $environment
   label_values(up{environment="$environment", application="$application"}, instance)
   → "order-service-7d8b4-x9z2m", "order-service-7d8b4-k3p1q"
   Multi-select + All option enabled
```

### Variable Usage in Queries

```promql
# Request rate for selected app in selected env
sum(rate(http_server_requests_seconds_count{
  application="$application",
  environment="$environment"
}[$__rate_interval]))

# Per-pod breakdown (uses $pod with multi-value regex matching)
sum(rate(http_server_requests_seconds_count{
  application="$application",
  environment="$environment",
  pod=~"$pod"
}[$__rate_interval])) by (pod)
```

### Chained Variable Dependency

When `$environment` changes:
1. `$application` variable re-runs its query filtered by new `$environment`
2. `$application` dropdown updates with new options
3. All panels using `$application` re-run their queries
4. `$pod` variable re-runs filtered by new `$application`

This creates a complete drill-down: select environment → select service → select specific pods.

---

## Repeat Panels and Rows

Variables enable panel and row repetition — automatically create one panel per variable value:

```json
{
  "type": "timeseries",
  "title": "$pod — Request Rate",
  "repeat": "pod",           // Repeat this panel for each selected pod
  "repeatDirection": "h"     // h=horizontal, v=vertical
}
```

When `$pod` has values `[pod-1, pod-2, pod-3]`, Grafana creates three panels side by side. Invaluable for multi-instance debugging.

---

# 8. Alerts in Grafana

## Grafana Alerting System Overview (Unified Alerting)

Since Grafana 8, the unified alerting system replaces the old per-panel alerts. Key components:

```
Alert Rules       → define what to check and when to fire
Contact Points    → where to send notifications (Slack, email, PagerDuty)
Notification Policies → routing: which alerts go to which contact points
Silences          → temporarily mute specific alerts
Alert Groups      → grouping of related firing alerts
```

**Grafana Alerting vs Prometheus Alertmanager**:
- Grafana Alerting: evaluates queries from any data source (not just Prometheus), visual alert rule editor, built into Grafana
- Prometheus Alertmanager: more flexible routing, better for Prometheus-only setups, standard in K8s

In production: many teams use **both**. Prometheus Alertmanager for infrastructure/platform alerts (always on, critical). Grafana Alerting for business/application alerts that need non-Prometheus data sources.

## Alert Rules

### Creating an Alert Rule

Alert rules are created in Grafana UI under `Alerting > Alert rules > New alert rule`.

**Rule components**:
1. **Query and conditions**: PromQL/LogQL query + threshold condition
2. **Evaluation behavior**: How often to evaluate, how long condition must be met
3. **Labels**: For routing and grouping
4. **Annotations**: Context for notification messages

### Example Alert Rules

**High Error Rate Alert**:
```
Name: High Error Rate
Query A:
  sum(rate(http_server_requests_seconds_count{application="$application",status=~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count{application="$application"}[5m]))

Condition: A IS ABOVE 0.05    (>5% error rate)

Evaluate every: 1m
For: 5m                        (must be above threshold for 5 continuous minutes)

Labels:
  severity: critical
  team: backend

Annotations:
  summary: High error rate on {{ $labels.application }}
  description: Error rate is {{ $values.A | humanizePercentage }} (threshold: 5%)
  runbook_url: https://wiki.example.com/runbooks/high-error-rate
```

**High Latency Alert**:
```
Name: High p99 Latency
Query A:
  histogram_quantile(0.99,
    sum(rate(http_server_requests_seconds_bucket{application="order-service"}[$__rate_interval]))
    by (le)
  )

Condition: A IS ABOVE 0.5     (>500ms p99)

Evaluate every: 1m
For: 10m

Labels:
  severity: warning
  team: backend
```

**Service Down Alert**:
```
Name: Service Down
Query A:
  up{job="order-service"}

Condition: A IS BELOW 1    (any instance is down)

Evaluate every: 30s
For: 1m

Labels:
  severity: critical
  pagerduty: true
```

**JVM Memory Alert**:
```
Name: High JVM Heap Usage
Query A:
  jvm_memory_used_bytes{area="heap", application="order-service"}
  / jvm_memory_max_bytes{area="heap", application="order-service"}

Condition: A IS ABOVE 0.9   (>90% heap)

Evaluate every: 1m
For: 5m

Labels:
  severity: warning
```

## Contact Points

```yaml
# Defined in Grafana UI or provisioning files

# Slack contact point
name: slack-backend-team
type: slack
settings:
  url: "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
  channel: "#backend-alerts"
  title: |
    {{ if eq .Status "firing" }}🔴{{ else }}✅{{ end }} {{ .GroupLabels.alertname }}
  text: |
    {{ range .Alerts }}
    *Summary:* {{ .Annotations.summary }}
    *Description:* {{ .Annotations.description }}
    *Severity:* {{ .Labels.severity }}
    {{ if .Annotations.runbook_url }}*Runbook:* {{ .Annotations.runbook_url }}{{ end }}
    {{ end }}

# PagerDuty contact point
name: pagerduty-critical
type: pagerduty
settings:
  integrationKey: "YOUR_PAGERDUTY_INTEGRATION_KEY"
  severity: critical

# Email contact point
name: email-oncall
type: email
settings:
  addresses: oncall@example.com
  subject: "{{ .GroupLabels.alertname }} - {{ .Status | title }}"
```

## Notification Policies

Notification policies define a routing tree — which alerts go to which contact points.

```yaml
# Default policy (catch-all)
receiver: slack-backend-team
group_by: [alertname, application]
group_wait: 30s
group_interval: 5m
repeat_interval: 4h

# Override policies (more specific first)
routes:
  # Critical alerts → PagerDuty
  - matchers:
      - severity = critical
    receiver: pagerduty-critical
    repeat_interval: 1h

  # Platform team alerts
  - matchers:
      - team = platform
    receiver: slack-platform-team

  # Business hours only for warnings
  - matchers:
      - severity = warning
    receiver: slack-backend-team
    mute_time_intervals:
      - non-business-hours
```

## Silences

Use silences to suppress alerts during planned maintenance:

```yaml
# Silence all alerts for order-service during maintenance window
matchers:
  - application = order-service
startsAt: 2024-01-20T02:00:00Z
endsAt: 2024-01-20T04:00:00Z
comment: "Database maintenance window"
createdBy: ops-team
```

---

# 9. Grafana + Prometheus Integration

## Connecting Prometheus to Grafana

### Via Docker Compose

```yaml
version: '3.8'
services:
  prometheus:
    image: prom/prometheus:v2.47.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    ports:
      - "9090:9090"

  grafana:
    image: grafana/grafana:10.2.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_INSTALL_PLUGINS: grafana-piechart-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

volumes:
  prometheus_data:
  grafana_data:
```

### Provisioning Data Source

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
      timeInterval: 15s
    version: 1
    editable: false
```

### Provisioning Dashboards

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1
providers:
  - name: default
    orgId: 1
    type: file
    disableDeletion: false       # Allow deleting via UI
    updateIntervalSeconds: 30   # Check for file changes every 30s
    allowUiUpdates: true        # Allow editing via UI (changes won't persist to file)
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: true   # Subfolders become Grafana folders
```

Place dashboard JSON files in `/var/lib/grafana/dashboards/`. Grafana loads them automatically on start and when files change.

---

## Building a Complete Spring Boot API Dashboard

### Dashboard Overview

A production dashboard for a Spring Boot service should have these panels in order:

```
Row 1: Service Status (overview)
├── Stat: Uptime
├── Stat: Request Rate (current)
├── Stat: Error Rate (current, colored by threshold)
└── Stat: p99 Latency (current, colored by threshold)

Row 2: Traffic & Errors
├── Time series: Request Rate (by endpoint)
└── Time series: Error Rate (by status code)

Row 3: Latency
├── Time series: p50/p95/p99 latency
└── Heatmap: Latency distribution

Row 4: JVM & Resources
├── Time series: Heap usage + GC activity
├── Time series: Thread count
└── Gauge: Heap utilization %

Row 5: Database
├── Gauge: DB connection pool utilization
├── Time series: Active + idle + pending connections
└── Time series: Query duration (if custom metrics)

Row 6: Business Metrics
├── Stat: Orders created (last hour)
└── Time series: Orders per minute
```

### Complete Dashboard JSON

```json
{
  "uid": "spring-boot-service",
  "title": "Spring Boot Service — $application",
  "tags": ["spring-boot", "production"],
  "refresh": "30s",
  "time": { "from": "now-1h", "to": "now" },
  "templating": {
    "list": [
      {
        "name": "datasource",
        "type": "datasource",
        "pluginId": "prometheus",
        "hide": 0
      },
      {
        "name": "application",
        "type": "query",
        "datasource": "${datasource}",
        "query": "label_values(up, application)",
        "refresh": 2,
        "sort": 1,
        "hide": 0
      },
      {
        "name": "instance",
        "type": "query",
        "datasource": "${datasource}",
        "query": "label_values(up{application=\"$application\"}, instance)",
        "refresh": 2,
        "includeAll": true,
        "multi": true,
        "hide": 0
      }
    ]
  },
  "annotations": {
    "list": [
      {
        "name": "Deployments",
        "datasource": "${datasource}",
        "expr": "changes(process_start_time_seconds{application=\"$application\"}[2m]) > 0",
        "step": "60s",
        "titleFormat": "Deploy",
        "textFormat": "{{instance}}",
        "iconColor": "blue",
        "enable": true
      }
    ]
  },
  "panels": [
    {
      "id": 1,
      "title": "Request Rate",
      "type": "stat",
      "gridPos": { "x": 0, "y": 0, "w": 6, "h": 4 },
      "targets": [
        {
          "datasource": "${datasource}",
          "expr": "sum(rate(http_server_requests_seconds_count{application=\"$application\"}[$__rate_interval]))",
          "legendFormat": "req/s"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "color": { "mode": "thresholds" },
          "thresholds": {
            "steps": [
              { "color": "green", "value": null }
            ]
          }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background",
        "graphMode": "area"
      }
    },
    {
      "id": 2,
      "title": "Error Rate",
      "type": "stat",
      "gridPos": { "x": 6, "y": 0, "w": 6, "h": 4 },
      "targets": [
        {
          "datasource": "${datasource}",
          "expr": "sum(rate(http_server_requests_seconds_count{application=\"$application\",status=~\"5..\"}[$__rate_interval])) / sum(rate(http_server_requests_seconds_count{application=\"$application\"}[$__rate_interval]))",
          "legendFormat": "error rate"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "color": { "mode": "thresholds" },
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 0.01 },
              { "color": "red", "value": 0.05 }
            ]
          }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background"
      }
    },
    {
      "id": 3,
      "title": "p99 Latency",
      "type": "stat",
      "gridPos": { "x": 12, "y": 0, "w": 6, "h": 4 },
      "targets": [
        {
          "datasource": "${datasource}",
          "expr": "histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{application=\"$application\"}[$__rate_interval])) by (le))",
          "legendFormat": "p99"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "s",
          "color": { "mode": "thresholds" },
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 0.2 },
              { "color": "red", "value": 0.5 }
            ]
          }
        }
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "colorMode": "background"
      }
    },
    {
      "id": 4,
      "title": "JVM Heap",
      "type": "stat",
      "gridPos": { "x": 18, "y": 0, "w": 6, "h": 4 },
      "targets": [
        {
          "datasource": "${datasource}",
          "expr": "max(jvm_memory_used_bytes{application=\"$application\",area=\"heap\"} / jvm_memory_max_bytes{application=\"$application\",area=\"heap\"})",
          "legendFormat": "heap"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "percentunit",
          "color": { "mode": "thresholds" },
          "thresholds": {
            "steps": [
              { "color": "green", "value": null },
              { "color": "yellow", "value": 0.7 },
              { "color": "red", "value": 0.9 }
            ]
          }
        }
      }
    },
    {
      "id": 5,
      "title": "Request Rate by Endpoint",
      "type": "timeseries",
      "gridPos": { "x": 0, "y": 4, "w": 12, "h": 8 },
      "targets": [
        {
          "datasource": "${datasource}",
          "expr": "sum(rate(http_server_requests_seconds_count{application=\"$application\"}[$__rate_interval])) by (uri, method)",
          "legendFormat": "{{method}} {{uri}}"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "reqps" }
      }
    },
    {
      "id": 6,
      "title": "Error Rate by Status",
      "type": "timeseries",
      "gridPos": { "x": 12, "y": 4, "w": 12, "h": 8 },
      "targets": [
        {
          "datasource": "${datasource}",
          "expr": "sum(rate(http_server_requests_seconds_count{application=\"$application\",status=~\"5..\"}[$__rate_interval])) by (status, uri)",
          "legendFormat": "{{status}} {{uri}}"
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "reqps",
          "color": { "mode": "fixed", "fixedColor": "red" }
        }
      }
    },
    {
      "id": 7,
      "title": "Request Latency Percentiles",
      "type": "timeseries",
      "gridPos": { "x": 0, "y": 12, "w": 12, "h": 8 },
      "targets": [
        {
          "datasource": "${datasource}",
          "expr": "histogram_quantile(0.50, sum(rate(http_server_requests_seconds_bucket{application=\"$application\"}[$__rate_interval])) by (le))",
          "legendFormat": "p50"
        },
        {
          "datasource": "${datasource}",
          "expr": "histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{application=\"$application\"}[$__rate_interval])) by (le))",
          "legendFormat": "p95"
        },
        {
          "datasource": "${datasource}",
          "expr": "histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{application=\"$application\"}[$__rate_interval])) by (le))",
          "legendFormat": "p99"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "s" }
      }
    },
    {
      "id": 8,
      "title": "DB Connection Pool",
      "type": "timeseries",
      "gridPos": { "x": 12, "y": 12, "w": 12, "h": 8 },
      "targets": [
        {
          "datasource": "${datasource}",
          "expr": "hikaricp_connections_active{application=\"$application\"}",
          "legendFormat": "active - {{pool}}"
        },
        {
          "datasource": "${datasource}",
          "expr": "hikaricp_connections_idle{application=\"$application\"}",
          "legendFormat": "idle - {{pool}}"
        },
        {
          "datasource": "${datasource}",
          "expr": "hikaricp_connections_pending{application=\"$application\"}",
          "legendFormat": "pending - {{pool}}"
        },
        {
          "datasource": "${datasource}",
          "expr": "hikaricp_connections_max{application=\"$application\"}",
          "legendFormat": "max - {{pool}}"
        }
      ],
      "fieldConfig": {
        "defaults": { "unit": "short" }
      }
    }
  ]
}
```

---

# 10. Production Best Practices

## Dashboard Design Principles

### Principle 1: Answer a Specific Question
Every dashboard should answer one question: "Is this service healthy?" or "What is the Kubernetes cluster status?" Not both on the same dashboard.

```
Good: "Order Service — Production Health"
Bad:  "All Services All Environments"
```

### Principle 2: Information Hierarchy
Most important information at the top-left. Operators should get the answer to "is everything OK?" in the first 3 seconds of looking at the dashboard.

```
Top row: Status overview (Stat panels — red/green at a glance)
Second row: Key metrics over time (Time series — trend visibility)
Lower rows: Detailed breakdown (per-endpoint, per-instance drill-down)
```

### Principle 3: Consistent Scales
All latency panels use the same Y-axis range. All utilization panels show 0–100%. Don't auto-scale Y-axes on operational dashboards — a line near the top should always mean "near capacity."

```
BAD: Auto-scaled Y-axis where 50ms looks the same as 5000ms
GOOD: Fixed Y-axis 0–2000ms, threshold line at 500ms
```

### Principle 4: Meaningful Colors
```
Red    = bad, action required
Yellow = warning, watch closely
Green  = healthy
Blue   = informational, neutral
```

Apply these consistently. Don't use red for "high traffic" — engineers will panic. Use neutral/blue for volume metrics, reserve red for actual problems.

### Principle 5: Annotations Over Stories
Mark deployments, incidents, and configuration changes on time series panels. When an engineer sees a spike, the first question is "what changed?" An annotation answers that immediately.

## Avoiding Noisy Dashboards

### Problem: Too Many Panels
A dashboard with 40+ panels is exhausting to scan. Nobody reads it regularly, so nobody maintains it.

**Fix**: Apply the "would I look at this during an incident?" test. If no, remove it or move it to a separate troubleshooting dashboard.

### Problem: Duplicate Information
Showing both request count AND request rate (when you only need rate), or showing average AND median latency.

**Fix**: One metric per concept. Request rate tells you everything count tells you and more.

### Problem: Low-Signal Panels
A panel that never shows problems (always green, always stable). It takes up space and trains engineers to ignore dashboards.

**Fix**: If it never alerts and never changes, it doesn't belong on the primary operational dashboard.

### Problem: Wrong Time Resolution
A 1-hour time range with 1-second resolution shows 3600 data points per series — too much to understand. A 30-day view with 1-second resolution misses the trend.

**Fix**: Match default time range to the operational context:
- Incident response: default to `last 1h` or `last 3h`
- Capacity planning: default to `last 30d`
- Weekly review: default to `last 7d`

## Naming Conventions

```
Dashboard names:
  Format: "[Service] — [Context]"
  Examples:
    "Order Service — Production Overview"
    "Kubernetes — Cluster Health"
    "PostgreSQL — Query Performance"
  
  BAD:
    "Dashboard 1"
    "Order Stuff"
    "k8s"

Panel titles:
  - Start with what it measures, not how: "Request Rate" not "sum(rate(...))"
  - Include units when not obvious: "Latency (ms)" 
  - Use consistent terminology across all dashboards

Folder structure:
  /Production Services/
    Order Service
    Payment Service
    User Service
  /Infrastructure/
    Kubernetes Cluster
    PostgreSQL
    Redis
  /SLO Dashboards/
    Order Service SLO
  /Runbooks/
    High Error Rate Investigation
```

## Version Control for Dashboards

### Export and Store as JSON

```bash
# Export dashboard via Grafana API
curl -s \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  "http://grafana:3000/api/dashboards/uid/order-service-overview" \
  | jq '.dashboard' \
  > dashboards/order-service-overview.json

# Import dashboard via API
curl -s -X POST \
  -H "Authorization: Bearer $GRAFANA_API_KEY" \
  -H "Content-Type: application/json" \
  "http://grafana:3000/api/dashboards/import" \
  -d "{\"dashboard\": $(cat dashboards/order-service-overview.json), \"overwrite\": true, \"folderId\": 1}"
```

### Automated Dashboard Provisioning

Store dashboard JSON files in Git. Deploy via:

```yaml
# Kubernetes ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  order-service.json: |
    { "uid": "order-service", "title": "Order Service", ... }
---
# Mount in Grafana deployment
volumes:
  - name: dashboards
    configMap:
      name: grafana-dashboards
volumeMounts:
  - name: dashboards
    mountPath: /var/lib/grafana/dashboards
```

### Version Control Workflow

```bash
# 1. Make changes in Grafana UI
# 2. Export updated dashboard
./scripts/export-dashboard.sh order-service-overview

# 3. Review diff
git diff dashboards/order-service-overview.json

# 4. Commit with context
git add dashboards/order-service-overview.json
git commit -m "Dashboard: Add DB connection pool panel to order-service

Added HikariCP connection pool utilization panels (active/idle/pending/max)
to support debugging the DB exhaustion incident from 2024-01-15."

# 5. CI/CD auto-deploys to staging, then production
```

## Security

### Authentication

```ini
# grafana.ini
[auth]
disable_login_form = false

# OIDC/OAuth integration (Keycloak example)
[auth.generic_oauth]
enabled = true
name = Keycloak
allow_sign_up = true
client_id = grafana
client_secret = ${KEYCLOAK_CLIENT_SECRET}
scopes = openid email profile
auth_url = https://keycloak.example.com/realms/myapp/protocol/openid-connect/auth
token_url = https://keycloak.example.com/realms/myapp/protocol/openid-connect/token
api_url = https://keycloak.example.com/realms/myapp/protocol/openid-connect/userinfo
role_attribute_path = contains(roles[*], 'grafana-admin') && 'Admin' || contains(roles[*], 'grafana-editor') && 'Editor' || 'Viewer'
```

### Roles

| Role | Can do |
|---|---|
| Viewer | View dashboards, cannot edit |
| Editor | Create/edit dashboards, create alerts |
| Admin | Full access, manage users, data sources |

**Best practice**: Most engineers = Viewer. On-call + senior engineers = Editor. Only SRE/ops leads = Admin.

### API Keys for Automation

```bash
# Create service account for CI/CD
curl -s -X POST \
  -H "Content-Type: application/json" \
  -u admin:password \
  "http://grafana:3000/api/serviceaccounts" \
  -d '{"name": "ci-deploy", "role": "Editor"}'

# Create token for the service account
curl -s -X POST \
  -H "Content-Type: application/json" \
  -u admin:password \
  "http://grafana:3000/api/serviceaccounts/1/tokens" \
  -d '{"name": "deploy-token"}'
```

## Performance Optimization

### Dashboard Load Time

```
Problem: Dashboard with 20 panels loading slowly

Diagnosis:
1. Open Network tab in browser DevTools
2. Look for slow /api/ds/query requests
3. Identify which panels are slow

Fixes:
- Use recording rules in Prometheus to pre-compute expensive queries
- Reduce time range resolution (increase $__step multiplier)
- Limit panel count: merge related panels using multiple queries
- Use caching: Prometheus query caching, Grafana query caching
```

### Reduce Query Load on Prometheus

```yaml
# Configure query caching in Grafana
[caching]
enabled = true
backend = memory      # or redis for distributed deployments
ttl = 30s            # Cache queries for 30 seconds

# Or in data source settings
jsonData:
  cacheLevel: medium   # none, low, medium, high
```

### Avoid Query Explosion with Variables

When a variable has "All" selected and repeats a panel query for each value:
- 50 services × 3 queries per panel = 150 simultaneous Prometheus queries
- This can overwhelm Prometheus

**Fixes**:
- Use `sum(...) by (service)` in a single query instead of per-service repetition
- Limit "All" option: `regex: /^production-.*/` to filter to relevant values
- Use server-side query batching (enterprise feature)

---

# 11. Grafana in System Design

## Monitoring Microservices

In a microservices architecture, you need a layered dashboard strategy:

### Layer 1: System-Wide Overview Dashboard
```
"All Services Health"
├── Row: Traffic (total RPS, global error rate)
├── Row: Service Health Matrix (table: each service × error rate × latency × uptime)
└── Row: Infrastructure (total pod count, node health)
```

This is the first screen an on-call engineer opens. It answers: "Is anything on fire?"

### Layer 2: Service Dashboard (per-service)
```
"Order Service — Production"
├── Row: Status (current RPS, error rate, p99 latency as Stat panels)
├── Row: Traffic & Errors (time series)
├── Row: Latency distribution (heatmap + percentiles)
├── Row: JVM health
└── Row: Dependencies (DB pool, downstream service latency)
```

### Layer 3: Troubleshooting Dashboards
```
"Order Service — Debug"
├── Row: Per-endpoint breakdown (table + per-endpoint time series)
├── Row: Per-pod breakdown (repeated panels per pod)
├── Row: JVM deep dive (GC details, memory areas, class loading)
└── Row: Correlation (logs panel side-by-side with metrics)
```

### Service Health Matrix Example

One table panel showing all services:

```promql
# Query A: error rates per service
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m])) by (application)
/ sum(rate(http_server_requests_seconds_count[5m])) by (application)

# Query B: p99 latency per service
histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application))

# Query C: uptime (is any instance down?)
min(up) by (application)
```

With transformations: Merge on `application`, sort by error rate. Thresholds color each cell red/yellow/green.

## Kubernetes Dashboards

### kube-prometheus-stack Helm Chart

The fastest path to production Kubernetes monitoring:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=your-secure-password \
  --set prometheus.prometheusSpec.retention=30d
```

This deploys:
- Prometheus + Alertmanager + node_exporter (DaemonSet) + kube-state-metrics
- Grafana with pre-built dashboards for:
  - Kubernetes cluster overview
  - Node resource utilization
  - Namespace resource utilization
  - Pod resource utilization
  - Kubernetes persistent volumes
  - CoreDNS, etcd, API server

### Critical K8s Panels to Build

```promql
# Panel: Pod restart rate (should be near 0)
sum(rate(kube_pod_container_status_restarts_total[15m])) by (namespace, pod)

# Panel: Node memory pressure
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

# Panel: Deployment replica mismatch (desired vs ready)
kube_deployment_spec_replicas - kube_deployment_status_replicas_ready

# Panel: Pod CPU throttling (high = CPU limits too restrictive)
sum(rate(container_cpu_cfs_throttled_seconds_total[5m])) by (pod, namespace)
/ sum(rate(container_cpu_cfs_periods_total[5m])) by (pod, namespace)

# Panel: Persistent volume almost full
kubelet_volume_stats_available_bytes / kubelet_volume_stats_capacity_bytes
```

## SRE Workflows

### Incident Response Workflow

```
1. Alert fires → Engineer gets paged
2. Click alert link → Opens service dashboard for affected service
3. Stat panels at top → Instant status: "error rate 8%, p99 2.3s"
4. Time series panels → When did it start? (correlate with deploy annotation)
5. Error breakdown panel → Which endpoints are failing?
6. Logs link from panel → Drill into logs for failing endpoint
7. Trace link → Find a specific slow request trace
8. Resolution: revert deploy or fix config
9. Monitor: error rate drops on dashboard, alert resolves
```

Good dashboard design compresses steps 2–6 from 30 minutes to 2 minutes.

## SLO/SLI Dashboards

```promql
# SLI: availability over rolling 30-day window
sum_over_time(
  (sum(rate(http_server_requests_seconds_count{status!~"5.."}[5m])) 
   / sum(rate(http_server_requests_seconds_count[5m])))[30d:5m]
) / count_over_time(
  (sum(rate(http_server_requests_seconds_count[5m])))[30d:5m]
)

# Error budget remaining (SLO = 99.9% availability)
# Error budget = 1 - SLO = 0.1% = 43.8 minutes per 30 days
# Budget consumed = (1 - actual_availability) / (1 - SLO_target) * 100
(1 - <availability_query>) / (1 - 0.999) * 100
```

**SLO Dashboard structure**:
```
Row: Current Status
├── Stat: 30-day availability (colored: ≥99.9% green, else red)
├── Stat: Error budget remaining % 
└── Stat: Budget burn rate (multiplier vs 1× expected)

Row: Historical
├── Time series: Availability over last 90 days
└── Time series: Error budget burn rate (alert if >5× for critical alerts)

Row: Breakdown
└── Table: Availability by endpoint
```

---

# 12. Real-World Scenarios

## Scenario 1: High-Load API Monitoring

### Context
E-commerce platform: 500 req/sec peak, 15 microservices, weekly flash sales cause 10× normal traffic.

### Dashboard Design

```
Dashboard: "API Platform — Live Traffic"
Variables: $environment, $service

Row: Platform Overview
├── Stat: Total RPS (all services)
├── Stat: Global error rate
├── Stat: p99 latency (worst service)
└── Time series: RPS trend (last 4 hours, baseline from last week overlaid)

Row: Service Health Matrix
└── Table panel:
    Columns: Service | RPS | Error Rate | p99 | Heap | DB Pool
    Sort by: Error Rate desc
    Thresholds on each column

Row: Traffic Breakdown
├── Time series: Top 5 services by traffic
└── Bar chart: Traffic distribution pie (current period)

Row: Errors Deep Dive  
├── Time series: Error rate by service (stacked)
└── Table: Top 10 error endpoints (uri, method, error count, error rate)
```

### Key Queries

```promql
# Global error rate
sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
/ sum(rate(http_server_requests_seconds_count[5m]))

# Worst-case p99 (max across all services)
max(
  histogram_quantile(0.99,
    sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application)
  )
) by (application)

# Week-over-week traffic comparison
sum(rate(http_server_requests_seconds_count[$__rate_interval]))
  VS
sum(rate(http_server_requests_seconds_count[$__rate_interval] offset 7d))
```

### Alerts Configured

```yaml
- name: GlobalHighErrorRate
  expr: |
    sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
    / sum(rate(http_server_requests_seconds_count[5m])) > 0.02
  for: 3m
  severity: critical

- name: ServiceLatencyDegradation  
  expr: |
    histogram_quantile(0.99, 
      sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application)
    ) > 0.5
  for: 5m
  severity: warning

- name: TrafficAnomaly
  expr: |
    sum(rate(http_server_requests_seconds_count[5m]))
    > 2 * sum(rate(http_server_requests_seconds_count[5m] offset 7d))
  for: 5m
  severity: warning  # Traffic spike — scale up?
```

---

## Scenario 2: Kubernetes Cluster Monitoring

### Context
30-node production cluster, 200 pods across 20 namespaces, 6-person ops team.

### Dashboard Hierarchy

```
L1: "K8s Cluster Overview" (ops team's default tab)
  ├── Cluster health (node count, pod count, alert count)
  ├── Node resource utilization (CPU, memory, disk per node)
  └── Namespace resource quotas

L2: "K8s Namespace: $namespace"
  ├── Deployment health (replicas desired vs ready)
  ├── Pod status breakdown (running/pending/failed/unknown)
  └── Container resource usage

L3: "K8s Pod: $pod" (drill-down)
  ├── CPU/memory over time
  ├── Restart history
  └── Log stream (Loki data source)
```

### Critical Cluster Panels

```promql
# Cluster-wide CPU utilization
sum(rate(node_cpu_seconds_total{mode!="idle"}[5m])) by (instance)
/ sum(rate(node_cpu_seconds_total[5m])) by (instance)

# Pods not running
count(kube_pod_status_phase{phase!="Running",phase!="Succeeded"}) by (namespace, phase)

# Node disk pressure (exclude tmpfs)
1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} 
     / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})

# Container OOMKilled
sum(increase(kube_pod_container_status_restarts_total[1h])) by (namespace, pod, container)
> 0
AND
kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1

# Pending pods (stuck scheduling)
kube_pod_status_phase{phase="Pending"} == 1
```

---

## Scenario 3: Distributed System Observability

### Context
Financial services: 40 microservices, strict compliance, need end-to-end request visibility.

### Unified Observability Dashboard

Grafana with multiple data sources on one dashboard:

```
Row: Request Journey
├── Time series: End-to-end request rate (Prometheus)
├── Time series: Per-hop latency (Prometheus / Istio metrics)
└── Service map: Dependency graph (Tempo / Grafana Explore)

Row: Error Investigation
├── Time series: Error rate by service (Prometheus)
├── Logs panel: Error logs matching current time range (Loki)
└── Trace panel: Sample traces for errors (Tempo/Jaeger)

Row: Infrastructure
├── Time series: Node-level CPU/memory (Prometheus/node_exporter)
└── Table: Database query performance (PostgreSQL data source)
```

### Cross-Signal Links

```json
// In Prometheus Time Series panel — link to Loki logs for same time range
{
  "links": [
    {
      "title": "View error logs",
      "url": "/explore?left={\"datasource\":\"Loki\",\"queries\":[{\"expr\":\"{application=\\\"${__field.labels.application}\\\"}|json|level=\\\"ERROR\\\"\"}],\"range\":{\"from\":\"${__from}\",\"to\":\"${__to}\"}}"
    }
  ]
}

// In Loki Logs panel — link to Jaeger trace for each log line
// Configure derived fields in Loki data source:
// matcherRegex: "traceId=([a-f0-9]+)"
// URL: /explore?datasource=Jaeger&traceId=${__value.raw}
```

---

# 13. Comparisons & Trade-offs

## Grafana vs Kibana

| Aspect | Grafana | Kibana |
|---|---|---|
| **Primary use case** | Metrics + multi-source visualization | Log analytics + search |
| **Data sources** | 30+ built-in (Prometheus, Loki, DBs, cloud) | Primarily Elasticsearch |
| **Log querying** | Loki (LogQL) — simpler, cheaper | Elasticsearch (Lucene/KQL) — more powerful full-text |
| **Metrics visualization** | Excellent (PromQL integration) | Limited (requires TSDB in Elasticsearch) |
| **Dashboard flexibility** | Very high (variables, transformations, plugins) | High but more complex |
| **Alerting** | Unified alerting system | Watcher (complex, requires license) |
| **Cost** | Free (OSS) | Free (OSS), but Elasticsearch cost is high at scale |
| **Setup complexity** | Low–Medium | Medium–High |

**Choose Grafana**: Cloud-native, Kubernetes, Prometheus-based monitoring. Multi-source dashboards.
**Choose Kibana**: You already have ELK stack, need deep log analytics, full-text search across logs.

In practice, many organizations use **both**: Grafana for metrics and operational dashboards, Kibana for log search and analysis.

## Grafana vs Built-in Cloud Dashboards

| Aspect | Grafana | AWS CloudWatch / Azure Monitor |
|---|---|---|
| **Multi-cloud** | Yes — one dashboard for all clouds | Single cloud only |
| **Self-hosted option** | Yes — full control | No — vendor managed |
| **Cost** | Infrastructure only (or Grafana Cloud pricing) | Per metric/API call — expensive at scale |
| **Prometheus integration** | Native | Requires CloudWatch agent or 3rd-party |
| **Customization** | Unlimited | Limited to provided widgets |
| **Vendor lock-in** | None | High |
| **Setup time** | Hours | Minutes |
| **Data retention** | Configurable (your storage) | Limited / expensive |

**Choose cloud-native (CloudWatch/Azure Monitor)**: Small team, single cloud, minimal ops overhead, just getting started.
**Choose Grafana**: Multi-cloud, Kubernetes + Prometheus, need custom dashboards, cost optimization at scale, want portability.

## When Grafana Is NOT Needed

1. **Pure log analysis**: If your entire use case is searching logs and building log-based reports, Kibana is more powerful. Grafana's log panel is good but not a full log analytics tool.

2. **Business intelligence and SQL reporting**: If you need pivot tables, scheduled email reports, non-technical user access, and SQL-first analytics — use Metabase, Redash, or Tableau. Grafana is built for operational data, not BI.

3. **You're fully on a managed platform**: If your team is small, fully on AWS, and CloudWatch meets your needs, adding Grafana is operational overhead without clear benefit. Add it when you outgrow CloudWatch.

4. **Purely event-driven system without time-series needs**: If you only care about discrete events (order placed, payment processed) and have no interest in rates, trends, or utilization — an event tracking system or database is sufficient.

5. **Short-lived or serverless workloads**: Grafana + Prometheus is optimized for long-running services. For Lambda-heavy architectures where cold starts dominate and per-invocation metrics matter more than trends, AWS X-Ray + CloudWatch may be more appropriate.

---

# 14. Production-Ready Checklist

**You are production-ready with Grafana if you can:**

## Fundamentals
- [ ] Explain what Grafana is and is not (visualization layer, not a database)
- [ ] Explain why Grafana proxies data source queries instead of the browser querying directly
- [ ] Connect a Prometheus data source and verify it works
- [ ] Understand dashboard JSON structure and export/import dashboards
- [ ] Use Explore mode to investigate metrics and logs without a pre-built dashboard

## Data Sources
- [ ] Configure Prometheus data source with correct `timeInterval` matching scrape interval
- [ ] Configure a Loki data source and write a basic LogQL query
- [ ] Create a read-only PostgreSQL user and connect it as a data source
- [ ] Set up derived fields in Loki to link trace IDs to Jaeger/Tempo

## Dashboard Building
- [ ] Build a complete service health dashboard with Stat, Time series, and Table panels
- [ ] Set correct units on all panels (seconds, bytes, reqps, percentunit)
- [ ] Configure meaningful thresholds with appropriate colors (green/yellow/red)
- [ ] Add deploy annotations to time series panels
- [ ] Use row organization to structure complex dashboards
- [ ] Configure panel data links to drill down to related dashboards or Explore

## Queries
- [ ] Write `rate()` and `histogram_quantile()` queries in Grafana correctly
- [ ] Use `$__rate_interval` instead of hardcoded time windows
- [ ] Use `=~` with multi-value variables (not `=`)
- [ ] Use Transformations to merge multiple queries into a single table
- [ ] Debug panel issues with Inspector and identify slow queries

## Variables
- [ ] Create a query variable that lists services from `label_values()`
- [ ] Chain variables (environment → service → pod)
- [ ] Enable multi-value selection with "All" option
- [ ] Use variables correctly in queries (regex matching for multi-value)
- [ ] Create hidden constant variables for configurable thresholds

## Alerting
- [ ] Create alert rules for error rate, latency, and service down
- [ ] Configure `for` duration to prevent alert flapping
- [ ] Set up Slack and/or email contact points
- [ ] Configure notification policy routing (critical → PagerDuty, warning → Slack)
- [ ] Create a silence for planned maintenance windows
- [ ] Add runbook URLs to alert annotations

## Production Operations
- [ ] Export dashboards to JSON and store in version control
- [ ] Provision dashboards automatically via file-based provisioning
- [ ] Configure Grafana with OAuth/OIDC for team authentication
- [ ] Set up role-based access (Viewer/Editor/Admin) appropriately
- [ ] Identify slow dashboard panels and optimize via recording rules
- [ ] Apply the "would I use this during an incident?" test to every panel

## Failure Scenarios — Know What Goes Wrong
- [ ] **No data on panels**: Check time range, verify metric name with Explore, check Prometheus `up` metric for target health
- [ ] **Slow dashboard load**: Use Inspector to find slow queries, create recording rules to pre-compute them
- [ ] **Variable shows stale values**: Set `refresh = 2` (on time range change) on variables
- [ ] **Multi-value variable breaks query**: Must use `=~` not `=` for multi-value variables; Grafana joins values with `|`
- [ ] **`histogram_quantile` returns NaN**: The `le` label is missing from the `by()` clause — always include `by (le, ...)`
- [ ] **Annotations missing**: Process restart annotation requires `changes(process_start_time_seconds[2m]) > 0` — verify metric exists
- [ ] **Alert fires and resolves in seconds (flapping)**: Missing or too-short `for` duration — add `for: 5m` minimum
- [ ] **Dashboard lost after Grafana restart**: Data stored in SQLite (default) that wasn't backed up, or using default embedded DB — use PostgreSQL for Grafana's backend DB in production
- [ ] **Variable dropdowns empty**: Query variable uses wrong label or the metric doesn't exist — test in Explore first
- [ ] **Grafana version upgrade breaks dashboards**: Panel types or query behavior changed — test in staging, keep JSON backups before upgrading

---

*Last reviewed: 2024. Grafana 10.x, Prometheus 2.47.x.*
