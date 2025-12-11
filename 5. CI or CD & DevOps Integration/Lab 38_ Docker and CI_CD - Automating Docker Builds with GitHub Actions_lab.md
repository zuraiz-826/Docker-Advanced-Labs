Lab 38: Docker and CI/CD - Automating Docker Builds with GitHub Actions
Lab Objectives
By the end of this lab, you will be able to:

• Set up a GitHub repository with automated Docker builds • Create and configure GitHub Actions workflows for CI/CD • Automate Docker image builds triggered by code pushes • Push Docker images to Docker Hub automatically • Integrate Docker Compose testing into your CI/CD pipeline • Understand the fundamentals of containerized application deployment

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker containers and images • Familiarity with Git version control • Basic knowledge of YAML file structure • A GitHub account (free account is sufficient) • A Docker Hub account (free account is sufficient) • Understanding of basic Linux command line operations

Lab Environment
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your cloud machine includes: • Docker Engine pre-installed • Git client configured • Text editors (nano, vim) • All necessary development tools

Lab Tasks Overview
This lab consists of 5 main tasks:

Set up a GitHub repository for your project
Create a GitHub Actions workflow file
Configure automated Docker image builds
Set up automatic Docker Hub publishing
Add Docker Compose testing to the pipeline
Task 1: Set up a GitHub Repository for Your Project
Step 1.1: Create a New GitHub Repository
Open your web browser and navigate to GitHub.com
Sign in to your GitHub account
Click the New button or the + icon in the top right corner
Select New repository
Configure your repository:
Repository name: docker-cicd-demo
Description: Docker CI/CD automation with GitHub Actions
Visibility: Public (required for free GitHub Actions)
Initialize with README: Check this box
Add .gitignore: Select Node (we'll create a simple Node.js app)
Choose a license: MIT License
Click Create repository
Step 1.2: Clone the Repository to Your Cloud Machine
In your cloud machine terminal, navigate to your home directory:
cd ~
Clone your newly created repository (replace YOUR_USERNAME with your GitHub username):
git clone https://github.com/YOUR_USERNAME/docker-cicd-demo.git
Navigate into the repository directory:
cd docker-cicd-demo
Verify the repository contents:
ls -la
Step 1.3: Create a Simple Node.js Application
Create a basic package.json file:
cat > package.json << 'EOF'
{
  "name": "docker-cicd-demo",
  "version": "1.0.0",
  "description": "A simple Node.js app for Docker CI/CD demonstration",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"Running tests...\" && node test.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "author": "Your Name",
  "license": "MIT"
}
EOF
Create the main application file app.js:
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Docker CI/CD Demo!',
    timestamp: new Date().toISOString(),
    version: '1.0.0'
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
EOF
Create a simple test file test.js:
cat > test.js << 'EOF'
const http = require('http');

// Simple test to check if the server responds
const options = {
  hostname: 'localhost',
  port: 3000,
  path: '/health',
  method: 'GET'
};

console.log('Starting basic health check test...');

const req = http.request(options, (res) => {
  console.log(`Status Code: ${res.statusCode}`);
  
  let data = '';
  res.on('data', (chunk) => {
    data += chunk;
  });
  
  res.on('end', () => {
    try {
      const response = JSON.parse(data);
      if (response.status === 'healthy') {
        console.log('✅ Health check passed!');
        process.exit(0);
      } else {
        console.log('❌ Health check failed!');
        process.exit(1);
      }
    } catch (error) {
      console.log('❌ Invalid response format');
      process.exit(1);
    }
  });
});

req.on('error', (error) => {
  console.log('❌ Test failed:', error.message);
  process.exit(1);
});

req.end();
EOF
Create a Dockerfile for the application:
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
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"

# Start the application
CMD ["npm", "start"]
EOF
Task 2: Create a GitHub Actions Workflow File
Step 2.1: Create the Workflows Directory Structure
Create the GitHub Actions directory structure:
mkdir -p .github/workflows
Verify the directory was created:
ls -la .github/
Step 2.2: Create the Docker Workflow File
Create the main workflow file docker.yml:
cat > .github/workflows/docker.yml << 'EOF'
name: Docker CI/CD Pipeline

# Trigger the workflow on push to main branch and pull requests
on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

# Environment variables available to all jobs
env:
  DOCKER_IMAGE_NAME: docker-cicd-demo
  DOCKER_REGISTRY: docker.io

jobs:
  # Job 1: Build and Test
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    # Step 1: Checkout the repository code
    - name: Checkout repository
      uses: actions/checkout@v4
    
    # Step 2: Set up Node.js environment for testing
    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
    
    # Step 3: Install dependencies
    - name: Install dependencies
      run: npm install
    
    # Step 4: Run application tests
    - name: Run tests
      run: |
        # Start the application in background
        npm start &
        APP_PID=$!
        
        # Wait for application to start
        sleep 5
        
        # Run tests
        npm test
        
        # Clean up
        kill $APP_PID
    
    # Step 5: Set up Docker Buildx
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    # Step 6: Build Docker image for testing
    - name: Build Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: false
        tags: ${{ env.DOCKER_IMAGE_NAME }}:test
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    # Step 7: Test Docker image
    - name: Test Docker image
      run: |
        # Run container in background
        docker run -d -p 3000:3000 --name test-container ${{ env.DOCKER_IMAGE_NAME }}:test
        
        # Wait for container to start
        sleep 10
        
        # Test the container
        curl -f http://localhost:3000/health || exit 1
        
        # Clean up
        docker stop test-container
        docker rm test-container

  # Job 2: Build and Push to Docker Hub (only on main branch)
  build-and-push:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'
    
    steps:
    # Step 1: Checkout repository
    - name: Checkout repository
      uses: actions/checkout@v4
    
    # Step 2: Set up Docker Buildx
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    # Step 3: Login to Docker Hub
    - name: Login to Docker Hub
      uses: docker/login-action@v3
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    # Step 4: Extract metadata for tags and labels
    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ secrets.DOCKER_USERNAME }}/${{ env.DOCKER_IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    # Step 5: Build and push Docker image
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
EOF
Step 2.3: Create a Docker Compose File for Testing
Create a docker-compose.yml file for local testing:
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
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

  # Optional: Add a simple nginx reverse proxy
  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    restart: unless-stopped
EOF
Create a simple nginx configuration:
cat > nginx.conf << 'EOF'
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
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        location /health {
            proxy_pass http://app/health;
        }
    }
}
EOF
Task 3: Configure GitHub Actions to Build Docker Images on Push Events
Step 3.1: Commit and Push Your Initial Code
Add all files to git:
git add .
Commit the changes:
git commit -m "Initial commit: Add Node.js app and GitHub Actions workflow"
Push to GitHub:
git push origin main
Step 3.2: Verify the Workflow Execution
Go to your GitHub repository in your web browser
Click on the Actions tab
You should see your workflow Docker CI/CD Pipeline running
Click on the workflow run to see detailed logs
Note: The workflow will fail at the Docker Hub push step because we haven't configured the secrets yet. This is expected and we'll fix it in the next task.

