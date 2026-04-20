# Day 76 — OpenTelemetry and Alerting

## Overview

Today I added the third pillar of observability — **traces** — using OpenTelemetry, and set up alerting so the system notifies on problems automatically. The stack now covers all three pillars: **metrics**, **logs**, and **traces**.

---

## Task 1: OpenTelemetry Concepts

### What is OpenTelemetry (OTEL)?

OpenTelemetry is a **vendor-neutral, open-source framework** for generating, collecting, and exporting telemetry data — specifically metrics, logs, and traces. Key points:

- It is **not a backend** — it collects and ships data to backends like Prometheus, Jaeger, Loki, or Datadog
- Backed by CNCF (Cloud Native Computing Foundation)
- Provides SDKs for virtually every major language (Python, Go, Java, Node.js, etc.)
- Replaces older, fragmented standards like OpenTracing and OpenCensus

### What is the OTEL Collector?

A standalone service that acts as a **pipeline agent** for telemetry. It has three component types:

| Component | Role | Examples |
|-----------|------|---------|
| **Receivers** | Accept incoming data | OTLP, Prometheus, Jaeger, Zipkin |
| **Processors** | Transform/filter data | Batch, memory limiter, attributes |
| **Exporters** | Send data to backends | Prometheus, Jaeger, Loki, debug console |

The collector decouples your applications from specific backends — change the exporter config without touching app code.

### What is OTLP?

**OpenTelemetry Protocol** — the standard wire format for sending telemetry data between OTEL components:

- Supports **gRPC** on port `4317`
- Supports **HTTP/JSON** on port `4318`
- Used by all OTEL SDKs and the collector

### What are Distributed Traces?

A **trace** tracks a single request as it travels through multiple services. Each step is a **span**.

```
User Request
  └─ span 1: API Gateway        (100ms)
       └─ span 2: Auth Service  (30ms)
       └─ span 3: DB Query      (50ms)
```

Each span contains:
- `traceId` — unique ID shared across the entire request
- `spanId` — unique ID for this specific step
- `parentSpanId` — links spans into a tree
- `startTimeUnixNano` / `endTimeUnixNano` — timing
- `attributes` — key-value metadata (e.g., `http.method`, `http.status_code`)

---

## Task 2: OpenTelemetry Collector Setup

### Collector Configuration

**File:** `otel-collector/otel-collector-config.yml`

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  debug:
    verbosity: detailed

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
    logs:
      receivers: [otlp]
      processors: [batch]
      exporters: [debug]
```

**Explanation of each section:**

- **receivers.otlp** — listens for incoming OTLP data on both gRPC (4317) and HTTP (4318)
- **processors.batch** — buffers data and sends in batches, reducing network overhead and backend load
- **exporters.prometheus** — exposes a `/metrics` endpoint on port 8889 that Prometheus can scrape
- **exporters.debug** — prints telemetry to stdout; useful for development, replaced by Jaeger/Tempo in production
- **service.pipelines** — three independent pipelines: metrics, traces, logs; each wires receivers → processors → exporters

### Docker Compose Addition

```yaml
otel-collector:
  image: otel/opentelemetry-collector-contrib:latest
  container_name: otel-collector
  ports:
    - "4317:4317"   # OTLP gRPC
    - "4318:4318"   # OTLP HTTP
    - "8889:8889"   # Prometheus exporter
  volumes:
    - ./otel-collector/otel-collector-config.yml:/etc/otelcol-contrib/config.yaml
  restart: unless-stopped
```

> **Note:** Always use the `contrib` image (`otel/opentelemetry-collector-contrib`) — it includes far more receivers and exporters than the core image.

### Prometheus Scrape Target

Added to `prometheus.yml`:

```yaml
- job_name: "otel-collector"
  static_configs:
    - targets: ["otel-collector:8889"]
```

### Verification

```bash
docker logs otel-collector 2>&1 | tail -5
# Expected: "Everything is ready. Begin running and processing data."
```

Prometheus Targets page shows `otel-collector` as **UP**.

---

## Task 3: Sending Test Traces and Metrics

### Test Trace via OTLP HTTP

```bash
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": { "stringValue": "my-test-service" }
        }]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "5b8efff798038103d269b633813fc60c",
          "spanId": "eee19b7ec3c1b174",
          "name": "test-span",
          "kind": 1,
          "startTimeUnixNano": "1544712660000000000",
          "endTimeUnixNano": "1544712661000000000",
          "attributes": [
            { "key": "http.method", "value": { "stringValue": "GET" } },
            { "key": "http.status_code", "value": { "intValue": "200" } }
          ]
        }]
      }]
    }]
  }'
