# Chapter 10: Interview Questions - Best Practices and Production-Ready Terraform

## 📋 Overview

This document contains interview questions covering production-ready Terraform practices, security, disaster recovery, and operational excellence.

**Topics Covered:**
- Production readiness
- Security best practices
- State management
- Disaster recovery
- Performance optimization
- Team collaboration
- Compliance and governance
- Operational excellence

---

## 🏗️ Production Readiness

### Q1: What makes Terraform infrastructure "production-ready"?

**Answer:**

Production-ready infrastructure must meet multiple criteria across security, reliability, and operations.

**Key Requirements:**

**1. Security:**
```hcl
# Encrypted state
terraform {
  backend "s3" {
    bucket         = "terraform-state-prod"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/12345678"
    dynamodb_table = "terraform-locks"
  }
}

# Secrets management
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod-db-password"
}

# Least privilege IAM
resource "aws_iam_role_policy" "app" {
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject"]
      Resource = "${aws_s3_bucket.data.arn}/*"
    }]
  })
}
```

**2. Reliability:**
```hcl
# Multi-AZ deployment
resource "aws_subnet" "private" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
}

# Auto-scaling
resource "aws_autoscaling_group" "app" {
  min_size         = 2
  max_size         = 10
  desired_capacity = 3
  
  health_check_type         = "ELB"
  health_check_grace_period = 300
}

# Lifecycle rules
resource "aws_instance" "app" {
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = var.environment == "prod"
  }
}
```

**3. Monitoring:**
```hcl
# CloudWatch alarms
resource "aws_cloudwatch_metric_alarm" "high_cpu" {
  alarm_name          = "high-cpu"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "CPUUtilization"
  threshold           = 80
  alarm_actions       = [aws_sns_topic.alerts.arn]
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
}
```

**4. Disaster Recovery:**
```hcl
# State backup
resource "aws_s3_bucket_replication_configuration" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    status = "Enabled"
    destination {
      bucket = aws_s3_bucket.terraform_state_replica.arn
    }
  }
}

# Database backups
resource "aws_db_instance" "main" {
  backup_retention_period = 30
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
}
```

**5. Compliance:**
```hcl
# Tagging strategy
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = var.owner
    CostCenter  = var.cost_center
    Compliance  = "PCI-DSS"
    DataClass   = "Confidential"
  }
}

# Audit logging
resource "aws_cloudtrail" "main" {
  name                          = "prod-audit-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
}
```

**Production Readiness Checklist:**

```
✅ Security
   ├── Encrypted state with KMS
   ├── Secrets in Secrets Manager/Vault
   ├── Least privilege IAM
   ├── Network segmentation
   ├── Security groups locked down
   └── Encryption at rest and in transit

✅ Reliability
   ├── Multi-AZ deployment
   ├── Auto-scaling configured
   ├── Health checks enabled
   ├── Lifecycle rules set
   └── Redundancy built-in

✅ Monitoring
   ├── CloudWatch alarms
   ├── VPC Flow Logs
   ├── Application logs
   ├── Cost monitoring
   └── Performance metrics

✅ Disaster Recovery
   ├── State backups automated
   ├── Cross-region replication
   ├── RTO/RPO defined
   ├── DR procedures documented
   └── Regular DR testing

✅ Operations
   ├── CI/CD pipeline
   ├── Automated testing
   ├── Change management
   ├── Runbooks created
   └── On-call procedures

✅ Compliance
   ├── Tagging enforced
   ├── Audit logging enabled
   ├── Policy enforcement
   ├── Documentation complete
   └── Regular audits
```

---

### Q2: How do you handle Terraform state in production?

**Answer:**

Production state management requires security, reliability, and team collaboration considerations.

**Best Practices:**

**1. Remote Backend with Encryption:**

```hcl
terraform {
  backend "s3" {
    bucket         = "terraform-state-prod"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    kms_key_id     = "arn:aws:kms:us-east-1:123456789012:key/12345678"
    dynamodb_table = "terraform-locks"
    
    # Additional security
    acl                     = "private"
    server_side_encryption  = "aws:kms"
  }
}
```

