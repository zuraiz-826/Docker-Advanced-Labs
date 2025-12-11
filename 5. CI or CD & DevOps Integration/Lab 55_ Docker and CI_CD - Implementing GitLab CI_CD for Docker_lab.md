Lab 55: Docker and CI/CD - Implementing GitLab CI/CD for Docker
Lab Objectives
By the end of this lab, students will be able to:

Set up a GitLab repository with proper CI/CD configuration
Create and configure .gitlab-ci.yml files for Docker-based pipelines
Build Docker images automatically using GitLab CI/CD
Push Docker images to Docker Hub registry through automated pipelines
Implement automated testing within Docker containers
Configure automatic deployment triggers based on code commits
Understand the complete CI/CD workflow for containerized applications
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with Git version control system
Basic knowledge of YAML syntax
Understanding of command-line interface operations
A GitLab account (free tier is sufficient)
A Docker Hub account (free tier is sufficient)
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment. No need to build your own VM or install software locally.

Your cloud machine includes:

Docker Engine (latest stable version)
Git client
Text editors (nano, vim)
curl and wget utilities
All necessary development tools
Task 1: Set up GitLab Repository and Create .gitlab-ci.yml File
Subtask 1.1: Create a New GitLab Project
Log into GitLab

Navigate to https://gitlab.com
Sign in with your GitLab credentials
Click on New Project button
Configure Project Settings

Select Create blank project
Project name: docker-cicd-demo
Project slug: docker-cicd-demo
Visibility level: Private (recommended for learning)
Initialize repository with README: checked
Click Create project
Subtask 1.2: Clone Repository to Cloud Machine
Access your cloud machine terminal

Clone the repository

# Replace YOUR_USERNAME with your actual GitLab username
git clone https://gitlab.com/YOUR_USERNAME/docker-cicd-demo.git
cd docker-cicd-demo
Configure Git credentials
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
Subtask 1.3: Create Sample Application
Create a simple Node.js application
# Create package.json
cat > package.json << 'EOF'
{
  "name": "docker-cicd-demo",
  "version": "1.0.0",
  "description": "Demo app for Docker CI/CD",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "node test.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.5.0"
  }
}
EOF
Create the main application file
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Docker CI/CD Demo!',
    version: '1.0.0',
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

const server = app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});

module.exports = { app, server };
EOF
Create a simple test file
cat > test.js << 'EOF'
const http = require('http');

// Simple test function
function testHealthEndpoint() {
  return new Promise((resolve, reject) => {
    const options = {
      hostname: 'localhost',
      port: 3000,
      path: '/health',
      method: 'GET'
    };

    const req = http.request(options, (res) => {
      let data = '';
      res.on('data', (chunk) => {
        data += chunk;
      });
      res.on('end', () => {
        try {
          const response = JSON.parse(data);
          if (response.status === 'healthy') {
            console.log('✓ Health check test passed');
            resolve(true);
          } else {
            console.log('✗ Health check test failed');
            reject(new Error('Health check failed'));
          }
        } catch (error) {
          reject(error);
        }
      });
    });

    req.on('error', (error) => {
      reject(error);
    });

    req.end();
  });
}

// Run test
console.log('Running tests...');
testHealthEndpoint()
  .then(() => {
    console.log('All tests passed!');
    process.exit(0);
  })
  .catch((error) => {
    console.error('Tests failed:', error.message);
    process.exit(1);
  });
EOF
Subtask 1.4: Create Dockerfile
Create Dockerfile for the application
cat > Dockerfile << 'EOF'
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory in container
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --only=production

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
EOF
Subtask 1.5: Create .gitlab-ci.yml File
Create the GitLab CI/CD configuration file
cat > .gitlab-ci.yml << 'EOF'
# Define stages for the pipeline
stages:
  - build
  - test
  - push
  - deploy

# Define variables
variables:
  DOCKER_IMAGE_NAME: $CI_REGISTRY_USER/docker-cicd-demo
  DOCKER_TAG: $CI_COMMIT_SHORT_SHA

