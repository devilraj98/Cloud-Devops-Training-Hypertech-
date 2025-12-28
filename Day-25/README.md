# Day 25: Docker Fundamentals and Containerization

## 📚 Learning Objectives
- Understand containerization concepts and benefits
- Master Docker installation and basic commands
- Create and manage Docker images and containers
- Practice Docker networking and volume management
- Implement Docker security best practices

## 🐳 Docker Overview

### What is Docker?
Docker is a containerization platform that packages applications and their dependencies into lightweight, portable containers that can run consistently across different environments.

### Key Concepts
- **Container**: Lightweight, standalone package containing everything needed to run an application
- **Image**: Read-only template used to create containers
- **Dockerfile**: Text file with instructions to build Docker images
- **Registry**: Repository for storing and sharing Docker images
- **Docker Hub**: Public registry for Docker images

### Benefits of Containerization
- **Consistency**: Same environment across development, testing, and production
- **Portability**: Run anywhere Docker is installed
- **Efficiency**: Lightweight compared to virtual machines
- **Scalability**: Easy to scale applications horizontally
- **Isolation**: Applications run in isolated environments

## 🛠️ Docker Installation and Setup

### Install Docker on Different Platforms

#### Ubuntu/Debian Installation
```bash
# Update package index
sudo apt update

# Install required packages
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add user to docker group
sudo usermod -aG docker $USER

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Verify installation
docker --version
docker run hello-world
```

#### CentOS/RHEL Installation
```bash
# Install required packages
sudo yum install -y yum-utils

# Add Docker repository
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Install Docker Engine
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Start and enable Docker
sudo systemctl start docker
sudo systemctl enable docker

# Add user to docker group
sudo usermod -aG docker $USER

# Verify installation
docker --version
```

#### Windows Installation
```bash
# Download Docker Desktop from https://www.docker.com/products/docker-desktop
# Run installer and follow setup wizard
# Restart computer after installation
# Verify in PowerShell:
docker --version
docker run hello-world
```

## 🔧 Docker Basic Commands

### Container Management
```bash
# Run a container
docker run hello-world
docker run -it ubuntu:20.04 /bin/bash
docker run -d --name webserver nginx
docker run -p 8080:80 nginx

# List containers
docker ps                    # Running containers
docker ps -a                 # All containers
docker ps -q                 # Only container IDs

# Container lifecycle
docker start container_name
docker stop container_name
docker restart container_name
docker pause container_name
docker unpause container_name

# Remove containers
docker rm container_name
docker rm -f container_name  # Force remove running container
docker container prune       # Remove all stopped containers

# Execute commands in running container
docker exec -it container_name /bin/bash
docker exec container_name ls -la

# View container logs
docker logs container_name
docker logs -f container_name  # Follow logs
docker logs --tail 50 container_name
```

### Image Management
```bash
# List images
docker images
docker image ls

# Pull images from registry
docker pull ubuntu:20.04
docker pull nginx:latest
docker pull python:3.11-slim

# Remove images
docker rmi image_name
docker rmi image_id
docker image prune           # Remove unused images
docker image prune -a        # Remove all unused images

# Image information
docker inspect image_name
docker history image_name

# Tag images
docker tag source_image:tag target_image:tag
docker tag nginx:latest myregistry.com/nginx:v1.0
```

### System Information
```bash
# Docker system information
docker info
docker version

# Disk usage
docker system df
docker system df -v

# Clean up system
docker system prune          # Remove unused data
docker system prune -a       # Remove all unused data
docker system prune --volumes # Include volumes
```

## 🏗️ Creating Docker Images

### Writing Dockerfiles

#### Basic Dockerfile Structure
```dockerfile
# Use official base image
FROM ubuntu:20.04

# Set maintainer information
LABEL maintainer="your-email@example.com"
LABEL description="Sample Docker image for learning"

# Set environment variables
ENV APP_HOME=/app
ENV DEBIAN_FRONTEND=noninteractive

# Set working directory
WORKDIR $APP_HOME

# Install system packages
RUN apt-get update && \
    apt-get install -y \
        python3 \
        python3-pip \
        curl \
        vim && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Copy application files
COPY requirements.txt .
COPY app.py .
COPY static/ ./static/
COPY templates/ ./templates/

# Install Python dependencies
RUN pip3 install --no-cache-dir -r requirements.txt

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser $APP_HOME
USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Default command
CMD ["python3", "app.py"]
```

