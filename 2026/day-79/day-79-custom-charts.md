# Day 79 — Custom Helm Chart for AI-BankApp

## Overview

Is task mein humne AI-BankApp ke 12 raw Kubernetes YAML files ko ek single, reusable Helm chart mein convert kiya. Ab poora stack ek `helm install` command se deploy hota hai.

**Project:** [AI-BankApp-DevOps](https://github.com/TrainWithShubham/AI-BankApp-DevOps) (branch: `feat/gitops`)

**Stack:**
- Spring Boot Banking App
- MySQL 8.0 Database
- Ollama AI Chatbot (tinyllama model)

---

## Task 1: Chart Scaffold

```bash
# Repo clone
git clone https://github.com/TrainWithShubham/AI-BankApp-DevOps
cd AI-BankApp-DevOps

# k8s directory study
ls k8s/
```

**Raw Manifests Mapping:**

| File | Purpose |
|------|---------|
| `namespace.yml` | `bankapp` namespace create karta hai |
| `configmap.yml` | MySQL host, port, database, Ollama URL |
| `secrets.yml` | MySQL credentials (base64 encoded) |
| `pv.yml` | StorageClass (gp3 via EBS CSI) |
| `pvc.yml` | PVCs for MySQL (5Gi) and Ollama (10Gi) |
| `bankapp-deployment.yml` | BankApp with init containers, probes, envFrom |
| `mysql-deployment.yml` | MySQL with EBS volume mount, probes |
| `ollama-deployment.yml` | Ollama with postStart model pull, probes |
| `service.yml` | ClusterIP services for all 3 components |
| `hpa.yml` | HPA for BankApp (2-4 replicas, 70% CPU) |
| `gateway.yml` | Envoy Gateway + HTTPRoute + TLS |
| `cert-manager.yml` | Let's Encrypt ClusterIssuer |

```bash
# Helm chart scaffold
mkdir helm-chart && cd helm-chart
helm create bankapp

# Generated templates clean karo
rm -rf bankapp/templates/*.yaml bankapp/templates/tests/
```

---

## Task 2: Chart.yaml aur values.yaml

### `bankapp/Chart.yaml`

```yaml
apiVersion: v2
name: bankapp
description: AI-BankApp -- Spring Boot banking application with MySQL and Ollama AI chatbot
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
  - name: TrainWithShubham
    url: https://github.com/TrainWithShubham
keywords:
  - bankapp
  - spring-boot
  - mysql
  - ollama
  - ai
```

### `bankapp/values.yaml` (with explanations)

```yaml
# ─── BankApp Configuration ───────────────────────────────────────────────────
bankapp:
  replicaCount: 4                        # Default replicas (HPA enabled hone par ignore hota hai)
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "latest"
    pullPolicy: Always                   # Har deploy pe fresh image pull
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  service:
    type: ClusterIP                      # Internal service; LoadBalancer for prod
    port: 8080
  autoscaling:
    enabled: true
    minReplicas: 2                       # Minimum 2 pods hamesha chalte rahenge
    maxReplicas: 4
    targetCPUUtilization: 70             # 70% CPU pe scale out

# ─── MySQL Configuration ─────────────────────────────────────────────────────
mysql:
  enabled: true                          # false karne se MySQL band ho jaata hai
  image:
    repository: mysql
    tag: "8.0"
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  persistence:
    size: 5Gi
    storageClass: gp3                    # Kind mein 'standard' use karo

# ─── Ollama AI Configuration ─────────────────────────────────────────────────
ollama:
  enabled: true                          # false karne se poora AI component hata jaata hai
  image:
    repository: ollama/ollama
    tag: "latest"
  model: tinyllama                       # Koi bhi Ollama model yahan dal sakte ho
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

# ─── Shared Configuration ────────────────────────────────────────────────────
config:
  mysqlDatabase: bankappdb
  ollamaUrl: ""                          # Khaali = service name se auto-generate hota hai

# ─── Secrets ─────────────────────────────────────────────────────────────────
secrets:
  mysqlRootPassword: Test@123            # Production mein --set ya external secret use karo
  mysqlUser: root
  mysqlPassword: Test@123

# ─── Storage ─────────────────────────────────────────────────────────────────
storageClass:
  create: true                           # Kind mein false karo
  name: gp3
  provisioner: ebs.csi.aws.com

# ─── Gateway (Optional) ──────────────────────────────────────────────────────
gateway:
  enabled: false
  hostname: ""
  tls:
    enabled: false
```

**Raw vs Helm Comparison — Secrets:**

| Raw `k8s/secrets.yml` | Helm `templates/secrets.yaml` |
|----------------------|-------------------------------|
| Manually base64 encoded values | `b64enc` function se auto encode |
| Hardcoded credentials | `values.yaml` se configurable |
| Env-specific editing zaroori | `--set secrets.mysqlRootPassword=xyz` se override |

---

## Task 3: Core Templates

### Side-by-Side Comparison — ConfigMap

**Raw (`k8s/configmap.yml`):**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: bankapp-config
  namespace: bankapp
data:
  MYSQL_HOST: "bankapp-mysql"
  MYSQL_PORT: "3306"
  MYSQL_DATABASE: "bankappdb"
  OLLAMA_URL: "http://bankapp-ollama:11434"
  SERVER_FORWARD_HEADERS_STRATEGY: "native"
```

**Helm Template (`templates/configmap.yaml`):**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "bankapp.fullname" . }}-config
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
data:
  MYSQL_HOST: {{ include "bankapp.fullname" . }}-mysql
  MYSQL_PORT: "3306"
  MYSQL_DATABASE: {{ .Values.config.mysqlDatabase | quote }}
  OLLAMA_URL: {{ default (printf "http://%s-ollama:11434" (include "bankapp.fullname" .)) .Values.config.ollamaUrl | quote }}
  SERVER_FORWARD_HEADERS_STRATEGY: "native"
```

**Faida:** Release name badalne se sab kuch automatically update ho jaata hai. Conflicts nahi hote.

---

### Side-by-Side Comparison — Secrets

**Raw (`k8s/secrets.yml`):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bankapp-secret
  namespace: bankapp
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: VGVzdEAxMjM=    # Manually encoded
  MYSQL_USER: cm9vdA==
  MYSQL_PASSWORD: VGVzdEAxMjM=
```

**Helm Template (`templates/secrets.yaml`):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "bankapp.fullname" . }}-secret
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: {{ .Values.secrets.mysqlRootPassword | b64enc | quote }}
  MYSQL_USER: {{ .Values.secrets.mysqlUser | b64enc | quote }}
  MYSQL_PASSWORD: {{ .Values.secrets.mysqlPassword | b64enc | quote }}
```

**Faida:** `b64enc` automatically encode karta hai. No manual `echo -n "value" | base64` needed.

---

### `templates/storage.yaml`

```yaml
{{- if .Values.storageClass.create }}
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: {{ .Values.storageClass.name }}
provisioner: {{ .Values.storageClass.provisioner }}
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
{{- end }}
---
{{- if .Values.mysql.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "bankapp.fullname" . }}-mysql-pvc
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
spec:
  storageClassName: {{ .Values.mysql.persistence.storageClass }}
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.mysql.persistence.size }}
{{- end }}
---
{{- if .Values.ollama.enabled }}
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: {{ include "bankapp.fullname" . }}-ollama-pvc
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
spec:
  storageClassName: {{ .Values.ollama.persistence.storageClass }}
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: {{ .Values.ollama.persistence.size }}
{{- end }}
```

---

## Task 4: Deployment Templates

### BankApp Deployment — Key Points

```yaml
# Init containers — MySQL ready hone ka wait
initContainers:
  - name: wait-for-mysql
    image: busybox:1.36
    command: ["/bin/sh", "-c", "until nc -z {{ include "bankapp.fullname" . }}-mysql 3306; do sleep 2; done"]

  # Ollama init container sirf tab jab enabled=true ho
  {{- if .Values.ollama.enabled }}
  - name: wait-for-ollama
    image: busybox:1.36
    command: ["/bin/sh", "-c", "until nc -z {{ include "bankapp.fullname" . }}-ollama 11434; do sleep 2; done"]
  {{- end }}

# Replicas: HPA enabled hone par Helm replicas set nahi karta
{{- if not .Values.bankapp.autoscaling.enabled }}
replicas: {{ .Values.bankapp.replicaCount }}
{{- end }}
```

### Ollama Deployment — PostStart Hook

```yaml
lifecycle:
  postStart:
    exec:
      command:
        - /bin/sh
        - -c
        - |
          until ollama list > /dev/null 2>&1; do sleep 2; done
          ollama pull {{ .Values.ollama.model }}   # Values se model name
```

---

## Task 5: Services aur HPA

### `templates/services.yaml`

```yaml
# MySQL Service (hamesha present, agar mysql.enabled=true)
apiVersion: v1
kind: Service
metadata:
  name: {{ include "bankapp.fullname" . }}-mysql
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: {{ include "bankapp.fullname" . }}-mysql
  ports:
    - port: 3306
---
# Ollama Service (sirf ollama.enabled=true par)
{{- if .Values.ollama.enabled }}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "bankapp.fullname" . }}-ollama
  namespace: {{ .Release.Namespace }}
spec:
  selector:
    app: {{ include "bankapp.fullname" . }}-ollama
  ports:
    - port: 11434
{{- end }}
---
# BankApp Service
apiVersion: v1
kind: Service
metadata:
  name: {{ include "bankapp.fullname" . }}-service
  namespace: {{ .Release.Namespace }}
spec:
  type: {{ .Values.bankapp.service.type }}
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600
  selector:
    app: {{ include "bankapp.fullname" . }}
  ports:
    - port: {{ .Values.bankapp.service.port }}
      targetPort: 8080
```

### `templates/hpa.yaml`

```yaml
{{- if .Values.bankapp.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "bankapp.fullname" . }}-hpa
  namespace: {{ .Release.Namespace }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "bankapp.fullname" . }}
  minReplicas: {{ .Values.bankapp.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.bankapp.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.bankapp.autoscaling.targetCPUUtilization }}
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
{{- end }}
```

---

## Task 6: Validation aur Deployment

### Helm Lint

```bash
helm lint bankapp/

# Expected output:
# ==> Linting bankapp/
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed
```

### `helm template` Output (rendered manifests)

```bash
helm template my-bankapp bankapp/
```

**Sample rendered ConfigMap:**
```yaml
# Source: bankapp/templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-bankapp-config
  namespace: default
  labels:
    helm.sh/chart: bankapp-0.1.0
    app.kubernetes.io/name: bankapp
    app.kubernetes.io/instance: my-bankapp
    app.kubernetes.io/version: "1.0.0"
    app.kubernetes.io/managed-by: Helm
data:
  MYSQL_HOST: my-bankapp-mysql
  MYSQL_PORT: "3306"
  MYSQL_DATABASE: "bankappdb"
  OLLAMA_URL: "http://my-bankapp-ollama:11434"
  SERVER_FORWARD_HEADERS_STRATEGY: "native"
```

### Override ke saath render

```bash
helm template my-bankapp bankapp/ \
  --set bankapp.image.tag=abc1234 \
  --set bankapp.replicaCount=2 \
  --set ollama.enabled=false
```

### Kind pe Deploy

```bash
# Dry run pehle
helm install my-bankapp bankapp/ --dry-run --debug -n bankapp --create-namespace

# Actual deploy (Kind ke liye StorageClass override)
helm install my-bankapp bankapp/ \
  -n bankapp --create-namespace \
  --set storageClass.create=false \
  --set mysql.persistence.storageClass=standard \
  --set ollama.persistence.storageClass=standard
```

### Verify

```bash
helm list -n bankapp
kubectl get all -n bankapp
kubectl get pvc -n bankapp
kubectl get configmap,secret -n bankapp
kubectl get pods -n bankapp -w
```

### App Access

```bash
kubectl port-forward svc/my-bankapp-bankapp-service -n bankapp 8080:8080
# Browser mein: http://localhost:8080
```

### Cleanup

```bash
helm uninstall my-bankapp -n bankapp
```

---

## Go Template Syntax Cheat Sheet

| Syntax | Use Case | Example |
|--------|----------|---------|
| `{{ .Values.key }}` | values.yaml se value lena | `{{ .Values.bankapp.replicaCount }}` |
| `{{ .Release.Name }}` | Helm release name | `my-bankapp` |
| `{{ .Release.Namespace }}` | Target namespace | `bankapp` |
| `{{- if .Values.flag }}` | Conditional block | Ollama resources ko enable/disable karna |
| `{{- end }}` | if/with/range close karna | - |
| `{{ include "name" . }}` | Helper function call (pipeable) | `{{ include "bankapp.fullname" . }}` |
| `{{ toYaml . \| nindent 4 }}` | YAML object ko proper indent ke saath render karna | Resources block |
| `{{ .Values.val \| b64enc \| quote }}` | Base64 encode + quote | Secret values |
| `{{ .Values.val \| quote }}` | String ko quote mein wrap karna | ConfigMap values |
| `{{ default "fallback" .Values.val }}` | Default value agar empty ho | ollamaUrl |
| `{{ printf "format" arg }}` | String formatting | Service URL banana |
| `{{- with .Values.resources }}` | Non-nil value ke liye scoped block | Resources apply karna |
| `{{- range .Values.list }}` | List iterate karna | Multiple items |

---

## `ollama.enabled=false` ka Effect

Ek boolean se poori AI component band:

```bash
helm template my-bankapp bankapp/ --set ollama.enabled=false
```

**Jo resources hata jaate hain:**

| Resource | `enabled=true` | `enabled=false` |
|----------|---------------|-----------------|
| Ollama Deployment | ✅ Present | ❌ Removed |
| Ollama Service | ✅ Present | ❌ Removed |
| Ollama PVC (10Gi) | ✅ Present | ❌ Removed |
| BankApp init container (wait-for-ollama) | ✅ Present | ❌ Removed |
| BankApp OLLAMA_URL config | ✅ Present | ✅ Present (unused) |

**Yani:** `--set ollama.enabled=false` se app bina AI chatbot ke deploy hoti hai. Storage nahi banta, init container nahi chalta, service nahi banti. Ek boolean = poora component.

---

## Raw k8s/ vs Helm Chart — Final Comparison

| Aspect | Raw k8s/ (12 files) | Helm Chart |
|--------|---------------------|------------|
| Deploy command | `kubectl apply -f k8s/` | `helm install my-bankapp bankapp/` |
| Environment override | Manually YAML edit karo | `--set key=value` |
| Secrets encoding | Manual `base64` | `b64enc` auto-handles |
| Component on/off | Manually delete files | `--set ollama.enabled=false` |
| Rollback | Manual | `helm rollback my-bankapp 1` |
| Versioning | Git diff only | `helm history my-bankapp` |
| Reusability | Copy-paste + edit | `helm install` with different values |
| Namespace conflict | Name manually change karo | Release name automatically prefix karta hai |

---

## Learnings

1. **Helm templates = Go templates** — `{{- if }}`, `{{ .Values }}`, `{{ include }}` — ye sab Go templating engine hai
2. **`b64enc` magic** — Raw YAML mein manually base64 encode karna padta tha, Helm automatically karta hai
3. **Conditional resources** — `{{- if .Values.component.enabled }}` se poora component on/off hota hai
4. **`toYaml | nindent`** — Resources jaise nested objects ke liye yahi pattern use karo
5. **`helm template` debugging** — Deploy karne se pehle hamesha local render karo
6. **`helm lint`** — YAML syntax aur Helm-specific errors pakadta hai
7. **Release name prefix** — `{{ include "bankapp.fullname" . }}` se same cluster mein multiple installs conflict nahi karte

---

*Day 79 of #90DaysOfDevOps | #DevOpsKaJosh | @TrainWithShubham*
