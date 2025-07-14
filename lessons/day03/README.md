# Day 3: Resources and Data Sources

## 🎯 Learning Objectives
- Understand Terraform Resources and their lifecycle
- Learn about Data Sources and their purpose
- Master resource dependencies and relationships
- Understand authentication and authorization concepts
- Differentiate between Resources and Data Sources

## 📚 Core Concepts

### What are Terraform Resources?
- **Definition**: The fundamental building blocks of Terraform that represent infrastructure objects
- **Purpose**: Define, create, and manage infrastructure components
- **Examples**: Virtual machines, databases, networks, storage buckets

### Resource Lifecycle
```
Create → Update → Destroy
```

1. **Create**: Terraform creates the resource in the target environment
2. **Update**: When configuration changes, Terraform updates the resource
3. **Destroy**: When removed from configuration, Terraform destroys the resource

## 🔧 Resource Syntax

### Basic Resource Structure
```hcl
resource "resource_type" "resource_name" {
  # Resource arguments
  argument1 = "value1"
  argument2 = "value2"
  
  # Resource blocks
  block_name {
    nested_argument = "nested_value"
  }
}
```

### Resource Examples
```hcl
# AWS S3 Bucket
resource "aws_s3_bucket" "my_bucket" {
  bucket = "my-unique-bucket-name"
  
  tags = {
    Name        = "My bucket"
    Environment = "Production"
  }
}

# AWS EC2 Instance
resource "aws_instance" "web_server" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  tags = {
    Name = "Web Server"
  }
}

# Azure Virtual Network
resource "azurerm_virtual_network" "main" {
  name                = "main-network"
  resource_group_name = azurerm_resource_group.main.name
  location            = azurerm_resource_group.main.location
  address_space       = ["10.0.0.0/16"]
}
```

## 🔗 Resource Dependencies

### Implicit Dependencies
```hcl
# Terraform automatically detects dependencies
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "main" {
  vpc_id     = aws_vpc.main.id  # Implicit dependency
  cidr_block = "10.0.1.0/24"
}
```

### Explicit Dependencies
```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  depends_on = [aws_security_group.web_sg]
}

resource "aws_security_group" "web_sg" {
  name = "web-security-group"
}
```

## 📊 Data Sources

### What are Data Sources?
- **Definition**: Read-only resources that fetch data from external sources
- **Purpose**: Reference existing infrastructure or external data
- **Key Difference**: Data sources don't create or manage resources

### Data Source Syntax
```hcl
data "data_source_type" "data_name" {
  # Data source arguments
  argument1 = "value1"
  argument2 = "value2"
}
```

### Data Source Examples
```hcl
# Get AWS AMI information
data "aws_ami" "ubuntu" {
  most_recent = true
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  
  owners = ["099720109477"] # Canonical
}

# Get existing VPC
data "aws_vpc" "existing" {
  tags = {
    Name = "existing-vpc"
  }
}

# Get Azure Resource Group
data "azurerm_resource_group" "existing" {
  name = "existing-resource-group"
}
```

## 🔄 Resource vs Data Source

| Aspect | Resource | Data Source |
|--------|----------|-------------|
| **Purpose** | Creates/manages infrastructure | Reads existing data |
| **Lifecycle** | Create → Update → Destroy | Read only |
| **State** | Tracked in Terraform state | Not tracked in state |
| **Changes** | Can be modified | Read-only, no changes |
| **Dependencies** | Can depend on other resources | Can reference other resources |

### Using Data Sources with Resources
```hcl
# Get existing AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# Use AMI data in resource
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  
  tags = {
    Name = "Web Server"
  }
}
```

## 🔐 Authentication and Authorization

### AWS Authentication
```hcl
# Method 1: AWS CLI configuration
provider "aws" {
  region = "us-east-1"
  # Uses ~/.aws/credentials or environment variables
}

# Method 2: Explicit credentials
provider "aws" {
  region     = "us-east-1"
  access_key = "YOUR_ACCESS_KEY"
  secret_key = "YOUR_SECRET_KEY"
}

# Method 3: IAM Role (recommended for production)
provider "aws" {
  region = "us-east-1"
  # Uses IAM role attached to EC2 instance or ECS task
}
```

### Azure Authentication
```hcl
# Method 1: Azure CLI
provider "azurerm" {
  features {}
  # Uses az login credentials
}

# Method 2: Service Principal
provider "azurerm" {
  features {}
  subscription_id = "your-subscription-id"
  tenant_id       = "your-tenant-id"
  client_id       = "your-client-id"
  client_secret   = "your-client-secret"
}
```

### Google Cloud Authentication
```hcl
# Method 1: Application Default Credentials
provider "google" {
  project = "your-project-id"
  region  = "us-central1"
}

# Method 2: Service Account Key
provider "google" {
  project     = "your-project-id"
  region      = "us-central1"
  credentials = file("path/to/service-account-key.json")
}
```

## 🚀 Best Practices

### Resource Naming
```hcl
# Use descriptive names
resource "aws_instance" "web_server_production" {
  # ...
}

# Use consistent naming conventions
resource "aws_security_group" "web_sg" {
  # ...
}

resource "aws_security_group" "db_sg" {
  # ...
}
```

### Resource Organization
```hcl
# Group related resources
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name        = "Main VPC"
    Environment = "Production"
    ManagedBy   = "Terraform"
  }
}

resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.1.0/24"
  
  tags = {
    Name        = "Public Subnet"
    Environment = "Production"
  }
}

resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.main.id
  cidr_block = "10.0.2.0/24"
  
  tags = {
    Name        = "Private Subnet"
    Environment = "Production"
  }
}
```

### Data Source Best Practices
```hcl
# Use specific filters for data sources
data "aws_ami" "ubuntu" {
  most_recent = true
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  
  owners = ["099720109477"]
}

# Reference existing resources
data "aws_vpc" "existing" {
  tags = {
    Name = "existing-vpc"
  }
}
```

## 🔍 Common Commands

```bash
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
- Resources create and manage infrastructure
- Data sources read existing infrastructure
- Use implicit dependencies when possible
- Always use proper authentication methods
- Follow naming conventions and best practices
- Validate and format your code regularly

## 🚀 Next Steps
- Practice creating resources and data sources
- Learn about resource lifecycle management
- Explore different provider resources
- Understand state management concepts

---
*Ready for Day 4: State Management* 