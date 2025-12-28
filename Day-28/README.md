# Day 28: Docker Volumes and Docker Compose Deep Dive

## 📚 Learning Objectives
- Master Docker volume types and management
- Implement data persistence strategies
- Advanced Docker Compose features and configurations
- Practice volume backup and recovery
- Understand Docker Compose networking and scaling

## 💾 Docker Volumes Deep Dive

### Volume Types Overview

#### 1. Named Volumes
```bash
# Create named volume
docker volume create mydata
docker volume create --driver local --opt type=none --opt device=/host/path --opt o=bind myvolume

# List volumes
docker volume ls

# Inspect volume
docker volume inspect mydata

# Use named volume
docker run -d -v mydata:/data nginx
```

#### 2. Bind Mounts
```bash
# Bind mount host directory
docker run -d -v /host/path:/container/path nginx
docker run -d --mount type=bind,source=/host/path,target=/container/path nginx

# Read-only bind mount
docker run -d -v /host/path:/container/path:ro nginx
```

#### 3. tmpfs Mounts
```bash
# Temporary filesystem in memory
docker run -d --tmpfs /tmp nginx
docker run -d --mount type=tmpfs,destination=/tmp nginx
```

## 🛠️ Hands-on Lab: Advanced Volume Management

### Lab 1: Database with Persistent Storage

#### 1.1 PostgreSQL with Volume Management
```bash
mkdir docker-volume-lab
cd docker-volume-lab

# Create volume for PostgreSQL data
docker volume create postgres_data
docker volume create postgres_backup

# Create PostgreSQL container with volume
docker run -d \
    --name postgres-db \
    -e POSTGRES_DB=testdb \
    -e POSTGRES_USER=dbuser \
    -e POSTGRES_PASSWORD=dbpass123 \
    -v postgres_data:/var/lib/postgresql/data \
    -v postgres_backup:/backup \
    -p 5432:5432 \
    postgres:13

# Create sample data
docker exec -it postgres-db psql -U dbuser -d testdb -c "
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO users (name, email) VALUES 
    ('John Doe', 'john@example.com'),
    ('Jane Smith', 'jane@example.com'),
    ('Bob Johnson', 'bob@example.com');
"

# Verify data
docker exec -it postgres-db psql -U dbuser -d testdb -c "SELECT * FROM users;"
```

#### 1.2 Volume Backup and Restore
```bash
# Create backup script
cat > backup-postgres.sh << 'EOF'
#!/bin/bash

CONTAINER_NAME="postgres-db"
BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="postgres_backup_${DATE}.sql"

echo "Creating backup: $BACKUP_FILE"

# Create database backup
docker exec $CONTAINER_NAME pg_dump -U dbuser testdb > "${BACKUP_DIR}/${BACKUP_FILE}"

# Compress backup
gzip "${BACKUP_DIR}/${BACKUP_FILE}"

echo "Backup completed: ${BACKUP_DIR}/${BACKUP_FILE}.gz"

# Keep only last 7 backups
find $BACKUP_DIR -name "postgres_backup_*.sql.gz" -mtime +7 -delete
EOF

chmod +x backup-postgres.sh

# Create restore script
cat > restore-postgres.sh << 'EOF'
#!/bin/bash

if [ $# -eq 0 ]; then
    echo "Usage: $0 <backup_file>"
    exit 1
fi

CONTAINER_NAME="postgres-db"
BACKUP_FILE=$1

echo "Restoring from backup: $BACKUP_FILE"

# Drop and recreate database
docker exec $CONTAINER_NAME psql -U dbuser -c "DROP DATABASE IF EXISTS testdb;"
docker exec $CONTAINER_NAME psql -U dbuser -c "CREATE DATABASE testdb;"

# Restore from backup
if [[ $BACKUP_FILE == *.gz ]]; then
    gunzip -c $BACKUP_FILE | docker exec -i $CONTAINER_NAME psql -U dbuser testdb
else
    docker exec -i $CONTAINER_NAME psql -U dbuser testdb < $BACKUP_FILE
fi

echo "Restore completed"
EOF

chmod +x restore-postgres.sh

# Test backup
./backup-postgres.sh
```

