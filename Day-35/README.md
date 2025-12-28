# Day 35: Kubernetes Fundamentals - Container Orchestration

## Learning Objectives
- Understand Kubernetes architecture and core components
- Master Pods, Services, and Deployments concepts
- Learn basic kubectl commands and cluster management
- Implement ConfigMaps and Secrets management
- Understand Kubernetes networking fundamentals

## Topics Covered

### 1. Kubernetes Architecture
```
Kubernetes Cluster Architecture:

Master Node (Control Plane):
├── API Server (kube-apiserver)
├── etcd (Key-Value Store)
├── Controller Manager (kube-controller-manager)
├── Scheduler (kube-scheduler)
└── Cloud Controller Manager

Worker Nodes:
├── kubelet (Node Agent)
├── kube-proxy (Network Proxy)
├── Container Runtime (Docker/containerd)
└── Pods (Application Containers)
```

### 2. Core Kubernetes Objects
```yaml
# Pod - Smallest deployable unit
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"

---
# Service - Network abstraction
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
  type: ClusterIP

---
# Deployment - Manages ReplicaSets
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
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
```

### 3. ConfigMaps and Secrets
```yaml
# ConfigMap for application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  database_url: "postgresql://db:5432/myapp"
  redis_url: "redis://redis:6379"
  log_level: "info"
  app.properties: |
    server.port=8080
    spring.datasource.url=jdbc:postgresql://db:5432/myapp
    spring.redis.host=redis
    spring.redis.port=6379

---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
data:
  database_password: cGFzc3dvcmQxMjM=  # base64 encoded
  api_key: YWJjZGVmZ2hpams=            # base64 encoded
stringData:
  username: admin  # automatically base64 encoded

---
# Using ConfigMap and Secret in Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
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
        image: myapp:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: database_url
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database_password
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: log_level
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        - name: secret-volume
          mountPath: /app/secrets
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: secret-volume
        secret:
          secretName: app-secrets
```

## Hands-on Labs

### Lab 1: Setting Up Local Kubernetes Cluster
```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Docker (if not already installed)
sudo apt-get update
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# Install Minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Start Minikube cluster
minikube start --driver=docker --cpus=2 --memory=4096

# Verify cluster
kubectl cluster-info
kubectl get nodes
kubectl get pods -A

# Enable useful addons
minikube addons enable dashboard
minikube addons enable metrics-server
minikube addons enable ingress
```

### Lab 2: Basic kubectl Commands
```bash
# Cluster information
kubectl cluster-info
kubectl get nodes
kubectl describe node minikube

# Namespace operations
kubectl get namespaces
kubectl create namespace development
kubectl config set-context --current --namespace=development

# Pod operations
kubectl run nginx --image=nginx:1.21
kubectl get pods
kubectl get pods -o wide
kubectl describe pod nginx
kubectl logs nginx
kubectl exec -it nginx -- /bin/bash

# Service operations
kubectl expose pod nginx --port=80 --type=NodePort
kubectl get services
kubectl describe service nginx

# Deployment operations
kubectl create deployment web-app --image=nginx:1.21 --replicas=3
kubectl get deployments
kubectl describe deployment web-app
kubectl scale deployment web-app --replicas=5
kubectl rollout status deployment web-app

# ConfigMap and Secret operations
kubectl create configmap app-config --from-literal=database_url=postgresql://db:5432/myapp
kubectl create secret generic app-secrets --from-literal=password=secret123
kubectl get configmaps
kubectl get secrets
kubectl describe configmap app-config
kubectl describe secret app-secrets

# Resource management
kubectl get all
kubectl delete pod nginx
kubectl delete deployment web-app
kubectl delete service nginx

# Apply YAML files
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f configmap.yaml

# Port forwarding for testing
kubectl port-forward pod/nginx 8080:80
kubectl port-forward service/nginx 8080:80
```

### Lab 3: Multi-Container Pod Application
```yaml
# multi-container-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-app
  labels:
    app: multi-container
spec:
  containers:
  # Main application container
  - name: web-app
    image: nginx:1.21
    ports:
    - containerPort: 80
    volumeMounts:
    - name: shared-data
      mountPath: /usr/share/nginx/html
    - name: config-volume
      mountPath: /etc/nginx/conf.d
  
  # Sidecar container for log processing
  - name: log-processor
    image: busybox:1.35
    command: ['sh', '-c']
    args:
    - |
      while true; do
        echo "$(date): Processing logs..." >> /shared/logs/app.log
        sleep 30
      done
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  
  # Init container for setup
  initContainers:
  - name: init-setup
    image: busybox:1.35
    command: ['sh', '-c']
    args:
    - |
      echo "<h1>Welcome to Multi-Container App</h1>" > /shared/index.html
      mkdir -p /shared/logs
      echo "$(date): Application initialized" > /shared/logs/init.log
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  
  volumes:
  - name: shared-data
    emptyDir: {}
  - name: config-volume
    configMap:
      name: nginx-config

---
# nginx-config ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
        
        location /logs {
            alias /usr/share/nginx/html/logs/;
            autoindex on;
        }
    }
```

