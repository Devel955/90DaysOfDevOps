# Day 77 — Full Observability Stack Integration Project

> **5-Day Observability Block Capstone** | Prometheus · Grafana · Loki · Promtail · OpenTelemetry · Node Exporter · cAdvisor · Alerting

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Service Summary](#service-summary)
3. [Setup & Launch](#setup--launch)
4. [Prometheus Targets Validation](#prometheus-targets-validation)
5. [Grafana Explore — Loki Logs](#grafana-explore--loki-logs)
6. [Production Overview Dashboard](#production-overview-dashboard)
7. [OTEL Trace Validation](#otel-trace-validation)
8. [Config Comparison: My Versions vs Reference Repo](#config-comparison-my-versions-vs-reference-repo)
9. [All Configuration Files](#all-configuration-files)
10. [Production Readiness Additions](#production-readiness-additions)
11. [Key Takeaways — 5-Day Observability Block](#key-takeaways--5-day-observability-block)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        OBSERVABILITY STACK — DATA FLOWS                     │
└─────────────────────────────────────────────────────────────────────────────┘

  ┌─────────────┐   METRICS (scrape)    ┌──────────────────┐
  │ Node Exporter│◄─────────────────────│                  │
  │  :9100       │                      │   PROMETHEUS      │
  ├─────────────┤   METRICS (scrape)    │     :9090         │
  │  cAdvisor   │◄─────────────────────│                  │
  │  :8080       │                      │  Stores TSDB      │
  ├─────────────┤   METRICS (scrape)    │  Evaluates Rules  │
  │ OTEL Collect│◄─────────────────────│  Fires Alerts     │
  │ :4317/:4318  │                      └────────┬─────────┘
  └─────────────┘                               │
         │                                      │ METRICS QUERY
         │ OTLP (traces in)                     ▼
         │                              ┌──────────────────┐
  ┌──────▼──────┐                      │     GRAFANA       │
  │    Notes    │ LOGS ──► PROMTAIL ──►│     :3000         │◄── LOGS QUERY
  │    App      │          :9080       │                  │
  │   :8000     │                      │  Dashboards       │
  └─────────────┘                      │  Explore          │
                                       │  Alerting         │
  ┌─────────────┐   LOGS (push)        └──────────────────┘
  │  Docker     │──────────────────►           │
  │  Containers │   (all container            │ LOGS QUERY
  │  (stdout)   │    logs via                 ▼
  └─────────────┘    Promtail)        ┌──────────────────┐
                                      │      LOKI         │
                                      │     :3100         │
                                      │  Log Aggregation  │
                                      │  LogQL Engine     │
                                      └──────────────────┘

  DATA FLOW LEGEND
  ──────────────────────────────────────────────────────────
  METRICS  →  Node Exporter / cAdvisor → Prometheus → Grafana
  LOGS     →  Docker stdout → Promtail → Loki → Grafana
  TRACES   →  App → OTEL Collector (OTLP) → Debug/Prometheus
  ALERTS   →  Prometheus (rules) → [Alertmanager] → Slack/PD
```

---

## Service Summary

| Service          | Port(s)    | Role                                           | Health Check                             |
|------------------|------------|------------------------------------------------|------------------------------------------|
| **Prometheus**   | 9090       | Metrics collection, TSDB, alert evaluation     | `http://localhost:9090/targets`          |
| **Node Exporter**| 9100       | Host-level metrics (CPU, memory, disk, net)    | `curl http://localhost:9100/metrics`     |
| **cAdvisor**     | 8080       | Per-container resource metrics                 | `http://localhost:8080`                  |
| **Grafana**      | 3000       | Visualization, dashboards, explore, alerting   | `http://localhost:3000` (admin/admin)    |
| **Loki**         | 3100       | Log aggregation and LogQL query engine         | `curl http://localhost:3100/ready`       |
| **Promtail**     | 9080       | Log collection agent (Docker → Loki)           | `curl http://localhost:9080/targets`     |
| **OTEL Collector**| 4317/4318 | Trace and metrics ingestion (OTLP gRPC/HTTP)   | `docker logs otel-collector`             |
| **Notes App**    | 8000       | Sample Django + React app (generates telemetry)| `http://localhost:8000/api/`            |

---

## Setup & Launch

### 1. Clone the Reference Repository

```bash
git clone https://github.com/LondheShubham153/observability-for-devops.git
cd observability-for-devops
```

### 2. Examine the Project Structure

```bash
tree -I 'node_modules|build|staticfiles|__pycache__'
```

Expected output:

```
observability-for-devops/
├── docker-compose.yml
├── prometheus.yml
├── alert-rules.yml
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   └── datasources.yml
│       └── dashboards/
│           └── dashboards.yml
├── loki/
│   └── loki-config.yml
├── promtail/
│   └── promtail-config.yml
├── otel-collector/
│   └── otel-collector-config.yml
└── notes-app/
    ├── Dockerfile
    ├── manage.py
    └── ...
```

### 3. Launch the Full Stack

```bash
docker compose up -d
```

### 4. Verify All Containers Are Running

```bash
docker compose ps
```

Expected — all 8 services in state `Up`:

```
NAME                COMMAND                  SERVICE         STATUS          PORTS
cadvisor            "/usr/bin/cadvisor -…"   cadvisor        Up              0.0.0.0:8080->8080/tcp
grafana             "/run.sh"                grafana         Up              0.0.0.0:3000->3000/tcp
loki                "/usr/bin/loki -conf…"   loki            Up              0.0.0.0:3100->3100/tcp
node-exporter       "/bin/node_exporter …"   node-exporter   Up              0.0.0.0:9100->9100/tcp
notes-app           "gunicorn notes.wsgi…"   notes-app       Up              0.0.0.0:8000->8000/tcp
otel-collector      "/otelcol-contrib --…"   otel-collector  Up              0.0.0.0:4317->4317/tcp, 0.0.0.0:4318->4318/tcp
prometheus          "/bin/prometheus --c…"   prometheus      Up              0.0.0.0:9090->9090/tcp
promtail            "/usr/bin/promtail -…"   promtail        Up              0.0.0.0:9080->9080/tcp
```

---

## Prometheus Targets Validation

**URL:** `http://localhost:9090/targets`

All 4 scrape jobs should show state **UP**:

| Job             | Target                         | State | Labels                        |
|-----------------|-------------------------------|-------|-------------------------------|
| prometheus      | localhost:9090                | UP    | instance="localhost:9090"     |
| node-exporter   | node-exporter:9100            | UP    | instance="node-exporter:9100" |
| cadvisor        | cadvisor:8080                 | UP    | instance="cadvisor:8080"      |
| otel-collector  | otel-collector:8888           | UP    | instance="otel-collector:8888"|

### Validation PromQL Queries

```promql
# 1. All targets healthy (should return 1 for each)
up

# 2. Host CPU usage (%)
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 3. Memory usage (%)
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# 4. Container CPU per container (%)
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100

# 5. Top 3 memory-hungry containers
topk(3, container_memory_usage_bytes{name!=""})
```

> **Screenshot placeholder:** Prometheus Targets page showing all jobs in UP state with green indicators.

---

## Grafana Explore — Loki Logs

### Generate Traffic First

```bash
for i in $(seq 1 50); do
  curl -s http://localhost:8000 > /dev/null
  curl -s http://localhost:8000/api/ > /dev/null
done
```

### LogQL Queries in Grafana Explore

Navigate to **Grafana → Explore → Select Loki datasource**

```logql
# All container logs
{job="docker"}

# Only notes-app logs
{container_name="notes-app"}

# Errors across all containers
{job="docker"} |= "error"

# HTTP GET requests from the app
{container_name="notes-app"} |= "GET"

# Rate of log lines per container (time series)
sum by (container_name) (rate({job="docker"}[5m]))
```

### Check Promtail Targets

```bash
curl -s http://localhost:9080/targets | head -30
```

> **Screenshot placeholder:** Grafana Explore showing log lines from `notes-app` container with timestamp, labels, and log content visible.

---

## Production Overview Dashboard

Dashboard saved as: **"Production Overview — Observability Stack"**

Settings:
- Time range: Last 30 minutes
- Auto-refresh: 10s

### Row 1 — System Health (Node Exporter)

| Panel          | Type        | Query                                                                                               |
|----------------|-------------|-----------------------------------------------------------------------------------------------------|
| CPU Usage      | Gauge       | `100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`                                |
| Memory Usage   | Gauge       | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100`                         |
| Disk Usage     | Gauge       | `(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100` |
| Targets Up     | Stat        | `sum(up) / count(up)`                                                                              |

### Row 2 — Container Metrics (cAdvisor)

| Panel            | Type        | Query                                                                            | Legend       |
|------------------|-------------|----------------------------------------------------------------------------------|--------------|
| Container CPU    | Time series | `rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100`                  | `{{name}}`   |
| Container Memory | Bar chart   | `container_memory_usage_bytes{name!=""} / 1024 / 1024`                         | `{{name}}`   |
| Container Count  | Stat        | `count(container_last_seen{name!=""})`                                          | —            |

### Row 3 — Application Logs (Loki datasource)

| Panel       | Type        | Query                                                                 |
|-------------|-------------|-----------------------------------------------------------------------|
| App Logs    | Logs        | `{container_name="notes-app"}`                                       |
| Error Rate  | Time series | `sum(rate({job="docker"} \|= "error" [5m]))`                        |
| Log Volume  | Time series | `sum by (container_name) (rate({job="docker"}[5m]))`                |

### Row 4 — Service Overview

| Panel                    | Type        | Query                                                          |
|--------------------------|-------------|----------------------------------------------------------------|
| Scrape Duration (p99)    | Time series | `prometheus_target_interval_length_seconds{quantile="0.99"}`  |
| OTEL Metrics Received    | Stat        | `otelcol_receiver_accepted_metric_points`                      |

> **Screenshot placeholder:** Production Overview dashboard showing all 4 rows with populated panels, green gauges, and log stream visible.

---

## OTEL Trace Validation

### Send a Test Trace (Two-Span)

```bash
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{
          "key": "service.name",
          "value": { "stringValue": "notes-app" }
        }]
      },
      "scopeSpans": [{
        "spans": [{
          "traceId": "aaaabbbbccccdddd1111222233334444",
          "spanId": "1111222233334444",
          "name": "GET /api/notes",
          "kind": 2,
          "startTimeUnixNano": "1700000000000000000",
          "endTimeUnixNano": "1700000000150000000",
          "attributes": [
            { "key": "http.method",      "value": { "stringValue": "GET" } },
            { "key": "http.route",       "value": { "stringValue": "/api/notes" } },
            { "key": "http.status_code", "value": { "intValue": "200" } }
          ],
          "status": { "code": 1 }
        },
        {
          "traceId": "aaaabbbbccccdddd1111222233334444",
          "spanId": "5555666677778888",
          "parentSpanId": "1111222233334444",
          "name": "SELECT notes FROM database",
          "kind": 3,
          "startTimeUnixNano": "1700000000020000000",
          "endTimeUnixNano": "1700000000120000000",
          "attributes": [
            { "key": "db.system",    "value": { "stringValue": "sqlite" } },
            { "key": "db.statement", "value": { "stringValue": "SELECT * FROM notes" } }
          ]
        }]
      }]
    }]
  }'
```

### Verify in Collector Logs

```bash
docker logs otel-collector 2>&1 | grep -A 20 "GET /api/notes"
```

Expected output shows:
- **Parent span:** `GET /api/notes` (spanId: `1111222233334444`, duration: 150ms)
- **Child span:** `SELECT notes FROM database` (spanId: `5555666677778888`, parentSpanId: `1111222233334444`, duration: 100ms)
- Both spans linked under `traceId: aaaabbbbccccdddd1111222233334444`
- Resource attribute: `service.name = notes-app`

> **Screenshot placeholder:** `docker logs otel-collector` terminal output showing the two spans, their parent-child relationship, and timing data.

---

## Config Comparison: My Versions vs Reference Repo

### prometheus.yml

| Aspect                   | My Version (Days 73–74)             | Reference Repo                         |
|--------------------------|-------------------------------------|----------------------------------------|
| Scrape interval          | `15s` global                        | `15s` global                           |
| Jobs defined             | prometheus, node-exporter, cadvisor | Same + otel-collector job added        |
| otel-collector job       | Not present                         | Present (`otel-collector:8888`)        |
| alerting rules file      | Inline rules                        | External `alert-rules.yml` file        |
| Target discovery         | Static configs                      | Static configs                         |

**Key difference:** The reference repo adds a scrape job for the OTEL Collector's own metrics endpoint (`:8888`), which enables monitoring the collector itself. The alert rules are cleanly separated into their own file.

---

### loki-config.yml

| Aspect              | My Version (Day 75)           | Reference Repo                      |
|---------------------|-------------------------------|-------------------------------------|
| Storage backend     | Filesystem (local)            | Filesystem (local)                  |
| Schema version      | v11                           | v12 (newer chunk format)            |
| Chunk store         | `boltdb-shipper`              | `tsdb` (more efficient)             |
| Retention period    | Not configured                | `retention_period: 7d`              |
| Auth                | `auth_enabled: false`         | `auth_enabled: false`               |

**Key difference:** The reference repo uses the newer `tsdb` index type and `v12` schema, which has better performance and compaction. It also sets explicit retention to avoid unbounded disk growth.

---

### promtail-config.yml

| Aspect              | My Version (Day 75)           | Reference Repo                      |
|---------------------|-------------------------------|-------------------------------------|
| Log discovery       | Static file paths             | Docker socket discovery             |
| Labels added        | `job`, `host`                 | `job`, `container_name`, `image`    |
| Pipeline stages     | Basic regex                   | Docker log format stage + JSON      |
| Targets             | `/var/log/*.log`              | Docker containers via socket        |

**Key difference:** The reference repo uses Docker socket discovery (`/var/run/docker.sock`) which automatically picks up all containers without needing to know paths. It also enriches logs with `container_name` and `image` labels automatically.

---

### otel-collector-config.yml

| Aspect              | My Version (Day 76)           | Reference Repo                      |
|---------------------|-------------------------------|-------------------------------------|
| Receivers           | `otlp` (gRPC + HTTP)          | `otlp` (gRPC + HTTP)                |
| Processors          | `batch`                       | `batch` + `memory_limiter`          |
| Exporters           | `debug`                       | `debug` + `prometheus`              |
| Metrics endpoint    | Not exposed                   | `:8888` (scraped by Prometheus)     |
| Traces backend      | Debug output only             | Debug output only                   |

**Key difference:** The reference repo adds `memory_limiter` processor to prevent OOM conditions, and a `prometheus` exporter so the collector's own metrics can be scraped and monitored. This is critical for production self-observability.

---

### grafana/provisioning/datasources/datasources.yml

| Aspect              | My Version (Day 74)           | Reference Repo                      |
|---------------------|-------------------------------|-------------------------------------|
| Datasources         | Prometheus only               | Prometheus + Loki                   |
| Loki URL            | Not configured                | `http://loki:3100`                  |
| Default datasource  | Prometheus                    | Prometheus                          |
| Provisioning method | Manual UI                     | YAML auto-provisioning              |

**Key difference:** The reference repo auto-provisions both datasources so Grafana comes up fully configured without any manual steps. This is the correct approach for reproducible deployments.

---

### docker-compose.yml

| Aspect              | My Version (Days 73–76)       | Reference Repo                      |
|---------------------|-------------------------------|-------------------------------------|
| Services count      | 6–7 (added one per day)       | 8 (all at once, wired correctly)    |
| Networks            | Default bridge                | Named `monitoring` network          |
| Volumes             | Ad hoc bind mounts            | Named volumes + bind mounts         |
| Restart policy      | Not set                       | `unless-stopped` on all services    |
| Notes App           | Not included                  | Full Django app with Dockerfile     |
| Dependencies        | Partial                       | `depends_on` for correct order      |

**Key difference:** The reference repo uses a dedicated `monitoring` network so all services resolve each other by container name. The `restart: unless-stopped` policy ensures services survive Docker daemon restarts.

---

## All Configuration Files

### docker-compose.yml

```yaml
version: "3.8"

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
  loki_data:

services:

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - ./alert-rules.yml:/etc/prometheus/alert-rules.yml
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.enable-lifecycle"
    ports:
      - "9090:9090"
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    ports:
      - "9100:9100"
    networks:
      - monitoring

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    restart: unless-stopped
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
      - loki
    networks:
      - monitoring

  loki:
    image: grafana/loki:latest
    container_name: loki
    restart: unless-stopped
    volumes:
      - ./loki/loki-config.yml:/etc/loki/config.yml
      - loki_data:/loki
    command: -config.file=/etc/loki/config.yml
    ports:
      - "3100:3100"
    networks:
      - monitoring

  promtail:
    image: grafana/promtail:latest
    container_name: promtail
    restart: unless-stopped
    volumes:
      - ./promtail/promtail-config.yml:/etc/promtail/config.yml
      - /var/run/docker.sock:/var/run/docker.sock
      - /var/log:/var/log:ro
    command: -config.file=/etc/promtail/config.yml
    ports:
      - "9080:9080"
    depends_on:
      - loki
    networks:
      - monitoring

  otel-collector:
    image: otel/opentelemetry-collector-contrib:latest
    container_name: otel-collector
    restart: unless-stopped
    volumes:
      - ./otel-collector/otel-collector-config.yml:/etc/otel-collector-config.yml
    command: ["--config=/etc/otel-collector-config.yml"]
    ports:
      - "4317:4317"   # OTLP gRPC
      - "4318:4318"   # OTLP HTTP
      - "8888:8888"   # Collector metrics
    networks:
      - monitoring

  notes-app:
    build: ./notes-app
    container_name: notes-app
    restart: unless-stopped
    ports:
      - "8000:8000"
    networks:
      - monitoring
```

---

### prometheus.yml

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
      - targets: ["otel-collector:8888"]
```

---

### alert-rules.yml

```yaml
groups:
  - name: system_alerts
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "CPU usage is above 85% for more than 2 minutes."

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage detected"
          description: "Memory usage is above 85% for more than 2 minutes."

      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Prometheus target is down"
          description: "Target {{ $labels.job }} / {{ $labels.instance }} has been down for more than 1 minute."

      - alert: HighDiskUsage
        expr: (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk usage critical"
          description: "Disk usage on / is above 90%."
```

---

### loki/loki-config.yml

```yaml
auth_enabled: false

server:
  http_listen_port: 3100

common:
  path_prefix: /loki
  storage:
    filesystem:
      chunks_directory: /loki/chunks
      rules_directory: /loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2024-01-01
      store: tsdb
      object_store: filesystem
      schema: v12
      index:
        prefix: index_
        period: 24h

limits_config:
  retention_period: 168h   # 7 days

compactor:
  working_directory: /loki/boltdb-shipper-compactor
  retention_enabled: true
```

---

### promtail/promtail-config.yml

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
        refresh_interval: 5s
    relabel_configs:
      - source_labels: ["__meta_docker_container_name"]
        regex: "/(.*)"
        target_label: container_name
      - source_labels: ["__meta_docker_container_image"]
        target_label: image
      - source_labels: ["__meta_docker_container_log_stream"]
        target_label: stream
      - target_label: job
        replacement: docker
    pipeline_stages:
      - docker: {}
```

---

### otel-collector/otel-collector-config.yml

```yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  memory_limiter:
    check_interval: 5s
    limit_mib: 512
    spike_limit_mib: 128
  batch:
    timeout: 10s

exporters:
  debug:
    verbosity: detailed
  prometheus:
    endpoint: "0.0.0.0:8888"

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, batch]
      exporters: [debug, prometheus]
```

---

### grafana/provisioning/datasources/datasources.yml

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

### grafana/provisioning/dashboards/dashboards.yml

```yaml
apiVersion: 1

providers:
  - name: default
    orgId: 1
    folder: ""
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    options:
      path: /etc/grafana/provisioning/dashboards
```

---

## Production Readiness Additions

| Addition                          | Why It Matters                                                                 |
|-----------------------------------|--------------------------------------------------------------------------------|
| **Alertmanager**                  | Route alerts to Slack, PagerDuty, or email. The current setup fires rules but has nowhere to send them |
| **Grafana Tempo**                 | Replace the debug exporter with a real trace backend; enables trace search and service maps |
| **HTTPS/TLS everywhere**          | All endpoints (Grafana, Prometheus, Loki) are currently plaintext HTTP — unacceptable in prod |
| **Grafana + Prometheus auth**     | Default `admin/admin` and open Prometheus are security vulnerabilities        |
| **Log retention policy**          | Without limits, Loki will fill the disk; enforce `retention_period` and volume quotas |
| **Prometheus remote_write**       | Push metrics to Thanos or Mimir for long-term storage beyond local TSDB retention |
| **High availability replicas**    | Single Prometheus and Loki are SPOFs; use Prometheus federation or Thanos for HA |
| **Resource limits in Compose**    | Without `mem_limit` / `cpus`, any container can starve the host              |
| **Structured logging in the app** | Django logs are unstructured; adding JSON logging unlocks richer LogQL queries |
| **Exemplars**                     | Link Prometheus metrics to Tempo traces for click-through from metric spikes to traces |
| **Dashboard as code (GitOps)**    | Export all Grafana dashboards as JSON and version them in Git                  |

---

## Key Takeaways — 5-Day Observability Block

### Day-by-Day Learning Map

| Day | What Was Built                                     | Core Concept                                                |
|-----|----------------------------------------------------|-------------------------------------------------------------|
| 73  | Prometheus, PromQL, scrape configs, basic metrics  | The pull model; TSDB; labels as the primary query dimension |
| 74  | Node Exporter, cAdvisor, Grafana dashboards        | Infrastructure metrics; container-level visibility; panels  |
| 75  | Loki, Promtail, LogQL, log-metric correlation      | Logs are just labeled streams; push vs pull for logs        |
| 76  | OTEL Collector, traces, alerting rules             | The three pillars unified; distributed trace context        |
| 77  | Full stack integration, unified dashboard          | How all signals connect; architecture thinking; handoffs    |

### The Three Pillars — Unified

```
METRICS  → "Is something wrong?"         → Prometheus / Grafana
LOGS     → "What happened?"              → Loki / Promtail / Grafana Explore  
TRACES   → "Where did the time go?"      → OTEL Collector / Tempo (future)
```

All three are most powerful when correlated:
1. A Grafana alert fires on a CPU spike (metric)
2. Drill into Loki logs at that timestamp to find error messages
3. Use a trace ID from the log line to find the exact slow span in Tempo

### Self-Hosted vs Managed Observability

| Factor              | This Stack (Self-Hosted)            | Datadog / New Relic / CloudWatch     |
|---------------------|-------------------------------------|--------------------------------------|
| Cost                | Infra cost only (low at small scale)| Per-host + per-GB pricing (scales up) |
| Setup time          | Hours to days                       | Minutes (agent install)              |
| Control             | Full; every config is yours         | Limited; vendor controls the backend |
| Customisation       | Unlimited                           | Within vendor's UI/API               |
| Long-term storage   | Requires Thanos/Mimir setup         | Included (with cost)                 |
| Trace backend       | Requires Tempo setup                | Included                             |
| Team overhead       | Your team maintains it              | Vendor maintains it                  |
| Best for            | Cost-sensitive, data-sensitive orgs | Speed-first teams, large enterprises |

### Architecture Insight

The most important thing this week was understanding **why each component exists**:

- **Prometheus** exists because metrics need to be queryable at arbitrary time granularity — you can't do that with just logs.
- **Loki** exists because storing full log text in Prometheus would be impossibly expensive; Loki indexes only labels, not content.
- **Promtail** exists because containers don't know about Loki — something external needs to watch Docker's stdout and push it forward.
- **cAdvisor** exists because Docker doesn't expose per-container metrics in Prometheus format natively.
- **OTEL Collector** exists to decouple applications from specific backends — apps speak OTLP, and the collector handles routing to Jaeger, Tempo, Datadog, or anything else.

---

## Cleanup

```bash
# Stop and remove all containers + named volumes
docker compose down -v
```

> ⚠️ The `-v` flag deletes all Prometheus, Grafana, and Loki stored data. Only run this when done exploring.

---

*Day 77 of 90 — Observability block complete.*  
*Reference repository: [github.com/LondheShubham153/observability-for-devops](https://github.com/LondheShubham153/observability-for-devops)*
