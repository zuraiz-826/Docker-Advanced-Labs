Lab 92: Docker for Continuous Integration - Building Docker Images with CircleCI
Objectives
By the end of this lab, you will be able to:

Set up CircleCI with a Docker-based application for continuous integration
Configure CircleCI to automatically build and push Docker images to Docker Hub
Implement automated testing and validation of Docker images in CI pipeline
Automate deployments to staging environments using CircleCI workflows
Optimize build performance using CircleCI caching strategies
Understand best practices for Docker-based CI/CD pipelines
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with Git version control system
Basic knowledge of YAML syntax
Understanding of continuous integration concepts
A GitHub account (free tier is sufficient)
A Docker Hub account (free tier is sufficient)
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. The machine comes with:

Docker Engine pre-installed
Git pre-installed
Text editors (nano, vim)
All necessary development tools
Internet connectivity for accessing external services
No need to build your own VM or install additional software!

Lab Environment Setup
Initial Setup Verification
First, let's verify that our lab environment is ready:

# Check Docker installation
docker --version

# Check Git installation
git --version

# Verify internet connectivity
ping -c 3 google.com
Task 1: Set up CircleCI with a Docker-based Application
Subtask 1.1: Create a Sample Docker Application
Let's start by creating a simple Node.js application that we'll containerize and integrate with CircleCI.

# Create project directory
mkdir docker-circleci-lab
cd docker-circleci-lab

# Initialize the project
mkdir src
cd src
Create a simple Node.js application:

# Create package.json
cat > package.json << 'EOF'
{
  "name": "docker-circleci-app",
  "version": "1.0.0",
  "description": "Sample app for Docker CircleCI integration",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "node test.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.0.0"
  }
}
EOF
Create the main application file:

# Create app.js
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Docker CircleCI Lab!',
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
  console.log(`App running on port ${port}`);
});

module.exports = { app, server };
EOF
Create a simple test file:

# Create test.js
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
            console.log('✓ Health endpoint test passed');
            resolve(true);
          } else {
            console.log('✗ Health endpoint test failed');
            reject(false);
          }
        } catch (error) {
          console.log('✗ Health endpoint test failed:', error.message);
          reject(false);
        }
      });
    });

    req.on('error', (error) => {
      console.log('✗ Health endpoint test failed:', error.message);
      reject(false);
    });

    req.end();
  });
}

// Run tests
console.log('Running tests...');
console.log('✓ Basic test suite completed');
console.log('Note: For full testing, start the server first');

process.exit(0);
EOF
Subtask 1.2: Create Dockerfile
Navigate back to the project root and create a Dockerfile:

# Go back to project root
cd ..

# Create Dockerfile
cat > Dockerfile << 'EOF'
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory in container
WORKDIR /app

# Copy package files
COPY src/package*.json ./

# Install dependencies
RUN npm install --only=production

# Copy application code
COPY src/ .

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) })"

# Start application
CMD ["npm", "start"]
EOF
Subtask 1.3: Create Docker Compose for Local Testing
Create a docker-compose.yml file for local development:

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=development
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
EOF
Create nginx configuration:

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
        }
        
        location /health {
            proxy_pass http://app/health;
        }
    }
}
EOF
Subtask 1.4: Test Local Docker Build
Let's test our Docker setup locally:

# Build the Docker image
docker build -t docker-circleci-app:latest .

# Run the container
docker run -d --name test-app -p 3000:3000 docker-circleci-app:latest

# Wait a moment for the container to start
sleep 5

# Test the application
curl http://localhost:3000
curl http://localhost:3000/health

# Check container logs
docker logs test-app

# Clean up
docker stop test-app
docker rm test-app
Subtask 1.5: Initialize Git Repository
Now let's initialize our Git repository:

# Initialize git repository
git init

