# Day 6: File Structure and Organization

## 🎯 Learning Objectives
- Understand Terraform file organization principles
- Learn the sequence of file loading
- Master best practices for project structure
- Understand module concepts introduction
- Learn effective file naming and organization strategies

## 📚 Core Concepts

### What is File Organization?
- **Purpose**: Organize Terraform code for maintainability and scalability
- **Benefits**: Easier navigation, team collaboration, and code reuse
- **Principles**: Separation of concerns, logical grouping, and consistency

### File Types in Terraform
| File Extension | Purpose | Example |
|----------------|---------|---------|
| **.tf** | Main Terraform configuration | `main.tf`, `variables.tf` |
| **.tfvars** | Variable values | `terraform.tfvars`, `production.tfvars` |
| **.tfstate** | State file | `terraform.tfstate` |
| **.tfstate.backup** | State backup | `terraform.tfstate.backup` |
| **.hcl** | HCL configuration | `terraform.hcl` |

## 📁 File Loading Sequence

### Loading Order
1. **terraform.tf** (if exists)
2. **All .tf files** (alphabetical order)
3. **terraform.tfvars** (if exists)
4. ***.auto.tfvars** files (alphabetical order)

### Example Loading Sequence
```
terraform.tf
main.tf
variables.tf
outputs.tf
providers.tf
terraform.tfvars
production.auto.tfvars
staging.auto.tfvars
```

## 🏗️ Project Structure Patterns

### Simple Project Structure
```
project/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
├── terraform.tfvars
└── README.md
```

### Environment-Based Structure
```
project/
├── environments/
│   ├── development/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars
├── modules/
│   ├── networking/
│   ├── compute/
│   └── database/
└── README.md
```

### Component-Based Structure
```
project/
├── networking/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars
├── compute/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars
├── database/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── terraform.tfvars
└── shared/
    ├── variables.tf
    └── outputs.tf
```

## 📄 File Organization Principles

### Main Configuration Files

#### main.tf
```hcl
# Main configuration file
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

# Provider configuration
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = local.common_tags
  }
}

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

# Resources
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = merge(local.common_tags, {
    Name = "${var.project_name}-vpc"
  })
}
```

#### variables.tf
```hcl
# Input variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

# Complex variables
variable "subnet_config" {
  description = "Subnet configuration"
  type = object({
    public_subnets  = list(string)
    private_subnets = list(string)
  })
}
```

#### outputs.tf
```hcl
# Output values
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "List of subnet IDs"
  value = {
    public  = aws_subnet.public[*].id
    private = aws_subnet.private[*].id
  }
}

output "availability_zones" {
  description = "List of availability zones"
  value       = data.aws_availability_zones.available.names
}
```

#### providers.tf
```hcl
# Provider configurations
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
    
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

# AWS Provider
provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = local.common_tags
  }
}

# Azure Provider (if needed)
provider "azurerm" {
  features {}
}
```

#### locals.tf
```hcl
# Local values
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
    Owner       = "DevOps Team"
  }
  
  name_prefix = "${var.project_name}-${var.environment}"
  
  # Computed values
  availability_zones = slice(data.aws_availability_zones.available.names, 0, var.az_count)
  
  subnet_cidrs = {
    public  = [for i in range(var.az_count) : cidrsubnet(var.vpc_cidr, 8, i)]
    private = [for i in range(var.az_count) : cidrsubnet(var.vpc_cidr, 8, i + var.az_count)]
  }
}
```

## 🔧 File Naming Conventions

### Standard File Names
```
main.tf          # Main configuration
variables.tf     # Input variables
outputs.tf       # Output values
providers.tf     # Provider configurations
locals.tf        # Local values
data.tf          # Data sources
resources.tf     # Resource definitions
```

### Environment-Specific Files
```
terraform.tfvars          # Default variables
production.tfvars         # Production variables
staging.tfvars           # Staging variables
development.tfvars       # Development variables
```

### Auto-Loaded Files
```
terraform.tfvars          # Automatically loaded
*.auto.tfvars            # Automatically loaded
production.auto.tfvars   # Auto-loaded for production
staging.auto.tfvars      # Auto-loaded for staging
```

## 📦 Module Concepts Introduction

### What are Modules?
- **Definition**: Reusable, self-contained Terraform configurations
- **Purpose**: Encapsulate and reuse infrastructure components
- **Benefits**: Code reuse, consistency, and maintainability

