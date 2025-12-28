# Day 21: AWS Lambda and Serverless Computing

## 📚 Learning Objectives
- Understand serverless computing concepts and benefits
- Master AWS Lambda function creation and configuration
- Implement event-driven architectures
- Practice Lambda with different triggers (S3, API Gateway, CloudWatch)
- Understand Lambda pricing and optimization

## ☁️ Serverless Computing Overview

### What is Serverless?
Serverless computing allows you to run code without provisioning or managing servers. You pay only for compute time consumed.

### Key Benefits
- **No Server Management**: AWS handles infrastructure
- **Automatic Scaling**: Scales from zero to thousands of requests
- **Pay-per-Use**: Only pay for execution time
- **High Availability**: Built-in fault tolerance
- **Event-Driven**: Responds to events automatically

## 🚀 AWS Lambda Deep Dive

### Lambda Characteristics
- **Runtime Support**: Python, Node.js, Java, C#, Go, Ruby, PowerShell
- **Memory**: 128 MB to 10,240 MB
- **Timeout**: Up to 15 minutes
- **Concurrent Executions**: 1,000 by default (can be increased)
- **Package Size**: 50 MB (zipped), 250 MB (unzipped)

## 🛠️ Hands-on Implementation

### Step 1: Create Your First Lambda Function

#### 1.1 Simple Hello World Function
```bash
# Navigate to Lambda Console
AWS Console → Lambda → Create Function

# Configuration:
Function name: HelloWorldFunction
Runtime: Python 3.11
Architecture: x86_64
Execution role: Create a new role with basic Lambda permissions
```

#### 1.2 Function Code (Python)
```python
import json
import datetime

def lambda_handler(event, context):
    """
    Simple Hello World Lambda function
    """
    
    # Get current timestamp
    current_time = datetime.datetime.now().isoformat()
    
    # Extract information from event
    source_ip = event.get('requestContext', {}).get('identity', {}).get('sourceIp', 'unknown')
    user_agent = event.get('headers', {}).get('User-Agent', 'unknown')
    
    # Create response
    response_body = {
        'message': 'Hello from AWS Lambda!',
        'timestamp': current_time,
        'source_ip': source_ip,
        'user_agent': user_agent,
        'event_received': event
    }
    
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps(response_body, indent=2)
    }
```

#### 1.3 Test the Function
```bash
# Create test event
Test Event Name: HelloWorldTest
Event Template: API Gateway AWS Proxy

# Test JSON:
{
  "httpMethod": "GET",
  "path": "/hello",
  "headers": {
    "User-Agent": "Mozilla/5.0"
  },
  "requestContext": {
    "identity": {
      "sourceIp": "192.168.1.100"
    }
  }
}
```

### Step 2: S3 Event-Triggered Lambda

#### 2.1 Create S3 Processing Function
```python
import json
import boto3
import urllib.parse
from datetime import datetime

# Initialize AWS clients
s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

def lambda_handler(event, context):
    """
    Process S3 events - log file uploads to DynamoDB
    """
    
    # Process each record in the event
    for record in event['Records']:
        # Get bucket and object key
        bucket = record['s3']['bucket']['name']
        key = urllib.parse.unquote_plus(record['s3']['object']['key'])
        
        print(f"Processing file: {key} from bucket: {bucket}")
        
        try:
            # Get object metadata
            response = s3_client.head_object(Bucket=bucket, Key=key)
            file_size = response['ContentLength']
            last_modified = response['LastModified'].isoformat()
            
            # Log to DynamoDB (optional)
            table = dynamodb.Table('FileProcessingLog')
            table.put_item(
                Item={
                    'file_key': key,
                    'bucket_name': bucket,
                    'file_size': file_size,
                    'processed_at': datetime.now().isoformat(),
                    'last_modified': last_modified,
                    'status': 'processed'
                }
            )
            
            print(f"Successfully processed {key}")
            
        except Exception as e:
            print(f"Error processing {key}: {str(e)}")
            raise e
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'Successfully processed {len(event["Records"])} files')
    }
```

#### 2.2 Configure S3 Trigger
```bash
# Create S3 bucket
aws s3 mb s3://lambda-demo-bucket-unique-name

# Add S3 trigger to Lambda function
Lambda Console → HelloWorldFunction → Add trigger
Trigger: S3
Bucket: lambda-demo-bucket-unique-name
Event type: All object create events
Prefix: uploads/
Suffix: .txt
```

### Step 3: API Gateway Integration

#### 3.1 Create REST API
```bash
# Create API Gateway
API Gateway Console → Create API → REST API → Build

# Configuration:
API name: LambdaAPI
Description: API for Lambda functions
Endpoint Type: Regional
```

