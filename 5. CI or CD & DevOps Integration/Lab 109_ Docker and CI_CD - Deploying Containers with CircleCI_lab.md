Lab 109: Docker and CI/CD - Deploying Containers with CircleCI
Objectives
By the end of this lab, you will be able to:

Set up a complete CI/CD pipeline using CircleCI for containerized applications
Automate Docker image builds and testing processes
Push Docker images to Docker Hub registry after successful builds
Deploy applications automatically to cloud platforms using containers
Integrate comprehensive testing into your CI/CD workflow
Monitor pipeline status and troubleshoot common deployment issues
Understand best practices for container-based continuous deployment
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with Git version control system
Basic knowledge of YAML configuration files
Understanding of command-line interface operations
A GitHub account for code repository management
A Docker Hub account for image registry
Basic understanding of web applications and APIs
Lab Environment Setup
Good News: Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click Start Lab and you'll have access to a fully configured environment with all necessary tools pre-installed, including:

Docker Engine and Docker Compose
Git version control system
Node.js runtime environment
Text editors (nano, vim)
curl and other networking tools
No need to build your own virtual machine or install software locally.

Task 1: Set up CircleCI to Automate Docker Image Builds
Subtask 1.1: Create a Sample Application
First, let's create a simple Node.js web application that we'll containerize and deploy.

Create project directory and navigate to it:
mkdir docker-cicd-lab
cd docker-cicd-lab
Initialize a new Node.js project:
npm init -y
Install Express.js framework:
npm install express
Create the main application file:
nano app.js
Add the following content:

const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Dockerized CI/CD Pipeline!',
    version: '1.0.0',
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    uptime: process.uptime()
  });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});

module.exports = app;
Create a test file:
npm install --save-dev jest supertest
nano app.test.js
Add the following test content:

const request = require('supertest');
const app = require('./app');

describe('App Tests', () => {
  test('GET / should return welcome message', async () => {
    const response = await request(app).get('/');
    expect(response.status).toBe(200);
    expect(response.body.message).toBe('Hello from Dockerized CI/CD Pipeline!');
  });

  test('GET /health should return health status', async () => {
    const response = await request(app).get('/health');
    expect(response.status).toBe(200);
    expect(response.body.status).toBe('healthy');
  });
});
Update package.json with test script:
nano package.json
Modify the scripts section:

{
  "name": "docker-cicd-lab",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "jest",
    "dev": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^6.3.3"
  }
}
Subtask 1.2: Create Dockerfile
Create Dockerfile for the application:
nano Dockerfile
Add the following content:

# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory in container
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Change ownership of app directory
RUN chown -R nextjs:nodejs /usr/src/app
USER nextjs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
Create .dockerignore file:
nano .dockerignore
Add the following content:

node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.nyc_output
coverage
.coverage
.cache
Test Docker build locally:
docker build -t docker-cicd-app:latest .
Test the container locally:
docker run -d -p 3000:3000 --name test-app docker-cicd-app:latest
Verify the application is running:
curl http://localhost:3000
curl http://localhost:3000/health
Stop and remove test container:
docker stop test-app
docker rm test-app
Subtask 1.3: Initialize Git Repository
Initialize Git repository:
git init
Create .gitignore file:
nano .gitignore
Add the following content:

node_modules/
npm-debug.log*
.npm
.env
.env.local
.env.development.local
.env.test.local
.env.production.local
coverage/
.nyc_output/
.cache/
dist/
build/
Add and commit initial files:
git add .
git commit -m "Initial commit: Node.js app with Docker configuration"
Subtask 1.4: Set up GitHub Repository
Create a new repository on GitHub (via web interface):

Go to github.com and sign in
Click "New repository"
Name it "docker-cicd-lab"
Make it public
Don't initialize with README (we already have files)
Connect local repository to GitHub:

git remote add origin https://github.com/YOUR_USERNAME/docker-cicd-lab.git
git branch -M main
git push -u origin main
Replace YOUR_USERNAME with your actual GitHub username.

Subtask 1.5: Configure CircleCI
Create CircleCI configuration directory:
mkdir -p .circleci
Create CircleCI configuration file:
nano .circleci/config.yml
Add the following configuration:

version: 2.1

# Define reusable commands
commands:
  install_dependencies:
    description: "Install Node.js dependencies"
    steps:
      - run:
          name: Install dependencies
          command: npm ci

