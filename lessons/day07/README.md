 # Day 7: Type Constraints

## 🎯 Learning Objectives
- Understand basic types: string, number, boolean
- Master complex types: map, set, list, tuple, object
- Learn type validation and constraints
- Understand type conversion concepts
- Apply type constraints in real-world scenarios

## 📚 Core Concepts

### What are Type Constraints?
- **Definition**: Rules that define the expected data type for variables and outputs
- **Purpose**: Ensure data integrity and prevent runtime errors
- **Benefits**: Catch errors early, improve code reliability, and provide better documentation

### Type System Overview
| Category | Types | Description |
|----------|-------|-------------|
| **Primitive** | string, number, bool | Basic data types |
| **Collection** | list, set, map | Multiple values |
| **Structural** | object, tuple | Complex data structures |
| **Special** | any | Flexible type |

## 🔤 Primitive Types

### String Type
```hcl
# Basic string variable
variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "my-project"
}

# String with validation
variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

# String with regex validation
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  
  validation {
    condition     = can(regex("^t2\\.", var.instance_type))
    error_message = "Instance type must be t2 family."
  }
}
```

### Number Type
```hcl
# Basic number variable
variable "instance_count" {
  description = "Number of instances"
  type        = number
  default     = 1
}

# Number with range validation
variable "port" {
  description = "Port number"
  type        = number
  
  validation {
    condition     = var.port >= 1 && var.port <= 65535
    error_message = "Port must be between 1 and 65535."
  }
}

# Number with multiple conditions
variable "disk_size" {
  description = "Disk size in GB"
  type        = number
  
  validation {
    condition     = var.disk_size >= 20 && var.disk_size <= 1000
    error_message = "Disk size must be between 20 and 1000 GB."
  }
}
```

### Boolean Type
```hcl
# Basic boolean variable
variable "enable_monitoring" {
  description = "Enable CloudWatch monitoring"
  type        = bool
  default     = true
}

# Boolean with conditional logic
variable "enable_backup" {
  description = "Enable automated backups"
  type        = bool
  default     = false
}

# Using boolean in resources
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  monitoring = var.enable_monitoring
  
  tags = {
    Monitoring = var.enable_monitoring ? "enabled" : "disabled"
    Backup     = var.enable_backup ? "enabled" : "disabled"
  }
}
```

## 📦 Collection Types

### List Type
```hcl
# Basic list of strings
variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b", "us-east-1c"]
}

# List of numbers
variable "ports" {
  description = "List of ports to open"
  type        = list(number)
  default     = [80, 443, 8080]
}

# List with validation
variable "instance_types" {
  description = "Allowed instance types"
  type        = list(string)
  default     = ["t2.micro", "t2.small", "t2.medium"]
  
  validation {
    condition = alltrue([
      for type in var.instance_types : can(regex("^t2\\.", type))
    ])
    error_message = "All instance types must be t2 family."
  }
}

# Using lists in resources
resource "aws_subnet" "public" {
  count             = length(var.availability_zones)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = var.availability_zones[count.index]
}
```

### Set Type
```hcl
# Set of strings (unique values)
variable "security_groups" {
  description = "Security group names"
  type        = set(string)
  default     = ["web-sg", "app-sg", "db-sg"]
}

# Set of numbers
variable "allowed_ports" {
  description = "Allowed ports for security group"
  type        = set(number)
  default     = [22, 80, 443, 8080]
}

# Using sets in resources
resource "aws_security_group" "web" {
  name = "web-security-group"
  
  dynamic "ingress" {
    for_each = var.allowed_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### Map Type
```hcl
# Map of strings
variable "tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    Environment = "development"
    Project     = "terraform-learning"
    ManagedBy   = "terraform"
  }
}

# Map of numbers
variable "instance_config" {
  description = "Instance configuration by environment"
  type        = map(number)
  default = {
    dev  = 1
    staging = 2
    prod = 3
  }
}

# Map with validation
variable "environment_config" {
  description = "Environment-specific configuration"
  type        = map(object({
    instance_type = string
    instance_count = number
    disk_size = number
  }))
  default = {
    dev = {
      instance_type = "t2.micro"
      instance_count = 1
      disk_size = 20
    }
    prod = {
      instance_type = "t2.large"
      instance_count = 3
      disk_size = 100
    }
  }
}
```

## 🏗️ Structural Types

### Object Type
```hcl
# Simple object
variable "network_config" {
  description = "Network configuration"
  type = object({
    vpc_cidr = string
    az_count = number
    enable_nat = bool
  })
  default = {
    vpc_cidr = "10.0.0.0/16"
    az_count = 2
    enable_nat = true
  }
}

# Complex object with nested structures
variable "application_config" {
  description = "Application configuration"
  type = object({
    name = string
    version = string
    environment = string
    scaling = object({
      min_size = number
      max_size = number
      desired_capacity = number
    })
    monitoring = object({
      enabled = bool
      retention_days = number
    })
  })
}

# Using objects in resources
resource "aws_vpc" "main" {
  cidr_block = var.network_config.vpc_cidr
  
  tags = {
    Name = "main-vpc"
    Environment = var.application_config.environment
  }
}

resource "aws_autoscaling_group" "app" {
  min_size = var.application_config.scaling.min_size
  max_size = var.application_config.scaling.max_size
  desired_capacity = var.application_config.scaling.desired_capacity
}
```

### Tuple Type
```hcl
# Tuple with mixed types
variable "subnet_config" {
  description = "Subnet configuration"
  type = tuple([
    string,  # CIDR block
    string,  # Availability zone
    bool     # Public subnet
  ])
  default = ["10.0.1.0/24", "us-east-1a", true]
}

