# Day 37: Kubeadm Multi-Node Cluster Setup

## Learning Objectives
- Install and configure Kubeadm for multi-node Kubernetes cluster
- Deploy web applications using YAML manifests
- Configure cluster networking with CNI plugins
- Implement node management and cluster scaling
- Set up RBAC and security configurations

## Topics Covered

### 1. Kubeadm Cluster Architecture
```
Multi-Node Kubernetes Cluster:
┌─────────────────────────────────────────────────────────────┐
│                    Control Plane Node                       │
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
│   Worker Node 1      │ │Worker   │ │   Worker Node 3        │
│ ┌─────────────────┐  │ │Node 2   │ │ ┌─────────────────────┐│
│ │     kubelet     │  │ │         │ │ │      kubelet        ││
│ └─────────────────┘  │ │         │ │ └─────────────────────┘│
│ ┌─────────────────┐  │ │         │ │ ┌─────────────────────┐│
│ │   kube-proxy    │  │ │         │ │ │    kube-proxy       ││
│ └─────────────────┘  │ │         │ │ └─────────────────────┘│
│ ┌─────────────────┐  │ │         │ │ ┌─────────────────────┐│
│ │Container Runtime│  │ │         │ │ │ Container Runtime   ││
│ └─────────────────┘  │ │         │ │ └─────────────────────┘│
└──────────────────────┘ └─────────┘ └─────────────────────────┘
```

### 2. Cluster Installation Script
```bash
#!/bin/bash
# install-kubeadm.sh

set -e

echo "🚀 Installing Kubernetes with Kubeadm"

# Update system
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

# Install Docker
sudo apt-get install -y docker.io
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker $USER

# Add Kubernetes repository
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

# Install Kubernetes components
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

# Configure Docker daemon
sudo mkdir -p /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl restart docker
sudo systemctl enable docker

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# Configure kubelet
sudo systemctl enable kubelet

echo "✅ Kubeadm installation completed!"
```

### 3. Master Node Initialization
```bash
#!/bin/bash
# init-master.sh

# Initialize the cluster
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=$(hostname -I | awk '{print $1}') \
  --node-name=$(hostname)

# Configure kubectl for regular user
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Wait for nodes to be ready
echo "Waiting for nodes to be ready..."
kubectl wait --for=condition=Ready nodes --all --timeout=300s

# Get join command for worker nodes
echo "🔗 Use this command to join worker nodes:"
kubeadm token create --print-join-command

echo "✅ Master node initialized successfully!"
```

## Hands-on Labs

### Lab 1: Complete Cluster Setup
```bash
# On Master Node
#!/bin/bash
# setup-master.sh

# Install prerequisites
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
sudo usermod -aG docker $USER

# Install kubeadm, kubelet, kubectl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# Initialize cluster
sudo kubeadm init --pod-network-cidr=10.244.0.0/16

# Configure kubectl
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install CNI (Flannel)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# On Worker Nodes
#!/bin/bash
# setup-worker.sh

# Install Docker and Kubernetes components (same as master)
# Then join the cluster using the command from master:
# sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

### Lab 2: Web Application Deployment
```yaml
# web-app-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: web-application
  labels:
    name: web-application

---
# web-app-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: web-application
  labels:
    app: web-app
spec:
  replicas: 4
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
        image: nginx:1.21
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
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
        volumeMounts:
        - name: web-content
          mountPath: /usr/share/nginx/html
      volumes:
      - name: web-content
        configMap:
          name: web-content

