# Day 16: Terraform Provisioners

## Learning Objectives
- Understand what provisioners are and when to use them
- Learn about local vs remote provisioners
- Master file provisioners for configuration management
- Understand provisioner best practices and limitations
- Know when to use provisioners vs other approaches

## Core Concepts

### What are Provisioners?

Provisioners are a way to execute scripts or commands on local or remote machines after resource creation. They allow you to:
- Install software on newly created resources
- Configure applications
- Run initialization scripts
- Perform post-deployment tasks

**Important**: Provisioners should be used as a last resort. Terraform's philosophy is to use declarative configuration over imperative scripts.

### Types of Provisioners

#### 1. Local Provisioners
Execute commands on the machine running Terraform:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  provisioner "local-exec" {
    command = "echo 'Instance created with ID: ${self.id}'"
  }
}
```

#### 2. Remote Provisioners
Execute commands on the created resource:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl start nginx"
    ]
  }
}
```

### File Provisioners

File provisioners copy files from the Terraform machine to the created resource:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"

  # Copy configuration file
  provisioner "file" {
    source      = "config/nginx.conf"
    destination = "/tmp/nginx.conf"
  }

  # Copy directory
  provisioner "file" {
    source      = "scripts/"
    destination = "/tmp/scripts/"
  }
}
```

## Practical Examples

### Example 1: Basic Local Provisioner

```hcl
# main.tf
resource "aws_instance" "example" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  tags = {
    Name = "Example Instance"
  }

  provisioner "local-exec" {
    command = "echo 'Instance ${self.id} created at $(date)' >> instance_log.txt"
  }
}
```

### Example 2: Remote Provisioner with Connection

```hcl
# main.tf
resource "aws_instance" "web_server" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name

  vpc_security_group_ids = [aws_security_group.web.id]

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx",
      "sudo systemctl enable nginx",
      "sudo systemctl start nginx"
    ]
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }
}

resource "aws_key_pair" "deployer" {
  key_name   = "deployer-key"
  public_key = file("~/.ssh/id_rsa.pub")
}

resource "aws_security_group" "web" {
  name_prefix = "web-sg"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

### Example 3: File Provisioner with Scripts

```hcl
# main.tf
resource "aws_instance" "app_server" {
  ami           = "ami-12345678"
  instance_type = "t2.micro"
  key_name      = aws_key_pair.deployer.key_name

  vpc_security_group_ids = [aws_security_group.app.id]

  # Copy application files
  provisioner "file" {
    source      = "app/"
    destination = "/home/ubuntu/app"
  }

  # Copy configuration
  provisioner "file" {
    content     = templatefile("config/app.conf.tpl", {
      db_host = aws_db_instance.main.endpoint
      db_name = aws_db_instance.main.db_name
    })
    destination = "/home/ubuntu/app/config.conf"
  }

  # Run setup script
  provisioner "remote-exec" {
    inline = [
      "chmod +x /home/ubuntu/app/setup.sh",
      "/home/ubuntu/app/setup.sh"
    ]
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
  }
}
```

### Example 4: Null Resource with Provisioners

```hcl
# main.tf
resource "null_resource" "database_setup" {
  triggers = {
    db_instance_id = aws_db_instance.main.id
  }

  provisioner "local-exec" {
    command = "echo 'Database ${aws_db_instance.main.id} is ready'"
  }

  provisioner "local-exec" {
    when    = destroy
    command = "echo 'Database ${aws_db_instance.main.id} is being destroyed'"
  }
}
```

## Provisioner Best Practices

### 1. Use Provisioners Sparingly
- Prefer declarative configuration over imperative scripts
- Use user data scripts for EC2 instances when possible
- Consider using configuration management tools (Ansible, Chef, Puppet)

### 2. Handle Failures Gracefully
```hcl
resource "aws_instance" "example" {
  # ... configuration ...

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]

    on_failure = continue  # Continue even if provisioner fails
  }
}
```

### 3. Use Triggers for Conditional Execution
```hcl
resource "null_resource" "example" {
  triggers = {
    instance_id = aws_instance.web.id
    timestamp   = timestamp()
  }

  provisioner "local-exec" {
    command = "echo 'Instance updated'"
  }
}
```

### 4. Proper Connection Configuration
```hcl
resource "aws_instance" "example" {
  # ... configuration ...

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
    timeout     = "5m"
    agent       = false
  }
}
```

## Common Commands

### Basic Commands
```bash
# Apply configuration with provisioners
terraform apply

# Plan to see provisioner actions
terraform plan

# Destroy resources (triggers destroy provisioners)
terraform destroy
```

### Debugging Provisioners
```bash
# Run with detailed logging
terraform apply -var-file="variables.tfvars"

# Check provisioner logs
terraform console
```

## Advanced Topics

### 1. Self-Signed Certificates
```hcl
resource "aws_instance" "example" {
  # ... configuration ...

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = self.public_ip
    target_host_key_fingerprint = "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQ..."
  }
}
```

### 2. Windows Remote Provisioners
```hcl
resource "aws_instance" "windows" {
  ami           = "ami-windows-123"
  instance_type = "t2.micro"

  provisioner "remote-exec" {
    inline = [
      "powershell -Command \"Install-WindowsFeature -Name Web-Server\"",
      "powershell -Command \"Start-Service W3SVC\""
    ]
  }

  connection {
    type     = "winrm"
    user     = "Administrator"
    password = rsadecrypt(self.password_data, file("~/.ssh/id_rsa"))
    host     = self.public_ip
  }
}
```

### 3. Provisioner Dependencies
```hcl
resource "aws_instance" "web" {
  # ... configuration ...
}

resource "null_resource" "web_setup" {
  depends_on = [aws_instance.web]

  provisioner "remote-exec" {
    inline = [
      "sudo apt-get update",
      "sudo apt-get install -y nginx"
    ]
  }

  connection {
    type        = "ssh"
    user        = "ubuntu"
    private_key = file("~/.ssh/id_rsa")
    host        = aws_instance.web.public_ip
  }
}
```

## Troubleshooting

### Common Issues

1. **Connection Timeouts**
   - Check security groups
   - Verify SSH key permissions
   - Ensure instance is fully booted

2. **Permission Errors**
   - Use `sudo` for system commands
   - Check file permissions
   - Verify user has appropriate access

3. **Script Failures**
   - Add error handling to scripts
   - Use `set -e` in bash scripts
   - Test scripts locally first

### Debugging Tips

```hcl
# Add debugging to provisioners
provisioner "remote-exec" {
  inline = [
    "set -x",  # Enable debug mode
    "echo 'Starting installation...'",
    "sudo apt-get update",
    "echo 'Installation complete'"
  ]
}
```

## Summary

Provisioners are powerful tools for post-deployment configuration, but they should be used judiciously. Always consider:
- Whether declarative configuration is possible
- Using user data scripts for EC2 instances
- Implementing proper error handling
- Following security best practices
- Testing provisioners thoroughly before production use

Remember: Provisioners are a last resort when other Terraform-native approaches aren't sufficient for your use case.