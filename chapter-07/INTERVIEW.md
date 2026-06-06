# Chapter 7: Interview Questions - Data Sources and Terraform Functions

## 📋 Overview

This document contains interview questions covering Terraform data sources, built-in functions, and their practical applications.

**Topics Covered:**
- Data Sources
- String Functions
- Numeric Functions
- Collection Functions
- Conditional Functions
- Type Conversion
- File and Encoding Functions
- Network Functions
- Advanced Patterns

---

## 📊 Data Sources

### Q1: What are Terraform data sources and how do they differ from resources?

**Answer:**

Data sources allow Terraform to fetch information from external sources or existing infrastructure.

**Key Differences:**

| Aspect | Data Sources | Resources |
|--------|-------------|-----------|
| **Purpose** | Read existing data | Create/manage infrastructure |
| **Keyword** | `data` | `resource` |
| **State** | Not managed | Managed in state |
| **Lifecycle** | Read-only | Create, update, delete |
| **Changes** | Don't modify infrastructure | Modify infrastructure |

**Example:**

```hcl
# Data source - reads existing AMI
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Resource - creates new instance
resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id  # Uses data source
  instance_type = "t2.micro"
}
```

**Use Cases:**
1. Query existing infrastructure
2. Get dynamic values (latest AMI, AZs)
3. Reference resources created outside Terraform
4. Fetch configuration from external sources
5. Generate policies dynamically

---

### Q2: How do you query the latest AMI using data sources?

**Answer:**

Use the `aws_ami` data source with filters and `most_recent = true`:

**Amazon Linux 2:**
```hcl
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

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}
```

**Ubuntu:**
```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}
```

**Custom AMI by Tag:**
```hcl
data "aws_ami" "custom" {
  most_recent = true
  owners      = ["self"]
  
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }
  
  filter {
    name   = "tag:Application"
    values = ["web-server"]
  }
}
```

**Key Points:**
- `most_recent = true` gets the latest matching AMI
- `owners` restricts to specific AWS accounts
- Multiple filters narrow down results
- Use `id` attribute to reference in resources

---

### Q3: How do you use data sources to query existing VPC and subnets?

**Answer:**

**Query Default VPC:**
```hcl
data "aws_vpc" "default" {
  default = true
}

output "default_vpc_id" {
  value = data.aws_vpc.default.id
}
```

**Query VPC by Tag:**
```hcl
data "aws_vpc" "selected" {
  tags = {
    Name = "production-vpc"
  }
}
```

**Query VPC by Filter:**
```hcl
data "aws_vpc" "main" {
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }
  
  filter {
    name   = "cidr-block"
    values = ["10.0.0.0/16"]
  }
}
```

**Query Subnets in VPC:**
```hcl
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.selected.id]
  }
  
  tags = {
    Type = "private"
  }
}

# Get details of each subnet
data "aws_subnet" "private" {
  count = length(data.aws_subnets.private.ids)
  id    = data.aws_subnets.private.ids[count.index]
}

# Use in resources
resource "aws_instance" "app" {
  count = length(data.aws_subnet.private)
  
  ami       = data.aws_ami.amazon_linux_2.id
  subnet_id = data.aws_subnet.private[count.index].id
}
```

---

### Q4: What is the aws_caller_identity data source and when is it useful?

**Answer:**

The `aws_caller_identity` data source provides information about the AWS account and credentials being used.

**Usage:**
```hcl
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "caller_arn" {
  value = data.aws_caller_identity.current.arn
}

output "user_id" {
  value = data.aws_caller_identity.current.user_id
}
```

**Use Cases:**

**1. Dynamic ARN Construction:**
```hcl
locals {
  bucket_arn = "arn:aws:s3:::my-bucket-${data.aws_caller_identity.current.account_id}"
}
```

**2. Account-Specific Resources:**
```hcl
resource "aws_s3_bucket" "logs" {
  bucket = "logs-${data.aws_caller_identity.current.account_id}"
}
```

**3. IAM Policy with Account ID:**
```hcl
data "aws_iam_policy_document" "bucket_policy" {
  statement {
    principals {
      type        = "AWS"
      identifiers = ["arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"]
    }
    
    actions = ["s3:GetObject"]
    resources = ["${aws_s3_bucket.data.arn}/*"]
  }
}
```

