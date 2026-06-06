# Chapter 2: Interview Questions - AWS Fundamentals

## 📋 Overview

This document contains interview questions covering AWS fundamentals including VPC, EC2, IAM, Security Groups, and networking concepts.

**Topics Covered:**
- VPC and Networking
- EC2 Instances
- IAM (Identity and Access Management)
- Security Groups and NACLs
- AWS Regions and Availability Zones

---

## 🌐 VPC and Networking

### Q1: What is a VPC and why is it important?

**Answer:**

A VPC (Virtual Private Cloud) is a logically isolated virtual network in AWS cloud where you can launch AWS resources.

**Key Features:**
- **Isolation:** Your own private network space
- **Control:** Full control over IP addressing, subnets, routing
- **Security:** Network-level security with security groups and NACLs
- **Connectivity:** Connect to on-premises networks via VPN/Direct Connect

**Example:**
```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "production-vpc"
  }
}
```

**Why Important:**
- Network isolation for security
- Custom IP addressing schemes
- Control over network topology
- Foundation for all AWS resources

---

### Q2: What's the difference between public and private subnets?

**Answer:**

| Aspect | Public Subnet | Private Subnet |
|--------|---------------|----------------|
| **Internet Access** | Direct via Internet Gateway | Via NAT Gateway/Instance |
| **Public IP** | Instances get public IPs | No public IPs |
| **Route Table** | Routes to Internet Gateway | Routes to NAT Gateway |
| **Use Case** | Web servers, load balancers | Databases, app servers |
| **Security** | More exposed | More secure |

**Public Subnet Example:**
```hcl
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true  # Key difference
  
  tags = {
    Name = "public-subnet"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id  # Direct to IGW
  }
}
```

**Private Subnet Example:**
```hcl
resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  # No map_public_ip_on_launch
  
  tags = {
    Name = "private-subnet"
  }
}

resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id  # Via NAT
  }
}
```

---

### Q3: What is an Internet Gateway and how does it work?

**Answer:**

An Internet Gateway (IGW) is a horizontally scaled, redundant, and highly available VPC component that allows communication between your VPC and the internet.

**Key Characteristics:**
- **Managed Service:** AWS handles scaling and availability
- **No Bandwidth Constraints:** Scales automatically
- **One per VPC:** Only one IGW can be attached to a VPC
- **Stateless:** Doesn't track connections

**Terraform Example:**
```hcl
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "main-igw"
  }
}

# Route table pointing to IGW
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
}
```

---

### Q4: What is a NAT Gateway and when would you use it?

**Answer:**

A NAT (Network Address Translation) Gateway allows instances in private subnets to access the internet while preventing inbound connections from the internet.

**Use Cases:**
- Software updates for private instances
- Downloading packages/dependencies
- API calls to external services
- Outbound-only internet access

**Terraform Example:**
```hcl
# Elastic IP for NAT Gateway
resource "aws_eip" "nat" {
  domain = "vpc"
}

# NAT Gateway in public subnet
resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public.id
  
  depends_on = [aws_internet_gateway.main]
}

# Private route table using NAT
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }
}
```

---

### Q5: Explain CIDR notation and subnet sizing.

**Answer:**

CIDR (Classless Inter-Domain Routing) notation represents IP address ranges.

**Format:** `IP_ADDRESS/PREFIX_LENGTH`

**Examples:**
- `10.0.0.0/16` = 65,536 IPs
- `10.0.1.0/24` = 256 IPs
- `10.0.1.0/28` = 16 IPs

**Subnet Sizing Table:**

| CIDR | Available IPs | AWS Usable IPs* | Use Case |
|------|---------------|-----------------|----------|
| /16 | 65,536 | 65,531 | Large VPC |
| /20 | 4,096 | 4,091 | Medium subnet |
| /24 | 256 | 251 | Standard subnet |
| /28 | 16 | 11 | Small subnet |

*AWS reserves 5 IPs per subnet

---

## 💻 EC2 Instances

### Q6: What are EC2 instance types and how do you choose one?

**Answer:**

EC2 instance types are optimized for different workloads.

**Instance Families:**

