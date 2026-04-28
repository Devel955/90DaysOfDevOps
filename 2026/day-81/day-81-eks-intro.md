# Day 81 – Introduction to Amazon EKS with Terraform

## Table of Contents
1. [EKS Architecture Overview](#eks-architecture-overview)
2. [Terraform Configuration Breakdown](#terraform-configuration-breakdown)
3. [Cluster Provisioning Steps](#cluster-provisioning-steps)
4. [Connecting to the Cluster](#connecting-to-the-cluster)
5. [Deploying the AI-BankApp](#deploying-the-ai-bankapp)
6. [EKS Cost Breakdown](#eks-cost-breakdown)
7. [Key Learnings](#key-learnings)

---

## EKS Architecture Overview

### What is "Managed Kubernetes"?

Amazon EKS is AWS's managed Kubernetes service. "Managed" means:

- **AWS manages the control plane** – The API server, etcd, scheduler, and controller manager are handled entirely by AWS. You never SSH into the control plane nodes.
- **You manage the data plane** – Worker nodes (EC2 instances) where your pods actually run are your responsibility.
- **AWS handles HA and upgrades** – The control plane is automatically spread across multiple Availability Zones, with patching and upgrades managed by AWS.

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                          AWS VPC (10.0.0.0/16)                  │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Public AZ-1  │  │  Public AZ-2  │  │  Public AZ-3  │         │
│  │ 10.0.1.0/24  │  │ 10.0.2.0/24  │  │ 10.0.3.0/24  │         │
│  │ (Load Balancers / NAT) │   ...  │         ...   │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │ NAT GW          │ NAT GW           │ NAT GW            │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐          │
│  │ Private AZ-1  │  │ Private AZ-2  │  │ Private AZ-3  │        │
│  │ 10.0.4.0/24  │  │ 10.0.5.0/24  │  │ 10.0.6.0/24  │        │
│  │  Worker Node  │  │  Worker Node  │  │  Worker Node  │        │
│  │ (t3.medium)  │  │ (t3.medium)  │  │ (t3.medium)  │         │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                   │
│  ┌──────▼──────────────────▼──────────────────▼───────┐         │
│  │              Intra Subnets (10.0.7-9.0/24)          │        │
│  │         EKS Control Plane ENIs (AWS-managed)         │        │
│  └──────────────────────┬──────────────────────────────┘         │
│                          │                                        │
│  ┌───────────────────────▼──────────────────────────────┐        │
│  │              EKS Control Plane (AWS-Managed)           │       │
│  │   API Server │ etcd │ Scheduler │ Controller Manager   │      │
│  └───────────────────────────────────────────────────────┘       │
│                                                                   │
│  ┌───────────────────────────────────────────────────────┐       │
│  │                    EKS Add-ons                         │       │
│  │  coredns | kube-proxy | vpc-cni | eks-pod-identity-    │      │
│  │  agent | aws-ebs-csi-driver | metrics-server           │      │
│  └───────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

### EKS Components Explained

| Component | Description |
|---|---|
| **EKS Control Plane** | Managed by AWS, runs in AWS-owned VPC, accessed via public/private API endpoint |
| **Managed Node Groups** | AWS automates EC2 provisioning, scaling, and AMI updates for worker nodes |
| **Self-Managed Nodes** | User manages EC2 instances — full control, more responsibility |
| **Fargate Profiles** | Serverless pods — no nodes to manage at all, pay per pod |
| **VPC & Networking** | EKS runs inside your VPC; pods get real VPC IPs via the vpc-cni plugin |
| **IAM Integration (IRSA)** | Pods can assume IAM roles via OIDC — fine-grained, no secrets on nodes |

### EKS Add-ons Used by AI-BankApp

| Add-on | Purpose |
|---|---|
| `coredns` | DNS resolution for service discovery inside the cluster |
| `kube-proxy` | Maintains network rules on nodes for Service routing |
| `vpc-cni` | Assigns real VPC IPs to pods — makes pods directly routable in the VPC |
| `eks-pod-identity-agent` | Enables IRSA — pods assume IAM roles without storing credentials |
| `aws-ebs-csi-driver` | Allows pods to dynamically provision and use EBS volumes (MySQL, Ollama) |
| `metrics-server` | Exposes CPU/memory metrics; needed for `kubectl top` and HPA auto-scaling |

---

## Terraform Configuration Breakdown

### Directory Structure

```
terraform/
├── argocd.tf        # ArgoCD Helm release
├── eks.tf           # EKS cluster, node group, add-ons, IRSA
├── outputs.tf       # Helper commands as outputs
├── provider.tf      # AWS + Helm providers, local values
├── terraform.tfvars # Variable defaults
└── vpc.tf           # VPC with public/private/intra subnets
```

### `variables.tf` & `terraform.tfvars`

Defines all tunable parameters for the deployment:

```hcl
aws_region         = "us-west-2"
cluster_name       = "bankapp-eks"
cluster_version    = "1.35"
node_instance_type = "t3.medium"
node_desired_count = 3
node_max_count     = 5
```

**Why these choices?**
- `us-west-2` is cost-effective and has 3 AZs for HA
- `t3.medium` (2 vCPU, 4 GB RAM) can host ~17 pods (limited by ENI IP slots)
- 3 nodes across 3 AZs ensures no single-AZ failure takes the app down

---

### `vpc.tf` — Networking Foundation

Uses the `terraform-aws-modules/vpc/aws` community module. Creates a production VPC with a three-tier subnet layout:

| Subnet Type | CIDR Range | Purpose | Tag |
|---|---|---|---|
| Public | 10.0.1-3.0/24 | Load balancers, NAT Gateway | `kubernetes.io/role/elb = 1` |
| Private | 10.0.4-6.0/24 | Worker nodes (internet via NAT) | `kubernetes.io/role/internal-elb = 1` |
| Intra | 10.0.7-9.0/24 | EKS control plane ENIs only | No internet access |

**Key design decisions:**
- Worker nodes sit in **private subnets** — they cannot be reached directly from the internet
- NAT Gateway enables outbound internet from private subnets (pulling Docker images, calling APIs)
- Intra subnets isolate the control plane communication path entirely
- Subnet tags are required by the AWS Load Balancer Controller to know which subnets to use

---

### `eks.tf` — The Cluster Itself

Uses `terraform-aws-modules/eks/aws` (version ~> 21.0). Key configuration:

```hcl
# Cluster
cluster_name    = var.cluster_name
cluster_version = var.cluster_version

# Both public and private endpoint access
cluster_endpoint_public_access  = true
cluster_endpoint_private_access = true

# Amazon Linux 2023 AMI
ami_type = "AL2023_x86_64_STANDARD"

# Managed node group
node_groups = {
  main = {
    instance_types = [var.node_instance_type]
    min_size       = 3
    max_size       = var.node_max_count
    desired_size   = var.node_desired_count
  }
}

# All 6 add-ons declared as cluster_addons
cluster_addons = {
  coredns              = {}
  kube-proxy           = {}
  vpc-cni              = {}
  eks-pod-identity-agent = {}
  aws-ebs-csi-driver   = {}
  metrics-server       = {}
}
```

**IRSA for EBS CSI Driver:**
The EBS CSI driver needs IAM permissions to create/attach EBS volumes. IRSA (IAM Roles for Service Accounts) provides this without storing credentials on nodes — the pod's service account is annotated with an IAM role ARN, and AWS's OIDC provider validates the token.

---

### `argocd.tf` — GitOps Controller via Helm

Installs ArgoCD using the official `argo-cd` Helm chart:

```hcl
resource "helm_release" "argocd" {
  name       = "argocd"
  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"
  namespace  = "argocd"

  set {
    name  = "server.service.type"
    value = "LoadBalancer"
  }

  depends_on = [module.eks]
}
```

**Why Terraform installs ArgoCD?**
ArgoCD is the GitOps engine for Days 84–86. Installing it via Terraform ensures it's ready the moment the cluster is live. The `depends_on` ensures the EKS module is fully ready before Helm deploys anything.

---

### `outputs.tf` — Helper Commands

After `terraform apply`, outputs print ready-to-run commands:

```hcl
output "configure_kubectl" {
  value = "aws eks update-kubeconfig --name ${var.cluster_name} --region ${var.aws_region}"
}

output "argocd_password" {
  value = "kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d"
}
```

---

## Cluster Provisioning Steps

### Prerequisites

```bash
terraform --version   # >= 1.0
aws --version         # AWS CLI v2
kubectl version --client
helm version
```

### Configure AWS Credentials

```bash
aws configure
# AWS Access Key ID: <your key>
# AWS Secret Access Key: <your secret>
# Default region: us-west-2
# Default output: json

# Verify identity
aws sts get-caller-identity
```

### Initialize and Apply

```bash
git clone -b feat/gitops https://github.com/TrainWithShubham/AI-BankApp-DevOps.git
cd AI-BankApp-DevOps/terraform

terraform init
terraform plan   # Review what will be created
terraform apply  # Takes 15–20 minutes
```

**What gets created:**
- 1 VPC with 9 subnets, 1 NAT Gateway, 1 Internet Gateway
- 1 EKS cluster (managed control plane)
- 1 Managed Node Group (3× t3.medium across 3 AZs)
- 6 EKS add-ons
- IAM roles for cluster, nodes, and EBS CSI (IRSA)
- ArgoCD via Helm

```bash
# Check outputs after apply
terraform output
```

---

## Connecting to the Cluster

```bash
# Update kubeconfig (use command from terraform output)
aws eks update-kubeconfig --name bankapp-eks --region us-west-2

# Verify context
kubectl config current-context

# Cluster info
kubectl cluster-info

# Expected: 3 nodes, status Ready, spread across 3 AZs
kubectl get nodes -o wide
```

**Expected output of `kubectl get nodes -o wide`:**
```
NAME                          STATUS   ROLES    AGE   VERSION   INTERNAL-IP   OS-IMAGE
ip-10-0-4-xxx.ec2.internal    Ready    <none>   5m    v1.35.x   10.0.4.xxx    Amazon Linux 2023
ip-10-0-5-xxx.ec2.internal    Ready    <none>   5m    v1.35.x   10.0.5.xxx    Amazon Linux 2023
ip-10-0-6-xxx.ec2.internal    Ready    <none>   5m    v1.35.x   10.0.6.xxx    Amazon Linux 2023
```

### Explore Add-ons

```bash
# All system pods (add-ons run here)
kubectl get pods -n kube-system

# DaemonSets (kube-proxy, vpc-cni run on every node)
kubectl get daemonsets -n kube-system

# EBS CSI driver pods
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver

# Node resource usage (requires metrics-server)
kubectl top nodes
```

### Access ArgoCD

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Get the LoadBalancer URL
kubectl get svc -n argocd argocd-server \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
```

Open the URL in your browser → log in with `admin` and the password above.

> **Screenshot placeholder:** `kubectl get pods -n kube-system` showing all 6 add-ons in Running state

> **Screenshot placeholder:** ArgoCD login page accessible via LoadBalancer URL

---

## Deploying the AI-BankApp

Manual deployment validates the cluster before setting up GitOps on Days 84–86.

### Apply Manifests in Order

```bash
cd ../   # Back to repo root

kubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/pv.yml
kubectl apply -f k8s/pvc.yml
kubectl apply -f k8s/configmap.yml
kubectl apply -f k8s/secrets.yml
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/service.yml
kubectl apply -f k8s/ollama-deployment.yml
kubectl apply -f k8s/bankapp-deployment.yml
kubectl apply -f k8s/hpa.yml
```

### Watch Startup Sequence

```bash
kubectl get pods -n bankapp -w
```

**Expected startup order:**
1. **MySQL** starts and becomes healthy (~15–30 seconds)
2. **Ollama** starts and pulls TinyLlama model (~2–5 minutes)
3. **BankApp** init containers wait for both dependencies, then app starts (~30–60 seconds)

### Verify EBS Volumes

```bash
kubectl get pvc -n bankapp
kubectl get pv
```

**Expected:** Two EBS-backed PVCs — one 5Gi for MySQL, one 10Gi for Ollama — both in `Bound` state.

> **Screenshot placeholder:** `kubectl get pvc -n bankapp` showing Bound status

### Access the App

```bash
kubectl port-forward svc/bankapp-service -n bankapp 8080:8080
# Open http://localhost:8080
```

> **Screenshot placeholder:** AI-BankApp login page at http://localhost:8080

### Check HPA

```bash
kubectl get hpa -n bankapp
```

---

## EKS Cost Breakdown

### AI-BankApp Cluster Running Costs

| Component | Specification | Hourly Cost | Monthly Cost |
|---|---|---|---|
| EKS Control Plane | Managed, multi-AZ | $0.10/hr | ~$73 |
| Worker Nodes | 3× t3.medium | $0.042/hr each | ~$91 |
| NAT Gateway | 1 gateway + data transfer | $0.045/hr + transfer | ~$33 |
| EBS Volumes | ~15Gi total (MySQL + Ollama) | — | ~$1.50 |
| LoadBalancer | ArgoCD service | $0.025/hr | ~$18 |
| **Total** | | | **~$217/month (~$7/day)** |

### Why is NAT Gateway Surprisingly Expensive?

The NAT Gateway has two cost components that add up quickly:

1. **Hourly charge ($0.045/hr ≈ $33/month)** — you pay just for the NAT Gateway existing, even if no traffic flows through it.

2. **Data processing charge ($0.045 per GB)** — every byte leaving private subnets through the NAT Gateway is charged. This includes:
   - Docker image pulls (a single large image pull can be hundreds of MBs)
   - Ollama model downloads (TinyLlama is ~1–2 GB)
   - API calls from pods to AWS services or the internet
   - Kubernetes node joining and add-on downloads at startup

In contrast, the EKS control plane charge is flat and predictable. The NAT Gateway can surprise you when workloads pull large images or models repeatedly.

### Clean-Up Commands

**Delete workloads only (keep cluster for Days 82–83):**
```bash
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

**Destroy everything (end of Day 83 or before a long break):**
```bash
cd terraform
terraform destroy
```

> ⚠️ **Important:** An idle cluster still costs ~$7/day just for the control plane, NAT Gateway, and nodes. Always destroy when not actively using it.

---

## Key Learnings

1. **EKS separates control plane from data plane** — AWS handles the hard part (HA control plane, etcd backups, API server scaling). You focus on your workloads.

2. **The VPC CNI plugin gives pods real VPC IPs** — unlike vanilla Kubernetes overlay networks, pods in EKS are directly routable within the VPC. This is powerful but means IP address exhaustion is a real concern (t3.medium supports ~17 pods due to ENI limits).

3. **IRSA is the right way to give pods AWS permissions** — no hard-coded credentials, no node-level IAM roles that grant all pods the same permissions. Each pod's service account maps to a specific IAM role.

4. **Terraform modules abstract the complexity** — the `terraform-aws-modules/eks/aws` module provisions hundreds of resources (IAM roles, security groups, node groups, add-ons) from a readable configuration file.

5. **ArgoCD as infrastructure** — installing ArgoCD via Terraform ensures it's always present and versioned alongside the cluster, not applied manually and forgotten.

6. **Cost discipline is critical** — a "dev cluster" left running overnight costs $7. A month of idle time costs $217. Terraform destroy is your friend.

---

*Day 81 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
