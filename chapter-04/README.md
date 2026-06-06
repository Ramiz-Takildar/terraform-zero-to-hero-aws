# Chapter 4: Modules

## 📚 Learning Objectives

By the end of this chapter, you will:
- Understand Terraform modules and their benefits
- Create reusable modules
- Use module inputs and outputs
- Implement module versioning
- Use public modules from Terraform Registry
- Follow module best practices

**Prerequisites:** Chapters 1-3 completed  
**Estimated Time:** 3 days  
**Labs:** 3 hands-on exercises

---

## 📦 What are Modules?

Modules are containers for multiple resources that are used together. They enable code reuse and organization.

**Benefits:**
- **Reusability:** Write once, use many times
- **Organization:** Group related resources
- **Abstraction:** Hide complexity
- **Standardization:** Enforce best practices
- **Collaboration:** Share with team

**Module Structure:**
```
module/
├── main.tf          # Main resources
├── variables.tf     # Input variables
├── outputs.tf       # Output values
├── README.md        # Documentation
└── versions.tf      # Provider requirements
```

---

## 🏗️ Module Basics

### Simple Module Example

**Module Definition (modules/vpc/main.tf):**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support
  
  tags = merge(
    var.tags,
    {
      Name = var.name
    }
  )
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-public-${count.index + 1}"
      Type = "public"
    }
  )
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-igw"
    }
  )
}
```

**Module Variables (modules/vpc/variables.tf):**
```hcl
variable "name" {
  description = "Name prefix for resources"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

variable "enable_dns_hostnames" {
  description = "Enable DNS hostnames"
  type        = bool
  default     = true
}

variable "enable_dns_support" {
  description = "Enable DNS support"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}
```

**Module Outputs (modules/vpc/outputs.tf):**
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

output "internet_gateway_id" {
  description = "Internet Gateway ID"
  value       = aws_internet_gateway.main.id
}
```

### Using the Module

**Root Configuration (main.tf):**
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  name                 = "production"
  cidr_block           = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  availability_zones   = ["us-east-1a", "us-east-1b"]
  
  tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
  }
}

# Use module outputs
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  subnet_id     = module.vpc.public_subnet_ids[0]
  
  tags = {
    Name = "web-server"
  }
}

output "vpc_id" {
  value = module.vpc.vpc_id
}
```

---

## 🔄 Module Sources

### Local Modules

```hcl
module "vpc" {
  source = "./modules/vpc"
}

module "networking" {
  source = "../shared-modules/networking"
}
```

### Git Repository

```hcl
module "vpc" {
  source = "git::https://github.com/company/terraform-modules.git//vpc?ref=v1.0.0"
}

# SSH
module "vpc" {
  source = "git::ssh://git@github.com/company/terraform-modules.git//vpc?ref=v1.0.0"
}

# Specific branch
module "vpc" {
  source = "git::https://github.com/company/terraform-modules.git//vpc?ref=main"
}
```

### Terraform Registry

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
}
```

### S3 Bucket

```hcl
module "vpc" {
  source = "s3::https://s3.amazonaws.com/my-bucket/modules/vpc.zip"
}
```

---

## 📝 Module Best Practices

### 1. Clear Variable Names

```hcl
# ❌ Bad
variable "c" {
  type = string
}

# ✅ Good
variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
  
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}
```

### 2. Provide Defaults

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}
```

### 3. Use Validation

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

### 4. Document Everything

```hcl
# modules/vpc/README.md
```markdown
# VPC Module

Creates a VPC with public and private subnets across multiple AZs.

## Usage

```hcl
module "vpc" {
  source = "./modules/vpc"
  
