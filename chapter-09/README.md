# Chapter 9: CI/CD Integration with Terraform

## 📋 Overview

This chapter covers integrating Terraform into CI/CD pipelines for automated infrastructure deployment. You'll learn how to implement GitOps workflows, automate testing, and deploy infrastructure safely across multiple environments.

**What You'll Learn:**
- CI/CD fundamentals for infrastructure
- GitHub Actions with Terraform
- GitLab CI/CD pipelines
- Jenkins integration
- Terraform Cloud workflows
- Security and best practices
- Automated testing strategies

**Prerequisites:**
- Completed Chapters 1-8
- Git knowledge
- Basic CI/CD understanding
- AWS account access
- GitHub/GitLab account

---

## 🎯 Learning Objectives

By the end of this chapter, you will be able to:

1. ✅ Design CI/CD pipelines for Terraform
2. ✅ Implement GitHub Actions workflows
3. ✅ Configure GitLab CI/CD pipelines
4. ✅ Integrate Terraform with Jenkins
5. ✅ Use Terraform Cloud for automation
6. ✅ Implement automated testing
7. ✅ Secure CI/CD pipelines
8. ✅ Handle secrets and credentials
9. ✅ Implement approval workflows
10. ✅ Monitor and troubleshoot pipelines

---

## 🔄 CI/CD Fundamentals for Infrastructure

### What is Infrastructure CI/CD?

Infrastructure CI/CD applies continuous integration and deployment practices to infrastructure code, enabling:

- **Automated Testing:** Validate infrastructure changes before deployment
- **Consistent Deployments:** Same process across all environments
- **Version Control:** Track all infrastructure changes
- **Rollback Capability:** Revert to previous states
- **Collaboration:** Team-based infrastructure management

### GitOps Workflow

```
┌─────────────┐
│   Developer │
│   Changes   │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  Git Commit │
│   & Push    │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│  CI Pipeline│
│   Triggered │
└──────┬──────┘
       │
       ├──────────────┐
       ▼              ▼
┌─────────────┐  ┌─────────────┐
│   Validate  │  │    Test     │
│   Terraform │  │Infrastructure│
└──────┬──────┘  └──────┬──────┘
       │                │
       └────────┬───────┘
                ▼
         ┌─────────────┐
         │  Plan & PR  │
         │   Comment   │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │   Manual    │
         │  Approval   │
         └──────┬──────┘
                │
                ▼
         ┌─────────────┐
         │   Apply &   │
         │   Deploy    │
         └─────────────┘
```

### Pipeline Stages

**1. Validation Stage:**
```bash
# Format check
terraform fmt -check -recursive

# Validation
terraform init -backend=false
terraform validate

# Linting
tflint --recursive
```

**2. Security Scanning:**
```bash
# Security scanning
tfsec .
checkov -d .
terrascan scan
```

**3. Plan Stage:**
```bash
# Initialize
terraform init

# Plan
terraform plan -out=tfplan

# Show plan
terraform show -json tfplan > plan.json
```

**4. Apply Stage:**
```bash
# Apply with approval
terraform apply tfplan

# Output results
terraform output -json > outputs.json
```

---

## 🐙 GitHub Actions Integration

### Basic Workflow Structure

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
    name: 'Terraform Validate'
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TF_VERSION }}
      
      - name: Terraform Format Check
        run: terraform fmt -check -recursive
      
      - name: Terraform Init
        run: terraform init -backend=false
      
      - name: Terraform Validate
        run: terraform validate

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
          soft_fail: false
      
      - name: Run Checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: .
          framework: terraform
          soft_fail: false

  terraform-plan:
    name: 'Terraform Plan'
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
      
      - name: Comment PR
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const fs = require('fs');
            const plan = fs.readFileSync('plan.txt', 'utf8');
            const output = `#### Terraform Plan 📖
            
            <details><summary>Show Plan</summary>
            
            \`\`\`terraform
            ${plan}
            \`\`\`
            
            </details>
            
            *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            });

  terraform-apply:
    name: 'Terraform Apply'
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
```

### Multi-Environment Workflow

**.github/workflows/terraform-multi-env.yml:**

