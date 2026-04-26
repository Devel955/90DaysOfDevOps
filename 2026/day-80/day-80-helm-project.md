# Day 80 — Helm Project: Multi-Environment Deployment and CI/CD

## Overview

This document covers the final day of the Helm block for the AI-BankApp (Spring Boot + MySQL + Ollama). The goal: take the custom Helm chart built on Day 79 and make it **production-ready** — with environment-specific values, pre-install hooks, chart packaging, and GitOps CI/CD integration.

---

## Task 1: Environment-Specific Values Files

One Helm chart. Three environments. Zero duplication.

### `bankapp/values-dev.yaml`

```yaml
bankapp:
  replicaCount: 1
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "latest"
    pullPolicy: Always
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "250m"
  autoscaling:
    enabled: false

mysql:
  enabled: true
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "250m"
  persistence:
    size: 2Gi
    storageClass: standard

ollama:
  enabled: true
  model: tinyllama
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "1.5Gi"
      cpu: "1000m"
  persistence:
    size: 5Gi
    storageClass: standard

storageClass:
  create: false
```

### `bankapp/values-staging.yaml`

```yaml
bankapp:
  replicaCount: 2
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 3
    targetCPUUtilization: 75

mysql:
  enabled: true
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  persistence:
    size: 5Gi
    storageClass: gp3

ollama:
  enabled: true
  model: tinyllama
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: StagingPass@456
  mysqlUser: root
  mysqlPassword: StagingPass@456

storageClass:
  create: true
```

### `bankapp/values-prod.yaml`

```yaml
bankapp:
  replicaCount: 4
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 4
    targetCPUUtilization: 70

mysql:
  enabled: true
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  persistence:
    size: 20Gi
    storageClass: gp3

ollama:
  enabled: true
  model: tinyllama
  resources:
    requests:
      memory: "2Gi"
      cpu: "900m"
    limits:
      memory: "2.5Gi"
      cpu: "1500m"
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: ProdSecure@789
  mysqlUser: root
  mysqlPassword: ProdSecure@789

storageClass:
  create: true

gateway:
  enabled: true
```

### Environment Comparison Table

| Setting | Dev | Staging | Prod |
|---|---|---|---|
| BankApp replicas | 1 (fixed) | 2–3 (HPA) | 2–4 (HPA) |
| Image tag | `latest` | `v1.2.0` | `v1.2.0` |
| Image pull policy | Always | IfNotPresent | IfNotPresent |
| HPA enabled | No | Yes | Yes |
| HPA CPU target | — | 75% | 70% |
| MySQL storage | 2 Gi | 5 Gi | 20 Gi |
| MySQL memory request | 128 Mi | 256 Mi | 512 Mi |
| MySQL CPU request | 100 m | 250 m | 500 m |
| Ollama memory limit | 1.5 Gi | not set | 2.5 Gi |
| StorageClass | standard | gp3 | gp3 |
| StorageClass created | No | Yes | Yes |
| Gateway | disabled | disabled | enabled |

### Deploying to Each Environment

```bash
# Dev — install on Kind cluster
helm install bankapp-dev bankapp/ -f bankapp/values-dev.yaml -n dev --create-namespace

# Staging — render and verify replicas
helm template bankapp-staging bankapp/ -f bankapp/values-staging.yaml | grep "replicas:"

# Prod — render and verify replicas
helm template bankapp-prod bankapp/ -f bankapp/values-prod.yaml | grep "replicas:"
```

**Key insight:** Multiple `-f` flags stack — later files override earlier ones. This means you can do:

```bash
helm install bankapp-prod bankapp/ -f bankapp/values.yaml -f bankapp/values-prod.yaml
```

The base `values.yaml` provides defaults; `values-prod.yaml` overrides only what differs.

---

## Task 2: Helm Hooks

### Pre-Install Hook — `bankapp/templates/pre-install-job.yaml`

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "bankapp.fullname" . }}-db-ready
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade       # (1)
    "helm.sh/hook-weight": "0"                    # (2)
    "helm.sh/hook-delete-policy": before-hook-creation  # (3)
spec:
  template:
    spec:
      containers:
        - name: db-check
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - |
              echo "Waiting for MySQL to be ready..."
              until nc -z {{ include "bankapp.fullname" . }}-mysql 3306; do
                echo "MySQL not ready, retrying in 3s..."
                sleep 3
              done
              echo "MySQL is ready!"
          resources:
            requests: { memory: "32Mi", cpu: "50m" }
            limits: { memory: "64Mi", cpu: "100m" }
      restartPolicy: Never
  backoffLimit: 10
