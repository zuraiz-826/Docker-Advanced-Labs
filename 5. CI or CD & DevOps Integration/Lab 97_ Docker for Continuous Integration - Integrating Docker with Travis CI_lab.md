Lab 97: Docker for Continuous Integration - Integrating Docker with Travis CI
Objectives
By the end of this lab, students will be able to:

Set up Travis CI for automated Docker builds and deployments
Configure .travis.yml file to build Docker images automatically
Push built Docker images to Docker Hub after successful builds
Implement testing steps within Docker containers in Travis CI
Monitor Travis CI build logs and troubleshoot common issues
Understand the fundamentals of CI/CD pipelines with Docker
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts and commands
Familiarity with Git version control
Basic knowledge of web applications (HTML, JavaScript)
Understanding of YAML file format
A GitHub account (free)
A Docker Hub account (free)
Lab Environment
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to begin - no need to build your own VM or install software locally.

Your cloud machine includes:

Docker Engine
Git
Node.js and npm
Text editors (nano, vim)
All necessary development tools
Task 1: Set up Travis CI with a Dockerized Web Application
Subtask 1.1: Create a Simple Web Application
First, let's create a basic Node.js web application that we'll containerize and deploy.

Create a new directory for your project:
mkdir docker-travis-ci-lab
cd docker-travis-ci-lab
Initialize a new Node.js project:
npm init -y
Install Express.js:
npm install express
Create the main application file:
nano app.js
Add the following content:

const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
    res.json({
        message: 'Hello from Dockerized Node.js App!',
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

const server = app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});

module.exports = { app, server };
Create a test file:
npm install --save-dev jest supertest
nano app.test.js
Add the following test content:

const request = require('supertest');
const { app, server } = require('./app');

