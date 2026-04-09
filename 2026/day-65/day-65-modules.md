# Day 65 — Terraform Modules: Build Reusable Infrastructure

## Overview

Ek bade `main.tf` mein sab kuch likhna learning ke liye theek hai, lekin real teams mein yeh approach fail ho jaati hai. Terraform modules programming mein functions ki tarah hain — ek baar likho, baar baar use karo. Aaj humne custom EC2 module, security group module, aur official VPC registry module use kiya.

---

## Task 1: Module Structure

### Directory Tree

```
terraform-modules/
├── main.tf                    # Root module — child modules ko call karta hai
├── variables.tf               # Root-level variables
├── outputs.tf                 # Root-level outputs
├── providers.tf               # AWS provider config
└── modules/
    ├── ec2-instance/
    │   ├── main.tf            # aws_instance resource definition
    │   ├── variables.tf       # Module inputs (ami_id, instance_type, etc.)
    │   └── outputs.tf         # Module outputs (instance_id, public_ip, etc.)
    └── security-group/
        ├── main.tf            # aws_security_group resource definition
        ├── variables.tf       # Module inputs (vpc_id, sg_name, ports)
        └── outputs.tf         # Module outputs (sg_id)
```

### Root Module vs Child Module

| Aspect | Root Module | Child Module |
|--------|-------------|--------------|
| **Location** | Project ka top-level directory | `modules/` ke andar subdirectory |
| **Call karta hai** | Child modules ko `module {}` block se | Directly resources define karta hai |
| **`terraform apply`** | Yahan se run hota hai | Directly apply nahi hota |
| **Source** | `.` (current directory) hota hai | `./modules/<name>` ya registry URL |
| **Analogy** | `main()` function | Helper functions |

> **Simple rule:** Jis directory mein `terraform apply` chalate ho — woh root module. Baaki sab child modules hain.

---

## Task 2: Custom EC2 Module

### `modules/ec2-instance/variables.tf`

```hcl
variable "ami_id" {
  description = "AMI ID for the EC2 instance"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "subnet_id" {
  description = "Subnet ID where instance will be launched"
  type        = string
}

variable "security_group_ids" {
  description = "List of security group IDs to attach"
  type        = list(string)
}

variable "instance_name" {
  description = "Name tag for the EC2 instance"
  type        = string
}

variable "tags" {
  description = "Additional tags to apply to the instance"
  type        = map(string)
  default     = {}
}
```

### `modules/ec2-instance/main.tf`

```hcl
resource "aws_instance" "this" {
  ami                    = var.ami_id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_id
  vpc_security_group_ids = var.security_group_ids

  tags = merge(
    { "Name" = var.instance_name },
    var.tags
  )
}
```

### `modules/ec2-instance/outputs.tf`

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.this.id
}

output "public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.this.public_ip
}

output "private_ip" {
  description = "Private IP address of the EC2 instance"
  value       = aws_instance.this.private_ip
}
```

---

## Task 3: Custom Security Group Module

### `modules/security-group/variables.tf`

```hcl
variable "vpc_id" {
  description = "VPC ID where security group will be created"
  type        = string
}

variable "sg_name" {
  description = "Name for the security group"
  type        = string
}

variable "ingress_ports" {
  description = "List of ports to allow inbound traffic"
  type        = list(number)
  default     = [22, 80]
}

variable "tags" {
  description = "Tags to apply to the security group"
  type        = map(string)
  default     = {}
}
```

### `modules/security-group/main.tf`

```hcl
resource "aws_security_group" "this" {
  name        = var.sg_name
  description = "Security group managed by Terraform module"
  vpc_id      = var.vpc_id

  # Dynamic block — ingress_ports list se automatically rules banata hai
  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "Allow port ${ingress.value}"
    }
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
    description = "Allow all outbound traffic"
  }

  tags = merge(
    { "Name" = var.sg_name },
    var.tags
  )
}
```

> **Dynamic Block kya hai?**
> `dynamic "ingress"` block `ingress_ports` list ko loop karta hai aur har port ke liye ek alag `ingress` rule create karta hai. `[22, 80, 443]` doge toh 3 ingress rules bantenge — manually likhne ki zaroorat nahi.

### `modules/security-group/outputs.tf`

```hcl
output "sg_id" {
  description = "ID of the created security group"
  value       = aws_security_group.this.id
}
```

---

## Task 4: Root Module — Modules Ko Wire Karna

### `providers.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }

  backend "s3" {
    bucket         = "terraweek-state-yourname"
    key            = "day65/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraweek-state-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = "ap-south-1"
}
```

### `variables.tf`

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "dev"
}
```

