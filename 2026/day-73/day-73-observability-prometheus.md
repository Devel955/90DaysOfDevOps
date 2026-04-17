# Day 73 – Introduction to Observability and Prometheus

## 1. The Three Pillars of Observability (In My Own Words)

Traditional monitoring is reactive: you set thresholds and get paged when something crosses them. Observability is proactive and explorative — it gives you the ability to ask *any* question about your system's internal state just by looking at the data it emits, even questions you didn't think to ask in advance.

### Metrics
Numerical measurements collected at regular intervals over time. Cheap to store, fast to query — ideal for dashboards and alerts.

- **Examples:** CPU usage %, HTTP requests/second, error rate, memory in use
- **Tools:** Prometheus, Datadog, AWS CloudWatch
- **Answers:** *What* is broken? (e.g., "error rate on `/api/users` spiked to 40%")

### Logs
Timestamped text records emitted by your application or system. Rich contextual detail but more expensive to store and search.

- **Examples:** `ERROR: DB connection timeout after 30s`, `INFO: user 42 logged in`
- **Tools:** Loki, ELK Stack, Fluentd
- **Answers:** *Why* is it broken? (e.g., "stack trace shows a database timeout")

### Traces
The end-to-end journey of a single request as it flows across multiple services. Each step (span) is timed and tagged.

- **Examples:** A checkout request hitting API → auth service → inventory → payment service
- **Tools:** OpenTelemetry, Jaeger, Zipkin
- **Answers:** *Where* is it broken? (e.g., "the payment service took 12s out of 13s total")

### Why DevOps Engineers Need All Three

| Pillar | Answers | Without It |
|--------|---------|------------|
| Metrics | *What* is wrong | You don't know something is broken |
| Logs | *Why* it is wrong | You can't root-cause the issue |
| Traces | *Where* it is wrong | You can't isolate which service is the culprit |

---

## 2. Observability Architecture (Days 73–77)

```
[Your App]  ──── metrics ──►  [Prometheus]       ──► [Grafana Dashboards]
[Your App]  ──── logs    ──►  [Promtail]     ──►  [Loki]   ──► [Grafana]
[Your App]  ──── traces  ──►  [OTEL Collector]    ──► [Grafana / Tempo]
[Host]      ──── metrics ──►  [Node Exporter]     ──► [Prometheus]
[Docker]    ──── metrics ──►  [cAdvisor]          ──► [Prometheus]
```

| Day | Topic |
|-----|-------|
| Day 73 | Prometheus basics, PromQL, scrape targets (today) |
| Day 74 | Node Exporter (host metrics) + cAdvisor (container metrics) |
| Day 75 | Grafana dashboards |
| Day 76 | Loki + Promtail (logs) |
| Day 77 | OpenTelemetry tracing |

---

## 3. Configuration Files

### `prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "notes-app"
    static_configs:
      - targets: ["notes-app:8000"]
```

### `docker-compose.yml`

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
      - '--storage.tsdb.retention.size=1GB'
    restart: unless-stopped

  notes-app:
    image: trainwithshubham/notes-app:latest
    container_name: notes-app
    ports:
      - "8000:8000"
    restart: unless-stopped

volumes:
  prometheus_data:
```

---

## 4. Prometheus Targets — Both UP ✅

**Status → Targets** page showing both scrape targets healthy:

![Prometheus Targets UP](./screenshots/targets.png)

| Job | Endpoint | State | Last Scrape |
|-----|----------|-------|-------------|
| notes-app | `http://notes-app:8000/metrics` | **UP** | 3.263s ago |
| prometheus | `http://localhost:9090/metrics` | **UP** | 300ms ago |

---

## 5. Counter vs Gauge

### Counter
A counter **only ever increases** (or resets to 0 on restart). It represents a running total.

- **Real-world example:** `http_requests_total` — every incoming HTTP request increments this. It never goes down. Use `rate()` to convert it to requests/second.

### Gauge
A gauge can **go up and down**. It represents a current snapshot of something that fluctuates.

- **Real-world example:** `process_resident_memory_bytes` — current RAM used by a process. Rises as app allocates memory, drops when freed. Query directly without `rate()`.

| | Counter | Gauge |
|---|---|---|
| Direction | Only up | Up and down |
| Query pattern | `rate(counter[5m])` | Direct query |
| Examples | total requests, total errors | CPU %, active connections, queue depth |

---

## 6. Five PromQL Queries With Screenshots

### Query 1 — Count all time series being collected

```promql
count({__name__=~".+"})
```

**Graph view** — spike after notes-app was added:

![Count All Metrics Graph](./screenshots/query1_count_graph.png)

**Table view** — exact count:

![Count All Metrics Table](./screenshots/query1_count_table.png)

**Result: `1115` time series** — Prometheus is collecting 1115 unique metric+label combinations across both targets.

---

### Query 2 — Memory used by each target (bytes → MB)

