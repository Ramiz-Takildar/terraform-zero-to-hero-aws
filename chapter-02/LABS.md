# Chapter 2: Labs - AWS Fundamentals

## 🎯 Lab Overview

These hands-on labs will help you practice AWS fundamentals with Terraform.

**Prerequisites:**
- AWS account with appropriate permissions
- AWS CLI configured
- Terraform installed (v1.0+)
- Completed Chapter 1 labs

**Estimated Time:** 2-3 hours total

---

## 🧪 Lab 2.1: Create a VPC with Subnets

**Objective:** Create a complete VPC with public and private subnets.

**Duration:** 45 minutes

### Architecture

```
┌─────────────────────────────────────────────────────┐
│              VPC (10.0.0.0/16)                      │
│                                                     │
│  ┌──────────────────────┐  ┌──────────────────────┐│
│  │  Public Subnet       │  │  Private Subnet      ││
│  │  (10.0.1.0/24)       │  │  (10.0.2.0/24)       ││
│  │  AZ: us-east-1a      │  │  AZ: us-east-1a      ││
│  └──────────────────────┘  └──────────────────────┘│
│  ┌──────────────────────┐  ┌──────────────────────┐│
│  │  Public Subnet       │  │  Private Subnet      ││
│  │  (10.0.3.0/24)       │  │  (10.0.4.0/24)       ││
│  │  AZ: us-east-1b      │  │  AZ: us-east-1b      ││
│  └──────────────────────┘  └──────────────────────┘│
│                                                     │
│  Internet Gateway + NAT Gateways                    │
└─────────────────────────────────────────────────────┘
```

### Step 1: Create Project Structure

```bash
mkdir -p lab-2.1-vpc
cd lab-2.1-vpc
```

### Step 2: Create `main.tf`

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
    tags = {
      Environment = "lab"
      ManagedBy   = "Terraform"
      Lab         = "2.1"
    }
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.project_name}-public-${count.index + 1}"
    Type = "public"
  }
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = var.availability_zones[count.index]
  
  tags = {
    Name = "${var.project_name}-private-${count.index + 1}"
    Type = "private"
  }
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count  = length(var.public_subnet_cidrs)
  domain = "vpc"
  
  tags = {
    Name = "${var.project_name}-nat-eip-${count.index + 1}"
  }
  
  depends_on = [aws_internet_gateway.main]
}

# NAT Gateways
resource "aws_nat_gateway" "main" {
  count = length(var.public_subnet_cidrs)
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = {
    Name = "${var.project_name}-nat-${count.index + 1}"
  }
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

# Public Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(var.public_subnet_cidrs)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Route Tables
resource "aws_route_table" "private" {
  count = length(var.private_subnet_cidrs)
  
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }
  
  tags = {
    Name = "${var.project_name}-private-rt-${count.index + 1}"
  }
}

# Private Route Table Associations
resource "aws_route_table_association" "private" {
  count = length(var.private_subnet_cidrs)
  
  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}
```

### Step 3: Create `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Project name for resource naming"
  type        = string
  default     = "lab-vpc"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "Availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

variable "public_subnet_cidrs" {
  description = "CIDR blocks for public subnets"
  type        = list(string)
  default     = ["10.0.1.0/24", "10.0.3.0/24"]
}

variable "private_subnet_cidrs" {
  description = "CIDR blocks for private subnets"
  type        = list(string)
  default     = ["10.0.2.0/24", "10.0.4.0/24"]
}
```

### Step 4: Create `outputs.tf`

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

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ips" {
  description = "NAT Gateway public IPs"
  value       = aws_eip.nat[*].public_ip
}

output "internet_gateway_id" {
  description = "Internet Gateway ID"
  value       = aws_internet_gateway.main.id
}
```

### Step 5: Deploy

```bash
# Initialize
terraform init

# Plan
terraform plan

# Apply
terraform apply -auto-approve

# Verify outputs
terraform output
```

### Step 6: Verify in AWS Console