describe('App Tests', () => {
    afterAll(() => {
        server.close();
    });

    test('GET / should return welcome message', async () => {
        const response = await request(app).get('/');
        expect(response.status).toBe(200);
        expect(response.body.message).toBe('Hello from Dockerized Node.js App!');
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
  "name": "docker-travis-ci-lab",
  "version": "1.0.0",
  "description": "",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "jest",
    "test:watch": "jest --watch"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^6.3.3"
  }
}
Subtask 1.2: Create a Dockerfile
Create a Dockerfile:
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

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Change ownership of app directory
RUN chown -R nodejs:nodejs /usr/src/app
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"

# Start application
CMD ["npm", "start"]
Create a .dockerignore file:
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
.travis.yml
Test the Docker build locally:
docker build -t my-node-app:test .
docker run -p 3000:3000 -d --name test-container my-node-app:test
Verify the application is running:
curl http://localhost:3000
curl http://localhost:3000/health
Clean up the test container:
docker stop test-container
docker rm test-container
docker rmi my-node-app:test
Subtask 1.3: Initialize Git Repository
Initialize Git repository:
git init
Create .gitignore file:
nano .gitignore
Add the following content:

node_modules/
npm-debug.log*
.env
.DS_Store
coverage/
.nyc_output/
Add and commit files:
git add .
git commit -m "Initial commit: Dockerized Node.js app"
Task 2: Configure .travis.yml to Build Docker Images Automatically
Subtask 2.1: Create Travis CI Configuration
Create .travis.yml file:
nano .travis.yml
Add the following configuration:

# Specify the programming language
language: node_js

# Specify Node.js version
node_js:
  - "18"

# Enable Docker service
services:
  - docker

# Environment variables
env:
  global:
    - DOCKER_IMAGE_NAME=your-dockerhub-username/docker-travis-ci-lab
    - NODE_ENV=test

# Cache node_modules for faster builds
cache:
  directories:
    - node_modules

# Install dependencies
install:
  - npm ci

# Run tests before building Docker image
script:
  - npm test
  - docker build -t $DOCKER_IMAGE_NAME:$TRAVIS_BUILD_NUMBER .
  - docker build -t $DOCKER_IMAGE_NAME:latest .

# Test the Docker image
before_deploy:
  - docker run -d -p 3000:3000 --name test-container $DOCKER_IMAGE_NAME:latest
  - sleep 10
  - docker exec test-container curl -f http://localhost:3000/health || exit 1
  - docker stop test-container
  - docker rm test-container

# Deploy section (will be configured in next task)
deploy:
  provider: script
  script: echo "Deploy configuration will be added in next task"
  on:
    branch: main

# Notification settings
notifications:
  email:
    on_success: change
    on_failure: always
Subtask 2.2: Set up GitHub Repository
Create a new repository on GitHub:

Go to GitHub.com and sign in
Click the "+" icon and select "New repository"
Name it "docker-travis-ci-lab"
Make it public
Don't initialize with README (we already have files)
Add GitHub remote and push:

git remote add origin https://github.com/YOUR-USERNAME/docker-travis-ci-lab.git
git branch -M main
git push -u origin main
Replace YOUR-USERNAME with your actual GitHub username.

Subtask 2.3: Enable Travis CI
Go to Travis CI:

Visit https://travis-ci.com
Sign in with your GitHub account
Authorize Travis CI to access your repositories
Enable your repository:

Find your "docker-travis-ci-lab" repository
Toggle the switch to enable Travis CI builds
Trigger first build:

echo "# Docker Travis CI Lab" > README.md
git add README.md
git commit -m "Add README to trigger Travis CI build"
git push origin main
Task 3: Push the Built Image to Docker Hub After Successful Build
Subtask 3.1: Set up Docker Hub Credentials
Create Docker Hub repository:

Go to https://hub.docker.com
Sign in to your account
Click "Create Repository"
Name it "docker-travis-ci-lab"
Make it public
Click "Create"
Add Docker Hub credentials to Travis CI:

Go to your Travis CI repository settings
Add environment variables:
DOCKER_USERNAME: Your Docker Hub username
DOCKER_PASSWORD: Your Docker Hub password (mark as hidden)
Subtask 3.2: Update Travis CI Configuration for Deployment
Update .travis.yml file:
nano .travis.yml
Replace the entire content with:

# Specify the programming language
language: node_js

# Specify Node.js version
node_js:
  - "18"

# Enable Docker service
services:
  - docker

# Environment variables
env:
  global:
    - DOCKER_IMAGE_NAME=$DOCKER_USERNAME/docker-travis-ci-lab
    - NODE_ENV=test

# Cache node_modules for faster builds
cache:
  directories:
    - node_modules

# Install dependencies
install:
  - npm ci

# Run tests and build Docker image
script:
  - npm test
  - docker build -t $DOCKER_IMAGE_NAME:$TRAVIS_BUILD_NUMBER .
  - docker build -t $DOCKER_IMAGE_NAME:latest .

# Test the Docker image
before_deploy:
  - docker run -d -p 3000:3000 --name test-container $DOCKER_IMAGE_NAME:latest
  - sleep 10
  - docker exec test-container curl -f http://localhost:3000/health || exit 1
  - docker stop test-container
  - docker rm test-container

# Deploy to Docker Hub
after_success:
  - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
  - docker push $DOCKER_IMAGE_NAME:$TRAVIS_BUILD_NUMBER
  - docker push $DOCKER_IMAGE_NAME:latest

# Only deploy on main branch
branches:
  only:
    - main

# Notification settings
notifications:
  email:
    on_success: change
    on_failure: always
Commit and push the changes:
git add .travis.yml
git commit -m "Configure Docker Hub deployment in Travis CI"
git push origin main
Subtask 3.3: Create Deployment Script (Alternative Method)
For more complex deployment scenarios, create a separate deployment script:

Create deploy.sh script:
nano deploy.sh
Add the following content:

#!/bin/bash

# Exit on any error
set -e

echo "Starting deployment process..."

# Login to Docker Hub
echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

# Tag images
docker tag $DOCKER_IMAGE_NAME:latest $DOCKER_IMAGE_NAME:$TRAVIS_BUILD_NUMBER
docker tag $DOCKER_IMAGE_NAME:latest $DOCKER_IMAGE_NAME:build-$TRAVIS_BUILD_NUMBER

# Push images
echo "Pushing Docker images..."
docker push $DOCKER_IMAGE_NAME:latest
docker push $DOCKER_IMAGE_NAME:$TRAVIS_BUILD_NUMBER
docker push $DOCKER_IMAGE_NAME:build-$TRAVIS_BUILD_NUMBER

echo "Deployment completed successfully!"

# Optional: Clean up local images to save space
docker rmi $DOCKER_IMAGE_NAME:build-$TRAVIS_BUILD_NUMBER || true

echo "Deployment process finished."
Make the script executable:
chmod +x deploy.sh
Update .travis.yml to use the deployment script:
nano .travis.yml
Replace the after_success section:

# Deploy using custom script
after_success:
  - ./deploy.sh
Commit the changes:
git add deploy.sh .travis.yml
git commit -m "Add custom deployment script"
git push origin main
Task 4: Set up Testing Steps Within the Docker Container in Travis CI
Subtask 4.1: Create Multi-Stage Dockerfile for Testing
Update Dockerfile for multi-stage build:
nano Dockerfile
Replace the content with:

# Multi-stage build for testing and production

# Stage 1: Testing stage
FROM node:18-alpine AS testing
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev dependencies)
RUN npm ci

# Copy source code
COPY . .

# Run tests
RUN npm test

# Stage 2: Production stage
FROM node:18-alpine AS production

# Set working directory
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install only production dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy application code
COPY --from=testing /usr/src/app/app.js ./

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Change ownership
RUN chown -R nodejs:nodejs /usr/src/app
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"

# Start application
CMD ["npm", "start"]
Subtask 4.2: Add Integration Tests
Create integration test file:
nano integration.test.js
Add the following content:

const request = require('supertest');
const { app, server } = require('./app');

describe('Integration Tests', () => {
    afterAll(() => {
        server.close();
    });

    test('Application should start successfully', async () => {
        const response = await request(app).get('/');
        expect(response.status).toBe(200);
    });

    test('Health endpoint should return correct format', async () => {
        const response = await request(app).get('/health');
        expect(response.status).toBe(200);
        expect(response.body).toHaveProperty('status');
        expect(response.body).toHaveProperty('uptime');
        expect(typeof response.body.uptime).toBe('number');
    });

    test('Root endpoint should return timestamp', async () => {
        const response = await request(app).get('/');
        expect(response.body).toHaveProperty('timestamp');
        expect(new Date(response.body.timestamp)).toBeInstanceOf(Date);
    });

    test('Application should handle invalid routes', async () => {
        const response = await request(app).get('/nonexistent');
        expect(response.status).toBe(404);
    });
});
Subtask 4.3: Update Travis CI Configuration for Container Testing
Update .travis.yml with comprehensive testing:
nano .travis.yml
Replace with the following enhanced configuration:

# Specify the programming language
language: node_js

# Specify Node.js version
node_js:
  - "18"

# Enable Docker service
services:
  - docker

# Environment variables
env:
  global:
    - DOCKER_IMAGE_NAME=$DOCKER_USERNAME/docker-travis-ci-lab
    - NODE_ENV=test

# Cache node_modules for faster builds
cache:
  directories:
    - node_modules

# Install dependencies
install:
  - npm ci

# Run tests in multiple stages
script:
  # Stage 1: Run unit tests locally
  - echo "Running unit tests..."
  - npm test
  
  # Stage 2: Build Docker image (includes testing stage)
  - echo "Building Docker image with tests..."
  - docker build --target testing -t $DOCKER_IMAGE_NAME:test .
  
  # Stage 3: Build production image
  - echo "Building production Docker image..."
  - docker build --target production -t $DOCKER_IMAGE_NAME:$TRAVIS_BUILD_NUMBER .
  - docker build --target production -t $DOCKER_IMAGE_NAME:latest .

# Test the production Docker image
before_deploy:
  - echo "Testing production Docker image..."
  
  # Start container
  - docker run -d -p 3000:3000 --name test-container $DOCKER_IMAGE_NAME:latest
  
  # Wait for container to start
  - sleep 15
  
  # Test health endpoint
  - docker exec test-container curl -f http://localhost:3000/health || exit 1
  
  # Test main endpoint
  - docker exec test-container curl -f http://localhost:3000 || exit 1
  
  # Check container logs
  - docker logs test-container
  
  # Test from host machine
  - curl -f http://localhost:3000/health || exit 1
  
  # Clean up
  - docker stop test-container
  - docker rm test-container

# Deploy to Docker Hub
after_success:
  - echo "Deploying to Docker Hub..."
  - echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
  - docker push $DOCKER_IMAGE_NAME:$TRAVIS_BUILD_NUMBER
  - docker push $DOCKER_IMAGE_NAME:latest
  - echo "Deployment completed successfully!"

# Only build main branch and pull requests
branches:
  only:
    - main

# Notification settings
notifications:
  email:
    on_success: change
    on_failure: always
Subtask 4.4: Add Docker Compose for Local Testing
Create docker-compose.yml for local development:
nano docker-compose.yml
Add the following content:

version: '3.8'

services:
  app:
    build:
      context: .
      target: production
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped

  test:
    build:
      context: .
      target: testing
    environment:
      - NODE_ENV=test
    command: npm test
Create docker-compose.test.yml for CI testing:
nano docker-compose.test.yml
Add the following content:

version: '3.8'

services:
  sut:
    build:
      context: .
      target: testing
    environment:
      - NODE_ENV=test
    command: npm test
Commit all changes:
git add .
git commit -m "Add comprehensive Docker testing setup"
git push origin main
Task 5: Monitor Travis CI Build Logs and Troubleshoot if Necessary
Subtask 5.1: Understanding Travis CI Build Logs
Access Travis CI Dashboard:

Go to https://travis-ci.com
Navigate to your repository
Click on the latest build
Understanding Build Stages: The build log will show these stages:

Install: Installing dependencies
Script: Running tests and building Docker images
Before Deploy: Testing the Docker container
After Success: Deploying to Docker Hub
Subtask 5.2: Common Issues and Troubleshooting
Create a troubleshooting guide:
nano TROUBLESHOOTING.md
Add the following content:

# Travis CI Docker Build Troubleshooting Guide

## Common Issues and Solutions

### 1. Docker Build Failures

**Issue**: Docker build fails with "COPY failed" error
**Solution**: Check .dockerignore file and ensure all required files are included

**Issue**: Node.js dependencies installation fails
**Solution**: Ensure package.json and package-lock.json are properly committed

### 2. Test Failures

**Issue**: Tests pass locally but fail in Travis CI
**Solution**: Check environment variables and ensure test database/services are available

**Issue**: Container health check fails
**Solution**: Increase sleep time in before_deploy section or adjust health check timeout

### 3. Docker Hub Push Failures

**Issue**: Authentication failed when pushing to Docker Hub
**Solution**: Verify DOCKER_USERNAME and DOCKER_PASSWORD environment variables in Travis CI settings

**Issue**: Repository not found error
**Solution**: Ensure Docker Hub repository exists and username is correct

### 4. Build Timeout Issues

**Issue**: Build times out during Docker operations
**Solution**: Optimize Dockerfile using multi-stage builds and .dockerignore

### 5. Environment Variable Issues

**Issue**: Environment variables not available in build
**Solution**: Check Travis CI repository settings and ensure variables are properly set
Subtask 5.3: Add Build Status and Monitoring
Add build status badge to README:
nano README.md
Replace the content with:

# Docker Travis CI Lab

[![Build Status](https://travis-ci.com/YOUR-USERNAME/docker-travis-ci-lab.svg?branch=main)](https://travis-ci.com/YOUR-USERNAME/docker-travis-ci-lab)
[![Docker Hub](https://img.shields.io/docker/pulls/YOUR-DOCKERHUB-USERNAME/docker-travis-ci-lab.svg)](https://hub.docker.com/r/YOUR-DOCKERHUB-USERNAME/docker-travis-ci-lab)

A sample Node.js application demonstrating Docker integration with Travis CI for continuous integration and deployment.

## Features

- Dockerized Node.js application
- Automated testing with Jest
- Multi-stage Docker builds
- Continuous integration with Travis CI
- Automated deployment to Docker Hub
- Health checks and monitoring

## Local Development

### Prerequisites
- Docker
- Node.js 18+
- npm

### Running Locally

1. Clone the repository:
```bash
git clone https://github.com/YOUR-USERNAME/docker-travis-ci-lab.git
cd docker-travis-ci-lab
Install dependencies:
npm install
Run tests:
npm test
Run with Docker:
docker-compose up --build
Access the application:
Main endpoint: http://localhost:3000
Health check: http://localhost:3000/health
CI/CD Pipeline
This project uses Travis CI for continuous integration and deployment:

Test Stage: Runs unit and integration tests
Build Stage: Creates Docker images
Test Stage: Tests the Docker container
Deploy Stage: Pushes images to Docker Hub
Docker Hub
The built images are available at: https://hub.docker.com/r/YOUR-DOCKERHUB-USERNAME/docker-travis-ci-lab


Replace `YOUR-USERNAME` and `YOUR-DOCKERHUB-USERNAME` with your actual usernames.

### Subtask 5.4: Create Monitoring Script

1. **Create a monitoring script**:
```bash
nano monitor-build.sh
Add the following content:

#!/bin/bash

# Travis CI Build Monitor Script

REPO_SLUG="YOUR-USERNAME/docker-travis-ci-lab"
TRAVIS_API="https://api.travis-ci.com"

echo "Monitoring Travis CI builds for $REPO_SLUG"
echo "=========================================="

# Get latest build status
curl -s -H "Travis-API-Version: 3" \
     "$TRAVIS_API/repo/$REPO_SLUG/builds?limit=5" | \
     jq -r '.builds[] | "Build #\(.number): \(.state) (\(.branch)) - \(.started_at)"'

echo ""
echo "For detailed logs, visit:"
echo "https://travis-ci.com/$REPO_SLUG"
Make it executable:
chmod +x monitor-build.sh
Subtask 5.5: Test the Complete Pipeline
Make a small change to trigger a build:
echo "console.log('Build triggered at: ' + new Date());" >> app.js
Commit and push:
git add .
git commit -m "Trigger Travis CI build for testing"
git push origin main
Monitor the build:

Go to Travis CI dashboard
Watch the build progress
Check each stage for any issues
Verify deployment:

Check Docker Hub for the new image
Pull and test the image locally:
docker pull YOUR-DOCKERHUB-USERNAME/docker-travis-ci-lab:latest
docker run -p 3000:3000 -d YOUR-DOCKERHUB-USERNAME/docker-travis-ci-lab:latest
curl http://localhost:3000
Subtask 5.6: Set up Build Notifications
Update .travis.yml with Slack notifications (optional):
notifications:
  email:
    on_success: change
    on_failure: always
  slack:
    rooms:
      - your-workspace:your-token#your-channel
    on_success: change
    on_failure: always
    template:
      - "Build <%{build_url}|#%{build_number}> (<%{compare_url}|%{commit}>) of %{repository_slug}@%{branch} by %{author} %{result} in %{duration}"
Final commit:
git add .
git commit -m "Complete Travis CI Docker integration setup"
git push origin main
Conclusion
Congratulations! You have successfully completed Lab 97: Docker for Continuous Integration - Integrating Docker with Travis CI.

What You Accomplished
In this lab, you have:

Created a Dockerized Web Application: Built a Node.js application with proper containerization using multi-stage Docker builds
Set up Travis CI Integration: Configured automated builds triggered by Git commits
Implemented Automated Testing: Created comprehensive test suites that run both locally and in Docker containers
Configured Automated Deployment: Set up automatic pushing of Docker images to Docker Hub after successful builds
Established Monitoring and Troubleshooting: Created monitoring tools and troubleshooting guides for the CI/CD pipeline
Why This Matters
This lab demonstrates essential DevOps practices that are crucial in modern software development:

Continuous Integration: Automatically testing code changes prevents bugs from reaching production
Containerization: Docker ensures consistent environments across development, testing, and production
Automated Deployment: Reduces manual errors and speeds up the release process
Infrastructure as Code: Configuration files like .travis.yml make the build process reproducible and version-controlled
Key Takeaways
Docker Multi-Stage Builds: Optimize image size and separate testing from production environments
CI/CD Pipeline Design: Structure builds with clear stages for testing, building, and deployment
Environment Management: Properly handle secrets and environment variables in CI systems
Monitoring and Alerting: Set up proper monitoring to quickly identify and resolve issues
Next Steps
To further enhance your CI/CD skills, consider:

Exploring other CI platforms like GitHub Actions or GitLab CI
Implementing more sophisticated deployment strategies (blue-green, canary)
Adding security scanning to your Docker images
Integrating with cloud platforms like AWS, Azure, or Google Cloud
Setting up monitoring and logging for production applications
This foundation in Docker and Travis CI will serve you well as you continue to build and deploy modern applications using DevOps best practices.