```

Check collector logs for the trace:

```bash
docker logs otel-collector 2>&1 | grep -A 10 "test-span"
```

**Expected debug output:**

```
-> test-span
    Trace ID: 5b8efff798038103d269b633813fc60c
    Span ID:  eee19b7ec3c1b174
    Kind:     Server
    -> http.method: GET
    -> http.status_code: 200
```

> 📸 **[Screenshot: OTEL Collector debug output showing test-span trace]**

### Test Metric via OTLP HTTP

```bash
curl -X POST http://localhost:4318/v1/metrics \
  -H "Content-Type: application/json" \
  -d '{
    "resourceMetrics": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": { "stringValue": "my-test-service" }
        }]
      },
      "scopeMetrics": [{
        "metrics": [{
          "name": "test_requests_total",
          "sum": {
            "dataPoints": [{
              "asInt": "42",
              "startTimeUnixNano": "1544712660000000000",
              "timeUnixNano": "1544712661000000000"
            }],
            "aggregationTemporality": 2,
            "isMonotonic": true
          }
        }]
      }]
    }]
  }'
```

**Data flow:**
```
curl OTLP → OTEL Collector (OTLP receiver)
          → batch processor
          → Prometheus exporter (port 8889)
          → Prometheus scrapes every 15s
          → Query: test_requests_total = 42
```

---

## Task 4: Prometheus Alerting Rules

### Alert Rules File

**File:** `alert-rules.yml`

```yaml
groups:
  - name: system-alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage has been above 80% for more than 2 minutes. Current value: {{ $value }}%"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 85%. Current value: {{ $value }}%"

      - alert: ContainerDown
        expr: absent(container_last_seen{name="notes-app"})
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container is down"
          description: "The notes-app container has not been seen for over 1 minute"

      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Scrape target is down"
          description: "{{ $labels.job }} target {{ $labels.instance }} is unreachable"

      - alert: HighDiskUsage
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk space running low"
          description: "Root filesystem usage is above 90%. Current value: {{ $value }}%"
```

### Alert Rule Explanations

| Alert | Expression Logic | `for` Period | Severity | Purpose |
|-------|-----------------|-------------|----------|---------|
| `HighCPUUsage` | 5-min rate of non-idle CPU > 80% | 2m | warning | Catches sustained CPU pressure, not brief spikes |
| `HighMemoryUsage` | Used memory / total memory > 85% | 2m | warning | Early warning before OOM kills |
| `ContainerDown` | `absent()` fires when `notes-app` label disappears | 1m | critical | Detects crashed or removed containers |
| `TargetDown` | Any scrape target returns `up == 0` | 1m | critical | Catches any exporter going offline |
| `HighDiskUsage` | Root filesystem used > 90% | 5m | critical | Prevents disk-full failures |

**Key concepts:**
- **`for`** — the *pending period*: the condition must hold continuously before the alert fires (prevents flapping on brief spikes)
- **`absent()`** — fires when a time series *disappears entirely* — essential for detecting dead containers
- **`labels.severity`** — used for routing; `warning` = investigate soon, `critical` = page now
- **`{{ $value }}`** — template variable that inserts the actual metric value into the alert message

### Updated prometheus.yml

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - /etc/prometheus/alert-rules.yml

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
  - job_name: "otel-collector"
    static_configs:
      - targets: ["otel-collector:8889"]
```

### Testing an Alert

```bash
# Stop the notes-app to trigger TargetDown
docker compose stop notes-app

# Wait ~90 seconds, then check Prometheus UI > Alerts
# Alert moves: inactive → pending → firing

# Restore
docker compose start notes-app
```

> 📸 **[Screenshot: Prometheus Alerts page showing alert states — inactive/pending/firing]**

---

## Task 5: Grafana Alerting

### Contact Point Setup

1. Go to **Alerting > Contact points > Add contact point**
2. Name: `DevOps Team`
3. Integration: Email (or Slack webhook)
4. Save

### Alert Rule: High Container Memory

1. Go to **Alerting > Alert rules > New alert rule**
2. Name: `High Container Memory`
3. Query: `container_memory_usage_bytes{name="notes-app"} / 1024 / 1024`
4. Condition: **IS ABOVE 100** (fires if container uses more than 100 MB)
5. Evaluation: every **1m**, for **2m**
6. Label: `severity = warning`
7. Notification: link to `DevOps Team` contact point

