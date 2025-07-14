 # Day 20: Terraform Testing and Validation

## Learning Objectives
- Understand Terraform testing strategies and approaches
- Learn validation concepts and best practices
- Master testing frameworks and tools
- Understand quality assurance practices
- Know how to implement comprehensive testing for Terraform code

## Core Concepts

### Why Test Terraform Code?

Testing Terraform code is essential because:
- **Infrastructure Reliability**: Ensure infrastructure works as expected
- **Change Safety**: Prevent breaking changes in production
- **Compliance**: Verify infrastructure meets requirements
- **Documentation**: Tests serve as living documentation
- **Confidence**: Build confidence in infrastructure changes

### Testing Pyramid for Infrastructure

```
    /\
   /  \     Manual Testing
  /____\    (Minimal)
 /      \
/________\   Integration Testing
|        |   (Some)
|________|   Unit Testing
|        |   (Most)
|________|
```

## Terraform Testing Strategies

### 1. Unit Testing

Unit tests validate individual components in isolation:

```hcl
# test/unit/variables_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestVariables(t *testing.T) {
  terraformOptions := terraform.Options{
    TerraformDir: "../",
    Vars: map[string]interface{}{
      "instance_type": "t2.micro",
      "environment":   "test",
    },
  }
  
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndPlan(t, terraformOptions)
}
```

### 2. Integration Testing

Integration tests validate how components work together:

```hcl
# test/integration/network_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/gruntwork-io/terratest/modules/aws"
)

func TestNetworkIntegration(t *testing.T) {
  terraformOptions := terraform.Options{
    TerraformDir: "../",
    Vars: map[string]interface{}{
      "environment": "test",
    },
  }
  
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)
  
  // Test VPC creation
  vpcId := terraform.Output(t, terraformOptions, "vpc_id")
  aws.AssertVpcExists(t, vpcId, "us-west-2")
  
  // Test subnet creation
  subnetId := terraform.Output(t, terraformOptions, "subnet_id")
  aws.AssertSubnetExists(t, subnetId, "us-west-2")
}
```

### 3. End-to-End Testing

E2E tests validate complete infrastructure workflows:

```hcl
# test/e2e/application_test.go
package test

import (
  "testing"
  "time"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/gruntwork-io/terratest/modules/http-helper"
)

func TestApplicationE2E(t *testing.T) {
  terraformOptions := terraform.Options{
    TerraformDir: "../",
    Vars: map[string]interface{}{
      "environment": "test",
    },
  }
  
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)
  
  // Test application endpoint
  publicIp := terraform.Output(t, terraformOptions, "public_ip")
  url := fmt.Sprintf("http://%s", publicIp)
  
  http_helper.HttpGetWithRetry(t, url, nil, 200, "Hello World", 30, 10*time.Second)
}
```

## Validation Concepts

### 1. Syntax Validation

```bash
# Validate Terraform syntax
terraform validate

# Format code
terraform fmt -check

# Check for unused variables
terraform plan -var-file="test.tfvars"
```

### 2. Configuration Validation

```hcl
# variables.tf
variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  
  validation {
    condition     = contains(["t2.micro", "t2.small", "t2.medium"], var.instance_type)
    error_message = "Instance type must be t2.micro, t2.small, or t2.medium."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}
```

### 3. Resource Validation

```hcl
# main.tf
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = var.instance_type
  
  tags = {
    Name        = "Web Server"
    Environment = var.environment
  }
  
  # Validate instance configuration
  lifecycle {
    precondition {
      condition     = self.instance_type != "t2.nano"
      error_message = "t2.nano instances are not allowed."
    }
  }
}
```

## Testing Frameworks

### 1. Terratest

Terratest is a Go library for testing Terraform code:

```go
// test/terratest_example_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/gruntwork-io/terratest/modules/aws"
)

func TestTerraformBasicExample(t *testing.T) {
  // Configure Terraform options
  terraformOptions := terraform.Options{
    TerraformDir: "../",
    Vars: map[string]interface{}{
      "instance_type": "t2.micro",
      "environment":   "test",
    },
  }
  
  // Clean up resources after test
  defer terraform.Destroy(t, terraformOptions)
  
  // Deploy infrastructure
  terraform.InitAndApply(t, terraformOptions)
  
  // Validate outputs
  publicIp := terraform.Output(t, terraformOptions, "public_ip")
  aws.AssertPublicIpExists(t, publicIp, "us-west-2")
}
```

### 2. Kitchen-Terraform

Kitchen-Terraform is a Ruby-based testing framework:

