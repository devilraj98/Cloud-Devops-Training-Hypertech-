# Day 36: Minikube Local Kubernetes Setup

## Learning Objectives
- Install and configure Minikube for local development
- Deploy React application using Docker Hub images
- Implement auto-scaling and resource management locally
- Master local Kubernetes development workflow
- Debug and troubleshoot pods effectively

## Topics Covered

### 1. Minikube Installation and Setup
```bash
# Install Minikube on Ubuntu/Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Install on macOS
brew install minikube

# Install on Windows
choco install minikube

# Start Minikube cluster
minikube start --driver=docker --cpus=4 --memory=8192
minikube status

# Enable addons
minikube addons enable dashboard
minikube addons enable metrics-server
minikube addons enable ingress
minikube addons list
```

### 2. React Application Deployment
```yaml
# react-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: react-app
  labels:
    app: react-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: react-app
  template:
    metadata:
      labels:
        app: react-app
    spec:
      containers:
      - name: react-app
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        volumeMounts:
        - name: react-build
          mountPath: /usr/share/nginx/html
      initContainers:
      - name: build-react
        image: node:16-alpine
        command: ['sh', '-c']
        args:
        - |
          echo "Building React app..."
          mkdir -p /app
          cd /app
          cat > package.json << 'EOF'
          {
            "name": "react-k8s-app",
            "version": "1.0.0",
            "dependencies": {
              "react": "^18.0.0",
              "react-dom": "^18.0.0",
              "react-scripts": "5.0.1"
            },
            "scripts": {
              "build": "react-scripts build"
            }
          }
          EOF
          cat > public/index.html << 'EOF'
          <!DOCTYPE html>
          <html><head><title>React K8s App</title></head>
          <body><div id="root"></div></body></html>
          EOF
          mkdir -p src
          cat > src/index.js << 'EOF'
          import React from 'react';
          import ReactDOM from 'react-dom/client';
          const App = () => <h1>Hello from Kubernetes!</h1>;
          ReactDOM.createRoot(document.getElementById('root')).render(<App />);
          EOF
          npm install
          npm run build
          cp -r build/* /shared/
        volumeMounts:
        - name: react-build
          mountPath: /shared
      volumes:
      - name: react-build
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: react-app-service
spec:
  selector:
    app: react-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: LoadBalancer
```

## Hands-on Labs

### Lab 1: Complete Minikube Setup
```bash
# Start Minikube with specific configuration
minikube start \
  --driver=docker \
  --cpus=4 \
  --memory=8192 \
  --disk-size=20g \
  --kubernetes-version=v1.28.0

# Verify installation
kubectl cluster-info
kubectl get nodes
kubectl get pods -A

# Access Kubernetes dashboard
minikube dashboard

# Enable useful addons
minikube addons enable dashboard
minikube addons enable metrics-server
minikube addons enable ingress
minikube addons enable registry
```

### Lab 2: Deploy React Application from Docker Hub
```bash
# Create namespace
kubectl create namespace react-app

# Deploy using Docker Hub image
kubectl create deployment react-app \
  --image=nginx:alpine \
  --replicas=3 \
  -n react-app

# Expose the deployment
kubectl expose deployment react-app \
  --type=LoadBalancer \
  --port=80 \
  -n react-app

# Get service URL
minikube service react-app -n react-app --url

# Scale the deployment
kubectl scale deployment react-app --replicas=5 -n react-app

# Check status
kubectl get pods -n react-app
kubectl get services -n react-app
```

### Lab 3: Auto-scaling Configuration
```yaml
# hpa.yaml - Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: react-app-hpa
  namespace: react-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: react-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max

---
# Load testing pod
apiVersion: v1
kind: Pod
metadata:
  name: load-test
  namespace: react-app
spec:
  containers:
  - name: load-test
    image: busybox:1.35
    command: ['sh', '-c']
    args:
    - |
      while true; do
        wget -q -O- http://react-app-service/
        sleep 0.1
      done
```