  name                 = "production"
  cidr_block           = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  availability_zones   = ["us-east-1a", "us-east-1b"]
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| name | Name prefix | string | n/a | yes |
| cidr_block | VPC CIDR | string | "10.0.0.0/16" | no |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | VPC ID |
| public_subnet_ids | Public subnet IDs |
```

### 5. Version Constraints

```hcl
# modules/vpc/versions.tf
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

## 🎯 Module Composition

### Nested Modules

```hcl
# modules/app/main.tf
module "vpc" {
  source = "../vpc"
  
  name               = var.name
  cidr_block         = var.vpc_cidr
  public_subnet_cidrs = var.public_subnet_cidrs
  availability_zones  = var.availability_zones
}

module "security" {
  source = "../security"
  
  vpc_id = module.vpc.vpc_id
  name   = var.name
}

module "compute" {
  source = "../compute"
  
  vpc_id            = module.vpc.vpc_id
  subnet_ids        = module.vpc.public_subnet_ids
  security_group_id = module.security.web_sg_id
  instance_count    = var.instance_count
}
```

### Module Dependencies

```hcl
module "database" {
  source = "./modules/database"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  
  # Explicit dependency
  depends_on = [module.vpc]
}
```

---

## 🔢 Module Count and For Each

### Module with Count

```hcl
module "vpc" {
  count = 3
  
  source = "./modules/vpc"
  
  name       = "vpc-${count.index + 1}"
  cidr_block = "10.${count.index}.0.0/16"
}

output "vpc_ids" {
  value = module.vpc[*].vpc_id
}
```

### Module with For Each

```hcl
variable "environments" {
  type = map(object({
    cidr_block = string
    azs        = list(string)
  }))
  
  default = {
    dev = {
      cidr_block = "10.0.0.0/16"
      azs        = ["us-east-1a"]
    }
    staging = {
      cidr_block = "10.1.0.0/16"
      azs        = ["us-east-1a", "us-east-1b"]
    }
    prod = {
      cidr_block = "10.2.0.0/16"
      azs        = ["us-east-1a", "us-east-1b", "us-east-1c"]
    }
  }
}

module "vpc" {
  for_each = var.environments
  
  source = "./modules/vpc"
  
  name       = each.key
  cidr_block = each.value.cidr_block
  azs        = each.value.azs
}

output "vpc_ids" {
  value = {
    for k, v in module.vpc : k => v.vpc_id
  }
}
```

---

## 📚 Public Module Example

### Using AWS VPC Module

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = false
  one_nat_gateway_per_az = true
  
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Environment = "production"
    Terraform   = "true"
  }
  
  vpc_tags = {
    Name = "my-vpc"
  }
  
  public_subnet_tags = {
    Type = "public"
  }
  
  private_subnet_tags = {
    Type = "private"
  }
}
```

### Using AWS EC2 Instance Module

```hcl
module "ec2_instance" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "5.0.0"
  
  name = "my-instance"
  
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  key_name               = "my-key"
  monitoring             = true
  vpc_security_group_ids = [module.security_group.security_group_id]
  subnet_id              = module.vpc.public_subnets[0]
  
  tags = {
    Environment = "production"
  }
}
```

---

## 🔐 Module Security

### Sensitive Outputs

```hcl
# modules/database/outputs.tf
output "db_password" {
  description = "Database password"
  value       = aws_db_instance.main.password
  sensitive   = true
}

output "db_endpoint" {
  description = "Database endpoint"
  value       = aws_db_instance.main.endpoint
}
```

### Secure Defaults

```hcl
# modules/s3/main.tf
resource "aws_s3_bucket" "main" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_public_access_block" "main" {
  bucket = aws_s3_bucket.main.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_versioning" "main" {
  bucket = aws_s3_bucket.main.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

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

## 📖 Key Takeaways

1. **Modules enable reusability** and code organization
2. **Use clear variable names** and descriptions
3. **Provide sensible defaults** where appropriate
4. **Document your modules** thoroughly
5. **Version your modules** for stability
6. **Use public modules** from Terraform Registry
7. **Implement validation** for inputs
8. **Mark sensitive outputs** appropriately

---

## 🔗 Next Steps

1. Review theory above
2. Complete [Lab 4.1](./LABS.md) - Create Custom Module
3. Complete [Lab 4.2](./LABS.md) - Use Public Modules
4. Complete [Lab 4.3](./LABS.md) - Module Composition
5. Update [CHECKLIST.md](../CHECKLIST.md)

**Next Chapter:** [Chapter 5: Variables and Outputs](../chapter-05/)
