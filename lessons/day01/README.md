# Day 1: Introduction to Terraform

## 🎯 Learning Objectives
- Understand Infrastructure as Code (IaC) concepts
- Learn why Terraform is essential for modern infrastructure
- Master the Terraform workflow
- Set up Terraform development environment

## 📚 Core Concepts

### Infrastructure as Code (IaC)
- **Definition**: Managing infrastructure through code instead of manual processes
- **Benefits**:
  - Version control for infrastructure
  - Consistent deployments
  - Reduced human error
  - Faster provisioning
  - Cost optimization

### Why Terraform?
- **Multi-cloud support**: AWS, Azure, GCP, and 100+ providers
- **Declarative syntax**: Define desired state, Terraform handles the rest
- **State management**: Tracks resource changes and dependencies
- **Community-driven**: Extensive provider ecosystem

### Traditional vs IaC Approach

| Traditional | Infrastructure as Code |
|-------------|----------------------|
| Manual configuration | Automated provisioning |
| Inconsistent environments | Reproducible deployments |
| No version control | Git-tracked changes |
| Slow deployments | Rapid scaling |
| Error-prone | Reduced human error |

## 🔄 Terraform Workflow

```
Write → Plan → Apply → Destroy
```

1. **Write**: Create `.tf` files with resource definitions
2. **Plan**: Preview changes before applying (`terraform plan`)
3. **Apply**: Execute the configuration (`terraform apply`)
4. **Destroy**: Clean up resources when done (`terraform destroy`)

## 🛠️ Installation

### Windows
```bash
# Using Chocolatey
choco install terraform

# Manual installation
# Download from https://www.terraform.io/downloads
```

### macOS
```bash
# Using Homebrew
brew install terraform
```

### Linux
```bash
# Using package manager
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo apt-key add -
sudo apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
sudo apt-get update && sudo apt-get install terraform
```

## ✅ Verification
```bash
terraform --version
```

## 📝 Key Takeaways
- IaC provides consistency, speed, and reliability
- Terraform is declarative and provider-agnostic
- Always plan before applying changes
- Version control your Terraform code

## 🚀 Next Steps
- Set up your development environment
- Create your first Terraform configuration
- Learn about providers and resources

---
*Ready for Day 2: Terraform Basics and First Configuration* 