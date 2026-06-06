# Chapter 4: Labs - Modules

## 🎯 Lab Overview

These hands-on labs will help you master Terraform modules, from creating custom modules to using public modules from the Terraform Registry.

**Prerequisites:**
- AWS account with appropriate permissions
- AWS CLI configured
- Terraform installed (v1.0+)
- Completed Chapters 1-3 labs

**Estimated Time:** 2-3 hours total

---

## 🧪 Lab 4.1: Create a Custom VPC Module

**Objective:** Build a reusable VPC module with public and private subnets.

**Duration:** 45 minutes

### Architecture

```
┌─────────────────────────────────────────────────────┐
│              VPC Module                             │
│                                                     │
│  Inputs:                                            │
│  - name                                             │
│  - cidr_block                                       │
│  - availability_zones                               │
│  - public_subnet_cidrs                              │
│  - private_subnet_cidrs                             │
│                                                     │
│  Resources:                                         │
│  - VPC                                              │
│  - Public Subnets (2)                               │
│  - Private Subnets (2)                              │
│  - Internet Gateway                                 │
│  - NAT Gateways (2)                                 │
│  - Route Tables                                     │
│                                                     │
│  Outputs:                                           │
│  - vpc_id                                           │
│  - public_subnet_ids                                │
│  - private_subnet_ids                               │
└─────────────────────────────────────────────────────┘
```

### Step 1: Create Module Structure

```bash
mkdir -p lab-4.1-custom-module/modules/vpc
cd lab-4.1-custom-module
```

### Step 2: Create Module Files

**modules/vpc/main.tf:**

```hcl
# VPC
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

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-igw"
    }
  )
}

# Public Subnets
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

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-private-${count.index + 1}"
      Type = "private"
    }
  )
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count  = var.enable_nat_gateway ? length(var.public_subnet_cidrs) : 0
  domain = "vpc"
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-nat-eip-${count.index + 1}"
    }
  )
  
  depends_on = [aws_internet_gateway.main]
}

# NAT Gateways
resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? length(var.public_subnet_cidrs) : 0
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-nat-${count.index + 1}"
    }
  )
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-public-rt"
    }
  )
}

# Public Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(var.public_subnet_cidrs)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Route Tables
resource "aws_route_table" "private" {
  count = var.enable_nat_gateway ? length(var.private_subnet_cidrs) : 0
  
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  
  tags = merge(
    var.tags,
    {
      Name = "${var.name}-private-rt-${count.index + 1}"
    }
  )
}

# Private Route Table Associations
resource "aws_route_table_association" "private" {
  count = var.enable_nat_gateway ? length(var.private_subnet_cidrs) : 0
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

**modules/vpc/variables.tf:**

```hcl
variable "name" {
  description = "Name prefix for all resources"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
  
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  
  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones required."
  }
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateway for private subnets"
  type        = bool
  default     = true
}

variable "enable_dns_hostnames" {
  description = "Enable DNS hostnames in VPC"
  type        = bool
  default     = true
}

variable "enable_dns_support" {
  description = "Enable DNS support in VPC"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Additional tags for all resources"
  type        = map(string)
  default     = {}
}
```

**modules/vpc/outputs.tf:**

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
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "public_subnet_cidrs" {
  description = "List of public subnet CIDR blocks"
  value       = aws_subnet.public[*].cidr_block
}

output "private_subnet_cidrs" {
  description = "List of private subnet CIDR blocks"
  value       = aws_subnet.private[*].cidr_block
}

output "internet_gateway_id" {
  description = "Internet Gateway ID"
  value       = aws_internet_gateway.main.id
}

output "nat_gateway_ids" {
  description = "List of NAT Gateway IDs"
  value       = aws_nat_gateway.main[*].id
}

output "nat_gateway_ips" {
  description = "List of NAT Gateway public IPs"
  value       = aws_eip.nat[*].public_ip
}
```

**modules/vpc/versions.tf:**

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

**modules/vpc/README.md:**

```markdown
# VPC Module

Creates a VPC with public and private subnets across multiple availability zones.

## Features

- VPC with configurable CIDR block
- Public subnets with Internet Gateway
- Private subnets with NAT Gateways
- Automatic route table configuration
- DNS support enabled by default

## Usage

```hcl
module "vpc" {
  source = "./modules/vpc"
  
  name                 = "production"
  cidr_block           = "10.0.0.0/16"
  availability_zones   = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]
  
  tags = {
    Environment = "production"
  }
}
```

## Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|----------|
| name | Name prefix | string | n/a | yes |
| cidr_block | VPC CIDR | string | "10.0.0.0/16" | no |
| availability_zones | AZs | list(string) | n/a | yes |
| public_subnet_cidrs | Public subnet CIDRs | list(string) | n/a | yes |
| private_subnet_cidrs | Private subnet CIDRs | list(string) | n/a | yes |
| enable_nat_gateway | Enable NAT Gateway | bool | true | no |

## Outputs

| Name | Description |
|------|-------------|
| vpc_id | VPC ID |
| public_subnet_ids | Public subnet IDs |
| private_subnet_ids | Private subnet IDs |
```

