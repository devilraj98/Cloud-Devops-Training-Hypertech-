# Day 16: EC2 Advanced Features - Auto Scaling and Load Balancing

## 📚 Learning Objectives
- Master Auto Scaling Groups (ASG) configuration
- Implement Application Load Balancer (ALB) and Network Load Balancer (NLB)
- Configure scaling policies and health checks
- Understand load balancer types and use cases
- Practice advanced EC2 networking configurations

## 🏗️ Architecture Overview

```
Internet
    |
Application Load Balancer
    |
┌───────────────────────────────┐
│        Auto Scaling Group     │
│  ┌─────────┐  ┌─────────┐    │
│  │  EC2-1  │  │  EC2-2  │    │
│  │  AZ-1a  │  │  AZ-1b  │    │
│  └─────────┘  └─────────┘    │
└───────────────────────────────┘
```

## 🔄 Auto Scaling Groups Deep Dive

### What is Auto Scaling?
Auto Scaling automatically adjusts the number of EC2 instances based on demand, ensuring optimal performance and cost efficiency.

### Key Components
- **Launch Template/Configuration**: Instance blueprint
- **Auto Scaling Group**: Manages instance lifecycle
- **Scaling Policies**: Rules for scaling up/down
- **Health Checks**: Monitor instance health

## 🛠️ Hands-on Implementation

### Step 1: Create Launch Template

#### 1.1 Launch Template Configuration
```bash
# Navigate to EC2 Console
EC2 Console → Launch Templates → Create Launch Template

# Configuration:
Launch template name: WebServer-Template-v1
Template version description: Initial web server template
AMI: Amazon Linux 2023
Instance type: t3.micro
Key pair: your-key-pair
Security groups: WebServer-SG
```

#### 1.2 Advanced Configuration
```bash
# Storage:
Volume type: gp3
Size: 8 GB
Delete on termination: Yes

# Network settings:
Subnet: Don't include in template (ASG will handle)
Auto-assign public IP: Enable

# Advanced details:
IAM instance profile: EC2-CloudWatch-Role (if needed)
Monitoring: Enable detailed monitoring
```

#### 1.3 User Data Script
```bash
#!/bin/bash
yum update -y
yum install -y httpd stress

# Start and enable Apache
systemctl start httpd
systemctl enable httpd

# Create dynamic web page
cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Auto Scaling Demo</title>
    <style>
        body { font-family: Arial; margin: 40px; background: #f0f0f0; }
        .container { background: white; padding: 20px; border-radius: 10px; }
        .info { background: #e7f3ff; padding: 10px; margin: 10px 0; border-radius: 5px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Auto Scaling Web Server</h1>
        <div class="info">
            <strong>Instance ID:</strong> <span id="instance-id">Loading...</span><br>
            <strong>Availability Zone:</strong> <span id="az">Loading...</span><br>
            <strong>Private IP:</strong> <span id="private-ip">Loading...</span><br>
            <strong>Launch Time:</strong> <span id="launch-time">Loading...</span>
        </div>
        <p>This server is part of an Auto Scaling Group!</p>
        <button onclick="loadTest()">Generate Load (CPU Test)</button>
        <div id="status"></div>
    </div>

    <script>
        // Fetch instance metadata
        fetch('/latest/meta-data/instance-id')
            .then(r => r.text())
            .then(data => document.getElementById('instance-id').textContent = data);
        
        fetch('/latest/meta-data/placement/availability-zone')
            .then(r => r.text())
            .then(data => document.getElementById('az').textContent = data);
            
        fetch('/latest/meta-data/local-ipv4')
            .then(r => r.text())
            .then(data => document.getElementById('private-ip').textContent = data);
            
        document.getElementById('launch-time').textContent = new Date().toLocaleString();
        
        function loadTest() {
            document.getElementById('status').innerHTML = '<p style="color: orange;">Generating CPU load for 2 minutes...</p>';
            fetch('/cgi-bin/load-test.sh');
        }
    </script>
</body>
</html>
EOF

# Create load test script
mkdir -p /var/www/cgi-bin
cat > /var/www/cgi-bin/load-test.sh << 'EOF'
#!/bin/bash
echo "Content-Type: text/plain"
echo ""
echo "Starting CPU load test..."
stress --cpu 2 --timeout 120s &
echo "Load test started for 2 minutes"
EOF

chmod +x /var/www/cgi-bin/load-test.sh

# Configure Apache for CGI
echo "LoadModule cgi_module modules/mod_cgi.so" >> /etc/httpd/conf/httpd.conf
echo "ScriptAlias /cgi-bin/ /var/www/cgi-bin/" >> /etc/httpd/conf/httpd.conf
systemctl restart httpd

# Install CloudWatch agent
yum install -y amazon-cloudwatch-agent
```

