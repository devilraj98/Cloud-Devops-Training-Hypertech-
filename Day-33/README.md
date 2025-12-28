# Day 33: Terraform Advanced - AWS Integration & Team Collaboration

## Learning Objectives
- Integrate Terraform with AWS services comprehensively
- Implement remote state management and team collaboration
- Configure Terraform Cloud and automation workflows
- Master Terraform workspaces and environments
- Set up state locking and backend configuration

## Topics Covered

### 1. Advanced AWS Integration
```hcl
# Complete AWS infrastructure setup
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.1"
    }
  }
}

# Multi-region provider setup
provider "aws" {
  alias  = "primary"
  region = var.primary_region
}

provider "aws" {
  alias  = "secondary"
  region = var.secondary_region
}

# Advanced VPC with multiple AZs
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-vpc"
  })
}

# Data source for availability zones
data "aws_availability_zones" "available" {
  state = "available"
}

# Public subnets across multiple AZs
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-public-subnet-${count.index + 1}"
    Type = "Public"
  })
}

# Private subnets
resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-private-subnet-${count.index + 1}"
    Type = "Private"
  })
}

# NAT Gateway with Elastic IP
resource "aws_eip" "nat" {
  count = length(aws_subnet.public)
  
  domain = "vpc"
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-nat-eip-${count.index + 1}"
  })
  
  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = length(aws_subnet.public)
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-nat-gateway-${count.index + 1}"
  })
}
```

### 2. Remote State Configuration
```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "my-terraform-state-bucket"
    key            = "environments/production/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    
    # Workspace-specific state
    workspace_key_prefix = "workspaces"
  }
}

# Create S3 bucket for state storage
resource "aws_s3_bucket" "terraform_state" {
  bucket = "my-terraform-state-bucket-${random_id.bucket_suffix.hex}"
  
  tags = {
    Name        = "Terraform State Bucket"
    Environment = "shared"
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# DynamoDB table for state locking
resource "aws_dynamodb_table" "terraform_state_lock" {
  name           = "terraform-state-lock"
  billing_mode   = "PAY_PER_REQUEST"
  hash_key       = "LockID"
  
  attribute {
    name = "LockID"
    type = "S"
  }
  
  tags = {
    Name        = "Terraform State Lock Table"
    Environment = "shared"
  }
}

resource "random_id" "bucket_suffix" {
  byte_length = 4
}
```

### 3. Terraform Workspaces
```bash
# Create and manage workspaces
terraform workspace new development
terraform workspace new staging
terraform workspace new production

# List workspaces
terraform workspace list

# Select workspace
terraform workspace select production

# Show current workspace
terraform workspace show

# Delete workspace
terraform workspace delete development
```

```hcl
# Workspace-specific configurations
locals {
  workspace_configs = {
    development = {
      instance_type = "t3.micro"
      min_size      = 1
      max_size      = 2
      environment   = "dev"
    }
    staging = {
      instance_type = "t3.small"
      min_size      = 2
      max_size      = 4
      environment   = "staging"
    }
    production = {
      instance_type = "t3.medium"
      min_size      = 3
      max_size      = 10
      environment   = "prod"
    }
  }
  
  current_config = local.workspace_configs[terraform.workspace]
}

resource "aws_instance" "app" {
  count = local.current_config.min_size
  
  ami           = data.aws_ami.amazon_linux.id
  instance_type = local.current_config.instance_type
  
  tags = {
    Name        = "${local.current_config.environment}-app-${count.index + 1}"
    Environment = local.current_config.environment
    Workspace   = terraform.workspace
  }
}
```

## Hands-on Labs

### Lab 1: Complete AWS Infrastructure with Auto Scaling
```hcl
# Auto Scaling Group with Launch Template
resource "aws_launch_template" "app" {
  name_prefix   = "${var.environment}-app-"
  image_id      = data.aws_ami.amazon_linux.id
  instance_type = var.instance_type
  
  vpc_security_group_ids = [aws_security_group.app.id]
  
  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    environment = var.environment
  }))
  
  tag_specifications {
    resource_type = "instance"
    tags = merge(var.common_tags, {
      Name = "${var.environment}-app-instance"
    })
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "app" {
  name                = "${var.environment}-app-asg"
  vpc_zone_identifier = aws_subnet.private[*].id
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"
  health_check_grace_period = 300
  
  min_size         = var.min_size
  max_size         = var.max_size
  desired_capacity = var.desired_capacity
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  tag {
    key                 = "Name"
    value               = "${var.environment}-app-asg"
    propagate_at_launch = false
  }
  
  dynamic "tag" {
    for_each = var.common_tags
    content {
      key                 = tag.key
      value               = tag.value
      propagate_at_launch = true
    }
  }
}

# Application Load Balancer
resource "aws_lb" "app" {
  name               = "${var.environment}-app-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id
  
  enable_deletion_protection = var.environment == "production" ? true : false
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-app-alb"
  })
}

resource "aws_lb_target_group" "app" {
  name     = "${var.environment}-app-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-app-target-group"
  })
}

resource "aws_lb_listener" "app" {
  load_balancer_arn = aws_lb.app.arn
  port              = "80"
  protocol          = "HTTP"
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}
```

