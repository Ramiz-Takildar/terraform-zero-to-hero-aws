# Chapter 4: Interview Questions - Modules

## 📋 Overview

This document contains interview questions covering Terraform modules, module composition, versioning, and best practices.

**Topics Covered:**
- Module Fundamentals
- Module Structure
- Module Sources
- Module Inputs and Outputs
- Module Versioning
- Public Modules
- Module Best Practices

---

## 📦 Module Fundamentals

### Q1: What are Terraform modules and why are they important?

**Answer:**

Modules are containers for multiple resources that are used together. They are the primary way to package and reuse Terraform configurations.

**Key Benefits:**

1. **Reusability:** Write once, use many times
2. **Organization:** Group related resources logically
3. **Abstraction:** Hide complexity from users
4. **Standardization:** Enforce organizational standards
5. **Collaboration:** Share infrastructure patterns

**Example:**

```hcl
# Without modules (repetitive)
resource "aws_vpc" "dev" {
  cidr_block = "10.0.0.0/16"
  # ... 50 lines of configuration
}

resource "aws_vpc" "staging" {
  cidr_block = "10.1.0.0/16"
  # ... same 50 lines repeated
}

resource "aws_vpc" "prod" {
  cidr_block = "10.2.0.0/16"
  # ... same 50 lines repeated again
}

# With modules (reusable)
module "vpc_dev" {
  source = "./modules/vpc"
  name   = "dev"
  cidr   = "10.0.0.0/16"
}

module "vpc_staging" {
  source = "./modules/vpc"
  name   = "staging"
  cidr   = "10.1.0.0/16"
}

module "vpc_prod" {
  source = "./modules/vpc"
  name   = "prod"
  cidr   = "10.2.0.0/16"
}
```

---

### Q2: What is the difference between root modules and child modules?

**Answer:**

| Aspect | Root Module | Child Module |
|--------|-------------|--------------|
| **Location** | Working directory | Called by root/other modules |
| **Purpose** | Entry point | Reusable component |
| **Backend** | Can have backend | Cannot have backend |
| **Providers** | Defines providers | Inherits providers |
| **State** | Has state file | Part of root state |

**Example:**

```
project/
├── main.tf           ← Root module
├── variables.tf      ← Root module
├── outputs.tf        ← Root module
└── modules/
    └── vpc/          ← Child module
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

**Root Module (main.tf):**
```hcl
terraform {
  backend "s3" {
    bucket = "my-state"
    key    = "terraform.tfstate"
  }
}

provider "aws" {
  region = "us-east-1"
}

module "vpc" {
  source = "./modules/vpc"
  name   = "production"
}
```

**Child Module (modules/vpc/main.tf):**
```hcl
# No backend configuration
# No provider configuration
# Just resources

resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
}
```

---

### Q3: What is the basic structure of a Terraform module?

**Answer:**

**Standard Module Structure:**

```
module/
├── main.tf          # Primary resource definitions
├── variables.tf     # Input variable declarations
├── outputs.tf       # Output value declarations
├── versions.tf      # Provider version constraints
├── README.md        # Module documentation
├── examples/        # Usage examples
│   └── basic/
│       └── main.tf
└── tests/           # Module tests
    └── basic_test.go
```

**main.tf:**
```hcl
resource "aws_vpc" "main" {
  cidr_block = var.cidr_block
  
  tags = merge(
    var.tags,
    {
      Name = var.name
    }
  )
}
```

**variables.tf:**
```hcl
variable "name" {
  description = "Name of the VPC"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}
```

**outputs.tf:**
```hcl
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}
```

**versions.tf:**
```hcl
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = ">= 5.0"
    }
  }
}
```

---

## 🔄 Module Sources

### Q4: What are the different module source types in Terraform?

**Answer:**

**1. Local Path:**
```hcl
module "vpc" {
  source = "./modules/vpc"
}

module "vpc" {
  source = "../shared-modules/vpc"
}
```

**2. Terraform Registry:**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}
```

