# Day 12: Advanced Functions

## 🎯 Learning Objectives
- Master file and path functions for file operations
- Understand encoding functions for data transformation
- Learn date and time functions for temporal operations
- Apply advanced type conversion functions
- Implement complex function combinations and patterns

## 📚 Core Concepts

### What are Advanced Functions?
- **Definition**: Specialized functions for complex operations and data handling
- **Purpose**: Handle file operations, encoding, time calculations, and advanced type conversions
- **Benefits**: Enable sophisticated data processing and system integration
- **Categories**: File/Path, Encoding, Date/Time, and Advanced Type Conversion

### Advanced Function Categories
| Category | Purpose | Examples |
|----------|---------|----------|
| **File & Path** | File system operations | `file()`, `fileexists()`, `path()` |
| **Encoding** | Data encoding/decoding | `base64encode()`, `base64decode()`, `jsonencode()` |
| **Date & Time** | Temporal operations | `timestamp()`, `formatdate()`, `timeadd()` |
| **Advanced Type** | Complex conversions | `try()`, `can()`, `sensitive()` |

## 📁 File and Path Functions

### File Reading and Existence
```hcl
# Basic file operations
locals {
  # Read file contents
  user_data = file("${path.module}/user-data.sh")
  config_json = file("${path.module}/config.json")
  
  # Check file existence
  user_data_exists = fileexists("${path.module}/user-data.sh")
  config_exists = fileexists("${path.module}/config.tf")
  
  # Conditional file reading
  user_data_content = local.user_data_exists ? local.user_data : "#!/bin/bash\necho 'default'"
}

# File operations with error handling
locals {
  # Safe file reading with try
  safe_user_data = try(
    file("${path.module}/user-data.sh"),
    "#!/bin/bash\necho 'No user data found'"
  )
  
  # Check multiple files
  required_files = [
    "main.tf",
    "variables.tf",
    "outputs.tf"
  ]
  
  missing_files = [
    for file in local.required_files : file
    if !fileexists("${path.module}/${file}")
  ]
}
```

### Path Functions and Manipulation
```hcl
# Path operations
locals {
  # Current working directory
  current_dir = path.cwd
  
  # Module directory
  module_dir = path.module
  
  # Root directory
  root_dir = path.root
  
  # Relative paths
  config_dir = "${path.module}/config"
  scripts_dir = "${path.module}/scripts"
  templates_dir = "${path.module}/templates"
}

# Dynamic path construction
locals {
  environment = var.environment
  
  # Environment-specific paths
  env_config_path = "${path.module}/config/${local.environment}"
  env_scripts_path = "${path.module}/scripts/${local.environment}"
  
  # Check environment-specific files
  env_config_exists = fileexists("${local.env_config_path}/config.json")
  env_scripts_exist = fileexists("${local.env_scripts_path}/setup.sh")
}
```

### Template and Configuration Files
```hcl
# Template file processing
locals {
  # Read template files
  nginx_config = file("${path.module}/templates/nginx.conf")
  apache_config = file("${path.module}/templates/httpd.conf")
  
  # Environment-specific templates
  template_files = {
    nginx = "${path.module}/templates/nginx-${var.environment}.conf"
    apache = "${path.module}/templates/httpd-${var.environment}.conf"
  }
  
  # Conditional template selection
  web_config = fileexists(local.template_files.nginx) ? 
    file(local.template_files.nginx) : 
    file("${path.module}/templates/nginx-default.conf")
}

# Configuration file processing
locals {
  # JSON configuration
  app_config_json = file("${path.module}/config/app.json")
  app_config = jsondecode(local.app_config_json)
  
  # YAML-like configuration (as JSON)
  db_config_json = file("${path.module}/config/database.json")
  db_config = jsondecode(local.db_config_json)
}
```

## 🔐 Encoding Functions

### Base64 Encoding and Decoding
```hcl
# Basic base64 operations
locals {
  # Encode string to base64
  secret_string = "my-secret-password"
  encoded_secret = base64encode(local.secret_string)
  
  # Decode base64 string
  decoded_secret = base64decode(local.encoded_secret)
  
  # Encode file contents
  user_data_file = file("${path.module}/user-data.sh")
  encoded_user_data = base64encode(local.user_data_file)
}

# Base64 with sensitive data
locals {
  # Sensitive data encoding
  database_password = "super-secret-password"
  encoded_password = base64encode(local.database_password)
  
  # Use in resources
  sensitive_config = {
    password = local.encoded_password
    username = base64encode("admin")
  }
}

resource "aws_instance" "database" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  user_data = base64encode(templatefile("${path.module}/templates/db-init.sh", {
    db_password = local.database_password
    db_username = "admin"
  }))
}
```

