# Day 31: Jenkins CI/CD Project - Advanced Pipeline Implementation

## Learning Objectives
- Deploy two-tier application using Jenkins CI/CD pipeline
- Implement Jenkins shared libraries and pipeline optimization
- Configure blue-green deployment strategies
- Set up pipeline monitoring and troubleshooting

## Topics Covered

### 1. Two-Tier Project Deployment
- **Frontend**: React/Angular application
- **Backend**: Node.js/Python API
- **Database**: MySQL/PostgreSQL
- **Containerization**: Docker containers
- **Orchestration**: Jenkins pipeline automation

### 2. Jenkins Shared Libraries
```groovy
// vars/deployApp.groovy
def call(Map config) {
    pipeline {
        agent any
        stages {
            stage('Build') {
                steps {
                    script {
                        docker.build("${config.imageName}:${config.tag}")
                    }
                }
            }
            stage('Deploy') {
                steps {
                    script {
                        sh "docker run -d -p ${config.port}:${config.port} ${config.imageName}:${config.tag}"
                    }
                }
            }
        }
    }
}
```

### 3. Blue-Green Deployment Pipeline
```groovy
pipeline {
    agent any
    environment {
        BLUE_PORT = '3000'
        GREEN_PORT = '3001'
        CURRENT_ENV = sh(script: 'cat /tmp/current_env || echo blue', returnStdout: true).trim()
    }
    stages {
        stage('Determine Target Environment') {
            steps {
                script {
                    env.TARGET_ENV = env.CURRENT_ENV == 'blue' ? 'green' : 'blue'
                    env.TARGET_PORT = env.TARGET_ENV == 'blue' ? env.BLUE_PORT : env.GREEN_PORT
                }
            }
        }
        stage('Deploy to Target') {
            steps {
                sh """
                    docker stop app-${TARGET_ENV} || true
                    docker rm app-${TARGET_ENV} || true
                    docker run -d --name app-${TARGET_ENV} -p ${TARGET_PORT}:3000 myapp:latest
                """
            }
        }
        stage('Health Check') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitUntil {
                            script {
                                def response = sh(script: "curl -s -o /dev/null -w '%{http_code}' http://localhost:${TARGET_PORT}/health", returnStdout: true)
                                return response == '200'
                            }
                        }
                    }
                }
            }
        }
        stage('Switch Traffic') {
            steps {
                sh """
                    # Update load balancer or reverse proxy
                    echo ${TARGET_ENV} > /tmp/current_env
                    # Stop old environment
                    OLD_ENV=\$([ "${TARGET_ENV}" = "blue" ] && echo "green" || echo "blue")
                    docker stop app-\$OLD_ENV || true
                """
            }
        }
    }
}
```

## Hands-on Labs

### Lab 1: Complete Two-Tier Application Setup
```bash
# 1. Create application structure
mkdir two-tier-app && cd two-tier-app
mkdir frontend backend database

# 2. Frontend Dockerfile
cat > frontend/Dockerfile << EOF
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build
EXPOSE 3000
CMD ["npm", "start"]
EOF

# 3. Backend Dockerfile
cat > backend/Dockerfile << EOF
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 5000
CMD ["npm", "start"]
EOF

# 4. Docker Compose for local testing
cat > docker-compose.yml << EOF
version: '3.8'
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"
    depends_on:
      - backend
  backend:
    build: ./backend
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=database
      - DB_USER=root
      - DB_PASSWORD=password
    depends_on:
      - database
  database:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=myapp
    ports:
      - "3306:3306"
EOF
```

### Lab 2: Jenkins Pipeline with GitHub Webhook
```groovy
pipeline {
    agent any
    
    environment {
        DOCKER_HUB_REPO = 'your-username/two-tier-app'
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/your-username/two-tier-app.git'
            }
        }
        
        stage('Build Images') {
            parallel {
                stage('Build Frontend') {
                    steps {
                        dir('frontend') {
                            script {
                                def frontendImage = docker.build("${DOCKER_HUB_REPO}-frontend:${BUILD_NUMBER}")
                                docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                                    frontendImage.push()
                                    frontendImage.push('latest')
                                }
                            }
                        }
                    }
                }
                stage('Build Backend') {
                    steps {
                        dir('backend') {
                            script {
                                def backendImage = docker.build("${DOCKER_HUB_REPO}-backend:${BUILD_NUMBER}")
                                docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                                    backendImage.push()
                                    backendImage.push('latest')
                                }
                            }
                        }
                    }
                }
            }
        }
        
        stage('Deploy') {
            steps {
                sh '''
                    docker-compose down || true
                    docker-compose pull
                    docker-compose up -d
                '''
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        waitUntil {
                            script {
                                def response = sh(script: 'curl -s -o /dev/null -w "%{http_code}" http://localhost:3000', returnStdout: true)
                                return response == '200'
                            }
                        }
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            slackSend(channel: '#deployments', color: 'good', message: "✅ Deployment successful: ${env.JOB_NAME} - ${env.BUILD_NUMBER}")
        }
        failure {
            slackSend(channel: '#deployments', color: 'danger', message: "❌ Deployment failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}")
        }
    }
}
```