```ruby
# .kitchen.yml
---
driver:
  name: terraform
  root_module_directory: test/fixtures/kitchen-terraform-example

provisioner:
  name: terraform

platforms:
  - name: ubuntu

suites:
  - name: default
    verifier:
      name: terraform
      systems:
        - name: basic
          backend: local
          controls:
            - name: output_public_ip
              desc: "Output public_ip should exist"
              it: should exist
```

### 3. Terragrunt Testing

Terragrunt provides testing capabilities:

```hcl
# terragrunt.hcl
include "root" {
  path = find_in_parent_folders()
}

inputs = {
  environment = "test"
  instance_type = "t2.micro"
}

# test/terragrunt_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestTerragruntExample(t *testing.T) {
  terraformOptions := terraform.Options{
    TerraformDir: "../",
    Vars: map[string]interface{}{
      "environment": "test",
    },
  }
  
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)
}
```

## Quality Assurance Practices

### 1. Code Quality Checks

```yaml
# .github/workflows/terraform-quality.yml
name: 'Terraform Quality'

on:
  pull_request:
    branches: [ main ]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      
    - name: Terraform Format Check
      run: terraform fmt -check -recursive
      
    - name: Terraform Validate
      run: terraform validate
      
    - name: Run tflint
      uses: terraform-linters/setup-tflint@v2
      with:
        tflint_config_file: .tflint.hcl
      run: tflint
```

### 2. Security Scanning

```yaml
# .github/workflows/security-scan.yml
name: 'Security Scan'

on:
  pull_request:
    branches: [ main ]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Run tfsec
      uses: aquasecurity/tfsec-action@v0.0.1
      with:
        working_directory: terraform/
    
    - name: Run Checkov
      run: |
        pip install checkov
        checkov -d terraform/ --framework terraform
```

### 3. Cost Estimation

```yaml
# .github/workflows/cost-estimation.yml
name: 'Cost Estimation'

on:
  pull_request:
    branches: [ main ]

jobs:
  cost:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v1
      
    - name: Terraform Plan
      run: terraform plan -out=tfplan
      
    - name: Infracost
      uses: infracost/actions/comment@v2
      with:
        path: terraform/
        terraform_plan_file: tfplan
```

## Practical Examples

### Example 1: Basic Unit Test

```hcl
# test/unit/variables_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestInstanceTypeValidation(t *testing.T) {
  testCases := []struct {
    instanceType string
    shouldPass   bool
  }{
    {"t2.micro", true},
    {"t2.small", true},
    {"t2.medium", true},
    {"t2.nano", false},
    {"m5.large", false},
  }
  
  for _, tc := range testCases {
    terraformOptions := terraform.Options{
      TerraformDir: "../",
      Vars: map[string]interface{}{
        "instance_type": tc.instanceType,
        "environment":   "test",
      },
    }
    
    if tc.shouldPass {
      terraform.InitAndPlan(t, terraformOptions)
    } else {
      // Expect validation to fail
      _, err := terraform.InitAndPlanE(t, terraformOptions)
      if err == nil {
        t.Errorf("Expected validation to fail for instance type: %s", tc.instanceType)
      }
    }
  }
}
```

### Example 2: Integration Test

```hcl
# test/integration/network_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/gruntwork-io/terratest/modules/aws"
)

func TestNetworkIntegration(t *testing.T) {
  terraformOptions := terraform.Options{
    TerraformDir: "../",
    Vars: map[string]interface{}{
      "environment": "test",
      "vpc_cidr":   "10.0.0.0/16",
    },
  }
  
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)
  
  // Test VPC
  vpcId := terraform.Output(t, terraformOptions, "vpc_id")
  vpc := aws.GetVpcById(t, vpcId, "us-west-2")
  
  if vpc.CidrBlock != "10.0.0.0/16" {
    t.Errorf("Expected VPC CIDR to be 10.0.0.0/16, got %s", vpc.CidrBlock)
  }
  
  // Test subnets
  subnetIds := terraform.OutputList(t, terraformOptions, "subnet_ids")
  if len(subnetIds) != 2 {
    t.Errorf("Expected 2 subnets, got %d", len(subnetIds))
  }
}
```

### Example 3: End-to-End Test

```hcl
# test/e2e/application_test.go
package test

import (
  "testing"
  "time"
  "fmt"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/gruntwork-io/terratest/modules/http-helper"
)

func TestApplicationE2E(t *testing.T) {
  terraformOptions := terraform.Options{
    TerraformDir: "../",
    Vars: map[string]interface{}{
      "environment": "test",
    },
  }
  
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)
  
  // Get application URL
  publicIp := terraform.Output(t, terraformOptions, "public_ip")
  url := fmt.Sprintf("http://%s", publicIp)
  
  // Test application response
  http_helper.HttpGetWithRetry(t, url, nil, 200, "Hello World", 30, 10*time.Second)
  
  // Test application health
  healthUrl := fmt.Sprintf("%s/health", url)
  http_helper.HttpGetWithRetry(t, healthUrl, nil, 200, "healthy", 30, 10*time.Second)
}
```