### JSON Encoding and Decoding
```hcl
# JSON operations
locals {
  # Encode data to JSON
  config_data = {
    environment = var.environment
    region      = var.aws_region
    instances   = var.instance_count
    features    = {
      monitoring = true
      backup     = var.environment == "production"
      ssl        = true
    }
  }
  
  config_json = jsonencode(local.config_data)
  
  # Decode JSON data
  external_config = file("${path.module}/external-config.json")
  parsed_config = jsondecode(local.external_config)
}

# JSON with complex structures
locals {
  # Complex nested structure
  application_config = {
    name = "my-app"
    version = "1.0.0"
    environments = {
      development = {
        instances = 1
        instance_type = "t2.micro"
        features = ["basic-monitoring"]
      }
      production = {
        instances = 3
        instance_type = "t2.large"
        features = ["monitoring", "backup", "ssl", "load-balancer"]
      }
    }
  }
  
  config_json_string = jsonencode(local.application_config)
}
```

### URL Encoding
```hcl
# URL encoding functions
locals {
  # Encode URL parameters
  api_url = "https://api.example.com"
  endpoint = "/users"
  params = {
    name = "John Doe"
    email = "john@example.com"
    role = "admin"
  }
  
  # Build URL with encoded parameters
  encoded_params = [
    for k, v in local.params : "${urlencode(k)}=${urlencode(v)}"
  ]
  
  full_url = "${local.api_url}${local.endpoint}?${join("&", local.encoded_params)}"
}

# URL encoding for API calls
locals {
  # Encode complex URLs
  s3_bucket = "my-terraform-state"
  s3_key = "production/terraform.tfstate"
  
  s3_url = "s3://${local.s3_bucket}/${urlencode(local.s3_key)}"
  
  # Encode special characters
  file_path = "config/production/special-folder (v2)/config.json"
  encoded_path = urlencode(local.file_path)
}
```

## 📅 Date and Time Functions

### Timestamp Functions
```hcl
# Basic timestamp operations
locals {
  # Current timestamp
  current_time = timestamp()
  
  # Parse timestamp
  time_parts = {
    year = formatdate("YYYY", local.current_time)
    month = formatdate("MM", local.current_time)
    day = formatdate("DD", local.current_time)
    hour = formatdate("HH", local.current_time)
    minute = formatdate("mm", local.current_time)
    second = formatdate("ss", local.current_time)
  }
}

# Timestamp for resource naming
locals {
  # Create timestamp-based names
  deployment_timestamp = formatdate("YYYY-MM-DD-HH-mm", timestamp())
  
  resource_names = {
    instance = "web-server-${local.deployment_timestamp}"
    bucket = "terraform-state-${local.deployment_timestamp}"
    log_group = "/aws/lambda/my-function-${local.deployment_timestamp}"
  }
}
```

### Date Formatting and Manipulation
```hcl
# Date formatting
locals {
  current_time = timestamp()
  
  # Different date formats
  iso_date = formatdate("YYYY-MM-DD", local.current_time)
  readable_date = formatdate("MMMM DD, YYYY", local.current_time)
  time_only = formatdate("HH:mm:ss", local.current_time)
  full_datetime = formatdate("YYYY-MM-DD HH:mm:ss", local.current_time)
}

# Date arithmetic
locals {
  current_time = timestamp()
  
  # Add time periods
  one_hour_later = timeadd(local.current_time, "1h")
  one_day_later = timeadd(local.current_time, "24h")
  one_week_later = timeadd(local.current_time, "168h")  # 7 days
  one_month_later = timeadd(local.current_time, "720h") # 30 days
}

# Time-based conditional logic
locals {
  current_time = timestamp()
  deployment_hour = formatdate("HH", local.current_time)
  
  # Deploy only during business hours
  is_business_hours = tonumber(local.deployment_hour) >= 9 && tonumber(local.deployment_hour) <= 17
  
  # Environment-specific deployment windows
  can_deploy_production = local.is_business_hours && var.environment == "production"
  can_deploy_development = true  # Always allow dev deployments
}
```

### Time-based Resource Management
```hcl
# Time-based resource creation
locals {
  current_time = timestamp()
  deployment_date = formatdate("YYYY-MM-DD", local.current_time)
  
  # Create resources with time-based tags
  time_based_tags = {
    DeployedAt = local.current_time
    DeployedDate = local.deployment_date
    DeploymentHour = formatdate("HH:mm", local.current_time)
    TimeZone = "UTC"
  }
}

resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = merge(local.time_based_tags, {
    Name = "web-server-${formatdate("YYYY-MM-DD-HH-mm", timestamp())}"
    Environment = var.environment
  })
}

# Time-based backup retention
locals {
  current_time = timestamp()
  
  # Calculate backup retention dates
  retention_periods = {
    daily = timeadd(local.current_time, "-7d")    # 7 days
    weekly = timeadd(local.current_time, "-30d")  # 30 days
    monthly = timeadd(local.current_time, "-365d") # 1 year
  }
}
```

## 🔄 Advanced Type Conversion Functions

