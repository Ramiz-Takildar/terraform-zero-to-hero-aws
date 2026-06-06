# Chapter 8: Advanced Patterns and Best Practices

## 📚 Learning Objectives

By the end of this chapter, you will:
- Master for_each and count patterns
- Implement dynamic blocks effectively
- Use depends_on strategically
- Work with lifecycle rules
- Implement import and moved blocks
- Use preconditions and postconditions
- Understand Terraform workspaces
- Implement advanced module patterns

**Prerequisites:** Chapters 1-7 completed  
**Estimated Time:** 2 days  
**Labs:** 3 hands-on exercises

---

## 🔄 For_Each vs Count

### When to Use Each

**Use `count` when:**
- Creating identical resources
- Number of resources is known
- Resources are interchangeable
- Simple iteration needed

**Use `for_each` when:**
- Resources have unique configurations
- Need to reference by key
- Want to avoid index-based issues
- Resources are not interchangeable

### Count Pattern

```hcl
# Simple count
resource "aws_instance" "web" {
  count = 3
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  tags = {
    Name = "web-${count.index + 1}"
  }
}

# Conditional count
resource "aws_eip" "web" {
  count = var.create_eip ? 1 : 0
  
  instance = aws_instance.web[0].id
}

# Count with list
variable "availability_zones" {
  type    = list(string)
  default = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

resource "aws_subnet" "public" {
  count = length(var.availability_zones)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 1}.0/24"
  availability_zone = var.availability_zones[count.index]
}
```

### For_Each Pattern

```hcl
# For_each with map
variable "instances" {
  type = map(object({
    instance_type = string
    ami           = string
  }))
  
  default = {
    web = {
      instance_type = "t2.micro"
      ami           = "ami-12345678"
    }
    app = {
      instance_type = "t2.small"
      ami           = "ami-87654321"
    }
  }
}

resource "aws_instance" "servers" {
  for_each = var.instances
  
  ami           = each.value.ami
  instance_type = each.value.instance_type
  
  tags = {
    Name = each.key
  }
}

# For_each with set
variable "bucket_names" {
  type    = set(string)
  default = ["logs", "data", "backups"]
}

resource "aws_s3_bucket" "buckets" {
  for_each = var.bucket_names
  
  bucket = "${var.project_name}-${each.value}"
  
  tags = {
    Name = each.value
  }
}

# For_each with list (convert to set)
variable "subnet_cidrs" {
  type = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "private" {
  for_each = toset(var.subnet_cidrs)
  
  vpc_id     = aws_vpc.main.id
  cidr_block = each.value
  
  tags = {
    Name = "private-${index(var.subnet_cidrs, each.value) + 1}"
  }
}
```

### Advanced For_Each Patterns

```hcl
# Complex object iteration
locals {
  subnets = {
    for idx, cidr in var.subnet_cidrs : 
    "subnet-${idx}" => {
      cidr_block        = cidr
      availability_zone = var.availability_zones[idx % length(var.availability_zones)]
      public            = idx < 3
    }
  }
}

resource "aws_subnet" "main" {
  for_each = local.subnets
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = each.value.cidr_block
  availability_zone       = each.value.availability_zone
  map_public_ip_on_launch = each.value.public
  
  tags = {
    Name   = each.key
    Public = each.value.public
  }
}

# Nested for_each
locals {
  security_rules = {
    web = {
      ingress = {
        http  = { port = 80, protocol = "tcp" }
        https = { port = 443, protocol = "tcp" }
      }
    }
    app = {
      ingress = {
        app = { port = 8080, protocol = "tcp" }
      }
    }
  }
  
  # Flatten for resource creation
  all_rules = flatten([
    for sg_name, sg_config in local.security_rules : [
      for rule_name, rule in sg_config.ingress : {
        sg_name   = sg_name
        rule_name = rule_name
        port      = rule.port
        protocol  = rule.protocol
        key       = "${sg_name}-${rule_name}"
      }
    ]
  ])
}

resource "aws_security_group_rule" "ingress" {
  for_each = {
    for rule in local.all_rules :
    rule.key => rule
  }
  
  type              = "ingress"
  from_port         = each.value.port
  to_port           = each.value.port
  protocol          = each.value.protocol
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.main[each.value.sg_name].id
}
```

---

## 🎯 Dynamic Blocks

### Basic Dynamic Block

```hcl
variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  
  default = [
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

resource "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.ingress_rules
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
}
```

### Nested Dynamic Blocks

