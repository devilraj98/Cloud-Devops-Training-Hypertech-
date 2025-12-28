# Day 34: Terraform Project - End-to-End Infrastructure Deployment

## Learning Objectives
- Implement end-to-end infrastructure deployment using Terraform
- Set up multi-environment infrastructure (dev, staging, prod)
- Perform infrastructure testing and validation
- Create reusable Terraform modules
- Implement cost optimization and resource tagging strategies

## Topics Covered

### 1. Complete Infrastructure Project Architecture
```
terraform-infrastructure-project/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── outputs.tf
│   ├── staging/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   ├── terraform.tfvars
│   │   └── outputs.tf
│   └── prod/
│       ├── main.tf
│       ├── variables.tf
│       ├── terraform.tfvars
│       └── outputs.tf
├── modules/
│   ├── vpc/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── shared/
│   ├── backend.tf
│   ├── versions.tf
│   └── locals.tf
├── scripts/
│   ├── deploy.sh
│   ├── destroy.sh
│   └── validate.sh
└── tests/
    ├── unit/
    └── integration/
```

### 2. VPC Module Implementation
```hcl
# modules/vpc/main.tf
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-vpc"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-igw"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)
  
  vpc_id                  = aws_vpc.main.id
  cidr_block              = var.public_subnet_cidrs[count.index]
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-public-subnet-${count.index + 1}"
    Type = "Public"
    Tier = "Web"
  })
}

# Private Subnets for Application
resource "aws_subnet" "private_app" {
  count = length(var.private_app_subnet_cidrs)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_app_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-private-app-subnet-${count.index + 1}"
    Type = "Private"
    Tier = "Application"
  })
}

# Private Subnets for Database
resource "aws_subnet" "private_db" {
  count = length(var.private_db_subnet_cidrs)
  
  vpc_id            = aws_vpc.main.id
  cidr_block        = var.private_db_subnet_cidrs[count.index]
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-private-db-subnet-${count.index + 1}"
    Type = "Private"
    Tier = "Database"
  })
}

# NAT Gateways
resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? length(aws_subnet.public) : 0
  
  domain = "vpc"
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-nat-eip-${count.index + 1}"
  })
  
  depends_on = [aws_internet_gateway.main]
}

resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? length(aws_subnet.public) : 0
  
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-nat-gateway-${count.index + 1}"
  })
}

# Route Tables
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-public-rt"
  })
}

resource "aws_route_table" "private_app" {
  count = var.enable_nat_gateway ? length(aws_nat_gateway.main) : 1
  
  vpc_id = aws_vpc.main.id
  
  dynamic "route" {
    for_each = var.enable_nat_gateway ? [1] : []
    content {
      cidr_block     = "0.0.0.0/0"
      nat_gateway_id = aws_nat_gateway.main[count.index].id
    }
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-private-app-rt-${count.index + 1}"
  })
}

resource "aws_route_table" "private_db" {
  vpc_id = aws_vpc.main.id
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-private-db-rt"
  })
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)
  
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private_app" {
  count = length(aws_subnet.private_app)
  
  subnet_id      = aws_subnet.private_app[count.index].id
  route_table_id = aws_route_table.private_app[count.index % length(aws_route_table.private_app)].id
}

resource "aws_route_table_association" "private_db" {
  count = length(aws_subnet.private_db)
  
  subnet_id      = aws_subnet.private_db[count.index].id
  route_table_id = aws_route_table.private_db.id
}

# VPC Flow Logs
resource "aws_flow_log" "vpc" {
  count = var.enable_flow_logs ? 1 : 0
  
  iam_role_arn    = aws_iam_role.flow_log[0].arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_log[0].arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-vpc-flow-logs"
  })
}

resource "aws_cloudwatch_log_group" "vpc_flow_log" {
  count = var.enable_flow_logs ? 1 : 0
  
  name              = "/aws/vpc/flowlogs/${var.environment}"
  retention_in_days = var.flow_log_retention_days
  
  tags = var.common_tags
}

resource "aws_iam_role" "flow_log" {
  count = var.enable_flow_logs ? 1 : 0
  
  name = "${var.environment}-vpc-flow-log-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
      }
    ]
  })
  
  tags = var.common_tags
}

resource "aws_iam_role_policy" "flow_log" {
  count = var.enable_flow_logs ? 1 : 0
  
  name = "${var.environment}-vpc-flow-log-policy"
  role = aws_iam_role.flow_log[0].id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Effect   = "Allow"
        Resource = "*"
      }
    ]
  })
}

data "aws_availability_zones" "available" {
  state = "available"
}
```

