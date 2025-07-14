 # Day 13: Data Sources Deep Dive

## 🎯 Learning Objectives
- Master data source concepts and advanced usage patterns
- Understand cross-resource references and dependencies
- Learn data source best practices and optimization
- Apply real-world data source scenarios
- Implement complex data source configurations

## 📚 Core Concepts

### What are Data Sources?
- **Definition**: Read-only resources that fetch data from external sources
- **Purpose**: Reference existing infrastructure and external data
- **Key Difference**: Data sources don't create or manage resources
- **Benefits**: Reuse existing infrastructure and avoid duplication

### Data Source Categories
| Category | Purpose | Examples |
|----------|---------|----------|
| **Infrastructure** | Existing cloud resources | VPCs, subnets, security groups |
| **Configuration** | External configuration | AMIs, certificates, policies |
| **Network** | Network information | Availability zones, IP ranges |
| **Identity** | Identity and access | IAM roles, users, policies |

## 🔍 Data Source Concepts and Usage

### Basic Data Source Structure
```hcl
# Data source syntax
data "data_source_type" "data_name" {
  # Data source arguments
  argument1 = "value1"
  argument2 = "value2"
  
  # Filters for specific data
  filter {
    name   = "filter_name"
    values = ["filter_value"]
  }
}
```

### Infrastructure Data Sources
```hcl
# Get existing VPC
data "aws_vpc" "existing" {
  tags = {
    Name = "existing-vpc"
    Environment = "production"
  }
}

# Get existing subnets
data "aws_subnet" "public" {
  vpc_id = data.aws_vpc.existing.id
  
  filter {
    name   = "tag:Type"
    values = ["public"]
  }
}

# Get existing security groups
data "aws_security_group" "web" {
  name = "web-security-group"
  vpc_id = data.aws_vpc.existing.id
}
```

### Configuration Data Sources
```hcl
# Get latest AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}

# Get availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Get current region
data "aws_region" "current" {}
```

### Network Data Sources
```hcl
# Get caller identity
data "aws_caller_identity" "current" {}

# Get current account
data "aws_caller_identity" "current" {}

# Get partition
data "aws_partition" "current" {}

# Get default VPC
data "aws_vpc" "default" {
  default = true
}
```

## 🔗 Cross-Resource References

### Data Source to Resource References
```hcl
# Get existing VPC data
data "aws_vpc" "existing" {
  tags = {
    Name = "existing-vpc"
  }
}

# Create resources using VPC data
resource "aws_subnet" "new" {
  vpc_id            = data.aws_vpc.existing.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = data.aws_availability_zones.available.names[0]
  
  tags = {
    Name = "new-subnet"
    VPC = data.aws_vpc.existing.tags["Name"]
  }
}

# Create security group using VPC data
resource "aws_security_group" "app" {
  name        = "app-security-group"
  description = "Security group for application"
  vpc_id      = data.aws_vpc.existing.id
  
  tags = {
    Name = "app-sg"
    VPC = data.aws_vpc.existing.id
  }
}
```

### Complex Cross-References
```hcl
# Get existing infrastructure data
data "aws_vpc" "main" {
  tags = {
    Name = "main-vpc"
  }
}

data "aws_subnet" "private" {
  vpc_id = data.aws_vpc.main.id
  
  filter {
    name   = "tag:Type"
    values = ["private"]
  }
}

data "aws_security_group" "database" {
  name = "database-sg"
  vpc_id = data.aws_vpc.main.id
}

# Create database using existing infrastructure
resource "aws_db_instance" "main" {
  identifier = "main-database"
  engine     = "mysql"
  instance_class = "db.t3.micro"
  allocated_storage = 20
  
  # Use existing subnet
  db_subnet_group_name = aws_db_subnet_group.main.name
  
  # Use existing security group
  vpc_security_group_ids = [data.aws_security_group.database.id]
  
  tags = {
    Name = "main-database"
    VPC = data.aws_vpc.main.id
  }
}

# Create subnet group using existing subnets
resource "aws_db_subnet_group" "main" {
  name = "main-db-subnet-group"
  
  subnet_ids = [
    data.aws_subnet.private.id,
    data.aws_subnet.private_2.id
  ]
  
  tags = {
    Name = "main-db-subnet-group"
  }
}
```