# Tuple with validation
variable "instance_spec" {
  description = "Instance specification"
  type = tuple([
    string,  # Instance type
    number,  # CPU cores
    number   # Memory in GB
  ])
  
  validation {
    condition = var.instance_spec[1] > 0 && var.instance_spec[2] > 0
    error_message = "CPU cores and memory must be positive values."
  }
}

# Using tuples
locals {
  subnet_cidr = var.subnet_config[0]
  subnet_az   = var.subnet_config[1]
  is_public   = var.subnet_config[2]
}

resource "aws_subnet" "main" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = local.subnet_cidr
  availability_zone = local.subnet_az
  
  map_public_ip_on_launch = local.is_public
}
```

## 🔍 Type Validation and Constraints

### Validation Blocks
```hcl
# String validation
variable "domain_name" {
  description = "Domain name"
  type        = string
  
  validation {
    condition     = can(regex("^[a-zA-Z0-9][a-zA-Z0-9-]{1,61}[a-zA-Z0-9]\\.[a-zA-Z]{2,}$", var.domain_name))
    error_message = "Domain name must be valid (e.g., example.com)."
  }
}

# Number validation with multiple conditions
variable "instance_count" {
  description = "Number of instances"
  type        = number
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

# List validation
variable "security_groups" {
  description = "Security group names"
  type        = list(string)
  
  validation {
    condition = length(var.security_groups) > 0 && length(var.security_groups) <= 5
    error_message = "Must provide 1-5 security groups."
  }
}

# Object validation
variable "database_config" {
  description = "Database configuration"
  type = object({
    engine = string
    version = string
    instance_class = string
    allocated_storage = number
  })
  
  validation {
    condition = contains(["mysql", "postgresql"], var.database_config.engine)
    error_message = "Engine must be mysql or postgresql."
  }
  
  validation {
    condition = var.database_config.allocated_storage >= 20
    error_message = "Allocated storage must be at least 20 GB."
  }
}
```

### Custom Validation Functions
```hcl
# CIDR validation
variable "vpc_cidr" {
  description = "VPC CIDR block"
  type        = string
  
  validation {
    condition     = can(cidrhost(var.vpc_cidr, 0))
    error_message = "Must be a valid CIDR block."
  }
}

# IP address validation
variable "allowed_ips" {
  description = "Allowed IP addresses"
  type        = list(string)
  
  validation {
    condition = alltrue([
      for ip in var.allowed_ips : can(regex("^(?:[0-9]{1,3}\\.){3}[0-9]{1,3}$", ip))
    ])
    error_message = "All values must be valid IP addresses."
  }
}

# Environment validation
variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

## 🔄 Type Conversion Concepts

### Implicit Conversions
```hcl
# String to number (when possible)
variable "port_string" {
  type    = string
  default = "8080"
}

locals {
  port_number = tonumber(var.port_string)
}

# Number to string
variable "instance_count" {
  type    = number
  default = 3
}

locals {
  count_string = tostring(var.instance_count)
}
```

### Explicit Conversions
```hcl
# Type conversion functions
locals {
  # String to number
  port = tonumber("8080")
  
  # Number to string
  count_str = tostring(3)
  
  # String to bool
  enabled = tobool("true")
  
  # List to set
  unique_ports = toset([80, 443, 80, 8080])  # Results in [80, 443, 8080]
  
  # Set to list
  port_list = tolist(toset([80, 443, 8080]))
}
```

### Type Coercion
```hcl
# Automatic type coercion
variable "mixed_list" {
  type = list(any)
  default = ["string", 123, true]
}

# Using type conversion in resources
resource "aws_security_group" "web" {
  name = "web-sg"
  
  dynamic "ingress" {
    for_each = var.ports
    content {
      from_port   = tonumber(ingress.value)
      to_port     = tonumber(ingress.value)
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

## 🚀 Best Practices

### Type Safety
```hcl
# Use specific types instead of 'any'
variable "instance_type" {
  type = string  # Good
  # type = any   # Avoid when possible
}

# Use object types for complex structures
variable "network_config" {
  type = object({
    vpc_cidr = string
    subnets = list(string)
  })
}
```

### Validation Strategies
```hcl
# Validate at variable level
variable "environment" {
  type = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Invalid environment."
  }
}

# Use locals for computed validations
locals {
  valid_ports = [for port in var.ports : port if port > 0 && port <= 65535]
  
  # Validate computed values
  computed_validation = length(local.valid_ports) == length(var.ports)
}
```

### Type Documentation
```hcl
# Document complex types
variable "application_config" {
  description = "Application configuration object"
  type = object({
    name = string                    # Application name
    version = string                 # Application version
    environment = string             # Deployment environment
    scaling = object({
      min_size = number             # Minimum instances
      max_size = number             # Maximum instances
      desired_capacity = number     # Desired instances
    })
  })
}
```

## 🔍 Common Commands

```bash
# Validate type constraints
terraform validate

# Format code
terraform fmt

# Plan with type checking
terraform plan

# Show variable types
terraform console
> var.instance_type
```

## 📝 Key Takeaways
- Use specific types instead of 'any' when possible
- Implement validation for critical variables
- Understand type conversion and coercion
- Use objects for complex data structures
- Validate computed values with locals
- Document complex type structures
- Test type constraints with terraform validate

## 🚀 Next Steps
- Practice with different type constraints
- Learn about meta-arguments and loops
- Explore advanced validation techniques
- Understand lifecycle management

---
*Ready for Day 8: Meta-arguments*