**2. State Bucket Configuration:**

```hcl
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-prod"
  
  lifecycle {
    prevent_destroy = true
  }
}

# Versioning
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.terraform_state.arn
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

# Logging
resource "aws_s3_bucket_logging" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  target_bucket = aws_s3_bucket.logs.id
  target_prefix = "terraform-state/"
}

# Lifecycle policy
resource "aws_s3_bucket_lifecycle_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    id     = "archive-old-versions"
    status = "Enabled"
    
    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "GLACIER"
    }
    
    noncurrent_version_expiration {
      noncurrent_days = 90
    }
  }
}
```

**3. State Locking:**

```hcl
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
}
```

**4. State Isolation:**

```
# Separate states per environment and component
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

**5. State Backup:**

```bash
#!/bin/bash
# Automated backup script

BUCKET="terraform-state-prod"
BACKUP_BUCKET="terraform-state-backups"
DATE=$(date +%Y%m%d-%H%M%S)

# Backup current state
aws s3 sync s3://${BUCKET}/ s3://${BACKUP_BUCKET}/${DATE}/ \
  --exclude "*" \
  --include "*.tfstate"

# Keep only last 30 days
aws s3 ls s3://${BACKUP_BUCKET}/ | \
  awk '{print $2}' | \
  sort -r | \
  tail -n +31 | \
  xargs -I {} aws s3 rm s3://${BACKUP_BUCKET}/{} --recursive
```

**6. Cross-Region Replication:**

```hcl
resource "aws_s3_bucket_replication_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
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
    }
  }
}
```

**State Management Anti-Patterns:**

❌ **Don't:**
- Store state in version control
- Use local state for production
- Share state files manually
- Skip state locking
- Ignore state backups
- Mix environments in one state

✅ **Do:**
- Use remote backend
- Enable encryption
- Implement locking
- Automate backups
- Separate environments
- Monitor state access

---

## 🔒 Security

### Q3: What are the security best practices for production Terraform?

**Answer:**

**1. Secrets Management:**

```hcl
# AWS Secrets Manager
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "prod-db-password"
  recovery_window_in_days = 30
  kms_key_id              = aws_kms_key.secrets.id
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id     = aws_secretsmanager_secret.db_password.id
  secret_string = random_password.db_password.result
}

# Use in resources
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

**2. IAM Least Privilege:**

```hcl
# Specific permissions only
resource "aws_iam_role_policy" "app" {
  name = "app-policy"
  role = aws_iam_role.app.id
  
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

# Deny dangerous actions
resource "aws_iam_role_policy" "deny_dangerous" {
  name = "deny-dangerous"
  role = aws_iam_role.app.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Deny"
        Action = [
          "iam:*",
          "organizations:*",
          "account:*"
        ]
        Resource = "*"
      }
    ]
  })
}
```

**3. Network Security:**

```hcl
# Private subnets for sensitive resources
resource "aws_subnet" "database" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.20.0/24"
  # No public IP
  map_public_ip_on_launch = false
}

# Restrictive security groups
resource "aws_security_group" "database" {
  name   = "database-sg"
  vpc_id = aws_vpc.main.id
  
  # Only from application tier
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.application.id]
  }
  
  # No internet access
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = [aws_vpc.main.cidr_block]
  }
}

# Network ACLs
resource "aws_network_acl" "database" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = [aws_subnet.database.id]
  
  # Explicit allow rules only
  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = aws_vpc.main.cidr_block
    from_port  = 5432
    to_port    = 5432
  }
}
```

**4. Encryption:**

```hcl
# KMS keys
resource "aws_kms_key" "main" {
  description             = "Main encryption key"
  deletion_window_in_days = 30
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
      }
    ]
  })
}

# Encrypted EBS volumes
resource "aws_ebs_volume" "data" {
  encrypted  = true
  kms_key_id = aws_kms_key.main.arn
}

# Encrypted RDS
resource "aws_db_instance" "main" {
  storage_encrypted = true
  kms_key_id        = aws_kms_key.main.arn
}

# Encrypted S3
resource "aws_s3_bucket_server_side_encryption_configuration" "data" {
  bucket = aws_s3_bucket.data.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.main.arn
    }
  }
}
```

