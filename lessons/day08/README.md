# Day 8: Meta-arguments

## 🎯 Learning Objectives
- Understand count meta-argument and its usage
- Master for_each loop concepts and implementation
- Learn for loop expressions and their applications
- Apply meta-argument best practices
- Differentiate between count and for_each

## 📚 Core Concepts

### What are Meta-arguments?
- **Definition**: Special arguments that control resource behavior and creation
- **Purpose**: Enable dynamic resource creation and configuration
- **Types**: count, for_each, depends_on, lifecycle, provider
- **Benefits**: Reduce code duplication and enable conditional resource creation

### Meta-arguments Overview
| Meta-argument | Purpose | Use Case |
|---------------|---------|----------|
| **count** | Create multiple similar resources | Sequential resources |
| **for_each** | Create resources from map/set | Named resources |
| **depends_on** | Explicit dependencies | Resource ordering |
| **lifecycle** | Resource lifecycle rules | Update/destroy behavior |
| **provider** | Specify provider instance | Multi-provider setups |

## 🔢 Count Meta-argument

### Basic Count Usage
```hcl
# Create multiple instances
resource "aws_instance" "web" {
  count         = 3
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = {
    Name = "web-server-${count.index + 1}"
  }
}

# Reference count resources
output "instance_ids" {
  value = aws_instance.web[*].id
}

output "instance_public_ips" {
  value = aws_instance.web[*].public_ip
}
```

### Count with Variables
```hcl
# Variable-driven count
variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 2
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

resource "aws_instance" "app" {
  count         = var.instance_count
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = {
    Name = "app-server-${count.index + 1}"
    Environment = var.environment
  }
}
```

### Count with Conditional Logic
```hcl
# Conditional count
variable "enable_monitoring" {
  description = "Enable monitoring instances"
  type        = bool
  default     = false
}

resource "aws_instance" "monitoring" {
  count         = var.enable_monitoring ? 1 : 0
  ami           = var.monitoring_ami
  instance_type = "t2.small"
  
  tags = {
    Name = "monitoring-server"
    Purpose = "monitoring"
  }
}

# Conditional outputs
output "monitoring_instance_id" {
  value = var.enable_monitoring ? aws_instance.monitoring[0].id : null
}
```

### Count with Data Sources
```hcl
# Count based on data source
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_subnet" "public" {
  count             = length(data.aws_availability_zones.available.names)
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}
```

## 🔄 For_each Meta-argument

### Basic For_each Usage
```hcl
# For_each with map
variable "instances" {
  description = "Instance configuration"
  type = map(object({
    instance_type = string
    ami_id       = string
  }))
  default = {
    web = {
      instance_type = "t2.micro"
      ami_id       = "ami-12345678"
    }
    app = {
      instance_type = "t2.small"
      ami_id       = "ami-87654321"
    }
    db = {
      instance_type = "t2.medium"
      ami_id       = "ami-11223344"
    }
  }
}

resource "aws_instance" "servers" {
  for_each = var.instances
  
  ami           = each.value.ami_id
  instance_type = each.value.instance_type
  
  tags = {
    Name = "${each.key}-server"
    Type = each.key
  }
}
```

### For_each with Sets
```hcl
# For_each with set
variable "security_groups" {
  description = "Security group names"
  type        = set(string)
  default     = ["web-sg", "app-sg", "db-sg"]
}

resource "aws_security_group" "groups" {
  for_each = var.security_groups
  
  name        = each.value
  description = "Security group for ${each.value}"
  vpc_id      = aws_vpc.main.id
  
  tags = {
    Name = each.value
  }
}
```

### For_each with Complex Data
```hcl
# Complex for_each structure
variable "subnet_config" {
  description = "Subnet configuration"
  type = map(object({
    cidr_block = string
    az         = string
    public     = bool
  }))
  default = {
    public1 = {
      cidr_block = "10.0.1.0/24"
      az         = "us-east-1a"
      public     = true
    }
    public2 = {
      cidr_block = "10.0.2.0/24"
      az         = "us-east-1b"
      public     = true
    }
    private1 = {
      cidr_block = "10.0.3.0/24"
      az         = "us-east-1a"
      public     = false
    }
  }
}

resource "aws_subnet" "subnets" {
  for_each = var.subnet_config
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr_block
  availability_zone = each.value.az
  
  map_public_ip_on_launch = each.value.public
  
  tags = {
    Name = "${each.key}-subnet"
    Type = each.value.public ? "public" : "private"
  }
}
```

## 🔁 For Loop Expressions

### For Loop with Lists
```hcl
# Transform list values
variable "ports" {
  description = "Ports to open"
  type        = list(number)
  default     = [80, 443, 8080]
}

locals {
  # Transform numbers to strings
  port_strings = [for port in var.ports : tostring(port)]
  
  # Filter and transform
  high_ports = [for port in var.ports : port if port > 1024]
  
  # Create map from list
  port_map = { for port in var.ports : "port-${port}" => port }
}

# Use in resource
resource "aws_security_group" "web" {
  name = "web-sg"
  
  dynamic "ingress" {
    for_each = var.ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
    }
  }
}
```

### For Loop with Maps
```hcl
# Transform map values
variable "instances" {
  type = map(object({
    instance_type = string
    ami_id       = string
  }))
}

locals {
  # Filter map entries
  web_instances = {
    for k, v in var.instances : k => v
    if can(regex("web", k))
  }
  
  # Transform map values
  instance_types = {
    for k, v in var.instances : k => v.instance_type
  }
  
  # Create new map structure
  instance_config = {
    for k, v in var.instances : k => {
      type = v.instance_type
      ami  = v.ami_id
      name = "${k}-instance"
    }
  }
}
```

