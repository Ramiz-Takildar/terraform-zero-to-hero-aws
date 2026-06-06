# Chapter 10: Labs - Best Practices and Production-Ready Terraform

## 🎯 Lab Overview

These hands-on labs will help you implement production-ready Terraform infrastructure with security, reliability, and operational excellence.

**Prerequisites:**
- AWS account with appropriate permissions
- Terraform installed (v1.0+)
- Completed Chapters 1-9 labs
- Understanding of AWS services

**Estimated Time:** 3-4 hours total

---

## 🧪 Lab 10.1: Production-Ready Infrastructure Setup

**Objective:** Build a complete production-ready infrastructure with security, monitoring, and disaster recovery.

**Duration:** 120 minutes

### Step 1: Create Project Structure

```bash
mkdir -p terraform-production
cd terraform-production

# Create directory structure
mkdir -p {modules/{networking,compute,database,monitoring},environments/{dev,staging,prod},scripts,docs}
```

### Step 2: Create Backend Infrastructure

**scripts/setup-backend.sh:**

```bash
#!/bin/bash

set -e

PROJECT_NAME="prod-infra"
REGION="us-east-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "Setting up Terraform backend infrastructure..."

# Create KMS key for encryption
KMS_KEY_ID=$(aws kms create-key \
  --description "Terraform state encryption key" \
  --query 'KeyMetadata.KeyId' \
  --output text)

aws kms create-alias \
  --alias-name alias/${PROJECT_NAME}-terraform-state \
  --target-key-id ${KMS_KEY_ID}

echo "KMS Key created: ${KMS_KEY_ID}"

# Create primary state bucket
PRIMARY_BUCKET="${PROJECT_NAME}-terraform-state-${ACCOUNT_ID}"
aws s3api create-bucket \
  --bucket ${PRIMARY_BUCKET} \
  --region ${REGION}

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket ${PRIMARY_BUCKET} \
  --versioning-configuration Status=Enabled

# Enable encryption
aws s3api put-bucket-encryption \
  --bucket ${PRIMARY_BUCKET} \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "'${KMS_KEY_ID}'"
      }
    }]
  }'

# Block public access
aws s3api put-public-access-block \
  --bucket ${PRIMARY_BUCKET} \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Enable logging
LOGS_BUCKET="${PROJECT_NAME}-logs-${ACCOUNT_ID}"
aws s3api create-bucket \
  --bucket ${LOGS_BUCKET} \
  --region ${REGION}

aws s3api put-bucket-logging \
  --bucket ${PRIMARY_BUCKET} \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "'${LOGS_BUCKET}'",
      "TargetPrefix": "terraform-state/"
    }
  }'

# Create backup bucket
BACKUP_BUCKET="${PROJECT_NAME}-terraform-backups-${ACCOUNT_ID}"
aws s3api create-bucket \
  --bucket ${BACKUP_BUCKET} \
  --region ${REGION}

# Create DynamoDB table for locking
aws dynamodb create-table \
  --table-name ${PROJECT_NAME}-terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ${REGION} \
  --tags Key=Name,Value=${PROJECT_NAME}-terraform-locks

# Enable point-in-time recovery
aws dynamodb update-continuous-backups \
  --table-name ${PROJECT_NAME}-terraform-locks \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true

echo ""
echo "Backend infrastructure created successfully!"
echo "Primary Bucket: ${PRIMARY_BUCKET}"
echo "Backup Bucket: ${BACKUP_BUCKET}"
echo "Logs Bucket: ${LOGS_BUCKET}"
echo "DynamoDB Table: ${PROJECT_NAME}-terraform-locks"
echo "KMS Key: ${KMS_KEY_ID}"
```

```bash
chmod +x scripts/setup-backend.sh
./scripts/setup-backend.sh
```

### Step 3: Create Networking Module

**modules/networking/main.tf:**

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

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-vpc"
    }
  )
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-igw"
    }
  )
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-public-${count.index + 1}"
      Type = "public"
      Tier = "public"
    }
  )
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-private-${count.index + 1}"
      Type = "private"
      Tier = "application"
    }
  )
}

# Database Subnets
resource "aws_subnet" "database" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 20)
  availability_zone = var.availability_zones[count.index]
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-database-${count.index + 1}"
      Type = "private"
      Tier = "database"
    }
  )
}

# NAT Gateways
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? length(var.availability_zones) : 0
  domain = "vpc"
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-nat-eip-${count.index + 1}"
    }
  )
  
  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? length(var.availability_zones) : 0
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-nat-${count.index + 1}"
    }
  )
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-public-rt"
    }
  )
}

