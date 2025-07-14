 # Day 9: Lifecycle Meta-arguments

## 🎯 Learning Objectives
- Understand create_before_destroy concept and usage
- Master prevent_destroy strategies for critical resources
- Learn ignore_changes usage for specific attributes
- Understand replace_triggered_by for conditional replacements
- Apply custom condition lifecycle rules
- Implement lifecycle best practices

## 📚 Core Concepts

### What are Lifecycle Meta-arguments?
- **Definition**: Special arguments that control resource lifecycle behavior
- **Purpose**: Manage how Terraform handles resource creation, updates, and destruction
- **Benefits**: Prevent data loss, control update behavior, and manage dependencies
- **Use Cases**: Critical resources, zero-downtime deployments, and data preservation

### Lifecycle Arguments Overview
| Argument | Purpose | Use Case |
|----------|---------|----------|
| **create_before_destroy** | Zero-downtime updates | Load balancers, databases |
| **prevent_destroy** | Protect critical resources | Production databases, state files |
| **ignore_changes** | Skip specific updates | Tags, metadata |
| **replace_triggered_by** | Conditional replacement | AMI updates, configuration changes |

## 🔄 Create Before Destroy

### Basic Concept
```hcl
# Standard lifecycle (destroy before create)
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  # Default behavior: destroy old, create new
}

# Create before destroy lifecycle
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### Load Balancer Example
```hcl
# Application Load Balancer with create_before_destroy
resource "aws_lb" "main" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  
  lifecycle {
    create_before_destroy = true
  }
}