**5. Compliance Monitoring:**

```hcl
# AWS Config
resource "aws_config_configuration_recorder" "main" {
  name     = "config-recorder"
  role_arn = aws_iam_role.config.arn
  
  recording_group {
    all_supported                 = true
    include_global_resource_types = true
  }
}

# GuardDuty
resource "aws_guardduty_detector" "main" {
  enable = true
  
  datasources {
    s3_logs {
      enable = true
    }
  }
}

# Security Hub
resource "aws_securityhub_account" "main" {}

resource "aws_securityhub_standards_subscription" "cis" {
  standards_arn = "arn:aws:securityhub:::ruleset/cis-aws-foundations-benchmark/v/1.4.0"
}
```

---

### Q4: How do you implement security scanning in Terraform workflows?

**Answer:**

**Security Scanning Tools:**

**1. tfsec:**

```yaml
# GitHub Actions
- name: Run tfsec
  uses: aquasecurity/tfsec-action@v1.0.0
  with:
    soft_fail: false
    format: sarif
    out: tfsec-results.sarif

- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: tfsec-results.sarif
```

**2. Checkov:**

```yaml
- name: Run Checkov
  uses: bridgecrewio/checkov-action@master
  with:
    directory: .
    framework: terraform
    soft_fail: false
    output_format: sarif
```

**3. Terrascan:**

```yaml
- name: Run Terrascan
  uses: tenable/terrascan-action@main
  with:
    iac_type: terraform
    policy_type: aws
    only_warn: false
```

**4. Custom Policy Checks:**

```hcl
# Sentinel policy (Terraform Cloud)
import "tfplan/v2" as tfplan

# Ensure all S3 buckets are encrypted
main = rule {
    all tfplan.resource_changes as _, rc {
        rc.type is "aws_s3_bucket" implies
        rc.change.after.server_side_encryption_configuration != null
    }
}

# Ensure production instances are not t2.micro
production_instance_types = rule {
    all tfplan.resource_changes as _, rc {
        rc.type is "aws_instance" and
        rc.change.after.tags.Environment is "production" implies
        rc.change.after.instance_type not in ["t2.micro", "t2.nano"]
    }
}
```

**5. Pre-commit Hooks:**

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.83.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_tfsec
```

---

## 🔄 Disaster Recovery

### Q5: How do you design a disaster recovery strategy for Terraform-managed infrastructure?

**Answer:**

**DR Strategy Components:**

**1. Define RTO/RPO:**

```
RTO (Recovery Time Objective): 4 hours
RPO (Recovery Point Objective): 1 hour

Meaning:
- Infrastructure must be recoverable within 4 hours
- Maximum data loss: 1 hour
```

**2. State Backup Strategy:**

```bash
#!/bin/bash
# Automated hourly backup

BUCKET="terraform-state-prod"
BACKUP_BUCKET="terraform-state-backups"
DATE=$(date +%Y%m%d-%H%M%S)

# Backup with metadata
aws s3 sync s3://${BUCKET}/ s3://${BACKUP_BUCKET}/${DATE}/ \
  --exclude "*" \
  --include "*.tfstate" \
  --metadata "backup-date=${DATE},rpo=1hour"

# Verify backup
aws s3 ls s3://${BACKUP_BUCKET}/${DATE}/ --recursive | \
  grep -q "terraform.tfstate" && \
  echo "Backup successful" || \
  echo "Backup failed" && exit 1
```

**3. Cross-Region Replication:**

```hcl
# Primary region (us-east-1)
resource "aws_s3_bucket" "terraform_state_primary" {
  provider = aws.us-east-1
  bucket   = "terraform-state-prod-primary"
}

