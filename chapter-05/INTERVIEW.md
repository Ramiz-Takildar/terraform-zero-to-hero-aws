# Chapter 5: Interview Questions - Variables and Outputs

## 📋 Overview

This document contains interview questions covering Terraform variables, outputs, local values, and variable management best practices.

**Topics Covered:**
- Variable Types and Declarations
- Variable Validation
- Variable Precedence
- Complex Variable Types
- Outputs
- Local Values
- Variable Files

---

## 📝 Variable Fundamentals

### Q1: What are Terraform variables and why are they important?

**Answer:**

Variables allow you to parameterize your Terraform configurations, making them reusable and flexible.

**Key Benefits:**
1. **Reusability:** Same code for different environments
2. **Flexibility:** Easy configuration changes
3. **Maintainability:** Centralized configuration
4. **Security:** Separate sensitive data from code

**Example:**

```hcl
# Without variables (hardcoded)
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  count         = 3
}

# With variables (flexible)
variable "ami_id" {
  type = string
}

variable "instance_type" {
  type    = string
  default = "t2.micro"
}

variable "instance_count" {
  type = number
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  count         = var.instance_count
}
```

---

### Q2: What are the different variable types in Terraform?

**Answer:**

**Simple Types:**

```hcl
# String
variable "region" {
  type    = string
  default = "us-east-1"
}

# Number
variable "instance_count" {
  type    = number
  default = 2
}

# Bool
variable "enable_monitoring" {
  type    = bool
  default = false
}
```

**Collection Types:**

```hcl
# List
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# Set (unique values)
variable "allowed_ports" {
  type    = set(number)
  default = [80, 443]
}

# Map
variable "instance_types" {
  type = map(string)
  default = {
    dev  = "t2.micro"
    prod = "t2.large"
  }
}
```

**Complex Types:**

```hcl
# Object
variable "database_config" {
  type = object({
    instance_class = string
    storage        = number
    multi_az       = bool
  })
  default = {
    instance_class = "db.t3.micro"
    storage        = 20
    multi_az       = false
  }
}

# Tuple
variable "subnet_config" {
  type    = tuple([string, string, bool])
  default = ["10.0.1.0/24", "us-east-1a", true]
}

# List of objects
variable "subnets" {
  type = list(object({
    cidr = string
    az   = string
  }))
  default = [
    { cidr = "10.0.1.0/24", az = "us-east-1a" },
    { cidr = "10.0.2.0/24", az = "us-east-1b" }
  ]
}
```

---

### Q3: How do you provide values to variables?

**Answer:**

**Multiple Methods (in order of precedence):**

**1. Command Line:**
```bash
terraform apply -var="instance_type=t2.large"
terraform apply -var="instance_count=5"
```

**2. Variable Files:**
```bash
# terraform.tfvars (auto-loaded)
instance_type = "t2.medium"
instance_count = 3

# Custom file
terraform apply -var-file="prod.tfvars"
```

**3. Environment Variables:**
```bash
export TF_VAR_instance_type="t2.small"
export TF_VAR_instance_count=2
terraform apply
```

**4. Default Values:**
```hcl
variable "instance_type" {
  type    = string
  default = "t2.micro"  # Used if no other value provided
}
```

**5. Interactive Prompt:**
```bash
# If no value provided, Terraform prompts
terraform apply
# var.instance_type
#   Enter a value:
```

---

### Q4: What is variable precedence in Terraform?

**Answer:**

Variables are loaded in this order (later sources override earlier):

```
1. Environment variables (TF_VAR_name)
2. terraform.tfvars
3. terraform.tfvars.json
4. *.auto.tfvars (alphabetical order)
5. -var and -var-file flags (command line)
```

**Example:**

```hcl
# variables.tf
variable "instance_type" {
  default = "t2.micro"  # Priority 0 (lowest)
}
```

```bash
# 1. Environment variable
export TF_VAR_instance_type="t2.small"

# 2. terraform.tfvars
instance_type = "t2.medium"

# 3. Command line (highest priority)
terraform apply -var="instance_type=t2.large"

# Result: t2.large is used
```

