# Day 41: Advanced Monitoring - Grafana & Prometheus

## Learning Objectives
- Install and configure Grafana and Prometheus
- Create comprehensive dashboards and visualizations
- Integrate multiple data sources (CloudWatch, Prometheus)
- Set up advanced alerting and notification systems
- Implement performance monitoring best practices
- Collect custom metrics and create application dashboards

## Topics Covered

### 1. Prometheus Architecture and Setup
```yaml
# prometheus-config.yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"
  - "recording_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'application'
    static_configs:
      - targets: ['app:8080']
    metrics_path: /metrics
    scrape_interval: 10s

  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)

  - job_name: 'kubernetes-services'
    kubernetes_sd_configs:
      - role: service
    relabel_configs:
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### 2. Grafana Dashboard Configuration
```json
{
  "dashboard": {
    "id": null,
    "title": "Application Performance Dashboard",
    "tags": ["application", "performance"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Request Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{method}} {{status}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {
              "mode": "palette-classic"
            },
            "custom": {
              "displayMode": "list",
              "orientation": "horizontal"
            },
            "mappings": [],
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "red", "value": 1000}
              ]
            }
          }
        },
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Response Time",
        "type": "timeseries",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {"mode": "palette-classic"},
            "custom": {
              "drawStyle": "line",
              "lineInterpolation": "linear",
              "barAlignment": 0,
              "lineWidth": 1,
              "fillOpacity": 0,
              "gradientMode": "none",
              "spanNulls": false,
              "insertNulls": false,
              "showPoints": "auto",
              "pointSize": 5
            },
            "unit": "s"
          }
        },
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "5s"
  }
}
```

### 3. Alert Rules Configuration
```yaml
# alert_rules.yml
groups:
- name: application_alerts
  rules:
  - alert: HighErrorRate
    expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value }} errors per second"

  - alert: HighResponseTime
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High response time detected"
      description: "95th percentile response time is {{ $value }}s"

  - alert: DatabaseConnectionsHigh
    expr: pg_stat_activity_count > 80
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "High database connections"
      description: "Database has {{ $value }} active connections"

- name: infrastructure_alerts
  rules:
  - alert: HighCPUUsage
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High CPU usage on {{ $labels.instance }}"
      description: "CPU usage is {{ $value }}%"

  - alert: HighMemoryUsage
    expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "High memory usage on {{ $labels.instance }}"
      description: "Memory usage is {{ $value }}%"

  - alert: DiskSpaceLow
    expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) * 100 > 90
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Low disk space on {{ $labels.instance }}"
      description: "Disk usage is {{ $value }}%"
```

## Hands-on Labs

### Lab 1: Complete Monitoring Stack Setup
```bash
#!/bin/bash
# setup-monitoring-stack.sh

# Create monitoring namespace
kubectl create namespace monitoring

# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Prometheus Stack
cat << EOF > prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    additionalScrapeConfigs:
    - job_name: 'custom-app'
      static_configs:
      - targets: ['app-service:8080']

grafana:
  adminPassword: admin123
  persistence:
    enabled: true
    size: 10Gi
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        editable: true
        options:
          path: /var/lib/grafana/dashboards/default
  
  dashboards:
    default:
      kubernetes-cluster:
        gnetId: 7249
        revision: 1
        datasource: Prometheus
      node-exporter:
        gnetId: 1860
        revision: 27
        datasource: Prometheus

alertmanager:
  config:
    global:
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'alerts@mycompany.com'
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'web.hook'
    receivers:
    - name: 'web.hook'
      email_configs:
      - to: 'admin@mycompany.com'
        subject: 'Alert: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          {{ end }}
EOF

helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml

# Wait for deployment
kubectl wait --for=condition=available --timeout=300s deployment/prometheus-grafana -n monitoring

echo "✅ Monitoring stack installed successfully!"
echo "Access Grafana: kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80"
```

### Lab 2: Custom Application Metrics
```python
# app_metrics.py
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time
import random
import threading

# Define metrics
REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP requests', ['method', 'endpoint', 'status'])
REQUEST_LATENCY = Histogram('http_request_duration_seconds', 'HTTP request latency')
ACTIVE_USERS = Gauge('active_users_total', 'Number of active users')
DATABASE_CONNECTIONS = Gauge('database_connections_active', 'Active database connections')
QUEUE_SIZE = Gauge('task_queue_size', 'Number of tasks in queue')