1. Navigate to VPC Dashboard
2. Check VPC, Subnets, Route Tables
3. Verify Internet Gateway and NAT Gateways

### Step 7: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] VPC created with correct CIDR
- [ ] 2 public and 2 private subnets in different AZs
- [ ] Internet Gateway attached
- [ ] NAT Gateways in each public subnet
- [ ] Route tables configured correctly
- [ ] All outputs displayed

---

## 🧪 Lab 2.2: Launch EC2 Instance with User Data

**Objective:** Launch an EC2 instance in the VPC with a web server.

**Duration:** 45 minutes

### Architecture

```
Internet
    │
    ▼
┌─────────────────────────────────┐
│  Public Subnet                  │
│                                 │
│  ┌───────────────────────────┐  │
│  │  EC2 Instance             │  │
│  │  - Apache Web Server      │  │
│  │  - Public IP              │  │
│  │  - Security Group         │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
```

### Step 1: Create Project Structure

```bash
mkdir -p lab-2.2-ec2
cd lab-2.2-ec2
```

### Step 2: Create `main.tf`

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

# Data source for latest Amazon Linux 2 AMI
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

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  
  tags = {
    Name = "lab-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "lab-igw"
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "lab-public-subnet"
  }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "lab-public-rt"
  }
}

# Route Table Association
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Group
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web server"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # In production, restrict to your IP
  }
  
  egress {
    description = "All outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "web-sg"
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = var.instance_type
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              
              # Create a simple web page
              cat > /var/www/html/index.html <<'HTML'
              <!DOCTYPE html>
              <html>
              <head>
                  <title>Terraform Lab 2.2</title>
                  <style>
                      body {
                          font-family: Arial, sans-serif;
                          max-width: 800px;
                          margin: 50px auto;
                          padding: 20px;
                          background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
                          color: white;
                      }
                      .container {
                          background: rgba(255, 255, 255, 0.1);
                          padding: 30px;
                          border-radius: 10px;
                          backdrop-filter: blur(10px);
                      }
                      h1 { color: #fff; }
                      .info { background: rgba(0,0,0,0.2); padding: 15px; border-radius: 5px; margin: 10px 0; }
                  </style>
              </head>
              <body>
                  <div class="container">
                      <h1>🚀 Terraform Lab 2.2 - EC2 Instance</h1>
                      <div class="info">
                          <p><strong>Instance ID:</strong> $(ec2-metadata --instance-id | cut -d " " -f 2)</p>
                          <p><strong>Instance Type:</strong> $(ec2-metadata --instance-type | cut -d " " -f 2)</p>
                          <p><strong>Availability Zone:</strong> $(ec2-metadata --availability-zone | cut -d " " -f 2)</p>
                          <p><strong>Public IP:</strong> $(ec2-metadata --public-ipv4 | cut -d " " -f 2)</p>
                          <p><strong>Private IP:</strong> $(ec2-metadata --local-ipv4 | cut -d " " -f 2)</p>
                      </div>
                      <p>✅ This web server was deployed using Terraform!</p>
                      <p>📚 Chapter 2: AWS Fundamentals</p>
                  </div>
              </body>
              </html>
              HTML
              EOF
  
  tags = {
    Name = "lab-web-server"
  }
}
```

### Step 3: Create `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}
```

### Step 4: Create `outputs.tf`

```hcl
output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP address"
  value       = aws_instance.web.public_ip
}

output "instance_public_dns" {
  description = "Public DNS name"
  value       = aws_instance.web.public_dns
}

output "web_url" {
  description = "Web server URL"
  value       = "http://${aws_instance.web.public_ip}"
}

output "ami_id" {
  description = "AMI ID used"
  value       = data.aws_ami.amazon_linux_2.id
}
```

### Step 5: Deploy and Test

```bash
# Initialize
terraform init

# Apply
terraform apply -auto-approve

# Get the web URL
terraform output web_url

# Test the web server (wait 2-3 minutes for user data to complete)
curl $(terraform output -raw web_url)

