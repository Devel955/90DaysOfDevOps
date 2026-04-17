# Day 74 — Node Exporter, cAdvisor, and Grafana Dashboards

## Overview

This document covers expanding the Prometheus observability stack with Node Exporter for host metrics, cAdvisor for container metrics, and Grafana for visualization.

---

## Updated `docker-compose.yml`

```yaml
version: "3.8"

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
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    restart: unless-stopped

  grafana:
    image: grafana/grafana-enterprise:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    restart: unless-stopped

  notes-app:
    image: your-notes-app:latest
    container_name: notes-app
    ports:
      - "8000:8000"
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
```

---

## Updated `prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

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
```

---

## Node Exporter vs cAdvisor — When to Use Each

### Node Exporter

Node Exporter collects metrics about the **host machine** — the physical or virtual server running your containers.

**What it monitors:**
- CPU usage per core and per mode (idle, user, system, iowait)
- RAM — total, available, used, cached, buffered
- Disk I/O — read/write throughput, IOPS
- Filesystem usage — bytes used vs available per mount point
- Network interfaces — bytes in/out, packets, errors, drops
- System load average, file descriptors, kernel statistics

**When to use it:**
- You need to know if the host itself is under pressure (high CPU steal, running out of disk space, saturated NIC)
- Capacity planning — understanding overall resource headroom on the machine
- Diagnosing host-level issues that affect all containers, such as I/O wait caused by a busy disk

**Key metric prefix:** `node_`

---

### cAdvisor (Container Advisor)

cAdvisor collects metrics about **individual Docker containers** running on the host.

**What it monitors:**
- Per-container CPU usage (throttling, usage seconds)
- Per-container memory (RSS, cache, working set, limits)
- Per-container network (bytes in/out per interface)
- Per-container filesystem (reads, writes, IO wait)
- Container lifecycle metadata (image, name, labels)

**When to use it:**
- You need to identify which specific container is consuming resources
- Debugging a memory-leaking service inside a container
- Enforcing and monitoring resource limits set in Docker or Kubernetes
- Understanding container-level trends over time

**Key metric prefix:** `container_`

---

### Summary Table

| Concern | Use |
|---|---|
| Is the host running out of memory? | Node Exporter |
| Which container is eating all the RAM? | cAdvisor |
| Is the disk filling up? | Node Exporter |
| How much CPU is the `notes-app` container using? | cAdvisor |
| What is the total network throughput on `eth0`? | Node Exporter |
| Which container is sending the most traffic? | cAdvisor |

In practice, both are deployed together. Node Exporter gives you the big picture; cAdvisor drills into the per-container detail.

---

## PromQL Queries

### Host Metrics (Node Exporter)

```promql
# CPU idle percentage per core
node_cpu_seconds_total{mode="idle"}

# Total vs available memory (bytes)
node_memory_MemTotal_bytes
node_memory_MemAvailable_bytes

# Memory usage as a percentage
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100

# Disk usage percentage per filesystem
(1 - node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100

# Root filesystem usage specifically
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100

# Network bytes received per second (5m rate)
rate(node_network_receive_bytes_total[5m])

# Overall CPU usage (all cores averaged)
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

### Container Metrics (cAdvisor)

```promql
# CPU usage per container (fraction of a core, 5m rate)
rate(container_cpu_usage_seconds_total{name!=""}[5m])

# CPU usage as a percentage
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100

# Memory usage per container in bytes
container_memory_usage_bytes{name!=""}

# Memory usage per container in MB
container_memory_usage_bytes{name!=""} / 1024 / 1024

# Network bytes received per container
rate(container_network_receive_bytes_total{name!=""}[5m])

# Top 3 containers by memory consumption
topk(3, container_memory_usage_bytes{name!=""})
```

> The `{name!=""}` filter removes system-level aggregation entries and shows only named containers.

---

## Grafana Dashboard: DevOps Observability Overview

### Panel 1 — CPU Usage % (Gauge)

```promql
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

- Visualization: Gauge
- Thresholds: green < 60, yellow < 80, red ≥ 80

### Panel 2 — Memory Usage % (Gauge)

```promql
(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100
```

- Visualization: Gauge

### Panel 3 — Container CPU Usage (Time Series)

```promql
rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
```

- Visualization: Time series
- Legend: `{{name}}`

### Panel 4 — Container Memory (Bar Chart)

```promql
container_memory_usage_bytes{name!=""} / 1024 / 1024
```

- Visualization: Bar chart
- Legend: `{{name}}`
- Unit: MB

### Panel 5 — Disk Usage % (Stat)

```promql
(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100
```

- Visualization: Stat

---

## Datasource Provisioning via YAML

### File: `grafana/provisioning/datasources/datasources.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### Why YAML Provisioning Is Better Than Manual UI Configuration

**Repeatability:** The datasource configuration is committed to version control alongside the rest of the stack. Anyone who clones the repository and runs `docker compose up` gets Prometheus wired into Grafana automatically — no manual steps.

**Idempotency:** Restarting Grafana or rebuilding the container does not wipe the datasource. The YAML is loaded fresh on every startup, so the state is always consistent with what is in the file.

**Team consistency:** When multiple people or environments (dev, staging, production) use the same provisioning files, they all connect to the same datasource configuration. There is no drift from someone making a UI change and forgetting to document it.

**Auditability:** Changes to datasource config appear as Git commits with author, timestamp, and diff — much clearer than "someone clicked Save in Grafana last Tuesday."

**CI/CD compatibility:** Provisioned configuration can be validated and tested in a pipeline. Manually configured datasources live only in Grafana's SQLite database and are not visible to automation tools.

The same principle applies to dashboards: provisioning them as JSON files means you version-control your dashboards exactly like application code.

---

## Community Dashboards

| Dashboard ID | Name | Description |
|---|---|---|
| 1860 | Node Exporter Full | Comprehensive host metrics — CPU, memory, disk, network, per-core stats |
| 193 | Docker monitoring via cAdvisor | Per-container CPU, memory, and network visualizations |

To import: **Dashboards → New → Import → enter ID → select Prometheus datasource → Import**

---

## Stack Verification

```bash
# Check all services are running
docker compose ps

# Verify Node Exporter is exposing metrics
curl http://localhost:9100/metrics | head -20

# Verify cAdvisor UI is accessible
open http://localhost:8080

# Verify Grafana is accessible
open http://localhost:3000
```

Prometheus Targets page (`http://localhost:9090/targets`) should show all three scrape targets as **UP**:
- `prometheus` on `localhost:9090`
- `node-exporter` on `node-exporter:9100`
- `cadvisor` on `cadvisor:8080`

---

## Key Networking Note

Grafana's Prometheus datasource URL must be `http://prometheus:9090` — using the **container name**, not `localhost`. All services share a Docker bridge network, so they resolve each other by container name. Using `localhost` from inside the Grafana container would point to Grafana itself, not Prometheus.

---

## References

- [Prometheus Node Exporter](https://github.com/prometheus/node_exporter)
- [cAdvisor](https://github.com/google/cadvisor)
- [Grafana Provisioning Docs](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [Node Exporter Full Dashboard (1860)](https://grafana.com/grafana/dashboards/1860)
- [Reference repo: observability-for-devops](https://github.com/LondheShubham153/observability-for-devops)
