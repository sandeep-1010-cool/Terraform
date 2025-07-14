# Day 23: CI/CD Integration

## Learning Objectives

By the end of this lesson, you will be able to:

- Understand Continuous Integration concepts and their application to Terraform
- Implement automated deployment strategies
- Design effective pipeline strategies for infrastructure
- Apply integration best practices for Terraform CI/CD
- Configure advanced CI/CD features and integrations
- Implement testing and validation in CI/CD pipelines

## Core Concepts

### 1. Continuous Integration Overview

Continuous Integration (CI) for Terraform involves:
- **Automated Testing**: Validate Terraform configurations
- **Code Quality**: Format and lint Terraform code
- **Security Scanning**: Check for security vulnerabilities
- **Automated Deployment**: Apply changes to environments
- **Rollback Capabilities**: Revert changes when needed

### 2. CI/CD Pipeline Architecture

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Code      │───▶│   Build     │───▶│    Test     │───▶│   Deploy    │
│  Changes    │    │   & Lint    │    │  & Validate │    │  & Monitor  │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                                              │
                                                              ▼
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Rollback  │◀───│  Monitoring │◀───│   Alerts    │◀───│   Health    │
│   Strategy  │    │   & Logs    │    │  & Metrics  │    │   Checks    │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

## Practical Examples

### 1. Advanced GitHub Actions Pipeline

```yaml
# .github/workflows/terraform-advanced.yml
name: "Advanced Terraform CI/CD"

on:
  push:
    branches: [ main, develop, feature/* ]
    paths:
      - 'terraform/**'
      - '.github/workflows/terraform-advanced.yml'
  pull_request:
    branches: [ main, develop ]
    paths:
      - 'terraform/**'

env:
  TF_VERSION: "1.5.0"
  AWS_REGION: "us-west-2"
  TF_WORKSPACE: "default"

jobs:
  # Pre-flight checks
  pre-flight:
    name: "Pre-flight Checks"
    runs-on: ubuntu-latest
    outputs:
      should-deploy: ${{ steps.check.outputs.should-deploy }}
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3
      with:
        fetch-depth: 0

    - name: Check for Terraform changes
      id: check
      run: |
        if git diff --name-only ${{ github.event.before }} ${{ github.sha } | grep -E '\.tf$|\.tfvars$'; then
          echo "should-deploy=true" >> $GITHUB_OUTPUT
        else
          echo "should-deploy=false" >> $GITHUB_OUTPUT
        fi

  # Code quality checks
  code-quality:
    name: "Code Quality"
    runs-on: ubuntu-latest
    needs: pre-flight
    if: needs.pre-flight.outputs.should-deploy == 'true'
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: ${{ env.TF_VERSION }}

    - name: Terraform Format Check
      working-directory: terraform
      run: terraform fmt -check -recursive

    - name: Terraform Validate
      working-directory: terraform
      run: terraform validate

    - name: Run TFLint
      working-directory: terraform
      run: |
        curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
        tflint --init
        tflint

    - name: Run Checkov Security Scan
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

  # Infrastructure testing
  infrastructure-test:
    name: "Infrastructure Testing"
    runs-on: ubuntu-latest
    needs: [pre-flight, code-quality]
    if: needs.pre-flight.outputs.should-deploy == 'true'
    
    strategy:
      matrix:
        environment: [development, staging]
    
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

    - name: Terraform Plan
      working-directory: terraform
      run: |
        terraform plan \
          -var-file="environments/${{ matrix.environment }}.tfvars" \
          -out=tfplan
      env:
        TF_VAR_environment: ${{ matrix.environment }}

    - name: Run Terratest
      working-directory: terraform/test
      run: |
        go test -v -timeout 30m -run TestTerraformBasicExample
      env:
        AWS_REGION: ${{ env.AWS_REGION }}

    - name: Upload Test Results
      uses: actions/upload-artifact@v3
      with:
        name: test-results-${{ matrix.environment }}
        path: terraform/test/test-results/

  # Deployment
  deploy-development:
    name: "Deploy to Development"
    runs-on: ubuntu-latest
    needs: [pre-flight, code-quality, infrastructure-test]
    if: |
      needs.pre-flight.outputs.should-deploy == 'true' &&
      github.ref == 'refs/heads/develop'
    
    environment: development
    
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

    - name: Terraform Apply
      working-directory: terraform
      run: |
        terraform apply \
          -var-file="environments/development.tfvars" \
          -auto-approve
      env:
        TF_VAR_environment: development

    - name: Run Post-deployment Tests
      run: |
        # Run health checks
        ./scripts/health-check.sh development
        
        # Run smoke tests
        ./scripts/smoke-tests.sh development

  deploy-production:
    name: "Deploy to Production"
    runs-on: ubuntu-latest
    needs: [pre-flight, code-quality, infrastructure-test]
    if: |
      needs.pre-flight.outputs.should-deploy == 'true' &&
      github.ref == 'refs/heads/main'
    
    environment: production
    
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

    - name: Terraform Plan
      working-directory: terraform
      run: |
        terraform plan \
          -var-file="environments/production.tfvars" \
          -out=tfplan
      env:
        TF_VAR_environment: production

    - name: Terraform Apply
      working-directory: terraform
      run: terraform apply tfplan
      env:
        TF_VAR_environment: production

    - name: Run Post-deployment Tests
      run: |
        # Run health checks
        ./scripts/health-check.sh production
        
        # Run smoke tests
        ./scripts/smoke-tests.sh production

    - name: Notify Deployment Success
      if: success()
      run: |
        curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
          -H 'Content-type: application/json' \
          -d '{"text":"✅ Production deployment completed successfully!"}'

  # Rollback job
  rollback:
    name: "Rollback"
    runs-on: ubuntu-latest
    needs: [deploy-production]
    if: failure() && github.ref == 'refs/heads/main'
    
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

    - name: Rollback to Previous Version
      run: |
        # Implement rollback logic
        ./scripts/rollback.sh production

    - name: Notify Rollback
      run: |
        curl -X POST ${{ secrets.SLACK_WEBHOOK_URL }} \
          -H 'Content-type: application/json' \
          -d '{"text":"🔄 Production deployment rolled back due to failure"}'
```