| Family | Purpose | Example Types | Use Case |
|--------|---------|---------------|----------|
| **T2/T3** | Burstable performance | t2.micro, t3.medium | Web servers, dev/test |
| **M5** | General purpose | m5.large, m5.xlarge | Balanced workloads |
| **C5** | Compute optimized | c5.xlarge | CPU-intensive apps |
| **R5** | Memory optimized | r5.large | In-memory databases |
| **I3** | Storage optimized | i3.large | NoSQL databases |
| **P3** | GPU instances | p3.2xlarge | ML/AI workloads |

---

### Q7: What is EC2 user data and how is it used?

**Answer:**

User data is a script that runs automatically when an EC2 instance launches.

**Terraform Example:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from Terraform</h1>" > /var/www/html/index.html
              EOF
}
```

---

### Q8: What are EC2 instance states and lifecycle?

**Answer:**

**Instance States:**
- **Pending:** Instance is launching
- **Running:** Instance is active (billed)
- **Stopping:** Shutting down
- **Stopped:** Shut down (EBS only billed)
- **Shutting-down:** Terminating
- **Terminated:** Deleted

---

## 🔒 Security Groups and NACLs

### Q9: What's the difference between Security Groups and NACLs?

**Answer:**

| Feature | Security Group | Network ACL |
|---------|----------------|-------------|
| **Level** | Instance level | Subnet level |
| **State** | Stateful | Stateless |
| **Rules** | Allow only | Allow and Deny |
| **Rule Processing** | All rules evaluated | Rules in order |
| **Return Traffic** | Automatic | Must be explicit |

**Security Group Example:**
```hcl
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
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
```

---

### Q10: How do you reference security groups in Terraform?

**Answer:**

**Self-Referencing:**
```hcl
resource "aws_security_group" "app" {
  name   = "app-sg"
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    self      = true  # References itself
  }
}
```

**Cross-Referencing:**
```hcl
resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
}
```

---

## 🔐 IAM

### Q11: What are IAM roles and when should you use them?

**Answer:**

IAM roles are identities with permissions that can be assumed by AWS services or users.

**EC2 Instance Role Example:**
```hcl
resource "aws_iam_role" "ec2_role" {
  name = "ec2-app-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ec2.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-profile"
  role = aws_iam_role.ec2_role.name
}
```

---

### Q12: What's the difference between IAM policies, roles, and users?

**Answer:**

| Component | Purpose | Has Credentials |
|-----------|---------|-----------------|
| **User** | Person or application | Yes (password/keys) |
| **Role** | Set of permissions | No (temporary) |
| **Policy** | Permission document | N/A |
| **Group** | Collection of users | No |

---

### Q13: How do you implement least privilege with IAM?

**Answer:**

**Resource-Specific Permissions:**
```hcl
resource "aws_iam_policy" "specific" {
  name = "specific-access"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Action = ["s3:GetObject", "s3:PutObject"]
      Resource = "arn:aws:s3:::my-bucket/app-data/*"
    }]
  })
}
```

**Tag-Based Access:**
```hcl
resource "aws_iam_policy" "tag_based" {
  name = "tag-based-access"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["ec2:StartInstances", "ec2:StopInstances"]
      Resource = "*"
      Condition = {
        StringEquals = {
          "ec2:ResourceTag/Environment" = "development"
        }
      }
    }]
  })
}
```

---

## 🌍 Regions and Availability Zones

### Q14: What are AWS Regions and Availability Zones?

**Answer:**

**Regions:**
- Geographic areas (e.g., us-east-1, eu-west-1)
- Completely independent
- Choose based on: latency, compliance, cost

**Availability Zones:**
- Isolated data centers within a region
- Connected via low-latency links
- Minimum 3 AZs per region

**Multi-AZ Example:**
```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  count = 3
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

---

### Q15: How do you handle AWS provider configuration in Terraform?

**Answer:**

**Basic Configuration:**
```hcl
terraform {
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
      Environment = "production"
      ManagedBy   = "Terraform"
    }
  }
}
```

