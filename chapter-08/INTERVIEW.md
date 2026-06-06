# Chapter 8: Interview Questions - Advanced Patterns and Best Practices

## 📋 Overview

This document contains interview questions covering advanced Terraform patterns including for_each, dynamic blocks, lifecycle rules, and workspaces.

**Topics Covered:**
- For_Each vs Count
- Dynamic Blocks
- Lifecycle Rules
- Dependencies
- Import and Moved Blocks
- Preconditions and Postconditions
- Terraform Workspaces
- Advanced Module Patterns

---

## 🔄 For_Each vs Count

### Q1: What is the difference between count and for_each? When should you use each?

**Answer:**

**Count:**
- Creates multiple identical resources
- References by index (0, 1, 2...)
- Index-based addressing
- Simple iteration

**For_Each:**
- Creates resources with unique identifiers
- References by key (name, id...)
- Key-based addressing
- Complex configurations

**Comparison Table:**

| Aspect | Count | For_Each |
|--------|-------|----------|
| **Addressing** | Index-based | Key-based |
| **Stability** | Changes when list reordered | Stable with keys |
| **Input Type** | Number or list | Map or set |
| **Use Case** | Identical resources | Unique resources |
| **Refactoring** | Difficult | Easier |

**Count Example:**
```hcl
# Simple, identical resources
resource "aws_instance" "web" {
  count = 3
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  tags = {
    Name = "web-${count.index + 1}"
  }
}

# Reference: aws_instance.web[0], aws_instance.web[1], aws_instance.web[2]
```

**For_Each Example:**
```hcl
# Unique, identifiable resources
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

# Reference: aws_instance.servers["web"], aws_instance.servers["app"]
```

**When to Use Count:**
1. Creating N identical resources
2. Simple conditional creation (count = condition ? 1 : 0)
3. Number of resources is straightforward

**When to Use For_Each:**
1. Resources have unique configurations
2. Need stable resource addressing
3. Resources are not interchangeable
4. Avoiding index-based issues

**Problem with Count:**
```hcl
# Initial: ["web-1", "web-2", "web-3"]
variable "servers" {
  default = ["web-1", "web-2", "web-3"]
}

resource "aws_instance" "web" {
  count = length(var.servers)
  
  tags = {
    Name = var.servers[count.index]
  }
}

# Remove "web-2": ["web-1", "web-3"]
# Result: web-2 (index 1) becomes web-3, web-3 (index 2) is destroyed!
# This is NOT what we want!
```

**Solution with For_Each:**
```hcl
variable "servers" {
  default = ["web-1", "web-2", "web-3"]
}

resource "aws_instance" "web" {
  for_each = toset(var.servers)
  
  tags = {
    Name = each.value
  }
}

# Remove "web-2"
# Result: Only web-2 is destroyed, web-1 and web-3 unchanged!
```

---

### Q2: How do you convert between count and for_each?

**Answer:**

**From Count to For_Each:**

```hcl
# Before (count)
variable "subnet_cidrs" {
  type = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "main" {
  count = length(var.subnet_cidrs)
  
  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_cidrs[count.index]
  
  tags = {
    Name = "subnet-${count.index + 1}"
  }
}

# After (for_each)
resource "aws_subnet" "main" {
  for_each = toset(var.subnet_cidrs)
  
  vpc_id     = aws_vpc.main.id
  cidr_block = each.value
  
  tags = {
    Name = "subnet-${index(var.subnet_cidrs, each.value) + 1}"
  }
}

# Use moved blocks to prevent recreation
moved {
  from = aws_subnet.main[0]
  to   = aws_subnet.main["10.0.1.0/24"]
}

moved {
  from = aws_subnet.main[1]
  to   = aws_subnet.main["10.0.2.0/24"]
}

moved {
  from = aws_subnet.main[2]
  to   = aws_subnet.main["10.0.3.0/24"]
}
```

