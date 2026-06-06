# Chapter 6: Interview Questions - Provisioners and Null Resources

## 📋 Overview

This document contains interview questions covering Terraform provisioners, null resources, and best practices for resource configuration.

**Topics Covered:**
- Provisioner Types
- Local-Exec Provisioner
- Remote-Exec Provisioner
- File Provisioner
- Null Resources
- Provisioner Alternatives
- Best Practices

---

## 📝 Provisioner Fundamentals

### Q1: What are Terraform provisioners and when should you use them?

**Answer:**

Provisioners execute scripts or commands on local or remote machines during resource creation or destruction.

**⚠️ Important:** Provisioners are a **last resort**. Use them only when no other option exists.

**Types:**
1. **local-exec:** Runs commands on the Terraform machine
2. **remote-exec:** Runs commands on the remote resource
3. **file:** Copies files to the remote resource

**When to Use:**
- Bootstrapping is unavoidable
- No native Terraform resource exists
- Temporary workarounds needed
- Integration with external systems

**Better Alternatives:**
1. **User data / cloud-init**
2. **Configuration management** (Ansible, Chef, Puppet)
3. **Custom AMIs** (Packer)
4. **Container images**
5. **Native Terraform resources**

**Example:**

```hcl
# ❌ Avoid this
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "remote-exec" {
    inline = [
      "sudo yum install -y httpd",
      "sudo systemctl start httpd"
    ]
  }
}

# ✅ Prefer this
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  user_data = <<-EOF
              #!/bin/bash
              yum install -y httpd
              systemctl start httpd
              EOF
}
```

---

### Q2: What is the difference between local-exec and remote-exec provisioners?

**Answer:**

| Aspect | local-exec | remote-exec |
|--------|-----------|-------------|
| **Execution Location** | Terraform machine | Remote resource |
| **Connection Required** | No | Yes (SSH/WinRM) |
| **Use Case** | Local tasks, scripts | Remote configuration |
| **Access** | Local filesystem | Remote filesystem |
| **Examples** | Ansible, notifications | Package installation |

**Local-Exec Example:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> inventory.txt"
  }
  
  provisioner "local-exec" {
    command = "ansible-playbook -i inventory.txt playbook.yml"
  }
}
```

**Remote-Exec Example:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name
  
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file(var.private_key_path)
    host        = self.public_ip
  }
  
  provisioner "remote-exec" {
    inline = [
      "sudo yum update -y",
      "sudo yum install -y docker",
      "sudo systemctl start docker"
    ]
  }
}
```

---

### Q3: How do you handle provisioner failures?

**Answer:**

Use the `on_failure` attribute to control behavior when provisioners fail.

**Options:**
1. **fail** (default): Stop and mark resource as tainted
2. **continue**: Log error but continue

**Fail on Error (Default):**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command    = "./critical-setup.sh"
    on_failure = fail  # Default behavior
  }
}
```

**Continue on Error:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command    = "./optional-notification.sh"
    on_failure = continue  # Don't fail if this fails
  }
  
  provisioner "local-exec" {
    command    = "./critical-setup.sh"
    on_failure = fail  # Must succeed
  }
}
```

**Best Practices:**
- Use `fail` for critical operations
- Use `continue` for optional tasks (notifications, logging)
- Always test provisioner scripts independently
- Make scripts idempotent

---

## 💻 Local-Exec Provisioner

### Q4: What are common use cases for local-exec provisioner?

**Answer:**

**1. Update Inventory Files:**

```hcl
resource "aws_instance" "web" {
  count = 3
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = <<-EOT
      echo "${self.public_ip} ansible_user=ec2-user" >> inventory.ini
    EOT
  }
}
```

**2. Run Ansible Playbooks:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = <<-EOT
      sleep 30
      ansible-playbook -i '${self.public_ip},' \
        -u ec2-user \
        --private-key ${var.private_key_path} \
        playbook.yml
    EOT
  }
}
```

**3. Send Notifications:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "curl -X POST https://hooks.slack.com/... -d '{\"text\":\"Instance ${self.id} created\"}'"
  }
}
```

