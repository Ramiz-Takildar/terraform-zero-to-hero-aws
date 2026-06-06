# Chapter 6: Labs - Provisioners and Null Resources

## 🎯 Lab Overview

These hands-on labs will help you understand provisioners, their use cases, and better alternatives.

**Prerequisites:**
- AWS account with appropriate permissions
- AWS CLI configured
- Terraform installed (v1.0+)
- SSH key pair created
- Completed Chapters 1-5 labs

**Estimated Time:** 2-3 hours total

---

## 🧪 Lab 6.1: Local-Exec and Remote-Exec Provisioners

**Objective:** Practice using local-exec and remote-exec provisioners.

**Duration:** 45 minutes

### Step 1: Create SSH Key Pair

```bash
mkdir -p lab-6.1-provisioners
cd lab-6.1-provisioners

# Generate SSH key pair
ssh-keygen -t rsa -b 4096 -f ./deployer-key -N ""
chmod 400 deployer-key
```

### Step 2: Create Configuration

**variables.tf:**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "private_key_path" {
  description = "Path to private SSH key"
  type        = string
  default     = "./deployer-key"
}

variable "public_key_path" {
  description = "Path to public SSH key"
  type        = string
  default     = "./deployer-key.pub"
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

# VPC and Networking
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  
  tags = {
    Name = "lab-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  map_public_ip_on_launch = true
  availability_zone       = data.aws_availability_zones.available.names[0]
  
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

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

data "aws_availability_zones" "available" {
  state = "available"
}

# Security Group
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web server"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    description = "HTTP"
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
    Name = "web-sg"
  }
}

# Key Pair
resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file(var.public_key_path)
}

# EC2 Instance with Provisioners
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = aws_key_pair.deployer.key_name
  
  tags = {
    Name = "web-server"
  }
  
  # Local-exec: Log instance creation
  provisioner "local-exec" {
    command = "echo 'Instance ${self.id} created at ${timestamp()}' >> instance-log.txt"
  }
  
  # Local-exec: Save instance details
  provisioner "local-exec" {
    command = <<-EOT
      echo "Instance Details:" > instance-details.txt
      echo "ID: ${self.id}" >> instance-details.txt
      echo "Public IP: ${self.public_ip}" >> instance-details.txt
      echo "Private IP: ${self.private_ip}" >> instance-details.txt
      echo "AZ: ${self.availability_zone}" >> instance-details.txt
    EOT
  }
  
  # Connection block for remote-exec
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file(var.private_key_path)
    host        = self.public_ip
    timeout     = "5m"
  }
  
  # Remote-exec: Wait for cloud-init
  provisioner "remote-exec" {
    inline = [
      "echo 'Waiting for cloud-init to complete...'",
      "cloud-init status --wait",
      "echo 'Cloud-init completed!'"
    ]
  }
  
  # Remote-exec: Install and configure Apache
  provisioner "remote-exec" {
    inline = [
      "echo 'Installing Apache...'",
      "sudo yum update -y",
      "sudo yum install -y httpd",
      "sudo systemctl start httpd",
      "sudo systemctl enable httpd",
      "echo '<h1>Hello from Terraform Provisioner!</h1>' | sudo tee /var/www/html/index.html",
      "echo '<p>Instance ID: ${self.id}</p>' | sudo tee -a /var/www/html/index.html",
      "echo '<p>Public IP: ${self.public_ip}</p>' | sudo tee -a /var/www/html/index.html",
      "echo 'Apache installed and configured!'"
    ]
  }
  
  # Local-exec: Test web server
  provisioner "local-exec" {
    command = <<-EOT
      echo 'Waiting for web server to be ready...'
      sleep 10
      curl -f http://${self.public_ip} || echo 'Web server not ready yet'
    EOT
  }
  
  # Destroy-time provisioner
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Instance ${self.id} destroyed at ${timestamp()}' >> instance-log.txt"
  }
}
```

**outputs.tf:**

```hcl
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "Public IP of the instance"
  value       = aws_instance.web.public_ip
}

output "web_url" {
  description = "URL to access the web server"
  value       = "http://${aws_instance.web.public_ip}"
}

output "ssh_command" {
  description = "SSH command to connect to instance"
  value       = "ssh -i ${var.private_key_path} ec2-user@${aws_instance.web.public_ip}"
}
```

### Step 3: Deploy Infrastructure

```bash
# Initialize
terraform init

# Plan
terraform plan

# Apply
terraform apply -auto-approve