### Step 3: Use the Module

**main.tf:**

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

# Use the VPC module
module "vpc" {
  source = "./modules/vpc"
  
  name                 = "lab-vpc"
  cidr_block           = "10.0.0.0/16"
  availability_zones   = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]
  enable_nat_gateway   = true
  
  tags = {
    Environment = "lab"
    Lab         = "4.1"
  }
}

# Use module outputs
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "public_subnets" {
  value = module.vpc.public_subnet_ids
}

output "private_subnets" {
  value = module.vpc.private_subnet_ids
}

output "nat_gateway_ips" {
  value = module.vpc.nat_gateway_ips
}
```

### Step 4: Deploy

```bash
# Initialize
terraform init

# Validate module
terraform validate

# Plan
terraform plan

# Apply
terraform apply -auto-approve

# View outputs
terraform output
```

### Step 5: Test Module Reusability

Create a second VPC using the same module:

```hcl
# Add to main.tf
module "vpc_dev" {
  source = "./modules/vpc"
  
  name                 = "dev-vpc"
  cidr_block           = "10.1.0.0/16"
  availability_zones   = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs  = ["10.1.1.0/24", "10.1.2.0/24"]
  private_subnet_cidrs = ["10.1.11.0/24", "10.1.12.0/24"]
  enable_nat_gateway   = false  # Save costs in dev
  
  tags = {
    Environment = "dev"
  }
}

output "dev_vpc_id" {
  value = module.vpc_dev.vpc_id
}
```

```bash
terraform apply -auto-approve
```

### Step 6: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] Module structure created correctly
- [ ] Module has clear inputs and outputs
- [ ] Module includes validation
- [ ] Module is well-documented
- [ ] Module can be reused multiple times
- [ ] All resources created successfully

---

## 🧪 Lab 4.2: Use Public Modules from Terraform Registry

**Objective:** Use official AWS modules from Terraform Registry.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-4.2-public-modules
cd lab-4.2-public-modules
```

### Step 2: Use AWS VPC Module

**main.tf:**

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

# Official AWS VPC Module
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = "lab-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = true  # Cost optimization
  enable_dns_hostnames = true
  
  tags = {
    Environment = "lab"
    Lab         = "4.2"
  }
  
  vpc_tags = {
    Name = "lab-vpc"
  }
}

# Official AWS Security Group Module
module "web_security_group" {
  source  = "terraform-aws-modules/security-group/aws"
  version = "5.0.0"
  
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = module.vpc.vpc_id
  
  ingress_cidr_blocks = ["0.0.0.0/0"]
  ingress_rules       = ["http-80-tcp", "https-443-tcp"]
  egress_rules        = ["all-all"]
  
  tags = {
    Name = "web-sg"
  }
}

# Official AWS EC2 Instance Module
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

module "ec2_instance" {
  source  = "terraform-aws-modules/ec2-instance/aws"
  version = "5.0.0"
  
  name = "web-server"
  
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  vpc_security_group_ids = [module.web_security_group.security_group_id]
  subnet_id              = module.vpc.public_subnets[0]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from Terraform Public Modules!</h1>" > /var/www/html/index.html
              EOF
  
  tags = {
    Name = "web-server"
  }
}

# Official AWS S3 Bucket Module
module "s3_bucket" {
  source  = "terraform-aws-modules/s3-bucket/aws"
  version = "3.15.0"
  
  bucket = "lab-terraform-${data.aws_caller_identity.current.account_id}"
  
  versioning = {
    enabled = true
  }
  
  server_side_encryption_configuration = {
    rule = {
      apply_server_side_encryption_by_default = {
        sse_algorithm = "AES256"
      }
    }
  }
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
  
  tags = {
    Name = "lab-bucket"
  }
}

data "aws_caller_identity" "current" {}
```

**outputs.tf:**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = module.vpc.vpc_id
}

output "public_subnets" {
  description = "Public subnet IDs"
  value       = module.vpc.public_subnets
}

output "private_subnets" {
  description = "Private subnet IDs"
  value       = module.vpc.private_subnets
}

output "security_group_id" {
  description = "Security group ID"
  value       = module.web_security_group.security_group_id
}

output "instance_id" {
  description = "EC2 instance ID"
  value       = module.ec2_instance.id
}

output "instance_public_ip" {
  description = "EC2 public IP"
  value       = module.ec2_instance.public_ip
}

output "web_url" {
  description = "Web server URL"
  value       = "http://${module.ec2_instance.public_ip}"
}

output "s3_bucket_name" {
  description = "S3 bucket name"
  value       = module.s3_bucket.s3_bucket_id
}
```

