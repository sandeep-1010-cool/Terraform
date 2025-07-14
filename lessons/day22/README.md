# Day 22: GitOps with Terraform

## Learning Objectives

By the end of this lesson, you will be able to:

- Understand GitOps principles and their application to Terraform
- Implement Infrastructure as Code workflows
- Set up automated deployment strategies
- Apply version control best practices for Terraform
- Configure CI/CD pipelines for Terraform
- Implement GitOps workflows with different tools and platforms

## Core Concepts

### 1. GitOps Overview

GitOps is a methodology where Git is the single source of truth for declarative infrastructure and applications. Key principles:

- **Declarative**: Infrastructure is described as code
- **Versioned**: All changes are version controlled
- **Automated**: Changes are applied automatically
- **Observable**: Current state is always visible

### 2. GitOps Workflow with Terraform

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Developer │───▶│     Git     │───▶│    CI/CD    │───▶│  Terraform  │
│   Changes   │    │  Repository │    │   Pipeline  │    │   Apply     │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                                              │
                                                              ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│ Monitoring  │◀───│   State     │◀───│  Terraform  │◀───│  Terraform  │
│ & Alerting  │    │  Backend    │    │   Plan      │    │   Init      │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

## Practical Examples

### 1. Basic GitOps Workflow Setup

```hcl
# terraform.tf
terraform {
  required_version = ">= 1.0"

  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "environments/production/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be one of: development, staging, production."
  }
}

variable "project_name" {
  description = "Project name"
  type        = string
}

# locals.tf
locals {
  name_prefix = "${var.project_name}-${var.environment}"
  
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "terraform"
    GitOps      = "true"
  }
}
```

### 2. GitHub Actions CI/CD Pipeline

```yaml
# .github/workflows/terraform.yml
name: "Terraform CI/CD"

on:
  push:
    branches: [ main, develop ]
    paths:
      - 'terraform/**'
      - '.github/workflows/terraform.yml'
  pull_request:
    branches: [ main ]
    paths:
      - 'terraform/**'

env:
  TF_VERSION: "1.5.0"
  AWS_REGION: "us-west-2"

jobs:
  terraform-plan:
    name: "Terraform Plan"
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        environment: [development, staging, production]
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Terraform Init
      working-directory: terraform
      run: terraform init

    - name: Terraform Format Check
      working-directory: terraform
      run: terraform fmt -check

    - name: Terraform Validate
      working-directory: terraform
      run: terraform validate

    - name: Terraform Plan
      working-directory: terraform
      run: |
        terraform plan \
          -var-file="environments/${{ matrix.environment }}.tfvars" \
          -out=tfplan
      env:
        TF_VAR_environment: ${{ matrix.environment }}

    - name: Upload Terraform Plan
      uses: actions/upload-artifact@v3
      with:
        name: tfplan-${{ matrix.environment }}
        path: terraform/tfplan

  terraform-apply:
    name: "Terraform Apply"
    runs-on: ubuntu-latest
    needs: terraform-plan
    if: github.ref == 'refs/heads/main'
    
    strategy:
      matrix:
        environment: [development, staging, production]
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}

    - name: Download Terraform Plan
      uses: actions/download-artifact@v3
      with:
        name: tfplan-${{ matrix.environment }}
        path: terraform

    - name: Terraform Init
      working-directory: terraform
      run: terraform init

    - name: Terraform Apply
      working-directory: terraform
      run: terraform apply tfplan
      env:
        TF_VAR_environment: ${{ matrix.environment }}
```

### 3. GitLab CI/CD Pipeline

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - plan
  - apply

variables:
  TF_ROOT: "${CI_PROJECT_DIR}/terraform"
  TF_ADDRESS: "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/terraform/state/${CI_ENVIRONMENT_NAME}"

before_script:
  - cd ${TF_ROOT}
  - terraform --version
  - terraform init

validate:
  stage: validate
  script:
    - terraform validate
    - terraform fmt -check
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

plan:development:
  stage: plan
  script:
    - terraform plan -var-file="environments/development.tfvars" -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week
  environment:
    name: development
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

plan:staging:
  stage: plan
  script:
    - terraform plan -var-file="environments/staging.tfvars" -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week
  environment:
    name: staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

plan:production:
  stage: plan
  script:
    - terraform plan -var-file="environments/production.tfvars" -out=tfplan
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual

apply:development:
  stage: apply
  script:
    - terraform apply tfplan
  dependencies:
    - plan:development
  environment:
    name: development
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

apply:staging:
  stage: apply
  script:
    - terraform apply tfplan
  dependencies:
    - plan:staging
  environment:
    name: staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

apply:production:
  stage: apply
  script:
    - terraform apply tfplan
  dependencies:
    - plan:production
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual
```

### 4. ArgoCD GitOps Workflow

```yaml
# argocd/terraform-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: terraform-infrastructure
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/your-org/your-repo
    targetRevision: HEAD
    path: terraform
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
```

### 5. Terraform Cloud GitOps

```hcl
# terraform-cloud.tf
terraform {
  cloud {
    organization = "your-org"
    
    workspaces {
      name = "infrastructure-${var.environment}"
    }
  }
}

# variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
}

# terraform.tfvars (managed by Terraform Cloud)
# environment = "production"
```

### 6. Flux GitOps Controller

```yaml
# flux/terraform-infrastructure.yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: GitRepository
metadata:
  name: terraform-repo
  namespace: flux-system
spec:
  interval: 1m
  url: https://github.com/your-org/your-repo
  ref:
    branch: main
---
apiVersion: kustomize.toolkit.fluxcd.io/v1beta2
kind: Kustomization
metadata:
  name: terraform-infrastructure
  namespace: flux-system
spec:
  interval: 10m
  path: ./terraform
  prune: true
  sourceRef:
    kind: GitRepository
    name: terraform-repo
  targetNamespace: flux-system
```

## Best Practices

### 1. Branch Strategy

```bash
# Git flow for Terraform
main          # Production infrastructure
├── develop   # Development environment
├── staging   # Staging environment
└── feature/* # Feature branches for changes
```

### 2. Environment Configuration

```hcl
# environments/development.tfvars
environment = "development"
instance_type = "t3.micro"
instance_count = 1
enable_monitoring = false

# environments/staging.tfvars
environment = "staging"
instance_type = "t3.small"
instance_count = 2
enable_monitoring = true

# environments/production.tfvars
environment = "production"
instance_type = "t3.medium"
instance_count = 3
enable_monitoring = true
```

### 3. State Management

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "terraform-state-${var.project_name}"
    key            = "environments/${var.environment}/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

### 4. Security Best Practices

```yaml
# .github/workflows/security.yml
name: "Security Scan"

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Run Checkov
      uses: bridgecrewio/checkov-action@master
      with:
        directory: terraform/
        framework: terraform
        output_format: sarif
        output_file_path: checkov.sarif

    - name: Upload SARIF file
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: checkov.sarif
```

## Common Commands

### 1. GitOps Workflow Commands

```bash
# Initialize GitOps workflow
git init
git add .
git commit -m "Initial Terraform configuration"

# Create feature branch
git checkout -b feature/new-infrastructure
# Make changes
git add .
git commit -m "Add new infrastructure components"
git push origin feature/new-infrastructure

# Create pull request (via GitHub/GitLab UI)
# After approval and merge
git checkout main
git pull origin main
```

### 2. Terraform GitOps Commands

```bash
# Validate changes before commit
terraform validate
terraform fmt -check

# Plan changes
terraform plan -var-file="environments/development.tfvars"

# Apply changes (in CI/CD pipeline)
terraform apply -auto-approve

# Check state
terraform show
terraform state list
```

### 3. GitOps Tools

```bash
# Install Flux CLI
curl -s https://fluxcd.io/install.sh | sudo bash

# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Install Terraform Cloud CLI
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform
```

## Troubleshooting

### 1. Common GitOps Issues

**Issue**: State drift between Git and actual infrastructure
```bash
# Solution: Import resources back to state
terraform import aws_instance.example i-1234567890abcdef0

# Or refresh state
terraform refresh
```

**Issue**: Merge conflicts in Terraform files
```bash
# Solution: Use terraform fmt to standardize formatting
terraform fmt -recursive

# Resolve conflicts manually, then validate
terraform validate
```

**Issue**: CI/CD pipeline failures
```bash
# Check Terraform logs
terraform plan -detailed-exitcode

# Validate configuration
terraform validate
terraform fmt -check
```

### 2. GitOps Monitoring

```hcl
# monitoring.tf
resource "aws_cloudwatch_dashboard" "gitops" {
  dashboard_name = "${var.environment}-gitops-dashboard"

  dashboard_body = jsonencode({
    widgets = [
      {
        type   = "metric"
        x      = 0
        y      = 0
        width  = 12
        height = 6
        properties = {
          metrics = [
            ["AWS/Terraform", "DeploymentCount", "Environment", var.environment]
          ]
          period = 300
          stat   = "Sum"
          region = "us-west-2"
          title  = "Terraform Deployments"
        }
      }
    ]
  })
}
```

## Summary

In this lesson, you learned about:

1. **GitOps Principles**: Declarative, versioned, automated, and observable infrastructure
2. **CI/CD Pipelines**: GitHub Actions, GitLab CI, and other automation tools
3. **GitOps Tools**: ArgoCD, Flux, and Terraform Cloud integration
4. **Best Practices**: Branch strategy, environment configuration, and security
5. **Troubleshooting**: Common issues and monitoring strategies

## Next Steps

- Set up a GitOps workflow for your infrastructure
- Implement automated testing and validation
- Explore advanced GitOps patterns and tools
- Practice with different CI/CD platforms

## Additional Resources

- [GitOps Principles](https://www.gitops.tech/)
- [Terraform Cloud Documentation](https://www.terraform.io/docs/cloud/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Flux Documentation](https://fluxcd.io/docs/) 