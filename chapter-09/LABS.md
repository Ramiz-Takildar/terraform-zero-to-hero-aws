# Chapter 9: Labs - CI/CD Integration with Terraform

## 🎯 Lab Overview

These hands-on labs will help you implement CI/CD pipelines for Terraform using GitHub Actions, GitLab CI, and automated testing strategies.

**Prerequisites:**
- AWS account with appropriate permissions
- GitHub or GitLab account
- Git installed and configured
- Terraform installed (v1.0+)
- Completed Chapters 1-8 labs

**Estimated Time:** 3-4 hours total

---

## 🧪 Lab 9.1: GitHub Actions CI/CD Pipeline

**Objective:** Implement a complete CI/CD pipeline using GitHub Actions for Terraform.

**Duration:** 90 minutes

### Step 1: Create GitHub Repository

```bash
# Create new directory
mkdir terraform-cicd-demo
cd terraform-cicd-demo

# Initialize git
git init

# Create GitHub repository (using GitHub CLI)
gh repo create terraform-cicd-demo --public --source=. --remote=origin

# Or create manually at github.com and add remote
git remote add origin https://github.com/YOUR_USERNAME/terraform-cicd-demo.git
```

### Step 2: Create Terraform Configuration

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
  
  backend "s3" {
    bucket         = "YOUR_BUCKET_NAME"  # Change this
    key            = "terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = var.environment
      ManagedBy   = "Terraform"
      Repository  = "terraform-cicd-demo"
    }
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "${var.project_name}-vpc"
  }
}

# Subnets
resource "aws_subnet" "public" {
  count = var.subnet_count
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.project_name}-public-${count.index + 1}"
    Type = "public"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.project_name}-igw"
  }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "${var.project_name}-public-rt"
  }
}

# Route Table Association
resource "aws_route_table_association" "public" {
  count = var.subnet_count
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${var.project_name}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTP"
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS"
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.project_name}-web-sg"
  }
}

# Data Sources
data "aws_availability_zones" "available" {
  state = "available"
}
```

**variables.tf:**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "project_name" {
  description = "Project name"
  type        = string
  default     = "cicd-demo"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_count" {
  description = "Number of subnets"
  type        = number
  default     = 2
  
  validation {
    condition     = var.subnet_count >= 2 && var.subnet_count <= 6
    error_message = "Subnet count must be between 2 and 6."
  }
}
```

**outputs.tf:**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "subnet_ids" {
  description = "Subnet IDs"
  value       = aws_subnet.public[*].id
}

output "security_group_id" {
  description = "Security group ID"
  value       = aws_security_group.web.id
}
```

### Step 3: Create Backend Resources

**setup-backend.sh:**

```bash
#!/bin/bash

# Variables
BUCKET_NAME="terraform-state-cicd-demo-$(date +%s)"
TABLE_NAME="terraform-locks"
REGION="us-east-1"

echo "Creating S3 bucket: $BUCKET_NAME"
aws s3api create-bucket \
  --bucket $BUCKET_NAME \
  --region $REGION

echo "Enabling versioning on S3 bucket"
aws s3api put-bucket-versioning \
  --bucket $BUCKET_NAME \
  --versioning-configuration Status=Enabled

echo "Enabling encryption on S3 bucket"
aws s3api put-bucket-encryption \
  --bucket $BUCKET_NAME \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      }
    }]
  }'

echo "Creating DynamoDB table: $TABLE_NAME"
aws dynamodb create-table \
  --table-name $TABLE_NAME \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region $REGION

echo ""
echo "Backend resources created successfully!"
echo "Update main.tf with bucket name: $BUCKET_NAME"
```

```bash
chmod +x setup-backend.sh
./setup-backend.sh
```

### Step 4: Create GitHub Actions Workflow

**.github/workflows/terraform.yml:**

```yaml
name: 'Terraform CI/CD'

on:
  push:
    branches:
      - main
      - develop
  pull_request:
    branches:
      - main

env:
  TF_VERSION: '1.6.0'
  AWS_REGION: 'us-east-1'

