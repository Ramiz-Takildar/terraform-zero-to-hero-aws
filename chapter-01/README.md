# Chapter 1: Terraform Basics & HCL

## 📚 Learning Objectives

By the end of this chapter, you will:
- Understand Infrastructure as Code (IaC) concepts
- Learn Terraform architecture and workflow
- Master HCL (HashiCorp Configuration Language) syntax
- Work with variables, outputs, and data sources
- Understand Terraform state management basics
- Apply Terraform best practices

**Prerequisites:** Basic command line knowledge  
**Estimated Time:** 2 days  
**Labs:** 3 hands-on exercises

---

## 🎯 What is Infrastructure as Code (IaC)?

Infrastructure as Code is the practice of managing and provisioning infrastructure through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools.

### Traditional vs IaC Approach

```
Traditional Approach:
┌─────────────────────────────────────────────┐
│  Manual Process                             │
│  1. Log into AWS Console                    │
│  2. Click through UI                        │
│  3. Configure settings manually             │
│  4. Repeat for each environment             │
│  5. Document changes (maybe)                │
│                                             │
│  Problems:                                  │
│  ❌ Time consuming                          │
│  ❌ Error prone                             │
│  ❌ Hard to replicate                       │
│  ❌ No version control                      │
│  ❌ Difficult to audit                      │
└─────────────────────────────────────────────┘

IaC Approach:
┌─────────────────────────────────────────────┐
│  Automated Process                          │
│  1. Write infrastructure code               │
│  2. Version control in Git                  │
│  3. Review changes (PR)                     │
│  4. Apply with single command               │
│  5. Automatic documentation                 │
│                                             │
│  Benefits:                                  │
│  ✅ Fast and consistent                     │
│  ✅ Repeatable                              │
│  ✅ Version controlled                      │
│  ✅ Auditable                               │
│  ✅ Collaborative                           │
└─────────────────────────────────────────────┘
```

### IaC Benefits

| Benefit | Description |
|---------|-------------|
| **Speed** | Deploy infrastructure in minutes, not hours |
| **Consistency** | Same code = same infrastructure every time |
| **Version Control** | Track all changes in Git |
| **Collaboration** | Team can review and approve changes |
| **Documentation** | Code serves as documentation |
| **Cost Reduction** | Automate repetitive tasks |
| **Disaster Recovery** | Rebuild infrastructure quickly |

---

## 🏗️ What is Terraform?

Terraform is an open-source Infrastructure as Code tool created by HashiCorp that allows you to define and provision infrastructure using a declarative configuration language.

### Terraform Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Terraform Core                     │
│                                                     │
│  ┌───────────────────────────────────────────────┐ │
│  │         Configuration Files (.tf)             │ │
│  │         - main.tf                             │ │
│  │         - variables.tf                        │ │
│  │         - outputs.tf                          │ │
│  └─────────────────┬─────────────────────────────┘ │
│                    │                               │
│                    ▼                               │
│  ┌───────────────────────────────────────────────┐ │
│  │         Terraform Core Engine                 │ │
│  │         - Parse configuration                 │ │
│  │         - Build dependency graph              │ │
│  │         - Plan execution                      │ │
│  └─────────────────┬─────────────────────────────┘ │
│                    │                               │
│                    ▼                               │
│  ┌───────────────────────────────────────────────┐ │
│  │              Providers                        │ │
│  │  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐     │ │
│  │  │ AWS  │  │Azure │  │ GCP  │  │K8s   │     │ │
│  │  └──────┘  └──────┘  └──────┘  └──────┘     │ │
│  └─────────────────┬─────────────────────────────┘ │
└────────────────────┼───────────────────────────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │   Cloud Providers     │
         │   - AWS               │
         │   - Azure             │
         │   - GCP               │
         │   - etc.              │
         └───────────────────────┘
