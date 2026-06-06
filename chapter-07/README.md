# Chapter 7: Data Sources and Terraform Functions

## 📚 Learning Objectives

By the end of this chapter, you will:
- Master Terraform data sources
- Use built-in functions effectively
- Work with string, numeric, and collection functions
- Implement conditional logic
- Use type conversion functions
- Work with file and encoding functions
- Combine functions for complex operations

**Prerequisites:** Chapters 1-6 completed  
**Estimated Time:** 2 days  
**Labs:** 3 hands-on exercises

---

## 📊 Data Sources

### What are Data Sources?

Data sources allow Terraform to fetch information from external sources or existing infrastructure.

**Key Characteristics:**
- **Read-only:** Don't create or modify resources
- **Dynamic:** Fetch current information
- **Reusable:** Reference existing infrastructure
- **Flexible:** Query with filters

**Basic Syntax:**

```hcl
data "aws_ami" "amazon_linux_2" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.amazon_linux_2.id
  instance_type = "t2.micro"
}
```

---

## 🔍 Common AWS Data Sources

### AMI Data Source

```hcl
# Latest Amazon Linux 2
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

# Latest Ubuntu
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]  # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*"]
  }
}

# Specific AMI by tag
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

### Availability Zones

```hcl
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_availability_zones" "excluded" {
  state = "available"
  
  exclude_names = ["us-east-1e"]
}