jobs:
  terraform-validate:
    name: 'Validate'
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Format Check
        id: fmt
        run: terraform fmt -check -recursive
        continue-on-error: true
      
      - name: Terraform Init
        run: terraform init -backend=false
      
      - name: Terraform Validate
        id: validate
        run: terraform validate -no-color
      
      - name: Comment Format Status
        if: github.event_name == 'pull_request' && steps.fmt.outcome == 'failure'
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '❌ **Terraform Format Check Failed**\n\nPlease run `terraform fmt -recursive` to fix formatting issues.'
            });

  terraform-security:
    name: 'Security Scan'
    runs-on: ubuntu-latest
    needs: terraform-validate
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: true
          format: sarif
          out: tfsec-results.sarif
      
      - name: Upload SARIF file
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: tfsec-results.sarif

  terraform-plan:
    name: 'Plan'
    runs-on: ubuntu-latest
    needs: [terraform-validate, terraform-security]
    if: github.event_name == 'pull_request'
    
    permissions:
      contents: read
      pull-requests: write
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Plan
        id: plan
        run: |
          terraform plan -no-color -out=tfplan
          terraform show -no-color tfplan > plan.txt
        continue-on-error: true
      
      - name: Update Pull Request
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan.txt', 'utf8');
            const maxLength = 65000;
            const truncatedPlan = plan.length > maxLength 
              ? plan.substring(0, maxLength) + '\n\n... (truncated)'
              : plan;
            
            const output = `#### Terraform Plan 📖
            
            <details><summary>Show Plan</summary>
            
            \`\`\`terraform
            ${truncatedPlan}
            \`\`\`
            
            </details>
            
            *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });
      
      - name: Terraform Plan Status
        if: steps.plan.outcome == 'failure'
        run: exit 1

  terraform-apply:
    name: 'Apply'
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    environment:
      name: production
      url: https://console.aws.amazon.com
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Apply
        run: terraform apply -auto-approve
      
      - name: Save Outputs
        run: terraform output -json > outputs.json
      
      - name: Upload Outputs
        uses: actions/upload-artifact@v4
        with:
          name: terraform-outputs
          path: outputs.json
          retention-days: 30
```

### Step 5: Configure GitHub Secrets

```bash
# Go to GitHub repository settings
# Settings > Secrets and variables > Actions > New repository secret

# Add the following secrets:
# - AWS_ACCESS_KEY_ID
# - AWS_SECRET_ACCESS_KEY
```

### Step 6: Test the Pipeline

```bash
# Create .gitignore
cat > .gitignore <<EOF
# Terraform
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
.terraform.lock.hcl

# IDE
.vscode/
.idea/

# OS
.DS_Store
EOF

# Commit and push
git add .
git commit -m "Initial commit with Terraform and GitHub Actions"
git push -u origin main

# Create a feature branch
git checkout -b feature/add-tags

# Make a change
cat >> main.tf <<EOF

# Additional tags
locals {
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
  }
}
EOF

# Commit and create PR
git add .
git commit -m "Add common tags"
git push -u origin feature/add-tags

# Create PR on GitHub
gh pr create --title "Add common tags" --body "Adding common tags to resources"
```

### Step 7: Verify Pipeline Execution

```bash
# Check workflow runs
gh run list

# View specific run
gh run view

# Watch run in real-time
gh run watch
```

### Step 8: Cleanup

```bash
# Destroy infrastructure
terraform destroy -auto-approve

# Delete backend resources
aws s3 rb s3://YOUR_BUCKET_NAME --force
aws dynamodb delete-table --table-name terraform-locks --region us-east-1
```

### ✅ Success Criteria

- [ ] GitHub Actions workflow created
- [ ] Pipeline validates Terraform code
- [ ] Security scanning runs successfully
- [ ] Plan posted as PR comment
- [ ] Apply runs on merge to main
- [ ] Outputs saved as artifacts
- [ ] All stages pass successfully

---

## 🧪 Lab 9.2: GitLab CI/CD Pipeline

**Objective:** Implement a multi-environment CI/CD pipeline using GitLab CI.

**Duration:** 90 minutes

### Step 1: Create GitLab Repository

```bash
# Create new directory
mkdir terraform-gitlab-cicd
cd terraform-gitlab-cicd

# Initialize git
git init

# Create GitLab project and add remote
git remote add origin https://gitlab.com/YOUR_USERNAME/terraform-gitlab-cicd.git
```

### Step 2: Create Terraform Configuration

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
  
  backend "s3" {
    bucket         = "terraform-state-gitlab-cicd"
    key            = "env/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks-gitlab"
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = {
    Name        = "${var.environment}-vpc"
    Environment = var.environment
  }
}

# Subnets
resource "aws_subnet" "public" {
  count = 2
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name        = "${var.environment}-public-${count.index + 1}"
    Environment = var.environment
  }
}

# Data Sources
data "aws_availability_zones" "available" {
  state = "available"
}
```

**variables.tf:**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}
```