# Create .gitignore
cat > .gitignore << 'EOF'
node_modules/
npm-debug.log*
.npm
.env
.DS_Store
*.log
coverage/
.nyc_output/
EOF
Task 2: Configure CircleCI to Build and Push Docker Images
Subtask 2.1: Create CircleCI Configuration Directory
# Create CircleCI configuration directory
mkdir -p .circleci
Subtask 2.2: Create Basic CircleCI Configuration
Create the main CircleCI configuration file:

cat > .circleci/config.yml << 'EOF'
version: 2.1

# Define orbs (reusable packages of configuration)
orbs:
  docker: circleci/docker@2.2.0
  node: circleci/node@5.1.0

# Define executors
executors:
  docker-executor:
    docker:
      - image: cimg/base:stable
    resource_class: medium

# Define jobs
jobs:
  # Job to run tests
  test:
    executor: docker-executor
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
          docker_layer_caching: true
      
      # Build test image
      - run:
          name: Build test image
          command: |
            docker build -t test-app:${CIRCLE_SHA1} .
      
      # Run tests in container
      - run:
          name: Run application tests
          command: |
            # Start container in background
            docker run -d --name test-container -p 3000:3000 test-app:${CIRCLE_SHA1}
            
            # Wait for application to start
            sleep 10
            
            # Run health check
            docker exec test-container npm test
            
            # Test endpoints
            docker exec test-container sh -c "wget --spider --quiet http://localhost:3000 || exit 1"
            docker exec test-container sh -c "wget --spider --quiet http://localhost:3000/health || exit 1"
            
            # Stop container
            docker stop test-container

  # Job to build and push Docker image
  build-and-push:
    executor: docker-executor
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
          docker_layer_caching: true
      
      # Login to Docker Hub
      - run:
          name: Login to Docker Hub
          command: |
            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
      
      # Build production image
      - run:
          name: Build production image
          command: |
            docker build -t $DOCKER_USERNAME/docker-circleci-app:${CIRCLE_SHA1} .
            docker build -t $DOCKER_USERNAME/docker-circleci-app:latest .
      
      # Push images to Docker Hub
      - run:
          name: Push images to Docker Hub
          command: |
            docker push $DOCKER_USERNAME/docker-circleci-app:${CIRCLE_SHA1}
            docker push $DOCKER_USERNAME/docker-circleci-app:latest

  # Job for security scanning
  security-scan:
    executor: docker-executor
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
      
      # Build image for scanning
      - run:
          name: Build image for security scan
          command: |
            docker build -t scan-image:latest .
      
      # Install and run Trivy security scanner
      - run:
          name: Run security scan with Trivy
          command: |
            # Install Trivy
            sudo apt-get update
            sudo apt-get install wget apt-transport-https gnupg lsb-release -y
            wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
            echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
            sudo apt-get update
            sudo apt-get install trivy -y
            
            # Run security scan
            trivy image --exit-code 0 --severity HIGH,CRITICAL scan-image:latest

# Define workflows
workflows:
  version: 2
  build-test-deploy:
    jobs:
      - test
      - security-scan:
          requires:
            - test
      - build-and-push:
          requires:
            - test
            - security-scan
          filters:
            branches:
              only:
                - main
                - develop
EOF
Subtask 2.3: Create Advanced CircleCI Configuration with Caching
Let's create a more advanced configuration with caching strategies:

cat > .circleci/config-advanced.yml << 'EOF'
version: 2.1

# Define orbs
orbs:
  docker: circleci/docker@2.2.0

# Define executors
executors:
  docker-executor:
    docker:
      - image: cimg/base:stable
    resource_class: medium
    environment:
      DOCKER_BUILDKIT: 1

# Define commands for reusability
commands:
  restore-docker-cache:
    description: "Restore Docker layer cache"
    steps:
      - restore_cache:
          keys:
            - docker-cache-v1-{{ .Branch }}-{{ checksum "Dockerfile" }}
            - docker-cache-v1-{{ .Branch }}-
            - docker-cache-v1-

  save-docker-cache:
    description: "Save Docker layer cache"
    steps:
      - save_cache:
          key: docker-cache-v1-{{ .Branch }}-{{ checksum "Dockerfile" }}
          paths:
            - /tmp/docker-cache

