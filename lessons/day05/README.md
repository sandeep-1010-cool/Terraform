# Day 5: Variables and Outputs

## 🎯 Learning Objectives
- Understand input variables and their types
- Learn output variables and their purpose
- Master local values (locals) for code reuse
- Understand variable precedence rules
- Learn variable files (tfvars) concepts and usage

## 📚 Core Concepts

### What are Variables?
- **Input Variables**: Parameters that customize Terraform configurations
- **Output Variables**: Values that are returned after applying configuration
- **Local Values**: Reusable values within a configuration
- **Purpose**: Make configurations flexible, reusable, and maintainable

### Variable Types Overview
| Type | Description | Example |
|------|-------------|---------|
| **string** | Text values | `"us-east-1"` |
| **number** | Numeric values | `3` or `3.14` |
| **bool** | True/false values | `true` or `false` |
| **list** | Ordered collection | `["a", "b", "c"]` |
| **map** | Key-value pairs | `{key = "value"}` |
| **object** | Structured data | `{name = "John", age = 30}` |
| **any** | Any type | `"any value"` |

## 🔧 Input Variables

### Variable Declaration
```hcl
# Basic variable declaration
variable "instance_type" {
  description = "The instance type to use for the server"
  type        = string
  default     = "t2.micro"
}

# Variable with validation
variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# Complex variable type
variable "subnet_config" {
  description = "Subnet configuration"
  type = object({
    cidr_block = string
    az         = string
    tags       = map(string)
  })
}
```

### Variable Types in Detail

#### String Variables
```hcl
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  # No default - must be provided
}
```

#### Number Variables
```hcl
variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 1
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

variable "port" {
  description = "Port number for the application"
  type        = number
  default     = 8080
}
```

#### Boolean Variables
```hcl
variable "enable_monitoring" {
  description = "Enable CloudWatch monitoring"
  type        = bool
  default     = true
}

variable "enable_backup" {
  description = "Enable automated backups"
  type        = bool
  default     = false
}
```

#### List Variables
```hcl
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

variable "instance_types" {
  description = "List of allowed instance types"
  type        = list(string)
  default     = ["t2.micro", "t2.small", "t2.medium"]
}
```

#### Map Variables
```hcl
variable "tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    Environment = "development"
    Project     = "terraform-learning"
    ManagedBy   = "terraform"
  }
}

variable "instance_config" {
  description = "Instance configuration by environment"
  type        = map(object({
    instance_type = string
    disk_size     = number
  }))
  default = {
    dev = {
      instance_type = "t2.micro"
      disk_size     = 20
    }
    prod = {
      instance_type = "t2.medium"
      disk_size     = 100
    }
  }
}
```

## 📤 Output Variables

### Output Declaration
```hcl
# Basic output
output "instance_id" {
  description = "The ID of the created instance"
  value       = aws_instance.web.id
}

# Output with sensitive data
output "database_password" {
  description = "The database password"
  value       = aws_db_instance.main.password
  sensitive   = true
}

# Computed output
output "public_ip" {
  description = "The public IP of the instance"
  value       = aws_instance.web.public_ip
}
```

### Output Examples
```hcl
# Network outputs
output "vpc_id" {
  description = "The ID of the VPC"
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "List of subnet IDs"
  value       = aws_subnet.main[*].id
}

# Application outputs
output "load_balancer_dns" {
  description = "The DNS name of the load balancer"
  value       = aws_lb.main.dns_name
}

output "database_endpoint" {
  description = "The endpoint of the database"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}
```

## 🏷️ Local Values (Locals)

### Local Value Declaration
```hcl
# Basic locals
locals {
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
  }
  
  instance_name = "${var.project_name}-${var.environment}-instance"
}

# Computed locals
locals {
  subnet_cidrs = {
    public  = cidrsubnet(var.vpc_cidr, 8, 0)
    private = cidrsubnet(var.vpc_cidr, 8, 1)
  }
  
  availability_zones = slice(data.aws_availability_zones.available.names, 0, var.az_count)
}
```

### Using Locals
```hcl
# Apply common tags
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = local.common_tags
}

# Use computed values
resource "aws_subnet" "public" {
  count             = var.az_count
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = local.availability_zones[count.index]
  
  tags = merge(local.common_tags, {
    Name = "public-subnet-${count.index + 1}"
  })
}
```

## 📋 Variable Precedence Rules

