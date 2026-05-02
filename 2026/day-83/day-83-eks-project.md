# Day 83 — EKS Project: AI-BankApp Production Deployment

## Overview

Day 83 marks the culmination of a 3-day EKS block. Today, the AI-BankApp was deployed as a fully production-grade application on Amazon EKS — Spring Boot frontend, MySQL database, and Ollama AI service, all wired together with Gateway API routing, EBS persistent storage, HPA autoscaling, and a Prometheus + Grafana monitoring stack.

---

## Full Architecture Diagram

```
                          ┌─────────────────────────────────────────────────┐
                          │                  AWS Cloud                      │
                          │                                                 │
   Internet               │   ┌──────────────────────────────────────────┐  │
      │                   │   │         VPC (bankapp-eks)                │  │
      ▼                   │   │   ┌──────────┐  ┌──────────┐            │  │
 ┌─────────┐              │   │   │ Public   │  │ Public   │            │  │
 │  User   │              │   │   │ Subnet   │  │ Subnet   │            │  │
 │ Browser │              │   │   │ (AZ-a)   │  │ (AZ-b)   │            │  │
 └────┬────┘              │   │   └────┬─────┘  └────┬─────┘            │  │
      │ HTTP              │   │        │              │                  │  │
      ▼                   │   │   ┌────▼──────────────▼────┐            │  │
 ┌──────────────┐         │   │   │  NLB (Network Load     │            │  │
 │  Route 53    │         │   │   │  Balancer) — External  │            │  │
 │  (optional)  │         │   │   └──────────┬─────────────┘            │  │
 └──────┬───────┘         │   │              │                          │  │
        │                 │   │   ┌──────────▼──────────────────────┐   │  │
        │                 │   │   │     Envoy Gateway (Gateway API) │   │  │
        └─────────────────┼───┼──►│  HTTPRoute → bankapp:8080       │   │  │
                          │   │   └──────────┬──────────────────────┘   │  │
                          │   │              │                          │  │
                          │   │   ┌──────────▼──────────────────────┐   │  │
                          │   │   │         EKS Cluster             │   │  │
                          │   │   │   Namespace: bankapp            │   │  │
                          │   │   │                                 │   │  │
                          │   │   │  ┌──────────────────────────┐   │   │  │
                          │   │   │  │  BankApp (Spring Boot)   │   │   │  │
                          │   │   │  │  Pods: 2–4 (HPA managed) │   │   │  │
                          │   │   │  └────────┬────────┬────────┘   │   │  │
                          │   │   │           │        │            │   │  │
                          │   │   │  ┌────────▼──┐  ┌──▼─────────┐ │   │  │
                          │   │   │  │  MySQL    │  │  Ollama     │ │   │  │
                          │   │   │  │  (1 pod)  │  │  (1 pod)    │ │   │  │
                          │   │   │  │  5Gi EBS  │  │  10Gi EBS   │ │   │  │
                          │   │   │  └──────┬────┘  └──────┬──────┘ │   │  │
                          │   │   │         │              │        │   │  │
                          │   │   │  ┌──────▼──────────────▼──────┐ │   │  │
                          │   │   │  │  EBS Volumes (gp2/gp3)     │ │   │  │
                          │   │   │  │  Provisioned by EBS CSI    │ │   │  │
                          │   │   │  └────────────────────────────┘ │   │  │
                          │   │   │                                 │   │  │
                          │   │   │  Namespace: monitoring          │   │  │
                          │   │   │  ┌──────────────────────────┐   │   │  │
                          │   │   │  │ Prometheus + Grafana      │   │   │  │
                          │   │   │  │ (kube-prometheus-stack)   │   │   │  │
                          │   │   │  └──────────────────────────┘   │   │  │
                          │   │   └─────────────────────────────────┘   │  │
                          │   └──────────────────────────────────────────┘  │
                          └─────────────────────────────────────────────────┘
```

---

## Task 1: Deploy the Complete AI-BankApp Stack

