# Day 39: Advanced Kubernetes & GitOps Implementation

## Learning Objectives
- Implement advanced Kubernetes monitoring and logging
- Configure persistent volumes and storage classes
- Set up ingress controllers and traffic management
- Master Helm charts and package management
- Deploy GitOps workflows with ArgoCD and Flux

## Topics Covered

### 1. Advanced Monitoring and Logging Stack
```yaml
# monitoring-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    name: monitoring

---
# prometheus-values.yaml
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
    resources:
      requests:
        memory: 2Gi
        cpu: 1000m
      limits:
        memory: 4Gi
        cpu: 2000m

grafana:
  adminPassword: admin123
  persistence:
    enabled: true
    storageClassName: gp3
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

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi

nodeExporter:
  enabled: true

kubeStateMetrics:
  enabled: true
```

### 2. Persistent Storage Configuration
```yaml
# storage-class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer

---
# persistent-volume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: database-pv
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: fast-ssd
  csi:
    driver: ebs.csi.aws.com
    volumeHandle: vol-0123456789abcdef0

---
# persistent-volume-claim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: database-pvc
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 100Gi

---
# statefulset-with-storage.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
  namespace: production
spec:
  serviceName: database
  replicas: 3
  selector:
    matchLabels:
      app: database
  template:
    metadata:
      labels:
        app: database
    spec:
      containers:
      - name: postgres
        image: postgres:13
        env:
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 50Gi
```

### 3. Advanced Ingress Configuration
```yaml
# nginx-ingress-controller.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ingress-nginx

---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: nginx-ingress
  namespace: ingress-nginx
spec:
  chart: ingress-nginx
  repo: https://kubernetes.github.io/ingress-nginx
  targetNamespace: ingress-nginx
  valuesContent: |-
    controller:
      replicaCount: 3
      resources:
        requests:
          cpu: 100m
          memory: 90Mi
        limits:
          cpu: 500m
          memory: 500Mi
      service:
        type: LoadBalancer
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-type: nlb
          service.beta.kubernetes.io/aws-load-balancer-cross-zone-load-balancing-enabled: "true"
      metrics:
        enabled: true
        serviceMonitor:
          enabled: true

---
# advanced-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.myapp.com
    - app.myapp.com
    secretName: myapp-tls
  rules:
  - host: api.myapp.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /api/v2
        pathType: Prefix
        backend:
          service:
            name: api-v2-service
            port:
              number: 80
  - host: app.myapp.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

## Hands-on Labs

### Lab 1: Complete Monitoring Stack Setup
```bash
#!/bin/bash
# setup-monitoring.sh

# Add Helm repositories
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Create monitoring namespace
kubectl create namespace monitoring

# Install Prometheus Operator
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values prometheus-values.yaml \
  --wait

# Install Loki for log aggregation
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set grafana.enabled=false \
  --set prometheus.enabled=false \
  --set promtail.enabled=true

# Verify installation
kubectl get pods -n monitoring
kubectl get services -n monitoring

# Port forward to access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

echo "✅ Monitoring stack installed successfully!"
echo "Access Grafana at: http://localhost:3000"
echo "Username: admin, Password: admin123"
```

### Lab 2: GitOps with ArgoCD
```yaml
# argocd-install.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: argocd

---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo-cd
    targetRevision: 5.46.7
    helm:
      values: |
        server:
          service:
            type: LoadBalancer
          config:
            repositories: |
              - type: git
                url: https://github.com/mycompany/k8s-manifests
              - type: helm
                name: stable
                url: https://charts.helm.sh/stable
        dex:
          enabled: false
        notifications:
          enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# application-deployment.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-application
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mycompany/k8s-manifests
    path: applications/web-app
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
  revisionHistoryLimit: 10
```

### Lab 3: Advanced Helm Chart with Dependencies
```yaml
# Chart.yaml
apiVersion: v2
name: microservices-app
description: Complete microservices application
type: application
version: 1.0.0
appVersion: "2.0.0"

dependencies:
- name: postgresql
  version: 12.1.2
  repository: https://charts.bitnami.com/bitnami
  condition: postgresql.enabled
- name: redis
  version: 17.3.7
  repository: https://charts.bitnami.com/bitnami
  condition: redis.enabled
- name: elasticsearch
  version: 8.5.1
  repository: https://helm.elastic.co
  condition: elasticsearch.enabled

---
# values.yaml
global:
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: "gp3"

frontend:
  enabled: true
  replicaCount: 3
  image:
    repository: mycompany/frontend
    tag: "2.0.0"
  service:
    type: ClusterIP
    port: 80
  ingress:
    enabled: true
    className: nginx
    annotations:
      nginx.ingress.kubernetes.io/rate-limit: "100"
    hosts:
      - host: app.mycompany.com
        paths:
          - path: /
            pathType: Prefix

backend:
  enabled: true
  replicaCount: 5
  image:
    repository: mycompany/backend
    tag: "2.0.0"
  service:
    type: ClusterIP
    port: 8080
  autoscaling:
    enabled: true
    minReplicas: 5
    maxReplicas: 20
    targetCPUUtilizationPercentage: 70

worker:
  enabled: true
  replicaCount: 2
  image:
    repository: mycompany/worker
    tag: "2.0.0"
  resources:
    requests:
      memory: "512Mi"
      cpu: "250m"
    limits:
      memory: "1Gi"
      cpu: "500m"

