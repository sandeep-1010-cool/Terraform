# Day 19: Terraform Security Best Practices

## Learning Objectives
- Understand secrets management in Terraform
- Learn IAM and authentication best practices
- Master security scanning and vulnerability assessment
- Understand compliance considerations and frameworks
- Know how to implement security controls in Terraform

## Core Concepts

### Security in Infrastructure as Code

Security in Terraform involves:
- **Secrets Management**: Secure handling of sensitive data
- **Access Control**: IAM and authentication mechanisms
- **Security Scanning**: Vulnerability assessment and compliance
- **Audit Trail**: Tracking and monitoring changes
- **Compliance**: Meeting regulatory requirements

### Security Principles

1. **Least Privilege**: Grant minimum necessary permissions
2. **Defense in Depth**: Multiple layers of security
3. **Zero Trust**: Never trust, always verify
4. **Security by Design**: Build security into infrastructure
5. **Continuous Monitoring**: Ongoing security assessment

## Secrets Management

### What are Secrets?

Secrets are sensitive information that should be protected:
- **Passwords**: Database, application, service passwords
- **API Keys**: Cloud provider API keys
- **Certificates**: SSL/TLS certificates and private keys
- **Tokens**: Authentication tokens and session keys
- **Connection Strings**: Database connection strings

### Secrets Management Best Practices

#### 1. Never Store Secrets in Code

```hcl
# ❌ BAD: Hardcoded secrets
resource "aws_db_instance" "example" {
  password = "my-super-secret-password"  # Never do this!
}

# ✅ GOOD: Use variables and external sources
resource "aws_db_instance" "example" {
  password = var.db_password  # Use variables
}
```

#### 2. Use Environment Variables

```bash
# Set environment variables
export TF_VAR_db_password="my-secure-password"
export TF_VAR_api_key="my-api-key"

# Use in Terraform
terraform apply
```

#### 3. Use Terraform Cloud Variables

```hcl
# In Terraform Cloud, mark variables as sensitive
variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true  # Mark as sensitive
}

variable "api_key" {
  description = "API key for external service"
  type        = string
  sensitive   = true
}
```

#### 4. Use External Secret Managers

```hcl
# AWS Secrets Manager
data "aws_secretsmanager_secret" "db_password" {
  name = "prod/database/password"
}

data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = data.aws_secretsmanager_secret.db_password.id
}

resource "aws_db_instance" "example" {
  password = jsondecode(data.aws_secretsmanager_secret_version.db_password.secret_string)["password"]
}
```

### HashiCorp Vault Integration

```hcl
# Configure Vault provider
provider "vault" {
  address = "https://vault.example.com"
  token   = var.vault_token
}

# Read secrets from Vault
data "vault_generic_secret" "db_credentials" {
  path = "secret/database"
}

resource "aws_db_instance" "example" {
  username = data.vault_generic_secret.db_credentials.data["username"]
  password = data.vault_generic_secret.db_credentials.data["password"]
}
```

## IAM and Authentication Concepts

### AWS IAM Best Practices

#### 1. Use IAM Roles Instead of Access Keys

```hcl
# ❌ BAD: Hardcoded access keys
provider "aws" {
  region     = "us-west-2"
  access_key = "AKIAIOSFODNN7EXAMPLE"
  secret_key = "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY"
}

# ✅ GOOD: Use IAM roles or environment variables
provider "aws" {
  region = "us-west-2"
  # Credentials from environment or IAM role
}
```

#### 2. Implement Least Privilege

```hcl
# Create IAM policy with minimal permissions
resource "aws_iam_policy" "terraform_policy" {
  name = "terraform-policy"
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "ec2:DescribeInstances",
          "ec2:CreateInstance",
          "ec2:DeleteInstance"
        ]
        Resource = "*"
      }
    ]
  })
}
```

#### 3. Use IAM Roles for EC2 Instances