# Use Docker-in-Docker service
services:
  - docker:24.0.5-dind

# Before script - runs before each job
before_script:
  - docker info
  - echo "Pipeline started for commit $CI_COMMIT_SHORT_SHA"

# Build stage - Build Docker image
build_image:
  stage: build
  image: docker:24.0.5
  script:
    - echo "Building Docker image..."
    - docker build -t $DOCKER_IMAGE_NAME:$DOCKER_TAG .
    - docker build -t $DOCKER_IMAGE_NAME:latest .
    - echo "Docker image built successfully"
  only:
    - main
    - develop

# Test stage - Run tests in Docker container
test_application:
  stage: test
  image: docker:24.0.5
  script:
    - echo "Running tests in Docker container..."
    - docker build -t test-image .
    - docker run --name test-container -d -p 3000:3000 test-image
    - sleep 10  # Wait for application to start
    - docker exec test-container npm test
    - docker stop test-container
    - docker rm test-container
    - echo "Tests completed successfully"
  only:
    - main
    - develop

# Push stage - Push to Docker Hub
push_to_dockerhub:
  stage: push
  image: docker:24.0.5
  script:
    - echo "Logging into Docker Hub..."
    - echo $DOCKER_HUB_PASSWORD | docker login -u $DOCKER_HUB_USERNAME --password-stdin
    - echo "Pushing Docker image to Docker Hub..."
    - docker build -t $DOCKER_HUB_USERNAME/docker-cicd-demo:$DOCKER_TAG .
    - docker build -t $DOCKER_HUB_USERNAME/docker-cicd-demo:latest .
    - docker push $DOCKER_HUB_USERNAME/docker-cicd-demo:$DOCKER_TAG
    - docker push $DOCKER_HUB_USERNAME/docker-cicd-demo:latest
    - echo "Docker image pushed successfully"
  only:
    - main

# Deploy stage - Deploy container (simulation)
deploy_container:
  stage: deploy
  image: docker:24.0.5
  script:
    - echo "Deploying application..."
    - echo "Pulling latest image from Docker Hub..."
    - docker pull $DOCKER_HUB_USERNAME/docker-cicd-demo:latest
    - echo "Stopping existing container if running..."
    - docker stop docker-cicd-demo || true
    - docker rm docker-cicd-demo || true
    - echo "Starting new container..."
    - docker run -d --name docker-cicd-demo -p 8080:3000 $DOCKER_HUB_USERNAME/docker-cicd-demo:latest
    - echo "Application deployed successfully on port 8080"
  only:
    - main
  when: manual  # Manual deployment trigger
EOF
Task 2: Configure GitLab CI to Build Docker Images
Subtask 2.1: Enable GitLab Runner
Verify GitLab Runner availability
In your GitLab project, navigate to Settings > CI/CD
Expand Runners section
Ensure Shared runners are enabled (they should be by default)
Subtask 2.2: Configure Docker-in-Docker
Update .gitlab-ci.yml for proper Docker-in-Docker configuration
cat > .gitlab-ci.yml << 'EOF'
# GitLab CI/CD Pipeline for Docker
image: docker:24.0.5

# Define stages
stages:
  - build
  - test
  - push
  - deploy

# Variables
variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_IMAGE_NAME: docker-cicd-demo

# Services
services:
  - docker:24.0.5-dind

# Before script
before_script:
  - docker --version
  - docker info

# Build Docker image
build_docker_image:
  stage: build
  script:
    - echo "Building Docker image for commit $CI_COMMIT_SHORT_SHA"
    - docker build -t $DOCKER_IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker build -t $DOCKER_IMAGE_NAME:latest .
    - docker images
    - echo "Build completed successfully"
  artifacts:
    reports:
      dotenv: build.env
  only:
    - main
    - develop
    - merge_requests

