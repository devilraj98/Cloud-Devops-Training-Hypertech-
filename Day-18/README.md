# Day 18: Classic Load Balancer - Deep Dive and Practical Implementation

## 📚 Learning Objectives
- Understand Classic Load Balancer (CLB) architecture and use cases
- Compare CLB with ALB and NLB
- Implement CLB with EC2 instances
- Configure health checks and connection settings
- Practice CLB monitoring and troubleshooting

## 🏗️ Classic Load Balancer Architecture

```
Internet
    |
Classic Load Balancer
(Layer 4 & Layer 7)
    |
┌─────────────────────────────┐
│     EC2 Instances           │
│  ┌─────────┐  ┌─────────┐  │
│  │  Web-1  │  │  Web-2  │  │
│  │  AZ-1a  │  │  AZ-1b  │  │
│  └─────────┘  └─────────┘  │
└─────────────────────────────┘
```

## 🔍 Classic Load Balancer Overview

### What is Classic Load Balancer?
CLB is the original AWS load balancer that operates at both Layer 4 (TCP) and Layer 7 (HTTP/HTTPS). It's the predecessor to ALB and NLB.

### Key Characteristics
- **Legacy Service**: First generation load balancer
- **Layer 4 & 7**: Supports both TCP and HTTP/HTTPS
- **Simple Configuration**: Basic load balancing features
- **Cost-Effective**: Lower cost for simple use cases

### When to Use CLB
- Legacy applications requiring CLB
- Simple load balancing needs
- Cost-sensitive environments
- Applications not requiring advanced routing

## 🆚 Load Balancer Comparison

| Feature | Classic LB | Application LB | Network LB |
|---------|------------|----------------|------------|
| **OSI Layer** | Layer 4 & 7 | Layer 7 | Layer 4 |
| **Protocols** | HTTP, HTTPS, TCP, SSL | HTTP, HTTPS | TCP, UDP, TLS |
| **Routing** | Basic | Advanced (path/host) | IP-based |
| **Target Types** | EC2 instances | Instances, IPs, Lambda | Instances, IPs |
| **Health Checks** | Basic | Advanced | Basic |
| **SSL Termination** | Yes | Yes | Yes |
| **WebSocket** | No | Yes | Yes |
| **Static IP** | No | No | Yes |
| **Pricing** | Lowest | Medium | Medium |

## 🛠️ Hands-on Implementation

### Step 1: Prepare EC2 Instances

#### 1.1 Launch Web Server Instances
```bash
# Launch 2 EC2 instances in different AZs
# Instance 1 Configuration:
Name: WebServer-1
AMI: Amazon Linux 2023
Instance Type: t2.micro
AZ: us-east-1a
Security Group: WebServer-SG
Key Pair: your-key-pair

# Instance 2 Configuration:
Name: WebServer-2
AMI: Amazon Linux 2023
Instance Type: t2.micro
AZ: us-east-1b
Security Group: WebServer-SG
Key Pair: your-key-pair
```

#### 1.2 Configure Web Servers
```bash
# User Data for both instances:
#!/bin/bash
yum update -y
yum install -y httpd

# Start and enable Apache
systemctl start httpd
systemctl enable httpd

# Get instance metadata
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

# Create custom web page
cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Classic Load Balancer Demo</title>
    <style>
        body { 
            font-family: Arial, sans-serif; 
            margin: 40px; 
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        .container { 
            background: rgba(255,255,255,0.1); 
            padding: 30px; 
            border-radius: 15px; 
            backdrop-filter: blur(10px);
        }
        .info { 
            background: rgba(255,255,255,0.2); 
            padding: 15px; 
            margin: 15px 0; 
            border-radius: 8px; 
        }
        .server-id { 
            font-size: 24px; 
            font-weight: bold; 
            color: #ffeb3b; 
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🔄 Classic Load Balancer Demo</h1>
        <div class="info">
            <div class="server-id">Server: $INSTANCE_ID</div>
            <p><strong>Availability Zone:</strong> $AZ</p>
            <p><strong>Private IP:</strong> $PRIVATE_IP</p>
            <p><strong>Timestamp:</strong> $(date)</p>
        </div>
        <p>This server is behind a Classic Load Balancer!</p>
        <div class="info">
            <h3>Load Balancer Features:</h3>
            <ul>
                <li>Layer 4 & Layer 7 load balancing</li>
                <li>Health checks and failover</li>
                <li>SSL termination support</li>
                <li>Cross-zone load balancing</li>
            </ul>
        </div>
    </div>
</body>
</html>
EOF

# Create health check endpoint
echo "Healthy" > /var/www/html/health.html

# Create status endpoint
cat > /var/www/html/status.html << EOF
{
    "status": "healthy",
    "instance_id": "$INSTANCE_ID",
    "availability_zone": "$AZ",
    "private_ip": "$PRIVATE_IP",
    "timestamp": "$(date -Iseconds)"
}
EOF
```

