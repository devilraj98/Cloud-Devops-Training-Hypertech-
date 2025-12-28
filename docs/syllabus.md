# AWS DevOps Batch 4 - Complete Syllabus

## 🎯 Course Objectives
By the end of this course, students will be able to:
- Design and implement scalable cloud infrastructure on AWS
- Automate deployment processes using CI/CD pipelines
- Implement Infrastructure as Code using Terraform and CloudFormation
- Set up monitoring, logging, and alerting systems
- Implement security best practices in cloud environments
- Troubleshoot and optimize cloud applications
- Master containerization and orchestration with Docker and Kubernetes
- Implement comprehensive CI/CD pipelines with modern DevOps tools

## 📅 Course Duration
- **Total Duration:** 14 Weeks
- **Sessions per Week:** 3 sessions (Theory + hands-on)
- **Session Duration:** 2 hours each
- **Total Hours:** 80 hours

## 📚 Module Breakdown

### **Module 1: AWS Fundamentals & Identity Management**

#### Cloud Fundamentals
- **Day 1:** Introduction to Cloud Computing & AWS
  - What is Cloud and why this evolve
  - Use cases why Cloud computing Evolve
  - What is On-Premise and what is the pro and cons of this
  - Cloud computing concepts and benefits
  - Why most of the companies are migrating to Cloud 
  - Multiple Cloud Providers Exists in market 
  - Why we choose AWS
  - AWS global infrastructure (Regions, AZs, Edge Locations)
  - AWS pricing models 
  - AWS management console and service overview

- **Day 2:** AWS Account Setup & Best Practices
  - Creating AWS account with proper security
  - Multi-factor authentication (MFA) setup
  - Billing alerts and cost management 
  - Launching EC2 instances with different configurations
  - Configuring security groups and networking
  - Setting up auto scaling groups
  - Implementing load balancing


- **Day 3:** Hands-on Lab - AWS Console Navigation
  - EC2 instance types and families (General Purpose, Compute Optimized, Memory Optimized, etc.)
  - AMIs and instance lifecycle management
  - Security groups and key pairs
  - Instance metadata service (IMDSv2)
  - EC2 placement groups and networking
  - Nginx installation 
  - Connecting using SSH and console with and without Pem file
  - Website Hosting Using Nginx 

#### Webserver and Multiple OS Working model
- **Day 4:** Webserver installation on multiple OS of AWS
  - Webserver installation on Amazon Linux
  - Website hosting using Nginx 
  - Edit Default configuration of Nginx
  - Common Linux commands 
  - Webserver installation on Suse Linux 
  - Configure the default config of Nginx
  - Website hosting Suse linux

- **Day 5:** Webserver and Multiple OS Working model 
  - About RedHat
  - Security Breach sol on RedHat
  - Webserver installation on Redhat and Windows
  - Website hosting using Webserver on Redhat
  - Windows VM connection using RDP client
  - Webserver config on Windows
  
- **Day 6:** Mobaxterm and putty software
  - EC2 Practical using Mobaxterm 
  - EC2 Practical with Putty

### **Module 2: Storage & Database Services**

#### Week 3-4: Storage Services
- **Day 7:** Block & File Storage (EBS, EFS, Storage Gateway)
  - Amazon EBS volumes and snapshots
  - EFS file systems and access points
  - Storage Gateway for hybrid storage
  - Storage optimization and performance tuning
  - Backup and disaster recovery strategies
  - Practical of Vol creation 
  - Attaching vol to EC2 

- **Day 8:** Volumes and their Types in depth Practical
  - Volume creation and attach with the EC2
  - Storing data in Vol
  - Data recovery in EBS 
  
- **Day 9:** Snapshot
  - About Snapshot 
  - Practical of creating snapshot in different-different AZs
  - Vol creation from Snapshot

- **Day 10:** S3 (Simple Storage Service)
  - S3 buckets and objects management
  - Storage classes (Standard, IA, Glacier, etc.) and lifecycle policies
  - Creating S3 buckets with proper security and encryption
  - Static Website hosting using S3
  - Implementing lifecycle policies and versioning 

- **Day 11:**Database Services
  - Relational Databases (RDS, Aurora)
  - Amazon RDS and supported database engines
  - Aurora database engine and architecture
  - Multi-AZ deployments and failover
  - Read replicas and read scaling
  - RDS backup and recovery strategies
  - Practical of RDS creation using EC2
  - Practical of MySQL database on EC2 and on local

