Lab 65: Docker and CI/CD - Integrating Docker with GitHub Actions for CI/CD
Objectives
By the end of this lab, students will be able to:

Set up a GitHub repository with a Docker-based application
Create and configure GitHub Actions workflows for Docker CI/CD
Build and push Docker images automatically to Docker Hub
Implement automated testing in CI/CD pipelines
Monitor pipeline executions and troubleshoot common failures
Understand the integration between Docker and GitHub Actions for continuous deployment
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts and commands
Familiarity with Git and GitHub
Basic knowledge of YAML syntax
Understanding of web applications (Node.js basics helpful)
A GitHub account (free tier is sufficient)
A Docker Hub account (free tier is sufficient)
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. The machine comes with:

Docker Engine pre-installed
Git pre-configured
Node.js and npm installed
Text editors (nano, vim) available
All necessary development tools ready to use
No need to build your own VM or install additional software!

Lab Tasks
Task 1: Set up a GitHub Repository with a Docker-based Application
Subtask 1.1: Create a Sample Node.js Application
First, let's create a simple web application that we'll containerize and deploy.

Create a new directory for your project:
mkdir docker-cicd-lab
cd docker-cicd-lab
Initialize a new Node.js project:
npm init -y
Install Express.js as a dependency:
npm install express
Create the main application file:
nano app.js
Add the following content to app.js:

const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

// Middleware to parse JSON
app.use(express.json());

// Health check endpoint
app.get('/health', (req, res) => {
    res.status(200).json({
        status: 'healthy',
        timestamp: new Date().toISOString(),
        version: '1.0.0'
    });
});

// Main route
app.get('/', (req, res) => {
    res.json({
        message: 'Welcome to Docker CI/CD Lab!',
        environment: process.env.NODE_ENV || 'development',
        version: '1.0.0'
    });
});

// API endpoint for testing
app.get('/api/users', (req, res) => {
    const users = [
        { id: 1, name: 'John Doe', email: 'john@example.com' },
        { id: 2, name: 'Jane Smith', email: 'jane@example.com' }
    ];
    res.json(users);
});

app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});

module.exports = app;
Subtask 1.2: Create a Dockerfile
Create a Dockerfile:
nano Dockerfile
Add the following content:

# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory in container
WORKDIR /usr/src/app

# Copy package.json and package-lock.json
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
    CMD node healthcheck.js

# Start the application
CMD ["node", "app.js"]
Create a health check script:
nano healthcheck.js
Add the following content:

const http = require('http');

const options = {
    hostname: 'localhost',
    port: 3000,
    path: '/health',
    method: 'GET'
};

const req = http.request(options, (res) => {
    if (res.statusCode === 200) {
        process.exit(0);
    } else {
        process.exit(1);
    }
});

req.on('error', () => {
    process.exit(1);
});

req.end();
Subtask 1.3: Create Additional Configuration Files
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
.coverage
Create a basic test file:
mkdir tests
nano tests/app.test.js
Add the following content:

const request = require('supertest');
const app = require('../app');

describe('App Tests', () => {
    test('GET / should return welcome message', async () => {
        const response = await request(app).get('/');
        expect(response.status).toBe(200);
        expect(response.body.message).toBe('Welcome to Docker CI/CD Lab!');
    });

    test('GET /health should return health status', async () => {
        const response = await request(app).get('/health');
        expect(response.status).toBe(200);
        expect(response.body.status).toBe('healthy');
    });

    test('GET /api/users should return users array', async () => {
        const response = await request(app).get('/api/users');
        expect(response.status).toBe(200);
        expect(Array.isArray(response.body)).toBe(true);
        expect(response.body.length).toBe(2);
    });
});
Install testing dependencies:
npm install --save-dev jest supertest
Update package.json to include test script:
nano package.json
Modify the scripts section:

{
  "scripts": {
    "start": "node app.js",
    "test": "jest",
    "test:watch": "jest --watch",
    "dev": "node app.js"
  }
}
Subtask 1.4: Initialize Git Repository and Push to GitHub
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
git commit -m "Initial commit: Docker CI/CD lab setup"
Create a new repository on GitHub:

