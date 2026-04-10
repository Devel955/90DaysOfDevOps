# Day 67 — TerraWeek Capstone: Multi-Environment Infrastructure with Workspaces and Modules

## Overview

This capstone project brings together every concept from the 7-day TerraWeek challenge into a single production-grade Terraform project. One codebase manages three isolated environments — `dev`, `staging`, and `prod` — using **custom modules** and **Terraform workspaces**.

---

## Project Structure (Directory Tree)

```
terraweek-capstone/
├── main.tf                    # Root module — calls child modules
├── variables.tf               # Root input variables
├── outputs.tf                 # Root outputs
├── providers.tf               # AWS provider and S3 backend config
├── locals.tf                  # Computed locals using terraform.workspace
├── dev.tfvars                 # Dev environment variable values
├── staging.tfvars             # Staging environment variable values
├── prod.tfvars                # Prod environment variable values
├── .gitignore                 # Excludes state, .terraform, tfvars secrets
└── modules/
    ├── vpc/
    │   ├── main.tf            # VPC, subnet, IGW, route table
    │   ├── variables.tf       # Module inputs
    │   └── outputs.tf         # vpc_id, subnet_id
    ├── security-group/
    │   ├── main.tf            # SG with dynamic ingress + egress
    │   ├── variables.tf       # Module inputs
    │   └── outputs.tf         # sg_id
    └── ec2-instance/
        ├── main.tf            # EC2 instance with tags
        ├── variables.tf       # Module inputs
        └── outputs.tf         # instance_id, public_ip
```

**Why this structure is best practice:**
- Separation of concerns: each file has a single responsibility
- Modules encapsulate reusable, testable infrastructure components
- Root module acts as an orchestrator, not a resource manager
- Environment-specific values live in `.tfvars`, not in code
- Secrets and state files are gitignored, never committed

---

## .gitignore

```gitignore
.terraform/
*.tfstate
*.tfstate.backup
*.tfvars
.terraform.lock.hcl
```

---

## Task 1: Understanding Workspaces

### What does `terraform.workspace` return inside a config?

`terraform.workspace` is a built-in string variable that returns the **name of the currently active workspace** (e.g., `"dev"`, `"staging"`, `"prod"`, or `"default"`). It can be used directly in resource names, tags, and conditional expressions.

```hcl
locals {
  environment = terraform.workspace
  name_prefix = "${var.project_name}-${terraform.workspace}"
}
```

### Where does each workspace store its state file?

Each workspace stores its state in a separate path:

| Workspace | Local State Path |
|-----------|-----------------|
| `default` | `terraform.tfstate` |
| `dev`     | `terraform.tfstate.d/dev/terraform.tfstate` |
| `staging` | `terraform.tfstate.d/staging/terraform.tfstate` |
| `prod`    | `terraform.tfstate.d/prod/terraform.tfstate` |

With a remote S3 backend, the key becomes:
```
env:/<workspace>/terraform.tfstate
```

### Workspaces vs. Separate Directories

| Aspect | Workspaces | Separate Directories |
|--------|-----------|----------------------|
| Code duplication | None — single codebase | Each env has its own copy |
| State isolation | Automatic per workspace | Manual path management |
| Risk of drift | Low — same code | High — configs diverge over time |
| Visibility | `terraform workspace list` | File system navigation |
| Complexity | Slightly higher initially | Simpler but harder to maintain at scale |

**Workspaces are preferred** when environments are structurally identical but differ only in sizing, CIDRs, or feature flags.

---

## Custom Module Configurations

### Module 1: modules/vpc/

**variables.tf**
```hcl
variable "cidr" {
  type        = string
  description = "VPC CIDR block"
}

variable "public_subnet_cidr" {
  type        = string
  description = "Public subnet CIDR block"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
}

variable "project_name" {
  type        = string
  description = "Project name for resource naming and tagging"
}
```

**main.tf**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-vpc"
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidr
  map_public_ip_on_launch = true

  tags = {
    Name        = "${var.project_name}-${var.environment}-public-subnet"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name        = "${var.project_name}-${var.environment}-igw"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name        = "${var.project_name}-${var.environment}-public-rt"
    Environment = var.environment
    Project     = var.project_name
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}
```

**outputs.tf**
```hcl
output "vpc_id" {
  value       = aws_vpc.main.id
  description = "ID of the created VPC"
}