**Multiple Regions:**
```hcl
provider "aws" {
  alias  = "west"
  region = "us-west-2"
}

resource "aws_instance" "west" {
  provider = aws.west
  # ...
}
```

**Authentication Methods:**
1. Environment variables
2. AWS credentials file
3. IAM role (recommended for EC2)
4. Assume role

---

## 🎓 Additional Questions

### Q16: What is the difference between EBS and Instance Store?

**Answer:**

| Feature | EBS | Instance Store |
|---------|-----|----------------|
| **Persistence** | Persistent | Ephemeral |
| **Lifecycle** | Independent of instance | Tied to instance |
| **Backup** | Snapshots supported | No snapshots |
| **Performance** | Network-attached | Physically attached |
| **Use Case** | Root volumes, databases | Temporary data, cache |

---

### Q17: How do you implement high availability in AWS?

**Answer:**

**Strategies:**
1. Deploy across multiple AZs
2. Use Auto Scaling Groups
3. Implement Load Balancers
4. Use managed services (RDS Multi-AZ)
5. Regular backups and snapshots

**Example:**
```hcl
resource "aws_autoscaling_group" "web" {
  name                = "web-asg"
  vpc_zone_identifier = aws_subnet.public[*].id  # Multiple AZs
  min_size            = 2
  max_size            = 6
  desired_capacity    = 2
  
  launch_template {
    id      = aws_launch_template.web.id
    version = "$Latest"
  }
}
```

---

### Q18: What are VPC Endpoints and when to use them?

**Answer:**

VPC Endpoints allow private connections to AWS services without internet gateway.

**Types:**
- **Interface Endpoints:** ENI with private IP
- **Gateway Endpoints:** Route table target (S3, DynamoDB)

**Benefits:**
- No internet gateway needed
- Lower data transfer costs
- Enhanced security

**Example:**
```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.us-east-1.s3"
  
  route_table_ids = [aws_route_table.private.id]
}
```

---

### Q19: How do you secure data in transit and at rest in AWS?

**Answer:**

**In Transit:**
- Use HTTPS/TLS for API calls
- Enable SSL/TLS for databases
- Use VPN for on-premises connections
- Enable encryption for ELB

**At Rest:**
- Enable EBS encryption
- Use S3 encryption (SSE-S3, SSE-KMS)
- Enable RDS encryption
- Use KMS for key management

**Example:**
```hcl
resource "aws_ebs_volume" "encrypted" {
  availability_zone = "us-east-1a"
  size              = 100
  encrypted         = true
  kms_key_id        = aws_kms_key.main.arn
}
```

---

### Q20: What is the AWS Shared Responsibility Model?

**Answer:**

**AWS Responsibility (Security OF the Cloud):**
- Physical infrastructure
- Hardware and networking
- Hypervisor
- Managed services

**Customer Responsibility (Security IN the Cloud):**
- Data encryption
- IAM and access management
- Network configuration
- Application security
- OS patches (for EC2)

**Example Security Implementation:**
```hcl
# Customer responsibility: Secure configuration
resource "aws_instance" "secure" {
  ami           = data.aws_ami.hardened.id
  instance_type = "t2.micro"
  
  # Enable detailed monitoring
  monitoring = true
  
  # Encrypted root volume
  root_block_device {
    encrypted = true
  }
  
  # Restrict access
  vpc_security_group_ids = [aws_security_group.restricted.id]
  
  # Use IAM role instead of keys
  iam_instance_profile = aws_iam_instance_profile.app.name
}
```

---

## 📚 Summary

**Key Topics Covered:**
- VPC architecture and components
- EC2 instance types and lifecycle
- Security Groups vs NACLs
- IAM roles, policies, and best practices
- Multi-AZ and multi-region deployments
- AWS provider configuration
- Security best practices

**Next Steps:**
1. Practice labs in [LABS.md](./LABS.md)
2. Review Terraform documentation
3. Proceed to [Chapter 3](../chapter-03/)

---

**💡 Pro Tips:**
- Always use least privilege for IAM
- Deploy across multiple AZs for HA
- Use managed services when possible
- Enable encryption by default
- Tag all resources consistently
- Monitor costs with AWS Cost Explorer