```yaml
name: 'Multi-Environment Terraform'

on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'

env:
  TF_VERSION: '1.6.0'

jobs:
  determine-environment:
    name: 'Determine Environment'
    runs-on: ubuntu-latest
    outputs:
      environment: ${{ steps.set-env.outputs.environment }}
      aws-region: ${{ steps.set-env.outputs.aws-region }}
    
    steps:
      - name: Set Environment
        id: set-env
        run: |
          if [[ "${{ github.ref }}" == "refs/heads/main" ]]; then
            echo "environment=production" >> $GITHUB_OUTPUT
            echo "aws-region=us-east-1" >> $GITHUB_OUTPUT
          elif [[ "${{ github.ref }}" == "refs/heads/develop" ]]; then
            echo "environment=staging" >> $GITHUB_OUTPUT
            echo "aws-region=us-west-2" >> $GITHUB_OUTPUT
          else
            echo "environment=development" >> $GITHUB_OUTPUT
            echo "aws-region=us-west-2" >> $GITHUB_OUTPUT
          fi

  terraform-deploy:
    name: 'Deploy to ${{ needs.determine-environment.outputs.environment }}'
    runs-on: ubuntu-latest
    needs: determine-environment
    
    environment:
      name: ${{ needs.determine-environment.outputs.environment }}
    
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
          aws-region: ${{ needs.determine-environment.outputs.aws-region }}
      
      - name: Terraform Init
        run: |
          terraform init \
            -backend-config="bucket=terraform-state-${{ needs.determine-environment.outputs.environment }}" \
            -backend-config="key=terraform.tfstate" \
            -backend-config="region=${{ needs.determine-environment.outputs.aws-region }}"
      
      - name: Terraform Workspace
        run: |
          terraform workspace select ${{ needs.determine-environment.outputs.environment }} || \
          terraform workspace new ${{ needs.determine-environment.outputs.environment }}
      
      - name: Terraform Plan
        run: |
          terraform plan \
            -var="environment=${{ needs.determine-environment.outputs.environment }}" \
            -out=tfplan
      
      - name: Terraform Apply
        if: github.event_name == 'push'
        run: terraform apply -auto-approve tfplan
```

### Reusable Workflows

**.github/workflows/terraform-reusable.yml:**

```yaml
name: 'Reusable Terraform Workflow'

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      terraform-version:
        required: false
        type: string
        default: '1.6.0'
      working-directory:
        required: false
        type: string
        default: '.'
    secrets:
      aws-access-key-id:
        required: true
      aws-secret-access-key:
        required: true

jobs:
  terraform:
    name: 'Terraform ${{ inputs.environment }}'
    runs-on: ubuntu-latest
    
    defaults:
      run:
        working-directory: ${{ inputs.working-directory }}
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ inputs.terraform-version }}
      
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.aws-access-key-id }}
          aws-secret-access-key: ${{ secrets.aws-secret-access-key }}
          aws-region: us-east-1
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Plan
        run: terraform plan -var="environment=${{ inputs.environment }}"
      
      - name: Terraform Apply
        if: github.event_name == 'push'
        run: terraform apply -auto-approve -var="environment=${{ inputs.environment }}"
```

**Usage:**

```yaml
name: 'Deploy Infrastructure'

on:
  push:
    branches: [main, develop]

jobs:
  deploy-dev:
    uses: ./.github/workflows/terraform-reusable.yml
    with:
      environment: development
    secrets:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
  
  deploy-prod:
    needs: deploy-dev
    if: github.ref == 'refs/heads/main'
    uses: ./.github/workflows/terraform-reusable.yml
    with:
      environment: production
    secrets:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## 🦊 GitLab CI/CD Integration

### Basic Pipeline

**.gitlab-ci.yml:**

```yaml
image:
  name: hashicorp/terraform:1.6
  entrypoint: [""]

variables:
  TF_ROOT: ${CI_PROJECT_DIR}
  TF_STATE_NAME: default
  AWS_DEFAULT_REGION: us-east-1

cache:
  paths:
    - ${TF_ROOT}/.terraform

stages:
  - validate
  - plan
  - apply
  - destroy

before_script:
  - cd ${TF_ROOT}
  - terraform --version
  - terraform init

validate:
  stage: validate
  script:
    - terraform fmt -check -recursive
    - terraform validate
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

security-scan:
  stage: validate
  image: aquasec/tfsec:latest
  script:
    - tfsec . --format json > tfsec-report.json
  artifacts:
    reports:
      sast: tfsec-report.json
    paths:
      - tfsec-report.json
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

plan:
  stage: plan
  script:
    - terraform plan -out=tfplan
    - terraform show -json tfplan > plan.json
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
      - ${TF_ROOT}/plan.json
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

apply:
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
    - terraform output -json > outputs.json
  artifacts:
    paths:
      - ${TF_ROOT}/outputs.json
    expire_in: 1 month
  dependencies:
    - plan
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
  environment:
    name: production
    action: start

destroy:
  stage: destroy
  script:
    - terraform destroy -auto-approve
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
  environment:
    name: production
    action: stop
```

### Multi-Environment Pipeline

**.gitlab-ci.yml:**

```yaml
image:
  name: hashicorp/terraform:1.6
  entrypoint: [""]

variables:
  TF_ROOT: ${CI_PROJECT_DIR}
  AWS_DEFAULT_REGION: us-east-1

