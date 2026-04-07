# Day 61 — Introduction to Terraform and Your First AWS Infrastructure

## Task 1: Understanding Infrastructure as Code (IaC)

### What is IaC and Why Does It Matter?

Infrastructure as Code (IaC) is the practice of defining and managing your cloud infrastructure — servers, networks, storage, databases — using configuration files rather than manually clicking through a web console. Instead of logging into AWS and spinning up an EC2 instance by hand, you write a `.tf` file that describes what you want, and a tool like Terraform makes it happen. This matters in DevOps because infrastructure changes become version-controlled, repeatable, and reviewable just like application code. Teams can collaborate on infrastructure changes through pull requests, roll back bad changes with `git revert`, and reproduce entire environments from scratch in minutes.

### Problems IaC Solves Compared to Manual Console Work

When you create resources manually in the AWS console, you introduce several painful problems. First, there's no audit trail — you don't know who clicked what or when, so debugging production issues becomes a guessing game. Second, environments drift: the dev environment that worked six months ago has been tweaked so many times it no longer matches staging or production. Third, manual work doesn't scale — spinning up 50 identical EC2 instances by hand is error-prone and slow. IaC fixes all three: every change is a code commit, all environments are defined from the same source of truth, and you can create 50 instances by changing one number in a file.

### Terraform vs CloudFormation vs Ansible vs Pulumi

| Tool | Key Difference |
|------|---------------|
| **Terraform** | Cloud-agnostic, declarative HCL, manages state externally, works with AWS/GCP/Azure/etc. |
| **AWS CloudFormation** | AWS-only, declarative YAML/JSON, state managed by AWS, tightly integrated with AWS services |
| **Ansible** | Procedural/imperative, primarily a configuration management tool, agentless, great for OS-level setup but not ideal for cloud resource lifecycle |
| **Pulumi** | Uses real programming languages (Python, TypeScript, Go), cloud-agnostic, similar concept to Terraform but code-first instead of DSL-first |

The core distinction: Terraform and CloudFormation are purpose-built for provisioning infrastructure. Ansible is better for configuring software on existing servers. Pulumi gives developers a Terraform alternative where they can write actual code instead of a configuration language.

### Declarative vs Imperative — and Cloud-Agnostic

**Declarative** means you describe the *desired end state*, not the steps to get there. In Terraform you write: "I want an S3 bucket named `my-bucket`." Terraform figures out what API calls to make. Compare this to imperative/procedural tools where you write: "Call the S3 CreateBucket API, then call PutBucketVersioning, then..." Declarative is less error-prone because you don't have to think about the order of operations or handle cases where a resource already exists.

**Cloud-agnostic** means Terraform uses a provider plugin system. The same Terraform core tool and HCL language can provision resources on AWS, GCP, Azure, Cloudflare, GitHub, Kubernetes, and hundreds of other platforms — you just swap the provider block. You don't need to learn a completely different tool for each cloud.

---

## Task 2: Install Terraform and Configure AWS

### Installation

```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform

# Linux (Ubuntu/Debian)
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

### Verification

```bash
terraform -version
# Output: Terraform v1.x.x on linux_amd64
```

### AWS CLI Configuration

```bash
aws configure
# AWS Access Key ID: <your-key>
# AWS Secret Access Key: <your-secret>
# Default region name: ap-south-1
# Default output format: json

aws sts get-caller-identity
# Output: Your account ID, UserID, and ARN
```

---

## Task 3: First Terraform Config — S3 Bucket

### main.tf

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
  region = "ap-south-1"
}

resource "aws_s3_bucket" "my_bucket" {
  bucket = "terraweek-yourname-2026"

  tags = {
    Name        = "TerraWeek-Day1-Bucket"
    Environment = "Dev"
  }
}

resource "aws_instance" "my_ec2" {
  ami           = "ami-0f5ee92e2d63afc18"  # Amazon Linux 2 in ap-south-1
  instance_type = "t2.micro"

  tags = {
    Name = "TerraWeek-Day1"
  }
}
```

### Terraform Lifecycle Commands

```bash
terraform init      # Download the AWS provider plugin
terraform plan      # Preview what will be created/changed/destroyed
terraform apply     # Execute the plan (type 'yes' to confirm)
terraform destroy   # Destroy all managed resources (type 'yes' to confirm)
```

### What `terraform init` Downloaded

`terraform init` downloaded the **HashiCorp AWS provider plugin** (`hashicorp/aws`) from the Terraform Registry. The `.terraform/` directory contains:
- `providers/` — the compiled provider binary (platform-specific, e.g., `terraform-provider-aws_v5.x.x_x5`)
- `.terraform.lock.hcl` — a lockfile that pins exact provider versions to ensure reproducible runs across machines

> 📸 *[Screenshot of `terraform apply` output showing S3 bucket creation]*

> 📸 *[Screenshot of S3 bucket visible in AWS Console]*

---

## Task 4: Adding the EC2 Instance

After the S3 bucket already exists, running `terraform plan` showed only **1 resource to add** — the EC2 instance. The S3 bucket appeared as no-change.

### How Terraform Knows What Already Exists