**outputs.tf:**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "environment" {
  description = "Environment name"
  value       = var.environment
}
```

### Step 3: Create Environment-Specific Variables

**environments/dev.tfvars:**

```hcl
environment = "dev"
vpc_cidr    = "10.0.0.0/16"
```

**environments/staging.tfvars:**

```hcl
environment = "staging"
vpc_cidr    = "10.1.0.0/16"
```

**environments/prod.tfvars:**

```hcl
environment = "prod"
vpc_cidr    = "10.2.0.0/16"
```

### Step 4: Create GitLab CI Pipeline

**.gitlab-ci.yml:**

```yaml
image:
  name: hashicorp/terraform:1.6
  entrypoint: [""]

variables:
  TF_ROOT: ${CI_PROJECT_DIR}
  AWS_DEFAULT_REGION: us-east-1

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - ${TF_ROOT}/.terraform

stages:
  - validate
  - plan-dev
  - apply-dev
  - plan-staging
  - apply-staging
  - plan-prod
  - apply-prod

before_script:
  - cd ${TF_ROOT}
  - terraform --version

.terraform-init:
  script:
    - |
      terraform init \
        -backend-config="key=${CI_ENVIRONMENT_NAME}/terraform.tfstate"

validate:
  stage: validate
  script:
    - terraform init -backend=false
    - terraform fmt -check -recursive
    - terraform validate
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'

security-scan:
  stage: validate
  image: aquasec/tfsec:latest
  script:
    - tfsec . --format json --out tfsec-report.json
  artifacts:
    reports:
      sast: tfsec-report.json
    paths:
      - tfsec-report.json
    expire_in: 1 week
  allow_failure: true
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'

# Development Environment
plan:dev:
  stage: plan-dev
  environment:
    name: development
    action: prepare
  script:
    - !reference [.terraform-init, script]
    - terraform plan -var-file="environments/dev.tfvars" -out=tfplan-dev
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan-dev
    expire_in: 1 week
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

apply:dev:
  stage: apply-dev
  environment:
    name: development
    action: start
  script:
    - !reference [.terraform-init, script]
    - terraform apply -auto-approve tfplan-dev
    - terraform output -json > outputs-dev.json
  artifacts:
    paths:
      - ${TF_ROOT}/outputs-dev.json
    expire_in: 1 month
  dependencies:
    - plan:dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      when: manual

# Staging Environment
plan:staging:
  stage: plan-staging
  environment:
    name: staging
    action: prepare
  script:
    - !reference [.terraform-init, script]
    - terraform plan -var-file="environments/staging.tfvars" -out=tfplan-staging
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan-staging
    expire_in: 1 week
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

apply:staging:
  stage: apply-staging
  environment:
    name: staging
    action: start
  script:
    - !reference [.terraform-init, script]
    - terraform apply -auto-approve tfplan-staging
    - terraform output -json > outputs-staging.json
  artifacts:
    paths:
      - ${TF_ROOT}/outputs-staging.json
    expire_in: 1 month
  dependencies:
    - plan:staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual

# Production Environment
plan:prod:
  stage: plan-prod
  environment:
    name: production
    action: prepare
  script:
    - !reference [.terraform-init, script]
    - terraform plan -var-file="environments/prod.tfvars" -out=tfplan-prod
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan-prod
    expire_in: 1 week
  needs:
    - apply:staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

apply:prod:
  stage: apply-prod
  environment:
    name: production
    action: start
  script:
    - !reference [.terraform-init, script]
    - terraform apply -auto-approve tfplan-prod
    - terraform output -json > outputs-prod.json
  artifacts:
    paths:
      - ${TF_ROOT}/outputs-prod.json
    expire_in: 1 month
  dependencies:
    - plan:prod
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

### Step 5: Configure GitLab CI/CD Variables

```bash
# Go to GitLab project settings
# Settings > CI/CD > Variables

# Add the following variables:
# - AWS_ACCESS_KEY_ID (Protected, Masked)
# - AWS_SECRET_ACCESS_KEY (Protected, Masked)
```

### Step 6: Test the Pipeline

```bash
# Create environments directory
mkdir -p environments

# Add all files
git add .
git commit -m "Initial commit with GitLab CI"
git push -u origin main

# Create develop branch
git checkout -b develop
git push -u origin develop

# Make a change
echo "# Test change" >> README.md
git add README.md
git commit -m "Test pipeline"
git push

# Create merge request
# Go to GitLab UI and create MR from develop to main
```

### Step 7: Monitor Pipeline

```bash
# View pipeline status in GitLab UI
# Project > CI/CD > Pipelines

# Check job logs
# Click on any job to view logs

# Download artifacts
# Click on job > Download artifacts
```

### Step 8: Cleanup

```bash
# Destroy all environments
for env in dev staging prod; do
  terraform init -backend-config="key=${env}/terraform.tfstate"
  terraform destroy -var-file="environments/${env}.tfvars" -auto-approve
done
```