```hcl
variable "load_balancer_config" {
  type = object({
    listeners = list(object({
      port     = number
      protocol = string
      default_actions = list(object({
        type             = string
        target_group_arn = string
      }))
    }))
  })
}

resource "aws_lb" "main" {
  name               = "main-lb"
  load_balancer_type = "application"
  subnets            = aws_subnet.public[*].id
  
  dynamic "listener" {
    for_each = var.load_balancer_config.listeners
    content {
      port     = listener.value.port
      protocol = listener.value.protocol
      
      dynamic "default_action" {
        for_each = listener.value.default_actions
        content {
          type             = default_action.value.type
          target_group_arn = default_action.value.target_group_arn
        }
      }
    }
  }
}
```

### Conditional Dynamic Blocks

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  # Only create EBS block device if specified
  dynamic "ebs_block_device" {
    for_each = var.ebs_volumes
    content {
      device_name = ebs_block_device.value.device_name
      volume_size = ebs_block_device.value.volume_size
      volume_type = ebs_block_device.value.volume_type
      encrypted   = ebs_block_device.value.encrypted
    }
  }
  
  # Only create network interface if specified
  dynamic "network_interface" {
    for_each = var.network_interfaces
    content {
      network_interface_id = network_interface.value.id
      device_index         = network_interface.value.device_index
    }
  }
}
```

---

## 🔗 Depends_On

### Explicit Dependencies

```hcl
# Implicit dependency (automatic)
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public.id  # Implicit dependency
}

# Explicit dependency (manual)
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  depends_on = [
    aws_internet_gateway.main,
    aws_route_table.public
  ]
}
```

### Module Dependencies

```hcl
module "vpc" {
  source = "./modules/vpc"
  
  cidr_block = "10.0.0.0/16"
}

module "database" {
  source = "./modules/rds"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  
  depends_on = [module.vpc]
}

module "application" {
  source = "./modules/app"
  
  vpc_id      = module.vpc.vpc_id
  db_endpoint = module.database.endpoint
  
  depends_on = [
    module.vpc,
    module.database
  ]
}
```

### Complex Dependencies

```hcl
# Wait for multiple resources
resource "null_resource" "wait_for_cluster" {
  depends_on = [
    aws_instance.master,
    aws_instance.workers,
    aws_security_group.cluster
  ]
  
  provisioner "local-exec" {
    command = "./configure-cluster.sh"
  }
}

# Ensure order of operations
resource "aws_iam_role_policy_attachment" "attach" {
  role       = aws_iam_role.instance.name
  policy_arn = aws_iam_policy.custom.arn
  
  depends_on = [
    aws_iam_role.instance,
    aws_iam_policy.custom
  ]
}
```

---

## ♻️ Lifecycle Rules

### Prevent Destroy

```hcl
resource "aws_db_instance" "production" {
  identifier = "prod-db"
  engine     = "postgres"
  
  lifecycle {
    prevent_destroy = true
  }
}
```

### Create Before Destroy

```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "app" {
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  min_size = 1
  max_size = 5
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### Ignore Changes

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  tags = {
    Name = "web-server"
  }
  
  lifecycle {
    ignore_changes = [
      tags,           # Ignore all tag changes
      ami,            # Ignore AMI updates
      user_data       # Ignore user_data changes
    ]
  }
}

# Ignore all changes
resource "aws_s3_bucket" "imported" {
  bucket = "existing-bucket"
  
  lifecycle {
    ignore_changes = all
  }
}
```

### Replace Triggered By

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  lifecycle {
    replace_triggered_by = [
      aws_security_group.web.id
    ]
  }
}
```

### Combined Lifecycle Rules

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = var.environment == "prod"
    
    ignore_changes = [
      tags["LastModified"],
      user_data
    ]
  }
}
```

---

## 📥 Import and Moved Blocks

### Import Block (Terraform 1.5+)

```hcl
# Import existing resource
import {
  to = aws_instance.existing
  id = "i-1234567890abcdef0"
}

resource "aws_instance" "existing" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  # Configuration must match existing resource
}

# Import with for_each
import {
  for_each = toset(["i-111", "i-222", "i-333"])
  to       = aws_instance.imported[each.key]
  id       = each.value
}

