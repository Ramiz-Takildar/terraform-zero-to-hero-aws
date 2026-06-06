# Chapter 9: Interview Questions - CI/CD Integration with Terraform

## 📋 Overview

This document contains interview questions covering CI/CD integration with Terraform, including GitHub Actions, GitLab CI, Jenkins, and automated testing strategies.

**Topics Covered:**
- CI/CD fundamentals for infrastructure
- GitHub Actions integration
- GitLab CI/CD pipelines
- Jenkins automation
- Terraform Cloud
- Security and secrets management
- Automated testing
- Best practices

---

## 🔄 CI/CD Fundamentals

### Q1: What is Infrastructure as Code CI/CD and why is it important?

**Answer:**

Infrastructure as Code (IaC) CI/CD applies continuous integration and continuous deployment practices to infrastructure management.

**Key Components:**

1. **Continuous Integration (CI):**
   - Validate infrastructure code
   - Run security scans
   - Execute automated tests
   - Generate deployment plans

2. **Continuous Deployment (CD):**
   - Automated infrastructure deployment
   - Environment promotion
   - Rollback capabilities
   - Audit trail

**Benefits:**

| Benefit | Description |
|---------|-------------|
| **Consistency** | Same deployment process across environments |
| **Speed** | Automated deployments reduce manual effort |
| **Quality** | Automated testing catches issues early |
| **Collaboration** | Team-based infrastructure management |
| **Auditability** | Complete history of changes |
| **Rollback** | Easy reversion to previous states |

**Example Workflow:**

```
Developer → Git Push → CI Pipeline → Tests → Plan → Approval → Apply → Deploy
```

**Why It's Important:**

1. **Reduces Human Error:** Automation eliminates manual mistakes
2. **Faster Deployments:** Automated processes are faster than manual
3. **Better Testing:** Automated tests run consistently
4. **Improved Security:** Security scans in every deployment
5. **Compliance:** Audit trail for regulatory requirements
6. **Scalability:** Handle multiple environments easily

---

### Q2: What are the key stages in a Terraform CI/CD pipeline?

**Answer:**

**Standard Pipeline Stages:**

```yaml
1. Validation Stage
   ├── Format Check (terraform fmt)
   ├── Syntax Validation (terraform validate)
   └── Linting (tflint)

2. Security Stage
   ├── Security Scanning (tfsec, checkov)
   ├── Policy Checks (OPA, Sentinel)
   └── Compliance Validation

3. Plan Stage
   ├── Terraform Init
   ├── Terraform Plan
   ├── Cost Estimation
   └── Plan Review/Comment

4. Approval Stage
   ├── Manual Review
   ├── Automated Checks
   └── Approval Gate

5. Apply Stage
   ├── Terraform Apply
   ├── Output Collection
   └── Notification

6. Testing Stage
   ├── Integration Tests
   ├── Smoke Tests
   └── Validation Tests
```

**Detailed Example:**

```yaml
# GitHub Actions Example
name: Terraform Pipeline

on: [push, pull_request]

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Format
        run: terraform fmt -check
      - name: Terraform Validate
        run: terraform validate

  security:
    needs: validate
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tfsec
        uses: aquasecurity/tfsec-action@v1.0.0

  plan:
    needs: security
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Plan
        run: terraform plan -out=tfplan

  apply:
    needs: plan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Apply
        run: terraform apply -auto-approve tfplan
```

**Best Practices:**

1. **Fail Fast:** Validate early to catch issues quickly
2. **Parallel Execution:** Run independent stages in parallel
3. **Caching:** Cache Terraform plugins and modules
4. **Artifacts:** Save plans and outputs
5. **Notifications:** Alert on failures
6. **Approval Gates:** Require manual approval for production

---

## 🐙 GitHub Actions

### Q3: How do you implement a Terraform workflow in GitHub Actions?

**Answer:**

**Basic Workflow Structure:**