#### Python Flask Application Dockerfile
```dockerfile
# Multi-stage build for Python Flask app
FROM python:3.11-slim as builder

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        build-essential \
        gcc && \
    rm -rf /var/lib/apt/lists/*

# Set work directory
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# Production stage
FROM python:3.11-slim

# Create app user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PATH=/home/appuser/.local/bin:$PATH

# Install runtime dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl && \
    rm -rf /var/lib/apt/lists/*

# Set work directory
WORKDIR /app

# Copy Python dependencies from builder stage
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appuser . .

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/api/health || exit 1

# Run application
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "4", "app:app"]
```

#### Node.js Application Dockerfile
```dockerfile
FROM node:18-alpine

# Set working directory
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# Copy application code
COPY --chown=nextjs:nodejs . .

# Switch to non-root user
USER nextjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
```

### Building and Managing Images
```bash
# Build image from Dockerfile
docker build -t myapp:latest .
docker build -t myapp:v1.0 -f Dockerfile.prod .

# Build with build arguments
docker build --build-arg VERSION=1.0 -t myapp:1.0 .

# Build with no cache
docker build --no-cache -t myapp:latest .

# Multi-platform build
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest .

# Save and load images
docker save myapp:latest > myapp.tar
docker load < myapp.tar

# Export and import containers
docker export container_name > container.tar
docker import container.tar myapp:imported
```

## 🌐 Docker Networking

### Network Types
```bash
# List networks
docker network ls

# Inspect network
docker network inspect bridge

# Create custom networks
docker network create mynetwork
docker network create --driver bridge --subnet=172.20.0.0/16 mybridge
docker network create --driver overlay myoverlay

# Connect containers to networks
docker run -d --name web --network mynetwork nginx
docker network connect mynetwork existing_container

# Disconnect from network
docker network disconnect mynetwork container_name

# Remove network
docker network rm mynetwork
```

### Container Communication
```bash
# Run containers on same network
docker network create webapp-network

# Database container
docker run -d \
    --name database \
    --network webapp-network \
    -e POSTGRES_DB=myapp \
    -e POSTGRES_USER=user \
    -e POSTGRES_PASSWORD=password \
    postgres:13

# Application container
docker run -d \
    --name webapp \
    --network webapp-network \
    -p 8080:5000 \
    -e DATABASE_URL=postgresql://user:password@database:5432/myapp \
    myapp:latest

# Test connectivity
docker exec webapp ping database
docker exec webapp nslookup database
```

## 💾 Docker Volumes and Data Management

### Volume Types and Management
```bash
# Create named volume
docker volume create mydata
docker volume create --driver local myvolume

# List volumes
docker volume ls

# Inspect volume
docker volume inspect mydata

# Use volumes in containers
docker run -d -v mydata:/data ubuntu:20.04
docker run -d --mount source=mydata,target=/data ubuntu:20.04

# Bind mounts
docker run -d -v /host/path:/container/path ubuntu:20.04
docker run -d --mount type=bind,source=/host/path,target=/container/path ubuntu:20.04

# Temporary filesystems
docker run -d --tmpfs /tmp ubuntu:20.04
docker run -d --mount type=tmpfs,destination=/tmp ubuntu:20.04

# Remove volumes
docker volume rm mydata
docker volume prune  # Remove unused volumes
```

### Data Persistence Examples
```bash
# Database with persistent data
docker run -d \
    --name postgres-db \
    -v postgres-data:/var/lib/postgresql/data \
    -e POSTGRES_DB=myapp \
    -e POSTGRES_USER=user \
    -e POSTGRES_PASSWORD=password \
    postgres:13

# Web server with configuration
docker run -d \
    --name nginx-server \
    -v /host/nginx.conf:/etc/nginx/nginx.conf:ro \
    -v /host/html:/usr/share/nginx/html:ro \
    -p 80:80 \
    nginx

# Development environment with code mounting
docker run -it \
    --name dev-env \
    -v $(pwd):/workspace \
    -w /workspace \
    python:3.11 \
    bash
```

## 🧪 Hands-on Lab: Containerizing Applications

### Lab 1: Simple Web Application

