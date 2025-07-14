 # Day 15: Workspaces

## 🎯 Learning Objectives
- Understand workspace concepts and their purpose
- Master environment management with workspaces
- Learn workspace best practices and strategies
- Apply multi-environment workspace patterns
- Implement workspace security and organization

## 📚 Core Concepts

### What are Terraform Workspaces?
- **Definition**: Named state environments within a single Terraform configuration
- **Purpose**: Manage multiple environments with the same configuration
- **Benefits**: State isolation, environment separation, and simplified management
- **Use Cases**: Development, staging, production environments

### Workspace vs Environment Management
| Approach | Pros | Cons | Use Case |
|----------|------|------|----------|
| **Workspaces** | Simple, same config | Limited isolation | Small teams |
| **Separate Directories** | Full isolation | Code duplication | Large teams |
| **Modules** | Reusable, flexible | Complex setup | Medium teams |

## 🔧 Workspace Concepts

### Basic Workspace Operations
```bash
# List workspaces
terraform workspace list

# Show current workspace
terraform workspace show

# Create new workspace
terraform workspace new development

# Switch to workspace
terraform workspace select production

# Delete workspace
terraform workspace delete staging
```

### Workspace State Management
```hcl
# Workspace-aware configuration
locals {
  # Get current workspace
  current_workspace = terraform.workspace
  
  # Environment-specific values
  environment_config = {
    development = {
      instance_count = 1
      instance_type = "t2.micro"
      environment = "dev"
    }
    staging = {
      instance_count = 2
      instance_type = "t2.small"
      environment = "staging"
    }
    production = {
      instance_count = 3
      instance_type = "t2.large"
      environment = "prod"
    }
  }
  
  # Current environment config
  config = local.environment_config[local.current_workspace]
}

# Use workspace-specific configuration
resource "aws_instance" "web" {
  count = local.config.instance_count
  
  ami           = var.ami_id
  instance_type = local.config.instance_type
  
  tags = {
    Name = "web-server-${count.index + 1}"
    Environment = local.config.environment
    Workspace = local.current_workspace
  }
}
```

### Workspace State Structure
```hcl
# State file organization with workspaces
# terraform.tfstate.d/
# ├── development/
# │   └── terraform.tfstate
# ├── staging/
# │   └── terraform.tfstate
# └── production/
#     └── terraform.tfstate

# Access workspace-specific data
output "workspace_info" {
  value = {
    current_workspace = terraform.workspace
    environment = local.config.environment
    instance_count = local.config.instance_count
  }
}
```

## 🌍 Environment Management

### Environment-Specific Variables
```hcl
# Workspace-based variable configuration
locals {
  workspace = terraform.workspace
  
  # Environment-specific variables
  environment_vars = {
    development = {
      vpc_cidr = "10.0.0.0/16"
      instance_type = "t2.micro"
      instance_count = 1
      enable_monitoring = false
      backup_retention = 1
    }
    staging = {
      vpc_cidr = "10.1.0.0/16"
      instance_type = "t2.small"
      instance_count = 2
      enable_monitoring = true
      backup_retention = 7
    }
    production = {
      vpc_cidr = "10.2.0.0/16"
      instance_type = "t2.large"
      instance_count = 3
      enable_monitoring = true
      backup_retention = 30
    }
  }
  
  # Current environment variables
  env_vars = local.environment_vars[local.workspace]
}

# Use environment-specific configuration
resource "aws_vpc" "main" {
  cidr_block = local.env_vars.vpc_cidr
  
  tags = {
    Name = "${local.workspace}-vpc"
    Environment = local.workspace
  }
}

resource "aws_instance" "app" {
  count = local.env_vars.instance_count
  
  ami           = var.ami_id
  instance_type = local.env_vars.instance_type
  
  monitoring = local.env_vars.enable_monitoring
  
  tags = {
    Name = "${local.workspace}-app-${count.index + 1}"
    Environment = local.workspace
  }
}
```

### Environment-Specific Backend Configuration
```hcl
# Workspace-aware backend configuration
terraform {
  backend "s3" {
    bucket = "terraform-state-bucket"
    key    = "workspaces/terraform.tfstate"
    region = "us-east-1"
  }
}

# Environment-specific backend keys
locals {
  workspace = terraform.workspace
  
  # Different state keys per workspace
  state_keys = {
    development = "dev/terraform.tfstate"
    staging = "staging/terraform.tfstate"
    production = "prod/terraform.tfstate"
  }
  
  current_state_key = local.state_keys[local.workspace]
}
```

