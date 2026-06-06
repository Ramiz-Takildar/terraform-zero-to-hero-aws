# Chapter 3: Labs - State Management

## 🎯 Lab Overview

These hands-on labs will help you master Terraform state management, remote backends, and workspaces.

**Prerequisites:**
- AWS account with appropriate permissions
- AWS CLI configured
- Terraform installed (v1.0+)
- Completed Chapters 1-2 labs

**Estimated Time:** 2-3 hours total

---

## 🧪 Lab 3.1: Configure Remote State with S3 and DynamoDB

**Objective:** Set up remote state backend with S3 and DynamoDB for state locking.

**Duration:** 45 minutes

### Architecture

```
┌─────────────────────────────────────────┐
│      Terraform Configuration            │
│  ┌──────────────────────────────────┐   │
│  │  Backend: S3 + DynamoDB          │   │
│  └────────────┬─────────────────────┘   │
└───────────────┼─────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│         S3 Bucket                       │
│  - Versioning enabled                   │
│  - Encryption enabled                   │
│  - Public access blocked                │
└─────────────────────────────────────────┘
                │
                ▼
┌─────────────────────────────────────────┐
│      DynamoDB Table                     │
│  - State locking                        │
│  - Pay-per-request billing              │
└─────────────────────────────────────────┘
```

### Step 1: Create Backend Infrastructure

```bash
mkdir -p lab-3.1-remote-state/backend-setup
cd lab-3.1-remote-state/backend-setup
```

Create `main.tf`:

```hcl
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
  region = var.aws_region
}

# Get current AWS account ID
data "aws_caller_identity" "current" {}

# S3 Bucket for Terraform state
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-${data.aws_caller_identity.current.account_id}"
  
  tags = {
    Name        = "Terraform State Bucket"
    Environment = "shared"
    ManagedBy   = "Terraform"
  }
}

# Enable versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Enable server-side encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Block all public access
resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Lifecycle policy to manage old versions
resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    id     = "expire-old-versions"
    status = "Enabled"
    
    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
  
  rule {
    id     = "delete-old-delete-markers"
    status = "Enabled"
    
    expiration {
      expired_object_delete_marker = true
    }
  }
}

# DynamoDB table for state locking
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
    ManagedBy   = "Terraform"
  }
}

# Outputs
output "s3_bucket_name" {
  description = "Name of the S3 bucket for Terraform state"
  value       = aws_s3_bucket.terraform_state.id
}

output "s3_bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.terraform_state.arn
}

output "dynamodb_table_name" {
  description = "Name of the DynamoDB table for state locking"
  value       = aws_dynamodb_table.terraform_locks.name
}

output "dynamodb_table_arn" {
  description = "ARN of the DynamoDB table"
  value       = aws_dynamodb_table.terraform_locks.arn
}

output "backend_config" {
  description = "Backend configuration to use in other projects"
  value = <<-EOT
    terraform {
      backend "s3" {
        bucket         = "${aws_s3_bucket.terraform_state.id}"
        key            = "path/to/terraform.tfstate"
        region         = "${var.aws_region}"
        encrypt        = true
        dynamodb_table = "${aws_dynamodb_table.terraform_locks.name}"
      }
    }
  EOT
}
```

Create `variables.tf`:

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}
```

### Step 2: Deploy Backend Infrastructure

```bash
# Initialize
terraform init

# Plan
terraform plan

# Apply
terraform apply -auto-approve

# Save outputs
terraform output -raw backend_config > ../backend-config.txt
terraform output -raw s3_bucket_name > ../bucket-name.txt
terraform output -raw dynamodb_table_name > ../table-name.txt
```

### Step 3: Create Project Using Remote State

```bash
cd ..
mkdir project
cd project
```

Create `main.tf`:

```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  # Remote backend configuration
  backend "s3" {
    bucket         = "terraform-state-123456789012"  # Replace with your bucket
    key            = "lab-3.1/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-state-locks"
  }
}

provider "aws" {
  region = "us-east-1"
}

# Simple VPC to test state
resource "aws_vpc" "test" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "test-vpc"
    Lab  = "3.1"
  }
}