resource "aws_instance" "imported" {
  for_each = toset(["i-111", "i-222", "i-333"])
  
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

### Moved Block

```hcl
# Rename resource
moved {
  from = aws_instance.web
  to   = aws_instance.application
}

# Move to module
moved {
  from = aws_instance.database
  to   = module.database.aws_instance.main
}

# Move from module
moved {
  from = module.old_vpc.aws_vpc.main
  to   = aws_vpc.main
}

# Refactor with count to for_each
moved {
  from = aws_subnet.public[0]
  to   = aws_subnet.public["subnet-1"]
}

moved {
  from = aws_subnet.public[1]
  to   = aws_subnet.public["subnet-2"]
}
```

---

## ✅ Preconditions and Postconditions

### Preconditions

```hcl
variable "instance_type" {
  type = string
  
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium"], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, or t2.medium."
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instance_type
  
  lifecycle {
    precondition {
      condition     = data.aws_ami.amazon_linux_2.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }
    
    precondition {
      condition     = var.environment == "prod" ? var.instance_type != "t2.micro" : true
      error_message = "Production instances cannot use t2.micro."
    }
  }
}
```

### Postconditions

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  lifecycle {
    postcondition {
      condition     = self.public_ip != ""
      error_message = "Instance must have a public IP address."
    }
    
    postcondition {
      condition     = self.instance_state == "running"
      error_message = "Instance must be in running state."
    }
  }
}

data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
  
  lifecycle {
    postcondition {
      condition     = self.architecture == "x86_64"
      error_message = "AMI must be x86_64 architecture."
    }
  }
}
```

---

## 🏢 Terraform Workspaces

### Basic Workspace Usage

```bash
# List workspaces
terraform workspace list

# Create workspace
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# Switch workspace
terraform workspace select dev

# Show current workspace
terraform workspace show

# Delete workspace
terraform workspace delete dev
```

### Workspace-Aware Configuration

```hcl
locals {
  workspace_config = {
    dev = {
      instance_type  = "t2.micro"
      instance_count = 1
    }
    staging = {
      instance_type  = "t2.small"
      instance_count = 2
    }
    prod = {
      instance_type  = "t2.large"
      instance_count = 5
    }
  }
  
  config = local.workspace_config[terraform.workspace]
}

resource "aws_instance" "app" {
  count = local.config.instance_count
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = local.config.instance_type
  
  tags = {
    Name        = "${terraform.workspace}-app-${count.index + 1}"
    Environment = terraform.workspace
  }
}

# Workspace-specific S3 backend
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "terraform.tfstate"
    region = "us-east-1"
    
    # Workspace creates separate state files
    workspace_key_prefix = "workspaces"
  }
}
```

---

## 🎨 Advanced Module Patterns

### Module with Conditional Resources

```hcl
# modules/vpc/main.tf
variable "create_nat_gateway" {
  type    = bool
  default = false
}

resource "aws_eip" "nat" {
  count = var.create_nat_gateway ? 1 : 0
  
  domain = "vpc"
}

resource "aws_nat_gateway" "main" {
  count = var.create_nat_gateway ? 1 : 0
  
  allocation_id = aws_eip.nat[0].id
  subnet_id     = aws_subnet.public[0].id
}
```

### Module with For_Each

```hcl
# Root module
module "vpc" {
  for_each = {
    dev     = { cidr = "10.0.0.0/16" }
    staging = { cidr = "10.1.0.0/16" }
    prod    = { cidr = "10.2.0.0/16" }
  }
  
  source = "./modules/vpc"
  
  name       = each.key
  cidr_block = each.value.cidr
}
```

### Module Composition

```hcl
# High-level module
module "application_stack" {
  source = "./modules/app-stack"
  
  environment = var.environment
  
  # Pass through to nested modules
  vpc_cidr        = var.vpc_cidr
  instance_type   = var.instance_type
  database_config = var.database_config
}

# modules/app-stack/main.tf
module "vpc" {
  source = "../vpc"
  
  cidr_block = var.vpc_cidr
}

module "database" {
  source = "../rds"
  
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnet_ids
  config     = var.database_config
}

module "application" {
  source = "../app"
  
  vpc_id      = module.vpc.vpc_id
  subnet_ids  = module.vpc.public_subnet_ids
  db_endpoint = module.database.endpoint
}
```

---

## 📖 Key Takeaways

1. **Use for_each** for unique, identifiable resources
2. **Use count** for identical resources
3. **Dynamic blocks** reduce repetition
4. **depends_on** for explicit dependencies
5. **Lifecycle rules** control resource behavior
6. **Import blocks** bring existing resources under management
7. **Moved blocks** refactor without destroying
8. **Preconditions/postconditions** validate assumptions
9. **Workspaces** manage multiple environments
10. **Module patterns** promote reusability

---

## 🔗 Next Steps

1. Review theory above
2. Complete [Lab 8.1](./LABS.md) - For_Each and Dynamic Blocks
3. Complete [Lab 8.2](./LABS.md) - Lifecycle and Dependencies
4. Complete [Lab 8.3](./LABS.md) - Workspaces and Advanced Patterns
5. Update [CHECKLIST.md](../CHECKLIST.md)

**Next Chapter:** [Chapter 9: CI/CD Integration](../chapter-09/)