### 3. Compute Module with Auto Scaling
```hcl
# modules/compute/main.tf
# Launch Template
resource "aws_launch_template" "app" {
  name_prefix   = "${var.environment}-${var.application_name}-"
  image_id      = data.aws_ami.app.id
  instance_type = var.instance_type
  
  vpc_security_group_ids = [aws_security_group.app.id]
  key_name               = var.key_pair_name
  
  user_data = base64encode(templatefile("${path.module}/templates/user_data.sh", {
    environment        = var.environment
    application_name   = var.application_name
    database_endpoint  = var.database_endpoint
    redis_endpoint     = var.redis_endpoint
    s3_bucket         = var.s3_bucket_name
  }))
  
  iam_instance_profile {
    name = aws_iam_instance_profile.app.name
  }
  
  block_device_mappings {
    device_name = "/dev/xvda"
    ebs {
      volume_size           = var.root_volume_size
      volume_type           = "gp3"
      encrypted             = true
      delete_on_termination = true
    }
  }
  
  monitoring {
    enabled = true
  }
  
  tag_specifications {
    resource_type = "instance"
    tags = merge(var.common_tags, {
      Name = "${var.environment}-${var.application_name}-instance"
    })
  }
  
  tag_specifications {
    resource_type = "volume"
    tags = merge(var.common_tags, {
      Name = "${var.environment}-${var.application_name}-volume"
    })
  }
  
  lifecycle {
    create_before_destroy = true
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "app" {
  name                = "${var.environment}-${var.application_name}-asg"
  vpc_zone_identifier = var.private_subnet_ids
  target_group_arns   = [aws_lb_target_group.app.arn]
  health_check_type   = "ELB"
  health_check_grace_period = var.health_check_grace_period
  
  min_size         = var.min_size
  max_size         = var.max_size
  desired_capacity = var.desired_capacity
  
  launch_template {
    id      = aws_launch_template.app.id
    version = "$Latest"
  }
  
  # Auto Scaling Policies
  enabled_metrics = [
    "GroupMinSize",
    "GroupMaxSize",
    "GroupDesiredCapacity",
    "GroupInServiceInstances",
    "GroupTotalInstances"
  ]
  
  tag {
    key                 = "Name"
    value               = "${var.environment}-${var.application_name}-asg"
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
  
  instance_refresh {
    strategy = "Rolling"
    preferences {
      min_healthy_percentage = 50
    }
  }
}

# Auto Scaling Policies
resource "aws_autoscaling_policy" "scale_up" {
  name                   = "${var.environment}-${var.application_name}-scale-up"
  scaling_adjustment     = 2
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.app.name
}

resource "aws_autoscaling_policy" "scale_down" {
  name                   = "${var.environment}-${var.application_name}-scale-down"
  scaling_adjustment     = -1
  adjustment_type        = "ChangeInCapacity"
  cooldown               = 300
  autoscaling_group_name = aws_autoscaling_group.app.name
}

# CloudWatch Alarms
resource "aws_cloudwatch_metric_alarm" "cpu_high" {
  alarm_name          = "${var.environment}-${var.application_name}-cpu-high"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "300"
  statistic           = "Average"
  threshold           = "80"
  alarm_description   = "This metric monitors ec2 cpu utilization"
  alarm_actions       = [aws_autoscaling_policy.scale_up.arn]
  
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }
  
  tags = var.common_tags
}

resource "aws_cloudwatch_metric_alarm" "cpu_low" {
  alarm_name          = "${var.environment}-${var.application_name}-cpu-low"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "CPUUtilization"
  namespace           = "AWS/EC2"
  period              = "300"
  statistic           = "Average"
  threshold           = "20"
  alarm_description   = "This metric monitors ec2 cpu utilization"
  alarm_actions       = [aws_autoscaling_policy.scale_down.arn]
  
  dimensions = {
    AutoScalingGroupName = aws_autoscaling_group.app.name
  }
  
  tags = var.common_tags
}

# Application Load Balancer
resource "aws_lb" "app" {
  name               = "${var.environment}-${var.application_name}-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnet_ids
  
  enable_deletion_protection = var.environment == "production" ? true : false
  
  access_logs {
    bucket  = var.alb_logs_bucket
    prefix  = "${var.environment}-${var.application_name}"
    enabled = var.enable_alb_logs
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-${var.application_name}-alb"
  })
}

resource "aws_lb_target_group" "app" {
  name     = "${var.environment}-${var.application_name}-tg"
  port     = var.application_port
  protocol = "HTTP"
  vpc_id   = var.vpc_id
  
  health_check {
    enabled             = true
    healthy_threshold   = 2
    interval            = 30
    matcher             = "200"
    path                = var.health_check_path
    port                = "traffic-port"
    protocol            = "HTTP"
    timeout             = 5
    unhealthy_threshold = 2
  }
  
  stickiness {
    type            = "lb_cookie"
    cookie_duration = 86400
    enabled         = var.enable_stickiness
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-${var.application_name}-target-group"
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

# HTTPS Listener (if SSL certificate is provided)
resource "aws_lb_listener" "app_https" {
  count = var.ssl_certificate_arn != "" ? 1 : 0
  
  load_balancer_arn = aws_lb.app.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = var.ssl_certificate_arn
  
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# Security Groups
resource "aws_security_group" "alb" {
  name_prefix = "${var.environment}-${var.application_name}-alb-"
  vpc_id      = var.vpc_id
  
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-${var.application_name}-alb-sg"
  })
  
  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_security_group" "app" {
  name_prefix = "${var.environment}-${var.application_name}-app-"
  vpc_id      = var.vpc_id
  
  ingress {
    from_port       = var.application_port
    to_port         = var.application_port
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }
  
  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = merge(var.common_tags, {
    Name = "${var.environment}-${var.application_name}-app-sg"
  })
  
  lifecycle {
    create_before_destroy = true
  }
}

# IAM Role for EC2 instances
resource "aws_iam_role" "app" {
  name = "${var.environment}-${var.application_name}-ec2-role"
  
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      }
    ]
  })
  
  tags = var.common_tags
}

resource "aws_iam_role_policy" "app" {
  name = "${var.environment}-${var.application_name}-ec2-policy"
  role = aws_iam_role.app.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "${var.s3_bucket_arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "cloudwatch:PutMetricData",
          "ec2:DescribeVolumes",
          "ec2:DescribeTags",
          "logs:PutLogEvents",
          "logs:CreateLogGroup",
          "logs:CreateLogStream"
        ]
        Resource = "*"
      }
    ]
  })
}

resource "aws_iam_instance_profile" "app" {
  name = "${var.environment}-${var.application_name}-ec2-profile"
  role = aws_iam_role.app.name
  
  tags = var.common_tags
}

data "aws_ami" "app" {
  most_recent = true
  owners      = ["amazon"]
  
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
  
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

## Hands-on Labs

### Lab 1: Multi-Environment Deployment
```bash
#!/bin/bash
# scripts/deploy.sh