### Lab 2: Multi-Service Application with Volumes

#### 2.1 Complete Docker Compose with Volumes
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # Web Application
  webapp:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: webapp
    environment:
      - DATABASE_URL=postgresql://dbuser:dbpass123@postgres:5432/appdb
      - REDIS_URL=redis://redis:6379/0
      - UPLOAD_PATH=/app/uploads
    volumes:
      - app_uploads:/app/uploads
      - app_logs:/app/logs
      - ./app/config:/app/config:ro
    ports:
      - "3000:3000"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped

  # PostgreSQL Database
  postgres:
    image: postgres:13
    container_name: postgres
    environment:
      - POSTGRES_DB=appdb
      - POSTGRES_USER=dbuser
      - POSTGRES_PASSWORD=dbpass123
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - postgres_backup:/backup
      - ./database/init:/docker-entrypoint-initdb.d:ro
      - ./database/postgresql.conf:/etc/postgresql/postgresql.conf:ro
    ports:
      - "5432:5432"
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dbuser -d appdb"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Redis Cache
  redis:
    image: redis:7-alpine
    container_name: redis
    command: redis-server /usr/local/etc/redis/redis.conf
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    ports:
      - "6379:6379"
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 3

  # File Storage Service
  minio:
    image: minio/minio:latest
    container_name: minio
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=minioadmin
      - MINIO_ROOT_PASSWORD=minioadmin123
    volumes:
      - minio_data:/data
    ports:
      - "9000:9000"
      - "9001:9001"
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - nginx_logs:/var/log/nginx
      - static_files:/usr/share/nginx/html:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - webapp
    networks:
      - app-network
    restart: unless-stopped

  # Backup Service
  backup:
    build:
      context: ./backup
      dockerfile: Dockerfile
    container_name: backup-service
    environment:
      - POSTGRES_HOST=postgres
      - POSTGRES_DB=appdb
      - POSTGRES_USER=dbuser
      - POSTGRES_PASSWORD=dbpass123
      - BACKUP_SCHEDULE=0 2 * * *
    volumes:
      - postgres_backup:/backup/postgres
      - app_backup:/backup/app
      - minio_backup:/backup/minio
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - postgres
    networks:
      - app-network
    restart: unless-stopped

volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: ./data/postgres
  postgres_backup:
    driver: local
  redis_data:
    driver: local
  minio_data:
    driver: local
  app_uploads:
    driver: local
  app_logs:
    driver: local
  app_backup:
    driver: local
  minio_backup:
    driver: local
  nginx_logs:
    driver: local
  static_files:
    driver: local

networks:
  app-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
EOF
```

#### 2.2 Application with Volume Integration
```bash
mkdir -p app
cat > app/Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Create directories for volumes
RUN mkdir -p /app/uploads /app/logs /app/config

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001 -G nodejs && \
    chown -R nodeuser:nodejs /app

USER nodeuser

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

CMD ["npm", "start"]
EOF

cat > app/package.json << 'EOF'
{
  "name": "volume-demo-app",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "multer": "^1.4.5-lts.1",
    "pg": "^8.11.3",
    "redis": "^4.6.10",
    "winston": "^3.11.0"
  }
}
EOF

cat > app/server.js << 'EOF'
const express = require('express');
const multer = require('multer');
const { Pool } = require('pg');
const redis = require('redis');
const winston = require('winston');
const fs = require('fs').promises;
const path = require('path');

const app = express();
const PORT = process.env.PORT || 3000;

// Configure logging
const logger = winston.createLogger({
    level: 'info',
    format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
    ),
    transports: [
        new winston.transports.File({ filename: '/app/logs/error.log', level: 'error' }),
        new winston.transports.File({ filename: '/app/logs/combined.log' }),
        new winston.transports.Console()
    ]
});

// Database connection
const pool = new Pool({
    connectionString: process.env.DATABASE_URL || 'postgresql://dbuser:dbpass123@localhost:5432/appdb'
});

// Redis connection
const redisClient = redis.createClient({
    url: process.env.REDIS_URL || 'redis://localhost:6379'
});

redisClient.connect().catch(console.error);