### **Module 3: Networking & Security**

#### Networking Fundamentals
- **Day 12:**Amazon VPC & Networking
  - VPC components and architecture (CIDR blocks, subnets)
  - Subnets (public, private, database) and route tables
  - Internet Gateway and NAT Gateway
  - VPC peering and Transit Gateway
  - VPC endpoints for AWS services

- **Day 13:** Hands-on Lab - Network Setup
  - Creating custom VPC with proper subnet design
  - Setting up public and private subnets with route tables
  - Configuring NAT Gateway and security groups
  - Setting up VPC endpoints
  - Connecting Private EC2 through Public EC2

- **Day 14:** Hands-on Lab - VPC Peering
  - VPC peering using EC2

- **Day 15:** Project 
  - Creation Of project from scratch 
  - Security practices 
  - Creating IAM users and groups

- **Day 16:** EC2 Advanced Features
  - Auto Scaling groups and policies
  - Application Load Balancer (ALB) and Network Load Balancer (NLB)
  - Launching EC2 instances with different configurations
  - Configuring security groups and networking
  - Setting up auto scaling groups
  - Practical use case of Auto scaling Working model.

- **Day 18:** Load Balancers 
  - Basics of Load Balancers
  - Use cases
  - Practical of Classic Load balancer

- **Day 19:** Application Load Balancer
  - Basics
  - Creation of Application load Balancer
  - Practical use case of ALB using multiple targets of EC2

- **Day 20:** Elastic IP
  - What and Why
  - Use Case of Elastic IP with EC2 
  
 #### Serverless Computing & Containers
- **Day 21:** AWS Lambda & Serverless
  - Lambda functions and runtime environments
  - Event sources and triggers (S3, DynamoDB, API Gateway, etc.)
  - Cold starts and optimization strategies
  - Lambda layers and extensions
  - Lambda monitoring and debugging

### **Module 4: DevOps Tools & CI/CD (Weeks 8-14)**

#### Week 9: Version Control & Collaboration
- **Day 22:** Git & GitHub/GitLab
  - Git fundamentals and version control concepts
  - Branching strategies (GitFlow, GitHub Flow)
  - Code review processes and pull requests
  - GitHub Actions and GitLab CI/CD basics
  - Repository management and collaboration

- **Day 23:** Code Quality & Testing
  - Static code analysis tools (SonarQube, ESLint, Pylint)
  - Unit and integration testing frameworks
  - Code coverage and quality gates
  - SonarQube integration and quality metrics
  - Automated testing strategies

- **Day 24:** Hands-on Lab - Git Workflow
  - Setting up Git repository with proper structure
  - Implementing branching strategy and workflows
  - Code review process and collaboration
  - Setting up GitHub Actions workflows

- **Day 25:** Docker 
  - Docker Basics 
  - Docker setup on both Local and EC2
  - Docker Commands
  - Docker Security
  
- **Day 26:** Hands-on Lab - Docker
  - Docker UseCase
  - Docker Images
  - Docker Containers
  - Hosting website using Docker and Nginx

- **Day 27:** Two Tier Project using Docker
  - Project setup for Frontend and backend using docker Network(Mysql and Node) 
  - Docker Network

- **Day 28:** Docker Vol and Docker Composer
  - Project with Docker Volume
  - Docker Compose


- **Day 29:** CICD - Jenkins
  - Jenkins installation and configuration
  - Pipeline creation and management
  - Build automation and deployment
  
- **Day 30:** CICD - Jenkins
  - Multi-stage pipelines
  - Integration with Git repositories
  - Automated testing in pipelines

#### Week 10-11: Infrastructure as Code
- **Day 31:** CICD - Jenkins - Project
  - Two tier project deployment using Docker, Github, Webhook in CICD pipeline on EC2
  - Jenkins shared libraries and pipeline optimization
  - Blue-green deployment strategies
  - Pipeline monitoring and troubleshooting
 
- **Day 32:** IAC-Terraform
  - Terraform basics and infrastructure as code
  - Resource provisioning and state management
  - Terraform modules and best practices
  - HCL syntax and configuration files
  - Terraform providers and resources

- **Day 33:** IAC-Terraform Advanced
  - Terraform with AWS services integration
  - Remote state management and team collaboration
  - Terraform Cloud and automation
  - Terraform workspaces and environments
  - State locking and backend configuration