# Define jobs
jobs:
  test:
    docker:
      - image: cimg/node:18.17
    steps:
      - checkout
      - install_dependencies
      - run:
          name: Run tests
          command: npm test
      - run:
          name: Generate test coverage
          command: npm test -- --coverage
      - store_test_results:
          path: coverage
      - store_artifacts:
          path: coverage

  build_and_push:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
          docker_layer_caching: true
      - run:
          name: Build Docker image
          command: |
            docker build -t $DOCKER_USERNAME/docker-cicd-app:$CIRCLE_SHA1 .
            docker build -t $DOCKER_USERNAME/docker-cicd-app:latest .
      - run:
          name: Login to Docker Hub
          command: echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
      - run:
          name: Push Docker image
          command: |
            docker push $DOCKER_USERNAME/docker-cicd-app:$CIRCLE_SHA1
            docker push $DOCKER_USERNAME/docker-cicd-app:latest

# Define workflows
workflows:
  version: 2
  test_build_deploy:
    jobs:
      - test
      - build_and_push:
          requires:
            - test
          filters:
            branches:
              only: main
Commit CircleCI configuration:
git add .circleci/config.yml
git commit -m "Add CircleCI configuration for automated builds"
git push origin main
Subtask 1.6: Connect Repository to CircleCI
Sign up/Login to CircleCI:

Go to circleci.com
Sign up with your GitHub account
Authorize CircleCI to access your repositories
Set up project in CircleCI:

Click "Set Up Project" next to your docker-cicd-lab repository
Choose "Use Existing Config" since we already created .circleci/config.yml
Click "Start Building"
Task 2: Push Docker Images to Docker Hub After Successful Tests
Subtask 2.1: Configure Docker Hub Credentials
In CircleCI dashboard, go to Project Settings:

Click on your project name
Go to "Project Settings"
Navigate to "Environment Variables"
Add Docker Hub credentials:

Click "Add Environment Variable"

Name: DOCKER_USERNAME

Value: Your Docker Hub username

Click "Add Environment Variable"

Click "Add Environment Variable" again

Name: DOCKER_PASSWORD

Value: Your Docker Hub password or access token

Click "Add Environment Variable"

Subtask 2.2: Test the Pipeline
Make a small change to trigger the pipeline:
nano app.js
Update the version in the response:

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Dockerized CI/CD Pipeline!',
    version: '1.1.0',  // Changed from 1.0.0
    timestamp: new Date().toISOString()
  });
});
Commit and push the change:
git add app.js
git commit -m "Update application version to 1.1.0"
git push origin main
Monitor the pipeline in CircleCI:

Go to your CircleCI dashboard
Watch the pipeline execute
Verify both test and build_and_push jobs complete successfully
Verify image was pushed to Docker Hub:

Go to hub.docker.com
Check your repositories
Confirm docker-cicd-app repository exists with latest tag
Task 3: Set up Automatic Deployments to Cloud Platform
Subtask 3.1: Prepare Deployment Configuration
For this lab, we'll simulate deployment to a cloud platform using Docker Compose. In production, you would adapt this for AWS ECS, DigitalOcean App Platform, or similar services.

Create docker-compose.yml for deployment:
nano docker-compose.yml
Add the following content:

version: '3.8'

services:
  app:
    image: ${DOCKER_USERNAME}/docker-cicd-app:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - PORT=3000
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
Create Nginx configuration:
nano nginx.conf
Add the following content:

events {
    worker_connections 1024;
}

http {
    upstream app {
        server app:3000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            proxy_pass http://app/health;
            access_log off;
        }
    }
}
Create deployment script:
nano deploy.sh
Add the following content:

#!/bin/bash

set -e

echo "Starting deployment process..."

# Pull latest images
echo "Pulling latest Docker images..."
docker-compose pull

# Stop existing containers
echo "Stopping existing containers..."
docker-compose down

# Start new containers
echo "Starting new containers..."
docker-compose up -d

# Wait for health check
echo "Waiting for application to be healthy..."
sleep 30

# Check if application is running
if curl -f http://localhost/health; then
    echo "Deployment successful! Application is healthy."
else
    echo "Deployment failed! Application health check failed."
    exit 1
fi

echo "Deployment completed successfully!"
Make deployment script executable:
chmod +x deploy.sh
Subtask 3.2: Update CircleCI Configuration for Deployment
Update .circleci/config.yml to include deployment:
nano .circleci/config.yml
Replace the content with:

version: 2.1

# Define reusable commands
commands:
  install_dependencies:
    description: "Install Node.js dependencies"
    steps:
      - run:
          name: Install dependencies
          command: npm ci

