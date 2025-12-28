# Day 20: Elastic IP and Advanced Networking Patterns

## 📚 Learning Objectives
- Understand Elastic IP (EIP) concepts and use cases
- Implement EIP with EC2 instances
- Practice EIP management and best practices
- Explore advanced networking patterns
- Understand cost implications of EIPs

## 🌐 Elastic IP Overview

### What is Elastic IP?
Elastic IP (EIP) is a static, public IPv4 address designed for dynamic cloud computing. It's associated with your AWS account and can be attached to any EC2 instance.

### Key Characteristics
- **Static Public IP**: Doesn't change when instance stops/starts
- **Remappable**: Can be moved between instances instantly
- **Regional Resource**: Specific to AWS region
- **Limited Quantity**: 5 EIPs per region by default

## 🎯 Use Cases for Elastic IP

### 1. High Availability Applications
```bash
# Scenario: Web server with failover
Primary Instance (EIP attached) → Fails
Secondary Instance (EIP reassigned) → Takes over
# Users continue accessing same IP address
```

### 2. DNS Management
```bash
# Scenario: Domain pointing to specific IP
Domain: example.com → EIP: 54.123.45.67
# No DNS changes needed when switching instances
```

### 3. Whitelisting Requirements
```bash
# Scenario: Third-party API access
External Service whitelist: 54.123.45.67
# Consistent IP for external integrations
```

## 🛠️ Hands-on Implementation

### Step 1: Allocate Elastic IP

#### 1.1 Allocate EIP via Console
```bash
# Navigate to EC2 Console
EC2 Console → Network & Security → Elastic IPs → Allocate Elastic IP address

# Configuration:
Network Border Group: us-east-1
Public IPv4 address pool: Amazon's pool of IPv4 addresses
Tags:
- Name: WebServer-EIP
- Environment: Production
- Purpose: High-Availability-Web-Server
```

#### 1.2 Allocate EIP via CLI
```bash
# Allocate new EIP
aws ec2 allocate-address --domain vpc

# Output example:
{
    "PublicIp": "54.123.45.67",
    "AllocationId": "eipalloc-12345678",
    "Domain": "vpc"
}

# Tag the EIP
aws ec2 create-tags \
    --resources eipalloc-12345678 \
    --tags Key=Name,Value=WebServer-EIP
```

### Step 2: Launch EC2 Instance

#### 2.1 Create Web Server Instance
```bash
# Launch EC2 instance
Name: WebServer-Primary
AMI: Amazon Linux 2023
Instance Type: t3.micro
VPC: Default VPC
Subnet: Public subnet
Security Group: WebServer-SG
Key Pair: your-key-pair

# User Data:
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Get instance metadata
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
PUBLIC_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)
PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

cat > /var/www/html/index.html << EOF
<!DOCTYPE html>
<html>
<head>
    <title>Elastic IP Demo</title>
    <style>
        body { font-family: Arial; margin: 40px; background: #f0f8ff; }
        .container { background: white; padding: 30px; border-radius: 10px; box-shadow: 0 4px 6px rgba(0,0,0,0.1); }
        .info { background: #e7f3ff; padding: 15px; margin: 15px 0; border-radius: 5px; }
        .eip-status { background: #d4edda; padding: 10px; border-radius: 5px; color: #155724; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🌐 Elastic IP Demonstration</h1>
        <div class="info">
            <h3>Instance Information:</h3>
            <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
            <p><strong>Current Public IP:</strong> $PUBLIC_IP</p>
            <p><strong>Private IP:</strong> $PRIVATE_IP</p>
            <p><strong>Timestamp:</strong> $(date)</p>
        </div>
        <div class="eip-status">
            <h3>Elastic IP Status:</h3>
            <p id="eip-status">Checking EIP association...</p>
        </div>
        <div class="info">
            <h3>Benefits of Elastic IP:</h3>
            <ul>
                <li>Static public IP address</li>
                <li>Quick failover capabilities</li>
                <li>No DNS propagation delays</li>
                <li>Consistent external access point</li>
            </ul>
        </div>
    </div>
    
    <script>
        // Check if current IP matches expected EIP
        const currentIP = '$PUBLIC_IP';
        const expectedEIP = 'WILL_BE_UPDATED_AFTER_ASSOCIATION';
        
        if (currentIP === expectedEIP) {
            document.getElementById('eip-status').innerHTML = 
                '<strong>✅ Elastic IP is associated with this instance</strong>';
        } else {
            document.getElementById('eip-status').innerHTML = 
                '<strong>⚠️ Using dynamic public IP (not Elastic IP)</strong>';
        }
    </script>
</body>
</html>
EOF
```

### Step 3: Associate EIP with Instance

