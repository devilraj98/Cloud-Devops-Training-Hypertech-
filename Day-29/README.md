# Day 29: CI/CD with Jenkins - Installation and Configuration

## 📚 Learning Objectives
- Understand CI/CD concepts and benefits
- Install and configure Jenkins
- Create basic Jenkins pipelines
- Integrate Jenkins with Git repositories
- Implement automated testing in pipelines
- Practice Jenkins security and best practices

## 🔄 CI/CD Overview

### What is CI/CD?
- **Continuous Integration (CI)**: Automatically build and test code changes
- **Continuous Delivery (CD)**: Automatically deploy to staging environments
- **Continuous Deployment**: Automatically deploy to production

### Benefits of CI/CD
- **Faster Delivery**: Automated processes reduce manual work
- **Quality Assurance**: Automated testing catches issues early
- **Reduced Risk**: Smaller, frequent deployments
- **Improved Collaboration**: Shared responsibility for code quality

## 🛠️ Jenkins Installation and Setup

### Method 1: Docker Installation (Recommended)

#### 1.1 Jenkins with Docker
```bash
# Create Jenkins directory
mkdir jenkins-setup
cd jenkins-setup

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  jenkins:
    image: jenkins/jenkins:lts
    container_name: jenkins-master
    user: root
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - jenkins_home:/var/jenkins_home
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    environment:
      - JENKINS_OPTS=--httpPort=8080
      - JAVA_OPTS=-Xmx2048m -Xms1024m
    restart: unless-stopped
    networks:
      - jenkins-network

  jenkins-agent:
    image: jenkins/inbound-agent:latest
    container_name: jenkins-agent
    environment:
      - JENKINS_URL=http://jenkins:8080
      - JENKINS_SECRET=${JENKINS_AGENT_SECRET}
      - JENKINS_AGENT_NAME=docker-agent
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /usr/bin/docker:/usr/bin/docker
    depends_on:
      - jenkins
    networks:
      - jenkins-network
    restart: unless-stopped

volumes:
  jenkins_home:
    driver: local

networks:
  jenkins-network:
    driver: bridge
EOF

# Start Jenkins
docker-compose up -d

# Get initial admin password
docker exec jenkins-master cat /var/jenkins_home/secrets/initialAdminPassword
```

#### 1.2 Custom Jenkins Dockerfile
```bash
cat > Dockerfile << 'EOF'
FROM jenkins/jenkins:lts

# Switch to root user to install additional packages
USER root

# Install Docker CLI, Node.js, Python, and other tools
RUN apt-get update && apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release \
    python3 \
    python3-pip \
    nodejs \
    npm \
    && rm -rf /var/lib/apt/lists/*

# Install Docker CLI
RUN curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
RUN echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
RUN apt-get update && apt-get install -y docker-ce-cli

# Install kubectl
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
RUN install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Install Terraform
RUN curl -fsSL https://apt.releases.hashicorp.com/gpg | apt-key add -
RUN apt-add-repository "deb [arch=amd64] https://apt.releases.hashicorp.com $(lsb_release -cs) main"
RUN apt-get update && apt-get install -y terraform

# Install Jenkins plugins
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

# Switch back to jenkins user
USER jenkins

# Copy Jenkins configuration
COPY jenkins.yaml /var/jenkins_home/casc_configs/jenkins.yaml
ENV CASC_JENKINS_CONFIG=/var/jenkins_home/casc_configs/jenkins.yaml
EOF

cat > plugins.txt << 'EOF'
ant:latest
antisamy-markup-formatter:latest
build-timeout:latest
credentials-binding:latest
email-ext:latest
git:latest
github:latest
gradle:latest
ldap:latest
mailer:latest
matrix-auth:latest
pam-auth:latest
pipeline-github-lib:latest
pipeline-stage-view:latest
ssh-slaves:latest
timestamper:latest
workflow-aggregator:latest
ws-cleanup:latest
blueocean:latest
docker-workflow:latest
kubernetes:latest
configuration-as-code:latest
job-dsl:latest
EOF
```

### Method 2: Native Installation

#### 2.1 Ubuntu/Debian Installation
```bash
# Update system
sudo apt update

# Install Java
sudo apt install -y openjdk-11-jdk

# Add Jenkins repository
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
    /usr/share/keyrings/jenkins-keyring.asc > /dev/null

echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
    https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
    /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install Jenkins
sudo apt update
sudo apt install -y jenkins

# Start and enable Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Check status
sudo systemctl status jenkins

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

#### 2.2 CentOS/RHEL Installation
```bash
# Install Java
sudo yum install -y java-11-openjdk-devel