### Try and Can Functions
```hcl
# Safe type conversions with try
locals {
  # Safe number conversion
  string_number = "123"
  safe_number = try(tonumber(local.string_number), 0)
  
  # Safe boolean conversion
  string_bool = "true"
  safe_bool = try(tobool(local.string_bool), false)
  
  # Safe map lookup
  config = {
    environment = "production"
    region = "us-east-1"
  }
  
  safe_value = try(lookup(local.config, "environment", "development"), "fallback")
}

# Conditional type checking with can
locals {
  # Check if value can be converted
  test_values = ["123", "abc", "true", "false", "not-a-number"]
  
  valid_numbers = [
    for value in local.test_values : value
    if can(tonumber(value))
  ]
  
  valid_booleans = [
    for value in local.test_values : value
    if can(tobool(value))
  ]
}
```

### Sensitive Data Handling
```hcl
# Sensitive data functions
locals {
  # Mark data as sensitive
  database_password = sensitive("super-secret-password")
  api_key = sensitive("sk-1234567890abcdef")
  
  # Sensitive configuration
  sensitive_config = {
    password = local.database_password
    api_key = local.api_key
    token = sensitive(base64encode("access-token"))
  }
}

# Sensitive outputs
output "database_endpoint" {
  value = aws_db_instance.main.endpoint
  sensitive = true
}

output "api_credentials" {
  value = {
    username = "admin"
    password = sensitive(local.database_password)
    api_key = local.api_key
  }
  sensitive = true
}
```

### Complex Type Conversions
```hcl
# Advanced type conversion patterns
locals {
  # Mixed data types
  mixed_data = [
    "string",
    123,
    true,
    { key = "value" },
    [1, 2, 3]
  ]
  
  # Type-aware processing
  processed_data = [
    for item in local.mixed_data : {
      original = item
      type = try(
        tonumber(item) != null ? "number" : null,
        try(tobool(item) != null ? "boolean" : null, "string")
      )
      converted = try(tonumber(item), try(tobool(item), tostring(item)))
    }
  ]
}

# Dynamic type conversion
locals {
  # Convert based on content
  convert_value = (value) => {
    return try(
      tonumber(value) != null ? tonumber(value) : null,
      try(tobool(value) != null ? tobool(value) : null, value)
    )
  }
  
  # Apply conversion to list
  string_values = ["123", "true", "hello", "456", "false"]
  converted_values = [
    for value in local.string_values : local.convert_value(value)
  ]
}
```

## 🚀 Advanced Function Patterns

### Function Composition
```hcl
# Complex function compositions
locals {
  # Multi-step data processing
  raw_data = file("${path.module}/data.json")
  parsed_data = jsondecode(local.raw_data)
  
  # Process and transform data
  processed_data = {
    for k, v in local.parsed_data : upper(k) => {
      value = try(tonumber(v), v)
      type = try(tonumber(v) != null ? "number" : "string")
      encoded = base64encode(tostring(v))
    }
  }
  
  # Create final configuration
  final_config = jsonencode(local.processed_data)
}

# Conditional function composition
locals {
  environment = var.environment
  
  # Environment-specific processing
  config_processor = (data) => {
    return var.environment == "production" ? 
      jsonencode(merge(data, { production = true })) :
      jsonencode(merge(data, { development = true }))
  }
  
  # Apply processing
  base_config = {
    region = var.aws_region
    instances = var.instance_count
  }
  
  processed_config = local.config_processor(local.base_config)
}
```

### Error Handling Patterns
```hcl
# Comprehensive error handling
locals {
  # Safe file operations
  safe_file_reader = (file_path, default_content) => {
    return try(
      file(file_path),
      default_content
    )
  }
  
  # Safe JSON parsing
  safe_json_parser = (json_string, default_value) => {
    return try(
      jsondecode(json_string),
      default_value
    )
  }
  
  # Safe type conversion
  safe_converter = (value, target_type) => {
    return target_type == "number" ? try(tonumber(value), 0) :
           target_type == "boolean" ? try(tobool(value), false) :
           target_type == "string" ? tostring(value) :
           value
  }
}

# Apply safe operations
locals {
  user_data = local.safe_file_reader("${path.module}/user-data.sh", "#!/bin/bash\necho 'default'")
  config_data = local.safe_json_parser(file("${path.module}/config.json"), {})
  instance_count = local.safe_converter(var.instance_count_string, "number")
}
```

## 🔍 Common Commands

```bash
# Test advanced functions
terraform console
> timestamp()
> formatdate("YYYY-MM-DD", timestamp())
> base64encode("hello world")
> jsonencode({name = "test", value = 123})

# Validate with advanced functions
terraform validate

# Plan with complex functions
terraform plan
```

## 📝 Key Takeaways
- File and path functions enable dynamic file system operations
- Encoding functions handle data transformation and security
- Date and time functions support temporal operations and resource management
- Advanced type conversion functions provide safe data handling
- Function composition enables complex data processing
- Error handling patterns ensure robust function execution
- Sensitive data functions protect confidential information

## 🚀 Next Steps
- Practice with complex function combinations
- Learn about data sources deep dive
- Explore advanced expression patterns
- Understand module concepts

---
*Ready for Day 13: Data Sources Deep Dive* 