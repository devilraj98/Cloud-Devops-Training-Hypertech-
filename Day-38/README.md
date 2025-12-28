# Day 38: Amazon EKS & Helm Chart Deployments

## Learning Objectives
- Set up Amazon EKS cluster with proper configuration
- Deploy web applications using Helm charts
- Implement auto-scaling and resource management
- Configure production deployment strategies
- Integrate AWS Load Balancer Controller and Fargate profiles

## Topics Covered

### 1. EKS Cluster Architecture
```
Amazon EKS Architecture:
┌─────────────────────────────────────────────────────────────┐
│                    AWS Managed Control Plane                │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐│
│  │ API Server  │ │    etcd     │ │ Controller Manager      ││
│  └─────────────┘ └─────────────┘ └─────────────────────────┘│
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐│
│  │  Scheduler  │ │   kubelet   │ │      kube-proxy         ││
│  └─────────────┘ └─────────────┘ └─────────────────────────┘│
└─────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┼─────────┐
                    │         │         │
┌───────────────────▼──┐ ┌────▼────┐ ┌──▼─────────────────────┐
│   Node Group 1       │ │Fargate  │ │   Node Group 2         │
│ ┌─────────────────┐  │ │Profile  │ │ ┌─────────────────────┐│
│ │   EC2 Instances │  │ │         │ │ │   EC2 Instances     ││
│ │   (On-Demand)   │  │ │         │ │ │   (Spot)            ││
│ └─────────────────┘  │ │         │ │ └─────────────────────┘│
└──────────────────────┘ └─────────┘ └─────────────────────────┘
```

### 2. EKS Cluster Setup with eksctl
```bash
#!/bin/bash
# setup-eks.sh

# Install eksctl
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin

# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Create EKS cluster
eksctl create cluster \
  --name production-cluster \
  --version 1.28 \
  --region us-west-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 1 \
  --nodes-max 6 \
  --managed \
  --with-oidc \
  --ssh-access \
  --ssh-public-key ~/.ssh/id_rsa.pub

# Update kubeconfig
aws eks update-kubeconfig --region us-west-2 --name production-cluster

echo "✅ EKS cluster created successfully!"
```

### 3. EKS Cluster Configuration YAML
```yaml
# eks-cluster.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: production-cluster
  region: us-west-2
  version: "1.28"

iam:
  withOIDC: true
  serviceAccounts:
  - metadata:
      name: aws-load-balancer-controller
      namespace: kube-system
    wellKnownPolicies:
      awsLoadBalancerController: true
  - metadata:
      name: cluster-autoscaler
      namespace: kube-system
    wellKnownPolicies:
      autoScaler: true

managedNodeGroups:
- name: standard-workers
  instanceType: t3.medium
  minSize: 2
  maxSize: 8
  desiredCapacity: 3
  volumeSize: 20
  ssh:
    allow: true
    publicKeyName: my-key-pair
  labels:
    role: worker
    environment: production
  tags:
    Environment: production
    Project: web-application

- name: spot-workers
  instanceTypes: ["t3.medium", "t3.large", "t2.medium"]
  spot: true
  minSize: 0
  maxSize: 10
  desiredCapacity: 2
  labels:
    role: spot-worker
    environment: production
  tags:
    Environment: production
    Project: web-application

fargateProfiles:
- name: fp-default
  selectors:
  - namespace: default
    labels:
      compute-type: fargate
- name: fp-kube-system
  selectors:
  - namespace: kube-system

addons:
- name: vpc-cni
  version: latest
- name: coredns
  version: latest
- name: kube-proxy
  version: latest
- name: aws-ebs-csi-driver
  version: latest

cloudWatch:
  clusterLogging:
    enable: ["audit", "authenticator", "controllerManager"]
```

## Hands-on Labs

