# Day 10: Dynamic Blocks and Expressions

## 🎯 Learning Objectives
- Understand dynamic blocks concept and implementation
- Master conditional expressions and their usage
- Learn splat expressions for data access
- Apply advanced expression techniques
- Implement dynamic resource configurations

## 📚 Core Concepts

### What are Dynamic Blocks?
- **Definition**: Blocks that are created dynamically based on variable data
- **Purpose**: Generate nested configuration blocks programmatically
- **Benefits**: Reduce code duplication and enable flexible configurations
- **Use Cases**: Security groups, tags, monitoring, and complex resource configurations

### Expression Types Overview
| Expression Type | Purpose | Example |
|----------------|---------|---------|
| **Dynamic Blocks** | Generate nested blocks | Security group rules |
| **Conditional** | If-then-else logic | Environment-based configs |
| **Splat** | Access multiple resources | Resource collections |
| **For** | Transform data | List/map operations |

## 🔧 Dynamic Blocks Concept

### Basic Dynamic Block Structure
```hcl
# Dynamic block syntax
resource "resource_type" "name" {
  # Static configuration
  
  dynamic "block_name" {
    for_each = var.data_structure
    content {
      # Block content using for_each values
      attribute = block_name.value.property
    }
  }
}
```

### Security Group Dynamic Blocks
```hcl
# Variable for security rules
variable "security_rules" {
  description = "Security group rules"
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = [
    {
      port        = 80
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTP access"
    },
    {
      port        = 443
      protocol    = "tcp"
      cidr_blocks = ["0.0.0.0/0"]
      description = "HTTPS access"
    },
    {
      port        = 22
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
      description = "SSH access"
    }
  ]
}

# Security group with dynamic blocks
resource "aws_security_group" "web" {
  name        = "web-security-group"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = var.security_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Tag Dynamic Blocks
```hcl
# Variable for tags
variable "resource_tags" {
  description = "Tags for resources"
  type = map(object({
    key   = string
    value = string
  }))
  default = {
    environment = {
      key   = "Environment"
      value = "production"
    }
    project = {
      key   = "Project"
      value = "terraform-learning"
    }
    owner = {
      key   = "Owner"
      value = "DevOps Team"
    }
  }
}

# Resource with dynamic tags
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  dynamic "tags" {
    for_each = var.resource_tags
    content {
      key   = tags.value.key
      value = tags.value.value
    }
  }
}
```

## 🔄 Conditional Expressions

### Basic Conditional Expressions
```hcl
# Simple conditional
variable "enable_monitoring" {
  description = "Enable detailed monitoring"
  type        = bool
  default     = false
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  monitoring = var.enable_monitoring ? "enabled" : "disabled"
  
  tags = {
    Name = "web-server"
    Monitoring = var.enable_monitoring ? "enabled" : "disabled"
  }
}
```

### Complex Conditional Expressions
```hcl
# Environment-based configuration
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

locals {
  # Conditional values based on environment
  instance_type = var.environment == "production" ? "t2.large" : "t2.micro"
  instance_count = var.environment == "production" ? 3 : 1
  enable_backup = var.environment == "production" ? true : false
}

resource "aws_instance" "app" {
  count = local.instance_count
  
  ami           = var.ami_id
  instance_type = local.instance_type
  
  tags = {
    Name = "app-server-${count.index + 1}"
    Environment = var.environment
    Backup = local.enable_backup ? "enabled" : "disabled"
  }
}
```

### Conditional Dynamic Blocks
```hcl
# Conditional monitoring configuration
variable "monitoring_config" {
  description = "Monitoring configuration"
  type = object({
    enabled = bool
    retention_days = number
    log_groups = list(string)
  })
  default = {
    enabled = false
    retention_days = 7
    log_groups = []
  }
}

