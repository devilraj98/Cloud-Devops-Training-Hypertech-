# Day 40: CloudWatch Fundamentals - Monitoring & Observability

## Learning Objectives
- Master CloudWatch metrics, alarms, and dashboards
- Implement custom metrics and log management
- Set up CloudWatch Logs and log insights
- Configure CloudWatch Events and automation
- Integrate SNS for alerting and notifications

## Topics Covered

### 1. CloudWatch Metrics and Alarms
```python
# Custom CloudWatch Metrics with Python
import boto3
import json
from datetime import datetime

cloudwatch = boto3.client('cloudwatch')

# Put custom metric
def put_custom_metric(metric_name, value, unit='Count', namespace='MyApp'):
    try:
        response = cloudwatch.put_metric_data(
            Namespace=namespace,
            MetricData=[
                {
                    'MetricName': metric_name,
                    'Value': value,
                    'Unit': unit,
                    'Timestamp': datetime.utcnow(),
                    'Dimensions': [
                        {
                            'Name': 'Environment',
                            'Value': 'Production'
                        },
                        {
                            'Name': 'Application',
                            'Value': 'WebApp'
                        }
                    ]
                }
            ]
        )
        print(f"Metric {metric_name} sent successfully")
        return response
    except Exception as e:
        print(f"Error sending metric: {e}")

# Example usage
put_custom_metric('UserLogins', 150, 'Count')
put_custom_metric('ResponseTime', 250, 'Milliseconds')
put_custom_metric('ErrorRate', 2.5, 'Percent')
```

### 2. CloudWatch Alarms Configuration
```yaml
# CloudFormation template for CloudWatch Alarms
AWSTemplateFormatVersion: '2010-09-09'
Description: 'CloudWatch Alarms for Web Application'

Parameters:
  EnvironmentName:
    Type: String
    Default: production
  
  SNSTopicArn:
    Type: String
    Description: SNS Topic ARN for notifications

Resources:
  # High CPU Utilization Alarm
  HighCPUAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-HighCPUUtilization'
      AlarmDescription: 'Alarm when CPU exceeds 80%'
      MetricName: CPUUtilization
      Namespace: AWS/EC2
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: InstanceId
          Value: !Ref WebServerInstance
      AlarmActions:
        - !Ref SNSTopicArn
      OKActions:
        - !Ref SNSTopicArn
      TreatMissingData: breaching

  # Application Load Balancer Target Response Time
  ALBResponseTimeAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-ALB-HighResponseTime'
      AlarmDescription: 'Alarm when ALB response time exceeds 2 seconds'
      MetricName: TargetResponseTime
      Namespace: AWS/ApplicationELB
      Statistic: Average
      Period: 300
      EvaluationPeriods: 3
      Threshold: 2
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: LoadBalancer
          Value: !GetAtt ApplicationLoadBalancer.LoadBalancerFullName
      AlarmActions:
        - !Ref SNSTopicArn

  # RDS Database Connection Alarm
  RDSConnectionAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-RDS-HighConnections'
      AlarmDescription: 'Alarm when RDS connections exceed 80% of max'
      MetricName: DatabaseConnections
      Namespace: AWS/RDS
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 80
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: DBInstanceIdentifier
          Value: !Ref DatabaseInstance
      AlarmActions:
        - !Ref SNSTopicArn

  # Custom Application Metrics Alarm
  CustomErrorRateAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${EnvironmentName}-Application-HighErrorRate'
      AlarmDescription: 'Alarm when application error rate exceeds 5%'
      MetricName: ErrorRate
      Namespace: MyApp
      Statistic: Average
      Period: 300
      EvaluationPeriods: 2
      Threshold: 5
      ComparisonOperator: GreaterThanThreshold
      Dimensions:
        - Name: Environment
          Value: !Ref EnvironmentName
      AlarmActions:
        - !Ref SNSTopicArn
      TreatMissingData: notBreaching
```

