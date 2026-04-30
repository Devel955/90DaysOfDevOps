# Day 82 — EKS Networking with Gateway API and Persistent Storage

## Overview

Today's focus: production-grade networking for the AI-BankApp on EKS using the Kubernetes Gateway API with Envoy Gateway, automated TLS via cert-manager, and EBS persistent storage for stateful workloads.

---

## Gateway API Architecture

```
[Internet]
     |
[AWS NLB]  ← auto-provisioned by Envoy Gateway
     |
[Gateway: bankapp-gateway]  (namespace: bankapp)
  ├── Listener: HTTP  (port 80)
  └── Listener: HTTPS (port 443, TLS terminated via bankapp-tls secret)
     |
[HTTPRoute: bankapp-route]
  └── match: PathPrefix /
     |
[Service: bankapp-service:8080]
     |
[Pods: bankapp x2–4]  ← session affinity via BANKAPP_AFFINITY cookie
```

---

## Task 1: Gateway API vs Ingress

| Feature | Ingress | Gateway API |
|---|---|---|
| API maturity | Stable but limited | GA since Kubernetes 1.26 |
| Traffic splitting | Not supported | Built-in (weighted backends) |
| Header matching | Annotation-dependent | Native HTTPRoute rules |
| Role separation | Single resource | GatewayClass → Gateway → HTTPRoute |
| TLS management | Annotation-based | Native TLS config in Gateway listeners |
| Session affinity | Not standardized | BackendTrafficPolicy (Envoy) |
| Extensibility | Controller-specific annotations | Policy attachment resources |

**Key insight**: Gateway API separates concerns by role — infrastructure teams manage `GatewayClass`, platform/ops teams manage `Gateway`, and developers manage `HTTPRoute`. This is a significant improvement over the annotation-heavy Ingress model where everything lives in a single resource.

---

## Task 2: Install Envoy Gateway

```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version v1.4.0 \
  -n envoy-gateway-system --create-namespace \
  --wait

kubectl get pods -n envoy-gateway-system
kubectl get gatewayclass
```

Install Gateway API CRDs (if not already present):

```bash
kubectl get crd gateways.gateway.networking.k8s.io 2>/dev/null || \
  kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

**Expected output:**
```
NAME             CONTROLLER                                    ACCEPTED   AGE
envoy-gateway    gateway.envoyproxy.io/gatewayclass-controller   True      2m
```

---

## Task 3: Gateway API Resources Explained

### 1. GatewayClass — Infrastructure Layer

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

Defines which controller implementation handles Gateway resources. One GatewayClass per cluster (per controller). Think of it as the "driver" for the networking stack.

---

### 2. Gateway — Operations Layer

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: bankapp-gateway
  namespace: bankapp
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      hostname: <your-ip>.nip.io
      tls:
        mode: Terminate
        certificateRefs:
          - name: bankapp-tls
```

When applied, Envoy Gateway automatically provisions an **AWS Network Load Balancer (NLB)**. The Gateway defines the entry points (listeners) and TLS termination. The `bankapp-tls` secret is populated by cert-manager.

---

### 3. HTTPRoute — Developer Layer

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: bankapp-route
  namespace: bankapp
spec:
  parentRefs:
    - name: bankapp-gateway
      sectionName: https
    - name: bankapp-gateway
      sectionName: http
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: bankapp-service
          port: 8080
```

Routes incoming requests to the BankApp service. Multiple HTTPRoutes can attach to the same Gateway, enabling multi-team workflows without sharing YAML files.

---

### 4. BackendTrafficPolicy — Cookie-Based Session Affinity

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: bankapp-session
  namespace: bankapp
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: bankapp-route
  loadBalancer:
    type: ConsistentHash
    consistentHash:
      type: Cookie
      cookie:
        name: BANKAPP_AFFINITY
        ttl: 3600s
```

**Why cookie-based session affinity is required:**

The AI-BankApp uses **Spring Security with form-based login**. HTTP sessions are stored in-memory on individual pods. Without session affinity:

1. User logs in → request hits Pod A → session created on Pod A
2. User navigates to `/dashboard` → request hits Pod B → no session found → **user is logged out**

The `BANKAPP_AFFINITY` cookie ensures all requests from a given user are consistently routed to the same pod via consistent hashing. The TTL of 3600s (1 hour) matches a typical banking session timeout.

Apply all Gateway resources:

```bash
kubectl apply -f k8s/gateway.yml

# Wait for NLB provisioning (takes 1–2 minutes)
kubectl get gateway -n bankapp -w

# Get external address
export GATEWAY_IP=$(kubectl get gateway bankapp-gateway -n bankapp \
  -o jsonpath='{.status.addresses[0].value}')
echo "App URL: http://$GATEWAY_IP"
```

---

## Task 4: cert-manager and Automated TLS

### Install cert-manager

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update

helm install cert-manager jetstack/cert-manager \
  -n cert-manager --create-namespace \
  --set crds.enabled=true \
  --wait

kubectl get pods -n cert-manager
```

### ClusterIssuer Configuration

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-account-key
    solvers:
      - http01:
          gatewayHTTPRoute:
            parentRefs:
              - group: gateway.networking.k8s.io
                kind: Gateway
                name: bankapp-gateway
                namespace: bankapp
```

### TLS Certificate Flow

