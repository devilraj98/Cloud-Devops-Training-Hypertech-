# Day 17: Advanced Load Balancer Features and Network Optimization

## 📚 Learning Objectives
- Master advanced ALB features (path-based routing, host-based routing)
- Implement SSL/TLS termination and certificate management
- Configure sticky sessions and connection draining
- Understand load balancer algorithms and health checks
- Practice advanced networking configurations

## 🏗️ Advanced Load Balancer Architecture

```
Internet
    |
Application Load Balancer
    ├── SSL Certificate (ACM)
    ├── Listener Rules (Path/Host-based)
    └── Target Groups
        ├── Web Servers (Port 80)
        ├── API Servers (Port 8080)
        └── Admin Panel (Port 3000)
```

## 🔧 Advanced ALB Features

### Path-Based Routing
Route requests based on URL path patterns to different target groups.

### Host-Based Routing
Route requests based on host header to different target groups.

### SSL/TLS Termination
Handle SSL encryption/decryption at load balancer level.

## 🛠️ Hands-on Implementation

### Step 1: Create Multiple Target Groups

#### 1.1 Web Application Target Group
```bash
# Create Web App Target Group
Name: WebApp-TG
Target type: Instances
Protocol: HTTP, Port: 80
VPC: Your VPC

Health Check:
- Protocol: HTTP
- Path: /health
- Port: 80
- Healthy threshold: 2
- Unhealthy threshold: 3
- Timeout: 5 seconds
- Interval: 30 seconds
```

#### 1.2 API Target Group
```bash
# Create API Target Group
Name: API-TG
Target type: Instances
Protocol: HTTP, Port: 8080
VPC: Your VPC

Health Check:
- Protocol: HTTP
- Path: /api/health
- Port: 8080
- Healthy threshold: 2
- Unhealthy threshold: 3
```

#### 1.3 Admin Target Group
```bash
# Create Admin Target Group
Name: Admin-TG
Target type: Instances
Protocol: HTTP, Port: 3000
VPC: Your VPC

Health Check:
- Protocol: HTTP
- Path: /admin/health
- Port: 3000
```

### Step 2: Launch Specialized Instances

#### 2.1 Web Server Instance
```bash
# Launch Template for Web Servers
Name: WebServer-Template
AMI: Amazon Linux 2023
Instance Type: t3.micro
Security Group: WebServer-SG

# User Data:
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Create web application
cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Web Application</title></head>
<body>
    <h1>🌐 Web Application Server</h1>
    <p>Instance: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>
    <p>This is the main web application</p>
</body>
</html>
EOF

# Health check endpoint
echo "OK" > /var/www/html/health
```

#### 2.2 API Server Instance
```bash
# Launch Template for API Servers
Name: APIServer-Template
AMI: Amazon Linux 2023
Instance Type: t3.micro
Security Group: APIServer-SG

# User Data:
#!/bin/bash
yum update -y
yum install -y nodejs npm

# Create simple API server
cat > /home/ec2-user/server.js << 'EOF'
const http = require('http');
const url = require('url');

const server = http.createServer((req, res) => {
    const pathname = url.parse(req.url).pathname;
    
    res.setHeader('Content-Type', 'application/json');
    
    if (pathname === '/api/health') {
        res.writeHead(200);
        res.end(JSON.stringify({ status: 'healthy', service: 'api' }));
    } else if (pathname.startsWith('/api/')) {
        res.writeHead(200);
        res.end(JSON.stringify({ 
            message: 'API Response', 
            path: pathname,
            instance: process.env.INSTANCE_ID || 'unknown'
        }));
    } else {
        res.writeHead(404);
        res.end(JSON.stringify({ error: 'Not Found' }));
    }
});

server.listen(8080, () => {
    console.log('API Server running on port 8080');
});
EOF

# Start API server
cd /home/ec2-user
node server.js &
```