## Advanced Testing Techniques

### 1. Parallel Testing

```go
// test/parallel_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestParallelEnvironments(t *testing.T) {
  environments := []string{"dev", "staging", "prod"}
  
  for _, env := range environments {
    env := env // Capture variable for closure
    t.Run(env, func(t *testing.T) {
      t.Parallel() // Run tests in parallel
      
      terraformOptions := terraform.Options{
        TerraformDir: "../",
        Vars: map[string]interface{}{
          "environment": env,
        },
      }
      
      defer terraform.Destroy(t, terraformOptions)
      terraform.InitAndApply(t, terraformOptions)
    })
  }
}
```

### 2. Performance Testing

```go
// test/performance_test.go
package test

import (
  "testing"
  "time"
  "github.com/gruntwork-io/terratest/modules/terraform"
)

func TestDeploymentPerformance(t *testing.T) {
  start := time.Now()
  
  terraformOptions := terraform.Options{
    TerraformDir: "../",
    Vars: map[string]interface{}{
      "environment": "test",
    },
  }
  
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)
  
  duration := time.Since(start)
  
  if duration > 5*time.Minute {
    t.Errorf("Deployment took too long: %v", duration)
  }
}
```

### 3. Chaos Testing

```go
// test/chaos_test.go
package test

import (
  "testing"
  "github.com/gruntwork-io/terratest/modules/terraform"
  "github.com/gruntwork-io/terratest/modules/aws"
)

func TestChaosResilience(t *testing.T) {
  terraformOptions := terraform.Options{
    TerraformDir: "../",
    Vars: map[string]interface{}{
      "environment": "test",
    },
  }
  
  defer terraform.Destroy(t, terraformOptions)
  terraform.InitAndApply(t, terraformOptions)
  
  // Simulate instance termination
  instanceId := terraform.Output(t, terraformOptions, "instance_id")
  aws.TerminateInstance(t, instanceId, "us-west-2")
  
  // Verify auto-scaling group replaces instance
  time.Sleep(2 * time.Minute)
  
  newInstanceId := terraform.Output(t, terraformOptions, "instance_id")
  if newInstanceId == instanceId {
    t.Error("Instance was not replaced after termination")
  }
}
```

## Common Commands

### Testing Commands

```bash
# Run unit tests
go test ./test/unit/...

# Run integration tests
go test ./test/integration/...

# Run all tests
go test ./test/...

# Run tests with coverage
go test -cover ./test/...

# Run tests in parallel
go test -parallel 4 ./test/...
```

### Validation Commands

```bash
# Validate Terraform
terraform validate

# Format check
terraform fmt -check -recursive

# Lint code
tflint

# Security scan
tfsec .

# Cost estimation
infracost breakdown --path .
```

## Troubleshooting

### Common Testing Issues

1. **Test Timeouts**
   ```bash
   # Increase test timeout
   go test -timeout 30m ./test/...
   
   # Use context with timeout
   ctx, cancel := context.WithTimeout(context.Background(), 10*time.Minute)
   defer cancel()
   ```

2. **Resource Cleanup**
   ```go
   // Ensure cleanup in defer
   defer terraform.Destroy(t, terraformOptions)
   
   // Use test cleanup
   t.Cleanup(func() {
     terraform.Destroy(t, terraformOptions)
   })
   ```

3. **Flaky Tests**
   ```go
   // Add retries for flaky operations
   http_helper.HttpGetWithRetry(t, url, nil, 200, "expected", 30, 10*time.Second)
   
   // Use polling for eventual consistency
   aws.WaitForS3Bucket(t, "us-west-2", bucketName)
   ```

### Debugging Tips

```bash
# Enable verbose output
go test -v ./test/...

# Run specific test
go test -run TestSpecificFunction ./test/...

# Show test coverage
go test -coverprofile=coverage.out ./test/...
go tool cover -html=coverage.out
```

## Summary

Testing Terraform code involves:
- **Unit Testing**: Validate individual components
- **Integration Testing**: Test component interactions
- **End-to-End Testing**: Validate complete workflows
- **Validation**: Syntax and configuration checks
- **Quality Assurance**: Code quality and security scanning

Key takeaways:
- Implement comprehensive testing strategy
- Use appropriate testing frameworks
- Validate configurations thoroughly
- Follow quality assurance practices
- Test in parallel when possible
- Clean up resources after tests

Remember: Good testing practices build confidence in infrastructure changes and prevent production issues.