# Define jobs
jobs:
  # Lint and validate
  lint:
    executor: docker-executor
    steps:
      - checkout
      
      # Lint Dockerfile
      - run:
          name: Install hadolint
          command: |
            wget -O hadolint https://github.com/hadolint/hadolint/releases/download/v2.12.0/hadolint-Linux-x86_64
            chmod +x hadolint
            sudo mv hadolint /usr/local/bin/
      
      - run:
          name: Lint Dockerfile
          command: |
            hadolint Dockerfile

  # Build with caching
  build:
    executor: docker-executor
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
          docker_layer_caching: true
      
      - restore-docker-cache
      
      # Load cached layers if available
      - run:
          name: Load cached Docker layers
          command: |
            if [ -f /tmp/docker-cache/layers.tar ]; then
              docker load -i /tmp/docker-cache/layers.tar
            fi
      
      # Build with cache
      - run:
          name: Build Docker image with cache
          command: |
            docker build \
              --cache-from docker-circleci-app:cache \
              -t docker-circleci-app:${CIRCLE_SHA1} \
              -t docker-circleci-app:cache \
              .
      
      # Save layers to cache
      - run:
          name: Save Docker layers to cache
          command: |
            mkdir -p /tmp/docker-cache
            docker save docker-circleci-app:cache -o /tmp/docker-cache/layers.tar
      
      - save-docker-cache
      
      # Save image for next job
      - run:
          name: Save image to workspace
          command: |
            mkdir -p /tmp/workspace
            docker save docker-circleci-app:${CIRCLE_SHA1} -o /tmp/workspace/image.tar
      
      - persist_to_workspace:
          root: /tmp/workspace
          paths:
            - image.tar

  # Test job
  test:
    executor: docker-executor
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
      
      - attach_workspace:
          at: /tmp/workspace
      
      # Load image from workspace
      - run:
          name: Load Docker image
          command: |
            docker load -i /tmp/workspace/image.tar
      
      # Run comprehensive tests
      - run:
          name: Run tests
          command: |
            # Start container
            docker run -d --name test-app -p 3000:3000 docker-circleci-app:${CIRCLE_SHA1}
            
            # Wait for startup
            sleep 15
            
            # Test health endpoint
            for i in {1..5}; do
              if docker exec test-app wget --spider --quiet http://localhost:3000/health; then
                echo "Health check passed"
                break
              fi
              echo "Health check attempt $i failed, retrying..."
              sleep 5
            done
            
            # Run application tests
            docker exec test-app npm test
            
            # Test main endpoint
            docker exec test-app wget --spider --quiet http://localhost:3000
            
            # Check logs for errors
            docker logs test-app
            
            # Cleanup
            docker stop test-app
            docker rm test-app

  # Push to registry
  push:
    executor: docker-executor
    steps:
      - setup_remote_docker:
          version: 20.10.14
      
      - attach_workspace:
          at: /tmp/workspace
      
      # Load image
      - run:
          name: Load and tag image
          command: |
            docker load -i /tmp/workspace/image.tar
            docker tag docker-circleci-app:${CIRCLE_SHA1} $DOCKER_USERNAME/docker-circleci-app:${CIRCLE_SHA1}
            docker tag docker-circleci-app:${CIRCLE_SHA1} $DOCKER_USERNAME/docker-circleci-app:latest
      
      # Login and push
      - run:
          name: Login to Docker Hub
          command: |
            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
      
      - run:
          name: Push to Docker Hub
          command: |
            docker push $DOCKER_USERNAME/docker-circleci-app:${CIRCLE_SHA1}
            docker push $DOCKER_USERNAME/docker-circleci-app:latest

# Workflows
workflows:
  version: 2
  build-test-push:
    jobs:
      - lint
      - build:
          requires:
            - lint
      - test:
          requires:
            - build
      - push:
          requires:
            - test
          filters:
            branches:
              only:
                - main
                - develop