### Lab 4: Complete Development Workflow
```bash
#!/bin/bash
# dev-workflow.sh

echo "🚀 Starting Minikube Development Workflow"

# Start Minikube if not running
if ! minikube status | grep -q "Running"; then
    echo "Starting Minikube..."
    minikube start --driver=docker --cpus=4 --memory=8192
fi

# Set kubectl context
kubectl config use-context minikube

# Create development namespace
kubectl create namespace development --dry-run=client -o yaml | kubectl apply -f -

# Deploy application
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: development
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: development
spec:
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: NodePort
EOF

# Wait for deployment
kubectl rollout status deployment/web-app -n development

# Get service URL
echo "🌐 Application URL:"
minikube service web-app-service -n development --url

# Enable port forwarding
echo "🔗 Port forwarding on localhost:8080"
kubectl port-forward service/web-app-service 8080:80 -n development &

echo "✅ Development environment ready!"
echo "Access your app at: http://localhost:8080"
```

## Real-world Scenarios

### Scenario 1: Local Development Environment
```yaml
# docker-compose.yml equivalent in Kubernetes
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DATABASE_URL: "postgresql://postgres:password@postgres:5432/myapp"
  REDIS_URL: "redis://redis:6379"

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          value: "password"
        - name: POSTGRES_DB
          value: "myapp"
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-data
        emptyDir: {}

---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector:
    app: postgres
  ports:
  - port: 5432

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379

---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
```

### Scenario 2: Debugging and Troubleshooting
```bash
# Debugging commands
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name> -f
kubectl exec -it <pod-name> -- /bin/sh

# Check resource usage
kubectl top nodes
kubectl top pods

# Debug networking
kubectl exec -it <pod-name> -- nslookup kubernetes.default
kubectl exec -it <pod-name> -- wget -qO- http://service-name

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Debug services
kubectl get endpoints
kubectl describe service <service-name>

# Port forwarding for debugging
kubectl port-forward pod/<pod-name> 8080:80
kubectl port-forward service/<service-name> 8080:80
```

## Interview Questions

### Beginner Level
1. **Q: What is Minikube and when would you use it?**
   A: Minikube is a tool that runs a single-node Kubernetes cluster locally for development, testing, and learning purposes.

2. **Q: How do you expose a service in Minikube?**
   A: Use `kubectl expose` command or create a Service manifest with type NodePort or LoadBalancer, then use `minikube service` to get the URL.

3. **Q: What are the advantages of using Minikube for development?**
   A: Local development, cost-effective, quick setup, isolated environment, and ability to test Kubernetes features locally.

### Intermediate Level
4. **Q: How do you configure resource limits in Minikube?**
   A: Set CPU and memory limits in deployment specs using resources.requests and resources.limits, and configure Minikube with appropriate CPU and memory allocation.

5. **Q: Explain how to implement auto-scaling in Minikube.**
   A: Enable metrics-server addon, create HorizontalPodAutoscaler with CPU/memory targets, and configure deployment with resource requests.

6. **Q: How do you debug a failing pod in Minikube?**
   A: Use kubectl describe, check logs with kubectl logs, exec into containers, check events, and verify resource constraints and networking.

### Advanced Level
7. **Q: How would you set up a complete development environment with Minikube?**
   A: Configure multi-service application, implement service discovery, set up persistent volumes, configure ingress, and integrate with CI/CD.

8. **Q: Explain the differences between Minikube and production Kubernetes.**
   A: Single-node vs multi-node, local storage vs distributed storage, simplified networking, and different resource constraints and availability requirements.

9. **Q: How do you optimize Minikube performance for development?**
   A: Allocate sufficient resources, use appropriate drivers, enable relevant addons, implement resource limits, and use local image registry.

10. **Q: Describe how to implement a complete CI/CD workflow with Minikube.**
    A: Set up local registry, implement automated testing, configure deployment pipelines, use GitOps practices, and integrate with development tools.

## Key Takeaways
- Minikube provides excellent local Kubernetes development environment
- Proper resource configuration is crucial for performance
- Auto-scaling works locally with metrics-server addon
- Debugging skills are essential for local development
- Local development workflow mirrors production patterns
- Minikube addons extend functionality significantly