### ✅ Success Criteria

- [ ] GitLab CI pipeline created
- [ ] Multi-environment workflow configured
- [ ] Validation runs on all branches
- [ ] Security scanning integrated
- [ ] Manual approval for apply stages
- [ ] Artifacts saved for each environment
- [ ] Pipeline executes successfully

---

## 🧪 Lab 9.3: Automated Testing with Terratest

**Objective:** Implement automated testing for Terraform infrastructure using Terratest.

**Duration:** 60 minutes

### Step 1: Setup Project Structure

```bash
mkdir terraform-testing
cd terraform-testing

# Create directory structure
mkdir -p {terraform,test}
```

### Step 2: Create Terraform Module

**terraform/main.tf:**

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

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name        = "${var.name_prefix}-vpc"
    Environment = var.environment
  }
}

# Subnets
resource "aws_subnet" "public" {
  count = var.subnet_count
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = "${var.name_prefix}-public-${count.index + 1}"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "${var.name_prefix}-igw"
  }
}

# Security Group
resource "aws_security_group" "web" {
  name        = "${var.name_prefix}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${var.name_prefix}-web-sg"
  }
}

# Data Sources
data "aws_availability_zones" "available" {
  state = "available"
}
```

**terraform/variables.tf:**

```hcl
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "name_prefix" {
  description = "Name prefix for resources"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "test"
}

variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  default     = "10.0.0.0/16"
}

variable "subnet_count" {
  description = "Number of subnets"
  type        = number
  default     = 2
}

variable "allowed_ports" {
  description = "List of allowed ports"
  type        = list(number)
  default     = [80, 443]
}
```

**terraform/outputs.tf:**

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "subnet_ids" {
  description = "List of subnet IDs"
  value       = aws_subnet.public[*].id
}

output "subnet_count" {
  description = "Number of subnets"
  value       = length(aws_subnet.public)
}

output "security_group_id" {
  description = "Security group ID"
  value       = aws_security_group.web.id
}

output "internet_gateway_id" {
  description = "Internet gateway ID"
  value       = aws_internet_gateway.main.id
}
```

### Step 3: Install Go and Terratest

```bash
# Install Go (if not already installed)
# macOS
brew install go

# Linux
wget https://go.dev/dl/go1.21.0.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.21.0.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin

# Initialize Go module
cd test
go mod init github.com/YOUR_USERNAME/terraform-testing
go get github.com/gruntwork-io/terratest/modules/terraform
go get github.com/stretchr/testify/assert
```

### Step 4: Create Terratest Tests

**test/terraform_test.go:**

```go
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
	"github.com/stretchr/testify/assert"
)

func TestTerraformVPC(t *testing.T) {
	t.Parallel()

	// Terraform options
	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../terraform",
		Vars: map[string]interface{}{
			"name_prefix":  "terratest",
			"environment":  "test",
			"vpc_cidr":     "10.0.0.0/16",
			"subnet_count": 2,
			"allowed_ports": []int{80, 443},
		},
	})

	// Clean up resources
	defer terraform.Destroy(t, terraformOptions)

	// Deploy infrastructure
	terraform.InitAndApply(t, terraformOptions)

	// Test VPC
	vpcID := terraform.Output(t, terraformOptions, "vpc_id")
	assert.NotEmpty(t, vpcID, "VPC ID should not be empty")

	vpcCIDR := terraform.Output(t, terraformOptions, "vpc_cidr")
	assert.Equal(t, "10.0.0.0/16", vpcCIDR, "VPC CIDR should match")

	// Test Subnets
	subnetCount := terraform.Output(t, terraformOptions, "subnet_count")
	assert.Equal(t, "2", subnetCount, "Should have 2 subnets")

	// Test Security Group
	sgID := terraform.Output(t, terraformOptions, "security_group_id")
	assert.NotEmpty(t, sgID, "Security group ID should not be empty")

	// Test Internet Gateway
	igwID := terraform.Output(t, terraformOptions, "internet_gateway_id")
	assert.NotEmpty(t, igwID, "Internet gateway ID should not be empty")
}

func TestTerraformVPCWithCustomCIDR(t *testing.T) {
	t.Parallel()

	terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
		TerraformDir: "../terraform",
		Vars: map[string]interface{}{
			"name_prefix":  "terratest-custom",
			"environment":  "test",
			"vpc_cidr":     "172.16.0.0/16",
			"subnet_count": 3,
		},
	})

	defer terraform.Destroy(t, terraformOptions)

	terraform.InitAndApply(t, terraformOptions)

	vpcCIDR := terraform.Output(t, terraformOptions, "vpc_cidr")
	assert.Equal(t, "172.16.0.0/16", vpcCIDR, "VPC CIDR should match custom value")

	subnetCount := terraform.Output(t, terraformOptions, "subnet_count")
	assert.Equal(t, "3", subnetCount, "Should have 3 subnets")
}
```

