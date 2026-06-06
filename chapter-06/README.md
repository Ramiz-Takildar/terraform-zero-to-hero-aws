# Chapter 6: Provisioners and Null Resources

## 📚 Learning Objectives

By the end of this chapter, you will:
- Understand when to use provisioners
- Master local-exec and remote-exec provisioners
- Use file provisioners for configuration
- Implement connection blocks
- Work with null resources
- Handle provisioner failures
- Understand provisioner alternatives

**Prerequisites:** Chapters 1-5 completed  
**Estimated Time:** 2 days  
**Labs:** 3 hands-on exercises

---

## ⚠️ Important Note

**Provisioners are a last resort!** Terraform recommends using them only when no other option exists. Better alternatives include:
- User data / cloud-init
- Configuration management tools (Ansible, Chef, Puppet)
- Custom AMIs / images
- Container orchestration

Use provisioners when:
- Bootstrapping is unavoidable
- No native Terraform resource exists
- Temporary workarounds are needed

---

## 📝 Provisioner Basics

### What are Provisioners?

Provisioners execute scripts or commands on local or remote machines during resource creation or destruction.

**Types:**
1. **local-exec:** Runs commands on the machine running Terraform
2. **remote-exec:** Runs commands on the remote resource
3. **file:** Copies files to the remote resource

**Basic Syntax:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "echo ${self.public_ip} >> ip_addresses.txt"
  }
}
```

---

## 💻 Local-Exec Provisioner

Executes commands on the machine running Terraform.

### Basic Usage

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "echo Instance ${self.id} created at ${timestamp()}"
  }
}
```

### Common Use Cases

**1. Update Inventory File:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = <<-EOT
      echo "[webservers]" > inventory.ini
      echo "${self.public_ip} ansible_user=ec2-user" >> inventory.ini
    EOT
  }
}
```

**2. Trigger External Script:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "./scripts/notify-slack.sh ${self.id} ${self.public_ip}"
  }
}
```

**3. Run Ansible Playbook:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name
  
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

**4. Working Directory:**

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

**5. Environment Variables:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "python3 notify.py"
    
    environment = {
      INSTANCE_ID = self.id
      INSTANCE_IP = self.public_ip
      ENVIRONMENT = var.environment
    }
  }
}
```

**6. Interpreter:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command     = "Write-Host 'Instance created: ${self.id}'"
    interpreter = ["PowerShell", "-Command"]
  }
}
```

---

## 🌐 Remote-Exec Provisioner

Executes commands on the remote resource after it's created.

### Connection Block

Required for remote-exec and file provisioners:

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
      "sudo yum install -y httpd",
      "sudo systemctl start httpd",
      "sudo systemctl enable httpd"
    ]
  }
}
```

### Inline Commands

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
      "echo 'Starting configuration...'",
      "sudo yum update -y",
      "sudo yum install -y docker",
      "sudo systemctl start docker",
      "sudo systemctl enable docker",
      "sudo usermod -aG docker ec2-user",
      "echo 'Configuration complete!'"
    ]
  }
}
```

### Script Execution

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
    script = "${path.module}/scripts/setup.sh"
  }
}
```

### Multiple Scripts

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
    scripts = [
      "${path.module}/scripts/install-dependencies.sh",
      "${path.module}/scripts/configure-app.sh",
      "${path.module}/scripts/start-services.sh"
    ]
  }
}
```

### Windows Connection

```hcl
resource "aws_instance" "windows" {
  ami           = data.aws_ami.windows_2019.id
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name
  
  connection {
    type     = "winrm"
    user     = "Administrator"
    password = rsadecrypt(self.password_data, file(var.private_key_path))
    host     = self.public_ip
    https    = true
    insecure = true
  }
  
  provisioner "remote-exec" {
    inline = [
      "powershell.exe Install-WindowsFeature -Name Web-Server",
      "powershell.exe New-Item -Path C:\\inetpub\\wwwroot\\index.html -ItemType File -Force",
      "powershell.exe Set-Content -Path C:\\inetpub\\wwwroot\\index.html -Value '<h1>Hello from Windows</h1>'"
    ]
  }
}
```

---

## 📁 File Provisioner

Copies files or directories to the remote resource.

### Copy Single File

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
  
  provisioner "file" {
    source      = "app.conf"
    destination = "/tmp/app.conf"
  }
  
  provisioner "remote-exec" {
    inline = [
      "sudo mv /tmp/app.conf /etc/app/app.conf",
      "sudo systemctl restart app"
    ]
  }
}
```

### Copy Directory

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
  
  provisioner "file" {
    source      = "configs/"
    destination = "/tmp/configs"
  }
}
```

### Inline Content

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
  
  provisioner "file" {
    content = templatefile("${path.module}/templates/config.tpl", {
      environment = var.environment
      app_port    = var.app_port
    })
    destination = "/tmp/app.conf"
  }
}
```

### Multiple Files

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
  
  provisioner "file" {
    source      = "app.conf"
    destination = "/tmp/app.conf"
  }
  
  provisioner "file" {
    source      = "nginx.conf"
    destination = "/tmp/nginx.conf"
  }
  
  provisioner "file" {
    source      = "ssl/"
    destination = "/tmp/ssl"
  }
}
```