### Lab 1: Complete EKS Setup with Terraform
```hcl
# eks-cluster.tf
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "production-cluster"
  cluster_version = "1.28"

  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true

  eks_managed_node_groups = {
    standard = {
      min_size     = 2
      max_size     = 6
      desired_size = 3

      instance_types = ["t3.medium"]
      capacity_type  = "ON_DEMAND"

      labels = {
        Environment = "production"
        NodeGroup   = "standard"
      }

      tags = {
        Environment = "production"
        Terraform   = "true"
      }
    }

    spot = {
      min_size     = 0
      max_size     = 10
      desired_size = 2

      instance_types = ["t3.medium", "t3.large"]
      capacity_type  = "SPOT"

      labels = {
        Environment = "production"
        NodeGroup   = "spot"
      }
    }
  }

  fargate_profiles = {
    default = {
      name = "default"
      selectors = [
        {
          namespace = "default"
          labels = {
            compute-type = "fargate"
          }
        }
      ]
    }
  }

  tags = {
    Environment = "production"
    Terraform   = "true"
  }
}

# Install AWS Load Balancer Controller
resource "helm_release" "aws_load_balancer_controller" {
  name       = "aws-load-balancer-controller"
  repository = "https://aws.github.io/eks-charts"
  chart      = "aws-load-balancer-controller"
  namespace  = "kube-system"
  version    = "1.6.2"

  set {
    name  = "clusterName"
    value = module.eks.cluster_name
  }

  set {
    name  = "serviceAccount.create"
    value = "false"
  }

  set {
    name  = "serviceAccount.name"
    value = "aws-load-balancer-controller"
  }

  depends_on = [module.eks]
}
```

### Lab 2: Helm Chart for Web Application
```yaml
# Chart.yaml
apiVersion: v2
name: web-application
description: A Helm chart for web application deployment
type: application
version: 0.1.0
appVersion: "1.0.0"

dependencies:
- name: postgresql
  version: 12.1.2
  repository: https://charts.bitnami.com/bitnami
- name: redis
  version: 17.3.7
  repository: https://charts.bitnami.com/bitnami

---
# values.yaml
replicaCount: 3

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.21"

service:
  type: LoadBalancer
  port: 80
  targetPort: 80

ingress:
  enabled: true
  className: "alb"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
  hosts:
    - host: webapp.example.com
      paths:
        - path: /
          pathType: Prefix

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

postgresql:
  enabled: true
  auth:
    postgresPassword: "secure-password"
    database: "webapp"
  primary:
    persistence:
      enabled: true
      size: 10Gi

redis:
  enabled: true
  auth:
    enabled: false
  master:
    persistence:
      enabled: true
      size: 5Gi

---
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "web-application.fullname" . }}
  labels:
    {{- include "web-application.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "web-application.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "web-application.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          livenessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          env:
          - name: DATABASE_URL
            value: "postgresql://postgres:{{ .Values.postgresql.auth.postgresPassword }}@{{ include "web-application.fullname" . }}-postgresql:5432/{{ .Values.postgresql.auth.database }}"
          - name: REDIS_URL
            value: "redis://{{ include "web-application.fullname" . }}-redis-master:6379"

---
# templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "web-application.fullname" . }}
  labels:
    {{- include "web-application.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "web-application.selectorLabels" . | nindent 4 }}

---
# templates/hpa.yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "web-application.fullname" . }}
  labels:
    {{- include "web-application.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "web-application.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### Lab 3: AWS Load Balancer Controller Setup
```bash
#!/bin/bash
# setup-alb-controller.sh

# Create IAM OIDC provider
eksctl utils associate-iam-oidc-provider \
    --region us-west-2 \
    --cluster production-cluster \
    --approve

# Create service account
eksctl create iamserviceaccount \
  --cluster=production-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::aws:policy/ElasticLoadBalancingFullAccess \
  --approve

# Install AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
helm repo update

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=production-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify installation
kubectl get deployment -n kube-system aws-load-balancer-controller
```

### Lab 4: Complete Application Deployment
```bash
#!/bin/bash
# deploy-application.sh

# Create namespace
kubectl create namespace web-app

# Add Helm repositories
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Install application using Helm
helm install web-application ./web-application-chart \
  --namespace web-app \
  --values values-production.yaml \
  --wait

# Check deployment status
kubectl get pods -n web-app
kubectl get services -n web-app
kubectl get ingress -n web-app

# Get Load Balancer URL
kubectl get ingress web-application -n web-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'

echo "✅ Application deployed successfully!"
```

## Real-world Scenarios

### Scenario 1: Production Deployment Strategy
```yaml
# production-values.yaml
replicaCount: 5

