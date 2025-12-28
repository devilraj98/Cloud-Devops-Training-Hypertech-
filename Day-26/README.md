# Day 26: Docker Advanced Features - Hands-on Lab

## 📚 Learning Objectives
- Master Docker images and containers in production
- Implement Docker networking and service discovery
- Practice Docker volumes and data persistence
- Host websites using Docker and Nginx
- Understand Docker security and optimization

## 🛠️ Hands-on Lab: Website Hosting with Docker

### Lab 1: Static Website with Nginx

#### 1.1 Create Website Structure
```bash
mkdir docker-website-lab
cd docker-website-lab

# Create website files
mkdir -p website/{css,js,images}

cat > website/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Docker Website Demo</title>
    <link rel="stylesheet" href="css/style.css">
</head>
<body>
    <header>
        <nav>
            <div class="logo">🐳 Docker Demo</div>
            <ul>
                <li><a href="#home">Home</a></li>
                <li><a href="#about">About</a></li>
                <li><a href="#services">Services</a></li>
                <li><a href="#contact">Contact</a></li>
            </ul>
        </nav>
    </header>

    <main>
        <section id="home" class="hero">
            <h1>Welcome to Docker Website</h1>
            <p>This website is hosted using Docker and Nginx</p>
            <button onclick="showInfo()">Show Container Info</button>
        </section>

        <section id="about" class="section">
            <h2>About Docker</h2>
            <div class="cards">
                <div class="card">
                    <h3>🚀 Fast Deployment</h3>
                    <p>Deploy applications quickly and consistently</p>
                </div>
                <div class="card">
                    <h3>📦 Portable</h3>
                    <p>Run anywhere Docker is installed</p>
                </div>
                <div class="card">
                    <h3>🔧 Scalable</h3>
                    <p>Scale applications horizontally with ease</p>
                </div>
            </div>
        </section>

        <section id="services" class="section">
            <h2>Our Services</h2>
            <ul>
                <li>Container Orchestration</li>
                <li>Microservices Architecture</li>
                <li>DevOps Consulting</li>
                <li>Cloud Migration</li>
            </ul>
        </section>
    </main>

    <footer>
        <p>&copy; 2024 Docker Demo Website. Containerized with ❤️</p>
        <div id="container-info"></div>
    </footer>

    <script src="js/script.js"></script>
</body>
</html>
EOF

cat > website/css/style.css << 'EOF'
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Arial', sans-serif;
    line-height: 1.6;
    color: #333;
}

header {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    padding: 1rem 0;
    position: fixed;
    width: 100%;
    top: 0;
    z-index: 1000;
}

nav {
    display: flex;
    justify-content: space-between;
    align-items: center;
    max-width: 1200px;
    margin: 0 auto;
    padding: 0 2rem;
}

.logo {
    font-size: 1.5rem;
    font-weight: bold;
}

nav ul {
    display: flex;
    list-style: none;
}

nav ul li {
    margin-left: 2rem;
}

nav ul li a {
    color: white;
    text-decoration: none;
    transition: opacity 0.3s;
}

nav ul li a:hover {
    opacity: 0.8;
}

.hero {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    text-align: center;
    padding: 150px 2rem 100px;
    margin-top: 60px;
}

.hero h1 {
    font-size: 3rem;
    margin-bottom: 1rem;
}

.hero p {
    font-size: 1.2rem;
    margin-bottom: 2rem;
}

button {
    background: white;
    color: #667eea;
    border: none;
    padding: 12px 24px;
    border-radius: 25px;
    cursor: pointer;
    font-size: 1rem;
    transition: transform 0.3s;
}

button:hover {
    transform: translateY(-2px);
}

.section {
    padding: 80px 2rem;
    max-width: 1200px;
    margin: 0 auto;
}

.section h2 {
    text-align: center;
    margin-bottom: 3rem;
    font-size: 2.5rem;
}

.cards {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 2rem;
    margin-top: 2rem;
}

.card {
    background: #f8f9fa;
    padding: 2rem;
    border-radius: 10px;
    text-align: center;
    box-shadow: 0 4px 6px rgba(0,0,0,0.1);
    transition: transform 0.3s;
}

.card:hover {
    transform: translateY(-5px);
}

.card h3 {
    margin-bottom: 1rem;
    color: #667eea;
}

#services ul {
    max-width: 600px;
    margin: 0 auto;
}

#services li {
    background: #e7f3ff;
    margin: 1rem 0;
    padding: 1rem;
    border-radius: 5px;
    border-left: 4px solid #667eea;
}

footer {
    background: #333;
    color: white;
    text-align: center;
    padding: 2rem;
}

#container-info {
    margin-top: 1rem;
    padding: 1rem;
    background: #444;
    border-radius: 5px;
    font-family: monospace;
    font-size: 0.9rem;
}
EOF

cat > website/js/script.js << 'EOF'
function showInfo() {
    const info = document.getElementById('container-info');
    
    // Simulate container information
    const containerInfo = {
        hostname: 'docker-container-' + Math.random().toString(36).substr(2, 9),
        timestamp: new Date().toISOString(),
        version: '1.0.0',
        environment: 'production'
    };
    
    info.innerHTML = `
        <strong>Container Information:</strong><br>
        Hostname: ${containerInfo.hostname}<br>
        Timestamp: ${containerInfo.timestamp}<br>
        Version: ${containerInfo.version}<br>
        Environment: ${containerInfo.environment}
    `;
    
    info.style.display = 'block';
}

// Smooth scrolling for navigation links
document.querySelectorAll('a[href^="#"]').forEach(anchor => {
    anchor.addEventListener('click', function (e) {
        e.preventDefault();
        const target = document.querySelector(this.getAttribute('href'));
        if (target) {
            target.scrollIntoView({
                behavior: 'smooth',
                block: 'start'
            });
        }
    });
});
EOF
```

