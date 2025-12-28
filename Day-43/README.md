# Day 43: Project Implementation & Deployment

## Learning Objectives
- Build complete DevOps pipeline with CI/CD automation
- Deploy infrastructure using Terraform/CloudFormation
- Implement application deployment on EKS using Helm charts
- Execute comprehensive testing strategies (unit, integration, performance)
- Set up monitoring, alerting, and observability
- Create documentation and operational runbooks

## Topics Covered

### 1. Complete CI/CD Pipeline Implementation
```yaml
# .github/workflows/main.yml
name: Complete CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  AWS_REGION: us-west-2
  EKS_CLUSTER_NAME: production-cluster
  ECR_REPOSITORY: ecommerce-app

jobs:
  code-quality:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linting
      run: npm run lint
    
    - name: Run unit tests
      run: npm run test:unit
    
    - name: Generate test coverage
      run: npm run test:coverage
    
    - name: SonarCloud Scan
      uses: SonarSource/sonarcloud-github-action@master
      env:
        GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: OWASP Dependency Check
      uses: dependency-check/Dependency-Check_Action@main
      with:
        project: 'ecommerce-app'
        path: '.'
        format: 'HTML'

  build-and-push:
    needs: [code-quality, security-scan]
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Login to Amazon ECR
      id: login-ecr
      uses: aws-actions/amazon-ecr-login@v1
    
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v4
      with:
        images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push Docker image
      id: build
      uses: docker/build-push-action@v4
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  integration-tests:
    needs: build-and-push
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      redis:
        image: redis:6
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run integration tests
      run: npm run test:integration
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/testdb
        REDIS_URL: redis://localhost:6379

  deploy-staging:
    needs: [build-and-push, integration-tests]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Update kubeconfig
      run: aws eks update-kubeconfig --region ${{ env.AWS_REGION }} --name ${{ env.EKS_CLUSTER_NAME }}
    
    - name: Deploy to staging
      run: |
        helm upgrade --install ecommerce-app ./helm-chart \
          --namespace staging \
          --create-namespace \
          --set image.tag=${{ needs.build-and-push.outputs.image-tag }} \
          --set environment=staging \
          --values values-staging.yaml \
          --wait
    
    - name: Run smoke tests
      run: |
        kubectl wait --for=condition=available --timeout=300s deployment/ecommerce-app -n staging
        npm run test:smoke -- --baseUrl=https://staging.ecommerce.com

  deploy-production:
    needs: [build-and-push, integration-tests]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v2
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ env.AWS_REGION }}
    
    - name: Update kubeconfig
      run: aws eks update-kubeconfig --region ${{ env.AWS_REGION }} --name ${{ env.EKS_CLUSTER_NAME }}
    
    - name: Blue-Green Deployment
      run: |
        # Deploy to green environment
        helm upgrade --install ecommerce-app-green ./helm-chart \
          --namespace production \
          --set image.tag=${{ needs.build-and-push.outputs.image-tag }} \
          --set environment=production \
          --set service.name=ecommerce-app-green \
          --values values-production.yaml \
          --wait
        
        # Health check
        kubectl wait --for=condition=available --timeout=600s deployment/ecommerce-app-green -n production
        
        # Switch traffic
        kubectl patch service ecommerce-app -n production -p '{"spec":{"selector":{"version":"green"}}}'
        
        # Cleanup old blue deployment after successful switch
        sleep 300
        helm uninstall ecommerce-app-blue -n production || true
    
    - name: Performance Tests
      run: |
        npm run test:performance -- --baseUrl=https://ecommerce.com
    
    - name: Notify deployment
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### 2. Infrastructure Deployment with Terraform
```hcl
# main.tf - Complete infrastructure deployment
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.11"
    }
  }
  
  backend "s3" {
    bucket         = "ecommerce-terraform-state"
    key            = "production/terraform.tfstate"
    region         = "us-west-2"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}

# VPC and Networking
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project_name}-vpc"
  cidr = var.vpc_cidr

  azs             = var.availability_zones
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs
  database_subnets = var.database_subnet_cidrs

  enable_nat_gateway = true
  enable_vpn_gateway = false
  enable_dns_hostnames = true
  enable_dns_support = true

  tags = local.common_tags
}