Go to GitHub.com and log in
Click the "+" icon and select "New repository"
Name it "docker-cicd-lab"
Keep it public
Don't initialize with README (we already have files)
Click "Create repository"
Connect local repository to GitHub:

git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/docker-cicd-lab.git
git push -u origin main
Replace YOUR_USERNAME with your actual GitHub username.

Task 2: Create GitHub Actions Workflow for Docker CI/CD
Subtask 2.1: Create Workflow Directory Structure
Create the GitHub Actions workflow directory:
mkdir -p .github/workflows
Create the main Docker workflow file:
nano .github/workflows/docker.yml
Subtask 2.2: Configure Basic Workflow Structure
Add the following content to docker.yml:

name: Docker CI/CD Pipeline

# Trigger the workflow on push and pull requests to main branch
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

# Environment variables
env:
  DOCKER_IMAGE: your-dockerhub-username/docker-cicd-lab
  NODE_VERSION: '18'

jobs:
  # Job 1: Run tests
  test:
    runs-on: ubuntu-latest
    name: Run Tests
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run tests
      run: npm test
      
    - name: Generate test coverage
      run: npm test -- --coverage
      
  # Job 2: Build and push Docker image
  build-and-push:
    runs-on: ubuntu-latest
    needs: test
    name: Build and Push Docker Image
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
      
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
        
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.DOCKER_IMAGE }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
          
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        platforms: linux/amd64,linux/arm64
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        
  # Job 3: Security scan
  security-scan:
    runs-on: ubuntu-latest
    needs: build-and-push
    name: Security Scan
    
    steps:
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ env.DOCKER_IMAGE }}:latest
        format: 'sarif'
        output: 'trivy-results.sarif'
        
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
Important: Replace your-dockerhub-username with your actual Docker Hub username in the DOCKER_IMAGE environment variable.

Task 3: Configure Docker Hub Integration
Subtask 3.1: Set up Docker Hub Repository
Log in to Docker Hub:

Go to hub.docker.com
Log in with your credentials
Create a new repository:

Click "Create Repository"
Name it "docker-cicd-lab"
Set visibility to "Public"
Click "Create"
Subtask 3.2: Configure GitHub Secrets
Add Docker Hub credentials to GitHub:

Go to your GitHub repository
Click "Settings" tab
Click "Secrets and variables" → "Actions"
Click "New repository secret"
Add DOCKER_USERNAME secret:

Name: DOCKER_USERNAME
Secret: Your Docker Hub username
Click "Add secret"
Add DOCKER_PASSWORD secret:

Name: DOCKER_PASSWORD
Secret: Your Docker Hub password or access token
Click "Add secret"
Security Note: It's recommended to use Docker Hub access tokens instead of passwords. Create one in Docker Hub Settings → Security → Access Tokens.

Task 4: Add Advanced Pipeline Features
Subtask 4.1: Create Environment-Specific Configurations
Create a docker-compose file for local development:
nano docker-compose.yml
Add the following content:

version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    volumes:
      - .:/usr/src/app
      - /usr/src/app/node_modules
    command: npm run dev
    
  app-prod:
    build: .
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=production
    restart: unless-stopped
Subtask 4.2: Enhance the Workflow with Deployment
Update the workflow file to include deployment:
nano .github/workflows/docker.yml
Add the following job at the end of the file:

  # Job 4: Deploy to staging (simulation)
  deploy-staging:
    runs-on: ubuntu-latest
    needs: [build-and-push, security-scan]
    name: Deploy to Staging
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Deploy to staging environment
      run: |
        echo "🚀 Deploying to staging environment..."
        echo "Image: ${{ env.DOCKER_IMAGE }}:latest"
        echo "Deployment would happen here in a real scenario"
        
        # Simulate deployment verification
        echo "✅ Deployment verification passed"
        echo "🌐 Application available at: https://staging.example.com"
        
    - name: Run smoke tests
      run: |
        echo "🧪 Running smoke tests..."
        # In a real scenario, you would run actual smoke tests here
        echo "✅ Smoke tests passed"
        
    - name: Notify deployment success
      run: |
        echo "📧 Sending deployment notification..."
        echo "✅ Deployment to staging completed successfully"