**3. GitHub:**
```hcl
# HTTPS
module "vpc" {
  source = "github.com/company/terraform-modules//vpc?ref=v1.0.0"
}

# SSH
module "vpc" {
  source = "git@github.com:company/terraform-modules.git//vpc?ref=v1.0.0"
}

# Specific branch
module "vpc" {
  source = "github.com/company/terraform-modules//vpc?ref=main"
}

# Specific commit
module "vpc" {
  source = "github.com/company/terraform-modules//vpc?ref=abc123"
}
```

**4. Generic Git:**
```hcl
module "vpc" {
  source = "git::https://gitlab.com/company/modules.git//vpc?ref=v1.0.0"
}
```

**5. S3 Bucket:**
```hcl
module "vpc" {
  source = "s3::https://s3.amazonaws.com/my-bucket/modules/vpc.zip"
}
```

**6. HTTP URL:**
```hcl
module "vpc" {
  source = "https://example.com/modules/vpc.zip"
}
```

**Best Practices:**
- Use version tags for Git sources
- Use version constraints for Registry modules
- Use local paths for development
- Use remote sources for production

---

### Q5: How do you version modules and why is it important?

**Answer:**

**Why Version Modules:**
- Stability: Prevent breaking changes
- Reproducibility: Same version = same behavior
- Testing: Test specific versions
- Rollback: Revert to previous versions

**Versioning Strategies:**

**1. Git Tags (Recommended):**
```bash
# Tag a release
git tag -a v1.0.0 -m "Initial release"
git push origin v1.0.0

# Use in Terraform
module "vpc" {
  source = "git::https://github.com/company/modules.git//vpc?ref=v1.0.0"
}
```

**2. Terraform Registry:**
```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"  # Exact version
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"  # Any 5.x version
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = ">= 5.0, < 6.0"  # Range
}
```

**3. Semantic Versioning:**
```
v1.2.3
│ │ │
│ │ └─ Patch: Bug fixes (backward compatible)
│ └─── Minor: New features (backward compatible)
└───── Major: Breaking changes
```

**Version Constraints:**
```hcl
# Exact version
version = "1.0.0"

# Greater than or equal
version = ">= 1.0.0"

# Pessimistic constraint (recommended)
version = "~> 1.0"  # Allows 1.0.x, not 1.1.0

# Range
version = ">= 1.0.0, < 2.0.0"
```

---

## 📝 Module Inputs and Outputs

### Q6: How do you pass data into and out of modules?

**Answer:**

**Passing Data IN (Variables):**

**Module Definition:**
```hcl
# modules/vpc/variables.tf
variable "name" {
  description = "VPC name"
  type        = string
}

variable "cidr_block" {
  description = "VPC CIDR"
  type        = string
  default     = "10.0.0.0/16"
}

variable "tags" {
  description = "Tags"
  type        = map(string)
  default     = {}
}
```

**Module Usage:**
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  name       = "production"
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Environment = "prod"
    Team        = "platform"
  }
}
```

**Getting Data OUT (Outputs):**

**Module Definition:**
```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "Subnet IDs"
  value       = aws_subnet.main[*].id
}
```

**Module Usage:**
```hcl
# Use module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.subnet_ids[0]
  # ...
}

# Pass to another module
module "compute" {
  source = "./modules/compute"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.subnet_ids
}

# Output from root
output "vpc_id" {
  value = module.vpc.vpc_id
}
```

---

### Q7: What are the best practices for module variables?

**Answer:**

**1. Clear Naming:**
```hcl
# ❌ Bad
variable "c" {
  type = string
}

# ✅ Good
variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
}
```

**2. Provide Descriptions:**
```hcl
variable "instance_type" {
  description = "EC2 instance type (e.g., t2.micro, t2.small)"
  type        = string
  default     = "t2.micro"
}
```

**3. Use Appropriate Types:**
```hcl
variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 1
}

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

