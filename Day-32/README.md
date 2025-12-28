# Day 32: Infrastructure as Code - Terraform Fundamentals

## Learning Objectives
- Understand Terraform basics and Infrastructure as Code principles
- Learn resource provisioning and state management
- Implement Terraform modules and best practices
- Master HCL syntax and configuration files
- Work with Terraform providers and resources

## Topics Covered

### 1. Terraform Fundamentals
- **Infrastructure as Code (IaC)**: Declarative infrastructure management
- **Terraform Architecture**: Providers, resources, state, and execution
- **HCL Syntax**: HashiCorp Configuration Language
- **Terraform Workflow**: Init, Plan, Apply, Destroy

### 2. Core Terraform Concepts
```hcl
# Provider configuration
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# Variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
}

# Data sources
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

# Resources
resource "aws_instance" "web_server" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  
  tags = {
    Name        = "WebServer"
    Environment = "Development"
  }
}

# Outputs
output "instance_id" {
  description = "ID of the EC2 instance"
  value       = aws_instance.web_server.id
}

output "public_ip" {
  description = "Public IP address of the EC2 instance"
  value       = aws_instance.web_server.public_ip
}
```

### 3. State Management
```hcl
# Local state (default)
# terraform.tfstate file in current directory

# Remote state configuration
terraform {
  backend "s3" {
    bucket = "my-terraform-state-bucket"
    key    = "infrastructure/terraform.tfstate"
    region = "us-west-2"
    
    # State locking
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}
```

## Hands-on Labs

### Lab 1: Basic AWS Infrastructure Setup
```hcl
# main.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "us-west-2"
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "main-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "main-igw"
  }
}

# Public Subnet
resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-west-2a"
  map_public_ip_on_launch = true
  
  tags = {
    Name = "public-subnet"
  }
}

# Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "public-rt"
  }
}

# Route Table Association
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# Security Group
resource "aws_security_group" "web" {
  name_prefix = "web-sg"
  vpc_id      = aws_vpc.main.id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name = "web-security-group"
  }
}

# EC2 Instance
resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]
  
  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html
              EOF
  
  tags = {
    Name = "web-server"
  }
}

data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}
```

### Lab 2: Variables and Outputs
```hcl
# variables.tf
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.micro"
  
  validation {
    condition = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "Instance type must be t3.micro, t3.small, or t3.medium."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "dev"
}

variable "tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    Project = "terraform-demo"
    Owner   = "devops-team"
  }
}

# terraform.tfvars
region        = "us-west-2"
instance_type = "t3.micro"
environment   = "development"
tags = {
  Project     = "web-application"
  Owner       = "devops-team"
  Environment = "development"
}

# outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "instance_public_ip" {
  description = "Public IP address of the instance"
  value       = aws_instance.web.public_ip
}

output "instance_dns" {
  description = "Public DNS name of the instance"
  value       = aws_instance.web.public_dns
}

output "security_group_id" {
  description = "ID of the security group"
  value       = aws_security_group.web.id
}
```

### Lab 3: Terraform Commands and Workflow
```bash
# 1. Initialize Terraform
terraform init

# 2. Validate configuration
terraform validate

# 3. Format code
terraform fmt

# 4. Plan infrastructure changes
terraform plan

# 5. Apply changes
terraform apply

# 6. Show current state
terraform show

# 7. List resources
terraform state list

# 8. Get specific resource info
terraform state show aws_instance.web

# 9. Destroy infrastructure
terraform destroy

# 10. Import existing resources
terraform import aws_instance.web i-1234567890abcdef0
```

## Real-world Scenarios

### Scenario 1: Multi-Environment Setup
```hcl
# environments/dev/main.tf
module "infrastructure" {
  source = "../../modules/infrastructure"
  
  environment   = "dev"
  instance_type = "t3.micro"
  min_size      = 1
  max_size      = 2
  
  tags = {
    Environment = "development"
    Project     = "web-app"
  }
}

# environments/prod/main.tf
module "infrastructure" {
  source = "../../modules/infrastructure"
  
  environment   = "prod"
  instance_type = "t3.medium"
  min_size      = 2
  max_size      = 10
  
  tags = {
    Environment = "production"
    Project     = "web-app"
  }
}
```

