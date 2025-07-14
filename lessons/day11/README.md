 # Day 11: Functions in Terraform

## 🎯 Learning Objectives
- Understand built-in functions overview and categories
- Master string manipulation functions and their usage
- Learn collection functions for list, map, and set operations
- Apply mathematical functions for calculations
- Implement function best practices and error handling

## 📚 Core Concepts

### What are Terraform Functions?
- **Definition**: Built-in operations that transform and manipulate data
- **Purpose**: Perform calculations, data transformations, and conditional logic
- **Benefits**: Reduce code complexity and enable dynamic configurations
- **Categories**: String, Collection, Mathematical, Type Conversion, and File functions

### Function Categories Overview
| Category | Purpose | Examples |
|----------|---------|----------|
| **String** | Text manipulation | `upper()`, `lower()`, `replace()` |
| **Collection** | List/map operations | `length()`, `keys()`, `values()` |
| **Mathematical** | Calculations | `max()`, `min()`, `abs()` |
| **Type Conversion** | Data type changes | `tostring()`, `tonumber()`, `tobool()` |
| **File** | File operations | `file()`, `fileexists()`, `path()` |

## 🔤 String Manipulation Functions

### Basic String Functions
```hcl
# String case conversion
locals {
  project_name = "my-terraform-project"
  upper_name   = upper(local.project_name)    # "MY-TERRAFORM-PROJECT"
  lower_name   = lower(local.project_name)    # "my-terraform-project"
  title_name   = title(local.project_name)    # "My-Terraform-Project"
}

# String length and trimming
locals {
  description = "  Hello World  "
  trimmed    = trim(local.description)        # "Hello World"
  length     = length(local.description)      # 15
  trimmed_length = length(trim(local.description)) # 11
}
```

### String Search and Replace
```hcl
# String replacement
locals {
  original = "Hello World"
  replaced = replace(local.original, "World", "Terraform")  # "Hello Terraform"
  
  # Multiple replacements
  multi_replace = replace(
    replace(local.original, "Hello", "Hi"),
    "World", "Terraform"
  )  # "Hi Terraform"
}

# String search
locals {
  text = "Terraform is awesome"
  contains_terraform = can(regex("Terraform", local.text))  # true
  starts_with = can(regex("^Terraform", local.text))       # true
  ends_with = can(regex("awesome$", local.text))           # true
}
```

### String Formatting and Concatenation
```hcl
# String concatenation
locals {
  prefix = "web"
  suffix = "server"
  full_name = "${local.prefix}-${local.suffix}"  # "web-server"
  
  # Using format function
  formatted_name = format("%s-%s-%d", local.prefix, local.suffix, 1)  # "web-server-1"
}

# String formatting with variables
locals {
  environment = "production"
  instance_count = 3
  
  formatted_string = format(
    "Environment: %s, Instances: %d",
    local.environment,
    local.instance_count
  )  # "Environment: production, Instances: 3"
}
```

### Advanced String Functions
```hcl
# String splitting and joining
locals {
  tags_string = "web,app,db"
  tags_list = split(",", local.tags_string)  # ["web", "app", "db"]
  tags_joined = join("-", local.tags_list)   # "web-app-db"
}

# String validation
locals {
  domain_name = "example.com"
  is_valid_domain = can(regex("^[a-zA-Z0-9][a-zA-Z0-9-]{1,61}[a-zA-Z0-9]\\.[a-zA-Z]{2,}$", local.domain_name))
  
  email = "user@example.com"
  is_valid_email = can(regex("^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$", local.email))
}
```

## 📦 Collection Functions

### List Functions
```hcl
# List operations
locals {
  numbers = [1, 2, 3, 4, 5]
  strings = ["apple", "banana", "cherry"]
  
  # List length
  count = length(local.numbers)  # 5
  
  # List indexing
  first = local.numbers[0]       # 1
  last  = local.numbers[length(local.numbers) - 1]  # 5
  
  # List slicing
  first_three = slice(local.numbers, 0, 3)  # [1, 2, 3]
}

# List filtering and transformation
locals {
  numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
  
  # Filter even numbers
  even_numbers = [
    for num in local.numbers : num
    if num % 2 == 0
  ]  # [2, 4, 6, 8, 10]
  
  # Transform numbers
  doubled = [
    for num in local.numbers : num * 2
  ]  # [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
}
```