# Add Jenkins repository
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo

sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key

# Install Jenkins
sudo yum install -y jenkins

# Start and enable Jenkins
sudo systemctl start jenkins
sudo systemctl enable jenkins

# Configure firewall
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload

# Get initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

## ⚙️ Jenkins Initial Configuration

### Step 1: Initial Setup Wizard
```bash
# Access Jenkins at http://localhost:8080
# Enter initial admin password
# Install suggested plugins or select custom plugins

# Recommended plugins for DevOps:
# - Git plugin
# - Pipeline plugin
# - Docker plugin
# - Blue Ocean
# - Credentials Binding
# - Email Extension
# - Build Timeout
# - Timestamper
```

### Step 2: Global Configuration

#### 2.1 Configure Global Tools
```bash
# Navigate to: Manage Jenkins → Global Tool Configuration

# Git Configuration
Name: Default
Path to Git executable: git (or /usr/bin/git)

# JDK Configuration
Name: JDK-11
JAVA_HOME: /usr/lib/jvm/java-11-openjdk-amd64

# Maven Configuration (if needed)
Name: Maven-3.8
Install automatically: Yes
Version: 3.8.6

# Node.js Configuration
Name: NodeJS-18
Install automatically: Yes
Version: 18.17.0

# Docker Configuration
Name: Docker
Install automatically: Yes
Docker version: latest
```

#### 2.2 Configure System Settings
```bash
# Navigate to: Manage Jenkins → Configure System

# Jenkins Location
Jenkins URL: http://your-jenkins-server:8080
System Admin e-mail address: admin@yourcompany.com

# GitHub Configuration
GitHub Server API URL: https://api.github.com
Credentials: Add GitHub personal access token

# Email Configuration
SMTP server: smtp.gmail.com
Use SMTP Authentication: Yes
User Name: your-email@gmail.com
Password: your-app-password
Use SSL: Yes
SMTP Port: 465
```

## 🔧 Creating Your First Jenkins Pipeline

### Pipeline 1: Simple Hello World

#### 1.1 Declarative Pipeline
```groovy
pipeline {
    agent any
    
    stages {
        stage('Hello') {
            steps {
                echo 'Hello World from Jenkins!'
                sh 'date'
                sh 'whoami'
            }
        }
        
        stage('Environment Info') {
            steps {
                sh 'java -version'
                sh 'docker --version'
                sh 'git --version'
            }
        }
        
        stage('System Info') {
            steps {
                sh 'uname -a'
                sh 'df -h'
                sh 'free -h'
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
        }
        success {
            echo 'Pipeline succeeded!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
```

### Pipeline 2: Node.js Application CI/CD

#### 2.1 Complete Node.js Pipeline
```groovy
pipeline {
    agent any
    
    environment {
        NODE_VERSION = '18'
        APP_NAME = 'nodejs-app'
        DOCKER_IMAGE = "${APP_NAME}:${BUILD_NUMBER}"
        DOCKER_REGISTRY = 'your-registry.com'
    }
    
    tools {
        nodejs "${NODE_VERSION}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'ls -la'
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
                sh 'npm list --depth=0'
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('Lint') {
                    steps {
                        sh 'npm run lint'
                    }
                    post {
                        always {
                            publishHTML([
                                allowMissing: false,
                                alwaysLinkToLastBuild: true,
                                keepAll: true,
                                reportDir: 'reports',
                                reportFiles: 'eslint-report.html',
                                reportName: 'ESLint Report'
                            ])
                        }
                    }
                }
                
                stage('Security Scan') {
                    steps {
                        sh 'npm audit --audit-level moderate'
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
            }
            post {
                always {
                    publishTestResults testResultsPattern: 'test-results.xml'
                    publishCoverage adapters: [
                        istanbulCoberturaAdapter('coverage/cobertura-coverage.xml')
                    ], sourceFileResolver: sourceFiles('STORE_LAST_BUILD')
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build'
                archiveArtifacts artifacts: 'dist/**/*', fingerprint: true
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    def image = docker.build("${DOCKER_IMAGE}")
                    docker.withRegistry("https://${DOCKER_REGISTRY}", 'docker-registry-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'develop'
            }
            steps {
                sh '''
                    docker stop ${APP_NAME}-staging || true
                    docker rm ${APP_NAME}-staging || true
                    docker run -d --name ${APP_NAME}-staging -p 3001:3000 ${DOCKER_IMAGE}
                '''
            }
        }
        
        stage('Integration Tests') {
            when {
                branch 'develop'
            }
            steps {
                sh 'npm run test:integration'
            }
        }
        
        stage('Deploy to Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to production?', ok: 'Deploy'
                sh '''
                    docker stop ${APP_NAME}-prod || true
                    docker rm ${APP_NAME}-prod || true
                    docker run -d --name ${APP_NAME}-prod -p 3000:3000 ${DOCKER_IMAGE}
                '''
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        success {
            emailext (
                subject: "✅ Pipeline Success: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "The pipeline ${env.JOB_NAME} build ${env.BUILD_NUMBER} completed successfully.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
        failure {
            emailext (
                subject: "❌ Pipeline Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "The pipeline ${env.JOB_NAME} build ${env.BUILD_NUMBER} failed. Please check the logs.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
```