```

### Key Features

1. **Declarative Syntax:** Describe what you want, not how to create it
2. **Multi-Cloud:** Works with AWS, Azure, GCP, and 1000+ providers
3. **State Management:** Tracks current infrastructure state
4. **Plan Before Apply:** Preview changes before execution
5. **Resource Graph:** Understands dependencies
6. **Modular:** Reusable components

---

## 🔄 Terraform Workflow

### The Core Workflow

```
┌─────────────────────────────────────────────────────┐
│                                                     │
│  1. WRITE                                           │
│     ┌─────────────────────────────────────┐        │
│     │  Write .tf configuration files      │        │
│     │  Define desired infrastructure      │        │
│     └─────────────────┬───────────────────┘        │
│                       │                            │
│                       ▼                            │
│  2. INIT                                            │
│     ┌─────────────────────────────────────┐        │
│     │  terraform init                     │        │
│     │  - Download providers               │        │
│     │  - Initialize backend               │        │
│     │  - Download modules                 │        │
│     └─────────────────┬───────────────────┘        │
│                       │                            │
│                       ▼                            │
│  3. PLAN                                            │
│     ┌─────────────────────────────────────┐        │
│     │  terraform plan                     │        │
│     │  - Read configuration               │        │
│     │  - Query current state              │        │
│     │  - Calculate differences            │        │
│     │  - Show execution plan              │        │
│     └─────────────────┬───────────────────┘        │
│                       │                            │
│                       ▼                            │
│  4. APPLY                                           │
│     ┌─────────────────────────────────────┐        │
│     │  terraform apply                    │        │
│     │  - Execute the plan                 │        │
│     │  - Create/modify/delete resources   │        │
│     │  - Update state file                │        │
│     └─────────────────┬───────────────────┘        │
│                       │                            │
│                       ▼                            │
│  5. DESTROY (Optional)                              │
│     ┌─────────────────────────────────────┐        │
│     │  terraform destroy                  │        │
│     │  - Plan destruction                 │        │
│     │  - Delete all resources             │        │
│     │  - Clean state                      │        │
│     └─────────────────────────────────────┘        │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### Command Reference

| Command | Purpose | When to Use |
|---------|---------|-------------|
| `terraform init` | Initialize working directory | First time, after adding providers |
| `terraform validate` | Check syntax | Before plan |
| `terraform fmt` | Format code | Before commit |
| `terraform plan` | Preview changes | Before apply |
| `terraform apply` | Create infrastructure | After reviewing plan |
| `terraform destroy` | Delete infrastructure | Cleanup |
| `terraform show` | Display state | Inspect current state |
| `terraform output` | Show outputs | Get values |

---

## 📝 HCL (HashiCorp Configuration Language)

### Basic Syntax

```hcl
# Block structure
<BLOCK_TYPE> "<BLOCK_LABEL>" "<BLOCK_NAME>" {
  # Block body
  <IDENTIFIER> = <EXPRESSION>
}

# Example: Resource block
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}
```

### Block Types

```hcl
# 1. Terraform Block - Configure Terraform itself
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# 2. Provider Block - Configure providers
provider "aws" {
  region = "us-east-1"
}

# 3. Resource Block - Define infrastructure
resource "aws_instance" "example" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# 4. Data Block - Query existing resources
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
}

# 5. Variable Block - Define inputs
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

# 6. Output Block - Define outputs
output "instance_ip" {
  description = "Public IP of instance"
  value       = aws_instance.example.public_ip
}

# 7. Locals Block - Define local values
locals {
  common_tags = {
    Environment = "dev"
    ManagedBy   = "Terraform"
  }
}

# 8. Module Block - Call reusable modules
module "vpc" {
  source = "./modules/vpc"
  
  cidr_block = "10.0.0.0/16"
}
```

### Data Types

```hcl
# String
variable "region" {
  type    = string
  default = "us-east-1"
}

# Number
variable "instance_count" {
  type    = number
  default = 3
}

# Boolean
variable "enable_monitoring" {
  type    = bool
  default = true
}

# List
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b"]
}

# Map
variable "tags" {
  type = map(string)
  default = {
    Environment = "dev"
    Project     = "demo"
  }
}

# Object
variable "instance_config" {
  type = object({
    instance_type = string
    ami           = string
    monitoring    = bool
  })
  default = {
    instance_type = "t2.micro"
    ami           = "ami-12345678"
    monitoring    = true
  }
}
```

### Expressions and Functions

```hcl
# String interpolation
resource "aws_instance" "web" {
  tags = {
    Name = "${var.project_name}-web-server"
  }
}

# Conditional expression
resource "aws_instance" "web" {
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
}

# For expression
locals {
  subnet_ids = [for s in aws_subnet.private : s.id]
}

# Built-in functions
locals {
  # String functions
  upper_name = upper(var.project_name)
  
  # Collection functions
  subnet_count = length(var.subnet_cidrs)
  
  # Encoding functions
  user_data = base64encode(file("user-data.sh"))
  
  # Date/Time functions
  timestamp = timestamp()
}
```

---

## 📦 Variables

### Variable Definition

```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
  
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium"], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, or t2.medium."
  }
}
```

### Variable Precedence (Highest to Lowest)

```
1. Command line flags:        -var="key=value"
2. *.auto.tfvars files:       terraform.auto.tfvars
3. terraform.tfvars file:     terraform.tfvars
4. Environment variables:     TF_VAR_key=value
5. Default value:             default = "value"
```