### For Loop with Objects
```hcl
# Complex for loop transformations
variable "network_config" {
  type = object({
    vpc_cidr = string
    subnets = map(object({
      cidr = string
      az   = string
    }))
  })
}

locals {
  # Extract subnet CIDRs
  subnet_cidrs = [for k, v in var.network_config.subnets : v.cidr]
  
  # Create availability zone map
  az_map = {
    for k, v in var.network_config.subnets : k => v.az
  }
  
  # Validate CIDR blocks
  valid_subnets = {
    for k, v in var.network_config.subnets : k => v
    if can(cidrhost(v.cidr, 0))
  }
}
```

## 🔗 Count vs For_each Comparison

### When to Use Count
```hcl
# Use count for:
# - Sequential resources
# - Simple numeric iterations
# - When order matters

resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}
```

### When to Use For_each
```hcl
# Use for_each for:
# - Named resources
# - Map-based configurations
# - When you need resource names

resource "aws_instance" "servers" {
  for_each = var.instances
  
  ami           = each.value.ami_id
  instance_type = each.value.instance_type
  
  tags = {
    Name = "${each.key}-server"
  }
}
```

### Comparison Table
| Aspect | Count | For_each |
|--------|-------|----------|
| **Resource Names** | `resource[0]`, `resource[1]` | `resource["name1"]`, `resource["name2"]` |
| **State Management** | Index-based | Key-based |
| **Use Case** | Sequential resources | Named resources |
| **Flexibility** | Less flexible | More flexible |
| **State Stability** | Sensitive to order changes | Stable with key changes |

## 🔧 Advanced Meta-argument Techniques

### Conditional Resource Creation
```hcl
# Conditional count
variable "environments" {
  type = list(string)
  default = ["dev", "staging", "prod"]
}

resource "aws_instance" "environment_servers" {
  count = contains(var.environments, var.environment) ? 1 : 0
  
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = {
    Name = "${var.environment}-server"
  }
}

# Conditional for_each
variable "enable_services" {
  type = map(bool)
  default = {
    web = true
    app = true
    db  = false
  }
}

resource "aws_instance" "service_servers" {
  for_each = {
    for k, v in var.service_config : k => v
    if var.enable_services[k]
  }
  
  ami           = each.value.ami_id
  instance_type = each.value.instance_type
  
  tags = {
    Name = "${each.key}-server"
  }
}
```

### Dynamic Blocks with Meta-arguments
```hcl
# Dynamic blocks with count
resource "aws_security_group" "web" {
  name = "web-sg"
  
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

# Dynamic blocks with for_each
resource "aws_security_group" "app" {
  name = "app-sg"
  
  dynamic "ingress" {
    for_each = var.security_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
    }
  }
}
```

### Complex Data Transformations
```hcl
# Transform complex data structures
variable "application_config" {
  type = map(object({
    instances = number
    ports     = list(number)
    tags      = map(string)
  }))
}

locals {
  # Flatten configuration for resource creation
  all_instances = flatten([
    for app_name, app_config in var.application_config : [
      for i in range(app_config.instances) : {
        app_name = app_name
        instance_index = i
        ports = app_config.ports
        tags  = app_config.tags
      }
    ]
  ])
  
  # Create instance map
  instance_map = {
    for instance in local.all_instances : 
    "${instance.app_name}-${instance.instance_index}" => instance
  }
}

resource "aws_instance" "app_instances" {
  for_each = local.instance_map
  
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = merge(each.value.tags, {
    Name = "${each.value.app_name}-instance-${each.value.instance_index}"
    App  = each.value.app_name
  })
}
```

## 🚀 Best Practices

### Resource Naming
```hcl
# Use descriptive names with meta-arguments
resource "aws_instance" "web_servers" {
  count = var.web_instance_count
  
  ami           = var.web_ami
  instance_type = var.web_instance_type
  
  tags = {
    Name = "web-server-${count.index + 1}"
    Type = "web"
  }
}

resource "aws_instance" "app_servers" {
  for_each = var.app_instances
  
  ami           = each.value.ami_id
  instance_type = each.value.instance_type
  
  tags = {
    Name = "${each.key}-server"
    Type = "app"
  }
}
```

### State Management
```hcl
# Use for_each for stable state management
variable "subnets" {
  type = map(object({
    cidr = string
    az   = string
  }))
}

resource "aws_subnet" "subnets" {
  for_each = var.subnets
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  
  tags = {
    Name = "${each.key}-subnet"
  }
}
```

### Error Handling
```hcl
# Validate meta-argument inputs
variable "instance_count" {
  type = number
  
  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

variable "instance_map" {
  type = map(object({
    instance_type = string
    ami_id       = string
  }))
  
  validation {
    condition = alltrue([
      for k, v in var.instance_map : can(regex("^t2\\.", v.instance_type))
    ])
    error_message = "All instance types must be t2 family."
  }
}
```

## 🔍 Common Commands

```bash
# Validate meta-arguments
terraform validate

# Plan with meta-arguments
terraform plan

# Show specific count resource
terraform state show 'aws_instance.web[0]'

# Show specific for_each resource
terraform state show 'aws_instance.servers["web"]'

# List all count resources
terraform state list | grep "aws_instance.web"

# List all for_each resources
terraform state list | grep "aws_instance.servers"
```

## 📝 Key Takeaways
- Use count for sequential, numeric-based resources
- Use for_each for named, map-based resources
- For loops enable data transformation and filtering
- Meta-arguments reduce code duplication
- Choose the right meta-argument for your use case
- Validate inputs for meta-arguments
- Use descriptive naming conventions

## 🚀 Next Steps
- Practice with different meta-argument scenarios
- Learn about lifecycle meta-arguments
- Explore advanced for loop expressions
- Understand resource dependencies

---
*Ready for Day 9: Lifecycle Meta-arguments* 