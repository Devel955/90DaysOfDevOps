# Day 57 – Resource Requests, Limits, and Probes

## Overview

This document covers Kubernetes resource management and pod health probes. Resources ensure pods are scheduled correctly and don't starve the node; probes let Kubernetes detect failures and recover automatically.

---

## Task 1: Resource Requests and Limits

### Manifest

```yaml
# resource-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "100m"
        memory: "128Mi"
      limits:
        cpu: "250m"
        memory: "256Mi"
```

### Commands

```bash
kubectl apply -f resource-pod.yaml
kubectl describe pod resource-demo
```

### Key Concepts

| Concept | Explanation |
|---|---|
| **Requests** | Guaranteed minimum. The scheduler uses this to find a node with enough free capacity. |
| **Limits** | Maximum allowed. The kubelet enforces this at runtime. |
| **CPU unit** | 100m = 0.1 core. 1000m = 1 full core. |
| **Memory unit** | Mi = mebibytes (1Mi = 1,048,576 bytes). Gi = gibibytes. |

### QoS Classes

| Class | Condition |
|---|---|
| **Guaranteed** | requests == limits for all containers |
| **Burstable** | requests < limits (at least one container) |
| **BestEffort** | No requests or limits set at all |

**Verify — QoS Class:** Since `requests < limits` in this pod, the QoS class is **Burstable**. You can confirm with:

```bash
kubectl describe pod resource-demo | grep "QoS Class"
# QoS Class: Burstable
```

---

## Task 2: OOMKilled — Exceeding Memory Limits

### Manifest

```yaml
# oom-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom-demo
spec:
  containers:
  - name: stress
    image: polinux/stress
    resources:
      limits:
        memory: "100Mi"
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "200M", "--vm-hang", "1"]
```

### Commands

```bash
kubectl apply -f oom-pod.yaml
kubectl get pod oom-demo -w
kubectl describe pod oom-demo
```

### What Happens

The container tries to allocate 200M of memory but the limit is 100Mi. The kernel OOM killer terminates the process immediately.

```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

### Key Concepts

| Resource | Behavior When Over Limit |
|---|---|
| **CPU** | Throttled — slowed down but not killed |
| **Memory** | OOMKilled — process is killed with SIGKILL |

**Verify — Exit Code:** An OOMKilled container exits with code **137** (128 + SIGKILL signal 9).

---

## Task 3: Pending Pod — Requesting Too Much

### Manifest

```yaml
# pending-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pending-demo
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: "100"
        memory: "128Gi"
```

### Commands

```bash
kubectl apply -f pending-pod.yaml
kubectl get pod pending-demo
# STATUS: Pending (forever)

kubectl describe pod pending-demo
# Look at Events section at the bottom
```

### Expected Event Message

```
Events:
  Warning  FailedScheduling  ...  0/1 nodes are available: 
           1 Insufficient cpu, 1 Insufficient memory.
```

**Verify — Scheduler Message:** The scheduler reports exactly which resources are insufficient on each node. The pod stays `Pending` indefinitely — no node can satisfy the request.

---

## Task 4: Liveness Probe

A liveness probe detects stuck or broken containers. **Failure triggers a container restart.**

### Manifest

```yaml
# liveness-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: liveness-demo
spec:
  containers:
  - name: busybox
    image: busybox
    command:
    - /bin/sh
    - -c
    - |
      touch /tmp/healthy
      sleep 30
      rm /tmp/healthy
      sleep 600
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/healthy
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
```

### Commands

```bash
kubectl apply -f liveness-pod.yaml
kubectl get pod liveness-demo -w
```

### What Happens

1. Container starts and creates `/tmp/healthy`
2. Liveness probe checks `cat /tmp/healthy` every 5 seconds — passes
3. After 30 seconds, the file is deleted
4. 3 consecutive failures × 5 seconds = ~15 seconds later, Kubernetes restarts the container
5. `RESTARTS` column increments in `kubectl get pod`

**Verify — Restart Count:** After roughly 45–50 seconds, the restart count should be **1** (and will increment each cycle).

```bash
kubectl describe pod liveness-demo | grep -A5 "Liveness"
```

---

## Task 5: Readiness Probe

A readiness probe controls whether a Pod receives traffic. **Failure removes the Pod from Service endpoints but does NOT restart the container.**

### Manifest

```yaml
# readiness-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: readiness-demo
  labels:
    app: readiness-demo
