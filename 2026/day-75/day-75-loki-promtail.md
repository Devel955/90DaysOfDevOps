# Day 75 – Log Management with Loki and Promtail

## Overview

Today we added the **second pillar of observability** to our monitoring stack: **logs**. While metrics (Day 74) tell you *what* is broken, logs tell you *why*. We set up Grafana Loki as a log aggregation backend and Promtail as the log shipping agent, integrating both with the existing Grafana instance.

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│                    Docker Host                                │
│                                                              │
│  ┌─────────────┐   ┌─────────────┐   ┌──────────────────┐   │
│  │  notes-app  │   │  prometheus │   │   other contrs   │   │
│  │  container  │   │  container  │   │                  │   │
│  └──────┬──────┘   └──────┬──────┘   └────────┬─────────┘   │
│         │                 │                    │             │
│         └─────────────────┴────────────────────┘            │
│                           │                                  │
│              Write JSON logs to:                             │
│         /var/lib/docker/containers/*/*-json.log              │
│                           │                                  │
│                           ▼                                  │
│                   ┌───────────────┐                          │
│                   │   Promtail    │  ← reads log files       │
│                   │  (port 9080)  │    adds labels           │
│                   │               │    tracks position        │
│                   └───────┬───────┘                          │
│                           │ HTTP push                        │
│                           ▼                                  │
│                   ┌───────────────┐                          │
│                   │     Loki      │  ← stores logs           │
│                   │  (port 3100)  │    indexes by labels     │
│                   │               │    serves LogQL queries  │
│                   └───────┬───────┘                          │
│                           │ LogQL                            │
│                           ▼                                  │
│                   ┌───────────────┐                          │
│                   │    Grafana    │  ← queries & displays    │
│                   │  (port 3000)  │    metrics + logs        │
│                   └───────┬───────┘                          │
│                           │                                  │
└──────────────────────────────────────────────────────────────┘
                            │
                            ▼
                          [You]
```

---

## Task 1: Why Loki Only Indexes Labels

### The Design Philosophy

Loki takes a fundamentally different approach from Elasticsearch/ELK:

| Feature | Loki | Elasticsearch |
|---|---|---|
| What gets indexed | Only labels (metadata) | Full log text |
| Storage cost | Very low | High |
| Query speed (exact match) | Fast | Faster |
| Full-text search | Scan-based (slower) | Index-based (faster) |
| Operational complexity | Low | High |
| Memory footprint | Low | High |

### The Trade-Off

**Why Loki only indexes labels:**

Loki is inspired by Prometheus. In Prometheus, you identify a time series by its labels (`job`, `instance`, `container_name`) — not by scanning the metric value. Loki applies the same philosophy: logs are identified by labels, and the log body is stored compressed but **not indexed**.

When you query `{container_name="notes-app"} |= "error"`, Loki:
1. Uses the label index to instantly find all log streams for `notes-app` (fast)
2. Scans those log lines for the word "error" (linear scan, slower)

**The trade-off:**
- ✅ **Cheap storage** — compressed chunks, no inverted index bloat
- ✅ **Simple operations** — no index management, shards, or replicas needed
- ✅ **Low cardinality requirement** — same discipline as Prometheus labels
- ❌ **Full-text search is slower** — no inverted index means scanning raw log chunks
- ❌ **Not suited for complex text analytics** — Elasticsearch wins for log mining/SIEM use cases

**Bottom line:** Loki is the right choice when you already use Prometheus/Grafana and want cheap, simple log aggregation. Elasticsearch is better when you need powerful full-text search, aggregations, or log analytics at scale.

---

## Configuration Files

### `loki/loki-config.yml`

```yaml
auth_enabled: false
# Single-tenant mode — no authentication needed.
# In production, set to true and configure tenants.

server:
  http_listen_port: 3100
  # Loki's HTTP API and push endpoint

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
      # For single-node: use in-memory key-value store.
      # In a cluster, this would be etcd or consul.
  replication_factor: 1
  # No replication — single instance setup (fine for dev/learning)
  path_prefix: /loki

schema_config:
  configs:
    - from: 2020-10-24
      store: tsdb
      # TSDB (Time Series Database) for label indexing —
      # Loki's modern, efficient index format
      object_store: filesystem
      # Store log chunks on local disk.
      # In production: S3, GCS, or Azure Blob
      schema: v13
      # Latest Loki schema version
      index:
        prefix: index_
        period: 24h
        # Create a new index table every 24 hours

storage_config:
  filesystem:
    directory: /loki/chunks
    # Where compressed log chunks are stored on disk
```

### `promtail/promtail-config.yml`

```yaml
server:
  http_listen_port: 9080
  # Promtail's own HTTP server (for /metrics, /targets, /ready)
  grpc_listen_port: 0
  # Disable gRPC — not needed for basic setup

positions:
  filename: /tmp/positions.yaml
  # Tracks how far Promtail has read each log file.
  # Like a bookmark — prevents re-sending already-shipped logs.
  # If deleted, Promtail re-reads all logs from the beginning.

clients:
  - url: http://loki:3100/loki/api/v1/push
  # The Loki push endpoint. Uses Docker internal DNS (loki = container name).

scrape_configs:
  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          # This label will appear on every log entry from this job
          __path__: /var/lib/docker/containers/*/*-json.log
          # Glob pattern to find ALL Docker container JSON log files.
          # Docker writes one file per container at this path.
    pipeline_stages:
      - docker: {}
      # Built-in stage that parses Docker's JSON log format:
      # {"log":"...", "stream":"stdout", "time":"..."}
      # Extracts: log line, stream (stdout/stderr), and timestamp.
      # Also discovers container metadata via /var/run/docker.sock.