#### 1.1 Create Flask Application
```bash
# Create project directory
mkdir docker-flask-app
cd docker-flask-app

# Create Flask application
cat > app.py << 'EOF'
from flask import Flask, jsonify, render_template_string
import os
import socket
from datetime import datetime

app = Flask(__name__)

HTML_TEMPLATE = '''
<!DOCTYPE html>
<html>
<head>
    <title>Docker Flask App</title>
    <style>
        body { font-family: Arial; margin: 40px; background: #f0f8ff; }
        .container { background: white; padding: 30px; border-radius: 10px; }
        .info { background: #e7f3ff; padding: 15px; margin: 15px 0; border-radius: 5px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🐳 Docker Flask Application</h1>
        <div class="info">
            <h3>Container Information:</h3>
            <p><strong>Hostname:</strong> {{ hostname }}</p>
            <p><strong>Environment:</strong> {{ environment }}</p>
            <p><strong>Timestamp:</strong> {{ timestamp }}</p>
            <p><strong>Version:</strong> {{ version }}</p>
        </div>
        <div class="info">
            <h3>Docker Benefits:</h3>
            <ul>
                <li>Consistent environments across development and production</li>
                <li>Easy application deployment and scaling</li>
                <li>Isolated application dependencies</li>
                <li>Efficient resource utilization</li>
            </ul>
        </div>
    </div>
</body>
</html>
'''

@app.route('/')
def home():
    return render_template_string(HTML_TEMPLATE,
        hostname=socket.gethostname(),
        environment=os.environ.get('ENVIRONMENT', 'development'),
        timestamp=datetime.now().strftime('%Y-%m-%d %H:%M:%S'),
        version=os.environ.get('APP_VERSION', '1.0.0')
    )

@app.route('/api/health')
def health():
    return jsonify({
        'status': 'healthy',
        'hostname': socket.gethostname(),
        'timestamp': datetime.now().isoformat(),
        'version': os.environ.get('APP_VERSION', '1.0.0')
    })

@app.route('/api/info')
def info():
    return jsonify({
        'hostname': socket.gethostname(),
        'environment': os.environ.get('ENVIRONMENT', 'development'),
        'python_version': os.sys.version,
        'flask_version': Flask.__version__
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
EOF

# Create requirements.txt
cat > requirements.txt << 'EOF'
Flask==3.0.0
gunicorn==21.2.0
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV APP_VERSION=1.0.0

# Set work directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app
USER appuser

# Copy application code
COPY --chown=appuser:appuser app.py .

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/api/health || exit 1

# Run application
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
EOF
```

#### 1.2 Build and Run Container
```bash
# Build Docker image
docker build -t flask-app:1.0 .

# Run container
docker run -d \
    --name flask-container \
    -p 8080:5000 \
    -e ENVIRONMENT=production \
    -e APP_VERSION=1.0.0 \
    flask-app:1.0

# Test application
curl http://localhost:8080/api/health
curl http://localhost:8080/api/info

# View logs
docker logs flask-container

# Access container shell
docker exec -it flask-container /bin/bash
```

### Lab 2: Multi-Container Application with Docker Compose

#### 2.1 Create Docker Compose Configuration
```bash
# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # Database service
  database:
    image: postgres:13
    container_name: postgres-db
    environment:
      POSTGRES_DB: taskmanager
      POSTGRES_USER: dbuser
      POSTGRES_PASSWORD: dbpassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - app-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dbuser -d taskmanager"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis cache service
  redis:
    image: redis:7-alpine
    container_name: redis-cache
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Web application service
  webapp:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: flask-webapp
    environment:
      - ENVIRONMENT=production
      - DATABASE_URL=postgresql://dbuser:dbpassword@database:5432/taskmanager
      - REDIS_URL=redis://redis:6379/0
      - APP_VERSION=2.0.0
    ports:
      - "8080:5000"
    depends_on:
      database:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped

  # Nginx reverse proxy
  nginx:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    depends_on:
      - webapp
    networks:
      - app-network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
EOF

# Create database initialization script
cat > init.sql << 'EOF'
-- Create tasks table
CREATE TABLE IF NOT EXISTS tasks (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    status VARCHAR(20) DEFAULT 'pending',
    priority VARCHAR(10) DEFAULT 'medium',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO tasks (title, description, priority) VALUES
    ('Setup Docker Environment', 'Configure Docker for development', 'high'),
    ('Learn Docker Compose', 'Understand multi-container applications', 'medium'),
    ('Implement CI/CD Pipeline', 'Setup automated deployment', 'high');
EOF

# Create Nginx configuration
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream webapp {
        server webapp:5000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://webapp;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF
```

