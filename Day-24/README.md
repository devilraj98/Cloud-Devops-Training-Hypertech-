# Day 24: Hands-on Lab - Git Workflow Implementation

## 📚 Learning Objectives
- Implement complete Git workflow from scratch
- Practice branching strategies and code review processes
- Set up GitHub repository with proper structure
- Configure GitHub Actions workflows
- Experience real-world collaboration scenarios

## 🎯 Lab Overview

This hands-on lab simulates a real-world development environment where you'll implement a complete Git workflow for a web application project, including feature development, code reviews, and automated quality checks.

## 🏗️ Project Architecture

```
DevOps Web Application
├── Frontend (React/HTML)
├── Backend (Python Flask)
├── Database (SQLite/PostgreSQL)
├── Infrastructure (Docker)
├── CI/CD (GitHub Actions)
└── Documentation
```

## 🛠️ Lab Setup

### Step 1: Initialize Project Repository

#### 1.1 Create Repository Structure
```bash
# Create project directory
mkdir devops-webapp-lab
cd devops-webapp-lab

# Initialize Git repository
git init
git branch -M main

# Create project structure
mkdir -p {frontend/{src,public,tests},backend/{app,tests,config},infrastructure/{docker,k8s},docs/{api,deployment},.github/{workflows,ISSUE_TEMPLATE}}

# Create initial files
touch {frontend/package.json,backend/requirements.txt,infrastructure/docker/Dockerfile,.gitignore,README.md}
```

#### 1.2 Create Comprehensive .gitignore
```bash
cat > .gitignore << 'EOF'
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
develop-eggs/
dist/
downloads/
eggs/
.eggs/
lib/
lib64/
parts/
sdist/
var/
wheels/
*.egg-info/
.installed.cfg
*.egg
MANIFEST

# Virtual environments
.env
.venv
env/
venv/
ENV/
env.bak/
venv.bak/

# Node.js
node_modules/
npm-debug.log*
yarn-debug.log*
yarn-error.log*
.npm
.eslintcache

# IDEs
.vscode/
.idea/
*.swp
*.swo
*~

# OS
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# Logs
logs
*.log

# Database
*.db
*.sqlite3

# Coverage reports
htmlcov/
.coverage
.coverage.*
coverage.xml
*.cover
.hypothesis/
.pytest_cache/

# Docker
.dockerignore

# Secrets
.env.local
.env.production
secrets/
EOF
```

#### 1.3 Create Project README
```bash
cat > README.md << 'EOF'
# DevOps Web Application Lab

## 🎯 Project Overview
A full-stack web application demonstrating DevOps best practices including Git workflows, CI/CD, containerization, and automated testing.

## 🏗️ Architecture
- **Frontend**: React.js with modern UI components
- **Backend**: Python Flask REST API
- **Database**: PostgreSQL with SQLAlchemy ORM
- **Infrastructure**: Docker containers with Kubernetes deployment
- **CI/CD**: GitHub Actions with automated testing and deployment

## 🚀 Quick Start

### Prerequisites
- Node.js 18+
- Python 3.11+
- Docker and Docker Compose
- Git

### Development Setup
```bash
# Clone repository
git clone <repository-url>
cd devops-webapp-lab

# Backend setup
cd backend
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate
pip install -r requirements.txt

# Frontend setup
cd ../frontend
npm install