**Precedence Test:**
```bash
# All set different values
export TF_VAR_region="us-west-1"
echo 'region = "us-west-2"' > terraform.tfvars
terraform apply -var="region=us-east-1"

# Result: us-east-1 (command line wins)
```

---

## ✅ Variable Validation

### Q5: How do you validate variable values in Terraform?

**Answer:**

Use `validation` blocks to enforce constraints:

**Basic Validation:**
```hcl
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

**Multiple Validations:**
```hcl
variable "instance_count" {
  type = number
  
  validation {
    condition     = var.instance_count > 0
    error_message = "Instance count must be positive."
  }
  
  validation {
    condition     = var.instance_count <= 10
    error_message = "Instance count cannot exceed 10."
  }
}
```

**Complex Validation:**
```hcl
variable "cidr_block" {
  type = string
  
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
  
  validation {
    condition     = tonumber(split("/", var.cidr_block)[1]) <= 24
    error_message = "CIDR must be /24 or larger."
  }
}
```

**Regex Validation:**
```hcl
variable "bucket_name" {
  type = string
  
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]*[a-z0-9]$", var.bucket_name))
    error_message = "Bucket name must be lowercase alphanumeric with hyphens."
  }
}
```

---

### Q6: What are some common validation patterns?

**Answer:**

**1. Enum Validation:**
```hcl
variable "instance_type" {
  type = string
  
  validation {
    condition = contains([
      "t2.micro", "t2.small", "t2.medium",
      "t3.micro", "t3.small", "t3.medium"
    ], var.instance_type)
    error_message = "Invalid instance type."
  }
}
```

**2. Range Validation:**
```hcl
variable "port" {
  type = number
  
  validation {
    condition     = var.port >= 1 && var.port <= 65535
    error_message = "Port must be between 1 and 65535."
  }
}
```

**3. Length Validation:**
```hcl
variable "name" {
  type = string
  
  validation {
    condition     = length(var.name) >= 3 && length(var.name) <= 32
    error_message = "Name must be 3-32 characters."
  }
}
```

**4. Format Validation:**
```hcl
variable "email" {
  type = string
  
  validation {
    condition     = can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", var.email))
    error_message = "Must be a valid email address."
  }
}
```

**5. IP Address Validation:**
```hcl
variable "ip_address" {
  type = string
  
  validation {
    condition     = can(cidrhost("${var.ip_address}/32", 0))
    error_message = "Must be a valid IP address."
  }
}
```

---

## 📤 Outputs

### Q7: What are Terraform outputs and when should you use them?

**Answer:**

Outputs expose values from your Terraform configuration for:
1. **Display:** Show important information after apply
2. **Module Integration:** Pass data between modules
3. **Automation:** Use in scripts and CI/CD
4. **Documentation:** Show what was created

**Basic Output:**
```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_ip" {
  description = "Public IP address"
  value       = aws_instance.web.public_ip
}
```

**Using Outputs:**
```bash
# View all outputs
terraform output

# View specific output
terraform output instance_id

# Get raw value (no quotes)
terraform output -raw instance_ip

# JSON format
terraform output -json

# Use in scripts
INSTANCE_IP=$(terraform output -raw instance_ip)
curl http://$INSTANCE_IP
```

---

### Q8: How do you handle sensitive data in outputs?

**Answer:**

Use the `sensitive` flag to prevent values from being displayed:

```hcl
output "db_password" {
  description = "Database password"
  value       = aws_db_instance.main.password
  sensitive   = true  # Won't show in console
}

output "db_endpoint" {
  description = "Database endpoint"
  value       = aws_db_instance.main.endpoint
  # Not sensitive, will display
}
```

**Behavior:**
```bash
# Regular output
terraform output
# db_endpoint = "mydb.abc123.us-east-1.rds.amazonaws.com:5432"
# db_password = <sensitive>

# Get sensitive value
terraform output db_password
# (sensitive value)

# Get raw sensitive value
terraform output -raw db_password
# actual-password-value
```

**Best Practices:**
- Always mark passwords as sensitive
- Mark API keys as sensitive
- Mark private keys as sensitive
- Don't mark public information as sensitive

---

### Q9: What are complex output patterns?

**Answer:**

**List Outputs:**
```hcl
output "instance_ids" {
  description = "All instance IDs"
  value       = aws_instance.web[*].id
}