**4. Update DNS Records:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "./update-dns.sh ${self.public_ip} web.example.com"
  }
}
```

**5. Generate Configuration Files:**

```hcl
resource "aws_instance" "web" {
  count = 3
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = <<-EOT
      cat >> haproxy.cfg <<EOF
      server web${count.index} ${self.private_ip}:80 check
      EOF
    EOT
  }
}
```

---

### Q5: How do you use environment variables and working directories with local-exec?

**Answer:**

**Environment Variables:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "python3 deploy.py"
    
    environment = {
      INSTANCE_ID  = self.id
      INSTANCE_IP  = self.public_ip
      ENVIRONMENT  = var.environment
      AWS_REGION   = var.aws_region
      API_KEY      = var.api_key
    }
  }
}
```

**Working Directory:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command     = "./deploy.sh ${self.public_ip}"
    working_dir = "${path.module}/scripts"
  }
}
```

**Custom Interpreter:**

```hcl
# PowerShell on Windows
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command     = "Write-Host 'Instance: ${self.id}'"
    interpreter = ["PowerShell", "-Command"]
  }
}

# Python
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command     = "print('Instance:', '${self.id}')"
    interpreter = ["python3", "-c"]
  }
}
```

---

## 🌐 Remote-Exec Provisioner

### Q6: How do you configure SSH connections for remote-exec?

**Answer:**

**Basic SSH Connection:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name
  
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file(var.private_key_path)
    host        = self.public_ip
    timeout     = "5m"
  }
  
  provisioner "remote-exec" {
    inline = ["echo 'Connected!'"]
  }
}
```

**SSH with Bastion Host:**

```hcl
resource "aws_instance" "private" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.private.id
  
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file(var.private_key_path)
    host        = self.private_ip
    
    # Bastion configuration
    bastion_host        = aws_instance.bastion.public_ip
    bastion_user        = "ec2-user"
    bastion_private_key = file(var.private_key_path)
  }
  
  provisioner "remote-exec" {
    inline = ["echo 'Connected via bastion!'"]
  }
}
```

**SSH with Password:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  
  connection {
    type     = "ssh"
    user     = "ubuntu"
    password = var.ssh_password
    host     = self.public_ip
  }
  
  provisioner "remote-exec" {
    inline = ["echo 'Connected with password!'"]
  }
}
```

**WinRM Connection (Windows):**

```hcl
resource "aws_instance" "windows" {
  ami           = data.aws_ami.windows_2019.id
  instance_type = "t2.micro"
  
  connection {
    type     = "winrm"
    user     = "Administrator"
    password = rsadecrypt(self.password_data, file(var.private_key_path))
    host     = self.public_ip
    https    = true
    insecure = true
    timeout  = "10m"
  }
  
  provisioner "remote-exec" {
    inline = [
      "powershell.exe Write-Host 'Connected to Windows!'"
    ]
  }
}
```

---

### Q7: What are the different ways to execute commands with remote-exec?

**Answer:**

**1. Inline Commands:**

```hcl
provisioner "remote-exec" {
  inline = [
    "sudo yum update -y",
    "sudo yum install -y httpd",
    "sudo systemctl start httpd",
    "sudo systemctl enable httpd",
    "echo '<h1>Hello</h1>' | sudo tee /var/www/html/index.html"
  ]
}
```

**2. Single Script:**

```hcl
provisioner "remote-exec" {
  script = "${path.module}/scripts/setup.sh"
}
```

**3. Multiple Scripts:**

```hcl
provisioner "remote-exec" {
  scripts = [
    "${path.module}/scripts/install-dependencies.sh",
    "${path.module}/scripts/configure-app.sh",
    "${path.module}/scripts/start-services.sh"
  ]
}
```

**4. Heredoc for Complex Commands:**

```hcl
provisioner "remote-exec" {
  inline = [
    <<-EOT
      sudo tee /etc/app/config.json <<EOF
      {
        "environment": "${var.environment}",
        "port": ${var.app_port},
        "database": "${aws_db_instance.main.endpoint}"
      }
      EOF
    EOT
  ]
}
```

---

## 📁 File Provisioner

### Q8: How do you use the file provisioner to copy files and directories?

**Answer:**

**Copy Single File:**

```hcl
provisioner "file" {
  source      = "app.conf"
  destination = "/tmp/app.conf"
}
```

**Copy Directory:**

```hcl
provisioner "file" {
  source      = "configs/"  # Trailing slash copies contents
  destination = "/tmp/configs"
}