# Test the built image
test_docker_image:
  stage: test
  script:
    - echo "Testing Docker image..."
    - docker build -t $DOCKER_IMAGE_NAME:test .
    - echo "Starting container for testing..."
    - docker run -d --name test-app -p 3000:3000 $DOCKER_IMAGE_NAME:test
    - sleep 15
    - echo "Running health check..."
    - docker exec test-app curl -f http://localhost:3000/health || exit 1
    - echo "Running application tests..."
    - docker exec test-app npm test
    - echo "Cleaning up test container..."
    - docker stop test-app
    - docker rm test-app
    - echo "Tests passed successfully"
  only:
    - main
    - develop
    - merge_requests
EOF
Subtask 2.3: Commit and Push Changes
Add all files to Git
git add .
git status
Commit the changes
git commit -m "Initial commit: Add Node.js app with Docker and GitLab CI/CD configuration"
Push to GitLab
git push origin main
Subtask 2.4: Monitor Pipeline Execution
View pipeline in GitLab
Navigate to your GitLab project
Click on CI/CD > Pipelines
Click on the running pipeline to view details
Monitor the build_docker_image job execution
Task 3: Push Docker Images to Docker Hub Automatically
Subtask 3.1: Set up Docker Hub Credentials
Create Docker Hub access token
Log into Docker Hub (https://hub.docker.com)
Go to Account Settings > Security
Click New Access Token
Token description: GitLab CI/CD
Access permissions: Read, Write, Delete
Click Generate and copy the token
Subtask 3.2: Configure GitLab CI/CD Variables
Add Docker Hub credentials to GitLab

In GitLab project, go to Settings > CI/CD
Expand Variables section
Add the following variables:
Variable 1:

Key: DOCKER_HUB_USERNAME
Value: Your Docker Hub username
Type: Variable
Environment scope: All
Protect variable: checked
Mask variable: unchecked
Variable 2:

Key: DOCKER_HUB_PASSWORD
Value: Your Docker Hub access token
Type: Variable
Environment scope: All
Protect variable: checked
Mask variable: checked
Subtask 3.3: Update Pipeline for Docker Hub Push
Update .gitlab-ci.yml to include Docker Hub push
cat > .gitlab-ci.yml << 'EOF'
# GitLab CI/CD Pipeline for Docker with Docker Hub Integration
image: docker:24.0.5

stages:
  - build
  - test
  - push
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_IMAGE_NAME: docker-cicd-demo

services:
  - docker:24.0.5-dind

before_script:
  - docker --version
  - docker info

# Build stage
build_docker_image:
  stage: build
  script:
    - echo "Building Docker image..."
    - docker build -t $DOCKER_IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker build -t $DOCKER_IMAGE_NAME:latest .
    - docker images
    - echo "✓ Build completed successfully"
  only:
    - main
    - develop

# Test stage
test_docker_image:
  stage: test
  script:
    - echo "Testing Docker image..."
    - docker build -t $DOCKER_IMAGE_NAME:test .
    - docker run -d --name test-app -p 3000:3000 $DOCKER_IMAGE_NAME:test
    - sleep 15
    - echo "Running health check..."
    - docker exec test-app curl -f http://localhost:3000/health
    - echo "Running application tests..."
    - docker exec test-app npm test
    - docker stop test-app && docker rm test-app
    - echo "✓ All tests passed"
  only:
    - main
    - develop

# Push to Docker Hub
push_to_dockerhub:
  stage: push
  script:
    - echo "Logging into Docker Hub..."
    - echo $DOCKER_HUB_PASSWORD | docker login -u $DOCKER_HUB_USERNAME --password-stdin
    - echo "Building images for Docker Hub..."
    - docker build -t $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker build -t $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:latest .
    - echo "Pushing images to Docker Hub..."
    - docker push $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:latest
    - echo "✓ Images pushed successfully to Docker Hub"
    - docker logout
  only:
    - main

# Simulate deployment
deploy_application:
  stage: deploy
  script:
    - echo "Deploying application..."
    - echo "Pulling latest image from Docker Hub..."
    - docker pull $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:latest
    - echo "Stopping existing container..."
    - docker stop $DOCKER_IMAGE_NAME || true
    - docker rm $DOCKER_IMAGE_NAME || true
    - echo "Starting new container..."
    - docker run -d --name $DOCKER_IMAGE_NAME -p 8080:3000 $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:latest
    - echo "✓ Application deployed on port 8080"
    - docker ps
  only:
    - main
  when: manual
EOF
Subtask 3.4: Test Docker Hub Integration
Commit and push the updated pipeline
git add .gitlab-ci.yml
git commit -m "Add Docker Hub integration to CI/CD pipeline"
git push origin main
Monitor the pipeline

Go to CI/CD > Pipelines in GitLab
Watch the pipeline execute through all stages
Verify the push_to_dockerhub stage completes successfully
Verify image in Docker Hub

Log into Docker Hub
Navigate to your repositories
Confirm the docker-cicd-demo repository exists with the pushed images
Task 4: Automate Tests Inside Docker Containers Using GitLab CI
Subtask 4.1: Create Comprehensive Test Suite
Create an advanced test file
cat > test.js << 'EOF'
const http = require('http');
const { spawn } = require('child_process');

class TestRunner {
  constructor() {
    this.tests = [];
    this.passed = 0;
    this.failed = 0;
  }

  addTest(name, testFunction) {
    this.tests.push({ name, testFunction });
  }

  async runTests() {
    console.log('🚀 Starting test suite...\n');
    
    for (const test of this.tests) {
      try {
        console.log(`Running: ${test.name}`);
        await test.testFunction();
        console.log(`✅ PASSED: ${test.name}\n`);
        this.passed++;
      } catch (error) {
        console.log(`❌ FAILED: ${test.name}`);
        console.log(`   Error: ${error.message}\n`);
        this.failed++;
      }
    }

    this.printResults();
    return this.failed === 0;
  }

  printResults() {
    console.log('📊 Test Results:');
    console.log(`   Passed: ${this.passed}`);
    console.log(`   Failed: ${this.failed}`);
    console.log(`   Total:  ${this.tests.length}`);
  }
}

// Test functions
function testHealthEndpoint() {
  return new Promise((resolve, reject) => {
    const options = {
      hostname: 'localhost',
      port: 3000,
      path: '/health',
      method: 'GET',
      timeout: 5000
    };

    const req = http.request(options, (res) => {
      let data = '';
      res.on('data', (chunk) => data += chunk);
      res.on('end', () => {
        try {
          const response = JSON.parse(data);
          if (res.statusCode === 200 && response.status === 'healthy') {
            resolve();
          } else {
            reject(new Error(`Health check failed: ${res.statusCode}`));
          }
        } catch (error) {
          reject(new Error(`Invalid JSON response: ${error.message}`));
        }
      });
    });

    req.on('error', reject);
    req.on('timeout', () => reject(new Error('Request timeout')));
    req.end();
  });
}

function testRootEndpoint() {
  return new Promise((resolve, reject) => {
    const options = {
      hostname: 'localhost',
      port: 3000,
      path: '/',
      method: 'GET',
      timeout: 5000
    };

    const req = http.request(options, (res) => {
      let data = '';
      res.on('data', (chunk) => data += chunk);
      res.on('end', () => {
        try {
          const response = JSON.parse(data);
          if (res.statusCode === 200 && response.message && response.version) {
            resolve();
          } else {
            reject(new Error(`Root endpoint test failed: ${res.statusCode}`));
          }
        } catch (error) {
          reject(new Error(`Invalid JSON response: ${error.message}`));
        }
      });
    });

    req.on('error', reject);
    req.on('timeout', () => reject(new Error('Request timeout')));
    req.end();
  });
}

function testInvalidEndpoint() {
  return new Promise((resolve, reject) => {
    const options = {
      hostname: 'localhost',
      port: 3000,
      path: '/nonexistent',
      method: 'GET',
      timeout: 5000
    };

    const req = http.request(options, (res) => {
      if (res.statusCode === 404) {
        resolve();
      } else {
        reject(new Error(`Expected 404, got ${res.statusCode}`));
      }
    });

    req.on('error', reject);
    req.on('timeout', () => reject(new Error('Request timeout')));
    req.end();
  });
}

// Run tests
async function main() {
  const runner = new TestRunner();
  
  runner.addTest('Health Endpoint Test', testHealthEndpoint);
  runner.addTest('Root Endpoint Test', testRootEndpoint);
  runner.addTest('404 Endpoint Test', testInvalidEndpoint);
  
  const success = await runner.runTests();
  process.exit(success ? 0 : 1);
}

main().catch(error => {
  console.error('Test runner failed:', error);
  process.exit(1);
});
EOF
Subtask 4.2: Create Test-Specific Dockerfile
Create Dockerfile for testing
cat > Dockerfile.test << 'EOF'
# Test-specific Dockerfile
FROM node:18-alpine

WORKDIR /usr/src/app

# Install curl for health checks
RUN apk add --no-cache curl

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev dependencies)
RUN npm install

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Add test user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
RUN chown -R nodejs:nodejs /usr/src/app
USER nodejs