### Map Functions
```hcl
# Map operations
locals {
  config = {
    environment = "production"
    region      = "us-east-1"
    instances   = 3
    monitoring  = true
  }
  
  # Map keys and values
  keys = keys(local.config)      # ["environment", "region", "instances", "monitoring"]
  values = values(local.config)  # ["production", "us-east-1", 3, true]
  
  # Map lookup
  env = lookup(local.config, "environment", "development")  # "production"
  missing = lookup(local.config, "missing_key", "default") # "default"
}

# Map filtering and transformation
locals {
  instances = {
    web = { type = "t2.micro", count = 2 }
    app = { type = "t2.small", count = 3 }
    db  = { type = "t2.medium", count = 1 }
  }
  
  # Filter instances by type
  micro_instances = {
    for k, v in local.instances : k => v
    if v.type == "t2.micro"
  }
  
  # Transform map values
  instance_types = {
    for k, v in local.instances : k => v.type
  }
}
```

### Set Functions
```hcl
# Set operations
locals {
  ports = toset([80, 443, 8080, 80, 443])  # [80, 443, 8080] (duplicates removed)
  
  # Set operations
  set_length = length(local.ports)  # 3
  
  # Convert set to list
  ports_list = tolist(local.ports)  # [80, 443, 8080]
}

# Set filtering
locals {
  all_ports = [22, 80, 443, 8080, 9000]
  allowed_ports = [80, 443, 8080]
  
  # Filter allowed ports
  filtered_ports = [
    for port in local.all_ports : port
    if contains(local.allowed_ports, port)
  ]  # [80, 443, 8080]
}
```

## 🔢 Mathematical Functions

### Basic Mathematical Functions
```hcl
# Basic arithmetic
locals {
  a = 10
  b = 5
  
  sum = local.a + local.b        # 15
  difference = local.a - local.b  # 5
  product = local.a * local.b     # 50
  quotient = local.a / local.b    # 2
  remainder = local.a % local.b   # 0
}

# Mathematical functions
locals {
  numbers = [1, 5, 10, 2, 8, 3]
  
  max_value = max(local.numbers...)  # 10
  min_value = min(local.numbers...)  # 1
  sum_all = sum(local.numbers)       # 29
  average = local.sum_all / length(local.numbers)  # 4.83
}
```

### Advanced Mathematical Functions
```hcl
# Absolute values and rounding
locals {
  negative = -10
  absolute = abs(local.negative)  # 10
  
  decimal = 3.7
  ceiling = ceil(local.decimal)   # 4
  floor = floor(local.decimal)    # 3
  rounded = round(local.decimal)  # 4
}

# Power and square root
locals {
  base = 2
  exponent = 8
  power = pow(local.base, local.exponent)  # 256
  
  number = 16
  square_root = sqrt(local.number)  # 4
}
```

### CIDR and Network Functions
```hcl
# CIDR functions
locals {
  vpc_cidr = "10.0.0.0/16"
  
  # Calculate subnet CIDRs
  public_subnet_1 = cidrsubnet(local.vpc_cidr, 8, 0)   # "10.0.0.0/24"
  public_subnet_2 = cidrsubnet(local.vpc_cidr, 8, 1)   # "10.0.1.0/24"
  private_subnet_1 = cidrsubnet(local.vpc_cidr, 8, 2)  # "10.0.2.0/24"
  
  # CIDR host calculation
  first_host = cidrhost(local.public_subnet_1, 1)  # "10.0.0.1"
  last_host = cidrhost(local.public_subnet_1, -2)  # "10.0.0.254"
}

# Network range functions
locals {
  cidr_block = "10.0.0.0/24"
  
  # Calculate network range
  network_address = cidrhost(local.cidr_block, 0)  # "10.0.0.0"
  broadcast_address = cidrhost(local.cidr_block, -1)  # "10.0.0.255"
}
```

## 🔄 Type Conversion Functions

### Basic Type Conversions
```hcl
# String conversions
locals {
  number = 42
  string_number = tostring(local.number)  # "42"
  
  boolean = true
  string_boolean = tostring(local.boolean)  # "true"
}

# Number conversions
locals {
  string_number = "123"
  number = tonumber(local.string_number)  # 123
  
  # Safe conversion with default
  invalid_string = "not-a-number"
  safe_number = try(tonumber(local.invalid_string), 0)  # 0
}

# Boolean conversions
locals {
  string_true = "true"
  string_false = "false"
  
  bool_true = tobool(local.string_true)    # true
  bool_false = tobool(local.string_false)  # false
}
```

