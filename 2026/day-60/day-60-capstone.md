# Day 60 – Capstone: Deploy WordPress + MySQL on Kubernetes

## Overview

This capstone brings together ten days of Kubernetes concepts into a single production-style deployment: WordPress backed by MySQL, running in a dedicated namespace with persistent storage, self-healing, autoscaling, and optional Helm comparison.

---

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │           capstone namespace             │
                        │                                          │
  Browser / Port-Forward│                                          │
        ───────────────▶│  NodePort Service (wordpress)            │
                        │    port: 30080 → targetPort: 80          │
                        │         │                                │
                        │         ▼                                │
                        │  Deployment: wordpress (2 replicas)      │
                        │  ┌──────────────┐ ┌──────────────┐      │
                        │  │  wordpress-0 │ │  wordpress-1 │      │
                        │  │  (Pod)       │ │  (Pod)       │      │
                        │  └──────┬───────┘ └──────┬───────┘      │
                        │         │  envFrom         │  envFrom    │
                        │         ▼                  ▼             │
                        │  ConfigMap: wordpress-config             │
                        │    WORDPRESS_DB_HOST                     │
                        │    WORDPRESS_DB_NAME                     │
                        │         │                                │
                        │         │  secretKeyRef                  │
                        │         ▼                                │
                        │  Secret: mysql-secret                    │
                        │    MYSQL_ROOT_PASSWORD                   │
                        │    MYSQL_DATABASE                        │
                        │    MYSQL_USER                            │
                        │    MYSQL_PASSWORD                        │
                        │         │                                │
                        │         ▼                                │
                        │  Headless Service: mysql                 │
                        │    clusterIP: None, port: 3306           │
                        │         │                                │
                        │         ▼                                │
                        │  StatefulSet: mysql (1 replica)          │
                        │  ┌──────────────┐                        │
                        │  │   mysql-0    │                        │
                        │  │   (Pod)      │                        │
                        │  └──────┬───────┘                        │
                        │         │  volumeMount: /var/lib/mysql   │
                        │         ▼                                │
                        │  PersistentVolumeClaim (1Gi)             │
                        │                                          │
                        │  HPA: wordpress                          │
                        │    min: 2, max: 10, CPU: 50%             │
                        └─────────────────────────────────────────┘
```

### Resource Connections

| Resource | Connects To | Via |
|---|---|---|
| WordPress Deployment | mysql-secret | secretKeyRef (WORDPRESS_DB_USER, WORDPRESS_DB_PASSWORD) |
| WordPress Deployment | wordpress-config ConfigMap | envFrom |
| WordPress Pods | MySQL StatefulSet | DNS: `mysql-0.mysql.capstone.svc.cluster.local:3306` |
| NodePort Service | WordPress Pods | selector: `app: wordpress` |
| HPA | WordPress Deployment | scaleTargetRef |
| MySQL StatefulSet | PVC (1Gi) | volumeClaimTemplates → mountPath `/var/lib/mysql` |
| MySQL StatefulSet | mysql-secret | envFrom |
| Headless Service | MySQL StatefulSet | selector: `app: mysql` |

---

## Manifests

### Task 1: Namespace

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: capstone
```

```bash
kubectl apply -f namespace.yaml
kubectl config set-context --current --namespace=capstone
```

---

### Task 2: MySQL – Secret, Headless Service, StatefulSet

```yaml
# mysql-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: capstone
type: Opaque
stringData:
  MYSQL_ROOT_PASSWORD: rootpassword
  MYSQL_DATABASE: wordpress
  MYSQL_USER: wpuser
  MYSQL_PASSWORD: wppassword
```

```yaml
# mysql-headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: capstone
spec:
  clusterIP: None
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

```yaml
# mysql-statefulset.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: capstone
spec:
  serviceName: mysql
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          envFrom:
            - secretRef:
                name: mysql-secret
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
            limits:
              cpu: 500m
              memory: 1Gi
          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql
  volumeClaimTemplates:
    - metadata:
        name: mysql-data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

**Verification:**
```bash
kubectl exec -it mysql-0 -n capstone -- mysql -u wpuser -pwppassword -e "SHOW DATABASES;"
# Expected output includes: wordpress
```