stages:
  - validate
  - plan-dev
  - apply-dev
  - plan-staging
  - apply-staging
  - plan-prod
  - apply-prod

.terraform-base:
  before_script:
    - cd ${TF_ROOT}
    - terraform init -backend-config="key=${CI_ENVIRONMENT_NAME}/terraform.tfstate"

.terraform-plan:
  extends: .terraform-base
  script:
    - terraform workspace select ${CI_ENVIRONMENT_NAME} || terraform workspace new ${CI_ENVIRONMENT_NAME}
    - terraform plan -var="environment=${CI_ENVIRONMENT_NAME}" -out=tfplan-${CI_ENVIRONMENT_NAME}
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan-${CI_ENVIRONMENT_NAME}
    expire_in: 1 week

.terraform-apply:
  extends: .terraform-base
  script:
    - terraform workspace select ${CI_ENVIRONMENT_NAME}
    - terraform apply -auto-approve tfplan-${CI_ENVIRONMENT_NAME}

validate:
  stage: validate
  before_script:
    - cd ${TF_ROOT}
    - terraform init -backend=false
  script:
    - terraform fmt -check -recursive
    - terraform validate

# Development Environment
plan:dev:
  extends: .terraform-plan
  stage: plan-dev
  environment:
    name: development
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

apply:dev:
  extends: .terraform-apply
  stage: apply-dev
  environment:
    name: development
  dependencies:
    - plan:dev
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      when: manual

# Staging Environment
plan:staging:
  extends: .terraform-plan
  stage: plan-staging
  environment:
    name: staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

apply:staging:
  extends: .terraform-apply
  stage: apply-staging
  environment:
    name: staging
  dependencies:
    - plan:staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual

# Production Environment
plan:prod:
  extends: .terraform-plan
  stage: plan-prod
  environment:
    name: production
  needs:
    - apply:staging
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

apply:prod:
  extends: .terraform-apply
  stage: apply-prod
  environment:
    name: production
  dependencies:
    - plan:prod
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## 🔧 Jenkins Integration

### Jenkinsfile

```groovy
pipeline {
    agent any
    
    parameters {
        choice(
            name: 'ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Terraform action to perform'
        )
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Environment to deploy'
        )
    }
    
    environment {
        TF_VERSION = '1.6.0'
        AWS_REGION = 'us-east-1'
        AWS_CREDENTIALS = credentials('aws-credentials')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup Terraform') {
            steps {
                sh '''
                    wget https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip
                    unzip -o terraform_${TF_VERSION}_linux_amd64.zip
                    chmod +x terraform
                    ./terraform version
                '''
            }
        }
        
        stage('Terraform Init') {
            steps {
                sh '''
                    ./terraform init \
                        -backend-config="bucket=terraform-state-${ENVIRONMENT}" \
                        -backend-config="key=terraform.tfstate" \
                        -backend-config="region=${AWS_REGION}"
                '''
            }
        }
        
        stage('Terraform Validate') {
            steps {
                sh './terraform fmt -check -recursive'
                sh './terraform validate'
            }
        }
        
        stage('Security Scan') {
            steps {
                sh '''
                    docker run --rm -v $(pwd):/src aquasec/tfsec /src
                '''
            }
        }
        
        stage('Terraform Plan') {
            when {
                expression { params.ACTION == 'plan' || params.ACTION == 'apply' }
            }
            steps {
                sh '''
                    ./terraform workspace select ${ENVIRONMENT} || ./terraform workspace new ${ENVIRONMENT}
                    ./terraform plan \
                        -var="environment=${ENVIRONMENT}" \
                        -out=tfplan
                '''
            }
        }
        
        stage('Approval') {
            when {
                expression { params.ACTION == 'apply' && params.ENVIRONMENT == 'prod' }
            }
            steps {
                input message: 'Apply Terraform changes to production?',
                      ok: 'Apply',
                      submitter: 'admin'
            }
        }
        
        stage('Terraform Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                sh './terraform apply -auto-approve tfplan'
            }
        }
        
        stage('Terraform Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                input message: "Destroy ${params.ENVIRONMENT} infrastructure?",
                      ok: 'Destroy'
                sh './terraform destroy -auto-approve -var="environment=${ENVIRONMENT}"'
            }
        }
        
        stage('Save Outputs') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                sh './terraform output -json > outputs.json'
                archiveArtifacts artifacts: 'outputs.json', fingerprint: true
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            echo 'Terraform pipeline completed successfully!'
        }
        failure {
            echo 'Terraform pipeline failed!'
        }
    }
}
```

### Shared Library

**vars/terraform.groovy:**

