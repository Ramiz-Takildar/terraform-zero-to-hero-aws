# Chapter 1 Labs: Terraform Basics & HCL

## Overview

These labs will help you understand Terraform fundamentals through hands-on practice. Each lab includes objectives, step-by-step instructions, and verification steps.

**Prerequisites:** 
- AWS Account (Free Tier)
- Terraform installed (v1.0+)
- AWS CLI configured
- Text editor

---

## Lab 1.1: Your First Terraform Configuration

### Learning Objectives
- Write your first Terraform configuration
- Understand the terraform workflow
- Create an EC2 instance in AWS

### Theory

Terraform follows a simple workflow:
1. **Write** - Define infrastructure in .tf files
2. **Init** - Initialize working directory
3. **Plan** - Preview changes
4. **Apply** - Create infrastructure
5. **Destroy** - Clean up resources

### Steps

#### Step 1: Create Project Directory

```bash
mkdir terraform-lab-01
cd terraform-lab-01
```

#### Step 2: Create main.tf

Create file `main.tf`:

```hcl
# Configure Terraform
terraform {
  required_version = ">= 1.0"
  
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure AWS Provider
provider "aws" {
  region = "us-east-1"
}

# Create EC2 Instance
resource "aws_instance" "my_first_instance" {
  ami           = "ami-0c55b159cbfafe1f0"  # Amazon Linux 2
  instance_type = "t2.micro"
  
  tags = {
    Name        = "MyFirstTerraformInstance"
    Environment = "Learning"
    ManagedBy   = "Terraform"
  }
}

# Output the instance details
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.my_first_instance.id
}

output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.my_first_instance.public_ip
}
```

#### Step 3: Initialize Terraform

```bash
terraform init
```

**Expected Output:**
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...

Terraform has been successfully initialized!
```

**What happened:**
- Downloaded AWS provider plugin
- Created `.terraform/` directory
- Created `.terraform.lock.hcl` file

#### Step 4: Validate Configuration

```bash
terraform validate
```

**Expected Output:**
```
Success! The configuration is valid.
```

#### Step 5: Format Code

```bash
terraform fmt
```

This formats all `.tf` files to standard style.

#### Step 6: Plan Infrastructure

```bash
terraform plan
```

**Expected Output:**
```
Terraform will perform the following actions:

  # aws_instance.my_first_instance will be created
  + resource "aws_instance" "my_first_instance" {
      + ami                          = "ami-0c55b159cbfafe1f0"
      + instance_type                = "t2.micro"
      + tags                         = {
          + "Environment" = "Learning"
          + "ManagedBy"   = "Terraform"
          + "Name"        = "MyFirstTerraformInstance"
        }
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

**What to observe:**
- `+` means resource will be created
- Shows all attributes that will be set
- Shows computed values (marked as `(known after apply)`)

#### Step 7: Apply Configuration

```bash
terraform apply
```

Type `yes` when prompted.

**Expected Output:**
```
aws_instance.my_first_instance: Creating...
aws_instance.my_first_instance: Still creating... [10s elapsed]
aws_instance.my_first_instance: Creation complete after 45s [id=i-1234567890abcdef0]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

Outputs:

instance_id = "i-1234567890abcdef0"
instance_public_ip = "54.123.45.67"
```

#### Step 8: Verify in AWS Console

1. Log into AWS Console
2. Navigate to EC2 → Instances
3. Find instance with tag `Name = MyFirstTerraformInstance`
4. Verify it's running

#### Step 9: Inspect State

```bash
terraform show
```

This displays the current state in human-readable format.

```bash
# View state file (JSON)
cat terraform.tfstate
```

#### Step 10: Destroy Resources

```bash
terraform destroy
```

Type `yes` when prompted.

**Expected Output:**
```
aws_instance.my_first_instance: Destroying... [id=i-1234567890abcdef0]
aws_instance.my_first_instance: Still destroying... [10s elapsed]
aws_instance.my_first_instance: Destruction complete after 45s

Destroy complete! Resources: 1 destroyed.
```

### Verification Checklist

- [ ] `terraform init` completed successfully
- [ ] `terraform validate` shows no errors
- [ ] `terraform plan` shows 1 resource to add
- [ ] `terraform apply` created EC2 instance
- [ ] Instance visible in AWS Console
- [ ] Outputs displayed instance ID and IP
- [ ] `terraform destroy` removed all resources

### Troubleshooting

**Error: "No valid credential sources found"**
```bash
# Configure AWS CLI
aws configure
# Enter: Access Key ID, Secret Access Key, Region (us-east-1)
```

**Error: "AMI not found"**
```bash
# AMI IDs are region-specific, update to your region's AMI
# Or use data source (Lab 1.3)
```

### Cleanup

```bash
cd ..
rm -rf terraform-lab-01
```

---

## Lab 1.2: Variables and Outputs

### Learning Objectives
- Use input variables for flexibility
- Implement variable validation
- Export values with outputs
- Use variable files

### Steps

#### Step 1: Create Project Structure

```bash
mkdir terraform-lab-02
cd terraform-lab-02
```

#### Step 2: Create variables.tf

Create file `variables.tf`:

```hcl
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
  
  validation {
    condition     = contains(["t2.micro", "t2.small", "t3.micro"], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, or t3.micro."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "project_name" {
  description = "Project name for tagging"
  type        = string
}

variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

variable "additional_tags" {
  description = "Additional tags for resources"
  type        = map(string)
  default     = {}
}
```

#### Step 3: Create main.tf

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

# Local values for computed tags
locals {
  common_tags = merge(
    {
      Environment = var.environment
      Project     = var.project_name
      ManagedBy   = "Terraform"
    },
    var.additional_tags
  )
}

resource "aws_instance" "app" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
  monitoring    = var.enable_monitoring
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-${var.environment}-instance"
    }
  )
}
```

#### Step 4: Create outputs.tf

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.app.id
}

output "instance_public_ip" {
  description = "Public IP address"
  value       = aws_instance.app.public_ip
}

output "instance_arn" {
  description = "ARN of the instance"
  value       = aws_instance.app.arn
}

output "instance_tags" {
  description = "Tags applied to instance"
  value       = aws_instance.app.tags
}
```