resource "aws_instance" "monitored" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  # Conditional monitoring
  monitoring = var.monitoring_config.enabled ? "enabled" : "disabled"
  
  # Conditional dynamic blocks
  dynamic "tags" {
    for_each = var.monitoring_config.enabled ? {
      Monitoring = "enabled"
      RetentionDays = tostring(var.monitoring_config.retention_days)
    } : {}
    content {
      key   = tags.key
      value = tags.value
    }
  }
}
```

## ⭐ Splat Expressions

### Basic Splat Expressions
```hcl
# Create multiple subnets
resource "aws_subnet" "public" {
  count             = 3
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

# Splat expression to get all subnet IDs
output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

# Splat expression to get all subnet CIDR blocks
output "public_subnet_cidrs" {
  value = aws_subnet.public[*].cidr_block
}
```

### Splat with For_each Resources
```hcl
# Create instances with for_each
resource "aws_instance" "servers" {
  for_each = var.instances
  
  ami           = each.value.ami_id
  instance_type = each.value.instance_type
  
  tags = {
    Name = "${each.key}-server"
  }
}

# Splat expression for for_each resources
output "server_ids" {
  value = values(aws_instance.servers)[*].id
}

output "server_names" {
  value = values(aws_instance.servers)[*].tags["Name"]
}
```

### Advanced Splat Expressions
```hcl
# Complex splat expressions
resource "aws_instance" "web" {
  count = 3
  
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = {
    Name = "web-server-${count.index + 1}"
    Type = "web"
  }
}

# Multiple splat expressions
output "web_instances" {
  value = {
    ids = aws_instance.web[*].id
    public_ips = aws_instance.web[*].public_ip
    private_ips = aws_instance.web[*].private_ip
    names = aws_instance.web[*].tags["Name"]
  }
}

# Filtered splat expressions
locals {
  # Get only instances with public IPs
  instances_with_public_ips = [
    for instance in aws_instance.web : instance
    if instance.public_ip != ""
  ]
}

output "instances_with_public_ips" {
  value = local.instances_with_public_ips[*].id
}
```

## 🔧 Advanced Expression Techniques

### Nested Dynamic Blocks
```hcl
# Complex nested dynamic blocks
variable "application_config" {
  description = "Application configuration"
  type = map(object({
    instances = number
    ports = list(object({
      port = number
      protocol = string
      description = string
    }))
    tags = map(string)
  }))
  default = {
    web = {
      instances = 2
      ports = [
        { port = 80, protocol = "tcp", description = "HTTP" },
        { port = 443, protocol = "tcp", description = "HTTPS" }
      ]
      tags = { Type = "web", Environment = "production" }
    }
    app = {
      instances = 3
      ports = [
        { port = 8080, protocol = "tcp", description = "Application" }
      ]
      tags = { Type = "app", Environment = "production" }
    }
  }
}

# Resource with nested dynamic blocks
resource "aws_security_group" "application" {
  for_each = var.application_config
  
  name        = "${each.key}-sg"
  description = "Security group for ${each.key} application"
  vpc_id      = aws_vpc.main.id
  
  # Dynamic ingress rules
  dynamic "ingress" {
    for_each = each.value.ports
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ["0.0.0.0/0"]
      description = ingress.value.description
    }
  }
  
  # Dynamic tags
  dynamic "tags" {
    for_each = each.value.tags
    content {
      key   = tags.key
      value = tags.value
    }
  }
}
```

### Conditional Dynamic Blocks
```hcl
# Conditional dynamic blocks based on environment
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

variable "monitoring_rules" {
  description = "Monitoring rules by environment"
  type = map(list(object({
    port = number
    protocol = string
    description = string
  })))
  default = {
    development = [
      { port = 22, protocol = "tcp", description = "SSH" }
    ]
    production = [
      { port = 22, protocol = "tcp", description = "SSH" },
      { port = 80, protocol = "tcp", description = "HTTP" },
      { port = 443, protocol = "tcp", description = "HTTPS" },
      { port = 8080, protocol = "tcp", description = "Application" }
    ]
  }
}