### 2. Jenkins Pipeline for Terraform

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    environment {
        TF_VERSION = '1.5.0'
        AWS_REGION = 'us-west-2'
        TF_WORKSPACE = 'default'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Terraform Format') {
                    steps {
                        dir('terraform') {
                            sh 'terraform fmt -check -recursive'
                        }
                    }
                }
                
                stage('Terraform Validate') {
                    steps {
                        dir('terraform') {
                            sh 'terraform validate'
                        }
                    }
                }
                
                stage('Security Scan') {
                    steps {
                        dir('terraform') {
                            sh '''
                                curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
                                tflint --init
                                tflint
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Infrastructure Testing') {
            parallel {
                stage('Development Plan') {
                    steps {
                        dir('terraform') {
                            withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                                           string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                                sh '''
                                    terraform init
                                    terraform plan -var-file="environments/development.tfvars" -out=tfplan-dev
                                '''
                            }
                        }
                    }
                }
                
                stage('Staging Plan') {
                    steps {
                        dir('terraform') {
                            withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                                           string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                                sh '''
                                    terraform init
                                    terraform plan -var-file="environments/staging.tfvars" -out=tfplan-staging
                                '''
                            }
                        }
                    }
                }
            }
        }
        
        stage('Deploy to Development') {
            when {
                branch 'develop'
            }
            steps {
                dir('terraform') {
                    withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                                   string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh '''
                            terraform init
                            terraform apply tfplan-dev
                        '''
                    }
                }
            }
            post {
                always {
                    dir('terraform') {
                        archiveArtifacts artifacts: '*.log', allowEmptyArchive: true
                    }
                }
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                dir('terraform') {
                    withCredentials([string(credentialsId: 'aws-access-key', variable: 'AWS_ACCESS_KEY_ID'),
                                   string(credentialsId: 'aws-secret-key', variable: 'AWS_SECRET_ACCESS_KEY')]) {
                        sh '''
                            terraform init
                            terraform apply tfplan-staging
                        '''
                    }
                }
            }
            post {
                success {
                    echo 'Production deployment successful!'
                }
                failure {
                    echo 'Production deployment failed!'
                    // Implement rollback logic
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
```

### 3. GitLab CI/CD with Advanced Features

```yaml
# .gitlab-ci.yml
stages:
  - validate
  - test
  - plan
  - deploy
  - monitor

variables:
  TF_ROOT: "${CI_PROJECT_DIR}/terraform"
  TF_VERSION: "1.5.0"
  AWS_REGION: "us-west-2"

before_script:
  - cd ${TF_ROOT}
  - terraform --version
  - terraform init

# Code quality and validation
validate:
  stage: validate
  script:
    - terraform validate
    - terraform fmt -check -recursive
    - |
      curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
      tflint --init
      tflint
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# Security scanning
security-scan:
  stage: validate
  script:
    - |
      docker run --rm -v $(pwd):/src bridgecrew/checkov:latest \
        -d /src \
        --framework terraform \
        --output sarif \
        --output-file-path /src/checkov.sarif
  artifacts:
    reports:
      sarif: terraform/checkov.sarif
    expire_in: 1 week
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# Infrastructure testing
test:development:
  stage: test
  script:
    - terraform plan -var-file="environments/development.tfvars" -out=tfplan
    - |
      # Run Terratest
      cd test
      go test -v -timeout 30m -run TestTerraformBasicExample
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week
  environment:
    name: development
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

test:staging:
  stage: test
  script:
    - terraform plan -var-file="environments/staging.tfvars" -out=tfplan
    - |
      # Run Terratest
      cd test
      go test -v -timeout 30m -run TestTerraformBasicExample
  artifacts:
    paths:
      - ${TF_ROOT}/tfplan
    expire_in: 1 week
  environment:
    name: staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# Planning
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

# Deployment
deploy:development:
  stage: deploy
  script:
    - terraform apply tfplan
    - |
      # Post-deployment health checks
      ./scripts/health-check.sh development
  dependencies:
    - test:development
  environment:
    name: development
  rules:
    - if: $CI_COMMIT_BRANCH == "develop"

deploy:staging:
  stage: deploy
  script:
    - terraform apply tfplan
    - |
      # Post-deployment health checks
      ./scripts/health-check.sh staging
  dependencies:
    - test:staging
  environment:
    name: staging
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

deploy:production:
  stage: deploy
  script:
    - terraform apply tfplan
    - |
      # Post-deployment health checks
      ./scripts/health-check.sh production
  dependencies:
    - plan:production
  environment:
    name: production
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: manual

# Monitoring
monitor:
  stage: monitor
  script:
    - |
      # Run monitoring checks
      ./scripts/monitor.sh
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
  when: always
```

### 4. Terraform Testing Framework

```go
// test/terraform_basic_test.go
package test

import (
    "testing"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestTerraformBasicExample(t *testing.T) {
    // Configure Terraform options
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
            "environment": "test",
        },
    })

    // Clean up resources after test
    defer terraform.Destroy(t, terraformOptions)

    // Deploy the infrastructure
    terraform.InitAndApply(t, terraformOptions)

    // Get outputs
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    subnetIds := terraform.OutputList(t, terraformOptions, "subnet_ids")

    // Assertions
    assert.NotEmpty(t, vpcId)
    assert.Len(t, subnetIds, 2)
}

func TestTerraformSecurityExample(t *testing.T) {
    terraformOptions := terraform.WithDefaultRetryableErrors(t, &terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
            "environment": "test",
        },
    })

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Test security group rules
    securityGroupId := terraform.Output(t, terraformOptions, "security_group_id")
    assert.NotEmpty(t, securityGroupId)
}
```

### 5. Advanced CI/CD Features

```yaml
# .github/workflows/advanced-features.yml
name: "Advanced CI/CD Features"

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # Automated testing with parallel execution
  parallel-testing:
    name: "Parallel Testing"
    runs-on: ubuntu-latest
    strategy:
      matrix:
        test-suite: [unit, integration, security, performance]
        environment: [development, staging]
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Run ${{ matrix.test-suite }} tests
      run: |
        ./scripts/run-${{ matrix.test-suite }}-tests.sh ${{ matrix.environment }}

  # Blue-green deployment
  blue-green-deployment:
    name: "Blue-Green Deployment"
    runs-on: ubuntu-latest
    needs: parallel-testing
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2

    - name: Deploy to Blue Environment
      run: |
        ./scripts/deploy-blue.sh

    - name: Run Health Checks
      run: |
        ./scripts/health-check.sh blue

    - name: Switch Traffic to Blue
      run: |
        ./scripts/switch-traffic.sh blue

    - name: Deploy to Green Environment
      run: |
        ./scripts/deploy-green.sh

    - name: Run Health Checks
      run: |
        ./scripts/health-check.sh green

    - name: Switch Traffic to Green
      run: |
        ./scripts/switch-traffic.sh green

  # Canary deployment
  canary-deployment:
    name: "Canary Deployment"
    runs-on: ubuntu-latest
    needs: parallel-testing
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Deploy Canary
      run: |
        ./scripts/deploy-canary.sh

    - name: Monitor Canary
      run: |
        ./scripts/monitor-canary.sh

    - name: Promote Canary
      if: success()
      run: |
        ./scripts/promote-canary.sh

  # Rollback automation
  auto-rollback:
    name: "Auto Rollback"
    runs-on: ubuntu-latest
    needs: [blue-green-deployment, canary-deployment]
    if: failure()
    
    steps:
    - name: Trigger Rollback
      run: |
        ./scripts/auto-rollback.sh
```

## Best Practices

### 1. Pipeline Security

```yaml
# Security best practices
security:
  - name: "Secret Scanning"
    uses: actions/checkout@v3
    with:
      token: ${{ secrets.GITHUB_TOKEN }}
  
  - name: "Dependency Scanning"
    uses: actions/dependency-review-action@v3
  
  - name: "Code Scanning"
    uses: github/codeql-action/init@v2
    with:
      languages: terraform
```

### 2. Environment Management

```hcl
# environments.tf
locals {
  environments = {
    development = {
      instance_type = "t3.micro"
      instance_count = 1
      enable_monitoring = false
    }
    staging = {
      instance_type = "t3.small"
      instance_count = 2
      enable_monitoring = true
    }
    production = {
      instance_type = "t3.medium"
      instance_count = 3
      enable_monitoring = true
    }
  }
}
```

### 3. Testing Strategy

```bash
#!/bin/bash
# scripts/test-strategy.sh

# Unit tests
echo "Running unit tests..."
terraform validate
terraform fmt -check

# Integration tests
echo "Running integration tests..."
terratest run --testdir test/

# Security tests
echo "Running security tests..."
checkov -d . --framework terraform

# Performance tests
echo "Running performance tests..."
terraform plan -var-file="environments/test.tfvars"
```

## Common Commands

### 1. CI/CD Pipeline Commands

```bash
# Initialize CI/CD pipeline
terraform init
terraform validate
terraform fmt -check

# Run tests
go test -v ./test/
terratest run --testdir test/

# Deploy with approval
terraform plan -out=tfplan
terraform apply tfplan

# Rollback
terraform apply -var-file="environments/previous.tfvars"
```

### 2. Advanced CI/CD Features

```bash
# Blue-green deployment
./scripts/deploy-blue.sh
./scripts/switch-traffic.sh blue
./scripts/deploy-green.sh
./scripts/switch-traffic.sh green

# Canary deployment
./scripts/deploy-canary.sh
./scripts/monitor-canary.sh
./scripts/promote-canary.sh

# Auto rollback
./scripts/auto-rollback.sh
```

## Troubleshooting

### 1. Common CI/CD Issues

**Issue**: Pipeline fails due to state conflicts
```bash
# Solution: Use workspace isolation
terraform workspace new $CI_ENVIRONMENT_NAME
terraform workspace select $CI_ENVIRONMENT_NAME
```

**Issue**: Tests timeout in CI/CD
```bash
# Solution: Optimize test execution
go test -v -timeout 10m -parallel 4 ./test/
```

**Issue**: Deployment fails due to resource dependencies
```bash
# Solution: Use depends_on and lifecycle rules
resource "aws_instance" "example" {
  depends_on = [aws_security_group.example]
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### 2. Monitoring and Alerting

```hcl
# monitoring.tf
resource "aws_cloudwatch_metric_alarm" "deployment_failure" {
  alarm_name          = "${var.environment}-deployment-failure"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "DeploymentFailures"
  namespace           = "AWS/Terraform"
  period              = "300"
  statistic           = "Sum"
  threshold           = "0"
  alarm_description   = "This metric monitors deployment failures"
  
  dimensions = {
    Environment = var.environment
  }
}
```

## Summary

In this lesson, you learned about:

1. **Continuous Integration Concepts**: Automated testing, validation, and deployment
2. **Advanced Pipeline Strategies**: Blue-green, canary, and automated rollback
3. **Testing Frameworks**: Terratest, security scanning, and performance testing
4. **Integration Best Practices**: Security, environment management, and monitoring
5. **Troubleshooting**: Common issues and solutions for CI/CD pipelines

## Next Steps

- Implement advanced CI/CD features in your projects
- Set up comprehensive testing strategies
- Explore monitoring and alerting for CI/CD
- Practice with different CI/CD platforms and tools

## Additional Resources

- [Terraform CI/CD Best Practices](https://www.terraform.io/docs/cloud/guides/recommended-practices/index.html)
- [Terratest Documentation](https://terratest.gruntwork.io/)
- [GitHub Actions for Terraform](https://github.com/hashicorp/setup-terraform)
- [Jenkins Pipeline Documentation](https://www.jenkins.io/doc/book/pipeline/) 