### Pipeline 3: Multi-Branch Pipeline

#### 3.1 Jenkinsfile for Multi-Branch
```groovy
pipeline {
    agent any
    
    parameters {
        choice(
            name: 'ENVIRONMENT',
            choices: ['dev', 'staging', 'prod'],
            description: 'Target environment'
        )
        booleanParam(
            name: 'SKIP_TESTS',
            defaultValue: false,
            description: 'Skip test execution'
        )
    }
    
    environment {
        APP_VERSION = "${env.BUILD_NUMBER}"
        DEPLOY_ENV = "${params.ENVIRONMENT}"
    }
    
    stages {
        stage('Preparation') {
            steps {
                script {
                    // Set environment-specific variables
                    switch(params.ENVIRONMENT) {
                        case 'dev':
                            env.PORT = '3001'
                            env.DB_HOST = 'dev-db.example.com'
                            break
                        case 'staging':
                            env.PORT = '3002'
                            env.DB_HOST = 'staging-db.example.com'
                            break
                        case 'prod':
                            env.PORT = '3000'
                            env.DB_HOST = 'prod-db.example.com'
                            break
                    }
                }
                echo "Deploying to ${params.ENVIRONMENT} environment"
                echo "Port: ${env.PORT}, DB Host: ${env.DB_HOST}"
            }
        }
        
        stage('Build') {
            steps {
                sh 'echo "Building application version ${APP_VERSION}"'
                sh 'npm ci'
                sh 'npm run build'
            }
        }
        
        stage('Test') {
            when {
                not { params.SKIP_TESTS }
            }
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh 'npm run test:unit'
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
                        timeout(time: 5, unit: 'MINUTES') {
                            input message: 'Approve production deployment?', 
                                  submitter: 'admin,deploy-team'
                        }
                    }
                }
                
                sh '''
                    echo "Deploying to ${DEPLOY_ENV}"
                    docker build -t myapp:${APP_VERSION} .
                    docker run -d --name myapp-${DEPLOY_ENV} -p ${PORT}:3000 \
                        -e NODE_ENV=${DEPLOY_ENV} \
                        -e DB_HOST=${DB_HOST} \
                        myapp:${APP_VERSION}
                '''
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    def maxRetries = 10
                    def retryCount = 0
                    def healthCheckPassed = false
                    
                    while (retryCount < maxRetries && !healthCheckPassed) {
                        try {
                            sh "curl -f http://localhost:${env.PORT}/health"
                            healthCheckPassed = true
                            echo "Health check passed!"
                        } catch (Exception e) {
                            retryCount++
                            echo "Health check failed, retry ${retryCount}/${maxRetries}"
                            sleep(10)
                        }
                    }
                    
                    if (!healthCheckPassed) {
                        error("Health check failed after ${maxRetries} attempts")
                    }
                }
            }
        }
    }
    
    post {
        always {
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'reports',
                reportFiles: 'index.html',
                reportName: 'Test Report'
            ])
        }
        
        success {
            slackSend(
                channel: '#deployments',
                color: 'good',
                message: "✅ Deployment successful: ${env.JOB_NAME} - ${env.BUILD_NUMBER} to ${params.ENVIRONMENT}"
            )
        }
        
        failure {
            slackSend(
                channel: '#deployments',
                color: 'danger',
                message: "❌ Deployment failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER} to ${params.ENVIRONMENT}"
            )
        }
    }
}
```

## 🔐 Jenkins Security Configuration

### Security Best Practices