```hcl
# Create IAM role for EC2
resource "aws_iam_role" "ec2_role" {
  name = "ec2-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
}

# Attach policy to role
resource "aws_iam_role_policy_attachment" "ec2_policy" {
  role       = aws_iam_role.ec2_role.name
  policy_arn = aws_iam_policy.ec2_policy.arn
}

# Create instance profile
resource "aws_iam_instance_profile" "ec2_profile" {
  name = "ec2-profile"
  role = aws_iam_role.ec2_role.name
}

# Use instance profile
resource "aws_instance" "web" {
  ami                  = "ami-12345678"
  instance_type        = "t2.micro"
  iam_instance_profile = aws_iam_instance_profile.ec2_profile.name
}
```

### Azure Authentication

```hcl
# Use service principal authentication
provider "azurerm" {
  features {}
  
  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id
  client_id       = var.client_id
  client_secret   = var.client_secret
}

# Or use managed identity
provider "azurerm" {
  features {}
  use_msi = true
}
```

### Google Cloud Authentication

```hcl
# Use service account key
provider "google" {
  project = "my-project"
  region  = "us-central1"
  # credentials from GOOGLE_APPLICATION_CREDENTIALS environment variable
}

# Or use workload identity
provider "google" {
  project = "my-project"
  region  = "us-central1"
  # Uses default credentials
}
```

## Security Scanning

### Static Code Analysis

#### 1. Use tfsec for Security Scanning

```bash
# Install tfsec
go install github.com/aquasecurity/tfsec/cmd/tfsec@latest

# Scan Terraform code
tfsec .

# Scan with specific checks
tfsec . --include-checks AWS018,AWS019
```

#### 2. Use Checkov for Compliance

```bash
# Install Checkov
pip install checkov

# Scan Terraform code
checkov -d .

# Scan with specific frameworks
checkov -d . --framework terraform_plan
```

#### 3. Use Terrascan for Policy Scanning

```bash
# Install Terrascan
curl -L "$(curl -s https://api.github.com/repos/tenable/terrascan/releases/latest | grep -o -E "https://.+?_Linux_x86_64.tar.gz")" | tar -xz
chmod +x terrascan

# Scan Terraform code
terrascan scan -i terraform -f .
```

### Security Scanning in CI/CD

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Run tfsec
      uses: aquasecurity/tfsec-action@v0.0.1
      with:
        working_directory: terraform/
    
    - name: Run Checkov
      run: |
        pip install checkov
        checkov -d terraform/ --framework terraform --output cli --output junitxml --output-file-path ./
    
    - name: Upload Checkov results
      uses: actions/upload-artifact@v2
      with:
        name: checkov-results
        path: ./checkov-report.xml
```

### Security Scanning Examples

#### 1. Network Security Scanning

```hcl
# Check for open security groups
resource "aws_security_group" "web" {
  name_prefix = "web-sg"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # This will be flagged by security scanners
  }
}

# Better approach with restricted access
resource "aws_security_group" "web" {
  name_prefix = "web-sg"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]  # Restricted to internal network
  }
}
```

#### 2. Encryption Scanning

```hcl
# Check for unencrypted resources
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
  # Missing encryption configuration
}

# Better approach with encryption
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
}