### Module Structure
```
module/
├── main.tf          # Main module configuration
├── variables.tf     # Module input variables
├── outputs.tf       # Module outputs
├── versions.tf      # Version constraints
└── README.md        # Module documentation
```

### Basic Module Example
```hcl
# modules/networking/main.tf
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = merge(var.tags, {
    Name = "${var.project_name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.public_subnets[count.index]
  availability_zone = var.availability_zones[count.index]
  
  tags = merge(var.tags, {
    Name = "${var.project_name}-public-subnet-${count.index + 1}"
  })
}
```

```hcl
# modules/networking/variables.tf
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
}

variable "project_name" {
  description = "Name of the project"
  type        = string
}

variable "az_count" {
  description = "Number of availability zones"
  type        = number
  default     = 2
}

variable "public_subnets" {
  description = "List of public subnet CIDR blocks"
  type        = list(string)
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

variable "tags" {
  description = "Common tags"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/networking/outputs.tf
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}
```

### Using Modules
```hcl
# Root configuration
module "networking" {
  source = "./modules/networking"
  
  vpc_cidr           = "10.0.0.0/16"
  project_name       = var.project_name
  az_count           = 2
  public_subnets     = ["10.0.1.0/24", "10.0.2.0/24"]
  availability_zones = data.aws_availability_zones.available.names
  
  tags = local.common_tags
}
```

## 🚀 Best Practices

### File Organization
```hcl
# Group related resources in separate files
# networking.tf
resource "aws_vpc" "main" { }
resource "aws_subnet" "public" { }
resource "aws_subnet" "private" { }

# compute.tf
resource "aws_instance" "web" { }
resource "aws_instance" "app" { }

# security.tf
resource "aws_security_group" "web" { }
resource "aws_security_group" "app" { }
```

### Variable Organization
```hcl
# Group related variables
variable "network" {
  description = "Network configuration"
  type = object({
    vpc_cidr = string
    az_count = number
    public_subnets = list(string)
  })
}

variable "compute" {
  description = "Compute configuration"
  type = object({
    instance_type = string
    instance_count = number
  })
}
```

### Output Organization
```hcl
# Group outputs by category
output "network" {
  description = "Network information"
  value = {
    vpc_id     = aws_vpc.main.id
    subnet_ids = aws_subnet.main[*].id
  }
}

output "compute" {
  description = "Compute information"
  value = {
    instance_ids = aws_instance.web[*].id
    public_ips   = aws_instance.web[*].public_ip
  }
}
```

### Environment Management
```hcl
# Use separate directories for environments
# environments/development/terraform.tfvars
environment = "development"
instance_type = "t2.micro"
instance_count = 1

# environments/production/terraform.tfvars
environment = "production"
instance_type = "t2.large"
instance_count = 3
```

## 📋 Project Structure Examples

### Small Project
```
my-project/
├── main.tf
├── variables.tf
├── outputs.tf
├── terraform.tfvars
└── README.md
```

### Medium Project
```
my-project/
├── environments/
│   ├── development/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── outputs.tf
│   │   └── terraform.tfvars
│   └── production/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── terraform.tfvars
├── modules/
│   ├── networking/
│   └── compute/
└── README.md
```

### Large Project
```
my-project/
├── environments/
│   ├── development/
│   ├── staging/
│   └── production/
├── modules/
│   ├── networking/
│   ├── compute/
│   ├── database/
│   ├── security/
│   └── monitoring/
├── shared/
│   ├── variables.tf
│   └── outputs.tf
├── scripts/
│   ├── deploy.sh
│   └── destroy.sh
├── docs/
│   ├── architecture.md
│   └── deployment.md
└── README.md
```

## 🔍 Common Commands

```bash
# Initialize project
terraform init

# Validate configuration
terraform validate

# Format code
terraform fmt

# Plan changes
terraform plan

# Apply changes
terraform apply

# Show current state
terraform show

# List resources
terraform state list
```

## 📝 Key Takeaways
- Organize files logically by function and purpose
- Use consistent naming conventions across projects
- Separate environments using directories or workspaces
- Group related resources in separate files
- Use modules for reusable infrastructure components
- Follow the file loading sequence for proper configuration
- Document your project structure and conventions

## 🚀 Next Steps
- Practice with different project structures
- Learn about advanced module concepts
- Explore workspace management
- Understand type constraints and validation

---
*Ready for Day 7: Type Constraints* 