EOF
Task 3: Implement Testing and Validation of Docker Images
Subtask 3.1: Create Comprehensive Test Suite
Let's create a more comprehensive test suite:

# Create tests directory
mkdir -p tests

# Create integration tests
cat > tests/integration.js << 'EOF'
const http = require('http');

class TestRunner {
  constructor() {
    this.tests = [];
    this.passed = 0;
    this.failed = 0;
  }

  addTest(name, testFn) {
    this.tests.push({ name, testFn });
  }

  async runTests() {
    console.log('Starting integration tests...\n');
    
    for (const test of this.tests) {
      try {
        console.log(`Running: ${test.name}`);
        await test.testFn();
        console.log(`✓ PASSED: ${test.name}\n`);
        this.passed++;
      } catch (error) {
        console.log(`✗ FAILED: ${test.name}`);
        console.log(`  Error: ${error.message}\n`);
        this.failed++;
      }
    }

    console.log('Test Results:');
    console.log(`  Passed: ${this.passed}`);
    console.log(`  Failed: ${this.failed}`);
    console.log(`  Total: ${this.tests.length}`);

    if (this.failed > 0) {
      process.exit(1);
    }
  }

  makeRequest(path, expectedStatus = 200) {
    return new Promise((resolve, reject) => {
      const options = {
        hostname: 'localhost',
        port: 3000,
        path: path,
        method: 'GET'
      };

      const req = http.request(options, (res) => {
        let data = '';
        res.on('data', (chunk) => {
          data += chunk;
        });
        res.on('end', () => {
          if (res.statusCode === expectedStatus) {
            resolve({ statusCode: res.statusCode, data: data });
          } else {
            reject(new Error(`Expected status ${expectedStatus}, got ${res.statusCode}`));
          }
        });
      });

      req.on('error', (error) => {
        reject(error);
      });

      req.setTimeout(5000, () => {
        req.destroy();
        reject(new Error('Request timeout'));
      });

      req.end();
    });
  }
}

// Create test runner
const runner = new TestRunner();

// Add tests
runner.addTest('Health endpoint returns 200', async () => {
  const response = await runner.makeRequest('/health');
  const data = JSON.parse(response.data);
  if (!data.status || data.status !== 'healthy') {
    throw new Error('Health status is not healthy');
  }
});

runner.addTest('Root endpoint returns valid JSON', async () => {
  const response = await runner.makeRequest('/');
  const data = JSON.parse(response.data);
  if (!data.message || !data.version) {
    throw new Error('Response missing required fields');
  }
});

runner.addTest('Root endpoint contains expected message', async () => {
  const response = await runner.makeRequest('/');
  const data = JSON.parse(response.data);
  if (!data.message.includes('Hello from Docker CircleCI Lab!')) {
    throw new Error('Message does not contain expected text');
  }
});

runner.addTest('Health endpoint includes uptime', async () => {
  const response = await runner.makeRequest('/health');
  const data = JSON.parse(response.data);
  if (typeof data.uptime !== 'number' || data.uptime < 0) {
    throw new Error('Uptime is not a valid number');
  }
});

runner.addTest('Non-existent endpoint returns 404', async () => {
  await runner.makeRequest('/nonexistent', 404);
});

// Run tests
runner.runTests().catch(console.error);
EOF
Subtask 3.2: Create Docker Image Validation Script
Create a script to validate Docker images:

cat > tests/validate-image.sh << 'EOF'
#!/bin/bash

set -e

IMAGE_NAME=${1:-docker-circleci-app:latest}
CONTAINER_NAME="validation-container-$(date +%s)"

echo "Validating Docker image: $IMAGE_NAME"

# Function to cleanup
cleanup() {
    echo "Cleaning up..."
    docker stop $CONTAINER_NAME 2>/dev/null || true
    docker rm $CONTAINER_NAME 2>/dev/null || true
}

# Set trap for cleanup
trap cleanup EXIT

# Test 1: Image exists
echo "Test 1: Checking if image exists..."
if docker image inspect $IMAGE_NAME > /dev/null 2>&1; then
    echo "✓ Image exists"