### Lab 2: RDS with Multi-AZ and Read Replicas
```hcl
# RDS Subnet Group
resource "aws_db_subnet_group" "main" {
  name       = "${var.environment}-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-db-subnet-group"
  })
}

# RDS Parameter Group
resource "aws_db_parameter_group" "main" {
  family = "mysql8.0"
  name   = "${var.environment}-db-params"
  
  parameter {
    name  = "innodb_buffer_pool_size"
    value = "{DBInstanceClassMemory*3/4}"
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-db-parameter-group"
  })
}

# Primary RDS Instance
resource "aws_db_instance" "main" {
  identifier = "${var.environment}-database"
  
  engine         = "mysql"
  engine_version = "8.0"
  instance_class = var.db_instance_class
  
  allocated_storage     = var.db_allocated_storage
  max_allocated_storage = var.db_max_allocated_storage
  storage_type          = "gp2"
  storage_encrypted     = true
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  db_subnet_group_name   = aws_db_subnet_group.main.name
  parameter_group_name   = aws_db_parameter_group.main.name
  vpc_security_group_ids = [aws_security_group.database.id]
  
  backup_retention_period = var.environment == "production" ? 7 : 1
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  multi_az               = var.environment == "production" ? true : false
  publicly_accessible    = false
  skip_final_snapshot    = var.environment != "production"
  final_snapshot_identifier = var.environment == "production" ? "${var.environment}-db-final-snapshot-${formatdate("YYYY-MM-DD-hhmm", timestamp())}" : null
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-database"
  })
}

# Read Replica (for production)
resource "aws_db_instance" "read_replica" {
  count = var.environment == "production" ? 1 : 0
  
  identifier = "${var.environment}-database-read-replica"
  
  replicate_source_db = aws_db_instance.main.identifier
  instance_class      = var.db_instance_class
  
  publicly_accessible = false
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-database-read-replica"
  })
}
```

### Lab 3: Terraform Cloud Integration
```hcl
# terraform cloud configuration
terraform {
  cloud {
    organization = "my-organization"
    
    workspaces {
      name = "production-infrastructure"
    }
  }
}

# Variable sets in Terraform Cloud
variable "tfc_aws_provider_auth" {
  type = bool
  default = false
  description = "Enable AWS provider authentication via Terraform Cloud"
}

variable "tfc_aws_run_role_arn" {
  type = string
  default = ""
  description = "AWS IAM role ARN for Terraform Cloud"
}

# Conditional provider configuration
provider "aws" {
  region = var.aws_region
  
  dynamic "assume_role" {
    for_each = var.tfc_aws_provider_auth ? [1] : []
    content {
      role_arn = var.tfc_aws_run_role_arn
    }
  }
}
```

## Real-world Scenarios

### Scenario 1: Multi-Environment Pipeline
```yaml
# .github/workflows/terraform.yml
name: Terraform CI/CD
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        environment: [development, staging, production]
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Terraform
      uses: hashicorp/setup-terraform@v2
      with:
        terraform_version: 1.5.0
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-west-2
    
    - name: Terraform Init
      run: terraform init
      working-directory: environments/${{ matrix.environment }}
    
    - name: Terraform Plan
      run: terraform plan -out=tfplan
      working-directory: environments/${{ matrix.environment }}
    
    - name: Terraform Apply
      if: github.ref == 'refs/heads/main' && matrix.environment != 'production'
      run: terraform apply tfplan
      working-directory: environments/${{ matrix.environment }}
    
    - name: Production Approval
      if: github.ref == 'refs/heads/main' && matrix.environment == 'production'
      uses: trstringer/manual-approval@v1
      with:
        secret: ${{ github.TOKEN }}
        approvers: devops-team
```