### Lab 4: Complete Web Application Deployment
```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web-app
  labels:
    name: web-app

---
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-app-config
  namespace: web-app
data:
  DATABASE_HOST: "postgres-service"
  DATABASE_PORT: "5432"
  DATABASE_NAME: "webapp"
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
  LOG_LEVEL: "info"

---
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: web-app-secrets
  namespace: web-app
type: Opaque
stringData:
  DATABASE_USER: "webapp_user"
  DATABASE_PASSWORD: "secure_password_123"
  JWT_SECRET: "jwt_secret_key_here"

---
# postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: web-app
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
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: DATABASE_NAME
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: web-app-secrets
              key: DATABASE_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: web-app-secrets
              key: DATABASE_PASSWORD
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
      volumes:
      - name: postgres-storage
        emptyDir: {}

---
# postgres-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: web-app
spec:
  selector:
    app: postgres
  ports:
  - protocol: TCP
    port: 5432
    targetPort: 5432
  type: ClusterIP

---
# redis-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: web-app
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
        image: redis:6-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"

---
# redis-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: web-app
spec:
  selector:
    app: redis
  ports:
  - protocol: TCP
    port: 6379
    targetPort: 6379
  type: ClusterIP

---
# web-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: web-app
spec:
  replicas: 3
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
        image: node:16-alpine
        ports:
        - containerPort: 3000
        env:
        - name: DATABASE_HOST
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: DATABASE_HOST
        - name: DATABASE_PORT
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: DATABASE_PORT
        - name: DATABASE_NAME
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: DATABASE_NAME
        - name: DATABASE_USER
          valueFrom:
            secretKeyRef:
              name: web-app-secrets
              key: DATABASE_USER
        - name: DATABASE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: web-app-secrets
              key: DATABASE_PASSWORD
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: REDIS_HOST
        - name: REDIS_PORT
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: REDIS_PORT
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: web-app-secrets
              key: JWT_SECRET
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"

---
# web-app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: web-app
spec:
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 3000
  type: LoadBalancer
```

## Real-world Scenarios

### Scenario 1: Application Scaling and Updates
```bash
# Deploy application
kubectl apply -f web-app-deployment.yaml

# Scale application
kubectl scale deployment web-app --replicas=5 -n web-app

# Update application image
kubectl set image deployment/web-app web-app=node:18-alpine -n web-app

# Monitor rollout
kubectl rollout status deployment/web-app -n web-app

# Rollback if needed
kubectl rollout undo deployment/web-app -n web-app

# Check rollout history
kubectl rollout history deployment/web-app -n web-app
```

### Scenario 2: Troubleshooting Applications
```bash
# Check pod status
kubectl get pods -n web-app
kubectl describe pod <pod-name> -n web-app

# Check logs
kubectl logs <pod-name> -n web-app
kubectl logs <pod-name> -c <container-name> -n web-app --previous

# Debug networking
kubectl exec -it <pod-name> -n web-app -- nslookup postgres-service
kubectl exec -it <pod-name> -n web-app -- wget -qO- http://redis-service:6379

# Resource usage
kubectl top pods -n web-app
kubectl top nodes

# Events
kubectl get events -n web-app --sort-by=.metadata.creationTimestamp
```

## Interview Questions

### Beginner Level
1. **Q: What is Kubernetes and why is it used?**
   A: Kubernetes is a container orchestration platform that automates deployment, scaling, and management of containerized applications across clusters of hosts.

2. **Q: What is the difference between a Pod and a Container?**
   A: A container is a single running instance of an application, while a Pod is the smallest deployable unit in Kubernetes that can contain one or more containers sharing network and storage.

3. **Q: What are the main components of Kubernetes architecture?**
   A: Control plane (API server, etcd, scheduler, controller manager) and worker nodes (kubelet, kube-proxy, container runtime).

### Intermediate Level
4. **Q: Explain the difference between ConfigMaps and Secrets.**
   A: ConfigMaps store non-sensitive configuration data in key-value pairs, while Secrets store sensitive data like passwords and tokens in base64 encoded format with additional security features.

5. **Q: What is a Kubernetes Service and what types are available?**
   A: A Service provides stable network access to Pods. Types include ClusterIP (internal), NodePort (external via node), LoadBalancer (cloud provider), and ExternalName (DNS).

6. **Q: How do you handle application configuration in Kubernetes?**
   A: Use ConfigMaps for non-sensitive data and Secrets for sensitive data, mount them as volumes or environment variables, and use init containers for setup tasks.

### Advanced Level
7. **Q: Explain Kubernetes networking model.**
   A: Every Pod gets its own IP, Pods can communicate without NAT, Services provide stable endpoints, and network policies control traffic flow between Pods.

8. **Q: How do you implement health checks in Kubernetes?**
   A: Use liveness probes (restart unhealthy containers), readiness probes (control traffic routing), and startup probes (handle slow-starting containers).

9. **Q: What are the best practices for resource management in Kubernetes?**
   A: Set resource requests and limits, use namespaces for isolation, implement resource quotas, use horizontal pod autoscaling, and monitor resource usage.

10. **Q: How would you troubleshoot a failing Pod in Kubernetes?**
    A: Check Pod status, examine events, review logs, verify resource constraints, test networking, validate configurations, and use debugging tools like kubectl exec.

## Key Takeaways
- Kubernetes provides powerful container orchestration capabilities
- Pods are the fundamental deployment unit containing one or more containers
- Services enable reliable networking and load balancing
- ConfigMaps and Secrets manage application configuration securely
- kubectl is the primary tool for cluster interaction and management
- Proper resource management and health checks ensure application reliability