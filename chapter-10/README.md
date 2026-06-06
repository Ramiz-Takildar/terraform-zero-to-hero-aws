# Chapter 10: Best Practices and Production-Ready Terraform

## 📋 Overview

This final chapter covers best practices for production-ready Terraform deployments. You'll learn how to build secure, scalable, and maintainable infrastructure that meets enterprise requirements.

**What You'll Learn:**
- Production readiness checklist
- Security best practices
- State management strategies
- Disaster recovery and backup
- Performance optimization
- Team collaboration
- Compliance and governance
- Operational excellence

**Prerequisites:**
- Completed Chapters 1-9
- Understanding of AWS services
- Experience with Terraform
- Basic security knowledge

---

## 🎯 Learning Objectives

By the end of this chapter, you will be able to:

1. ✅ Design production-ready infrastructure
2. ✅ Implement security best practices
3. ✅ Manage state securely
4. ✅ Plan disaster recovery strategies
5. ✅ Optimize Terraform performance
6. ✅ Enable team collaboration
7. ✅ Ensure compliance and governance
8. ✅ Monitor and maintain infrastructure
9. ✅ Handle incidents and rollbacks
10. ✅ Document infrastructure effectively

---

## 🏗️ Production Readiness Checklist

### Infrastructure Design

```
✅ Architecture
   ├── Multi-region deployment
   ├── High availability
   ├── Fault tolerance
   ├── Scalability
   └── Cost optimization

✅ Security
   ├── Encryption at rest
   ├── Encryption in transit
   ├── Network segmentation
   ├── IAM least privilege
   └── Security monitoring

✅ Compliance
   ├── Regulatory requirements
   ├── Data residency
   ├── Audit logging
   ├── Policy enforcement
   └── Documentation

✅ Operations
   ├── Monitoring and alerting
   ├── Backup and recovery
   ├── Incident response
   ├── Change management
   └── Runbooks
```

### Code Quality

```hcl
# ✅ Version Pinning
terraform {
  required_version = "~> 1.6.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# ✅ Input Validation
variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# ✅ Preconditions
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  
  lifecycle {
    precondition {
      condition     = var.environment == "prod" ? var.instance_type != "t2.micro" : true
      error_message = "Production instances cannot use t2.micro."
    }
  }
}

# ✅ Tagging Strategy
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = var.project_name
    Owner       = var.owner
    CostCenter  = var.cost_center
    Compliance  = var.compliance_level
  }
}

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-vpc"
    }
  )
}
```

---

## 🔒 Security Best Practices

### 1. State Security

**Remote Backend with Encryption:**

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-prod"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/12345678"
    dynamodb_table = "terraform-locks"
    
    # Enable versioning
    versioning = true
    
    # Server-side encryption
    server_side_encryption_configuration {
      rule {
        apply_server_side_encryption_by_default {
          sse_algorithm     = "aws:kms"
          kms_master_key_id = "arn:aws:kms:us-east-1:123456789012:key/12345678"
        }
      }
    }
  }
}
```

**State Bucket Configuration:**

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-prod"
  
  lifecycle {
    prevent_destroy = true
  }
  
  tags = local.common_tags
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_logging" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  target_bucket = aws_s3_bucket.logs.id
  target_prefix = "terraform-state/"
}
```

### 2. Secrets Management

**AWS Secrets Manager:**

```hcl
# Store secrets in AWS Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "${var.project_name}-db-password"
  recovery_window_in_days = 30
  
  tags = local.common_tags
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db_password.result
}

# Reference secrets in resources
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
}

resource "aws_db_instance" "main" {
  identifier = "${var.project_name}-db"
  
  username = "admin"
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
  
  # Other configuration...
}
```

**HashiCorp Vault:**

```hcl
provider "vault" {
  address = "https://vault.example.com"
}

data "vault_generic_secret" "db_credentials" {
  path = "secret/database/credentials"
}

resource "aws_db_instance" "main" {
  username = data.vault_generic_secret.db_credentials.data["username"]
  password = data.vault_generic_secret.db_credentials.data["password"]
}
```

### 3. Network Security

**VPC Security:**

```hcl
# VPC with private subnets
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  # Enable VPC Flow Logs
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-vpc"
    }
  )
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
  
  tags = local.common_tags
}

# Private subnets for databases
resource "aws_subnet" "private" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-private-${count.index + 1}"
      Type = "private"
    }
  )
}

# Network ACLs
resource "aws_network_acl" "private" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.private[*].id
  
  # Deny all inbound by default
  ingress {
    protocol   = -1
    rule_no    = 100
    action     = "deny"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }
  
  # Allow specific traffic
  ingress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = var.vpc_cidr
    from_port  = 3306
    to_port    = 3306
  }
  
  tags = local.common_tags
}
```

**Security Groups:**

