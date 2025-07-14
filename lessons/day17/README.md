# Day 17: Terraform Backend Configuration

## Learning Objectives
- Understand backend types and their use cases
- Learn remote state management concepts
- Master backend configuration best practices
- Understand state locking and its importance
- Know how to configure and migrate between backends

## Core Concepts

### What are Backends?

Backends determine where Terraform stores its state file. They control:
- **State Storage**: Where the state file is stored
- **State Locking**: Preventing concurrent modifications
- **State Encryption**: Securing sensitive data
- **Team Collaboration**: Enabling multiple users to work together

### Backend Types

#### 1. Local Backend (Default)
Stores state file locally on your machine:

```hcl
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}
```

#### 2. Remote Backends
Store state in remote systems for team collaboration:

**AWS S3 Backend:**
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-west-2"
  }
}
```

**Azure Storage Backend:**
```hcl
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                 = "prod.terraform.tfstate"
  }
}
```

**Google Cloud Storage Backend:**
```hcl
terraform {
  backend "gcs" {
    bucket = "terraform-state-bucket"
    prefix = "prod/terraform.tfstate"
  }
}
```

## Remote State Management

### Benefits of Remote State

1. **Team Collaboration**
   - Multiple team members can work on the same infrastructure
   - State is shared and synchronized
   - Prevents conflicts and overwrites

2. **State Locking**
   - Prevents concurrent modifications
   - Ensures data consistency
   - Provides audit trail

3. **Security**
   - State encryption at rest
   - Access control and permissions
   - Audit logging

4. **Disaster Recovery**
   - State backup and versioning
   - Geographic redundancy
   - Point-in-time recovery

### State Locking Concepts

State locking prevents multiple users from modifying the same infrastructure simultaneously:

```hcl
# Example: AWS S3 with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

## Practical Examples

### Example 1: AWS S3 Backend Configuration

```hcl
# backend.tf
terraform {
  required_version = ">= 1.0"
  
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "prod/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"
    encrypt        = true
    
    # Optional: Use different keys for different environments
    # key = "dev/terraform.tfstate"
    # key = "staging/terraform.tfstate"
    # key = "prod/terraform.tfstate"
  }
}

# main.tf
provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "Web Server"
  }
}
```

### Example 2: Azure Storage Backend

```hcl
# backend.tf
terraform {
  backend "azurerm" {
    resource_group_name  = "terraform-state-rg"
    storage_account_name = "tfstate12345"
    container_name       = "tfstate"
    key                 = "prod.terraform.tfstate"
    use_azuread_auth    = true
  }
}

# main.tf
provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "example" {
  name     = "example-resources"
  location = "West US 2"
}
```

### Example 3: Google Cloud Storage Backend

```hcl
# backend.tf
terraform {
  backend "gcs" {
    bucket = "terraform-state-bucket"
    prefix = "prod/terraform.tfstate"
  }
}

# main.tf
provider "google" {
  project = "my-project-id"
  region  = "us-central1"
}

resource "google_compute_instance" "web" {
  name         = "web-instance"
  machine_type = "e2-micro"
  zone         = "us-central1-a"
}
```

### Example 4: HashiCorp Consul Backend

```hcl
# backend.tf
terraform {
  backend "consul" {
    address = "demo.consul.io"
    scheme  = "https"
    path    = "terraform/state"
  }
}

# main.tf
provider "aws" {
  region = "us-west-2"
}

resource "aws_s3_bucket" "example" {
  bucket = "my-unique-bucket-name"
}
```

## Backend Configuration Best Practices

### 1. Environment Separation
```hcl
# Use different state files for different environments
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "dev/terraform.tfstate"    # Development
    # key    = "staging/terraform.tfstate" # Staging
    # key    = "prod/terraform.tfstate"    # Production
    region = "us-west-2"
  }
}
```

### 2. State Encryption
```hcl
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-west-2"
    encrypt = true  # Always enable encryption
  }
}
```

### 3. State Locking
```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/terraform.tfstate"
    region         = "us-west-2"
    dynamodb_table = "terraform-locks"  # Enable state locking
    encrypt        = true
  }
}
```