#### 1.2 Create Dockerfile for Static Website
```bash
cat > Dockerfile << 'EOF'
FROM nginx:alpine

# Copy website files
COPY website/ /usr/share/nginx/html/

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Create non-root user
RUN addgroup -g 1001 -S www && \
    adduser -S www -u 1001 -G www

# Set permissions
RUN chown -R www:www /usr/share/nginx/html && \
    chown -R www:www /var/cache/nginx && \
    chown -R www:www /var/log/nginx && \
    chown -R www:www /etc/nginx/conf.d

# Switch to non-root user
USER www

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/ || exit 1

# Start nginx
CMD ["nginx", "-g", "daemon off;"]
EOF

cat > nginx.conf << 'EOF'
user www;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /tmp/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
    
    access_log /var/log/nginx/access.log main;
    
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;
    
    gzip on;
    gzip_vary on;
    gzip_min_length 10240;
    gzip_proxied expired no-cache no-store private must-revalidate auth;
    gzip_types
        text/plain
        text/css
        text/xml
        text/javascript
        application/x-javascript
        application/xml+rss;
    
    server {
        listen 8080;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;
        
        location / {
            try_files $uri $uri/ /index.html;
        }
        
        location ~* \.(css|js|png|jpg|jpeg|gif|ico|svg)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
        
        location = /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF
```

#### 1.3 Build and Run Website Container
```bash
# Build image
docker build -t docker-website:1.0 .

# Run container
docker run -d \
    --name website-container \
    -p 8080:8080 \
    docker-website:1.0

# Test website
curl http://localhost:8080
curl http://localhost:8080/health

# View logs
docker logs website-container

# Access website in browser: http://localhost:8080
```

### Lab 2: Multi-Container Web Application

#### 2.1 Create Docker Compose Setup
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # Load balancer
  nginx:
    image: nginx:alpine
    container_name: nginx-lb
    ports:
      - "80:80"
    volumes:
      - ./nginx-lb.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web1
      - web2
      - web3
    networks:
      - web-network
    restart: unless-stopped

  # Web server instances
  web1:
    build: .
    container_name: web-server-1
    environment:
      - SERVER_ID=web-1
    networks:
      - web-network
    restart: unless-stopped

  web2:
    build: .
    container_name: web-server-2
    environment:
      - SERVER_ID=web-2
    networks:
      - web-network
    restart: unless-stopped

  web3:
    build: .
    container_name: web-server-3
    environment:
      - SERVER_ID=web-3
    networks:
      - web-network
    restart: unless-stopped

  # Monitoring
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - portainer_data:/data
    networks:
      - web-network
    restart: unless-stopped

volumes:
  portainer_data:

networks:
  web-network:
    driver: bridge
EOF