### Scenario 2: Resource Dependencies
```hcl
# Database subnet group
resource "aws_db_subnet_group" "main" {
  name       = "${var.environment}-db-subnet-group"
  subnet_ids = [aws_subnet.private_a.id, aws_subnet.private_b.id]
  
  tags = {
    Name = "${var.environment}-db-subnet-group"
  }
}

# RDS instance depends on subnet group
resource "aws_db_instance" "main" {
  identifier = "${var.environment}-database"
  
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.micro"
  
  allocated_storage = 20
  storage_type      = "gp2"
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]
  
  skip_final_snapshot = true
  
  depends_on = [aws_db_subnet_group.main]
  
  tags = {
    Name = "${var.environment}-database"
  }
}
```

## Best Practices

### 1. Code Organization
```
terraform-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── terraform.tfvars
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       └── terraform.tfvars
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── ec2/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── shared/
    ├── backend.tf
    └── versions.tf
```

### 2. Naming Conventions
```hcl
# Resource naming
resource "aws_instance" "web_server" {
  # Use descriptive names
  tags = {
    Name = "${var.environment}-web-server-${count.index + 1}"
  }
}

# Variable naming
variable "vpc_cidr_block" {
  description = "CIDR block for VPC"
  type        = string
}
```

## Troubleshooting Guide

### Common Issues and Solutions

1. **State Lock Issues**
```bash
# Force unlock (use with caution)
terraform force-unlock LOCK_ID

# Check state lock table
aws dynamodb scan --table-name terraform-state-lock
```

2. **Provider Version Conflicts**
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

3. **Resource Import Issues**
```bash
# Import existing resource
terraform import aws_instance.web i-1234567890abcdef0

# Generate configuration from import
terraform show -no-color > imported.tf
```

## Interview Questions

### Beginner Level
1. **Q: What is Infrastructure as Code (IaC)?**
   A: IaC is the practice of managing and provisioning infrastructure through machine-readable definition files, rather than physical hardware configuration or interactive configuration tools.

2. **Q: What are the main components of Terraform?**
   A: Providers (interface to APIs), Resources (infrastructure objects), Data Sources (read-only information), Variables (input parameters), and Outputs (return values).

3. **Q: What is Terraform state and why is it important?**
   A: Terraform state is a file that maps real-world resources to configuration, tracks metadata, and improves performance by caching resource attributes.

### Intermediate Level
4. **Q: Explain the difference between terraform plan and terraform apply.**
   A: `terraform plan` creates an execution plan showing what actions Terraform will take, while `terraform apply` executes the planned actions to reach the desired state.

5. **Q: How do you manage sensitive data in Terraform?**
   A: Use sensitive variables, environment variables, external secret management systems, or Terraform Cloud/Enterprise for secure variable storage.

6. **Q: What are Terraform modules and their benefits?**
   A: Modules are containers for multiple resources used together. Benefits include reusability, organization, encapsulation, and consistency across environments.

### Advanced Level
7. **Q: How do you handle Terraform state in a team environment?**
   A: Use remote state backends (S3, Terraform Cloud), implement state locking with DynamoDB, use workspaces for environment separation, and follow GitOps practices.

8. **Q: Explain Terraform's dependency resolution.**
   A: Terraform builds a dependency graph based on resource references, implicit dependencies (resource attributes), and explicit dependencies (depends_on), then executes in the correct order.

9. **Q: How would you implement blue-green deployments with Terraform?**
   A: Use workspaces or separate state files for each environment, implement traffic switching with load balancers, and use conditional resource creation based on variables.

10. **Q: What strategies would you use for Terraform code testing?**
    A: Unit testing with tools like Terratest, integration testing in isolated environments, policy as code with Sentinel/OPA, and automated validation in CI/CD pipelines.

## Key Takeaways
- Terraform enables declarative infrastructure management
- State management is crucial for team collaboration
- Modules promote code reusability and organization
- Remote backends provide state sharing and locking
- Proper planning and validation prevent infrastructure issues
- Version control and CI/CD integration ensure reliable deployments