#### 2.2 Run Multi-Container Application
```bash
# Start all services
docker-compose up -d

# View running services
docker-compose ps

# View logs
docker-compose logs webapp
docker-compose logs -f  # Follow all logs

# Scale web application
docker-compose up -d --scale webapp=3

# Stop services
docker-compose stop

# Remove services and volumes
docker-compose down -v
```

## 🔒 Docker Security Best Practices

### Image Security
```dockerfile
# Use official base images
FROM python:3.11-slim

# Don't run as root
RUN useradd -m -u 1000 appuser
USER appuser

# Use specific image tags
FROM nginx:1.21.6-alpine

# Minimize attack surface
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Use multi-stage builds
FROM node:18-alpine as builder
# Build stage
FROM nginx:alpine as production
# Production stage with minimal dependencies
```

### Runtime Security
```bash
# Run with limited privileges
docker run --user 1000:1000 myapp

# Limit resources
docker run --memory=512m --cpus=1.0 myapp

# Read-only filesystem
docker run --read-only --tmpfs /tmp myapp

# Drop capabilities
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE myapp

# Use security profiles
docker run --security-opt=no-new-privileges myapp
```

### Secrets Management
```bash
# Use Docker secrets (Swarm mode)
echo "mysecretpassword" | docker secret create db_password -

# Use environment files
docker run --env-file .env myapp

# Mount secrets as files
docker run -v /host/secrets:/run/secrets:ro myapp
```

## 📚 Interview Questions & Answers

### Fresher Level (1-10)

**Q1: What is the difference between a Docker image and a container?**
A: A Docker image is a read-only template containing application code, dependencies, and configuration. A container is a running instance of an image with its own filesystem, network, and process space.

**Q2: What is a Dockerfile?**
A: A Dockerfile is a text file containing instructions to build a Docker image. It defines the base image, dependencies, configuration, and commands needed to create the application environment.

**Q3: How do you list running Docker containers?**
A: Use `docker ps` to list running containers, `docker ps -a` to list all containers (including stopped ones), and `docker ps -q` to list only container IDs.

**Q4: What is the purpose of Docker volumes?**
A: Docker volumes provide persistent data storage that survives container restarts and deletions. They enable data sharing between containers and between host and containers.

**Q5: How do you expose a port in Docker?**
A: Use the `EXPOSE` instruction in Dockerfile to document ports, and `-p` flag when running containers: `docker run -p 8080:80 nginx` maps host port 8080 to container port 80.

### Intermediate Level (6-15)

**Q6: What are the advantages of multi-stage Docker builds?**
A: Multi-stage builds reduce image size by separating build and runtime environments, improve security by excluding build tools from production images, and enable better caching and optimization.

**Q7: How do you handle secrets in Docker containers?**
A: Use Docker secrets (in Swarm mode), environment variables for non-sensitive config, mounted files for secrets, external secret management systems, and avoid hardcoding secrets in images.

**Q8: What is Docker networking and what are the different network types?**
A: Docker networking enables container communication. Types include bridge (default, single host), host (uses host network), overlay (multi-host), macvlan (MAC addresses), and none (no networking).

**Q9: How do you optimize Docker images for production?**
A: Use minimal base images (Alpine), multi-stage builds, specific tags, remove unnecessary packages, use .dockerignore, minimize layers, and implement proper caching strategies.

**Q10: What is Docker Compose and when would you use it?**
A: Docker Compose is a tool for defining and running multi-container applications using YAML files. Use it for development environments, testing, single-host deployments, and orchestrating related services.

## 🔑 Key Takeaways

- **Containerization Benefits**: Consistency, portability, efficiency, and isolation
- **Image Optimization**: Use multi-stage builds and minimal base images
- **Security First**: Follow security best practices from development to production
- **Data Management**: Understand volumes and persistent storage patterns
- **Networking**: Master container communication and network types
- **Orchestration**: Use Docker Compose for multi-container applications

## 🚀 Next Steps

- Day 26: Hands-on Lab - Docker Advanced Features
- Day 27: Two-Tier Project using Docker
- Day 28: Docker Volumes and Docker Compose Deep Dive
- Container orchestration with Kubernetes

---

**Hands-on Completed:** ✅ Docker Installation, Image Creation, Container Management, Multi-Container Apps  
**Duration:** 4-5 hours  
**Difficulty:** Beginner to Intermediate