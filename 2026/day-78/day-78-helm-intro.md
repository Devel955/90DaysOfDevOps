# Day 78 – Introduction to Helm and Chart Basics

## Table of Contents
1. [What is Helm?](#what-is-helm)
2. [Core Concepts](#core-concepts)
3. [Installing Helm](#installing-helm)
4. [MySQL: Raw YAML vs Helm](#mysql-raw-yaml-vs-helm)
5. [mysql-values.yaml – Explained](#mysql-valuesyaml--explained)
6. [Helm Release Management](#helm-release-management)
7. [Helm Chart Directory Structure](#helm-chart-directory-structure)
8. [Why AI-BankApp Needs Helm](#why-ai-bankapp-needs-helm)
9. [Key Takeaways](#key-takeaways)

---

## What is Helm?

Helm is the **package manager for Kubernetes** — think of it as `apt` for Ubuntu or `yum` for RHEL, but for Kubernetes workloads. Instead of maintaining a pile of raw YAML files, Helm lets you package, version, share, and deploy Kubernetes applications as reusable units called **charts**.

The two biggest pain points Helm solves:

- **Repetition**: Without Helm, deploying the same app to dev, staging, and prod means maintaining three near-identical sets of YAML, changed only by image tags, replica counts, or environment variables. A single Helm chart + three values files replaces all of that.
- **No rollback**: `kubectl apply` does not track history. If a bad deploy goes out, you have to git revert and reapply manually. Helm tracks every change as a numbered revision and gives you `helm rollback` out of the box.

---

## Core Concepts

### Chart
A **chart** is a collection of files that describe a complete set of Kubernetes resources needed to run an application. Think of it as an application's "blueprint". A typical chart bundles:

- A `Deployment` or `StatefulSet`
- A `Service`
- A `ConfigMap`
- A `Secret`
- Optional: `PersistentVolumeClaim`, `HorizontalPodAutoscaler`, `Ingress`

All of this ships as a single, versioned, installable unit.

### Release
A **release** is a running instance of a chart installed in a cluster. You can install the same chart multiple times in the same or different namespaces, each with a different release name. For example:

```
helm install bankapp-mysql bitnami/mysql    # release: bankapp-mysql
helm install staging-mysql bitnami/mysql   # release: staging-mysql  (same chart, different config)
```

Each release gets its own revision history, so upgrades and rollbacks are independent.

### Repository
A **repository** is a place where packaged charts are stored and shared — similar to DockerHub for container images. The most widely used public repo is **Bitnami**, which hosts charts for MySQL, Redis, PostgreSQL, Prometheus, ArgoCD, and hundreds more. You add repos with:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update  # refresh the local cache
```

### Values
**Values** are the configuration inputs that customise a chart for each deployment. Every chart ships with a `values.yaml` file containing sensible defaults. You override them at install time with `--set key=value` flags or by passing your own `-f custom-values.yaml` file.

```yaml
# values.yaml (chart default)
replicaCount: 1

# your override
replicaCount: 3
```

Values flow into chart templates via Go template syntax:

```yaml
replicas: {{ .Values.replicaCount }}
```

---

## Installing Helm

### Prerequisites
- A running Kubernetes cluster (Kind, Minikube, or Docker Desktop)
- `kubectl` configured and pointing at the cluster

### Set up a Kind cluster using the AI-BankApp config
```bash
git clone -b feat/gitops https://github.com/TrainWithShubham/AI-BankApp-DevOps.git
cd AI-BankApp-DevOps
kind create cluster --config setup-k8s/kind-config.yml
```

### Install Helm

**Linux (script):**
```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**macOS:**
```bash
brew install helm
```

### Verify
```bash
helm version
# version.BuildInfo{Version:"v3.x.x", ...}

kubectl cluster-info
helm list
# NAME    NAMESPACE    REVISION    ...  (empty — no releases yet)
```

---

## MySQL: Raw YAML vs Helm

### The raw manifest approach

Deploying MySQL for the AI-BankApp without Helm requires maintaining **five separate files**:

| File | Purpose |
|------|---------|
| `mysql-deployment.yml` | StatefulSet / Deployment definition |
| `secrets.yml` | Root password and DB credentials (base64-encoded) |
| `pvc.yml` | PersistentVolumeClaim for data storage |
| `pv.yml` | PersistentVolume (if not using dynamic provisioning) |
| `service.yml` | Headless service for pod DNS |

Each file is mostly boilerplate with a few hardcoded values scattered throughout. Changing the database name means hunting across multiple files. There is no version tracking and no built-in rollback.

### The Helm approach

One command replaces all five files:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

helm install bankapp-mysql bitnami/mysql \
  --set auth.rootPassword=Test@123 \
  --set auth.database=bankappdb \
  --set primary.resources.requests.memory=256Mi \
  --set primary.resources.requests.cpu=250m \
  --set primary.resources.limits.memory=512Mi \
  --set primary.resources.limits.cpu=500m \
  --set primary.persistence.size=5Gi
```

Helm automatically creates the StatefulSet, headless Service, Secret (with the password), and PVC. Every resource is labelled with the release name, making it easy to inspect:

```bash
helm list
kubectl get all -l app.kubernetes.io/instance=bankapp-mysql
kubectl get pvc -l app.kubernetes.io/instance=bankapp-mysql
kubectl get secret -l app.kubernetes.io/instance=bankapp-mysql
```

Verify MySQL is running:

```bash
kubectl exec -it bankapp-mysql-0 -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"
# Output includes: bankappdb
```

### Side-by-side comparison

| Aspect | Raw k8s/mysql-deployment.yml | Bitnami MySQL Helm Chart |
|--------|------------------------------|--------------------------|
| Files to manage | 5 separate YAML files | 1 chart install command |
| Secrets | Hardcoded base64 in secrets.yml | Generated and managed by Helm |
| Storage | Manual PV + PVC YAML | Configured via `persistence.size` value |
| Replicas | Hardcoded in Deployment YAML | `primary.replicaCount` value |
| Metrics/Observability | Not included | `metrics.enabled: true` |
| Environment switching | Edit multiple files manually | Different values files per environment |
| Rollback | Manual git revert + kubectl apply | `helm rollback bankapp-mysql 1` |
| Versioning | None | Revision history tracked automatically |

---

## mysql-values.yaml – Explained

Using `-f values.yaml` is the preferred approach over chaining multiple `--set` flags. It is readable, version-controllable, and easy to diff between environments.

```yaml
# mysql-values.yaml

auth:
  rootPassword: Test@123   # MySQL root user password
  database: bankappdb      # Database created on first boot (matches AI-BankApp's DB_NAME env var)

primary:
  resources:
    limits:
      cpu: 500m            # Hard cap: MySQL cannot use more than 0.5 CPU cores
      memory: 512Mi        # Hard cap: 512 MiB RAM maximum
    requests:
      cpu: 250m            # Scheduler uses this to find a node with enough CPU headroom
      memory: 256Mi        # Scheduler uses this to find a node with enough memory headroom
  persistence:
    size: 5Gi              # PVC size for MySQL data directory
    storageClass: ""       # Empty string = use the cluster's default StorageClass

metrics:
  enabled: true            # Deploy a mysqld_exporter sidecar for Prometheus scraping
  serviceMonitor:
    enabled: false         # Set to true if you have the Prometheus Operator CRD installed
```

Deploy with the values file:

```bash
helm install bankapp-mysql bitnami/mysql -f mysql-values.yaml
```

To see every configurable knob in the chart:

```bash
helm show values bitnami/mysql | head -80
```

---

## Helm Release Management

### Install
```bash
helm install <release-name> <chart>
# helm upgrade --install is preferred in CI: installs if missing, upgrades if exists
helm upgrade --install bankapp-mysql bitnami/mysql -f mysql-values.yaml
```

### Upgrade
```bash
helm upgrade bankapp-mysql bitnami/mysql \
  --set auth.rootPassword=Test@123 \
  --set auth.database=bankappdb \
  --set metrics.enabled=true
```

### View history
```bash
helm history bankapp-mysql

# Example output:
# REVISION  UPDATED       STATUS      CHART           DESCRIPTION
# 1         ...           superseded  mysql-12.2.1    Install complete
# 2         ...           deployed    mysql-12.2.1    Upgrade complete
```

Each change creates a new revision. Helm stores the full rendered manifest for every revision so it can compute diffs and perform rollbacks.

### Rollback
```bash
helm rollback bankapp-mysql 1   # Roll back to revision 1
helm history bankapp-mysql

# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         superseded  Upgrade complete
# 3         deployed    Rollback to 1
```

Notice that a rollback creates revision 3 — it does not delete history. This is intentional: you always have a full audit trail.

### Uninstall
```bash
helm uninstall bankapp-mysql
# Removes all Kubernetes resources associated with the release
```

---

## Helm Chart Directory Structure

Pull the MySQL chart locally to inspect it:

```bash
helm pull bitnami/mysql --untar
ls mysql/
```

```
mysql/
├── Chart.yaml          # Chart metadata
├── values.yaml         # Default configuration values
├── charts/             # Bundled subchart dependencies
└── templates/          # Kubernetes manifest templates
    ├── primary/
    │   ├── statefulset.yaml   # StatefulSet with Go templating
    │   └── svc.yaml           # Headless service
    ├── _helpers.tpl    # Reusable template helper functions
    ├── NOTES.txt       # Post-install message printed to the terminal
    └── secrets.yaml    # Secret template
```

### Chart.yaml

```yaml
apiVersion: v2
name: mysql
description: A Helm chart for MySQL
version: 12.2.1       # Chart version — increments when the chart's structure changes
appVersion: "8.0.40"  # Version of the application inside the chart (MySQL 8.0.40)
```

**`version` vs `appVersion` — what is the difference?**

- `version` is the **chart's own version**. It changes when the chart's templates, helpers, or default values change — regardless of what MySQL version is inside it. This is what you pin when you write `helm install --version 12.2.1`.
- `appVersion` is the **application version** — the MySQL server version that the chart deploys by default. It is informational and helps operators know what software they are running, but it does not drive any Helm logic on its own.

You can have `version: 12.2.1` with `appVersion: "8.0.40"`, and later release `version: 12.3.0` with `appVersion: "8.0.41"` (bumping MySQL). Or you could release `version: 12.2.2` with `appVersion: "8.0.40"` if you only fixed a bug in a template.

### templates/primary/statefulset.yaml (excerpt)

```yaml
replicas: {{ .Values.primary.replicaCount }}
image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
```

`{{ .Values.X }}` is Go template syntax. When Helm renders the chart, it replaces these expressions with the values from `values.yaml` (or your overrides). This is the core of Helm's templating power — one template file generates different manifests for every environment.

### _helpers.tpl

Contains reusable template fragments (Go `define` blocks) that other templates call with `include`. For example, a helper might define the standard set of labels for every resource:

```yaml
{{- define "mysql.labels" -}}
app.kubernetes.io/name: {{ .Chart.Name }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}
```

### NOTES.txt

Displayed in the terminal after a successful install. The Bitnami MySQL NOTES.txt prints the commands needed to retrieve the root password and connect to the database — useful for onboarding new team members.

---

## Why AI-BankApp Needs Helm

The AI-BankApp's `k8s/` directory contains 12 raw YAML files:

```
bankapp-deployment.yml   configmap.yml      gateway.yml
mysql-deployment.yml     namespace.yml      ollama-deployment.yml
pv.yml                   pvc.yml            secrets.yml
service.yml              hpa.yml            cert-manager.yml
```

### Problems with the current approach

**1. Hardcoded values everywhere**
The image tag in `bankapp-deployment.yml` is a static string. Bumping to a new image requires manually editing the file, committing, and reapplying. With Helm, it becomes:

```bash
helm upgrade bankapp . --set image.tag=v2.1.0
```

**2. No environment separation**
Dev, staging, and prod need different resource limits, replica counts, and secrets. Today you either maintain three copies of every file or manually sed-replace values in a CI pipeline. Helm gives you:

```bash
helm install bankapp . -f values-dev.yaml       # dev
helm install bankapp . -f values-staging.yaml   # staging
helm install bankapp . -f values-prod.yaml      # production
```

**3. Dependency management**
`mysql-deployment.yml` is a custom-written StatefulSet that reinvents wheels the Bitnami chart already handles (backup hooks, metrics, replication). A Helm chart for the AI-BankApp can declare the Bitnami MySQL chart as a **dependency** in `Chart.yaml`:

```yaml
dependencies:
  - name: mysql
    version: "12.2.1"
    repository: https://charts.bitnami.com/bitnami
```

Now `helm dependency update` pulls MySQL in automatically. No more maintaining `mysql-deployment.yml`, `pv.yml`, `pvc.yml`, `secrets.yml` by hand.

**4. No rollback**
If a bad deployment reaches prod, `kubectl apply` cannot undo it automatically. With Helm:

```bash
helm rollback bankapp 3   # back to revision 3 in seconds
```

**5. Secrets sprawl**
`secrets.yml` contains base64-encoded credentials committed to the repo. A Helm chart can accept secrets as values at install time (never committed), or integrate with external secret managers.

### The Helm benefit summary

| Pain point | Raw YAML | With Helm |
|------------|----------|-----------|
| Image tag update | Edit YAML file | `--set image.tag=v2` |
| Multi-environment config | Maintain 3× copies | 1 chart + 3 values files |
| MySQL dependency | 4 manual YAML files | `dependencies:` in Chart.yaml |
| Rollback | Manual git revert | `helm rollback` |
| Audit trail | Git log only | `helm history` per release |
| Secret management | base64 in repo | Values at deploy time |

---

## Key Takeaways

- Helm is the **package manager for Kubernetes** — it templates, packages, versions, and deploys Kubernetes applications as charts.
- A **chart** is the blueprint; a **release** is a running instance of that blueprint; a **repository** hosts charts; **values** customise charts per environment.
- A single `helm install bitnami/mysql` command replaces five raw YAML files and gives you automatic Secret generation, PVC creation, metrics, and rollback support.
- Use **`-f values.yaml`** over multiple `--set` flags for anything beyond a one-off override — it is readable, diffable, and version-controllable.
- **`version`** in `Chart.yaml` tracks the chart's own structure; **`appVersion`** is the version of the software running inside it. They evolve independently.
- `helm rollback` creates a new revision (it does not delete history) — every change is auditable.
- The AI-BankApp's 12 raw YAML files are an ideal candidate for Helm: hardcoded values, no environment separation, no dependency management, no rollback — all problems Helm directly solves.

---

*Part of the #90DaysOfDevOps challenge — Day 78 of the Helm block.*
*Reference project: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: feat/gitops)*