```

---

## Updated `docker-compose.yml`

```yaml
version: "3.8"

services:
  notes-app:
    image: notes-app:latest
    container_name: notes-app
    ports:
      - "8000:8000"
    restart: unless-stopped

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    restart: unless-stopped

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    volumes:
      - ./loki/loki-config.yml:/etc/loki/loki-config.yml
      - loki_data:/loki
    command: -config.file=/etc/loki/loki-config.yml
    restart: unless-stopped

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/promtail-config.yml
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      # Read-only access to Docker log files on the host
      - /var/run/docker.sock:/var/run/docker.sock
      # Docker socket — lets Promtail discover container names/labels
    command: -config.file=/etc/promtail/promtail-config.yml
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
```

---

## Grafana Datasource Provisioning

`grafana/provisioning/datasources/datasources.yml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false

  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    editable: false
```

---

## Five LogQL Queries

### 1. All Docker container logs (basic stream selector)
```logql
{job="docker"}
```
**What it returned:** A live stream of log lines from every running container — Prometheus, Grafana, Loki, Promtail, notes-app, etc. Each line showed the container name, timestamp, and log message.

---

### 2. Filter by container + keyword search
```logql
{container_name="notes-app"} |= "error"
```
**What it returned:** Only log lines from the `notes-app` container that contained the word "error". Useful for quick debugging without noise from other containers.

---

### 3. HTTP 4xx/5xx error detection with regex
```logql
{job="docker"} |~ "status=[45]\\d{2}"
```
**What it returned:** Log lines from any container matching HTTP status codes like `status=404`, `status=500`, `status=503`. The regex `[45]\d{2}` matches any 4xx or 5xx code. Useful for detecting errors across all services at once.

---

### 4. Rate of logs per second (log metric query)
```logql
rate({job="docker"}[5m])
```
**What it returned:** A time-series graph showing the number of log lines per second across all containers, averaged over a 5-minute window. Helps detect "log storms" which often indicate a crash loop or flood of errors.

---

### 5. Top containers by log volume
```logql
topk(5, sum by (container_name) (rate({job="docker"}[5m])))
```
**What it returned:** The 5 containers generating the most log output, ranked by log lines per second. In our stack, Prometheus and cAdvisor topped the list due to frequent scrape activity and health check logs.

---

## Correlating Metrics and Logs During Incidents

Having both Prometheus metrics and Loki logs inside the same Grafana instance provides a powerful, unified debugging workflow:

**In separate systems (traditional approach):**
1. Alert fires → open Datadog/Prometheus to find the spike
2. Copy the timestamp → open Kibana/Splunk
3. Manually filter logs to match the timeframe
4. Context-switch between tabs, lose time, miss correlations

**In Grafana with Prometheus + Loki:**
1. Alert fires → open Grafana Explore (split view)
2. Left panel: Prometheus shows the CPU spike at 14:32:05
3. Click on the spike → both panels instantly zoom to 14:32:00–14:32:30
4. Right panel: Loki shows the exact log lines from that 30-second window
5. You see `ERROR: database connection timeout` appearing at 14:32:03 — root cause found in under 60 seconds

The time synchronization between panels is the key feature. In production, seconds matter during an incident, and not having to manually coordinate timestamps between two separate systems dramatically cuts mean-time-to-resolution (MTTR).

---

## Loki vs ELK Stack — When to Use Each

| Criteria | Loki | ELK Stack (Elasticsearch + Logstash + Kibana) |
|---|---|---|
| **Setup complexity** | Low — single binary or compose | High — multiple components, heap tuning |
| **Storage cost** | Very low (compressed chunks) | High (inverted indexes are large) |
| **Full-text search** | Scan-based (adequate) | Index-based (fast, powerful) |
| **Log analytics / aggregations** | Basic | Advanced (Kibana Lens, ESQL) |
| **Already using Grafana?** | ✅ Perfect fit | Adds a separate UI (Kibana) |
| **Already using Prometheus?** | ✅ Same label model | Different mental model |
| **Need SIEM / security analytics** | ❌ Not designed for it | ✅ Elastic Security |
| **Multi-tenancy** | Supported | Supported |
| **Schema-on-read** | ✅ (no pre-mapping needed) | ❌ (requires index mappings) |
| **Horizontal scalability** | Good (with object storage) | Excellent (mature, proven) |

### Use **Loki** when:
- You already run Prometheus + Grafana
- You want simple, low-cost log aggregation
- Your primary use case is debugging — "what happened in this container around this time?"
- You want to avoid operational overhead of Elasticsearch (heap tuning, index lifecycle management, cluster management)

### Use **ELK** when:
- You need powerful full-text search across billions of log events
- You do log analytics, reporting, or dashboards over structured log data
- You need SIEM (security information and event management)
- Your team is already invested in the Elastic ecosystem

---

## Key Takeaways

1. **Loki = "Prometheus for logs"** — same label-based model, same Grafana UI, minimal operational cost
2. **Promtail is the agent** — runs on the host, reads Docker JSON log files, ships to Loki with labels
3. **LogQL is powerful** — stream selectors + filter expressions + metric queries cover most production debugging needs
4. **Correlation is the superpower** — metrics + logs in the same tool eliminates context-switching during incidents
5. **Keep label cardinality low** — `container_name`, `job`, `host` are good labels; request IDs or user IDs as labels would create millions of streams and break Loki

---

*Day 75 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