### Cluster Verification

```bash
kubectl get nodes
# NAME                                           STATUS   ROLES    AGE   VERSION
# ip-10-0-1-xx.us-west-2.compute.internal        Ready    <none>   2d    v1.29.x
# ip-10-0-2-xx.us-west-2.compute.internal        Ready    <none>   2d    v1.29.x
# ip-10-0-3-xx.us-west-2.compute.internal        Ready    <none>   2d    v1.29.x
```
## Screenshots

### Nodes Running
![kubectl get nodes](Screenshot%202026-05-01%20213033.png)

### Deployment Sequence

```bash
cd AI-BankApp-DevOps

# 1. Namespace and storage
kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/pv.yml
kubectl apply -f k8s/pvc.yml

# 2. Configuration
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/secrets.yml

# 3. Database and AI service
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/ollama-deployment.yml

# 4. Wait for dependencies
kubectl wait --for=condition=ready pod -l app=mysql -n bankapp --timeout=120s
kubectl wait --for=condition=ready pod -l app=ollama -n bankapp --timeout=600s

# 5. Application
kubectl apply -f k8s/bankapp-deployment.yml
kubectl apply -f k8s/hpa.yml

# 6. Wait for BankApp
kubectl wait --for=condition=ready pod -l app=bankapp -n bankapp --timeout=300s
```


### Stack Verification

```bash
kubectl get all -n bankapp
# NAME                           READY   STATUS    RESTARTS   AGE
# pod/bankapp-xxx-yyy            1/1     Running   0          5m
# pod/bankapp-xxx-zzz            1/1     Running   0          5m
# pod/mysql-xxx-yyy              1/1     Running   0          8m
# pod/ollama-xxx-yyy             1/1     Running   0          7m
#
# NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)
# service/bankapp    ClusterIP   172.20.x.x      <none>        8080/TCP
# service/mysql      ClusterIP   172.20.x.x      <none>        3306/TCP
# service/ollama     ClusterIP   172.20.x.x      <none>        11434/TCP
#
# NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
# deployment.apps/bankapp   2/2     2            2           5m
# deployment.apps/mysql     1/1     1            1           8m
# deployment.apps/ollama    1/1     1            1           7m

kubectl get pvc -n bankapp
# NAME             STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# mysql-pvc        Bound    pvc-xxx-yyy    5Gi        RWO            gp2            8m
# ollama-pvc       Bound    pvc-aaa-bbb    10Gi       RWO            gp2            7m
```

**Observed:**
- MySQL: 1 pod running, 5Gi PVC bound to EBS
- Ollama: 1 pod running, 10Gi PVC bound to EBS (TinyLlama model pulled via PostStart hook)
- BankApp: 2 pods running, managed by HPA
- Services: 3 ClusterIP services (bankapp:8080, mysql:3306, ollama:11434)

---
### All Pods Running
![kubectl get all](Screenshot%202026-05-01%20213439.png)


## Task 2: Gateway API and Application Access

### Envoy Gateway Setup

```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version v1.4.0 \
  -n envoy-gateway-system --create-namespace \
  --wait
```

### Gateway Configuration

```bash
kubectl apply -f k8s/gateway.yml

# Watch for NLB provisioning
kubectl get gateway -n bankapp -w
# NAME               CLASS                ADDRESS                                          PROGRAMMED   AGE
# bankapp-gateway    eg                   xxxx.elb.us-west-2.amazonaws.com                True         3m

export APP_URL=$(kubectl get gateway bankapp-gateway -n bankapp \
  -o jsonpath='{.status.addresses[0].value}')
echo "AI-BankApp URL: http://$APP_URL"
```

### Application Tests

```bash
# Health check
curl -s http://$APP_URL/actuator/health | python3 -m json.tool
# {
#   "status": "UP",
#   "components": {
#     "db": { "status": "UP" },
#     "diskSpace": { "status": "UP" }
#   }
# }

# HTTP status check
curl -s -o /dev/null -w "%{http_code}" http://$APP_URL
# 200
```