#### 2.3 Admin Panel Instance
```bash
# Launch Template for Admin Servers
Name: AdminServer-Template
AMI: Amazon Linux 2023
Instance Type: t3.micro
Security Group: AdminServer-SG

# User Data:
#!/bin/bash
yum update -y
yum install -y nodejs npm

# Create admin panel
cat > /home/ec2-user/admin.js << 'EOF'
const http = require('http');
const url = require('url');

const server = http.createServer((req, res) => {
    const pathname = url.parse(req.url).pathname;
    
    if (pathname === '/admin/health') {
        res.writeHead(200, {'Content-Type': 'text/plain'});
        res.end('OK');
    } else if (pathname.startsWith('/admin/')) {
        res.writeHead(200, {'Content-Type': 'text/html'});
        res.end(`
            <html>
            <head><title>Admin Panel</title></head>
            <body>
                <h1>🔧 Admin Panel</h1>
                <p>Secure admin interface</p>
                <p>Path: ${pathname}</p>
            </body>
            </html>
        `);
    } else {
        res.writeHead(404);
        res.end('Not Found');
    }
});

server.listen(3000, () => {
    console.log('Admin server running on port 3000');
});
EOF

cd /home/ec2-user
node admin.js &
```

### Step 3: Configure Advanced ALB

#### 3.1 Create ALB with Multiple Listeners
```bash
# Create Application Load Balancer
Name: Advanced-ALB
Scheme: Internet-facing
IP address type: IPv4
VPC: Your VPC
Subnets: Select public subnets in multiple AZs
Security Groups: ALB-SG
```

#### 3.2 SSL Certificate Setup
```bash
# Request SSL Certificate via ACM
AWS Console → Certificate Manager → Request Certificate

# Configuration:
Domain name: yourdomain.com
Additional names: *.yourdomain.com, api.yourdomain.com, admin.yourdomain.com
Validation method: DNS validation

# Add CNAME records to your DNS provider
# Wait for certificate validation
```

#### 3.3 Configure Listeners and Rules

##### HTTP Listener (Port 80)
```bash
# Default action: Redirect to HTTPS
Protocol: HTTP, Port: 80
Default actions: Redirect to HTTPS
Status code: 301
```

##### HTTPS Listener (Port 443)
```bash
# Main HTTPS listener
Protocol: HTTPS, Port: 443
SSL Certificate: Select from ACM
Security policy: ELBSecurityPolicy-TLS-1-2-2017-01

# Default action: Forward to WebApp-TG
```

#### 3.4 Advanced Routing Rules

##### Path-Based Routing Rules
```bash
# Rule 1: API Traffic
Conditions: Path is /api/*
Actions: Forward to API-TG
Priority: 100

# Rule 2: Admin Traffic
Conditions: Path is /admin/*
Actions: Forward to Admin-TG
Priority: 200

# Rule 3: Static Assets
Conditions: Path is /static/*
Actions: Forward to WebApp-TG
Priority: 300
```

##### Host-Based Routing Rules
```bash
# Rule 4: API Subdomain
Conditions: Host header is api.yourdomain.com
Actions: Forward to API-TG
Priority: 50

# Rule 5: Admin Subdomain
Conditions: Host header is admin.yourdomain.com
Actions: Forward to Admin-TG
Priority: 60
```

### Step 4: Advanced Target Group Configuration

#### 4.1 Sticky Sessions Configuration
```bash
# Enable sticky sessions for WebApp-TG
Target Groups → WebApp-TG → Attributes → Edit

# Stickiness settings:
Stickiness: Enabled
Stickiness type: Load balancer generated cookie
Stickiness duration: 1 day (86400 seconds)
```

#### 4.2 Connection Draining
```bash
# Configure connection draining
Target Groups → WebApp-TG → Attributes → Edit

# Deregistration delay: 300 seconds
# This allows in-flight requests to complete before removing targets
```