// Configure multer for file uploads
const storage = multer.diskStorage({
    destination: (req, file, cb) => {
        cb(null, '/app/uploads/');
    },
    filename: (req, file, cb) => {
        const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1E9);
        cb(null, file.fieldname + '-' + uniqueSuffix + path.extname(file.originalname));
    }
});

const upload = multer({ 
    storage: storage,
    limits: { fileSize: 10 * 1024 * 1024 }, // 10MB limit
    fileFilter: (req, file, cb) => {
        const allowedTypes = /jpeg|jpg|png|gif|pdf|txt/;
        const extname = allowedTypes.test(path.extname(file.originalname).toLowerCase());
        const mimetype = allowedTypes.test(file.mimetype);
        
        if (mimetype && extname) {
            return cb(null, true);
        } else {
            cb(new Error('Invalid file type'));
        }
    }
});

app.use(express.json());
app.use(express.static('/app/uploads'));

// Health check endpoint
app.get('/health', async (req, res) => {
    try {
        // Check database
        await pool.query('SELECT 1');
        
        // Check Redis
        await redisClient.ping();
        
        // Check upload directory
        await fs.access('/app/uploads');
        
        res.json({
            status: 'healthy',
            timestamp: new Date().toISOString(),
            services: {
                database: 'connected',
                redis: 'connected',
                uploads: 'accessible'
            }
        });
    } catch (error) {
        logger.error('Health check failed', { error: error.message });
        res.status(503).json({
            status: 'unhealthy',
            error: error.message
        });
    }
});

// File upload endpoint
app.post('/upload', upload.single('file'), async (req, res) => {
    try {
        if (!req.file) {
            return res.status(400).json({ error: 'No file uploaded' });
        }

        // Store file info in database
        const result = await pool.query(
            'INSERT INTO files (filename, original_name, size, mimetype, upload_date) VALUES ($1, $2, $3, $4, $5) RETURNING id',
            [req.file.filename, req.file.originalname, req.file.size, req.file.mimetype, new Date()]
        );

        // Cache file info in Redis
        await redisClient.setEx(`file:${result.rows[0].id}`, 3600, JSON.stringify({
            id: result.rows[0].id,
            filename: req.file.filename,
            originalname: req.file.originalname,
            size: req.file.size
        }));

        logger.info('File uploaded successfully', {
            fileId: result.rows[0].id,
            filename: req.file.filename,
            size: req.file.size
        });

        res.json({
            message: 'File uploaded successfully',
            fileId: result.rows[0].id,
            filename: req.file.filename,
            url: `/uploads/${req.file.filename}`
        });
    } catch (error) {
        logger.error('File upload failed', { error: error.message });
        res.status(500).json({ error: 'Upload failed' });
    }
});

// Get file info
app.get('/files/:id', async (req, res) => {
    try {
        const fileId = req.params.id;
        
        // Try Redis cache first
        const cached = await redisClient.get(`file:${fileId}`);
        if (cached) {
            return res.json(JSON.parse(cached));
        }

        // Fallback to database
        const result = await pool.query('SELECT * FROM files WHERE id = $1', [fileId]);
        if (result.rows.length === 0) {
            return res.status(404).json({ error: 'File not found' });
        }

        const fileInfo = result.rows[0];
        
        // Cache for future requests
        await redisClient.setEx(`file:${fileId}`, 3600, JSON.stringify(fileInfo));

        res.json(fileInfo);
    } catch (error) {
        logger.error('Error fetching file info', { error: error.message, fileId: req.params.id });
        res.status(500).json({ error: 'Internal server error' });
    }
});