### Application Features Validated

- User registration and login
- Deposit, withdrawal, and transfer operations
- AI chatbot (Ollama TinyLlama) responding to financial questions
- Dark/light mode toggle
- Session persistence via cookie-based routing

---

## Task 3: Monitoring Stack

### Prometheus + Grafana Deployment

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install monitoring prometheus-community/kube-prometheus-stack \
  -n monitoring --create-namespace \
  --set grafana.adminPassword=admin123 \
  --set prometheus.prometheusSpec.retention=3d \
  --set prometheus.prometheusSpec.resources.requests.memory=256Mi \
  --set prometheus.prometheusSpec.resources.requests.cpu=100m \
  --wait --timeout 600s

kubectl get pods -n monitoring
# NAME                                          READY   STATUS    RESTARTS   AGE
# alertmanager-monitoring-kube-prometheus-...   2/2     Running   0          4m
# monitoring-grafana-xxx                        3/3     Running   0          4m
# monitoring-kube-prometheus-operator-xxx       1/1     Running   0          4m
# monitoring-kube-state-metrics-xxx             1/1     Running   0          4m
# monitoring-prometheus-node-exporter-xxx       1/1     Running   0          4m
# prometheus-monitoring-kube-prometheus-...     2/2     Running   0          3m
```

### ServiceMonitor for BankApp

```yaml
# bankapp-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: bankapp-monitor
  namespace: monitoring
  labels:
    release: monitoring
spec:
  namespaceSelector:
    matchNames:
      - bankapp
  selector:
    matchLabels:
      app: bankapp
  endpoints:
    - port: "8080"
      path: /actuator/prometheus
      interval: 15s
```

```bash
kubectl apply -f bankapp-servicemonitor.yaml
```

### Grafana Access

```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
# Open http://localhost:3000  |  admin / admin123
```

**Dashboards used:**
- `Kubernetes / Compute Resources / Namespace (Pods)` → bankapp namespace
- `Kubernetes / Compute Resources / Pod` → individual pod drill-down
- `Node Exporter / Nodes` → EKS worker node health

### PromQL Queries

```promql
# JVM heap memory in use
jvm_memory_used_bytes{namespace="bankapp", area="heap"}

# HTTP request rate (5-min window)
rate(http_server_requests_seconds_count{namespace="bankapp"}[5m])

# HTTP p95 latency
histogram_quantile(0.95,
  rate(http_server_requests_seconds_bucket{namespace="bankapp"}[5m])
)

# Active DB connections
hikaricp_connections_active{namespace="bankapp"}

# GC pause time rate
rate(jvm_gc_pause_seconds_sum{namespace="bankapp"}[5m])
```

---

## Task 4: End-to-End Validation Checklist

### Application Layer

| Check | Command | Result |
|-------|---------|--------|
| All pods running | `kubectl get pods -n bankapp` | ✅ MySQL, Ollama, 2× BankApp |
| Health endpoint | `curl http://$APP_URL/actuator/health` | ✅ `{"status":"UP"}` |
| HPA active | `kubectl get hpa -n bankapp` | ✅ min:2 max:10, CPU target 70% |
| Prometheus metrics | `curl http://$APP_URL/actuator/prometheus` | ✅ Returns JVM + HTTP metrics |

### Data Layer

| Check | Command | Result |
|-------|---------|--------|
| MySQL healthy | `kubectl exec -n bankapp deploy/mysql -- mysqladmin ping -h localhost -uroot -pTest@123` | ✅ `mysqld is alive` |
| PVCs bound | `kubectl get pvc -n bankapp` | ✅ mysql-pvc 5Gi Bound, ollama-pvc 10Gi Bound |
| Ollama model loaded | `kubectl exec -n bankapp deploy/ollama -- ollama list` | ✅ `tinyllama:latest` present |

### Infrastructure Layer

