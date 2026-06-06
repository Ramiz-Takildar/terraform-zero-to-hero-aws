# Chapter 5: Variables and Outputs

## 📚 Learning Objectives

By the end of this chapter, you will:
- Master variable types and declarations
- Use variable validation and defaults
- Understand variable precedence
- Work with complex variable types
- Create meaningful outputs
- Use locals for computed values
- Implement variable files effectively

**Prerequisites:** Chapters 1-4 completed  
**Estimated Time:** 2 days  
**Labs:** 3 hands-on exercises

---

## 📝 Variable Basics

### What are Variables?

Variables allow you to parameterize your Terraform configurations, making them reusable and flexible.

**Basic Variable Declaration:**

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 1
}

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}
```

**Using Variables:**

```hcl
resource "aws_instance" "web" {
  count = var.instance_count
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  monitoring    = var.enable_monitoring
  
  tags = {
    Name = "web-${count.index + 1}"
  }
}
```

---

## 🎯 Variable Types

### Simple Types

**String:**
```hcl
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}
```

**Number:**
```hcl
variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 2
}
```

**Bool:**
```hcl
variable "enable_vpn" {
  description = "Enable VPN gateway"
  type        = bool
  default     = false
}
```

### Collection Types

**List:**
```hcl
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# Usage
resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  availability_zone = var.availability_zones[count.index]
  cidr_block        = "10.0.${count.index + 1}.0/24"
}
```

**Set:**
```hcl
variable "allowed_ports" {
  description = "Set of allowed ports"
  type        = set(number)
  default     = [80, 443, 22]
}

# Usage
resource "aws_security_group_rule" "ingress" {
  for_each = var.allowed_ports
  
  type        = "ingress"
  from_port   = each.value
  to_port     = each.value
  protocol    = "tcp"
  cidr_blocks = ["0.0.0.0/0"]
}
```

**Map:**
```hcl
variable "instance_types" {
  description = "Map of environment to instance type"
  type        = map(string)
  default = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t2.medium"
  }
}

# Usage
resource "aws_instance" "app" {
  instance_type = var.instance_types[var.environment]
}
```

### Complex Types

**Object:**
```hcl
variable "database_config" {
  description = "Database configuration"
  type = object({
    instance_class    = string
    allocated_storage = number
    engine_version    = string
    multi_az          = bool
    backup_retention  = number
  })
  
  default = {
    instance_class    = "db.t3.micro"
    allocated_storage = 20
    engine_version    = "14.7"
    multi_az          = false
    backup_retention  = 7
  }
}

# Usage
resource "aws_db_instance" "main" {
  instance_class         = var.database_config.instance_class
  allocated_storage      = var.database_config.allocated_storage
  engine_version         = var.database_config.engine_version
  multi_az               = var.database_config.multi_az
  backup_retention_period = var.database_config.backup_retention
}
```

**Tuple:**
```hcl
variable "subnet_config" {
  description = "Subnet configuration [cidr, az, public]"
  type        = tuple([string, string, bool])
  default     = ["10.0.1.0/24", "us-east-1a", true]
}
```

**List of Objects:**
```hcl
variable "subnets" {
  description = "List of subnet configurations"
  type = list(object({
    cidr_block        = string
    availability_zone = string
    public            = bool
  }))
  
  default = [
    {
      cidr_block        = "10.0.1.0/24"
      availability_zone = "us-east-1a"
      public            = true
    },
    {
      cidr_block        = "10.0.2.0/24"
      availability_zone = "us-east-1b"
      public            = true
    }
  ]
}

# Usage
resource "aws_subnet" "main" {
  count = length(var.subnets)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.subnets[count.index].cidr_block
  availability_zone       = var.subnets[count.index].availability_zone
  map_public_ip_on_launch = var.subnets[count.index].public
}
```

---

## ✅ Variable Validation

### Basic Validation

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

### Multiple Validations

```hcl
variable "instance_count" {
  description = "Number of instances"
  type        = number
  
  validation {
    condition     = var.instance_count > 0
    error_message = "Instance count must be greater than 0."
  }
  
  validation {
    condition     = var.instance_count <= 10
    error_message = "Instance count must not exceed 10."
  }
}
```

### Complex Validation

```hcl
variable "cidr_block" {
  description = "VPC CIDR block"
  type        = string
  
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
  
  validation {
    condition     = tonumber(split("/", var.cidr_block)[1]) <= 24
    error_message = "CIDR block must be /24 or larger."
  }
}
```

### Regex Validation

```hcl
variable "bucket_name" {
  description = "S3 bucket name"
  type        = string
  
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]*[a-z0-9]$", var.bucket_name))
    error_message = "Bucket name must be lowercase alphanumeric with hyphens."
  }
  
  validation {
    condition     = length(var.bucket_name) >= 3 && length(var.bucket_name) <= 63
    error_message = "Bucket name must be between 3 and 63 characters."
  }
}
```

---

## 🔢 Variable Precedence

Terraform loads variables in this order (later sources override earlier):

1. **Environment variables** (`TF_VAR_name`)
2. **terraform.tfvars** file
3. **terraform.tfvars.json** file
4. ***.auto.tfvars** files (alphabetical order)
5. **-var** and **-var-file** command-line flags

**Example:**

```bash
# 1. Environment variable
export TF_VAR_instance_type="t2.small"

# 2. terraform.tfvars
instance_type = "t2.medium"

# 3. Command line (highest priority)
terraform apply -var="instance_type=t2.large"

# Result: t2.large is used
```

---

## 📁 Variable Files

### terraform.tfvars

```hcl
# terraform.tfvars
region          = "us-east-1"
instance_type   = "t2.micro"
instance_count  = 3
enable_monitoring = true