#### 3.2 Create Resource and Method
```bash
# Create resource
Actions → Create Resource
Resource Name: hello
Resource Path: /hello

# Create GET method
Actions → Create Method → GET
Integration type: Lambda Function
Lambda Region: us-east-1
Lambda Function: HelloWorldFunction
Use Default Timeout: Yes
```

#### 3.3 Deploy API
```bash
# Deploy API
Actions → Deploy API
Deployment stage: [New Stage]
Stage name: prod
Stage description: Production stage
Deployment description: Initial deployment

# Note the Invoke URL: https://api-id.execute-api.region.amazonaws.com/prod
```

### Step 4: CloudWatch Event-Triggered Lambda

#### 4.1 Create Scheduled Lambda Function
```python
import json
import boto3
from datetime import datetime

# Initialize CloudWatch client
cloudwatch = boto3.client('cloudwatch')
ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    """
    Scheduled function to monitor EC2 instances and send custom metrics
    """
    
    try:
        # Get all running EC2 instances
        response = ec2.describe_instances(
            Filters=[
                {
                    'Name': 'instance-state-name',
                    'Values': ['running']
                }
            ]
        )
        
        running_instances = 0
        for reservation in response['Reservations']:
            running_instances += len(reservation['Instances'])
        
        # Send custom metric to CloudWatch
        cloudwatch.put_metric_data(
            Namespace='Custom/EC2',
            MetricData=[
                {
                    'MetricName': 'RunningInstances',
                    'Value': running_instances,
                    'Unit': 'Count',
                    'Timestamp': datetime.now()
                }
            ]
        )
        
        print(f"Found {running_instances} running instances")
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'message': 'Metrics sent successfully',
                'running_instances': running_instances,
                'timestamp': datetime.now().isoformat()
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({
                'error': str(e)
            })
        }
```

#### 4.2 Create CloudWatch Event Rule
```bash
# Create EventBridge rule
EventBridge Console → Rules → Create rule

# Configuration:
Name: EC2MonitoringSchedule
Description: Trigger Lambda every 5 minutes
Event bus: default
Rule type: Schedule

# Schedule:
Schedule expression: rate(5 minutes)

# Target:
Target type: AWS service
Service: Lambda function
Function: EC2MonitoringFunction
```

## 🧪 Testing Lambda Functions

### Test 1: API Gateway Integration
```bash
# Test API endpoint
API_URL="https://your-api-id.execute-api.us-east-1.amazonaws.com/prod/hello"
curl -X GET $API_URL

# Expected response: JSON with hello message and metadata
```

### Test 2: S3 Event Trigger
```bash
# Upload file to S3 bucket
echo "Hello Lambda!" > test-file.txt
aws s3 cp test-file.txt s3://lambda-demo-bucket-unique-name/uploads/

# Check Lambda logs
aws logs describe-log-groups --log-group-name-prefix "/aws/lambda/"
aws logs get-log-events --log-group-name "/aws/lambda/S3ProcessorFunction" --log-stream-name "latest"
```

### Test 3: Scheduled Function
```bash
# Check CloudWatch metrics
aws cloudwatch get-metric-statistics \
    --namespace "Custom/EC2" \
    --metric-name "RunningInstances" \
    --start-time 2024-01-01T00:00:00Z \
    --end-time 2024-01-01T23:59:59Z \
    --period 300 \
    --statistics Average
```

## 📊 Lambda Monitoring and Optimization

### CloudWatch Metrics
```bash
# Key Lambda metrics:
- Duration: Execution time
- Invocations: Number of function calls
- Errors: Number of failed executions
- Throttles: Number of throttled invocations
- ConcurrentExecutions: Number of concurrent executions
- DeadLetterErrors: Failed async invocations
```

### Performance Optimization
```python
# Optimized Lambda function example
import json
import boto3
import os
from datetime import datetime

# Initialize clients outside handler (connection reuse)
s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')

# Use environment variables
TABLE_NAME = os.environ.get('TABLE_NAME', 'DefaultTable')
BUCKET_NAME = os.environ.get('BUCKET_NAME', 'default-bucket')

def lambda_handler(event, context):
    """
    Optimized Lambda function with best practices
    """
    
    # Early return for warm-up calls
    if event.get('source') == 'aws.events' and event.get('detail-type') == 'Scheduled Event':
        return {'statusCode': 200, 'body': 'Warm-up successful'}
    
    try:
        # Process event efficiently
        result = process_event(event)
        
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
        
    except Exception as e:
        # Log error details
        print(f"Error processing event: {str(e)}")
        print(f"Event: {json.dumps(event)}")
        
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }

def process_event(event):
    """
    Separate business logic for better testing
    """
    # Business logic here
    return {'processed': True, 'timestamp': datetime.now().isoformat()}
```