### Step 3: Deploy

```bash
# Initialize (downloads modules)
terraform init

# Review what will be created
terraform plan

# Apply
terraform apply -auto-approve

# Get web URL
terraform output web_url
```

### Step 4: Test

```bash
# Wait 2-3 minutes for user data to complete
sleep 180

# Test web server
curl $(terraform output -raw web_url)

# Or open in browser
open $(terraform output -raw web_url)

# Test S3 bucket
aws s3 ls s3://$(terraform output -raw s3_bucket_name)
```

### Step 5: Explore Module Sources

```bash
# View downloaded modules
ls -la .terraform/modules/

# View module source
cat .terraform/modules/modules.json | jq
```

### Step 6: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] Public modules downloaded successfully
- [ ] VPC created with all components
- [ ] Security group configured correctly
- [ ] EC2 instance accessible via HTTP
- [ ] S3 bucket created with encryption
- [ ] All module outputs working

---

## 🧪 Lab 4.3: Module Composition and Nested Modules

**Objective:** Create a complete application stack using nested modules.

**Duration:** 45 minutes

### Architecture

```
┌─────────────────────────────────────────────────────┐
│              Application Module                     │
│                                                     │
│  ┌─────────────────────────────────────────────┐   │
│  │         VPC Module                          │   │
│  │  - VPC, Subnets, IGW, NAT                   │   │
│  └─────────────────────────────────────────────┘   │
│                      │                             │
│                      ▼                             │
│  ┌─────────────────────────────────────────────┐   │
│  │      Security Module                        │   │
│  │  - Web SG, App SG, DB SG                    │   │
│  └─────────────────────────────────────────────┘   │
│                      │                             │
│                      ▼                             │
│  ┌─────────────────────────────────────────────┐   │
│  │       Compute Module                        │   │
│  │  - EC2 Instances, ALB                       │   │
│  └─────────────────────────────────────────────┘   │
│                      │                             │
│                      ▼                             │
│  ┌─────────────────────────────────────────────┐   │
│  │      Database Module                        │   │
│  │  - RDS Instance                             │   │
│  └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Step 1: Create Module Structure

```bash
mkdir -p lab-4.3-composition/modules/{networking,security,compute,application}
cd lab-4.3-composition
```

### Step 2: Create Networking Module

**modules/networking/main.tf:**

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.0.0"
  
  name = var.name
  cidr = var.cidr_block
  
  azs             = var.availability_zones
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs
  
  enable_nat_gateway = var.enable_nat_gateway
  single_nat_gateway = var.single_nat_gateway
  enable_dns_hostnames = true
  
  tags = var.tags
}
```

**modules/networking/variables.tf:**

```hcl
variable "name" {
  description = "Name prefix"
  type        = string
}

variable "cidr_block" {
  description = "VPC CIDR block"
  type        = string
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

variable "public_subnet_cidrs" {
  description = "Public subnet CIDRs"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "Private subnet CIDRs"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateway"
  type        = bool
  default     = true
}

variable "single_nat_gateway" {
  description = "Use single NAT Gateway"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Tags"
  type        = map(string)
  default     = {}
}
```

**modules/networking/outputs.tf:**

```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "public_subnet_ids" {
  value = module.vpc.public_subnets
}

output "private_subnet_ids" {
  value = module.vpc.private_subnets
}
```

### Step 3: Create Security Module

**modules/security/main.tf:**

```hcl
resource "aws_security_group" "web" {
  name        = "${var.name}-web-sg"
  description = "Security group for web tier"
  vpc_id      = var.vpc_id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(var.tags, {
    Name = "${var.name}-web-sg"
  })
}

resource "aws_security_group" "app" {
  name        = "${var.name}-app-sg"
  description = "Security group for app tier"
  vpc_id      = var.vpc_id
  
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(var.tags, {
    Name = "${var.name}-app-sg"
  })
}
```

**modules/security/variables.tf:**

```hcl
variable "name" {
  description = "Name prefix"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID"
  type        = string
}

variable "tags" {
  description = "Tags"
  type        = map(string)
  default     = {}
}
```

**modules/security/outputs.tf:**

```hcl
output "web_sg_id" {
  value = aws_security_group.web.id
}

output "app_sg_id" {
  value = aws_security_group.app.id
}
```

### Step 4: Create Compute Module

**modules/compute/main.tf:**

