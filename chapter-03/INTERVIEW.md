# Chapter 3: Interview Questions - State Management

## 📋 Overview

This document contains interview questions covering Terraform state management, remote backends, state locking, and workspaces.

**Topics Covered:**
- Terraform State Fundamentals
- Remote State Backends
- State Locking
- State Commands
- Workspaces
- State Security

---

## 🗄️ State Fundamentals

### Q1: What is Terraform state and why is it important?

**Answer:**

Terraform state is a JSON file that maps your Terraform configuration to real-world resources.

**Key Purposes:**
1. **Resource Tracking:** Maps config to actual infrastructure
2. **Metadata Storage:** Stores resource dependencies and relationships
3. **Performance:** Caches resource attributes to avoid API calls
4. **Collaboration:** Enables team workflows with locking

**State File Structure:**
```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "serial": 1,
  "lineage": "unique-identifier",
  "outputs": {},
  "resources": [
    {
      "mode": "managed",
      "type": "aws_instance",
      "name": "web",
      "instances": [{
        "attributes": {
          "id": "i-1234567890abcdef0",
          "ami": "ami-12345678"
        }
      }]
    }
  ]
}
```

**Why Important:**
- Without state, Terraform can't track what it manages
- Enables drift detection
- Supports resource dependencies
- Required for team collaboration

---

### Q2: What's the difference between local and remote state?

**Answer:**

| Aspect | Local State | Remote State |
|--------|-------------|--------------|
| **Location** | Local filesystem | Remote storage (S3, etc.) |
| **Collaboration** | ❌ Single user | ✅ Team access |
| **Locking** | ❌ No locking | ✅ State locking |
| **Versioning** | ❌ Manual backups | ✅ Automatic versioning |
| **Security** | ❌ Plain text | ✅ Encryption |
| **Backup** | ❌ Manual | ✅ Automatic |

**Local State Example:**
```hcl
# No backend configuration - uses local state
terraform {
  required_version = ">= 1.0"
}

# State stored in: terraform.tfstate
```

**Remote State Example:**
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "project/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**When to Use:**
- **Local:** Personal projects, learning, testing
- **Remote:** Production, team projects, CI/CD

---

### Q3: What information is stored in the state file?

**Answer:**

**State File Contents:**

1. **Resource Attributes:**
```json
{
  "resources": [{
    "type": "aws_instance",
    "name": "web",
    "instances": [{
      "attributes": {
        "id": "i-1234567890abcdef0",
        "ami": "ami-12345678",
        "instance_type": "t2.micro",
        "private_ip": "10.0.1.10",
        "public_ip": "54.123.45.67"
      }
    }]
  }]
}
```

2. **Dependencies:**
```json
{
  "dependencies": [
    "aws_vpc.main",
    "aws_subnet.public"
  ]
}
```

3. **Outputs:**
```json
{
  "outputs": {
    "instance_ip": {
      "value": "54.123.45.67",
      "type": "string"
    }
  }
}
```

4. **Metadata:**
- Terraform version
- Provider versions
- Serial number (increments with changes)
- Lineage (unique ID for state file)

**Security Concern:**
State files may contain sensitive data:
- Database passwords
- API keys
- Private IPs
- Resource IDs

**Solution:** Always encrypt state files!

---

## 🔐 Remote State Backends

### Q4: How do you configure S3 as a remote backend?

**Answer:**

**Step 1: Create S3 Bucket and DynamoDB Table**

```hcl
# S3 bucket for state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-${data.aws_caller_identity.current.account_id}"
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

# DynamoDB for locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

**Step 2: Configure Backend**

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-123456789012"
    key            = "project/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}
```

**Step 3: Initialize**

```bash
terraform init
```

---

### Q5: What is state locking and why is it important?

**Answer:**

State locking prevents concurrent operations that could corrupt the state file.

**How It Works:**

```
User A                    DynamoDB Lock Table
  │                              │
  ├─► terraform apply            │
  │   ├─► Acquire lock ─────────►│ Lock acquired
  │   │                          │ (LockID: xyz)
  │   ├─► Make changes           │
  │   └─► Release lock ──────────►│ Lock released
  │                              │
User B                           │
  │                              │
  ├─► terraform apply            │
  │   ├─► Acquire lock ─────────►│ ❌ Lock held by A
  │   │◄──── Error ──────────────┤ Wait or fail
```