Subtask 4.3: Add Workflow Status Badges
Create a comprehensive README file:
nano README.md
Add the following content:

# Docker CI/CD Lab

![Docker CI/CD Pipeline](https://github.com/YOUR_USERNAME/docker-cicd-lab/workflows/Docker%20CI/CD%20Pipeline/badge.svg)
![Docker Image Size](https://img.shields.io/docker/image-size/YOUR_DOCKERHUB_USERNAME/docker-cicd-lab/latest)
![Docker Pulls](https://img.shields.io/docker/pulls/YOUR_DOCKERHUB_USERNAME/docker-cicd-lab)

A sample Node.js application demonstrating Docker integration with GitHub Actions for CI/CD.

## Features

- 🐳 Dockerized Node.js application
- 🔄 Automated CI/CD with GitHub Actions
- 🧪 Automated testing
- 🔒 Security scanning with Trivy
- 📦 Multi-platform Docker builds
- 🚀 Automated deployment pipeline

## Quick Start

### Local Development

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/docker-cicd-lab.git
cd docker-cicd-lab

# Install dependencies
npm install

# Run tests
npm test

# Start the application
npm start
Docker
# Build the image
docker build -t docker-cicd-lab .

# Run the container
docker run -p 3000:3000 docker-cicd-lab
Docker Compose
# Development environment
docker-compose up app

# Production environment
docker-compose up app-prod
API Endpoints
GET / - Welcome message
GET /health - Health check
GET /api/users - Sample users data
CI/CD Pipeline
The pipeline includes:

Testing - Runs unit tests and generates coverage
Building - Builds multi-platform Docker images
Security - Scans for vulnerabilities
Deployment - Deploys to staging environment
Environment Variables
NODE_ENV - Environment (development/production)
PORT - Server port (default: 3000)
Contributing
Fork the repository
Create a feature branch
Make your changes
Run tests
Submit a pull request

Replace `YOUR_USERNAME` and `YOUR_DOCKERHUB_USERNAME` with your actual usernames.

### Task 5: Monitor Pipeline Executions and Troubleshoot Failures

#### Subtask 5.1: Commit and Push Changes

1. **Add all new files:**

```bash
git add .
git commit -m "Add comprehensive CI/CD pipeline with Docker integration"
git push origin main
Monitor the pipeline execution:
Go to your GitHub repository
Click the "Actions" tab
Watch the workflow execution in real-time
Subtask 5.2: Understanding Pipeline Monitoring
Key areas to monitor:

Workflow Overview:

Total execution time
Success/failure status
Individual job status
Job Details:

Step-by-step execution logs
Error messages and stack traces
Resource usage information
Artifacts and Outputs:

Test results
Coverage reports
Built Docker images
Subtask 5.3: Common Troubleshooting Scenarios
Test Failures:
If tests fail, check the test logs:

# Run tests locally to debug
npm test -- --verbose
Docker Build Failures:
Common issues and solutions:

# Check Dockerfile syntax
docker build --no-cache -t test-build .

# Verify base image availability
docker pull node:18-alpine
Docker Hub Push Failures:
Verify credentials and permissions:

# Test Docker Hub login locally
docker login
docker tag your-image your-dockerhub-username/docker-cicd-lab:test
docker push your-dockerhub-username/docker-cicd-lab:test
Subtask 5.4: Create a Workflow for Pull Requests
Create a separate workflow for PR validation:
nano .github/workflows/pr-validation.yml
Add the following content:

name: Pull Request Validation

on:
  pull_request:
    branches: [ main ]

jobs:
  validate:
    runs-on: ubuntu-latest
    name: Validate Pull Request
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        
    - name: Install dependencies
      run: npm ci
      
    - name: Run linting
      run: |
        npx eslint . --ext .js --ignore-path .gitignore || echo "Linting completed with warnings"
        
    - name: Run tests with coverage
      run: npm test -- --coverage --watchAll=false
      
    - name: Build Docker image (no push)
      run: |
        docker build -t pr-validation:${{ github.event.number }} .
        echo "✅ Docker image built successfully"
        
    - name: Test Docker container
      run: |
        docker run -d --name test-container -p 3000:3000 pr-validation:${{ github.event.number }}
        sleep 10
        curl -f http://localhost:3000/health || exit 1
        docker stop test-container
        docker rm test-container
        echo "✅ Container health check passed"
Subtask 5.5: Set up Monitoring and Notifications
Create a workflow status notification:
nano .github/workflows/notify.yml
Add the following content:

name: Workflow Notifications

on:
  workflow_run:
    workflows: ["Docker CI/CD Pipeline"]
    types:
      - completed

jobs:
  notify:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion != 'cancelled' }}
    
    steps:
    - name: Notify on success
      if: ${{ github.event.workflow_run.conclusion == 'success' }}
      run: |
        echo "🎉 Pipeline succeeded!"
        echo "✅ All tests passed"
        echo "🐳 Docker image pushed successfully"
        echo "🚀 Deployment completed"
        
    - name: Notify on failure
      if: ${{ github.event.workflow_run.conclusion == 'failure' }}
      run: |
        echo "❌ Pipeline failed!"
        echo "🔍 Check the logs for details"
        echo "📧 Notification sent to development team"
Subtask 5.6: Test the Complete Pipeline
Make a small change to trigger the pipeline:
nano app.js
Update the version number:

// Change version from '1.0.0' to '1.0.1'
version: '1.0.1'
Commit and push the change:
git add app.js
git commit -m "Update version to 1.0.1"
git push origin main
Monitor the pipeline execution:

Watch the Actions tab in GitHub
Verify all jobs complete successfully
Check Docker Hub for the new image
Create a test pull request:

git checkout -b feature/test-pr
echo "# Test PR" >> test-file.md
git add test-file.md
git commit -m "Add test file for PR validation"
git push origin feature/test-pr
Create a pull request on GitHub and observe the PR validation workflow.

Troubleshooting Common Issues
Issue 1: Docker Hub Authentication Failure
Symptoms:

Error: "unauthorized: authentication required"
Build succeeds but push fails
Solutions:

Verify Docker Hub credentials in GitHub secrets
Check if Docker Hub username is correct
Use access token instead of password
Ensure repository exists on Docker Hub
Issue 2: Test Failures in CI
Symptoms:

Tests pass locally but fail in CI
Timeout errors in GitHub Actions
Solutions:

Check Node.js version compatibility
Verify all dependencies are in package.json
Add longer timeout for tests
Check for environment-specific issues
Issue 3: Docker Build Failures
Symptoms:

"COPY failed" errors
Base image pull failures
Permission denied errors
Solutions:

Verify .dockerignore is properly configured
Check Dockerfile syntax
Ensure base image is available
Review file permissions and ownership
Issue 4: Workflow Not Triggering
Symptoms:

Push to main branch doesn't trigger workflow
Workflow file exists but doesn't run
Solutions:

Check YAML syntax in workflow file
Verify branch names match trigger conditions
Ensure workflow file is in correct location
Check repository permissions
Conclusion
Congratulations! You have successfully completed Lab 65: Docker and CI/CD Integration with GitHub Actions.

What You Accomplished
In this lab, you have:

Created a complete Docker-based application with a Node.js web server, proper health checks, and comprehensive testing
Set up automated CI/CD pipelines using GitHub Actions that trigger on code changes
Integrated Docker Hub for automated image building and distribution
Implemented security scanning to identify vulnerabilities in your Docker images
Added monitoring and notification systems to track pipeline health and failures
Learned troubleshooting techniques for common CI/CD and Docker integration issues
Why This Matters
This integration of Docker with GitHub Actions represents modern DevOps practices that are essential in today's software development landscape:

Automation reduces manual errors and speeds up deployment cycles
Consistency ensures your application runs the same way across all environments
Security scanning helps identify vulnerabilities before they reach production
Scalability allows teams to deploy multiple times per day with confidence
Monitoring provides visibility into the health of your deployment pipeline
Next Steps
To further enhance your CI/CD pipeline, consider:

Adding integration tests with external services
Implementing blue-green deployments
Setting up monitoring and alerting for production deployments
Adding automated rollback capabilities
Integrating with cloud platforms like AWS, Azure, or Google Cloud
This foundation will serve you well as you continue to build more complex applications and deployment strategies in your DevOps journey.