### Notification Policies

- **Default policy** → `DevOps Team`
- **Nested policy**: match `severity=critical` → can route to a higher-urgency contact (PagerDuty, on-call phone, etc)

### Prometheus Alerts vs Grafana Alerts — When to Use Each

| | Prometheus Alerts | Grafana Alerts |
|---|---|---|
| **Evaluation** | Prometheus server evaluates PromQL | Grafana evaluates queries against any data source |
| **Notifications** | Requires Alertmanager (separate service) | Built-in notification system (email, Slack, PagerDuty, etc.) |
| **Data sources** | Metrics only | Metrics, logs (Loki), traces, SQL, cloud APIs |
| **Best for** | Infrastructure and SRE alerts on metrics; GitOps-managed rule files | Dashboard-linked alerts; multi-source conditions; getting started quickly without Alertmanager |
| **Routing** | Alertmanager provides advanced routing, grouping, silencing | Notification policies provide routing; simpler but less powerful |

**Rule of thumb:** Use Prometheus alert rules for metric-based SLOs and infrastructure alarms managed in code. Use Grafana alerts when you need cross-data-source conditions, or want a UI-first workflow for teams without Alertmanager experience.

---

## Task 6: Full Stack Architecture

### Architecture Diagram

```
╔══════════════════════════════════════════════════════════════╗
║                    METRICS PIPELINE                          ║
║                                                              ║
║  [Node Exporter :9100] ──┐                                   ║
║  [cAdvisor :8080] ───────┼──► [Prometheus :9090] ──► [Grafana :3000]
║  [OTEL Collector :8889] ─┘          │                  Dashboards
║                                     └──► Alert Rules         ║
║                                          → Grafana Alerts    ║
║                                          → Notifications     ║
╠══════════════════════════════════════════════════════════════╣
║                    LOGS PIPELINE                             ║
║                                                              ║
║  [Docker Containers]                                         ║
║       │                                                      ║
║       ▼                                                      ║
║  [Promtail :9080] ──────────────► [Loki :3100] ──► [Grafana :3000]
║                                                      Explore  ║
╠══════════════════════════════════════════════════════════════╣
║                    TRACES PIPELINE                           ║
║                                                              ║
║  [App/curl OTLP]                                             ║
║       │                                                      ║
║       ▼                                                      ║
║  [OTEL Collector :4317/:4318]                                ║
║       │                                                      ║
║       └──► Debug stdout (dev)                                ║
║       └──► [Jaeger / Grafana Tempo] (production)             ║
╚══════════════════════════════════════════════════════════════╝
```

### Services Reference Table

| Service | Port(s) | Purpose |
|---------|---------|---------|
| Prometheus | 9090 | Metrics storage and PromQL querying |
| Node Exporter | 9100 | Host system metrics (CPU, memory, disk, network) |
| cAdvisor | 8080 | Per-container resource metrics |
| Grafana | 3000 | Visualization, dashboards, and alerting UI |
| Loki | 3100 | Log storage and querying |
| Promtail | 9080 | Log collection agent (tails Docker logs → Loki) |
| OTEL Collector | 4317 (gRPC), 4318 (HTTP), 8889 (Prometheus) | Telemetry collection, processing, and export |
| Notes App | 8000 | Sample application being monitored |

### Verify All Services

```bash
docker compose ps
# All 8 containers should show status: running (healthy)
```

---

## Key Takeaways

1. **OpenTelemetry is a collection framework, not a backend.** It standardizes how telemetry is emitted and transported, but you still need Prometheus, Loki, Jaeger, etc. for storage and querying.

2. **The OTEL Collector is a pipeline.** Think of it as a configurable ETL for telemetry: receive in one format, process (batch, filter, enrich), export in another format.

3. **`for:` in alert rules prevents noise.** A condition must hold for the entire pending period before firing — this eliminates false alarms from brief spikes.

4. **`absent()` is essential for detecting dead services.** Normal threshold alerts can't fire if a metric has disappeared; `absent()` specifically catches missing series.

5. **Grafana alerts are the easier starting point for notifications.** Prometheus alerts require Alertmanager for delivery; Grafana's built-in alerting handles email/Slack/PagerDuty out of the box.

6. **All three pillars complement each other:**
   - **Metrics** tell you *something is wrong* (high latency, error rate spike)
   - **Logs** tell you *what happened* (error messages, stack traces)
   - **Traces** tell you *where* the problem is (which service, which call)
