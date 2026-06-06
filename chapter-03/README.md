# Chapter 3: State Management

## 📚 Learning Objectives

By the end of this chapter, you will:
- Understand Terraform state and its importance
- Configure remote state backends (S3)
- Implement state locking with DynamoDB
- Use state commands effectively
- Handle state file security
- Implement workspaces for environment management

**Prerequisites:** Chapters 1-2 completed  
**Estimated Time:** 3 days  
**Labs:** 3 hands-on exercises

---

## 🗄️ What is Terraform State?

### State File Purpose

Terraform state is a JSON file that maps your configuration to real-world resources.

**Key Functions:**
1. **Resource Tracking:** Maps config to actual resources
2. **Metadata Storage:** Stores resource dependencies
3. **Performance:** Caches resource attributes
4. **Collaboration:** Enables team workflows

**State File Structure:**
```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "serial": 1,
  "lineage": "unique-id",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "schema_version": 1,
          "attributes": {
            "id": "i-1234567890abcdef0",
            "ami": "ami-12345678",
            "instance_type": "t2.micro"
          }
        }
      ]
    }
  ]
}
```

---

## 🏠 Local vs Remote State

### Local State (Default)

```
project/
├── main.tf
├── terraform.tfstate          ← Local state file
└── terraform.tfstate.backup   ← Previous state
```

**Pros:**
- Simple setup
- No additional infrastructure
- Fast access

**Cons:**
- ❌ No collaboration
- ❌ No locking
- ❌ Risk of loss
- ❌ Secrets in plain text

### Remote State (Recommended)

```
┌─────────────────────────────────────────┐
│         Developer Workstation           │
│  ┌──────────────────────────────────┐   │
│  │  Terraform CLI                   │   │
│  └────────────┬─────────────────────┘   │
└───────────────┼─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│           AWS S3 Bucket                 │
│  ┌──────────────────────────────────┐   │
│  │  terraform.tfstate               │   │
│  │  (Encrypted, Versioned)          │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│         DynamoDB Table                  │
│  ┌──────────────────────────────────┐   │
│  │  State Locking                   │   │
│  │  (Prevents concurrent changes)   │   │
│  └──────────────────────────────────┘   │
└─────────────────────────────────────────┘
```

**Pros:**
- ✅ Team collaboration
- ✅ State locking
- ✅ Versioning
- ✅ Encryption
- ✅ Backup and recovery

---

## 🔐 S3 Backend Configuration

### Step 1: Create S3 Bucket and DynamoDB Table

```hcl
# backend-setup/main.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# S3 Bucket for state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-${data.aws_caller_identity.current.account_id}"
  
  tags = {
    Name        = "Terraform State"
    Environment = "shared"
  }
}

# Enable versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB table for locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  tags = {
    Name        = "Terraform State Locks"
    Environment = "shared"
  }
}

# Get current AWS account ID
data "aws_caller_identity" "current" {}

# Outputs
output "s3_bucket_name" {
  value = aws_s3_bucket.terraform_state.id
}

output "dynamodb_table_name" {
  value = aws_dynamodb_table.terraform_locks.name
}
```

### Step 2: Configure Backend in Your Project

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "my-terraform-state-123456789012"
    key            = "project/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}

provider "aws" {
  region = "us-east-1"
}
```

### Step 3: Initialize Backend

```bash
# Initialize with backend
terraform init

# Migrate existing state
terraform init -migrate-state
```

---

## 🔒 State Locking

### How It Works

```
Developer A                    DynamoDB Lock Table
    │                                 │
    ├─► terraform apply               │
    │   ├─► Acquire lock ────────────►│
    │   │                             │ Lock acquired
    │   │                             │ (LockID: unique-id)
    │   │                             │
    │   ├─► Make changes              │
    │   │                             │
    │   └─► Release lock ─────────────►│
    │                                 │ Lock released
    │                                 │
Developer B                           │
    │                                 │
    ├─► terraform apply               │
    │   ├─► Acquire lock ────────────►│
    │   │                             │ ❌ Lock held by A
    │   │                             │ Wait or fail
    │   │◄──── Error ─────────────────┤
```

**Lock Information:**
```json
{
  "ID": "unique-lock-id",
  "Operation": "OperationTypeApply",
  "Info": "user@hostname",
  "Who": "user@hostname",
  "Version": "1.5.0",
  "Created": "2024-01-15T10:30:00Z",
  "Path": "project/terraform.tfstate"
}
```

### Force Unlock (Use Carefully!)

```bash
# If lock is stuck
terraform force-unlock <LOCK_ID>

# Example
terraform force-unlock a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

## 🛠️ State Commands

### View State

```bash
# List all resources
terraform state list

# Show specific resource
terraform state show aws_instance.web

# Show all state
terraform show
```

### Move Resources

```bash
# Rename resource in state
terraform state mv aws_instance.web aws_instance.web_server

# Move to module
terraform state mv aws_instance.web module.web.aws_instance.server

# Move from module
terraform state mv module.web.aws_instance.server aws_instance.web
```

### Remove Resources

```bash
# Remove from state (doesn't destroy)
terraform state rm aws_instance.web

# Remove module
terraform state rm module.web
```