# EKS Cluster
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "${var.project_name}-cluster"
  cluster_version = var.kubernetes_version

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    general = {
      min_size     = var.node_group_min_size
      max_size     = var.node_group_max_size
      desired_size = var.node_group_desired_size

      instance_types = var.node_instance_types
      capacity_type  = "ON_DEMAND"

      k8s_labels = {
        Environment = var.environment
        NodeGroup   = "general"
      }
    }

    spot = {
      min_size     = 0
      max_size     = 10
      desired_size = 2

      instance_types = var.spot_instance_types
      capacity_type  = "SPOT"

      k8s_labels = {
        Environment = var.environment
        NodeGroup   = "spot"
      }
    }
  }

  tags = local.common_tags
}

# RDS Database
module "rds" {
  source = "terraform-aws-modules/rds/aws"
  version = "~> 6.0"

  identifier = "${var.project_name}-db"

  engine            = "postgres"
  engine_version    = "13.13"
  instance_class    = var.db_instance_class
  allocated_storage = var.db_allocated_storage

  db_name  = var.db_name
  username = var.db_username
  password = var.db_password

  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = module.vpc.database_subnet_group

  backup_retention_period = var.db_backup_retention
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"

  multi_az               = var.environment == "production"
  publicly_accessible    = false
  storage_encrypted      = true

  tags = local.common_tags
}

# ElastiCache Redis
resource "aws_elasticache_subnet_group" "redis" {
  name       = "${var.project_name}-redis-subnet-group"
  subnet_ids = module.vpc.private_subnets
}

resource "aws_elasticache_replication_group" "redis" {
  replication_group_id       = "${var.project_name}-redis"
  description                = "Redis cluster for ${var.project_name}"

  node_type                  = var.redis_node_type
  port                       = 6379
  parameter_group_name       = "default.redis7"

  num_cache_clusters         = var.redis_num_nodes
  automatic_failover_enabled = var.redis_num_nodes > 1
  multi_az_enabled          = var.redis_num_nodes > 1

  subnet_group_name = aws_elasticache_subnet_group.redis.name
  security_group_ids = [aws_security_group.redis.id]

  at_rest_encryption_enabled = true
  transit_encryption_enabled = true

  tags = local.common_tags
}

# Monitoring Stack
resource "helm_release" "prometheus_stack" {
  name       = "prometheus-stack"
  repository = "https://prometheus-community.github.io/helm-charts"
  chart      = "kube-prometheus-stack"
  namespace  = "monitoring"
  version    = "51.2.0"

  create_namespace = true

  values = [
    templatefile("${path.module}/helm-values/prometheus-values.yaml", {
      grafana_admin_password = var.grafana_admin_password
      storage_class         = "gp3"
      prometheus_storage    = "50Gi"
      grafana_storage      = "10Gi"
    })
  ]

  depends_on = [module.eks]
}

# Application Deployment
resource "helm_release" "ecommerce_app" {
  name       = "ecommerce-app"
  chart      = "./helm-chart"
  namespace  = "production"
  version    = "1.0.0"

  create_namespace = true

  values = [
    templatefile("${path.module}/helm-values/app-values.yaml", {
      image_tag = var.app_image_tag
      environment = var.environment
      database_host = module.rds.db_instance_endpoint
      redis_host = aws_elasticache_replication_group.redis.primary_endpoint_address
      domain_name = var.domain_name
    })
  ]

  depends_on = [module.eks, module.rds, aws_elasticache_replication_group.redis]
}

# Outputs
output "cluster_endpoint" {
  description = "Endpoint for EKS control plane"
  value       = module.eks.cluster_endpoint
}

output "cluster_name" {
  description = "Kubernetes Cluster Name"
  value       = module.eks.cluster_name
}

output "database_endpoint" {
  description = "RDS instance endpoint"
  value       = module.rds.db_instance_endpoint
  sensitive   = true
}

