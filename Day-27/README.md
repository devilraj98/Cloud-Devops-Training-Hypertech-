# Day 27: Two-Tier Project using Docker Network (MySQL and Node.js)

## 📚 Learning Objectives
- Build a complete two-tier application with Docker
- Implement Docker networking for service communication
- Practice database integration with containerized applications
- Understand Docker network security and isolation
- Master multi-container application deployment

## 🏗️ Project Architecture

```
Frontend (Node.js/Express)
         ↓
Docker Network (app-network)
         ↓
Backend Database (MySQL)
```

## 🛠️ Two-Tier Application Implementation

### Step 1: Project Setup

#### 1.1 Create Project Structure
```bash
mkdir docker-two-tier-app
cd docker-two-tier-app

# Create directory structure
mkdir -p {frontend/{src,public,views},database/{init,data},docker,docs}

# Create main files
touch {frontend/package.json,frontend/app.js,database/init/init.sql,docker-compose.yml,.env,.gitignore}
```

#### 1.2 Environment Configuration
```bash
cat > .env << 'EOF'
# Database Configuration
MYSQL_ROOT_PASSWORD=rootpassword123
MYSQL_DATABASE=taskmanager
MYSQL_USER=appuser
MYSQL_PASSWORD=apppassword123

# Application Configuration
NODE_ENV=production
APP_PORT=3000
DB_HOST=mysql-db
DB_PORT=3306
DB_NAME=taskmanager
DB_USER=appuser
DB_PASSWORD=apppassword123

# Network Configuration
NETWORK_NAME=app-network
EOF

cat > .gitignore << 'EOF'
node_modules/
npm-debug.log*
.env.local
.env.production
database/data/*
!database/data/.gitkeep
logs/
*.log
.DS_Store
Thumbs.db
EOF
```

### Step 2: Database Layer (MySQL)

#### 2.1 Database Initialization Script
```bash
cat > database/init/init.sql << 'EOF'
-- Create database if not exists
CREATE DATABASE IF NOT EXISTS taskmanager;
USE taskmanager;

-- Create users table
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- Create tasks table
CREATE TABLE IF NOT EXISTS tasks (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT NOT NULL,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    status ENUM('pending', 'in_progress', 'completed') DEFAULT 'pending',
    priority ENUM('low', 'medium', 'high') DEFAULT 'medium',
    due_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

-- Create categories table
CREATE TABLE IF NOT EXISTS categories (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    color VARCHAR(7) DEFAULT '#007bff',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create task_categories junction table
CREATE TABLE IF NOT EXISTS task_categories (
    task_id INT,
    category_id INT,
    PRIMARY KEY (task_id, category_id),
    FOREIGN KEY (task_id) REFERENCES tasks(id) ON DELETE CASCADE,
    FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE CASCADE
);

-- Insert sample data
INSERT INTO users (username, email, password_hash) VALUES
('admin', 'admin@example.com', '$2b$10$rQZ9QmSTnIc8nFWM8zitO.Qo.Z9QmSTnIc8nFWM8zitO.Qo.Z9QmST'),
('john_doe', 'john@example.com', '$2b$10$rQZ9QmSTnIc8nFWM8zitO.Qo.Z9QmSTnIc8nFWM8zitO.Qo.Z9QmST'),
('jane_smith', 'jane@example.com', '$2b$10$rQZ9QmSTnIc8nFWM8zitO.Qo.Z9QmSTnIc8nFWM8zitO.Qo.Z9QmST');

INSERT INTO categories (name, color) VALUES
('Work', '#007bff'),
('Personal', '#28a745'),
('Urgent', '#dc3545'),
('Learning', '#ffc107');

INSERT INTO tasks (user_id, title, description, status, priority, due_date) VALUES
(1, 'Setup Docker Environment', 'Configure Docker for development and production', 'completed', 'high', '2024-01-15'),
(1, 'Learn Docker Networking', 'Understand Docker network types and communication', 'in_progress', 'medium', '2024-01-20'),
(2, 'Build Two-Tier Application', 'Create Node.js app with MySQL database', 'pending', 'high', '2024-01-25'),
(2, 'Implement User Authentication', 'Add login and registration functionality', 'pending', 'medium', '2024-01-30'),
(3, 'Write API Documentation', 'Document all REST API endpoints', 'pending', 'low', '2024-02-05');

-- Insert task-category relationships
INSERT INTO task_categories (task_id, category_id) VALUES
(1, 1), (1, 4),  -- Work, Learning
(2, 4),          -- Learning
(3, 1),          -- Work
(4, 1), (4, 3),  -- Work, Urgent
(5, 1);          -- Work

-- Create indexes for better performance
CREATE INDEX idx_tasks_user_id ON tasks(user_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_priority ON tasks(priority);
CREATE INDEX idx_tasks_due_date ON tasks(due_date);

-- Create a view for task summary
CREATE VIEW task_summary AS
SELECT 
    u.username,
    COUNT(t.id) as total_tasks,
    SUM(CASE WHEN t.status = 'completed' THEN 1 ELSE 0 END) as completed_tasks,
    SUM(CASE WHEN t.status = 'pending' THEN 1 ELSE 0 END) as pending_tasks,
    SUM(CASE WHEN t.status = 'in_progress' THEN 1 ELSE 0 END) as in_progress_tasks
FROM users u
LEFT JOIN tasks t ON u.id = t.user_id
GROUP BY u.id, u.username;

COMMIT;
EOF

# Create data directory placeholder
touch database/data/.gitkeep
```

