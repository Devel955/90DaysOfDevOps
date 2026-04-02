# Day 58 – Metrics Server and Horizontal Pod Autoscaler (HPA)

## Overview

Today's challenge covers two tightly coupled Kubernetes features:

1. **Metrics Server** – a lightweight, in-cluster component that collects real-time CPU and memory usage from kubelets and exposes it via the Kubernetes Metrics API.
2. **Horizontal Pod Autoscaler (HPA)** – a controller that automatically adjusts the number of pod replicas in a Deployment (or StatefulSet/ReplicaSet) based on observed resource usage.

Neither works without the other: HPA queries the Metrics API every 15 seconds to decide whether to scale up or down, so without the Metrics Server, HPA targets will always show `<unknown>`.

---

## Task 1: Install the Metrics Server

### Check if already running

```bash
kubectl get pods -n kube-system | grep metrics-server
```

### Install on Minikube

```bash
minikube addons enable metrics-server
```

### Install on Kind / kubeadm

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

For local clusters (self-signed certs), patch the Deployment to add `--kubelet-insecure-tls`:

```bash
kubectl patch deployment metrics-server -n kube-system \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

> ⚠️ **Never use `--kubelet-insecure-tls` in production.** It disables TLS verification between the Metrics Server and kubelets.

### Verify (wait ~60 seconds after install)

```bash
kubectl top nodes
kubectl top pods -A
```

**Sample output – `kubectl top nodes`:**

```
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
minikube   243m         6%     1121Mi          14%
```

**Sample output – `kubectl top pods -A`:**

```
NAMESPACE     NAME                               CPU(cores)   MEMORY(bytes)
kube-system   coredns-787d4945fb-xkl7v           3m           15Mi
kube-system   etcd-minikube                      22m          52Mi
kube-system   kube-apiserver-minikube            52m          249Mi
kube-system   metrics-server-6588d95b98-t9t2s    4m           18Mi
```

> **Current node CPU/memory usage:** ~243m CPU (6%) and ~1121Mi memory (14%) on a typical Minikube instance. Values will vary by workload.

---

## Task 2: Explore `kubectl top`

```bash
# Node resource usage
kubectl top nodes

# All pod usage
kubectl top pods -A

# Sort by CPU to find the heaviest consumer
kubectl top pods -A --sort-by=cpu
```

### Key Concepts

| Command | What it shows |
|---|---|
| `kubectl top` | **Actual real-time usage** (from Metrics Server) |
| `kubectl describe pod` | **Configured requests/limits** (from the manifest) |

These are **different things**. A pod with `requests.cpu: 200m` might only be using `5m` at idle, or spike to `800m` under load.

- Metrics Server polls kubelets every **15 seconds**.
- Data is held in memory only — it is not persisted.

> **Highest CPU pod:** Typically `kube-apiserver` on a quiet cluster. Under load, the `php-apache` deployment will top the list.

---

## Task 3: Create a Deployment with CPU Requests

```yaml
# php-apache-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      app: php-apache
  replicas: 1
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
          limits:
            cpu: 500m