provisioner "file" {
  source      = "configs"   # No trailing slash copies directory
  destination = "/tmp"
}
```

**Inline Content:**

```hcl
provisioner "file" {
  content     = "server_name ${var.domain};"
  destination = "/tmp/nginx.conf"
}
```

**Template Content:**

```hcl
provisioner "file" {
  content = templatefile("${path.module}/templates/config.tpl", {
    environment = var.environment
    app_port    = var.app_port
    db_host     = aws_db_instance.main.address
  })
  destination = "/tmp/app.conf"
}
```

**Complete Example:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name
  
  connection {
    type        = "ssh"
    user        = "ec2-user"
    private_key = file(var.private_key_path)
    host        = self.public_ip
  }
  
  # Copy configuration file
  provisioner "file" {
    source      = "nginx.conf"
    destination = "/tmp/nginx.conf"
  }
  
  # Copy SSL certificates
  provisioner "file" {
    source      = "ssl/"
    destination = "/tmp/ssl"
  }
  
  # Generate and copy app config
  provisioner "file" {
    content = templatefile("app.conf.tpl", {
      domain = var.domain
      port   = var.port
    })
    destination = "/tmp/app.conf"
  }
  
  # Move files to final location
  provisioner "remote-exec" {
    inline = [
      "sudo mv /tmp/nginx.conf /etc/nginx/nginx.conf",
      "sudo mv /tmp/ssl /etc/nginx/ssl",
      "sudo mv /tmp/app.conf /etc/app/app.conf",
      "sudo systemctl restart nginx"
    ]
  }
}
```

---

## 🔄 Null Resources

### Q9: What are null resources and when should you use them?

**Answer:**

Null resources are placeholder resources that don't create any infrastructure but can run provisioners and manage dependencies.

**Use Cases:**
1. **Run provisioners without creating resources**
2. **Orchestrate workflows**
3. **Trigger actions based on changes**
4. **Manage dependencies**
5. **Run local scripts**

**Basic Null Resource:**

```hcl
resource "null_resource" "example" {
  provisioner "local-exec" {
    command = "echo 'Hello from null resource'"
  }
}
```

**With Triggers:**

```hcl
resource "null_resource" "cluster_config" {
  # Re-run when instance IDs change
  triggers = {
    cluster_instance_ids = join(",", aws_instance.cluster[*].id)
  }
  
  provisioner "local-exec" {
    command = "./update-cluster-config.sh"
  }
}
```

**With Dependencies:**

```hcl
resource "aws_instance" "web" {
  count = 3
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}

resource "null_resource" "configure_cluster" {
  depends_on = [aws_instance.web]
  
  triggers = {
    instance_ids = join(",", aws_instance.web[*].id)
  }
  
  provisioner "local-exec" {
    command = "ansible-playbook -i inventory.ini cluster.yml"
  }
}
```

---

### Q10: How do you use triggers with null resources?

**Answer:**

Triggers determine when a null resource should be recreated and its provisioners re-run.

**Always Run (timestamp):**

```hcl
resource "null_resource" "always_run" {
  triggers = {
    always_run = timestamp()
  }
  
  provisioner "local-exec" {
    command = "echo 'Running at ${timestamp()}'"
  }
}
```

**Run on Resource Changes:**

```hcl
resource "null_resource" "on_instance_change" {
  triggers = {
    instance_id = aws_instance.web.id
    instance_ip = aws_instance.web.public_ip
  }
  
  provisioner "local-exec" {
    command = "./update-dns.sh ${aws_instance.web.public_ip}"
  }
}
```

**Run on Variable Changes:**

```hcl
resource "null_resource" "on_config_change" {
  triggers = {
    config_hash = md5(jsonencode({
      environment = var.environment
      app_version = var.app_version
      feature_flags = var.feature_flags
    }))
  }
  
  provisioner "local-exec" {
    command = "./deploy-config.sh"
  }
}
```

**Run on File Changes:**

```hcl
resource "null_resource" "on_file_change" {
  triggers = {
    config_file = filemd5("${path.module}/config.yaml")
  }
  
  provisioner "local-exec" {
    command = "./apply-config.sh"
  }
}
```

**Complex Workflow:**