```
cert-manager
  │
  ├─ 1. Requests certificate from Let's Encrypt ACME endpoint
  │
  ├─ 2. Let's Encrypt issues HTTP-01 challenge
  │      (GET /.well-known/acme-challenge/<token>)
  │
  ├─ 3. cert-manager creates a temporary HTTPRoute to serve the challenge
  │
  ├─ 4. Let's Encrypt verifies the response via the NLB → Gateway → HTTPRoute
  │
  ├─ 5. Certificate issued and stored in Secret: bankapp-tls
  │
  └─ 6. Gateway Listener references bankapp-tls for HTTPS termination
```

### Quick DNS with nip.io

```bash
export HOSTNAME="${GATEWAY_IP}.nip.io"
echo "HTTPS URL: https://$HOSTNAME"
# Example: https://1.2.3.4.nip.io
```

`nip.io` is a wildcard DNS service — `1.2.3.4.nip.io` resolves to `1.2.3.4`. No domain registration needed for TLS testing.

---

## Task 5: EBS Persistent Storage

### Storage Flow

```
StorageClass (gp3)
     │
     │  (dynamic provisioning via EBS CSI driver)
     ▼
PersistentVolumeClaim (mysql-pvc: 5Gi, ollama-pvc: 10Gi)
     │
     │  (Kubernetes binds PVC to a PV)
     ▼
PersistentVolume (auto-created)
     │
     │  (EBS CSI attaches block device to EC2 node)
     ▼
EBS Volume (AWS)
     │
     │  (mounted into pod at /var/lib/mysql, /root/.ollama)
     ▼
Pod (mysql / ollama)
```

### Check Storage Status

```bash
kubectl get storageclass gp3
kubectl get pvc -n bankapp
kubectl get pv

# Find EBS volumes in AWS
aws ec2 describe-volumes \
  --filters "Name=tag:kubernetes.io/created-by,Values=ebs.csi.aws.com" \
  --query "Volumes[*].{ID:VolumeId,Size:Size,AZ:AvailabilityZone,State:State}" \
  --output table \
  --region us-west-2
```

### Key EBS Concepts on EKS

| Concept | Explanation |
|---|---|
| `WaitForFirstConsumer` | Volume is created in the same AZ as the scheduled pod — prevents cross-AZ attachment failures |
| `ReadWriteOnce` | EBS can only attach to **one EC2 node at a time** |
| `Recreate` strategy | MySQL and Ollama use `Recreate` (not `RollingUpdate`) because a new pod cannot attach the EBS volume while the old pod is still running |
| `allowVolumeExpansion: true` | Volumes can be expanded in-place without recreating the PVC |
| `gp3` | Latest-gen SSD: 3000 IOPS baseline, 125 MB/s throughput, ~20% cheaper than gp2 |

### Persistence Test

```bash
# Verify current data
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"

# Delete the pod (simulates crash/restart)
kubectl delete pod -n bankapp -l app=mysql

# Watch recreation
kubectl get pods -n bankapp -l app=mysql -w

# Confirm data survived
kubectl exec -n bankapp deploy/mysql -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"
```

The EBS volume persists independently of the pod lifecycle — deleting the pod does not delete the volume or its data.

---

## Task 6: HPA and Resource Budget

```bash
kubectl get hpa -n bankapp
kubectl top nodes
kubectl top pods -n bankapp
```

### Resource Budget (3x t3.medium nodes = 6 vCPU / 12 GiB RAM)

| Component | CPU Request | Memory Request | Max Instances |
|---|---|---|---|
| BankApp | 250m | 256Mi | 4 pods (HPA max) |
| MySQL | 250m | 256Mi | 1 pod |
| Ollama | 900m | 2Gi | 1 pod |
| Init containers | 50m | 32Mi | Temporary |
| System pods | ~500m | ~500Mi | Per node |
| **Total at max scale** | **~2900m + system** | **~6.3Gi + system** | |
| **Available** | **6000m** | **12Gi** | |

Ollama is the dominant consumer at 900m CPU and 2Gi memory. At 4 BankApp replicas, the cluster is close to capacity — this is intentional sizing for a cost-effective demo environment.

---

## Cleanup (Keep Cluster for Day 83)

```bash
kubectl delete -f k8s/gateway.yml 2>/dev/null
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
```

---

## Key Takeaways

1. **Gateway API is the future** — officially GA since Kubernetes 1.26, it replaces the annotation-heavy Ingress pattern with a role-based, extensible model.

2. **Envoy Gateway auto-provisions AWS NLB** — applying a `Gateway` resource is all it takes; no manual load balancer configuration needed.

3. **Cookie affinity is required for stateful Spring apps** — `BackendTrafficPolicy` with `ConsistentHash/Cookie` ensures Spring Security sessions are not invalidated by load balancing.

4. **EBS volumes are AZ-locked** — `WaitForFirstConsumer` is critical to prevent volumes being provisioned in the wrong AZ before pod scheduling.

5. **cert-manager integrates natively with Gateway API** — the HTTP-01 solver uses `HTTPRoute` resources directly, no Ingress needed.

6. **`Recreate` strategy is mandatory for RWO volumes** — rolling updates would attempt to attach the same EBS volume to two nodes simultaneously, which is not supported.

---

*#90DaysOfDevOps #DevOpsKaJosh #TrainWithShubham*
