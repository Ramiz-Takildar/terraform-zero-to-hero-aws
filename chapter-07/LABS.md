# Chapter 7: Labs - Data Sources and Terraform Functions

## 🎯 Lab Overview

These hands-on labs will help you master Terraform data sources and built-in functions.

**Prerequisites:**
- AWS account with appropriate permissions
- AWS CLI configured
- Terraform installed (v1.0+)
- Completed Chapters 1-6 labs

**Estimated Time:** 2-3 hours total

---

## 🧪 Lab 7.1: Data Sources and Dynamic Infrastructure

**Objective:** Use data sources to query existing infrastructure and make dynamic decisions.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-7.1-data-sources
cd lab-7.1-data-sources
```

### Step 2: Create Configuration

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
  region = var.aws_region
}

# Data Sources

# Get current AWS account information
data "aws_caller_identity" "current" {}

# Get available availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

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

# Get latest Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# Get default VPC
data "aws_vpc" "default" {
  default = true
}

# Get subnets in default VPC
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Get specific subnet details
data "aws_subnet" "selected" {
  count = length(data.aws_subnets.default.ids)
  id    = data.aws_subnets.default.ids[count.index]
}

# IAM policy document for EC2 assume role
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    actions = ["sts:AssumeRole"]
    
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

# IAM policy document for S3 access
data "aws_iam_policy_document" "s3_access" {
  statement {
    sid    = "AllowS3Read"
    effect = "Allow"
    
    actions = [
      "s3:GetObject",
      "s3:ListBucket"
    ]
    
    resources = [
      "arn:aws:s3:::${var.bucket_name}",
      "arn:aws:s3:::${var.bucket_name}/*"
    ]
  }
}

# Resources using data sources

# IAM Role
resource "aws_iam_role" "instance" {
  name               = "${var.project_name}-instance-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
  
  tags = {
    Name = "${var.project_name}-instance-role"
  }
}

# IAM Policy
resource "aws_iam_role_policy" "s3_access" {
  name   = "s3-access"
  role   = aws_iam_role.instance.id
  policy = data.aws_iam_policy_document.s3_access.json
}

# IAM Instance Profile
resource "aws_iam_instance_profile" "instance" {
  name = "${var.project_name}-instance-profile"
  role = aws_iam_role.instance.name
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Security group for web servers"
  vpc_id      = data.aws_vpc.default.id
  
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-web-sg"
  }
}

# EC2 Instances - One per AZ
resource "aws_instance" "web" {
  count = min(length(data.aws_availability_zones.available.names), 3)
  
  # Use AMI based on variable
  ami           = var.os_type == "amazon-linux" ? data.aws_ami.amazon_linux_2.id : data.aws_ami.ubuntu.id
  instance_type = var.instance_type
  
  # Use subnet from data source
  subnet_id = data.aws_subnet.selected[count.index].id
  
  vpc_security_group_ids = [aws_security_group.web.id]
  iam_instance_profile   = aws_iam_instance_profile.instance.name
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y || apt-get update -y
              yum install -y httpd || apt-get install -y apache2
              systemctl start httpd || systemctl start apache2
              systemctl enable httpd || systemctl enable apache2
              
              cat > /var/www/html/index.html <<HTML
              <h1>Instance in ${data.aws_availability_zones.available.names[count.index]}</h1>
              <p>AMI: ${var.os_type == "amazon-linux" ? data.aws_ami.amazon_linux_2.id : data.aws_ami.ubuntu.id}</p>
              <p>Account: ${data.aws_caller_identity.current.account_id}</p>
              <p>Region: ${var.aws_region}</p>
              <p>VPC: ${data.aws_vpc.default.id}</p>
              <p>Subnet: ${data.aws_subnet.selected[count.index].id}</p>
              HTML
              EOF
  
  tags = {
    Name = "${var.project_name}-web-${count.index + 1}"
    AZ   = data.aws_availability_zones.available.names[count.index]
  }
}

# S3 Bucket
resource "aws_s3_bucket" "data" {
  bucket = var.bucket_name
  
  tags = {
    Name = var.bucket_name
  }
}
```

**variables.tf:**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "data-sources-lab"
}