Step 3.3: Understanding the Workflow Structure
The workflow we created has two main jobs:

build-and-test job:

Runs on every push and pull request
Sets up Node.js environment
Installs dependencies and runs tests
Builds Docker image for testing
Tests the Docker container
build-and-push job:

Only runs on main branch pushes
Requires the build-and-test job to succeed
Logs into Docker Hub
Builds and pushes the image with proper tags
Task 4: Push the Image to Docker Hub Automatically After Each Successful Build
Step 4.1: Create Docker Hub Repository
Go to hub.docker.com and sign in to your account
Click Create Repository
Set the repository name to: docker-cicd-demo
Set visibility to Public
Click Create
Step 4.2: Configure GitHub Secrets
In your GitHub repository, go to Settings tab
In the left sidebar, click Secrets and variables → Actions
Click New repository secret
Add the first secret:
Name: DOCKER_USERNAME
Secret: Your Docker Hub username
Click Add secret
Add the second secret:
Name: DOCKER_PASSWORD
Secret: Your Docker Hub password or access token
Click Add secret
Security Best Practice: Instead of using your password, create a Docker Hub access token:

Go to Docker Hub → Account Settings → Security
Click New Access Token
Give it a name like "GitHub Actions"
Copy the token and use it as your DOCKER_PASSWORD secret
Step 4.3: Test the Complete Pipeline
Make a small change to trigger the workflow. Edit the app.js file:
sed -i 's/version: '\''1.0.0'\''/version: '\''1.0.1'\''/' app.js
Commit and push the change:
git add app.js
git commit -m "Update version to 1.0.1"
git push origin main
Go to GitHub Actions and watch the workflow:
The build-and-test job should complete successfully
The build-and-push job should now also complete successfully
Check your Docker Hub repository to see the pushed image
Step 4.4: Verify the Docker Image
Check your Docker Hub repository at https://hub.docker.com/r/YOUR_USERNAME/docker-cicd-demo
You should see tags like:
latest
main-<commit-sha>
Branch-specific tags
Task 5: Add a Docker Compose Step to Test the Container in the Pipeline
Step 5.1: Enhance the Workflow with Docker Compose Testing
Update the workflow file to include Docker Compose testing:
cat > .github/workflows/docker.yml << 'EOF'
name: Docker CI/CD Pipeline