# Default command for testing
CMD ["npm", "test"]
EOF
Subtask 4.3: Update Pipeline with Advanced Testing
Update .gitlab-ci.yml with comprehensive testing
cat > .gitlab-ci.yml << 'EOF'
# Advanced GitLab CI/CD Pipeline with Comprehensive Testing
image: docker:24.0.5

stages:
  - build
  - test
  - security
  - push
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_IMAGE_NAME: docker-cicd-demo

services:
  - docker:24.0.5-dind

before_script:
  - docker --version
  - docker info

# Build stage
build_application:
  stage: build
  script:
    - echo "🏗️ Building application images..."
    - docker build -t $DOCKER_IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker build -t $DOCKER_IMAGE_NAME:latest .
    - docker build -f Dockerfile.test -t $DOCKER_IMAGE_NAME:test .
    - docker images
    - echo "✅ Build completed"
  only:
    - main
    - develop
    - merge_requests

# Unit tests in container
unit_tests:
  stage: test
  script:
    - echo "🧪 Running unit tests in container..."
    - docker run --rm $DOCKER_IMAGE_NAME:test
    - echo "✅ Unit tests passed"
  only:
    - main
    - develop
    - merge_requests

# Integration tests
integration_tests:
  stage: test
  script:
    - echo "🔗 Running integration tests..."
    - docker run -d --name integration-test -p 3000:3000 $DOCKER_IMAGE_NAME:latest
    - sleep 20
    - echo "Testing application endpoints..."
    - docker exec integration-test curl -f http://localhost:3000/health
    - docker exec integration-test curl -f http://localhost:3000/
    - echo "Running comprehensive test suite..."
    - docker exec integration-test npm test
    - docker stop integration-test
    - docker rm integration-test
    - echo "✅ Integration tests passed"
  only:
    - main
    - develop
    - merge_requests