### Environment-Specific Provider Configuration
```hcl
# Workspace-based provider configuration
locals {
  workspace = terraform.workspace
  
  # Environment-specific provider configs
  provider_configs = {
    development = {
      region = "us-east-1"
      profile = "dev"
    }
    staging = {
      region = "us-east-1"
      profile = "staging"
    }
    production = {
      region = "us-west-2"
      profile = "prod"
    }
  }
  
  current_provider_config = local.provider_configs[local.workspace]
}

# Configure provider based on workspace
provider "aws" {
  region  = local.current_provider_config.region
  profile = local.current_provider_config.profile
  
  default_tags {
    tags = {
      Environment = local.workspace
      ManagedBy   = "Terraform"
    }
  }
}
```

## 🚀 Workspace Best Practices

### Workspace Naming Conventions
```hcl
# Standard workspace naming
locals {
  workspace = terraform.workspace
  
  # Validate workspace name
  valid_workspaces = ["development", "staging", "production"]
  
  # Check if current workspace is valid
  is_valid_workspace = contains(local.valid_workspaces, local.workspace)
}

# Fail if invalid workspace
resource "null_resource" "workspace_validation" {
  count = local.is_valid_workspace ? 0 : 1
  
  provisioner "local-exec" {
    command = "echo 'Invalid workspace: ${local.workspace}. Valid workspaces: ${join(", ", local.valid_workspaces)}' && exit 1"
  }
}
```

### Workspace-Specific Security
```hcl
# Security groups based on workspace
locals {
  workspace = terraform.workspace
  
  # Security rules by environment
  security_rules = {
    development = [
      { port = 22, cidr = "0.0.0.0/0" },   # SSH from anywhere in dev
      { port = 80, cidr = "0.0.0.0/0" },   # HTTP from anywhere in dev
      { port = 443, cidr = "0.0.0.0/0" }   # HTTPS from anywhere in dev
    ]
    staging = [
      { port = 22, cidr = "10.0.0.0/8" },  # SSH from internal in staging
      { port = 80, cidr = "0.0.0.0/0" },   # HTTP from anywhere in staging
      { port = 443, cidr = "0.0.0.0/0" }   # HTTPS from anywhere in staging
    ]
    production = [
      { port = 22, cidr = "10.0.0.0/8" },  # SSH from internal only
      { port = 80, cidr = "0.0.0.0/0" },   # HTTP from anywhere
      { port = 443, cidr = "0.0.0.0/0" }   # HTTPS from anywhere
    ]
  }
  
  current_security_rules = local.security_rules[local.workspace]
}

# Create security group with workspace-specific rules
resource "aws_security_group" "web" {
  name        = "${local.workspace}-web-sg"
  description = "Security group for ${local.workspace} environment"
  vpc_id      = aws_vpc.main.id
  
  dynamic "ingress" {
    for_each = local.current_security_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = "tcp"
      cidr_blocks = [ingress.value.cidr]
      description = "Allow ${ingress.value.port} from ${ingress.value.cidr}"
    }
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "${local.workspace}-web-sg"
    Environment = local.workspace
  }
}
```

### Workspace Resource Naming
```hcl
# Consistent resource naming across workspaces
locals {
  workspace = terraform.workspace
  
  # Resource naming patterns
  resource_names = {
    vpc = "${local.workspace}-vpc"
    subnet = "${local.workspace}-subnet"
    instance = "${local.workspace}-instance"
    security_group = "${local.workspace}-sg"
  }
}

# Use consistent naming
resource "aws_vpc" "main" {
  cidr_block = local.env_vars.vpc_cidr
  
  tags = {
    Name = local.resource_names.vpc
    Environment = local.workspace
  }
}

resource "aws_subnet" "public" {
  count = local.env_vars.az_count
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(local.env_vars.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = "${local.resource_names.subnet}-${count.index + 1}"
    Environment = local.workspace
  }
}
```

## 🌐 Multi-Environment Strategies