**From For_Each to Count:**
```hcl
# Before (for_each)
variable "instances" {
  type = map(string)
  default = {
    web = "t2.micro"
    app = "t2.small"
  }
}

resource "aws_instance" "main" {
  for_each = var.instances
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = each.value
  
  tags = {
    Name = each.key
  }
}

# After (count) - requires restructuring
variable "instances" {
  type = list(object({
    name = string
    type = string
  }))
  default = [
    { name = "web", type = "t2.micro" },
    { name = "app", type = "t2.small" }
  ]
}

resource "aws_instance" "main" {
  count = length(var.instances)
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = var.instances[count.index].type
  
  tags = {
    Name = var.instances[count.index].name
  }
}

# Use moved blocks
moved {
  from = aws_instance.main["web"]
  to   = aws_instance.main[0]
}

moved {
  from = aws_instance.main["app"]
  to   = aws_instance.main[1]
}
```

---

## 🎯 Dynamic Blocks

### Q3: What are dynamic blocks and when should you use them?

**Answer:**

Dynamic blocks generate nested configuration blocks dynamically based on a collection.

**Use Cases:**
1. Variable number of nested blocks
2. Conditional nested blocks
3. Reduce code repetition
4. Generate configurations from data

**Basic Example:**
```hcl
variable "ingress_rules" {
  type = list(object({
    from_port   = number
    to_port     = number
    protocol    = string
    cidr_blocks = list(string)
  }))
  
  default = [
    {
      from_port   = 80
      to_port     = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    },
    {
      from_port   = 443
      to_port     = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
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
    }
  }
}
```

**Nested Dynamic Blocks:**
```hcl
variable "listeners" {
  type = list(object({
    port     = number
    protocol = string
    rules = list(object({
      priority = number
      action   = string
    }))
  }))
}

resource "aws_lb" "main" {
  name = "main-lb"
  
  dynamic "listener" {
    for_each = var.listeners
    content {
      port     = listener.value.port
      protocol = listener.value.protocol
      
      dynamic "default_action" {
        for_each = listener.value.rules
        content {
          type     = default_action.value.action
          priority = default_action.value.priority
        }
      }
    }
  }
}
```

**Conditional Dynamic Blocks:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  # Only create if volumes are specified
  dynamic "ebs_block_device" {
    for_each = var.ebs_volumes != null ? var.ebs_volumes : []
    content {
      device_name = ebs_block_device.value.device_name
      volume_size = ebs_block_device.value.volume_size
    }
  }
}
```

**Best Practices:**
1. Use for complex nested structures
2. Don't overuse - can reduce readability
3. Keep dynamic blocks simple
4. Document the expected structure
5. Validate input data

---

### Q4: How do you handle complex nested structures with dynamic blocks?

**Answer:**

**Flatten Nested Structures:**
```hcl
locals {
  # Input: nested structure
  security_groups = {
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
    for sg_name, sg_config in local.security_groups : [
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
  security_group_id = aws_security_group.main[each.value.sg_name].id
}
```

---

## ♻️ Lifecycle Rules

### Q5: What are Terraform lifecycle rules and what are the different types?

**Answer:**

Lifecycle rules control how Terraform manages resource lifecycles.

**Types:**

**1. create_before_destroy:**
```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  lifecycle {
    create_before_destroy = true
  }
}
```

**Use Cases:**
- Resources that can't have downtime
- Resources referenced by other resources
- Launch templates, security groups
- Blue-green deployments

**2. prevent_destroy:**
```hcl
resource "aws_db_instance" "production" {
  identifier = "prod-db"
  engine     = "postgres"
  
  lifecycle {
    prevent_destroy = true
  }
}
```

**Use Cases:**
- Production databases
- Critical infrastructure
- Stateful resources
- Prevent accidental deletion

**3. ignore_changes:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  tags = {
    Name         = "web-server"
    LastModified = timestamp()
  }
  
  lifecycle {
    ignore_changes = [
      tags["LastModified"],
      user_data,
      ami
    ]
  }
}

# Ignore all changes
lifecycle {
  ignore_changes = all
}
```

**Use Cases:**
- Attributes managed externally
- Auto-scaling managed attributes
- Tags updated by other systems
- Imported resources

**4. replace_triggered_by:**
```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  lifecycle {
    replace_triggered_by = [
      aws_security_group.web.id,
      aws_key_pair.deployer.id
    ]
  }
}
```

**Use Cases:**
- Force replacement when dependencies change
- Ensure consistency
- Trigger updates

**Combined Rules:**
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

### Q6: When should you use create_before_destroy?

**Answer:**

**Use create_before_destroy when:**

1. **Zero-downtime deployments:**
```hcl
resource "aws_launch_template" "app" {
  name_prefix   = "app-"
  image_id      = var.ami_id
  instance_type = var.instance_type
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "app" {
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  lifecycle {
    create_before_destroy = true
  }
}
```

2. **Resources with dependencies:**
```hcl
resource "aws_security_group" "web" {
  name   = "web-sg-${timestamp()}"
  vpc_id = aws_vpc.main.id
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux_2.id
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.web.id]
}
```

3. **Name conflicts:**
```hcl
# Without create_before_destroy:
# 1. Destroy old resource (name freed)
# 2. Create new resource (can use same name)

# With create_before_destroy:
# 1. Create new resource (needs unique name)
# 2. Destroy old resource

resource "aws_iam_role" "app" {
  name = "app-role-${timestamp()}"  # Unique name required
  
  lifecycle {
    create_before_destroy = true
  }
}
```

**Considerations:**
- May temporarily double resources (cost)
- Requires unique naming strategy
- May hit resource limits
- Dependencies must support it

---

## 🔗 Dependencies

### Q7: What is the difference between implicit and explicit dependencies?

**Answer:**

**Implicit Dependencies (Automatic):**

Terraform automatically detects dependencies through resource references.

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id  # Implicit dependency
  cidr_block = "10.0.1.0/24"
}

