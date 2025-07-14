# Day 14: Modules

## 🎯 Learning Objectives
- Understand module structure and organization principles
- Master module inputs and outputs configuration
- Learn module versioning strategies and best practices
- Apply module best practices for reusability and maintainability
- Implement complex module patterns and compositions

## 📚 Core Concepts

### What are Terraform Modules?
- **Definition**: Reusable, self-contained Terraform configurations
- **Purpose**: Encapsulate and reuse infrastructure components
- **Benefits**: Code reuse, consistency, maintainability, and version control
- **Structure**: Organized collection of Terraform files with defined interfaces

### Module Types Overview
| Type | Purpose | Example |
|------|---------|---------|
| **Root Module** | Main configuration | Your main Terraform files |
| **Child Module** | Reusable components | Networking, compute, database modules |
| **Published Module** | Registry modules | Terraform Registry modules |
| **Local Module** | Project-specific | Custom modules in your project |

## 🏗️ Module Structure and Organization

### Basic Module Structure
```
module/
├── main.tf          # Main module configuration
├── variables.tf     # Module input variables
├── outputs.tf       # Module outputs
├── versions.tf      # Version constraints
├── README.md        # Module documentation
└── examples/        # Usage examples
    ├── basic/
    └── advanced/
```

### Advanced Module Structure
```
modules/
├── networking/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   ├── README.md
│   └── examples/
│       ├── basic/
│       └── multi-az/
├── compute/
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   ├── versions.tf
│   ├── README.md
│   └── examples/
│       ├── single-instance/
│       └── auto-scaling/
└── database/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── versions.tf
    ├── README.md
    └── examples/
        ├── single-instance/
        └── multi-az/
```

### Module File Organization
```hcl
# modules/networking/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}

# Module resources
resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr
  
  tags = merge(var.tags, {
    Name = "${var.project_name}-vpc"
  })
}

resource "aws_subnet" "public" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
  
  map_public_ip_on_launch = true
  
  tags = merge(var.tags, {
    Name = "${var.project_name}-public-subnet-${count.index + 1}"
  })
}
```

## 🔧 Module Inputs and Outputs

### Module Variables (Inputs)
```hcl
# modules/networking/variables.tf
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
  
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  
  validation {
    condition     = length(var.project_name) > 0
    error_message = "Project name must not be empty."
  }
}

variable "az_count" {
  description = "Number of availability zones"
  type        = number
  default     = 2
  
  validation {
    condition     = var.az_count >= 1 && var.az_count <= 4
    error_message = "AZ count must be between 1 and 4."
  }
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
}

variable "tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default     = {}
}

# Complex variable types
variable "subnet_config" {
  description = "Subnet configuration"
  type = object({
    public_subnets = list(string)
    private_subnets = list(string)
    enable_nat_gateway = bool
  })
  default = {
    public_subnets = []
    private_subnets = []
    enable_nat_gateway = false
  }
}
```

### Module Outputs
```hcl
# modules/networking/outputs.tf
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_cidr_block" {
  description = "The CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "public_subnet_cidrs" {
  description = "List of public subnet CIDR blocks"
  value       = aws_subnet.public[*].cidr_block
}

# Complex outputs
output "network_info" {
  description = "Complete network information"
  value = {
    vpc_id = aws_vpc.main.id
    vpc_cidr = aws_vpc.main.cidr_block
    public_subnets = [
      for subnet in aws_subnet.public : {
        id = subnet.id
        cidr = subnet.cidr_block
        az = subnet.availability_zone
      }
    ]
    availability_zones = var.availability_zones
  }
}

# Conditional outputs
output "nat_gateway_ids" {
  description = "List of NAT Gateway IDs"
  value       = var.subnet_config.enable_nat_gateway ? aws_nat_gateway.main[*].id : []
}
```

### Using Modules
```hcl
# Root configuration using modules
module "networking" {
  source = "./modules/networking"
  
  vpc_cidr = "10.0.0.0/16"
  project_name = var.project_name
  az_count = 2
  availability_zones = data.aws_availability_zones.available.names
  
  tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
}

# Reference module outputs
module "compute" {
  source = "./modules/compute"
  
  vpc_id = module.networking.vpc_id
  subnet_ids = module.networking.public_subnet_ids
  
  instance_count = var.instance_count
  instance_type = var.instance_type
  
  depends_on = [module.networking]
}
```

## 🏷️ Module Versioning

### Version Constraints
```hcl
# modules/networking/versions.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.0"
    }
  }
}
```

### Module Source Versioning
```hcl
# Using specific module versions
module "networking" {
  source = "./modules/networking"
  version = "1.0.0"  # For published modules
}

# Using Git repository with specific version
module "networking" {
  source = "git::https://github.com/example/terraform-modules.git//networking"
  ref    = "v1.0.0"
}

# Using Terraform Registry
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 3.0"
  
  name = "my-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["us-east-1a", "us-east-1b", "us-east-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
}
```