# Or open in browser
open $(terraform output -raw web_url)
```

### Step 6: Verify

1. Check EC2 instance in AWS Console
2. Verify security group rules
3. Access web page in browser
4. Check instance metadata displayed

### Step 7: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] EC2 instance launched successfully
- [ ] Web server accessible via public IP
- [ ] Security group allows HTTP and SSH
- [ ] User data script executed
- [ ] Instance metadata displayed on web page

---

## 🧪 Lab 2.3: IAM Role for EC2 with S3 Access

**Objective:** Create IAM role and attach to EC2 for S3 access.

**Duration:** 45 minutes

### Architecture

```
┌─────────────────────────────────────────┐
│  EC2 Instance                           │
│  ┌───────────────────────────────────┐  │
│  │  IAM Instance Profile             │  │
│  │  └─► IAM Role                     │  │
│  │      └─► IAM Policy (S3 Access)   │  │
│  └───────────────────────────────────┘  │
│              │                          │
│              ▼                          │
│  ┌───────────────────────────────────┐  │
│  │  Can read/write to S3 bucket      │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### Step 1: Create Project Structure

```bash
mkdir -p lab-2.3-iam
cd lab-2.3-iam
```

### Step 2: Create `main.tf`

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

# Random suffix for unique bucket name
resource "random_id" "bucket_suffix" {
  byte_length = 4
}

# S3 Bucket
resource "aws_s3_bucket" "lab_bucket" {
  bucket = "lab-terraform-${random_id.bucket_suffix.hex}"
  
  tags = {
    Name = "lab-bucket"
  }
}

# Block public access
resource "aws_s3_bucket_public_access_block" "lab_bucket" {
  bucket = aws_s3_bucket.lab_bucket.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# IAM Role for EC2
resource "aws_iam_role" "ec2_s3_role" {
  name = "ec2-s3-access-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
  
  tags = {
    Name = "ec2-s3-role"
  }
}

# IAM Policy for S3 Access
resource "aws_iam_role_policy" "s3_access_policy" {
  name = "s3-access-policy"
  role = aws_iam_role.ec2_s3_role.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.lab_bucket.arn,
          "${aws_s3_bucket.lab_bucket.arn}/*"
        ]
      }
    ]
  })
}

# IAM Instance Profile
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-s3-profile"
  role = aws_iam_role.ec2_s3_role.name
}

# VPC and Networking (simplified)
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  
  tags = {
    Name = "lab-vpc"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "lab-igw"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "${var.aws_region}a"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "lab-public-subnet"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "lab-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Group
resource "aws_security_group" "ec2" {
  name        = "ec2-sg"
  description = "Security group for EC2"
  vpc_id      = aws_vpc.main.id
  
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
    Name = "ec2-sg"
  }
}

# Data source for AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# EC2 Instance with IAM Role
resource "aws_instance" "app" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.ec2.id]
  iam_instance_profile   = aws_iam_instance_profile.ec2_profile.name
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              
              # Create test script
              cat > /home/ec2-user/test-s3.sh <<'SCRIPT'
              #!/bin/bash
              BUCKET="${aws_s3_bucket.lab_bucket.id}"
              
              echo "Testing S3 access..."
              echo "Bucket: $BUCKET"
              
              # Create test file
              echo "Hello from EC2 at $(date)" > /tmp/test.txt
              
              # Upload to S3
              echo "Uploading file to S3..."
              aws s3 cp /tmp/test.txt s3://$BUCKET/test.txt
              
              # List bucket contents
              echo "Listing bucket contents..."
              aws s3 ls s3://$BUCKET/
              
              # Download file
              echo "Downloading file from S3..."
              aws s3 cp s3://$BUCKET/test.txt /tmp/downloaded.txt
              
              # Display content
              echo "Downloaded content:"
              cat /tmp/downloaded.txt
              
              echo "✅ S3 access test completed!"
              SCRIPT
              
              chmod +x /home/ec2-user/test-s3.sh
              chown ec2-user:ec2-user /home/ec2-user/test-s3.sh
              EOF
  
  tags = {
    Name = "lab-ec2-with-iam"
  }
}
```

### Step 3: Create `variables.tf`

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}
```

