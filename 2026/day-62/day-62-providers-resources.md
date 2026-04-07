# Day 62 — Providers, Resources and Dependencies in Terraform

## Overview

Today we build a complete AWS networking stack using Terraform — covering providers, implicit/explicit dependencies, lifecycle rules, and the dependency graph. Real infrastructure is connected, and Terraform's dependency resolution is what makes declarative IaC powerful.

---

## Task 1: Explore the AWS Provider

### `providers.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"  # Change to your region
}
```

### Version Constraint Meanings

| Constraint | Meaning |
|---|---|
| `~> 5.0` | Allows `>= 5.0` and `< 6.0` (minor/patch updates only) |
| `>= 5.0` | Allows any version 5.0 or higher, including 6.x, 7.x, etc. |
| `= 5.0.0` | Pins to exactly version 5.0.0 — no updates allowed |

`~> 5.0` is the recommended approach for production: you get bug fixes and minor improvements but are protected from breaking major-version changes.

### The Lock File: `.terraform.lock.hcl`

After running `terraform init`, Terraform creates `.terraform.lock.hcl`. This file:

- Records the **exact provider version** installed (e.g., `version = "5.43.0"`)
- Stores **checksums** (hashes) of the provider binary for integrity verification
- Should be **committed to version control** so all team members and CI pipelines use identical provider versions
- Can be updated intentionally with `terraform init -upgrade`

```hcl
# Example .terraform.lock.hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.43.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",
    "zh:def456...",
  ]
}
```

---

## Task 2: Build a VPC from Scratch

### `main.tf` — Core Networking Resources

```hcl
# ─────────────────────────────────────────────
# VPC — the top-level network container
# All other resources live inside this VPC
# ─────────────────────────────────────────────
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"   # 65,536 available IP addresses

  tags = {
    Name = "TerraWeek-VPC"
  }
}

# ─────────────────────────────────────────────
# Subnet — a slice of the VPC's address space
# Implicitly depends on aws_vpc.main (references its id)
# ─────────────────────────────────────────────
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id   # implicit dependency on VPC
  cidr_block              = "10.0.1.0/24"     # 256 IP addresses
  map_public_ip_on_launch = true              # instances get a public IP automatically

  tags = {
    Name = "TerraWeek-Public-Subnet"
  }
}

# ─────────────────────────────────────────────
# Internet Gateway — the door between VPC and internet
# Implicitly depends on aws_vpc.main
# ─────────────────────────────────────────────
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id   # implicit dependency on VPC

  tags = {
    Name = "TerraWeek-IGW"
  }
}

# ─────────────────────────────────────────────
# Route Table — defines how traffic is routed
# 0.0.0.0/0 (all internet traffic) → internet gateway
# Implicitly depends on VPC and internet gateway
# ─────────────────────────────────────────────
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id   # implicit dependency on VPC

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id  # implicit dependency on IGW
  }

  tags = {
    Name = "TerraWeek-RT"
  }
}

# ─────────────────────────────────────────────
# Route Table Association — links route table to subnet
# Implicitly depends on both the route table and the subnet
# ─────────────────────────────────────────────
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id          # implicit dependency on subnet
  route_table_id = aws_route_table.public.id     # implicit dependency on route table
}
```

Running `terraform plan` shows **5 resources to create**, in the correct dependency order.

---

## Task 3: Implicit Dependencies Explained

### How Terraform Resolves Order

When you write `vpc_id = aws_vpc.main.id`, Terraform records that `aws_subnet.public` **reads an attribute from** `aws_vpc.main`. This means the VPC must be fully created before the subnet can be provisioned.

Terraform builds an internal **dependency graph** from all these references before doing anything. It walks the graph and creates resources in topological order — parents before children.

### What Happens Without the VPC?

If you tried to create the subnet before the VPC existed, AWS would return an error like:
`InvalidVpcID.NotFound: The vpc ID 'vpc-xxxxx' does not exist`

Terraform prevents this by understanding the graph and never attempting to create a child resource until all its parents succeed.

### All Implicit Dependencies in `main.tf`

| Resource | Depends On | Via |
|---|---|---|
| `aws_subnet.public` | `aws_vpc.main` | `vpc_id` |
| `aws_internet_gateway.main` | `aws_vpc.main` | `vpc_id` |
| `aws_route_table.public` | `aws_vpc.main` | `vpc_id` |
| `aws_route_table.public` | `aws_internet_gateway.main` | `gateway_id` inside route block |
| `aws_route_table_association.public` | `aws_subnet.public` | `subnet_id` |
| `aws_route_table_association.public` | `aws_route_table.public` | `route_table_id` |

---

## Task 4: Security Group and EC2 Instance

```hcl
# ─────────────────────────────────────────────
# Security Group — virtual firewall for EC2
# Controls inbound and outbound traffic
# ─────────────────────────────────────────────
resource "aws_security_group" "web" {
  name        = "TerraWeek-SG"
  description = "Allow SSH and HTTP inbound, all outbound"
  vpc_id      = aws_vpc.main.id   # implicit dependency on VPC

  # Allow SSH from anywhere (port 22)
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "SSH access"
  }

  # Allow HTTP from anywhere (port 80)
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP access"
  }

  # Allow all outbound traffic
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"   # -1 means all protocols
    cidr_blocks = ["0.0.0.0/0"]
    description = "All outbound traffic"
  }

  tags = {
    Name = "TerraWeek-SG"
  }
}

