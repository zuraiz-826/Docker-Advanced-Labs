Lab 66: Docker for Automation - Automating Docker Container Builds with Jenkins
Lab Objectives
By the end of this lab, students will be able to:

Install and configure Docker within Jenkins environment
Set up Jenkins pipelines to automate Docker image builds
Push Docker images to Docker Hub registry
Integrate static code analysis using SonarQube in CI/CD pipelines
Deploy Docker containers to cloud services
Implement automated testing within Docker-based CI/CD workflows
Understand the complete DevOps workflow using Docker and Jenkins
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with Jenkins fundamentals
Basic knowledge of Linux command line
Understanding of CI/CD concepts
GitHub account for source code management
Docker Hub account for image registry
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment - no need to build your own VM or install software locally.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker pre-installed
Jenkins server ready to configure
Git and other development tools
Network access to Docker Hub and cloud services
Task 1: Install Docker inside Jenkins and Configure Docker Plugin
Subtask 1.1: Verify Docker Installation
First, let's verify that Docker is properly installed and running on your system.

# Check Docker version
docker --version

# Check Docker service status
sudo systemctl status docker

# Test Docker installation
docker run hello-world
Subtask 1.2: Install Jenkins Docker Plugin
Access Jenkins web interface by opening your browser and navigating to http://localhost:8080

Log in with your Jenkins credentials

Navigate to Manage Jenkins → Manage Plugins

Click on the Available tab and search for "Docker"

Install the following plugins:

Docker Pipeline
Docker plugin
Docker Commons Plugin
Restart Jenkins after installation:

sudo systemctl restart jenkins
Subtask 1.3: Configure Docker in Jenkins
Go to Manage Jenkins → Global Tool Configuration

Scroll down to Docker section

Click Add Docker and configure:

Name: docker
Installation root: /usr/bin/docker
Check Install automatically if needed
Save the configuration

Subtask 1.4: Add Jenkins User to Docker Group
# Add jenkins user to docker group
sudo usermod -aG docker jenkins

# Restart Jenkins service
sudo systemctl restart jenkins

# Verify jenkins user can run docker commands
sudo -u jenkins docker ps
Task 2: Set up Jenkins Pipeline to Build Docker Images and Push to Docker Hub
Subtask 2.1: Create Sample Application
Create a simple Node.js application for our pipeline:

# Create project directory
mkdir ~/docker-jenkins-demo
cd ~/docker-jenkins-demo