resource "aws_instance" "web" {
  ami       = data.aws_ami.amazon_linux_2.id
  subnet_id = aws_subnet.public.id  # Implicit dependency
}

# Creation order: VPC → Subnet → Instance
# Destruction order: Instance → Subnet → VPC
```

**Explicit Dependencies (Manual):**

Use `depends_on` when Terraform can't detect the dependency automatically.

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  
  # Explicit dependency
  depends_on = [
    aws_internet_gateway.main,
    aws_route_table.public
  ]
}
```

**When to Use depends_on:**

1. **Hidden dependencies:**
```hcl
resource "aws_iam_role_policy_attachment" "attach" {
  role       = aws_iam_role.instance.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
  
  # Ensure role exists first
  depends_on = [aws_iam_role.instance]
}
```

2. **Ordering requirements:**
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

3. **Module dependencies:**
```hcl
module "database" {
  source = "./modules/rds"
  
  vpc_id = module.vpc.vpc_id
  
  depends_on = [module.vpc]
}
```

**Best Practices:**
- Prefer implicit dependencies
- Use depends_on sparingly
- Document why depends_on is needed
- Avoid circular dependencies

---

## 📥 Import and Moved Blocks

### Q8: How do you import existing resources into Terraform?

**Answer:**

**Terraform 1.5+ (Import Block):**

```hcl
# Import single resource
import {
  to = aws_instance.existing
  id = "i-1234567890abcdef0"
}

resource "aws_instance" "existing" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  # Configuration must match existing resource
}
```

**Import with For_Each:**
```hcl
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

**Legacy Method (Pre-1.5):**
```bash
# Import command
terraform import aws_instance.existing i-1234567890abcdef0

# Then add resource configuration
resource "aws_instance" "existing" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}
```

**Import Process:**
1. Write resource configuration
2. Add import block (or run import command)
3. Run `terraform plan` to verify
4. Adjust configuration to match
5. Run `terraform apply`

---

### Q9: What are moved blocks and when should you use them?

**Answer:**

Moved blocks tell Terraform that a resource has been renamed or moved, preventing destruction and recreation.

**Rename Resource:**
```hcl
# Old configuration
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}

# New configuration
resource "aws_instance" "application" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}

# Moved block
moved {
  from = aws_instance.web
  to   = aws_instance.application
}
```

**Move to Module:**
```hcl
moved {
  from = aws_instance.database
  to   = module.database.aws_instance.main
}
```

**Move from Module:**
```hcl
moved {
  from = module.old_vpc.aws_vpc.main
  to   = aws_vpc.main
}
```

**Refactor Count to For_Each:**
```hcl
# Before
resource "aws_subnet" "public" {
  count = 3
  # ...
}

# After
resource "aws_subnet" "public" {
  for_each = toset(["subnet-1", "subnet-2", "subnet-3"])
  # ...
}

# Moved blocks
moved {
  from = aws_subnet.public[0]
  to   = aws_subnet.public["subnet-1"]
}

moved {
  from = aws_subnet.public[1]
  to   = aws_subnet.public["subnet-2"]
}