---

### Task 3: WordPress – ConfigMap, Deployment

```yaml
# wordpress-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: wordpress-config
  namespace: capstone
data:
  WORDPRESS_DB_HOST: "mysql-0.mysql.capstone.svc.cluster.local:3306"
  WORDPRESS_DB_NAME: "wordpress"
```

```yaml
# wordpress-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  namespace: capstone
spec:
  replicas: 2
  selector:
    matchLabels:
      app: wordpress
  template:
    metadata:
      labels:
        app: wordpress
    spec:
      containers:
        - name: wordpress
          image: wordpress:latest
          envFrom:
            - configMapRef:
                name: wordpress-config
          env:
            - name: WORDPRESS_DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_USER
            - name: WORDPRESS_DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: MYSQL_PASSWORD
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /wp-login.php
              port: 80
            initialDelaySeconds: 60
            periodSeconds: 10
            failureThreshold: 6
          readinessProbe:
            httpGet:
              path: /wp-login.php
              port: 80
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
```

**Verification:**
```bash
kubectl get pods -n capstone -l app=wordpress
# Both pods show 1/1 Running
```

---

### Task 4: Expose WordPress – NodePort Service

```yaml
# wordpress-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  namespace: capstone
spec:
  type: NodePort
  selector:
    app: wordpress
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

**Access:**
```bash
# Minikube
minikube service wordpress -n capstone

# Kind / generic
kubectl port-forward svc/wordpress 8080:80 -n capstone
# Then open http://localhost:8080
```

✅ **Verified:** WordPress setup wizard loaded at http://localhost:8080. Completed wizard, created a test blog post titled "Hello from Day 60!".

---

### Task 5: Self-Healing and Persistence Tests

#### WordPress Pod Self-Healing

```bash
kubectl delete pod -l app=wordpress -n capstone
kubectl get pods -n capstone -w
```

- **Result:** Both WordPress pods were recreated within ~15 seconds by the Deployment controller. The site remained accessible throughout (one replica stayed up while the other restarted). Blog post was visible immediately after pods recovered.

#### MySQL Pod Recovery

```bash
kubectl delete pod mysql-0 -n capstone
kubectl get pods -n capstone -w
```

- **Result:** The StatefulSet automatically recreated `mysql-0` within ~20 seconds and reattached the existing PVC. After MySQL recovered, WordPress reconnected automatically and the blog post was still present — confirming data persistence via the PersistentVolume.

✅ **Persistence confirmed:** Blog post "Hello from Day 60!" survived deletion of both WordPress pods and the MySQL pod.

---

### Task 6: Horizontal Pod Autoscaler

```yaml
# wordpress-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: wordpress
  namespace: capstone
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: wordpress
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
```

```bash
kubectl apply -f wordpress-hpa.yaml
kubectl get hpa -n capstone
```

**Sample output:**
```
NAME        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
wordpress   Deployment/wordpress   8%/50%    2         10        2          2m
```

✅ **HPA verified:** Shows min=2, max=10, CPU target=50%, currently at 2 replicas with low load.

---

### Task 7 (Bonus): Helm Comparison

```bash
# Add Bitnami repo and install
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
kubectl create namespace helm-wp
helm install wp-helm bitnami/wordpress -n helm-wp
```

| Aspect | Manual (capstone) | Helm (bitnami/wordpress) |
|---|---|---|
| Resources created | ~8 (manually crafted) | ~20+ (auto-generated) |
| Database | MySQL 8.0 | MariaDB (MySQL-compatible) |
| Control | Full — every field explicit | Limited — values.yaml overrides only |
| Repeatability | Manual apply steps | Single `helm install` command |
| Upgrades | Manual manifest edits | `helm upgrade` with diff tracking |
| Customization | Unlimited | Bounded by chart's exposed values |
| Learning value | High — see every piece | Low — abstraction hides wiring |

**Verdict:** Helm is faster for standard deployments; manual gives deeper understanding and full control. For production, Helm with a custom values file (or Helmfile) is preferred. For learning, manual is essential.

```bash
# Cleanup
helm uninstall wp-helm -n helm-wp
kubectl delete namespace helm-wp
```

---

### Task 8: Final State and Cleanup

**`kubectl get all -n capstone` (before cleanup):**

```
NAME                             READY   STATUS    RESTARTS   AGE
pod/mysql-0                      1/1     Running   0          18m
pod/wordpress-6d8f9b7c4d-k2xpq   1/1     Running   0          12m
pod/wordpress-6d8f9b7c4d-n9vtz   1/1     Running   0          12m