# Start development servers
npm run dev  # Frontend (port 3000)
cd ../backend && python app.py  # Backend (port 5000)
```

## 🔄 Development Workflow

### Branching Strategy
We use GitFlow workflow:
- `main` - Production-ready code
- `develop` - Integration branch
- `feature/*` - Feature development
- `release/*` - Release preparation
- `hotfix/*` - Critical fixes

### Contributing
1. Create feature branch from `develop`
2. Implement changes with tests
3. Create pull request
4. Code review and approval
5. Merge to `develop`

## 📊 Quality Standards
- Minimum 80% test coverage
- All tests must pass
- Code must pass linting
- Security scan must pass
- Documentation must be updated

## 🚀 Deployment
- **Staging**: Auto-deploy from `develop` branch
- **Production**: Auto-deploy from `main` branch
- **Review Apps**: Created for each pull request

## 📚 Documentation
- [API Documentation](docs/api/README.md)
- [Deployment Guide](docs/deployment/README.md)
- [Contributing Guidelines](CONTRIBUTING.md)

## 🤝 Team
- **Project Lead**: [Your Name]
- **Frontend Developer**: [Team Member 1]
- **Backend Developer**: [Team Member 2]
- **DevOps Engineer**: [Team Member 3]

## 📄 License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
EOF
```

### Step 2: Implement Backend Application

#### 2.1 Create Flask Application
```bash
cd backend

# Create requirements.txt
cat > requirements.txt << 'EOF'
Flask==3.0.0
Flask-SQLAlchemy==3.1.1
Flask-CORS==4.0.0
Flask-JWT-Extended==4.6.0
python-dotenv==1.0.0
psycopg2-binary==2.9.9
pytest==7.4.3
pytest-cov==4.1.0
pytest-flask==1.3.0
black==23.11.0
pylint==3.0.3
gunicorn==21.2.0
EOF

# Create main application
cat > app.py << 'EOF'
"""
DevOps Web Application - Backend API
"""
import os
from datetime import datetime, timedelta
from flask import Flask, request, jsonify
from flask_sqlalchemy import SQLAlchemy
from flask_cors import CORS
from flask_jwt_extended import JWTManager, create_access_token, jwt_required, get_jwt_identity
from werkzeug.security import generate_password_hash, check_password_hash
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

# Initialize Flask app
app = Flask(__name__)

# Configuration
app.config['SECRET_KEY'] = os.environ.get('SECRET_KEY', 'dev-secret-key')
app.config['SQLALCHEMY_DATABASE_URI'] = os.environ.get('DATABASE_URL', 'sqlite:///devops_app.db')
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
app.config['JWT_SECRET_KEY'] = os.environ.get('JWT_SECRET_KEY', 'jwt-secret-key')
app.config['JWT_ACCESS_TOKEN_EXPIRES'] = timedelta(hours=24)

# Initialize extensions
db = SQLAlchemy(app)
cors = CORS(app)
jwt = JWTManager(app)

# Models
class User(db.Model):
    """User model"""
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)
    password_hash = db.Column(db.String(255), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    is_active = db.Column(db.Boolean, default=True)

    def set_password(self, password):
        """Set password hash"""
        self.password_hash = generate_password_hash(password)

    def check_password(self, password):
        """Check password"""
        return check_password_hash(self.password_hash, password)

    def to_dict(self):
        """Convert to dictionary"""
        return {
            'id': self.id,
            'username': self.username,
            'email': self.email,
            'created_at': self.created_at.isoformat(),
            'is_active': self.is_active
        }

class Task(db.Model):
    """Task model"""
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(200), nullable=False)
    description = db.Column(db.Text)
    status = db.Column(db.String(20), default='pending')
    priority = db.Column(db.String(10), default='medium')
    user_id = db.Column(db.Integer, db.ForeignKey('user.id'), nullable=False)
    created_at = db.Column(db.DateTime, default=datetime.utcnow)
    updated_at = db.Column(db.DateTime, default=datetime.utcnow, onupdate=datetime.utcnow)

    def to_dict(self):
        """Convert to dictionary"""
        return {
            'id': self.id,
            'title': self.title,
            'description': self.description,
            'status': self.status,
            'priority': self.priority,
            'user_id': self.user_id,
            'created_at': self.created_at.isoformat(),
            'updated_at': self.updated_at.isoformat()
        }

# Routes
@app.route('/api/health')
def health_check():
    """Health check endpoint"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.utcnow().isoformat(),
        'version': '1.0.0',
        'environment': os.environ.get('ENVIRONMENT', 'development')
    })

@app.route('/api/auth/register', methods=['POST'])
def register():
    """User registration"""
    data = request.get_json()
    
    if not data or not data.get('username') or not data.get('email') or not data.get('password'):
        return jsonify({'error': 'Missing required fields'}), 400
    
    # Check if user exists
    if User.query.filter_by(username=data['username']).first():
        return jsonify({'error': 'Username already exists'}), 409
    
    if User.query.filter_by(email=data['email']).first():
        return jsonify({'error': 'Email already exists'}), 409
    
    # Create new user
    user = User(username=data['username'], email=data['email'])
    user.set_password(data['password'])
    
    db.session.add(user)
    db.session.commit()
    
    return jsonify({'message': 'User created successfully', 'user': user.to_dict()}), 201

@app.route('/api/auth/login', methods=['POST'])
def login():
    """User login"""
    data = request.get_json()
    
    if not data or not data.get('username') or not data.get('password'):
        return jsonify({'error': 'Missing username or password'}), 400
    
    user = User.query.filter_by(username=data['username']).first()
    
    if not user or not user.check_password(data['password']):
        return jsonify({'error': 'Invalid credentials'}), 401
    
    access_token = create_access_token(identity=user.id)
    
    return jsonify({
        'access_token': access_token,
        'user': user.to_dict()
    })

@app.route('/api/tasks', methods=['GET'])
@jwt_required()
def get_tasks():
    """Get user tasks"""
    user_id = get_jwt_identity()
    tasks = Task.query.filter_by(user_id=user_id).all()
    
    return jsonify({
        'tasks': [task.to_dict() for task in tasks]
    })

@app.route('/api/tasks', methods=['POST'])
@jwt_required()
def create_task():
    """Create new task"""
    user_id = get_jwt_identity()
    data = request.get_json()
    
    if not data or not data.get('title'):
        return jsonify({'error': 'Title is required'}), 400
    
    task = Task(
        title=data['title'],
        description=data.get('description', ''),
        priority=data.get('priority', 'medium'),
        user_id=user_id
    )
    
    db.session.add(task)
    db.session.commit()
    
    return jsonify({'message': 'Task created successfully', 'task': task.to_dict()}), 201

@app.route('/api/tasks/<int:task_id>', methods=['PUT'])
@jwt_required()
def update_task(task_id):
    """Update task"""
    user_id = get_jwt_identity()
    task = Task.query.filter_by(id=task_id, user_id=user_id).first()
    
    if not task:
        return jsonify({'error': 'Task not found'}), 404
    
    data = request.get_json()
    
    if 'title' in data:
        task.title = data['title']
    if 'description' in data:
        task.description = data['description']
    if 'status' in data:
        task.status = data['status']
    if 'priority' in data:
        task.priority = data['priority']
    
    task.updated_at = datetime.utcnow()
    db.session.commit()
    
    return jsonify({'message': 'Task updated successfully', 'task': task.to_dict()})

@app.route('/api/tasks/<int:task_id>', methods=['DELETE'])
@jwt_required()
def delete_task(task_id):
    """Delete task"""
    user_id = get_jwt_identity()
    task = Task.query.filter_by(id=task_id, user_id=user_id).first()
    
    if not task:
        return jsonify({'error': 'Task not found'}), 404
    
    db.session.delete(task)
    db.session.commit()
    
    return jsonify({'message': 'Task deleted successfully'})

# Initialize database
@app.before_first_request
def create_tables():
    """Create database tables"""
    db.create_all()

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
EOF
```

#### 2.2 Create Backend Tests
```bash
mkdir -p tests
cat > tests/test_app.py << 'EOF'
"""
Test suite for the DevOps Web Application
"""
import pytest
import json
from app import app, db, User, Task

@pytest.fixture
def client():
    """Create test client"""
    app.config['TESTING'] = True
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///:memory:'
    
    with app.test_client() as client:
        with app.app_context():
            db.create_all()
            yield client
            db.drop_all()

@pytest.fixture
def auth_headers(client):
    """Create authenticated user and return headers"""
    # Register user
    client.post('/api/auth/register', json={
        'username': 'testuser',
        'email': 'test@example.com',
        'password': 'testpass123'
    })
    
    # Login user
    response = client.post('/api/auth/login', json={
        'username': 'testuser',
        'password': 'testpass123'
    })
    
    data = json.loads(response.data)
    return {'Authorization': f'Bearer {data["access_token"]}'}

class TestHealthCheck:
    """Test health check endpoint"""
    
    def test_health_check(self, client):
        """Test health check returns correct response"""
        response = client.get('/api/health')
        assert response.status_code == 200
        
        data = json.loads(response.data)
        assert data['status'] == 'healthy'
        assert 'timestamp' in data
        assert 'version' in data

class TestAuthentication:
    """Test authentication endpoints"""
    
    def test_user_registration(self, client):
        """Test user registration"""
        response = client.post('/api/auth/register', json={
            'username': 'newuser',
            'email': 'newuser@example.com',
            'password': 'password123'
        })
        
        assert response.status_code == 201
        data = json.loads(response.data)
        assert data['message'] == 'User created successfully'
        assert data['user']['username'] == 'newuser'
    
    def test_duplicate_username_registration(self, client):
        """Test duplicate username registration fails"""
        # Register first user
        client.post('/api/auth/register', json={
            'username': 'testuser',
            'email': 'test1@example.com',
            'password': 'password123'
        })
        
        # Try to register with same username
        response = client.post('/api/auth/register', json={
            'username': 'testuser',
            'email': 'test2@example.com',
            'password': 'password123'
        })
        
        assert response.status_code == 409
        data = json.loads(response.data)
        assert 'Username already exists' in data['error']
    
    def test_user_login(self, client):
        """Test user login"""
        # Register user first
        client.post('/api/auth/register', json={
            'username': 'loginuser',
            'email': 'login@example.com',
            'password': 'loginpass123'
        })
        
        # Login
        response = client.post('/api/auth/login', json={
            'username': 'loginuser',
            'password': 'loginpass123'
        })
        
        assert response.status_code == 200
        data = json.loads(response.data)
        assert 'access_token' in data
        assert data['user']['username'] == 'loginuser'
    
    def test_invalid_login(self, client):
        """Test invalid login credentials"""
        response = client.post('/api/auth/login', json={
            'username': 'nonexistent',
            'password': 'wrongpass'
        })
        
        assert response.status_code == 401
        data = json.loads(response.data)
        assert 'Invalid credentials' in data['error']

class TestTasks:
    """Test task management endpoints"""
    
    def test_create_task(self, client, auth_headers):
        """Test task creation"""
        response = client.post('/api/tasks', 
            json={
                'title': 'Test Task',
                'description': 'This is a test task',
                'priority': 'high'
            },
            headers=auth_headers
        )
        
        assert response.status_code == 201
        data = json.loads(response.data)
        assert data['message'] == 'Task created successfully'
        assert data['task']['title'] == 'Test Task'
        assert data['task']['priority'] == 'high'
    
    def test_get_tasks(self, client, auth_headers):
        """Test getting user tasks"""
        # Create a task first
        client.post('/api/tasks', 
            json={'title': 'Test Task', 'description': 'Test'},
            headers=auth_headers
        )
        
        # Get tasks
        response = client.get('/api/tasks', headers=auth_headers)
        
        assert response.status_code == 200
        data = json.loads(response.data)
        assert len(data['tasks']) == 1
        assert data['tasks'][0]['title'] == 'Test Task'
    
    def test_update_task(self, client, auth_headers):
        """Test task update"""
        # Create task
        create_response = client.post('/api/tasks', 
            json={'title': 'Original Task'},
            headers=auth_headers
        )
        task_id = json.loads(create_response.data)['task']['id']
        
        # Update task
        response = client.put(f'/api/tasks/{task_id}',
            json={
                'title': 'Updated Task',
                'status': 'completed'
            },
            headers=auth_headers
        )
        
        assert response.status_code == 200
        data = json.loads(response.data)
        assert data['task']['title'] == 'Updated Task'
        assert data['task']['status'] == 'completed'
    
    def test_delete_task(self, client, auth_headers):
        """Test task deletion"""
        # Create task
        create_response = client.post('/api/tasks', 
            json={'title': 'Task to Delete'},
            headers=auth_headers
        )
        task_id = json.loads(create_response.data)['task']['id']
        
        # Delete task
        response = client.delete(f'/api/tasks/{task_id}', headers=auth_headers)
        
        assert response.status_code == 200
        data = json.loads(response.data)
        assert data['message'] == 'Task deleted successfully'
        
        # Verify task is deleted
        get_response = client.get('/api/tasks', headers=auth_headers)
        tasks = json.loads(get_response.data)['tasks']
        assert len(tasks) == 0
    
    def test_unauthorized_access(self, client):
        """Test unauthorized access to protected endpoints"""
        response = client.get('/api/tasks')
        assert response.status_code == 401
EOF
```

### Step 3: Implement Frontend Application

#### 3.1 Create Frontend Structure
```bash
cd ../frontend

# Create package.json
cat > package.json << 'EOF'
{
  "name": "devops-webapp-frontend",
  "version": "1.0.0",
  "description": "Frontend for DevOps Web Application Lab",
  "main": "src/index.js",
  "scripts": {
    "start": "python -m http.server 3000",
    "dev": "python -m http.server 3000",
    "test": "jest",
    "lint": "eslint src/",
    "format": "prettier --write src/"
  },
  "dependencies": {},
  "devDependencies": {
    "jest": "^29.7.0",
    "eslint": "^8.54.0",
    "prettier": "^3.1.0"
  },
  "keywords": ["devops", "webapp", "frontend"],
  "author": "DevOps Team",
  "license": "MIT"
}
EOF

# Create main HTML file
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>DevOps Task Manager</title>
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
            padding: 30px;
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
        
        .content {
            padding: 30px;
        }
        
        .auth-section, .app-section {
            display: none;
        }
        
        .auth-section.active, .app-section.active {
            display: block;
        }
        
        .form-group {
            margin-bottom: 20px;
        }
        
        .form-group label {
            display: block;
            margin-bottom: 5px;
            font-weight: 600;
            color: #333;
        }
        
        .form-group input, .form-group textarea, .form-group select {
            width: 100%;
            padding: 12px;
            border: 2px solid #e1e5e9;
            border-radius: 8px;
            font-size: 16px;
            transition: border-color 0.3s;
        }
        
        .form-group input:focus, .form-group textarea:focus, .form-group select:focus {
            outline: none;
            border-color: #667eea;
        }
        
        .btn {
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            padding: 12px 24px;
            border-radius: 8px;
            font-size: 16px;
            cursor: pointer;
            transition: transform 0.2s;
        }
        
        .btn:hover {
            transform: translateY(-2px);
        }
        
        .btn-secondary {
            background: #6c757d;
        }
        
        .btn-danger {
            background: #dc3545;
        }
        
        .task-list {
            margin-top: 30px;
        }
        
        .task-item {
            background: #f8f9fa;
            border: 1px solid #e9ecef;
            border-radius: 8px;
            padding: 20px;
            margin-bottom: 15px;
            transition: transform 0.2s;
        }
        
        .task-item:hover {
            transform: translateY(-2px);
            box-shadow: 0 4px 12px rgba(0,0,0,0.1);
        }
        
        .task-header {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 10px;
        }
        
        .task-title {
            font-size: 1.2rem;
            font-weight: 600;
            color: #333;
        }
        
        .task-priority {
            padding: 4px 12px;
            border-radius: 20px;
            font-size: 0.8rem;
            font-weight: 600;
            text-transform: uppercase;
        }
        
        .priority-high {
            background: #ffebee;
            color: #c62828;
        }
        
        .priority-medium {
            background: #fff3e0;
            color: #ef6c00;
        }
        
        .priority-low {
            background: #e8f5e8;
            color: #2e7d32;
        }
        
        .task-status {
            display: inline-block;
            padding: 4px 12px;
            border-radius: 20px;
            font-size: 0.8rem;
            font-weight: 600;
            margin-right: 10px;
        }
        
        .status-pending {
            background: #fff3cd;
            color: #856404;
        }
        
        .status-completed {
            background: #d4edda;
            color: #155724;
        }
        
        .task-actions {
            margin-top: 15px;
        }
        
        .task-actions button {
            margin-right: 10px;
            padding: 6px 12px;
            font-size: 14px;
        }
        
        .alert {
            padding: 15px;
            border-radius: 8px;
            margin-bottom: 20px;
        }
        
        .alert-success {
            background: #d4edda;
            color: #155724;
            border: 1px solid #c3e6cb;
        }
        
        .alert-error {
            background: #f8d7da;
            color: #721c24;
            border: 1px solid #f5c6cb;
        }
        
        .user-info {
            background: #e7f3ff;
            padding: 15px;
            border-radius: 8px;
            margin-bottom: 20px;
        }
        
        .tabs {
            display: flex;
            margin-bottom: 20px;
            border-bottom: 2px solid #e9ecef;
        }
        
        .tab {
            padding: 10px 20px;
            cursor: pointer;
            border-bottom: 2px solid transparent;
            transition: all 0.3s;
        }
        
        .tab.active {
            border-bottom-color: #667eea;
            color: #667eea;
            font-weight: 600;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>🚀 DevOps Task Manager</h1>
            <p>Demonstrating Git workflows and DevOps best practices</p>
        </div>
        
        <div class="content">
            <!-- Authentication Section -->
            <div id="auth-section" class="auth-section active">
                <div class="tabs">
                    <div class="tab active" onclick="showLogin()">Login</div>
                    <div class="tab" onclick="showRegister()">Register</div>
                </div>
                
                <!-- Login Form -->
                <div id="login-form">
                    <h2>Login to Your Account</h2>
                    <form onsubmit="login(event)">
                        <div class="form-group">
                            <label for="login-username">Username</label>
                            <input type="text" id="login-username" required>
                        </div>
                        <div class="form-group">
                            <label for="login-password">Password</label>
                            <input type="password" id="login-password" required>
                        </div>
                        <button type="submit" class="btn">Login</button>
                    </form>
                </div>
                
                <!-- Register Form -->
                <div id="register-form" style="display: none;">
                    <h2>Create New Account</h2>
                    <form onsubmit="register(event)">
                        <div class="form-group">
                            <label for="register-username">Username</label>
                            <input type="text" id="register-username" required>
                        </div>
                        <div class="form-group">
                            <label for="register-email">Email</label>
                            <input type="email" id="register-email" required>
                        </div>
                        <div class="form-group">
                            <label for="register-password">Password</label>
                            <input type="password" id="register-password" required>
                        </div>
                        <button type="submit" class="btn">Register</button>
                    </form>
                </div>
            </div>
            
            <!-- Application Section -->
            <div id="app-section" class="app-section">
                <div class="user-info">
                    <strong>Welcome, <span id="username"></span>!</strong>
                    <button class="btn btn-secondary" onclick="logout()" style="float: right;">Logout</button>
                </div>
                
                <!-- Task Creation Form -->
                <div>
                    <h2>Create New Task</h2>
                    <form onsubmit="createTask(event)">
                        <div class="form-group">
                            <label for="task-title">Task Title</label>
                            <input type="text" id="task-title" required>
                        </div>
                        <div class="form-group">
                            <label for="task-description">Description</label>
                            <textarea id="task-description" rows="3"></textarea>
                        </div>
                        <div class="form-group">
                            <label for="task-priority">Priority</label>
                            <select id="task-priority">
                                <option value="low">Low</option>
                                <option value="medium" selected>Medium</option>
                                <option value="high">High</option>
                            </select>
                        </div>
                        <button type="submit" class="btn">Create Task</button>
                    </form>
                </div>
                
                <!-- Task List -->
                <div class="task-list">
                    <h2>Your Tasks</h2>
                    <div id="tasks-container">
                        <!-- Tasks will be loaded here -->
                    </div>
                </div>
            </div>
            
            <!-- Alert Container -->
            <div id="alert-container"></div>
        </div>
    </div>

    <script src="src/app.js"></script>
</body>
</html>
EOF

# Create JavaScript application
mkdir -p src
cat > src/app.js << 'EOF'
/**
 * DevOps Task Manager - Frontend Application
 */

