# Day 63 — Variables, Outputs, Data Sources and Expressions

## Overview

Today we take the hardcoded Day 62 infrastructure and make it fully dynamic. Every value that could change between environments — region, CIDR blocks, AMI IDs, instance types, tags — becomes a variable, data source, or local. The result is a config that works across dev, staging, and prod without touching `main.tf`.

---

## Task 1: Variables

### `variables.tf`

```hcl
# ─────────────────────────────────────────────
# Provider / Region
# ─────────────────────────────────────────────
variable "region" {
  description = "AWS region to deploy resources into"
  type        = string
  default     = "us-east-1"
}

# ─────────────────────────────────────────────
# Networking
# ─────────────────────────────────────────────
variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_cidr" {
  description = "CIDR block for the public subnet"
  type        = string
  default     = "10.0.1.0/24"
}

# ─────────────────────────────────────────────
# Compute
# ─────────────────────────────────────────────
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

# ─────────────────────────────────────────────
# Project metadata — no default, user must supply
# ─────────────────────────────────────────────
variable "project_name" {
  description = "Name of the project — used in resource names and tags"
  type        = string
  # No default: Terraform will prompt for this value interactively
  # or it must be provided via tfvars / CLI / env var
}

variable "environment" {
  description = "Deployment environment (dev, staging, prod)"
  type        = string
  default     = "dev"
}

# ─────────────────────────────────────────────
# Security Group — list of ports to open inbound
# ─────────────────────────────────────────────
variable "allowed_ports" {
  description = "List of TCP ports to allow inbound in the security group"
  type        = list(number)
  default     = [22, 80, 443]
}

# ─────────────────────────────────────────────
# Tags — map of extra key/value pairs merged into all resources
# ─────────────────────────────────────────────
variable "extra_tags" {
  description = "Additional tags to merge onto every resource"
  type        = map(string)
  default     = {}
}
```

### The Five Terraform Variable Types

| Type | Description | Example |
|---|---|---|
| `string` | A single text value | `"us-east-1"` |
| `number` | An integer or float | `2`, `3.14` |
| `bool` | True or false | `true`, `false` |
| `list(type)` | An ordered sequence of values | `["a", "b", "c"]` |
| `map(type)` | Key-value pairs (string keys) | `{ env = "dev", team = "ops" }` |

There are also complex types built from these: `set` (unordered unique list), `object` (named attribute map with typed fields), and `tuple` (fixed-length sequence of mixed types).

---

## Task 2: Variable Files and Precedence

### `terraform.tfvars` (loaded automatically)

```hcl
project_name  = "terraweek"
environment   = "dev"
instance_type = "t2.micro"
```

### `prod.tfvars` (loaded with `-var-file`)

```hcl
project_name  = "terraweek"
environment   = "prod"
instance_type = "t3.small"
vpc_cidr      = "10.1.0.0/16"
subnet_cidr   = "10.1.1.0/24"
```

### Usage

```bash
# Uses terraform.tfvars automatically
terraform plan

# Uses prod.tfvars (overrides anything in terraform.tfvars for matching keys)
terraform plan -var-file="prod.tfvars"

# CLI flag overrides everything
terraform plan -var="instance_type=t2.nano"

# Environment variable — overrides default but NOT tfvars or CLI
export TF_VAR_environment="staging"
terraform plan
```

### Variable Precedence (Lowest → Highest)

```
1. Default value in variables.tf           (lowest priority)
2. terraform.tfvars (auto-loaded)
3. *.auto.tfvars files (alphabetical order)
4. -var-file="file.tfvars" (CLI flag)
5. -var="key=value" (CLI flag)
6. TF_VAR_<name> environment variables     (highest priority)
```

**Practical example:** If `environment = "dev"` in `terraform.tfvars` but you also run `export TF_VAR_environment=staging`, the env var wins and Terraform uses `"staging"`. If you then add `-var="environment=prod"` to the command, the CLI flag wins and Terraform uses `"prod"`.

---

## Task 3: Outputs

### `outputs.tf`