// List all files
app.get('/files', async (req, res) => {
    try {
        const result = await pool.query('SELECT * FROM files ORDER BY upload_date DESC LIMIT 50');
        res.json({ files: result.rows });
    } catch (error) {
        logger.error('Error listing files', { error: error.message });
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Volume statistics
app.get('/stats/volumes', async (req, res) => {
    try {
        const stats = {};
        
        // Upload directory stats
        const uploadFiles = await fs.readdir('/app/uploads');
        stats.uploads = {
            fileCount: uploadFiles.length,
            path: '/app/uploads'
        };

        // Log directory stats
        const logFiles = await fs.readdir('/app/logs');
        stats.logs = {
            fileCount: logFiles.length,
            path: '/app/logs'
        };

        // Database stats
        const dbResult = await pool.query('SELECT COUNT(*) as file_count FROM files');
        stats.database = {
            fileRecords: parseInt(dbResult.rows[0].file_count)
        };

        res.json({ stats });
    } catch (error) {
        logger.error('Error getting volume stats', { error: error.message });
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Initialize database tables
async function initializeDatabase() {
    try {
        await pool.query(`
            CREATE TABLE IF NOT EXISTS files (
                id SERIAL PRIMARY KEY,
                filename VARCHAR(255) NOT NULL,
                original_name VARCHAR(255) NOT NULL,
                size INTEGER NOT NULL,
                mimetype VARCHAR(100) NOT NULL,
                upload_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            )
        `);
        
        logger.info('Database initialized successfully');
    } catch (error) {
        logger.error('Database initialization failed', { error: error.message });
        throw error;
    }
}

// Start server
async function startServer() {
    try {
        await initializeDatabase();
        
        app.listen(PORT, '0.0.0.0', () => {
            logger.info(`Server running on port ${PORT}`);
        });
    } catch (error) {
        logger.error('Failed to start server', { error: error.message });
        process.exit(1);
    }
}

startServer();
EOF
```

#### 2.3 Backup Service Implementation
```bash
mkdir -p backup
cat > backup/Dockerfile << 'EOF'
FROM alpine:latest

# Install required packages
RUN apk add --no-cache \
    postgresql-client \
    redis \
    curl \
    docker-cli \
    bash \
    gzip \
    tar

# Copy backup scripts
COPY backup-script.sh /usr/local/bin/
COPY restore-script.sh /usr/local/bin/
RUN chmod +x /usr/local/bin/*.sh

# Install cron
RUN apk add --no-cache dcron

# Create crontab
RUN echo "${BACKUP_SCHEDULE:-0 2 * * *} /usr/local/bin/backup-script.sh" > /etc/crontabs/root

CMD ["crond", "-f", "-d", "8"]
EOF

cat > backup/backup-script.sh << 'EOF'
#!/bin/bash

set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup"

echo "Starting backup process at $(date)"

# PostgreSQL backup
echo "Backing up PostgreSQL..."
PGPASSWORD=$POSTGRES_PASSWORD pg_dump \
    -h $POSTGRES_HOST \
    -U $POSTGRES_USER \
    -d $POSTGRES_DB \
    --no-owner --no-privileges \
    > "${BACKUP_DIR}/postgres/postgres_${TIMESTAMP}.sql"

gzip "${BACKUP_DIR}/postgres/postgres_${TIMESTAMP}.sql"

# Application data backup
echo "Backing up application data..."
tar -czf "${BACKUP_DIR}/app/app_uploads_${TIMESTAMP}.tar.gz" -C /app uploads/ || true
tar -czf "${BACKUP_DIR}/app/app_logs_${TIMESTAMP}.tar.gz" -C /app logs/ || true

# MinIO backup (if accessible)
echo "Backing up MinIO data..."
tar -czf "${BACKUP_DIR}/minio/minio_data_${TIMESTAMP}.tar.gz" -C /data . || true

# Cleanup old backups (keep last 7 days)
find ${BACKUP_DIR}/postgres -name "postgres_*.sql.gz" -mtime +7 -delete || true
find ${BACKUP_DIR}/app -name "app_*.tar.gz" -mtime +7 -delete || true
find ${BACKUP_DIR}/minio -name "minio_*.tar.gz" -mtime +7 -delete || true

echo "Backup completed at $(date)"
EOF

cat > backup/restore-script.sh << 'EOF'
#!/bin/bash

if [ $# -lt 2 ]; then
    echo "Usage: $0 <backup_type> <backup_file>"
    echo "Backup types: postgres, app, minio"
    exit 1
fi

BACKUP_TYPE=$1
BACKUP_FILE=$2

case $BACKUP_TYPE in
    postgres)
        echo "Restoring PostgreSQL backup: $BACKUP_FILE"
        if [[ $BACKUP_FILE == *.gz ]]; then
            gunzip -c $BACKUP_FILE | PGPASSWORD=$POSTGRES_PASSWORD psql -h $POSTGRES_HOST -U $POSTGRES_USER -d $POSTGRES_DB
        else
            PGPASSWORD=$POSTGRES_PASSWORD psql -h $POSTGRES_HOST -U $POSTGRES_USER -d $POSTGRES_DB < $BACKUP_FILE
        fi
        ;;
    app)
        echo "Restoring application backup: $BACKUP_FILE"
        tar -xzf $BACKUP_FILE -C /
        ;;
    minio)
        echo "Restoring MinIO backup: $BACKUP_FILE"
        tar -xzf $BACKUP_FILE -C /data/
        ;;
    *)
        echo "Unknown backup type: $BACKUP_TYPE"
        exit 1
        ;;
esac

echo "Restore completed"
EOF
```

### Lab 3: Volume Performance and Monitoring

#### 3.1 Volume Performance Testing
```bash
cat > test-volume-performance.sh << 'EOF'
#!/bin/bash

echo "🔍 Testing Docker Volume Performance"
echo "=================================="

# Create test volumes
docker volume create test-volume-local
docker volume create test-volume-bind

# Test write performance - Named Volume
echo "Testing named volume write performance..."
docker run --rm -v test-volume-local:/data alpine sh -c "
    time dd if=/dev/zero of=/data/testfile bs=1M count=100
"

# Test write performance - Bind Mount
mkdir -p /tmp/docker-bind-test
echo "Testing bind mount write performance..."
docker run --rm -v /tmp/docker-bind-test:/data alpine sh -c "
    time dd if=/dev/zero of=/data/testfile bs=1M count=100
"

# Test read performance - Named Volume
echo "Testing named volume read performance..."
docker run --rm -v test-volume-local:/data alpine sh -c "
    time dd if=/data/testfile of=/dev/null bs=1M
"

# Test read performance - Bind Mount
echo "Testing bind mount read performance..."
docker run --rm -v /tmp/docker-bind-test:/data alpine sh -c "
    time dd if=/data/testfile of=/dev/null bs=1M
"

# Cleanup
docker volume rm test-volume-local test-volume-bind
rm -rf /tmp/docker-bind-test

echo "Performance testing completed"
EOF

chmod +x test-volume-performance.sh
./test-volume-performance.sh
```

#### 3.2 Volume Monitoring Script
```bash
cat > monitor-volumes.sh << 'EOF'
#!/bin/bash

echo "📊 Docker Volume Monitoring Report"
echo "================================="
echo "Generated at: $(date)"
echo

# List all volumes
echo "📁 All Docker Volumes:"
docker volume ls --format "table {{.Driver}}\t{{.Name}}\t{{.Scope}}"
echo

# Volume usage statistics
echo "💾 Volume Disk Usage:"
for volume in $(docker volume ls -q); do
    size=$(docker run --rm -v $volume:/data alpine du -sh /data 2>/dev/null | cut -f1)
    echo "  $volume: $size"
done
echo

# Container volume mounts
echo "🔗 Container Volume Mounts:"
docker ps --format "table {{.Names}}\t{{.Mounts}}" | head -20
echo

# Orphaned volumes
echo "🗑️  Orphaned Volumes:"
docker volume ls -f dangling=true --format "table {{.Name}}\t{{.Driver}}"
echo

# Volume inspect details for named volumes
echo "🔍 Volume Details:"
for volume in $(docker volume ls --format "{{.Name}}" | head -5); do
    echo "Volume: $volume"
    docker volume inspect $volume --format "  Mountpoint: {{.Mountpoint}}"
    docker volume inspect $volume --format "  Driver: {{.Driver}}"
    echo
done
EOF

chmod +x monitor-volumes.sh
./monitor-volumes.sh
```

## 🔧 Advanced Docker Compose Features

### Multi-Environment Configuration
```bash
# Base configuration
cat > docker-compose.base.yml << 'EOF'
version: '3.8'

services:
  webapp:
    build: ./app
    environment:
      - NODE_ENV=${NODE_ENV:-development}
    volumes:
      - app_logs:/app/logs
    networks:
      - app-network

  postgres:
    image: postgres:13
    environment:
      - POSTGRES_DB=${POSTGRES_DB:-appdb}
      - POSTGRES_USER=${POSTGRES_USER:-dbuser}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD:-dbpass123}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  postgres_data:
  app_logs:

networks:
  app-network:
EOF

# Development override
cat > docker-compose.dev.yml << 'EOF'
version: '3.8'

services:
  webapp:
    volumes:
      - ./app:/app
      - /app/node_modules
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    command: npm run dev

  postgres:
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_DB=appdb_dev
EOF

# Production override
cat > docker-compose.prod.yml << 'EOF'
version: '3.8'

services:
  webapp:
    restart: unless-stopped
    environment:
      - NODE_ENV=production
    deploy:
      replicas: 3
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

  postgres:
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 1G
        reservations:
          memory: 512M

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/prod.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - webapp
    networks:
      - app-network
    restart: unless-stopped
EOF

# Usage examples
echo "# Development environment"
echo "docker-compose -f docker-compose.base.yml -f docker-compose.dev.yml up"
echo
echo "# Production environment"
echo "docker-compose -f docker-compose.base.yml -f docker-compose.prod.yml up -d"
```

## 📚 Interview Questions & Answers

### Intermediate Level (1-10)

**Q1: What are the different types of Docker volumes and when to use each?**
A: Named volumes (managed by Docker, portable), bind mounts (direct host access, development), tmpfs mounts (temporary, in-memory). Use named volumes for production data, bind mounts for development, tmpfs for temporary data.

**Q2: How do you backup and restore Docker volumes?**
A: Use `docker run` with volume mounts to create backups: `docker run --rm -v myvolume:/data -v $(pwd):/backup alpine tar czf /backup/backup.tar.gz /data`. Restore by reversing the process.

**Q3: What is the difference between COPY and volumes in Docker?**
A: COPY adds files to image layers (immutable), volumes provide runtime data persistence (mutable). COPY is for application code, volumes for data that changes during runtime.

**Q4: How do you handle volume permissions in Docker?**
A: Set proper user/group ownership, use init containers to set permissions, match host and container user IDs, or use Docker's user namespace mapping.

**Q5: What are Docker Compose override files?**
A: Override files (docker-compose.override.yml) extend base configurations for different environments. Use multiple compose files with `-f` flag for environment-specific configurations.

### Advanced Level (6-15)

**Q6: How do you implement volume encryption in Docker?**
A: Use encrypted filesystems (LUKS), encrypted storage drivers, cloud provider encrypted volumes, or application-level encryption. Consider performance impact and key management.

**Q7: What are the performance implications of different volume types?**
A: Named volumes: good performance, managed by Docker. Bind mounts: direct filesystem access, potential performance issues on Windows/Mac. tmpfs: fastest, but temporary.

**Q8: How do you implement volume backup strategies for production?**
A: Automated scheduled backups, incremental backups, cross-region replication, backup verification, retention policies, and disaster recovery testing.

**Q9: How do you handle volume migrations between Docker hosts?**
A: Use volume backup/restore, shared storage systems (NFS, cloud storage), Docker volume plugins, or container orchestration tools like Kubernetes with persistent volumes.

**Q10: What are Docker volume plugins and when to use them?**
A: Volume plugins extend Docker's storage capabilities with external storage systems (AWS EBS, Azure Disk, NFS). Use for cloud integration, shared storage, or specialized storage requirements.

## 🔑 Key Takeaways

- **Volume Types**: Choose appropriate volume type based on use case
- **Data Persistence**: Implement proper backup and recovery strategies
- **Performance**: Understand performance characteristics of different volume types
- **Security**: Consider encryption and access control for sensitive data
- **Monitoring**: Implement volume monitoring and alerting
- **Compose**: Use multi-file compose configurations for different environments

## 🚀 Next Steps

- Day 29: CI/CD with Jenkins
- Day 30: Advanced Jenkins Pipelines
- Container orchestration with Kubernetes
- Advanced storage solutions and CSI drivers

---

**Hands-on Completed:** ✅ Volume Management, Backup/Restore, Advanced Compose Features  
**Duration:** 4-5 hours  
**Difficulty:** Intermediate to Advanced