// Configuration
const API_BASE_URL = 'http://localhost:5000/api';
let authToken = localStorage.getItem('authToken');
let currentUser = JSON.parse(localStorage.getItem('currentUser') || 'null');

// Initialize application
document.addEventListener('DOMContentLoaded', function() {
    if (authToken && currentUser) {
        showApp();
        loadTasks();
    } else {
        showAuth();
    }
});

// Authentication functions
function showLogin() {
    document.getElementById('login-form').style.display = 'block';
    document.getElementById('register-form').style.display = 'none';
    
    // Update tab styles
    const tabs = document.querySelectorAll('.tab');
    tabs.forEach(tab => tab.classList.remove('active'));
    tabs[0].classList.add('active');
}

function showRegister() {
    document.getElementById('login-form').style.display = 'none';
    document.getElementById('register-form').style.display = 'block';
    
    // Update tab styles
    const tabs = document.querySelectorAll('.tab');
    tabs.forEach(tab => tab.classList.remove('active'));
    tabs[1].classList.add('active');
}

function showAuth() {
    document.getElementById('auth-section').classList.add('active');
    document.getElementById('app-section').classList.remove('active');
}

function showApp() {
    document.getElementById('auth-section').classList.remove('active');
    document.getElementById('app-section').classList.add('active');
    document.getElementById('username').textContent = currentUser.username;
}