### Lab 3: Pipeline Monitoring Setup
```bash
# 1. Install Jenkins plugins
# - Blue Ocean
# - Pipeline Stage View
# - Build Monitor View
# - Prometheus metrics

# 2. Configure Prometheus metrics
# Add to Jenkins configuration
prometheus:
  path: /prometheus
  defaultNamespace: jenkins
  processingDisabledBuilds: false
```

## Real-world Scenarios

### Scenario 1: Multi-Environment Deployment
```groovy
pipeline {
    agent any
    parameters {
        choice(name: 'ENVIRONMENT', choices: ['dev', 'staging', 'prod'], description: 'Target environment')
        booleanParam(name: 'SKIP_TESTS', defaultValue: false, description: 'Skip test execution')
    }
    
    stages {
        stage('Environment Setup') {
            steps {
                script {
                    env.TARGET_HOST = params.ENVIRONMENT == 'prod' ? 'prod-server.com' : "${params.ENVIRONMENT}-server.com"
                    env.DB_HOST = params.ENVIRONMENT == 'prod' ? 'prod-db.com' : "${params.ENVIRONMENT}-db.com"
                }
            }
        }
        
        stage('Tests') {
            when {
                not { params.SKIP_TESTS }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm test'
                    }
                }
                stage('Integration Tests') {
                    steps {
                        sh 'npm run test:integration'
                    }
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    if (params.ENVIRONMENT == 'prod') {
                        input message: 'Deploy to production?', ok: 'Deploy'
                    }
                }
                sh """
                    ssh deploy@${TARGET_HOST} '
                        docker-compose down
                        docker-compose pull
                        docker-compose up -d
                    '
                """
            }
        }
    }
}
```

## Troubleshooting Guide

### Common Issues and Solutions

1. **Pipeline Fails at Docker Build**
```bash
# Check Docker daemon
sudo systemctl status docker
sudo systemctl start docker

# Clean Docker cache
docker system prune -f
```

2. **GitHub Webhook Not Triggering**
```bash
# Check Jenkins webhook URL
# Format: http://jenkins-server:8080/github-webhook/

# Verify GitHub webhook settings
# Content type: application/json
# Events: Push events, Pull requests
```

3. **Blue-Green Deployment Issues**
```bash
# Check container status
docker ps -a

# Check application logs
docker logs app-blue
docker logs app-green

# Verify port availability
netstat -tulpn | grep :3000
```

## Interview Questions

### Beginner Level
1. **Q: What is the difference between Continuous Integration and Continuous Deployment?**
   A: CI focuses on automatically integrating code changes and running tests, while CD extends this to automatically deploy applications to production environments.

2. **Q: How do you configure a Jenkins pipeline to trigger on code commits?**
   A: Use GitHub webhooks or polling SCM. Configure webhook URL in GitHub repository settings pointing to Jenkins webhook endpoint.

3. **Q: What are Jenkins shared libraries?**
   A: Reusable pipeline code stored in version control that can be shared across multiple Jenkins pipelines to avoid code duplication.

### Intermediate Level
4. **Q: Explain blue-green deployment strategy and its benefits.**
   A: Blue-green deployment maintains two identical production environments. Traffic switches between them during deployments, enabling zero-downtime deployments and quick rollbacks.

5. **Q: How do you handle secrets in Jenkins pipelines?**
   A: Use Jenkins credentials store, environment variables, or external secret management tools like HashiCorp Vault or AWS Secrets Manager.

6. **Q: What is the purpose of pipeline stages and how do they help in CI/CD?**
   A: Stages organize pipeline execution into logical phases (build, test, deploy), provide visual feedback, enable parallel execution, and allow for stage-specific configurations.

### Advanced Level
7. **Q: How would you implement a canary deployment strategy in Jenkins?**
   A: Deploy new version to a subset of servers, monitor metrics, gradually increase traffic to new version based on success criteria, with automatic rollback on failures.

8. **Q: Describe how to implement pipeline as code with version control.**
   A: Store Jenkinsfile in source code repository, use multibranch pipelines, implement code review process for pipeline changes, and maintain pipeline versioning.

9. **Q: How do you optimize Jenkins pipeline performance?**
   A: Use parallel stages, optimize Docker builds with multi-stage builds and caching, implement pipeline caching, use appropriate agents, and minimize checkout operations.

10. **Q: Explain how to implement cross-pipeline dependencies and orchestration.**
    A: Use pipeline triggers, upstream/downstream job relationships, build parameters passing, shared workspaces, and external orchestration tools like Jenkins X or Tekton.

## Key Takeaways
- Jenkins pipelines enable automated, repeatable deployments
- Blue-green deployments provide zero-downtime releases
- Shared libraries promote code reusability across pipelines
- Proper monitoring and alerting are crucial for production pipelines
- Security and secret management must be integrated into CI/CD processes
- Pipeline optimization improves development velocity and resource utilization