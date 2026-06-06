# Chapter 1: Terraform Basics & HCL - Interview Questions

> 20+ Interview Questions with Detailed Answers

---

## Basic Level Questions

### Q1: What is Infrastructure as Code (IaC)?

**Answer:**

Infrastructure as Code (IaC) is the practice of managing and provisioning infrastructure through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools.

**Key Benefits:**
- **Version Control:** Track changes over time in Git
- **Repeatability:** Same code produces same infrastructure
- **Automation:** Reduce manual errors
- **Documentation:** Code serves as documentation
- **Collaboration:** Team can review and approve changes

**Example:**
```hcl
# This code creates the same VPC every time
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "main-vpc"
  }
}
```

---

### Q2: What is Terraform and how does it differ from other IaC tools?

**Answer:**

**Terraform** is an open-source IaC tool by HashiCorp that uses declarative configuration files to manage infrastructure.

**Key Differences:**

| Feature | Terraform | CloudFormation | Ansible |
|---------|-----------|----------------|---------|
| **Approach** | Declarative | Declarative | Procedural |
| **Cloud Support** | Multi-cloud | AWS only | Multi-cloud |
| **State Management** | Yes (tfstate) | AWS manages | No state |
| **Language** | HCL | JSON/YAML | YAML |
| **Agent** | Agentless | Agentless | Agentless |

**Terraform Advantages:**
- Multi-cloud support (AWS, Azure, GCP, etc.)
- Large provider ecosystem (1000+ providers)
- Immutable infrastructure approach
- Plan before apply (preview changes)

---

### Q3: Explain the Terraform workflow (init, plan, apply, destroy)

**Answer:**

**Terraform Workflow:**

```
1. terraform init
   └─► Downloads providers
   └─► Initializes backend
   └─► Downloads modules
   └─► Creates .terraform/ directory

2. terraform plan
   └─► Reads configuration files
   └─► Queries current state
   └─► Calculates differences
   └─► Shows execution plan (what will change)

3. terraform apply
   └─► Executes the plan
   └─► Creates/modifies/deletes resources
   └─► Updates state file
   └─► Shows outputs

4. terraform destroy
   └─► Plans destruction of all resources
   └─► Prompts for confirmation
   └─► Deletes all managed resources
   └─► Cleans state file
```

**Best Practice:** Always run `plan` before `apply` to review changes.

---

### Q4: What is HCL (HashiCorp Configuration Language)?

**Answer:**

HCL is a declarative language designed by HashiCorp for writing Terraform configurations.

**Key Features:**
- Human-readable and writable
- JSON-compatible
- Supports comments
- Built-in functions
- Interpolation syntax

**Basic Syntax:**
```hcl
# Block structure
<BLOCK_TYPE> "<BLOCK_LABEL>" {
  <IDENTIFIER> = <EXPRESSION>
}

# Example
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "WebServer"
  }
}
```

---

### Q5: What is a Terraform provider?

**Answer:**

A **provider** is a plugin that enables Terraform to interact with cloud platforms, SaaS providers, and other APIs.

**Common Providers:**
- `aws` - Amazon Web Services
- `azurerm` - Microsoft Azure
- `google` - Google Cloud Platform
- `kubernetes` - Kubernetes
- `github` - GitHub

**Configuration:**
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
}
```

**Key Points:**
- Providers are downloaded during `terraform init`
- Each provider has its own resources and data sources
- Version constraints ensure compatibility

---

## Intermediate Level Questions

### Q6: What is Terraform state and why is it important?

**Answer:**

**Terraform State** is a JSON file (`terraform.tfstate`) that maps your configuration to real-world resources.

**Purpose:**
1. **Resource Mapping:** Links config to actual resources
2. **Metadata Storage:** Stores resource dependencies
3. **Performance:** Caches resource attributes
4. **Collaboration:** Enables team workflows

**State File Structure:**
```json
{
  "version": 4,
  "terraform_version": "1.5.0",
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id": "i-1234567890abcdef0",
            "public_ip": "54.123.45.67"
          }
        }
      ]
    }
  ]
}
```

**Critical Rules:**
- ❌ Never edit state file manually
- ✅ Use remote state for teams
- ✅ Enable state locking
- ✅ Backup state regularly

---

### Q7: What are Terraform variables and how do you use them?

**Answer:**

**Variables** make Terraform configurations flexible and reusable.

**Variable Types:**

| Type | Example | Usage |
|------|---------|-------|
| `string` | `"t2.micro"` | Single text value |
| `number` | `42` | Numeric value |
| `bool` | `true` | Boolean |
| `list(string)` | `["a", "b"]` | Ordered list |
| `map(string)` | `{key = "val"}` | Key-value pairs |
| `object({...})` | Complex | Nested structure |

**Definition:**
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
  
  validation {
    condition     = contains(["t2.micro", "t2.small"], var.instance_type)
    error_message = "Must be t2.micro or t2.small."
  }
}
```