```hcl
# Restrictive security groups
resource "aws_security_group" "database" {
  name        = "${var.project_name}-database-sg"
  description = "Security group for database"
  vpc_id      = aws_vpc.main.id
  
  # Only allow from application tier
  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.application.id]
    description     = "MySQL from application tier"
  }
  
  # No outbound internet access
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [var.vpc_cidr]
    description = "Internal VPC only"
  }
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-database-sg"
    }
  )
}
```

### 4. IAM Security

**Least Privilege Policies:**

```hcl
# IAM role with minimal permissions
resource "aws_iam_role" "ec2_instance" {
  name = "${var.project_name}-ec2-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
  
  tags = local.common_tags
}

# Specific permissions only
resource "aws_iam_role_policy" "ec2_instance" {
  name = "${var.project_name}-ec2-policy"
  role = aws_iam_role.ec2_instance.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.data.arn,
          "${aws_s3_bucket.data.arn}/*"
        ]
      },
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue"
        ]
        Resource = aws_secretsmanager_secret.app_config.arn
      }
    ]
  })
}
```

---

## 💾 State Management Strategies

### State Locking

```hcl
# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-locks"
  billing_mode = "PAY_PER_REQUEST"
  hash_key     = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  point_in_time_recovery {
    enabled = true
  }
  
  server_side_encryption {
    enabled = true
  }
  
  tags = merge(
    local.common_tags,
    {
      Name = "terraform-locks"
    }
  )
}
```

### State Isolation

**Separate States per Environment:**

```
terraform-state-prod/
├── networking/terraform.tfstate
├── compute/terraform.tfstate
├── database/terraform.tfstate
└── monitoring/terraform.tfstate

terraform-state-staging/
├── networking/terraform.tfstate
├── compute/terraform.tfstate
└── database/terraform.tfstate
```

**Backend Configuration:**

```hcl
# Production
terraform {
  backend "s3" {
    bucket = "terraform-state-prod"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}

# Staging
terraform {
  backend "s3" {
    bucket = "terraform-state-staging"
    key    = "networking/terraform.tfstate"
    region = "us-east-1"
  }
}
```

### State Backup

**Automated Backup Script:**

```bash
#!/bin/bash

# Backup Terraform state
BUCKET="terraform-state-prod"
BACKUP_BUCKET="terraform-state-backups"
DATE=$(date +%Y%m%d-%H%M%S)

# Copy current state to backup
aws s3 sync s3://${BUCKET}/ s3://${BACKUP_BUCKET}/${DATE}/ \
  --exclude "*" \
  --include "*.tfstate"

# Keep only last 30 days of backups
aws s3 ls s3://${BACKUP_BUCKET}/ | \
  awk '{print $2}' | \
  sort -r | \
  tail -n +31 | \
  xargs -I {} aws s3 rm s3://${BACKUP_BUCKET}/{} --recursive
```

---

## 🔄 Disaster Recovery

### Backup Strategy

**Multi-Region State Replication:**

```hcl
# Primary state bucket
resource "aws_s3_bucket" "terraform_state_primary" {
  provider = aws.us-east-1
  bucket   = "terraform-state-prod-primary"
  
  tags = local.common_tags
}

# Replica state bucket
resource "aws_s3_bucket" "terraform_state_replica" {
  provider = aws.us-west-2
  bucket   = "terraform-state-prod-replica"
  
  tags = local.common_tags
}

# Replication configuration
resource "aws_s3_bucket_replication_configuration" "terraform_state" {
  provider = aws.us-east-1
  
  bucket = aws_s3_bucket.terraform_state_primary.id
  role   = aws_iam_role.replication.arn
  
  rule {
    id     = "replicate-all"
    status = "Enabled"
    
    destination {
      bucket        = aws_s3_bucket.terraform_state_replica.arn
      storage_class = "STANDARD"
      
      replication_time {
        status = "Enabled"
        time {
          minutes = 15
        }
      }
      
      metrics {
        status = "Enabled"
        event_threshold {
          minutes = 15
        }
      }
    }
  }
}
```

### Recovery Procedures

**State Recovery:**

```bash
#!/bin/bash

# Recover from backup
BACKUP_DATE="20240101-120000"
BUCKET="terraform-state-prod"
BACKUP_BUCKET="terraform-state-backups"

# List available backups
echo "Available backups:"
aws s3 ls s3://${BACKUP_BUCKET}/

# Restore specific backup
aws s3 sync s3://${BACKUP_BUCKET}/${BACKUP_DATE}/ s3://${BUCKET}/ \
  --exclude "*" \
  --include "*.tfstate"

echo "State restored from backup: ${BACKUP_DATE}"
```

**Infrastructure Recovery:**

```bash
#!/bin/bash

# Recovery script
ENVIRONMENT="prod"
REGION="us-east-1"

# 1. Restore state from backup
./restore-state.sh

# 2. Verify state integrity
terraform init
terraform state list

# 3. Refresh state
terraform refresh

# 4. Plan recovery
terraform plan -out=recovery.tfplan

# 5. Apply recovery (with approval)
read -p "Apply recovery plan? (yes/no): " CONFIRM
if [ "$CONFIRM" = "yes" ]; then
  terraform apply recovery.tfplan
fi
```