postgresql:
  enabled: true
  auth:
    postgresPassword: "secure-password"
    database: "appdb"
  primary:
    persistence:
      enabled: true
      size: 100Gi
      storageClass: "gp3"

redis:
  enabled: true
  auth:
    enabled: true
    password: "redis-password"
  master:
    persistence:
      enabled: true
      size: 20Gi

elasticsearch:
  enabled: true
  replicas: 3
  minimumMasterNodes: 2
  volumeClaimTemplate:
    accessModes: ["ReadWriteOnce"]
    storageClassName: "gp3"
    resources:
      requests:
        storage: 50Gi

monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
  prometheusRule:
    enabled: true
```

### Lab 4: Complete GitOps Workflow
```bash
#!/bin/bash
# setup-gitops.sh

# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd

# Get ArgoCD admin password
ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
echo "ArgoCD admin password: $ARGOCD_PASSWORD"

# Port forward to access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Install ArgoCD CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Login to ArgoCD
argocd login localhost:8080 --username admin --password $ARGOCD_PASSWORD --insecure

# Add Git repository
argocd repo add https://github.com/mycompany/k8s-manifests --username myuser --password mytoken

# Create application
argocd app create web-app \
  --repo https://github.com/mycompany/k8s-manifests \
  --path applications/web-app \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace production \
  --sync-policy automated \
  --auto-prune \
  --self-heal

echo "✅ GitOps workflow setup completed!"
echo "Access ArgoCD at: https://localhost:8080"
```

## Real-world Scenarios

### Scenario 1: Multi-Environment GitOps
```yaml
# environments/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patchesStrategicMerge:
- deployment-patch.yaml
- service-patch.yaml

images:
- name: myapp
  newTag: dev-latest

replicas:
- name: web-app
  count: 2

---
# environments/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../base

patchesStrategicMerge:
- deployment-patch.yaml
- hpa-patch.yaml

images:
- name: myapp
  newTag: v2.1.0

replicas:
- name: web-app
  count: 5

configMapGenerator:
- name: app-config
  files:
  - config.properties
```

### Scenario 2: Advanced Traffic Management
```yaml
# istio-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: web-app-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - app.mycompany.com
    tls:
      httpsRedirect: true
  - port:
      number: 443
      name: https
      protocol: HTTPS
    tls:
      mode: SIMPLE
      credentialName: myapp-tls
    hosts:
    - app.mycompany.com

---
# virtual-service.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: web-app-vs
spec:
  hosts:
  - app.mycompany.com
  gateways:
  - web-app-gateway
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: web-app-canary
        port:
          number: 80
      weight: 100
  - route:
    - destination:
        host: web-app-stable
        port:
          number: 80
      weight: 90
    - destination:
        host: web-app-canary
        port:
          number: 80
      weight: 10
```

## Interview Questions

### Beginner Level
1. **Q: What is GitOps and how does it differ from traditional CI/CD?**
   A: GitOps uses Git as the single source of truth for infrastructure and applications, with automated deployment agents pulling changes rather than pushing from CI/CD pipelines.

2. **Q: What are persistent volumes and why are they important?**
   A: Persistent volumes provide durable storage that survives pod restarts and rescheduling, essential for stateful applications like databases.

3. **Q: What is the purpose of an ingress controller?**
   A: Ingress controllers manage external access to services, providing HTTP/HTTPS routing, SSL termination, and load balancing.

### Intermediate Level
4. **Q: How do you implement canary deployments in Kubernetes?**
   A: Use service mesh like Istio for traffic splitting, or deploy multiple versions with weighted routing through ingress controllers or service configurations.

5. **Q: Explain the difference between StatefulSets and Deployments.**
   A: StatefulSets provide stable network identities and persistent storage for stateful applications, while Deployments are for stateless applications with interchangeable pods.

6. **Q: How do you handle secrets in a GitOps workflow?**
   A: Use sealed secrets, external secret operators, or tools like SOPS to encrypt secrets in Git, or integrate with external secret management systems.

### Advanced Level
7. **Q: How would you implement a complete observability stack in Kubernetes?**
   A: Deploy Prometheus for metrics, Grafana for visualization, Jaeger for tracing, and ELK/Loki for logging, with proper service discovery and alerting.

8. **Q: Describe a multi-cluster GitOps strategy.**
   A: Use ArgoCD ApplicationSets for multi-cluster deployments, implement cluster-specific configurations with Kustomize, and maintain separate Git repositories per environment.

9. **Q: How do you implement progressive delivery with GitOps?**
   A: Use Argo Rollouts for advanced deployment strategies, integrate with service mesh for traffic management, and implement automated rollback based on metrics.

10. **Q: Explain how to implement disaster recovery for Kubernetes applications.**
    A: Use cross-region persistent volume replication, implement backup strategies with Velero, maintain infrastructure as code, and practice regular disaster recovery drills.

## Key Takeaways
- Advanced monitoring provides comprehensive observability across the stack
- Persistent storage requires careful planning for performance and durability
- Ingress controllers enable sophisticated traffic management and routing
- GitOps provides declarative, auditable, and automated deployment workflows
- Service mesh technologies enable advanced traffic management and security
- Multi-environment strategies require proper configuration management and automation