output "subnet_id" {
  value       = aws_subnet.public.id
  description = "ID of the public subnet"
}
```

---

### Module 2: modules/security-group/

**variables.tf**
```hcl
variable "vpc_id" {
  type        = string
  description = "VPC ID to attach the security group to"
}

variable "ingress_ports" {
  type        = list(number)
  description = "List of ingress ports to allow"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
}

variable "project_name" {
  type        = string
  description = "Project name for resource naming and tagging"
}
```

**main.tf**
```hcl
resource "aws_security_group" "main" {
  name        = "${var.project_name}-${var.environment}-sg"
  description = "Security group for ${var.environment} environment"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name        = "${var.project_name}-${var.environment}-sg"
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
}
```

**outputs.tf**
```hcl
output "sg_id" {
  value       = aws_security_group.main.id
  description = "ID of the security group"
}
```

---

### Module 3: modules/ec2-instance/

**variables.tf**
```hcl
variable "ami_id" {
  type        = string
  description = "AMI ID for the EC2 instance"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
}

variable "subnet_id" {
  type        = string
  description = "Subnet ID to launch the instance in"
}

variable "security_group_ids" {
  type        = list(string)
  description = "List of security group IDs to attach"
}

variable "environment" {
  type        = string
  description = "Deployment environment"
}

variable "project_name" {
  type        = string
  description = "Project name for resource naming and tagging"
}
```

**main.tf**
```hcl
resource "aws_instance" "main" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = {
    Name        = "${var.project_name}-${var.environment}-server"
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
}
```

**outputs.tf**
```hcl
output "instance_id" {
  value       = aws_instance.main.id
  description = "EC2 instance ID"
}

output "public_ip" {
  value       = aws_instance.main.public_ip
  description = "Public IP address of the EC2 instance"
}
```

---

## Root Module Configuration

### providers.tf

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "terraweek-tfstate-bucket"
    key            = "terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraweek-locks"
    encrypt        = true
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### locals.tf

```hcl
locals {
  environment = terraform.workspace
  name_prefix = "${var.project_name}-${local.environment}"

  common_tags = {
    Project     = var.project_name
    Environment = local.environment
    ManagedBy   = "Terraform"
    Workspace   = terraform.workspace
  }
}
```

### variables.tf

```hcl
variable "project_name" {
  type        = string
  default     = "terraweek"
  description = "Project name used as prefix for all resources"
}

variable "vpc_cidr" {
  type        = string
  description = "VPC CIDR block"
}

variable "subnet_cidr" {
  type        = string
  description = "Public subnet CIDR block"
}

variable "instance_type" {
  type        = string
  description = "EC2 instance type"
}

variable "ingress_ports" {
  type        = list(number)
  default     = [22, 80]
  description = "Ports to open in the security group"
}

variable "ami_id" {
  type        = string
  default     = "ami-0c02fb55956c7d316"   # Amazon Linux 2 us-east-1
  description = "AMI ID for EC2 instance"
}
```

### Root main.tf (Workspace-Aware Module Calls)

```hcl
module "vpc" {
  source = "./modules/vpc"

  cidr               = var.vpc_cidr
  public_subnet_cidr = var.subnet_cidr
  environment        = local.environment
  project_name       = var.project_name
}

module "security_group" {
  source = "./modules/security-group"

  vpc_id        = module.vpc.vpc_id
  ingress_ports = var.ingress_ports
  environment   = local.environment
  project_name  = var.project_name
}

module "ec2_instance" {
  source = "./modules/ec2-instance"

  ami_id             = var.ami_id
  instance_type      = var.instance_type
  subnet_id          = module.vpc.subnet_id
  security_group_ids = [module.security_group.sg_id]
  environment        = local.environment
  project_name       = var.project_name
}
```

### outputs.tf

```hcl
output "vpc_id" {
  value       = module.vpc.vpc_id
  description = "VPC ID for current environment"
}