```promql
process_resident_memory_bytes / 1024 / 1024
```

**Graph view** — two separate gauges over time:

![Memory Graph](./screenshots/query2_memory_graph.png)

**Table view** — current values:

![Memory Table](./screenshots/query2_memory_table.png)

**Results:**
- `notes-app` → **45.98 MB**
- `prometheus` → **103.71 MB**

---

### Query 3 — Total HTTP requests broken down by handler

```promql
prometheus_http_requests_total
```

**Table view** — 60 result series, each handler/code combination:

![HTTP Requests Total Table](./screenshots/query3_http_table.png)

**Graph view** — stacked counters growing over time:

![HTTP Requests Total Graph](./screenshots/query3_http_graph.png)

**Result:** 60 time series returned. Each unique `{code, handler, instance, job}` label combination is a separate time series. This is the power of labels in Prometheus.

---

### Query 4 — Per-second rate of HTTP requests over 5 minutes

```promql
rate(prometheus_http_requests_total[5m])
```

**Graph view** — peak ~0.17 req/sec across all handlers:

![Rate Graph](./screenshots/query4_rate_graph.png)

**Table view** — fractional per-second values per handler:

![Rate Table](./screenshots/query4_rate_table.png)

**Result:** `rate()` converts the ever-growing counter into a useful speed (requests/second). This is the most important PromQL pattern — **always use `rate()` on counters**.

---

### Query 5 — Challenge Query: Rate of non-200 HTTP requests (last 5 minutes)

```promql
rate(prometheus_http_requests_total{code!="200"}[5m])
```

**Result:** Returns `0` for all series during normal operation — a healthy sign. In production, any spike here would indicate errors worth investigating. This query combines label filtering (`code!="200"`) with `rate()` to detect error traffic.

---

## 7. Notes-App Metrics Endpoint

The `notes-app` is a **Django + React** application with `django-prometheus` library built in. Visiting `http://localhost:8000/metrics` in the browser shows raw Prometheus-format metrics:

```
# HELP django_http_requests_total_by_method_total Count of requests by method.
django_http_requests_total_by_method_total{method="GET"} 61.0

# HELP django_http_responses_total_by_status_total Count of responses by status.
django_http_responses_total_by_status_total{status="200"} 57.0
django_http_responses_total_by_status_total{status="404"} 3.0

# HELP process_resident_memory_bytes Resident memory size in bytes.
process_resident_memory_bytes 4.9524736e+07   # ~45.98 MB

# HELP python_info Python platform information.
python_info{implementation="CPython", version="3.9.25"} 1.0
```

### What these metrics tell us:

| Metric | Value | Meaning |
|--------|-------|---------|
| `django_http_requests_total_by_method_total{method="GET"}` | 61 | Total GET requests served |
| `django_http_responses_total_by_status_total{status="200"}` | 57 | Successful responses |
| `django_http_responses_total_by_status_total{status="404"}` | 3 | Not found errors |
| `process_resident_memory_bytes` | ~46 MB | RAM used by Django process |
| `django_http_requests_latency_seconds_by_view_method` | Histogram | Request duration per view |

This is what a **real application exposing Prometheus metrics** looks like. Prometheus scrapes this endpoint every 15 seconds and stores all these time series in its TSDB.

---

## 8. Data Retention and Storage

### What happens when retention is exceeded?
Prometheus stores data in a local time-series database (TSDB) using **2-hour blocks**. When the configured retention limit is hit (default: **15 days**), Prometheus automatically **deletes the oldest blocks**. No archiving — data is permanently removed.

Retention can be configured by time, size, or both (whichever is hit first):

```yaml
command:
  - '--storage.tsdb.retention.time=30d'
  - '--storage.tsdb.retention.size=1GB'
```

### Why is a volume mount critical?
Without `prometheus_data:/prometheus`, all scraped data lives only in the container's writable layer. Any restart, upgrade, or `docker compose down` **permanently destroys all historical metrics**.

With the volume mount:
- Data survives container restarts and image upgrades
- `docker compose down && docker compose up` retains full history
- Volume can be backed up independently

```bash
# Check how much disk Prometheus is using
docker exec prometheus du -sh /prometheus
```

---

## 8. Key Takeaways

1. **Pull model** — Prometheus *scrapes* targets at `scrape_interval`. Unlike push-based systems, targets don't send data — Prometheus fetches it.
2. **`up` metric** — Auto-created for every target. `1` = healthy, `0` = unreachable. First thing to check when debugging.
3. **`rate()` before `sum()`** — `sum(rate(metric[5m]))` ✅ NOT `rate(sum(metric[5m]))` ❌
4. **Labels add dimensions** — `http_requests_total{method="GET", status="200"}` is a different time series from `{method="POST", status="500"}`.
5. **PromQL is declarative** — Master `rate()`, `sum()`, `topk()`, and label matchers and you can answer 90% of operational questions.