image:
  repository: mycompany/webapp
  tag: "v2.1.0"

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

autoscaling:
  enabled: true
  minReplicas: 5
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60

ingress:
  enabled: true
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-west-2:123456789:certificate/abc123
  hosts:
    - host: api.mycompany.com
      paths:
        - path: /
          pathType: Prefix

postgresql:
  enabled: true
  auth:
    postgresPassword: "super-secure-password"
  primary:
    persistence:
      enabled: true
      size: 100Gi
      storageClass: gp3
  metrics:
    enabled: true

redis:
  enabled: true
  auth:
    enabled: true
    password: "redis-secure-password"
  master:
    persistence:
      enabled: true
      size: 20Gi
```

### Scenario 2: Blue-Green Deployment with Helm
```bash
#!/bin/bash
# blue-green-deployment.sh

NAMESPACE="web-app"
CURRENT_VERSION=$(helm get values web-application -n $NAMESPACE -o json | jq -r '.image.tag')
NEW_VERSION="v2.2.0"

echo "Current version: $CURRENT_VERSION"
echo "Deploying version: $NEW_VERSION"

# Deploy green version
helm upgrade web-application ./web-application-chart \
  --namespace $NAMESPACE \
  --set image.tag=$NEW_VERSION \
  --set service.name=web-application-green \
  --wait

# Health check
kubectl wait --for=condition=available --timeout=300s deployment/web-application -n $NAMESPACE

# Switch traffic (update ingress)
kubectl patch ingress web-application -n $NAMESPACE -p '{"spec":{"rules":[{"host":"webapp.example.com","http":{"paths":[{"path":"/","pathType":"Prefix","backend":{"service":{"name":"web-application-green","port":{"number":80}}}}]}}]}}'

echo "✅ Blue-green deployment completed!"
```

## Interview Questions

### Beginner Level
1. **Q: What are the advantages of using EKS over self-managed Kubernetes?**
   A: Managed control plane, automatic updates, AWS integration, high availability, security patches, and reduced operational overhead.

2. **Q: What is Helm and why is it useful for Kubernetes deployments?**
   A: Helm is a package manager for Kubernetes that simplifies application deployment, manages dependencies, and enables templating and versioning.

3. **Q: How do you expose a service to the internet in EKS?**
   A: Use LoadBalancer service type with AWS Load Balancer Controller, or create an Ingress resource with ALB Ingress Controller.

### Intermediate Level
4. **Q: Explain the difference between EKS node groups and Fargate profiles.**
   A: Node groups use EC2 instances you manage, while Fargate provides serverless compute where AWS manages the infrastructure automatically.

5. **Q: How do you implement auto-scaling in EKS?**
   A: Use Horizontal Pod Autoscaler for pod scaling, Vertical Pod Autoscaler for resource scaling, and Cluster Autoscaler for node scaling.

6. **Q: What are Helm hooks and when would you use them?**
   A: Helm hooks allow you to intervene at certain points in a release lifecycle, useful for database migrations, backups, or cleanup tasks.

### Advanced Level
7. **Q: How would you implement a multi-environment deployment strategy with Helm?**
   A: Use separate values files per environment, implement GitOps with ArgoCD, use Helm chart dependencies, and implement proper secret management.

8. **Q: Explain how to implement cost optimization in EKS.**
   A: Use spot instances, right-size resources, implement cluster autoscaling, use Fargate for appropriate workloads, and monitor resource utilization.

9. **Q: How do you handle secrets management in EKS with Helm?**
   A: Use AWS Secrets Manager with External Secrets Operator, implement sealed secrets, use Helm secrets plugin, or integrate with HashiCorp Vault.

10. **Q: Describe a complete CI/CD pipeline for EKS deployments.**
    A: Include code build, container image creation, security scanning, Helm chart packaging, deployment to staging, testing, and production deployment with approval gates.

## Key Takeaways
- EKS provides managed Kubernetes with AWS integration
- Helm simplifies complex application deployments and management
- AWS Load Balancer Controller enables advanced ingress capabilities
- Fargate provides serverless container compute options
- Auto-scaling ensures optimal resource utilization and cost management
- Production deployments require careful planning for security and reliability