**Usage:**
```hcl
resource "aws_instance" "web" {
  instance_type = var.instance_type
}
```

**Setting Values:**
1. Default value in variable block
2. `terraform.tfvars` file
3. Command line: `-var="key=value"`
4. Environment variable: `TF_VAR_key=value`

---

### Q8: What are Terraform outputs?

**Answer:**

**Outputs** export values from your Terraform configuration.

**Use Cases:**
- Display important information after apply
- Pass values to other Terraform configurations
- Share data with external systems

**Definition:**
```hcl
output "instance_public_ip" {
  description = "Public IP of the EC2 instance"
  value       = aws_instance.web.public_ip
  sensitive   = false  # Set true to hide from logs
}
```

**Accessing Outputs:**
```bash
# View all outputs
terraform output

# View specific output
terraform output instance_public_ip

# JSON format
terraform output -json
```

---

### Q9: What are data sources in Terraform?

**Answer:**

**Data sources** allow Terraform to query information from existing resources or external systems.

**Difference from Resources:**

| Resource | Data Source |
|----------|-------------|
| Creates infrastructure | Queries existing infrastructure |
| `resource` block | `data` block |
| Managed by Terraform | Read-only |

**Example:**
```hcl
# Query latest Amazon Linux 2 AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Use the data source
resource "aws_instance" "web" {
  ami = data.aws_ami.amazon_linux_2.id
}
```

**Benefits:**
- No hardcoded values
- Always uses latest data
- Reduces maintenance

---

### Q10: What is the difference between `terraform plan` and `terraform apply`?

**Answer:**

| terraform plan | terraform apply |
|----------------|-----------------|
| **Read-only** | **Makes changes** |
| Shows what will happen | Actually executes changes |
| No state modifications | Updates state file |
| No AWS API calls to create | Creates real resources |
| Safe to run anytime | Requires confirmation |
| No cost | May incur AWS costs |

**Workflow:**
```bash
# 1. Preview changes
terraform plan

# 2. Save plan to file
terraform plan -out=tfplan

# 3. Apply saved plan (no confirmation needed)
terraform apply tfplan

# Or apply directly (requires confirmation)
terraform apply
```

**Best Practice:** Always review plan output before applying.

---

## Advanced Level Questions

### Q11: What are Terraform workspaces?

**Answer:**

**Workspaces** allow you to manage multiple environments (dev, staging, prod) with the same configuration.

**Commands:**
```bash
# List workspaces
terraform workspace list

# Create workspace
terraform workspace new dev

# Switch workspace
terraform workspace select prod

# Show current workspace
terraform workspace show
```

**Usage:**
```hcl
resource "aws_instance" "web" {
  instance_type = terraform.workspace == "prod" ? "t2.large" : "t2.micro"
  
  tags = {
    Environment = terraform.workspace
  }
}
```

**State Isolation:**
- Each workspace has its own state file
- `terraform.tfstate.d/dev/terraform.tfstate`
- `terraform.tfstate.d/prod/terraform.tfstate`

---

### Q12: What are local values (locals) in Terraform?

**Answer:**

**Locals** are named values that can be reused within a module.

**Use Cases:**
- Avoid repeating complex expressions
- Compute derived values
- Improve code readability

**Example:**
```hcl
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
  
  instance_name = "${var.project_name}-${var.environment}-web"
  
  # Conditional logic
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
}

resource "aws_instance" "web" {
  instance_type = local.instance_type
  
  tags = merge(
    local.common_tags,
    {
      Name = local.instance_name
    }
  )
}
```