### Step 2: Create Application Load Balancer

#### 2.1 ALB Configuration
```bash
# Navigate to EC2 Console
EC2 Console → Load Balancers → Create Load Balancer

# Choose Application Load Balancer
Load balancer name: WebServer-ALB
Scheme: Internet-facing
IP address type: IPv4

# Network mapping:
VPC: Default VPC (or your custom VPC)
Availability Zones: Select at least 2 AZs
Subnets: Public subnets in selected AZs
```

#### 2.2 Security Groups for ALB
```bash
# Create ALB Security Group
Name: ALB-SG
Description: Security group for Application Load Balancer

Inbound Rules:
- HTTP (80) from 0.0.0.0/0
- HTTPS (443) from 0.0.0.0/0

Outbound Rules:
- HTTP (80) to WebServer-SG
- HTTPS (443) to WebServer-SG
```

#### 2.3 Target Group Configuration
```bash
# Create Target Group
Target group name: WebServer-TG
Target type: Instances
Protocol: HTTP
Port: 80
VPC: Default VPC (or your custom VPC)

# Health check settings:
Health check protocol: HTTP
Health check path: /
Health check port: Traffic port
Healthy threshold: 2
Unhealthy threshold: 2
Timeout: 5 seconds
Interval: 30 seconds
Success codes: 200
```

#### 2.4 Configure Listeners
```bash
# Default listener:
Protocol: HTTP
Port: 80
Default actions: Forward to WebServer-TG

# Optional HTTPS listener (if you have SSL certificate):
Protocol: HTTPS
Port: 443
SSL certificate: Choose from ACM or upload
Default actions: Forward to WebServer-TG
```

### Step 3: Create Auto Scaling Group

#### 3.1 ASG Basic Configuration
```bash
# Navigate to EC2 Console
EC2 Console → Auto Scaling Groups → Create Auto Scaling Group

# Step 1: Choose launch template
Auto Scaling group name: WebServer-ASG
Launch template: WebServer-Template-v1
Version: Latest
```

#### 3.2 Network Configuration
```bash
# Step 2: Choose instance launch options
VPC: Default VPC (or your custom VPC)
Availability Zones and subnets: Select multiple AZs
- us-east-1a: subnet-xxx (public)
- us-east-1b: subnet-yyy (public)
- us-east-1c: subnet-zzz (public)
```

#### 3.3 Load Balancer Integration
```bash
# Step 3: Configure advanced options
Load balancing: Attach to an existing load balancer
Existing load balancer target groups: WebServer-TG

# Health checks:
Health check type: ELB
Health check grace period: 300 seconds
```

#### 3.4 Group Size and Scaling
```bash
# Step 4: Configure group size and scaling policies
Desired capacity: 2
Minimum capacity: 1
Maximum capacity: 6

# Scaling policies:
Target tracking scaling policy: Yes
Scaling policy name: CPU-Scaling-Policy
Metric type: Average CPU Utilization
Target value: 70
Instance warmup: 300 seconds
```

#### 3.5 Notifications and Tags
```bash
# Step 5: Add notifications (optional)
SNS Topic: Create new topic or use existing
Events: Launch, Terminate, Fail to launch, Fail to terminate

# Step 6: Add tags
Key: Name, Value: WebServer-ASG-Instance
Key: Environment, Value: Production
Key: Project, Value: AutoScaling-Demo
```