#### 2.2 MySQL Dockerfile
```bash
cat > database/Dockerfile << 'EOF'
FROM mysql:8.0

# Set environment variables
ENV MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
ENV MYSQL_DATABASE=${MYSQL_DATABASE}
ENV MYSQL_USER=${MYSQL_USER}
ENV MYSQL_PASSWORD=${MYSQL_PASSWORD}

# Copy initialization scripts
COPY init/ /docker-entrypoint-initdb.d/

# Copy custom MySQL configuration
COPY my.cnf /etc/mysql/conf.d/

# Create mysql user and set permissions
RUN chown -R mysql:mysql /var/lib/mysql

# Expose MySQL port
EXPOSE 3306

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD mysqladmin ping -h localhost -u root -p${MYSQL_ROOT_PASSWORD} || exit 1
EOF

cat > database/my.cnf << 'EOF'
[mysqld]
# Basic settings
default-storage-engine=innodb
sql_mode=STRICT_TRANS_TABLES,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION

# Connection settings
max_connections=200
connect_timeout=60
wait_timeout=120

# Buffer settings
innodb_buffer_pool_size=256M
innodb_log_file_size=64M
innodb_flush_log_at_trx_commit=1
innodb_lock_wait_timeout=50

# Query cache
query_cache_type=1
query_cache_size=32M
query_cache_limit=2M

# Logging
general_log=1
general_log_file=/var/lib/mysql/general.log
slow_query_log=1
slow_query_log_file=/var/lib/mysql/slow.log
long_query_time=2

# Character set
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

[mysql]
default-character-set=utf8mb4

[client]
default-character-set=utf8mb4
EOF
```

### Step 3: Frontend Application Layer (Node.js)