### Variable Files

```hcl
# terraform.tfvars
instance_type = "t2.small"
environment   = "production"
region        = "us-east-1"

tags = {
  Owner   = "DevOps Team"
  Project = "WebApp"
}
```

---

## 📤 Outputs

### Output Definition

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP address"
  value       = aws_instance.web.public_ip
  sensitive   = false
}

output "db_password" {
  description = "Database password"
  value       = aws_db_instance.main.password
  sensitive   = true  # Won't show in logs
}
```

### Using Outputs

```bash
# View all outputs
terraform output

# View specific output
terraform output instance_id

# JSON format
terraform output -json

# Use in scripts
INSTANCE_IP=$(terraform output -raw instance_public_ip)
```

---

## 🗄️ State Management

### What is State?

Terraform state is a JSON file that maps your configuration to real-world resources.

```
┌─────────────────────────────────────────────┐
│         terraform.tfstate                   │
│                                             │
│  {                                          │
│    "version": 4,                            │
│    "resources": [                           │
│      {                                      │
│        "type": "aws_instance",              │
│        "name": "web",                       │
│        "instances": [{                      │
│          "attributes": {                    │
│            "id": "i-1234567890",            │
│            "public_ip": "54.123.45.67"      │
│          }                                  │
│        }]                                   │
│      }                                      │
│    ]                                        │
│  }                                          │
└─────────────────────────────────────────────┘
```

### State Best Practices

| Practice | Why |
|----------|-----|
| **Remote State** | Enable team collaboration |
| **State Locking** | Prevent concurrent modifications |
| **Encryption** | Protect sensitive data |
| **Backup** | Prevent data loss |
| **Never Edit Manually** | Avoid corruption |

---

## 🔍 Data Sources

Data sources allow Terraform to query information from existing resources.

```hcl
# Query latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use the data source
resource "aws_instance" "web" {
  ami = data.aws_ami.amazon_linux_2.id
  # ...
}
```

---

## 🎨 Best Practices

### File Organization

```
project/
├── main.tf           # Main resources
├── variables.tf      # Variable definitions
├── outputs.tf        # Output definitions
├── terraform.tfvars  # Variable values
├── providers.tf      # Provider configuration
└── versions.tf       # Version constraints
```

### Naming Conventions

```hcl
# Use descriptive names
resource "aws_instance" "web_server" {  # ✅ Good
  # ...
}

resource "aws_instance" "i1" {  # ❌ Bad
  # ...
}

# Use underscores, not hyphens
resource "aws_instance" "web_server" {  # ✅ Good
  # ...
}

resource "aws_instance" "web-server" {  # ❌ Bad
  # ...
}
```

### Comments

```hcl
# Single line comment

/*
  Multi-line
  comment
*/

resource "aws_instance" "web" {
  ami           = "ami-12345678"  # Inline comment
  instance_type = "t2.micro"
}
```

---

## 📊 Complete Example

```hcl
# versions.tf
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# providers.tf
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = local.common_tags
  }
}

# variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

# locals.tf
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = "learning"
  }
}

# main.tf
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.environment}-web-server"
    }
  )
}

# outputs.tf
output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP address"
  value       = aws_instance.web.public_ip
}
```

---

## 📖 Key Takeaways

1. **IaC Benefits:** Speed, consistency, version control, collaboration
2. **Terraform Workflow:** Write → Init → Plan → Apply → Destroy
3. **HCL Syntax:** Declarative, human-readable configuration language
4. **Variables:** Make configurations flexible and reusable
5. **Outputs:** Export values for use elsewhere
6. **Data Sources:** Query existing infrastructure
7. **State:** Maps configuration to real resources
8. **Best Practices:** Organize files, use meaningful names, comment code

---

## 🔗 Next Steps

1. Review the concepts above
2. Complete [Lab 1.1](./LABS.md) - First Terraform Configuration
3. Complete [Lab 1.2](./LABS.md) - Variables and Outputs
4. Complete [Lab 1.3](./LABS.md) - Data Sources
5. Review [Interview Questions](./INTERVIEW.md)
6. Update [CHECKLIST.md](../CHECKLIST.md)

**Next Chapter:** [Chapter 2: AWS Fundamentals](../chapter-02/)

---

## 📚 Additional Resources

- [Terraform Documentation](https://www.terraform.io/docs)
- [HCL Syntax Reference](https://www.terraform.io/docs/language/syntax/configuration.html)
- [Terraform Registry](https://registry.terraform.io/)
- [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)