Terraform compares two things on every `plan`:
1. **The desired state** — what your `.tf` files describe
2. **The current state** — what's recorded in `terraform.tfstate`

When you ran `terraform apply` for the S3 bucket, Terraform wrote its details (ARN, region, bucket name, etc.) into `terraform.tfstate`. On the next `plan`, Terraform read that state file, saw the bucket was already managed, queried AWS to confirm it still exists with the right attributes, and marked it as unchanged. Only the EC2 instance was new (not in state), so it was the only resource planned for creation.

> 📸 *[Screenshot of EC2 instance in AWS Console with tag "TerraWeek-Day1"]*

---

## Task 5: The State File

### State File Structure

`terraform.tfstate` is a JSON file with this rough structure:

```json
{
  "version": 4,
  "terraform_version": "1.x.x",
  "serial": 3,
  "lineage": "a-unique-uuid",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_s3_bucket",
      "name": "my_bucket",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 0,
          "attributes": {
            "arn": "arn:aws:s3:::terraweek-yourname-2026",
            "bucket": "terraweek-yourname-2026",
            "id": "terraweek-yourname-2026",
            "region": "ap-south-1",
            ...
          }
        }
      ]
    }
  ]
}
```

### State Inspection Commands

```bash
terraform show                                        # Human-readable current state
terraform state list                                  # List all managed resource addresses
terraform state show aws_s3_bucket.my_bucket          # Full details of the S3 bucket
terraform state show aws_instance.my_ec2              # Full details of the EC2 instance
```

### State File Q&A

**What information does the state file store about each resource?**
It stores every attribute AWS returned about the resource: the ARN, resource ID, region, tags, creation timestamps, and all configuration values. This is Terraform's source of truth — it's how Terraform knows what already exists and what needs to change.

**Why should you never manually edit the state file?**
The state file contains tightly coupled references between resources (IDs, ARNs, dependency chains). Editing it by hand can corrupt these references, causing Terraform to try to create resources that already exist, delete the wrong resources, or completely lose track of your infrastructure. Always use `terraform state` subcommands if you need to manipulate state.

**Why should the state file not be committed to Git?**
The state file can contain sensitive data like database passwords, private keys, and access credentials in plaintext. Committing it exposes secrets to everyone with repository access. Also, concurrent team members editing infrastructure could produce merge conflicts in the state file, which are extremely dangerous to resolve. The correct solution is **remote state** (e.g., an S3 bucket with DynamoDB locking) for team workflows.

---

## Task 6: Modify, Plan, and Destroy

### Plan Symbols Explained

| Symbol | Meaning |
|--------|---------|
| `+` | Resource will be **created** |
| `-` | Resource will be **destroyed** |
| `~` | Resource will be **updated in-place** (modified without destroy/recreate) |
| `-/+` | Resource will be **destroyed and recreated** (change requires replacement) |

### Changing the EC2 Tag

Changing `Name = "TerraWeek-Day1"` to `Name = "TerraWeek-Modified"` in main.tf, then running `terraform plan` showed:

```
  ~ resource "aws_instance" "my_ec2" {
      ~ tags = {
          ~ "Name" = "TerraWeek-Day1" -> "TerraWeek-Modified"
        }
    }
```

This is an **in-place update** (`~`), not a destroy-and-recreate. AWS can modify EC2 tags without stopping or replacing the instance. If you changed the AMI or instance type, you would see `-/+` (destroy and recreate).

### Destroy Everything

```bash
terraform destroy
# Type 'yes' to confirm
# Both the S3 bucket and EC2 instance are deleted
```

> 📸 *[Screenshot showing empty EC2 and S3 console after destroy]*

---

## Command Reference Summary

| Command | What It Does |
|---------|-------------|
| `terraform init` | Initializes the working directory, downloads providers and modules |
| `terraform plan` | Shows a preview of changes without applying anything |
| `terraform apply` | Applies the plan — creates/modifies/destroys real resources |
| `terraform destroy` | Destroys all resources managed by the current state |
| `terraform show` | Displays human-readable current state |
| `terraform state list` | Lists all resource addresses tracked in state |
| `terraform state show <resource>` | Shows full attributes of a specific resource |
| `terraform fmt` | Auto-formats `.tf` files to canonical style |
| `terraform validate` | Validates syntax without connecting to AWS |

---

## .gitignore for Terraform Projects

```
# Terraform state files — contain sensitive data, never commit
*.tfstate
*.tfstate.backup

# Provider plugins — large binaries, regenerated by terraform init
.terraform/

# Variable files may contain secrets
*.tfvars
*.tfvars.json
```

---

## Key Takeaways

- **IaC = version-controlled, repeatable infrastructure.** Write it once, run it anywhere.
- **Terraform's state file is its brain.** It tracks what exists so plans are accurate. Protect it.
- **Declarative means describe the goal, not the steps.** Terraform handles the API calls.
- **`plan` before every `apply`.** Read the output carefully — understand what changes before confirming.
- **S3 bucket names are globally unique.** Add your name and year to avoid conflicts.
- **AMI IDs are region-specific.** Always verify the correct AMI for your target region.