```hcl
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  count = var.instance_count
  
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = var.instance_type
  subnet_id              = var.subnet_ids[count.index % length(var.subnet_ids)]
  vpc_security_group_ids = [var.security_group_id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>${var.name} - Instance ${count.index + 1}</h1>" > /var/www/html/index.html
              EOF
  
  tags = merge(var.tags, {
    Name = "${var.name}-web-${count.index + 1}"
  })
}
```

**modules/compute/variables.tf:**

```hcl
variable "name" {
  description = "Name prefix"
  type        = string
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 2
}

variable "instance_type" {
  description = "Instance type"
  type        = string
  default     = "t2.micro"
}

variable "subnet_ids" {
  description = "Subnet IDs"
  type        = list(string)
}

variable "security_group_id" {
  description = "Security group ID"
  type        = string
}

variable "tags" {
  description = "Tags"
  type        = map(string)
  default     = {}
}
```

**modules/compute/outputs.tf:**

```hcl
output "instance_ids" {
  value = aws_instance.web[*].id
}

output "instance_ips" {
  value = aws_instance.web[*].public_ip
}
```

### Step 5: Create Application Module (Composition)

**modules/application/main.tf:**

```hcl
module "networking" {
  source = "../networking"
  
  name                 = var.name
  cidr_block           = var.vpc_cidr
  availability_zones   = var.availability_zones
  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs
  single_nat_gateway   = var.single_nat_gateway
  
  tags = var.tags
}

module "security" {
  source = "../security"
  
  name   = var.name
  vpc_id = module.networking.vpc_id
  
  tags = var.tags
}

module "compute" {
  source = "../compute"
  
  name              = var.name
  instance_count    = var.instance_count
  instance_type     = var.instance_type
  subnet_ids        = module.networking.public_subnet_ids
  security_group_id = module.security.web_sg_id
  
  tags = var.tags
  
  depends_on = [module.networking, module.security]
}
```

**modules/application/variables.tf:**

```hcl
variable "name" {
  description = "Application name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
}

variable "public_subnet_cidrs" {
  description = "Public subnet CIDRs"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "Private subnet CIDRs"
  type        = list(string)
}

variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 2
}

variable "instance_type" {
  description = "Instance type"
  type        = string
  default     = "t2.micro"
}

variable "single_nat_gateway" {
  description = "Use single NAT Gateway"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Tags"
  type        = map(string)
  default     = {}
}
```

**modules/application/outputs.tf:**

```hcl
output "vpc_id" {
  value = module.networking.vpc_id
}

output "instance_ips" {
  value = module.compute.instance_ips
}
```

### Step 6: Use Application Module

**main.tf:**

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

module "app" {
  source = "./modules/application"
  
  name                 = "myapp"
  vpc_cidr             = "10.0.0.0/16"
  availability_zones   = ["us-east-1a", "us-east-1b"]
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.11.0/24", "10.0.12.0/24"]
  instance_count       = 2
  instance_type        = "t2.micro"
  single_nat_gateway   = true
  
  tags = {
    Environment = "lab"
    Lab         = "4.3"
  }
}

output "vpc_id" {
  value = module.app.vpc_id
}

output "instance_ips" {
  value = module.app.instance_ips
}
```

### Step 7: Deploy

```bash
terraform init
terraform apply -auto-approve
```

### Step 8: Test

```bash
# Get instance IPs
terraform output instance_ips

# Test each instance
for ip in $(terraform output -json instance_ips | jq -r '.[]'); do
  echo "Testing $ip..."
  curl http://$ip
done
```

### Step 9: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] Nested modules created successfully
- [ ] Module composition works correctly
- [ ] Dependencies handled properly
- [ ] All tiers deployed (networking, security, compute)
- [ ] Instances accessible
- [ ] Clean module structure

---

## 🎓 Lab Summary

### What You Learned

1. **Lab 4.1:** Created custom reusable VPC module
2. **Lab 4.2:** Used public modules from Terraform Registry
3. **Lab 4.3:** Composed complex infrastructure with nested modules

### Key Concepts Practiced

- Module structure and organization
- Input variables and outputs
- Module validation
- Module documentation
- Public module usage
- Module composition
- Nested module dependencies

### Next Steps

1. ✅ Mark labs complete in [CHECKLIST.md](../CHECKLIST.md)
2. 📝 Review [INTERVIEW.md](./INTERVIEW.md) questions
3. 🚀 Proceed to [Chapter 5](../chapter-05/)

---

## 💡 Troubleshooting Tips

### Module Not Found

```bash
# Reinitialize to download modules
terraform init -upgrade
```

### Module Version Conflicts

```bash
# Check module versions
terraform version

# Update modules
terraform init -upgrade
```

### Circular Dependencies

```bash
# Use depends_on to break cycles
module "app" {
  source = "./modules/app"
  
  depends_on = [module.database]
}
```

---

**🎉 Congratulations!** You've mastered Terraform modules!