```groovy
def init(Map config = [:]) {
    sh """
        terraform init \
            -backend-config="bucket=${config.bucket}" \
            -backend-config="key=${config.key}" \
            -backend-config="region=${config.region}"
    """
}

def validate() {
    sh 'terraform fmt -check -recursive'
    sh 'terraform validate'
}

def plan(String environment) {
    sh """
        terraform workspace select ${environment} || terraform workspace new ${environment}
        terraform plan -var="environment=${environment}" -out=tfplan
    """
}

def apply() {
    sh 'terraform apply -auto-approve tfplan'
}

def destroy(String environment) {
    sh "terraform destroy -auto-approve -var='environment=${environment}'"
}
```

**Usage:**

```groovy
@Library('terraform-library') _

pipeline {
    agent any
    
    stages {
        stage('Init') {
            steps {
                script {
                    terraform.init(
                        bucket: 'my-terraform-state',
                        key: 'terraform.tfstate',
                        region: 'us-east-1'
                    )
                }
            }
        }
        
        stage('Validate') {
            steps {
                script {
                    terraform.validate()
                }
            }
        }
        
        stage('Plan') {
            steps {
                script {
                    terraform.plan('dev')
                }
            }
        }
        
        stage('Apply') {
            steps {
                script {
                    terraform.apply()
                }
            }
        }
    }
}
```

---

## ☁️ Terraform Cloud Integration

### Workspace Configuration

**main.tf:**

```hcl
terraform {
  cloud {
    organization = "my-organization"
    
    workspaces {
      name = "production"
    }
  }
  
  required_version = ">= 1.6.0"
}
```

### VCS-Driven Workflow

```hcl
terraform {
  cloud {
    organization = "my-organization"
    
    workspaces {
      tags = ["production", "aws"]
    }
  }
}
```

### CLI-Driven Workflow

```bash
# Login to Terraform Cloud
terraform login

# Initialize
terraform init

# Plan
terraform plan

# Apply
terraform apply
```

### API-Driven Workflow

```bash
# Create workspace
curl \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @payload.json \
  https://app.terraform.io/api/v2/organizations/my-org/workspaces

# Trigger run
curl \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @run-payload.json \
  https://app.terraform.io/api/v2/runs
```

---

## 🔒 Security Best Practices

### Secret Management

**GitHub Actions:**

```yaml
- name: Configure AWS Credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: us-east-1
```

**GitLab CI:**

```yaml
variables:
  AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
  AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
```

**Jenkins:**

```groovy
environment {
    AWS_CREDENTIALS = credentials('aws-credentials')
}
```

### OIDC Authentication

**GitHub Actions with OIDC:**

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS Credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
      role-session-name: GitHubActions
      aws-region: us-east-1
```

**AWS IAM Role Trust Policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:*"
        }
      }
    }
  ]
}
```

---

## 🧪 Automated Testing

### Terraform Test

**tests/main.tftest.hcl:**

```hcl
run "verify_vpc" {
  command = plan
  
  assert {
    condition     = aws_vpc.main.cidr_block == "10.0.0.0/16"
    error_message = "VPC CIDR must be 10.0.0.0/16"
  }
}

run "verify_subnets" {
  command = plan
  
  assert {
    condition     = length(aws_subnet.public) == 2
    error_message = "Must have exactly 2 public subnets"
  }
}
```

### Terratest

**test/terraform_test.go:**

```go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerraformInfrastructure(t *testing.T) {
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
            "environment": "test",
        },
    })
    
    defer terraform.Destroy(t, terraformOptions)
    
    terraform.InitAndApply(t, terraformOptions)
    
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcId)
}
```

---

## 📊 Monitoring and Notifications

### Slack Notifications

**GitHub Actions:**

```yaml
- name: Notify Slack
  if: always()
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    text: 'Terraform deployment ${{ job.status }}'
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### Email Notifications

**Jenkins:**

```groovy
post {
    failure {
        emailext (
            subject: "Terraform Pipeline Failed: ${env.JOB_NAME}",
            body: "Build failed. Check console output.",
            to: "team@example.com"
        )
    }
}
```

---

## 📚 Summary

**Key Concepts:**
- CI/CD pipeline design for infrastructure
- GitHub Actions, GitLab CI, Jenkins integration
- Terraform Cloud workflows
- Security and secret management
- Automated testing strategies
- Monitoring and notifications

**Best Practices:**
1. Always validate and test before apply
2. Use approval gates for production
3. Implement security scanning
4. Use OIDC for authentication
5. Store state remotely
6. Tag and version infrastructure
7. Monitor pipeline execution
8. Implement rollback strategies

**Next Steps:**
1. Complete hands-on labs in [LABS.md](./LABS.md)
2. Review interview questions in [INTERVIEW.md](./INTERVIEW.md)
3. Proceed to [Chapter 10](../chapter-10/) for production best practices

---

**🎉 Congratulations!** You now understand how to integrate Terraform with CI/CD pipelines!