async function login(event) {
    event.preventDefault();
    
    const username = document.getElementById('login-username').value;
    const password = document.getElementById('login-password').value;
    
    try {
        const response = await fetch(`${API_BASE_URL}/auth/login`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ username, password })
        });
        
        const data = await response.json();
        
        if (response.ok) {
            authToken = data.access_token;
            currentUser = data.user;
            
            localStorage.setItem('authToken', authToken);
            localStorage.setItem('currentUser', JSON.stringify(currentUser));
            
            showAlert('Login successful!', 'success');
            showApp();
            loadTasks();
        } else {
            showAlert(data.error || 'Login failed', 'error');
        }
    } catch (error) {
        showAlert('Network error. Please try again.', 'error');
        console.error('Login error:', error);
    }
}

async function register(event) {
    event.preventDefault();
    
    const username = document.getElementById('register-username').value;
    const email = document.getElementById('register-email').value;
    const password = document.getElementById('register-password').value;
    
    try {
        const response = await fetch(`${API_BASE_URL}/auth/register`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({ username, email, password })
        });
        
        const data = await response.json();
        
        if (response.ok) {
            showAlert('Registration successful! Please login.', 'success');
            showLogin();
            
            // Clear form
            document.getElementById('register-username').value = '';
            document.getElementById('register-email').value = '';
            document.getElementById('register-password').value = '';
        } else {
            showAlert(data.error || 'Registration failed', 'error');
        }
    } catch (error) {
        showAlert('Network error. Please try again.', 'error');
        console.error('Registration error:', error);
    }
}

