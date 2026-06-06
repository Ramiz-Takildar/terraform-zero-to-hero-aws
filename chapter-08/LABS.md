# Chapter 8: Labs - Advanced Patterns and Best Practices

## 🎯 Lab Overview

These hands-on labs will help you master advanced Terraform patterns including for_each, dynamic blocks, lifecycle rules, and workspaces.

**Prerequisites:**
- AWS account with appropriate permissions
- AWS CLI configured
- Terraform installed (v1.0+)
- Completed Chapters 1-7 labs

**Estimated Time:** 2-3 hours total

---

## 🧪 Lab 8.1: For_Each and Dynamic Blocks

**Objective:** Master for_each iteration and dynamic block patterns.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-8.1-foreach-dynamic
cd lab-8.1-foreach-dynamic
```

### Step 2: Create Configuration

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
}

variable "instances" {
  description = "Map of instances to create"
  type = map(object({
    instance_type = string
    subnet_type   = string
    monitoring    = bool
  }))
  
  default = {
    web-1 = {
      instance_type = "t2.micro"
      subnet_type   = "public"
      monitoring    = false
    }
    web-2 = {
      instance_type = "t2.micro"
      subnet_type   = "public"
      monitoring    = false
    }
    app-1 = {
      instance_type = "t2.small"
      subnet_type   = "private"
      monitoring    = true
    }
  }
}

variable "security_groups" {
  description = "Security group configurations"
  type = map(object({
    description = string
    ingress_rules = list(object({
      from_port   = number
      to_port     = number
      protocol    = string
      cidr_blocks = list(string)
      description = string
    }))
  }))
  
  default = {
    web = {
      description = "Security group for web servers"
      ingress_rules = [
        {
          from_port   = 80
          to_port     = 80
          protocol    = "tcp"
          cidr_blocks = ["0.0.0.0/0"]
          description = "HTTP"
        },
        {
          from_port   = 443
          to_port     = 443
          protocol    = "tcp"
          cidr_blocks = ["0.0.0.0/0"]
          description = "HTTPS"
        }
      ]
    }
    app = {
      description = "Security group for application servers"
      ingress_rules = [
        {
          from_port   = 8080
          to_port     = 8080
          protocol    = "tcp"
          cidr_blocks = ["10.0.0.0/16"]
          description = "Application"
        }
      ]
    }
  }
}

variable "s3_buckets" {
  description = "S3 buckets to create"
  type        = set(string)
  default     = ["logs", "data", "backups"]
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
  region = var.aws_region
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

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  
  tags = {
    Name        = "${var.project_name}-vpc"
    Environment = var.environment
  }
}

# Subnets using for_each
locals {
  subnets = {
    public-1 = {
      cidr_block = "10.0.1.0/24"
      az_index   = 0
      public     = true
    }
    public-2 = {
      cidr_block = "10.0.2.0/24"
      az_index   = 1
      public     = true
    }
    private-1 = {
      cidr_block = "10.0.11.0/24"
      az_index   = 0
      public     = false
    }
    private-2 = {
      cidr_block = "10.0.12.0/24"
      az_index   = 1
      public     = false
    }
  }
}

resource "aws_subnet" "main" {
  for_each = local.subnets
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr_block
  availability_zone       = data.aws_availability_zones.available.names[each.value.az_index]
  map_public_ip_on_launch = each.value.public
  
  tags = {
    Name        = "${var.project_name}-${each.key}"
    Environment = var.environment
    Type        = each.value.public ? "public" : "private"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name        = "${var.project_name}-igw"
    Environment = var.environment
  }
}

# Security Groups with dynamic blocks
resource "aws_security_group" "main" {
  for_each = var.security_groups
  
  name        = "${var.project_name}-${each.key}-sg"
  description = each.value.description
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = each.value.ingress_rules
    content {
      from_port   = ingress.value.from_port
      to_port     = ingress.value.to_port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name        = "${var.project_name}-${each.key}-sg"
    Environment = var.environment
  }
}

# EC2 Instances with for_each
resource "aws_instance" "main" {
  for_each = var.instances
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = each.value.instance_type
  
  # Select subnet based on type
  subnet_id = [
    for name, subnet in aws_subnet.main :
    subnet.id if startswith(name, each.value.subnet_type)
  ][0]
  
  # Select security group based on instance name
  vpc_security_group_ids = [
    startswith(each.key, "web") ? aws_security_group.main["web"].id : aws_security_group.main["app"].id
  ]
  
  monitoring = each.value.monitoring
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>${each.key}</h1>" > /var/www/html/index.html
              echo "<p>Instance Type: ${each.value.instance_type}</p>" >> /var/www/html/index.html
              echo "<p>Subnet Type: ${each.value.subnet_type}</p>" >> /var/www/html/index.html
              EOF
  
  tags = {
    Name        = "${var.project_name}-${each.key}"
    Environment = var.environment
    Type        = each.value.subnet_type
  }
}

# S3 Buckets with for_each on set
resource "aws_s3_bucket" "main" {
  for_each = var.s3_buckets
  
  bucket = "${var.project_name}-${var.environment}-${each.value}"
  
  tags = {
    Name        = each.value
    Environment = var.environment
  }
}

resource "aws_s3_bucket_versioning" "main" {
  for_each = aws_s3_bucket.main
  
  bucket = each.value.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Complex for_each pattern - flatten nested structure
locals {
  # Create all combinations of instances and buckets for access
  instance_bucket_pairs = flatten([
    for instance_name, instance in var.instances : [
      for bucket_name in var.s3_buckets : {
        instance_name = instance_name
        bucket_name   = bucket_name
        key           = "${instance_name}-${bucket_name}"
      }
    ]
  ])
}

# Example output showing the flattened structure
output "instance_bucket_pairs" {
  description = "All instance-bucket combinations"
  value = {
    for pair in local.instance_bucket_pairs :
    pair.key => {
      instance = pair.instance_name
      bucket   = pair.bucket_name
    }
  }
}
```