output "instance_ips" {
  description = "All instance IPs"
  value       = aws_instance.web[*].public_ip
}
```

**Map Outputs:**
```hcl
output "instance_map" {
  description = "Map of instance names to IPs"
  value = {
    for instance in aws_instance.web :
    instance.tags["Name"] => instance.public_ip
  }
}
```

**Object Outputs:**
```hcl
output "instance_details" {
  description = "Complete instance information"
  value = {
    id         = aws_instance.web.id
    public_ip  = aws_instance.web.public_ip
    private_ip = aws_instance.web.private_ip
    az         = aws_instance.web.availability_zone
    type       = aws_instance.web.instance_type
  }
}
```

**Conditional Outputs:**
```hcl
output "nat_gateway_ip" {
  description = "NAT Gateway IP (if enabled)"
  value       = var.enable_nat ? aws_eip.nat[0].public_ip : null
}
```

**Computed Outputs:**
```hcl
output "web_urls" {
  description = "URLs to access instances"
  value = [
    for ip in aws_instance.web[*].public_ip :
    "http://${ip}"
  ]
}
```

---

## 🔧 Local Values

### Q10: What are local values and when should you use them?

**Answer:**

Local values assign names to expressions for reuse within a module.

**Use Cases:**
1. **Avoid Repetition:** DRY principle
2. **Computed Values:** Complex calculations
3. **Conditional Logic:** Environment-specific config
4. **Readability:** Name complex expressions

**Basic Locals:**
```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = var.project_name
  }
  
  name_prefix = "${var.project_name}-${var.environment}"
}

resource "aws_instance" "web" {
  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-web"
    }
  )
}
```

**Computed Locals:**
```hcl
locals {
  # Calculate subnet CIDRs
  subnet_cidrs = [
    for i in range(var.subnet_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]
  
  # Environment-specific config
  instance_count = {
    dev  = 1
    prod = 5
  }
  
  current_count = local.instance_count[var.environment]
}
```

---

### Q11: What's the difference between variables and locals?

**Answer:**

| Aspect | Variables | Locals |
|--------|-----------|--------|
| **Input** | From outside | Computed internally |
| **Can be set** | Yes (CLI, files, etc.) | No |
| **Validation** | Yes | No |
| **Scope** | Module input | Module internal |
| **Purpose** | Configuration | Computation |

**Example:**

```hcl
# Variables: Input from user
variable "environment" {
  type = string
}

variable "project_name" {
  type = string
}

# Locals: Computed from variables
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  
  instance_config = {
    dev  = { type = "t2.micro", count = 1 }
    prod = { type = "t2.large", count = 5 }
  }
  
  current_config = local.instance_config[var.environment]
}

# Usage
resource "aws_instance" "app" {
  count = local.current_config.count
  
  instance_type = local.current_config.type
  
  tags = {
    Name = "${local.name_prefix}-app-${count.index}"
  }
}
```

---

## 📁 Variable Files

### Q12: What are the different types of variable files?

**Answer:**

**1. terraform.tfvars (Auto-loaded):**
```hcl
# terraform.tfvars
region         = "us-east-1"
instance_type  = "t2.micro"
instance_count = 3
```

**2. *.auto.tfvars (Auto-loaded):**
```hcl
# common.auto.tfvars
tags = {
  ManagedBy = "Terraform"
  Team      = "Platform"
}

# dev.auto.tfvars
environment = "dev"
```

**3. Custom .tfvars (Manual):**
```hcl
# prod.tfvars
environment    = "prod"
instance_type  = "t2.large"
instance_count = 5