```yaml
name: 'Terraform'

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  TF_VERSION: '1.6.0'

jobs:
  terraform:
    name: 'Terraform'
    runs-on: ubuntu-latest
    
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
      
      - name: Terraform Init
        run: terraform init
      
      - name: Terraform Format
        run: terraform fmt -check
      
      - name: Terraform Validate
        run: terraform validate
      
      - name: Terraform Plan
        id: plan
        run: terraform plan -no-color
        continue-on-error: true
      
      - name: Comment PR
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: 'Terraform Plan: ${{ steps.plan.outputs.stdout }}'
            })
      
      - name: Terraform Apply
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
        run: terraform apply -auto-approve
```

**Key Features:**

1. **Triggers:** Push and pull request events
2. **Permissions:** Granular access control
3. **Actions:** Reusable workflow components
4. **Secrets:** Secure credential management
5. **Environments:** Deployment protection rules
6. **Artifacts:** Save outputs and plans

**Advanced Patterns:**

```yaml
# Reusable Workflow
name: 'Reusable Terraform'

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
    secrets:
      aws_access_key_id:
        required: true
      aws_secret_access_key:
        required: true

jobs:
  terraform:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    steps:
      - uses: actions/checkout@v4
      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.aws_access_key_id }}
          aws-secret-access-key: ${{ secrets.aws_secret_access_key }}
          aws-region: us-east-1
      - name: Terraform Apply
        run: terraform apply -auto-approve
```

---

### Q4: How do you handle secrets in GitHub Actions for Terraform?

**Answer:**

**Methods for Secret Management:**

**1. GitHub Secrets:**

```yaml
steps:
  - name: Configure AWS Credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: us-east-1
```

**2. OIDC (Recommended):**

```yaml
permissions:
  id-token: write
  contents: read

steps:
  - name: Configure AWS Credentials
    uses: aws-actions/configure-aws-credentials@v4
    with:
      role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
      aws-region: us-east-1
```

**AWS IAM Trust Policy for OIDC:**

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
          "token.actions.githubusercontent.com:sub": "repo:org/repo:*"
        }
      }
    }
  ]
}
```

**3. Environment Secrets:**

```yaml
jobs:
  deploy:
    environment: production
    steps:
      - name: Use Environment Secret
        run: echo ${{ secrets.PROD_API_KEY }}
```

**4. HashiCorp Vault:**

```yaml
steps:
  - name: Import Secrets
    uses: hashicorp/vault-action@v2
    with:
      url: https://vault.example.com
      token: ${{ secrets.VAULT_TOKEN }}
      secrets: |
        secret/data/aws access_key | AWS_ACCESS_KEY_ID ;
        secret/data/aws secret_key | AWS_SECRET_ACCESS_KEY