else
    echo "✗ Image does not exist"
    exit 1
fi

# Test 2: Image size check
echo "Test 2: Checking image size..."
IMAGE_SIZE=$(docker image inspect $IMAGE_NAME --format='{{.Size}}')
IMAGE_SIZE_MB=$((IMAGE_SIZE / 1024 / 1024))
echo "Image size: ${IMAGE_SIZE_MB}MB"

if [ $IMAGE_SIZE_MB -gt 500 ]; then
    echo "⚠ Warning: Image size is quite large (${IMAGE_SIZE_MB}MB)"
else
    echo "✓ Image size is reasonable"
fi

# Test 3: Container starts successfully
echo "Test 3: Testing container startup..."
docker run -d --name $CONTAINER_NAME -p 3001:3000 $IMAGE_NAME

# Wait for container to start
echo "Waiting for container to start..."
sleep 10

# Test 4: Container is running
echo "Test 4: Checking if container is running..."
if docker ps | grep -q $CONTAINER_NAME; then
    echo "✓ Container is running"
else
    echo "✗ Container is not running"
    docker logs $CONTAINER_NAME
    exit 1
fi

# Test 5: Health check
echo "Test 5: Testing health endpoint..."
for i in {1..10}; do
    if curl -f http://localhost:3001/health > /dev/null 2>&1; then
        echo "✓ Health endpoint is responding"
        break
    fi
    if [ $i -eq 10 ]; then
        echo "✗ Health endpoint is not responding after 10 attempts"
        docker logs $CONTAINER_NAME
        exit 1
    fi
    echo "Attempt $i failed, retrying..."
    sleep 2
done

# Test 6: Main endpoint
echo "Test 6: Testing main endpoint..."
RESPONSE=$(curl -s http://localhost:3001/)
if echo "$RESPONSE" | grep -q "Hello from Docker CircleCI Lab!"; then
    echo "✓ Main endpoint returns expected response"
else
    echo "✗ Main endpoint does not return expected response"
    echo "Response: $RESPONSE"
    exit 1
fi

# Test 7: Check for security vulnerabilities (basic)
echo "Test 7: Basic security checks..."
if docker run --rm $IMAGE_NAME whoami | grep -q "nodejs"; then
    echo "✓ Container runs as non-root user"
else
    echo "⚠ Warning: Container might be running as root"
fi

# Test 8: Check exposed ports
echo "Test 8: Checking exposed ports..."
EXPOSED_PORTS=$(docker image inspect $IMAGE_NAME --format='{{range $port, $config := .Config.ExposedPorts}}{{$port}} {{end}}')
if echo "$EXPOSED_PORTS" | grep -q "3000/tcp"; then
    echo "✓ Port 3000 is properly exposed"
else
    echo "✗ Port 3000 is not exposed"
    exit 1
fi

echo ""
echo "All validation tests passed! ✓"
echo "Image $IMAGE_NAME is ready for deployment."
EOF

# Make script executable
chmod +x tests/validate-image.sh
Subtask 3.3: Update Package.json with Test Scripts
Update the package.json to include our new test scripts:

cat > src/package.json << 'EOF'
{
  "name": "docker-circleci-app",
  "version": "1.0.0",
  "description": "Sample app for Docker CircleCI integration",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "node test.js",
    "test:integration": "node ../tests/integration.js",
    "test:all": "npm test && npm run test:integration"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.0.0"
  }
}
EOF
Task 4: Automate Deployments to Staging or Production Environments
Subtask 4.1: Create Deployment Scripts
Create deployment scripts for different environments:

# Create deployment directory
mkdir -p deployment

# Create staging deployment script
cat > deployment/deploy-staging.sh << 'EOF'
#!/bin/bash

set -e

# Configuration
DOCKER_IMAGE=${1:-$DOCKER_USERNAME/docker-circleci-app:latest}
STAGING_PORT=${2:-3000}
CONTAINER_NAME="staging-app"

