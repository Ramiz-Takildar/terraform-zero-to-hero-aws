# Chapter 5: Labs - Variables and Outputs

## 🎯 Lab Overview

These hands-on labs will help you master Terraform variables, outputs, and local values.

**Prerequisites:**
- AWS account with appropriate permissions
- AWS CLI configured
- Terraform installed (v1.0+)
- Completed Chapters 1-4 labs

**Estimated Time:** 2-3 hours total

---

## 🧪 Lab 5.1: Variable Types and Validation

**Objective:** Practice using different variable types with validation rules.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-5.1-variables
cd lab-5.1-variables
```

### Step 2: Create Variables File

**variables.tf:**

```hcl
# Simple types
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
  
  validation {
    condition     = can(regex("^[a-z]{2}-[a-z]+-[0-9]$", var.aws_region))
    error_message = "Must be a valid AWS region format (e.g., us-east-1)."
  }
}

variable "instance_count" {
  description = "Number of EC2 instances to create"
  type        = number
  default     = 2
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 5
    error_message = "Instance count must be between 1 and 5."
  }
}

variable "enable_monitoring" {
  description = "Enable detailed CloudWatch monitoring"
  type        = bool
  default     = false
}

# Collection types
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
  
  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least 2 availability zones required for high availability."
  }
}

variable "allowed_ssh_ips" {
  description = "Set of IP addresses allowed to SSH"
  type        = set(string)
  default     = ["0.0.0.0/0"]
}

variable "instance_types" {
  description = "Map of environment to instance type"
  type        = map(string)
  default = {
    dev     = "t2.micro"
    staging = "t2.small"
    prod    = "t2.medium"
  }
}

# Complex types
variable "vpc_config" {
  description = "VPC configuration object"
  type = object({
    cidr_block           = string
    enable_dns_hostnames = bool
    enable_dns_support   = bool
  })
  
  default = {
    cidr_block           = "10.0.0.0/16"
    enable_dns_hostnames = true
    enable_dns_support   = true
  }
  
  validation {
    condition     = can(cidrhost(var.vpc_config.cidr_block, 0))
    error_message = "VPC CIDR block must be valid."
  }
}

variable "subnets" {
  description = "List of subnet configurations"
  type = list(object({
    cidr_block        = string
    availability_zone = string
    public            = bool
    name              = string
  }))
  
  default = [
    {
      cidr_block        = "10.0.1.0/24"
      availability_zone = "us-east-1a"
      public            = true
      name              = "public-1"
    },
    {
      cidr_block        = "10.0.2.0/24"
      availability_zone = "us-east-1b"
      public            = true
      name              = "public-2"
    }
  ]
}