### Precedence Order (Highest to Lowest)
1. **Command line flags** (`-var` or `-var-file`)
2. **Environment variables** (`TF_VAR_*`)
3. **Variable files** (`.tfvars` files)
4. **Variable defaults** (in variable declaration)
5. **Interactive prompts** (if no value provided)

### Examples

#### Command Line Variables
```bash
# Set variable via command line
terraform apply -var="instance_type=t2.large" -var="environment=production"

# Use variable file
terraform apply -var-file="production.tfvars"
```

#### Environment Variables
```bash
# Set environment variables
export TF_VAR_instance_type="t2.large"
export TF_VAR_environment="production"

# Windows PowerShell
$env:TF_VAR_instance_type="t2.large"
$env:TF_VAR_environment="production"
```

#### Variable Files
```hcl
# terraform.tfvars (automatically loaded)
instance_type = "t2.micro"
environment   = "development"

# production.tfvars
instance_type = "t2.large"
environment   = "production"
instance_count = 3

# staging.tfvars
instance_type = "t2.small"
environment   = "staging"
instance_count = 2
```

## 📁 Variable Files (TFVARS)

### Automatic Loading
```hcl
# terraform.tfvars (loaded automatically)
region         = "us-east-1"
instance_type  = "t2.micro"
environment    = "development"
instance_count = 1

tags = {
  Environment = "development"
  Project     = "terraform-learning"
}
```

### Named Variable Files
```hcl
# production.tfvars
region         = "us-east-1"
instance_type  = "t2.large"
environment    = "production"
instance_count = 3

tags = {
  Environment = "production"
  Project     = "terraform-learning"
  CostCenter  = "IT-001"
}

# staging.tfvars
region         = "us-east-1"
instance_type  = "t2.small"
environment    = "staging"
instance_count = 2

tags = {
  Environment = "staging"
  Project     = "terraform-learning"
}
```

### Using Variable Files
```bash
# Apply with specific variable file
terraform apply -var-file="production.tfvars"

# Plan with variable file
terraform plan -var-file="staging.tfvars"

# Use multiple variable files
terraform apply -var-file="common.tfvars" -var-file="production.tfvars"
```

## 🔧 Advanced Variable Techniques

### Variable Validation
```hcl
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  
  validation {
    condition     = can(regex("^t2\\.", var.instance_type))
    error_message = "Instance type must be t2 family."
  }
}

variable "cidr_block" {
  description = "CIDR block for VPC"
  type        = string
  
  validation {
    condition     = can(cidrhost(var.cidr_block, 0))
    error_message = "Must be a valid CIDR block."
  }
}
```

### Conditional Variables
```hcl
variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

# Use conditional logic
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  monitoring = var.enable_monitoring
  
  tags = merge(local.common_tags, {
    Monitoring = var.enable_monitoring ? "enabled" : "disabled"
  })
}
```

### Dynamic Outputs
```hcl
# Conditional outputs
output "load_balancer_dns" {
  description = "Load balancer DNS name"
  value       = var.enable_load_balancer ? aws_lb.main.dns_name : null
}

output "instance_public_ip" {
  description = "Instance public IP"
  value       = var.enable_public_ip ? aws_instance.web.public_ip : null
}
```

## 🚀 Best Practices

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
    enable_monitoring = bool
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

### Local Value Best Practices
```hcl
locals {
  # Common tags
  common_tags = {
    Environment = var.environment
    Project     = var.project_name
    ManagedBy   = "Terraform"
    Owner       = "DevOps Team"
  }
  
  # Computed values
  name_prefix = "${var.project_name}-${var.environment}"
  
  # Conditional values
  instance_type = var.environment == "production" ? "t2.large" : "t2.micro"
}
```

## 🔍 Common Commands

```bash
# Validate variables
terraform validate

# Plan with specific variables
terraform plan -var="instance_type=t2.large"

# Apply with variable file
terraform apply -var-file="production.tfvars"

# Show outputs
terraform output

# Show specific output
terraform output instance_id

# Show outputs in JSON
terraform output -json
```

## 📝 Key Takeaways
- Variables make configurations flexible and reusable
- Use appropriate variable types for validation
- Outputs expose important information from your infrastructure
- Locals help reduce code duplication
- Follow variable precedence rules for value assignment
- Use variable files for environment-specific configurations
- Implement validation for critical variables

## 🚀 Next Steps
- Practice with different variable types and validation
- Learn about file structure and organization
- Explore advanced variable techniques
- Understand type constraints and validation

---
*Ready for Day 6: File Structure and Organization* 