```hcl
resource "aws_instance" "app" {
  count = 3
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}

# Wait for instances
resource "null_resource" "wait" {
  depends_on = [aws_instance.app]
  
  provisioner "local-exec" {
    command = "sleep 30"
  }
}

# Health check each instance
resource "null_resource" "health_check" {
  count = length(aws_instance.app)
  
  depends_on = [null_resource.wait]
  
  triggers = {
    instance_id = aws_instance.app[count.index].id
  }
  
  provisioner "local-exec" {
    command = "curl -f http://${aws_instance.app[count.index].public_ip}/health"
  }
}

# Configure load balancer
resource "null_resource" "configure_lb" {
  depends_on = [null_resource.health_check]
  
  triggers = {
    instance_ips = join(",", aws_instance.app[*].public_ip)
  }
  
  provisioner "local-exec" {
    command = "./configure-lb.sh ${join(" ", aws_instance.app[*].public_ip)}"
  }
}
```

---

## ⚙️ Provisioner Behavior

### Q11: What is the difference between creation-time and destroy-time provisioners?

**Answer:**

**Creation-Time (Default):**
Runs when resource is created.

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "echo 'Instance ${self.id} created'"
  }
}
```

**Destroy-Time:**
Runs before resource is destroyed.

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Instance ${self.id} will be destroyed'"
  }
}
```

**Both:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  # Creation-time
  provisioner "local-exec" {
    command = "./register-instance.sh ${self.id}"
  }
  
  # Destroy-time
  provisioner "local-exec" {
    when    = destroy
    command = "./deregister-instance.sh ${self.id}"
  }
}
```

**Use Cases for Destroy-Time:**
- Deregister from service discovery
- Clean up external resources
- Send notifications
- Update external systems
- Backup data

**Important Notes:**
- Destroy provisioners can only reference `self`
- They run before resource destruction
- Failures don't prevent destruction
- Limited access to resource attributes

---

### Q12: How do you handle timing issues with provisioners?

**Answer:**

**1. Wait for Cloud-Init:**

```hcl
provisioner "remote-exec" {
  inline = [
    "cloud-init status --wait",  # Wait for cloud-init
    "sudo yum install -y httpd"
  ]
}
```

**2. Add Sleep Delays:**

```hcl
provisioner "local-exec" {
  command = <<-EOT
    sleep 30
    ansible-playbook -i '${self.public_ip},' playbook.yml
  EOT
}
```

**3. Retry Logic:**

```hcl
provisioner "local-exec" {
  command = <<-EOT
    for i in {1..10}; do
      if curl -f http://${self.public_ip}/health; then
        exit 0
      fi
      echo "Attempt $i failed, retrying..."
      sleep 10
    done
    exit 1
  EOT
}
```

**4. Connection Timeout:**

```hcl
connection {
  type    = "ssh"
  timeout = "10m"  # Increase timeout
}
```

**5. Depends On:**

```hcl
resource "null_resource" "configure" {
  depends_on = [
    aws_instance.web,
    aws_security_group.web,
    aws_internet_gateway.main
  ]
  
  provisioner "local-exec" {
    command = "./configure.sh"
  }
}
```

---

## 🎯 Best Practices

### Q13: What are the best practices for using provisioners?

**Answer:**

**1. Avoid Provisioners When Possible:**

```hcl
# ❌ Don't use provisioners
resource "aws_instance" "web" {
  provisioner "remote-exec" {
    inline = ["sudo yum install -y httpd"]
  }
}

# ✅ Use user_data instead
resource "aws_instance" "web" {
  user_data = <<-EOF
              #!/bin/bash
              yum install -y httpd
              EOF
}
```

**2. Make Scripts Idempotent:**

```bash
#!/bin/bash
# ✅ Idempotent script

if systemctl is-active --quiet httpd; then
    echo "Apache already running"
    exit 0
fi

yum install -y httpd
systemctl start httpd
systemctl enable httpd
```

**3. Handle Errors Appropriately:**

```hcl
provisioner "local-exec" {
  command    = "./critical-setup.sh"
  on_failure = fail  # Must succeed
}