| Check | Command | Result |
|-------|---------|--------|
| Nodes healthy | `kubectl get nodes` | ✅ 3 nodes Ready |
| Node resource usage | `kubectl top nodes` | ✅ CPU/memory within limits |
| Gateway active | `kubectl get gateway -n bankapp` | ✅ Programmed=True, NLB address assigned |
| Monitoring running | `kubectl get pods -n monitoring` | ✅ All pods Running |

### Security Layer

| Check | Command | Result |
|-------|---------|--------|
| Non-root user | `kubectl exec -n bankapp deploy/bankapp -- whoami` | ✅ `devsecops` |
| Secret not exposed | `kubectl get secret bankapp-secret -n bankapp -o yaml \| grep -c "MYSQL_ROOT_PASSWORD"` | ✅ Returns `1` (key name only, value base64-encoded) |

---

## Task 5: 3-Day EKS Journey — Concepts Mapped

| Day | What Was Built | AI-BankApp Connection |
|-----|---------------|----------------------|
| 81 | EKS cluster via Terraform, kubectl connection, first manual deploy | Used `terraform/` configs to provision VPC, node group, IAM roles, 6 add-ons |
| 82 | Gateway API + Envoy, TLS with cert-manager, EBS storage, session persistence | Used `k8s/gateway.yml`, `k8s/cert-manager.yml`, `k8s/pv.yml`, StorageClass gp2 |
| 83 | Full production deployment, HPA, monitoring, end-to-end validation | Complete stack: Spring Boot + MySQL + Ollama + Gateway + Prometheus + Grafana |

### What the AI-BankApp EKS Setup Includes

- Terraform-provisioned VPC with 3-AZ networking (public + private subnets, NAT gateway)
- Managed node group (3× t3.medium) with cluster autoscaler support
- 6 EKS add-ons: CoreDNS, VPC CNI, kube-proxy, Pod Identity, EBS CSI Driver, Metrics Server
- ArgoCD pre-installed (used on Days 84–86 for GitOps)
- Gateway API with Envoy for external traffic management
- cert-manager for automated HTTPS certificate provisioning
- Cookie-based session persistence for stateful Spring Boot sessions
- EBS persistent volumes for MySQL (5Gi) and Ollama (10Gi)
- HPA configured with CPU target 70%, min 2, max 10 replicas
- Spring Boot Actuator + Micrometer → Prometheus metrics at `/actuator/prometheus`
- Init containers for dependency ordering (bankapp waits for MySQL)
- PostStart lifecycle hook for Ollama model pull (`ollama pull tinyllama`)

### What a Real Production Deployment Would Add

- **DNS**: Route 53 + ExternalDNS for human-readable URLs
- **Network Policies**: Restrict pod-to-pod traffic (MySQL only reachable from BankApp)
- **Pod Disruption Budgets**: Safe node draining during upgrades
- **External Secrets Operator**: Pull secrets from AWS Secrets Manager instead of k8s secrets
- **Database backups**: Scheduled MySQL dumps to S3 via CronJob
- **Log aggregation**: Loki + Promtail for centralized logging (covered Day 75)
- **Multi-environment**: Separate dev and prod clusters, promoted via ArgoCD
- **WAF**: AWS WAF in front of NLB for SQL injection / XSS protection

---

## Task 6: Teardown Procedure

### 1. Delete Workloads

```bash
# Monitoring
helm uninstall monitoring -n monitoring

# Gateway (releases NLB — wait for this before Terraform destroy)
kubectl delete -f k8s/gateway.yml

# BankApp stack
kubectl delete -f k8s/hpa.yml
kubectl delete -f k8s/bankapp-deployment.yml
kubectl delete -f k8s/ollama-deployment.yml
kubectl delete -f k8s/mysql-deployment.yml
kubectl delete -f k8s/service.yml
kubectl delete -f k8s/secrets.yml
kubectl delete -f k8s/configmap.yml
kubectl delete -f k8s/pvc.yml
kubectl delete -f k8s/pv.yml
kubectl delete -f k8s/namespace.yml

# Supporting Helm releases
helm uninstall envoy-gateway -n envoy-gateway-system
helm uninstall cert-manager -n cert-manager

# Namespaces
kubectl delete namespace monitoring envoy-gateway-system cert-manager
```