### Semantic Versioning
```hcl
# Version management strategy
module "networking" {
  source = "./modules/networking"
  
  # Major version changes - breaking changes
  # Minor version changes - new features, backward compatible
  # Patch version changes - bug fixes
  
  vpc_cidr = "10.0.0.0/16"
  project_name = var.project_name
}
```

## 🚀 Module Best Practices

### Module Design Principles
```hcl
# Single responsibility principle
# modules/networking/main.tf - Only networking resources
resource "aws_vpc" "main" { }
resource "aws_subnet" "public" { }
resource "aws_subnet" "private" { }
resource "aws_internet_gateway" "main" { }
resource "aws_nat_gateway" "main" { }

# modules/compute/main.tf - Only compute resources
resource "aws_instance" "web" { }
resource "aws_instance" "app" { }
resource "aws_autoscaling_group" "main" { }
```

### Variable Validation
```hcl
# Comprehensive variable validation
variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
  
  validation {
    condition     = can(regex("^[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}\\.[0-9]{1,3}/[0-9]{1,2}$", var.vpc_cidr))
    error_message = "Must be a valid CIDR notation (e.g., 10.0.0.0/16)."
  }
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  
  validation {
    condition     = can(regex("^t[23]\\.[a-z]+$", var.instance_type))
    error_message = "Instance type must be t2 or t3 family."
  }
}
```

### Output Documentation
```hcl
# Well-documented outputs
output "vpc_id" {
  description = "The ID of the VPC created by this module"
  value       = aws_vpc.main.id
}

output "public_subnet_ids" {
  description = "List of public subnet IDs created by this module"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs created by this module"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ips" {
  description = "List of NAT Gateway public IPs"
  value       = aws_nat_gateway.main[*].public_ip
  sensitive   = false
}
```

### Module Composition
```hcl
# Compose modules for complex infrastructure
module "networking" {
  source = "./modules/networking"
  
  vpc_cidr = "10.0.0.0/16"
  project_name = var.project_name
  az_count = 3
  availability_zones = data.aws_availability_zones.available.names
  
  tags = local.common_tags
}

module "compute" {
  source = "./modules/compute"
  
  vpc_id = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
  
  instance_count = var.instance_count
  instance_type = var.instance_type
  
  depends_on = [module.networking]
}

module "database" {
  source = "./modules/database"
  
  vpc_id = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
  
  instance_class = var.db_instance_class
  allocated_storage = var.db_allocated_storage
  
  depends_on = [module.networking]
}
```

### Environment-Specific Modules
```hcl
# Environment-specific module usage
module "networking" {
  source = "./modules/networking"
  
  vpc_cidr = var.environment == "production" ? "10.0.0.0/16" : "10.1.0.0/16"
  project_name = var.project_name
  az_count = var.environment == "production" ? 3 : 2
  availability_zones = data.aws_availability_zones.available.names
  
  tags = merge(local.common_tags, {
    Environment = var.environment
  })
}

module "compute" {
  source = "./modules/compute"
  
  vpc_id = module.networking.vpc_id
  subnet_ids = module.networking.private_subnet_ids
  
  instance_count = var.environment == "production" ? 3 : 1
  instance_type = var.environment == "production" ? "t2.large" : "t2.micro"
  
  depends_on = [module.networking]
}
```

## 📋 Module Documentation

### README.md Template
```markdown
# Networking Module

## Description
This module creates a VPC with public and private subnets across multiple availability zones.

## Usage
```hcl
module "networking" {
  source = "./modules/networking"
  
  vpc_cidr = "10.0.0.0/16"
  project_name = "my-project"
  az_count = 2
  availability_zones = ["us-east-1a", "us-east-1b"]
  
  tags = {
    Environment = "production"
    Project     = "my-project"
  }
}
```

## Inputs
| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| vpc_cidr | CIDR block for VPC | `string` | `"10.0.0.0/16"` | no |
| project_name | Name of the project | `string` | n/a | yes |
| az_count | Number of availability zones | `number` | `2` | no |
| availability_zones | List of availability zones | `list(string)` | n/a | yes |
| tags | Common tags for all resources | `map(string)` | `{}` | no |

## Outputs
| Name | Description |
|------|-------------|
| vpc_id | The ID of the VPC |
| public_subnet_ids | List of public subnet IDs |
| private_subnet_ids | List of private subnet IDs |

## Examples
See the `examples/` directory for usage examples.
```

## 🔍 Common Commands

```bash
# Initialize modules
terraform init

# Validate modules
terraform validate

# Plan with modules
terraform plan

# Apply modules
terraform apply

# Show module outputs
terraform output

# Destroy modules
terraform destroy
```

## 📝 Key Takeaways
- Modules enable code reuse and maintainability
- Use clear input/output interfaces for modules
- Implement proper versioning for module stability
- Follow single responsibility principle for module design
- Document modules thoroughly with examples
- Validate module inputs for reliability
- Use module composition for complex infrastructure
- Implement environment-specific module configurations

## 🚀 Next Steps
- Practice with different module patterns
- Learn about workspace management
- Explore module testing strategies
- Understand module publishing and sharing

---
*Ready for Day 15: Workspaces* 