## 💰 Lambda Pricing and Cost Optimization

### Pricing Model
```bash
# Lambda Pricing (us-east-1):
# Requests: $0.20 per 1M requests
# Duration: $0.0000166667 per GB-second

# Example calculation:
# Function: 512 MB memory, 100ms duration, 1M invocations/month
# Memory cost: (512/1024) × 0.1 × 1,000,000 × $0.0000166667 = $0.83
# Request cost: 1,000,000 × $0.20/1,000,000 = $0.20
# Total: $1.03/month
```

### Cost Optimization Strategies
```bash
# 1. Right-size memory allocation
# 2. Optimize function duration
# 3. Use provisioned concurrency wisely
# 4. Implement efficient error handling
# 5. Use Lambda layers for shared code
# 6. Monitor and analyze CloudWatch metrics
```

## 🔒 Security Best Practices

### IAM Roles and Policies
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject"
            ],
            "Resource": "arn:aws:s3:::lambda-demo-bucket/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "dynamodb:PutItem",
                "dynamodb:GetItem"
            ],
            "Resource": "arn:aws:dynamodb:*:*:table/FileProcessingLog"
        }
    ]
}
```

### Environment Variables and Secrets
```python
import os
import boto3
import json

def get_secret(secret_name):
    """
    Retrieve secret from AWS Secrets Manager
    """
    client = boto3.client('secretsmanager')
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return json.loads(response['SecretString'])
    except Exception as e:
        print(f"Error retrieving secret: {e}")
        return None

def lambda_handler(event, context):
    # Use environment variables
    api_key = os.environ.get('API_KEY')
    
    # Use Secrets Manager for sensitive data
    db_credentials = get_secret('prod/database/credentials')
    
    # Your function logic here
    return {'statusCode': 200}
```

## 📚 Interview Questions & Answers

### Fresher Level (1-10)

**Q1: What is AWS Lambda?**
A: AWS Lambda is a serverless compute service that runs code in response to events without provisioning or managing servers. You pay only for compute time consumed.

**Q2: What are the benefits of serverless computing?**
A: No server management, automatic scaling, pay-per-use pricing, high availability, faster time to market, and reduced operational overhead.

**Q3: What triggers can invoke Lambda functions?**
A: S3 events, API Gateway, CloudWatch Events/EventBridge, DynamoDB streams, SQS, SNS, Kinesis, ALB, CloudFront, and many other AWS services.

**Q4: What are Lambda's execution limits?**
A: 15-minute maximum timeout, 10 GB memory maximum, 512 MB /tmp storage, 1,000 concurrent executions by default, and 50 MB deployment package size (zipped).

**Q5: How does Lambda pricing work?**
A: You pay for requests ($0.20 per 1M requests) and duration (based on memory allocated and execution time). First 1M requests and 400,000 GB-seconds are free monthly.

### Intermediate Level (6-15)

**Q6: What is cold start in Lambda and how to minimize it?**
A: Cold start is the initialization time when Lambda creates new execution environment. Minimize by using provisioned concurrency, optimizing package size, choosing appropriate runtime, and keeping functions warm.

**Q7: How do you handle errors in Lambda functions?**
A: Implement try-catch blocks, use dead letter queues for failed async invocations, set up CloudWatch alarms, implement retry logic, and use proper logging for debugging.

**Q8: What are Lambda layers and when to use them?**
A: Lambda layers allow sharing code and dependencies across multiple functions. Use for common libraries, custom runtimes, shared utilities, and reducing deployment package size.

**Q9: How do you implement security in Lambda functions?**
A: Use least privilege IAM roles, encrypt environment variables, use VPC for network isolation, implement input validation, use AWS Secrets Manager for sensitive data, and enable AWS X-Ray for tracing.

**Q10: What's the difference between synchronous and asynchronous Lambda invocations?**
A: Synchronous invocations wait for response (API Gateway, ALB), while asynchronous invocations don't wait (S3, SNS). Async invocations have built-in retry logic and dead letter queue support.

## 🔑 Key Takeaways

- **Serverless Benefits**: No server management, automatic scaling, pay-per-use
- **Event-Driven**: Perfect for event-driven architectures
- **Cost-Effective**: Pay only for actual usage
- **Integration**: Seamlessly integrates with AWS services
- **Monitoring**: Use CloudWatch for comprehensive monitoring
- **Security**: Implement proper IAM roles and security practices

## 🚀 Next Steps

- Day 22: Git and GitHub Advanced Features
- Day 23: Code Quality and Testing
- Implement Lambda with containers and custom runtimes

---

**Hands-on Completed:** ✅ Lambda Functions, Event Triggers, API Gateway Integration  
**Duration:** 3-4 hours  
**Difficulty:** Intermediate