NAME                TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/mysql       ClusterIP   None            <none>        3306/TCP       20m
service/wordpress   NodePort    10.96.142.88    <none>        80:30080/TCP   16m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/wordpress   2/2     2            2           14m

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/wordpress-6d8f9b7c4d   2         2         2       14m

NAME                                  READY   AGE
statefulset.apps/mysql                1/1     20m

NAME                                        REFERENCE              TARGETS   MINPODS   MAXPODS   REPLICAS
horizontalpodautoscaler.autoscaling/wordpress   Deployment/wordpress   8%/50%    2         10        2
```

**Cleanup:**
```bash
kubectl delete namespace capstone
kubectl config set-context --current --namespace=default
```

✅ **Confirmed:** Deleting the namespace removed all resources — Pods, Services, Deployments, StatefulSet, HPA, ConfigMap, Secret, and PVCs — in a single command.

---

## Concept Map

| Concept | Day Learned | Used In This Capstone |
|---|---|---|
| Namespace | Day 52 | `capstone` namespace isolating all resources |
| Deployments | Day 52 | WordPress Deployment with 2 replicas |
| Services (NodePort) | Day 53 | Exposing WordPress on port 30080 |
| Services (Headless) | Day 54 | MySQL DNS-based discovery |
| Secrets | Day 54 | MySQL credentials (stringData) |
| ConfigMaps | Day 54 | WordPress DB host and name |
| PersistentVolumeClaims | Day 55 | MySQL data at `/var/lib/mysql` |
| StatefulSets | Day 56 | MySQL with stable identity and storage |
| Resource Limits | Day 57 | CPU/memory requests and limits on all containers |
| Liveness/Readiness Probes | Day 57 | WordPress probe on `/wp-login.php` |
| Horizontal Pod Autoscaler | Day 58 | Scale WordPress 2–10 replicas at 50% CPU |
| Helm | Day 59 | Bitnami WordPress comparison install |

**12 concepts. 1 deployment. All connected.**

---

## Reflection

### What was hardest

**MySQL startup timing** was the trickiest part. WordPress pods came up before MySQL finished initializing, causing CrashLoopBackOff errors. The fix was adding `initialDelaySeconds: 60` to the liveness probe and `initialDelaySeconds: 30` to the readiness probe — giving MySQL time to fully boot before WordPress attempted connections.

**DNS resolution** also required care. The `WORDPRESS_DB_HOST` must exactly match the StatefulSet DNS pattern `<pod>.<service>.<namespace>.svc.cluster.local`. A single typo here causes silent connection failures that are hard to debug.

### What clicked

**StatefulSets vs Deployments** finally made sense in practice. The stable pod name `mysql-0` is what makes the DNS-based connection from WordPress reliable. A regular Deployment would give a random pod name and wouldn't work for this pattern.

**The namespace as a blast radius boundary** was satisfying — `kubectl delete namespace capstone` cleanly removed every resource, including PVCs, in one command.

### What I would add for production

| Enhancement | Why |
|---|---|
| `PodDisruptionBudget` for WordPress | Ensure at least 1 replica stays up during node drains |
| MySQL replicas (read replicas) | Single `mysql-0` is a SPOF; use MySQL Operator or Galera cluster |
| `NetworkPolicy` | Restrict WordPress → MySQL traffic; deny all others |
| `Ingress` with TLS | Replace NodePort with a proper domain + cert-manager |
| External Secrets Operator | Pull secrets from Vault / AWS Secrets Manager instead of in-cluster Secrets |
| Resource `LimitRange` on namespace | Enforce guardrails for any new pods added to the namespace |
| Monitoring | Prometheus + Grafana stack with WordPress and MySQL exporters |
| Backup CronJob | Scheduled `mysqldump` to object storage (S3/GCS) |

---

*Day 60 complete. Ten days, twelve concepts, one working stack.*