set -e

ENVIRONMENT=$1
ACTION=${2:-plan}

if [ -z "$ENVIRONMENT" ]; then
    echo "Usage: $0 <environment> [plan|apply|destroy]"
    echo "Environments: dev, staging, prod"
    exit 1
fi

if [ ! -d "environments/$ENVIRONMENT" ]; then
    echo "Environment $ENVIRONMENT does not exist"
    exit 1
fi

cd "environments/$ENVIRONMENT"

echo "🚀 Running Terraform $ACTION for $ENVIRONMENT environment..."

# Initialize Terraform
terraform init -upgrade

# Validate configuration
terraform validate

# Format code
terraform fmt -recursive

case $ACTION in
    "plan")
        terraform plan -out=tfplan
        ;;
    "apply")
        if [ "$ENVIRONMENT" = "prod" ]; then
            echo "⚠️  Production deployment requires manual approval"
            read -p "Are you sure you want to deploy to production? (yes/no): " confirm
            if [ "$confirm" != "yes" ]; then
                echo "Deployment cancelled"
                exit 0
            fi
        fi
        terraform apply tfplan
        ;;
    "destroy")
        echo "⚠️  This will destroy all resources in $ENVIRONMENT"
        read -p "Are you sure? (yes/no): " confirm
        if [ "$confirm" != "yes" ]; then
            echo "Destroy cancelled"
            exit 0
        fi
        terraform destroy -auto-approve
        ;;
    *)
        echo "Invalid action: $ACTION"
        echo "Valid actions: plan, apply, destroy"
        exit 1
        ;;