resource "aws_s3_bucket_server_side_encryption_configuration" "example" {
  bucket = aws_s3_bucket.example.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

## Compliance Considerations

### Regulatory Frameworks

#### 1. SOC 2 Compliance

```hcl
# Implement logging and monitoring
resource "aws_cloudwatch_log_group" "application" {
  name              = "/aws/application"
  retention_in_days = 90  # Compliance requirement
}

# Enable CloudTrail for audit
resource "aws_cloudtrail" "main" {
  name                          = "main-trail"
  s3_bucket_name               = aws_s3_bucket.cloudtrail.id
  include_global_service_events = true
  is_multi_region_trail        = true
  enable_logging               = true
}
```

#### 2. GDPR Compliance

```hcl
# Data classification and tagging
resource "aws_s3_bucket" "personal_data" {
  bucket = "personal-data-bucket"
  
  tags = {
    DataClassification = "Personal"
    GDPRCompliant     = "true"
    DataRetention     = "7-years"
  }
}

# Encryption for data at rest
resource "aws_s3_bucket_server_side_encryption_configuration" "personal_data" {
  bucket = aws_s3_bucket.personal_data.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

#### 3. HIPAA Compliance

```hcl
# HIPAA-compliant database
resource "aws_db_instance" "hipaa_db" {
  identifier = "hipaa-database"
  
  # Enable encryption
  storage_encrypted = true
  
  # Enable backup
  backup_retention_period = 35
  backup_window          = "03:00-04:00"
  
  # Enable monitoring
  monitoring_interval = 60
  monitoring_role_arn = aws_iam_role.rds_monitoring.arn
  
  tags = {
    HIPAACompliant = "true"
    DataType       = "PHI"
  }
}
```

### Compliance Scanning

```hcl
# Use Sentinel policies for compliance
# compliance.sentinel
import "tfplan/v2" as tfplan

# Ensure encryption is enabled
encryption_required = rule {
  all tfplan.resource_changes as _, changes {
    changes.type is "aws_s3_bucket" implies
    all changes.after as _, resource {
      resource.server_side_encryption_configuration is not null
    }
  }
}

# Ensure backup is enabled
backup_required = rule {
  all tfplan.resource_changes as _, changes {
    changes.type is "aws_db_instance" implies
    all changes.after as _, resource {
      resource.backup_retention_period > 0
    }
  }
}
```

## Security Best Practices

### 1. State Security

```hcl
# Use encrypted backend
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-west-2"
    encrypt = true  # Always enable encryption
  }
}
```

### 2. Network Security

```hcl
# Use private subnets
resource "aws_subnet" "private" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = "us-west-2a"
  
  tags = {
    Name = "Private Subnet"
  }
}

# Use security groups with minimal access
resource "aws_security_group" "web" {
  name_prefix = "web-sg"
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/8"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### 3. Access Control

```hcl
# Use IAM roles with minimal permissions
resource "aws_iam_role" "lambda_role" {
  name = "lambda-execution-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

# Attach minimal policy
resource "aws_iam_role_policy" "lambda_policy" {
  name = "lambda-policy"
  role = aws_iam_role.lambda_role.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:*"
      }
    ]
  })
}
```

## Common Commands

### Security Scanning Commands

```bash
# Run tfsec
tfsec .

# Run Checkov
checkov -d .

# Run Terrascan
terrascan scan -i terraform -f .

# Validate Terraform
terraform validate

# Format code
terraform fmt -check
```

### Secrets Management Commands

```bash
# Set environment variables
export TF_VAR_db_password="secure-password"

# Use AWS CLI for secrets
aws secretsmanager get-secret-value --secret-id "prod/database/password"

# Use Vault CLI
vault kv get secret/database
```

## Troubleshooting

### Common Security Issues

1. **Exposed Secrets**
   ```bash
   # Check for hardcoded secrets
   grep -r "password\|secret\|key" .
   
   # Use git-secrets
   git secrets --scan
   ```

2. **Insufficient Permissions**
   ```bash
   # Check IAM permissions
   aws iam get-user
   aws iam list-attached-user-policies --user-name terraform
   ```

3. **Unencrypted Resources**
   ```bash
   # Check for unencrypted resources
   tfsec . --include-checks AWS017,AWS018
   ```

### Debugging Security Issues

```bash
# Enable debug logging
export TF_LOG=DEBUG
terraform plan

# Check for security issues
tfsec . --format json | jq '.[] | select(.severity == "ERROR")'
```

## Summary

Security in Terraform involves:
- **Secrets Management**: Use external secret managers and environment variables
- **IAM Best Practices**: Implement least privilege and use roles
- **Security Scanning**: Use tools like tfsec, Checkov, and Terrascan
- **Compliance**: Follow regulatory frameworks and implement controls

Key takeaways:
- Never hardcode secrets in Terraform code
- Use IAM roles instead of access keys
- Implement security scanning in CI/CD
- Follow compliance requirements
- Enable encryption and monitoring
- Use least privilege principle

Remember: Security should be built into your infrastructure from the start, not added as an afterthought. 