resource "aws_security_group" "monitoring" {
  name        = "monitoring-sg"
  description = "Security group for monitoring"
  vpc_id      = aws_vpc.main.id
  
  # Conditional dynamic ingress
  dynamic "ingress" {
    for_each = var.monitoring_rules[var.environment]
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ["0.0.0.0/0"]
      description = ingress.value.description
    }
  }
}
```

### Data Transformation with Expressions
```hcl
# Transform data for dynamic blocks
variable "subnet_config" {
  description = "Subnet configuration"
  type = map(object({
    cidr = string
    az   = string
    public = bool
  }))
  default = {
    public1 = { cidr = "10.0.1.0/24", az = "us-east-1a", public = true }
    public2 = { cidr = "10.0.2.0/24", az = "us-east-1b", public = true }
    private1 = { cidr = "10.0.3.0/24", az = "us-east-1a", public = false }
    private2 = { cidr = "10.0.4.0/24", az = "us-east-1b", public = false }
  }
}

locals {
  # Transform subnet config for dynamic blocks
  public_subnets = {
    for k, v in var.subnet_config : k => v
    if v.public
  }
  
  private_subnets = {
    for k, v in var.subnet_config : k => v
    if !v.public
  }
}

resource "aws_subnet" "subnets" {
  for_each = var.subnet_config
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = each.value.cidr
  availability_zone = each.value.az
  
  map_public_ip_on_launch = each.value.public
  
  tags = {
    Name = "${each.key}-subnet"
    Type = each.value.public ? "public" : "private"
  }
}
```

## 🚀 Best Practices

### Dynamic Block Organization
```hcl
# Organize dynamic blocks logically
resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id
  
  # Ingress rules
  dynamic "ingress" {
    for_each = var.web_ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }
  
  # Egress rules
  dynamic "egress" {
    for_each = var.web_egress_rules
    content {
      from_port   = egress.value.port
      to_port     = egress.value.port
      protocol    = egress.value.protocol
      cidr_blocks = egress.value.cidr_blocks
      description = egress.value.description
    }
  }
  
  # Tags
  dynamic "tags" {
    for_each = var.web_tags
    content {
      key   = tags.key
      value = tags.value
    }
  }
}
```

### Expression Performance
```hcl
# Use locals for complex expressions
locals {
  # Pre-compute complex expressions
  instance_configs = {
    for k, v in var.instances : k => {
      ami = v.ami_id
      type = v.instance_type
      tags = merge(v.tags, {
        Name = "${k}-instance"
        Environment = var.environment
      })
    }
  }
  
  # Filter data once
  production_instances = {
    for k, v in var.instances : k => v
    if var.environment == "production"
  }
}

# Use pre-computed values in resources
resource "aws_instance" "servers" {
  for_each = local.instance_configs
  
  ami           = each.value.ami
  instance_type = each.value.type
  
  tags = each.value.tags
}
```

### Error Handling
```hcl
# Validate dynamic block inputs
variable "security_rules" {
  description = "Security group rules"
  type = list(object({
    port = number
    protocol = string
    cidr_blocks = list(string)
  }))
  
  validation {
    condition = alltrue([
      for rule in var.security_rules : rule.port > 0 && rule.port <= 65535
    ])
    error_message = "Port must be between 1 and 65535."
  }
  
  validation {
    condition = alltrue([
      for rule in var.security_rules : contains(["tcp", "udp", "icmp"], rule.protocol)
    ])
    error_message = "Protocol must be tcp, udp, or icmp."
  }
}
```

## 🔍 Common Commands

```bash
# Validate dynamic blocks
terraform validate

# Plan with dynamic expressions
terraform plan

# Show dynamic block results
terraform show

# Format dynamic blocks
terraform fmt
```

## 📝 Key Takeaways
- Dynamic blocks reduce code duplication and enable flexible configurations
- Conditional expressions provide environment-specific behavior
- Splat expressions simplify access to multiple resources
- Use locals for complex expression pre-computation
- Validate dynamic block inputs for reliability
- Organize dynamic blocks logically for maintainability
- Consider performance implications of complex expressions

## 🚀 Next Steps
- Practice with different dynamic block scenarios
- Learn about functions in Terraform
- Explore advanced expression patterns
- Understand data source expressions

---
*Ready for Day 11: Functions in Terraform*