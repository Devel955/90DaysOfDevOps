# Day 50 – Kubernetes Architecture and Cluster Setup

## Task 1: The Kubernetes Story (From Memory, Then Verified)

Kubernetes was created to solve the problem of **orchestrating containers at scale** — something Docker alone cannot do. Docker is excellent for building and running individual containers, but when you need to manage hundreds or thousands of containers across multiple machines (handling failures, scaling, load balancing, rolling updates), you need an orchestrator.

**Kubernetes was created by Google**, inspired by an internal system called **Borg** (and later Omega), which Google had been using for over a decade to manage billions of containers per week. Google open-sourced Kubernetes in 2014 and donated it to the **Cloud Native Computing Foundation (CNCF)** in 2016.

The name **"Kubernetes"** comes from the Greek word for **"helmsman"** or **"pilot"** — the person who steers a ship. This is fitting: just as a helmsman guides a ship through rough waters, Kubernetes steers your containerized workloads through the complexity of distributed systems. The abbreviation **"k8s"** comes from the 8 letters between the "K" and the "s".

---

## Task 2: Kubernetes Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                        KUBERNETES CLUSTER                           │
│                                                                     │
│  ┌──────────────────────────── CONTROL PLANE ──────────────────┐   │
│  │                                                              │   │
│  │  ┌─────────────────┐    ┌──────────────────────────────┐   │   │
│  │  │   API Server    │    │           etcd               │   │   │
│  │  │                 │◄──►│  (Distributed Key-Value DB)  │   │   │
│  │  │ "The Front Door"│    │  Stores ALL cluster state    │   │   │
│  │  │ Every kubectl   │    │  Nodes, pods, configs, etc.  │   │   │
│  │  │ command hits    │    └──────────────────────────────┘   │   │
│  │  │ this first      │                                        │   │
│  │  └────────┬────────┘                                        │   │
│  │           │                                                  │   │
│  │  ┌────────▼────────┐    ┌──────────────────────────────┐   │   │
│  │  │    Scheduler    │    │     Controller Manager       │   │   │
│  │  │                 │    │                              │   │   │
│  │  │ Decides WHICH   │    │ Watches cluster state        │   │   │
│  │  │ node a new pod  │    │ Reconciles desired vs        │   │   │
│  │  │ should run on   │    │ actual state                 │   │   │
│  │  │ (based on       │    │ (ReplicaSet, Node, Job,      │   │   │
│  │  │ resources,      │    │  Endpoint controllers)       │   │   │
│  │  │ affinity, etc.) │    └──────────────────────────────┘   │   │
│  │  └─────────────────┘                                        │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                              ▲  ▲                                   │
│                    watches & │  │ schedules pods                    │
│                    reports   │  │                                   │
│  ┌──── WORKER NODE 1 ────────┴──┴───────────────────────────────┐  │
│  │                                                               │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │  │
│  │  │    kubelet   │  │  kube-proxy  │  │ Container Runtime │  │  │
│  │  │              │  │              │  │  (containerd /    │  │  │
│  │  │ Agent on the │  │ Manages iptables│ │   CRI-O)         │  │  │
│  │  │ node. Talks  │  │ / network rules│ │ Actually RUNS the │  │  │
│  │  │ to API server│  │ so pods can  │  │ containers        │  │  │
│  │  │ Ensures pods │  │ communicate  │  │                   │  │  │
│  │  │ are running  │  │ with each    │  │                   │  │  │
│  │  │ as expected  │  │ other & world│  │                   │  │  │
│  │  └──────────────┘  └──────────────┘  └───────────────────┘  │  │
│  │                                                               │  │
│  │  ┌────────────────┐  ┌────────────────┐                      │  │
│  │  │     Pod 1      │  │     Pod 2      │  ...                 │  │
│  │  │  ┌──────────┐  │  │  ┌──────────┐  │                      │  │
│  │  │  │Container │  │  │  │Container │  │                      │  │
│  │  │  └──────────┘  │  │  └──────────┘  │                      │  │
│  │  └────────────────┘  └────────────────┘                      │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  ┌──── WORKER NODE 2 ────────────────────────────────────────────┐  │
│  │   (same structure as Worker Node 1)                           │  │
│  └───────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Summary

