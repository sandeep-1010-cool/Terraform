# Installation Guide

## 🛠️ Prerequisites
- Windows 10/11 or Linux (Ubuntu/Debian/CentOS/RHEL)
- Internet connection for downloads
- Administrative privileges

---

## 📦 Terraform Installation

### Windows
```bash
# Method 1: Using Chocolatey (Recommended)
choco install terraform

# Method 2: Manual Installation
# 1. Download from https://www.terraform.io/downloads
# 2. Extract to C:\terraform
# 3. Add C:\terraform to PATH environment variable

# Method 3: Using winget
winget install HashiCorp.Terraform
```

### Linux (Ubuntu/Debian)
```bash
# Add HashiCorp GPG key
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Add HashiCorp repository
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# Update and install
sudo apt update && sudo apt install terraform
```

### Linux (CentOS/RHEL/Amazon Linux)
```bash
# Install using yum
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/RHEL/hashicorp.repo
sudo yum -y install terraform
```

### Verification
```bash
terraform --version
```

---

## ☁️ AWS CLI Installation

### Windows
```bash
# Method 1: Using MSI Installer
# Download from: https://awscli.amazonaws.com/AWSCLIV2.msi
# Run the installer and follow prompts

# Method 2: Using Chocolatey
choco install awscli

# Method 3: Using winget
winget install Amazon.AWSCLI
```

### Linux (Ubuntu/Debian)
```bash
# Update package list
sudo apt update

# Install AWS CLI
sudo apt install awscli

# Or install latest version
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### Linux (CentOS/RHEL/Amazon Linux)
```bash
# Using yum
sudo yum install awscli

# Or install latest version
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

### AWS CLI Configuration
```bash
# Configure AWS credentials
aws configure

# Enter your:
# - AWS Access Key ID
# - AWS Secret Access Key
# - Default region (e.g., us-east-1)
# - Default output format (json)
```

### Verification
```bash
aws --version
aws sts get-caller-identity
```

---

## 🔷 Azure CLI Installation

### Windows
```bash
# Method 1: Using MSI Installer
# Download from: https://aka.ms/installazurecliwindows
# Run the installer and follow prompts

# Method 2: Using Chocolatey
choco install azure-cli

# Method 3: Using winget
winget install Microsoft.AzureCLI
```

### Linux (Ubuntu/Debian)
```bash
# Add Microsoft signing key
curl -sL https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/microsoft.gpg > /dev/null

# Add Azure CLI repository
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/azure-cli.list

# Update and install
sudo apt update && sudo apt install azure-cli
```

### Linux (CentOS/RHEL/Amazon Linux)
```bash
# Add Microsoft repository
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc

# Add Azure CLI repository
echo -e "[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc" | sudo tee /etc/yum.repos.d/azure-cli.repo

# Install Azure CLI
sudo yum install azure-cli
```

### Azure CLI Login
```bash
# Login to Azure
az login

# Set subscription (if you have multiple)
az account set --subscription "Your-Subscription-Name"
```

### Verification
```bash
az --version
az account show
```

---

## 🔧 Post-Installation Setup

### 1. Verify All Tools
```bash
# Check Terraform
terraform --version

# Check AWS CLI
aws --version

# Check Azure CLI
az --version
```

### 2. Configure Cloud Providers

#### AWS Configuration
```bash
# Create AWS credentials file
mkdir ~/.aws
nano ~/.aws/credentials

# Add your credentials:
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
```

#### Azure Configuration
```bash
# Login and set subscription
az login
az account set --subscription "Your-Subscription-Name"
```

### 3. Test Connectivity
```bash
# Test AWS
aws sts get-caller-identity

# Test Azure
az account show
```

---

## 🚀 Quick Start Commands

### Terraform
```bash
# Initialize a new Terraform project
terraform init

# Plan your infrastructure
terraform plan

# Apply changes
terraform apply
```

### AWS
```bash
# List S3 buckets
aws s3 ls

# List EC2 instances
aws ec2 describe-instances
```

### Azure
```bash
# List resource groups
az group list

# List virtual machines
az vm list
```

---

## 📝 Troubleshooting

### Common Issues

1. **Terraform not found**
   - Check PATH environment variable
   - Restart terminal/command prompt

2. **AWS CLI authentication errors**
   - Verify credentials in `~/.aws/credentials`
   - Check IAM permissions

3. **Azure CLI login issues**
   - Clear cached credentials: `az account clear`
   - Re-login: `az login`

### Environment Variables
```bash
# Set AWS region
export AWS_DEFAULT_REGION=us-east-1

# Set Azure subscription
export AZURE_SUBSCRIPTION_ID=your-subscription-id
```

---

## ✅ Installation Checklist

- [ ] Terraform installed and verified
- [ ] AWS CLI installed and configured
- [ ] Azure CLI installed and logged in
- [ ] All tools responding to version commands
- [ ] Cloud provider connectivity tested

---

*Ready to start your Infrastructure as Code journey! 🚀*