**4. Cross-Account Access:**
```hcl
resource "aws_iam_role" "cross_account" {
  name = "cross-account-role"
  
  assume_role_policy = jsonencode({
    Statement = [{
      Effect = "Allow"
      Principal = {
        AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
      }
      Action = "sts:AssumeRole"
    }]
  })
}
```

---

### Q5: How do you use the aws_iam_policy_document data source?

**Answer:**

The `aws_iam_policy_document` data source generates IAM policy JSON documents.

**Basic Policy:**
```hcl
data "aws_iam_policy_document" "s3_read" {
  statement {
    sid    = "AllowS3Read"
    effect = "Allow"
    
    actions = [
      "s3:GetObject",
      "s3:ListBucket"
    ]
    
    resources = [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ]
  }
}

resource "aws_iam_policy" "s3_read" {
  name   = "s3-read-policy"
  policy = data.aws_iam_policy_document.s3_read.json
}
```

**Assume Role Policy:**
```hcl
data "aws_iam_policy_document" "ec2_assume_role" {
  statement {
    actions = ["sts:AssumeRole"]
    
    principals {
      type        = "Service"
      identifiers = ["ec2.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "instance" {
  name               = "instance-role"
  assume_role_policy = data.aws_iam_policy_document.ec2_assume_role.json
}
```

**Multiple Statements:**
```hcl
data "aws_iam_policy_document" "combined" {
  statement {
    sid    = "AllowS3"
    effect = "Allow"
    actions = ["s3:*"]
    resources = ["*"]
  }
  
  statement {
    sid    = "AllowEC2"
    effect = "Allow"
    actions = ["ec2:Describe*"]
    resources = ["*"]
  }
  
  statement {
    sid    = "DenyDelete"
    effect = "Deny"
    actions = ["s3:DeleteBucket"]
    resources = ["*"]
  }
}
```

**With Conditions:**
```hcl
data "aws_iam_policy_document" "conditional" {
  statement {
    effect = "Allow"
    actions = ["s3:GetObject"]
    resources = ["arn:aws:s3:::my-bucket/*"]
    
    condition {
      test     = "IpAddress"
      variable = "aws:SourceIp"
      values   = ["10.0.0.0/16"]
    }
  }
}
```

**Merge Policies:**
```hcl
data "aws_iam_policy_document" "base" {
  statement {
    actions = ["s3:GetObject"]
    resources = ["*"]
  }
}

data "aws_iam_policy_document" "additional" {
  statement {
    actions = ["s3:PutObject"]
    resources = ["*"]
  }
}

data "aws_iam_policy_document" "combined" {
  source_policy_documents = [
    data.aws_iam_policy_document.base.json,
    data.aws_iam_policy_document.additional.json
  ]
}
```

---

## 🔧 String Functions

### Q6: What are the most commonly used string functions in Terraform?

**Answer:**

**Case Conversion:**
```hcl
locals {
  name = "Hello World"
  
  upper_case = upper(local.name)  # "HELLO WORLD"
  lower_case = lower(local.name)  # "hello world"
  title_case = title(local.name)  # "Hello World"
}
```

**Trimming:**
```hcl
locals {
  text = "  hello world  "
  
  trimmed       = trimspace(local.text)           # "hello world"
  trim_custom   = trim(local.text, " ")           # "hello world"
  trim_prefix   = trimprefix("hello-world", "hello-")  # "world"
  trim_suffix   = trimsuffix("hello-world", "-world")  # "hello"
}
```

**Replace:**
```hcl
locals {
  text = "hello-world-test"
  
  replaced = replace(local.text, "-", "_")  # "hello_world_test"
  
  # Regex replace
  cleaned = replace("hello123world456", "/[0-9]/", "")  # "helloworld"
}
```

**Split and Join:**
```hcl
locals {
  text = "hello-world-test"
  
  parts  = split("-", local.text)      # ["hello", "world", "test"]
  joined = join("_", local.parts)      # "hello_world_test"
}
```

**Substring:**
```hcl
locals {
  text = "hello world"
  
  first_five = substr(local.text, 0, 5)   # "hello"
  last_five  = substr(local.text, 6, -1)  # "world"
}
```