# Use in resources
resource "aws_subnet" "public" {
  count = length(data.aws_availability_zones.available.names)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

### VPC and Subnets

```hcl
# Default VPC
data "aws_vpc" "default" {
  default = true
}

# VPC by tag
data "aws_vpc" "selected" {
  tags = {
    Name = "production-vpc"
  }
}

# VPC by filter
data "aws_vpc" "main" {
  filter {
    name   = "tag:Environment"
    values = ["production"]
  }
}

# Subnets in VPC
data "aws_subnets" "private" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.selected.id]
  }
  
  tags = {
    Type = "private"
  }
}

# Specific subnet
data "aws_subnet" "selected" {
  id = "subnet-12345678"
}
```

### Security Groups

```hcl
# Security group by name
data "aws_security_group" "web" {
  name   = "web-sg"
  vpc_id = data.aws_vpc.main.id
}

# Security group by tag
data "aws_security_group" "database" {
  tags = {
    Name        = "database-sg"
    Environment = "production"
  }
}

# Security group by filter
data "aws_security_group" "selected" {
  filter {
    name   = "group-name"
    values = ["app-*"]
  }
  
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.main.id]
  }
}
```

### IAM Resources

```hcl
# Current AWS account
data "aws_caller_identity" "current" {}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}

output "caller_arn" {
  value = data.aws_caller_identity.current.arn
}

# IAM policy document
data "aws_iam_policy_document" "assume_role" {
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
  assume_role_policy = data.aws_iam_policy_document.assume_role.json
}

# Existing IAM role
data "aws_iam_role" "existing" {
  name = "existing-role"
}
```

### Route53

```hcl
# Hosted zone
data "aws_route53_zone" "main" {
  name         = "example.com"
  private_zone = false
}

resource "aws_route53_record" "www" {
  zone_id = data.aws_route53_zone.main.zone_id
  name    = "www.${data.aws_route53_zone.main.name}"
  type    = "A"
  ttl     = 300
  records = [aws_instance.web.public_ip]
}
```

### S3 Buckets

```hcl
# Existing bucket
data "aws_s3_bucket" "existing" {
  bucket = "my-existing-bucket"
}

# S3 objects
data "aws_s3_object" "config" {
  bucket = data.aws_s3_bucket.existing.id
  key    = "config/app.json"
}
```

### Secrets Manager

```hcl
data "aws_secretsmanager_secret" "db_password" {
  name = "production/db/password"
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}

resource "aws_db_instance" "main" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}
```

---

## 🔧 String Functions

### Basic String Operations

```hcl
# upper / lower
locals {
  name_upper = upper("hello")        # "HELLO"
  name_lower = lower("WORLD")        # "world"
  name_title = title("hello world")  # "Hello World"
}

# trim / trimspace
locals {
  trimmed       = trim("  hello  ", " ")     # "hello"
  trimmed_space = trimspace("  hello  ")     # "hello"
  trim_prefix   = trimprefix("hello", "he")  # "llo"
  trim_suffix   = trimsuffix("hello", "lo")  # "hel"
}

# replace
locals {
  replaced = replace("hello-world", "-", "_")  # "hello_world"
  
  # Regex replace
  cleaned = replace("hello123world456", "/[0-9]/", "")  # "helloworld"
}
```

### String Manipulation

```hcl
# split / join
locals {
  parts  = split("-", "hello-world-test")  # ["hello", "world", "test"]
  joined = join("-", ["hello", "world"])   # "hello-world"
}

# substr
locals {
  substring = substr("hello world", 0, 5)  # "hello"
  last_part = substr("hello world", 6, -1) # "world"
}

# format / formatlist
locals {
  formatted = format("Hello, %s!", "World")  # "Hello, World!"
  
  urls = formatlist("https://%s.example.com", ["www", "api", "cdn"])
  # ["https://www.example.com", "https://api.example.com", "https://cdn.example.com"]
}

# regex / regexall
locals {
  # Extract version number
  version = regex("v([0-9.]+)", "app-v1.2.3")[0]  # "1.2.3"
  
  # Find all matches
  numbers = regexall("[0-9]+", "abc123def456")  # ["123", "456"]
}
```

### String Checks

```hcl
# startswith / endswith
locals {
  starts = startswith("hello world", "hello")  # true
  ends   = endswith("hello world", "world")    # true
}

# contains (for lists, but works with strings via split)
locals {
  has_substring = contains(split("", "hello"), "e")  # true
}
```

---

## 🔢 Numeric Functions

### Math Operations

```hcl
# abs / ceil / floor
locals {
  absolute = abs(-5)      # 5
  ceiling  = ceil(4.3)    # 5
  floored  = floor(4.7)   # 4
}

# min / max
locals {
  minimum = min(1, 2, 3, 4, 5)  # 1
  maximum = max(1, 2, 3, 4, 5)  # 5
}

# pow / log
locals {
  power     = pow(2, 3)   # 8
  logarithm = log(100, 10) # 2
}

# signum
locals {
  sign_pos = signum(5)   # 1
  sign_neg = signum(-5)  # -1
  sign_zero = signum(0)  # 0
}
```

### Parsing Numbers

```hcl
# parseint
locals {
  decimal = parseint("42", 10)      # 42
  hex     = parseint("2A", 16)      # 42
  binary  = parseint("101010", 2)   # 42
}
```

---

## 📋 Collection Functions

### List Functions

```hcl
# length
locals {
  list_length = length([1, 2, 3, 4, 5])  # 5
  map_length  = length({a = 1, b = 2})   # 2
  string_length = length("hello")         # 5
}

# element / index
locals {
  first = element([1, 2, 3], 0)           # 1
  idx   = index(["a", "b", "c"], "b")     # 1
}

# concat
locals {
  combined = concat([1, 2], [3, 4], [5])  # [1, 2, 3, 4, 5]
}

# contains
locals {
  has_item = contains([1, 2, 3], 2)  # true
}

# distinct
locals {
  unique = distinct([1, 2, 2, 3, 3, 3])  # [1, 2, 3]
}

# flatten
locals {
  flat = flatten([[1, 2], [3, 4], [5]])  # [1, 2, 3, 4, 5]
}

# reverse
locals {
  reversed = reverse([1, 2, 3, 4, 5])  # [5, 4, 3, 2, 1]
}

# slice
locals {
  sliced = slice([1, 2, 3, 4, 5], 1, 4)  # [2, 3, 4]
}

# sort
locals {
  sorted = sort(["c", "a", "b"])  # ["a", "b", "c"]
}

# chunklist
locals {
  chunks = chunklist([1, 2, 3, 4, 5, 6], 2)  # [[1, 2], [3, 4], [5, 6]]
}
```

### Map Functions

```hcl
# keys / values
locals {
  map = {
    name = "John"
    age  = 30
    city = "NYC"
  }
  
  all_keys   = keys(local.map)    # ["age", "city", "name"]
  all_values = values(local.map)  # [30, "NYC", "John"]
}

# lookup
locals {
  value = lookup(local.map, "name", "default")  # "John"
  missing = lookup(local.map, "missing", "N/A") # "N/A"
}

# merge
locals {
  merged = merge(
    {a = 1, b = 2},
    {b = 3, c = 4}
  )  # {a = 1, b = 3, c = 4}
}

# zipmap
locals {
  zipped = zipmap(
    ["name", "age", "city"],
    ["John", 30, "NYC"]
  )  # {name = "John", age = 30, city = "NYC"}
}
```

### Set Functions

```hcl
# setintersection
locals {
  common = setintersection(
    [1, 2, 3],
    [2, 3, 4]
  )  # [2, 3]
}

# setproduct
locals {
  product = setproduct(
    ["a", "b"],
    [1, 2]
  )  # [["a", 1], ["a", 2], ["b", 1], ["b", 2]]
}

# setsubtract
locals {
  difference = setsubtract(
    [1, 2, 3, 4],
    [2, 4]
  )  # [1, 3]
}

# setunion
locals {
  union = setunion(
    [1, 2, 3],
    [3, 4, 5]
  )  # [1, 2, 3, 4, 5]
}
```

---

## 🎯 Conditional Functions

### Conditional Expressions

```hcl
# Ternary operator
locals {
  environment = "prod"
  instance_type = var.environment == "prod" ? "t2.large" : "t2.micro"
}

# Nested conditions
locals {
  instance_type = (
    var.environment == "prod" ? "t2.large" :
    var.environment == "staging" ? "t2.medium" :
    "t2.micro"
  )
}
```

### Conditional Functions

```hcl
# can
locals {
  # Check if expression is valid
  is_valid_cidr = can(cidrhost("10.0.0.0/16", 0))  # true
  is_invalid    = can(cidrhost("invalid", 0))       # false
}

# try
locals {
  # Try expressions in order
  value = try(
    var.optional_value,
    "default"
  )
  
  # Try parsing JSON
  config = try(
    jsondecode(file("config.json")),
    {}
  )
}

# coalesce
locals {
  # Return first non-null value
  name = coalesce(
    var.custom_name,
    var.default_name,
    "fallback-name"
  )
}

# coalescelist
locals {
  # Return first non-empty list
  zones = coalescelist(
    var.custom_zones,
    data.aws_availability_zones.available.names
  )
}
```

---

## 🔄 Type Conversion Functions

### Type Conversions

```hcl
# tostring / tonumber / tobool
locals {
  str  = tostring(42)        # "42"
  num  = tonumber("42")      # 42
  bool = tobool("true")      # true
}

# tolist / toset / tomap
locals {
  list = tolist(["a", "b", "c"])
  set  = toset(["a", "b", "b", "c"])  # ["a", "b", "c"]
  map  = tomap({a = 1, b = 2})
}

# type
locals {
  list_type   = type([1, 2, 3])           # "list"
  map_type    = type({a = 1})             # "map"
  string_type = type("hello")             # "string"
  number_type = type(42)                  # "number"
  bool_type   = type(true)                # "bool"
}
```

---

## 📁 File and Encoding Functions

### File Functions

```hcl
# file
locals {
  content = file("${path.module}/config.txt")
}

# fileexists
locals {
  has_config = fileexists("${path.module}/config.txt")
}

# fileset
locals {
  # Find all .tf files
  tf_files = fileset(path.module, "*.tf")
  
  # Find all files in subdirectories
  all_configs = fileset(path.module, "configs/**/*.yaml")
}

# templatefile
locals {
  rendered = templatefile("${path.module}/template.tpl", {
    name        = "John"
    environment = var.environment
  })
}

# filebase64
locals {
  encoded = filebase64("${path.module}/image.png")
}

# filemd5 / filesha256 / filesha512
locals {
  md5    = filemd5("${path.module}/file.txt")
  sha256 = filesha256("${path.module}/file.txt")
  sha512 = filesha512("${path.module}/file.txt")
}
```

### Encoding Functions

```hcl
# base64encode / base64decode
locals {
  encoded = base64encode("hello world")
  decoded = base64decode(local.encoded)
}

# base64gzip
locals {
  compressed = base64gzip("large text content")
}

# jsonencode / jsondecode
locals {
  json_string = jsonencode({
    name = "John"
    age  = 30
  })
  
  json_object = jsondecode(local.json_string)
}

# yamlencode / yamldecode
locals {
  yaml_string = yamlencode({
    name = "John"
    age  = 30
  })
  
  yaml_object = yamldecode(local.yaml_string)
}

# urlencode
locals {
  encoded_url = urlencode("hello world!")  # "hello+world%21"
}
```

### Hash Functions

```hcl
# md5 / sha1 / sha256 / sha512
locals {
  md5_hash    = md5("hello world")
  sha1_hash   = sha1("hello world")
  sha256_hash = sha256("hello world")
  sha512_hash = sha512("hello world")
}

# bcrypt
locals {
  hashed_password = bcrypt("my-password")
}

# uuid / uuidv5
locals {
  random_uuid = uuid()
  namespace_uuid = uuidv5("dns", "example.com")
}
```

---

## 🌐 Network Functions

### CIDR Functions

```hcl
# cidrhost
locals {
  first_ip = cidrhost("10.0.0.0/24", 0)    # "10.0.0.0"
  last_ip  = cidrhost("10.0.0.0/24", 255)  # "10.0.0.255"
}

# cidrnetmask
locals {
  netmask = cidrnetmask("10.0.0.0/24")  # "255.255.255.0"
}

# cidrsubnet
locals {
  # Split /16 into /24 subnets
  subnet_1 = cidrsubnet("10.0.0.0/16", 8, 0)  # "10.0.0.0/24"
  subnet_2 = cidrsubnet("10.0.0.0/16", 8, 1)  # "10.0.1.0/24"
  subnet_3 = cidrsubnet("10.0.0.0/16", 8, 2)  # "10.0.2.0/24"
}

# cidrsubnets
locals {
  # Create multiple subnets at once
  subnets = cidrsubnets("10.0.0.0/16", 8, 8, 8, 8)
  # ["10.0.0.0/24", "10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}
```

---

## 🕐 Date and Time Functions

```hcl
# timestamp
locals {
  current_time = timestamp()  # "2024-01-15T10:30:00Z"
}

# formatdate
locals {
  formatted = formatdate("DD MMM YYYY hh:mm:ss", timestamp())
  # "15 Jan 2024 10:30:00"
}

# timeadd
locals {
  future = timeadd(timestamp(), "24h")  # Add 24 hours
  past   = timeadd(timestamp(), "-1h")  # Subtract 1 hour
}
```

---

## 🎨 Advanced Patterns

### Dynamic Subnet Creation

```hcl
locals {
  vpc_cidr = "10.0.0.0/16"
  az_count = 3
  
  # Create public and private subnets
  public_subnets = [
    for i in range(local.az_count) :
    cidrsubnet(local.vpc_cidr, 8, i)
  ]
  
  private_subnets = [
    for i in range(local.az_count) :
    cidrsubnet(local.vpc_cidr, 8, i + 100)
  ]
}

resource "aws_subnet" "public" {
  count = length(local.public_subnets)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.public_subnets[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

### Environment-Based Configuration

```hcl
locals {
  env_config = {
    dev = {
      instance_type = "t2.micro"
      instance_count = 1
      enable_monitoring = false
    }
    prod = {
      instance_type = "t2.large"
      instance_count = 5
      enable_monitoring = true
    }
  }
  
  config = lookup(local.env_config, var.environment, local.env_config["dev"])
}
```

### Tag Generation

```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "Terraform"
    Project     = var.project_name
    CreatedAt   = formatdate("YYYY-MM-DD", timestamp())
  }
  
  # Merge with resource-specific tags
  instance_tags = merge(
    local.common_tags,
    {
      Name = "${var.project_name}-${var.environment}-instance"
      Type = "compute"
    }
  )
}
```

### Complex Data Transformation

```hcl
locals {
  # Transform list of objects
  instances = [
    {name = "web-1", type = "t2.micro", az = "us-east-1a"},
    {name = "web-2", type = "t2.micro", az = "us-east-1b"},
    {name = "db-1", type = "t2.small", az = "us-east-1a"}
  ]
  
  # Group by availability zone
  instances_by_az = {
    for instance in local.instances :
    instance.az => instance...
  }
  
  # Filter by type
  web_instances = [
    for instance in local.instances :
    instance if startswith(instance.name, "web-")
  ]
}
```

---

## 📖 Key Takeaways

1. **Data sources** fetch existing infrastructure information
2. **String functions** manipulate text
3. **Numeric functions** perform calculations
4. **Collection functions** work with lists, maps, and sets
5. **Conditional functions** enable dynamic logic
6. **Type conversion** ensures correct data types
7. **File functions** read and process files
8. **Network functions** handle CIDR calculations
9. **Combine functions** for powerful transformations

---

## 🔗 Next Steps

1. Review theory above
2. Complete [Lab 7.1](./LABS.md) - Data Sources
3. Complete [Lab 7.2](./LABS.md) - String and Collection Functions
4. Complete [Lab 7.3](./LABS.md) - Advanced Function Patterns
5. Update [CHECKLIST.md](../CHECKLIST.md)

**Next Chapter:** [Chapter 8: Advanced Patterns](../chapter-08/)
