# Day 64 — Terraform State Management and Remote Backends

## Overview

Terraform ka state file infrastructure ka "source of truth" hota hai — yeh `.tf` files aur actual cloud resources ke beech ka map hai. Agar yeh file kho jaye ya corrupt ho jaye, toh Terraform sab kuch bhool jaata hai. Aaj humne state ko professionally manage karna seekha — remote backends, locking, import, aur drift handling.

---

## Architecture Diagram: Local State vs Remote State

```
LOCAL STATE (❌ Risky)
┌─────────────────────────────┐
│         Developer           │
│  ┌──────────────────────┐   │
│  │  terraform.tfstate   │   │  ← Single point of failure
│  │  (local disk)        │   │  ← No locking
│  └──────────────────────┘   │  ← No versioning
│         │                   │  ← Team collaboration impossible
│    terraform apply           │
└─────────────────────────────┘

REMOTE STATE (✅ Production-Ready)
┌──────────────┐    ┌──────────────────────────────────────┐
│  Developer 1 │───▶│                                      │
└──────────────┘    │         S3 Bucket                    │
                    │   terraweek-state-<yourname>         │
┌──────────────┐    │   dev/terraform.tfstate              │◀─── Versioning ON
│  Developer 2 │───▶│                                      │
└──────────────┘    └───────────────┬──────────────────────┘
                                    │
                    ┌───────────────▼──────────────────────┐
                    │         DynamoDB Table               │
                    │      terraweek-state-lock            │
                    │  LockID (String) → State Lock        │◀─── Prevents concurrent writes
                    └──────────────────────────────────────┘
```

---

## Task 1: Current State Inspection

### Commands Run

```bash
terraform show                        # Full state in human-readable format
terraform state list                  # All resources tracked by Terraform
terraform state show aws_instance.web # Every attribute of the instance
terraform state show aws_vpc.main     # Every attribute of the VPC
```

### Answers

**Q: How many resources does Terraform track?**
> Terraform is tracking **4 resources** in this config:
> - `aws_vpc.main`
> - `aws_subnet.public`
> - `aws_instance.web`
> - `aws_security_group.allow_ssh`

**Q: What attributes does the state store for an EC2 instance?**
> Terraform state `.tf` file mein sirf `ami`, `instance_type`, `tags` define karte hain, lekin state file bahut zyada store karta hai:
> - `id`, `arn`, `private_ip`, `public_ip`, `public_dns`
> - `availability_zone`, `subnet_id`, `vpc_security_group_ids`
> - `root_block_device` details (volume_id, size, type, encrypted)
> - `credit_specification`, `metadata_options`, `ebs_optimized`
> - `tenancy`, `placement_group`, `key_name`
> - `monitoring`, `source_dest_check`, `hibernation`

**Q: What does the serial number in `terraform.tfstate` represent?**
> `serial` number ek monotonically increasing integer hai. Har baar state file update hoti hai (apply ya refresh ke baad), serial number 1 se badhta hai. Yeh conflict detection ke kaam aata hai — agar do processes same serial number ke saath write karein, Terraform error deta hai.

---

## Task 2: S3 Remote Backend Setup

### Step 1 — Backend Infrastructure Create Karna

```bash
# S3 bucket create karo state storage ke liye
aws s3api create-bucket \
  --bucket terraweek-state-yourname \
  --region ap-south-1 \
  --create-bucket-configuration LocationConstraint=ap-south-1

# Versioning enable karo (state recovery ke liye)
aws s3api put-bucket-versioning \
  --bucket terraweek-state-yourname \
  --versioning-configuration Status=Enabled

# DynamoDB table create karo state locking ke liye
aws dynamodb create-table \
  --table-name terraweek-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```

### Step 2 — Backend Block Add Karna `main.tf` mein

```hcl
terraform {
  backend "s3" {
    bucket         = "terraweek-state-yourname"
    key            = "dev/terraform.tfstate"
    region         = "ap-south-1"
    dynamodb_table = "terraweek-state-lock"
    encrypt        = true
  }
}
```

### Step 3 — Migration

```bash
terraform init
# Prompt: "Do you want to copy existing state to the new backend?" → YES
```

### Verification