# Wait for provisioners to complete
```

### Step 4: Verify Provisioners

```bash
# Check local-exec output
cat instance-log.txt
cat instance-details.txt

# Test web server
INSTANCE_IP=$(terraform output -raw instance_public_ip)
curl http://$INSTANCE_IP

# SSH to instance
ssh -i ./deployer-key ec2-user@$INSTANCE_IP

# On instance, verify Apache
sudo systemctl status httpd
exit
```

### Step 5: Test Destroy Provisioner

```bash
# Destroy infrastructure
terraform destroy -auto-approve

# Check destroy-time provisioner output
cat instance-log.txt
# Should show both creation and destruction timestamps
```

### ✅ Success Criteria

- [ ] SSH key pair created successfully
- [ ] Instance created with provisioners
- [ ] Local-exec created log files
- [ ] Remote-exec installed Apache
- [ ] Web server accessible
- [ ] Destroy provisioner logged destruction

---

## 🧪 Lab 6.2: File Provisioner and Configuration

**Objective:** Use file provisioner to deploy application configuration.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-6.2-file-provisioner/{scripts,configs,templates}
cd lab-6.2-file-provisioner

# Generate SSH key
ssh-keygen -t rsa -b 4096 -f ./deployer-key -N ""
chmod 400 deployer-key
```

### Step 2: Create Application Files

**configs/app.conf:**

```nginx
server {
    listen 80;
    server_name _;
    
    location / {
        root /var/www/html;
        index index.html;
    }
    
    location /api {
        proxy_pass http://localhost:3000;
    }
}
```

**scripts/setup.sh:**

```bash
#!/bin/bash
set -e

echo "Starting setup..."

# Update system
sudo yum update -y

# Install Nginx
sudo amazon-linux-extras install -y nginx1
sudo systemctl start nginx
sudo systemctl enable nginx

# Install Node.js
curl -sL https://rpm.nodesource.com/setup_16.x | sudo bash -
sudo yum install -y nodejs

echo "Setup completed!"
```

**scripts/deploy-app.sh:**

```bash
#!/bin/bash
set -e

echo "Deploying application..."

# Move configuration
sudo mv /tmp/app.conf /etc/nginx/conf.d/app.conf

# Create app directory
sudo mkdir -p /var/www/html
sudo mv /tmp/index.html /var/www/html/

# Restart Nginx
sudo systemctl restart nginx

echo "Application deployed!"
```

**templates/index.html.tpl:**

```html
<!DOCTYPE html>
<html>
<head>
    <title>${app_name}</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
        }
        .info { background: #f0f0f0; padding: 10px; margin: 10px 0; }
    </style>
</head>
<body>
    <h1>${app_name}</h1>
    <div class="info">
        <p><strong>Environment:</strong> ${environment}</p>
        <p><strong>Version:</strong> ${version}</p>
        <p><strong>Deployed:</strong> ${timestamp}</p>
    </div>
</body>
</html>
```

### Step 3: Create Terraform Configuration

**variables.tf:**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "app_name" {
  description = "Application name"
  type        = string
  default     = "MyApp"
}

variable "environment" {
  description = "Environment"
  type        = string
  default     = "development"
}

variable "app_version" {
  description = "Application version"
  type        = string
  default     = "1.0.0"
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

# Networking (simplified)
resource "aws_default_vpc" "default" {}

resource "aws_default_subnet" "default" {
  availability_zone = data.aws_availability_zones.available.names[0]
}

data "aws_availability_zones" "available" {
  state = "available"
}

# Security Group
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web server"
  vpc_id      = aws_default_vpc.default.id
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
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

# Key Pair
resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("./deployer-key.pub")
}

