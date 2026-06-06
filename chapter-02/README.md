# Chapter 2: AWS Fundamentals

## 📚 Learning Objectives

By the end of this chapter, you will:
- Understand AWS core services (VPC, EC2, IAM)
- Create and configure VPCs with subnets
- Launch and manage EC2 instances
- Implement IAM roles and policies
- Understand AWS networking basics
- Apply AWS best practices

**Prerequisites:** Chapter 1 completed  
**Estimated Time:** 3 days  
**Labs:** 3 hands-on exercises

---

## 🌐 AWS Regions and Availability Zones

### Architecture

```
┌─────────────────────────────────────────────────────┐
│                   AWS Region                        │
│                  (us-east-1)                        │
│                                                     │
│  ┌──────────────────┐  ┌──────────────────┐        │
│  │  Availability    │  │  Availability    │        │
│  │  Zone A          │  │  Zone B          │        │
│  │  (us-east-1a)    │  │  (us-east-1b)    │        │
│  │                  │  │                  │        │
│  │  ┌────────────┐  │  │  ┌────────────┐  │        │
│  │  │ Data Center│  │  │  │ Data Center│  │        │
│  │  └────────────┘  │  │  └────────────┘  │        │
│  └──────────────────┘  └──────────────────┘        │
└─────────────────────────────────────────────────────┘
```

**Key Concepts:**
- **Region:** Geographic area (e.g., us-east-1, eu-west-1)
- **Availability Zone:** Isolated data center within region
- **High Availability:** Deploy across multiple AZs

---

## 🏗️ VPC (Virtual Private Cloud)

### What is a VPC?

A VPC is your own isolated network in AWS cloud.

```
┌─────────────────────────────────────────────────────┐
│                VPC (10.0.0.0/16)                    │
│                                                     │
│  ┌──────────────────────┐  ┌──────────────────────┐│
│  │  Public Subnet       │  │  Private Subnet      ││
│  │  (10.0.1.0/24)       │  │  (10.0.2.0/24)       ││
│  │                      │  │                      ││
│  │  ┌────────────────┐  │  │  ┌────────────────┐ ││
│  │  │  EC2 Instance  │  │  │  │  RDS Database  │ ││
│  │  │  (Web Server)  │  │  │  │                │ ││
│  │  └────────────────┘  │  │  └────────────────┘ ││
│  │         │            │  │         │           ││
│  └─────────┼────────────┘  └─────────┼───────────┘│
│            │                         │            │
│            ▼                         ▼            │
│    ┌──────────────┐         ┌──────────────┐     │
│    │ Internet     │         │ NAT Gateway  │     │
│    │ Gateway      │         │              │     │
│    └──────────────┘         └──────────────┘     │
└─────────────────────────────────────────────────────┘
```

### VPC Components

| Component | Purpose |
|-----------|---------|
| **CIDR Block** | IP address range (e.g., 10.0.0.0/16) |
| **Subnets** | Subdivisions of VPC |
| **Internet Gateway** | Connect to internet |
| **NAT Gateway** | Private subnet internet access |
| **Route Tables** | Control traffic routing |
| **Security Groups** | Instance-level firewall |
| **Network ACLs** | Subnet-level firewall |

### Terraform VPC Example

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "public-subnet"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "main-igw"
  }
}
```

---

## 💻 EC2 (Elastic Compute Cloud)

### Instance Types

| Family | Use Case | Example |
|--------|----------|---------|
| **t2/t3** | General purpose, burstable | t2.micro, t3.small |
| **m5** | Balanced compute/memory | m5.large |
| **c5** | Compute optimized | c5.xlarge |
| **r5** | Memory optimized | r5.2xlarge |
| **p3** | GPU instances | p3.2xlarge |

### EC2 Lifecycle

```
┌──────────┐
│ Pending  │ ← Instance launching
└────┬─────┘
     │
     ▼
┌──────────┐
│ Running  │ ← Instance active
└────┬─────┘
     │
     ├──► Stopping ──► Stopped ──► Starting ──► Running
     │
     └──► Shutting Down ──► Terminated