**outputs.tf:**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "subnets" {
  description = "Created subnets"
  value = {
    for name, subnet in aws_subnet.main :
    name => {
      id         = subnet.id
      cidr_block = subnet.cidr_block
      az         = subnet.availability_zone
      public     = subnet.map_public_ip_on_launch
    }
  }
}

output "security_groups" {
  description = "Created security groups"
  value = {
    for name, sg in aws_security_group.main :
    name => {
      id          = sg.id
      description = sg.description
    }
  }
}

output "instances" {
  description = "Created instances"
  value = {
    for name, instance in aws_instance.main :
    name => {
      id         = instance.id
      public_ip  = instance.public_ip
      private_ip = instance.private_ip
      subnet_id  = instance.subnet_id
    }
  }
}

output "buckets" {
  description = "Created S3 buckets"
  value = {
    for name, bucket in aws_s3_bucket.main :
    name => {
      id  = bucket.id
      arn = bucket.arn
    }
  }
}
```

### Step 3: Deploy Infrastructure

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

### Step 4: Verify For_Each Behavior

```bash
# View all outputs
terraform output

# View specific resource
terraform state show 'aws_instance.main["web-1"]'
terraform state show 'aws_s3_bucket.main["logs"]'

# List all instances
terraform state list | grep aws_instance

# Test adding a new instance
cat >> terraform.tfvars <<EOF
instances = {
  web-1 = {
    instance_type = "t2.micro"
    subnet_type   = "public"
    monitoring    = false
  }
  web-2 = {
    instance_type = "t2.micro"
    subnet_type   = "public"
    monitoring    = false
  }
  app-1 = {
    instance_type = "t2.small"
    subnet_type   = "private"
    monitoring    = true
  }
  app-2 = {
    instance_type = "t2.small"
    subnet_type   = "private"
    monitoring    = true
  }
}
EOF

terraform apply -auto-approve
# Notice: Only app-2 is created, existing resources unchanged
```

### Step 5: Test Dynamic Blocks

```bash
# Add new security group rule
cat >> terraform.tfvars <<EOF
security_groups = {
  web = {
    description = "Security group for web servers"
    ingress_rules = [
      {
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        description = "HTTP"
      },
      {
        from_port   = 443
        to_port     = 443
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
        description = "HTTPS"
      },
      {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["10.0.0.0/16"]
        description = "SSH from VPC"
      }
    ]
  }
  app = {
    description = "Security group for application servers"
    ingress_rules = [
      {
        from_port   = 8080
        to_port     = 8080
        protocol    = "tcp"
        cidr_blocks = ["10.0.0.0/16"]
        description = "Application"
      }
    ]
  }
}
EOF