### 2. Verify No Orphans

```bash
# No lingering LoadBalancers (would block Terraform destroy)
kubectl get svc -A | grep LoadBalancer

# No lingering PVCs (EBS volumes must be released)
kubectl get pvc -A
```

### 3. Terraform Destroy

```bash
cd terraform
terraform destroy
# Takes 10–15 minutes. Destroys:
# - EKS control plane and node groups
# - EC2 instances (t3.medium × 3)
# - NLB (if not already released above)
# - EBS volumes
# - VPC, subnets, NAT gateway, IGW
# - IAM roles and policies
# - ArgoCD Helm release
```

### 4. AWS Console Verification

| Resource | Expected State |
|----------|---------------|
| EKS → Clusters | No clusters |
| EC2 → Instances | No running instances |
| EC2 → Load Balancers | No load balancers |
| EC2 → Volumes | No unattached volumes |
| VPC | `bankapp-eks` VPC deleted |
| CloudFormation | No lingering stacks |

---

## Cost Report

| Resource | Hours | Unit Cost | Estimated Cost |
|----------|-------|-----------|----------------|
| 3× t3.medium nodes | ~72h (3 days) | ~$0.042/hr each | ~$9.07 |
| EKS control plane | ~72h | $0.10/hr | ~$7.20 |
| NLB | ~24h (Day 82–83) | ~$0.008/hr + LCUs | ~$0.50 |
| EBS volumes (15Gi total) | ~24h | ~$0.10/GB-month | ~$0.05 |
| NAT Gateway | ~72h | $0.045/hr + data | ~$3.50 |
| **Total** | | | **~$20–25** |

> Costs can be reduced by ~40% by tearing down overnight (stopping at end of Day 81 and Day 82).

---

## Key Takeaways from the 3-Day EKS Block

**Day 81 — Foundation:** Terraform is the right tool for EKS provisioning. `eksctl` is faster to start but Terraform gives you reproducible, versionable infrastructure. The 6 EKS add-ons (especially EBS CSI and Metrics Server) are non-negotiable for a production cluster.

**Day 82 — Networking & Storage:** Gateway API is a significant upgrade over Ingress — HTTPRoute is more expressive, the separation between cluster admins (Gateway) and app teams (HTTPRoute) is cleaner, and Envoy Gateway handles the translation to AWS NLB natively. EBS CSI with dynamic provisioning means PVCs just work.

**Day 83 — Production Readiness:** `kubectl wait --for=condition=ready` transforms deployment scripts from "run and pray" into deterministic ordered deployments. The Spring Boot Actuator + Micrometer + ServiceMonitor pattern is the standard JVM observability stack — once you've seen it, you'll use it everywhere. `terraform destroy` cleans up most things, but always manually verify EBS volumes and load balancers that Kubernetes created outside Terraform's knowledge.

**Overall:** Three days to go from `terraform init` to a fully observed, auto-scaling banking application with an AI chatbot — with monitoring, persistent storage, and a clean teardown. This is the shape of real production Kubernetes deployments.

---

## References

- Project repo: [TrainWithShubham/AI-BankApp-DevOps](https://github.com/TrainWithShubham/AI-BankApp-DevOps) (branch: `feat/gitops`)
- [EKS Add-ons documentation](https://docs.aws.amazon.com/eks/latest/userguide/eks-add-ons.html)
- [Gateway API — Envoy Gateway](https://gateway.envoyproxy.io/)
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Spring Boot Actuator Metrics](https://docs.spring.io/spring-boot/reference/actuator/metrics.html)
- [EBS CSI Driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver)
