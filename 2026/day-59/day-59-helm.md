# Day 59 — Helm: Kubernetes Package Manager

## What Is Helm?

Helm is the **package manager for Kubernetes** — think of it as `apt` for Ubuntu or `brew` for macOS, but for Kubernetes workloads. Instead of writing and applying individual YAML files for Deployments, Services, ConfigMaps, Secrets, and PVCs separately, Helm bundles them into a single **Chart** that can be installed, upgraded, and rolled back with one command.

---

## Three Core Concepts

| Concept | Description | Analogy |
|---------|-------------|---------|
| **Chart** | A package of parameterised Kubernetes manifest templates | A `.deb` package |
| **Release** | One installed instance of a chart in your cluster | An installed application |
| **Repository** | A hosted collection of charts | `apt` sources list / PyPI |

A chart is rendered into real Kubernetes YAML at install time by combining the chart's **templates** with **values** (defaults from `values.yaml` overridden by user-supplied flags or files).

---

## Task 1 — Install Helm

```bash
# macOS
brew install helm

# Linux (curl script)
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Windows
choco install kubernetes-helm
```

### Verify

```bash
helm version
# version.BuildInfo{Version:"v3.x.x", ...}

helm env
# Lists HELM_CACHE_HOME, HELM_DATA_HOME, HELM_PLUGINS, etc.
```

**Installed version:** Helm v3.x (Helm 3 removed Tiller, the in-cluster server component present in Helm 2, making it far simpler and more secure.)

---

## Task 2 — Add a Repository and Search

```bash
# Add Bitnami's public chart repo
helm repo add bitnami https://charts.bitnami.com/bitnami

# Refresh the local index cache
helm repo update

# Search all charts in the repo
helm search repo bitnami

# Search for a specific chart
helm search repo nginx
```

Bitnami maintains **hundreds of charts** covering databases, web servers, message queues, monitoring stacks, and more — all production-hardened and updated regularly.

---

## Task 3 — Install a Chart

```bash
# Install nginx under the release name "my-nginx"
helm install my-nginx bitnami/nginx

# What did Helm create?
kubectl get all
```

A single command created a **Deployment**, a **Service**, and (where relevant) a **ConfigMap** — replacing multiple manual YAML files.

### Inspect the release

```bash
helm list                        # all releases in the current namespace
helm status my-nginx             # live status + notes from the chart
helm get manifest my-nginx       # the rendered Kubernetes YAML Helm applied
```

**Default state:** 1 Pod running, Service type = `LoadBalancer` (Bitnami nginx default).

---

## Task 4 — Customize with Values

### View chart defaults

```bash
helm show values bitnami/nginx
```

### Override with `--set` (quick, one-off)

```bash
helm install my-nginx-custom bitnami/nginx \
  --set replicaCount=3 \
  --set service.type=NodePort
```

Use dot notation for nested keys: `service.type`, `resources.limits.cpu`.

### Override with a values file (repeatable, version-controlled)

```bash
# Install using our custom-values.yaml
helm install my-nginx-file bitnami/nginx -f custom-values.yaml

# Confirm the overrides took effect
helm get values my-nginx-file
helm get values my-nginx-file --all   # includes defaults too
```

### `custom-values.yaml` — annotated

```yaml
# Replica count: 3 (default = 1)
replicaCount: 3

# NodePort so local clusters (Minikube / kind) can reach the service
service:
  type: NodePort
  nodePorts:
    http: 30080   # fixed port → always http://<node-ip>:30080

# Resource guardrails — prevents one Pod from starving the node
resources:
  requests:
    cpu: "50m"
    memory: "64Mi"
  limits:
    cpu: "200m"
    memory: "128Mi"

# Health probes — chart exposes these as values, no template editing needed
livenessProbe:
  enabled: true
  initialDelaySeconds: 30
readinessProbe:
  enabled: true
  initialDelaySeconds: 5

# Optional: annotations for observability tooling
podAnnotations:
  managed-by: "helm-day59-lab"
```

**Result:** 3 Pods, NodePort service, bounded resource usage — all without touching a single template file.