### 3. CloudWatch Dashboards
```json
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/EC2", "CPUUtilization", "InstanceId", "i-1234567890abcdef0" ],
          [ ".", "NetworkIn", ".", "." ],
          [ ".", "NetworkOut", ".", "." ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-west-2",
        "title": "EC2 Instance Metrics",
        "period": 300,
        "stat": "Average"
      }
    },
    {
      "type": "metric",
      "x": 12,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/my-load-balancer/50dc6c495c0c9188" ],
          [ ".", "TargetResponseTime", ".", "." ],
          [ ".", "HTTPCode_Target_2XX_Count", ".", "." ],
          [ ".", "HTTPCode_Target_4XX_Count", ".", "." ],
          [ ".", "HTTPCode_Target_5XX_Count", ".", "." ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-west-2",
        "title": "Application Load Balancer Metrics",
        "period": 300
      }
    },
    {
      "type": "log",
      "x": 0,
      "y": 6,
      "width": 24,
      "height": 6,
      "properties": {
        "query": "SOURCE '/aws/lambda/my-function' | fields @timestamp, @message\n| filter @message like /ERROR/\n| sort @timestamp desc\n| limit 100",
        "region": "us-west-2",
        "title": "Recent Errors",
        "view": "table"
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 12,
      "width": 8,
      "height": 6,
      "properties": {
        "metrics": [
          [ "MyApp", "UserLogins", "Environment", "Production" ],
          [ ".", "ActiveUsers", ".", "." ],
          [ ".", "PageViews", ".", "." ]
        ],
        "view": "singleValue",
        "region": "us-west-2",
        "title": "Application KPIs",
        "period": 300,
        "stat": "Sum"
      }
    }
  ]
}
```

## Hands-on Labs

### Lab 1: Setting Up Comprehensive Monitoring
```bash
#!/bin/bash
# setup-monitoring.sh

# Create SNS topic for alerts
aws sns create-topic --name production-alerts
SNS_TOPIC_ARN=$(aws sns list-topics --query 'Topics[?contains(TopicArn, `production-alerts`)].TopicArn' --output text)

# Subscribe email to SNS topic
aws sns subscribe \
    --topic-arn $SNS_TOPIC_ARN \
    --protocol email \
    --notification-endpoint admin@company.com

# Create CloudWatch Log Group
aws logs create-log-group --log-group-name /aws/ec2/application

# Create custom dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "Production-Overview" \
    --dashboard-body file://dashboard-config.json

# Create CloudWatch alarms
aws cloudwatch put-metric-alarm \
    --alarm-name "High-CPU-Usage" \
    --alarm-description "Alarm when CPU exceeds 80%" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions $SNS_TOPIC_ARN \
    --dimensions Name=InstanceId,Value=i-1234567890abcdef0

echo "Monitoring setup completed!"
```

### Lab 2: Custom Metrics Application
```python
# app_monitoring.py
import boto3
import time
import random
import json
from datetime import datetime
import logging

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ApplicationMonitoring:
    def __init__(self, namespace='MyWebApp'):
        self.cloudwatch = boto3.client('cloudwatch')
        self.logs_client = boto3.client('logs')
        self.namespace = namespace
        self.log_group = '/aws/application/webapp'
        
    def send_metric(self, metric_name, value, unit='Count', dimensions=None):
        """Send custom metric to CloudWatch"""
        try:
            metric_data = {
                'MetricName': metric_name,
                'Value': value,
                'Unit': unit,
                'Timestamp': datetime.utcnow()
            }
            
            if dimensions:
                metric_data['Dimensions'] = dimensions
                
            response = self.cloudwatch.put_metric_data(
                Namespace=self.namespace,
                MetricData=[metric_data]
            )
            logger.info(f"Sent metric {metric_name}: {value} {unit}")
            return response
        except Exception as e:
            logger.error(f"Error sending metric {metric_name}: {e}")
            
    def send_log(self, message, level='INFO'):
        """Send log message to CloudWatch Logs"""
        try:
            log_event = {
                'timestamp': int(time.time() * 1000),
                'message': json.dumps({
                    'level': level,
                    'message': message,
                    'timestamp': datetime.utcnow().isoformat()
                })
            }
            
            # Create log stream if it doesn't exist
            stream_name = f"app-{datetime.now().strftime('%Y-%m-%d')}"
            
            try:
                self.logs_client.create_log_stream(
                    logGroupName=self.log_group,
                    logStreamName=stream_name
                )
            except self.logs_client.exceptions.ResourceAlreadyExistsException:
                pass
                
            self.logs_client.put_log_events(
                logGroupName=self.log_group,
                logStreamName=stream_name,
                logEvents=[log_event]
            )
            
        except Exception as e:
            logger.error(f"Error sending log: {e}")
    
    def simulate_application_metrics(self):
        """Simulate real application metrics"""
        while True:
            # Simulate user activity
            user_logins = random.randint(10, 100)
            active_users = random.randint(50, 500)
            page_views = random.randint(100, 1000)
            
            # Simulate performance metrics
            response_time = random.uniform(100, 2000)  # milliseconds
            error_rate = random.uniform(0, 10)  # percentage
            
            # Simulate business metrics
            orders_placed = random.randint(5, 50)
            revenue = random.uniform(1000, 10000)
            
            # Send metrics
            dimensions = [
                {'Name': 'Environment', 'Value': 'Production'},
                {'Name': 'Application', 'Value': 'WebApp'}
            ]
            
            self.send_metric('UserLogins', user_logins, 'Count', dimensions)
            self.send_metric('ActiveUsers', active_users, 'Count', dimensions)
            self.send_metric('PageViews', page_views, 'Count', dimensions)
            self.send_metric('ResponseTime', response_time, 'Milliseconds', dimensions)
            self.send_metric('ErrorRate', error_rate, 'Percent', dimensions)
            self.send_metric('OrdersPlaced', orders_placed, 'Count', dimensions)
            self.send_metric('Revenue', revenue, 'None', dimensions)
            
            # Send logs
            if error_rate > 5:
                self.send_log(f"High error rate detected: {error_rate}%", 'ERROR')
            
            if response_time > 1500:
                self.send_log(f"High response time: {response_time}ms", 'WARNING')
                
            self.send_log(f"Application metrics sent - Users: {active_users}, Orders: {orders_placed}")
            
            # Wait before next iteration
            time.sleep(60)  # Send metrics every minute

if __name__ == "__main__":
    monitor = ApplicationMonitoring()
    monitor.simulate_application_metrics()
```