cat > nginx-lb.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream web_servers {
        server web1:8080;
        server web2:8080;
        server web3:8080;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://web_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        
        location /health {
            access_log off;
            return 200 "Load Balancer OK\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF
```

#### 2.2 Run Multi-Container Setup
```bash
# Start all services
docker-compose up -d

# Scale web servers
docker-compose up -d --scale web1=2 --scale web2=2

# View running containers
docker-compose ps

# Test load balancing
for i in {1..10}; do
    curl -s http://localhost/ | grep -o "web-[0-9]"
done

# Access Portainer: http://localhost:9000
# Monitor containers and resources
```

### Lab 3: Docker Security Implementation

#### 3.1 Secure Dockerfile
```bash
cat > Dockerfile.secure << 'EOF'
# Use specific version, not latest
FROM nginx:1.21.6-alpine

# Install security updates
RUN apk update && apk upgrade && \
    apk add --no-cache curl && \
    rm -rf /var/cache/apk/*

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Copy files with proper ownership
COPY --chown=appuser:appgroup website/ /usr/share/nginx/html/
COPY --chown=appuser:appgroup nginx-secure.conf /etc/nginx/nginx.conf

# Set proper permissions
RUN chmod -R 755 /usr/share/nginx/html && \
    chown -R appuser:appgroup /var/cache/nginx && \
    chown -R appuser:appgroup /var/log/nginx && \
    touch /var/run/nginx.pid && \
    chown appuser:appgroup /var/run/nginx.pid

# Switch to non-root user
USER appuser

# Use non-privileged port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

# Run nginx
CMD ["nginx", "-g", "daemon off;"]
EOF

cat > nginx-secure.conf << 'EOF'
user appuser;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "no-referrer-when-downgrade" always;
    
    # Hide nginx version
    server_tokens off;
    
    server {
        listen 8080;
        server_name localhost;
        root /usr/share/nginx/html;
        index index.html;
        
        # Security configurations
        location / {
            try_files $uri $uri/ /index.html;
        }
        
        # Deny access to hidden files
        location ~ /\. {
            deny all;
        }
        
        location = /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF
```

#### 3.2 Run Secure Container
```bash
# Build secure image
docker build -f Dockerfile.secure -t secure-website:1.0 .

# Run with security options
docker run -d \
    --name secure-website \
    --user 1001:1001 \
    --read-only \
    --tmpfs /tmp \
    --tmpfs /var/run \
    --tmpfs /var/cache/nginx \
    --cap-drop=ALL \
    --cap-add=NET_BIND_SERVICE \
    --security-opt=no-new-privileges \
    -p 8080:8080 \
    secure-website:1.0

# Test security
docker exec secure-website whoami
docker exec secure-website ps aux
```

## 🔧 Docker Image Optimization

### Multi-Stage Build Example
```bash
cat > Dockerfile.optimized << 'EOF'
# Build stage
FROM node:18-alpine as builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

# Production stage
FROM nginx:alpine as production

# Install curl for health checks
RUN apk add --no-cache curl

# Copy built application
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf

# Create non-root user
RUN addgroup -g 1001 -S www && \
    adduser -S www -u 1001 -G www && \
    chown -R www:www /usr/share/nginx/html

USER www
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

CMD ["nginx", "-g", "daemon off;"]
EOF
```

### Image Size Comparison
```bash
# Compare image sizes
docker images | grep website

# Analyze image layers
docker history docker-website:1.0

# Use dive tool for detailed analysis
docker run --rm -it \
    -v /var/run/docker.sock:/var/run/docker.sock \
    wagoodman/dive:latest docker-website:1.0
```

## 📚 Interview Questions & Answers

### Intermediate Level (1-10)

**Q1: How do you optimize Docker images for production?**
A: Use multi-stage builds, minimal base images (Alpine), specific tags, remove unnecessary packages, use .dockerignore, minimize layers, implement proper caching, and use non-root users.

**Q2: What are Docker security best practices?**
A: Use official images, run as non-root user, use specific tags, scan for vulnerabilities, limit container capabilities, use read-only filesystems, implement proper secrets management, and regular updates.

**Q3: How do you handle persistent data in Docker?**
A: Use Docker volumes for data persistence, bind mounts for development, named volumes for production, backup strategies, and proper volume management with lifecycle policies.

**Q4: What is the difference between COPY and ADD in Dockerfile?**
A: COPY simply copies files/directories from build context. ADD has additional features like extracting tar files and downloading from URLs, but COPY is preferred for simple file copying due to transparency.

**Q5: How do you troubleshoot Docker container issues?**
A: Check container logs (`docker logs`), inspect container (`docker inspect`), execute into container (`docker exec`), check resource usage, verify network connectivity, and analyze image layers.

### Advanced Level (6-15)

**Q6: How do you implement container orchestration without Kubernetes?**
A: Use Docker Swarm for clustering, Docker Compose for multi-container apps, implement service discovery, load balancing, health checks, rolling updates, and proper networking strategies.

**Q7: What are Docker networking modes and when to use each?**
A: Bridge (default, single host), Host (performance, direct host network), Overlay (multi-host), Macvlan (legacy apps needing MAC), None (maximum isolation). Choose based on requirements.

**Q8: How do you implement CI/CD with Docker?**
A: Build images in CI pipeline, use multi-stage builds, implement image scanning, tag strategies, push to registry, deploy with orchestration tools, and implement rollback strategies.

**Q9: What are Docker resource limits and how to set them?**
A: Use `--memory`, `--cpus`, `--pids-limit` flags or Docker Compose resource constraints. Monitor with `docker stats`, implement proper limits to prevent resource exhaustion and ensure fair sharing.

**Q10: How do you handle Docker secrets and sensitive data?**
A: Use Docker secrets (Swarm), external secret management (Vault), environment variables for non-sensitive config, mounted files, and avoid hardcoding secrets in images or logs.

## 🔑 Key Takeaways

- **Production Ready**: Implement security, optimization, and monitoring
- **Multi-Container**: Use Docker Compose for complex applications
- **Security First**: Follow security best practices from development to production
- **Optimization**: Focus on image size, build time, and runtime performance
- **Monitoring**: Implement proper logging, health checks, and resource monitoring
- **Best Practices**: Use official images, non-root users, and proper networking

## 🚀 Next Steps

- Day 27: Two-Tier Project using Docker Network
- Day 28: Docker Volumes and Docker Compose Deep Dive
- Container orchestration with Kubernetes
- Advanced Docker networking and service mesh

---

**Hands-on Completed:** ✅ Website Hosting, Multi-Container Apps, Security Implementation  
**Duration:** 4-5 hours  
**Difficulty:** Intermediate to Advanced