# EC2 Instance
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  subnet_id              = aws_default_subnet.default.id
  vpc_security_group_ids = [aws_security_group.web.id]
  key_name               = aws_key_pair.deployer.key_name
  
  tags = {
    Name        = "${var.app_name}-server"
    Environment = var.environment
  }
  
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file("./deployer-key")
    host        = self.public_ip
    timeout     = "5m"
  }
  
  # Copy setup script
  provisioner "file" {
    source      = "scripts/setup.sh"
    destination = "/tmp/setup.sh"
  }
  
  # Copy deployment script
  provisioner "file" {
    source      = "scripts/deploy-app.sh"
    destination = "/tmp/deploy-app.sh"
  }
  
  # Copy Nginx configuration
  provisioner "file" {
    source      = "configs/app.conf"
    destination = "/tmp/app.conf"
  }
  
  # Copy HTML from template
  provisioner "file" {
    content = templatefile("${path.module}/templates/index.html.tpl", {
      app_name    = var.app_name
      environment = var.environment
      version     = var.app_version
      timestamp   = timestamp()
    })
    destination = "/tmp/index.html"
  }
  
  # Run setup
  provisioner "remote-exec" {
    inline = [
      "chmod +x /tmp/setup.sh",
      "chmod +x /tmp/deploy-app.sh",
      "/tmp/setup.sh",
      "/tmp/deploy-app.sh"
    ]
  }
  
  # Verify deployment
  provisioner "local-exec" {
    command = <<-EOT
      echo "Waiting for web server..."
      sleep 15
      curl -f http://${self.public_ip} && echo "Deployment successful!" || echo "Deployment verification failed"
    EOT
  }
}
```

**outputs.tf:**

```hcl
output "web_url" {
  description = "Application URL"
  value       = "http://${aws_instance.web.public_ip}"
}

output "ssh_command" {
  description = "SSH command"
  value       = "ssh -i ./deployer-key ec2-user@${aws_instance.web.public_ip}"
}
```

### Step 4: Deploy

```bash
terraform init
terraform apply -auto-approve
```

### Step 5: Verify Deployment

```bash
# Get URL
WEB_URL=$(terraform output -raw web_url)

# Test application
curl $WEB_URL

# SSH and verify files
ssh -i ./deployer-key ec2-user@$(terraform output -raw instance_public_ip)

# On instance
ls -la /tmp/
cat /etc/nginx/conf.d/app.conf
cat /var/www/html/index.html
sudo systemctl status nginx
exit
```

### Step 6: Cleanup

```bash
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] All files copied successfully
- [ ] Scripts executed correctly
- [ ] Nginx installed and configured
- [ ] Application accessible via browser
- [ ] Template variables rendered correctly

---

## 🧪 Lab 6.3: Null Resources and Workflows

**Objective:** Use null resources for complex workflows and orchestration.

**Duration:** 45 minutes

### Step 1: Create Project Structure

```bash
mkdir -p lab-6.3-null-resources
cd lab-6.3-null-resources
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
    null = {
      source  = "hashicorp/null"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}

# Create multiple instances
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "app" {
  count = 3
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Server ${count.index + 1}</h1>" > /var/www/html/index.html
              EOF
  
  tags = {
    Name  = "app-${count.index + 1}"
    Index = count.index
  }
}

# Null resource: Wait for all instances
resource "null_resource" "wait_for_instances" {
  depends_on = [aws_instance.app]
  
  triggers = {
    instance_ids = join(",", aws_instance.app[*].id)
  }
  
  provisioner "local-exec" {
    command = <<-EOT
      echo "Waiting for all instances to be ready..."
      sleep 30
      echo "All instances should be ready now"
    EOT
  }
}

# Null resource: Health check for each instance
resource "null_resource" "health_check" {
  count = length(aws_instance.app)
  
  depends_on = [null_resource.wait_for_instances]
  
  triggers = {
    instance_id = aws_instance.app[count.index].id
    instance_ip = aws_instance.app[count.index].public_ip
  }
  
  provisioner "local-exec" {
    command = <<-EOT
      echo "Health checking instance ${count.index + 1}..."
      for i in {1..10}; do
        if curl -f -s http://${aws_instance.app[count.index].public_ip} > /dev/null; then
          echo "Instance ${count.index + 1} is healthy!"
          exit 0
        fi
        echo "Attempt $i failed, retrying..."
        sleep 5
      done
      echo "Instance ${count.index + 1} health check failed"
      exit 1
    EOT
  }
}

# Null resource: Generate inventory file
resource "null_resource" "generate_inventory" {
  depends_on = [null_resource.health_check]
  
  triggers = {
    instance_ips = join(",", aws_instance.app[*].public_ip)
  }
  
  provisioner "local-exec" {
    command = <<-EOT
      echo "[webservers]" > inventory.ini
      %{for instance in aws_instance.app~}
      echo "${instance.public_ip} ansible_user=ec2-user" >> inventory.ini
      %{endfor~}
      echo "" >> inventory.ini
      echo "[webservers:vars]" >> inventory.ini
      echo "ansible_ssh_common_args='-o StrictHostKeyChecking=no'" >> inventory.ini
      cat inventory.ini
    EOT
  }
}

# Null resource: Notification
resource "null_resource" "notify" {
  depends_on = [null_resource.generate_inventory]
  
  triggers = {
    always_run = timestamp()
  }
  
  provisioner "local-exec" {
    command = <<-EOT
      echo "==================================="
      echo "Deployment Complete!"
      echo "==================================="
      echo "Instances created: ${length(aws_instance.app)}"
      echo "All health checks passed"
      echo "Inventory file generated"
      echo "==================================="
    EOT
  }
}

# Null resource: Cleanup on destroy
resource "null_resource" "cleanup" {
  triggers = {
    instance_ids = join(",", aws_instance.app[*].id)
  }
  
  provisioner "local-exec" {
    when    = destroy
    command = <<-EOT
      echo "Cleaning up resources..."
      rm -f inventory.ini
      echo "Cleanup complete"
    EOT
  }
}
```