**Format:**
```hcl
locals {
  name = "World"
  
  formatted = format("Hello, %s!", local.name)  # "Hello, World!"
  
  # Format list
  urls = formatlist("https://%s.example.com", ["www", "api", "cdn"])
  # ["https://www.example.com", "https://api.example.com", "https://cdn.example.com"]
}
```

---

### Q7: How do you use regex functions in Terraform?

**Answer:**

**regex - Extract Matches:**
```hcl
locals {
  # Extract version number
  version_string = "app-v1.2.3"
  version = regex("v([0-9.]+)", local.version_string)[0]  # "1.2.3"
  
  # Extract multiple groups
  email = "user@example.com"
  parts = regex("([^@]+)@(.+)", local.email)
  # ["user", "example.com"]
  username = local.parts[0]  # "user"
  domain   = local.parts[1]  # "example.com"
}
```

**regexall - Find All Matches:**
```hcl
locals {
  text = "abc123def456ghi789"
  
  # Find all numbers
  numbers = regexall("[0-9]+", local.text)  # ["123", "456", "789"]
  
  # Find all words
  words = regexall("[a-z]+", local.text)  # ["abc", "def", "ghi"]
}
```

**Validation with regex:**
```hcl
variable "email" {
  type = string
  
  validation {
    condition     = can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", var.email))
    error_message = "Must be a valid email address."
  }
}

variable "version" {
  type = string
  
  validation {
    condition     = can(regex("^v[0-9]+\\.[0-9]+\\.[0-9]+$", var.version))
    error_message = "Version must be in format v1.2.3"
  }
}
```

**Replace with regex:**
```hcl
locals {
  text = "hello123world456"
  
  # Remove all numbers
  no_numbers = replace(local.text, "/[0-9]/", "")  # "helloworld"
  
  # Replace multiple spaces with single space
  normalized = replace("hello    world", "/\\s+/", " ")  # "hello world"
}
```

---

## 🔢 Collection Functions

### Q8: How do you work with lists in Terraform?

**Answer:**

**Basic List Operations:**
```hcl
locals {
  numbers = [1, 2, 3, 4, 5]
  
  # Length
  count = length(local.numbers)  # 5
  
  # Element access
  first = element(local.numbers, 0)  # 1
  last  = element(local.numbers, length(local.numbers) - 1)  # 5
  
  # Index
  index_of_3 = index(local.numbers, 3)  # 2
}
```

**List Manipulation:**
```hcl
locals {
  list1 = [1, 2, 3]
  list2 = [4, 5, 6]
  
  # Concat
  combined = concat(local.list1, local.list2)  # [1, 2, 3, 4, 5, 6]
  
  # Distinct
  with_dupes = [1, 2, 2, 3, 3, 3]
  unique = distinct(local.with_dupes)  # [1, 2, 3]
  
  # Flatten
  nested = [[1, 2], [3, 4], [5]]
  flat = flatten(local.nested)  # [1, 2, 3, 4, 5]
  
  # Reverse
  reversed = reverse([1, 2, 3, 4, 5])  # [5, 4, 3, 2, 1]
  
  # Sort
  unsorted = ["c", "a", "b"]
  sorted = sort(local.unsorted)  # ["a", "b", "c"]
  
  # Slice
  sliced = slice([1, 2, 3, 4, 5], 1, 4)  # [2, 3, 4]
  
  # Chunklist
  chunks = chunklist([1, 2, 3, 4, 5, 6], 2)  # [[1, 2], [3, 4], [5, 6]]
}
```

**Contains:**
```hcl
locals {
  fruits = ["apple", "banana", "orange"]
  
  has_apple = contains(local.fruits, "apple")  # true
  has_grape = contains(local.fruits, "grape")  # false
}
```

---

### Q9: How do you work with maps in Terraform?

**Answer:**

**Basic Map Operations:**
```hcl
locals {
  config = {
    name = "John"
    age  = 30
    city = "NYC"
  }
  
  # Keys and values
  all_keys   = keys(local.config)    # ["age", "city", "name"]
  all_values = values(local.config)  # [30, "NYC", "John"]
  
  # Lookup
  name = lookup(local.config, "name", "default")  # "John"
  missing = lookup(local.config, "missing", "N/A")  # "N/A"
}
```