### Conditional Data Source References
```hcl
# Conditional data source usage
variable "use_existing_vpc" {
  description = "Use existing VPC or create new one"
  type        = bool
  default     = false
}

# Get existing VPC if specified
data "aws_vpc" "existing" {
  count = var.use_existing_vpc ? 1 : 0
  
  tags = {
    Name = "existing-vpc"
  }
}

# Create new VPC if not using existing
resource "aws_vpc" "new" {
  count = var.use_existing_vpc ? 0 : 1
  
  cidr_block = "10.0.0.0/16"
  
  tags = {
    Name = "new-vpc"
  }
}

# Use either existing or new VPC
locals {
  vpc_id = var.use_existing_vpc ? data.aws_vpc.existing[0].id : aws_vpc.new[0].id
  vpc_cidr = var.use_existing_vpc ? data.aws_vpc.existing[0].cidr_block : aws_vpc.new[0].cidr_block
}

# Create subnet using the selected VPC
resource "aws_subnet" "main" {
  vpc_id            = local.vpc_id
  cidr_block        = cidrsubnet(local.vpc_cidr, 8, 0)
  availability_zone = data.aws_availability_zones.available.names[0]
  
  tags = {
    Name = "main-subnet"
  }
}
```

## 🚀 Data Source Best Practices

### Performance Optimization
```hcl
# Use specific filters for better performance
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  # Specific filters reduce API calls
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  
  filter {
    name   = "root-device-type"
    values = ["ebs"]
  }
}

# Cache frequently used data
locals {
  # Cache availability zones
  availability_zones = data.aws_availability_zones.available.names
  
  # Cache region
  current_region = data.aws_region.current.name
  
  # Cache account ID
  account_id = data.aws_caller_identity.current.account_id
}
```

### Error Handling and Validation
```hcl
# Validate data source results
data "aws_vpc" "main" {
  tags = {
    Name = var.vpc_name
  }
}

# Validate VPC exists
locals {
  vpc_exists = length(data.aws_vpc.main) > 0
  
  # Provide fallback if VPC doesn't exist
  vpc_id = local.vpc_exists ? data.aws_vpc.main[0].id : null
  vpc_cidr = local.vpc_exists ? data.aws_vpc.main[0].cidr_block : "10.0.0.0/16"
}

# Conditional resource creation
resource "aws_subnet" "main" {
  count = local.vpc_exists ? 1 : 0
  
  vpc_id            = local.vpc_id
  cidr_block        = cidrsubnet(local.vpc_cidr, 8, 0)
  availability_zone = data.aws_availability_zones.available.names[0]
}
```

### Data Source Organization
```hcl
# Organize data sources by purpose
# Infrastructure data sources
data "aws_vpc" "main" {
  tags = { Name = "main-vpc" }
}

data "aws_subnet" "public" {
  vpc_id = data.aws_vpc.main.id
  
  filter {
    name   = "tag:Type"
    values = ["public"]
  }
}

# Configuration data sources
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
  
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"]
  }
}

# Identity data sources
data "aws_caller_identity" "current" {}
data "aws_region" "current" {}
data "aws_availability_zones" "available" {
  state = "available"
}
```

## 🌍 Real-World Data Source Scenarios

### Multi-Environment Infrastructure
```hcl
# Environment-specific data sources
variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

# Get environment-specific VPC
data "aws_vpc" "environment" {
  tags = {
    Name = "${var.environment}-vpc"
    Environment = var.environment
  }
}

# Get environment-specific subnets
data "aws_subnet" "private" {
  count = length(data.aws_availability_zones.available.names)
  
  vpc_id = data.aws_vpc.environment.id
  
  filter {
    name   = "tag:Type"
    values = ["private"]
  }
  
  filter {
    name   = "tag:Environment"
    values = [var.environment]
  }
}

# Create application using environment data
resource "aws_instance" "app" {
  count = var.environment == "production" ? 3 : 1
  
  ami           = data.aws_ami.ubuntu.id
  instance_type = var.environment == "production" ? "t2.large" : "t2.micro"
  subnet_id     = data.aws_subnet.private[count.index].id
  
  tags = {
    Name = "app-server-${count.index + 1}"
    Environment = var.environment
    VPC = data.aws_vpc.environment.id
  }
}
```