variable "subnet_cidrs" {
  description = "List of subnet CIDR blocks"
  type        = list(string)
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}
}
```

**4. Add Validation:**
```hcl
variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}
```

**5. Provide Sensible Defaults:**
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"  # Free tier eligible
}

variable "enable_deletion_protection" {
  description = "Enable deletion protection"
  type        = bool
  default     = true  # Safe default
}
```

**6. Use Objects for Complex Data:**
```hcl
variable "database_config" {
  description = "Database configuration"
  type = object({
    instance_class    = string
    allocated_storage = number
    engine_version    = string
    multi_az          = bool
  })
  
  default = {
    instance_class    = "db.t3.micro"
    allocated_storage = 20
    engine_version    = "14.7"
    multi_az          = false
  }
}
```

---

## 🎯 Module Composition

### Q8: How do you compose modules (nested modules)?

**Answer:**

Module composition allows building complex infrastructure from simpler modules.

**Example: Application Stack**

```hcl
# modules/application/main.tf

# Networking layer
module "networking" {
  source = "../networking"
  
  name               = var.name
  cidr_block         = var.vpc_cidr
  availability_zones = var.availability_zones
}

# Security layer
module "security" {
  source = "../security"
  
  name   = var.name
  vpc_id = module.networking.vpc_id
}

# Compute layer
module "compute" {
  source = "../compute"
  
  name              = var.name
  vpc_id            = module.networking.vpc_id
  subnet_ids        = module.networking.public_subnet_ids
  security_group_id = module.security.web_sg_id
  
  # Explicit dependency
  depends_on = [module.networking, module.security]
}

# Database layer
module "database" {
  source = "../database"
  
  name              = var.name
  vpc_id            = module.networking.vpc_id
  subnet_ids        = module.networking.private_subnet_ids
  security_group_id = module.security.db_sg_id
  
  depends_on = [module.networking, module.security]
}
```

**Usage:**
```hcl
module "app" {
  source = "./modules/application"
  
  name               = "myapp"
  vpc_cidr           = "10.0.0.0/16"
  availability_zones = ["us-east-1a", "us-east-1b"]
}
```

**Benefits:**
- Single module call creates entire stack
- Enforces dependencies
- Simplifies management
- Promotes reusability

---

### Q9: How do you handle module dependencies?

**Answer:**

**Implicit Dependencies (Automatic):**
```hcl
module "vpc" {
  source = "./modules/vpc"
  name   = "main"
}

module "compute" {
  source = "./modules/compute"
  
  # Implicit dependency via output reference
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.subnet_ids
}

# Terraform automatically knows compute depends on vpc
```

**Explicit Dependencies (Manual):**
```hcl
module "database" {
  source = "./modules/database"
  
  vpc_id = module.vpc.vpc_id
  
  # Explicit dependency
  depends_on = [
    module.vpc,
    module.security
  ]
}
```

**When to Use depends_on:**
- Hidden dependencies (not via outputs)
- Ordering requirements
- Resource initialization order
- Breaking circular dependencies

**Example with Hidden Dependency:**
```hcl
module "iam" {
  source = "./modules/iam"
  # Creates IAM roles
}

module "lambda" {
  source = "./modules/lambda"
  
  # Lambda needs IAM role to exist first
  # But doesn't reference it directly
  depends_on = [module.iam]
}
```

---

## 📚 Public Modules

### Q10: How do you use modules from the Terraform Registry?

**Answer:**

**Finding Modules:**
1. Visit https://registry.terraform.io
2. Search for modules (e.g., "aws vpc")
3. Review documentation and examples

**Using Registry Modules:**

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  
  tags = {
    Environment = "production"
  }
}
```

**Popular AWS Modules:**

```hcl
# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
}

# EC2 Instance
module "ec2" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "5.0.0"
}

# Security Group
module "sg" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "5.0.0"
}