class ApplicationMetrics:
    def __init__(self):
        self.active_users = 0
        self.db_connections = 0
        self.queue_size = 0
        
    def simulate_requests(self):
        """Simulate HTTP requests with metrics"""
        endpoints = ['/api/users', '/api/orders', '/api/products', '/health']
        methods = ['GET', 'POST', 'PUT', 'DELETE']
        
        while True:
            endpoint = random.choice(endpoints)
            method = random.choice(methods)
            
            # Simulate request processing time
            with REQUEST_LATENCY.time():
                processing_time = random.uniform(0.1, 2.0)
                time.sleep(processing_time)
                
                # Determine status code
                if endpoint == '/health':
                    status = '200'
                elif processing_time > 1.5:
                    status = '500'
                elif processing_time > 1.0:
                    status = '404'
                else:
                    status = '200'
                
                REQUEST_COUNT.labels(method=method, endpoint=endpoint, status=status).inc()
            
            time.sleep(random.uniform(0.1, 1.0))
    
    def update_system_metrics(self):
        """Update system-level metrics"""
        while True:
            # Simulate changing metrics
            self.active_users = random.randint(50, 500)
            self.db_connections = random.randint(10, 100)
            self.queue_size = random.randint(0, 50)
            
            ACTIVE_USERS.set(self.active_users)
            DATABASE_CONNECTIONS.set(self.db_connections)
            QUEUE_SIZE.set(self.queue_size)
            
            time.sleep(30)  # Update every 30 seconds

if __name__ == '__main__':
    # Start Prometheus metrics server
    start_http_server(8080)
    
    app_metrics = ApplicationMetrics()
    
    # Start background threads
    request_thread = threading.Thread(target=app_metrics.simulate_requests)
    metrics_thread = threading.Thread(target=app_metrics.update_system_metrics)
    
    request_thread.daemon = True
    metrics_thread.daemon = True
    
    request_thread.start()
    metrics_thread.start()
    
    print("Application metrics server started on port 8080")
    print("Metrics available at http://localhost:8080/metrics")
    
    # Keep the main thread alive
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("Shutting down...")
```

### Lab 3: Advanced Grafana Dashboard
```json
{
  "dashboard": {
    "title": "Comprehensive Application Dashboard",
    "panels": [
      {
        "title": "Request Rate by Endpoint",
        "type": "timeseries",
        "targets": [
          {
            "expr": "rate(http_requests_total[5m])",
            "legendFormat": "{{endpoint}} - {{status}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps",
            "custom": {
              "drawStyle": "line",
              "lineWidth": 2,
              "fillOpacity": 10
            }
          }
        }
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "rate(http_requests_total{status=~\"5..\"}[5m]) / rate(http_requests_total[5m]) * 100",
            "legendFormat": "Error Rate %"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 5}
              ]
            }
          }
        }
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
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "99th percentile"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "custom": {
              "drawStyle": "line",
              "lineWidth": 2
            }
          }
        }
      },
      {
        "title": "System Resources",
        "type": "timeseries",
        "targets": [
          {
            "expr": "active_users_total",
            "legendFormat": "Active Users"
          },
          {
            "expr": "database_connections_active",
            "legendFormat": "DB Connections"
          },
          {
            "expr": "task_queue_size",
            "legendFormat": "Queue Size"
          }
        ]
      }
    ],
    "templating": {
      "list": [
        {
          "name": "instance",
          "type": "query",
          "query": "label_values(up, instance)",
          "refresh": 1
        },
        {
          "name": "endpoint",
          "type": "query",
          "query": "label_values(http_requests_total, endpoint)",
          "refresh": 1
        }
      ]
    },
    "annotations": {
      "list": [
        {
          "name": "Deployments",
          "datasource": "Prometheus",
          "expr": "changes(prometheus_config_last_reload_success_timestamp_seconds[1h]) > 0",
          "titleFormat": "Config Reload",
          "textFormat": "Prometheus configuration reloaded"
        }
      ]
    }
  }
}
```

### Lab 4: Multi-Data Source Integration
```yaml
# grafana-datasources.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasources
  namespace: monitoring
data:
  datasources.yaml: |
    apiVersion: 1
    datasources:
    - name: Prometheus
      type: prometheus
      access: proxy
      url: http://prometheus-server:80
      isDefault: true
      
    - name: CloudWatch
      type: cloudwatch
      access: proxy
      jsonData:
        authType: keys
        defaultRegion: us-west-2
      secureJsonData:
        accessKey: ${AWS_ACCESS_KEY_ID}
        secretKey: ${AWS_SECRET_ACCESS_KEY}
        
    - name: Loki
      type: loki
      access: proxy
      url: http://loki:3100
      
    - name: Jaeger
      type: jaeger
      access: proxy
      url: http://jaeger-query:16686