variable "os_type" {
  description = "Operating system type"
  type        = string
  default     = "amazon-linux"
  
  validation {
    condition     = contains(["amazon-linux", "ubuntu"], var.os_type)
    error_message = "OS type must be amazon-linux or ubuntu."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

variable "bucket_name" {
  description = "S3 bucket name"
  type        = string
  default     = "my-data-sources-lab-bucket-12345"
}
```

**outputs.tf:**

```hcl
# Account Information
output "account_id" {
  description = "AWS Account ID"
  value       = data.aws_caller_identity.current.account_id
}

output "caller_arn" {
  description = "ARN of the caller"
  value       = data.aws_caller_identity.current.arn
}

# Availability Zones
output "availability_zones" {
  description = "Available AZs"
  value       = data.aws_availability_zones.available.names
}

output "az_count" {
  description = "Number of AZs"
  value       = length(data.aws_availability_zones.available.names)
}

# AMI Information
output "amazon_linux_ami" {
  description = "Amazon Linux 2 AMI"
  value = {
    id           = data.aws_ami.amazon_linux_2.id
    name         = data.aws_ami.amazon_linux_2.name
    architecture = data.aws_ami.amazon_linux_2.architecture
  }
}

output "ubuntu_ami" {
  description = "Ubuntu AMI"
  value = {
    id           = data.aws_ami.ubuntu.id
    name         = data.aws_ami.ubuntu.name
    architecture = data.aws_ami.ubuntu.architecture
  }
}

# VPC Information
output "vpc_info" {
  description = "VPC information"
  value = {
    id         = data.aws_vpc.default.id
    cidr_block = data.aws_vpc.default.cidr_block
  }
}

# Subnet Information
output "subnet_info" {
  description = "Subnet information"
  value = [
    for subnet in data.aws_subnet.selected : {
      id                = subnet.id
      cidr_block        = subnet.cidr_block
      availability_zone = subnet.availability_zone
    }
  ]
}

# Instance Information
output "instance_urls" {
  description = "URLs to access instances"
  value = [
    for instance in aws_instance.web :
    "http://${instance.public_ip}"
  ]
}

output "instances_by_az" {
  description = "Instances grouped by AZ"
  value = {
    for instance in aws_instance.web :
    instance.availability_zone => instance.public_ip...
  }
}
```

### Step 3: Deploy with Amazon Linux

```bash
terraform init
terraform plan
terraform apply -auto-approve

# View outputs
terraform output
terraform output -json | jq
```

### Step 4: Test Instances

```bash
# Get instance URLs
terraform output -json instance_urls | jq -r '.[]' | while read url; do
  echo "Testing $url"
  curl $url
  echo ""
done
```

### Step 5: Switch to Ubuntu

```bash
# Change OS type
terraform apply -var="os_type=ubuntu" -auto-approve

# Verify new AMI is used
terraform output ubuntu_ami
```

### Step 6: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] Data sources query existing infrastructure
- [ ] AMI data sources return latest images
- [ ] Availability zones retrieved dynamically
- [ ] VPC and subnet information fetched
- [ ] IAM policy documents generated
- [ ] Instances created in multiple AZs
- [ ] OS type switchable via variable

---

## 🧪 Lab 7.2: String and Collection Functions

**Objective:** Master Terraform's built-in functions for string and collection manipulation.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-7.2-functions
cd lab-7.2-functions
```

### Step 2: Create Configuration

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

# String Functions
locals {
  # Basic string operations
  project_name_upper = upper(var.project_name)
  project_name_lower = lower(var.project_name)
  project_name_title = title(var.project_name)
  
  # String manipulation
  sanitized_name = replace(lower(var.project_name), " ", "-")
  bucket_name    = "${local.sanitized_name}-${formatdate("YYYYMMDD", timestamp())}"
  
  # String splitting and joining
  name_parts = split("-", local.sanitized_name)
  name_joined = join("_", local.name_parts)
  
  # Substring operations
  short_name = substr(local.sanitized_name, 0, min(length(local.sanitized_name), 10))
  
  # Format strings
  instance_names = formatlist("%s-instance-%02d", [local.sanitized_name], range(1, 4))
  
  # Regex operations
  version_match = regex("v([0-9.]+)", var.app_version)
  version_number = local.version_match[0]
}

# Collection Functions
locals {
  # List operations
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  az_count = length(local.availability_zones)
  first_az = element(local.availability_zones, 0)
  
  # List manipulation
  reversed_azs = reverse(local.availability_zones)
  sorted_azs = sort(local.availability_zones)
  unique_ports = distinct([80, 443, 80, 8080, 443])
  
  # Flatten nested lists
  nested_cidrs = [
    ["10.0.1.0/24", "10.0.2.0/24"],
    ["10.0.3.0/24", "10.0.4.0/24"]
  ]
  flat_cidrs = flatten(local.nested_cidrs)
  
  # Chunklist
  all_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24", "10.0.4.0/24", "10.0.5.0/24", "10.0.6.0/24"]
  subnet_pairs = chunklist(local.all_subnets, 2)
  
  # Map operations
  environment_config = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t2.large"
  }
  
  all_keys = keys(local.environment_config)
  all_values = values(local.environment_config)
  
  # Merge maps
  default_tags = {
    ManagedBy = "Terraform"
    Project   = var.project_name
  }
  
  environment_tags = {
    Environment = var.environment
    CostCenter  = var.cost_center
  }
  
  all_tags = merge(local.default_tags, local.environment_tags)
  
  # Zipmap
  tag_keys = ["Owner", "Team", "Application"]
  tag_values = ["Platform", "DevOps", "WebApp"]
  custom_tags = zipmap(local.tag_keys, local.tag_values)
  
  # Set operations
  dev_features = ["feature-a", "feature-b", "feature-c"]
  prod_features = ["feature-b", "feature-c", "feature-d"]
  
  common_features = setintersection(local.dev_features, local.prod_features)
  all_features = setunion(local.dev_features, local.prod_features)
  dev_only_features = setsubtract(local.dev_features, local.prod_features)
}

# CIDR Functions
locals {
  vpc_cidr = "10.0.0.0/16"
  
  # Calculate subnet CIDRs
  public_subnets = [
    for i in range(3) :
    cidrsubnet(local.vpc_cidr, 8, i)
  ]
  
  private_subnets = [
    for i in range(3) :
    cidrsubnet(local.vpc_cidr, 8, i + 100)
  ]
  
  # Get first IP in subnet
  first_ips = [
    for subnet in local.public_subnets :
    cidrhost(subnet, 0)
  ]
  
  # Get netmask
  netmask = cidrnetmask(local.vpc_cidr)
}

# Conditional Logic
locals {
  # Ternary operator
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
  
  # Nested conditions
  instance_count = (
    var.environment == "prod" ? 5 :
    var.environment == "staging" ? 3 :
    1
  )
  
  # Coalesce
  final_name = coalesce(var.custom_name, local.sanitized_name, "default-name")
  
  # Try
  config_file = try(
    jsondecode(file("${path.module}/config.json")),
    {
      default = true
      settings = {}
    }
  )
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block = local.vpc_cidr
  
  tags = merge(
    local.all_tags,
    {
      Name = "${local.sanitized_name}-vpc"
    }
  )
}

# Create Subnets
resource "aws_subnet" "public" {
  count = length(local.public_subnets)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnets[count.index]
  availability_zone = local.availability_zones[count.index]
  
  tags = merge(
    local.all_tags,
    {
      Name = "${local.sanitized_name}-public-${count.index + 1}"
      Type = "public"
    }
  )
}

resource "aws_subnet" "private" {
  count = length(local.private_subnets)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.private_subnets[count.index]
  availability_zone = local.availability_zones[count.index]
  
  tags = merge(
    local.all_tags,
    {
      Name = "${local.sanitized_name}-private-${count.index + 1}"
      Type = "private"
    }
  )
}

# Security Group with dynamic ports
resource "aws_security_group" "web" {
  name        = "${local.sanitized_name}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = local.unique_ports
    content {
      description = "Port ${ingress.value}"
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(
    local.all_tags,
    {
      Name = "${local.sanitized_name}-web-sg"
    }
  )
}
```

**variables.tf:**

```hcl
variable "project_name" {
  description = "Project name"
  type        = string
  default     = "My Awesome Project"
}

variable "environment" {
  description = "Environment"
  type        = string
  default     = "dev"
}

variable "cost_center" {
  description = "Cost center"
  type        = string
  default     = "engineering"
}

variable "app_version" {
  description = "Application version"
  type        = string
  default     = "v1.2.3"
}

variable "custom_name" {
  description = "Custom name (optional)"
  type        = string
  default     = ""
}
```

**outputs.tf:**

```hcl
# String Function Results
output "string_operations" {
  description = "String function results"
  value = {
    original       = var.project_name
    upper          = local.project_name_upper
    lower          = local.project_name_lower
    title          = local.project_name_title
    sanitized      = local.sanitized_name
    bucket_name    = local.bucket_name
    short_name     = local.short_name
    version_number = local.version_number
  }
}

# Collection Function Results
output "collection_operations" {
  description = "Collection function results"
  value = {
    availability_zones = local.availability_zones
    az_count          = local.az_count
    reversed_azs      = local.reversed_azs
    unique_ports      = local.unique_ports
    flat_cidrs        = local.flat_cidrs
    subnet_pairs      = local.subnet_pairs
    common_features   = local.common_features
    all_features      = local.all_features
  }
}

# CIDR Function Results
output "cidr_operations" {
  description = "CIDR function results"
  value = {
    vpc_cidr        = local.vpc_cidr
    public_subnets  = local.public_subnets
    private_subnets = local.private_subnets
    first_ips       = local.first_ips
    netmask         = local.netmask
  }
}

# Conditional Logic Results
output "conditional_results" {
  description = "Conditional logic results"
  value = {
    instance_type  = local.instance_type
    instance_count = local.instance_count
    final_name     = local.final_name
  }
}

# Tag Results
output "tags" {
  description = "Generated tags"
  value = {
    default_tags     = local.default_tags
    environment_tags = local.environment_tags
    all_tags         = local.all_tags
    custom_tags      = local.custom_tags
  }
}
```

### Step 3: Test Functions

```bash
terraform init
terraform plan

# View all outputs
terraform apply -auto-approve
terraform output

# View specific outputs
terraform output string_operations
terraform output collection_operations
terraform output cidr_operations
terraform output conditional_results
terraform output tags

# Test with different variables
terraform apply -var="environment=prod" -var="project_name=Production App" -auto-approve
terraform output conditional_results
```

### Step 4: Experiment with Terraform Console

```bash
# Open Terraform console
terraform console

# Test string functions
> upper("hello world")
> lower("HELLO WORLD")
> replace("hello-world", "-", "_")
> split("-", "hello-world-test")
> join("_", ["hello", "world", "test"])

# Test collection functions
> length([1, 2, 3, 4, 5])
> reverse([1, 2, 3, 4, 5])
> distinct([1, 2, 2, 3, 3, 3])
> flatten([[1, 2], [3, 4], [5]])

# Test CIDR functions
> cidrsubnet("10.0.0.0/16", 8, 0)
> cidrsubnet("10.0.0.0/16", 8, 1)
> cidrhost("10.0.1.0/24", 0)
> cidrnetmask("10.0.0.0/16")

# Test conditional functions
> var.environment == "prod" ? "t2.large" : "t2.micro"
> coalesce("", "", "default")

# Exit console
> exit
```

### Step 5: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] String functions manipulate text correctly
- [ ] Collection functions work with lists and maps
- [ ] CIDR functions calculate subnets
- [ ] Conditional logic produces expected results
- [ ] Tags generated and merged correctly
- [ ] Terraform console experiments successful

---

## 🧪 Lab 7.3: Advanced Function Patterns

**Objective:** Combine multiple functions for complex transformations and dynamic configurations.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-7.3-advanced-patterns/{templates,configs}
cd lab-7.3-advanced-patterns
```

### Step 2: Create Template Files

**templates/user_data.sh.tpl:**

```bash
#!/bin/bash
set -e

# Instance Configuration
INSTANCE_NAME="${instance_name}"
ENVIRONMENT="${environment}"
APP_VERSION="${app_version}"

# Update system
yum update -y

# Install packages
%{ for package in packages ~}
yum install -y ${package}
%{ endfor ~}

# Configure application
cat > /etc/app/config.json <<EOF
{
  "name": "$INSTANCE_NAME",
  "environment": "$ENVIRONMENT",
  "version": "$APP_VERSION",
  "features": ${jsonencode(features)},
  "database": {
    "host": "${db_host}",
    "port": ${db_port}
  }
}
EOF

# Start services
%{ for service in services ~}
systemctl start ${service}
systemctl enable ${service}
%{ endfor ~}

echo "Configuration complete!"
```

**configs/application.yaml:**

```yaml
application:
  name: MyApp
  version: 1.0.0
  
environments:
  dev:
    instance_type: t2.micro
    instance_count: 1
    enable_monitoring: false
  staging:
    instance_type: t2.small
    instance_count: 2
    enable_monitoring: true
  prod:
    instance_type: t2.large
    instance_count: 5
    enable_monitoring: true
```

### Step 3: Create Main Configuration

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
  region = var.aws_region
}

# Load and parse configuration file
locals {
  config_file = yamldecode(file("${path.module}/configs/application.yaml"))
  app_config  = local.config_file.application
  env_config  = local.config_file.environments[var.environment]
}

# Complex data transformations
locals {
  # Generate subnet CIDRs dynamically
  vpc_cidr = "10.0.0.0/16"
  az_count = 3
  
  subnets = flatten([
    for tier in ["public", "private", "database"] : [
      for i in range(local.az_count) : {
        name       = "${tier}-${i + 1}"
        cidr_block = cidrsubnet(local.vpc_cidr, 8, (tier == "public" ? i : tier == "private" ? i + 10 : i + 20))
        tier       = tier
        az_index   = i
      }
    ]
  ])
  
  # Group subnets by tier
  subnets_by_tier = {
    for subnet in local.subnets :
    subnet.tier => subnet...
  }
  
  # Create subnet map for easy lookup
  subnet_map = {
    for subnet in local.subnets :
    subnet.name => subnet
  }
}

# Instance configuration with complex logic
locals {
  # Base instance configuration
  base_instances = [
    for i in range(local.env_config.instance_count) : {
      name      = "${var.project_name}-${var.environment}-${i + 1}"
      subnet    = local.subnets_by_tier["public"][i % length(local.subnets_by_tier["public"])]
      user_data = templatefile("${path.module}/templates/user_data.sh.tpl", {
        instance_name = "${var.project_name}-${var.environment}-${i + 1}"
        environment   = var.environment
        app_version   = local.app_config.version
        packages      = ["httpd", "git", "docker"]
        services      = ["httpd", "docker"]
        features      = var.features
        db_host       = "db.example.com"
        db_port       = 5432
      })
    }
  ]
  
  # Add monitoring configuration
  instances = [
    for instance in local.base_instances :
    merge(instance, {
      monitoring = local.env_config.enable_monitoring
      tags = merge(
        local.common_tags,
        {
          Name        = instance.name
          Subnet      = instance.subnet.name
          SubnetTier  = instance.subnet.tier
        }
      )
    })
  ]
}

# Common tags with dynamic values
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    CreatedAt   = formatdate("YYYY-MM-DD", timestamp())
    Version     = local.app_config.version
    ConfigHash  = md5(jsonencode(local.env_config))
  }
}