```

### Terraform EC2 Example

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id
  
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "Hello from Terraform" > /var/www/html/index.html
              EOF
  
  tags = {
    Name = "web-server"
  }
}
```

---

## 🔒 Security Groups

### Firewall Rules

```
┌─────────────────────────────────────────────┐
│         Security Group (Stateful)           │
│                                             │
│  Inbound Rules:                             │
│  ┌─────────────────────────────────────┐   │
│  │ Port 80  (HTTP)   from 0.0.0.0/0   │   │
│  │ Port 443 (HTTPS)  from 0.0.0.0/0   │   │
│  │ Port 22  (SSH)    from My IP       │   │
│  └─────────────────────────────────────┘   │
│                                             │
│  Outbound Rules:                            │
│  ┌─────────────────────────────────────┐   │
│  │ All traffic to 0.0.0.0/0           │   │
│  └─────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

### Terraform Security Group

```hcl
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "SSH from my IP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["1.2.3.4/32"]  # Your IP
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
```

---

## 🔐 IAM (Identity and Access Management)

### IAM Components

```
┌─────────────────────────────────────────────┐
│                  IAM                        │
│                                             │
│  ┌──────────┐      ┌──────────┐            │
│  │  Users   │      │  Groups  │            │
│  └────┬─────┘      └────┬─────┘            │
│       │                 │                  │
│       └────────┬────────┘                  │
│                │                           │
│                ▼                           │
│         ┌──────────┐                       │
│         │ Policies │                       │
│         └────┬─────┘                       │
│              │                             │
│              ▼                             │
│         ┌──────────┐                       │
│         │  Roles   │                       │
│         └──────────┘                       │
└─────────────────────────────────────────────┘
```

### IAM Policy Example

```hcl
resource "aws_iam_role" "ec2_role" {
  name = "ec2-role"
  
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
}

resource "aws_iam_role_policy" "ec2_policy" {
  name = "ec2-policy"
  role = aws_iam_role.ec2_role.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::my-bucket",
          "arn:aws:s3:::my-bucket/*"
        ]
      }
    ]
  })
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-profile"
  role = aws_iam_role.ec2_role.name
}
```

---

## 📊 AWS Provider Configuration

### Basic Configuration

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
      Environment = "dev"
      ManagedBy   = "Terraform"
      Project     = "learning"
    }
  }
}
```

### Multiple Regions

```hcl
provider "aws" {
  alias  = "us_east"
  region = "us-east-1"
}

provider "aws" {
  alias  = "us_west"
  region = "us-west-2"
}

resource "aws_instance" "east" {
  provider = aws.us_east
  # ...
}

resource "aws_instance" "west" {
  provider = aws.us_west
  # ...
}
```

---

## 📖 Key Takeaways

1. **VPC:** Your isolated network in AWS
2. **Subnets:** Public (internet access) vs Private (no direct internet)
3. **EC2:** Virtual servers in the cloud
4. **Security Groups:** Stateful firewall at instance level
5. **IAM:** Control who can do what in AWS
6. **Regions/AZs:** Geographic distribution for HA
7. **Default Tags:** Apply tags to all resources

---

## ❓ Common Patterns

### Multi-Tier Architecture

```
Internet
    │
    ▼
┌─────────────────────────────────┐
│  Public Subnet (Web Tier)       │
│  - Load Balancer                │
│  - Web Servers                  │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Private Subnet (App Tier)      │
│  - Application Servers          │
└────────────┬────────────────────┘
             │
             ▼
┌─────────────────────────────────┐
│  Private Subnet (Data Tier)     │
│  - RDS Database                 │
└─────────────────────────────────┘
```

---

## 🔗 Next Steps

1. Review theory above
2. Complete [Lab 2.1](./LABS.md) - VPC Creation
3. Complete [Lab 2.2](./LABS.md) - EC2 Instance
4. Complete [Lab 2.3](./LABS.md) - IAM Roles
5. Update [CHECKLIST.md](../CHECKLIST.md)

**Next Chapter:** [Chapter 3: State Management](../chapter-03/)