#### Step 5: Create terraform.tfvars

```hcl
environment    = "dev"
project_name   = "myapp"
instance_type  = "t2.micro"

additional_tags = {
  Owner = "DevOps Team"
  Cost  = "Learning"
}
```

#### Step 6: Apply with Variables

```bash
terraform init
terraform plan
terraform apply
```

#### Step 7: Override Variables

```bash
# Override via command line
terraform plan -var="environment=staging" -var="instance_type=t2.small"

# Use different var file
terraform plan -var-file="prod.tfvars"
```

#### Step 8: Test Validation

```bash
# This should fail validation
terraform plan -var="environment=test"

# Expected error:
# Error: Invalid value for variable
# Environment must be dev, staging, or prod.
```

### Verification Checklist

- [ ] Variables defined with types and defaults
- [ ] Validation rules work correctly
- [ ] Outputs display expected values
- [ ] Tags include all common tags
- [ ] Variable override works

### Cleanup

```bash
terraform destroy
cd ..
rm -rf terraform-lab-02
```

---

## Lab 1.3: Data Sources

### Learning Objectives
- Query existing AWS resources
- Use data sources for dynamic values
- Find latest AMI automatically

### Steps

#### Step 1: Create Project

```bash
mkdir terraform-lab-03
cd terraform-lab-03
```

#### Step 2: Create data-sources.tf

```hcl
# Get latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Get default VPC
data "aws_vpc" "default" {
  default = true
}

# Get availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Get current AWS account ID
data "aws_caller_identity" "current" {}

# Get current region
data "aws_region" "current" {}
```

#### Step 3: Create main.tf

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

resource "aws_instance" "app" {
  # Use data source for AMI
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  # Use first available AZ
  availability_zone = data.aws_availability_zones.available.names[0]
  
  tags = {
    Name      = "DataSourceExample"
    AMI       = data.aws_ami.amazon_linux_2.name
    AccountID = data.aws_caller_identity.current.account_id
    Region    = data.aws_region.current.name
  }
}
```

#### Step 4: Create outputs.tf

```hcl
output "ami_id" {
  description = "AMI ID used"
  value       = data.aws_ami.amazon_linux_2.id
}

output "ami_name" {
  description = "AMI name"
  value       = data.aws_ami.amazon_linux_2.name
}

output "vpc_id" {
  description = "Default VPC ID"
  value       = data.aws_vpc.default.id
}

output "availability_zones" {
  description = "Available AZs"
  value       = data.aws_availability_zones.available.names
}

output "account_id" {
  description = "AWS Account ID"
  value       = data.aws_caller_identity.current.account_id
}

output "region" {
  description = "Current region"
  value       = data.aws_region.current.name
}
```

#### Step 5: Apply and Observe

```bash
terraform init
terraform plan
```

**Observe:**
- Data sources are read during plan
- AMI ID is automatically found
- No hardcoded AMI ID needed

```bash
terraform apply
```

### Verification Checklist

- [ ] Data sources query successfully
- [ ] Latest AMI found automatically
- [ ] Instance uses data source values
- [ ] Outputs show queried information

### Cleanup

```bash
terraform destroy
cd ..
rm -rf terraform-lab-03
```

---

## Completion Checklist

| Lab | Description | Status |
|-----|-------------|--------|
| 1.1 | First Terraform Configuration | [ ] |
| 1.2 | Variables and Outputs | [ ] |
| 1.3 | Data Sources | [ ] |

**Mark complete in [CHECKLIST.md](../CHECKLIST.md)**

---

## Common Issues and Solutions

### Issue 1: AWS Credentials Not Found

**Error:**
```
Error: No valid credential sources found
```

**Solution:**
```bash
aws configure
# Or set environment variables:
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_DEFAULT_REGION="us-east-1"
```

### Issue 2: State File Locked

**Error:**
```
Error: Error acquiring the state lock
```

**Solution:**
```bash
# If you're sure no other process is running:
terraform force-unlock <LOCK_ID>
```

### Issue 3: Resource Already Exists

**Error:**
```
Error: resource already exists
```

**Solution:**
```bash
# Import existing resource
terraform import aws_instance.app i-1234567890abcdef0
```

---

## Best Practices Learned

1. **Always run plan before apply**
2. **Use variables for flexibility**
3. **Add validation to variables**
4. **Use data sources instead of hardcoding**
5. **Tag all resources**
6. **Use meaningful resource names**
7. **Destroy resources after learning**
8. **Keep state file secure**

---

**Next:** [Chapter 2 Labs](../chapter-02/LABS.md)