output "redis_endpoint" {
  description = "ElastiCache Redis endpoint"
  value       = aws_elasticache_replication_group.redis.primary_endpoint_address
  sensitive   = true
}
```

### 3. Comprehensive Testing Strategy
```javascript
// tests/integration/api.test.js
const request = require('supertest');
const app = require('../../src/app');
const { setupTestDB, cleanupTestDB } = require('../helpers/database');

describe('API Integration Tests', () => {
  beforeAll(async () => {
    await setupTestDB();
  });

  afterAll(async () => {
    await cleanupTestDB();
  });

  describe('User Management', () => {
    test('should create a new user', async () => {
      const userData = {
        email: 'test@example.com',
        password: 'password123',
        firstName: 'John',
        lastName: 'Doe'
      };

      const response = await request(app)
        .post('/api/users')
        .send(userData)
        .expect(201);

      expect(response.body).toHaveProperty('id');
      expect(response.body.email).toBe(userData.email);
      expect(response.body).not.toHaveProperty('password');
    });

    test('should authenticate user', async () => {
      const loginData = {
        email: 'test@example.com',
        password: 'password123'
      };

      const response = await request(app)
        .post('/api/auth/login')
        .send(loginData)
        .expect(200);

      expect(response.body).toHaveProperty('token');
      expect(response.body).toHaveProperty('user');
    });
  });

  describe('Product Management', () => {
    let authToken;

    beforeEach(async () => {
      const loginResponse = await request(app)
        .post('/api/auth/login')
        .send({ email: 'test@example.com', password: 'password123' });
      
      authToken = loginResponse.body.token;
    });

    test('should create a product', async () => {
      const productData = {
        name: 'Test Product',
        description: 'A test product',
        price: 29.99,
        category: 'electronics'
      };

      const response = await request(app)
        .post('/api/products')
        .set('Authorization', `Bearer ${authToken}`)
        .send(productData)
        .expect(201);

      expect(response.body.name).toBe(productData.name);
      expect(response.body.price).toBe(productData.price);
    });

    test('should search products', async () => {
      const response = await request(app)
        .get('/api/products/search?q=test')
        .expect(200);

      expect(Array.isArray(response.body.products)).toBe(true);
      expect(response.body).toHaveProperty('total');
      expect(response.body).toHaveProperty('page');
    });
  });
});

// tests/performance/load.test.js
const autocannon = require('autocannon');

describe('Performance Tests', () => {
  const baseUrl = process.env.BASE_URL || 'http://localhost:3000';

  test('API should handle concurrent requests', async () => {
    const result = await autocannon({
      url: `${baseUrl}/api/health`,
      connections: 100,
      duration: 30,
      pipelining: 1
    });

    expect(result.errors).toBe(0);
    expect(result.timeouts).toBe(0);
    expect(result.latency.mean).toBeLessThan(100); // 100ms average
    expect(result.requests.mean).toBeGreaterThan(1000); // 1000 req/sec
  });

  test('Product search should perform under load', async () => {
    const result = await autocannon({
      url: `${baseUrl}/api/products/search?q=electronics`,
      connections: 50,
      duration: 60,
      pipelining: 1
    });

    expect(result.errors).toBe(0);
    expect(result.latency.p95).toBeLessThan(500); // 95th percentile < 500ms
  });
});

// tests/e2e/user-journey.test.js
const { chromium } = require('playwright');