resource "aws_route_table" "private" {
  count = length(var.availability_zones)
  
  vpc_id = aws_vpc.main.id
  
  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.main[count.index].id
    }
  }
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-private-rt-${count.index + 1}"
    }
  )
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(var.availability_zones)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private" {
  count = length(var.availability_zones)
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
  
  tags = var.common_tags
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/${var.name_prefix}"
  retention_in_days = var.log_retention_days
  
  tags = var.common_tags
}

resource "aws_iam_role" "flow_logs" {
  name = "${var.name_prefix}-vpc-flow-logs"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
      }
    ]
  })
  
  tags = var.common_tags
}

resource "aws_iam_role_policy" "flow_logs" {
  name = "${var.name_prefix}-vpc-flow-logs"
  role = aws_iam_role.flow_logs.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}

# Network ACLs
resource "aws_network_acl" "database" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.database[*].id
  
  # Allow inbound from application tier
  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = var.vpc_cidr
    from_port  = 3306
    to_port    = 3306
  }
  
  ingress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = var.vpc_cidr
    from_port  = 5432
    to_port    = 5432
  }
  
  # Allow return traffic
  ingress {
    protocol   = "tcp"
    rule_no    = 120
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }
  
  # Allow outbound to VPC
  egress {
    protocol   = -1
    rule_no    = 100
    action     = "allow"
    cidr_block = var.vpc_cidr
    from_port  = 0
    to_port    = 0
  }
  
  tags = merge(
    var.common_tags,
    {
      Name = "${var.name_prefix}-database-nacl"
    }
  )
}
```

**modules/networking/variables.tf:**

```hcl
variable "name_prefix" {
  description = "Name prefix for resources"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Enable NAT gateway"
  type        = bool
  default     = true
}

variable "log_retention_days" {
  description = "CloudWatch log retention in days"
  type        = number
  default     = 30
}

variable "common_tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default     = {}
}
```

**modules/networking/outputs.tf:**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "database_subnet_ids" {
  description = "Database subnet IDs"
  value       = aws_subnet.database[*].id
}

output "nat_gateway_ips" {
  description = "NAT Gateway public IPs"
  value       = aws_eip.nat[*].public_ip
}
```

### Step 4: Create Production Environment

**environments/prod/main.tf:**

```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket         = "prod-infra-terraform-state-ACCOUNT_ID"  # Update with your account ID
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "KMS_KEY_ID"  # Update with your KMS key
    dynamodb_table = "prod-infra-terraform-locks"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = local.common_tags
  }
}

# Local variables
locals {
  environment = "production"
  
  common_tags = {
    Environment = local.environment
    ManagedBy   = "Terraform"
    Project     = var.project_name
    Owner       = var.owner
    CostCenter  = var.cost_center
    Compliance  = "PCI-DSS"
  }
}

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

# Networking
module "networking" {
  source = "../../modules/networking"
  
  name_prefix        = "${var.project_name}-${local.environment}"
  vpc_cidr           = var.vpc_cidr
  availability_zones = slice(data.aws_availability_zones.available.names, 0, 3)
  enable_nat_gateway = true
  log_retention_days = 90
  common_tags        = local.common_tags
}

# Security Groups
resource "aws_security_group" "alb" {
  name        = "${var.project_name}-alb-sg"
  description = "Security group for Application Load Balancer"
  vpc_id      = module.networking.vpc_id
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS from internet"
  }
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP from internet (redirect to HTTPS)"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-alb-sg"
    }
  )
}

resource "aws_security_group" "application" {
  name        = "${var.project_name}-app-sg"
  description = "Security group for application servers"
  vpc_id      = module.networking.vpc_id
  
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "Application port from ALB"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-app-sg"
    }
  )
}

resource "aws_security_group" "database" {
  name        = "${var.project_name}-db-sg"
  description = "Security group for database"
  vpc_id      = module.networking.vpc_id
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.application.id]
    description     = "PostgreSQL from application"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [module.networking.vpc_cidr]
    description = "Internal VPC only"
  }
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-db-sg"
    }
  )
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "nat_gateway_packets_drop" {
  count = length(module.networking.nat_gateway_ips)
  
  alarm_name          = "${var.project_name}-nat-packets-drop-${count.index + 1}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "PacketsDropCount"
  namespace           = "AWS/NATGateway"
  period              = 300
  statistic           = "Sum"
  threshold           = 1000
  alarm_description   = "NAT Gateway dropping packets"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  tags = local.common_tags
}

# SNS Topic for Alerts
resource "aws_sns_topic" "alerts" {
  name              = "${var.project_name}-alerts"
  kms_master_key_id = aws_kms_key.sns.id
  
  tags = local.common_tags
}

resource "aws_sns_topic_subscription" "alerts_email" {
  topic_arn = aws_sns_topic.alerts.arn
  protocol  = "email"
  endpoint  = var.alert_email
}

# KMS Keys
resource "aws_kms_key" "sns" {
  description             = "KMS key for SNS encryption"
  deletion_window_in_days = 30
  enable_key_rotation     = true
  
  tags = local.common_tags
}

resource "aws_kms_alias" "sns" {
  name          = "alias/${var.project_name}-sns"
  target_key_id = aws_kms_key.sns.key_id
}
```