# Usage
terraform apply -var-file="prod.tfvars"
```

**4. JSON Format:**
```json
{
  "region": "us-east-1",
  "instance_type": "t2.micro",
  "tags": {
    "Environment": "production"
  }
}
```

**Loading Order:**
1. terraform.tfvars
2. terraform.tfvars.json
3. *.auto.tfvars (alphabetical)
4. -var-file flags (order specified)

---

### Q13: How do you organize variables for multiple environments?

**Answer:**

**Strategy 1: Separate Variable Files**

```
project/
├── main.tf
├── variables.tf
├── dev.tfvars
├── staging.tfvars
└── prod.tfvars
```

```bash
terraform apply -var-file="dev.tfvars"
terraform apply -var-file="prod.tfvars"
```

**Strategy 2: Workspaces + Locals**

```hcl
locals {
  env_config = {
    dev = {
      instance_type = "t2.micro"
      instance_count = 1
    }
    prod = {
      instance_type = "t2.large"
      instance_count = 5
    }
  }
  
  config = local.env_config[terraform.workspace]
}
```

**Strategy 3: Separate Directories**

```
terraform/
├── dev/
│   ├── main.tf
│   └── terraform.tfvars
├── staging/
│   ├── main.tf
│   └── terraform.tfvars
└── prod/
    ├── main.tf
    └── terraform.tfvars
```

**Strategy 4: Environment Variable**

```hcl
variable "environment" {
  type = string
}

locals {
  config = {
    dev  = { /* dev config */ }
    prod = { /* prod config */ }
  }[var.environment]
}
```

```bash
terraform apply -var="environment=dev"
terraform apply -var="environment=prod"
```

---

## 🎨 Advanced Patterns

### Q14: How do you implement feature flags with variables?

**Answer:**

```hcl
variable "features" {
  description = "Feature flags"
  type = object({
    enable_cdn        = bool
    enable_waf        = bool
    enable_monitoring = bool
    enable_backup     = bool
  })
  
  default = {
    enable_cdn        = false
    enable_waf        = false
    enable_monitoring = true
    enable_backup     = true
  }
}

# Conditional resources
resource "aws_cloudfront_distribution" "main" {
  count = var.features.enable_cdn ? 1 : 0
  # CloudFront configuration
}

resource "aws_wafv2_web_acl" "main" {
  count = var.features.enable_waf ? 1 : 0
  # WAF configuration
}

resource "aws_cloudwatch_metric_alarm" "main" {
  count = var.features.enable_monitoring ? 1 : 0
  # Monitoring configuration
}
```

**Usage:**
```hcl
# dev.tfvars
features = {
  enable_cdn        = false
  enable_waf        = false
  enable_monitoring = false
  enable_backup     = false
}

# prod.tfvars
features = {
  enable_cdn        = true
  enable_waf        = true
  enable_monitoring = true
  enable_backup     = true
}
```

---

### Q15: How do you handle dynamic tagging?

**Answer:**

```hcl
variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}

variable "environment" {
  type = string
}

variable "project_name" {
  type = string
}

locals {
  # Default tags
  default_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
    CreatedAt   = timestamp()
  }
  
  # Merge with user tags
  all_tags = merge(local.default_tags, var.tags)
}

# Apply to all resources
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  tags = local.all_tags
}

resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = merge(
    local.all_tags,
    {
      Name = "${var.project_name}-vpc"
    }
  )
}
```

**Usage:**
```hcl
# terraform.tfvars
environment  = "production"
project_name = "myapp"

tags = {
  CostCenter = "engineering"
  Owner      = "platform-team"
  Compliance = "pci-dss"
}
```

---

## 📚 Summary

**Key Topics Covered:**
- Variable types (simple, collection, complex)
- Variable validation patterns
- Variable precedence and loading
- Output declarations and usage
- Sensitive outputs
- Local values for computation
- Variable files and organization
- Environment-specific configuration
- Feature flags and dynamic tagging

**Best Practices:**
1. Always add descriptions to variables
2. Use appropriate types
3. Add validation rules
4. Provide sensible defaults
5. Mark sensitive outputs
6. Use locals for DRY code
7. Organize variables logically
8. Document variable files

**Next Steps:**
1. Practice labs in [LABS.md](./LABS.md)
2. Implement validation in your projects
3. Proceed to [Chapter 6](../chapter-06/)

---

**💡 Pro Tips:**
- Use `terraform console` to test expressions
- Validate variables early in development
- Keep variable files in version control
- Use `.gitignore` for sensitive tfvars
- Document expected variable values
- Use consistent naming conventions
