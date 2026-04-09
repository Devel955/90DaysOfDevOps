# Day 66 — Provision an EKS Cluster with Terraform Modules

## Overview

Today I provisioned a production-grade AWS EKS cluster entirely through Terraform — no manual clicking, no guesswork. Using official Terraform registry modules, I created a full VPC, EKS cluster, managed node group, deployed Nginx on it, and destroyed everything cleanly.

This is exactly what infrastructure teams do every day in real-world production environments.

---

## Project File Structure

```
terraform-eks/
  providers.tf        # Provider and backend config
  vpc.tf              # VPC module call
  eks.tf              # EKS module call
  variables.tf        # All input variables
  outputs.tf          # Cluster outputs
  terraform.tfvars    # Variable values
  k8s/
    nginx-deployment.yaml  # Nginx deployment + LoadBalancer service
```

---

## Key Configuration Files

### providers.tf

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

### variables.tf

```hcl
variable "region" {
  type    = string
  default = "ap-south-1"
}

variable "cluster_name" {
  type    = string
  default = "terraweek-eks"
}

variable "cluster_version" {
  type    = string
  default = "1.31"
}

variable "node_instance_type" {
  type    = string
  default = "t3.small"
}

variable "node_desired_count" {
  type    = number
  default = 1
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}
```

### vpc.tf

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "terraweek-vpc"
  cidr = var.vpc_cidr

  azs             = data.aws_availability_zones.available.names
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway   = true
  single_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = 1
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = 1
  }

  tags = {
    Environment = "dev"
    Project     = "TerraWeek"
    ManagedBy   = "Terraform"
  }
}
```

### eks.tf

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = var.cluster_name
  cluster_version = var.cluster_version

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    terraweek_nodes = {
      ami_type       = "AL2_x86_64"
      instance_types = ["t3.small"]

      min_size     = 1
      max_size     = 2
      desired_size = 1
    }
  }

  tags = {
    Environment = "dev"
    Project     = "TerraWeek"
    ManagedBy   = "Terraform"
  }
}
```

### outputs.tf

```hcl
output "cluster_name" {
  value = module.eks.cluster_name
}

output "cluster_endpoint" {
  value = module.eks.cluster_endpoint
}

output "cluster_region" {
  value = var.region
}

output "vpc_id" {
  value = module.vpc.vpc_id
}

output "configure_kubectl" {
  value = "aws eks update-kubeconfig --name ${module.eks.cluster_name} --region ${var.region}"
}
```

### k8s/nginx-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-terraweek
  labels:
    app: nginx
spec:
  replicas: 1
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
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
```

---

## Task 1: Project Setup ✅

Created the project directory with proper file structure. Pinned AWS provider to `~> 5.0` and Kubernetes provider to `~> 2.0`. Defined all input variables with sensible defaults.

---

## Task 2: VPC Creation ✅

Used the `terraform-aws-modules/vpc/aws` module to create:
- VPC with CIDR `10.0.0.0/16`
- 2 public subnets and 2 private subnets across 2 AZs
- Single NAT Gateway (cost optimization)
- DNS hostnames enabled

### Why does EKS need both public and private subnets?

**Private subnets** are where the EKS worker nodes run. Keeping nodes in private subnets means they are not directly exposed to the internet, which is a security best practice. Nodes communicate outbound via the NAT Gateway.

**Public subnets** are used by AWS Load Balancers (ELB/ALB/NLB). When you create a Kubernetes `Service` of type `LoadBalancer`, AWS provisions an ELB in the public subnet so it can receive traffic from the internet and forward it to the private nodes.

### What do the subnet tags do?

The tags tell the AWS Load Balancer Controller which subnets to use:

- `kubernetes.io/role/elb = 1` on public subnets → external Load Balancers are created here
- `kubernetes.io/role/internal-elb = 1` on private subnets → internal Load Balancers are created here

Without these tags, AWS cannot automatically discover the correct subnets for Load Balancer provisioning.

---

## Task 3: EKS Cluster Creation ✅

Used the `terraform-aws-modules/eks/aws` module version `~> 20.0`.

```bash
terraform init      # Downloaded EKS module and all dependencies
terraform plan      # Reviewed 30+ resources to be created
terraform apply     # Applied — took ~15 minutes
```

**Resources created by Terraform:**
- EKS Control Plane (cluster)
- IAM roles for cluster and node group (auto-created by module)
- Security groups for cluster and nodes
- KMS key for envelope encryption
- CloudWatch log group
- OIDC provider
- Managed Node Group with `t3.small` instances
- Launch template for nodes
- VPC, subnets, NAT gateway, route tables, internet gateway

**Total: 30+ resources created with a single `terraform apply`**

**Terraform Apply Output:**
```
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:
cluster_endpoint = "https://3AD4866BA81FB23CDF62F6BF58365A85.gr7.ap-south-1.eks.amazonaws.com"
cluster_name     = "terraweek-eks"
cluster_region   = "ap-south-1"
configure_kubectl = "aws eks update-kubeconfig --name terraweek-eks --region ap-south-1"
vpc_id           = "vpc-0f23f3fe26a2a0136"
```

---

## Task 4: Connect kubectl ✅

```bash
aws eks update-kubeconfig --name terraweek-eks --region ap-south-1
# Updated context arn:aws:eks:ap-south-1:613884141188:cluster/terraweek-eks
```

Since the cluster was created by `terraform-user`, kubectl access had to be explicitly granted using EKS Access Entries (new EKS auth method):

```bash
aws eks create-access-entry \
  --cluster-name terraweek-eks \
  --principal-arn arn:aws:iam::613884141188:user/terraform-user \
  --region ap-south-1