terraform apply -auto-approve
# Notice: Only web security group updated with new rule
```

### Step 6: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] For_each creates resources with unique keys
- [ ] Adding/removing items doesn't affect others
- [ ] Dynamic blocks generate multiple nested blocks
- [ ] Security group rules created dynamically
- [ ] Complex nested structures flattened correctly
- [ ] Resources referenced by key, not index

---

## 🧪 Lab 8.2: Lifecycle Rules and Dependencies

**Objective:** Master lifecycle rules and dependency management.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-8.2-lifecycle
cd lab-8.2-lifecycle
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
    Name = "lifecycle-demo-vpc"
  }
}

# Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "public-subnet"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "main-igw"
  }
}

# Security Group with create_before_destroy
resource "aws_security_group" "web" {
  name        = "web-sg-${timestamp()}"
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
  
  lifecycle {
    create_before_destroy = true
  }
  
  tags = {
    Name = "web-sg"
  }
}

# Launch Template with create_before_destroy
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = base64encode(<<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Version: ${var.app_version}</h1>" > /var/www/html/index.html
              EOF
  )
  
  lifecycle {
    create_before_destroy = true
  }
  
  tags = {
    Name    = "app-template"
    Version = var.app_version
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  vpc_zone_identifier = [aws_subnet.public.id]
  min_size            = 1
  max_size            = 3
  desired_capacity    = 2
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  lifecycle {
    create_before_destroy = true
  }
  
  tag {
    key                 = "Name"
    value               = "app-instance"
    propagate_at_launch = true
  }
}

# Database with prevent_destroy
resource "aws_db_subnet_group" "main" {
  name       = "main"
  subnet_ids = [aws_subnet.public.id]
  
  tags = {
    Name = "main-db-subnet-group"
  }
}

resource "aws_db_instance" "main" {
  identifier           = "demo-db"
  engine               = "postgres"
  engine_version       = "14.7"
  instance_class       = "db.t3.micro"
  allocated_storage    = 20
  db_name              = "demodb"
  username             = "admin"
  password             = "changeme123"
  skip_final_snapshot  = true
  db_subnet_group_name = aws_db_subnet_group.main.name
  
  lifecycle {
    prevent_destroy = var.environment == "prod"
  }
  
  tags = {
    Name        = "demo-db"
    Environment = var.environment
  }
}

# Instance with ignore_changes
resource "aws_instance" "managed_externally" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id
  
  tags = {
    Name         = "externally-managed"
    ManagedBy    = "External System"
    LastModified = timestamp()
  }
  
  lifecycle {
    ignore_changes = [
      tags["LastModified"],
      tags["ManagedBy"],
      user_data
    ]
  }
}

# Resource with replace_triggered_by
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id
  
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              echo "Security Group: ${aws_security_group.web.id}"
              EOF
  
  lifecycle {
    replace_triggered_by = [
      aws_security_group.web.id
    ]
  }
  
  tags = {
    Name = "app-instance"
  }
}

# Explicit dependencies
resource "null_resource" "wait_for_instances" {
  depends_on = [
    aws_autoscaling_group.app,
    aws_instance.app
  ]
  
  provisioner "local-exec" {
    command = "echo 'All instances are ready'"
  }
}

# Preconditions and Postconditions
resource "aws_instance" "validated" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  subnet_id     = aws_subnet.public.id
  
  lifecycle {
    precondition {
      condition     = data.aws_ami.amazon_linux_2.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }
    
    precondition {
      condition     = var.environment == "prod" ? var.instance_type != "t2.micro" : true
      error_message = "Production instances cannot use t2.micro."
    }
    
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP."
    }
  }
  
  tags = {
    Name = "validated-instance"
  }
}
```

**variables.tf:**

```hcl
variable "environment" {
  description = "Environment"
  type        = string
  default     = "dev"
}

variable "instance_type" {
  description = "Instance type"
  type        = string
  default     = "t2.micro"
}

variable "app_version" {
  description = "Application version"
  type        = string
  default     = "1.0.0"
}
```

**outputs.tf:**

```hcl
output "asg_name" {
  description = "Auto Scaling Group name"
  value       = aws_autoscaling_group.app.name
}

output "launch_template_id" {
  description = "Launch Template ID"
  value       = aws_launch_template.app.id
}

output "database_endpoint" {
  description = "Database endpoint"
  value       = aws_db_instance.main.endpoint
}
```

### Step 3: Test Lifecycle Rules