### Step 4: Create `outputs.tf`

```hcl
output "bucket_name" {
  description = "S3 bucket name"
  value       = aws_s3_bucket.lab_bucket.id
}

output "bucket_arn" {
  description = "S3 bucket ARN"
  value       = aws_s3_bucket.lab_bucket.arn
}

output "iam_role_name" {
  description = "IAM role name"
  value       = aws_iam_role.ec2_s3_role.name
}

output "iam_role_arn" {
  description = "IAM role ARN"
  value       = aws_iam_role.ec2_s3_role.arn
}

output "instance_id" {
  description = "EC2 instance ID"
  value       = aws_instance.app.id
}

output "instance_public_ip" {
  description = "EC2 public IP"
  value       = aws_instance.app.public_ip
}

output "test_command" {
  description = "Command to test S3 access"
  value       = "ssh ec2-user@${aws_instance.app.public_ip} './test-s3.sh'"
}
```

### Step 5: Deploy

```bash
# Initialize
terraform init

# Apply
terraform apply -auto-approve

# View outputs
terraform output
```

### Step 6: Test S3 Access

```bash
# Get instance IP
INSTANCE_IP=$(terraform output -raw instance_public_ip)

# SSH to instance (you'll need a key pair)
# If you don't have one, add key_name to the instance resource
ssh ec2-user@$INSTANCE_IP

# Once connected, run the test script
./test-s3.sh

# You should see successful S3 operations
```

### Alternative Test (using AWS Systems Manager Session Manager)

```bash
# Get instance ID
INSTANCE_ID=$(terraform output -raw instance_id)

# Start session (no SSH key needed!)
aws ssm start-session --target $INSTANCE_ID

# Run test script
./test-s3.sh
```

### Step 7: Verify in AWS Console

1. **IAM Console:**
   - Check role `ec2-s3-access-role`
   - Verify policy attached
   - Check trust relationship

2. **EC2 Console:**
   - Verify instance has IAM role attached
   - Check instance profile

3. **S3 Console:**
   - Verify bucket created
   - Check for test.txt file

### Step 8: Cleanup

```bash
# Empty S3 bucket first
aws s3 rm s3://$(terraform output -raw bucket_name) --recursive

# Destroy infrastructure
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] S3 bucket created
- [ ] IAM role created with correct trust policy
- [ ] IAM policy allows S3 operations
- [ ] Instance profile attached to EC2
- [ ] EC2 can upload/download from S3
- [ ] Test script executes successfully

---

## 🎓 Lab Summary

### What You Learned

1. **Lab 2.1:** VPC architecture with public/private subnets
2. **Lab 2.2:** EC2 deployment with user data
3. **Lab 2.3:** IAM roles and policies for AWS services

### Key Concepts Practiced

- VPC networking and routing
- Security groups and NACLs
- EC2 instance configuration
- IAM roles and policies
- S3 bucket operations
- Infrastructure as Code patterns

### Next Steps

1. ✅ Mark labs complete in [CHECKLIST.md](../CHECKLIST.md)
2. 📝 Review [INTERVIEW.md](./INTERVIEW.md) questions
3. 🚀 Proceed to [Chapter 3](../chapter-03/)

---

## 💡 Troubleshooting Tips

### VPC Issues
- Verify CIDR blocks don't overlap
- Check route table associations
- Ensure Internet Gateway attached

### EC2 Issues
- Wait 2-3 minutes for user data to complete
- Check security group rules
- Verify subnet has public IP assignment

### IAM Issues
- Verify trust policy allows EC2 service
- Check policy permissions are correct
- Ensure instance profile attached

### Common Errors

```bash
# Error: VPC limit reached
# Solution: Delete unused VPCs or request limit increase

# Error: Instance fails to start
# Solution: Check AMI availability in your region

# Error: S3 access denied
# Solution: Verify IAM policy and role attachment
```

---

**🎉 Congratulations!** You've completed Chapter 2 labs!