**Merge Maps:**
```hcl
locals {
  default_tags = {
    Environment = "production"
    ManagedBy   = "Terraform"
  }
  
  custom_tags = {
    Project = "MyApp"
    Owner   = "Platform Team"
  }
  
  # Merge (later values override)
  all_tags = merge(local.default_tags, local.custom_tags)
  # {
  #   Environment = "production"
  #   ManagedBy   = "Terraform"
  #   Project     = "MyApp"
  #   Owner       = "Platform Team"
  # }
  
  # Override values
  override = merge(
    {a = 1, b = 2},
    {b = 3, c = 4}
  )  # {a = 1, b = 3, c = 4}
}
```

**Zipmap:**
```hcl
locals {
  keys   = ["name", "age", "city"]
  values = ["John", 30, "NYC"]
  
  map = zipmap(local.keys, local.values)
  # {
  #   name = "John"
  #   age  = 30
  #   city = "NYC"
  # }
}
```

**For Expressions with Maps:**
```hcl
locals {
  instances = {
    web-1 = "10.0.1.10"
    web-2 = "10.0.1.11"
    db-1  = "10.0.2.10"
  }
  
  # Filter
  web_instances = {
    for name, ip in local.instances :
    name => ip
    if startswith(name, "web-")
  }  # {web-1 = "10.0.1.10", web-2 = "10.0.1.11"}
  
  # Transform
  uppercase_names = {
    for name, ip in local.instances :
    upper(name) => ip
  }
}
```

---

### Q10: What are set operations in Terraform?

**Answer:**

**Set Intersection:**
```hcl
locals {
  set1 = ["a", "b", "c", "d"]
  set2 = ["c", "d", "e", "f"]
  
  # Common elements
  common = setintersection(local.set1, local.set2)  # ["c", "d"]
}
```

**Set Union:**
```hcl
locals {
  set1 = ["a", "b", "c"]
  set2 = ["c", "d", "e"]
  
  # All unique elements
  all = setunion(local.set1, local.set2)  # ["a", "b", "c", "d", "e"]
}
```

**Set Subtract:**
```hcl
locals {
  all_features = ["feature-a", "feature-b", "feature-c", "feature-d"]
  disabled = ["feature-b", "feature-d"]
  
  # Elements in first set but not in second
  enabled = setsubtract(local.all_features, local.disabled)  # ["feature-a", "feature-c"]
}
```

**Set Product:**
```hcl
locals {
  environments = ["dev", "staging", "prod"]
  regions = ["us-east-1", "us-west-2"]
  
  # Cartesian product
  combinations = setproduct(local.environments, local.regions)
  # [
  #   ["dev", "us-east-1"],
  #   ["dev", "us-west-2"],
  #   ["staging", "us-east-1"],
  #   ["staging", "us-west-2"],
  #   ["prod", "us-east-1"],
  #   ["prod", "us-west-2"]
  # ]
}

# Use in resources
resource "aws_instance" "multi_region" {
  for_each = {
    for combo in local.combinations :
    "${combo[0]}-${combo[1]}" => {
      environment = combo[0]
      region      = combo[1]
    }
  }
  
  # Configuration using each.value.environment and each.value.region
}
```

---

## 🎯 Conditional and Type Functions

### Q11: How do you use conditional expressions in Terraform?

**Answer:**

**Ternary Operator:**
```hcl
locals {
  environment = "prod"
  
  # Simple condition
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
  
  # Nested conditions
  instance_count = (
    var.environment == "prod" ? 5 :
    var.environment == "staging" ? 3 :
    1
  )
}
```

**Conditional Resources:**
```hcl
resource "aws_instance" "web" {
  count = var.create_instance ? 1 : 0
  
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}

resource "aws_eip" "web" {
  count = var.create_instance && var.assign_eip ? 1 : 0
  
  instance = aws_instance.web[0].id
}
```

**can Function:**
```hcl
locals {
  # Check if expression is valid
  is_valid_cidr = can(cidrhost("10.0.0.0/16", 0))  # true
  is_invalid    = can(cidrhost("invalid", 0))       # false
  
  # Use in validation
  cidr_valid = can(regex("^[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}/[0-9]{1,2}$", var.cidr))
}
```