esac

echo "✅ Terraform $ACTION completed for $ENVIRONMENT"
```

### Lab 2: Infrastructure Testing
```go
// tests/integration/infrastructure_test.go
package test

import (
    "testing"
    "time"
    
    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
)

func TestInfrastructure(t *testing.T) {
    t.Parallel()
    
    // Configure Terraform options
    terraformOptions := &terraform.Options{
        TerraformDir: "../../environments/dev",
        Vars: map[string]interface{}{
            "environment": "test",
        },
    }
    
    // Clean up resources after test
    defer terraform.Destroy(t, terraformOptions)
    
    // Deploy infrastructure
    terraform.InitAndApply(t, terraformOptions)
    
    // Get outputs
    vpcId := terraform.Output(t, terraformOptions, "vpc_id")
    albDns := terraform.Output(t, terraformOptions, "load_balancer_dns")
    
    // Validate VPC exists
    aws.GetVpcById(t, vpcId, "us-west-2")
    
    // Test load balancer health
    maxRetries := 30
    timeBetweenRetries := 10 * time.Second
    
    url := "http://" + albDns + "/health"
    expectedStatus := 200
    
    aws.HttpGetWithRetryWithCustomValidation(
        t,
        url,
        nil,
        maxRetries,
        timeBetweenRetries,
        func(statusCode int, body string) bool {
            return statusCode == expectedStatus
        },
    )
}

func TestAutoScaling(t *testing.T) {
    t.Parallel()
    
    terraformOptions := &terraform.Options{
        TerraformDir: "../../environments/dev",
    }
    
    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)
    
    asgName := terraform.Output(t, terraformOptions, "autoscaling_group_name")
    region := "us-west-2"
    
    // Get Auto Scaling Group
    asg := aws.GetAsgByName(t, region, asgName)
    
    // Validate ASG configuration
    assert.Equal(t, int64(1), *asg.MinSize)
    assert.Equal(t, int64(3), *asg.MaxSize)
    assert.Equal(t, int64(2), *asg.DesiredCapacity)
}
```

### Lab 3: Cost Optimization Implementation
```hcl
# modules/cost-optimization/main.tf
# Spot Instance Configuration
resource "aws_launch_template" "spot" {
  count = var.enable_spot_instances ? 1 : 0
  
  name_prefix   = "${var.environment}-${var.application_name}-spot-"
  image_id      = data.aws_ami.app.id
  instance_type = var.spot_instance_type
  
  vpc_security_group_ids = [var.security_group_id]
  
  instance_market_options {
    market_type = "spot"
    spot_options {
      max_price = var.spot_max_price
    }
  }
  
  user_data = base64encode(var.user_data)
  
  tag_specifications {
    resource_type = "instance"
    tags = merge(var.common_tags, {
      Name = "${var.environment}-${var.application_name}-spot-instance"
      Type = "Spot"
    })
  }
}

# Mixed Instance Policy for ASG
resource "aws_autoscaling_group" "mixed" {
  name                = "${var.environment}-${var.application_name}-mixed-asg"
  vpc_zone_identifier = var.subnet_ids
  target_group_arns   = var.target_group_arns
  health_check_type   = "ELB"
  
  min_size         = var.min_size
  max_size         = var.max_size
  desired_capacity = var.desired_capacity
  
  mixed_instances_policy {
    instances_distribution {
      on_demand_base_capacity                  = var.on_demand_base_capacity
      on_demand_percentage_above_base_capacity = var.on_demand_percentage
      spot_allocation_strategy                 = "diversified"
    }
    
    launch_template {
      launch_template_specification {
        launch_template_id = var.launch_template_id
        version           = "$Latest"
      }
      
      dynamic "override" {
        for_each = var.instance_types
        content {
          instance_type = override.value
        }
      }
    }
  }
  
  tag {
    key                 = "Name"
    value               = "${var.environment}-${var.application_name}-mixed-asg"
    propagate_at_launch = false
  }
}