**Without Locking:**
```
User A: terraform apply (starts)
User B: terraform apply (starts)
Both: Read same state
User A: Writes changes
User B: Writes changes (overwrites A's changes!)
Result: Corrupted state, lost changes
```

**With Locking:**
```
User A: terraform apply (acquires lock)
User B: terraform apply (waits for lock)
User A: Completes, releases lock
User B: Acquires lock, proceeds
Result: Safe, sequential operations
```

**DynamoDB Lock Entry:**
```json
{
  "LockID": "my-bucket/project/terraform.tfstate",
  "Info": "user@hostname",
  "Operation": "OperationTypeApply",
  "Who": "user@hostname",
  "Version": "1.5.0",
  "Created": "2024-01-15T10:30:00Z"
}
```

**Force Unlock (Emergency Only):**
```bash
terraform force-unlock <LOCK_ID>
```

---

### Q6: What are the benefits of using remote state?

**Answer:**

**1. Team Collaboration:**
- Multiple team members can work together
- Shared source of truth
- Prevents conflicts

**2. State Locking:**
- Prevents concurrent modifications
- Avoids state corruption
- Safe parallel workflows

**3. Versioning:**
- Automatic state history
- Easy rollback to previous versions
- Audit trail of changes

**4. Security:**
- Encryption at rest
- Encryption in transit
- Access control via IAM

**5. Backup and Recovery:**
- Automatic backups
- Point-in-time recovery
- Disaster recovery

**6. CI/CD Integration:**
- Automated deployments
- Consistent state access
- Pipeline integration

**Example Benefits in Practice:**

```hcl
# Before: Local state
# - Only one person can work
# - No backups
# - Secrets in plain text
# - Risk of loss

# After: Remote state
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true              # ✅ Encrypted
    dynamodb_table = "terraform-locks" # ✅ Locked
    # ✅ Versioned (S3 feature)
    # ✅ Team accessible
    # ✅ Backed up
  }
}
```

---

## 🛠️ State Commands

### Q7: What are the most important Terraform state commands?

**Answer:**

**1. terraform state list**
```bash
# List all resources
terraform state list

# Filter by type
terraform state list | grep aws_instance
```

**2. terraform state show**
```bash
# Show resource details
terraform state show aws_instance.web

# Output:
# resource "aws_instance" "web" {
#   id            = "i-1234567890abcdef0"
#   ami           = "ami-12345678"
#   instance_type = "t2.micro"
#   ...
# }
```

**3. terraform state mv**
```bash
# Rename resource
terraform state mv aws_instance.web aws_instance.web_server

# Move to module
terraform state mv aws_instance.web module.web.aws_instance.server
```

**4. terraform state rm**
```bash
# Remove from state (doesn't destroy)
terraform state rm aws_instance.web

# Resource still exists in AWS, just not tracked
```

**5. terraform state pull**
```bash
# Download state
terraform state pull > backup.tfstate
```

**6. terraform state push**
```bash
# Upload state (dangerous!)
terraform state push backup.tfstate
```

**7. terraform import**
```bash
# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0
```

---

### Q8: How do you import existing AWS resources into Terraform?

**Answer:**

**Step 1: Add Resource to Configuration**

```hcl
resource "aws_instance" "existing" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "existing-instance"
  }
}
```

**Step 2: Get Resource ID**

```bash
# Find instance ID
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=existing-instance" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text
```

**Step 3: Import**

```bash
terraform import aws_instance.existing i-1234567890abcdef0
```

**Step 4: Verify**

```bash
# Check state
terraform state show aws_instance.existing

# Plan should show no changes
terraform plan
```

**Common Resources to Import:**

```bash
# VPC
terraform import aws_vpc.main vpc-12345678

# Subnet
terraform import aws_subnet.public subnet-12345678

# Security Group
terraform import aws_security_group.web sg-12345678

# S3 Bucket
terraform import aws_s3_bucket.data my-bucket-name

# RDS Instance
terraform import aws_db_instance.main my-db-instance
```

**Bulk Import Script:**

```bash
#!/bin/bash
# import-instances.sh

# Get all instance IDs
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=production" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text)

# Import each instance
for ID in $INSTANCE_IDS; do
  echo "Importing $ID..."
  terraform import "aws_instance.prod[\"$ID\"]" $ID
done
```

---

### Q9: What's the difference between terraform state rm and terraform destroy?

**Answer:**

| Command | State | AWS Resource | Use Case |
|---------|-------|--------------|----------|
| **state rm** | Removed | ✅ Remains | Stop managing resource |
| **destroy** | Removed | ❌ Deleted | Delete resource |