| Component | Location | Role |
|-----------|----------|------|
| **API Server** | Control Plane | The single entry point for all cluster communication |
| **etcd** | Control Plane | Distributed database storing all cluster state |
| **Scheduler** | Control Plane | Assigns pods to nodes based on resources/constraints |
| **Controller Manager** | Control Plane | Ensures desired state matches actual state |
| **kubelet** | Worker Node | Node agent — manages pod lifecycle on the node |
| **kube-proxy** | Worker Node | Maintains network rules for pod communication |
| **Container Runtime** | Worker Node | Runs the actual containers (containerd/CRI-O) |

### Request Tracing: What happens when you run `kubectl apply -f pod.yaml`?

1. **kubectl** reads `pod.yaml` and sends an HTTP request to the **API Server**
2. **API Server** authenticates & authorizes the request, validates the manifest, then writes the desired state to **etcd**
3. The **Scheduler** watches etcd (via API Server) for unscheduled pods, picks the best node, and writes the node assignment back to etcd
4. The **kubelet** on the chosen worker node watches the API Server and sees a new pod assigned to it
5. **kubelet** calls the **Container Runtime** (containerd) to pull the image and start the container
6. **kubelet** reports pod status back to the **API Server**, which stores it in **etcd**
7. **kube-proxy** updates networking rules so the pod is reachable

### Failure Scenarios

**If the API Server goes down:**
- No new commands can be executed (`kubectl` stops working)
- **Existing pods keep running** — kubelet and containers continue without the API server
- No new scheduling decisions can be made
- The cluster is "frozen" but not dead

**If a Worker Node goes down:**
- The **Node Controller** (part of Controller Manager) detects the node is unresponsive after ~5 minutes
- Pods on that node are marked as `Unknown` or `Failed`
- If pods were managed by a ReplicaSet/Deployment, the scheduler places replacement pods on healthy nodes
- Standalone pods (not managed) are lost

---

## Task 3: kubectl Installation

```bash
# Verified installation
kubectl version --client
```

**Output:**
```
Client Version: v1.32.x
Kustomize Version: v5.x.x
```
<img width="745" height="103" alt="Screenshot 2026-03-30 051549" src="https://github.com/user-attachments/assets/18832275-5229-43f0-be3f-71c05b02807d" />


---

## Task 4: Local Cluster Setup

### Tool Chosen: **kind** (Kubernetes in Docker)

**Why kind over minikube?**
- kind runs Kubernetes nodes as Docker containers — it's lightweight and fast
- No VM overhead (minikube can spin up a VM by default)
- Already have Docker running, so kind works out-of-the-box
- Better for CI/CD pipelines and reproducible environments
- Multi-node cluster support is simpler with kind

### Setup Commands

```bash
# Install kind (Linux)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# Create the cluster
kind create cluster --name devops-cluster
```

**Output:**
```
Creating cluster "devops-cluster" ...
 ✓ Ensuring node image (kindest/node:v1.32.x) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-devops-cluster"
You can now use your cluster with:

kubectl cluster-info --context kind-devops-cluster
<img width="1920" height="381" alt="Screenshot 2026-03-30 084146" src="https://github.com/user-attachments/assets/d097923d-0d56-4c51-8dea-d6da689625c5" />

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

---

## Task 5: Cluster Exploration

### `kubectl get nodes`

```
NAME                           STATUS   ROLES           AGE   VERSION
devops-cluster-control-plane   Ready    control-plane   2m    v1.32.x
```

### `kubectl cluster-info`

```
Kubernetes control plane is running at https://127.0.0.1:PORT
CoreDNS is running at https://127.0.0.1:PORT/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

### `kubectl get namespaces`

```
NAME              STATUS   AGE
default           Active   3m
kube-node-lease   Active   3m
kube-public       Active   3m
kube-system       Active   3m
```

### `kubectl get pods -n kube-system`