# Security group rules from configuration
locals {
  security_rules = {
    web = {
      ingress = [
        { port = 80, protocol = "tcp", cidr = "0.0.0.0/0", description = "HTTP" },
        { port = 443, protocol = "tcp", cidr = "0.0.0.0/0", description = "HTTPS" }
      ]
    }
    app = {
      ingress = [
        { port = 8080, protocol = "tcp", cidr = "10.0.0.0/16", description = "Application" },
        { port = 8443, protocol = "tcp", cidr = "10.0.0.0/16", description = "Application HTTPS" }
      ]
    }
    database = {
      ingress = [
        { port = 5432, protocol = "tcp", cidr = "10.0.0.0/16", description = "PostgreSQL" },
        { port = 3306, protocol = "tcp", cidr = "10.0.0.0/16", description = "MySQL" }
      ]
    }
  }
  
  # Flatten security rules for creation
  all_security_rules = flatten([
    for sg_name, sg_config in local.security_rules : [
      for rule in sg_config.ingress : {
        sg_name     = sg_name
        port        = rule.port
        protocol    = rule.protocol
        cidr        = rule.cidr
        description = rule.description
        key         = "${sg_name}-${rule.port}-${rule.protocol}"
      }
    ]
  ])
}

# Data Sources
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Resources

