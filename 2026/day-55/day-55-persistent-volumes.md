🚀 Day 55 – Persistent Volumes (PV) & Persistent Volume Claims (PVC)
Solving the problem of data loss in Kubernetes by implementing persistent storage.
.

📌 Overview

Containers in Kubernetes are ephemeral — when a Pod is deleted, all its internal data is lost.

This is a major problem for:

Databases
Logs
Stateful applications

To solve this, Kubernetes provides:

PersistentVolume (PV) → actual storage
PersistentVolumeClaim (PVC) → request for storage

❌ Task 1: Problem – Data Lost (emptyDir)

Created a Pod using emptyDir volume and wrote data:

kubectl apply -f emptydir-pod.yaml
kubectl exec emptydir-pod -- cat //data/message.txt

Deleted and recreated Pod:

kubectl delete pod emptydir-pod
kubectl apply -f emptydir-pod.yaml
kubectl exec emptydir-pod -- cat //data/message.txt
📸 Screenshots:
<img width="745" height="449" alt="Screenshot 2026-04-01 085021" src="https://github.com/user-attachments/assets/3262d1cb-bf1d-4358-888f-7273f59bf697" />
🔍 Observation
Timestamp changed after recreation
💡 Conclusion

emptyDir is ephemeral storage → data is lost when Pod is deleted

💾 Task 2: Create PersistentVolume (PV)

Created a PV:

capacity: 1Gi
accessModes: ReadWriteOnce
reclaimPolicy: Retain
hostPath: /tmp/k8s-pv-data
kubectl apply -f pv.yaml
kubectl get pv
📸 Screenshot:
<img width="745" height="449" alt="Screenshot 2026-04-01 085230" src="https://github.com/user-attachments/assets/72ec414c-2cc7-4b31-b1d9-e5466458f21a" />

🔍 Observation
PV Status: Available

Task 3: Create PersistentVolumeClaim (PVC)
kubectl apply -f pvc.yaml
kubectl get pvc
kubectl get pv
📸 Screenshot:
<img width="745" height="449" alt="Screenshot 2026-04-01 085347" src="https://github.com/user-attachments/assets/d6eba920-88a3-4ecc-b0ca-046b1d52d1c2" />

🔍 Observation
PVC Status: Bound
PV Status: Bound
VOLUME: my-pv
⚠️ Issue Faced

PVC was initially Pending because of StorageClass mismatch.

✅ Fix
storageClassName: ""

🔗 Task 4: Use PVC in Pod
kubectl apply -f pvc-pod.yaml
kubectl exec pvc-pod -- cat //data/message.txt

Deleted & recreated Pod:

kubectl delete pod pvc-pod
kubectl apply -f pvc-pod.yaml
kubectl exec pvc-pod -- cat //data/message.txt
📸 Screenshots:
<img width="745" height="449" alt="Screenshot 2026-04-01 085856" src="https://github.com/user-attachments/assets/69b9d220-c3f0-4c72-89fa-43766dfbe7f9" />

Observation
File contains data from both runs
💡 Conclusion
Data persisted even after Pod deletion

⚙️ Task 5: StorageClass
kubectl get storageclass
kubectl describe storageclass
📸 Screenshot:
<img width="745" height="449" alt="Screenshot 2026-04-01 090427" src="https://github.com/user-attachments/assets/6ee59794-1ad2-4640-9476-d538cef94139" />

🔍 Observation
Default StorageClass: standard (varies per cluster)

⚡ Task 6: Dynamic Provisioning

Created PVC:

storageClassName: standard
kubectl apply -f dynamic-pvc.yaml
kubectl get pv
📸 Screenshot:
<img width="745" height="449" alt="Screenshot 2026-04-01 090534" src="https://github.com/user-attachments/assets/2086491e-45b8-4825-a9f3-30c3e58de017" />

🔍 Observation
New PV created automatically
💡 Conclusion
StorageClass dynamically provisions PVs

🧹 Task 7: Cleanup
kubectl delete pod --all
kubectl delete pvc --all
kubectl delete pv my-pv
🔍 Observation
Dynamic PV → auto deleted
Manual PV → retained (Retain policy

🧠 Key Concepts
🔹 PV vs PVC
Component	Description
PV	Actual storage
PVC	Request for storage
🔹 Static vs Dynamic Provisioning

Type	Behavior
Static	Manual PV creation
Dynamic	Auto PV creation via StorageClass
🔹 Access Modes
RWO → ReadWriteOnce
ROX → ReadOnlyMany
RWX → ReadWriteMany
🔹 Reclaim Policies
Policy	Behavior
Retain	Data preserved
Delete	Data removed automatically

🎯 Learning Outcome
Understood ephemeral vs persistent storage
Implemented PV & PVC binding
Solved StorageClass mismatch issue
Verified real data persistence
Learned dynamic provisioning

🚀 Why This Matters
Persistent storage is critical for:
Databases (MySQL, PostgreSQL)
Logging systems
Stateful applications
Without PV/PVC, production systems would lose data on every restart.