```
✅ S3 bucket mein dev/terraform.tfstate file visible hai
✅ Local terraform.tfstate ab empty/gone hai
✅ terraform plan → "No changes. Your infrastructure matches the configuration."
```

> **Screenshot placeholder:** S3 bucket console mein `dev/terraform.tfstate` file dikhti hai with versioning enabled.

---

## Task 3: State Locking Test

### Test Steps

**Terminal 1:**
```bash
terraform apply
# Waiting for approval (yes/no)...
```

**Terminal 2 (simultaneously):**
```bash
terraform plan
```

### Lock Error Message

```
╷
│ Error: Error acquiring the state lock
│
│ Error message: ConditionalCheckFailedException: The conditional request failed
│ Lock Info:
│   ID:        a3f8c2d1-7b4e-4f9a-b2e1-1234567890ab
│   Path:      terraweek-state-yourname/dev/terraform.tfstate
│   Operation: OperationTypeApply
│   Who:       user@hostname
│   Version:   1.6.0
│   Created:   2026-04-08 10:30:00.123456789 +0000 UTC
│   Info:
│
│ Terraform acquires a state lock to protect the state from being written
│ by multiple users at the same time. Please resolve the issue above and try
│ again. For most commands, you can disable locking with the "-lock=false"
│ flag, but this is not recommended.
╵
```

### Stale Lock Remove Karna (agar zaroorat pade)

```bash
terraform force-unlock a3f8c2d1-7b4e-4f9a-b2e1-1234567890ab
```

### Locking Kyun Critical Hai?

> Team environments mein agar do log ek saath `terraform apply` chalayein aur locking na ho, toh:
> - Dono processes ek hi state file ko simultaneously overwrite kar sakti hain
> - Resources duplicate ho sakte hain ya delete ho sakte hain
> - State file corrupt ho sakti hai — yeh production disaster hai
>
> DynamoDB locking ensure karta hai ki ek waqt mein sirf ek hi operation state file write kar sake.

---

## Task 4: Existing Resource Import

### Scenario

AWS Console se manually ek S3 bucket banaya: `terraweek-import-test-yourname`

### Step 1 — Resource Block Likhna

```hcl
resource "aws_s3_bucket" "imported" {
  bucket = "terraweek-import-test-yourname"
}
```

### Step 2 — Import Karna

```bash
terraform import aws_s3_bucket.imported terraweek-import-test-yourname
```

**Output:**
```
aws_s3_bucket.imported: Importing from ID "terraweek-import-test-yourname"...
aws_s3_bucket.imported: Import prepared!
  Prepared aws_s3_bucket for import
aws_s3_bucket.imported: Refreshing state... [id=terraweek-import-test-yourname]

Import successful!
The resources that were imported are shown above. These resources are now in
your Terraform state and will henceforth be managed by Terraform.
```

### Step 3 — Plan Verify Karna

```bash
terraform plan
# Result: No changes. Your infrastructure matches the configuration.
```

```bash
terraform state list
# aws_instance.web
# aws_vpc.main
# aws_subnet.public
# aws_security_group.allow_ssh
# aws_s3_bucket.imported   ← ✅ New resource successfully imported
```

### `terraform import` vs Resource from Scratch — Difference

| Aspect | `terraform import` | Resource from Scratch |
|--------|-------------------|----------------------|
| **Resource existence** | Already exists in AWS | Terraform create karta hai |
| **State update** | State mein add karta hai, AWS mein kuch nahi banega | AWS mein naya resource banana padta hai |
| **Config likhna** | Manually config likhni padti hai | Config se hi resource banta hai |
| **Use case** | Legacy/manually created resources ko manage karna | Naye resources |
| **Risk** | Low (existing resource safe rehta hai) | N/A (naya resource) |
| **`terraform plan` after** | "No changes" chahiye (config match karni chahiye) | Resources "will be created" dikhenge |

---

## Task 5: State Surgery — `mv` aur `rm`

### 5a — Resource Rename (state mv)

```bash
# Current state list dekhna
terraform state list

# Resource rename karna state mein
terraform state mv aws_s3_bucket.imported aws_s3_bucket.logs_bucket
```

**Output:**
```
Move "aws_s3_bucket.imported" to "aws_s3_bucket.logs_bucket"
Successfully moved 1 object(s).
```