### Lab 3: CloudWatch Insights Queries
```sql
-- Find all ERROR logs in the last hour
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

-- Analyze response times by endpoint
fields @timestamp, @message
| filter @message like /response_time/
| parse @message "endpoint=* response_time=*" as endpoint, response_time
| stats avg(response_time), max(response_time), min(response_time) by endpoint
| sort avg desc

-- Count errors by type
fields @timestamp, @message
| filter @message like /ERROR/
| parse @message "error_type=*" as error_type
| stats count() by error_type
| sort count desc

-- Monitor user activity patterns
fields @timestamp, @message
| filter @message like /user_login/
| parse @message "user_id=* action=*" as user_id, action
| stats count() by bin(5m)
| sort @timestamp desc

-- Database query performance analysis
fields @timestamp, @message
| filter @message like /database_query/
| parse @message "query_time=* table=*" as query_time, table
| stats avg(query_time), max(query_time) by table
| sort avg desc

-- API endpoint usage statistics
fields @timestamp, @message
| filter @message like /api_request/
| parse @message "method=* endpoint=* status_code=*" as method, endpoint, status_code
| stats count() by endpoint, status_code
| sort count desc
```

## Real-world Scenarios

### Scenario 1: Application Performance Monitoring
```python
# performance_monitor.py
import boto3
import psutil
import requests
import time
from datetime import datetime

class PerformanceMonitor:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        
    def monitor_system_resources(self):
        """Monitor system-level resources"""
        # CPU Usage
        cpu_percent = psutil.cpu_percent(interval=1)
        self.send_metric('SystemCPUUtilization', cpu_percent, 'Percent')
        
        # Memory Usage
        memory = psutil.virtual_memory()
        memory_percent = memory.percent
        self.send_metric('SystemMemoryUtilization', memory_percent, 'Percent')
        
        # Disk Usage
        disk = psutil.disk_usage('/')
        disk_percent = (disk.used / disk.total) * 100
        self.send_metric('SystemDiskUtilization', disk_percent, 'Percent')
        
        # Network I/O
        network = psutil.net_io_counters()
        self.send_metric('NetworkBytesReceived', network.bytes_recv, 'Bytes')
        self.send_metric('NetworkBytesSent', network.bytes_sent, 'Bytes')
        
    def monitor_application_health(self, endpoints):
        """Monitor application endpoint health"""
        for endpoint in endpoints:
            try:
                start_time = time.time()
                response = requests.get(endpoint['url'], timeout=10)
                response_time = (time.time() - start_time) * 1000
                
                # Send response time metric
                self.send_metric(
                    f"EndpointResponseTime",
                    response_time,
                    'Milliseconds',
                    [{'Name': 'Endpoint', 'Value': endpoint['name']}]
                )
                
                # Send availability metric
                availability = 1 if response.status_code == 200 else 0
                self.send_metric(
                    f"EndpointAvailability",
                    availability,
                    'Count',
                    [{'Name': 'Endpoint', 'Value': endpoint['name']}]
                )
                
                print(f"✅ {endpoint['name']}: {response.status_code} - {response_time:.2f}ms")
                
            except Exception as e:
                print(f"❌ {endpoint['name']}: Error - {e}")
                self.send_metric(
                    f"EndpointAvailability",
                    0,
                    'Count',
                    [{'Name': 'Endpoint', 'Value': endpoint['name']}]
                )
    
    def send_metric(self, metric_name, value, unit, dimensions=None):
        """Send metric to CloudWatch"""
        try:
            metric_data = {
                'MetricName': metric_name,
                'Value': value,
                'Unit': unit,
                'Timestamp': datetime.utcnow()
            }
            
            if dimensions:
                metric_data['Dimensions'] = dimensions
                
            self.cloudwatch.put_metric_data(
                Namespace='CustomApp/Performance',
                MetricData=[metric_data]
            )
        except Exception as e:
            print(f"Error sending metric: {e}")

# Usage
if __name__ == "__main__":
    monitor = PerformanceMonitor()
    
    endpoints = [
        {'name': 'Homepage', 'url': 'https://myapp.com/'},
        {'name': 'API Health', 'url': 'https://api.myapp.com/health'},
        {'name': 'Database Health', 'url': 'https://api.myapp.com/db-health'}
    ]
    
    while True:
        monitor.monitor_system_resources()
        monitor.monitor_application_health(endpoints)
        time.sleep(300)  # Monitor every 5 minutes
```