# Container security scan (basic)
security_scan:
  stage: security
  script:
    - echo "🔒 Running basic security checks..."
    - docker run --rm -v /var/run/docker.sock:/var/run/docker.sock 
      -v $(pwd):/tmp alpine:latest sh -c "
      echo 'Checking for common security issues...'
      echo 'Verifying non-root user in container...'
      docker inspect $DOCKER_IMAGE_NAME:latest | grep -i user || echo 'Warning: Running as root'
      echo 'Security scan completed'
      "
    - echo "✅ Security checks completed"
  only:
    - main
    - develop
  allow_failure: true

# Performance test
performance_test:
  stage: test
  script:
    - echo "⚡ Running performance tests..."
    - docker run -d --name perf-test -p 3000:3000 $DOCKER_IMAGE_NAME:latest
    - sleep 15
    - echo "Testing application response time..."
    - docker exec perf-test sh -c "
      for i in \$(seq 1 10); do
        time curl -s http://localhost:3000/ > /dev/null
      done
      "
    - docker stop perf-test
    - docker rm perf-test
    - echo "✅ Performance tests completed"
  only:
    - main
    - develop
  allow_failure: true

# Push to Docker Hub
push_to_registry:
  stage: push
  script:
    - echo "📦 Pushing to Docker Hub..."
    - echo $DOCKER_HUB_PASSWORD | docker login -u $DOCKER_HUB_USERNAME --password-stdin
    - docker build -t $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker build -t $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:latest .
    - docker push $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:latest
    - docker logout
    - echo "✅ Images pushed successfully"
  only:
    - main