```

```bash
kubectl apply -f php-apache-deployment.yaml
kubectl expose deployment php-apache --port=80
```

> **Why `resources.requests.cpu` is mandatory for HPA:**
> HPA calculates utilization as a percentage of the *requested* CPU, not the node's total CPU. Without a request, the denominator is undefined and HPA cannot compute a ratio — `TARGETS` will show `<unknown>` forever.

**Verify idle CPU usage:**

```bash
kubectl top pods -l app=php-apache
```

```
NAME                          CPU(cores)   MEMORY(bytes)
php-apache-7d8b9f6c9d-xm4pb   1m           10Mi
```

At idle, the pod uses ~1m CPU (0.5% of its 200m request).

---

## Task 4: Create an HPA (Imperative)

```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
```

```bash
kubectl get hpa
kubectl describe hpa php-apache
```

**Sample `kubectl get hpa` output (after metrics arrive):**

```
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   1%/50%    1         10        1          45s
```

- `TARGETS` format: `currentUtilization/targetUtilization`
- Initially shows `<unknown>/50%` — wait 30 seconds for the first metrics scrape.

**How the target percentage works:**

The HPA watches average CPU utilization across all pods in the Deployment. When the average exceeds 50% of `requests.cpu` (i.e., > 100m), it adds replicas. When it drops below 50%, it removes them.

---

## Task 5: Generate Load and Watch Autoscaling

### Start load generator

```bash
kubectl run load-generator \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"
```

### Watch HPA in real time

```bash
kubectl get hpa php-apache --watch
```

**Observed scaling progression:**

```
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   1%/50%     1         10        1          2m
php-apache   Deployment/php-apache   248%/50%   1         10        1          2m15s
php-apache   Deployment/php-apache   248%/50%   1         10        5          2m30s
php-apache   Deployment/php-apache   90%/50%    1         10        5          2m45s
php-apache   Deployment/php-apache   62%/50%    1         10        7          3m
php-apache   Deployment/php-apache   51%/50%    1         10        7          3m30s
```

**Peak replicas under load: 7** (will vary; commonly 5–10 depending on hardware).

### Stop the load

```bash
kubectl delete pod load-generator
```

Scale-down is intentionally slow due to the **5-minute stabilization window** — this prevents thrashing when traffic is bursty. You don't need to wait for it.

---

## Task 6: Create an HPA from YAML (Declarative – `autoscaling/v2`)

### Delete the imperative HPA first

```bash
kubectl delete hpa php-apache
```

### HPA manifest with behavior control

```yaml
# php-apache-hpa-v2.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0        # Scale up immediately
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15                  # Double replicas every 15s if needed
    scaleDown:
      stabilizationWindowSeconds: 300      # Wait 5 min before scaling down
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60                  # Remove at most 10% of pods per minute
```

```bash
kubectl apply -f php-apache-hpa-v2.yaml
kubectl describe hpa php-apache
```

### What the `behavior` section controls

| Field | Effect |
|---|---|
| `scaleUp.stabilizationWindowSeconds: 0` | No waiting before scaling up — reacts to load immediately |
| `scaleUp.policies[0]` | Allows up to 100% more pods every 15 seconds (aggressive ramp-up) |
| `scaleDown.stabilizationWindowSeconds: 300` | Waits 5 minutes of sustained low load before shrinking |
| `scaleDown.policies[0]` | Removes at most 10% of current replicas per 60-second window (gradual ramp-down) |

This is the canonical production pattern: **scale up aggressively, scale down conservatively**.

---

## Task 7: Clean Up

```bash
kubectl delete hpa php-apache
kubectl delete service php-apache
kubectl delete deployment php-apache
kubectl delete pod load-generator --ignore-not-found
# Leave Metrics Server running
```

---

## Key Concepts Summary

### What is the Metrics Server?

The Metrics Server is a cluster-level component that aggregates real-time CPU and memory usage from all node kubelets and exposes them via the `metrics.k8s.io` API. It is the data source for both `kubectl top` and HPA.

- Scrapes kubelets every **15 seconds**
- Stores data **in memory only** (not Prometheus — use Prometheus Adapter for that)
- Required by HPA to function

### How HPA Calculates Desired Replicas

```
desiredReplicas = ceil(currentReplicas × (currentUsage / targetUsage))
```

**Example:** 2 pods running at 200m CPU each, target is 100m (50% of 200m request):

```
desiredReplicas = ceil(2 × (200m / 100m)) = ceil(4) = 4
```

HPA will scale to 4 replicas to bring average usage back to the target.

### `autoscaling/v1` vs `autoscaling/v2`

| Feature | `autoscaling/v1` | `autoscaling/v2` |
|---|---|---|
| Metrics supported | CPU only | CPU, Memory, custom, external |
| Behavior control | ❌ Not supported | ✅ Full `behavior` block |
| Multiple metrics | ❌ | ✅ |
| Status conditions | Basic | Rich conditions |
| Recommended | Legacy | **Use this** |

Always use `autoscaling/v2` for new workloads. It is a strict superset of v1.

---

## Common Pitfalls

| Problem | Cause | Fix |
|---|---|---|
| `TARGETS` shows `<unknown>` | Missing `resources.requests.cpu` | Add CPU request to the container spec |
| HPA not scaling | Metrics Server not installed | `kubectl get pods -n kube-system \| grep metrics` |
| Scale-down too slow | Expected behavior (stabilization window) | Tune `scaleDown.stabilizationWindowSeconds` |
| `kubectl top` returns "not found" | Metrics Server not ready | Wait 60s after install |

---

*Day 58 of 100 Days of Kubernetes*