output "vpc_id" {
  value = aws_vpc.test.id
}
```

### Step 4: Initialize with Remote Backend

```bash
# Initialize with backend
terraform init

# Verify backend configuration
cat .terraform/terraform.tfstate

# Apply configuration
terraform apply -auto-approve
```

### Step 5: Verify Remote State

```bash
# Check S3 bucket
aws s3 ls s3://$(cat ../bucket-name.txt)/lab-3.1/

# Download state file
aws s3 cp s3://$(cat ../bucket-name.txt)/lab-3.1/terraform.tfstate - | jq

# Check DynamoDB for locks (during apply)
aws dynamodb scan --table-name $(cat ../table-name.txt)
```

### Step 6: Test State Locking

Open two terminal windows:

**Terminal 1:**
```bash
# Start a long-running operation
terraform apply -auto-approve
```

**Terminal 2 (while Terminal 1 is running):**
```bash
# Try to run another operation
terraform plan

# You should see:
# Error: Error acquiring the state lock
```

### Step 7: Verify State Versioning

```bash
# Make a change
terraform apply -auto-approve

# List versions
aws s3api list-object-versions \
  --bucket $(cat ../bucket-name.txt) \
  --prefix lab-3.1/terraform.tfstate

# Download previous version
aws s3api get-object \
  --bucket $(cat ../bucket-name.txt) \
  --key lab-3.1/terraform.tfstate \
  --version-id <VERSION_ID> \
  previous-state.json
```

### Step 8: Cleanup

```bash
# Destroy project resources
terraform destroy -auto-approve

# Go back to backend-setup
cd ../backend-setup

# Destroy backend (after emptying bucket)
aws s3 rm s3://$(terraform output -raw s3_bucket_name) --recursive
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] S3 bucket created with versioning and encryption
- [ ] DynamoDB table created for locking
- [ ] Remote backend configured successfully
- [ ] State file stored in S3
- [ ] State locking works (tested with concurrent operations)
- [ ] State versioning verified

---

## 🧪 Lab 3.2: State Commands and Management

**Objective:** Master Terraform state commands for resource management.

**Duration:** 45 minutes

### Step 1: Create Test Infrastructure

```bash
mkdir -p lab-3.2-state-commands
cd lab-3.2-state-commands
```

Create `main.tf`:

```hcl
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

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "lab-vpc"
  }
}

# Subnets
resource "aws_subnet" "public_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-east-1a"
  
  tags = {
    Name = "public-subnet-1"
  }
}

resource "aws_subnet" "public_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = "us-east-1b"
  
  tags = {
    Name = "public-subnet-2"
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web server security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "web-sg"
  }
}

# EC2 Instance
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public_1.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  tags = {
    Name = "web-server"
  }
}
```

### Step 2: Deploy Infrastructure

```bash
terraform init
terraform apply -auto-approve
```

### Step 3: Practice State List

```bash
# List all resources
terraform state list

# Expected output:
# aws_instance.web
# aws_security_group.web
# aws_subnet.public_1
# aws_subnet.public_2
# aws_vpc.main
# data.aws_ami.amazon_linux_2

# Filter by type
terraform state list | grep aws_subnet

# Count resources
terraform state list | wc -l
```

### Step 4: Practice State Show

```bash
# Show VPC details
terraform state show aws_vpc.main

# Show instance details
terraform state show aws_instance.web

# Show in JSON format
terraform show -json | jq '.values.root_module.resources[] | select(.address == "aws_instance.web")'
```

### Step 5: Practice State Move

```bash
# Rename instance
terraform state mv aws_instance.web aws_instance.web_server

# Verify
terraform state list | grep instance

# Move back
terraform state mv aws_instance.web_server aws_instance.web
```

### Step 6: Practice State Remove

```bash
# Remove subnet from state (doesn't destroy)
terraform state rm aws_subnet.public_2

# Verify it's gone from state
terraform state list | grep subnet

# Check in AWS (still exists)
aws ec2 describe-subnets --filters "Name=tag:Name,Values=public-subnet-2"

# Import it back
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=public-subnet-2" --query 'Subnets[0].SubnetId' --output text)
terraform import aws_subnet.public_2 $SUBNET_ID

# Verify
terraform state list | grep subnet
```

