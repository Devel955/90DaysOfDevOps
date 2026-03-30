# 🚀 Day 51 – Kubernetes Pods & Manifests

## 📌 Overview
Today I explored Kubernetes Pods and learned how to define, create, and manage them using YAML manifests. I also understood imperative vs declarative approaches, labels, and validation techniques.

---

# 🔹 Task 1: Create Nginx Pod

## YAML
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ngnix-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80

Commands
kubectl apply -f nginx-pod.yaml
kubectl get pods
kubectl describe pod ngnix-pod
kubectl logs ngnix-pod
kubectl exec -it ngnix-pod -- sh
Verification
Pod running successfully
Accessed Nginx using port-forward
kubectl port-forward pod/ngnix-pod 8081:80

🔹 Task 2: BusyBox Pod
YAML
apiVersion: v1
kind: Pod
metadata:
  name: busybox-pod
  labels:
    app: busybox
    environment: dev
spec:
  containers:
  - name: busybox
    image: busybox
    command: ["sh", "-c", "echo Hello Anand && sleep 3600"]
Commands
kubectl apply -f busybox-pod.yaml
kubectl logs busybox-pod
Output
Hello Anand
🔹 Task 3: Imperative vs Declarative
Imperative
kubectl run redis-pod --image=redis
Declarative
kubectl apply -f nginx-pod.yaml
Learning
Imperative → quick commands
Declarative → production standard
🔹 Task 4: Generate YAML (Dry Run)
kubectl run test-pod --image=nginx --dry-run=client -o yaml
Learning
Used to generate YAML templates quickly
🔹 Task 5: Validation
kubectl apply -f nginx-pod.yaml --dry-run=client
kubectl apply -f nginx-pod.yaml --dry-run=server
🔹 Task 6: Labels & Filtering
kubectl get pods --show-labels
kubectl get pods -l app=nginx
kubectl get pods -l environment=dev
Add label
kubectl label pod ngnix-pod environment=production
🔹 Task 7: Cleanup
kubectl delete pod ngnix-pod
kubectl delete pod busybox-pod
kubectl delete pod redis-pod

🎯 Key Learnings
Pod is the smallest unit in Kubernetes
YAML defines desired state
BusyBox is useful for testing/debugging
Nginx serves web traffic
Labels help organize and filter resources
Imperative vs Declarative approaches
Pods are not self-healing