# Create package.json
cat > package.json << EOF
{
  "name": "docker-jenkins-demo",
  "version": "1.0.0",
  "description": "Demo app for Docker Jenkins pipeline",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"Running tests...\" && exit 0"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
EOF

# Create app.js
cat > app.js << EOF
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Docker Jenkins Pipeline!',
    timestamp: new Date().toISOString(),
    version: '1.0.0'
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(port, () => {
  console.log(`App listening at http://localhost:${port}`);
});
EOF

# Create Dockerfile
cat > Dockerfile << EOF
FROM node:16-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
EOF
Subtask 2.2: Create Jenkinsfile
Create a comprehensive Jenkins pipeline file:

cat > Jenkinsfile << EOF
pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        IMAGE_NAME = 'your-dockerhub-username/docker-jenkins-demo'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Application') {
            steps {
                script {
                    echo 'Building Node.js application...'
                    sh 'npm install'
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                script {
                    echo 'Running application tests...'
                    sh 'npm test'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    def image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                    docker.build("${IMAGE_NAME}:latest")
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo 'Pushing image to Docker Hub...'
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        def image = docker.image("${IMAGE_NAME}:${IMAGE_TAG}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Clean Up') {
            steps {
                script {
                    echo 'Cleaning up local images...'
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
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
EOF
Subtask 2.3: Configure Docker Hub Credentials in Jenkins
In Jenkins, go to Manage Jenkins → Manage Credentials

Click on (global) domain

Click Add Credentials

Configure the credential:

Kind: Username with password
Username: Your Docker Hub username
Password: Your Docker Hub password or access token
ID: docker-hub-credentials
Description: Docker Hub Credentials
Click OK to save

Subtask 2.4: Create Jenkins Pipeline Job
In Jenkins dashboard, click New Item

Enter job name: docker-jenkins-pipeline

Select Pipeline and click OK

In the pipeline configuration:

Description: Docker Jenkins automation pipeline
Pipeline Definition: Pipeline script from SCM
SCM: Git
Repository URL: Your Git repository URL
Script Path: Jenkinsfile
Save the configuration

Task 3: Integrate Static Code Analysis with SonarQube
Subtask 3.1: Set up SonarQube with Docker
# Create SonarQube directory
mkdir ~/sonarqube-data
chmod 777 ~/sonarqube-data

# Run SonarQube container
docker run -d \
  --name sonarqube \
  -p 9000:9000 \
  -v ~/sonarqube-data:/opt/sonarqube/data \
  sonarqube:community

# Wait for SonarQube to start
echo "Waiting for SonarQube to start..."
sleep 60

# Check SonarQube status
curl -f http://localhost:9000/api/system/status
Subtask 3.2: Configure SonarQube
Access SonarQube at http://localhost:9000

Login with default credentials:

Username: admin
Password: admin
Change the default password when prompted

Create a new project:

Project key: docker-jenkins-demo
Display name: Docker Jenkins Demo
Generate a token for Jenkins integration:

Go to My Account → Security
Generate token named jenkins-integration
Copy the token for later use
Subtask 3.3: Install SonarQube Scanner in Jenkins
Go to Manage Jenkins → Manage Plugins

Install SonarQube Scanner plugin

Go to Manage Jenkins → Global Tool Configuration

Add SonarQube Scanner:

Name: sonar-scanner
Check Install automatically
Go to Manage Jenkins → Configure System

Add SonarQube server:

Name: sonarqube
Server URL: http://localhost:9000
Server authentication token: Add credential with SonarQube token
Subtask 3.4: Update Jenkinsfile with SonarQube Integration
cat > Jenkinsfile << EOF
pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        SONAR_TOKEN = credentials('sonarqube-token')
        IMAGE_NAME = 'your-dockerhub-username/docker-jenkins-demo'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Application') {
            steps {
                script {
                    echo 'Building Node.js application...'
                    sh 'npm install'
                }
            }
        }
        
        stage('Static Code Analysis') {
            steps {
                script {
                    echo 'Running SonarQube analysis...'
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            sonar-scanner \
                            -Dsonar.projectKey=docker-jenkins-demo \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://localhost:9000 \
                            -Dsonar.login=${SONAR_TOKEN}
                        '''
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    echo 'Waiting for Quality Gate...'
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                script {
                    echo 'Running application tests...'
                    sh 'npm test'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    def image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                    docker.build("${IMAGE_NAME}:latest")
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    echo 'Scanning Docker image for vulnerabilities...'
                    sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo 'Pushing image to Docker Hub...'
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        def image = docker.image("${IMAGE_NAME}:${IMAGE_TAG}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Clean Up') {
            steps {
                script {
                    echo 'Cleaning up local images...'
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
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
EOF
Task 4: Configure Jenkins to Deploy Docker Containers to Cloud Service
Subtask 4.1: Set up Docker Compose for Deployment
Create a Docker Compose file for deployment:

cat > docker-compose.yml << EOF
version: '3.8'

services:
  app:
    image: your-dockerhub-username/docker-jenkins-demo:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    restart: unless-stopped

networks:
  default:
    driver: bridge
EOF

# Create nginx configuration
cat > nginx.conf << EOF
events {
    worker_connections 1024;
}

http {
    upstream app {
        server app:3000;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://app;
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }
        
        location /health {
            proxy_pass http://app/health;
        }
    }
}
EOF
Subtask 4.2: Create Deployment Script
cat > deploy.sh << EOF
#!/bin/bash

set -e

echo "Starting deployment process..."

# Pull latest images
docker-compose pull

# Stop existing containers
docker-compose down

# Start new containers
docker-compose up -d

# Wait for services to be healthy
echo "Waiting for services to be ready..."
sleep 30

# Check application health
if curl -f http://localhost/health; then
    echo "Deployment successful!"
    echo "Application is running at http://localhost"
else
    echo "Deployment failed - health check failed"
    exit 1
fi

# Clean up unused images
docker image prune -f

echo "Deployment completed successfully!"
EOF

chmod +x deploy.sh
Subtask 4.3: Update Jenkinsfile with Deployment Stage
cat > Jenkinsfile << EOF
pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        SONAR_TOKEN = credentials('sonarqube-token')
        IMAGE_NAME = 'your-dockerhub-username/docker-jenkins-demo'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Application') {
            steps {
                script {
                    echo 'Building Node.js application...'
                    sh 'npm install'
                }
            }
        }
        
        stage('Static Code Analysis') {
            steps {
                script {
                    echo 'Running SonarQube analysis...'
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            sonar-scanner \
                            -Dsonar.projectKey=docker-jenkins-demo \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://localhost:9000 \
                            -Dsonar.login=${SONAR_TOKEN}
                        '''
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    echo 'Waiting for Quality Gate...'
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        
        stage('Run Tests') {
            steps {
                script {
                    echo 'Running application tests...'
                    sh 'npm test'
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    def image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                    docker.build("${IMAGE_NAME}:latest")
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    echo 'Scanning Docker image for vulnerabilities...'
                    sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo 'Pushing image to Docker Hub...'
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        def image = docker.image("${IMAGE_NAME}:${IMAGE_TAG}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                script {
                    echo 'Deploying to staging environment...'
                    sh './deploy.sh'
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                script {
                    echo 'Running integration tests...'
                    sh '''
                        # Wait for application to be ready
                        sleep 10
                        
                        # Test application endpoints
                        curl -f http://localhost/health
                        curl -f http://localhost/ | grep "Hello from Docker"
                        
                        echo "Integration tests passed!"
                    '''
                }
            }
        }
        
        stage('Clean Up') {
            steps {
                script {
                    echo 'Cleaning up local images...'
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
        }
        success {
            echo 'Pipeline succeeded!'
            echo 'Application deployed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
            sh 'docker-compose logs || true'
        }
    }
}
EOF
Task 5: Automate Tests in CI/CD Pipeline with Docker Containers
Subtask 5.1: Create Test Suite
Create comprehensive test files:

# Create test directory
mkdir tests

# Create unit tests
cat > tests/unit.test.js << EOF
const request = require('supertest');
const express = require('express');

// Mock the app
const app = express();
app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Docker Jenkins Pipeline!',
    timestamp: new Date().toISOString(),
    version: '1.0.0'
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

describe('Application Tests', () => {
  test('GET / should return welcome message', async () => {
    const response = await request(app).get('/');
    expect(response.status).toBe(200);
    expect(response.body.message).toBe('Hello from Docker Jenkins Pipeline!');
    expect(response.body.version).toBe('1.0.0');
  });

  test('GET /health should return healthy status', async () => {
    const response = await request(app).get('/health');
    expect(response.status).toBe(200);
    expect(response.body.status).toBe('healthy');
  });
});
EOF

# Create integration tests
cat > tests/integration.test.js << EOF
const axios = require('axios');

const BASE_URL = process.env.TEST_URL || 'http://localhost:3000';

describe('Integration Tests', () => {
  test('Application should be accessible', async () => {
    const response = await axios.get(BASE_URL);
    expect(response.status).toBe(200);
    expect(response.data.message).toContain('Hello from Docker');
  });

  test('Health endpoint should work', async () => {
    const response = await axios.get(`${BASE_URL}/health`);
    expect(response.status).toBe(200);
    expect(response.data.status).toBe('healthy');
  });
});
EOF

# Update package.json with test dependencies
cat > package.json << EOF
{
  "name": "docker-jenkins-demo",
  "version": "1.0.0",
  "description": "Demo app for Docker Jenkins pipeline",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "jest --testTimeout=10000",
    "test:unit": "jest tests/unit.test.js",
    "test:integration": "jest tests/integration.test.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "supertest": "^6.3.0",
    "axios": "^1.0.0"
  },
  "jest": {
    "testEnvironment": "node"
  }
}
EOF
Subtask 5.2: Create Test Docker Containers
# Create test Dockerfile
cat > Dockerfile.test << EOF
FROM node:16-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm install

# Copy source code and tests
COPY . .

# Run tests
CMD ["npm", "test"]
EOF

# Create Docker Compose for testing
cat > docker-compose.test.yml << EOF
version: '3.8'

services:
  app-test:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=test
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s

  unit-tests:
    build:
      context: .
      dockerfile: Dockerfile.test
    command: npm run test:unit
    volumes:
      - .:/app
      - /app/node_modules

  integration-tests:
    build:
      context: .
      dockerfile: Dockerfile.test
    command: npm run test:integration
    environment:
      - TEST_URL=http://app-test:3000
    depends_on:
      app-test:
        condition: service_healthy
    volumes:
      - .:/app
      - /app/node_modules
EOF
Subtask 5.3: Final Jenkinsfile with Complete Testing
cat > Jenkinsfile << EOF
pipeline {
    agent any
    
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        SONAR_TOKEN = credentials('sonarqube-token')
        IMAGE_NAME = 'your-dockerhub-username/docker-jenkins-demo'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Application') {
            steps {
                script {
                    echo 'Building Node.js application...'
                    sh 'npm install'
                }
            }
        }
        
        stage('Unit Tests') {
            steps {
                script {
                    echo 'Running unit tests in Docker container...'
                    sh 'docker-compose -f docker-compose.test.yml run --rm unit-tests'
                }
            }
        }
        
        stage('Static Code Analysis') {
            steps {
                script {
                    echo 'Running SonarQube analysis...'
                    withSonarQubeEnv('sonarqube') {
                        sh '''
                            sonar-scanner \
                            -Dsonar.projectKey=docker-jenkins-demo \
                            -Dsonar.sources=. \
                            -Dsonar.exclusions=node_modules/**,tests/** \
                            -Dsonar.host.url=http://localhost:9000 \
                            -Dsonar.login=${SONAR_TOKEN}
                        '''
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    echo 'Waiting for Quality Gate...'
                    timeout(time: 5, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    echo 'Building Docker image...'
                    def image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                    docker.build("${IMAGE_NAME}:latest")
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    echo 'Scanning Docker image for vulnerabilities...'
                    sh "docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image ${IMAGE_NAME}:${IMAGE_TAG}"
                }
            }
        }
        
        stage('Integration Tests') {
            steps {
                script {
                    echo 'Running integration tests...'
                    sh 'docker-compose -f docker-compose.test.yml up -d app-test'
                    sh 'sleep 30' // Wait for app to be ready
                    sh 'docker-compose -f docker-compose.test.yml run --rm integration-tests'
                }
            }
            post {
                always {
                    sh 'docker-compose -f docker-compose.test.yml down || true'
                }
            }
        }
        
        stage('Push to Docker Hub') {
            steps {
                script {
                    echo 'Pushing image to Docker Hub...'
                    docker.withRegistry('https://registry.hub.docker.com', 'docker-hub-credentials') {
                        def image = docker.image("${IMAGE_NAME}:${IMAGE_TAG}")
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy to Staging') {
            steps {
                script {
                    echo 'Deploying to staging environment...'
                    sh './deploy.sh'
                }
            }
        }
        
        stage('Smoke Tests') {
            steps {
                script {
                    echo 'Running smoke tests on deployed application...'
                    sh '''
                        # Wait for deployment to complete
                        sleep 20
                        
                        # Run smoke tests
                        curl -f http://localhost/health
                        curl -f http://localhost/ | grep "Hello from Docker"
                        
                        # Test response time
                        response_time=$(curl -o /dev/null -s -w '%{time_total}' http://localhost/)
                        echo "Response time: ${response_time}s"
                        
                        echo "Smoke tests passed!"
                    '''
                }
            }
        }
        
        stage('Performance Tests') {
            steps {
                script {
                    echo 'Running performance tests...'
                    sh '''
                        # Install Apache Bench if not available
                        which ab || sudo apt-get update && sudo apt-get install -y apache2-utils
                        
                        # Run performance test
                        ab -n 100 -c 10 http://localhost/ > performance_results.txt
                        cat performance_results.txt
                        
                        echo "Performance tests completed!"
                    '''
                }
            }
        }
        
        stage('Clean Up') {
            steps {
                script {
                    echo 'Cleaning up local images...'
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh 'docker system prune -f'
                }
            }
        }
    }
    
    post {
        always {
            echo 'Pipeline completed!'
            archiveArtifacts artifacts: 'performance_results.txt', allowEmptyArchive: true
        }
        success {
            echo 'Pipeline succeeded!'
            echo 'Application deployed and tested successfully!'
            emailext (
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Good news! The pipeline succeeded. Application is deployed and running.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
        failure {
            echo 'Pipeline failed!'
            sh 'docker-compose logs || true'
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Bad news! The pipeline failed. Please check the logs.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
EOF
Subtask 5.4: Create Git Repository and Push Code
# Initialize git repository
git init
git add .
git commit -m "Initial commit: Docker Jenkins automation pipeline"

# Add remote repository (replace with your GitHub repo URL)
git remote add origin https://github.com/your-username/docker-jenkins-demo.git
git branch -M main
git push -u origin main
Testing and Validation
Verify Pipeline Execution
Trigger Pipeline: Go to Jenkins and run your pipeline job

Monitor Stages: Watch each stage execute in the Jenkins Blue Ocean interface

Check Logs: Review console output for each stage

Verify Deployment: Access your application at http://localhost

Check Docker Hub: Verify images are pushed to your Docker Hub repository

Troubleshooting Common Issues
Docker Permission Issues:

sudo usermod -aG docker jenkins
sudo systemctl restart jenkins
SonarQube Connection Issues:

# Check SonarQube status
docker logs sonarqube
curl http://localhost:9000/api/system/status
Pipeline Failures:

Check Jenkins console logs
Verify all credentials are properly configured
Ensure Docker Hub username is correct in Jenkinsfile
Check network connectivity to external services
Conclusion
Congratulations! You have successfully completed Lab 66: Docker for Automation. In this comprehensive lab, you have accomplished the following:

Key Achievements
Docker-Jenkins Integration: Successfully installed and configured Docker within Jenkins, enabling containerized CI/CD workflows

Automated Pipeline Creation: Built a complete Jenkins pipeline that automatically builds Docker images and pushes them to Docker Hub registry

Quality Assurance Integration: Integrated SonarQube for static code analysis and implemented quality gates to ensure code quality standards

Cloud Deployment Automation: Configured automated deployment of Docker containers with health checks and monitoring

Comprehensive Testing Strategy: Implemented multiple testing layers including unit tests, integration tests, smoke tests, and performance tests, all running in Docker containers