variable "tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    Environment = "lab"
    ManagedBy   = "Terraform"
    Lab         = "5.1"
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "project_name" {
  description = "Project name (lowercase alphanumeric and hyphens only)"
  type        = string
  default     = "terraform-lab"
  
  validation {
    condition     = can(regex("^[a-z0-9][a-z0-9-]*[a-z0-9]$", var.project_name))
    error_message = "Project name must be lowercase alphanumeric with hyphens."
  }
  
  validation {
    condition     = length(var.project_name) >= 3 && length(var.project_name) <= 32
    error_message = "Project name must be between 3 and 32 characters."
  }
}
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
  
  default_tags {
    tags = var.tags
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_config.cidr_block
  enable_dns_hostnames = var.vpc_config.enable_dns_hostnames
  enable_dns_support   = var.vpc_config.enable_dns_support
  
  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Subnets
resource "aws_subnet" "main" {
  count = length(var.subnets)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.subnets[count.index].cidr_block
  availability_zone       = var.subnets[count.index].availability_zone
  map_public_ip_on_launch = var.subnets[count.index].public
  
  tags = {
    Name   = "${var.project_name}-${var.subnets[count.index].name}"
    Public = var.subnets[count.index].public
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.allowed_ssh_ips
    content {
      description = "SSH from ${ingress.value}"
      from_port   = 22
      to_port     = 22
      protocol    = "tcp"
      cidr_blocks = [ingress.value]
    }
  }
  
  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
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

# EC2 Instances
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
  instance_type          = var.instance_types[var.environment]
  subnet_id              = aws_subnet.main[count.index % length(aws_subnet.main)].id
  vpc_security_group_ids = [aws_security_group.web.id]
  monitoring             = var.enable_monitoring
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>${var.project_name} - Instance ${count.index + 1}</h1>" > /var/www/html/index.html
              echo "<p>Environment: ${var.environment}</p>" >> /var/www/html/index.html
              echo "<p>Instance Type: ${var.instance_types[var.environment]}</p>" >> /var/www/html/index.html
              EOF
  
  tags = {
    Name = "${var.project_name}-web-${count.index + 1}"
  }
}
```

### Step 4: Create Variable Files

**dev.tfvars:**

```hcl
environment      = "dev"
instance_count   = 1
enable_monitoring = false
project_name     = "myapp-dev"

vpc_config = {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
}

subnets = [
  {
    cidr_block        = "10.0.1.0/24"
    availability_zone = "us-east-1a"
    public            = true
    name              = "public-1"
  }
]

tags = {
  Environment = "dev"
  CostCenter  = "engineering"
}
```

**prod.tfvars:**

```hcl
environment      = "prod"
instance_count   = 3
enable_monitoring = true
project_name     = "myapp-prod"

vpc_config = {
  cidr_block           = "10.1.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
}

subnets = [
  {
    cidr_block        = "10.1.1.0/24"
    availability_zone = "us-east-1a"
    public            = true
    name              = "public-1"
  },
  {
    cidr_block        = "10.1.2.0/24"
    availability_zone = "us-east-1b"
    public            = true
    name              = "public-2"
  }
]

tags = {
  Environment = "prod"
  CostCenter  = "operations"
}
```

### Step 5: Test Variable Validation

```bash
# Initialize
terraform init

# Test with invalid environment
terraform plan -var="environment=invalid"
# Should fail: Environment must be dev, staging, or prod

# Test with invalid instance count
terraform plan -var="instance_count=10"
# Should fail: Instance count must be between 1 and 5

# Test with invalid project name
terraform plan -var="project_name=My_Project"
# Should fail: Project name must be lowercase alphanumeric

# Test with valid dev configuration
terraform plan -var-file="dev.tfvars"

# Test with valid prod configuration
terraform plan -var-file="prod.tfvars"
```

### Step 6: Deploy Dev Environment

```bash
terraform apply -var-file="dev.tfvars" -auto-approve
```

### Step 7: Verify Variables

```bash
# Check created resources
terraform show

# Verify instance type matches environment
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=myapp-dev-web-*" \
  --query 'Reservations[].Instances[].InstanceType'
```

### Step 8: Cleanup

```bash
terraform destroy -var-file="dev.tfvars" -auto-approve
```

### ✅ Success Criteria

- [ ] All variable types declared correctly
- [ ] Validation rules work as expected
- [ ] Invalid values rejected with clear error messages
- [ ] Different environments use different configurations
- [ ] Complex types (objects, lists) work correctly
- [ ] Variable files load successfully

---

## 🧪 Lab 5.2: Outputs and Data Flow

**Objective:** Master Terraform outputs and data flow between resources.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-5.2-outputs
cd lab-5.2-outputs
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

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "lab-vpc"
  }
}

# Subnets
resource "aws_subnet" "public" {
  count = 2
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index + 1}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "public-${count.index + 1}"
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

# Security Group
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Web server security group"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# EC2 Instances
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  count = 2
  
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public[count.index].id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Server ${count.index + 1}</h1>" > /var/www/html/index.html
              EOF
  
  tags = {
    Name = "web-${count.index + 1}"
  }
}

# S3 Bucket
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

resource "aws_s3_bucket" "data" {
  bucket = "lab-data-${random_id.bucket_suffix.hex}"
  
  tags = {
    Name = "lab-data-bucket"
  }
}

resource "aws_s3_bucket_versioning" "data" {
  bucket = aws_s3_bucket.data.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# RDS Instance (for sensitive output example)
resource "random_password" "db_password" {
  length  = 16
  special = true
}

resource "aws_db_subnet_group" "main" {
  name       = "main"
  subnet_ids = aws_subnet.public[*].id
}

resource "aws_db_instance" "main" {
  identifier           = "lab-db"
  engine               = "postgres"
  engine_version       = "14.7"
  instance_class       = "db.t3.micro"
  allocated_storage    = 20
  db_name              = "labdb"
  username             = "admin"
  password             = random_password.db_password.result
  skip_final_snapshot  = true
  db_subnet_group_name = aws_db_subnet_group.main.name
  
  tags = {
    Name = "lab-db"
  }
}
```

**outputs.tf:**

```hcl
# Simple outputs
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

# List outputs
output "subnet_ids" {
  description = "List of subnet IDs"
  value       = aws_subnet.public[*].id
}

output "availability_zones" {
  description = "Availability zones used"
  value       = aws_subnet.public[*].availability_zone
}

# Instance outputs
output "instance_ids" {
  description = "List of instance IDs"
  value       = aws_instance.web[*].id
}

output "instance_public_ips" {
  description = "List of instance public IPs"
  value       = aws_instance.web[*].public_ip
}

output "instance_private_ips" {
  description = "List of instance private IPs"
  value       = aws_instance.web[*].private_ip
}

# Complex object output
output "instance_details" {
  description = "Detailed information about instances"
  value = [
    for instance in aws_instance.web : {
      id         = instance.id
      public_ip  = instance.public_ip
      private_ip = instance.private_ip
      az         = instance.availability_zone
    }
  ]
}

# Map output
output "instance_map" {
  description = "Map of instance names to IPs"
  value = {
    for instance in aws_instance.web :
    instance.tags["Name"] => instance.public_ip
  }
}

# S3 outputs
output "bucket_name" {
  description = "Name of the S3 bucket"
  value       = aws_s3_bucket.data.id
}

output "bucket_arn" {
  description = "ARN of the S3 bucket"
  value       = aws_s3_bucket.data.arn
}

output "bucket_domain_name" {
  description = "Domain name of the S3 bucket"
  value       = aws_s3_bucket.data.bucket_domain_name
}

# Database outputs
output "db_endpoint" {
  description = "Database endpoint"
  value       = aws_db_instance.main.endpoint
}

output "db_address" {
  description = "Database address"
  value       = aws_db_instance.main.address
}

output "db_port" {
  description = "Database port"
  value       = aws_db_instance.main.port
}

output "db_name" {
  description = "Database name"
  value       = aws_db_instance.main.db_name
}

output "db_username" {
  description = "Database username"
  value       = aws_db_instance.main.username
}

# Sensitive output
output "db_password" {
  description = "Database password (sensitive)"
  value       = random_password.db_password.result
  sensitive   = true
}

# Connection string (sensitive)
output "db_connection_string" {
  description = "Database connection string"
  value       = "postgresql://${aws_db_instance.main.username}:${random_password.db_password.result}@${aws_db_instance.main.endpoint}/${aws_db_instance.main.db_name}"
  sensitive   = true
}

# Computed outputs
output "web_urls" {
  description = "URLs to access web servers"
  value = [
    for ip in aws_instance.web[*].public_ip :
    "http://${ip}"
  ]
}

output "resource_count" {
  description = "Count of created resources"
  value = {
    instances = length(aws_instance.web)
    subnets   = length(aws_subnet.public)
  }
}

# Conditional output
output "monitoring_enabled" {
  description = "Whether monitoring is enabled"
  value       = alltrue([for instance in aws_instance.web : instance.monitoring])
}
```

### Step 3: Deploy

```bash
terraform init
terraform apply -auto-approve
```

### Step 4: Explore Outputs

```bash
# View all outputs
terraform output

# View specific output
terraform output vpc_id

# View list output
terraform output subnet_ids

# View complex output
terraform output instance_details

# View sensitive output (won't show by default)
terraform output db_password
# Shows: (sensitive value)

# Get raw sensitive value
terraform output -raw db_password

# JSON format
terraform output -json

# Pretty print JSON
terraform output -json | jq

# Get specific value from JSON
terraform output -json instance_map | jq -r '."web-1"'
```

### Step 5: Use Outputs in Scripts

Create `test-servers.sh`:

```bash
#!/bin/bash

# Get instance IPs from Terraform output
IPS=$(terraform output -json instance_public_ips | jq -r '.[]')

echo "Testing web servers..."
for IP in $IPS; do
    echo "Testing http://$IP"
    curl -s http://$IP
    echo ""
done

# Test database connection
DB_ENDPOINT=$(terraform output -raw db_endpoint)
DB_USER=$(terraform output -raw db_username)
DB_PASS=$(terraform output -raw db_password)
DB_NAME=$(terraform output -raw db_name)

echo "Database endpoint: $DB_ENDPOINT"
echo "Database user: $DB_USER"
echo "Database name: $DB_NAME"
```

```bash
chmod +x test-servers.sh
./test-servers.sh
```

### Step 6: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] All outputs display correctly
- [ ] Sensitive outputs hidden by default
- [ ] List and map outputs work
- [ ] Complex object outputs structured correctly
- [ ] Outputs usable in scripts
- [ ] JSON output format works

---

## 🧪 Lab 5.3: Local Values and Computed Configuration

**Objective:** Use local values for computed configuration and DRY principles.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-5.3-locals
cd lab-5.3-locals
```

### Step 2: Create Configuration

**variables.tf:**

```hcl
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "myapp"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_count" {
  description = "Number of subnets to create"
  type        = number
  default     = 3
}
```

**locals.tf:**

```hcl
locals {
  # Common tags
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
    CreatedAt   = timestamp()
  }
  
  # Name prefix
  name_prefix = "${var.project_name}-${var.environment}"
  
  # Environment-specific configuration
  env_config = {
    dev = {
      instance_type      = "t2.micro"
      instance_count     = 1
      enable_monitoring  = false
      enable_backup      = false
      db_instance_class  = "db.t3.micro"
      db_allocated_storage = 20
    }
    staging = {
      instance_type      = "t2.small"
      instance_count     = 2
      enable_monitoring  = true
      enable_backup      = true
      db_instance_class  = "db.t3.small"
      db_allocated_storage = 50
    }
    prod = {
      instance_type      = "t2.medium"
      instance_count     = 5
      enable_monitoring  = true
      enable_backup      = true
      db_instance_class  = "db.t3.medium"
      db_allocated_storage = 100
    }
  }
  
  # Current environment config
  current_config = local.env_config[var.environment]
  
  # Compute subnet CIDRs
  subnet_cidrs = [
    for i in range(var.subnet_count) :
    cidrsubnet(var.vpc_cidr, 8, i)
  ]
  
  # Availability zones
  azs = slice(data.aws_availability_zones.available.names, 0, var.subnet_count)
  
  # Port mappings
  ports = {
    http  = 80
    https = 443
    ssh   = 22
  }
  
  # Conditional features
  enable_nat_gateway = var.environment == "prod" ? true : false
  enable_vpn_gateway = var.environment == "prod" ? true : false
  
  # String manipulation
  bucket_name = lower(replace("${local.name_prefix}-data", "_", "-"))
  
  # Flatten nested structures
  instance_subnet_pairs = flatten([
    for i in range(local.current_config.instance_count) : [
      for subnet_id in aws_subnet.public[*].id : {
        instance_index = i
        subnet_id      = subnet_id
        key            = "${i}-${subnet_id}"
      }
    ]
  ])
}
```

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
  
  default_tags {
    tags = local.common_tags
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = {
    Name = "${local.name_prefix}-vpc"
  }
}

# Subnets (using computed CIDRs)
resource "aws_subnet" "public" {
  count = var.subnet_count
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = local.subnet_cidrs[count.index]
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${local.name_prefix}-public-${count.index + 1}"
    Type = "public"
  }
}

# Security Group (using local ports)
resource "aws_security_group" "web" {
  name        = "${local.name_prefix}-web-sg"
  description = "Web server security group"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = local.ports
    content {
      description = "${ingress.key} access"
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
  
  tags = {
    Name = "${local.name_prefix}-web-sg"
  }
}

# EC2 Instances (using environment config)
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  count = local.current_config.instance_count
  
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = local.current_config.instance_type
  subnet_id              = aws_subnet.public[count.index % length(aws_subnet.public)].id
  vpc_security_group_ids = [aws_security_group.web.id]
  monitoring             = local.current_config.enable_monitoring
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              cat > /var/www/html/index.html <<HTML
              <h1>${local.name_prefix} - Instance ${count.index + 1}</h1>
              <p>Environment: ${var.environment}</p>
              <p>Instance Type: ${local.current_config.instance_type}</p>
              <p>Monitoring: ${local.current_config.enable_monitoring}</p>
              HTML
              EOF
  
  tags = {
    Name  = "${local.name_prefix}-web-${count.index + 1}"
    Index = count.index
  }
}

# S3 Bucket (using computed name)
resource "aws_s3_bucket" "data" {
  bucket = local.bucket_name
  
  tags = {
    Name = local.bucket_name
  }
}
```