---
# Multi-source dashboard query examples
# Prometheus + CloudWatch correlation
{
  "targets": [
    {
      "datasource": "Prometheus",
      "expr": "rate(http_requests_total[5m])",
      "legendFormat": "App Requests (Prometheus)"
    },
    {
      "datasource": "CloudWatch",
      "namespace": "AWS/ApplicationELB",
      "metricName": "RequestCount",
      "dimensions": {
        "LoadBalancer": "app/my-alb/1234567890"
      },
      "statistics": ["Sum"],
      "period": 300,
      "legendFormat": "ALB Requests (CloudWatch)"
    }
  ]
}
```

## Real-world Scenarios

### Scenario 1: SRE Dashboard Implementation
```json
{
  "dashboard": {
    "title": "SRE Golden Signals Dashboard",
    "panels": [
      {
        "title": "Latency - SLI",
        "type": "stat",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "99th percentile latency"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "thresholds": {
              "steps": [
                {"color": "green", "value": null},
                {"color": "yellow", "value": 0.5},
                {"color": "red", "value": 1.0}
              ]
            }
          }
        }
      },
      {
        "title": "Traffic - Requests per Second",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m]))",
            "legendFormat": "Total RPS"
          }
        ]
      },
      {
        "title": "Errors - Error Budget",
        "type": "stat",
        "targets": [
          {
            "expr": "(1 - (rate(http_requests_total{status=~\"5..\"}[30d]) / rate(http_requests_total[30d]))) * 100",
            "legendFormat": "Error Budget Remaining"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percent",
            "thresholds": {
              "steps": [
                {"color": "red", "value": null},
                {"color": "yellow", "value": 10},
                {"color": "green", "value": 50}
              ]
            }
          }
        }
      },
      {
        "title": "Saturation - Resource Utilization",
        "type": "timeseries",
        "targets": [
          {
            "expr": "avg(rate(container_cpu_usage_seconds_total[5m])) * 100",
            "legendFormat": "CPU Utilization %"
          },
          {
            "expr": "avg(container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100",
            "legendFormat": "Memory Utilization %"
          }
        ]
      }
    ]
  }
}
```

### Scenario 2: Business Metrics Dashboard
```python
# business_metrics.py
from prometheus_client import Counter, Gauge, Histogram
import time
import random

# Business metrics
ORDERS_TOTAL = Counter('orders_total', 'Total orders', ['status', 'payment_method'])
REVENUE_TOTAL = Counter('revenue_total', 'Total revenue in cents')
ACTIVE_SUBSCRIPTIONS = Gauge('subscriptions_active', 'Active subscriptions', ['plan'])
CUSTOMER_SATISFACTION = Histogram('customer_satisfaction_score', 'Customer satisfaction scores')

def simulate_business_metrics():
    """Simulate business metrics"""
    while True:
        # Simulate orders
        status = random.choice(['completed', 'pending', 'cancelled'])
        payment_method = random.choice(['credit_card', 'paypal', 'bank_transfer'])
        
        ORDERS_TOTAL.labels(status=status, payment_method=payment_method).inc()
        
        if status == 'completed':
            revenue = random.randint(1000, 50000)  # Revenue in cents
            REVENUE_TOTAL.inc(revenue)
        
        # Update subscription metrics
        for plan in ['basic', 'premium', 'enterprise']:
            count = random.randint(100, 1000)
            ACTIVE_SUBSCRIPTIONS.labels(plan=plan).set(count)
        
        # Customer satisfaction
        satisfaction = random.uniform(1.0, 5.0)
        CUSTOMER_SATISFACTION.observe(satisfaction)
        
        time.sleep(random.uniform(1, 5))

if __name__ == '__main__':
    simulate_business_metrics()
```

## Interview Questions

### Beginner Level
1. **Q: What is the difference between Prometheus and Grafana?**
   A: Prometheus is a time-series database and monitoring system that collects and stores metrics, while Grafana is a visualization tool that creates dashboards and graphs from various data sources including Prometheus.

2. **Q: What are the four golden signals of monitoring?**
   A: Latency (response time), Traffic (request rate), Errors (error rate), and Saturation (resource utilization).

3. **Q: How do you expose metrics from an application for Prometheus?**
   A: Implement a /metrics endpoint that returns metrics in Prometheus format, or use client libraries to instrument your application code.

### Intermediate Level
4. **Q: Explain the difference between recording rules and alerting rules in Prometheus.**
   A: Recording rules precompute frequently used queries and store results as new time series, while alerting rules define conditions that trigger alerts when thresholds are exceeded.

5. **Q: How do you implement high availability for Prometheus?**
   A: Run multiple Prometheus instances with identical configurations, use external storage like Thanos or Cortex, and implement proper service discovery and load balancing.

6. **Q: What are Grafana templating variables and how are they useful?**
   A: Template variables allow dynamic dashboard content based on user selection or query results, enabling reusable dashboards across different environments or services.

### Advanced Level
7. **Q: How would you implement a complete observability strategy?**
   A: Combine metrics (Prometheus), logs (ELK/Loki), and traces (Jaeger), implement SLI/SLO monitoring, create runbooks, and establish incident response procedures.

8. **Q: Explain how to optimize Prometheus for large-scale environments.**
   A: Use federation, implement proper retention policies, optimize queries, use recording rules, implement sharding, and consider long-term storage solutions.

9. **Q: How do you implement custom business metrics monitoring?**
   A: Instrument application code with business-specific metrics, create custom dashboards, implement SLIs based on business outcomes, and set up alerts for business-critical thresholds.

10. **Q: Describe a complete alerting strategy with escalation procedures.**
    A: Implement tiered alerting (info, warning, critical), use alert routing based on severity and team, implement escalation policies, integrate with incident management systems, and maintain alert fatigue prevention.

## Key Takeaways
- Prometheus provides powerful time-series data collection and querying
- Grafana enables rich visualization and dashboard creation
- Proper alerting prevents issues from becoming incidents
- Multi-data source integration provides comprehensive observability
- Business metrics monitoring aligns technical metrics with business outcomes
- SRE practices like SLI/SLO monitoring improve service reliability