# Target group with create_before_destroy
resource "aws_lb_target_group" "app" {
  name     = "app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### Database Example
```hcl
# RDS instance with create_before_destroy
resource "aws_db_instance" "main" {
  identifier = "main-db"
  engine     = "mysql"
  instance_class = "db.t3.micro"
  allocated_storage = 20
  
  lifecycle {
    create_before_destroy = true
  }
}

# ElastiCache cluster with create_before_destroy
resource "aws_elasticache_cluster" "main" {
  cluster_id = "main-cache"
  engine     = "redis"
  node_type  = "cache.t3.micro"
  num_cache_nodes = 1
  
  lifecycle {
    create_before_destroy = true
  }
}
```

### Auto Scaling Group Example
```hcl
# Launch template with create_before_destroy
resource "aws_launch_template" "app" {
  name_prefix   = "app-template"
  image_id      = var.ami_id
  instance_type = var.instance_type
  
  lifecycle {
    create_before_destroy = true
  }
}

# Auto scaling group
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  desired_capacity    = 2
  max_size           = 4
  min_size           = 1
  target_group_arns  = [aws_lb_target_group.app.arn]
  vpc_zone_identifier = aws_subnet.private[*].id
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  lifecycle {
    create_before_destroy = true
  }
}
```

## 🛡️ Prevent Destroy Strategies

### Basic Prevent Destroy
```hcl
# Critical database with prevent_destroy
resource "aws_db_instance" "production" {
  identifier = "prod-db"
  engine     = "postgresql"
  instance_class = "db.t3.medium"
  allocated_storage = 100
  
  lifecycle {
    prevent_destroy = true
  }
}

# State file bucket with prevent_destroy
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-bucket"
  
  lifecycle {
    prevent_destroy = true
  }
}
```

### Conditional Prevent Destroy
```hcl
# Prevent destroy based on environment
resource "aws_db_instance" "database" {
  identifier = "app-db"
  engine     = "mysql"
  instance_class = var.instance_class
  allocated_storage = var.allocated_storage
  
  lifecycle {
    prevent_destroy = var.environment == "production"
  }
}

# Critical resources with environment check
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
  
  lifecycle {
    prevent_destroy = var.environment == "production"
  }
}
```

### Multiple Lifecycle Rules
```hcl
# Resource with multiple lifecycle rules
resource "aws_instance" "critical" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = var.environment == "production"
  }
}
```

## 🚫 Ignore Changes Usage

### Basic Ignore Changes
```hcl
# Ignore specific attributes
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  tags = {
    Name = "web-server"
    Environment = var.environment
  }
  
  lifecycle {
    ignore_changes = [
      tags["Environment"],  # Ignore environment tag changes
      ami                  # Ignore AMI changes
    ]
  }
}
```

### Ignore All Changes
```hcl
# Ignore all changes to a resource
resource "aws_instance" "legacy" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  
  lifecycle {
    ignore_changes = all
  }
}
```

### Selective Ignore Changes
```hcl
# Ignore specific attributes in complex resources
resource "aws_db_instance" "main" {
  identifier = "main-db"
  engine     = "mysql"
  instance_class = "db.t3.micro"
  allocated_storage = 20
  
  tags = {
    Name = "main-database"
    Environment = var.environment
  }
  
  lifecycle {
    ignore_changes = [
      allocated_storage,  # Ignore storage changes
      tags["Environment"], # Ignore environment tag
      engine_version      # Ignore engine version updates
    ]
  }
}
```

### Map-Based Ignore Changes
```hcl
# Ignore changes in map attributes
resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  desired_capacity    = 2
  max_size           = 4
  min_size           = 1
  
  tag {
    key                 = "Name"
    value               = "app-instance"
    propagate_at_launch = true
  }
  
  tag {
    key                 = "Environment"
    value               = var.environment
    propagate_at_launch = true
  }
  
  lifecycle {
    ignore_changes = [
      tag["Environment"],  # Ignore environment tag changes
      desired_capacity     # Ignore desired capacity changes
    ]
  }
}
```

## 🔄 Replace Triggered By

### Basic Replace Triggered By
```hcl
# Replace instance when AMI changes
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  lifecycle {
    replace_triggered_by = [
      var.ami_id  # Replace when AMI changes
    ]
  }
}
```

### Multiple Replace Triggers
```hcl
# Replace when multiple variables change
resource "aws_instance" "app" {
  ami           = var.ami_id
  instance_type = var.instance_type
  user_data     = var.user_data
  
  lifecycle {
    replace_triggered_by = [
      var.ami_id,      # Replace when AMI changes
      var.user_data    # Replace when user data changes
    ]
  }
}
```

### Resource-Based Replace Triggers
```hcl
# Replace when dependent resource changes
resource "aws_launch_template" "app" {
  name_prefix   = "app-template"
  image_id      = var.ami_id
  instance_type = var.instance_type
}

resource "aws_autoscaling_group" "app" {
  name                = "app-asg"
  desired_capacity    = 2
  max_size           = 4
  min_size           = 1
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  lifecycle {
    replace_triggered_by = [
      aws_launch_template.app  # Replace when launch template changes
    ]
  }
}
```

### Conditional Replace Triggers
```hcl
# Conditional replacement based on environment
resource "aws_instance" "web" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  lifecycle {
    replace_triggered_by = var.environment == "production" ? [
      var.ami_id  # Only replace in production
    ] : []
  }
}
```

## 🎯 Custom Condition Lifecycle Rules

### Conditional Lifecycle Rules
```hcl
# Conditional lifecycle based on environment
resource "aws_db_instance" "database" {
  identifier = "app-db"
  engine     = "mysql"
  instance_class = var.instance_class
  allocated_storage = var.allocated_storage
  
  lifecycle {
    create_before_destroy = var.environment == "production"
    prevent_destroy       = var.environment == "production"
    ignore_changes = var.environment == "development" ? [
      allocated_storage,
      instance_class
    ] : []
  }
}
```

### Complex Conditional Rules
```hcl
# Complex lifecycle rules with multiple conditions
resource "aws_instance" "critical" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  lifecycle {
    create_before_destroy = var.enable_zero_downtime
    prevent_destroy       = var.environment == "production" && var.critical_resource
    ignore_changes = var.auto_update_disabled ? [
      ami,
      instance_type
    ] : []
    replace_triggered_by = var.environment == "production" ? [
      var.ami_id
    ] : []
  }
}
```

### Data-Driven Lifecycle Rules
```hcl
# Lifecycle rules based on data sources
data "aws_ami" "latest" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami           = data.aws_ami.latest.id
  instance_type = var.instance_type
  
  lifecycle {
    create_before_destroy = true
    replace_triggered_by = [
      data.aws_ami.latest.id  # Replace when AMI updates
    ]
  }
}
```

## 🚀 Best Practices

### Resource-Specific Lifecycle Rules
```hcl
# Database resources
resource "aws_db_instance" "main" {
  identifier = "main-db"
  engine     = "mysql"
  instance_class = "db.t3.micro"
  allocated_storage = 20
  
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = var.environment == "production"
    ignore_changes = [
      allocated_storage  # Ignore storage changes
    ]
  }
}

# Load balancer resources
resource "aws_lb" "main" {
  name               = "main-alb"
  internal           = false
  load_balancer_type = "application"
  
  lifecycle {
    create_before_destroy = true
  }
}

# State management resources
resource "aws_s3_bucket" "terraform_state" {
  bucket = "terraform-state-bucket"
  
  lifecycle {
    prevent_destroy = true
  }
}
```

### Environment-Based Lifecycle Rules
```hcl
# Development environment
resource "aws_instance" "dev" {
  ami           = var.ami_id
  instance_type = "t2.micro"
  
  lifecycle {
    ignore_changes = [
      ami,  # Allow AMI changes in dev
      tags  # Allow tag changes in dev
    ]
  }
}

# Production environment
resource "aws_instance" "prod" {
  ami           = var.ami_id
  instance_type = "t2.large"
  
  lifecycle {
    create_before_destroy = true
    prevent_destroy       = true
    replace_triggered_by = [
      var.ami_id  # Only replace when AMI changes
    ]
  }
}
```

### Monitoring and Alerting
```hcl
# Resources with monitoring lifecycle rules
resource "aws_instance" "monitored" {
  ami           = var.ami_id
  instance_type = var.instance_type
  
  lifecycle {
    create_before_destroy = true
    ignore_changes = [
      tags["LastModified"]  # Ignore timestamp updates
    ]
  }
  
  tags = {
    Name = "monitored-instance"
    Environment = var.environment
    LastModified = timestamp()
  }
}
```

## 🔍 Common Commands

```bash
# Plan with lifecycle rules
terraform plan

# Apply with lifecycle considerations
terraform apply

# Show resource lifecycle
terraform show

# Force replacement (bypass lifecycle)
terraform taint aws_instance.web

# Remove from state (bypass lifecycle)
terraform state rm aws_instance.web
```

## 📝 Key Takeaways
- Use create_before_destroy for zero-downtime deployments
- Implement prevent_destroy for critical resources
- Use ignore_changes to skip specific attribute updates
- Apply replace_triggered_by for conditional replacements
- Combine lifecycle rules for complex scenarios
- Consider environment-specific lifecycle rules
- Test lifecycle rules in non-production environments

## 🚀 Next Steps
- Practice with different lifecycle scenarios
- Learn about dynamic blocks and expressions
- Explore advanced lifecycle patterns
- Understand resource dependencies

---
*Ready for Day 10: Dynamic Blocks and Expressions*