## 🧪 Testing Auto Scaling

### Test 1: Basic Functionality
```bash
# Check initial instances
AWS Console → EC2 → Auto Scaling Groups → WebServer-ASG → Instance management

# Test load balancer
curl http://<ALB-DNS-Name>
# Should return web page from one of the instances

# Refresh multiple times to see load balancing
for i in {1..10}; do
    curl -s http://<ALB-DNS-Name> | grep "Instance ID"
    sleep 1
done
```

### Test 2: Scale Out Testing
```bash
# Method 1: Generate CPU load via web interface
# Open browser: http://<ALB-DNS-Name>
# Click "Generate Load" button on multiple instances

# Method 2: SSH to instances and generate load
ssh -i your-key.pem ec2-user@<instance-ip>
stress --cpu 2 --timeout 300

# Method 3: Use CloudWatch to simulate high CPU
aws cloudwatch put-metric-data \
    --namespace "AWS/EC2" \
    --metric-data MetricName=CPUUtilization,Value=85,Unit=Percent,Dimensions=InstanceId=<instance-id>
```

### Test 3: Scale In Testing
```bash
# Stop load generation and wait for scale-in
# Monitor Auto Scaling activity
AWS Console → EC2 → Auto Scaling Groups → WebServer-ASG → Activity

# Check scaling events
aws autoscaling describe-scaling-activities \
    --auto-scaling-group-name WebServer-ASG \
    --max-items 10
```

### Test 4: Instance Failure Simulation
```bash
# Terminate an instance manually
AWS Console → EC2 → Instances → Select instance → Terminate

# Verify Auto Scaling launches replacement
# Check target group health
AWS Console → EC2 → Target Groups → WebServer-TG → Targets
```

## 📊 Monitoring and Metrics

### CloudWatch Metrics
```bash
# Key Auto Scaling Metrics:
- GroupMinSize, GroupMaxSize, GroupDesiredCapacity
- GroupInServiceInstances, GroupTotalInstances
- GroupPendingInstances, GroupTerminatingInstances

# Key ALB Metrics:
- RequestCount, TargetResponseTime
- HTTPCode_Target_2XX_Count, HTTPCode_Target_4XX_Count
- HealthyHostCount, UnHealthyHostCount
```

### Custom Dashboard
```bash
# Create CloudWatch Dashboard
CloudWatch Console → Dashboards → Create Dashboard

# Add widgets for:
1. Auto Scaling Group metrics
2. ALB performance metrics
3. EC2 instance CPU utilization
4. Target group health
```

## 🔧 Advanced Scaling Policies

### Step-by-Step Scaling Policy
```bash
# Create custom scaling policy
Auto Scaling Groups → WebServer-ASG → Automatic scaling → Create dynamic scaling policy

# Scale-out policy:
Policy type: Step scaling
Scaling policy name: Scale-Out-Policy
CloudWatch alarm: Create new alarm
Metric: Average CPU Utilization > 70%
Take the action: Add 2 instances when 70 <= CPU < 85
                 Add 3 instances when CPU >= 85
```

### Scheduled Scaling
```bash
# Create scheduled action
Auto Scaling Groups → WebServer-ASG → Automatic scaling → Create scheduled action

# Configuration:
Name: Morning-Scale-Out
Recurrence: 0 8 * * MON-FRI (8 AM weekdays)
Desired capacity: 4
Min: 2, Max: 6

# Evening scale-in:
Name: Evening-Scale-In
Recurrence: 0 18 * * MON-FRI (6 PM weekdays)
Desired capacity: 2
Min: 1, Max: 6
```

## 🛡️ Security Best Practices

### Security Group Configuration
```bash
# WebServer-SG (for EC2 instances):
Inbound:
- HTTP (80) from ALB-SG only
- SSH (22) from Bastion-SG or your IP
- HTTPS (443) from ALB-SG only

Outbound:
- All traffic to 0.0.0.0/0 (for updates)
```