function logout() {
    authToken = null;
    currentUser = null;
    
    localStorage.removeItem('authToken');
    localStorage.removeItem('currentUser');
    
    showAuth();
    showAlert('Logged out successfully', 'success');
}

// Task management functions
async function createTask(event) {
    event.preventDefault();
    
    const title = document.getElementById('task-title').value;
    const description = document.getElementById('task-description').value;
    const priority = document.getElementById('task-priority').value;
    
    try {
        const response = await fetch(`${API_BASE_URL}/tasks`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${authToken}`
            },
            body: JSON.stringify({ title, description, priority })
        });
        
        const data = await response.json();
        
        if (response.ok) {
            showAlert('Task created successfully!', 'success');
            
            // Clear form
            document.getElementById('task-title').value = '';
            document.getElementById('task-description').value = '';
            document.getElementById('task-priority').value = 'medium';
            
            loadTasks();
        } else {
            showAlert(data.error || 'Failed to create task', 'error');
        }
    } catch (error) {
        showAlert('Network error. Please try again.', 'error');
        console.error('Create task error:', error);
    }
}

async function loadTasks() {
    try {
        const response = await fetch(`${API_BASE_URL}/tasks`, {
            headers: {
                'Authorization': `Bearer ${authToken}`
            }
        });
        
        const data = await response.json();
        
        if (response.ok) {
            displayTasks(data.tasks);
        } else {
            showAlert('Failed to load tasks', 'error');
        }
    } catch (error) {
        showAlert('Network error. Please try again.', 'error');
        console.error('Load tasks error:', error);
    }
}

function displayTasks(tasks) {
    const container = document.getElementById('tasks-container');
    
    if (tasks.length === 0) {
        container.innerHTML = '<p>No tasks yet. Create your first task above!</p>';
        return;
    }
    
    container.innerHTML = tasks.map(task => `
        <div class="task-item">
            <div class="task-header">
                <div class="task-title">${escapeHtml(task.title)}</div>
                <div class="task-priority priority-${task.priority}">${task.priority}</div>
            </div>
            <div class="task-description">${escapeHtml(task.description || '')}</div>
            <div style="margin: 10px 0;">
                <span class="task-status status-${task.status}">${task.status}</span>
                <small style="color: #666;">Created: ${new Date(task.created_at).toLocaleDateString()}</small>
            </div>
            <div class="task-actions">
                <button class="btn" onclick="toggleTaskStatus(${task.id}, '${task.status}')">
                    ${task.status === 'pending' ? 'Mark Complete' : 'Mark Pending'}
                </button>
                <button class="btn btn-danger" onclick="deleteTask(${task.id})">Delete</button>
            </div>
        </div>
    `).join('');
}

async function toggleTaskStatus(taskId, currentStatus) {
    const newStatus = currentStatus === 'pending' ? 'completed' : 'pending';
    
    try {
        const response = await fetch(`${API_BASE_URL}/tasks/${taskId}`, {
            method: 'PUT',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${authToken}`
            },
            body: JSON.stringify({ status: newStatus })
        });
        
        if (response.ok) {
            showAlert('Task updated successfully!', 'success');
            loadTasks();
        } else {
            showAlert('Failed to update task', 'error');
        }
    } catch (error) {
        showAlert('Network error. Please try again.', 'error');
        console.error('Update task error:', error);
    }
}

