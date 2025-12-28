# Day 19: Application Load Balancer - Advanced Features and Use Cases

## 📚 Learning Objectives
- Master ALB advanced routing capabilities
- Implement multiple target groups with different services
- Configure weighted routing for blue-green deployments
- Practice ALB with microservices architecture
- Understand ALB performance optimization

## 🏗️ Advanced ALB Architecture

```
Internet
    |
Application Load Balancer
    ├── Listener Rules (Priority-based)
    ├── Target Group 1 (Web App - 70%)
    ├── Target Group 2 (New Version - 30%)
    └── Target Group 3 (API Services)
```

## 🛠️ Hands-on Implementation

### Step 1: Create Multiple Target Groups

#### 1.1 Production Web App Target Group
```bash
# Create Production TG
Name: WebApp-Prod-TG
Target type: Instances
Protocol: HTTP, Port: 80
Health check path: /health
```

#### 1.2 Staging Web App Target Group
```bash
# Create Staging TG
Name: WebApp-Staging-TG
Target type: Instances
Protocol: HTTP, Port: 8080
Health check path: /health
```

#### 1.3 API Services Target Group
```bash
# Create API TG
Name: API-Services-TG
Target type: Instances
Protocol: HTTP, Port: 3000
Health check path: /api/health
```

### Step 2: Launch Specialized Instances

#### 2.1 Production Web Server
```bash
# User Data for Production
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Production App v1.0</title></head>
<body style="background: #4CAF50; color: white; font-family: Arial; padding: 40px;">
    <h1>🚀 Production Application v1.0</h1>
    <p>Instance: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>
    <p>This is the stable production version</p>
</body>
</html>
EOF

echo "OK" > /var/www/html/health
```

#### 2.2 Staging Web Server
```bash
# User Data for Staging (Port 8080)
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Configure Apache on port 8080
echo "Listen 8080" >> /etc/httpd/conf/httpd.conf
cat > /etc/httpd/conf.d/port8080.conf << 'EOF'
<VirtualHost *:8080>
    DocumentRoot /var/www/html
</VirtualHost>
EOF

systemctl restart httpd

cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>Staging App v2.0</title></head>
<body style="background: #FF9800; color: white; font-family: Arial; padding: 40px;">
    <h1>🧪 Staging Application v2.0</h1>
    <p>Instance: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>
    <p>This is the new version being tested</p>
</body>
</html>
EOF

echo "OK" > /var/www/html/health
```

### Step 3: Configure Advanced ALB Rules

#### 3.1 Weighted Routing (Blue-Green Deployment)
```bash
# Create listener rule for weighted routing
Priority: 100
Conditions: Path is /*
Actions: 
- Forward to WebApp-Prod-TG (Weight: 70)
- Forward to WebApp-Staging-TG (Weight: 30)
```

#### 3.2 Header-Based Routing
```bash
# Route based on custom headers
Priority: 50
Conditions: HTTP header "X-Version" is "v2"
Actions: Forward to WebApp-Staging-TG (Weight: 100)
```

#### 3.3 Query Parameter Routing
```bash
# Route based on query parameters
Priority: 60
Conditions: Query string is "version=beta"
Actions: Forward to WebApp-Staging-TG (Weight: 100)
```

### Step 4: API Services Integration

#### 4.1 Launch API Server
```bash
# User Data for API Server
#!/bin/bash
yum update -y
yum install -y nodejs npm

cat > /home/ec2-user/api-server.js << 'EOF'
const http = require('http');
const url = require('url');

const server = http.createServer((req, res) => {
    const pathname = url.parse(req.url).pathname;
    const query = url.parse(req.url, true).query;
    
    res.setHeader('Content-Type', 'application/json');
    
    if (pathname === '/api/health') {
        res.writeHead(200);
        res.end(JSON.stringify({ status: 'healthy', service: 'api' }));
    } else if (pathname === '/api/users') {
        res.writeHead(200);
        res.end(JSON.stringify({ 
            users: ['Alice', 'Bob', 'Charlie'],
            instance: process.env.INSTANCE_ID || 'unknown'
        }));
    } else if (pathname === '/api/products') {
        res.writeHead(200);
        res.end(JSON.stringify({ 
            products: ['Laptop', 'Phone', 'Tablet'],
            instance: process.env.INSTANCE_ID || 'unknown'
        }));
    } else {
        res.writeHead(404);
        res.end(JSON.stringify({ error: 'Not Found' }));
    }
});

server.listen(3000, () => {
    console.log('API Server running on port 3000');
});
EOF

cd /home/ec2-user
node api-server.js &
```

#### 4.2 API Routing Rules
```bash
# Route API traffic
Priority: 10
Conditions: Path is /api/*
Actions: Forward to API-Services-TG (Weight: 100)
```

## 🧪 Testing Advanced Features