### Step 2: Create Security Groups

#### 2.1 Classic Load Balancer Security Group
```bash
# Create CLB Security Group
Name: CLB-SG
Description: Security group for Classic Load Balancer

Inbound Rules:
- HTTP (80) from 0.0.0.0/0
- HTTPS (443) from 0.0.0.0/0

Outbound Rules:
- HTTP (80) to WebServer-SG
- HTTPS (443) to WebServer-SG
```

#### 2.2 Web Server Security Group
```bash
# Create/Update WebServer-SG
Name: WebServer-SG
Description: Security group for web servers

Inbound Rules:
- HTTP (80) from CLB-SG
- SSH (22) from your IP or Bastion-SG

Outbound Rules:
- All Traffic to 0.0.0.0/0
```

### Step 3: Create Classic Load Balancer

#### 3.1 Basic Configuration
```bash
# Navigate to EC2 Console
EC2 Console → Load Balancers → Create Load Balancer
→ Classic Load Balancer → Create

# Basic Configuration:
Load Balancer name: WebApp-CLB
Create LB inside: Default VPC (or your custom VPC)
Create an internal load balancer: No (for internet-facing)
```

#### 3.2 Listener Configuration
```bash
# Listener Configuration:
Load Balancer Protocol: HTTP
Load Balancer Port: 80
Instance Protocol: HTTP
Instance Port: 80

# Optional HTTPS Listener:
Load Balancer Protocol: HTTPS
Load Balancer Port: 443
Instance Protocol: HTTP
Instance Port: 80
SSL Certificate: Choose from ACM or upload
```

#### 3.3 Availability Zones
```bash
# Select Availability Zones:
Available subnets:
☑ us-east-1a: subnet-xxx (public)
☑ us-east-1b: subnet-yyy (public)
☑ us-east-1c: subnet-zzz (public)
```

#### 3.4 Security Groups
```bash
# Assign Security Groups:
Select existing security group: CLB-SG
```

#### 3.5 Health Check Configuration
```bash
# Configure Health Check:
Ping Protocol: HTTP
Ping Port: 80
Ping Path: /health.html

# Advanced Details:
Response Timeout: 5 seconds
Health Check Interval: 30 seconds
Unhealthy Threshold: 2
Healthy Threshold: 10
```

#### 3.6 Add EC2 Instances
```bash
# Add Instances:
☑ WebServer-1 (i-xxxxx)
☑ WebServer-2 (i-yyyyy)

# Enable Cross-Zone Load Balancing: Yes
# Enable Connection Draining: Yes
# Connection Draining Timeout: 300 seconds
```

### Step 4: Advanced Configuration

#### 4.1 Cross-Zone Load Balancing
```bash
# Enable Cross-Zone Load Balancing
Load Balancers → WebApp-CLB → Description → Edit attributes

# Cross-Zone Load Balancing: Enable
# This ensures even distribution across all AZs
```

#### 4.2 Connection Draining
```bash
# Configure Connection Draining
Load Balancers → WebApp-CLB → Description → Edit attributes

# Connection Draining: Enable
# Connection Draining Timeout: 300 seconds
# Allows in-flight requests to complete during instance removal
```