# Define jobs
jobs:
  test:
    docker:
      - image: cimg/node:18.17
    steps:
      - checkout
      - install_dependencies
      - run:
          name: Run tests
          command: npm test
      - run:
          name: Generate test coverage
          command: npm test -- --coverage
      - store_test_results:
          path: coverage
      - store_artifacts:
          path: coverage

  build_and_push:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
          docker_layer_caching: true
      - run:
          name: Build Docker image
          command: |
            docker build -t $DOCKER_USERNAME/docker-cicd-app:$CIRCLE_SHA1 .
            docker build -t $DOCKER_USERNAME/docker-cicd-app:latest .
      - run:
          name: Login to Docker Hub
          command: echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
      - run:
          name: Push Docker image
          command: |
            docker push $DOCKER_USERNAME/docker-cicd-app:$CIRCLE_SHA1
            docker push $DOCKER_USERNAME/docker-cicd-app:latest

  deploy:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
      - run:
          name: Install Docker Compose
          command: |
            curl -L "https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-$(uname -s)-$(uname -m)" -o /tmp/docker-compose
            sudo mv /tmp/docker-compose /usr/local/bin/docker-compose
            sudo chmod +x /usr/local/bin/docker-compose
      - run:
          name: Deploy application
          command: |
            export DOCKER_USERNAME=$DOCKER_USERNAME
            ./deploy.sh

# Define workflows
workflows:
  version: 2
  test_build_deploy:
    jobs:
      - test
      - build_and_push:
          requires:
            - test
          filters:
            branches:
              only: main
      - deploy:
          requires:
            - build_and_push
          filters:
            branches:
              only: main
Commit deployment configuration:
git add .
git commit -m "Add deployment configuration and scripts"
git push origin main
Task 4: Integrate Tests into the CircleCI Pipeline Using Docker Containers
Subtask 4.1: Enhance Test Suite
Add more comprehensive tests:
nano app.test.js
Replace with enhanced test suite:

const request = require('supertest');
const app = require('./app');

describe('App Tests', () => {
  test('GET / should return welcome message', async () => {
    const response = await request(app).get('/');
    expect(response.status).toBe(200);
    expect(response.body.message).toBe('Hello from Dockerized CI/CD Pipeline!');
    expect(response.body.version).toBe('1.1.0');
    expect(response.body.timestamp).toBeDefined();
  });

  test('GET /health should return health status', async () => {
    const response = await request(app).get('/health');
    expect(response.status).toBe(200);
    expect(response.body.status).toBe('healthy');
    expect(response.body.uptime).toBeDefined();
    expect(typeof response.body.uptime).toBe('number');
  });

  test('GET /nonexistent should return 404', async () => {
    const response = await request(app).get('/nonexistent');
    expect(response.status).toBe(404);
  });

  test('Response should have correct content type', async () => {
    const response = await request(app).get('/');
    expect(response.headers['content-type']).toMatch(/json/);
  });

  test('Health endpoint should respond quickly', async () => {
    const start = Date.now();
    await request(app).get('/health');
    const duration = Date.now() - start;
    expect(duration).toBeLessThan(1000); // Should respond within 1 second
  });
});
Subtask 4.2: Add Integration Tests with Docker
Create integration test script:
nano integration-test.sh
Add the following content:

#!/bin/bash

set -e

echo "Starting integration tests..."

# Build test image
echo "Building Docker image for testing..."
docker build -t docker-cicd-app:test .

# Run container for testing
echo "Starting container for integration tests..."
docker run -d -p 3001:3000 --name integration-test-app docker-cicd-app:test

# Wait for container to be ready
echo "Waiting for container to be ready..."
sleep 10

# Run integration tests
echo "Running integration tests..."

# Test main endpoint
echo "Testing main endpoint..."
response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3001/)
if [ "$response" != "200" ]; then
    echo "Main endpoint test failed with status code: $response"
    docker logs integration-test-app
    docker stop integration-test-app
    docker rm integration-test-app
    exit 1
fi

# Test health endpoint
echo "Testing health endpoint..."
response=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3001/health)
if [ "$response" != "200" ]; then
    echo "Health endpoint test failed with status code: $response"
    docker logs integration-test-app
    docker stop integration-test-app
    docker rm integration-test-app
    exit 1
fi