# Deploy application
deploy_production:
  stage: deploy
  script:
    - echo "🚀 Deploying to production..."
    - docker pull $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:latest
    - docker stop $DOCKER_IMAGE_NAME || true
    - docker rm $DOCKER_IMAGE_NAME || true
    - docker run -d --name $DOCKER_IMAGE_NAME -p 8080:3000 --restart unless-stopped $DOCKER_HUB_USERNAME/$DOCKER_IMAGE_NAME:latest
    - sleep 10
    - echo "Verifying deployment..."
    - docker exec $DOCKER_IMAGE_NAME curl -f http://localhost:3000/health
    - echo "✅ Deployment successful - Application running on port 8080"
  only:
    - main
  when: manual
  environment:
    name: production
    url: http://localhost:8080
EOF
Task 5: Trigger Pipeline on Commits to Automatically Deploy Docker Containers
Subtask 5.1: Configure Automatic Triggers
Create branch protection and trigger rules
# Create a feature branch to test automatic triggers
git checkout -b feature/auto-deploy-test

# Make a small change to trigger the pipeline
echo "# Docker CI/CD Demo Application

This application demonstrates GitLab CI/CD integration with Docker.

## Features
- Automated Docker image building
- Comprehensive testing in containers
- Automatic deployment on commits
- Docker Hub integration

## Version: 1.1.0" > README.md
Update application version
# Update package.json version
sed -i 's/"version": "1.0.0"/"version": "1.1.0"/' package.json

# Update app.js version
sed -i 's/version.*1.0.0/version: "1.1.0"/' app.js
Subtask 5.2: Create Advanced Pipeline with Auto-Deploy
Create production-ready pipeline configuration
cat > .gitlab-ci.yml << 'EOF'
# Production-Ready GitLab CI/CD Pipeline
image: docker:24.0.5

stages:
  - validate
  - build
  - test
  - security
  - push
  - deploy-staging
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_IMAGE_NAME: docker-cicd-demo
  STAGING_PORT: 8081
  PRODUCTION_PORT: 8080

services:
  - docker:24.0.5-dind

# Global before script
before_script:
  - docker --version
  - docker info
  - echo "Pipeline triggered by $GITLAB_USER_NAME for commit $CI_COMMIT_SHORT_SHA"

# Validate stage - Check code quality
validate_code:
  stage: validate
  script:
    - echo "🔍 Validating code structure..."
    - test -f Dockerfile || (echo "❌ Dockerfile missing" && exit 1)
    - test -f package.json || (echo "❌ package.json missing" && exit 1)
    - test -f app.js || (echo "❌ app.js missing" && exit 1)
    - echo "✅ Code validation passed"
  only:
    - main
    - develop
    - merge_requests

# Build stage
build_images:
  stage: build
  script:
    - echo "🏗️ Building Docker images..."
    - docker build -t $DOCKER_IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker build -t $DOCKER_IMAGE_NAME:latest .
    - docker build -f Dockerfile.test -t $DOCKER_IMAGE_NAME:test . || docker build -t $DOCKER_IMAGE_NAME:test .
    - docker images | grep $DOCKER_IMAGE_NAME
    - echo "✅ Images built successfully"
  artifacts:
    expire_in: 1 hour
    reports:
      dotenv: build.env
  only:
    - main
    - develop
    - merge_requests

# Test stage - Comprehensive testing
run_tests:
  stage: test
  script:
    - echo "🧪 Running comprehensive test suite..."
    - echo "Starting application container..."
    - docker run -d --name test-app -p 3000:3000 $DOCKER_IMAGE_NAME:latest
    - sleep 20
    - echo "Running health checks..."
    - docker exec test-app curl -f http://localhost:3000/