`.tf` file mein bhi naam update karo:
```hcl
resource "aws_s3_bucket" "logs_bucket" {  # ← imported se logs_bucket
  bucket = "terraweek-import-test-yourname"
}
```

```bash
terraform plan
# No changes. ✅
```

### 5b — Resource Remove from State (state rm)

```bash
terraform state rm aws_s3_bucket.logs_bucket
```

**Output:**
```
Removed aws_s3_bucket.logs_bucket
Successfully removed 1 object(s).
```

```bash
terraform plan
# Plan: 1 to add.
# Terraform bucket ke baare mein bhool gaya, lekin AWS mein abhi bhi hai
```

### 5c — Re-Import

```bash
terraform import aws_s3_bucket.logs_bucket terraweek-import-test-yourname
# Successfully imported again ✅
```

### Real-World Use Cases

| Command | Kab Use Karein |
|---------|----------------|
| `terraform state mv` | Jab `.tf` file mein resource ka naam refactor karna ho (e.g., `web` → `app_server`); module restructuring; resource ek module se doosre mein move karna |
| `terraform state rm` | Jab kisi resource ko Terraform management se hatana ho bina destroy kiye (e.g., koi aur tool manage karega); resource permanently exempt karna |

---

## Task 6: State Drift — Simulation aur Fix

### Drift Simulate Karna

1. `terraform apply` run kiya — sab kuch sync mein hai
2. AWS Console mein EC2 instance ki **Name tag** manually `ManuallyChanged` kar di
3. Ek aur tag add kiya: `Environment: manual-test`

### Drift Detect Karna

```bash
terraform plan
```

**Output:**
```
Terraform will perform the following actions:

  # aws_instance.web will be updated in-place
  ~ resource "aws_instance" "web" {
      ~ tags = {
          ~ "Name"        = "ManuallyChanged" -> "web-server"
          - "Environment" = "manual-test"
        }
        # (26 unchanged attributes hidden)
    }

Plan: 0 to add, 1 to change, 0 to destroy.
```

> Terraform ne detect kar liya ki reality (AWS) aur desired state (`.tf` file) mein difference hai.

### Option A — Drift Reconcile Karna (Apply)

```bash
terraform apply
```

```
Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
```

```bash
terraform plan
# No changes. Your infrastructure matches the configuration. ✅
# Drift resolved!
```

### Production Mein Drift Prevent Kaise Karein?

1. **AWS Console access restrict karo** — IAM policies se developers ko direct console changes se rokna
2. **CI/CD pipeline use karo** — Saari infrastructure changes GitHub Actions / GitLab CI / Jenkins se ho, directly terminal se nahi
3. **`terraform plan` regularly run karo** — Cron job se scheduled drift detection
4. **`terraform apply -refresh-only`** — State ko reality ke saath sync karo bina changes kiye
5. **AWS Config / CloudTrail enable karo** — Manual changes audit karte rehna
6. **Terraform Cloud / Atlantis use karo** — PR-based workflow enforce karta hai

---

## Command Quick Reference

| Command | Purpose |
|---------|---------|
| `terraform state list` | Tracked resources list karo |
| `terraform state show <resource>` | Resource ke saare attributes dekho |
| `terraform state mv <old> <new>` | Resource ko rename karo state mein |
| `terraform state rm <resource>` | Resource ko state se hatao (destroy nahi) |
| `terraform import <resource> <id>` | Existing AWS resource ko state mein laao |
| `terraform force-unlock <LOCK_ID>` | Stale lock remove karo |
| `terraform apply -refresh-only` | State ko reality se sync karo bina changes kiye |
| `terraform init -migrate-state` | State ko naye backend mein migrate karo |

---

## Key Learnings

- **State file = Terraform ki memory** — Isko remote backend par rakhna mandatory hai production mein
- **S3 + DynamoDB = Best combo** — Versioning + locking dono milte hain
- **`terraform import`** se legacy resources ko IaC mein laaya ja sakta hai
- **State surgery** (`mv`, `rm`) refactoring ke liye powerful tools hain — soch samajh ke use karo
- **Drift** real projects mein common hai — CI/CD aur access control se prevent karo

---

*Day 64 of #90DaysOfDevOps | #TerraWeek | #DevOpsKaJosh | #TrainWithShubham*
