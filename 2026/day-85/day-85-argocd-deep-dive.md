# Day 85 — ArgoCD Deep Dive: Sync Strategies, Rollbacks, and Multi-App Management

## 📋 Table of Contents

- [Overview](#overview)
- [Task 1: Sync Strategies — Automated vs Manual](#task-1-sync-strategies--automated-vs-manual)
- [Task 2: Sync Waves and Resource Ordering](#task-2-sync-waves-and-resource-ordering)
- [Task 3: ArgoCD Rollbacks](#task-3-argocd-rollbacks)
- [Task 4: App of Apps Pattern](#task-4-app-of-apps-pattern)
- [Task 5: ArgoCD Notifications](#task-5-argocd-notifications)
- [Task 6: ArgoCD Projects and RBAC](#task-6-argocd-projects-and-rbac)
- [Key Learnings](#key-learnings)
- [References](#references)

---

## Overview

Day 85 focuses on advanced ArgoCD patterns used in production Kubernetes environments. Building upon yesterday's AI-BankApp GitOps deployment, today we explore:

- **Sync Waves** for ordered, dependency-aware deployments
- **Manual vs Automated sync** strategies and when to use each
- **Rollback mechanisms** and the GitOps-correct approach
- **App of Apps pattern** for managing multiple applications at scale
- **Notifications** for real-time deployment awareness
- **Projects & RBAC** for multi-team access control

> **Reference Repo:** [TrainWithShubham/AI-BankApp-DevOps](https://github.com/TrainWithShubham/AI-BankApp-DevOps) — branch: `feat/gitops`

---

## Task 1: Sync Strategies — Automated vs Manual

### What is a Sync Strategy?

ArgoCD's sync strategy controls **how and when** changes from Git are applied to the Kubernetes cluster.

---

### Automated Sync

```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources removed from Git
    selfHeal: true   # Revert manual cluster changes
```

**How it works:**
- ArgoCD polls Git every **3 minutes** by default
- Any commit to the tracked branch is automatically applied
- `prune: true` removes Kubernetes resources that no longer exist in Git
- `selfHeal: true` reverts any manual `kubectl` changes — the cluster always matches Git

**When to use:**
- ✅ Development environments
- ✅ Staging environments
- ✅ When you want true GitOps automation with no human gate
- ❌ Not recommended for production without a review process

---

### Manual Sync

```yaml
syncPolicy: {}   # Empty — no automated sync
```

**How it works:**
- ArgoCD detects when the cluster is **OutOfSync** with Git
- It **does NOT apply** changes automatically
- A human must approve and trigger the sync

**Commands used:**

```bash
# Switch to manual sync
argocd app set bankapp --sync-policy none

# Check drift status
argocd app get bankapp

# Preview what will change (dry run)
argocd app diff bankapp
argocd app sync bankapp --dry-run

# Apply the sync
argocd app sync bankapp

# Switch back to automated
argocd app set bankapp --sync-policy automated --self-heal --auto-prune
```

**When to use:**
- ✅ Production environments needing a human review gate
- ✅ Regulated industries (finance, healthcare) with change approval requirements
- ✅ High-risk deployments where drift should be visible but not auto-applied
- ❌ Slower feedback loop for dev/staging

---

### Comparison Table

| Feature | Automated Sync | Manual Sync |
|---|---|---|
| Applies Git changes | Automatically (≤3 min) | Only on human trigger |
| Reverts manual kubectl changes | Yes (`selfHeal: true`) | No |
| Prunes deleted resources | Yes (`prune: true`) | Only when synced |
| Human approval required | No | Yes |
| Best environment | Dev / Staging | Production |
| Audit visibility | Git log | Git log + sync approval |

---

## Task 2: Sync Waves and Resource Ordering

### What are Sync Waves?

Sync waves are ArgoCD annotations that define the **order** in which resources are created or updated. Resources with lower wave numbers are processed first. ArgoCD waits for each wave to be **healthy** before moving to the next.

**Annotation format:**
```yaml
annotations:
  argocd.argoproj.io/sync-wave: "<integer>"
```

---

### AI-BankApp Sync Wave Configuration

#### Wave -2 — Infrastructure (First)

**`k8s/namespace.yml`**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: bankapp
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
```

**`k8s/pv.yml`** (StorageClass)
```yaml
metadata:
  name: gp3
  annotations:
    argocd.argoproj.io/sync-wave: "-2"
```

---

#### Wave -1 — Configuration

**`k8s/pvc.yml`** (both PVCs)
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

**`k8s/configmap.yml`** and **`k8s/secrets.yml`**
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "-1"
```

---

#### Wave 0 — Databases and Networking

**`k8s/mysql-deployment.yml`**
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

**`k8s/ollama-deployment.yml`**
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

**`k8s/service.yml`** (all three services)
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"
```

---

#### Wave 1 — Application

**`k8s/bankapp-deployment.yml`**
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

---

#### Wave 2 — Scaling (Last)

**`k8s/hpa.yml`**
```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "2"
```

---

### Sync Wave Order Diagram

```
┌─────────────────────────────────────────────────────────┐
│              AI-BankApp Sync Wave Order                  │
├──────────┬──────────────────────────────────────────────┤
│  Wave -2 │  Namespace, StorageClass (gp3)               │
│          │  ↓ Wait for healthy                          │
├──────────┼──────────────────────────────────────────────┤
│  Wave -1 │  PVCs, ConfigMap, Secret                     │
│          │  ↓ Wait for healthy                          │
├──────────┼──────────────────────────────────────────────┤
│  Wave  0 │  MySQL Deployment, Ollama, Services          │
│          │  (parallel within wave)                      │
│          │  ↓ Wait for healthy                          │
├──────────┼──────────────────────────────────────────────┤
│  Wave  1 │  BankApp Deployment                          │
│          │  ↓ Wait for healthy                          │
├──────────┼──────────────────────────────────────────────┤
│  Wave  2 │  HPA (Horizontal Pod Autoscaler)             │
└──────────┴──────────────────────────────────────────────┘
```

**Key behavior:**
- Resources **within the same wave** sync in **parallel**
- ArgoCD **waits** for all resources in a wave to be healthy before starting the next
- Negative integers run **before** zero, which runs before positive integers

---

## Task 3: ArgoCD Rollbacks

### Checking Sync History

```bash
argocd app history bankapp
```

**Output:**
```
ID  DATE                 REVISION
1   2026-04-10 10:00:00  abc1234
2   2026-04-10 10:15:00  def5678   (sync wave annotations added)
```

---

### Method 1: ArgoCD Rollback (Cluster-Level)

```bash
# Rollback to revision ID 1
argocd app rollback bankapp 1

# Check status after rollback
argocd app get bankapp
```

**Via UI:** Application → History → Select revision → Click **"Rollback"**

**What happens:**
- The cluster state is reverted to match the older Git revision
- Status shows **`OutOfSync`** because Git still has the newer commit
- Git history is **not changed**

---

### Method 2: Git Revert (GitOps-Correct Approach)

```bash
# Create a new commit that undoes the last change
git revert HEAD
git push
```

**What happens:**
- A new commit is created in Git that undoes the previous change
- ArgoCD detects the new commit and syncs it automatically
- The cluster is updated AND Git reflects the full audit trail

---

### ArgoCD Rollback vs Git Revert — Key Difference

| | ArgoCD Rollback | Git Revert |
|---|---|---|
| Changes cluster | ✅ Yes | ✅ Yes (via ArgoCD sync) |
| Changes Git | ❌ No | ✅ Yes (new commit) |
| Audit trail in Git | ❌ No | ✅ Full history preserved |
| GitOps-correct | ❌ No | ✅ Yes |
| Cluster stays `OutOfSync` | ✅ Yes (temporary state) | ❌ No (re-synced) |
| Recommended approach | Emergency only | Always preferred |

> **⚠️ GitOps Rule:** ArgoCD rollback is a **temporary emergency fix** only. Always follow up with `git revert` to restore GitOps integrity. The cluster should always reflect what's in Git — not the other way around.

---

## Task 4: App of Apps Pattern

### What is the App of Apps Pattern?

Instead of creating each ArgoCD Application manually, a **parent Application** watches a Git directory containing child Application manifests. Adding a new app to the cluster = adding a YAML file to Git.

```
root-app (parent)
    ├── bankapp.yaml      → creates "bankapp" Application
    ├── monitoring.yaml   → creates "monitoring" Application
    └── envoy-gateway.yaml → creates "envoy-gateway" Application
```

---

### Directory Structure

```
argocd-apps/
├── root-app.yaml          # Parent Application (applied manually once)
├── bankapp.yaml           # Child: AI-BankApp
├── monitoring.yaml        # Child: Prometheus + Grafana
└── envoy-gateway.yaml     # Child: Envoy Gateway
```

---

### Child App: BankApp (`argocd-apps/bankapp.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: bankapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

---

### Child App: Monitoring (`argocd-apps/monitoring.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: "65.*"
    helm:
      values: |
        grafana:
          adminPassword: admin123
        prometheus:
          prometheusSpec:
            retention: 3d
            resources:
              requests:
                memory: 256Mi
                cpu: 100m
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

---

### Child App: Envoy Gateway (`argocd-apps/envoy-gateway.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: envoy-gateway
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: docker.io/envoyproxy
    chart: gateway-helm
    targetRevision: "v1.4.*"
  destination:
    server: https://kubernetes.default.svc
    namespace: envoy-gateway-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

### Parent App: Root App (`argocd-apps/root-app.yaml`)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/AI-BankApp-DevOps.git
    targetRevision: feat/gitops
    path: argocd-apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

### Deploying the Root App

```bash
# Apply the parent app (one-time manual step)
kubectl apply -f argocd-apps/root-app.yaml

# Verify all applications are created
argocd app list
```

**Expected output:**
```
NAME            CLUSTER    NAMESPACE              PROJECT   STATUS  HEALTH
root-app        in-cluster argocd                 default   Synced  Healthy
bankapp         in-cluster bankapp                default   Synced  Healthy
monitoring      in-cluster monitoring             default   Synced  Healthy
envoy-gateway   in-cluster envoy-gateway-system   default   Synced  Healthy
```

---

### App of Apps Architecture Diagram

```
                        ┌─────────────────────────┐
                        │      Git Repository      │
                        │  /argocd-apps/           │
                        │  ├── bankapp.yaml         │
                        │  ├── monitoring.yaml      │
                        │  └── envoy-gateway.yaml   │
                        └────────────┬────────────┘
                                     │ watches
                        ┌────────────▼────────────┐
                        │       root-app           │
                        │  (Parent Application)    │
                        └────┬──────┬──────┬──────┘
                             │      │      │
               creates       │      │      │  creates
          ┌──────────────────┘      │      └──────────────────┐
          │                 creates │                         │
          ▼                         ▼                         ▼
  ┌───────────────┐       ┌─────────────────┐      ┌──────────────────┐
  │   bankapp     │       │   monitoring    │      │  envoy-gateway   │
  │  Application  │       │  Application   │      │  Application     │
  └───────┬───────┘       └───────┬─────────┘      └────────┬─────────┘
          │                       │                          │
          ▼                       ▼                          ▼
  bankapp namespace       monitoring namespace    envoy-gateway-system
  (MySQL, BankApp,        (Prometheus, Grafana)   (Envoy Gateway)
   Ollama, HPA)
```

> **Benefit:** Adding a new application to the cluster = adding one YAML file to Git. No manual ArgoCD commands needed after the root-app is set up.

---

## Task 5: ArgoCD Notifications

### Install Check

```bash
kubectl get pods -n argocd -l app.kubernetes.io/component=notifications-controller
```

---

### Notification ConfigMap

```bash
kubectl apply -n argocd -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # Triggers — define WHEN to notify
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-sync-succeeded]

  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]

  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]

  # Templates — define WHAT to send
  template.app-sync-succeeded: |
    message: "Application {{.app.metadata.name}} sync succeeded. Revision: {{.app.status.sync.revision}}"

  template.app-sync-failed: |
    message: "Application {{.app.metadata.name}} sync FAILED! Check ArgoCD for details."

  template.app-health-degraded: |
    message: "Application {{.app.metadata.name}} health is DEGRADED. Investigate immediately."
EOF
```

---

### Subscribe Application to Notifications

```bash
kubectl annotate application bankapp -n argocd \
  notifications.argoproj.io/subscribe.on-sync-succeeded.webhook="" \
  notifications.argoproj.io/subscribe.on-sync-failed.webhook="" \
  notifications.argoproj.io/subscribe.on-health-degraded.webhook=""
```

---

### View Notification History

```bash
kubectl get applications bankapp -n argocd \
  -o jsonpath='{.status.operationState.message}'
```

---

### Notification Flow

```
Event Occurs in Cluster
        │
        ▼
   ArgoCD detects
   (sync failed / health degraded / sync succeeded)
        │
        ▼
   Trigger evaluates
   condition → true
        │
        ▼
   Template renders
   the message
        │
        ▼
   Service delivers
   (Slack / Webhook / Email / PagerDuty)
```

---

### Slack Integration (Additional Config)

To add Slack as a notification channel, add to the ConfigMap:

```yaml
service.slack: |
  token: $slack-token
```

Then annotate:
```bash
kubectl annotate application bankapp -n argocd \
  notifications.argoproj.io/subscribe.on-sync-failed.slack="your-channel"
```

---

## Task 6: ArgoCD Projects and RBAC

### What are ArgoCD Projects?

Projects provide **multi-tenancy** in ArgoCD. They restrict:
- **Source repos** an application can use
- **Destination namespaces** where apps can deploy
- **Resource types** that can be created

---

### Create a Project for the BankApp Team

```bash
argocd proj create bankapp-team \
  --description "AI-BankApp team project" \
  --src "https://github.com/<your-username>/AI-BankApp-DevOps.git" \
  --dest "https://kubernetes.default.svc,bankapp" \
  --dest "https://kubernetes.default.svc,monitoring"
```

**This project:**
- Can only source from the AI-BankApp repository
- Can only deploy to `bankapp` and `monitoring` namespaces
- **Cannot** touch `kube-system`, `argocd`, or any other namespace

---

### Move Application to Project

```bash
argocd app set bankapp --project bankapp-team
```

---

### RBAC Policy Configuration

**`argocd-rbac-cm` ConfigMap:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-rbac-cm
  namespace: argocd
data:
  policy.csv: |
    # BankApp developers can view and sync — but NOT rollback
    p, role:bankapp-dev, applications, get,      bankapp-team/*, allow
    p, role:bankapp-dev, applications, sync,     bankapp-team/*, allow
    p, role:bankapp-dev, applications, rollback, bankapp-team/*, deny

    # Assign role to group
    g, bankapp-developers, role:bankapp-dev
```

---

### RBAC Permission Table

| Action | bankapp-developers | Senior/Admin |
|---|---|---|
| View applications | ✅ Allowed | ✅ Allowed |
| Sync applications | ✅ Allowed | ✅ Allowed |
| Rollback | ❌ Denied | ✅ Allowed |
| Deploy to kube-system | ❌ Restricted by Project | ✅ Allowed |
| Access other teams' apps | ❌ No | ✅ Allowed |

---

### How Projects Prevent Team Conflicts

```
Without Projects:                    With Projects:
─────────────────                    ────────────────────────────
Team A accidentally syncs           Team A's project restricts them
to Team B's namespace     ───►      to only bankapp + monitoring
                                    namespaces. Attempt to deploy
Team B's database gets              anywhere else → REJECTED by
overwritten. 💥                     ArgoCD. ✅
```

**Projects and RBAC together ensure:**
1. **Namespace isolation** — One team cannot accidentally affect another's workloads
2. **Repo isolation** — Teams cannot deploy arbitrary code from untrusted repos
3. **Action isolation** — Junior devs can sync but seniors must approve rollbacks
4. **Full audit trail** — Every action is logged against an identity

---

## Key Learnings

### 1. GitOps is an Operational Framework

GitOps is not just "deploy from Git." It includes:
- Git as the **single source of truth**
- Automated reconciliation (self-healing)
- Full audit trail through Git history
- Human review gates via Manual sync in production

### 2. Sync Waves Enable Dependency Management

Without sync waves, ArgoCD applies all resources simultaneously — MySQL and BankApp start at the same time, causing connection failures. Sync waves enforce the correct order.

### 3. Always Git Revert, Never Just ArgoCD Rollback

| Step | ArgoCD Rollback | GitOps Rollback |
|---|---|---|
| Cluster state | ✅ Fixed | ✅ Fixed |
| Git state | ❌ Stale | ✅ Updated |
| Audit trail | ❌ Missing | ✅ Complete |
| Ongoing sync | ❌ Broken (OutOfSync) | ✅ Working |

### 4. App of Apps Scales GitOps

Managing 1 app = create 1 ArgoCD Application
Managing 50 apps = add 50 YAML files to Git → root-app handles the rest

### 5. Projects + RBAC = Production Multi-tenancy

Never run multiple teams against a single unrestricted ArgoCD. Projects and RBAC are not optional for production clusters.


## 📁 File Location

```
2026/
└── day-85/
    └── day-85-argocd-deep-dive.md   ← This file
```

---

*Day 85 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