```bash
terraform init
terraform apply -auto-approve

# Test create_before_destroy
# Change app version to trigger launch template recreation
terraform apply -var="app_version=2.0.0" -auto-approve
# Notice: New launch template created before old one destroyed

# Test ignore_changes
# Manually modify tags in AWS Console for "externally-managed" instance
# Then run terraform plan
terraform plan
# Notice: Tag changes ignored

# Test prevent_destroy (will fail)
terraform destroy -target=aws_db_instance.main
# Error: prevent_destroy is set

# Override for testing
terraform apply -var="environment=dev" -auto-approve
terraform destroy -target=aws_db_instance.main -auto-approve
# Now it works
```

### Step 4: Test Preconditions

```bash
# Try to create prod instance with t2.micro (should fail)
terraform apply -var="environment=prod" -var="instance_type=t2.micro"
# Error: Production instances cannot use t2.micro

# Use valid instance type
terraform apply -var="environment=prod" -var="instance_type=t2.small" -auto-approve
```

### Step 5: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] create_before_destroy prevents downtime
- [ ] prevent_destroy blocks destruction
- [ ] ignore_changes skips specified attributes
- [ ] replace_triggered_by forces replacement
- [ ] depends_on enforces order
- [ ] Preconditions validate before creation
- [ ] Postconditions validate after creation

---

## 🧪 Lab 8.3: Workspaces and Advanced Patterns

**Objective:** Use Terraform workspaces for multi-environment management.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-8.3-workspaces
cd lab-8.3-workspaces
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
  
  default_tags {
    tags = {
      Workspace   = terraform.workspace
      ManagedBy   = "Terraform"
      Environment = local.config.environment
    }
  }
}

# Workspace-specific configuration
locals {
  workspace_config = {
    dev = {
      environment    = "development"
      instance_type  = "t2.micro"
      instance_count = 1
      db_instance    = "db.t3.micro"
      enable_backup  = false
    }
    staging = {
      environment    = "staging"
      instance_type  = "t2.small"
      instance_count = 2
      db_instance    = "db.t3.small"
      enable_backup  = true
    }
    prod = {
      environment    = "production"
      instance_type  = "t2.large"
      instance_count = 5
      db_instance    = "db.t3.medium"
      enable_backup  = true
    }
  }
  
  config = local.workspace_config[terraform.workspace]
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = "10.${terraform.workspace == "dev" ? 0 : terraform.workspace == "staging" ? 1 : 2}.0.0/16"
  
  tags = {
    Name = "${terraform.workspace}-vpc"
  }
}