spec:
  containers:
  - name: nginx
    image: nginx
    readinessProbe:
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5
      periodSeconds: 5
```

### Commands

```bash
kubectl apply -f readiness-pod.yaml
kubectl expose pod readiness-demo --port=80 --name=readiness-svc

# Pod shows 1/1 READY, endpoint is registered
kubectl get endpoints readiness-svc

# Break the probe — remove the index page
kubectl exec readiness-demo -- rm /usr/share/nginx/html/index.html

# Wait ~15 seconds
kubectl get pod readiness-demo
# Shows: 0/1 READY

kubectl get endpoints readiness-svc
# Endpoint list is now empty
```

**Verify — Was the container restarted?** **No.** The container keeps running, but traffic is cut off. `RESTARTS` stays at 0. This is the key distinction: readiness controls traffic routing, liveness controls container lifecycle.

---

## Task 6: Startup Probe

A startup probe gives slow-starting containers time to initialize. **While the startup probe is running, liveness and readiness probes are disabled.** If the startup probe exceeds its budget (`periodSeconds × failureThreshold`), the container is killed.

### Manifest

```yaml
# startup-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: startup-demo
spec:
  containers:
  - name: slow-start
    image: busybox
    command:
    - /bin/sh
    - -c
    - |
      sleep 20
      touch /tmp/started
      sleep 600
    startupProbe:
      exec:
        command:
        - cat
        - /tmp/started
      periodSeconds: 5
      failureThreshold: 12    # Budget: 5 × 12 = 60 seconds
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/started
      periodSeconds: 10
      failureThreshold: 3
```

### Commands

```bash
kubectl apply -f startup-pod.yaml
kubectl get pod startup-demo -w
# Pod transitions: ContainerCreating → Running → 1/1
```

**Verify — What if `failureThreshold` were 2?** With `periodSeconds: 5` and `failureThreshold: 2`, the startup budget is only **10 seconds**. Since the container takes 20 seconds to create `/tmp/started`, the startup probe would fail its limit, and Kubernetes would **kill and restart the container in a CrashLoopBackOff** — it would never successfully start.

---

## Task 7: Clean Up

```bash
kubectl delete pod resource-demo oom-demo pending-demo liveness-demo readiness-demo startup-demo
kubectl delete svc readiness-svc
```

---

## Summary Reference

### Requests vs Limits

| | Requests | Limits |
|---|---|---|
| **Purpose** | Scheduling guarantee | Runtime cap |
| **Used by** | Scheduler (placement) | Kubelet (enforcement) |
| **CPU behavior** | Minimum reserved | Throttled if exceeded |
| **Memory behavior** | Minimum reserved | OOMKilled if exceeded |

### What Happens When Limits Are Exceeded

| Resource | Result |
|---|---|
| CPU | Process is **throttled** (slowed down, not killed) |
| Memory | Process is **OOMKilled** (exit code 137 = 128 + SIGKILL) |

### Probe Comparison

| Probe | Failure Action | Use Case |
|---|---|---|
| **Liveness** | Restarts the container | Detect deadlocks, stuck processes |
| **Readiness** | Removes from Service endpoints | Control traffic during startup or degradation |
| **Startup** | Kills container if budget exceeded | Protect slow-starting apps from liveness killing them too early |

### Probe Types

| Type | Example |
|---|---|
| `exec` | `command: [cat, /tmp/healthy]` |
| `httpGet` | `path: /, port: 80` |
| `tcpSocket` | `port: 3306` |

### QoS Quick Reference

```
requests == limits  →  Guaranteed
requests < limits   →  Burstable
no requests/limits  →  BestEffort
```

### Exit Codes

| Code | Meaning |
|---|---|
| 0 | Success |
| 137 | OOMKilled (128 + SIGKILL 9) |
| 143 | SIGTERM (graceful shutdown, 128 + 15) |