```
NAME                                                   READY   STATUS    RESTARTS   AGE
coredns-668d6bf9bc-7h4xb                               1/1     Running   0          3m
coredns-668d6bf9bc-r9x2w                               1/1     Running   0          3m
etcd-devops-cluster-control-plane                      1/1     Running   0          3m
kube-apiserver-devops-cluster-control-plane            1/1     Running   0          3m
kube-controller-manager-devops-cluster-control-plane   1/1     Running   0          3m
kube-proxy-xxxxx                                       1/1     Running   0          3m
kube-scheduler-devops-cluster-control-plane            1/1     Running   0          3m
kindnet-xxxxx                                          1/1     Running   0          3m
local-path-provisioner-xxxxx                           1/1     Running   0          3m
```

### kube-system Pods → Architecture Mapping

| Pod Name | Architecture Component | What It Does |
|----------|----------------------|--------------|
| `etcd-*` | **etcd** | The cluster's database — stores all state (nodes, pods, secrets, configs) |
| `kube-apiserver-*` | **API Server** | The front door — handles all kubectl and internal API requests |
| `kube-controller-manager-*` | **Controller Manager** | Watches cluster state and reconciles desired vs actual |
| `kube-scheduler-*` | **Scheduler** | Assigns pending pods to appropriate nodes |
| `kube-proxy-*` | **kube-proxy** | Manages network rules for pod-to-pod and pod-to-service communication |
| `coredns-*` | **DNS** | Provides DNS resolution inside the cluster (pods can find services by name) |
| `kindnet-*` | **CNI Plugin** | kind's Container Network Interface — handles pod networking |
| `local-path-provisioner-*` | **Storage** | Provides dynamic local PersistentVolume provisioning for kind |

✅ Every component from the architecture diagram is running as a pod inside the cluster!
<img width="745" height="360" alt="Screenshot 2026-03-30 084329" src="https://github.com/user-attachments/assets/c9250b1c-2bc8-4bf5-a1b6-d9ba9b440611" />

---

## Task 6: Cluster Lifecycle & kubeconfig

### Cluster Lifecycle Commands

```bash
# Delete the cluster
kind delete cluster --name devops-cluster
# Output: Deleting cluster "devops-cluster" ...

# Recreate it
kind create cluster --name devops-cluster
# Output: cluster is up ✓

# Verify it's back
kubectl get nodes
# NAME                           STATUS   ROLES           AGE   VERSION
# devops-cluster-control-plane   Ready    control-plane   30s   v1.32.x
```

### Context Management

```bash
# Check current context
kubectl config current-context
# kind-devops-cluster

# List all contexts
kubectl config get-contexts
# CURRENT   NAME                    CLUSTER                AUTHINFO               NAMESPACE
# *         kind-devops-cluster     kind-devops-cluster    kind-devops-cluster

# View full kubeconfig
kubectl config view
```

### What is a kubeconfig?

A **kubeconfig** is a YAML configuration file that tells `kubectl` **how to connect to a Kubernetes cluster**. It contains:

- **Clusters**: The API server address and CA certificate for each cluster
- **Users**: Credentials (certificates, tokens, or exec plugins) to authenticate as
- **Contexts**: Named combinations of cluster + user + namespace (a "connection profile")

`kubectl` reads the **current-context** to know which cluster to talk to and as which user.

**Default location:** `~/.kube/config`

You can have multiple clusters in one kubeconfig file (e.g., local dev cluster, staging, production) and switch between them with:

```bash
kubectl config use-context <context-name>
```

When kind creates a cluster, it automatically adds a new context to `~/.kube/config` and sets it as the current context.

---

## Summary

| Item | Detail |
|------|--------|
| **Tool Used** | kind (Kubernetes in Docker) |
| **Cluster Name** | devops-cluster |
| **Kubernetes Version** | v1.32.x |
| **Nodes** | 1 (control-plane) |
| **kubeconfig location** | `~/.kube/config` |
| **Current Context** | `kind-devops-cluster` |

Today's key insight: **Everything in Kubernetes is a pod** — even the control plane components (API server, etcd, scheduler, controller manager) run as pods in the `kube-system` namespace. This means the cluster manages itself using the same primitives it uses for your applications.