---

## ⚡ Performance Optimization

### Parallel Execution

```hcl
# Enable parallelism
terraform apply -parallelism=20
```

### Resource Targeting

```bash
# Target specific resources
terraform apply -target=aws_vpc.main
terraform apply -target=module.networking
```

### State Optimization

```bash
# Remove unused resources from state
terraform state rm aws_instance.old

# Move resources
terraform state mv aws_instance.old aws_instance.new
```

### Module Caching

```hcl
# Use local module cache
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Cache modules
terraform init -plugin-dir=/path/to/cache
```

---

## 👥 Team Collaboration

### Code Organization

```
terraform-infrastructure/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── modules/
│   ├── networking/
│   ├── compute/
│   └── database/
├── policies/
│   └── sentinel/
├── scripts/
│   ├── backup.sh
│   └── restore.sh
└── docs/
    ├── architecture.md
    └── runbooks/
```

### Git Workflow

```bash
# Feature branch workflow
git checkout -b feature/add-monitoring
# Make changes
git add .
git commit -m "feat: add CloudWatch monitoring"
git push origin feature/add-monitoring
# Create pull request

# Code review process
# 1. Automated checks (CI/CD)
# 2. Peer review
# 3. Approval
# 4. Merge to main
```

### Documentation Standards

**README.md Template:**

```markdown
# Infrastructure Component

## Overview
Brief description of what this infrastructure manages.

## Architecture
Diagram or description of the architecture.

## Prerequisites
- AWS account
- Terraform >= 1.6.0
- Required permissions

## Usage
```bash
terraform init
terraform plan
terraform apply
```

## Variables
| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| vpc_cidr | VPC CIDR block | string | - | yes |

## Outputs
| Name | Description |
|------|-------------|
| vpc_id | VPC ID |

## Security
- Encryption: Enabled
- Network: Private subnets
- IAM: Least privilege

## Disaster Recovery
- Backup frequency: Daily
- Retention: 30 days
- RTO: 4 hours
- RPO: 1 hour

## Support
Contact: infrastructure-team@example.com
```

---

## 📊 Monitoring and Observability

### CloudWatch Integration

```hcl
# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "application" {
  name              = "/aws/application/${var.project_name}"
  retention_in_days = 30
  kms_key_id        = aws_kms_key.logs.arn
  
  tags = local.common_tags
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "${var.project_name}-high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_description   = "This metric monitors ec2 cpu utilization"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.main.name
  }
  
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
```

### Cost Monitoring

```hcl
# Cost anomaly detection
resource "aws_ce_anomaly_monitor" "service" {
  name              = "${var.project_name}-cost-monitor"
  monitor_type      = "DIMENSIONAL"
  monitor_dimension = "SERVICE"
}

resource "aws_ce_anomaly_subscription" "alerts" {
  name      = "${var.project_name}-cost-alerts"
  frequency = "DAILY"
  
  monitor_arn_list = [
    aws_ce_anomaly_monitor.service.arn
  ]
  
  subscriber {
    type    = "EMAIL"
    address = var.cost_alert_email
  }
  
  threshold_expression {
    dimension {
      key           = "ANOMALY_TOTAL_IMPACT_ABSOLUTE"
      values        = ["100"]
      match_options = ["GREATER_THAN_OR_EQUAL"]
    }
  }
}

# Budget alerts
resource "aws_budgets_budget" "monthly" {
  name              = "${var.project_name}-monthly-budget"
  budget_type       = "COST"
  limit_amount      = var.monthly_budget
  limit_unit        = "USD"
  time_period_start = "2024-01-01_00:00"
  time_unit         = "MONTHLY"
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_alert_email]
  }
}
```

---

## 📚 Summary

**Production Readiness:**
- Comprehensive security measures
- State management and backup
- Disaster recovery planning
- Performance optimization
- Team collaboration
- Monitoring and alerting

**Key Takeaways:**
1. Security is paramount in production
2. State management requires careful planning
3. Disaster recovery must be tested
4. Performance optimization is ongoing
5. Team collaboration needs structure
6. Monitoring is essential
7. Documentation is critical
8. Compliance must be maintained

**Best Practices:**
1. Use remote state with encryption
2. Implement least privilege IAM
3. Enable comprehensive logging
4. Automate backups
5. Test disaster recovery
6. Monitor costs and performance
7. Document everything
8. Review and audit regularly

**Next Steps:**
1. Complete hands-on labs in [LABS.md](./LABS.md)
2. Review interview questions in [INTERVIEW.md](./INTERVIEW.md)
3. Apply learnings to your projects
4. Continue learning and improving

---

**🎉 Congratulations!** You've completed the Terraform Zero to Hero AWS course!