#### 1.1 User Management
```bash
# Navigate to: Manage Jenkins → Configure Global Security

# Security Realm: Jenkins' own user database
# Authorization: Matrix-based security

# Create users with appropriate permissions:
# - Admin: Overall/Administer
# - Developer: Job/Build, Job/Read, Job/Workspace
# - Viewer: Overall/Read, Job/Read
```

#### 1.2 Credentials Management
```bash
# Navigate to: Manage Jenkins → Manage Credentials

# Add credentials for:
# - GitHub/GitLab access tokens
# - Docker registry credentials
# - SSH keys for deployment
# - Database passwords
# - API keys

# Use credential binding in pipelines:
withCredentials([
    string(credentialsId: 'api-key', variable: 'API_KEY'),
    usernamePassword(credentialsId: 'db-creds', usernameVariable: 'DB_USER', passwordVariable: 'DB_PASS')
]) {
    sh 'echo "Using API key: $API_KEY"'
    sh 'echo "Database user: $DB_USER"'
}
```

#### 1.3 Agent Security
```bash
# Configure agent security
# Navigate to: Manage Jenkins → Configure Global Security

# Agent protocols: Enable only JNLP4-connect
# TCP port for inbound agents: Fixed port (50000)
# Agent → Master Access Control: Enable
```

## 📚 Interview Questions & Answers

### Fresher Level (1-10)

**Q1: What is Jenkins and why is it used?**
A: Jenkins is an open-source automation server used for CI/CD. It automates building, testing, and deploying applications, enabling faster and more reliable software delivery with reduced manual errors.

**Q2: What is the difference between Freestyle and Pipeline jobs?**
A: Freestyle jobs use GUI configuration with limited flexibility. Pipeline jobs use code (Jenkinsfile) for configuration, offering better version control, reusability, and complex workflow support.

**Q3: What is a Jenkinsfile?**
A: A Jenkinsfile is a text file containing Jenkins Pipeline definition written in Groovy. It can be stored in source control alongside application code, enabling "Pipeline as Code."

**Q4: What are Jenkins agents/slaves?**
A: Jenkins agents are separate machines that execute build jobs. They distribute workload from the master, enable parallel execution, and provide different environments for builds.

**Q5: How do you secure Jenkins?**
A: Enable authentication, use role-based access control, secure agent connections, manage credentials properly, keep Jenkins updated, use HTTPS, and implement network security.

### Intermediate Level (6-15)

**Q6: What are the different types of Jenkins pipelines?**
A: Declarative Pipeline (structured, easier syntax) and Scripted Pipeline (more flexible, Groovy-based). Declarative is recommended for most use cases due to better error handling and syntax validation.

**Q7: How do you handle secrets in Jenkins pipelines?**
A: Use Jenkins Credentials Plugin, environment variables, credential binding, external secret management tools (HashiCorp Vault), and avoid hardcoding secrets in pipeline code.

**Q8: What is Blue Ocean in Jenkins?**
A: Blue Ocean is a modern UI for Jenkins that provides better visualization of pipelines, easier pipeline creation, and improved user experience for both developers and non-technical users.

**Q9: How do you implement parallel execution in Jenkins pipelines?**
A: Use `parallel` blocks in declarative pipelines or `parallel` step in scripted pipelines. This allows multiple stages or steps to run simultaneously, reducing overall build time.

**Q10: What are Jenkins shared libraries?**
A: Shared libraries allow sharing common pipeline code across multiple projects. They promote code reuse, standardization, and easier maintenance of pipeline logic across an organization.

## 🔑 Key Takeaways

- **CI/CD Benefits**: Automation improves speed, quality, and reliability
- **Pipeline as Code**: Use Jenkinsfiles for version-controlled pipeline definitions
- **Security First**: Implement proper authentication, authorization, and credential management
- **Scalability**: Use agents for distributed builds and parallel execution
- **Integration**: Connect with Git, Docker, cloud services, and notification systems
- **Monitoring**: Implement proper logging, notifications, and health checks

## 🚀 Next Steps

- Day 30: Advanced Jenkins Pipelines and Multi-stage Deployments
- Day 31: Jenkins Project - Two-tier Application CI/CD
- Integration with Kubernetes and cloud platforms
- Advanced pipeline patterns and best practices

---

**Hands-on Completed:** ✅ Jenkins Installation, Pipeline Creation, Security Configuration  
**Duration:** 4-5 hours  
**Difficulty:** Intermediate