```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "subnet_id" {
  description = "ID of the public subnet"
  value       = aws_subnet.public.id
}

output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.main.id
}

output "instance_public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.main.public_ip
}

output "instance_public_dns" {
  description = "Public DNS name of the EC2 instance"
  value       = aws_instance.main.public_dns
}

output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.web.id
}
```

### Using Outputs

```bash
terraform apply                          # Outputs printed at the end of apply

terraform output                         # Show all outputs
terraform output instance_public_ip      # Show a specific output
terraform output -json                   # JSON — useful for piping into scripts

# Example: grab the IP in a shell script
IP=$(terraform output -raw instance_public_ip)
ssh ec2-user@$IP
```

**Sample output after `terraform apply`:**

```
Apply complete! Resources: 7 added, 0 changed, 0 destroyed.

Outputs:

instance_id         = "i-0abc123def456789"
instance_public_dns = "ec2-54-123-45-67.compute-1.amazonaws.com"
instance_public_ip  = "54.123.45.67"
security_group_id   = "sg-0def123456789abc"
subnet_id           = "subnet-0abc123def456789"
vpc_id              = "vpc-0abc123def456789"
```

---

## Task 4: Data Sources

### `data.tf`

```hcl
# ─────────────────────────────────────────────
# Fetch the latest Amazon Linux 2 AMI dynamically
# No more hardcoded AMI IDs that go stale or break in other regions
# ─────────────────────────────────────────────
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }

  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

# ─────────────────────────────────────────────
# Fetch all available AZs in the current region
# Use names[0] to pin the subnet to the first AZ
# ─────────────────────────────────────────────
data "aws_availability_zones" "available" {
  state = "available"
}
```

### Using Data Sources in `main.tf`

```hcl
resource "aws_instance" "main" {
  ami           = data.aws_ami.amazon_linux.id    # dynamic — always latest AMI
  instance_type = var.instance_type
  # ...
}

resource "aws_subnet" "public" {
  availability_zone = data.aws_availability_zones.available.names[0]  # first AZ
  # ...
}
```

### Resource vs Data Source

| Aspect | `resource` | `data` |
|---|---|---|
| Purpose | Creates and manages infrastructure | Reads existing infrastructure |
| State | Tracked in `terraform.state` | Not stored in state |
| Lifecycle | Terraform owns it (create/update/destroy) | Read-only, Terraform never modifies it |
| Example | `resource "aws_vpc" "main"` | `data "aws_ami" "amazon_linux"` |
| Use case | Build new things | Look up AMIs, existing VPCs, account IDs, AZs |

A **resource** is something Terraform creates and takes responsibility for. A **data source** is a read-only query — Terraform asks AWS "what does this look like right now?" and uses the answer, but never touches the underlying object.

---

## Task 5: Locals for Dynamic Values

### `locals.tf`

```hcl
locals {
  # Consistent name prefix used across all resource names
  name_prefix = "${var.project_name}-${var.environment}"

  # Tags applied to every resource, merged with resource-specific tags
  common_tags = merge(
    {
      Project     = var.project_name
      Environment = var.environment
      ManagedBy   = "Terraform"
    },
    var.extra_tags  # user-supplied extra tags layered on top
  )
}
```

### Using Locals in `main.tf`

```hcl
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-vpc"
  })
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.subnet_cidr
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[0]
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-subnet"
  })
}

resource "aws_instance" "main" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.environment == "prod" ? "t3.small" : "t2.micro"
  subnet_id     = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  associate_public_ip_address = true
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-server"
  })
}
```

After `terraform apply`, every resource in the AWS console carries `Project`, `Environment`, and `ManagedBy` tags automatically, with no duplication in the config.

---

## Task 6: Built-in Functions and Conditional Expressions

### Exploring in `terraform console`

```bash
terraform console
```

**String functions:**

```hcl
upper("terraweek")
# → "TERRAWEEK"

join("-", ["terra", "week", "2026"])
# → "terra-week-2026"

format("arn:aws:s3:::%s", "my-bucket")
# → "arn:aws:s3:::my-bucket"
```

**Collection functions:**