### Collection Type Conversions
```hcl
# List and set conversions
locals {
  list_data = ["a", "b", "c", "a", "b"]
  
  # Convert list to set (removes duplicates)
  set_data = toset(local.list_data)  # ["a", "b", "c"]
  
  # Convert set back to list
  list_from_set = tolist(local.set_data)  # ["a", "b", "c"]
}

# Map conversions
locals {
  list_of_objects = [
    { key = "name", value = "web" },
    { key = "environment", value = "production" }
  ]
  
  # Convert list of objects to map
  map_from_list = {
    for item in local.list_of_objects : item.key => item.value
  }  # { name = "web", environment = "production" }
}
```

## 📁 File Functions

### File Reading Functions
```hcl
# Read file contents
locals {
  user_data = file("${path.module}/user-data.sh")
  config_json = file("${path.module}/config.json")
  
  # Parse JSON
  config = jsondecode(local.config_json)
}

# File existence check
locals {
  config_file_exists = fileexists("${path.module}/config.tf")
  user_data_exists = fileexists("${path.module}/user-data.sh")
}
```

### Path Functions
```hcl
# Path manipulation
locals {
  current_dir = path.root
  module_dir = path.module
  cwd = path.cwd
  
  # File path operations
  config_path = "${path.module}/config"
  user_data_path = "${path.module}/scripts/user-data.sh"
}
```

## 🚀 Function Best Practices

### Performance Optimization
```hcl
# Use locals for complex function calls
locals {
  # Pre-compute expensive operations
  instance_configs = {
    for k, v in var.instances : k => {
      name = format("%s-%s", k, var.environment)
      tags = merge(v.tags, {
        Environment = var.environment
        ManagedBy = "Terraform"
      })
    }
  }
  
  # Pre-compute CIDR calculations
  subnet_cidrs = {
    for i in range(var.subnet_count) : "subnet-${i}" => cidrsubnet(var.vpc_cidr, 8, i)
  }
}

# Use pre-computed values in resources
resource "aws_instance" "servers" {
  for_each = local.instance_configs
  
  ami           = each.value.ami_id
  instance_type = each.value.instance_type
  
  tags = each.value.tags
}
```

### Error Handling
```hcl
# Safe function calls with try/can
locals {
  # Safe type conversion
  safe_number = try(tonumber(var.string_number), 0)
  
  # Safe map lookup
  safe_value = try(lookup(var.config, "key", "default"), "fallback")
  
  # Safe file reading
  user_data = try(file("${path.module}/user-data.sh"), "#!/bin/bash\necho 'default'")
}

# Validation with functions
variable "domain_name" {
  type = string
  
  validation {
    condition = can(regex("^[a-zA-Z0-9][a-zA-Z0-9-]{1,61}[a-zA-Z0-9]\\.[a-zA-Z]{2,}$", var.domain_name))
    error_message = "Domain name must be valid."
  }
}
```

### Function Composition
```hcl
# Compose multiple functions
locals {
  # Complex string manipulation
  formatted_name = upper(
    replace(
      format("%s-%s", var.project_name, var.environment),
      "_", "-"
    )
  )
  
  # Complex list operations
  filtered_and_transformed = [
    for item in var.items : upper(item)
    if length(item) > 3
  ]
  
  # Complex map operations
  transformed_config = {
    for k, v in var.config : upper(k) => {
      value = v.value
      type = title(v.type)
    }
  }
}
```

## 🔍 Common Commands

```bash
# Validate functions
terraform validate

# Plan with functions
terraform plan

# Test functions in console
terraform console
> upper("hello world")
> length([1, 2, 3])
> max(1, 5, 3, 8, 2)

# Format function usage
terraform fmt
```

## 📝 Key Takeaways
- Functions enable dynamic data manipulation and calculations
- Use string functions for text processing and formatting
- Collection functions simplify list, map, and set operations
- Mathematical functions support calculations and CIDR operations
- Type conversion functions ensure data type compatibility
- Pre-compute complex function calls in locals for performance
- Use try/can for safe function execution and error handling
- Compose functions for complex data transformations

## 🚀 Next Steps
- Practice with different function combinations
- Learn about advanced functions and patterns
- Explore file and path functions
- Understand function performance implications

---
*Ready for Day 12: Advanced Functions*