**environments/prod/variables.tf:**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "prod-infra"
}

variable "owner" {
  description = "Infrastructure owner"
  type        = string
}

variable "cost_center" {
  description = "Cost center"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "alert_email" {
  description = "Email for alerts"
  type        = string
}
```

**environments/prod/outputs.tf:**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = module.networking.vpc_id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = module.networking.public_subnet_ids
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = module.networking.private_subnet_ids
}

output "database_subnet_ids" {
  description = "Database subnet IDs"
  value       = module.networking.database_subnet_ids
}

output "nat_gateway_ips" {
  description = "NAT Gateway IPs"
  value       = module.networking.nat_gateway_ips
}
```

**environments/prod/terraform.tfvars:**

```hcl
project_name = "prod-infra"
owner        = "infrastructure-team"
cost_center  = "engineering"
vpc_cidr     = "10.0.0.0/16"
alert_email  = "alerts@example.com"
```

### Step 5: Create Backup Script

**scripts/backup-state.sh:**

```bash
#!/bin/bash

set -e

PROJECT_NAME="prod-infra"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
PRIMARY_BUCKET="${PROJECT_NAME}-terraform-state-${ACCOUNT_ID}"
BACKUP_BUCKET="${PROJECT_NAME}-terraform-backups-${ACCOUNT_ID}"
DATE=$(date +%Y%m%d-%H%M%S)

echo "Backing up Terraform state..."

# Create backup directory
BACKUP_DIR="${BACKUP_BUCKET}/${DATE}"

# Copy all state files
aws s3 sync s3://${PRIMARY_BUCKET}/ s3://${BACKUP_DIR}/ \
  --exclude "*" \
  --include "*.tfstate"

echo "Backup completed: s3://${BACKUP_DIR}/"

# List recent backups
echo ""
echo "Recent backups:"
aws s3 ls s3://${BACKUP_BUCKET}/ | tail -5

# Cleanup old backups (keep last 30 days)
echo ""
echo "Cleaning up old backups..."
aws s3 ls s3://${BACKUP_BUCKET}/ | \
  awk '{print $2}' | \
  sort -r | \
  tail -n +31 | \
  while read backup; do
    echo "Deleting: ${backup}"
    aws s3 rm s3://${BACKUP_BUCKET}/${backup} --recursive
  done

echo "Backup process completed!"
```

```bash
chmod +x scripts/backup-state.sh
```

### Step 6: Deploy Infrastructure

```bash
cd environments/prod

# Initialize
terraform init

# Validate
terraform validate

# Plan
terraform plan -out=prod.tfplan

# Review plan carefully
terraform show prod.tfplan

# Apply
terraform apply prod.tfplan

# Save outputs
terraform output -json > outputs.json
```

### Step 7: Test Backup and Recovery

```bash
# Create backup
cd ../../scripts
./backup-state.sh

# Simulate state corruption (DO NOT DO IN REAL PRODUCTION!)
# aws s3 rm s3://prod-infra-terraform-state-ACCOUNT_ID/prod/terraform.tfstate

# Restore from backup
LATEST_BACKUP=$(aws s3 ls s3://prod-infra-terraform-backups-ACCOUNT_ID/ | tail -1 | awk '{print $2}')
aws s3 sync s3://prod-infra-terraform-backups-ACCOUNT_ID/${LATEST_BACKUP}/ \
  s3://prod-infra-terraform-state-ACCOUNT_ID/

# Verify restoration
cd ../environments/prod
terraform init
terraform plan
```

### Step 8: Cleanup

```bash
cd environments/prod
terraform destroy -auto-approve

# Cleanup backend (optional)
cd ../../scripts
# Create cleanup script if needed
```

### ✅ Success Criteria

- [ ] Backend infrastructure created with encryption
- [ ] VPC with public, private, and database subnets
- [ ] NAT gateways for high availability
- [ ] VPC Flow Logs enabled
- [ ] Security groups with least privilege
- [ ] CloudWatch alarms configured
- [ ] SNS notifications working
- [ ] Backup script functional
- [ ] State recovery tested
- [ ] All resources properly tagged

---

## 🧪 Lab 10.2: Security Hardening

**Objective:** Implement comprehensive security controls for production infrastructure.

**Duration:** 60 minutes

### Step 1: Create Security Module

**modules/security/main.tf:**

```hcl
# KMS Keys
resource "aws_kms_key" "main" {
  for_each = var.kms_keys
  
  description             = each.value.description
  deletion_window_in_days = each.value.deletion_window
  enable_key_rotation     = true
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow services to use the key"
        Effect = "Allow"
        Principal = {
          Service = each.value.services
        }
        Action = [
          "kms:Decrypt",
          "kms:GenerateDataKey"
        ]
        Resource = "*"
      }
    ]
  })
  
  tags = var.common_tags
}

resource "aws_kms_alias" "main" {
  for_each = var.kms_keys
  
  name          = "alias/${var.name_prefix}-${each.key}"
  target_key_id = aws_kms_key.main[each.key].key_id
}

# Secrets Manager
resource "aws_secretsmanager_secret" "main" {
  for_each = var.secrets
  
  name                    = "${var.name_prefix}-${each.key}"
  description             = each.value.description
  kms_key_id              = aws_kms_key.main["secrets"].id
  recovery_window_in_days = 30
  
  tags = var.common_tags
}

resource "aws_secretsmanager_secret_version" "main" {
  for_each = var.secrets
  
  secret_id     = aws_secretsmanager_secret.main[each.key].id
  secret_string = jsonencode(each.value.value)
}

# IAM Password Policy
resource "aws_iam_account_password_policy" "strict" {
  minimum_password_length        = 14
  require_lowercase_characters   = true
  require_uppercase_characters   = true
  require_numbers                = true
  require_symbols                = true
  allow_users_to_change_password = true
  max_password_age               = 90
  password_reuse_prevention      = 24
}

# GuardDuty
resource "aws_guardduty_detector" "main" {
  enable = true
  
  datasources {
    s3_logs {
      enable = true
    }
    kubernetes {
      audit_logs {
        enable = true
      }
    }
  }
  
  tags = var.common_tags
}

# Security Hub
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:${data.aws_region.current.name}::standards/cis-aws-foundations-benchmark/v/1.4.0"
  
  depends_on = [aws_securityhub_account.main]
}

# Config
resource "aws_config_configuration_recorder" "main" {
  name     = "${var.name_prefix}-config-recorder"
  role_arn = aws_iam_role.config.arn
  
  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

resource "aws_config_delivery_channel" "main" {
  name           = "${var.name_prefix}-config-delivery"
  s3_bucket_name = var.config_bucket_name
  
  depends_on = [aws_config_configuration_recorder.main]
}

resource "aws_config_configuration_recorder_status" "main" {
  name       = aws_config_configuration_recorder.main.name
  is_enabled = true
  
  depends_on = [aws_config_delivery_channel.main]
}

# IAM Role for Config
resource "aws_iam_role" "config" {
  name = "${var.name_prefix}-config-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "config.amazonaws.com"
        }
      }
    ]
  })
  
  tags = var.common_tags
}

resource "aws_iam_role_policy_attachment" "config" {
  role       = aws_iam_role.config.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/ConfigRole"
}

# Data sources
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
```

**modules/security/variables.tf:**

```hcl
variable "name_prefix" {
  description = "Name prefix for resources"
  type        = string
}

variable "kms_keys" {
  description = "KMS keys to create"
  type = map(object({
    description      = string
    deletion_window  = number
    services         = list(string)
  }))
}

variable "secrets" {
  description = "Secrets to store"
  type = map(object({
    description = string
    value       = map(string)
  }))
  sensitive = true
}

variable "config_bucket_name" {
  description = "S3 bucket for AWS Config"
  type        = string
}

variable "common_tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}
```

### Step 2: Implement Security Controls

```bash
cd environments/prod

# Add security module
cat >> main.tf <<'EOF'

# Security Module
module "security" {
  source = "../../modules/security"
  
  name_prefix = "${var.project_name}-${local.environment}"
  
  kms_keys = {
    secrets = {
      description     = "KMS key for Secrets Manager"
      deletion_window = 30
      services        = ["secretsmanager.amazonaws.com"]
    }
    rds = {
      description     = "KMS key for RDS encryption"
      deletion_window = 30
      services        = ["rds.amazonaws.com"]
    }
    s3 = {
      description     = "KMS key for S3 encryption"
      deletion_window = 30
      services        = ["s3.amazonaws.com"]
    }
  }
  
  secrets = {
    database = {
      description = "Database credentials"
      value = {
        username = "admin"
        password = random_password.db_password.result
      }
    }
  }
  
  config_bucket_name = aws_s3_bucket.config.id
  common_tags        = local.common_tags
}

# Random password for database
resource "random_password" "db_password" {
  length  = 32
  special = true
}

# S3 bucket for AWS Config
resource "aws_s3_bucket" "config" {
  bucket = "${var.project_name}-config-${data.aws_caller_identity.current.account_id}"
  
  tags = local.common_tags
}

resource "aws_s3_bucket_versioning" "config" {
  bucket = aws_s3_bucket.config.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

data "aws_caller_identity" "current" {}
EOF

# Apply changes
terraform init
terraform plan
terraform apply -auto-approve
```

### Step 3: Verify Security Controls

```bash
# Check GuardDuty
aws guardduty list-detectors

# Check Security Hub
aws securityhub describe-hub

# Check AWS Config
aws configservice describe-configuration-recorders

# Check KMS keys
aws kms list-keys

# Check secrets
aws secretsmanager list-secrets
```

### ✅ Success Criteria

- [ ] KMS keys created with rotation enabled
- [ ] Secrets stored in Secrets Manager
- [ ] IAM password policy enforced
- [ ] GuardDuty enabled
- [ ] Security Hub enabled
- [ ] AWS Config recording
- [ ] All encryption at rest enabled

---

## 🧪 Lab 10.3: Disaster Recovery Testing

**Objective:** Test disaster recovery procedures and validate backup/restore processes.

**Duration:** 60 minutes

### Step 1: Create DR Script

**scripts/disaster-recovery.sh:**

```bash
#!/bin/bash

set -e

PROJECT_NAME="prod-infra"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
PRIMARY_BUCKET="${PROJECT_NAME}-terraform-state-${ACCOUNT_ID}"
BACKUP_BUCKET="${PROJECT_NAME}-terraform-backups-${ACCOUNT_ID}"

echo "=== Disaster Recovery Simulation ==="
echo ""

# Function to list backups
list_backups() {
    echo "Available backups:"
    aws s3 ls s3://${BACKUP_BUCKET}/ | nl
}

# Function to restore from backup
restore_backup() {
    local backup_date=$1
    
    echo "Restoring from backup: ${backup_date}"
    
    # Create temporary directory
    TEMP_DIR=$(mktemp -d)
    
    # Download backup
    aws s3 sync s3://${BACKUP_BUCKET}/${backup_date}/ ${TEMP_DIR}/
    
    # Verify backup integrity
    if [ ! -f "${TEMP_DIR}/prod/terraform.tfstate" ]; then
        echo "Error: Backup is incomplete or corrupted"
        rm -rf ${TEMP_DIR}
        exit 1
    fi
    
    # Restore to primary bucket
    aws s3 sync ${TEMP_DIR}/ s3://${PRIMARY_BUCKET}/
    
    # Cleanup
    rm -rf ${TEMP_DIR}
    
    echo "Restore completed successfully"
}

# Function to verify infrastructure
verify_infrastructure() {
    echo "Verifying infrastructure state..."
    
    cd ../environments/prod
    
    # Initialize
    terraform init -reconfigure
    
    # Refresh state
    terraform refresh
    
    # Validate
    terraform validate
    
    # Plan (should show no changes)
    terraform plan -detailed-exitcode
    
    if [ $? -eq 0 ]; then
        echo "✓ Infrastructure state is consistent"
    else
        echo "✗ Infrastructure state has drift"
        return 1
    fi
}

# Main menu
echo "1. List available backups"
echo "2. Restore from backup"
echo "3. Verify infrastructure"
echo "4. Full DR test"
echo ""
read -p "Select option: " option

case $option in
    1)
        list_backups
        ;;
    2)
        list_backups
        read -p "Enter backup date (YYYYMMDD-HHMMSS): " backup_date
        restore_backup $backup_date
        ;;
    3)
        verify_infrastructure
        ;;
    4)
        echo "Running full DR test..."
        list_backups
        read -p "Enter backup date for test: " backup_date
        restore_backup $backup_date
        verify_infrastructure
        echo "DR test completed"
        ;;
    *)
        echo "Invalid option"
        exit 1
        ;;
esac
```

```bash
chmod +x scripts/disaster-recovery.sh
```

### Step 2: Test Backup Process

```bash
# Create initial backup
cd scripts
./backup-state.sh

# Verify backup exists
aws s3 ls s3://prod-infra-terraform-backups-ACCOUNT_ID/
```

### Step 3: Simulate Disaster

```bash
# DO NOT RUN IN REAL PRODUCTION!
# This is for testing only

# Create a test state backup first
cd ../environments/prod
terraform state pull > state-backup.json

# Make a change to test recovery
terraform taint aws_security_group.alb
terraform apply -auto-approve
```

### Step 4: Test Recovery

```bash
# Run DR script
cd ../../scripts
./disaster-recovery.sh

# Select option 4 (Full DR test)
# Follow prompts to restore and verify
```

### Step 5: Document RTO/RPO

**docs/disaster-recovery.md:**

```markdown
# Disaster Recovery Plan

## Objectives

- **RTO (Recovery Time Objective):** 4 hours
- **RPO (Recovery Point Objective):** 1 hour

## Backup Schedule

- **Frequency:** Hourly automated backups
- **Retention:** 30 days
- **Location:** S3 with cross-region replication

## Recovery Procedures

### State File Recovery

1. Identify latest valid backup
2. Run disaster-recovery.sh script
3. Select restore option
4. Verify infrastructure state
5. Communicate status to team

### Infrastructure Recovery

1. Restore state file
2. Run terraform plan
3. Apply any necessary changes
4. Verify all services
5. Update monitoring

## Testing Schedule

- **Monthly:** DR drill
- **Quarterly:** Full failover test
- **Annually:** Multi-region failover

## Contact Information

- **Primary:** infrastructure-team@example.com
- **Escalation:** cto@example.com
- **On-call:** +1-555-0123
```

### ✅ Success Criteria

- [ ] Backup script runs successfully
- [ ] Backups stored in S3
- [ ] DR script functional
- [ ] State restoration works
- [ ] Infrastructure verification passes
- [ ] RTO/RPO documented
- [ ] DR procedures documented

---

## 🎓 Lab Summary

### What You Learned

1. **Lab 10.1:** Production-ready infrastructure with security and monitoring
2. **Lab 10.2:** Security hardening with KMS, Secrets Manager, and compliance tools
3. **Lab 10.3:** Disaster recovery testing and procedures

### Key Concepts Practiced

- Production infrastructure design
- Security best practices
- State management and backup
- Disaster recovery procedures
- Monitoring and alerting
- Compliance controls
- Documentation standards

### Best Practices Learned

1. Always encrypt state files
2. Use remote backends with locking
3. Implement comprehensive tagging
4. Enable VPC Flow Logs
5. Use least privilege IAM
6. Regular backup testing
7. Document DR procedures
8. Monitor and alert on key metrics

### Next Steps

1. ✅ Mark labs complete in [CHECKLIST.md](../CHECKLIST.md)
2. 📝 Review [INTERVIEW.md](./INTERVIEW.md) questions
3. 🚀 Apply learnings to your production infrastructure
4. 📚 Continue learning and improving

---

## 💡 Troubleshooting Tips

### State Issues

```bash
# Backup state before operations
terraform state pull > backup.tfstate

# Restore state if needed
terraform state push backup.tfstate

# Remove corrupted resources
terraform state rm resource_type.name
```

### Security Issues

```bash
# Check security group rules
aws ec2 describe-security-groups --group-ids sg-xxxxx

# Verify KMS key policies
aws kms get-key-policy --key-id xxxxx --policy-name default

# Check IAM policies
aws iam get-role-policy --role-name role-name --policy-name policy-name
```

### Recovery Issues

```bash
# Verify backup integrity
aws s3 ls s3://backup-bucket/date/ --recursive

# Check state file version
terraform state pull | jq '.version'

# Force state refresh
terraform refresh -lock=false
```

---

**🎉 Congratulations!** You've completed all Terraform labs and are ready for production!