#### 4.3 Health Check Optimization
```bash
# Advanced health check settings
Health check protocol: HTTP
Health check path: /health
Health check port: Traffic port
Healthy threshold: 2
Unhealthy threshold: 2
Timeout: 5 seconds
Interval: 30 seconds
Success codes: 200,201,202
```

## 🧪 Testing Advanced Features

### Test 1: Path-Based Routing
```bash
# Test different paths
curl https://yourdomain.com/
curl https://yourdomain.com/api/users
curl https://yourdomain.com/admin/dashboard

# Verify routing to correct target groups
# Check target group metrics in AWS Console
```

### Test 2: Host-Based Routing
```bash
# Test different subdomains
curl https://api.yourdomain.com/users
curl https://admin.yourdomain.com/dashboard

# Add host headers for testing
curl -H "Host: api.yourdomain.com" https://your-alb-dns-name/
curl -H "Host: admin.yourdomain.com" https://your-alb-dns-name/
```

### Test 3: SSL/TLS Termination
```bash
# Test SSL certificate
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com

# Check certificate details
curl -vI https://yourdomain.com/

# Test HTTP to HTTPS redirect
curl -I http://yourdomain.com/
# Should return 301 redirect to HTTPS
```

### Test 4: Sticky Sessions
```bash
# Test session stickiness
for i in {1..10}; do
    curl -c cookies.txt -b cookies.txt https://yourdomain.com/
    sleep 1
done

# Should consistently hit the same backend server
```

## 📊 Monitoring and Metrics

### Advanced CloudWatch Metrics
```bash
# ALB-specific metrics:
- RequestCount, ActiveConnectionCount
- NewConnectionCount, RejectedConnectionCount
- TargetResponseTime, TargetConnectionErrorCount
- HTTPCode_ELB_4XX_Count, HTTPCode_ELB_5XX_Count
- HTTPCode_Target_2XX_Count, HTTPCode_Target_4XX_Count

# Target Group metrics:
- HealthyHostCount, UnHealthyHostCount
- RequestCountPerTarget, TargetResponseTime
```

### Custom Alarms
```bash
# Create CloudWatch Alarms
CloudWatch Console → Alarms → Create Alarm

# High response time alarm:
Metric: TargetResponseTime
Threshold: > 2 seconds
Period: 5 minutes
Evaluation periods: 2

# Unhealthy targets alarm:
Metric: UnHealthyHostCount
Threshold: >= 1
Period: 1 minute
```

## 🔒 Security Best Practices

### Security Groups Configuration
```bash
# ALB Security Group (ALB-SG):
Inbound:
- HTTP (80) from 0.0.0.0/0
- HTTPS (443) from 0.0.0.0/0

Outbound:
- HTTP (80) to WebServer-SG
- HTTP (8080) to APIServer-SG
- HTTP (3000) to AdminServer-SG

# WebServer Security Group:
Inbound:
- HTTP (80) from ALB-SG only
- SSH (22) from Bastion-SG

# APIServer Security Group:
Inbound:
- HTTP (8080) from ALB-SG only
- SSH (22) from Bastion-SG

# AdminServer Security Group:
Inbound:
- HTTP (3000) from ALB-SG only
- SSH (22) from Bastion-SG
```

### WAF Integration
```bash
# Create Web Application Firewall
AWS Console → WAF & Shield → Web ACLs → Create web ACL

# Configuration:
Name: Advanced-ALB-WAF
Resource type: Application Load Balancer
Associated resources: Advanced-ALB

# Add rules:
1. AWS Managed Rules - Core Rule Set
2. AWS Managed Rules - Known Bad Inputs
3. Rate limiting rule (1000 requests per 5 minutes)
4. IP whitelist for admin panel
```

## 💰 Cost Optimization

### Load Balancer Costs
```bash
# ALB Pricing (us-east-1):
- $0.0225 per hour
- $0.008 per LCU (Load Balancer Capacity Unit)

# LCU dimensions:
- New connections per second
- Active connections per minute
- Bandwidth (Mbps)
- Rule evaluations per second
```