**try Function:**
```hcl
locals {
  # Try expressions in order
  config = try(
    jsondecode(file("config.json")),
    yamldecode(file("config.yaml")),
    {}  # Default if both fail
  )
  
  # Safe attribute access
  instance_type = try(
    var.instance_config.type,
    "t2.micro"
  )
}
```

**coalesce Function:**
```hcl
locals {
  # Return first non-null value
  name = coalesce(
    var.custom_name,
    var.default_name,
    "fallback-name"
  )
  
  # With empty strings
  region = coalesce(
    var.override_region != "" ? var.override_region : null,
    var.default_region,
    "us-east-1"
  )
}
```

---

### Q12: How do you perform type conversions in Terraform?

**Answer:**

**Basic Type Conversions:**
```hcl
locals {
  # To string
  str_from_num  = tostring(42)        # "42"
  str_from_bool = tostring(true)      # "true"
  
  # To number
  num_from_str = tonumber("42")       # 42
  num_from_str_float = tonumber("3.14")  # 3.14
  
  # To bool
  bool_from_str = tobool("true")      # true
  bool_from_str_false = tobool("false")  # false
}
```

**Collection Type Conversions:**
```hcl
locals {
  # To list
  list_from_set = tolist(toset(["a", "b", "c"]))
  
  # To set (removes duplicates)
  set_from_list = toset(["a", "b", "b", "c"])  # ["a", "b", "c"]
  
  # To map
  map_from_object = tomap({
    name = "John"
    age  = 30
  })
}
```

**Type Checking:**
```hcl
locals {
  value = [1, 2, 3]
  
  value_type = type(local.value)  # "list"
  
  # Check types
  is_list   = type(local.value) == "list"    # true
  is_map    = type(local.value) == "map"     # false
  is_string = type(local.value) == "string"  # false
}
```

---

## 🌐 Network and File Functions

### Q13: How do you use CIDR functions for network calculations?

**Answer:**

**cidrsubnet - Calculate Subnets:**
```hcl
locals {
  vpc_cidr = "10.0.0.0/16"
  
  # Split /16 into /24 subnets
  subnet_1 = cidrsubnet(local.vpc_cidr, 8, 0)  # "10.0.0.0/24"
  subnet_2 = cidrsubnet(local.vpc_cidr, 8, 1)  # "10.0.1.0/24"
  subnet_3 = cidrsubnet(local.vpc_cidr, 8, 2)  # "10.0.2.0/24"
  
  # Dynamic subnet creation
  public_subnets = [
    for i in range(3) :
    cidrsubnet(local.vpc_cidr, 8, i)
  ]
  
  private_subnets = [
    for i in range(3) :
    cidrsubnet(local.vpc_cidr, 8, i + 100)
  ]
}
```

**cidrhost - Get IP Address:**
```hcl
locals {
  subnet = "10.0.1.0/24"
  
  first_ip = cidrhost(local.subnet, 0)    # "10.0.1.0"
  second_ip = cidrhost(local.subnet, 1)   # "10.0.1.1"
  last_ip = cidrhost(local.subnet, 255)   # "10.0.1.255"
}
```

**cidrnetmask - Get Netmask:**
```hcl
locals {
  cidr = "10.0.0.0/16"
  
  netmask = cidrnetmask(local.cidr)  # "255.255.0.0"
}
```

**cidrsubnets - Multiple Subnets:**
```hcl
locals {
  vpc_cidr = "10.0.0.0/16"
  
  # Create multiple subnets with different sizes
  subnets = cidrsubnets(local.vpc_cidr, 4, 4, 8, 8)
  # [
  #   "10.0.0.0/20",   # /16 + 4 = /20
  #   "10.0.16.0/20",  # /16 + 4 = /20
  #   "10.0.32.0/24",  # /16 + 8 = /24
  #   "10.0.33.0/24"   # /16 + 8 = /24
  # ]
}
```

---

### Q14: How do you work with files in Terraform?

**Answer:**

**Read Files:**
```hcl
locals {
  # Read text file
  config_text = file("${path.module}/config.txt")
  
  # Read and parse JSON
  config_json = jsondecode(file("${path.module}/config.json"))
  
  # Read and parse YAML
  config_yaml = yamldecode(file("${path.module}/config.yaml"))
  
  # Read binary file as base64
  image_data = filebase64("${path.module}/logo.png")
}
```