```

### Hook Annotations Explained

| Annotation | Value | Meaning |
|---|---|---|
| `helm.sh/hook` | `pre-install,pre-upgrade` | **(1)** Run this Job **before** the main resources are created on both fresh install and upgrade |
| `helm.sh/hook-weight` | `"0"` | **(2)** Execution order when multiple hooks exist — lower weight runs first; use `-5`, `0`, `5`, `10` etc. |
| `helm.sh/hook-delete-policy` | `before-hook-creation` | **(3)** Delete any leftover Job from a previous run before creating a new one — prevents "already exists" errors on re-installs |

### Why This Matters for AI-BankApp

The Spring Boot application will crash-loop if it starts before MySQL is accepting connections. This hook provides a **cluster-side gate** — even before the Deployment's own `initContainers` run. The two layers together give defense-in-depth:

1. **Helm pre-install hook** — MySQL must be reachable before the Helm release is considered installed.
2. **Deployment initContainer** — Each pod also waits for MySQL before the main container starts.

### Other Useful Hook Types

| Hook | Use Case for AI-BankApp |
|---|---|
| `post-install` | Run Flyway/Liquibase DB migrations after first deploy |
| `pre-upgrade` | Take a MySQL backup before schema changes |
| `pre-delete` | Backup data before `helm uninstall` |
| `test` | Validate the app is healthy after deploy |

> **Note:** Helm hooks are **not tracked** in `helm get manifest`. They run outside the normal release lifecycle and are managed separately.

### Helm Test — `bankapp/templates/tests/test-connection.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "bankapp.fullname" . }}-test
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ['sh', '-c', 'wget -qO- http://{{ include "bankapp.fullname" . }}-service:8080/actuator/health']
  restartPolicy: Never
```

**Running the test:**

```bash
helm test bankapp-dev -n dev
```

This spins up a busybox pod that hits the Spring Boot `/actuator/health` endpoint. If the app is healthy, the pod exits 0 and Helm reports `PASSED`. A failed health check exits non-zero and Helm reports `FAILED`.

### Simulated Output of `helm list -A`

```
NAME          NAMESPACE  REVISION  UPDATED                   STATUS    CHART           APP VERSION
bankapp-dev   dev        1         2026-04-26 10:32:15 UTC   deployed  bankapp-0.2.0   1.1.0
```

### Simulated Output of `helm test bankapp-dev -n dev`

```
NAME: bankapp-dev
LAST DEPLOYED: Sun Apr 26 10:32:15 2026
NAMESPACE: dev
STATUS: deployed
REVISION: 1
TEST SUITE:     bankapp-dev-test
Last Started:   Sun Apr 26 10:34:00 2026
Last Completed: Sun Apr 26 10:34:08 2026
Phase:          Succeeded
```

---

## Task 3: Package and Version the Chart

### Lint and Package

```bash
# Validate chart structure and template syntax
helm lint bankapp/

# Package into a distributable .tgz
helm package bankapp/
# Output: bankapp-0.1.0.tgz
```

### Version Bumping Strategy

After adding hooks (structural change to the chart), bump `Chart.yaml`:

```yaml
# bankapp/Chart.yaml
apiVersion: v2
name: bankapp
description: AI-BankApp Helm chart (Spring Boot + MySQL + Ollama)
version: 0.2.0        # Chart version — bump when chart structure changes
appVersion: "1.1.0"   # App version — bump when the application version changes
```

```bash
helm package bankapp/
# Now you have: bankapp-0.1.0.tgz  and  bankapp-0.2.0.tgz
```

### Install from a Packaged Chart

```bash
helm install my-bankapp bankapp-0.2.0.tgz \
  -f bankapp/values-dev.yaml \
  -n bankapp \
  --create-namespace
```

This is useful for air-gapped environments or when distributing charts without access to the source directory.

### Creating a Chart Repository (GitHub Pages)

```bash
mkdir chart-repo
cp bankapp-*.tgz chart-repo/
helm repo index chart-repo/ --url https://your-username.github.io/helm-charts
# Generates chart-repo/index.yaml
```

Push `chart-repo/` to the `gh-pages` branch. Users can then:

```bash
helm repo add bankapp https://your-username.github.io/helm-charts
helm install bankapp-prod bankapp/bankapp -f values-prod.yaml
```

---

## Task 4: Helm in the AI-BankApp GitOps Pipeline

### Current Pipeline (Raw Manifests)

```
Developer push
  → GitHub Actions builds Docker image
  → Tags with git commit SHA
  → Updates image tag in k8s/bankapp-deployment.yml via sed
  → Commits change back to repo
  → ArgoCD detects change → syncs raw manifests to EKS