### IAM Roles and Policies
```bash
# Create IAM role for EC2 instances
Role name: EC2-AutoScaling-Role
Trusted entity: EC2

# Attach policies:
- CloudWatchAgentServerPolicy
- AmazonSSMManagedInstanceCore
- Custom policy for application needs
```

## 💰 Cost Optimization

### Right-sizing Instances
```bash
# Monitor instance utilization
CloudWatch → Metrics → EC2 → Per-Instance Metrics
# Look for consistently low CPU/memory usage

# Use AWS Compute Optimizer
AWS Console → Compute Optimizer → EC2 instances
# Review recommendations for instance type optimization
```

### Spot Instances Integration
```bash
# Mixed instance types policy
Launch Template → Create new version
Instance types: t3.micro, t3.small, t2.micro
Purchase options: On-Demand and Spot
Spot allocation strategy: Diversified
On-Demand percentage: 20%
```

## 📚 Interview Questions & Answers

### Fresher Level (1-10)

**Q1: What is Auto Scaling in AWS?**
A: Auto Scaling automatically adjusts the number of EC2 instances based on demand. It ensures optimal performance during high traffic and cost savings during low traffic by scaling instances up or down.

**Q2: What are the main components of Auto Scaling?**
A: Launch Template/Configuration (instance blueprint), Auto Scaling Group (manages instances), Scaling Policies (rules for scaling), and Health Checks (monitor instance health).

**Q3: What is the difference between Application Load Balancer and Network Load Balancer?**
A: ALB operates at Layer 7 (HTTP/HTTPS), supports path-based routing, and is ideal for web applications. NLB operates at Layer 4 (TCP/UDP), provides ultra-low latency, and handles millions of requests per second.

**Q4: What is a Launch Template?**
A: Launch Template is a blueprint that defines instance configuration including AMI, instance type, security groups, and user data. It's used by Auto Scaling Groups to launch instances consistently.

**Q5: How does health check work in Auto Scaling?**
A: Auto Scaling performs health checks to determine if instances are healthy. Unhealthy instances are terminated and replaced. Health checks can be EC2 (instance status) or ELB (application health).

### Intermediate Level (6-15)

**Q6: Explain different types of scaling policies.**
A: Target Tracking (maintains specific metric value), Step Scaling (scales based on metric thresholds), Simple Scaling (single scaling action), and Scheduled Scaling (time-based scaling).

**Q7: What is the difference between desired, minimum, and maximum capacity?**
A: Desired capacity is the target number of instances, minimum is the lowest number allowed, maximum is the highest number allowed. Auto Scaling maintains desired capacity within min/max bounds.

**Q8: How do you handle database connections with Auto Scaling?**
A: Use connection pooling, implement proper connection management in application code, use RDS Proxy for database connection pooling, and ensure database can handle concurrent connections.

**Q9: What are the best practices for Auto Scaling?**
A: Use multiple AZs, implement proper health checks, set appropriate scaling policies, use CloudWatch alarms, implement graceful shutdown, and monitor costs regularly.

**Q10: How do you troubleshoot Auto Scaling issues?**
A: Check Auto Scaling activity history, review CloudWatch metrics and alarms, verify launch template configuration, check security groups and subnets, and examine instance logs.

## 🔑 Key Takeaways

- **Auto Scaling**: Provides elasticity and cost optimization
- **Load Balancers**: Distribute traffic and improve availability
- **Health Checks**: Critical for maintaining application reliability
- **Scaling Policies**: Choose appropriate policy based on workload patterns
- **Monitoring**: Essential for optimization and troubleshooting
- **Security**: Implement proper security groups and IAM roles

## 🚀 Next Steps

- Day 17: Advanced Load Balancer Features
- Day 18: Classic Load Balancer Deep Dive
- Day 19: Application Load Balancer Advanced Features
- Day 20: Elastic IP and Advanced Networking

---

**Hands-on Completed:** ✅ Auto Scaling Groups, Application Load Balancer, Scaling Policies  
**Duration:** 3-4 hours  
**Difficulty:** Intermediate