### Workspace-Based Environment Strategy
```hcl
# Complete workspace environment strategy
locals {
  workspace = terraform.workspace
  
  # Comprehensive environment configuration
  environments = {
    development = {
      vpc_cidr = "10.0.0.0/16"
      instance_type = "t2.micro"
      instance_count = 1
      enable_monitoring = false
      enable_backup = false
      enable_ssl = false
      retention_days = 1
      auto_scaling = false
      load_balancer = false
    }
    staging = {
      vpc_cidr = "10.1.0.0/16"
      instance_type = "t2.small"
      instance_count = 2
      enable_monitoring = true
      enable_backup = true
      enable_ssl = true
      retention_days = 7
      auto_scaling = false
      load_balancer = true
    }
    production = {
      vpc_cidr = "10.2.0.0/16"
      instance_type = "t2.large"
      instance_count = 3
      enable_monitoring = true
      enable_backup = true
      enable_ssl = true
      retention_days = 30
      auto_scaling = true
      load_balancer = true
    }
  }
  
  current_env = local.environments[local.workspace]
}

# Create VPC
resource "aws_vpc" "main" {
  cidr_block = local.current_env.vpc_cidr
  
  tags = {
    Name = "${local.workspace}-vpc"
    Environment = local.workspace
  }
}

# Create instances
resource "aws_instance" "app" {
  count = local.current_env.instance_count
  
  ami           = var.ami_id
  instance_type = local.current_env.instance_type
  
  monitoring = local.current_env.enable_monitoring
  
  tags = {
    Name = "${local.workspace}-app-${count.index + 1}"
    Environment = local.workspace
  }
}

# Conditional load balancer
resource "aws_lb" "main" {
  count = local.current_env.load_balancer ? 1 : 0
  
  name               = "${local.workspace}-alb"
  internal           = false
  load_balancer_type = "application"
  
  tags = {
    Name = "${local.workspace}-alb"
    Environment = local.workspace
  }
}

# Conditional auto scaling
resource "aws_autoscaling_group" "app" {
  count = local.current_env.auto_scaling ? 1 : 0
  
  name                = "${local.workspace}-asg"
  desired_capacity    = local.current_env.instance_count
  max_size           = local.current_env.instance_count * 2
  min_size           = 1
  
  tags = {
    Name = "${local.workspace}-asg"
    Environment = local.workspace
  }
}
```

### Workspace State Management Strategy
```hcl
# Workspace state management
locals {
  workspace = terraform.workspace
  
  # State file organization
  state_organization = {
    development = {
      backend_key = "environments/dev/terraform.tfstate"
      state_bucket = "terraform-state-dev"
      dynamodb_table = "terraform-locks-dev"
    }
    staging = {
      backend_key = "environments/staging/terraform.tfstate"
      state_bucket = "terraform-state-staging"
      dynamodb_table = "terraform-locks-staging"
    }
    production = {
      backend_key = "environments/prod/terraform.tfstate"
      state_bucket = "terraform-state-prod"
      dynamodb_table = "terraform-locks-prod"
    }
  }
  
  current_state_config = local.state_organization[local.workspace]
}

# Backend configuration (configured per workspace)
terraform {
  backend "s3" {
    # These values are configured per workspace
    # bucket = "terraform-state-${workspace}"
    # key    = "environments/${workspace}/terraform.tfstate"
    # region = "us-east-1"
    # dynamodb_table = "terraform-locks-${workspace}"
  }
}
```

### Workspace Deployment Strategy
```hcl
# Workspace deployment configuration
locals {
  workspace = terraform.workspace
  
  # Deployment strategies by environment
  deployment_strategies = {
    development = {
      strategy = "immediate"
      approval_required = false
      rollback_enabled = true
      health_check = false
    }
    staging = {
      strategy = "rolling"
      approval_required = false
      rollback_enabled = true
      health_check = true
    }
    production = {
      strategy = "blue-green"
      approval_required = true
      rollback_enabled = true
      health_check = true
    }
  }
  
  current_deployment = local.deployment_strategies[local.workspace]
}

# Conditional deployment resources
resource "aws_codedeploy_app" "main" {
  count = local.current_deployment.strategy != "immediate" ? 1 : 0
  
  name = "${local.workspace}-app"
  
  tags = {
    Name = "${local.workspace}-codedeploy-app"
    Environment = local.workspace
  }
}

resource "aws_codedeploy_deployment_group" "main" {
  count = local.current_deployment.strategy != "immediate" ? 1 : 0
  
  app_name = aws_codedeploy_app.main[0].name
  deployment_group_name = "${local.workspace}-deployment-group"
  
  deployment_style {
    deployment_option = "WITH_TRAFFIC_CONTROL"
    deployment_type   = "IN_PLACE"
  }
  
  tags = {
    Name = "${local.workspace}-deployment-group"
    Environment = local.workspace
  }
}
```

## 🔍 Common Commands

```bash
# Workspace management
terraform workspace list
terraform workspace show
terraform workspace new development
terraform workspace select production
terraform workspace delete staging

# Workspace operations
terraform init
terraform plan
terraform apply
terraform destroy

# Workspace-specific operations
terraform plan -var-file="${workspace}.tfvars"
terraform apply -var-file="${workspace}.tfvars"
```

## 📝 Key Takeaways
- Workspaces provide state isolation for multiple environments
- Use consistent naming conventions across workspaces
- Implement workspace-specific security and configuration
- Plan workspace state management strategy carefully
- Use conditional resources for environment-specific features
- Validate workspace names and configurations
- Document workspace-specific requirements and procedures
- Consider workspace limitations for complex environments

## 🚀 Next Steps
- Practice with different workspace scenarios
- Learn about provisioners and their usage
- Explore backend configuration strategies
- Understand Terraform Cloud integration

---
*Ready for Day 16: Provisioners*