---
# web-app-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-content
  namespace: web-application
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Kubeadm Cluster App</title>
        <style>
            body { font-family: Arial; text-align: center; margin-top: 50px; }
            .container { max-width: 800px; margin: 0 auto; }
            .status { background: #e8f5e8; padding: 20px; border-radius: 10px; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>🚀 Welcome to Kubeadm Cluster!</h1>
            <div class="status">
                <h2>Application Status: Running</h2>
                <p>Deployed on Multi-Node Kubernetes Cluster</p>
                <p>Pod: <span id="hostname"></span></p>
            </div>
        </div>
        <script>
            document.getElementById('hostname').textContent = window.location.hostname;
        </script>
    </body>
    </html>

---
# web-app-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
  namespace: web-application
spec:
  selector:
    app: web-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30080
  type: NodePort

---
# web-app-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-app-ingress
  namespace: web-application
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: webapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-app-service
            port:
              number: 80
```

### Lab 3: CNI Plugin Configuration
```bash
# Install different CNI plugins

# Option 1: Flannel (Simple overlay network)
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml

# Option 2: Calico (Advanced networking and security)
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml

# Option 3: Weave Net (Simple setup with encryption)
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Verify CNI installation
kubectl get pods -n kube-system
kubectl get nodes -o wide
```

### Lab 4: RBAC Configuration
```yaml
# rbac-config.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app-sa
  namespace: web-application

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: web-application
  name: web-app-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: web-app-rolebinding
  namespace: web-application
subjects:
- kind: ServiceAccount
  name: web-app-sa
  namespace: web-application
roleRef:
  kind: Role
  name: web-app-role
  apiGroup: rbac.authorization.k8s.io

---
# Update deployment to use service account
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: web-application
spec:
  template:
    spec:
      serviceAccountName: web-app-sa
      containers:
      - name: web-app
        image: nginx:1.21
```

## Real-world Scenarios

### Scenario 1: Node Management and Scaling
```bash
# Add new worker node
# On new node, run the join command from master initialization

# Check node status
kubectl get nodes
kubectl describe node <node-name>

# Drain node for maintenance
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Mark node as unschedulable
kubectl cordon <node-name>

# Remove node from cluster
kubectl delete node <node-name>

# On the node being removed:
sudo kubeadm reset
sudo rm -rf /etc/kubernetes/
sudo rm -rf ~/.kube/
sudo rm -rf /var/lib/etcd/
```

### Scenario 2: Cluster Backup and Recovery
```bash
#!/bin/bash
# backup-cluster.sh

# Backup etcd
sudo ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Backup certificates
sudo cp -r /etc/kubernetes/pki /backup/

# Backup kubeadm config
kubectl get configmap kubeadm-config -n kube-system -o yaml > /backup/kubeadm-config.yaml

echo "✅ Cluster backup completed"

# Restore etcd (if needed)
sudo ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore

# Update etcd configuration to use restored data
sudo systemctl stop etcd
sudo mv /var/lib/etcd /var/lib/etcd-old
sudo mv /var/lib/etcd-restore /var/lib/etcd
sudo systemctl start etcd
```

## Interview Questions

### Beginner Level
1. **Q: What is Kubeadm and how does it differ from managed Kubernetes services?**
   A: Kubeadm is a tool to bootstrap Kubernetes clusters on existing infrastructure, giving full control over cluster configuration unlike managed services like EKS or GKE.

2. **Q: What are the main components of a Kubernetes control plane?**
   A: API Server, etcd, Controller Manager, Scheduler, and optionally Cloud Controller Manager.

3. **Q: How do you add a new worker node to an existing Kubeadm cluster?**
   A: Install Kubernetes components on the new node, then run the kubeadm join command with the token from the master node.

### Intermediate Level
4. **Q: Explain the role of CNI plugins in Kubernetes networking.**
   A: CNI plugins provide network connectivity between pods, implement network policies, and handle IP address management across the cluster.

5. **Q: How do you troubleshoot a node that's not joining the cluster?**
   A: Check network connectivity, verify certificates, check kubelet logs, ensure proper DNS resolution, and validate join token.

6. **Q: What is the difference between cordon and drain operations?**
   A: Cordon marks a node as unschedulable, while drain safely evicts all pods from a node and marks it unschedulable.

### Advanced Level
7. **Q: How would you implement high availability for a Kubeadm cluster?**
   A: Set up multiple master nodes with external etcd cluster, use load balancer for API server, and implement proper backup and recovery procedures.

8. **Q: Explain the process of upgrading a Kubeadm cluster.**
   A: Upgrade kubeadm, drain and upgrade master nodes one by one, upgrade kubelet and kubectl, then upgrade worker nodes following the same pattern.

9. **Q: How do you implement custom RBAC policies for different teams?**
   A: Create namespaces for team isolation, define roles with specific permissions, create service accounts, and bind roles to users or groups using RoleBindings.

10. **Q: Describe disaster recovery strategies for Kubeadm clusters.**
    A: Regular etcd backups, certificate backup, configuration backup, documented recovery procedures, and testing recovery processes regularly.

## Key Takeaways
- Kubeadm provides production-ready Kubernetes clusters with full control
- Multi-node setup requires careful network and security configuration
- CNI plugins are essential for pod-to-pod communication
- RBAC provides fine-grained access control and security
- Regular backups and maintenance procedures are crucial
- Node management operations require understanding of cluster state