#### 3.1 Associate via Console
```bash
# Associate EIP
EC2 Console → Elastic IPs → Select EIP → Actions → Associate Elastic IP address

# Configuration:
Resource type: Instance
Instance: WebServer-Primary
Private IP address: (auto-selected)
Reassociation: Allow this Elastic IP address to be reassociated
```

#### 3.2 Associate via CLI
```bash
# Get instance ID
INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=WebServer-Primary" \
    --query 'Reservations[0].Instances[0].InstanceId' \
    --output text)

# Associate EIP with instance
aws ec2 associate-address \
    --instance-id $INSTANCE_ID \
    --allocation-id eipalloc-12345678

# Verify association
aws ec2 describe-addresses \
    --allocation-ids eipalloc-12345678
```

### Step 4: Test EIP Functionality

#### 4.1 Basic Connectivity Test
```bash
# Test web server access via EIP
EIP_ADDRESS="54.123.45.67"
curl http://$EIP_ADDRESS

# Should return the web page
```

#### 4.2 Instance Stop/Start Test
```bash
# Stop instance
aws ec2 stop-instances --instance-ids $INSTANCE_ID

# Wait for instance to stop
aws ec2 wait instance-stopped --instance-ids $INSTANCE_ID

# Start instance
aws ec2 start-instances --instance-ids $INSTANCE_ID

# Wait for instance to start
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# Test that EIP is still associated
curl http://$EIP_ADDRESS
# Should still work with same IP
```

## 🔄 High Availability with EIP

### Step 1: Create Backup Instance

#### 1.1 Launch Secondary Instance
```bash
# Launch backup instance
Name: WebServer-Backup
AMI: Amazon Linux 2023
Instance Type: t3.micro
VPC: Default VPC
Subnet: Different AZ public subnet
Security Group: WebServer-SG

# Same user data as primary instance
```

### Step 2: Implement Failover Script

#### 2.1 Create Failover Script
```bash
#!/bin/bash
# failover.sh - EIP failover script

# Configuration
EIP_ALLOCATION_ID="eipalloc-12345678"
PRIMARY_INSTANCE_ID="i-primary123"
BACKUP_INSTANCE_ID="i-backup456"
REGION="us-east-1"

# Function to check instance health
check_instance_health() {
    local instance_id=$1
    local status=$(aws ec2 describe-instance-status \
        --instance-ids $instance_id \
        --region $REGION \
        --query 'InstanceStatuses[0].InstanceStatus.Status' \
        --output text 2>/dev/null)
    
    if [ "$status" = "ok" ]; then
        return 0
    else
        return 1
    fi
}

# Function to associate EIP
associate_eip() {
    local instance_id=$1
    echo "Associating EIP with instance: $instance_id"
    
    aws ec2 associate-address \
        --instance-id $instance_id \
        --allocation-id $EIP_ALLOCATION_ID \
        --region $REGION
    
    if [ $? -eq 0 ]; then
        echo "EIP successfully associated with $instance_id"
        return 0
    else
        echo "Failed to associate EIP with $instance_id"
        return 1
    fi
}

# Main failover logic
echo "Starting health check..."

if check_instance_health $PRIMARY_INSTANCE_ID; then
    echo "Primary instance is healthy"
else
    echo "Primary instance is unhealthy, initiating failover..."
    
    # Start backup instance if not running
    aws ec2 start-instances --instance-ids $BACKUP_INSTANCE_ID --region $REGION
    
    # Wait for backup instance to be running
    echo "Waiting for backup instance to start..."
    aws ec2 wait instance-running --instance-ids $BACKUP_INSTANCE_ID --region $REGION
    
    # Associate EIP with backup instance
    if associate_eip $BACKUP_INSTANCE_ID; then
        echo "Failover completed successfully"
        
        # Send notification (optional)
        aws sns publish \
            --topic-arn "arn:aws:sns:us-east-1:123456789012:failover-alerts" \
            --message "EIP failover completed. Traffic now routed to backup instance." \
            --region $REGION
    else
        echo "Failover failed"
        exit 1
    fi
fi
```

### Step 3: Automate with CloudWatch

#### 3.1 Create CloudWatch Alarm
```bash
# Create alarm for instance health
aws cloudwatch put-metric-alarm \
    --alarm-name "WebServer-Primary-Health" \
    --alarm-description "Monitor primary web server health" \
    --metric-name StatusCheckFailed_Instance \
    --namespace AWS/EC2 \
    --statistic Maximum \
    --period 300 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --dimensions Name=InstanceId,Value=$PRIMARY_INSTANCE_ID \
    --evaluation-periods 2 \
    --alarm-actions "arn:aws:sns:us-east-1:123456789012:failover-alerts"
```

## 💰 Cost Management