### 4. Workspace Integration
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "terraform.tfstate"  # Workspaces will append workspace name
    region = "us-west-2"
  }
}

# Use workspaces for environment separation
# terraform workspace new dev
# terraform workspace new staging
# terraform workspace new prod
```

## Common Commands

### Backend Initialization
```bash
# Initialize backend
terraform init

# Initialize with backend configuration
terraform init -backend-config="bucket=my-terraform-state"
terraform init -backend-config="key=prod/terraform.tfstate"
terraform init -backend-config="region=us-west-2"

# Migrate from local to remote backend
terraform init -migrate-state

# Force migration (use with caution)
terraform init -force-copy
```

### Backend Configuration
```bash
# Configure backend interactively
terraform init

# Configure backend with partial configuration
terraform init -backend-config="backend.hcl"

# Reconfigure backend
terraform init -reconfigure
```

### State Management
```bash
# List workspaces
terraform workspace list

# Create new workspace
terraform workspace new dev

# Switch workspace
terraform workspace select prod

# Show current workspace
terraform workspace show
```

## Advanced Topics

### 1. Partial Backend Configuration
```hcl
# backend.hcl
bucket = "my-terraform-state"
key    = "prod/terraform.tfstate"
region = "us-west-2"
```

```bash
# Initialize with partial configuration
terraform init -backend-config="backend.hcl"
```

### 2. Backend Migration
```hcl
# From local to S3
terraform {
  backend "local" {
    path = "terraform.tfstate"
  }
}

# To S3 backend
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-west-2"
  }
}
```

### 3. Multi-Environment Setup
```hcl
# environments/dev/backend.tf
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "dev/terraform.tfstate"
    region = "us-west-2"
  }
}

# environments/prod/backend.tf
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "prod/terraform.tfstate"
    region = "us-west-2"
  }
}
```

### 4. Backend with Workspaces
```hcl
terraform {
  backend "s3" {
    bucket = "my-terraform-state"
    key    = "terraform.tfstate"  # Workspace name will be appended
    region = "us-west-2"
  }
}

# State files will be:
# terraform.tfstate/env:dev
# terraform.tfstate/env:staging
# terraform.tfstate/env:prod
```

## Troubleshooting

### Common Issues

1. **Backend Configuration Errors**
   ```bash
   # Error: Failed to get existing workspaces
   # Solution: Check backend permissions and configuration
   terraform init -reconfigure
   ```

2. **State Locking Issues**
   ```bash
   # Error: Error acquiring the state lock
   # Solution: Check if another operation is running
   # Or force unlock (use with caution)
   terraform force-unlock <lock-id>
   ```

3. **Authentication Issues**
   ```bash
   # Error: No valid credential sources found
   # Solution: Configure AWS credentials
   aws configure
   # Or use environment variables
   export AWS_ACCESS_KEY_ID="your-access-key"
   export AWS_SECRET_ACCESS_KEY="your-secret-key"
   ```

### Debugging Tips

```bash
# Enable debug logging
export TF_LOG=DEBUG
terraform plan

# Check backend configuration
terraform show

# Validate backend configuration
terraform validate
```

## Security Best Practices

### 1. Access Control
- Use IAM roles and policies for S3 access
- Implement least privilege principle
- Enable MFA for state access

### 2. Encryption
```hcl
terraform {
  backend "s3" {
    bucket  = "my-terraform-state"
    key     = "prod/terraform.tfstate"
    region  = "us-west-2"
    encrypt = true  # Server-side encryption
  }
}
```

### 3. Versioning and Backup
```hcl
# Enable versioning on S3 bucket
resource "aws_s3_bucket_versioning" "state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

## Summary

Backend configuration is crucial for:
- **Team Collaboration**: Multiple users working on infrastructure
- **State Security**: Encrypted storage and access control
- **State Locking**: Preventing concurrent modifications
- **Disaster Recovery**: Backup and versioning capabilities

Key takeaways:
- Choose the right backend for your use case
- Always enable encryption and state locking
- Use separate state files for different environments
- Follow security best practices
- Test backend migrations thoroughly

Remember: Proper backend configuration is essential for production Terraform deployments and team collaboration. 