# DR region (us-west-2)
resource "aws_s3_bucket" "terraform_state_dr" {
  provider = aws.us-west-2
  bucket   = "terraform-state-prod-dr"
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
      bucket        = aws_s3_bucket.terraform_state_dr.arn
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

**4. Recovery Procedures:**

```bash
#!/bin/bash
# disaster-recovery.sh

set -e

echo "=== Disaster Recovery Procedure ==="

# Step 1: Assess damage
echo "1. Assessing infrastructure state..."
terraform init
terraform refresh || echo "State refresh failed"

# Step 2: Restore state from backup
echo "2. Restoring state from backup..."
LATEST_BACKUP=$(aws s3 ls s3://terraform-state-backups/ | tail -1 | awk '{print $2}')
aws s3 sync s3://terraform-state-backups/${LATEST_BACKUP}/ s3://terraform-state-prod/

# Step 3: Verify state integrity
echo "3. Verifying state integrity..."
terraform init -reconfigure
terraform validate

# Step 4: Plan recovery
echo "4. Planning recovery..."
terraform plan -out=recovery.tfplan

# Step 5: Review and apply
echo "5. Review plan and apply..."
terraform show recovery.tfplan
read -p "Apply recovery plan? (yes/no): " CONFIRM

if [ "$CONFIRM" = "yes" ]; then
    terraform apply recovery.tfplan
    echo "Recovery completed successfully"
else
    echo "Recovery cancelled"
    exit 1
fi

# Step 6: Verify services
echo "6. Verifying services..."
./verify-services.sh

echo "=== DR Complete ==="
```

**5. Multi-Region Infrastructure:**

```hcl
# Primary region
module "infrastructure_primary" {
  source = "./modules/infrastructure"
  
  providers = {
    aws = aws.us-east-1
  }
  
  region      = "us-east-1"
  environment = "production"
  is_primary  = true
}

# DR region
module "infrastructure_dr" {
  source = "./modules/infrastructure"
  
  providers = {
    aws = aws.us-west-2
  }
  
  region      = "us-west-2"
  environment = "production-dr"
  is_primary  = false
}

# Route53 failover
resource "aws_route53_record" "app" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  
  set_identifier = "primary"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  alias {
    name                   = module.infrastructure_primary.alb_dns_name
    zone_id                = module.infrastructure_primary.alb_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "app_dr" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "app.example.com"
  type    = "A"
  
  set_identifier = "secondary"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
  
  alias {
    name                   = module.infrastructure_dr.alb_dns_name
    zone_id                = module.infrastructure_dr.alb_zone_id
    evaluate_target_health = true
  }
}
```

**6. DR Testing Schedule:**

```markdown
# DR Testing Schedule

## Monthly
- State backup verification
- Restore test in non-prod
- Documentation review

## Quarterly
- Full DR drill
- Failover test
- RTO/RPO validation

## Annually
- Multi-region failover
- Complete infrastructure rebuild
- Team training
```

---

## ⚡ Performance

### Q6: How do you optimize Terraform performance for large infrastructures?

**Answer:**

**Optimization Strategies:**

**1. Parallelism:**

```bash
# Increase parallel resource operations
terraform apply -parallelism=20

# Default is 10, increase for large infrastructures
# Be careful not to hit API rate limits
```

**2. Targeted Operations:**

```bash
# Target specific resources
terraform apply -target=module.networking
terraform apply -target=aws_instance.web[0]

# Target multiple resources
terraform apply \
  -target=module.networking \
  -target=module.compute
```

**3. State Optimization:**

```bash
# Remove unused resources
terraform state rm aws_instance.old

# Move resources
terraform state mv aws_instance.old aws_instance.new

# List resources
terraform state list | wc -l  # Count resources
```

**4. Module Caching:**

```bash
# Use plugin cache
export TF_PLUGIN_CACHE_DIR="$HOME/.terraform.d/plugin-cache"
mkdir -p $TF_PLUGIN_CACHE_DIR

# Pre-download providers
terraform providers mirror $TF_PLUGIN_CACHE_DIR
```

**5. Workspace Isolation:**

```hcl
# Separate large infrastructures
terraform-infrastructure/
├── networking/      # Separate workspace
├── compute/         # Separate workspace
├── database/        # Separate workspace
└── monitoring/      # Separate workspace

# Each can be managed independently
```

**6. Efficient Data Sources:**

```hcl
# Cache data source results
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use locals to avoid repeated lookups
locals {
  ami_id = data.aws_ami.amazon_linux_2.id
}

resource "aws_instance" "web" {
  count = 10
  ami   = local.ami_id  # Reuse cached value
}
```

**7. Reduce Dependencies:**

```hcl
# Avoid unnecessary depends_on
# Let Terraform detect implicit dependencies

# Bad
resource "aws_instance" "web" {
  depends_on = [aws_vpc.main]  # Unnecessary
  subnet_id  = aws_subnet.public.id  # Implicit dependency
}

# Good
resource "aws_instance" "web" {
  subnet_id = aws_subnet.public.id  # Terraform handles dependency
}
```

**Performance Benchmarks:**

| Infrastructure Size | Resources | Plan Time | Apply Time |
|---------------------|-----------|-----------|------------|
| Small | < 50 | < 30s | < 2min |
| Medium | 50-200 | 30s-2min | 2-10min |
| Large | 200-1000 | 2-10min | 10-30min |
| Very Large | > 1000 | > 10min | > 30min |

**Optimization Tips:**

1. Split large states into smaller workspaces
2. Use `-refresh=false` when state is current
3. Implement module caching
4. Use targeted applies for changes
5. Increase parallelism carefully
6. Monitor API rate limits
7. Use local state for development

---

## 👥 Team Collaboration

### Q7: How do you enable effective team collaboration with Terraform?

**Answer:**

**Collaboration Strategies:**

**1. Code Organization:**

```
terraform-infrastructure/
├── .github/
│   └── workflows/
│       └── terraform.yml
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
├── modules/
│   ├── networking/
│   ├── compute/
│   └── database/
├── policies/
│   └── sentinel/
├── scripts/
│   ├── backup.sh
│   └── restore.sh
├── docs/
│   ├── architecture.md
│   ├── runbooks/
│   └── CONTRIBUTING.md
├── .gitignore
├── .pre-commit-config.yaml
└── README.md
```

**2. Git Workflow:**

```bash
# Feature branch workflow
git checkout -b feature/add-monitoring
# Make changes
git add .
git commit -m "feat: add CloudWatch monitoring"
git push origin feature/add-monitoring

# Create pull request
# Code review
# CI/CD checks
# Approval
# Merge to main
```

**3. Code Review Process:**

```markdown
# Pull Request Template

## Description
Brief description of changes

## Type of Change
- [ ] New feature
- [ ] Bug fix
- [ ] Breaking change
- [ ] Documentation update

## Checklist
- [ ] Code follows style guidelines
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] terraform fmt run
- [ ] terraform validate passed
- [ ] Security scan passed
- [ ] Plan reviewed

## Testing
How was this tested?

## Screenshots
If applicable
```

**4. Documentation Standards:**

```markdown
# Module Documentation Template

## Overview
Brief description

## Usage
```hcl
module "example" {
  source = "./modules/example"
  
  name = "my-resource"
}
```

## Requirements
| Name | Version |
|------|---------|
| terraform | >= 1.0 |
| aws | ~> 5.0 |

## Inputs
| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| name | Resource name | string | - | yes |

## Outputs
| Name | Description |
|------|-------------|
| id | Resource ID |

## Examples
See examples/ directory
```

**5. State Access Control:**

```hcl
# S3 bucket policy for team access
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
            "s3:x-amz-server-side-encryption" = "aws:kms"
          }
        }
      },
      {
        Sid    = "AllowTeamAccess"
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::123456789012:role/TerraformRole",
            "arn:aws:iam::123456789012:user/terraform-ci"
          ]
        }
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.terraform_state.arn,
          "${aws_s3_bucket.terraform_state.arn}/*"
        ]
      }
    ]
  })
}
```

**6. Communication:**

```yaml
# Slack notifications
- name: Notify Slack
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: |
      Terraform ${{ job.status }}
      Environment: ${{ env.ENVIRONMENT }}
      Author: ${{ github.actor }}
      Changes: ${{ github.event.head_commit.message }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## 📊 Monitoring

### Q8: What monitoring should be in place for Terraform-managed infrastructure?

**Answer:**

**Monitoring Layers:**

**1. Infrastructure Monitoring:**

```hcl
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
  alarm_description   = "CPU utilization is too high"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.main.name
  }
}