provisioner "local-exec" {
  command    = "./optional-notification.sh"
  on_failure = continue  # Can fail
}
```

**4. Use Null Resources for Orchestration:**

```hcl
resource "null_resource" "workflow" {
  depends_on = [aws_instance.web]
  
  triggers = {
    instance_id = aws_instance.web.id
  }
  
  provisioner "local-exec" {
    command = "./orchestrate.sh"
  }
}
```

**5. Prefer Configuration Management:**

```hcl
resource "aws_instance" "web" {
  user_data = <<-EOF
              #!/bin/bash
              # Install Ansible
              yum install -y ansible
              
              # Pull and run playbook
              ansible-pull -U https://github.com/org/repo.git
              EOF
}
```

---

### Q14: What are better alternatives to provisioners?

**Answer:**

**1. User Data / Cloud-Init:**

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
              EOF
}
```

**2. Custom AMIs with Packer:**

```json
{
  "builders": [{
    "type": "amazon-ebs",
    "ami_name": "web-server-{{timestamp}}",
    "instance_type": "t2.micro",
    "source_ami_filter": {
      "filters": {
        "name": "amzn2-ami-hvm-*"
      }
    }
  }],
  "provisioners": [{
    "type": "shell",
    "script": "setup.sh"
  }]
}
```

```hcl
# Use custom AMI
data "aws_ami" "custom" {
  most_recent = true
  owners      = ["self"]
  
  filter {
    name   = "name"
    values = ["web-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.custom.id
  instance_type = "t2.micro"
}
```

**3. Configuration Management:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  user_data = <<-EOF
              #!/bin/bash
              # Bootstrap Ansible
              yum install -y ansible
              ansible-pull -U https://github.com/org/ansible.git
              EOF
}
```

**4. Container Images:**

```hcl
resource "aws_ecs_task_definition" "app" {
  family = "app"
  
  container_definitions = jsonencode([{
    name  = "app"
    image = "myapp:latest"  # Pre-configured image
  }])
}
```

**5. Native Terraform Resources:**

```hcl
# Instead of provisioner to create S3 object
resource "aws_s3_object" "config" {
  bucket  = aws_s3_bucket.main.id
  key     = "config.json"
  content = jsonencode({
    environment = var.environment
  })
}
```

---

### Q15: How do you debug provisioner issues?

**Answer:**

**1. Enable Detailed Logging:**

```bash
export TF_LOG=DEBUG
terraform apply
```

**2. Test Scripts Independently:**

```bash
# Test script locally first
./setup.sh

# Test SSH connection
ssh -i key.pem ec2-user@<ip>
```

**3. Add Debug Output:**

```hcl
provisioner "remote-exec" {
  inline = [
    "set -x",  # Enable bash debug mode
    "echo 'Starting setup...'",
    "sudo yum install -y httpd",
    "echo 'Setup complete!'"
  ]
}
```

**4. Use Smaller Steps:**

```hcl
# Break into smaller provisioners
provisioner "remote-exec" {
  inline = ["sudo yum update -y"]
}

provisioner "remote-exec" {
  inline = ["sudo yum install -y httpd"]
}

provisioner "remote-exec" {
  inline = ["sudo systemctl start httpd"]
}
```

**5. Check Connection:**

```hcl
connection {
  type        = "ssh"
  user        = "ec2-user"
  private_key = file(var.private_key_path)
  host        = self.public_ip
  timeout     = "10m"
  
  # Add for debugging
  agent = false
}
```

**6. Verify Prerequisites:**

```bash
# Check security group
aws ec2 describe-security-groups --group-ids <sg-id>

# Check instance state
aws ec2 describe-instances --instance-ids <instance-id>

# Test network connectivity
nc -zv <ip> 22
```

---

## 📚 Summary

**Key Topics Covered:**
- Provisioner types and use cases
- Local-exec for local commands
- Remote-exec for remote configuration
- File provisioner for copying files
- Null resources for workflows
- Provisioner alternatives
- Best practices and debugging

**Important Reminders:**
1. **Provisioners are a last resort**
2. **Prefer user_data over remote-exec**
3. **Use Packer for custom AMIs**
4. **Use configuration management tools**
5. **Make scripts idempotent**
6. **Handle errors appropriately**
7. **Test scripts independently**
8. **Use null resources for orchestration**

**Next Steps:**
1. Practice labs in [LABS.md](./LABS.md)
2. Explore provisioner alternatives
3. Proceed to [Chapter 7](../chapter-07/)

---

**💡 Pro Tips:**
- Always have a rollback plan
- Test provisioners in dev first
- Use version control for scripts
- Document provisioner dependencies
- Monitor provisioner execution time
- Consider using Terraform Cloud for better logging