**outputs.tf:**

```hcl
output "environment_config" {
  description = "Current environment configuration"
  value       = local.current_config
}

output "name_prefix" {
  description = "Name prefix used for resources"
  value       = local.name_prefix
}

output "subnet_cidrs" {
  description = "Computed subnet CIDRs"
  value       = local.subnet_cidrs
}

output "availability_zones" {
  description = "Availability zones used"
  value       = local.azs
}

output "instance_details" {
  description = "Instance details"
  value = {
    count         = local.current_config.instance_count
    type          = local.current_config.instance_type
    monitoring    = local.current_config.enable_monitoring
    public_ips    = aws_instance.web[*].public_ip
  }
}

output "bucket_name" {
  description = "S3 bucket name"
  value       = local.bucket_name
}

output "common_tags" {
  description = "Common tags applied to all resources"
  value       = local.common_tags
}
```

### Step 3: Deploy Different Environments

```bash
# Initialize
terraform init

# Deploy dev environment
terraform apply -var="environment=dev" -auto-approve

# View outputs
terraform output

# Destroy dev
terraform destroy -var="environment=dev" -auto-approve

# Deploy prod environment
terraform apply -var="environment=prod" -auto-approve

# View outputs (notice different values)
terraform output

# Destroy prod
terraform destroy -var="environment=prod" -auto-approve
```