```hcl
length(["a", "b", "c"])
# → 3

lookup({dev = "t2.micro", prod = "t3.small"}, "dev")
# → "t2.micro"

toset(["a", "b", "a"])
# → toset(["a", "b"])   — duplicates removed
```

**Networking function:**

```hcl
cidrsubnet("10.0.0.0/16", 8, 1)
# → "10.0.1.0/24"
# Carves out the 2nd /24 block from a /16
```

**Conditional expression:**

```hcl
instance_type = var.environment == "prod" ? "t3.small" : "t2.micro"
# If environment is "prod" → t3.small, otherwise → t2.micro
```

### Five Most Useful Functions

**1. `merge(map1, map2, ...)`**
Combines multiple maps into one, with later maps overriding earlier ones for duplicate keys. Indispensable for tag management — define common tags once in a local, then merge per-resource tags on top.

```hcl
merge({ Project = "web", Env = "dev" }, { Name = "my-vpc" })
# → { Project = "web", Env = "dev", Name = "my-vpc" }
```

**2. `cidrsubnet(prefix, newbits, netnum)`**
Calculates a subnet CIDR from a parent block mathematically. Instead of hardcoding `10.0.1.0/24`, `10.0.2.0/24`, etc., generate them programmatically — great with `count` or `for_each` to create multiple subnets.

```hcl
cidrsubnet("10.0.0.0/16", 8, 0)  # → "10.0.0.0/24"
cidrsubnet("10.0.0.0/16", 8, 1)  # → "10.0.1.0/24"
cidrsubnet("10.0.0.0/16", 8, 2)  # → "10.0.2.0/24"
```

**3. `lookup(map, key, default)`**
Safely retrieves a value from a map with a fallback if the key doesn't exist. Useful for environment-to-instance-type mappings without complex conditionals.

```hcl
lookup({ dev = "t2.micro", prod = "t3.small" }, var.environment, "t2.micro")
```

**4. `format(format_string, values...)`**
Produces formatted strings — cleaner than string interpolation for complex patterns like ARNs, bucket names with account IDs, or S3 paths.

```hcl
format("arn:aws:iam::%s:role/%s", data.aws_caller_identity.current.account_id, var.role_name)
```

**5. `toset(list)`**
Converts a list to a set, removing duplicates and losing ordering. Required when using `for_each` with a list variable, because `for_each` requires a set or map — not a list.

```hcl
for_each = toset(var.allowed_ports)
```

---

## Concept Comparison: variable vs local vs output vs data

| Concept | Direction | Purpose | When to Use |
|---|---|---|---|
| `variable` | Input → config | Accept values from outside (user, tfvars, env) | Anything that changes between environments or users |
| `local` | Internal | Compute derived values inside the config | Avoid repeating the same expression; combine variables into a prefix/tag |
| `output` | Config → outside | Expose values after apply | Share resource IDs with other configs, scripts, or humans |
| `data` | AWS → config | Query existing infrastructure | Look up AMIs, existing VPCs, account IDs, AZs without creating anything |

---

## Complete File Structure

```
terraform-aws-infra/
├── providers.tf         # terraform block + provider "aws"
├── variables.tf         # all input variable declarations
├── locals.tf            # computed locals (name_prefix, common_tags)
├── data.tf              # data sources (AMI, AZs)
├── main.tf              # resources (VPC, subnet, IGW, RT, SG, EC2)
├── outputs.tf           # output values
├── terraform.tfvars     # dev defaults (auto-loaded)
├── prod.tfvars          # prod overrides (-var-file)
└── .terraform.lock.hcl  # provider lock file (commit to git)
```

---

## Key Commands Reference

```bash
terraform init                           # Download providers
terraform plan                           # Preview with terraform.tfvars
terraform plan -var-file="prod.tfvars"   # Preview prod environment
terraform plan -var="instance_type=t2.nano"  # CLI override
terraform apply                          # Apply and print outputs
terraform output                         # Show all outputs
terraform output -json                   # JSON for scripting
terraform console                        # Interactive expression REPL
terraform destroy                        # Tear down all resources
```

---

*Day 63 of #90DaysOfDevOps | #TerraWeek | TrainWithShubham*