### Step 7: Practice State Pull/Push

```bash
# Pull state to local file
terraform state pull > state-backup.json

# View state
cat state-backup.json | jq '.resources[].type' | sort | uniq

# Modify state (example: change a tag)
cat state-backup.json | jq '.resources[] | select(.type == "aws_vpc") | .instances[0].attributes.tags.Modified = "true"' > modified-state.json

# Note: Don't actually push modified state in production!
# This is just for learning
```

### Step 8: Practice Import

```bash
# Create a resource manually in AWS
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.1.0.0/16 --query 'Vpc.VpcId' --output text)
aws ec2 create-tags --resources $VPC_ID --tags Key=Name,Value=manual-vpc

# Add to configuration
cat >> main.tf <<EOF

resource "aws_vpc" "imported" {
  cidr_block = "10.1.0.0/16"
  
  tags = {
    Name = "manual-vpc"
  }
}
EOF

# Import
terraform import aws_vpc.imported $VPC_ID

# Verify
terraform state show aws_vpc.imported

# Plan should show no changes
terraform plan
```

### Step 9: Practice State Replace

```bash
# Force replacement of instance
terraform apply -replace=aws_instance.web -auto-approve

# This destroys and recreates the instance
```

### Step 10: Cleanup

```bash
# Delete manually created VPC
aws ec2 delete-vpc --vpc-id $VPC_ID

# Destroy all
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] Listed all resources in state
- [ ] Showed detailed resource information
- [ ] Renamed resources using state mv
- [ ] Removed and re-imported resources
- [ ] Pulled state to local file
- [ ] Imported existing AWS resources
- [ ] Forced resource replacement

---

## 🧪 Lab 3.3: Workspaces for Environment Management

**Objective:** Use Terraform workspaces to manage multiple environments.

**Duration:** 45 minutes

### Architecture

```
┌─────────────────────────────────────────┐
│         Terraform Configuration         │
│                                         │
│  Workspace: dev                         │
│  ├─ 1 t2.micro instance                │
│  └─ Small RDS instance                  │
│                                         │
│  Workspace: staging                     │
│  ├─ 2 t2.small instances                │
│  └─ Medium RDS instance                 │
│                                         │
│  Workspace: prod                        │
│  ├─ 5 t2.medium instances               │
│  └─ Large RDS instance                  │
└─────────────────────────────────────────┘
```

### Step 1: Create Project Structure

```bash
mkdir -p lab-3.3-workspaces
cd lab-3.3-workspaces
```

Create `main.tf`:

```hcl
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
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = terraform.workspace
      ManagedBy   = "Terraform"
      Workspace   = terraform.workspace
    }
  }
}

# Environment-specific configuration
locals {
  environment = terraform.workspace
  
  # Instance counts per environment
  instance_counts = {
    dev     = 1
    staging = 2
    prod    = 5
  }
  
  # Instance types per environment
  instance_types = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t2.medium"
  }
  
  # RDS instance classes
  db_instance_classes = {
    dev     = "db.t3.micro"
    staging = "db.t3.small"
    prod    = "db.t3.medium"
  }
  
  # Enable multi-AZ for production
  db_multi_az = {
    dev     = false
    staging = false
    prod    = true
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  
  tags = {
    Name = "${local.environment}-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${local.environment}-igw"
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${local.environment}-public-subnet"
  }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "${local.environment}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${local.environment}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${local.environment}-web-sg"
  }
}

# EC2 Instances (count varies by environment)
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "app" {
  count = local.instance_counts[local.environment]
  
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = local.instance_types[local.environment]
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>${local.environment} - Instance ${count.index + 1}</h1>" > /var/www/html/index.html
              EOF
  
  tags = {
    Name = "${local.environment}-app-${count.index + 1}"
  }
}
```

Create `variables.tf`:

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}
```

Create `outputs.tf`:

```hcl
output "environment" {
  description = "Current workspace/environment"
  value       = terraform.workspace
}

output "instance_count" {
  description = "Number of instances deployed"
  value       = length(aws_instance.app)
}

output "instance_type" {
  description = "Instance type used"
  value       = local.instance_types[local.environment]
}

output "instance_ips" {
  description = "Public IPs of instances"
  value       = aws_instance.app[*].public_ip
}

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}
```

### Step 2: Create Workspaces

```bash
# Initialize
terraform init

# List workspaces (only 'default' exists)
terraform workspace list

# Create dev workspace
terraform workspace new dev

# Create staging workspace
terraform workspace new staging

# Create prod workspace
terraform workspace new prod

# List all workspaces
terraform workspace list
```

### Step 3: Deploy to Dev

```bash
# Switch to dev
terraform workspace select dev

# Verify current workspace
terraform workspace show

# Deploy
terraform apply -auto-approve

# Check outputs
terraform output
```

### Step 4: Deploy to Staging

```bash
# Switch to staging
terraform workspace select staging

# Deploy
terraform apply -auto-approve

# Check outputs (should show 2 instances)
terraform output
```

### Step 5: Deploy to Prod

```bash
# Switch to prod
terraform workspace select prod

# Deploy
terraform apply -auto-approve

# Check outputs (should show 5 instances)
terraform output
```

### Step 6: Compare Environments

```bash
# Dev
terraform workspace select dev
terraform output instance_count
terraform output instance_type

# Staging
terraform workspace select staging
terraform output instance_count
terraform output instance_type

# Prod
terraform workspace select prod
terraform output instance_count
terraform output instance_type
```

### Step 7: Verify in AWS Console

1. Go to EC2 Dashboard
2. Filter by tags: `Workspace = dev`
3. Verify 1 t2.micro instance
4. Filter by tags: `Workspace = staging`
5. Verify 2 t2.small instances
6. Filter by tags: `Workspace = prod`
7. Verify 5 t2.medium instances

### Step 8: Test Workspace Isolation

```bash
# Destroy dev environment
terraform workspace select dev
terraform destroy -auto-approve

# Verify staging and prod still exist
terraform workspace select staging
terraform state list

terraform workspace select prod
terraform state list
```

### Step 9: Cleanup All Environments

```bash
# Destroy staging
terraform workspace select staging
terraform destroy -auto-approve

# Destroy prod
terraform workspace select prod
terraform destroy -auto-approve

# Switch to default
terraform workspace select default

# Delete workspaces
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod
```

### ✅ Success Criteria

- [ ] Created multiple workspaces (dev, staging, prod)
- [ ] Deployed different configurations per workspace
- [ ] Verified instance counts match environment
- [ ] Verified instance types match environment
- [ ] Confirmed workspace isolation
- [ ] Successfully destroyed environments independently

---

## 🎓 Lab Summary

### What You Learned

1. **Lab 3.1:** Remote state with S3 and DynamoDB
2. **Lab 3.2:** State commands for resource management
3. **Lab 3.3:** Workspaces for environment separation

### Key Concepts Practiced

- S3 backend configuration
- State locking with DynamoDB
- State versioning and recovery
- State manipulation commands
- Resource import and export
- Workspace management
- Environment-specific configurations

### Next Steps

1. ✅ Mark labs complete in [CHECKLIST.md](../CHECKLIST.md)
2. 📝 Review [INTERVIEW.md](./INTERVIEW.md) questions
3. 🚀 Proceed to [Chapter 4](../chapter-04/)

---

## 💡 Troubleshooting Tips

### State Lock Issues

```bash
# If lock is stuck
terraform force-unlock <LOCK_ID>

# Find lock ID
aws dynamodb scan --table-name terraform-state-locks
```

### State Corruption

```bash
# Restore from S3 version
aws s3api list-object-versions --bucket <bucket> --prefix <key>
aws s3api get-object --bucket <bucket> --key <key> --version-id <id> restored.tfstate
terraform state push restored.tfstate
```

### Import Errors

```bash
# Get resource ID first
aws ec2 describe-instances --filters "Name=tag:Name,Values=my-instance"

# Then import
terraform import aws_instance.web i-1234567890abcdef0
```

---

**🎉 Congratulations!** You've mastered Terraform state management!