# VPC
resource "aws_vpc" "main" {
  cidr_block           = local.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-${var.environment}-vpc"
    }
  )
}

# Subnets
resource "aws_subnet" "main" {
  for_each = local.subnet_map
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr_block
  availability_zone       = data.aws_availability_zones.available.names[each.value.az_index]
  map_public_ip_on_launch = each.value.tier == "public"
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-${var.environment}-${each.key}"
      Tier = each.value.tier
    }
  )
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-${var.environment}-igw"
    }
  )
}

# Security Groups
resource "aws_security_group" "main" {
  for_each = local.security_rules
  
  name        = "${var.project_name}-${var.environment}-${each.key}-sg"
  description = "Security group for ${each.key}"
  vpc_id      = aws_vpc.main.id
  
  tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-${var.environment}-${each.key}-sg"
      Type = each.key
    }
  )
}

# Security Group Rules
resource "aws_security_group_rule" "ingress" {
  for_each = {
    for rule in local.all_security_rules :
    rule.key => rule
  }
  
  type              = "ingress"
  from_port         = each.value.port
  to_port           = each.value.port
  protocol          = each.value.protocol
  cidr_blocks       = [each.value.cidr]
  description       = each.value.description
  security_group_id = aws_security_group.main[each.value.sg_name].id
}