**test/terraform_validation_test.go:**

```go
package test

import (
	"testing"

	"github.com/gruntwork-io/terratest/modules/terraform"
)

func TestTerraformValidation(t *testing.T) {
	terraformOptions := &terraform.Options{
		TerraformDir: "../terraform",
	}

	// Test terraform init
	terraform.Init(t, terraformOptions)

	// Test terraform validate
	terraform.Validate(t, terraformOptions)
}

func TestTerraformFormat(t *testing.T) {
	terraformOptions := &terraform.Options{
		TerraformDir: "../terraform",
	}

	// Test terraform fmt
	terraform.Format(t, terraformOptions)
}
```

### Step 5: Run Tests

```bash
cd test

# Run all tests
go test -v -timeout 30m

# Run specific test
go test -v -run TestTerraformVPC -timeout 30m

# Run tests in parallel
go test -v -parallel 2 -timeout 30m

# Run with coverage
go test -v -cover -timeout 30m
```

### Step 6: Integrate with CI/CD

**.github/workflows/test.yml:**

```yaml
name: 'Terraform Tests'

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  terratest:
    name: 'Terratest'
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.21'
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: '1.6.0'
          terraform_wrapper: false
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Download Go modules
        working-directory: test
        run: go mod download
      
      - name: Run Terratest
        working-directory: test
        run: go test -v -timeout 30m
```

### Step 7: View Test Results

```bash
# Test output shows:
# - Resources created
# - Test assertions
# - Resources destroyed
# - Test results (PASS/FAIL)

# Example output:
# === RUN   TestTerraformVPC
# TestTerraformVPC 2024/01/01 12:00:00 Running terraform init...
# TestTerraformVPC 2024/01/01 12:00:05 Running terraform apply...
# TestTerraformVPC 2024/01/01 12:01:30 Running terraform output...
# TestTerraformVPC 2024/01/01 12:01:31 Running terraform destroy...
# --- PASS: TestTerraformVPC (120.45s)
# PASS
```

### Step 8: Cleanup

```bash
# Tests automatically clean up resources
# Manual cleanup if needed:
cd ../terraform
terraform destroy -auto-approve
```

### ✅ Success Criteria

- [ ] Terratest installed and configured
- [ ] Test files created
- [ ] Tests validate infrastructure
- [ ] Tests run successfully
- [ ] Resources created and destroyed
- [ ] CI/CD integration working
- [ ] All assertions pass

---

## 🎓 Lab Summary

### What You Learned

1. **Lab 9.1:** GitHub Actions CI/CD pipeline for Terraform
2. **Lab 9.2:** GitLab CI/CD multi-environment pipeline
3. **Lab 9.3:** Automated testing with Terratest

### Key Concepts Practiced

- CI/CD pipeline design
- GitHub Actions workflows
- GitLab CI/CD pipelines
- Security scanning integration
- Multi-environment deployments
- Automated testing
- Infrastructure validation
- Approval workflows

### Best Practices Learned

1. Always validate before plan
2. Run security scans in pipeline
3. Use manual approvals for production
4. Save plan outputs as artifacts
5. Test infrastructure automatically
6. Clean up test resources
7. Use environment-specific configurations
8. Implement proper secret management

### Next Steps

1. ✅ Mark labs complete in [CHECKLIST.md](../CHECKLIST.md)
2. 📝 Review [INTERVIEW.md](./INTERVIEW.md) questions
3. 🚀 Proceed to [Chapter 10](../chapter-10/)

---

## 💡 Troubleshooting Tips

### GitHub Actions Issues

```bash
# Check workflow syntax
gh workflow view terraform.yml

# View run logs
gh run view --log

# Re-run failed jobs
gh run rerun <run-id>
```

### GitLab CI Issues

```bash
# Validate .gitlab-ci.yml
# Project > CI/CD > Editor > Validate

# Check job logs in UI
# Project > CI/CD > Pipelines > Job

# Download artifacts
# Job > Download artifacts button
```

### Terratest Issues

```bash
# Increase timeout
go test -v -timeout 60m

# Run with verbose output
go test -v -run TestName

# Clean up stuck resources
terraform destroy -auto-approve
```

---

**🎉 Congratulations!** You've mastered CI/CD integration with Terraform!