### Optimization Strategies
```bash
# 1. Right-size target groups
# 2. Optimize health check intervals
# 3. Use connection multiplexing
# 4. Implement efficient routing rules
# 5. Monitor LCU usage
```

## 🔧 Troubleshooting Guide

### Common Issues

#### Issue 1: 502 Bad Gateway
```bash
# Check:
1. Target group health status
2. Security group rules
3. Application health check endpoint
4. Instance application status
```

#### Issue 2: SSL Certificate Issues
```bash
# Check:
1. Certificate validation status
2. Domain name matches
3. Certificate expiration
4. Security policy compatibility
```

#### Issue 3: Routing Not Working
```bash
# Check:
1. Listener rule priorities
2. Condition configurations
3. Target group associations
4. Health check status
```

## 📚 Interview Questions & Answers

### Intermediate Level (1-10)

**Q1: What is path-based routing in ALB?**
A: Path-based routing allows ALB to route requests to different target groups based on URL path patterns. For example, /api/* routes to API servers, /admin/* routes to admin servers.

**Q2: How does SSL termination work in ALB?**
A: ALB handles SSL encryption/decryption, reducing CPU load on backend servers. SSL certificates are managed by ACM, and ALB forwards decrypted HTTP traffic to targets.

**Q3: What are sticky sessions and when to use them?**
A: Sticky sessions bind user sessions to specific backend servers using cookies. Use for applications that store session data locally, but prefer stateless applications with external session storage.

**Q4: Explain connection draining in ALB.**
A: Connection draining allows in-flight requests to complete before removing targets from service. Default is 300 seconds, preventing abrupt connection termination during deployments.

**Q5: What is the difference between ALB and NLB?**
A: ALB operates at Layer 7 (HTTP/HTTPS), supports content-based routing, and handles SSL termination. NLB operates at Layer 4 (TCP/UDP), provides ultra-low latency, and preserves source IP.

### Advanced Level (6-15)

**Q6: How do you implement blue-green deployments with ALB?**
A: Create two target groups (blue/green), gradually shift traffic using weighted routing, monitor metrics, and rollback by adjusting weights if issues occur.

**Q7: What are Load Balancer Capacity Units (LCUs)?**
A: LCUs measure ALB usage across four dimensions: new connections/sec, active connections/min, bandwidth (Mbps), and rule evaluations/sec. Billing is based on highest dimension.

**Q8: How do you secure admin interfaces behind ALB?**
A: Use host-based routing for admin subdomain, implement WAF rules, restrict source IPs, use strong authentication, and consider VPN access for sensitive operations.

**Q9: What are the limitations of ALB?**
A: ALB doesn't preserve source IP (use NLB for this), has rule evaluation limits, doesn't support static IP addresses, and has latency overhead compared to NLB.

**Q10: How do you troubleshoot high response times in ALB?**
A: Check target response times, review target group health, analyze CloudWatch metrics, examine application logs, verify resource utilization, and consider scaling targets.

## 🔑 Key Takeaways

- **Advanced Routing**: Path and host-based routing enable microservices architecture
- **SSL Termination**: Reduces backend server load and centralizes certificate management
- **Sticky Sessions**: Use carefully, prefer stateless applications
- **Health Checks**: Critical for maintaining application availability
- **Security**: Implement WAF, proper security groups, and access controls
- **Monitoring**: Essential for performance optimization and troubleshooting

## 🚀 Next Steps

- Day 18: Classic Load Balancer Deep Dive
- Day 19: Network Load Balancer Advanced Features
- Day 20: Elastic IP and Advanced Networking Patterns

---

**Hands-on Completed:** ✅ Advanced ALB Features, SSL Termination, Path/Host Routing  
**Duration:** 3-4 hours  
**Difficulty:** Advanced