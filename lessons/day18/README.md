# Day 18: Terraform Cloud

## Learning Objectives
- Understand Terraform Cloud concepts and benefits
- Learn workspace management and organization
- Master team collaboration features
- Understand Policy as Code (Sentinel) introduction
- Know how to integrate with Terraform Cloud

## Core Concepts

### What is Terraform Cloud?

Terraform Cloud is HashiCorp's managed service that provides:
- **Remote State Management**: Centralized state storage
- **Team Collaboration**: Shared workspaces and permissions
- **Policy as Code**: Sentinel policies for governance
- **Private Module Registry**: Share and version modules
- **Run Management**: Automated Terraform operations
- **VCS Integration**: Git-based workflows

### Terraform Cloud vs Terraform Enterprise

**Terraform Cloud (Free/Paid):**
- Hosted service by HashiCorp
- Free tier available
- Limited to Terraform functionality

**Terraform Enterprise (Self-hosted):**
- On-premises deployment
- Full enterprise features
- Integration with other HashiCorp products

## Workspace Management

### Workspace Concepts

Workspaces in Terraform Cloud are isolated environments for managing infrastructure:

```hcl
# Example: Different workspaces for environments
# dev-workspace
# staging-workspace  
# prod-workspace
```

### Workspace Types

#### 1. Local Workspaces
Managed locally with Terraform CLI:
```bash
# Create workspace
terraform workspace new dev

# List workspaces
terraform workspace list

# Switch workspace
terraform workspace select prod
```

#### 2. Remote Workspaces
Managed in Terraform Cloud:
- **Remote State**: Stored in Terraform Cloud
- **Remote Execution**: Runs happen in Terraform Cloud
- **Team Access**: Multiple users can collaborate

### Workspace Configuration

```hcl
# Configure Terraform Cloud backend
terraform {
  cloud {
    organization = "my-organization"
    workspaces {
      name = "prod-infrastructure"
    }
  }
}

# Alternative: Use CLI configuration
terraform {
  backend "remote" {
    organization = "my-organization"
    workspaces {
      name = "prod-infrastructure"
    }
  }
}
```

## Team Collaboration Features

### Organization Structure

```
Organization: my-company
├── Workspaces
│   ├── dev-infrastructure
│   ├── staging-infrastructure
│   └── prod-infrastructure
├── Teams
│   ├── developers
│   ├── operations
│   └── security
└── Members
    ├── alice@company.com
    ├── bob@company.com
    └── charlie@company.com
```

### Team Permissions

#### Workspace Permissions
- **Read**: View workspace and state
- **Plan**: Create and view plans
- **Write**: Apply changes and manage workspace
- **Admin**: Full workspace management

#### Organization Permissions
- **Read**: View organization and workspaces
- **Manage Workspaces**: Create and manage workspaces
- **Manage Policies**: Create and manage Sentinel policies
- **Admin**: Full organization management

### VCS Integration

```yaml
# .github/workflows/terraform.yml
name: 'Terraform'

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      
    - name: Terraform Init
      run: terraform init
      
    - name: Terraform Plan
      run: terraform plan
      env:
        TF_TOKEN: ${{ secrets.TF_TOKEN }}
```

## Practical Examples

### Example 1: Basic Terraform Cloud Configuration

```hcl
# main.tf
terraform {
  required_version = ">= 1.0"
  
  cloud {
    organization = "my-organization"
    workspaces {
      name = "prod-infrastructure"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "Web Server"
    Environment = "Production"
  }
}
```

### Example 2: Multi-Environment Setup

```hcl
# environments/dev/main.tf
terraform {
  cloud {
    organization = "my-organization"
    workspaces {
      name = "dev-infrastructure"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "Web Server"
    Environment = "Development"
  }
}
```

```hcl
# environments/prod/main.tf
terraform {
  cloud {
    organization = "my-organization"
    workspaces {
      name = "prod-infrastructure"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.large"  # Larger instance for production
  
  tags = {
    Name = "Web Server"
    Environment = "Production"
  }
}
```

### Example 3: Team Collaboration with Variables

```hcl
# main.tf
terraform {
  cloud {
    organization = "my-organization"
    workspaces {
      name = "shared-infrastructure"
    }
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t2.micro"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = var.instance_type
  
  tags = {
    Name        = "Web Server"
    Environment = var.environment
    ManagedBy   = "Terraform Cloud"
  }
}
```

## Policy as Code (Sentinel)

### What is Sentinel?

Sentinel is HashiCorp's policy as code framework that:
- Enforces policies on Terraform operations
- Ensures compliance and governance
- Prevents unsafe changes
- Provides audit trails

### Basic Sentinel Policy

```hcl
# policy.sentinel
import "tfplan/v2" as tfplan

# Ensure all instances have required tags
main = rule {
  all tfplan.resource_changes as _, changes {
    changes.type is "aws_instance" implies
    all changes.after as _, resource {
      resource.tags contains "Environment" and
      resource.tags contains "Name"
    }
  }
}
```