# Subnets
resource "aws_subnet" "public" {
  count = local.config.instance_count
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.${terraform.workspace == "dev" ? 0 : terraform.workspace == "staging" ? 1 : 2}.${count.index + 1}.0/24"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${terraform.workspace}-public-${count.index + 1}"
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${terraform.workspace}-web-sg"
  description = "Web server security group for ${terraform.workspace}"
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
  
  tags = {
    Name = "${terraform.workspace}-web-sg"
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

resource "aws_instance" "app" {
  count = local.config.instance_count
  
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = local.config.instance_type
  subnet_id              = aws_subnet.public[count.index % length(aws_subnet.public)].id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              cat > /var/www/html/index.html <<HTML
              <h1>${terraform.workspace} - Instance ${count.index + 1}</h1>
              <p>Environment: ${local.config.environment}</p>
              <p>Instance Type: ${local.config.instance_type}</p>
              <p>Workspace: ${terraform.workspace}</p>
              HTML
              EOF
  
  tags = {
    Name  = "${terraform.workspace}-app-${count.index + 1}"
    Index = count.index
  }
}

# S3 Bucket (workspace-specific)
resource "aws_s3_bucket" "data" {
  bucket = "terraform-workspace-demo-${terraform.workspace}-${data.aws_caller_identity.current.account_id}"
  
  tags = {
    Name = "${terraform.workspace}-data"
  }
}

data "aws_caller_identity" "current" {}

# Conditional resources based on workspace
resource "aws_s3_bucket_versioning" "data" {
  count = local.config.enable_backup ? 1 : 0
  
  bucket = aws_s3_bucket.data.id
  
  versioning_configuration {
    status = "Enabled"
  }
}
```

**outputs.tf:**

```hcl
output "workspace" {
  description = "Current workspace"
  value       = terraform.workspace
}

output "configuration" {
  description = "Current workspace configuration"
  value       = local.config
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "instance_count" {
  description = "Number of instances"
  value       = length(aws_instance.app)
}

output "instance_ips" {
  description = "Instance public IPs"
  value       = aws_instance.app[*].public_ip
}

output "bucket_name" {
  description = "S3 bucket name"
  value       = aws_s3_bucket.data.id
}

output "backup_enabled" {
  description = "Whether backup is enabled"
  value       = local.config.enable_backup
}
```

### Step 3: Work with Workspaces

```bash
# Initialize
terraform init

# List workspaces
terraform workspace list

# Create dev workspace
terraform workspace new dev
terraform apply -auto-approve

# View outputs
terraform output

# Create staging workspace
terraform workspace new staging
terraform apply -auto-approve

# View outputs (different configuration)
terraform output

# Create prod workspace
terraform workspace new prod
terraform apply -auto-approve

# View outputs (different configuration)
terraform output

# Switch between workspaces
terraform workspace select dev
terraform output

terraform workspace select staging
terraform output

terraform workspace select prod
terraform output
```

### Step 4: Compare Workspaces

```bash
# Create comparison script
cat > compare-workspaces.sh <<'EOF'
#!/bin/bash

echo "=== Workspace Comparison ==="
echo ""

for workspace in dev staging prod; do
    echo "--- $workspace ---"
    terraform workspace select $workspace > /dev/null 2>&1
    echo "VPC CIDR: $(terraform output -raw vpc_cidr)"
    echo "Instance Count: $(terraform output -raw instance_count)"
    echo "Instance Type: $(terraform output -json configuration | jq -r '.instance_type')"
    echo "Backup Enabled: $(terraform output -raw backup_enabled)"
    echo ""
done
EOF

chmod +x compare-workspaces.sh
./compare-workspaces.sh
```

### Step 5: Test Workspace Isolation

```bash
# Modify dev workspace
terraform workspace select dev
terraform apply -var="extra_instance=true" -auto-approve

# Check staging (should be unchanged)
terraform workspace select staging
terraform show

# Destroy dev workspace only
terraform workspace select dev
terraform destroy -auto-approve

# Verify other workspaces still exist
terraform workspace select staging
terraform show

terraform workspace select prod
terraform show
```

### Step 6: Cleanup All Workspaces

```bash
# Destroy all workspaces
for workspace in dev staging prod; do
    terraform workspace select $workspace
    terraform destroy -auto-approve
done

# Delete workspaces (can't delete current workspace)
terraform workspace select default
terraform workspace delete dev
terraform workspace delete staging
terraform workspace delete prod
```

### ✅ Success Criteria

- [ ] Multiple workspaces created successfully
- [ ] Each workspace has isolated state
- [ ] Workspace-specific configurations applied
- [ ] Resources tagged with workspace name
- [ ] Conditional resources based on workspace
- [ ] Easy switching between workspaces
- [ ] Workspace isolation verified

---

## 🎓 Lab Summary

### What You Learned

1. **Lab 8.1:** For_each and dynamic blocks for flexible resource creation
2. **Lab 8.2:** Lifecycle rules and dependency management
3. **Lab 8.3:** Terraform workspaces for multi-environment management

### Key Concepts Practiced

- for_each with maps and sets
- Dynamic blocks for nested configurations
- Lifecycle rules (create_before_destroy, prevent_destroy, ignore_changes)
- Explicit dependencies with depends_on
- Preconditions and postconditions
- Terraform workspaces
- Workspace-specific configurations
- State isolation

### Best Practices Learned

1. Use for_each for unique, identifiable resources
2. Use dynamic blocks to reduce repetition
3. Apply lifecycle rules strategically
4. Use preconditions to validate early
5. Use workspaces for environment separation
6. Tag resources with workspace information
7. Keep workspace configurations in locals

### Next Steps

1. ✅ Mark labs complete in [CHECKLIST.md](../CHECKLIST.md)
2. 📝 Review [INTERVIEW.md](./INTERVIEW.md) questions
3. 🚀 Proceed to [Chapter 9](../chapter-09/)

---

## 💡 Troubleshooting Tips

### For_Each Issues

```bash
# Check resource keys
terraform state list

# View specific resource
terraform state show 'aws_instance.main["web-1"]'

# Convert list to set for for_each
toset(var.my_list)
```

### Lifecycle Errors

```bash
# Override prevent_destroy temporarily
terraform destroy -target=resource_type.name -auto-approve

# Check ignored attributes
terraform plan -out=plan.out
terraform show plan.out
```

### Workspace Issues

```bash
# List all workspaces
terraform workspace list

# Show current workspace
terraform workspace show

# Force workspace deletion (dangerous!)
rm -rf terraform.tfstate.d/workspace_name
```

---

**🎉 Congratulations!** You've mastered advanced Terraform patterns!