```

### GitOps Pipeline with Helm

```
Developer push
  → GitHub Actions builds Docker image
  → Tags with git commit SHA
  → Updates bankapp.image.tag in helm-chart/bankapp/values-prod.yaml via yq
  → Commits change back to repo
  → ArgoCD detects change → runs helm upgrade on EKS
```

### CI Step (GitHub Actions)

```yaml
- name: Update Helm values with new image tag
  run: |
    TAG=${{ steps.tag.outputs.sha_short }}
    yq -i '.bankapp.image.tag = "'$TAG'"' helm-chart/bankapp/values-prod.yaml

- name: Commit updated Helm values
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add helm-chart/bankapp/values-prod.yaml
    git diff --staged --quiet || git commit -m "ci: update bankapp image to $TAG [skip ci]"
    git push
```

> **Why `yq` over `sed`?** `yq` is YAML-aware — it preserves structure, comments, and formatting. `sed` does text substitution which can corrupt YAML if the value appears in unexpected places.

### ArgoCD Application: Before and After

**Before (raw manifests):**

```yaml
source:
  repoURL: https://github.com/your-org/AI-BankApp-DevOps
  targetRevision: feat/gitops
  path: k8s
```

**After (Helm):**

```yaml
source:
  repoURL: https://github.com/your-org/AI-BankApp-DevOps
  targetRevision: feat/gitops
  path: helm-chart/bankapp
  helm:
    valueFiles:
      - values-prod.yaml
```

### Advantages of ArgoCD + Helm vs Raw Manifests

| Aspect | Raw Manifests | Helm + ArgoCD |
|---|---|---|
| Environment config | Separate manifest sets per env | One chart, multiple values files |
| Drift detection | ArgoCD diffs raw YAML | ArgoCD diffs **rendered** templates against live state |
| Rollback | `git revert` + ArgoCD sync | `helm rollback` OR `git revert` + ArgoCD sync |
| Dependency management | Manual | `helm dependency update` |
| Parameterisation | sed/envsubst hacks | Native `{{ .Values.x }}` templating |
| Hook support | initContainers only | Helm pre/post hooks + initContainers |
| Secrets | Hardcoded or external tool | `--set` in CI, never committed |

ArgoCD renders Helm templates server-side and tracks drift against the **rendered output** — so if someone manually edits a resource in the cluster, ArgoCD will detect the diff and flag it as out-of-sync.

---

## Task 5: Production Best Practices

### 1. Always Use `--atomic` in CI/CD

```bash
helm upgrade --install bankapp bankapp/ \
  -f bankapp/values-prod.yaml \
  --set bankapp.image.tag=$GIT_SHA \
  -n bankapp \
  --create-namespace \
  --wait \
  --timeout 300s \
  --atomic
```

| Flag | Purpose |
|---|---|
| `--install` | Creates the release if it doesn't exist; upgrades if it does |
| `--set bankapp.image.tag=$GIT_SHA` | Pins to exact git commit (overrides values file) |
| `--wait` | Blocks until all pods are Ready |
| `--timeout 300s` | Fails after 5 min (prevents hanging pipelines) |
| `--atomic` | **Auto-rollbacks on failure** — critical for CI/CD |

### 2. Use `helm diff` Before Every Upgrade

```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade bankapp bankapp/ -f bankapp/values-prod.yaml
```

This shows a coloured diff of **exactly what would change** before committing to the upgrade. Never upgrade blind in production.

### 3. Resource Quotas Per Namespace

```yaml
# bankapp/templates/resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {{ include "bankapp.fullname" . }}-quota
  namespace: {{ .Release.Namespace }}
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
```

Prevents a runaway deployment from consuming all cluster resources.

### 4. Production Secrets Management

**Never store real secrets in `values.yaml` or any committed file.** The staging/prod values files above contain placeholder passwords for illustration only. In a real production deployment, use one of:

| Tool | How It Works |
|---|---|
| **External Secrets Operator + AWS Secrets Manager** | ESO syncs secrets from AWS into Kubernetes Secrets automatically |
| **Sealed Secrets** | Encrypt secrets with a cluster public key; safe to commit; decrypted only by the cluster |
| **HashiCorp Vault** | Vault Agent sidecar or CSI driver injects secrets directly into pods |

In CI/CD, pass secrets via `--set`:

```bash
helm upgrade --install bankapp bankapp/ \
  -f bankapp/values-prod.yaml \
  --set secrets.mysqlRootPassword=$MYSQL_ROOT_PASSWORD \
  --set secrets.mysqlPassword=$MYSQL_PASSWORD \
  --atomic
