📘 Day 56 – Kubernetes StatefulSets
📌 Task Overview

Today I explored StatefulSets in Kubernetes, which are used for stateful applications like databases (MySQL, PostgreSQL, MongoDB, etc.).

Unlike Deployments, StatefulSets provide:
Stable pod identities
Ordered deployment & scaling
Persistent storage per pod
Stable DNS names

⚖️ Deployment vs StatefulSet
| Feature   | Deployment         | StatefulSet                        |
| --------- | ------------------ | ---------------------------------- |
| Pod Names | Random             | Stable (`web-0`, `web-1`, `web-2`) |
| Startup   | Parallel           | Ordered                            |
| Storage   | Shared             | Per Pod PVC                        |
| DNS       | No stable identity | Stable DNS                         |
| Use Case  | Stateless apps     | Databases                          |

🚀 Task 1: Deployment Issue (Stateless Problem)
Deployment YAML
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx
        image: nginx
Observation
kubectl get pods

Pods created:
nginx-deploy-xxxxx

👉 Pod names are random
👉 Not suitable for databases

🌐 Task 2:
Headless Service
YAML
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
    - port: 80
Apply
kubectl apply -f headless-service.yaml
kubectl get svc
Verification
CLUSTER-IP = None

🧱 Task 3: 
StatefulSet Creation
YAML
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: nginx-headless
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: web-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 100Mi
Apply
kubectl apply -f statefulset-nginx.yaml
kubectl get pods -w
Output
web-0
web-1
web-2

👉 Pods created in order
💾 PVC Verification
kubectl get pvc

Output:
web-data-web-0
web-data-web-1
web-data-web-2
🌐 Task 4: Stable DNS

DNS format:
<pod-name>.<service-name>.default.svc.cluster.local
Test
kubectl run -it --rm dns-test --image=busybox -- sh

Inside pod:
nslookup web-0.nginx-headless.default.svc.cluster.local
Verify

👉 IP matches:
kubectl get pods -o wide
💽 Task 5: Data Persistence
Write data
kubectl exec web-0 -- sh -c "echo 'Data from web-0' > /usr/share/nginx/html/index.html"
kubectl exec web-1 -- sh -c "echo 'Data from web-1' > /usr/share/nginx/html/index.html"
kubectl exec web-2 -- sh -c "echo 'Data from web-2' > /usr/share/nginx/html/index.html"

Verify
MSYS_NO_PATHCONV=1 kubectl exec web-0 -- sh -c "cat /usr/share/nginx/html/index.html"
Delete pod
kubectl delete pod web-0
After recreation
kubectl exec web-0 -- sh -c "cat /usr/share/nginx/html/index.html"
Result
Data from web-0

✅ Data persisted

📈 Task 6: Ordered Scaling
Scale up
kubectl scale statefulset web --replicas=5

Pods:
web-3
web-4
Scale down
kubectl scale statefulset web --replicas=3
PVC check
kubectl get pvc

👉 All 5 PVCs still exist
🧹 Task 7: Cleanup
kubectl delete statefulset web
kubectl delete service nginx-headless
kubectl get pvc

👉 PVCs NOT deleted automatically
Delete manually
kubectl delete pvc --all

🎯 Key Learnings
StatefulSets provide stable identity
Each pod gets dedicated storage
Data survives pod restart
Pods scale in order
DNS is stable per pod
PVCs are not auto-deleted

🧠 Why StatefulSets?
Stateful apps require:
Fixed identity (db-0, db-1)
Persistent storage
Ordered startup

👉 Deployment cannot guarantee this
👉 StatefulSet solves it