on:
  push:
    branches: [ main, master ]
  pull_request:
    branches: [ main, master ]

env:
  DOCKER_IMAGE_NAME: docker-cicd-demo
  DOCKER_REGISTRY: docker.io

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout repository
      uses: actions/checkout@v4
    
    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm install
    
    - name: Run unit tests
      run: |
        npm start &
        APP_PID=$!
        sleep 5
        npm test
        kill $APP_PID
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Build Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: false
        tags: ${{ env.DOCKER_IMAGE_NAME }}:test
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    - name: Test Docker image
      run: |
        docker run -d -p 3000:3000 --name test-container ${{ env.DOCKER_IMAGE_NAME }}:test
        sleep 10
        curl -f http://localhost:3000/health || exit 1
        docker stop test-container
        docker rm test-container
    
    # New Docker Compose testing steps
    - name: Create test environment with Docker Compose
      run: |
        # Update docker-compose.yml to use the test image
        sed -i 's/build: \./image: ${{ env.DOCKER_IMAGE_NAME }}:test/' docker-compose.yml
    
    - name: Start services with Docker Compose
      run: |
        docker-compose up -d
        sleep 15
    
    - name: Test Docker Compose services
      run: |
        # Test direct app access
        curl -f http://localhost:3000/health || exit 1
        echo "✅ Direct app access successful"
        
        # Test nginx proxy access
        curl -f http://localhost:8080/health || exit 1
        echo "✅ Nginx proxy access successful"
        
        # Test app functionality
        response=$(curl -s http://localhost:3000/)
        echo "App response: $response"
        
        # Verify response contains expected content
        if echo "$response" | grep -q "Hello from Docker CI/CD Demo"; then
          echo "✅ App functionality test passed"
        else
          echo "❌ App functionality test failed"
          exit 1
        fi
    
    - name: Check service logs
      if: always()
      run: |
        echo "=== App Service Logs ==="
        docker-compose logs app
        echo "=== Nginx Service Logs ==="
        docker-compose logs nginx
    
    - name: Cleanup Docker Compose
      if: always()
      run: |
        docker-compose down -v
        docker system prune -f

  build-and-push:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' || github.ref == 'refs/heads/master'
    
    steps:
    - name: Checkout repository
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
        images: ${{ secrets.DOCKER_USERNAME }}/${{ env.DOCKER_IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}
    
    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
    
    # Post-deployment verification
    - name: Verify pushed image
      run: |
        # Pull the image we just pushed
        docker pull ${{ secrets.DOCKER_USERNAME }}/${{ env.DOCKER_IMAGE_NAME }}:latest
        
        # Quick test of the pulled image
        docker run -d -p 3001:3000 --name verify-container ${{ secrets.DOCKER_USERNAME }}/${{ env.DOCKER_IMAGE_NAME }}:latest
        sleep 10
        curl -f http://localhost:3001/health || exit 1
        echo "✅ Pushed image verification successful"
        
        # Cleanup
        docker stop verify-container
        docker rm verify-container
EOF
Step 5.2: Create a More Comprehensive Docker Compose Test File
Create a separate Docker Compose file for testing:
cat > docker-compose.test.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=test
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      app:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost/health || exit 1"]
      interval: 10s
      timeout: 5s
      retries: 3

  # Test service to run integration tests
  test-runner:
    image: curlimages/curl:latest
    depends_on:
      app:
        condition: service_healthy
      nginx:
        condition: service_healthy
    command: |
      sh -c "
        echo 'Running integration tests...'
        
        # Test app directly
        curl -f http://app:3000/health || exit 1
        echo '✅ App health check passed'
        
        # Test app functionality
        response=$$(curl -s http://app:3000/)
        echo 'App response:' $$response
        
        # Test nginx proxy
        curl -f http://nginx/health || exit 1
        echo '✅ Nginx proxy test passed'
        
        echo '✅ All integration tests passed!'
      "
EOF
Step 5.3: Add Integration Test Scripts
Create a test script directory:
mkdir -p scripts
Create an integration test script:
cat > scripts/integration-test.sh << 'EOF'
#!/bin/bash

set -e

echo "🚀 Starting integration tests..."

# Function to wait for service to be ready
wait_for_service() {
    local url=$1
    local service_name=$2
    local max_attempts=30
    local attempt=1
    
    echo "⏳ Waiting for $service_name to be ready..."
    
    while [ $attempt -le $max_attempts ]; do
        if curl -f -s "$url" > /dev/null 2>&1; then
            echo "✅ $service_name is ready!"
            return 0
        fi
        
        echo "Attempt $attempt/$max_attempts: $service_name not ready yet..."
        sleep 2
        attempt=$((attempt + 1))
    done
    
    echo "❌ $service_name failed to become ready after $max_attempts attempts"
    return 1
}

# Test 1: App health check
echo "🔍 Test 1: App health check"
wait_for_service "http://localhost:3000/health" "App"

# Test 2: App functionality
echo "🔍 Test 2: App functionality"
response=$(curl -s http://localhost:3000/)
if echo "$response" | grep -q "Hello from Docker CI/CD Demo"; then
    echo "✅ App functionality test passed"
else
    echo "❌ App functionality test failed"
    echo "Response: $response"
    exit 1
fi

# Test 3: Nginx proxy
echo "🔍 Test 3: Nginx proxy"
wait_for_service "http://localhost:8080/health" "Nginx proxy"

# Test 4: Load test (simple)
echo "🔍 Test 4: Simple load test"
for i in {1..10}; do
    curl -f -s http://localhost:3000/ > /dev/null || {
        echo "❌ Load test failed on request $i"
        exit 1
    }
done
echo "✅ Load test passed (10 requests)"

echo "🎉 All integration tests passed!"
EOF
Make the script executable:
chmod +x scripts/integration-test.sh
Step 5.4: Test the Enhanced Pipeline
Commit all the new changes:
git add .
git commit -m "Add comprehensive Docker Compose testing to CI/CD pipeline"
git push origin main
Monitor the GitHub Actions workflow:
Go to your repository's Actions tab
Watch the enhanced workflow run
Verify that all Docker Compose tests pass
Troubleshooting Common Issues
Issue 1: Docker Hub Authentication Fails
Symptoms: Build-and-push job fails with authentication error

Solutions:

Verify your Docker Hub credentials in GitHub Secrets
Use an access token instead of password
Check that secret names match exactly: DOCKER_USERNAME and DOCKER_PASSWORD
Issue 2: Docker Compose Services Don't Start
Symptoms: Services fail health checks or don't respond

Solutions:

Increase wait times in the workflow
Check service dependencies in docker-compose.yml
Verify port mappings don't conflict
Review service logs in the workflow output
Issue 3: Tests Fail Intermittently
Symptoms: Tests pass locally but fail in CI/CD

Solutions:

Add longer sleep/wait times for services to start
Implement proper health checks
Use retry logic for network requests
Check for port conflicts
Issue 4: Image Build Fails
Symptoms: Docker build step fails

Solutions:

Verify Dockerfile syntax
Check that all required files are included
Ensure base image is available
Review build context and .dockerignore
Lab Verification
To verify your lab completion, ensure the following:

Checklist:
 GitHub repository created with all required files
 GitHub Actions workflow file (.github/workflows/docker.yml) configured
 Docker Hub repository created and accessible
 GitHub Secrets configured for Docker Hub authentication
 Workflow successfully builds Docker images on push events
 Images are automatically pushed to Docker Hub
 Docker Compose testing integrated into pipeline
 All tests pass in the CI/CD pipeline
Test Your Setup:
Make a test change:
echo "console.log('Pipeline test');" >> app.js
git add app.js
git commit -m "Test pipeline trigger"
git push origin main
Verify workflow execution:

Check GitHub Actions tab
Confirm all jobs complete successfully
Verify new image appears in Docker Hub
Test locally:

docker-compose up -d
sleep 15
./scripts/integration-test.sh
docker-compose down
Conclusion
Congratulations! You have successfully completed Lab 38: Docker and CI/CD - Automating Docker Builds with GitHub Actions.

What You Accomplished:
Technical Skills Gained: • CI/CD Pipeline Creation: Built a complete automated pipeline using GitHub Actions • Docker Automation: Automated Docker image building, testing, and deployment • Container Registry Integration: Configured automatic pushing to Docker Hub • Multi-Service Testing: Implemented Docker Compose testing in CI/CD • Security Best Practices: Used GitHub Secrets for secure credential management

Real-World Applications: • DevOps Automation: Your pipeline automates the entire software delivery process • Quality Assurance: Automated testing ensures code quality before deployment • Container Distribution: Images are automatically available for deployment • Scalable Architecture: The setup scales to larger, more complex applications

Why This Matters:
Industry Relevance:

95% of organizations use CI/CD practices for faster software delivery
Docker containerization is the standard for modern application deployment
GitHub Actions is one of the most popular CI/CD platforms
Automated testing reduces bugs in production by up to 80%
Career Benefits:

DevOps Skills: Essential for modern software development roles
Automation Expertise: Highly valued in the job market
Container Technology: Critical for cloud-native applications
CI/CD Knowledge: Required for senior development positions
Next Steps:
Immediate Enhancements: • Add more comprehensive test suites (unit, integration, security) • Implement multi-stage deployments (dev, staging, production) • Add monitoring and alerting to your pipeline • Integrate security scanning tools

Advanced Topics to Explore: • Kubernetes deployment automation • Infrastructure as Code (IaC) with Terraform • Advanced Docker security practices • Microservices CI/CD patterns

Certification Path: This lab directly supports your Docker Certified Associate (DCA) certification preparation, specifically covering:

Container orchestration and deployment
Docker security and best practices
CI/CD integration with Docker
Production container management
You now have a solid foundation in modern DevOps practices that will serve you well in your technology career!