# Scheduled Scaling for Cost Optimization
resource "aws_autoscaling_schedule" "scale_down_evening" {
  count = var.enable_scheduled_scaling ? 1 : 0
  
  scheduled_action_name  = "${var.environment}-scale-down-evening"
  min_size               = var.evening_min_size
  max_size               = var.evening_max_size
  desired_capacity       = var.evening_desired_capacity
  recurrence             = "0 18 * * MON-FRI"  # 6 PM weekdays
  autoscaling_group_name = aws_autoscaling_group.mixed.name
}

resource "aws_autoscaling_schedule" "scale_up_morning" {
  count = var.enable_scheduled_scaling ? 1 : 0
  
  scheduled_action_name  = "${var.environment}-scale-up-morning"
  min_size               = var.min_size
  max_size               = var.max_size
  desired_capacity       = var.desired_capacity
  recurrence             = "0 8 * * MON-FRI"   # 8 AM weekdays
  autoscaling_group_name = aws_autoscaling_group.mixed.name
}

# Cost Allocation Tags
locals {
  cost_tags = {
    CostCenter    = var.cost_center
    Project       = var.project_name
    Owner         = var.owner
    Environment   = var.environment
    Application   = var.application_name
    BillingGroup  = var.billing_group
  }
}

# Resource tagging for cost tracking
resource "aws_default_tags" "cost_tracking" {
  tags = local.cost_tags
}
```

## Interview Questions

### Beginner Level
1. **Q: What are the key components of a well-structured Terraform project?**
   A: Modules for reusability, environment separation, remote state management, proper variable organization, outputs for inter-module communication, and comprehensive documentation.

2. **Q: How do you handle different configurations for multiple environments?**
   A: Use separate tfvars files, workspaces, or directory structures for each environment, with shared modules and environment-specific variable values.

3. **Q: What is the purpose of Terraform modules?**
   A: Modules provide reusability, encapsulation, organization, and consistency across environments while reducing code duplication.

### Intermediate Level
4. **Q: How do you implement infrastructure testing with Terraform?**
   A: Use tools like Terratest for integration testing, terraform validate for syntax checking, terraform plan for change validation, and policy as code tools for compliance testing.

5. **Q: Explain cost optimization strategies in Terraform infrastructure.**
   A: Use spot instances, right-sizing, scheduled scaling, resource tagging for cost allocation, lifecycle policies for storage, and monitoring with cost alerts.

6. **Q: How do you handle secrets and sensitive data in Terraform projects?**
   A: Use external secret management systems, environment variables, Terraform Cloud sensitive variables, or data sources to fetch secrets at runtime.

### Advanced Level
7. **Q: How would you implement a blue-green deployment strategy using Terraform?**
   A: Use conditional resource creation, separate target groups, traffic switching with load balancers, and automated rollback mechanisms based on health checks.

8. **Q: Describe a complete CI/CD pipeline for Terraform infrastructure.**
   A: Include version control triggers, automated testing, security scanning, approval workflows, deployment automation, monitoring integration, and rollback capabilities.

9. **Q: How do you handle Terraform state management in a large organization?**
   A: Implement state isolation, use remote backends with proper access controls, implement state locking, use Terraform Cloud/Enterprise, and establish backup procedures.

10. **Q: What strategies would you use for disaster recovery with Terraform?**
    A: Multi-region deployments, automated backups, infrastructure as code for quick recovery, health checks with failover routing, and regular disaster recovery testing.

## Key Takeaways
- Modular architecture enables reusability and maintainability
- Multi-environment setup requires careful planning and organization
- Infrastructure testing validates deployments and prevents issues
- Cost optimization strategies significantly reduce cloud expenses
- Proper tagging and resource organization improve management
- Security and compliance must be built into infrastructure code