```

**Best Practices:**

1. **Use OIDC:** Avoid long-lived credentials
2. **Least Privilege:** Minimal required permissions
3. **Environment Protection:** Use environment secrets for production
4. **Rotation:** Regularly rotate credentials
5. **Audit:** Monitor secret usage
6. **Never Commit:** Never commit secrets to repository

---

## 🦊 GitLab CI/CD

### Q5: How does GitLab CI/CD differ from GitHub Actions for Terraform?

**Answer:**

**Key Differences:**

| Feature | GitHub Actions | GitLab CI/CD |
|---------|---------------|--------------|
| **Configuration** | `.github/workflows/*.yml` | `.gitlab-ci.yml` |
| **Runners** | GitHub-hosted or self-hosted | GitLab-hosted or self-hosted |
| **Caching** | actions/cache | Built-in cache |
| **Artifacts** | actions/upload-artifact | Built-in artifacts |
| **Environments** | Environment protection rules | Environment deployments |
| **Variables** | Secrets and variables | CI/CD variables |
| **Approval** | Environment protection | Manual jobs |

**GitLab CI/CD Example:**

```yaml
image:
  name: hashicorp/terraform:1.6
  entrypoint: [""]

variables:
  TF_ROOT: ${CI_PROJECT_DIR}

cache:
  paths:
    - ${TF_ROOT}/.terraform

stages:
  - validate
  - plan
  - apply

before_script:
  - cd ${TF_ROOT}
  - terraform init

validate:
  stage: validate
  script:
    - terraform fmt -check
    - terraform validate

plan:
  stage: plan
  script:
    - terraform plan -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week

apply:
  stage: apply
  script:
    - terraform apply -auto-approve tfplan
  dependencies:
    - plan
  when: manual
  only:
    - main
```

**GitLab Advantages:**

1. **Built-in Features:** Native caching, artifacts, environments
2. **Auto DevOps:** Automatic pipeline generation
3. **Review Apps:** Temporary environments for testing
4. **Protected Variables:** Environment-specific variables
5. **Deployment Boards:** Visual deployment tracking

**GitHub Actions Advantages:**

1. **Marketplace:** Large ecosystem of actions
2. **Matrix Builds:** Easy parallel execution
3. **Reusable Workflows:** Share workflows across repos
4. **OIDC Support:** Better cloud provider integration
5. **GitHub Integration:** Tight integration with GitHub features

---

### Q6: How do you implement multi-environment deployments in GitLab CI?

**Answer:**

**Multi-Environment Pipeline:**

```yaml
image:
  name: hashicorp/terraform:1.6
  entrypoint: [""]

variables:
  TF_ROOT: ${CI_PROJECT_DIR}

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
    - terraform plan -var="environment=${CI_ENVIRONMENT_NAME}" -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week

.terraform-apply:
  extends: .terraform-base
  script:
    - terraform apply -auto-approve tfplan

# Development
plan:dev:
  extends: .terraform-plan
  stage: plan-dev
  environment:
    name: development
  only:
    - develop

apply:dev:
  extends: .terraform-apply
  stage: apply-dev
  environment:
    name: development
  dependencies:
    - plan:dev
  when: manual
  only:
    - develop

# Staging
plan:staging:
  extends: .terraform-plan
  stage: plan-staging
  environment:
    name: staging
  only:
    - main

apply:staging:
  extends: .terraform-apply
  stage: apply-staging
  environment:
    name: staging
  dependencies:
    - plan:staging
  when: manual
  only:
    - main

# Production
plan:prod:
  extends: .terraform-plan
  stage: plan-prod
  environment:
    name: production
  needs:
    - apply:staging
  only:
    - main

apply:prod:
  extends: .terraform-apply
  stage: apply-prod
  environment:
    name: production
  dependencies:
    - plan:prod
  when: manual
  only:
    - main
```

**Key Features:**

1. **Template Jobs:** `.terraform-base` for reusability
2. **Environment Variables:** `CI_ENVIRONMENT_NAME` for dynamic config
3. **Dependencies:** Control job execution order
4. **Manual Gates:** `when: manual` for approvals
5. **Branch Rules:** `only` for branch-specific deployments
6. **Needs:** Explicit dependencies between stages

---

## 🔧 Jenkins

### Q7: How do you integrate Terraform with Jenkins?

**Answer:**

**Jenkinsfile Example:**

```groovy
pipeline {
    agent any
    
    parameters {
        choice(
            name: 'ACTION',
            choices: ['plan', 'apply', 'destroy'],
            description: 'Terraform action'
        )
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Environment'
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
        
        stage('Setup') {
            steps {
                sh '''
                    wget https://releases.hashicorp.com/terraform/${TF_VERSION}/terraform_${TF_VERSION}_linux_amd64.zip
                    unzip -o terraform_${TF_VERSION}_linux_amd64.zip
                    chmod +x terraform
                '''
            }
        }
        
        stage('Init') {
            steps {
                sh '''
                    ./terraform init \
                        -backend-config="bucket=terraform-state-${ENVIRONMENT}" \
                        -backend-config="key=terraform.tfstate"
                '''
            }
        }
        
        stage('Validate') {
            steps {
                sh './terraform fmt -check'
                sh './terraform validate'
            }
        }
        
        stage('Plan') {
            when {
                expression { params.ACTION == 'plan' || params.ACTION == 'apply' }
            }
            steps {
                sh './terraform plan -var="environment=${ENVIRONMENT}" -out=tfplan'
            }
        }
        
        stage('Approval') {
            when {
                expression { params.ACTION == 'apply' && params.ENVIRONMENT == 'prod' }
            }
            steps {
                input message: 'Apply to production?',
                      ok: 'Apply',
                      submitter: 'admin'
            }
        }
        
        stage('Apply') {
            when {
                expression { params.ACTION == 'apply' }
            }
            steps {
                sh './terraform apply -auto-approve tfplan'
            }
        }
        
        stage('Destroy') {
            when {
                expression { params.ACTION == 'destroy' }
            }
            steps {
                input message: 'Destroy infrastructure?'
                sh './terraform destroy -auto-approve -var="environment=${ENVIRONMENT}"'
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            emailext (
                subject: "Terraform ${params.ACTION} succeeded",
                body: "Environment: ${params.ENVIRONMENT}",
                to: "team@example.com"
            )
        }
        failure {
            emailext (
                subject: "Terraform ${params.ACTION} failed",
                body: "Check console output",
                to: "team@example.com"
            )
        }
    }
}
```

**Jenkins Shared Library:**

```groovy
// vars/terraform.groovy
def init(Map config) {
    sh """
        terraform init \
            -backend-config="bucket=${config.bucket}" \
            -backend-config="key=${config.key}"
    """
}

def plan(String environment) {
    sh "terraform plan -var='environment=${environment}' -out=tfplan"
}

def apply() {
    sh "terraform apply -auto-approve tfplan"
}

// Usage in Jenkinsfile
@Library('terraform-library') _

pipeline {
    agent any
    stages {
        stage('Init') {
            steps {
                script {
                    terraform.init(bucket: 'my-state', key: 'terraform.tfstate')
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

## ☁️ Terraform Cloud

### Q8: What are the benefits of using Terraform Cloud for CI/CD?

**Answer:**

**Key Benefits:**

1. **Remote State Management:**
   - Secure state storage
   - State locking
   - State versioning
   - Team collaboration

2. **VCS Integration:**
   - Automatic runs on commits
   - Plan on pull requests
   - Apply on merge

3. **Private Registry:**
   - Module versioning
   - Module sharing
   - Access control

4. **Policy as Code:**
   - Sentinel policies
   - Cost estimation
   - Compliance checks

5. **Team Management:**
   - Role-based access
   - Workspace permissions
   - Audit logs

**Configuration:**

```hcl
terraform {
  cloud {
    organization = "my-org"
    
    workspaces {
      name = "production"
    }
  }
}
```

**VCS-Driven Workflow:**

```
Git Push → Terraform Cloud → Plan → Review → Apply
```

**API-Driven Workflow:**

```bash
# Create run
curl \
  --header "Authorization: Bearer $TOKEN" \
  --header "Content-Type: application/vnd.api+json" \
  --request POST \
  --data @payload.json \
  https://app.terraform.io/api/v2/runs
```

**Comparison:**

| Feature | Self-Managed | Terraform Cloud |
|---------|--------------|-----------------|
| **State Storage** | Manual setup | Built-in |
| **Locking** | DynamoDB/etc | Built-in |
| **UI** | None | Web interface |
| **Cost** | Infrastructure | Subscription |
| **Policies** | Manual | Sentinel |
| **Collaboration** | Manual | Built-in |

---

## 🔒 Security

### Q9: What security best practices should you follow in Terraform CI/CD?

**Answer:**

**1. Secret Management:**

```yaml
# Use OIDC instead of static credentials
- name: Configure AWS
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubRole
    aws-region: us-east-1
```

**2. Least Privilege:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:CreateTags",
        "ec2:RunInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

**3. Security Scanning:**

```yaml
- name: Run tfsec
  uses: aquasecurity/tfsec-action@v1.0.0

- name: Run Checkov
  uses: bridgecrewio/checkov-action@master

- name: Run Terrascan
  uses: tenable/terrascan-action@main
```

**4. State Encryption:**

```hcl
terraform {
  backend "s3" {
    bucket  = "terraform-state"
    key     = "terraform.tfstate"
    encrypt = true
    kms_key_id = "arn:aws:kms:us-east-1:123456789012:key/12345678"
  }
}
```

**5. Approval Gates:**

```yaml
environment:
  name: production
  # Requires manual approval
```

**6. Audit Logging:**

```yaml
- name: Log Deployment
  run: |
    echo "Deployed by: ${{ github.actor }}"
    echo "Commit: ${{ github.sha }}"
    echo "Timestamp: $(date)"
```

**7. Network Security:**

```yaml
# Use private runners for sensitive workloads
runs-on: self-hosted
```

**8. Code Review:**

```yaml
# Require PR reviews
on:
  pull_request:
    branches: [main]
```

---

## 🧪 Testing

### Q10: How do you implement automated testing in Terraform CI/CD?

**Answer:**

**Testing Levels:**

**1. Static Analysis:**

```yaml
- name: Terraform Validate
  run: terraform validate

- name: Terraform Format
  run: terraform fmt -check

- name: TFLint
  run: tflint --recursive
```

**2. Security Testing:**

```yaml
- name: tfsec
  run: tfsec .

- name: Checkov
  run: checkov -d .

- name: Terrascan
  run: terrascan scan
```

**3. Unit Testing (Terratest):**

```go
func TestTerraformVPC(t *testing.T) {
    terraformOptions := &terraform.Options{
        TerraformDir: "../",
    }
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    vpcID := terraform.Output(t, terraformOptions, "vpc_id")
    assert.NotEmpty(t, vpcID)
}
```

**4. Integration Testing:**

```yaml
- name: Deploy Test Environment
  run: terraform apply -auto-approve

- name: Run Integration Tests
  run: ./run-integration-tests.sh

- name: Cleanup
  run: terraform destroy -auto-approve
```

**5. Policy Testing:**

```hcl
# Sentinel policy
import "tfplan/v2" as tfplan

main = rule {
    all tfplan.resource_changes as _, rc {
        rc.type is "aws_instance" implies
        rc.change.after.instance_type in ["t2.micro", "t2.small"]
    }
}
```

**Complete Testing Pipeline:**

```yaml
jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Terraform Validate
        run: terraform validate

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Security Scan
        run: tfsec .

  unit-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
      - name: Run Terratest
        run: go test -v ./test/...

  integration-test:
    runs-on: ubuntu-latest
    needs: unit-test
    steps:
      - uses: actions/checkout@v4
      - name: Deploy
        run: terraform apply -auto-approve
      - name: Test
        run: ./integration-tests.sh
      - name: Cleanup
        run: terraform destroy -auto-approve
```

---

## 📊 Monitoring

### Q11: How do you monitor Terraform CI/CD pipelines?

**Answer:**

**Monitoring Strategies:**

**1. Pipeline Metrics:**

```yaml
- name: Track Metrics
  run: |
    echo "Duration: ${{ job.duration }}"
    echo "Status: ${{ job.status }}"
    echo "Commit: ${{ github.sha }}"
```

**2. Notifications:**

```yaml
# Slack
- name: Notify Slack
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}

# Email
- name: Send Email
  uses: dawidd6/action-send-mail@v3
  with:
    server_address: smtp.gmail.com
    subject: Pipeline ${{ job.status }}
    to: team@example.com
```

**3. Logging:**

```yaml
- name: Upload Logs
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: terraform-logs
    path: |
      *.log
      terraform.tfstate
```

**4. Dashboards:**

- GitHub Actions insights
- GitLab CI/CD analytics
- Jenkins build history
- Terraform Cloud runs

**5. Alerting:**

```yaml
post:
  failure:
    emailext (
      subject: "Pipeline Failed",
      body: "Check console output",
      to: "oncall@example.com"
    )
```

---

## 📚 Summary

**Key Topics Covered:**
- CI/CD fundamentals for infrastructure
- GitHub Actions workflows
- GitLab CI/CD pipelines
- Jenkins integration
- Terraform Cloud
- Security best practices
- Automated testing
- Monitoring and notifications

**Best Practices:**
1. Validate before plan
2. Run security scans
3. Use approval gates for production
4. Implement automated testing
5. Use OIDC for authentication
6. Monitor pipeline execution
7. Save artifacts and outputs
8. Implement proper secret management

**Next Steps:**
1. Practice labs in [LABS.md](./LABS.md)
2. Implement CI/CD in your projects
3. Proceed to [Chapter 10](../chapter-10/)

---

**💡 Pro Tips:**
- Start with simple pipelines, add complexity gradually
- Use reusable workflows/templates
- Test pipelines in non-production first
- Document pipeline behavior
- Monitor costs of CI/CD infrastructure
- Keep pipelines fast with caching
- Use matrix builds for parallel execution
- Implement proper error handling
