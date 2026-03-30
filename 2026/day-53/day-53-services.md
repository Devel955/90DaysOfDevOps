# Day 53 – Kubernetes Services
## 💡 Why Kubernetes Services?

In Kubernetes, Pods are ephemeral and their IP addresses are not stable. Whenever a Pod restarts or gets replaced, it receives a new IP address. This makes direct communication between Pods unreliable.
Kubernetes Services solve this problem by providing a stable networking abstraction.
They:
- Provide a **stable IP address and DNS name**
- Enable **load balancing across Pods**
- Decouple application access from Pod lifecycle
- Ensure **high availability and fault tolerance**

Services act as a bridge between clients and Pods, making communication reliable in distributed systems.
## 🚀 Overview

Today I implemented Kubernetes Services to expose a Deployment using different service types:
- ClusterIP
- NodePort
- LoadBalancer

I also verified internal communication, DNS resolution, and external access.
Task 1
Created a Deployment with 3 replicas:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
Verification
kubectl apply -f app-deployment.yaml
kubectl get pods -o wide

IPs

## Task 2: ClusterIP Service
apiVersion: v1
kind: Service
metadata:
  name: web-app-clusterip
spec:
  type: ClusterIP
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
✅ Verification
kubectl apply -f clusterip-service.yaml
kubectl get services
✔ Stable ClusterIP assigned

🧪 Testing (Inside Cluster)
kubectl run test-client --image=busybox --rm -it -- sh
wget -qO- http://web-app-clusterip

✔ Successfully received Nginx response

🌐 Task 3: DNS Resolution

Kubernetes automatically creates DNS entries:

web-app-clusterip.default.svc.cluster.local
✅ Verification
nslookup web-app-clusterip

✔ DNS resolved to ClusterIP

🟠 Task 4: NodePort Service
apiVersion: v1
kind: Service
metadata:
  name: web-app-nodeport
spec:
  type: NodePort
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080
✅ Verification
kubectl apply -f nodeport-service.yaml
kubectl get services
kubectl get endpoints web-app-nodeport

✔ Endpoints correctly mapped to all Pods

🧪 Testing (Port Forward)
kubectl port-forward service/web-app-nodeport 9090:80
curl http://localhost:9090

✔ Nginx Welcome Page received

⚠️ Note

Direct access using localhost:30080 did not work due to kind + Docker + WSL networking limitations, but the service was functioning correctly.

🟢 Task 5: LoadBalancer Service
apiVersion: v1
kind: Service
metadata:
  name: web-app-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
  - port: 80
    targetPort: 80
✅ Verification
kubectl apply -f loadbalancer-service.yaml
kubectl get services

✔ EXTERNAL-IP = <pending> (expected in local cluster)

📌 Explanation
Local clusters (kind, Docker Desktop) do not support cloud load balancers, so EXTERNAL-IP remains pending.

🔍 Task 6: Service Comparison
Type	Access	Use Case
ClusterIP	Internal	Service communication
NodePort	External (Node Port)	Testing
LoadBalancer	External (Cloud)	Production

🔎 Task 7: Endpoints
kubectl get endpoints web-app-nodeport
✔ Service correctly routing traffic to Pods