---

## 🔄 Null Resource

A placeholder resource that doesn't create any infrastructure but can run provisioners.

### Basic Null Resource

```hcl
resource "null_resource" "example" {
  provisioner "local-exec" {
    command = "echo 'Hello from null resource'"
  }
}
```

### Triggers

Re-run provisioners when values change:

```hcl
resource "null_resource" "cluster" {
  triggers = {
    cluster_instance_ids = join(",", aws_instance.cluster[*].id)
  }
  
  provisioner "local-exec" {
    command = "echo 'Cluster configuration changed'"
  }
}
```

### Depends On

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}

resource "null_resource" "configure_web" {
  depends_on = [aws_instance.web]
  
  triggers = {
    instance_id = aws_instance.web.id
  }
  
  provisioner "local-exec" {
    command = "ansible-playbook -i '${aws_instance.web.public_ip},' playbook.yml"
  }
}
```

### Complex Workflow

```hcl
resource "aws_instance" "app" {
  count = 3
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}

resource "null_resource" "health_check" {
  count = length(aws_instance.app)
  
  triggers = {
    instance_id = aws_instance.app[count.index].id
  }
  
  provisioner "local-exec" {
    command = <<-EOT
      for i in {1..30}; do
        if curl -f http://${aws_instance.app[count.index].public_ip}/health; then
          echo "Instance ${count.index} is healthy"
          exit 0
        fi
        echo "Waiting for instance ${count.index}..."
        sleep 10
      done
      echo "Instance ${count.index} failed health check"
      exit 1
    EOT
  }
}
```

---

## ⚙️ Provisioner Behavior

### Creation-Time Provisioners

Run when resource is created (default):

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command = "echo 'Instance created'"
  }
}
```

### Destroy-Time Provisioners

Run before resource is destroyed:

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

### Failure Behavior

**Continue on Failure (default: fail):**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command     = "exit 1"  # This will fail
    on_failure  = continue  # But Terraform continues
  }
}
```

**Fail on Error:**

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  provisioner "local-exec" {
    command    = "./critical-setup.sh"
    on_failure = fail  # Terraform stops (default)
  }
}
```

---

## 🎯 Best Practices

### 1. Prefer User Data

**Instead of:**
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
      "sudo yum install -y httpd",
      "sudo systemctl start httpd"
    ]
  }
}
```

**Use:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  user_data = <<-EOF
              #!/bin/bash
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              EOF
}
```

### 2. Use Configuration Management

**Instead of provisioners:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  user_data = <<-EOF
              #!/bin/bash
              # Install Ansible
              yum install -y ansible
              
              # Pull configuration
              ansible-pull -U https://github.com/org/ansible-repo.git
              EOF
}
```

### 3. Build Custom AMIs

**Use Packer to create AMIs:**
```json
{
  "builders": [{
    "type": "amazon-ebs",
    "ami_name": "web-server-{{timestamp}}",
    "instance_type": "t2.micro",
    "source_ami_filter": {
      "filters": {
        "name": "amzn2-ami-hvm-*-x86_64-gp2"
      }
    }
  }],
  "provisioners": [{
    "type": "shell",
    "script": "setup.sh"
  }]
}
```

**Then use in Terraform:**
```hcl
data "aws_ami" "custom_web" {
  most_recent = true
  owners      = ["self"]
  
  filter {
    name   = "name"
    values = ["web-server-*"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.custom_web.id
  instance_type = "t2.micro"
  # No provisioners needed!
}
```

### 4. Handle Timing Issues

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
    inline = [
      "cloud-init status --wait",  # Wait for cloud-init
      "sudo yum install -y httpd"
    ]
  }
}
```

### 5. Idempotent Scripts

```bash
#!/bin/bash
# setup.sh - Idempotent installation script

# Check if already installed
if systemctl is-active --quiet httpd; then
    echo "Apache already running"
    exit 0
fi

# Install and configure
yum install -y httpd
systemctl start httpd
systemctl enable httpd

echo "Apache installed and started"
```

---

## 📖 Key Takeaways

1. **Provisioners are a last resort** - use alternatives when possible
2. **local-exec** runs on Terraform machine
3. **remote-exec** runs on remote resource
4. **file** provisioner copies files
5. **null_resource** for workflows without infrastructure
6. **Prefer user data** over remote-exec
7. **Use configuration management** for complex setups
8. **Build custom images** with Packer
9. **Handle failures** appropriately
10. **Make scripts idempotent**

---

## 🔗 Next Steps

1. Review theory above
2. Complete [Lab 6.1](./LABS.md) - Local and Remote Exec
3. Complete [Lab 6.2](./LABS.md) - File Provisioner
4. Complete [Lab 6.3](./LABS.md) - Null Resources
5. Update [CHECKLIST.md](../CHECKLIST.md)

**Next Chapter:** [Chapter 7: Data Sources and Functions](../chapter-07/)