### `main.tf` (Root — Custom Modules + VPC)

```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = "TerraWeek"
    ManagedBy   = "Terraform"
    Day         = "65"
  }
}

# Latest Amazon Linux 2 AMI fetch karo
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# VPC aur Subnet (Task 5 mein registry module se replace hoga)
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true

  tags = merge({ "Name" = "terraweek-vpc" }, local.common_tags)
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "ap-south-1a"
  map_public_ip_on_launch = true

  tags = merge({ "Name" = "terraweek-public-subnet" }, local.common_tags)
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  tags   = merge({ "Name" = "terraweek-igw" }, local.common_tags)
}

# Security Group Module call karo
module "web_sg" {
  source        = "./modules/security-group"
  vpc_id        = aws_vpc.main.id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = local.common_tags
}

# Web Server — EC2 Module ka pehla call
module "web_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-web"
  tags               = local.common_tags
}

# API Server — Wahi module, alag config!
module "api_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = aws_subnet.public.id
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-api"
  tags               = local.common_tags
}
```

### `outputs.tf` (Root)

```hcl
output "web_server_ip" {
  description = "Public IP of the web server"
  value       = module.web_server.public_ip
}

output "api_server_ip" {
  description = "Public IP of the API server"
  value       = module.api_server.public_ip
}

output "web_server_id" {
  description = "Instance ID of the web server"
  value       = module.web_server.instance_id
}

output "api_server_id" {
  description = "Instance ID of the API server"
  value       = module.api_server.instance_id
}

output "security_group_id" {
  description = "ID of the web security group"
  value       = module.web_sg.sg_id
}
```

### Apply Karna

```bash
terraform init    # Local modules link karta hai
terraform plan    # Saare resources preview
terraform apply   # Infrastructure create karo
```

### Expected Plan Output

```
Plan: 7 to add, 0 to change, 0 to destroy.

  + module.web_sg.aws_security_group.this
  + module.web_server.aws_instance.this
  + module.api_server.aws_instance.this
  + aws_vpc.main
  + aws_subnet.public
  + aws_internet_gateway.main
  + (route table etc.)
```

> **Screenshot placeholder:** AWS Console mein `terraweek-web` aur `terraweek-api` — dono instances running dikhte hain, same security group attached.

---

## Task 5: Public Registry Module — Official VPC Module

### Hand-written VPC ko replace karo

`main.tf` mein `aws_vpc`, `aws_subnet`, `aws_internet_gateway` blocks hatao aur yeh add karo:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "terraweek-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-south-1a", "ap-south-1b"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnets = ["10.0.3.0/24", "10.0.4.0/24"]

  enable_nat_gateway   = false
  enable_dns_hostnames = true

  tags = local.common_tags
}
```

### Module References Update Karo

```hcl
# Security Group module mein vpc_id update
module "web_sg" {
  source        = "./modules/security-group"
  vpc_id        = module.vpc.vpc_id          # ← module.vpc.vpc_id
  sg_name       = "terraweek-web-sg"
  ingress_ports = [22, 80, 443]
  tags          = local.common_tags
}

# EC2 modules mein subnet_id update
module "web_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = module.vpc.public_subnets[0]  # ← module.vpc.public_subnets[0]
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-web"
  tags               = local.common_tags
}