output "subnet_id" {
  value       = module.vpc.subnet_id
  description = "Subnet ID for current environment"
}

output "security_group_id" {
  value       = module.security_group.sg_id
  description = "Security Group ID for current environment"
}

output "instance_id" {
  value       = module.ec2_instance.instance_id
  description = "EC2 Instance ID for current environment"
}

output "instance_public_ip" {
  value       = module.ec2_instance.public_ip
  description = "Public IP of EC2 instance for current environment"
}

output "environment" {
  value       = local.environment
  description = "Active workspace / environment name"
}
```

---

## Environment tfvars Files — Differences Highlighted

### dev.tfvars

```hcl
vpc_cidr      = "10.0.0.0/16"       # ← Dev CIDR range
subnet_cidr   = "10.0.1.0/24"
instance_type = "t2.micro"           # ← Smallest / cheapest
ingress_ports = [22, 80]             # ← SSH allowed in dev
```

### staging.tfvars

```hcl
vpc_cidr      = "10.1.0.0/16"       # ← Staging CIDR (no overlap with dev)
subnet_cidr   = "10.1.1.0/24"
instance_type = "t2.small"           # ← One step up from dev
ingress_ports = [22, 80, 443]        # ← SSH + HTTP + HTTPS
```

### prod.tfvars

```hcl
vpc_cidr      = "10.2.0.0/16"       # ← Prod CIDR (no overlap with dev/staging)
subnet_cidr   = "10.2.1.0/24"
instance_type = "t3.small"           # ← Larger, newer gen for production load
ingress_ports = [80, 443]            # ← SSH REMOVED — prod is not directly accessible
```

**Key differences between environments:**

| Config | Dev | Staging | Prod |
|--------|-----|---------|------|
| VPC CIDR | 10.0.0.0/16 | 10.1.0.0/16 | 10.2.0.0/16 |
| Instance Type | t2.micro | t2.small | t3.small |
| SSH (port 22) | ✅ Allowed | ✅ Allowed | ❌ Blocked |
| HTTPS (port 443) | ❌ | ✅ | ✅ |

---

## Deployment Commands

### Initialize

```bash
mkdir terraweek-capstone && cd terraweek-capstone
terraform init
```

### Create Workspaces

```bash
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod
terraform workspace list
# Output:
#   default
# * dev
#   staging
#   prod
```

### Deploy Dev

```bash
terraform workspace select dev
terraform plan -var-file="dev.tfvars"
terraform apply -var-file="dev.tfvars"
```

### Deploy Staging

```bash
terraform workspace select staging
terraform plan -var-file="staging.tfvars"
terraform apply -var-file="staging.tfvars"
```

### Deploy Prod

```bash
terraform workspace select prod
terraform plan -var-file="prod.tfvars"
terraform apply -var-file="prod.tfvars"
```

### Verify All Environments

```bash
terraform workspace select dev && terraform output
terraform workspace select staging && terraform output
terraform workspace select prod && terraform output
```

---

## Verification Checklist

After deployment, verified in the AWS Console:

- [x] Three separate VPCs with CIDRs `10.0.0.0/16`, `10.1.0.0/16`, `10.2.0.0/16`
- [x] Three EC2 instances with types `t2.micro`, `t2.small`, `t3.small`
- [x] Name tags: `terraweek-dev-server`, `terraweek-staging-server`, `terraweek-prod-server`
- [x] Dev and staging have port 22 open; prod does not
- [x] Each environment's VPCs are completely isolated (no overlapping CIDRs)
- [x] All resources tagged with `Project`, `Environment`, `ManagedBy`, `Workspace`

---

## Terraform Best Practices Guide

### 1. File Structure

Separate every concern into its own file. Never put everything in a single `main.tf`:

- `providers.tf` — provider versions and backend config
- `variables.tf` — all input variable declarations
- `outputs.tf` — all output declarations
- `locals.tf` — computed values, common tags, name prefixes
- `main.tf` — resource and module calls only

### 2. State Management

- **Always use a remote backend** (S3 + DynamoDB for AWS) — local state is lost when machines change
- **Enable state locking** via DynamoDB to prevent concurrent `apply` collisions
- **Enable versioning on the S3 bucket** so state can be rolled back after mistakes
- **Enable encryption** (`encrypt = true`) for state at rest — state contains sensitive values
- **Never commit `.tfstate` files** — they contain secrets and belong in the backend

### 3. Variables

- **Never hardcode values** — everything environment-specific belongs in `.tfvars`
- Use `validation` blocks to catch bad input before a plan runs
- Use `sensitive = true` for secrets so they don't appear in CLI output
- Provide a `description` for every variable
- Set sensible `default` values only for truly optional variables

### 4. Modules

- **One concern per module** — VPC module should not create EC2 instances
- Always define explicit `variables.tf` and `outputs.tf` — never rely on implicit references
- **Pin registry module versions**: `version = "~> 5.0"` not `version = "latest"`
- Validate modules independently with `terraform validate` before wiring them in
- Use `terraform-docs` to auto-generate module documentation

### 5. Workspaces

- Use workspaces when environments are structurally identical but differ in sizing/config
- Always reference `terraform.workspace` via a local for consistency
- Use different `.tfvars` files per workspace to drive environment-specific values
- Never use the `default` workspace for real infrastructure — always create named workspaces

### 6. Security

- `.gitignore`: always exclude `.terraform/`, `*.tfstate`, `*.tfstate.backup`, `*.tfvars`
- Encrypt state at rest (S3 SSE or KMS)
- Use IAM roles with least privilege for the backend and provider
- Never store credentials in `.tf` files — use environment variables or IAM instance profiles
- Use `sensitive = true` on outputs containing passwords or keys

### 7. Commands

Always follow this workflow:

```bash
terraform fmt        # Format code before committing
terraform validate   # Catch syntax and config errors
terraform plan       # Always review before applying
terraform apply      # Apply only after reviewing the plan
```

- Use `terraform plan -out=tfplan` and `terraform apply tfplan` for CI/CD safety
- Use `terraform show` to inspect current state
- Use `terraform state list` to audit managed resources

### 8. Tagging

Tag **every resource** at creation. Enforce via policy if possible:

```hcl
tags = {
  Project     = var.project_name
  Environment = local.environment
  ManagedBy   = "Terraform"
  Workspace   = terraform.workspace
}
```

Tags are your only defense against "what is this resource and can I delete it?"

### 9. Naming Convention

Use a consistent prefix pattern across all resources:

```
<project>-<environment>-<resource>
```

Examples:
- `terraweek-prod-vpc`
- `terraweek-dev-sg`
- `terraweek-staging-server`

This makes filtering in the console, billing reports, and CloudWatch trivial.

### 10. Cleanup

- Always `terraform destroy` non-production environments when not in use — idle EC2 and NAT gateways cost money
- Destroy in reverse order: prod → staging → dev
- Verify in the console after destroy — orphaned resources accrue costs silently
- Delete unused workspaces to keep the state backend clean

---

## Destroy Commands

```bash
# Destroy in reverse order
terraform workspace select prod
terraform destroy -var-file="prod.tfvars"

terraform workspace select staging
terraform destroy -var-file="staging.tfvars"

terraform workspace select dev
terraform destroy -var-file="dev.tfvars"

# Delete workspaces (must be on default first)
terraform workspace select default
terraform workspace delete prod
terraform workspace delete staging
terraform workspace delete dev
```

---

## TerraWeek — Day-by-Day Concepts

| Day | Concepts |
|-----|----------|
| 61 | IaC fundamentals, HCL syntax, `init` / `plan` / `apply` / `destroy`, state basics |
| 62 | Providers, resources, resource dependencies, `lifecycle` meta-arguments |
| 63 | Variables, outputs, data sources, locals, built-in functions |
| 64 | Remote backends (S3 + DynamoDB), state locking, `terraform import`, drift detection |
| 65 | Custom modules (inputs/outputs/structure), registry modules, module versioning |
| 66 | EKS cluster with modules, real-world multi-resource provisioning |
| 67 | Terraform workspaces, multi-environment architecture, capstone project |

---

*TerraWeek Challenge — Day 67 of 90 Days of DevOps | #90DaysOfDevOps #TerraWeek #DevOpsKaJosh*
