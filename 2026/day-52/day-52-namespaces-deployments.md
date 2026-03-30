# 🚀 Day 52 – Kubernetes Namespaces & Deployments

## 📌 Overview
Today I learned how to organize resources using Namespaces and manage applications using Deployments in Kubernetes. I explored self-healing, scaling, rolling updates, and rollback mechanisms.

---
# 🔹 What are Namespaces?

Namespaces are used to organize and isolate resources within a Kubernetes cluster.

### Types:
- default → default namespace
- kube-system → system components
- kube-public → public resources
- kube-node-lease → node tracking

---
# 🔹 Task 1: Explore Namespaces

kubectl get namespaces
kubectl get pods -n kube-system

 # 🔹 Task 2: Create Namespaces
kubectl create namespace dev
kubectl create namespace staging

# 🔹 Task 3: Run Pods in Namespaces
kubectl run nginx-dev --image=nginx -n dev
kubectl run nginx-staging --image=nginx -n staging
kubectl get pods -A

# 🔹 Task 4: Create Deployment
YAML File
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
spec:
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
        image: nginx:1.24
        ports:
        - containerPort: 80

Apply Deployment
kubectl apply -f nginx-deployment.yaml
kubectl get deployments -n dev
kubectl get pods -n dev

# 🔹 Task 5: Self-Healing
kubectl get pods -n dev
kubectl delete pod <pod-name> -n dev
kubectl get pods -n dev

👉 Deleted pod automatically recreated by Deployment

# 🔹 Task 6: Scaling

Scale Up
kubectl scale deployment nginx-deployment --replicas=5 -n dev
kubectl get pods -n dev
Scale Down
kubectl scale deployment nginx-deployment --replicas=2 -n dev
kubectl get pods -n dev

# 🔹 Task 7: Rolling Update

kubectl set image deployment/nginx-deployment nginx=nginx:1.25 -n dev
kubectl rollout status deployment/nginx-deployment -n dev
kubectl rollout history deployment/nginx-deployment -n dev

 # 🔹 Task 8: Rollback
kubectl rollout undo deployment/nginx-deployment -n dev
kubectl rollout status deployment/nginx-deployment -n dev
Verify Image
kubectl describe deployment nginx-deployment -n dev

# 🔹 Task 9: Cleanup
kubectl delete deployment nginx-deployment -n dev
kubectl delete pod nginx-dev -n dev
kubectl delete pod nginx-staging -n staging
kubectl delete namespace dev staging production

🎯 Key Learnings
Namespaces isolate resources within a cluster
Deployments manage Pods and ensure availability
Self-healing automatically replaces failed Pods
Scaling adjusts the number of running Pods
Rolling updates ensure zero downtime deployments
Rollback restores previous working versions