moved {
  from = aws_subnet.public[2]
  to   = aws_subnet.public["subnet-3"]
}
```

**Use Cases:**
- Refactoring code structure
- Renaming resources
- Moving resources to/from modules
- Converting between count and for_each
- Reorganizing state

---

## ✅ Preconditions and Postconditions

### Q10: What are preconditions and postconditions? How do you use them?

**Answer:**

**Preconditions** validate assumptions before resource creation.
**Postconditions** validate results after resource creation.

**Preconditions:**
```hcl
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

**Postconditions:**
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
```

**Data Source Postconditions:**
```hcl
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
    
    postcondition {
      condition     = self.state == "available"
      error_message = "AMI must be available."
    }
  }
}
```

**Use Cases:**
- Validate input assumptions
- Ensure resource state
- Catch configuration errors early
- Document requirements
- Prevent invalid deployments

---

## 🏢 Terraform Workspaces

### Q11: What are Terraform workspaces and when should you use them?

**Answer:**

Workspaces allow multiple state files for the same configuration, typically for different environments.

**Basic Commands:**
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

**Workspace-Aware Configuration:**
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
```

**Workspace-Specific Resources:**
```hcl
resource "aws_s3_bucket" "data" {
  bucket = "myapp-${terraform.workspace}-data"
  
  tags = {
    Workspace = terraform.workspace
  }
}
```

**When to Use Workspaces:**
1. **Multiple environments** (dev, staging, prod)
2. **Testing configurations**
3. **Temporary deployments**
4. **Feature branches**

**When NOT to Use Workspaces:**
1. **Different AWS accounts** (use separate backends)
2. **Completely different infrastructure**
3. **Different teams** (use separate state files)
4. **Production isolation** (use separate backends)

**Alternatives:**
- Separate directories per environment
- Separate repositories per environment
- Terragrunt for environment management
- Terraform Cloud workspaces

---

### Q12: What are the limitations of Terraform workspaces?

**Answer:**

**Limitations:**

1. **Same Backend:**
```hcl
# All workspaces share same backend
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "terraform.tfstate"
    region = "us-east-1"
    
    # Workspaces create: workspaces/dev/terraform.tfstate
    #                    workspaces/staging/terraform.tfstate
    #                    workspaces/prod/terraform.tfstate
  }
}
```

2. **Same AWS Account:**
- Workspaces don't change provider configuration
- All workspaces deploy to same AWS account
- Need separate backends for different accounts

3. **State File Visibility:**
- All workspace states in same S3 bucket
- Same IAM permissions for all workspaces
- Can't isolate production state

4. **No Workspace-Specific Variables:**
```hcl
# Can't do this:
terraform {
  backend "s3" {
    bucket = var.state_bucket  # Variables not allowed
  }
}

# Must use workspace-aware logic:
locals {
  config = local.workspace_config[terraform.workspace]
}
```

5. **Easy to Forget Workspace:**
```bash
# Dangerous: might deploy to wrong workspace
terraform apply

# Better: always check
terraform workspace show
terraform apply
```

**Best Practices:**
- Use workspaces for non-production environments
- Use separate backends for production
- Always check current workspace before applying
- Tag resources with workspace name
- Document workspace usage
- Consider alternatives for complex setups

---

## 📚 Summary

**Key Topics Covered:**
- For_each vs count patterns
- Dynamic blocks for nested configurations
- Lifecycle rules for resource management
- Dependency management
- Import and moved blocks
- Preconditions and postconditions
- Terraform workspaces
- Advanced patterns and best practices

**Best Practices:**
1. Use for_each for unique, identifiable resources
2. Use dynamic blocks to reduce repetition
3. Apply lifecycle rules strategically
4. Prefer implicit dependencies
5. Use moved blocks for refactoring
6. Validate with preconditions/postconditions
7. Use workspaces carefully
8. Document complex patterns

**Next Steps:**
1. Practice labs in [LABS.md](./LABS.md)
2. Experiment with different patterns
3. Proceed to [Chapter 9](../chapter-09/)

---

**💡 Pro Tips:**
- Test for_each patterns in dev first
- Use moved blocks to avoid recreation
- Combine lifecycle rules carefully
- Always validate workspace before applying
- Document why you use depends_on
- Use preconditions to fail fast
- Keep workspace configurations simple