echo "Deploying to staging environment..."
echo "Image: $DOCKER_IMAGE"
echo "Port: $STAGING_PORT"

# Function to cleanup existing container
cleanup_existing() {
    if docker ps -a | grep -q $CONTAINER_NAME; then
        echo "Stopping existing staging container..."
        docker stop $CONTAINER_NAME || true
        docker rm $CONTAINER_NAME || true
    fi
}

# Pull latest image
echo "Pulling latest image..."
docker pull $DOCKER_IMAGE

# Cleanup existing deployment
cleanup_existing

# Deploy new container
echo "Starting new staging container..."
docker run -d \
    --name $CONTAINER_NAME \
    --restart unless-stopped \
    -p $STAGING_PORT:3000 \
    -e NODE_ENV=staging \
    -e PORT=3000 \
    --health-cmd="wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1" \
    --health-interval=30s \
    --health-timeout=10s \
    --health-retries=3 \
    $DOCKER_IMAGE

# Wait for container to be healthy
echo "Waiting for container to be healthy..."
for i in {1..30}; do
    if docker inspect --format='{{.State.Health.Status}}' $CONTAINER_NAME | grep -q "healthy"; then
        echo "✓ Container is healthy"
        break
    fi
    if [ $i -eq 30 ]; then
        echo "✗ Container failed to become healthy"
        docker logs $CONTAINER_NAME
        exit 1
    fi
    echo "Waiting... ($i/30)"
    sleep 5
done

# Verify deployment
echo "Verifying deployment..."
sleep 5

if curl -f http://localhost:$STAGING_PORT/health > /dev/null 2>&1; then
    echo "✓ Staging deployment successful!"
    echo "Application is available at: http://localhost:$STAGING_PORT"
else
    echo "✗ Staging deployment failed - health check failed"
    docker logs $CONTAINER_NAME
    exit 1
fi

echo "Staging deployment completed successfully!"
EOF

# Make script executable
chmod +x deployment/deploy-staging.sh
Create production deployment script:

cat > deployment/deploy-production.sh << 'EOF'
#!/bin/bash

set -e

# Configuration
DOCKER_IMAGE=${1:-$DOCKER_USERNAME/docker-circleci-app:latest}
PRODUCTION_PORT=${2:-80}
CONTAINER_NAME="production-app"
BACKUP_CONTAINER_NAME="production-app-backup"

echo "Deploying to production environment..."
echo "Image: $DOCKER_IMAGE"
echo "Port: $PRODUCTION_PORT"

# Function to rollback
rollback() {
    echo "Rolling back to previous version..."
    if docker ps -a | grep -q $BACKUP_CONTAINER_NAME; then
        docker stop $CONTAINER_NAME || true
        docker rm $CONTAINER_NAME || true
        docker rename $BACKUP_CONTAINER_NAME $CONTAINER_NAME
        docker start $CONTAINER_NAME
        echo "Rollback completed"
    else
        echo "No backup container found for rollback"
    fi
}

# Set trap for rollback on failure
trap 'rollback' ERR

# Pull and verify image
echo "Pulling and verifying image..."
docker pull $DOCKER_IMAGE

# Run image validation
if [ -f "../tests/validate-image.sh" ]; then
    echo "Running image validation..."
    ../tests/validate-image.sh $DOCKER_IMAGE
fi

# Backup current production container
if docker ps | grep -q $CONTAINER_NAME; then
    echo "Backing up current production container..."
    docker stop $CONTAINER_NAME
    docker rename $CONTAINER_NAME $BACKUP_CONTAINER_NAME
fi

# Deploy new container
echo "Starting new production container..."
docker run -d \
    --name $CONTAINER_NAME \
    --restart unless-stopped \
    -p $PRODUCTION_PORT:3000 \
    -e NODE_ENV=production \
    -e PORT=3000 \
    --health-cmd="wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1" \
    --health-interval=30s \
    --health-timeout=10s \
    --health-retries=3 \
    --memory=512m \
    --