# ─────────────────────────────────────────────
# EC2 Instance — the virtual server
# Implicitly depends on subnet and security group
# ─────────────────────────────────────────────
resource "aws_instance" "main" {
  # Amazon Linux 2 AMI (us-east-1) — update for your region
  ami                         = "ami-0c02fb55956c7d316"
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.public.id              # implicit dep on subnet
  vpc_security_group_ids      = [aws_security_group.web.id]      # implicit dep on SG
  associate_public_ip_address = true

  lifecycle {
    create_before_destroy = true  # see Task 6
  }

  tags = {
    Name = "TerraWeek-Server"
  }
}
```

After `terraform apply`, your EC2 instance will have a public IP. Verify with:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=TerraWeek-Server" \
  --query "Reservations[].Instances[].PublicIpAddress"
```

---

## Task 5: Explicit Dependencies with `depends_on`

### S3 Bucket with Explicit Dependency

```hcl
# ─────────────────────────────────────────────
# S3 Bucket for application logs
# No direct reference to the EC2 instance,
# but we want it created only after the instance
# is ready — so we use explicit depends_on
# ─────────────────────────────────────────────
resource "aws_s3_bucket" "app_logs" {
  bucket = "terraweek-app-logs-${random_id.suffix.hex}"

  tags = {
    Name = "TerraWeek-Logs"
  }

  # Explicit dependency — Terraform cannot infer this
  # because there is no attribute reference between these two resources
  depends_on = [aws_instance.main]
}

resource "random_id" "suffix" {
  byte_length = 4
}
```

### Visualizing the Dependency Graph

```bash
# Generate graph in DOT format and render to PNG (requires Graphviz)
terraform graph | dot -Tpng > graph.png

# If Graphviz is not installed, output raw DOT and paste into webgraphviz.com
terraform graph
```

**Example dependency graph (text representation):**

```
[root] → aws_instance.main
  → aws_subnet.public → aws_vpc.main
  → aws_security_group.web → aws_vpc.main
  → aws_internet_gateway.main → aws_vpc.main
  → aws_route_table.public
      → aws_vpc.main
      → aws_internet_gateway.main
  → aws_route_table_association.public
      → aws_subnet.public
      → aws_route_table.public

aws_s3_bucket.app_logs → aws_instance.main (explicit depends_on)
```

### When to Use `depends_on` in Real Projects

**Example 1 — IAM Role Propagation**
When you create an IAM role and immediately attach it to a Lambda function, AWS sometimes hasn't fully propagated the role yet. Adding `depends_on = [aws_iam_role_policy_attachment.lambda_exec]` ensures the policy is fully attached before the function is created, preventing `InvalidParameterValueException` errors.

**Example 2 — Database Schema Before Application**
If you have a script (via `null_resource` or `terraform-aws-modules/rds`) that runs database migrations, your application server should `depends_on` that migration resource, even if the app server config doesn't directly reference any migration output. Without it, Terraform might provision the app and the migration in parallel, causing the app to start before its schema exists.

---

## Task 6: Lifecycle Rules

### The Three Lifecycle Arguments

**`create_before_destroy`**

```hcl
lifecycle {
  create_before_destroy = true
}
```

Terraform's default behavior on resource replacement is destroy → create. This causes downtime. Setting `create_before_destroy = true` reverses this: the new resource is created first, then the old one is destroyed. Use this for EC2 instances, load balancers, and any resource where downtime is unacceptable.

**`prevent_destroy`**

```hcl
lifecycle {
  prevent_destroy = true
}
```

Terraform will refuse to destroy this resource and exit with an error if a `terraform destroy` or plan includes its deletion. Use this for production databases, S3 buckets with irreplaceable data, or any resource that would cause data loss if accidentally deleted.

**`ignore_changes`**

```hcl
lifecycle {
  ignore_changes = [ami, tags]
}
```

Terraform will not treat changes to the listed attributes as drift. Use this when:
- An external process (like an auto-scaler or deployment tool) modifies resource attributes after creation
- You manually update AMI versions via a deployment pipeline and don't want Terraform to revert them
- Tags are managed by another system (e.g., a tagging compliance tool)

### Destroy Order

Terraform always destroys resources in **reverse dependency order** — children before parents. In our stack:

```
1. aws_instance.main (depends on subnet + SG)
2. aws_s3_bucket.app_logs (depends on instance via depends_on)
3. aws_security_group.web (depends on VPC)
4. aws_route_table_association.public (depends on RT + subnet)
5. aws_route_table.public (depends on VPC + IGW)
6. aws_internet_gateway.main (depends on VPC)
7. aws_subnet.public (depends on VPC)
8. aws_vpc.main (root resource)
```

Clean up all resources when done to avoid AWS charges:

```bash
terraform destroy
```

---

## Implicit vs Explicit Dependencies — Summary

**Implicit dependencies** are created automatically when one resource references an attribute of another using the `<type>.<name>.<attribute>` syntax (e.g., `aws_vpc.main.id`). Terraform reads these references and builds the dependency graph without any extra instruction from you. This covers the vast majority of real-world use cases.

**Explicit dependencies** via `depends_on` are needed when two resources must be sequenced but share no attribute references. Common situations include: ordering side effects (IAM propagation delays, schema migrations), sequencing `null_resource` scripts, or ensuring a monitoring resource is set up before a service goes live. Use `depends_on` sparingly — it makes configs harder to read and can prevent parallelism that Terraform would otherwise exploit.

---

## Complete File Structure

```
terraform-aws-infra/
├── providers.tf
├── main.tf
├── .terraform.lock.hcl   (auto-generated, commit to git)
└── .terraform/           (auto-generated, do NOT commit)
```

---

## Key Commands Reference

```bash
terraform init          # Download providers, create lock file
terraform fmt           # Auto-format HCL files
terraform validate      # Check syntax and structure
terraform plan          # Preview changes
terraform apply         # Apply changes
terraform destroy       # Tear down all resources
terraform graph         # Output dependency graph in DOT format
```

---

*Day 62 of #90DaysOfDevOps | #TerraWeek | TrainWithShubham*