tags = {
  Environment = "production"
  Team        = "platform"
}
```

### Environment-Specific Files

```hcl
# dev.tfvars
environment    = "dev"
instance_type  = "t2.micro"
instance_count = 1
enable_monitoring = false

# prod.tfvars
environment    = "prod"
instance_type  = "t2.large"
instance_count = 5
enable_monitoring = true
```

**Usage:**
```bash
terraform apply -var-file="dev.tfvars"
terraform apply -var-file="prod.tfvars"
```

### Auto-loaded Files

```
project/
├── terraform.tfvars       # Auto-loaded
├── terraform.tfvars.json  # Auto-loaded
├── dev.auto.tfvars        # Auto-loaded
├── prod.auto.tfvars       # Auto-loaded
└── custom.tfvars          # Must specify with -var-file
```

---

## 📤 Outputs

### Basic Outputs

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP address"
  value       = aws_instance.web.public_ip
}

output "instance_private_ip" {
  description = "Private IP address"
  value       = aws_instance.web.private_ip
}
```

### Sensitive Outputs

```hcl
output "db_password" {
  description = "Database password"
  value       = aws_db_instance.main.password
  sensitive   = true  # Won't show in console
}

output "db_endpoint" {
  description = "Database endpoint"
  value       = aws_db_instance.main.endpoint
}
```

### Complex Outputs

```hcl
output "instance_details" {
  description = "Complete instance details"
  value = {
    id         = aws_instance.web.id
    public_ip  = aws_instance.web.public_ip
    private_ip = aws_instance.web.private_ip
    az         = aws_instance.web.availability_zone
  }
}

output "all_instance_ips" {
  description = "All instance public IPs"
  value       = aws_instance.web[*].public_ip
}
```

### Conditional Outputs

```hcl
output "nat_gateway_ip" {
  description = "NAT Gateway public IP"
  value       = var.enable_nat_gateway ? aws_eip.nat[0].public_ip : null
}
```

### Using Outputs

```bash
# View all outputs
terraform output

# View specific output
terraform output instance_id

# Get raw value (no quotes)
terraform output -raw instance_public_ip

# JSON format
terraform output -json

# Use in scripts
INSTANCE_IP=$(terraform output -raw instance_public_ip)
curl http://$INSTANCE_IP
```

---

## 🔧 Local Values

Locals allow you to assign names to expressions for reuse.

### Basic Locals

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
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  
  tags = merge(
    local.common_tags,
    {
      Name = "${local.name_prefix}-web"
    }
  )
}
```

### Computed Locals

```hcl
locals {
  # Compute subnet CIDRs
  subnet_cidrs = [
    for i in range(var.subnet_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]
  
  # Environment-specific settings
  instance_count = {
    dev     = 1
    staging = 2
    prod    = 5
  }
  
  # Get count for current environment
  current_instance_count = local.instance_count[var.environment]
}

resource "aws_subnet" "main" {
  count = length(local.subnet_cidrs)
  
  vpc_id     = aws_vpc.main.id
  cidr_block = local.subnet_cidrs[count.index]
}
```

### Complex Locals

```hcl
locals {
  # Flatten nested structures
  subnet_route_associations = flatten([
    for subnet_key, subnet in aws_subnet.private : [
      for rt_key, rt in aws_route_table.private : {
        subnet_id      = subnet.id
        route_table_id = rt.id
        key            = "${subnet_key}-${rt_key}"
      }
    ]
  ])
  
  # Conditional logic
  enable_nat = var.environment == "prod" ? true : false
  
  # String manipulation
  bucket_name = lower(replace(var.project_name, "_", "-"))
}
```

---

## 🎨 Variable Patterns

### Environment Configuration

```hcl
variable "environment" {
  description = "Environment name"
  type        = string
}

locals {
  env_config = {
    dev = {
      instance_type = "t2.micro"
      instance_count = 1
      enable_monitoring = false
      enable_backup = false
    }
    staging = {
      instance_type = "t2.small"
      instance_count = 2
      enable_monitoring = true
      enable_backup = true
    }
    prod = {
      instance_type = "t2.large"
      instance_count = 5
      enable_monitoring = true
      enable_backup = true
    }
  }
  
  current_config = local.env_config[var.environment]
}

resource "aws_instance" "app" {
  count = local.current_config.instance_count
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = local.current_config.instance_type
  monitoring    = local.current_config.enable_monitoring
}
```

### Feature Flags

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

resource "aws_cloudfront_distribution" "main" {
  count = var.features.enable_cdn ? 1 : 0
  # CloudFront configuration
}

resource "aws_wafv2_web_acl" "main" {
  count = var.features.enable_waf ? 1 : 0
  # WAF configuration
}
```

### Dynamic Tagging

```hcl
variable "tags" {
  description = "Additional tags"
  type        = map(string)
  default     = {}
}

locals {
  default_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = var.project_name
    CreatedAt   = timestamp()
  }
  
  all_tags = merge(local.default_tags, var.tags)
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  
  tags = local.all_tags
}
```

---

## 📖 Key Takeaways

1. **Variables** make configurations reusable and flexible
2. **Use appropriate types** for better validation
3. **Add validation rules** to catch errors early
4. **Understand precedence** for variable values
5. **Use locals** for computed values
6. **Mark sensitive outputs** to protect secrets
7. **Organize variables** in separate files
8. **Document variables** with descriptions

---

## 🔗 Next Steps

1. Review theory above
2. Complete [Lab 5.1](./LABS.md) - Variable Types
3. Complete [Lab 5.2](./LABS.md) - Variable Validation
4. Complete [Lab 5.3](./LABS.md) - Outputs and Locals
5. Update [CHECKLIST.md](../CHECKLIST.md)

**Next Chapter:** [Chapter 6: Provisioners and Null Resources](../chapter-06/)