## Interview Questions

### Beginner Level
1. **Q: What is CloudWatch and what are its main components?**
   A: CloudWatch is AWS's monitoring service with components including Metrics (performance data), Alarms (notifications), Dashboards (visualization), Logs (log management), and Events (automation triggers).

2. **Q: What is the difference between CloudWatch Metrics and CloudWatch Logs?**
   A: Metrics are numerical data points over time (CPU usage, request count), while Logs are text-based records of events and activities from applications and services.

3. **Q: How do you create a CloudWatch alarm?**
   A: Define a metric to monitor, set a threshold value, specify comparison operator, configure evaluation periods, and set actions (SNS notifications, Auto Scaling, etc.).

### Intermediate Level
4. **Q: Explain CloudWatch custom metrics and when to use them.**
   A: Custom metrics allow you to publish application-specific data to CloudWatch using the PutMetricData API, useful for business metrics, application performance, and custom KPIs.

5. **Q: How do you optimize CloudWatch costs?**
   A: Use appropriate metric resolution, set log retention periods, use log filtering, implement metric filters, and avoid unnecessary custom metrics or high-frequency publishing.

6. **Q: What are CloudWatch Insights and how do they help?**
   A: CloudWatch Insights provides interactive log analytics with SQL-like queries, enabling real-time log analysis, pattern detection, and troubleshooting across multiple log groups.

### Advanced Level
7. **Q: How would you implement comprehensive application monitoring strategy?**
   A: Combine infrastructure metrics, custom application metrics, distributed tracing, log aggregation, synthetic monitoring, and alerting with proper dashboards and runbooks.

8. **Q: Explain CloudWatch Events and their use cases.**
   A: CloudWatch Events (now EventBridge) enables event-driven automation, triggering Lambda functions, SNS notifications, or other AWS services based on AWS service events or custom events.

9. **Q: How do you implement multi-dimensional metrics in CloudWatch?**
   A: Use dimensions to add metadata to metrics, enabling filtering and aggregation across different attributes like environment, instance type, or application version.

10. **Q: Describe a complete monitoring and alerting strategy for a production application.**
    A: Include infrastructure monitoring, application performance monitoring, business metrics, log analysis, synthetic monitoring, escalation procedures, and integration with incident management systems.

## Key Takeaways
- CloudWatch provides comprehensive monitoring for AWS resources and applications
- Custom metrics enable application-specific monitoring and business KPIs
- Proper alarm configuration prevents alert fatigue and ensures timely notifications
- CloudWatch Insights enables powerful log analysis and troubleshooting
- Dashboards provide centralized visibility into system health and performance
- Cost optimization requires careful consideration of metric frequency and retention