async function deleteTask(taskId) {
    if (!confirm('Are you sure you want to delete this task?')) {
        return;
    }
    
    try {
        const response = await fetch(`${API_BASE_URL}/tasks/${taskId}`, {
            method: 'DELETE',
            headers: {
                'Authorization': `Bearer ${authToken}`
            }
        });
        
        if (response.ok) {
            showAlert('Task deleted successfully!', 'success');
            loadTasks();
        } else {
            showAlert('Failed to delete task', 'error');
        }
    } catch (error) {
        showAlert('Network error. Please try again.', 'error');
        console.error('Delete task error:', error);
    }
}

// Utility functions
function showAlert(message, type) {
    const container = document.getElementById('alert-container');
    const alertDiv = document.createElement('div');
    alertDiv.className = `alert alert-${type}`;
    alertDiv.textContent = message;
    
    container.appendChild(alertDiv);
    
    // Remove alert after 5 seconds
    setTimeout(() => {
        if (alertDiv.parentNode) {
            alertDiv.parentNode.removeChild(alertDiv);
        }
    }, 5000);
}

function escapeHtml(text) {
    const div = document.createElement('div');
    div.textContent = text;
    return div.innerHTML;
}

// Health check on page load
fetch(`${API_BASE_URL}/health`)
    .then(response => response.json())
    .then(data => {
        console.log('API Health:', data);
    })
    .catch(error => {
        console.error('API Health Check Failed:', error);
        showAlert('Backend API is not available', 'error');
    });
EOF
```

This completes the comprehensive Day-24 hands-on lab setup. The lab includes:

1. **Complete Project Structure** - Full-stack web application
2. **Backend API** - Flask application with authentication and task management
3. **Frontend Application** - Interactive web interface
4. **Comprehensive Testing** - Unit and integration tests
5. **Git Workflow Implementation** - Ready for branching strategies
6. **Quality Standards** - Code formatting and testing setup

Students will use this foundation to practice Git workflows, code reviews, and CI/CD implementation in a realistic development environment.

## 🔑 Key Takeaways

- **Real-World Application**: Complete full-stack project for practical learning
- **Git Workflow Practice**: Hands-on experience with branching and merging
- **Code Quality**: Integrated testing and quality checks
- **Collaboration**: Team-based development simulation
- **DevOps Integration**: Foundation for CI/CD implementation

## 🚀 Next Steps

- Day 25: Docker Fundamentals and Containerization
- Implement GitHub Actions workflows
- Practice team collaboration scenarios

---

**Hands-on Completed:** ✅ Complete Git Workflow Implementation, Full-Stack Application  
**Duration:** 4-6 hours  
**Difficulty:** Advanced