#### 3.1 Package Configuration
```bash
cat > frontend/package.json << 'EOF'
{
  "name": "docker-two-tier-frontend",
  "version": "1.0.0",
  "description": "Node.js frontend for two-tier Docker application",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "dev": "nodemon app.js",
    "test": "jest",
    "lint": "eslint .",
    "format": "prettier --write ."
  },
  "dependencies": {
    "express": "^4.18.2",
    "mysql2": "^3.6.5",
    "bcrypt": "^5.1.1",
    "jsonwebtoken": "^9.0.2",
    "cors": "^2.8.5",
    "helmet": "^7.1.0",
    "morgan": "^1.10.0",
    "express-rate-limit": "^7.1.5",
    "joi": "^17.11.0",
    "dotenv": "^16.3.1",
    "ejs": "^3.1.9"
  },
  "devDependencies": {
    "nodemon": "^3.0.2",
    "jest": "^29.7.0",
    "eslint": "^8.55.0",
    "prettier": "^3.1.1"
  },
  "keywords": ["docker", "nodejs", "mysql", "two-tier"],
  "author": "DevOps Team",
  "license": "MIT"
}
EOF
```

#### 3.2 Main Application File
```bash
cat > frontend/app.js << 'EOF'
const express = require('express');
const mysql = require('mysql2/promise');
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const rateLimit = require('express-rate-limit');
const Joi = require('joi');
require('dotenv').config();

const app = express();
const PORT = process.env.APP_PORT || 3000;

// Database configuration
const dbConfig = {
    host: process.env.DB_HOST || 'mysql-db',
    port: process.env.DB_PORT || 3306,
    user: process.env.DB_USER || 'appuser',
    password: process.env.DB_PASSWORD || 'apppassword123',
    database: process.env.DB_NAME || 'taskmanager',
    waitForConnections: true,
    connectionLimit: 10,
    queueLimit: 0,
    acquireTimeout: 60000,
    timeout: 60000
};

// Create database connection pool
let pool;

async function initializeDatabase() {
    try {
        pool = mysql.createPool(dbConfig);
        
        // Test connection
        const connection = await pool.getConnection();
        console.log('✅ Database connected successfully');
        connection.release();
        
        return true;
    } catch (error) {
        console.error('❌ Database connection failed:', error.message);
        return false;
    }
}

// Middleware
app.use(helmet());
app.use(cors());
app.use(morgan('combined'));
app.use(express.json({ limit: '10mb' }));
app.use(express.urlencoded({ extended: true }));

// Rate limiting
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
    message: 'Too many requests from this IP, please try again later.'
});
app.use(limiter);

// Set view engine
app.set('view engine', 'ejs');
app.set('views', './views');

// Serve static files
app.use(express.static('public'));

// Validation schemas
const userSchema = Joi.object({
    username: Joi.string().alphanum().min(3).max(30).required(),
    email: Joi.string().email().required(),
    password: Joi.string().min(6).required()
});

const taskSchema = Joi.object({
    title: Joi.string().min(1).max(200).required(),
    description: Joi.string().max(1000).optional(),
    status: Joi.string().valid('pending', 'in_progress', 'completed').optional(),
    priority: Joi.string().valid('low', 'medium', 'high').optional(),
    due_date: Joi.date().optional()
});

// Authentication middleware
const authenticateToken = (req, res, next) => {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];

    if (!token) {
        return res.status(401).json({ error: 'Access token required' });
    }

    jwt.verify(token, process.env.JWT_SECRET || 'fallback-secret', (err, user) => {
        if (err) {
            return res.status(403).json({ error: 'Invalid or expired token' });
        }
        req.user = user;
        next();
    });
};

// Routes

// Home page
app.get('/', async (req, res) => {
    try {
        const [taskStats] = await pool.execute(`
            SELECT 
                COUNT(*) as total_tasks,
                SUM(CASE WHEN status = 'completed' THEN 1 ELSE 0 END) as completed_tasks,
                SUM(CASE WHEN status = 'pending' THEN 1 ELSE 0 END) as pending_tasks,
                SUM(CASE WHEN status = 'in_progress' THEN 1 ELSE 0 END) as in_progress_tasks
            FROM tasks
        `);

        const [userCount] = await pool.execute('SELECT COUNT(*) as user_count FROM users');

        res.render('index', {
            title: 'Docker Two-Tier Application',
            stats: taskStats[0],
            userCount: userCount[0].user_count
        });
    } catch (error) {
        console.error('Error fetching stats:', error);
        res.render('index', {
            title: 'Docker Two-Tier Application',
            stats: { total_tasks: 0, completed_tasks: 0, pending_tasks: 0, in_progress_tasks: 0 },
            userCount: 0
        });
    }
});

// Health check endpoint
app.get('/health', async (req, res) => {
    try {
        // Check database connection
        await pool.execute('SELECT 1');
        
        res.json({
            status: 'healthy',
            timestamp: new Date().toISOString(),
            database: 'connected',
            uptime: process.uptime(),
            memory: process.memoryUsage(),
            version: '1.0.0'
        });
    } catch (error) {
        res.status(503).json({
            status: 'unhealthy',
            timestamp: new Date().toISOString(),
            database: 'disconnected',
            error: error.message
        });
    }
});

// API Routes

// User registration
app.post('/api/auth/register', async (req, res) => {
    try {
        const { error, value } = userSchema.validate(req.body);
        if (error) {
            return res.status(400).json({ error: error.details[0].message });
        }

        const { username, email, password } = value;

        // Check if user exists
        const [existingUsers] = await pool.execute(
            'SELECT id FROM users WHERE username = ? OR email = ?',
            [username, email]
        );

        if (existingUsers.length > 0) {
            return res.status(409).json({ error: 'Username or email already exists' });
        }

        // Hash password
        const saltRounds = 10;
        const passwordHash = await bcrypt.hash(password, saltRounds);

        // Insert user
        const [result] = await pool.execute(
            'INSERT INTO users (username, email, password_hash) VALUES (?, ?, ?)',
            [username, email, passwordHash]
        );

        res.status(201).json({
            message: 'User registered successfully',
            userId: result.insertId
        });
    } catch (error) {
        console.error('Registration error:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// User login
app.post('/api/auth/login', async (req, res) => {
    try {
        const { username, password } = req.body;

        if (!username || !password) {
            return res.status(400).json({ error: 'Username and password required' });
        }

        // Find user
        const [users] = await pool.execute(
            'SELECT id, username, email, password_hash FROM users WHERE username = ?',
            [username]
        );

        if (users.length === 0) {
            return res.status(401).json({ error: 'Invalid credentials' });
        }

        const user = users[0];

        // Verify password
        const isValidPassword = await bcrypt.compare(password, user.password_hash);
        if (!isValidPassword) {
            return res.status(401).json({ error: 'Invalid credentials' });
        }

        // Generate JWT token
        const token = jwt.sign(
            { userId: user.id, username: user.username },
            process.env.JWT_SECRET || 'fallback-secret',
            { expiresIn: '24h' }
        );

        res.json({
            message: 'Login successful',
            token,
            user: {
                id: user.id,
                username: user.username,
                email: user.email
            }
        });
    } catch (error) {
        console.error('Login error:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Get all tasks for authenticated user
app.get('/api/tasks', authenticateToken, async (req, res) => {
    try {
        const [tasks] = await pool.execute(`
            SELECT t.*, GROUP_CONCAT(c.name) as categories
            FROM tasks t
            LEFT JOIN task_categories tc ON t.id = tc.task_id
            LEFT JOIN categories c ON tc.category_id = c.id
            WHERE t.user_id = ?
            GROUP BY t.id
            ORDER BY t.created_at DESC
        `, [req.user.userId]);

        res.json({ tasks });
    } catch (error) {
        console.error('Error fetching tasks:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Create new task
app.post('/api/tasks', authenticateToken, async (req, res) => {
    try {
        const { error, value } = taskSchema.validate(req.body);
        if (error) {
            return res.status(400).json({ error: error.details[0].message });
        }

        const { title, description, status, priority, due_date } = value;

        const [result] = await pool.execute(
            'INSERT INTO tasks (user_id, title, description, status, priority, due_date) VALUES (?, ?, ?, ?, ?, ?)',
            [req.user.userId, title, description || null, status || 'pending', priority || 'medium', due_date || null]
        );

        res.status(201).json({
            message: 'Task created successfully',
            taskId: result.insertId
        });
    } catch (error) {
        console.error('Error creating task:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Update task
app.put('/api/tasks/:id', authenticateToken, async (req, res) => {
    try {
        const taskId = parseInt(req.params.id);
        const { title, description, status, priority, due_date } = req.body;

        // Verify task belongs to user
        const [tasks] = await pool.execute(
            'SELECT id FROM tasks WHERE id = ? AND user_id = ?',
            [taskId, req.user.userId]
        );

        if (tasks.length === 0) {
            return res.status(404).json({ error: 'Task not found' });
        }

        // Update task
        await pool.execute(
            'UPDATE tasks SET title = ?, description = ?, status = ?, priority = ?, due_date = ? WHERE id = ?',
            [title, description, status, priority, due_date, taskId]
        );

        res.json({ message: 'Task updated successfully' });
    } catch (error) {
        console.error('Error updating task:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Delete task
app.delete('/api/tasks/:id', authenticateToken, async (req, res) => {
    try {
        const taskId = parseInt(req.params.id);

        // Verify task belongs to user and delete
        const [result] = await pool.execute(
            'DELETE FROM tasks WHERE id = ? AND user_id = ?',
            [taskId, req.user.userId]
        );

        if (result.affectedRows === 0) {
            return res.status(404).json({ error: 'Task not found' });
        }

        res.json({ message: 'Task deleted successfully' });
    } catch (error) {
        console.error('Error deleting task:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Get database statistics
app.get('/api/stats', async (req, res) => {
    try {
        const [stats] = await pool.execute('SELECT * FROM task_summary');
        res.json({ stats });
    } catch (error) {
        console.error('Error fetching stats:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// Error handling middleware
app.use((err, req, res, next) => {
    console.error(err.stack);
    res.status(500).json({ error: 'Something went wrong!' });
});

// 404 handler
app.use((req, res) => {
    res.status(404).json({ error: 'Route not found' });
});

// Start server
async function startServer() {
    const dbConnected = await initializeDatabase();
    
    if (!dbConnected) {
        console.error('Failed to connect to database. Exiting...');
        process.exit(1);
    }

    app.listen(PORT, '0.0.0.0', () => {
        console.log(`🚀 Server running on port ${PORT}`);
        console.log(`📊 Health check: http://localhost:${PORT}/health`);
        console.log(`🌐 Application: http://localhost:${PORT}`);
    });
}