# EC2 Instances
resource "aws_instance" "app" {
  count = length(local.instances)
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = local.env_config.instance_type
  subnet_id     = aws_subnet.main[local.instances[count.index].subnet.name].id
  
  vpc_security_group_ids = [aws_security_group.main["web"].id]
  monitoring             = local.instances[count.index].monitoring
  
  user_data = local.instances[count.index].user_data
  
  tags = local.instances[count.index].tags
}
```

**variables.tf:**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "advanced-patterns"
}

variable "environment" {
  description = "Environment"
  type        = string
  default     = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "features" {
  description = "Feature flags"
  type        = map(bool)
  default = {
    caching    = true
    monitoring = true
    backup     = false
  }
}
```

**outputs.tf:**

```hcl
output "configuration" {
  description = "Loaded configuration"
  value = {
    app_config = local.app_config
    env_config = local.env_config
  }
}

output "subnet_structure" {
  description = "Generated subnet structure"
  value = {
    all_subnets     = local.subnets
    subnets_by_tier = local.subnets_by_tier
    subnet_map      = local.subnet_map
  }
}

output "instance_configuration" {
  description = "Instance configuration"
  value = [
    for instance in local.instances : {
      name       = instance.name
      subnet     = instance.subnet.name
      monitoring = instance.monitoring
    }
  ]
}

output "security_rules" {
  description = "Generated security rules"
  value = local.all_security_rules
}

output "instance_details" {
  description = "Created instance details"
  value = [
    for instance in aws_instance.app : {
      id         = instance.id
      public_ip  = instance.public_ip
      private_ip = instance.private_ip
      subnet_id  = instance.subnet_id
    }
  ]
}

output "tags" {
  description = "Common tags"
  value       = local.common_tags
}
```