### Import Existing Resources

```bash
# Import EC2 instance
terraform import aws_instance.web i-1234567890abcdef0

# Import S3 bucket
terraform import aws_s3_bucket.data my-bucket-name

# Import VPC
terraform import aws_vpc.main vpc-12345678
```

### Pull/Push State

```bash
# Download state
terraform state pull > terraform.tfstate.backup

# Upload state (dangerous!)
terraform state push terraform.tfstate.backup
```

---

## 🌍 Workspaces

### What are Workspaces?

Workspaces allow multiple state files for the same configuration.

```
project/
├── main.tf
└── terraform.tfstate.d/
    ├── dev/
    │   └── terraform.tfstate
    ├── staging/
    │   └── terraform.tfstate
    └── prod/
        └── terraform.tfstate
```

### Workspace Commands

```bash
# List workspaces
terraform workspace list

# Create workspace
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch workspace
terraform workspace select dev

# Show current workspace
terraform workspace show

# Delete workspace
terraform workspace delete dev
```

### Using Workspaces in Configuration

```hcl
# main.tf
locals {
  environment = terraform.workspace
  
  instance_counts = {
    dev     = 1
    staging = 2
    prod    = 5
  }
  
  instance_types = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t2.medium"
  }
}

resource "aws_instance" "app" {
  count = local.instance_counts[local.environment]
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = local.instance_types[local.environment]
  
  tags = {
    Name        = "app-${local.environment}-${count.index + 1}"
    Environment = local.environment
  }
}

output "environment" {
  value = terraform.workspace
}
```

### Workspace-Specific Backend

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "env/${terraform.workspace}/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

---

## 🔐 State Security Best Practices

### 1. Encryption at Rest

```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform.arn
    }
  }
}
```

### 2. Encryption in Transit

```hcl
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "terraform.tfstate"
    region  = "us-east-1"
    encrypt = true  # Enables encryption
  }
}
```

### 3. Access Control

```hcl
resource "aws_s3_bucket_policy" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "DenyUnencryptedObjectUploads"
        Effect = "Deny"
        Principal = "*"
        Action = "s3:PutObject"
        Resource = "${aws_s3_bucket.terraform_state.arn}/*"
        Condition = {
          StringNotEquals = {
            "s3:x-amz-server-side-encryption" = "AES256"
          }
        }
      },
      {
        Sid    = "AllowTerraformAccess"
        Effect = "Allow"
        Principal = {
          AWS = aws_iam_role.terraform.arn
        }
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "${aws_s3_bucket.terraform_state.arn}/*"
      }
    ]
  })
}
```

### 4. Versioning

```hcl
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Lifecycle rule to manage old versions
resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    id     = "expire-old-versions"
    status = "Enabled"
    
    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}
```

---

## 🚨 Common State Issues

### Issue 1: State Drift

**Problem:** Real resources differ from state

**Detection:**
```bash
terraform plan  # Shows drift
terraform refresh  # Updates state
```

**Solution:**
```bash
# Option 1: Update state to match reality
terraform refresh

# Option 2: Update resources to match state
terraform apply
```

### Issue 2: Corrupted State

**Problem:** State file is corrupted

**Solution:**
```bash
# Restore from S3 version
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix terraform.tfstate

# Download specific version
aws s3api get-object \
  --bucket my-terraform-state \
  --key terraform.tfstate \
  --version-id <VERSION_ID> \
  terraform.tfstate.restored

# Push restored state
terraform state push terraform.tfstate.restored
```

### Issue 3: Lost State

**Problem:** State file deleted

**Solution:**
```bash
# If using S3 with versioning
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix terraform.tfstate

# Restore latest version
aws s3api get-object \
  --bucket my-terraform-state \
  --key terraform.tfstate \
  --version-id <VERSION_ID> \
  terraform.tfstate

# Or import resources manually
terraform import aws_instance.web i-1234567890abcdef0
```

---

## 📊 State File Analysis

### Inspect State

```bash
# View state in JSON
terraform show -json | jq

# List resources
terraform state list

# Show resource details
terraform state show aws_instance.web

# Get outputs
terraform output
terraform output -json
```

### State Statistics

```bash
# Count resources
terraform state list | wc -l

# Resources by type
terraform state list | cut -d. -f1 | sort | uniq -c

# Find specific resources
terraform state list | grep aws_instance
```

---

## 📖 Key Takeaways

1. **Always use remote state** for team collaboration
2. **Enable state locking** to prevent conflicts
3. **Encrypt state files** (contains sensitive data)
4. **Enable versioning** for state recovery
5. **Use workspaces** for environment separation
6. **Regular backups** of state files
7. **Restrict access** to state bucket
8. **Never edit state manually** (use terraform state commands)

---

## 🔗 Next Steps

1. Review theory above
2. Complete [Lab 3.1](./LABS.md) - Remote State Setup
3. Complete [Lab 3.2](./LABS.md) - State Commands
4. Complete [Lab 3.3](./LABS.md) - Workspaces
5. Update [CHECKLIST.md](../CHECKLIST.md)

**Next Chapter:** [Chapter 4: Modules](../chapter-04/)