describe('E2E User Journey Tests', () => {
  let browser, page;

  beforeAll(async () => {
    browser = await chromium.launch();
    page = await browser.newPage();
  });

  afterAll(async () => {
    await browser.close();
  });

  test('Complete user purchase journey', async () => {
    // Navigate to homepage
    await page.goto(process.env.BASE_URL || 'http://localhost:3000');
    
    // Register new user
    await page.click('[data-testid="register-button"]');
    await page.fill('[data-testid="email-input"]', 'e2e@example.com');
    await page.fill('[data-testid="password-input"]', 'password123');
    await page.fill('[data-testid="firstName-input"]', 'E2E');
    await page.fill('[data-testid="lastName-input"]', 'User');
    await page.click('[data-testid="submit-register"]');
    
    // Search for product
    await page.fill('[data-testid="search-input"]', 'laptop');
    await page.press('[data-testid="search-input"]', 'Enter');
    
    // Add product to cart
    await page.click('[data-testid="product-card"]:first-child');
    await page.click('[data-testid="add-to-cart"]');
    
    // Proceed to checkout
    await page.click('[data-testid="cart-icon"]');
    await page.click('[data-testid="checkout-button"]');
    
    // Fill shipping information
    await page.fill('[data-testid="address-input"]', '123 Test St');
    await page.fill('[data-testid="city-input"]', 'Test City');
    await page.fill('[data-testid="zipcode-input"]', '12345');
    
    // Complete payment (test mode)
    await page.fill('[data-testid="card-number"]', '4242424242424242');
    await page.fill('[data-testid="card-expiry"]', '12/25');
    await page.fill('[data-testid="card-cvc"]', '123');
    
    await page.click('[data-testid="place-order"]');
    
    // Verify order confirmation
    await page.waitForSelector('[data-testid="order-confirmation"]');
    const orderNumber = await page.textContent('[data-testid="order-number"]');
    
    expect(orderNumber).toMatch(/^ORD-\d+$/);
  });
});
```

## Hands-on Labs

### Lab 1: Complete Application Deployment
```bash
#!/bin/bash
# deploy-complete-application.sh

set -e

PROJECT_NAME="ecommerce-platform"
ENVIRONMENT="production"
AWS_REGION="us-west-2"

echo "🚀 Starting complete application deployment..."

# Step 1: Deploy Infrastructure
echo "📦 Deploying infrastructure with Terraform..."
cd infrastructure/
terraform init
terraform plan -var-file="environments/${ENVIRONMENT}.tfvars"
terraform apply -var-file="environments/${ENVIRONMENT}.tfvars" -auto-approve

# Get infrastructure outputs
CLUSTER_NAME=$(terraform output -raw cluster_name)
DB_ENDPOINT=$(terraform output -raw database_endpoint)
REDIS_ENDPOINT=$(terraform output -raw redis_endpoint)

# Step 2: Configure kubectl
echo "⚙️ Configuring kubectl..."
aws eks update-kubeconfig --region $AWS_REGION --name $CLUSTER_NAME

# Step 3: Deploy monitoring stack
echo "📊 Deploying monitoring stack..."
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm upgrade --install prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --values helm-values/prometheus-values.yaml \
  --wait

# Step 4: Deploy application
echo "🚀 Deploying application..."
helm upgrade --install $PROJECT_NAME ./helm-chart \
  --namespace production \
  --create-namespace \
  --set image.tag=${IMAGE_TAG:-latest} \
  --set database.host=$DB_ENDPOINT \
  --set redis.host=$REDIS_ENDPOINT \
  --values helm-values/production-values.yaml \
  --wait

# Step 5: Run health checks
echo "🏥 Running health checks..."
kubectl wait --for=condition=available --timeout=300s deployment/$PROJECT_NAME -n production