**outputs.tf:**

```hcl
output "instance_ips" {
  description = "Public IPs of all instances"
  value       = aws_instance.app[*].public_ip
}

output "instance_urls" {
  description = "URLs to access instances"
  value = [
    for ip in aws_instance.app[*].public_ip :
    "http://${ip}"
  ]
}

output "inventory_file" {
  description = "Ansible inventory file location"
  value       = "${path.module}/inventory.ini"
}
```

### Step 3: Deploy

```bash
terraform init
terraform apply -auto-approve

# Watch the null resource workflow
# 1. Instances created
# 2. Wait for instances
# 3. Health checks run
# 4. Inventory generated
# 5. Notification displayed
```

### Step 4: Verify Workflow

```bash
# Check inventory file
cat inventory.ini

# Test all instances
for url in $(terraform output -json instance_urls | jq -r '.[]'); do
  echo "Testing $url"
  curl $url
  echo ""
done

# Trigger re-run by changing triggers
terraform apply -auto-approve
# Only null_resource.notify should re-run (timestamp trigger)
```

### Step 5: Test Destroy Workflow

```bash
# Destroy infrastructure
terraform destroy -auto-approve

# Verify cleanup
ls -la inventory.ini
# Should not exist
```

### ✅ Success Criteria

- [ ] All instances created successfully
- [ ] Null resources executed in correct order
- [ ] Health checks passed for all instances
- [ ] Inventory file generated correctly
- [ ] Notification displayed
- [ ] Cleanup executed on destroy

---

## 🎓 Lab Summary

### What You Learned

1. **Lab 6.1:** Local-exec and remote-exec provisioners
2. **Lab 6.2:** File provisioner for configuration deployment
3. **Lab 6.3:** Null resources for workflow orchestration

### Key Concepts Practiced

- Local-exec for local commands
- Remote-exec for remote configuration
- File provisioner for copying files
- Connection blocks for SSH
- Null resources for workflows
- Provisioner triggers
- Destroy-time provisioners
- Error handling

### Important Reminders

⚠️ **Provisioners are a last resort!**

**Better alternatives:**
- Use **user_data** for instance bootstrapping
- Use **Packer** to build custom AMIs
- Use **Ansible/Chef/Puppet** for configuration management
- Use **cloud-init** for initialization
- Use **container images** for applications

**When to use provisioners:**
- No other option available
- Temporary workarounds
- Local orchestration tasks
- Integration with external systems

### Next Steps

1. ✅ Mark labs complete in [CHECKLIST.md](../CHECKLIST.md)
2. 📝 Review [INTERVIEW.md](./INTERVIEW.md) questions
3. 🚀 Proceed to [Chapter 7](../chapter-07/)

---

## 💡 Troubleshooting Tips

### SSH Connection Issues

```bash
# Test SSH connectivity
ssh -i ./deployer-key -o StrictHostKeyChecking=no ec2-user@<IP>

# Check security group rules
aws ec2 describe-security-groups --group-ids <sg-id>

# Verify key permissions
chmod 400 deployer-key
```

### Provisioner Timeout

```hcl
connection {
  type    = "ssh"
  timeout = "10m"  # Increase timeout
}
```

### Cloud-Init Not Complete

```hcl
provisioner "remote-exec" {
  inline = [
    "cloud-init status --wait",  # Wait for cloud-init
    "# Your commands here"
  ]
}
```

### File Permission Issues

```bash
# On remote instance
sudo chown ec2-user:ec2-user /tmp/file
chmod +x /tmp/script.sh
```

---

**🎉 Congratulations!** You've learned about provisioners and their alternatives!