# RDS
module "rds" {
  source  = "terraform-aws-modules/rds/aws"
  version = "6.0.0"
}

# S3 Bucket
module "s3" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "3.15.0"
}
```

**Best Practices:**
- Always specify version
- Review module source code
- Check module documentation
- Test before production use
- Pin to specific versions

---

### Q11: What should you consider when choosing a public module?

**Answer:**

**Evaluation Criteria:**

**1. Popularity and Usage:**
- Download count
- GitHub stars
- Community adoption

**2. Maintenance:**
- Recent updates
- Active maintainers
- Issue response time
- Pull request activity

**3. Documentation:**
- Clear README
- Usage examples
- Input/output documentation
- Changelog

**4. Code Quality:**
- Well-structured
- Follows best practices
- Includes tests
- Clear variable names

**5. Compatibility:**
- Terraform version requirements
- Provider version requirements
- AWS region support

**6. Security:**
- Security best practices
- No hardcoded credentials
- Secure defaults
- Regular security updates

**Example Evaluation:**

```hcl
# ✅ Good Module Choice
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  # Pros:
  # - Official AWS module
  # - 10M+ downloads
  # - Active maintenance
  # - Comprehensive documentation
  # - Well-tested
  # - Follows AWS best practices
}

# ❌ Questionable Module
module "vpc" {
  source  = "random-user/vpc/aws"
  version = "0.1.0"
  
  # Concerns:
  # - Unknown maintainer
  # - Low download count
  # - No recent updates
  # - Minimal documentation
  # - No tests
}
```

---

## 🔐 Module Best Practices

### Q12: What are the best practices for creating modules?

**Answer:**

**1. Single Responsibility:**
```hcl
# ✅ Good: Focused module
module "vpc" {
  # Only VPC-related resources
}

# ❌ Bad: Too many responsibilities
module "infrastructure" {
  # VPC, EC2, RDS, S3, IAM, etc.
}
```

**2. Encapsulation:**
```hcl
# ✅ Good: Hide complexity
module "database" {
  source = "./modules/database"
  
  name = "mydb"
  # Module handles subnet groups, parameter groups, etc.
}

# ❌ Bad: Expose too much
resource "aws_db_subnet_group" "main" { }
resource "aws_db_parameter_group" "main" { }
resource "aws_db_instance" "main" { }
# User must manage all details
```

**3. Sensible Defaults:**
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"  # Safe, cost-effective default
}

variable "enable_deletion_protection" {
  description = "Enable deletion protection"
  type        = bool
  default     = true  # Safe default
}
```

**4. Validation:**
```hcl
variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

**5. Documentation:**
```markdown
# VPC Module

## Usage

```hcl
module "vpc" {
  source = "./modules/vpc"
  
  name = "production"
  cidr = "10.0.0.0/16"
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| name | VPC name | string | n/a | yes |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | VPC ID |
```

**6. Testing:**
```go
// tests/vpc_test.go
func TestVPCCreation(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/basic",
    }
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcID)
}
```

---

### Q13: How do you handle sensitive data in modules?

**Answer:**

**1. Mark Outputs as Sensitive:**
```hcl
# modules/database/outputs.tf
output "db_password" {
  description = "Database password"
  value       = aws_db_instance.main.password
  sensitive   = true  # Won't show in logs
}
```

**2. Use Sensitive Variables:**
```hcl
# modules/database/variables.tf
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true  # Won't show in plan output
}
```

**3. Don't Hardcode Secrets:**
```hcl
# ❌ Bad
resource "aws_db_instance" "main" {
  password = "hardcoded-password"
}

# ✅ Good: Use Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

**4. Secure Defaults:**
```hcl
resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name
}