// Graceful shutdown
process.on('SIGTERM', async () => {
    console.log('SIGTERM received, shutting down gracefully');
    if (pool) {
        await pool.end();
    }
    process.exit(0);
});

process.on('SIGINT', async () => {
    console.log('SIGINT received, shutting down gracefully');
    if (pool) {
        await pool.end();
    }
    process.exit(0);
});

startServer();
EOF
```

#### 3.3 Frontend Views
```bash
mkdir -p frontend/views
cat > frontend/views/index.ejs << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title><%= title %></title>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }
        
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            padding: 20px;
        }
        
        .container {
            max-width: 1200px;
            margin: 0 auto;
            background: white;
            border-radius: 15px;
            box-shadow: 0 20px 40px rgba(0,0,0,0.1);
            overflow: hidden;
        }
        
        .header {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            padding: 40px;
            text-align: center;
        }
        
        .header h1 {
            font-size: 2.5rem;
            margin-bottom: 10px;
        }
        
        .header p {
            font-size: 1.1rem;
            opacity: 0.9;
        }
        
        .stats {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 20px;
            padding: 40px;
            background: #f8f9fa;
        }
        
        .stat-card {
            background: white;
            padding: 30px;
            border-radius: 10px;
            text-align: center;
            box-shadow: 0 4px 6px rgba(0,0,0,0.1);
            transition: transform 0.3s;
        }
        
        .stat-card:hover {
            transform: translateY(-5px);
        }
        
        .stat-number {
            font-size: 2.5rem;
            font-weight: bold;
            color: #667eea;
            margin-bottom: 10px;
        }
        
        .stat-label {
            color: #666;
            font-size: 1rem;
        }
        
        .content {
            padding: 40px;
        }
        
        .features {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
            gap: 30px;
            margin-top: 30px;
        }
        
        .feature {
            padding: 30px;
            border: 2px solid #e9ecef;
            border-radius: 10px;
            text-align: center;
        }
        
        .feature h3 {
            color: #667eea;
            margin-bottom: 15px;
        }
        
        .api-section {
            background: #f8f9fa;
            padding: 40px;
            margin-top: 40px;
            border-radius: 10px;
        }
        
        .api-endpoint {
            background: white;
            padding: 15px;
            margin: 10px 0;
            border-radius: 5px;
            border-left: 4px solid #667eea;
            font-family: monospace;
        }
        
        .method {
            display: inline-block;
            padding: 4px 8px;
            border-radius: 4px;
            color: white;
            font-size: 0.8rem;
            margin-right: 10px;
        }
        
        .get { background: #28a745; }
        .post { background: #007bff; }
        .put { background: #ffc107; color: #333; }
        .delete { background: #dc3545; }
        
        footer {
            background: #333;
            color: white;
            text-align: center;
            padding: 30px;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🐳 <%= title %></h1>
            <p>Node.js + MySQL Two-Tier Architecture with Docker Networking</p>
        </div>
        
        <div class="stats">
            <div class="stat-card">
                <div class="stat-number"><%= stats.total_tasks %></div>
                <div class="stat-label">Total Tasks</div>
            </div>
            <div class="stat-card">
                <div class="stat-number"><%= stats.completed_tasks %></div>
                <div class="stat-label">Completed</div>
            </div>
            <div class="stat-card">
                <div class="stat-number"><%= stats.pending_tasks %></div>
                <div class="stat-label">Pending</div>
            </div>
            <div class="stat-card">
                <div class="stat-number"><%= userCount %></div>
                <div class="stat-label">Users</div>
            </div>
        </div>
        
        <div class="content">
            <h2>Application Features</h2>
            <div class="features">
                <div class="feature">
                    <h3>🔐 User Authentication</h3>
                    <p>Secure user registration and login with JWT tokens</p>
                </div>
                <div class="feature">
                    <h3>📝 Task Management</h3>
                    <p>Create, update, delete, and organize tasks efficiently</p>
                </div>
                <div class="feature">
                    <h3>🗄️ Database Integration</h3>
                    <p>MySQL database with proper relationships and indexing</p>
                </div>
                <div class="feature">
                    <h3>🌐 Docker Networking</h3>
                    <p>Containerized services communicating through Docker networks</p>
                </div>
            </div>
            
            <div class="api-section">
                <h2>API Endpoints</h2>
                <div class="api-endpoint">
                    <span class="method get">GET</span> /health - Health check endpoint
                </div>
                <div class="api-endpoint">
                    <span class="method post">POST</span> /api/auth/register - User registration
                </div>
                <div class="api-endpoint">
                    <span class="method post">POST</span> /api/auth/login - User login
                </div>
                <div class="api-endpoint">
                    <span class="method get">GET</span> /api/tasks - Get user tasks (requires auth)
                </div>
                <div class="api-endpoint">
                    <span class="method post">POST</span> /api/tasks - Create new task (requires auth)
                </div>
                <div class="api-endpoint">
                    <span class="method put">PUT</span> /api/tasks/:id - Update task (requires auth)
                </div>
                <div class="api-endpoint">
                    <span class="method delete">DELETE</span> /api/tasks/:id - Delete task (requires auth)
                </div>
                <div class="api-endpoint">
                    <span class="method get">GET</span> /api/stats - Database statistics
                </div>
            </div>
        </div>
        
        <footer>
            <p>&copy; 2024 Docker Two-Tier Application. Built with Node.js, MySQL, and Docker.</p>
            <p>Demonstrating container networking and multi-service architecture.</p>
        </footer>
    </div>
</body>
</html>
EOF
```

#### 3.4 Frontend Dockerfile
```bash
cat > frontend/Dockerfile << 'EOF'
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001 -G nodejs

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application code
COPY --chown=nodeuser:nodejs . .

# Switch to non-root user
USER nodeuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
EOF
```

### Step 4: Docker Compose Configuration

#### 4.1 Complete Docker Compose Setup
```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # MySQL Database
  mysql-db:
    build:
      context: ./database
      dockerfile: Dockerfile
    container_name: mysql-database
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./database/init:/docker-entrypoint-initdb.d
      - ./database/my.cnf:/etc/mysql/conf.d/my.cnf
    networks:
      - app-network
    ports:
      - "3306:3306"
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD}"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  # Node.js Frontend Application
  frontend-app:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: nodejs-frontend
    environment:
      - NODE_ENV=production
      - APP_PORT=3000
      - DB_HOST=mysql-db
      - DB_PORT=3306
      - DB_NAME=${MYSQL_DATABASE}
      - DB_USER=${MYSQL_USER}
      - DB_PASSWORD=${MYSQL_PASSWORD}
      - JWT_SECRET=your-super-secret-jwt-key-change-in-production
    ports:
      - "3000:3000"
    depends_on:
      mysql-db:
        condition: service_healthy
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  # Nginx Load Balancer (Optional)
  nginx-lb:
    image: nginx:alpine
    container_name: nginx-loadbalancer
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - frontend-app
    networks:
      - app-network
    restart: unless-stopped

  # Redis Cache (Optional)
  redis-cache:
    image: redis:7-alpine
    container_name: redis-cache
    command: redis-server --appendonly yes --requirepass ${REDIS_PASSWORD:-redispassword}
    volumes:
      - redis_data:/data
    networks:
      - app-network
    restart: unless-stopped

volumes:
  mysql_data:
    driver: local
  redis_data:
    driver: local

networks:
  app-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
EOF

cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream frontend_servers {
        server frontend-app:3000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://frontend_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            access_log off;
            return 200 "Nginx OK\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF
```

### Step 5: Testing and Deployment

#### 5.1 Build and Run Application
```bash
# Build and start all services
docker-compose up -d --build

# Check service status
docker-compose ps

# View logs
docker-compose logs -f frontend-app
docker-compose logs -f mysql-db

# Test database connection
docker-compose exec mysql-db mysql -u appuser -p${MYSQL_PASSWORD} -e "SHOW DATABASES;"

# Test application endpoints
curl http://localhost:3000/health
curl http://localhost:3000/api/stats
```

#### 5.2 API Testing Script
```bash
cat > test-api.sh << 'EOF'
#!/bin/bash

BASE_URL="http://localhost:3000"

echo "🧪 Testing Two-Tier Application API"
echo "=================================="

# Test health endpoint
echo "1. Testing health endpoint..."
curl -s "$BASE_URL/health" | jq '.'

# Test user registration
echo -e "\n2. Testing user registration..."
REGISTER_RESPONSE=$(curl -s -X POST "$BASE_URL/api/auth/register" \
    -H "Content-Type: application/json" \
    -d '{
        "username": "testuser",
        "email": "test@example.com",
        "password": "testpass123"
    }')
echo $REGISTER_RESPONSE | jq '.'

# Test user login
echo -e "\n3. Testing user login..."
LOGIN_RESPONSE=$(curl -s -X POST "$BASE_URL/api/auth/login" \
    -H "Content-Type: application/json" \
    -d '{
        "username": "testuser",
        "password": "testpass123"
    }')
echo $LOGIN_RESPONSE | jq '.'

# Extract token
TOKEN=$(echo $LOGIN_RESPONSE | jq -r '.token')

if [ "$TOKEN" != "null" ]; then
    # Test creating task
    echo -e "\n4. Testing task creation..."
    curl -s -X POST "$BASE_URL/api/tasks" \
        -H "Content-Type: application/json" \
        -H "Authorization: Bearer $TOKEN" \
        -d '{
            "title": "Test Docker Task",
            "description": "Testing Docker networking",
            "priority": "high"
        }' | jq '.'

    # Test getting tasks
    echo -e "\n5. Testing get tasks..."
    curl -s "$BASE_URL/api/tasks" \
        -H "Authorization: Bearer $TOKEN" | jq '.'
fi

# Test statistics
echo -e "\n6. Testing statistics..."
curl -s "$BASE_URL/api/stats" | jq '.'

echo -e "\n✅ API testing completed!"
EOF

chmod +x test-api.sh
./test-api.sh
```

## 📚 Interview Questions & Answers

### Intermediate Level (1-10)

**Q1: How does Docker networking enable communication between containers?**
A: Docker creates virtual networks that allow containers to communicate using service names as hostnames. Containers on the same network can reach each other using container names, which Docker's internal DNS resolves to container IP addresses.

**Q2: What are the advantages of using Docker networks over linking containers?**
A: Docker networks provide better isolation, support for multiple containers, automatic service discovery, network-level security, and the ability to connect/disconnect containers dynamically without restarting them.

**Q3: How do you handle database connections in containerized applications?**
A: Use connection pooling, implement retry logic with exponential backoff, use health checks, set appropriate timeouts, handle connection failures gracefully, and use environment variables for configuration.

**Q4: What is the difference between bridge and overlay networks in Docker?**
A: Bridge networks are for single-host container communication, while overlay networks enable multi-host container communication across Docker Swarm clusters or multiple Docker hosts.

**Q5: How do you ensure data persistence in containerized databases?**
A: Use Docker volumes for data persistence, implement regular backups, use named volumes for better management, and ensure proper volume mounting and permissions.

### Advanced Level (6-15)

**Q6: How do you implement service discovery in Docker networks?**
A: Docker provides built-in DNS-based service discovery where containers can reach each other using service names. For advanced scenarios, use external service discovery tools like Consul, etcd, or Kubernetes services.

**Q7: What are the security considerations for Docker networking?**
A: Use custom networks instead of default bridge, implement network segmentation, use secrets management, enable TLS for inter-service communication, and regularly update base images and dependencies.

**Q8: How do you handle database migrations in containerized environments?**
A: Use init containers, implement migration scripts in application startup, use database migration tools, maintain backward compatibility, and implement rollback strategies.

**Q9: What strategies do you use for scaling two-tier applications?**
A: Horizontal scaling of application tier, database read replicas, connection pooling, caching layers, load balancing, and proper resource allocation and monitoring.

**Q10: How do you monitor and troubleshoot Docker network issues?**
A: Use `docker network inspect`, check container logs, use network debugging tools, monitor network metrics, implement health checks, and use distributed tracing for complex applications.

## 🔑 Key Takeaways

- **Docker Networking**: Enables seamless container communication
- **Service Discovery**: Use container names for internal communication
- **Data Persistence**: Implement proper volume management for databases
- **Security**: Follow network security best practices
- **Monitoring**: Implement comprehensive health checks and logging
- **Scalability**: Design for horizontal scaling from the start

## 🚀 Next Steps

- Day 28: Docker Volumes and Docker Compose Deep Dive
- Day 29: CI/CD with Jenkins
- Container orchestration with Kubernetes
- Advanced networking with service mesh

---

**Hands-on Completed:** ✅ Two-Tier Application, Docker Networking, Database Integration  
**Duration:** 4-5 hours  
**Difficulty:** Intermediate to Advanced