#### 4.3 Access Logs
```bash
# Enable Access Logs
Load Balancers → WebApp-CLB → Description → Edit attributes

# Access Logs: Enable
# S3 Location: your-clb-logs-bucket/clb-logs
# Create S3 bucket if needed
```

#### 4.4 Idle Timeout
```bash
# Configure Idle Timeout
Load Balancers → WebApp-CLB → Description → Edit attributes

# Idle Timeout: 60 seconds
# Adjust based on application requirements
```

## 🧪 Testing Classic Load Balancer

### Test 1: Basic Load Balancing
```bash
# Get CLB DNS name
CLB_DNS=$(aws elb describe-load-balancers \
    --load-balancer-names WebApp-CLB \
    --query 'LoadBalancerDescriptions[0].DNSName' \
    --output text)

echo "CLB DNS: $CLB_DNS"

# Test load balancing
for i in {1..10}; do
    echo "Request $i:"
    curl -s http://$CLB_DNS | grep "Server:"
    sleep 1
done
```

### Test 2: Health Check Validation
```bash
# Check instance health
aws elb describe-instance-health \
    --load-balancer-name WebApp-CLB

# Stop Apache on one instance to test failover
ssh -i your-key.pem ec2-user@<instance-ip>
sudo systemctl stop httpd

# Test that traffic only goes to healthy instance
for i in {1..5}; do
    curl -s http://$CLB_DNS | grep "Server:"
done

# Restart Apache
sudo systemctl start httpd
```

### Test 3: SSL/HTTPS Testing (if configured)
```bash
# Test HTTPS endpoint
curl -k https://$CLB_DNS

# Check SSL certificate
openssl s_client -connect $CLB_DNS:443 -servername $CLB_DNS
```

### Test 4: Performance Testing
```bash
# Install Apache Bench
sudo yum install -y httpd-tools

# Performance test
ab -n 1000 -c 10 http://$CLB_DNS/

# Results will show:
# - Requests per second
# - Time per request
# - Connection times
# - Distribution of response times
```

## 📊 Monitoring Classic Load Balancer

### CloudWatch Metrics
```bash
# Key CLB Metrics:
- RequestCount: Total requests
- LatencyHigh: Response time
- HTTPCode_ELB_4XX: 4xx errors from ELB
- HTTPCode_ELB_5XX: 5xx errors from ELB
- HTTPCode_Backend_2XX: 2xx responses from instances
- HTTPCode_Backend_4XX: 4xx responses from instances
- HTTPCode_Backend_5XX: 5xx responses from instances
- HealthyHostCount: Number of healthy instances
- UnHealthyHostCount: Number of unhealthy instances
- BackendConnectionErrors: Connection errors to instances
```

### Create CloudWatch Dashboard
```bash
# Create Custom Dashboard
CloudWatch Console → Dashboards → Create Dashboard

# Add widgets for:
1. Request Count (Line graph)
2. Latency (Line graph)
3. Healthy/Unhealthy Host Count (Number)
4. HTTP Response Codes (Stacked area)
5. Backend Connection Errors (Line graph)
```

### Set Up Alarms
```bash
# High Latency Alarm
CloudWatch Console → Alarms → Create Alarm

Metric: Latency
Statistic: Average
Period: 5 minutes
Threshold: > 2 seconds
Comparison: Greater than threshold
Treat missing data: As breaching threshold

# Unhealthy Hosts Alarm
Metric: UnHealthyHostCount
Statistic: Average
Period: 1 minute
Threshold: >= 1
```

## 🔧 Troubleshooting Guide

### Common Issues

#### Issue 1: 503 Service Unavailable
```bash
# Possible causes:
1. All instances are unhealthy
2. No instances registered
3. Security group blocking traffic
4. Health check failing

# Troubleshooting steps:
1. Check instance health status
2. Verify security group rules
3. Test health check endpoint manually
4. Check instance application status
```