### Step 4: Compare Configurations

Create `compare.sh`:

```bash
#!/bin/bash

echo "=== Dev Configuration ==="
terraform plan -var="environment=dev" | grep "instance_type\|instance_count\|monitoring"

echo ""
echo "=== Staging Configuration ==="
terraform plan -var="environment=staging" | grep "instance_type\|instance_count\|monitoring"

echo ""
echo "=== Prod Configuration ==="
terraform plan -var="environment=prod" | grep "instance_type\|instance_count\|monitoring"
```

```bash
chmod +x compare.sh
./compare.sh
```

### ✅ Success Criteria

- [ ] Local values computed correctly
- [ ] Environment-specific configs work
- [ ] Subnet CIDRs calculated automatically
- [ ] Common tags applied to all resources
- [ ] Name prefix used consistently
- [ ] Different environments have different configurations
- [ ] Locals reduce code duplication

---

## 🎓 Lab Summary

### What You Learned

1. **Lab 5.1:** Variable types, validation, and variable files
2. **Lab 5.2:** Outputs for data flow and integration
3. **Lab 5.3:** Local values for computed configuration

### Key Concepts Practiced

- Simple and complex variable types
- Variable validation rules
- Variable precedence and files
- Output declarations
- Sensitive outputs
- Local values for DRY code
- Environment-specific configuration
- Computed values

### Next Steps

1. ✅ Mark labs complete in [CHECKLIST.md](../CHECKLIST.md)
2. 📝 Review [INTERVIEW.md](./INTERVIEW.md) questions
3. 🚀 Proceed to [Chapter 6](../chapter-06/)

---

## 💡 Troubleshooting Tips

### Variable Validation Errors

```bash
# Check validation rules
terraform validate

# Test with different values
terraform plan -var="environment=test"
```

### Output Not Showing

```bash
# Refresh state
terraform refresh

# Force output display
terraform output -json
```

### Local Value Errors

```bash
# Check local value computation
terraform console
> local.subnet_cidrs
> local.current_config
```

---

**🎉 Congratulations!** You've mastered Terraform variables and outputs!