- **Day 34:** IAC-Terraform Project
  - End-to-end infrastructure deployment
  - Multi-environment setup (dev, staging, prod)
  - Infrastructure testing and validation
  - Terraform modules for reusability
  - Cost optimization and resource tagging

#### Week 12-13: Container Orchestration
- **Day 35:** Container Orchestration - Kubernetes
  - Kubernetes architecture and components
  - Pods, services, and deployments
  - Basic kubectl commands and cluster management
  - ConfigMaps and Secrets management
  - Kubernetes networking concepts

- **Day 36:** Minikube setup on local
  - Minikube installation and configuration
  - React project deployment using Docker hub
  - Auto-scaling and resource management in local
  - Local development workflow with Kubernetes
  - Debugging and troubleshooting pods
  
- **Day 37:** Container Orchestration - Kubeadm installation
  - Kubeadm setup on Cloud (multi-node cluster)
  - Deployment of web-app on Kubeadm using YAML
  - Cluster networking with CNI plugins
  - Node management and cluster scaling
  - RBAC and security configurations

- **Day 38:** Container Orchestration - EKS
  - Setup EKS on AWS and deployment of web-app using Helm
  - Auto-scaling and resource management
  - Production deployment strategies
  - EKS node groups and Fargate profiles
  - AWS Load Balancer Controller integration

- **Day 39:** Container Orchestration - Kubernetes Advanced
  - Kubernetes monitoring and logging
  - Persistent volumes and storage classes
  - Ingress controllers and traffic management
  - Helm charts and package management
  - GitOps with ArgoCD/Flux
 

### **Module 5: Monitoring & Logging (Week 14)**

#### Monitoring & Observability
- **Day 40:** CloudWatch Fundamentals
  - CloudWatch metrics and alarms
  - Custom metrics and dashboards
  - CloudWatch logs and log groups
  - CloudWatch insights and log queries
  - CloudWatch events and event rules
  - SNS integration for alerting
  - Cost monitoring and billing alerts

- **Day 41:** Advanced Monitoring - Grafana & Prometheus
  - Grafana installation and configuration
  - Dashboard creation and visualization
  - Data source integration (CloudWatch, Prometheus)
  - Alerting and notification setup
  - Performance monitoring best practices
  - Prometheus metrics collection
  - Custom dashboards for applications

### **Module 6: Final Projects & Certification Prep (Week 15-16)**

#### Week 15: Capstone Projects
- **Day 42:** Project Planning & Architecture
  - Project requirements analysis and scope definition
  - Architecture design and system planning
  - Implementation planning and timeline
  - Team collaboration and project management
  - Risk assessment and mitigation strategies
  - Technology stack selection
  - Security and compliance considerations

- **Day 43:** Project Implementation & Deployment
  - Building complete DevOps pipeline with CI/CD
  - Infrastructure deployment using Terraform/CloudFormation
  - Application deployment on EKS using Helm charts and configuration
  - Testing and validation (unit, integration, performance)
  - Monitoring and alerting setup
  - Documentation and runbooks
  - Performance optimization and scaling

#### Week 16: Final Assessment & Career Preparation
- **Day 44:** Project Presentation & Certification Prep
  - Project demonstrations and presentations
  - AWS certification overview and exam paths
  - Exam preparation strategies and study plans
  - Course wrap-up and next steps
  - Industry best practices and career guidance
  - Resume building for DevOps roles
  - Interview preparation and technical discussions


  ## 🎓 Certification Path

### **Recommended AWS Certifications**
1. **AWS Certified Cloud Practitioner** (Entry-level)
2. **AWS Certified Solutions Architect - Associate**
3. **AWS Certified Developer - Associate**
4. **AWS Certified DevOps Engineer - Professional**

### **Additional Certifications**
- **Terraform Associate**
- **Docker Certified Associate**
- **Kubernetes Administrator (CKA)**

## 📚 Additional Resources

### **Books**
- "AWS Certified Solutions Architect Study Guide" by Ben Piper
- "Terraform: Up & Running" by Yevgeniy Brikman
- "The Phoenix Project" by Gene Kim

### **Online Resources**
- AWS Documentation and Whitepapers
- AWS Well-Architected Framework
- HashiCorp Learn
- Kubernetes Documentation

### **Practice Platforms**
- AWS Free Tier
- Terraform Cloud (Free tier)
- GitHub Student Developer Pack
- AWS Skill Builder

---


**Note:** This syllabus is subject to updates based on AWS service changes and industry best practices.