### Step 4: Deploy and Test

```bash
terraform init
terraform plan
terraform apply -auto-approve

# View complex outputs
terraform output configuration
terraform output subnet_structure
terraform output instance_configuration
terraform output security_rules
```

### Step 5: Test Different Environments

```bash
# Deploy staging
terraform apply -var="environment=staging" -auto-approve
terraform output instance_configuration

# Deploy production
terraform apply -var="environment=prod" -auto-approve
terraform output instance_configuration
```

### Step 6: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] YAML configuration loaded and parsed
- [ ] Complex subnet structure generated
- [ ] Instances configured dynamically
- [ ] Security rules created from configuration
- [ ] Template files rendered correctly
- [ ] Different environments have different configs
- [ ] All functions combined successfully

---

## 🎓 Lab Summary

### What You Learned

1. **Lab 7.1:** Data sources for querying infrastructure
2. **Lab 7.2:** String and collection functions
3. **Lab 7.3:** Advanced patterns combining multiple functions

### Key Concepts Practiced

- Data source queries
- AMI and VPC lookups
- String manipulation
- Collection operations
- CIDR calculations
- Conditional logic
- Template rendering
- Complex transformations
- Dynamic configurations

### Next Steps

1. ✅ Mark labs complete in [CHECKLIST.md](../CHECKLIST.md)
2. 📝 Review [INTERVIEW.md](./INTERVIEW.md) questions
3. 🚀 Proceed to [Chapter 8](../chapter-08/)

---

## 💡 Troubleshooting Tips

### Data Source Not Found

```bash
# Verify resource exists
aws ec2 describe-vpcs --filters "Name=isDefault,Values=true"

# Check filters
terraform console
> data.aws_vpc.default
```

### Function Errors

```bash
# Test in console
terraform console
> cidrsubnet("10.0.0.0/16", 8, 0)
> upper("hello")
> length([1, 2, 3])
```

### Template Rendering Issues

```bash
# Check template syntax
cat templates/user_data.sh.tpl

# Test rendering
terraform console
> templatefile("templates/user_data.sh.tpl", {...})
```

---

**🎉 Congratulations!** You've mastered Terraform data sources and functions!