# Always block public access
resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Always enable encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "main" {
  bucket = aws_s3_bucket.main.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

---

### Q14: How do you test Terraform modules?

**Answer:**

**Testing Strategies:**

**1. Manual Testing:**
```bash
cd examples/basic
terraform init
terraform plan
terraform apply
terraform destroy
```

**2. Automated Testing (Terratest):**
```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestVPCModule(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/basic",
        Vars: map[string]interface{}{
            "name": "test-vpc",
            "cidr": "10.0.0.0/16",
        },
    }
    
    defer terraform.Destroy(t, terraformOptions)
    
    terraform.InitAndApply(t, terraformOptions)
    
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcID)
    
    vpcCIDR := terraform.Output(t, terraformOptions, "vpc_cidr")
    assert.Equal(t, "10.0.0.0/16", vpcCIDR)
}
```

**3. Validation Testing:**
```hcl
# Test variable validation
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Invalid environment."
  }
}

# Test with invalid value
# terraform plan -var="environment=invalid"
# Should fail with validation error
```

**4. Integration Testing:**
```bash
# Test module with real AWS resources
terraform apply -auto-approve

# Verify resources created
aws ec2 describe-vpcs --vpc-ids $(terraform output -raw vpc_id)

# Cleanup
terraform destroy -auto-approve
```

**5. Example Directory:**
```
module/
├── main.tf
├── variables.tf
├── outputs.tf
└── examples/
    ├── basic/
    │   └── main.tf
    ├── complete/
    │   └── main.tf
    └── minimal/
        └── main.tf
```

---

### Q15: What are common module anti-patterns to avoid?

**Answer:**

**1. God Modules (Too Much Responsibility):**
```hcl
# ❌ Bad: One module does everything
module "everything" {
  source = "./modules/infrastructure"
  # Creates VPC, EC2, RDS, S3, IAM, etc.
}

# ✅ Good: Focused modules
module "networking" { }
module "compute" { }
module "database" { }
```

**2. Hardcoded Values:**
```hcl
# ❌ Bad
resource "aws_instance" "web" {
  instance_type = "t2.micro"  # Hardcoded
  ami           = "ami-12345"  # Hardcoded
}

# ✅ Good
resource "aws_instance" "web" {
  instance_type = var.instance_type
  ami           = data.aws_ami.latest.id
}
```

**3. No Versioning:**
```hcl
# ❌ Bad: No version control
module "vpc" {
  source = "github.com/company/modules//vpc"
}

# ✅ Good: Pinned version
module "vpc" {
  source = "github.com/company/modules//vpc?ref=v1.0.0"
}
```

**4. Tight Coupling:**
```hcl
# ❌ Bad: Modules tightly coupled
module "app" {
  source = "./modules/app"
  # Assumes specific VPC structure
}

# ✅ Good: Loose coupling via inputs
module "app" {
  source = "./modules/app"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.subnet_ids
}
```

**5. No Documentation:**
```hcl
# ❌ Bad: No README, no comments
module "mystery" {
  source = "./modules/mystery"
  x      = "value"
  y      = 42
}

# ✅ Good: Well documented
module "vpc" {
  source = "./modules/vpc"
  
  # VPC name (used for tagging)
  name = "production"
  
  # VPC CIDR block
  cidr_block = "10.0.0.0/16"
}
```

---

## 📚 Summary

**Key Topics Covered:**
- Module fundamentals and benefits
- Module structure and organization
- Module sources and versioning
- Input variables and outputs
- Module composition
- Public modules from Registry
- Module best practices
- Testing strategies
- Common anti-patterns

**Best Practices:**
1. Keep modules focused (single responsibility)
2. Provide clear documentation
3. Use semantic versioning
4. Add input validation
5. Mark sensitive outputs
6. Provide sensible defaults
7. Test thoroughly
8. Follow naming conventions

**Next Steps:**
1. Practice labs in [LABS.md](./LABS.md)
2. Create your own modules
3. Proceed to [Chapter 5](../chapter-05/)

---

**💡 Pro Tips:**
- Start with simple modules
- Refactor as patterns emerge
- Use public modules as examples
- Version everything
- Document thoroughly
- Test before sharing