```

The `$MYSQL_ROOT_PASSWORD` comes from GitHub Actions secrets — it is never written to disk or committed.

---

## Task 6: The 3-Day Helm Journey

| Day | Concept Covered | AI-BankApp Connection |
|---|---|---|
| **78** | Helm install, repos, values, upgrade, rollback | Deployed MySQL for the BankApp via Bitnami chart |
| **79** | Custom chart from scratch, Go templates, helpers | Converted 12 raw `k8s/` manifests into a reusable Helm chart |
| **80** | Multi-env values, hooks, packaging, CI/CD | Production-ready chart with dev/staging/prod configs and GitOps integration |

---

## Approach Comparison: Helm vs Raw Manifests vs Kustomize

| Aspect | Raw Manifests | Helm | Kustomize |
|---|---|---|---|
| **Best for** | Simple, single-env apps | Multi-env, complex apps with dependencies | Overlaying patches on existing manifests |
| **Templating** | None (copy-paste) | Go templates with conditionals and loops | No templating — patch-based |
| **Dependencies** | Manual (`kubectl apply -f`) | `helm dependency update` | No native support |
| **Versioning** | Git tags only | Chart `version` + `appVersion` | Git tags only |
| **Rollback** | `git revert` + re-apply | `helm rollback` | `git revert` + re-apply |
| **Secrets** | Manual | `--set` in CI + ESO/Vault | `secretGenerator` (base64 only) |
| **Learning curve** | Low | Medium (Go templates) | Low–Medium |
| **AI-BankApp fit** | Current `k8s/` directory | This chart (3 services, HPA, hooks, multi-env) | Good if patching existing `k8s/` without rewriting |

### When to Choose Each Approach

**Raw manifests:** You have a single environment, the app is simple, and you want zero abstraction overhead. The current AI-BankApp `k8s/` directory is a good example.

**Helm:** You deploy the same application to multiple environments with different configs, you have dependencies (MySQL, Redis, etc.), and you want package-level versioning and rollback. This is where the AI-BankApp chart shines.

**Kustomize:** You inherited a raw manifest setup and want to add environment-specific patches without rewriting everything. Kustomize can also compose well with ArgoCD without the Helm learning curve.

---

## Production Secrets Management Plan

For the AI-BankApp running on EKS, the recommended approach is **External Secrets Operator (ESO) + AWS Secrets Manager**:

1. Store `MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`, and any API keys in AWS Secrets Manager.
2. Deploy ESO to the EKS cluster via its own Helm chart.
3. Create an `ExternalSecret` resource in the bankapp namespace that syncs from AWS Secrets Manager to a Kubernetes Secret.
4. Reference that Kubernetes Secret in the bankapp Deployment as `envFrom.secretRef`.
5. The `secrets:` block in `values-prod.yaml` becomes empty (or removed) — no credentials ever touch Git.

This approach gives you: automatic secret rotation support, audit logs in AWS CloudTrail, IAM-based access control, and zero credentials in version control.

---

## Cleanup Commands

```bash
# Remove the dev release
helm uninstall bankapp-dev -n dev

# Delete the namespace
kubectl delete namespace dev

# Tear down the Kind cluster
kind delete cluster --name tws-cluster

# Confirm nothing is left
helm list -A
kubectl get namespaces
```

---

## Key Takeaways

- **One chart, many environments** — `values-dev.yaml`, `values-staging.yaml`, and `values-prod.yaml` give you wildly different deployments from identical templates.
- **Hooks fill gaps that Kubernetes doesn't** — a pre-install Job provides a release-level readiness gate that initContainers can't.
- **`--atomic` is non-negotiable in CI/CD** — automatic rollback on failure prevents half-deployed states from reaching production.
- **Never commit real secrets** — use ESO, Sealed Secrets, or Vault; pass credentials via `--set` from pipeline secrets.
- **ArgoCD + Helm is a natural fit** — ArgoCD renders templates, tracks drift on the output, and triggers `helm upgrade` on every Git change.
- **`helm diff` before every upgrade** — know exactly what changes before it happens.
