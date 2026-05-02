# Day 83 — EKS Project: AI-BankApp Production Deployment

## Task 1: Deploy the Complete AI-BankApp Stack

### Cluster Running
```bash
kubectl get nodes
```
![kubectl get nodes](Screenshot%202026-05-01%20213033.png)

### Deploy Stack
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
```
![Deploy Stack](Screenshot%202026-05-01%20105349.png)

### Wait for Dependencies
```bash
echo "Waiting for MySQL..."
kubectl wait --for=condition=ready pod -l app=mysql -n bankapp --timeout=120s

echo "Waiting for Ollama (this takes 2-5 minutes for model pull)..."
kubectl wait --for=condition=ready pod -l app=ollama -n bankapp --timeout=600s
```
![MySQL and Ollama Ready](Screenshot%202026-05-01%20213215.png)

### Application
```bash
kubectl apply -f k8s/bankapp-deployment.yml
kubectl apply -f k8s/hpa.yml
```
![BankApp Deploy](Screenshot%202026-05-01%20213256.png)

### Wait for BankApp
```bash
echo "Waiting for BankApp..."
kubectl wait --for=condition=ready pod -l app=bankapp -n bankapp --timeout=300s
```
![BankApp Ready](Screenshot%202026-05-01%20213353.png)

### Verify Everything Running
```bash
kubectl get all -n bankapp
kubectl get pvc -n bankapp
```
![kubectl get all](Screenshot%202026-05-01%20213439.png)

---

## Task 2: Set Up Gateway API and Access the App

### Install Envoy Gateway
```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version v1.4.0 \
  -n envoy-gateway-system --create-namespace \
  --wait 2>/dev/null || echo "Already installed"
```
![Envoy Gateway](Screenshot%202026-05-01%20213703.png)

### Apply Gateway Configuration
```bash
kubectl apply -f k8s/gateway.yml
kubectl get gateway -n bankapp -w
```
![Gateway](Screenshot%202026-05-01%20213756.png)

### Get App URL
```bash
export APP_URL=$(kubectl get gateway bankapp-gateway -n bankapp -o jsonpath='{.status.addresses[0].value}')
echo "AI-BankApp URL: http://$APP_URL"
```
![App URL](Screenshot%202026-05-01%20214015.png)

### Test Application
```bash
curl -s http://$APP_URL/actuator/health | python3 -m json.tool
curl -s -o /dev/null -w "%{http_code}" http://$APP_URL
```
![Health Check](Screenshot%202026-05-01%20215336.png)

### BankApp Running in Browser
![BankApp Login](Screenshot%202026-05-01%20221224.png)

![BankApp Dashboard](Screenshot%202026-05-01%20221409.png)

![Transaction History](Screenshot%202026-05-01%20222514.png)

---

## Task 3: Deploy the Monitoring Stack

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
```
![Monitoring Install](Screenshot%202026-05-01%20222705.png)

### Verify Monitoring Pods
```bash
kubectl get pods -n monitoring
```
![Monitoring Pods](Screenshot%202026-05-01%20222514.png)

### Access Grafana
```bash
kubectl port-forward svc/monitoring-grafana -n monitoring 3000:80
```
Open http://localhost:3000 — Login: admin / admin123

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

### Access Prometheus
```bash
kubectl port-forward svc/monitoring-kube-prometheus-prometheus -n monitoring 9090:9090
```
Open http://localhost:9090

### PromQL Queries

**JVM memory usage**
```promql
jvm_memory_used_bytes{namespace="bankapp"}
```
![Prometheus JVM](Screenshot%202026-05-01%20225449.png)

**HTTP request rate**
```promql
rate(http_server_requests_seconds_count{namespace="bankapp"}[5m])
```

**HTTP p95 latency**
```promql
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{namespace="bankapp"}[5m]))
```
### PromQL Queries

**JVM memory usage**
```promql
jvm_memory_used_bytes{namespace="bankapp"}
```
![Prometheus JVM](Screenshot%202026-05-01%20225449.png)

**HTTP request rate**
```promql
rate(http_server_requests_seconds_count{namespace="bankapp"}[5m])
```

**HTTP p95 latency**
```promql
histogram_quantile(0.95, rate(http_server_requests_seconds_bucket{namespace="bankapp"}[5m]))
```

### Grafana Dashboards
![Grafana BankApp](Screenshot%202026-05-01%20230144.png)

![Grafana Monitoring](Screenshot%202026-05-01%20230103.png)

![Grafana kube-system](Screenshot%202026-05-01%20230314.png)

---

## Task 4: End-to-End Validation Checklist

### Application Layer
```bash
kubectl get pods -n bankapp
curl -s http://$APP_URL/actuator/health
kubectl get hpa -n bankapp
```
![Application Layer](Screenshot%202026-05-01%20230931.png)

### Data Layer
```bash
kubectl exec -n bankapp deploy/mysql -- mysqladmin ping -h localhost -uroot -pTest@123
kubectl get pvc -n bankapp
kubectl exec -n bankapp deploy/ollama -- ollama list
```
![Data Layer](Screenshot%202026-05-01%20231119.png)

### Security Layer
```bash
kubectl exec -n bankapp deploy/bankapp -- whoami
kubectl get secret bankapp-secret -n bankapp -o yaml | grep -c "MYSQL_ROOT_PASSWORD"
```
![Security](Screenshot%202026-05-01%20231312.png)

---

## Task 5: Reflect on the Full EKS Journey

| Day | What You Built | AI-BankApp Connection |
|-----|---------------|----------------------|
| 81 | EKS cluster via Terraform | terraform/ configs, VPC, IAM |
| 82 | Gateway API, EBS storage | k8s/gateway.yml, k8s/pv.yml |
| 83 | Full production deployment | Complete stack + monitoring |

### What the AI-BankApp EKS Setup Includes
- Terraform-provisioned VPC with 3-AZ networking
- Managed node group with auto-scaling
- 6 EKS add-ons (CoreDNS, VPC CNI, kube-proxy, Pod Identity, EBS CSI, Metrics Server)
- Gateway API with Envoy for traffic management
- EBS persistent storage for MySQL and Ollama
- HPA with scale-up/down policies
- Spring Boot Actuator metrics for Prometheus

### What You Would Add for Production
- DNS with Route 53 and ExternalDNS
- Network Policies for pod-to-pod isolation
- Pod Disruption Budgets
- External Secrets Operator
- Database backups to S3
- Log aggregation with Loki

---

## Task 6: Complete Teardown

```bash
helm uninstall monitoring -n monitoring
kubectl delete -f k8s/gateway.yml
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
helm uninstall envoy-gateway -n envoy-gateway-system
cd terraform && terraform destroy
```
![Teardown](Screenshot%202026-05-01%20231814.png)

### Terraform Destroy Complete
![Terraform Destroy](Screenshot%202026-05-01%20233134.png)

---

## Architecture Diagram
Internet → NLB → Envoy Gateway → EKS Cluster
├── bankapp namespace
│   ├── BankApp (4 pods, HPA)
│   ├── MySQL (5Gi EBS)
│   └── Ollama TinyLlama (10Gi EBS)
└── monitoring namespace
├── Prometheus
└── Grafana
VPC (us-west-2) → 3 AZs → m7i-flex.large nodes (8GB RAM, Free Tier eligible)

## Key Takeaways

1. Set up a production Kubernetes cluster on Amazon EKS using Terraform
2. Deployed full application stack (app + database + AI) with autoscaling and EBS storage
3. Handled traffic routing using Gateway API and Envoy
4. Monitored system using Prometheus + Grafana dashboards
5. Clean teardown — 84 resources destroyed with terraform destroy