# Test response content
echo "Testing response content..."
content=$(curl -s http://localhost:3001/ | jq -r '.message')
expected="Hello from Dockerized CI/CD Pipeline!"
if [ "$content" != "$expected" ]; then
    echo "Content test failed. Expected: $expected, Got: $content"
    docker stop integration-test-app
    docker rm integration-test-app
    exit 1
fi

# Cleanup
echo "Cleaning up test container..."
docker stop integration-test-app
docker rm integration-test-app

echo "All integration tests passed!"
Make integration test script executable:
chmod +x integration-test.sh
Subtask 4.3: Update CircleCI Configuration with Enhanced Testing
Update .circleci/config.yml with integration tests:
nano .circleci/config.yml
Replace with enhanced configuration:

version: 2.1

# Define reusable commands
commands:
  install_dependencies:
    description: "Install Node.js dependencies"
    steps:
      - run:
          name: Install dependencies
          command: npm ci

# Define jobs
jobs:
  unit_test:
    docker:
      - image: cimg/node:18.17
    steps:
      - checkout
      - install_dependencies
      - run:
          name: Run unit tests
          command: npm test
      - run:
          name: Generate test coverage
          command: npm test -- --coverage
      - store_test_results:
          path: coverage
      - store_artifacts:
          path: coverage

  integration_test:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
      - run:
          name: Install jq for JSON parsing
          command: sudo apt-get update && sudo apt-get install -y jq
      - run:
          name: Run integration tests
          command: ./integration-test.sh

  security_scan:
    docker:
      - image: cimg/node:18.17
    steps:
      - checkout
      - install_dependencies
      - run:
          name: Run security audit
          command: npm audit --audit-level moderate

  build_and_push:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
          docker_layer_caching: true
      - run:
          name: Build Docker image
          command: |
            docker build -t $DOCKER_USERNAME/docker-cicd-app:$CIRCLE_SHA1 .
            docker build -t $DOCKER_USERNAME/docker-cicd-app:latest .
      - run:
          name: Test Docker image
          command: |
            # Test that the image runs correctly
            docker run -d -p 3000:3000 --name test-container $DOCKER_USERNAME/docker-cicd-app:latest
            sleep 10
            curl -f http://localhost:3000/health || exit 1
            docker stop test-container
            docker rm test-container
      - run:
          name: Login to Docker Hub
          command: echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
      - run:
          name: Push Docker image
          command: |
            docker push $DOCKER_USERNAME/docker-cicd-app:$CIRCLE_SHA1
            docker push $DOCKER_USERNAME/docker-cicd-app:latest

  deploy:
    docker:
      - image: cimg/base:stable
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
      - run:
          name: Install Docker Compose
          command: |
            curl -L "https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-$(uname -s)-$(uname -m)" -o /tmp/docker-compose
            sudo mv /tmp/docker-compose /usr/local/bin/docker-compose
            sudo chmod +x /usr/local/bin/docker-compose
      - run:
          name: Deploy application
          command: |
            export DOCKER_USERNAME=$DOCKER_USERNAME
            ./deploy.sh

# Define workflows
workflows:
  version: 2
  comprehensive_pipeline:
    jobs:
      - unit_test
      - integration_test
      - security_scan
      - build_and_push:
          requires:
            - unit_test
            - integration_test
            - security_scan
          filters:
            branches:
              only: main
      - deploy:
          requires:
            - build_and_push
          filters:
            branches:
              only: main
Commit enhanced testing configuration:
git add .
git commit -m "Add comprehensive testing pipeline with unit, integration, and security tests"
git push origin main
Task 5: Monitor CircleCI Pipeline Status and Troubleshoot Failures
Subtask 5.1: Set up Pipeline Monitoring
Create a status monitoring script:
nano monitor-pipeline.sh
Add the following content:

#!/bin/bash

# Pipeline monitoring script
echo "=== CircleCI Pipeline Monitor ==="
echo "Timestamp: $(date)"
echo "Repository: docker-cicd-lab"
echo "Branch: main"
echo ""

# Check if curl is available
if ! command -v curl &> /dev/null; then
    echo "curl is required but not installed."
    exit 1
fi

# Function to check pipeline status
check_pipeline_status() {
    echo "To monitor your pipeline:"
    echo "1. Visit: https://app.circleci.com/pipelines/github/YOUR_USERNAME/docker-cicd-lab"
    echo "2. Check the status of each job:"
    echo "   - unit_test: Runs Jest unit tests"
    echo "   - integration_test: Tests Docker container functionality"
    echo "   - security_scan: Checks for security vulnerabilities"
    echo "   - build_and_push: Builds and pushes Docker image"
    echo "   - deploy: Deploys application using Docker Compose"
    echo ""
    echo "Pipeline Status Indicators:"
    echo "🟢 Green: Job completed successfully"
    echo "🔴 Red: Job failed"
    echo "🟡 Yellow: Job is running"
    echo "⚪ Gray: Job is queued or not started"
}

check_pipeline_status
Make monitoring script executable:
chmod +x monitor-pipeline.sh
Subtask 5.2: Create Troubleshooting Guide
Create troubleshooting documentation:
nano TROUBLESHOOTING.md
Add the following content:

# CircleCI Pipeline Troubleshooting Guide

## Common Issues and Solutions

### 1. Unit Test Failures

**Symptoms:**
- unit_test job fails with test errors
- Red status in CircleCI dashboard

**Common Causes:**
- Code changes broke existing functionality
- Missing dependencies
- Environment differences

**Solutions:**
```bash
# Run tests locally first
npm test

# Check test coverage
npm test -- --coverage

# Fix failing tests and commit
git add .
git commit -m "Fix failing unit tests"
git push origin main
2. Integration Test Failures
Symptoms:

integration_test job fails
Docker container doesn't respond to health checks
Common Causes:

Application doesn't start properly in container
Port conflicts
Missing environment variables
Solutions:

# Test Docker build locally
docker build -t test-app .
docker run -p 3000:3000 test-app

# Check container logs
docker logs <container-id>

# Test integration script locally
./integration-test.sh
3. Security Scan Failures
Symptoms:

security_scan job fails with vulnerability warnings
npm audit reports issues
Solutions:

# Check security issues locally
npm audit

# Fix automatically where possible
npm audit fix

# Update dependencies
npm update

# For manual fixes, update package.json
4. Build and Push Failures
Symptoms:

build_and_push job fails
Docker Hub authentication errors
Image push failures
Common Causes:

Incorrect Docker Hub credentials
Network issues
Dockerfile syntax errors
Solutions:

Verify Docker Hub credentials in CircleCI:

Go to Project Settings > Environment Variables
Check DOCKER_USERNAME and DOCKER_PASSWORD
Test Docker build locally:

docker build -t test-image .
Test Docker Hub login:
docker login
5. Deployment Failures
Symptoms:

deploy job fails
Application doesn't start after deployment
Health check failures
Solutions:

# Test deployment locally
export DOCKER_USERNAME=your-username
./deploy.sh

# Check container status
docker-compose ps

# View container logs
docker-compose logs app

# Restart services
docker-compose restart
Debugging Steps
Step 1: Check CircleCI Logs
Go to CircleCI dashboard
Click on failed job
Expand failed step
Read error messages carefully
Step 2: Reproduce Locally
# Pull latest code
git pull origin main

# Run the same commands that failed in CircleCI
npm test
docker build -t test .
./integration-test.sh
Step 3: Check Environment Variables
Verify all required environment variables are set in CircleCI
Check for typos in variable names
Ensure sensitive data is properly masked
Step 4: Review Recent Changes
# Check recent commits
git log --oneline -10

# Compare with last working version
git diff HEAD~1
Prevention Tips
Always test locally before pushing
Use feature branches for major changes
Keep dependencies updated
Monitor pipeline regularly
Set up notifications for failures

### Subtask 5.3: Implement Pipeline Notifications

1. **Add notification configuration to CircleCI**:
```bash
nano .circleci/config.yml
Add notification steps to the deploy job:

# Add this to the deploy job after the deployment step
      - run:
          name: Notify deployment success
          command: |
            echo "Deployment completed successfully!"
            echo "Application URL: http://localhost"
            echo "Health Check: http://localhost/health"
          when: on_success
      - run:
          name: Notify deployment failure
          command: |
            echo "Deployment failed! Check logs for details."
            echo "Pipeline URL: $CIRCLE_BUILD_URL"
          when: on_fail
Subtask 5.4: Test Failure Scenarios
Intentionally break a test to see failure handling:
nano app.test.js
Temporarily modify a test to fail:

test('GET / should return welcome message', async () => {
  const response = await request(app).get('/');
  expect(response.status).toBe(200);
  expect(response.body.message).toBe('This will fail'); // Changed expected message
});
Commit and observe the failure:
git add app.test.js
git commit -m "Test pipeline failure handling"
git push origin main
Monitor the pipeline failure in CircleCI dashboard

Fix the test and push again:

nano app.test.js
Restore the correct test:

test('GET / should return welcome message', async () => {
  const response = await request(app).get('/');
  expect(response.status).toBe(200);
  expect(response.body.message).toBe('Hello from Dockerized CI/CD Pipeline!');
});
Commit the fix:
git add app.test.js
git commit -m "Fix intentionally broken test"
git push origin main
Subtask 5.5: Create Pipeline Dashboard
1