aws eks associate-access-policy \
  --cluster-name terraweek-eks \
  --principal-arn arn:aws:iam::613884141188:user/terraform-user \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster \
  --region ap-south-1
```

**Node verified:**
```
NAME                                       STATUS   ROLES    AGE     VERSION
ip-10-0-4-21.ap-south-1.compute.internal   Ready    <none>   8m15s   v1.31.13-eks-ecaa3a6
```

**kube-system pods running:**
```
NAMESPACE     NAME                               READY   STATUS
kube-system   aws-node-qwp9x                     2/2     Running
kube-system   coredns-5f7c9b478-hgs84            1/1     Running
kube-system   coredns-5f7c9b478-v9j6z            1/1     Running
kube-system   kube-proxy-pmvvm                   1/1     Running
```

---

## Task 5: Deploy Nginx Workload ✅

```bash
kubectl apply -f k8s/nginx-deployment.yaml
# deployment.apps/nginx-terraweek created
# service/nginx-service created
```

**LoadBalancer provisioned:**
```bash
kubectl get svc nginx-service
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)
nginx-service   LoadBalancer   172.20.189.82   af92c4cb187b94a4f8329e2b71532264-534866221.ap-south-1.elb.amazonaws.com   80:31155/TCP
```

**Nginx accessible at:**
```
http://af92c4cb187b94a4f8329e2b71532264-534866221.ap-south-1.elb.amazonaws.com
```

**Full cluster verification:**
```
NAME                                       STATUS   ROLES    AGE
ip-10-0-4-21.ap-south-1.compute.internal   Ready    <none>   8m15s

NAME                               READY   STATUS    RESTARTS
nginx-terraweek-54b9c68f67-jdm8k   1/1     Running   0
```

### Lesson Learned — t3.micro vs t3.small

Initially used `t3.micro` (1GB RAM) which caused pods to stay in `Pending` state with error:
```
0/1 nodes are available: 1 Too many pods
```

`t3.micro` supports a maximum of 4 pods, and kube-system pods (aws-node, coredns x2, kube-proxy) already consumed all slots. Upgrading to `t3.small` (2GB RAM, supports more pods) resolved the issue.

**Lesson: For EKS, minimum recommended instance type is `t3.small`. `t3.medium` is ideal for production.**

---

## Task 6: Destroy Everything ✅

```bash
# Step 1: Remove Kubernetes resources first (so ELB gets deleted)
kubectl delete -f k8s/nginx-deployment.yaml

# Step 2: Destroy all Terraform resources
terraform destroy
# Destruction complete after ~15 minutes
```

**AWS Console verified clean:**
- EKS clusters: empty
- EC2 instances: terminated
- VPC: deleted
- NAT Gateway: deleted
- Elastic IPs: released

---

## Reflection: Terraform EKS vs Manual Kind/Minikube (Day 50)

| Aspect | Kind/Minikube (Day 50) | Terraform EKS (Day 66) |
|--------|----------------------|----------------------|
| Setup time | 2-3 minutes | 10-15 minutes |
| Environment | Local only | Real AWS cloud |
| Reproducibility | Manual steps | 100% automated |
| Teardown | `kind delete cluster` | `terraform destroy` |
| Production ready | ❌ No | ✅ Yes |
| Cost | Free | ~$0.10/hr (control plane) + EC2 |
| IAM/Networking | Not needed | Auto-created by module |
| Load Balancer | Not real | Real AWS ELB |
| Scalability | Limited | Auto Scaling Group |

**Key Insight:** Kind/Minikube is great for local development and learning Kubernetes concepts. Terraform + EKS is what you actually use in production. The Terraform module abstracted away 30+ AWS resources — IAM roles, security groups, KMS keys, OIDC providers — into a clean, readable config file.

This is the power of Infrastructure as Code: complex, repeatable, and destroyable with a single command.

---

## Key Learnings

1. **Terraform modules** make complex cloud infrastructure simple and repeatable
2. **EKS needs public + private subnets** — nodes in private, load balancers in public
3. **Subnet tags are mandatory** for AWS Load Balancer Controller to work correctly
4. **`t3.micro` is too small for EKS** — use `t3.small` minimum
5. **Always delete K8s LoadBalancer services** before `terraform destroy` to avoid stuck VPC deletion
6. **EKS Access Entries** is the modern way to manage kubectl access (replaces aws-auth ConfigMap)
7. **NAT Gateway costs money** (~$0.045/hr) — always destroy when done

---

*Day 66 of #90DaysOfDevOps | #TerraWeek | #DevOpsKaJosh | #TrainWithShubham*