**terraform state rm:**
```bash
# Remove from state only
terraform state rm aws_instance.web

# Resource still exists in AWS
# Terraform no longer manages it
# Next apply won't touch it
```

**terraform destroy:**
```bash
# Remove from state AND AWS
terraform destroy -target=aws_instance.web

# Resource deleted from AWS
# Also removed from state
```

**Example Scenario:**

```hcl
# You have an instance managed by Terraform
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# Scenario 1: Hand off to another team
# Use: terraform state rm aws_instance.web
# Result: Instance stays, you stop managing it

# Scenario 2: Decommission instance
# Use: terraform destroy -target=aws_instance.web
# Result: Instance deleted from AWS
```

---

## 🌍 Workspaces

### Q10: What are Terraform workspaces and when should you use them?

**Answer:**

Workspaces allow multiple state files for the same configuration.

**Structure:**
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

**Commands:**
```bash
# Create workspace
terraform workspace new dev

# List workspaces
terraform workspace list

# Switch workspace
terraform workspace select dev

# Show current
terraform workspace show

# Delete workspace
terraform workspace delete dev
```

**Using in Configuration:**
```hcl
locals {
  environment = terraform.workspace
  
  instance_counts = {
    dev     = 1
    staging = 2
    prod    = 5
  }
}

resource "aws_instance" "app" {
  count = local.instance_counts[local.environment]
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  tags = {
    Name        = "${local.environment}-app-${count.index}"
    Environment = local.environment
  }
}
```

**When to Use:**
- ✅ Same infrastructure, different environments
- ✅ Testing changes before production
- ✅ Simple environment separation

**When NOT to Use:**
- ❌ Different infrastructure per environment
- ❌ Different regions per environment
- ❌ Complex environment differences

**Alternative: Separate Directories**
```
terraform/
├── dev/
│   ├── main.tf
│   └── terraform.tfstate
├── staging/
│   ├── main.tf
│   └── terraform.tfstate
└── prod/
    ├── main.tf
    └── terraform.tfstate
```

---

### Q11: How do you handle sensitive data in state files?

**Answer:**

**Problem:** State files contain sensitive data in plain text.

**Solutions:**

**1. Encrypt State at Rest:**
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

**2. Encrypt in Transit:**
```hcl
terraform {
  backend "s3" {
    bucket  = "terraform-state"
    key     = "terraform.tfstate"
    region  = "us-east-1"
    encrypt = true  # Enables encryption
  }
}
```

**3. Use Secrets Manager:**
```hcl
# Don't store passwords in variables
# variable "db_password" {
#   type = string
# }

# Use Secrets Manager instead
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

**4. Restrict Access:**
```hcl
resource "aws_s3_bucket_policy" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = [
          "arn:aws:iam::123456789012:role/TerraformRole"
        ]
      }
      Action = [
        "s3:GetObject",
        "s3:PutObject"
      ]
      Resource = "${aws_s3_bucket.terraform_state.arn}/*"
    }]
  })
}
```

**5. Use sensitive Flag:**
```hcl
variable "db_password" {
  type      = string
  sensitive = true  # Won't show in logs
}

output "db_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = true  # Won't show in output
}
```

---

## 🚨 State Issues

### Q12: What causes state drift and how do you detect it?

**Answer:**

**State Drift:** When real infrastructure differs from Terraform state.

**Common Causes:**

1. **Manual Changes:**
```bash
# Someone modifies resource in AWS Console
aws ec2 modify-instance-attribute \
  --instance-id i-1234567890abcdef0 \
  --instance-type t2.small

# Terraform state still shows t2.micro
```

2. **External Tools:**
```bash
# CloudFormation, AWS CLI, or other tools modify resources
# Terraform doesn't know about changes
```

3. **Auto Scaling:**
```bash
# Auto Scaling Group creates/destroys instances
# Terraform state becomes outdated
```

**Detection:**

```bash
# Run plan to detect drift
terraform plan

# Output shows:
# ~ instance_type = "t2.micro" -> "t2.small"
```

**Resolution:**

**Option 1: Update State (Accept Changes)**
```bash
terraform refresh
# or
terraform apply -refresh-only
```

**Option 2: Revert Changes (Enforce Config)**
```bash
terraform apply
# Reverts instance back to t2.micro
```

**Prevention:**
- Use IAM policies to restrict manual changes
- Enable CloudTrail for audit
- Regular terraform plan checks
- Use drift detection tools

---

### Q13: How do you recover from a corrupted state file?

**Answer:**

**Scenario:** State file is corrupted or lost.

**Recovery Steps:**

**1. Restore from S3 Versioning:**
```bash
# List versions
aws s3api list-object-versions \
  --bucket my-terraform-state \
  --prefix terraform.tfstate