### Test 1: Weighted Routing
```bash
# Test traffic distribution
for i in {1..20}; do
    curl -s http://<ALB-DNS>/ | grep -E "(Production|Staging)"
done

# Should show ~70% Production, ~30% Staging
```

### Test 2: Header-Based Routing
```bash
# Test version header routing
curl -H "X-Version: v2" http://<ALB-DNS>/
# Should route to staging

curl -H "X-Version: v1" http://<ALB-DNS>/
# Should route to production
```

### Test 3: Query Parameter Routing
```bash
# Test query parameter routing
curl "http://<ALB-DNS>/?version=beta"
# Should route to staging

curl "http://<ALB-DNS>/?version=stable"
# Should route to production
```

### Test 4: API Services
```bash
# Test API endpoints
curl http://<ALB-DNS>/api/users
curl http://<ALB-DNS>/api/products
curl http://<ALB-DNS>/api/health
```

## 📊 Advanced Monitoring

### Custom Metrics Dashboard
```bash
# Create comprehensive dashboard with:
1. Request count by target group
2. Response time by target group
3. HTTP status codes distribution
4. Target health status
5. Rule evaluation metrics
```

### A/B Testing Metrics
```bash
# Track metrics for blue-green deployment:
- Conversion rates by version
- Error rates by version
- Response times by version
- User engagement by version
```

## 🔄 Blue-Green Deployment Process

### Step 1: Initial State (100% Blue)
```bash
# Configure initial routing
Blue (Production): 100%
Green (Staging): 0%
```

### Step 2: Canary Release (90% Blue, 10% Green)
```bash
# Gradually shift traffic
Blue (Production): 90%
Green (Staging): 10%

# Monitor metrics for 30 minutes
```

### Step 3: Increased Testing (70% Blue, 30% Green)
```bash
# Increase green traffic
Blue (Production): 70%
Green (Staging): 30%

# Monitor for issues
```

### Step 4: Full Deployment (0% Blue, 100% Green)
```bash
# Complete migration
Blue (Production): 0%
Green (Staging): 100%

# Update target group names
```

## 📚 Interview Questions & Answers

### Intermediate Level (1-10)

**Q1: How do you implement blue-green deployment with ALB?**
A: Use weighted routing with two target groups. Gradually shift traffic weights from blue (current) to green (new version), monitor metrics, and rollback by adjusting weights if issues occur.

**Q2: What are ALB listener rules and how do they work?**
A: Listener rules determine how ALB routes requests based on conditions (path, host, headers, query parameters). Rules are evaluated by priority (lowest number first) until a match is found.

**Q3: How do you implement A/B testing with ALB?**
A: Use weighted routing to split traffic between versions, implement tracking mechanisms, monitor conversion metrics, and gradually shift traffic based on performance results.

**Q4: What is the difference between path-based and host-based routing?**
A: Path-based routing uses URL paths (/api/, /admin/) to route to different services. Host-based routing uses domain names (api.example.com, admin.example.com) for routing.

**Q5: How do you handle session persistence in ALB?**
A: Use sticky sessions with application-controlled cookies or load balancer-generated cookies. However, prefer stateless applications with external session storage for better scalability.

### Advanced Level (6-15)

**Q6: How do you optimize ALB performance for microservices?**
A: Use appropriate target group configurations, implement efficient health checks, optimize connection settings, use HTTP/2, and implement proper caching strategies.

**Q7: What are the security considerations for ALB?**
A: Implement WAF rules, use security groups properly, enable access logs, implement SSL/TLS best practices, and regularly audit listener rules and target groups.

**Q8: How do you troubleshoot ALB routing issues?**
A: Check listener rule priorities and conditions, verify target group health, analyze access logs, use AWS X-Ray for tracing, and monitor CloudWatch metrics.

**Q9: What are ALB target group attributes and their impact?**
A: Deregistration delay affects deployment speed, stickiness impacts load distribution, health check settings affect failover time, and slow start helps with instance warm-up.

**Q10: How do you implement multi-region load balancing?**
A: Use Route 53 with health checks for DNS-based routing, implement cross-region ALBs, use Global Accelerator for performance, and consider data consistency requirements.

## 🔑 Key Takeaways

- **Advanced Routing**: Enables sophisticated traffic management
- **Blue-Green Deployments**: Minimize deployment risks
- **Microservices Support**: Perfect for containerized applications
- **Performance Optimization**: Critical for high-traffic applications
- **Monitoring**: Essential for maintaining service quality

## 🚀 Next Steps

- Day 20: Elastic IP and Advanced Networking
- Day 21: AWS Lambda and Serverless Computing
- Implement container-based targets with ECS

---

**Hands-on Completed:** ✅ Advanced ALB Features, Blue-Green Deployment, Microservices Routing  
**Duration:** 3-4 hours  
**Difficulty:** Advanced