### EIP Pricing
```bash
# EIP Costs:
- Free when associated with running instance
- $0.005 per hour when not associated with running instance
- $0.005 per hour for additional EIPs beyond first one per instance

# Monthly cost examples:
# 1 EIP associated with running instance: $0
# 1 EIP not associated: $0.005 × 24 × 30 = $3.60/month
# 2 EIPs on same instance: $0.005 × 24 × 30 = $3.60/month (for 2nd EIP)
```

### Cost Optimization Best Practices
```bash
# 1. Release unused EIPs immediately
aws ec2 release-address --allocation-id eipalloc-unused123

# 2. Monitor EIP usage
aws ec2 describe-addresses \
    --query 'Addresses[?InstanceId==null].[PublicIp,AllocationId]' \
    --output table

# 3. Use Auto Scaling with dynamic IPs when possible
# 4. Consider Application Load Balancer for high availability
# 5. Implement automated EIP management
```

## 🔍 Monitoring and Alerting

### CloudWatch Metrics
```bash
# Key metrics to monitor:
- EC2 Instance Status Checks
- Network In/Out
- CPU Utilization
- Custom application metrics

# Create dashboard for EIP monitoring
aws cloudwatch put-dashboard \
    --dashboard-name "EIP-Monitoring" \
    --dashboard-body file://eip-dashboard.json
```

### EIP Usage Tracking
```bash
# Script to track EIP usage
#!/bin/bash
# eip-usage-report.sh

echo "=== Elastic IP Usage Report ==="
echo "Date: $(date)"
echo

# List all EIPs
aws ec2 describe-addresses \
    --query 'Addresses[].[PublicIp,AllocationId,InstanceId,AssociationId]' \
    --output table

echo
echo "=== Unassociated EIPs (Incurring Charges) ==="
aws ec2 describe-addresses \
    --query 'Addresses[?InstanceId==null].[PublicIp,AllocationId]' \
    --output table

# Calculate potential monthly cost
UNASSOCIATED_COUNT=$(aws ec2 describe-addresses \
    --query 'length(Addresses[?InstanceId==null])' \
    --output text)

MONTHLY_COST=$(echo "$UNASSOCIATED_COUNT * 0.005 * 24 * 30" | bc -l)
echo
echo "Estimated monthly cost for unassociated EIPs: \$$(printf "%.2f" $MONTHLY_COST)"
```

## 📚 Interview Questions & Answers

### Fresher Level (1-10)

**Q1: What is an Elastic IP address?**
A: Elastic IP is a static public IPv4 address that can be associated with EC2 instances. Unlike regular public IPs that change when instances stop/start, EIPs remain constant.

**Q2: When should you use Elastic IP?**
A: Use EIP for high availability setups, when you need consistent public IP for DNS/whitelisting, for quick failover scenarios, or when external services require static IP addresses.

**Q3: How much does Elastic IP cost?**
A: EIP is free when associated with a running instance. It costs $0.005/hour when not associated or when you have multiple EIPs on the same instance.

**Q4: What happens to EIP when instance is stopped?**
A: EIP remains associated with the stopped instance and you're not charged for it. When instance restarts, it keeps the same EIP.

**Q5: How many Elastic IPs can you have?**
A: By default, you can have 5 EIPs per region. This limit can be increased by requesting AWS support.

### Intermediate Level (6-15)

**Q6: How do you implement high availability using EIP?**
A: Create primary and backup instances, monitor primary instance health, automatically reassociate EIP to backup instance when primary fails, and implement health checks and alerting.

**Q7: What are the limitations of Elastic IP?**
A: Limited to 5 per region by default, costs money when unassociated, IPv4 only (no IPv6), regional resource (can't move between regions), and doesn't provide load balancing.

**Q8: How do you automate EIP failover?**
A: Use CloudWatch alarms to monitor instance health, Lambda functions to reassociate EIP, SNS for notifications, and scripts for health checking and failover logic.

**Q9: What's the difference between EIP and Load Balancer for HA?**
A: EIP provides single point of access with manual/automated failover, while Load Balancer automatically distributes traffic across healthy instances with built-in health checks.

**Q10: How do you troubleshoot EIP association issues?**
A: Check instance state, verify VPC/subnet configuration, ensure proper IAM permissions, check for existing associations, and review security group rules.

## 🔑 Key Takeaways

- **Static IP**: EIP provides consistent public IP address
- **High Availability**: Enables quick failover scenarios
- **Cost Awareness**: Free when associated, charged when idle
- **Automation**: Implement automated failover for production systems
- **Monitoring**: Essential for maintaining service availability
- **Alternatives**: Consider ALB/NLB for better scalability

## 🚀 Next Steps

- Day 21: AWS Lambda and Serverless Computing
- Day 22: Git and GitHub Advanced Features
- Implement EIP with Auto Scaling Groups

---

**Hands-on Completed:** ✅ Elastic IP Management, High Availability Setup, Cost Optimization  
**Duration:** 2-3 hours  
**Difficulty:** Intermediate