# Download previous version
aws s3api get-object \
  --bucket my-terraform-state \
  --key terraform.tfstate \
  --version-id <VERSION_ID> \
  restored-state.json

# Verify state
cat restored-state.json | jq

# Push restored state
terraform state push restored-state.json
```

**2. Restore from Backup:**
```bash
# If you have local backup
terraform state push terraform.tfstate.backup
```

**3. Rebuild State (Last Resort):**
```bash
# Import all resources manually
terraform import aws_vpc.main vpc-12345678
terraform import aws_subnet.public subnet-12345678
terraform import aws_instance.web i-1234567890abcdef0
# ... continue for all resources
```

**Prevention:**
- Enable S3 versioning
- Regular state backups
- Use remote state
- Enable state locking

---

### Q14: What happens if two people run terraform apply simultaneously?

**Answer:**

**Without State Locking:**
```
Time  User A                    User B
0s    terraform apply (start)   
1s    Read state (serial: 5)    terraform apply (start)
2s    Make changes              Read state (serial: 5)
3s    Write state (serial: 6)   Make changes
4s    Complete                  Write state (serial: 6) ❌
5s                              Complete

Result: User B overwrites User A's changes!
State corruption, lost changes
```

**With State Locking:**
```
Time  User A                    User B
0s    terraform apply           
1s    Acquire lock ✅           
2s    Read state                terraform apply
3s    Make changes              Try to acquire lock ❌
4s    Write state               Wait for lock...
5s    Release lock              Acquire lock ✅
6s    Complete                  Read state
7s                              Make changes
8s                              Write state
9s                              Release lock
10s                             Complete

Result: Safe, sequential operations
No corruption, all changes preserved
```

**Lock Error Message:**
```
Error: Error acquiring the state lock

Error message: ConditionalCheckFailedException: The conditional request failed
Lock Info:
  ID:        a1b2c3d4-e5f6-7890-abcd-ef1234567890
  Path:      my-bucket/terraform.tfstate
  Operation: OperationTypeApply
  Who:       user@hostname
  Version:   1.5.0
  Created:   2024-01-15 10:30:00
```

**Solution:**
```bash
# Wait for other user to finish
# Or force unlock (if lock is stuck)
terraform force-unlock a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

### Q15: How do you migrate from local state to remote state?

**Answer:**

**Step 1: Create Remote Backend Infrastructure**

```hcl
# backend-setup/main.tf
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state"
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
}
```

```bash
cd backend-setup
terraform init
terraform apply
```

**Step 2: Add Backend Configuration**

```hcl
# main.tf (your project)
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "project/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

**Step 3: Migrate State**

```bash
# Backup local state first
cp terraform.tfstate terraform.tfstate.backup

# Initialize with new backend
terraform init -migrate-state

# Terraform will ask:
# Do you want to copy existing state to the new backend?
# Answer: yes

# Verify migration
terraform state list

# Check S3
aws s3 ls s3://my-terraform-state/project/
```

**Step 4: Verify and Cleanup**

```bash
# Verify state works
terraform plan

# Should show no changes

# Remove local state files (after verification)
rm terraform.tfstate
rm terraform.tfstate.backup
```

---

## 📚 Summary

**Key Topics Covered:**
- State file purpose and structure
- Local vs remote state
- S3 backend configuration
- State locking with DynamoDB
- State commands (list, show, mv, rm, import)
- Workspaces for environment management
- State security and encryption
- State drift detection and recovery
- Concurrent operation handling

**Best Practices:**
1. Always use remote state for teams
2. Enable state locking
3. Encrypt state files
4. Enable versioning
5. Regular backups
6. Restrict access
7. Never edit state manually
8. Use workspaces wisely

**Next Steps:**
1. Practice labs in [LABS.md](./LABS.md)
2. Set up remote state for your projects
3. Proceed to [Chapter 4](../chapter-04/)

---

**💡 Pro Tips:**
- Use `terraform state pull` for backups
- Enable CloudTrail for state access audit
- Use separate state files per environment
- Implement state file lifecycle policies
- Monitor state file size
- Use `terraform refresh` carefully