module "api_server" {
  source             = "./modules/ec2-instance"
  ami_id             = data.aws_ami.amazon_linux.id
  instance_type      = "t2.micro"
  subnet_id          = module.vpc.public_subnets[0]
  security_group_ids = [module.web_sg.sg_id]
  instance_name      = "terraweek-api"
  tags               = local.common_tags
}
```

```bash
terraform init     # Registry module download hoga
terraform plan
terraform apply
```

### Hand-written VPC vs Registry VPC Module — Comparison

| Metric | Hand-written VPC (Day 62) | Registry Module `terraform-aws-modules/vpc/aws` |
|--------|--------------------------|------------------------------------------------|
| **Lines of code** | ~50 lines | ~12 lines |
| **Resources created** | 4-5 (VPC, subnet, IGW, route table, association) | **20+ resources** automatically |
| **Multi-AZ support** | Manual | Built-in |
| **Private subnets** | Manual | Built-in |
| **NAT Gateway option** | Manual | `enable_nat_gateway = true` se ho jaata hai |
| **DNS settings** | Manual | `enable_dns_hostnames = true` |
| **Maintenance** | Khud karo | Community maintained |

> Registry module ne ~20 resources banaye: VPC, 2 public subnets, 2 private subnets, IGW, route tables (public + private), route table associations, default security group, default network ACL, default route table.

### Modules Download Location

```bash
ls .terraform/modules/
```

```
.terraform/modules/
├── modules.json                          # Module registry/source map
├── vpc/                                  # terraform-aws-modules/vpc/aws
│   └── terraform-aws-modules-terraform-aws-vpc-<hash>/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── ...
```

> Terraform registry modules `.terraform/modules/` directory mein download hote hain. Isko `.gitignore` mein daalna chahiye — yeh generated files hain, source control mein nahi jani chahiye.

---

## Task 6: Module Versioning aur Best Practices

### Version Pinning Options

```hcl
# Exact version (most strict — CI/CD ke liye best)
version = "5.1.0"

# Minor version range (recommended — patches allow karta hai)
version = "~> 5.0"

# Range (flexible)
version = ">= 5.0, < 6.0"
```

```bash
# Newer versions check karo
terraform init -upgrade
```

### State mein Module Prefix

```bash
terraform state list
```

```
module.vpc.aws_vpc.this[0]
module.vpc.aws_subnet.public[0]
module.vpc.aws_subnet.public[1]
module.vpc.aws_subnet.private[0]
module.vpc.aws_subnet.private[1]
module.vpc.aws_internet_gateway.this[0]
module.vpc.aws_route_table.public[0]
module.web_sg.aws_security_group.this
module.web_server.aws_instance.this
module.api_server.aws_instance.this
```

> State mein `module.<name>.<resource_type>.<resource_name>` format follow hota hai. Isse clearly pata chalta hai kaunsa resource kis module se aaya.

### Destroy

```bash
terraform destroy
# Sab kuch clean ho jaata hai
```

---

## Module Best Practices (Apne Shabdon Mein)

### 1. Registry Modules ke liye Version Hamesha Pin Karo
Registry module ka version fix na karo toh `terraform init` pe kabhi bhi breaking change aa sakti hai. `version = "~> 5.0"` likhne se team ke sab log same version use karte hain.

### 2. Module Focused Rakho — Ek Kaam, Ek Module
`ec2-instance` module sirf EC2 banaye, VPC nahi. `security-group` module sirf SG banaye. Agar ek module sab kuch karne lage toh testing mushkil ho jaati hai aur reuse nahi ho pata.

### 3. Variables se Sab Kuch Pass Karo — Hardcode Kuch Nahi
Module ke andar koi bhi value directly mat likho. `ami_id`, `region`, `instance_type` — sab variables se aana chahiye. Isse ek hi module dev/staging/prod teeno mein use ho sakta hai.

### 4. Outputs Hamesha Define Karo
Agar module se output nahi nikaloge toh caller module us resource ki koi bhi property use nahi kar sakta. `instance_id`, `sg_id`, `public_ip` — yeh sab expose karo taaki root module mein reference kar sako.

### 5. Har Custom Module mein `README.md` Daalo
Module ka `README.md` batata hai: kya karta hai, kaun se variables required hain, kaun se outputs milte hain, aur example usage. Bina README ke team ka koi bhi member module samajhne ke liye source code padhega — time waste.

---

## Key Learnings

- **Modules = Reusability** — Ek EC2 module se `web_server` aur `api_server` dono bane, code duplication zero
- **`dynamic` block** — List se automatically repeated blocks generate karta hai
- **Registry modules** — 50 lines ka VPC code 12 lines mein aur 20+ resources automatically
- **Module outputs** — `module.<name>.<output>` se parent module mein values access hoti hain
- **`.terraform/modules/`** — Registry modules yahan download hote hain, git mein nahi jaate

---

*Day 65 of #90DaysOfDevOps | #TerraWeek | #DevOpsKaJosh | #TrainWithShubham*