**Difference from Variables:**
- Variables: Input from outside
- Locals: Computed within module

---

### Q13: What are Terraform modules?

**Answer:**

**Modules** are containers for multiple resources that are used together.

**Benefits:**
- Code reusability
- Encapsulation
- Versioning
- Easier testing

**Structure:**
```
modules/
└── vpc/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    └── README.md
```

**Usage:**
```hcl
module "vpc" {
  source = "./modules/vpc"
  
  cidr_block = "10.0.0.0/16"
  environment = "prod"
}

# Access module outputs
resource "aws_instance" "web" {
  subnet_id = module.vpc.public_subnet_id
}
```

**Module Sources:**
- Local path: `./modules/vpc`
- Terraform Registry: `terraform-aws-modules/vpc/aws`
- Git: `git::https://github.com/user/repo.git`
- S3: `s3::https://s3.amazonaws.com/bucket/key`

---

### Q14: How do you handle sensitive data in Terraform?

**Answer:**

**Methods:**

1. **Mark outputs as sensitive:**
```hcl
output "db_password" {
  value     = aws_db_instance.main.password
  sensitive = true  # Won't show in logs
}
```

2. **Use AWS Secrets Manager:**
```hcl
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "prod/db/password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

3. **Environment variables:**
```bash
export TF_VAR_db_password="secret123"
```

4. **Never commit:**
- Add `*.tfvars` to `.gitignore`
- Use encrypted backends
- Use Terraform Cloud for encryption

---

### Q15: What is the Terraform lock file (.terraform.lock.hcl)?

**Answer:**

**Purpose:** Ensures consistent provider versions across team members.

**What it contains:**
- Provider versions used
- Provider checksums
- Provider sources

**Example:**
```hcl
provider "registry.terraform.io/hashicorp/aws" {
  version     = "5.31.0"
  constraints = "~> 5.0"
  hashes = [
    "h1:abc123...",
    "zh:def456...",
  ]
}
```

**Best Practices:**
- ✅ Commit to version control
- ✅ Update with `terraform init -upgrade`
- ❌ Don't edit manually

---

## Scenario-Based Questions

### S1: How would you migrate existing AWS infrastructure to Terraform?

**Answer:**

**Approach:**

1. **Inventory existing resources**
```bash
aws ec2 describe-instances
aws s3 ls
```

2. **Write Terraform configuration**
```hcl
resource "aws_instance" "existing" {
  # Match existing configuration
}
```

3. **Import resources**
```bash
terraform import aws_instance.existing i-1234567890abcdef0
```

4. **Verify state**
```bash
terraform plan  # Should show no changes
```

5. **Gradually refactor**

**Tools:**
- `terraformer` - Auto-generate Terraform from existing infrastructure
- `terraform import` - Import individual resources

---

### S2: Terraform apply failed midway. What do you do?

**Answer:**

**Steps:**

1. **Check error message**
```bash
terraform apply
# Read error carefully
```

2. **Check state**
```bash
terraform show
# See what was created
```

3. **Fix the issue**
- Correct configuration
- Fix AWS permissions
- Resolve resource conflicts

4. **Re-apply**
```bash
terraform apply
# Terraform will continue from where it stopped
```

**Key Point:** Terraform state tracks what was created, so re-running apply is safe.

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `terraform init` | Initialize working directory |
| `terraform validate` | Check syntax |
| `terraform fmt` | Format code |
| `terraform plan` | Preview changes |
| `terraform apply` | Create infrastructure |
| `terraform destroy` | Delete infrastructure |
| `terraform show` | Display state |
| `terraform output` | Show outputs |

---

## Key Takeaways

1. **IaC = Version Control:** Infrastructure is code
2. **Declarative:** Describe desired state, not steps
3. **State is Critical:** Maps config to reality
4. **Plan Before Apply:** Always preview changes
5. **Variables = Flexibility:** Reusable configurations
6. **Modules = Reusability:** DRY principle
7. **Providers = Extensibility:** Support any API

---

**Previous:** [Chapter 1 README](./README.md)  
**Next:** [Chapter 2 Interview Questions](../chapter-02/INTERVIEW.md)