### Existing Infrastructure Integration
```hcl
# Get existing load balancer
data "aws_lb" "existing" {
  name = "existing-alb"
}

# Get existing target groups
data "aws_lb_target_group" "web" {
  name = "web-target-group"
  arn  = data.aws_lb.existing.arn
}

# Create new instances and attach to existing ALB
resource "aws_instance" "web" {
  count = 2
  
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  subnet_id     = data.aws_subnet.public[count.index].id
  
  tags = {
    Name = "web-server-${count.index + 1}"
  }
}

# Attach instances to existing target group
resource "aws_lb_target_group_attachment" "web" {
  count = length(aws_instance.web)
  
  target_group_arn = data.aws_lb_target_group.web.arn
  target_id        = aws_instance.web[count.index].id
  port             = 80
}
```

### Database and Storage Integration
```hcl
# Get existing RDS instance
data "aws_db_instance" "existing" {
  db_instance_identifier = "existing-database"
}

# Get existing S3 bucket
data "aws_s3_bucket" "data" {
  bucket = "existing-data-bucket"
}

# Create application using existing database
resource "aws_instance" "app" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  
  user_data = templatefile("${path.module}/user-data.sh", {
    db_endpoint = data.aws_db_instance.existing.endpoint
    db_name     = data.aws_db_instance.existing.db_name
    s3_bucket   = data.aws_s3_bucket.data.bucket
  })
  
  tags = {
    Name = "app-server"
    Database = data.aws_db_instance.existing.db_instance_identifier
    Storage = data.aws_s3_bucket.data.bucket
  }
}
```

### Security and Compliance
```hcl
# Get existing IAM roles
data "aws_iam_role" "ec2_role" {
  name = "EC2InstanceRole"
}

data "aws_iam_role" "lambda_role" {
  name = "LambdaExecutionRole"
}

# Get existing KMS key
data "aws_kms_key" "encryption" {
  key_id = "alias/encryption-key"
}

# Create encrypted resources using existing keys
resource "aws_instance" "encrypted" {
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t2.micro"
  
  iam_instance_profile = data.aws_iam_role.ec2_role.name
  
  root_block_device {
    volume_size = 20
    encrypted   = true
    kms_key_id  = data.aws_kms_key.encryption.arn
  }
  
  tags = {
    Name = "encrypted-instance"
    Encryption = data.aws_kms_key.encryption.key_id
  }
}
```

## 🔍 Advanced Data Source Patterns

### Dynamic Data Source Selection
```hcl
# Dynamic data source based on environment
locals {
  vpc_data_sources = {
    development = "dev-vpc"
    staging     = "staging-vpc"
    production  = "prod-vpc"
  }
  
  vpc_name = local.vpc_data_sources[var.environment]
}

data "aws_vpc" "environment" {
  tags = {
    Name = local.vpc_name
  }
}

# Use dynamic VPC data
resource "aws_subnet" "main" {
  vpc_id            = data.aws_vpc.environment.id
  cidr_block        = cidrsubnet(data.aws_vpc.environment.cidr_block, 8, 0)
  availability_zone = data.aws_availability_zones.available.names[0]
}
```

### Data Source Composition
```hcl
# Compose multiple data sources
data "aws_vpc" "main" {
  tags = { Name = "main-vpc" }
}

data "aws_subnet" "public" {
  count = length(data.aws_availability_zones.available.names)
  
  vpc_id = data.aws_vpc.main.id
  
  filter {
    name   = "tag:Type"
    values = ["public"]
  }
}

data "aws_security_group" "web" {
  name = "web-sg"
  vpc_id = data.aws_vpc.main.id
}

# Use composed data
locals {
  network_config = {
    vpc_id = data.aws_vpc.main.id
    vpc_cidr = data.aws_vpc.main.cidr_block
    subnets = data.aws_subnet.public[*].id
    security_groups = [data.aws_security_group.web.id]
  }
}
```

## 🔍 Common Commands

```bash
# Validate data sources
terraform validate

# Plan with data sources
terraform plan

# Show data source information
terraform show

# Refresh data sources
terraform refresh
```

## 📝 Key Takeaways
- Data sources enable reuse of existing infrastructure
- Use specific filters for better performance
- Implement proper error handling for data sources
- Organize data sources by purpose and functionality
- Leverage cross-resource references for complex scenarios
- Cache frequently used data in locals
- Validate data source results before use
- Use conditional data sources for flexible configurations

## 🚀 Next Steps
- Practice with different data source scenarios
- Learn about modules and their concepts
- Explore workspace management
- Understand backend configuration

---
*Ready for Day 14: Modules*