# Database monitoring
resource "aws_cloudwatch_metric_alarm" "database_connections" {
  alarm_name          = "${var.project_name}-db-connections"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "DatabaseConnections"
  namespace           = "AWS/RDS"
  period              = 300
  statistic           = "Average"
  threshold           = 80
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    DBInstanceIdentifier = aws_db_instance.main.id
  }
}
```

**2. Cost Monitoring:**

```hcl
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
  
  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = [var.budget_alert_email]
  }
}

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
```

**3. Security Monitoring:**

```hcl
# VPC Flow Logs
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
  
  tags = var.common_tags
}

# CloudTrail
resource "aws_cloudtrail" "main" {
  name                          = "${var.project_name}-audit-trail"
  s3_bucket_name                = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail         = true
  enable_log_file_validation    = true
  
  event_selector {
    read_write_type           = "All"
    include_management_events = true
    
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::${aws_s3_bucket.data.id}/*"]
    }
  }
  
  tags = var.common_tags
}

# GuardDuty findings
resource "aws_cloudwatch_event_rule" "guardduty_findings" {
  name        = "${var.project_name}-guardduty-findings"
  description = "Capture GuardDuty findings"
  
  event_pattern = jsonencode({
    source      = ["aws.guardduty"]
    detail-type = ["GuardDuty Finding"]
  })
}

resource "aws_cloudwatch_event_target" "sns" {
  rule      = aws_cloudwatch_event_rule.guardduty_findings.name
  target_id = "SendToSNS"
  arn       = aws_sns_topic.security_alerts.arn
}
```

**4. Application Monitoring:**

```hcl
# Application logs
resource "aws_cloudwatch_log_group" "application" {
  name              = "/aws/application/${var.project_name}"
  retention_in_days = 30
  kms_key_id        = aws_kms_key.logs.arn
  
  tags = var.common_tags
}

# Custom metrics
resource "aws_cloudwatch_log_metric_filter" "error_count" {
  name           = "ErrorCount"
  log_group_name = aws_cloudwatch_log_group.application.name
  pattern        = "[ERROR]"
  
  metric_transformation {
    name      = "ErrorCount"
    namespace = "Application"
    value     = "1"
  }
}

resource "aws_cloudwatch_metric_alarm" "error_rate" {
  alarm_name          = "${var.project_name}-error-rate"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ErrorCount"
  namespace           = "Application"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_actions       = [aws_sns_topic.alerts.arn]
}
```

---

## 📚 Summary

**Key Topics Covered:**
- Production readiness requirements
- State management best practices
- Security implementation
- Disaster recovery strategies
- Performance optimization
- Team collaboration
- Monitoring and alerting
- Operational excellence

**Best Practices:**
1. Encrypt everything (state, data, logs)
2. Implement least privilege access
3. Automate backups and test recovery
4. Monitor infrastructure and costs
5. Enable team collaboration
6. Document thoroughly
7. Test disaster recovery regularly
8. Optimize for performance

**Next Steps:**
1. Practice labs in [LABS.md](./LABS.md)
2. Apply learnings to production
3. Continue learning and improving
4. Share knowledge with team

---

**💡 Pro Tips:**
- Start with security, not as an afterthought
- Test DR procedures regularly
- Monitor costs proactively
- Document everything
- Automate repetitive tasks
- Review and audit regularly
- Keep learning and adapting
- Share knowledge with team

**🎉 Congratulations!** You've completed the Terraform Zero to Hero AWS course!