---

## Task 5 — Upgrade and Rollback

```bash
# Upgrade: scale to 5 replicas
helm upgrade my-nginx bitnami/nginx --set replicaCount=5

# View revision history
helm history my-nginx
# REVISION  STATUS      CHART           DESCRIPTION
# 1         superseded  nginx-x.y.z     Install complete
# 2         deployed    nginx-x.y.z     Upgrade complete

# Roll back to revision 1
helm rollback my-nginx 1

# History now shows THREE revisions
helm history my-nginx
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         superseded  Upgrade complete
# 3         deployed    Rollback to 1
```

**Key insight:** Rollback does **not** overwrite the failed revision. It creates a **new revision** (3) whose state mirrors revision 1. This gives you a full audit trail — equivalent to `kubectl rollout undo` but at the full-stack level (Deployment + Service + ConfigMap + …).

**After rollback:** 3 revisions total.

---

## Task 6 — Create Your Own Chart

### Scaffold

```bash
helm create my-app
```

This generates:

```
my-app/
├── Chart.yaml          # Chart metadata (name, version, description)
├── values.yaml         # Default values — override at install time
├── charts/             # Dependency charts (sub-charts)
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── serviceaccount.yaml
    ├── hpa.yaml
    ├── ingress.yaml
    ├── NOTES.txt       # Printed after helm install
    └── _helpers.tpl    # Reusable template fragments (partials)
```

### Go Templating

Helm uses Go's `text/template` engine. Three main sources:

| Source | Example |
|--------|---------|
| `.Values` | `{{ .Values.replicaCount }}` — from values.yaml |
| `.Chart` | `{{ .Chart.Name }}` — from Chart.yaml |
| `.Release` | `{{ .Release.Name }}` — from the `helm install <name>` argument |

**Example snippet from `templates/deployment.yaml`:**

```yaml
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

### Edit `values.yaml`

```yaml
replicaCount: 3

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent
```

### Validate, Preview, Install, Upgrade

```bash
# Lint — check chart structure and template syntax
helm lint my-app

# Dry-run render — see the YAML without touching the cluster
helm template my-release ./my-app

# Install
helm install my-release ./my-app
kubectl get pods   # → 3 replicas

# Upgrade to 5 replicas with --set override
helm upgrade my-release ./my-app --set replicaCount=5
kubectl get pods   # → 5 replicas
```

---

## Task 7 — Clean Up

```bash
# Uninstall all releases (removes all Kubernetes resources they created)
helm uninstall my-nginx
helm uninstall my-nginx-custom
helm uninstall my-nginx-file
helm uninstall my-release

# Keep release history for auditing (resources still deleted from cluster)
helm uninstall my-nginx --keep-history

# Confirm zero active releases
helm list
# NAME  NAMESPACE  REVISION  ...
# (empty)
```

---

## Summary — When to Use Each Mechanism

| Scenario | Approach |
|----------|----------|
| Try something quickly | `--set key=value` |
| Repeatable / team / CI-CD | `-f custom-values.yaml` |
| Need the raw YAML for debugging | `helm template` |
| Validate before committing | `helm lint` |
| Roll back an upgrade | `helm rollback <release> <revision>` |
| Audit all changes | `helm history <release>` |

---

## Key Commands Reference

```bash
helm repo add <name> <url>          # Add a chart repository
helm repo update                    # Refresh local index cache
helm search repo <keyword>          # Search available charts
helm show values <chart>            # View chart default values
helm install <release> <chart>      # Install a chart
helm upgrade <release> <chart>      # Upgrade an installed release
helm rollback <release> <revision>  # Roll back to a previous revision
helm history <release>              # View revision history
helm list                           # List all releases
helm status <release>               # Live release status
helm get manifest <release>         # View rendered Kubernetes YAML
helm get values <release>           # View active value overrides
helm uninstall <release>            # Remove a release from the cluster
helm create <name>                  # Scaffold a new chart
helm lint <chart-dir>               # Validate chart structure
helm template <release> <chart-dir> # Render templates without installing
```