### Scenario 2: Cross-Region Disaster Recovery
```hcl
# Primary region resources
module "primary_infrastructure" {
  source = "./modules/infrastructure"
  
  providers = {
    aws = aws.primary
  }
  
  region      = var.primary_region
  environment = var.environment
  is_primary  = true
  
  # Cross-region replication settings
  enable_cross_region_backup = true
  backup_destination_region  = var.secondary_region
}

# Secondary region resources
module "secondary_infrastructure" {
  source = "./modules/infrastructure"
  
  providers = {
    aws = aws.secondary
  }
  
  region      = var.secondary_region
  environment = "${var.environment}-dr"
  is_primary  = false
  
  # Reduced capacity for DR
  min_size         = 1
  max_size         = 3
  desired_capacity = 1
}

# Route 53 health checks and failover
resource "aws_route53_health_check" "primary" {
  fqdn                            = module.primary_infrastructure.load_balancer_dns
  port                            = 80
  type                            = "HTTP"
  resource_path                   = "/health"
  failure_threshold               = 3
  request_interval                = 30
  
  tags = {
    Name = "${var.environment}-primary-health-check"
  }
}

resource "aws_route53_record" "primary" {
  zone_id = var.route53_zone_id
  name    = var.domain_name
  type    = "A"
  
  set_identifier = "primary"
  
  failover_routing_policy {
    type = "PRIMARY"
  }
  
  health_check_id = aws_route53_health_check.primary.id
  
  alias {
    name                   = module.primary_infrastructure.load_balancer_dns
    zone_id                = module.primary_infrastructure.load_balancer_zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "secondary" {
  zone_id = var.route53_zone_id
  name    = var.domain_name
  type    = "A"
  
  set_identifier = "secondary"
  
  failover_routing_policy {
    type = "SECONDARY"
  }
  
  alias {
    name                   = module.secondary_infrastructure.load_balancer_dns
    zone_id                = module.secondary_infrastructure.load_balancer_zone_id
    evaluate_target_health = true
  }
}
```

## Interview Questions

### Beginner Level
1. **Q: What is the difference between Terraform workspaces and environments?**
   A: Workspaces are Terraform's way to manage multiple instances of the same configuration, while environments are logical separations (dev/staging/prod) that can be implemented using workspaces or separate state files.

2. **Q: How do you share Terraform state among team members?**
   A: Use remote backends like S3 with DynamoDB for locking, Terraform Cloud, or other supported backends to store state centrally and enable team collaboration.

3. **Q: What is the purpose of terraform.tfvars file?**
   A: It contains variable values for Terraform configurations, allowing you to customize deployments without modifying the main configuration files.

### Intermediate Level
4. **Q: How do you handle secrets in Terraform configurations?**
   A: Use environment variables, external secret management systems (AWS Secrets Manager, HashiCorp Vault), Terraform Cloud sensitive variables, or data sources to fetch secrets at runtime.

5. **Q: Explain Terraform's resource lifecycle and meta-arguments.**
   A: Meta-arguments like count, for_each, depends_on, lifecycle, and provider control resource behavior. Lifecycle includes create_before_destroy, prevent_destroy, and ignore_changes.

6. **Q: How do you implement blue-green deployments with Terraform?**
   A: Use conditional resource creation, separate target groups, traffic switching with load balancers, and workspace or variable-based environment switching.

### Advanced Level
7. **Q: How would you implement a multi-region, highly available architecture with Terraform?**
   A: Use multiple providers for different regions, implement cross-region replication, set up Route 53 health checks and failover routing, and use modules for consistent deployments.

8. **Q: Describe strategies for Terraform state management in large organizations.**
   A: Implement state isolation per environment/team, use remote backends with proper access controls, implement state locking, use Terraform Cloud/Enterprise for governance, and establish backup and recovery procedures.

9. **Q: How do you handle Terraform configuration drift and compliance?**
   A: Implement regular terraform plan runs, use policy as code tools (Sentinel, OPA), set up monitoring and alerting for infrastructure changes, and use drift detection tools.

10. **Q: Explain how you would implement a complete CI/CD pipeline for Terraform.**
    A: Use version control triggers, implement automated testing (terratest, policy validation), use approval workflows for production, implement rollback strategies, and integrate with monitoring and alerting systems.

## Key Takeaways
- Remote state management is essential for team collaboration
- Workspaces enable environment separation within single configuration
- Terraform Cloud provides enterprise features and governance
- State locking prevents concurrent modifications and corruption
- Multi-region deployments require careful provider and state management
- CI/CD integration automates infrastructure deployment and validation