# Get application URL
APP_URL=$(kubectl get ingress $PROJECT_NAME -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Step 6: Run smoke tests
echo "🧪 Running smoke tests..."
curl -f "http://$APP_URL/health" || exit 1
curl -f "http://$APP_URL/api/health" || exit 1

echo "✅ Deployment completed successfully!"
echo "🌐 Application URL: http://$APP_URL"
echo "📊 Grafana: kubectl port-forward -n monitoring svc/prometheus-stack-grafana 3000:80"
```

### Lab 2: Monitoring and Alerting Setup
```yaml
# monitoring-setup.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-alerts
  namespace: monitoring
data:
  custom-alerts.yaml: |
    groups:
    - name: application.rules
      rules:
      - alert: ApplicationDown
        expr: up{job="ecommerce-app"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
          description: "Application {{ $labels.instance }} has been down for more than 1 minute"
      
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"
      
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High response time"
          description: "95th percentile response time is {{ $value }}s"
      
      - alert: DatabaseConnectionsHigh
        expr: pg_stat_activity_count > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High database connections"
          description: "Database has {{ $value }} active connections"

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  application-dashboard.json: |
    {
      "dashboard": {
        "title": "E-Commerce Application Dashboard",
        "panels": [
          {
            "title": "Request Rate",
            "type": "timeseries",
            "targets": [
              {
                "expr": "rate(http_requests_total[5m])",
                "legendFormat": "{{method}} {{endpoint}}"
              }
            ]
          },
          {
            "title": "Error Rate",
            "type": "stat",
            "targets": [
              {
                "expr": "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m]) * 100",
                "legendFormat": "Error Rate %"
              }
            ]
          },
          {
            "title": "Response Time Percentiles",
            "type": "timeseries",
            "targets": [
              {
                "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
                "legendFormat": "50th percentile"
              },
              {
                "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
                "legendFormat": "95th percentile"
              }
            ]
          }
        ]
      }
    }
```

### Lab 3: Documentation and Runbooks
```markdown
# Production Runbook

## Application Overview
- **Name:** E-Commerce Platform
- **Environment:** Production
- **Cluster:** production-cluster (EKS)
- **Namespace:** production
- **Monitoring:** Grafana dashboard available

## Architecture
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Frontend  │────│ API Gateway │────│ Microservices│
│   (S3/CF)   │    │   (ALB)     │    │   (EKS)     │
└─────────────┘    └─────────────┘    └─────────────┘
                                              │
                   ┌─────────────┐    ┌─────────────┐
                   │  Database   │    │    Cache    │
                   │   (RDS)     │    │   (Redis)   │
                   └─────────────┘    └─────────────┘
```

## Common Operations

### Deployment
```bash
# Deploy new version
helm upgrade ecommerce-platform ./helm-chart \
  --namespace production \
  --set image.tag=v1.2.3 \
  --wait

# Rollback deployment
helm rollback ecommerce-platform -n production

# Check deployment status
kubectl rollout status deployment/ecommerce-platform -n production
```

### Scaling
```bash
# Scale application
kubectl scale deployment ecommerce-platform --replicas=10 -n production

# Check HPA status
kubectl get hpa -n production

# View resource usage
kubectl top pods -n production
```

### Troubleshooting

#### Application Not Responding
1. Check pod status: `kubectl get pods -n production`
2. Check logs: `kubectl logs -f deployment/ecommerce-platform -n production`
3. Check service endpoints: `kubectl get endpoints -n production`
4. Check ingress: `kubectl describe ingress ecommerce-platform -n production`

#### High Error Rate
1. Check application logs for errors
2. Verify database connectivity
3. Check Redis connectivity
4. Review recent deployments
5. Check resource utilization

#### Database Issues
1. Check RDS metrics in CloudWatch
2. Verify connection pool settings
3. Check for long-running queries
4. Review database logs

### Monitoring URLs
- **Grafana:** https://grafana.company.com
- **Prometheus:** https://prometheus.company.com
- **Application:** https://ecommerce.company.com

### Emergency Contacts
- **On-call Engineer:** +1-555-0123
- **DevOps Team:** devops@company.com
- **Database Admin:** dba@company.com

### Escalation Procedures
1. **Severity 1 (Critical):** Page on-call immediately
2. **Severity 2 (High):** Notify team within 15 minutes
3. **Severity 3 (Medium):** Create ticket, notify during business hours
```

## Real-world Scenarios

### Scenario 1: Production Incident Response
```bash
#!/bin/bash
# incident-response.sh

INCIDENT_ID="INC-$(date +%Y%m%d-%H%M%S)"
NAMESPACE="production"

echo "🚨 Incident Response Started: $INCIDENT_ID"

# Step 1: Assess the situation
echo "📊 Gathering system status..."
kubectl get pods -n $NAMESPACE
kubectl get services -n $NAMESPACE
kubectl top nodes
kubectl top pods -n $NAMESPACE

# Step 2: Check recent changes
echo "🔍 Checking recent deployments..."
kubectl rollout history deployment/ecommerce-platform -n $NAMESPACE

# Step 3: Collect logs
echo "📝 Collecting logs..."
mkdir -p /tmp/incident-logs/$INCIDENT_ID
kubectl logs deployment/ecommerce-platform -n $NAMESPACE --tail=1000 > /tmp/incident-logs/$INCIDENT_ID/app-logs.txt
kubectl get events -n $NAMESPACE --sort-by=.metadata.creationTimestamp > /tmp/incident-logs/$INCIDENT_ID/events.txt

# Step 4: Check monitoring
echo "📈 Checking metrics..."
curl -s "http://prometheus:9090/api/v1/query?query=up{job=\"ecommerce-app\"}" > /tmp/incident-logs/$INCIDENT_ID/app-status.json

# Step 5: Immediate mitigation (if needed)
read -p "Do you want to rollback to previous version? (y/n): " rollback
if [ "$rollback" = "y" ]; then
    echo "⏪ Rolling back deployment..."
    helm rollback ecommerce-platform -n $NAMESPACE
    kubectl rollout status deployment/ecommerce-platform -n $NAMESPACE
fi

echo "✅ Incident response completed. Logs saved to /tmp/incident-logs/$INCIDENT_ID"
```

### Scenario 2: Performance Optimization
```python
# performance-analysis.py
import requests
import time
import statistics
import concurrent.futures
import matplotlib.pyplot as plt

class PerformanceAnalyzer:
    def __init__(self, base_url):
        self.base_url = base_url
        self.results = []
    
    def test_endpoint(self, endpoint, num_requests=100, concurrency=10):
        """Test endpoint performance"""
        print(f"Testing {endpoint} with {num_requests} requests, {concurrency} concurrent")
        
        def make_request():
            start_time = time.time()
            try:
                response = requests.get(f"{self.base_url}{endpoint}", timeout=10)
                end_time = time.time()
                return {
                    'status_code': response.status_code,
                    'response_time': end_time - start_time,
                    'success': response.status_code == 200
                }
            except Exception as e:
                return {
                    'status_code': 0,
                    'response_time': 10.0,
                    'success': False,
                    'error': str(e)
                }
        
        # Execute concurrent requests
        with concurrent.futures.ThreadPoolExecutor(max_workers=concurrency) as executor:
            futures = [executor.submit(make_request) for _ in range(num_requests)]
            results = [future.result() for future in concurrent.futures.as_completed(futures)]
        
        # Analyze results
        response_times = [r['response_time'] for r in results if r['success']]
        success_rate = sum(1 for r in results if r['success']) / len(results) * 100
        
        analysis = {
            'endpoint': endpoint,
            'total_requests': num_requests,
            'successful_requests': len(response_times),
            'success_rate': success_rate,
            'avg_response_time': statistics.mean(response_times) if response_times else 0,
            'median_response_time': statistics.median(response_times) if response_times else 0,
            'p95_response_time': statistics.quantiles(response_times, n=20)[18] if len(response_times) > 20 else 0,
            'min_response_time': min(response_times) if response_times else 0,
            'max_response_time': max(response_times) if response_times else 0
        }
        
        self.results.append(analysis)
        return analysis
    
    def generate_report(self):
        """Generate performance report"""
        print("\n" + "="*80)
        print("PERFORMANCE ANALYSIS REPORT")
        print("="*80)
        
        for result in self.results:
            print(f"\nEndpoint: {result['endpoint']}")
            print(f"Success Rate: {result['success_rate']:.1f}%")
            print(f"Average Response Time: {result['avg_response_time']*1000:.1f}ms")
            print(f"95th Percentile: {result['p95_response_time']*1000:.1f}ms")
            print(f"Min/Max: {result['min_response_time']*1000:.1f}ms / {result['max_response_time']*1000:.1f}ms")
            
            # Performance assessment
            if result['avg_response_time'] < 0.2:
                print("✅ Performance: Excellent")
            elif result['avg_response_time'] < 0.5:
                print("🟡 Performance: Good")
            elif result['avg_response_time'] < 1.0:
                print("🟠 Performance: Fair")
            else:
                print("🔴 Performance: Poor - Needs optimization")
    
    def plot_results(self):
        """Create performance visualization"""
        endpoints = [r['endpoint'] for r in self.results]
        avg_times = [r['avg_response_time']*1000 for r in self.results]
        p95_times = [r['p95_response_time']*1000 for r in self.results]
        
        fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(15, 6))
        
        # Response times
        ax1.bar(endpoints, avg_times, alpha=0.7, label='Average')
        ax1.bar(endpoints, p95_times, alpha=0.7, label='95th Percentile')
        ax1.set_ylabel('Response Time (ms)')
        ax1.set_title('Response Time by Endpoint')
        ax1.legend()
        ax1.tick_params(axis='x', rotation=45)
        
        # Success rates
        success_rates = [r['success_rate'] for r in self.results]
        ax2.bar(endpoints, success_rates, alpha=0.7, color='green')
        ax2.set_ylabel('Success Rate (%)')
        ax2.set_title('Success Rate by Endpoint')
        ax2.set_ylim(0, 100)
        ax2.tick_params(axis='x', rotation=45)
        
        plt.tight_layout()
        plt.savefig('performance_analysis.png', dpi=300, bbox_inches='tight')
        plt.show()

# Usage
if __name__ == "__main__":
    analyzer = PerformanceAnalyzer("https://ecommerce.company.com")
    
    # Test critical endpoints
    analyzer.test_endpoint("/api/health", 50, 5)
    analyzer.test_endpoint("/api/products", 100, 10)
    analyzer.test_endpoint("/api/users/profile", 100, 10)
    analyzer.test_endpoint("/api/orders", 100, 10)
    
    # Generate report and visualization
    analyzer.generate_report()
    analyzer.plot_results()
```

## Interview Questions

### Beginner Level
1. **Q: What are the key components of a complete CI/CD pipeline?**
   A: Source control integration, automated testing (unit, integration, security), build automation, artifact management, deployment automation, and monitoring/feedback loops.

2. **Q: How do you ensure zero-downtime deployments?**
   A: Use blue-green deployments, rolling updates, health checks, proper load balancer configuration, and automated rollback mechanisms.

3. **Q: What testing strategies should be included in a production deployment?**
   A: Unit tests, integration tests, security scans, performance tests, smoke tests, and end-to-end user journey tests.

### Intermediate Level
4. **Q: How do you implement monitoring and alerting for a production application?**
   A: Deploy monitoring stack (Prometheus/Grafana), define SLIs/SLOs, create custom dashboards, set up alerting rules, implement escalation procedures, and maintain runbooks.

5. **Q: Explain the infrastructure as code approach for production deployments.**
   A: Use tools like Terraform to define infrastructure declaratively, version control infrastructure code, implement automated provisioning, and maintain environment consistency.

6. **Q: How do you handle secrets and sensitive configuration in production?**
   A: Use dedicated secret management systems (AWS Secrets Manager, Vault), encrypt secrets at rest and in transit, implement least privilege access, and rotate secrets regularly.

### Advanced Level
7. **Q: How would you implement a complete disaster recovery strategy?**
   A: Multi-region deployment, automated backups, infrastructure replication, documented recovery procedures, regular DR testing, and RTO/RPO requirements definition.

8. **Q: Describe your approach to performance optimization in production.**
   A: Implement comprehensive monitoring, conduct regular performance testing, optimize database queries, implement caching strategies, and use auto-scaling based on metrics.

9. **Q: How do you ensure security throughout the deployment pipeline?**
   A: Implement security scanning in CI/CD, use container image scanning, implement network security policies, regular security audits, and compliance monitoring.

10. **Q: Explain how you would handle a critical production incident.**
    A: Follow incident response procedures, assess impact quickly, implement immediate mitigation, communicate with stakeholders, conduct thorough investigation, and implement preventive measures.

## Key Takeaways
- Complete CI/CD pipelines automate the entire software delivery process
- Infrastructure as code ensures consistent and repeatable deployments
- Comprehensive testing strategies prevent production issues
- Monitoring and alerting provide visibility into system health
- Documentation and runbooks enable effective incident response
- Security must be integrated throughout the deployment pipeline