#### Issue 2: Uneven Traffic Distribution
```bash
# Possible causes:
1. Cross-zone load balancing disabled
2. Instances in different AZs
3. Connection draining issues

# Solutions:
1. Enable cross-zone load balancing
2. Ensure instances in multiple AZs
3. Check connection draining settings
```

#### Issue 3: High Latency
```bash
# Possible causes:
1. Instance performance issues
2. Network connectivity problems
3. Application bottlenecks

# Troubleshooting:
1. Check instance CPU/memory usage
2. Review application logs
3. Test direct instance access
4. Monitor network metrics
```

## 💰 Cost Analysis

### CLB Pricing
```bash
# Classic Load Balancer Costs (us-east-1):
- $0.025 per hour
- $0.008 per GB of data processed

# Monthly cost example:
# 1 CLB running 24/7: $0.025 × 24 × 30 = $18/month
# 100GB data transfer: 100 × $0.008 = $0.80
# Total: ~$18.80/month
```

### Cost Optimization
```bash
# Optimization strategies:
1. Use CLB for simple use cases only
2. Consider ALB for advanced features
3. Monitor data transfer costs
4. Right-size backend instances
5. Implement efficient health checks
```

## 📚 Interview Questions & Answers

### Fresher Level (1-10)

**Q1: What is Classic Load Balancer?**
A: Classic Load Balancer is AWS's first-generation load balancer that operates at both Layer 4 (TCP) and Layer 7 (HTTP/HTTPS). It provides basic load balancing capabilities for EC2 instances.

**Q2: What's the difference between CLB and ALB?**
A: CLB is basic load balancer with simple routing, while ALB offers advanced features like path-based routing, host-based routing, and support for microservices architecture.

**Q3: How does health check work in CLB?**
A: CLB sends periodic requests to specified path on instances. If instance fails health checks (default 2 consecutive failures), it's marked unhealthy and removed from rotation.

**Q4: What is cross-zone load balancing?**
A: Cross-zone load balancing distributes traffic evenly across all instances in all enabled AZs, rather than just within each AZ. This ensures better traffic distribution.

**Q5: What is connection draining in CLB?**
A: Connection draining allows in-flight requests to complete before removing an instance from service. Default timeout is 300 seconds, preventing abrupt connection termination.

### Intermediate Level (6-15)

**Q6: When would you choose CLB over ALB?**
A: Choose CLB for legacy applications requiring CLB support, simple load balancing needs, cost-sensitive environments, or when advanced routing features aren't needed.

**Q7: What are the limitations of Classic Load Balancer?**
A: CLB doesn't support path-based routing, host-based routing, WebSocket, HTTP/2, or container-based targets. It also has limited SSL/TLS policy options.

**Q8: How do you troubleshoot CLB performance issues?**
A: Check CloudWatch metrics (latency, request count, error rates), verify instance health, review security groups, test direct instance access, and analyze access logs.

**Q9: What is the difference between CLB and NLB?**
A: CLB operates at Layer 4 & 7 with basic features, while NLB operates at Layer 4 only but provides ultra-low latency, static IPs, and higher performance.

**Q10: How do you implement SSL termination with CLB?**
A: Configure HTTPS listener on CLB with SSL certificate from ACM or uploaded certificate. CLB handles SSL encryption/decryption and forwards HTTP traffic to instances.

## 🔑 Key Takeaways

- **Legacy Service**: CLB is first-generation, suitable for simple use cases
- **Layer 4 & 7**: Supports both TCP and HTTP/HTTPS protocols
- **Basic Features**: Simple load balancing without advanced routing
- **Cost-Effective**: Lower cost for basic load balancing needs
- **Health Checks**: Essential for maintaining application availability
- **Migration Path**: Consider upgrading to ALB for advanced features

## 🚀 Next Steps

- Day 19: Application Load Balancer Advanced Features
- Day 20: AWS VPC - Prod level Project
- Day 21: AWS Lambda and Serverless Computing

---

**Hands-on Completed:** ✅ Classic Load Balancer Setup, Health Checks, Monitoring  
**Duration:** 2-3 hours  
**Difficulty:** Beginner to Intermediate