**Check File Existence:**
```hcl
locals {
  has_config = fileexists("${path.module}/config.json")
  
  # Conditional file reading
  config = local.has_config ? jsondecode(file("${path.module}/config.json")) : {}
}
```

**Find Files:**
```hcl
locals {
  # Find all .tf files
  tf_files = fileset(path.module, "*.tf")
  
  # Find files in subdirectories
  yaml_configs = fileset(path.module, "configs/**/*.yaml")
  
  # Use in resources
  config_files = {
    for file in fileset(path.module, "configs/*.json") :
    file => jsondecode(file("${path.module}/${file}"))
  }
}
```

**Template Files:**
```hcl
locals {
  user_data = templatefile("${path.module}/user_data.sh.tpl", {
    instance_name = "web-server"
    environment   = var.environment
    app_version   = var.app_version
  })
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
  user_data     = local.user_data
}
```

**File Hashing:**
```hcl
locals {
  # MD5 hash
  config_md5 = filemd5("${path.module}/config.json")
  
  # SHA256 hash
  config_sha256 = filesha256("${path.module}/config.json")
  
  # Use for change detection
  config_hash = md5(file("${path.module}/config.json"))
}

resource "null_resource" "config_changed" {
  triggers = {
    config_hash = local.config_hash
  }
  
  provisioner "local-exec" {
    command = "./apply-config.sh"
  }
}
```

---

### Q15: How do you encode and decode data in Terraform?

**Answer:**

**Base64 Encoding:**
```hcl
locals {
  # Encode string
  encoded = base64encode("hello world")
  
  # Decode string
  decoded = base64decode(local.encoded)
  
  # Encode and compress
  compressed = base64gzip("large text content")
}
```

**JSON Encoding:**
```hcl
locals {
  config = {
    name = "John"
    age  = 30
    tags = ["admin", "user"]
  }
  
  # Encode to JSON
  json_string = jsonencode(local.config)
  # '{"age":30,"name":"John","tags":["admin","user"]}'
  
  # Decode from JSON
  parsed = jsondecode(local.json_string)
}

# Use in resources
resource "aws_s3_object" "config" {
  bucket  = aws_s3_bucket.main.id
  key     = "config.json"
  content = jsonencode(local.config)
}
```

**YAML Encoding:**
```hcl
locals {
  config = {
    name = "John"
    age  = 30
    tags = ["admin", "user"]
  }
  
  # Encode to YAML
  yaml_string = yamlencode(local.config)
  
  # Decode from YAML
  parsed = yamldecode(local.yaml_string)
}
```

**URL Encoding:**
```hcl
locals {
  query_param = "hello world!"
  
  # URL encode
  encoded = urlencode(local.query_param)  # "hello+world%21"
  
  # Use in URL
  api_url = "https://api.example.com/search?q=${urlencode(var.search_term)}"
}
```

**Hashing:**
```hcl
locals {
  password = "my-password"
  
  # MD5
  md5_hash = md5(local.password)
  
  # SHA256
  sha256_hash = sha256(local.password)
  
  # Bcrypt (for passwords)
  bcrypt_hash = bcrypt(local.password)
  
  # UUID
  random_id = uuid()
  namespace_id = uuidv5("dns", "example.com")
}
```

---

## 📚 Summary

**Key Topics Covered:**
- Data sources for querying infrastructure
- String manipulation functions
- Numeric operations
- Collection functions (lists, maps, sets)
- Conditional logic and type conversion
- File operations and templates
- Network CIDR calculations
- Encoding and hashing

**Best Practices:**
1. Use data sources to query existing infrastructure
2. Leverage built-in functions instead of external scripts
3. Combine functions for complex transformations
4. Use `terraform console` to test functions
5. Validate inputs with `can()` and regex
6. Use `try()` for safe attribute access
7. Cache expensive operations in locals
8. Document complex function chains

**Next Steps:**
1. Practice labs in [LABS.md](./LABS.md)
2. Experiment with `terraform console`
3. Proceed to [Chapter 8](../chapter-08/)

---

**💡 Pro Tips:**
- Use `terraform console` to test functions interactively
- Combine multiple functions for powerful transformations
- Use `can()` for validation without errors
- Leverage `try()` for safe optional values
- Cache computed values in locals for reuse
- Use data sources instead of hardcoding values