### Advanced Sentinel Policy

```hcl
# advanced-policy.sentinel
import "tfplan/v2" as tfplan
import "strings"

# Prevent creation of resources without proper naming
main = rule {
  all tfplan.resource_changes as _, changes {
    changes.type is "aws_instance" implies
    all changes.after as _, resource {
      strings.has_prefix(resource.tags["Name"], "prod-") or
      strings.has_prefix(resource.tags["Name"], "dev-")
    }
  }
}

# Ensure cost controls
cost_control = rule {
  all tfplan.resource_changes as _, changes {
    changes.type is "aws_instance" implies
    all changes.after as _, resource {
      resource.instance_type in ["t2.micro", "t2.small", "t2.medium"]
    }
  }
}
```

## Terraform Cloud Features

### 1. Run Management

**Manual Runs:**
- Triggered by users
- Manual approval required
- Full control over execution

**Automatic Runs:**
- Triggered by VCS changes
- Automatic plan execution
- Configurable approval policies

### 2. Private Module Registry

```hcl
# Use private modules
module "vpc" {
  source = "app.terraform.io/my-organization/vpc/aws"
  version = "1.0.0"
  
  vpc_cidr = "10.0.0.0/16"
  environment = "production"
}
```

### 3. Cost Estimation

```hcl
# Enable cost estimation
terraform {
  cloud {
    organization = "my-organization"
    workspaces {
      name = "prod-infrastructure"
    }
  }
}

# Terraform Cloud will automatically estimate costs
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.large"  # Cost will be estimated
  
  tags = {
    Name = "Web Server"
  }
}
```

## Common Commands

### Terraform Cloud CLI

```bash
# Login to Terraform Cloud
terraform login

# Initialize with Terraform Cloud
terraform init

# Plan and apply (runs in Terraform Cloud)
terraform plan
terraform apply

# View workspace information
terraform workspace show
```

### API and CLI Integration

```bash
# Set Terraform Cloud token
export TF_TOKEN="your-terraform-cloud-token"

# Use Terraform Cloud as backend
terraform init -backend-config="organization=my-organization" \
               -backend-config="workspaces.name=prod-infrastructure"
```

## Advanced Topics

### 1. Workspace Variables

```hcl
# Set in Terraform Cloud UI or via API
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}
```

### 2. Run Triggers

```hcl
# Trigger runs based on workspace changes
resource "tfe_workspace" "example" {
  name         = "example-workspace"
  organization = "my-organization"
  
  vcs_repo {
    identifier = "my-organization/terraform-example"
    branch     = "main"
  }
  
  auto_apply = true
}
```

### 3. Team Management

```hcl
# Create team
resource "tfe_team" "developers" {
  name         = "developers"
  organization = "my-organization"
}

# Add team to workspace
resource "tfe_team_access" "developers" {
  team_id      = tfe_team.developers.id
  workspace_id = tfe_workspace.example.id
  access       = "write"
}
```

## Best Practices

### 1. Workspace Organization
- Use descriptive workspace names
- Separate environments into different workspaces
- Use consistent naming conventions

### 2. Team Collaboration
- Set appropriate permissions
- Use team-based access control
- Document workspace purposes

### 3. Policy Implementation
- Start with basic policies
- Gradually add complexity
- Test policies thoroughly

### 4. Security
- Use sensitive variables for secrets
- Enable audit logging
- Implement least privilege access

## Troubleshooting

### Common Issues

1. **Authentication Errors**
   ```bash
   # Error: Failed to get existing workspaces
   # Solution: Check Terraform Cloud token
   terraform login
   ```

2. **Workspace Not Found**
   ```bash
   # Error: Workspace does not exist
   # Solution: Create workspace in Terraform Cloud
   # Or check workspace name spelling
   ```

3. **Policy Violations**
   ```bash
   # Error: Policy check failed
   # Solution: Review Sentinel policy requirements
   # Update Terraform configuration to comply
   ```

### Debugging Tips

```bash
# Enable debug logging
export TF_LOG=DEBUG
terraform plan

# Check Terraform Cloud status
terraform workspace show

# View run logs in Terraform Cloud UI
```

## Summary

Terraform Cloud provides:
- **Centralized Management**: Remote state and execution
- **Team Collaboration**: Shared workspaces and permissions
- **Policy Enforcement**: Sentinel policies for governance
- **Automation**: VCS integration and automated runs
- **Cost Control**: Cost estimation and tracking

Key takeaways:
- Choose appropriate workspace structure
- Implement proper team permissions
- Use Policy as Code for governance
- Follow security best practices
- Leverage automation features

Remember: Terraform Cloud enhances team collaboration and provides enterprise-grade features for managing infrastructure at scale. 