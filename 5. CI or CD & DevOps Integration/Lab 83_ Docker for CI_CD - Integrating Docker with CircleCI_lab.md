Lab 83: Docker for CI/CD - Integrating Docker with CircleCI
Objectives
By the end of this lab, students will be able to:

Set up a CircleCI account and configure the .circleci/config.yml file for Docker-based CI/CD pipelines
Build and test Docker images automatically within CircleCI workflows
Push Docker images to Docker Hub after successful test execution
Deploy Docker containers to cloud services using CircleCI automation
Implement automated container testing and deployment strategies
Understand the integration between Docker and CircleCI for continuous integration and deployment
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with Git version control system
Basic knowledge of YAML file structure
Understanding of CI/CD concepts
A GitHub account for code repository management
Basic command-line interface experience
Lab Environment Setup
Al Nafi Cloud Machines: This lab uses Al Nafi's pre-configured Linux-based cloud machines. Simply click Start Lab to access your ready-to-use environment with all necessary tools pre-installed including Docker, Git, and text editors. No need to build your own VM or install additional software.

Task 1: Set up CircleCI Account and Configure .circleci/config.yml
Subtask 1.1: Create CircleCI Account and Connect Repository
Step 1: Create a new GitHub repository for this lab

Go to GitHub and create a new repository named docker-circleci-lab
Initialize it with a README file
Clone the repository to your Al Nafi cloud machine:
git clone https://github.com/YOUR_USERNAME/docker-circleci-lab.git
cd docker-circleci-lab
Step 2: Sign up for CircleCI

Visit https://circleci.com/signup/
Click Sign Up with GitHub
Authorize CircleCI to access your GitHub account
Select your docker-circleci-lab repository to follow
Subtask 1.2: Create Sample Application
Step 1: Create a simple Node.js application

Create the main application file:

nano app.js
Add the following content:

const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Docker CI/CD Lab!',
    timestamp: new Date().toISOString(),
    version: '1.0.0'
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});

module.exports = app;
Step 2: Create package.json file

nano package.json
Add the following content:

{
  "name": "docker-circleci-lab",
  "version": "1.0.0",
  "description": "Sample app for Docker CircleCI integration",
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
    "jest": "^29.5.0",
    "supertest": "^6.3.3"
  }
}
Step 3: Create test file

mkdir tests
nano tests/app.test.js
Add the following test content:

const request = require('supertest');
const app = require('../app');

describe('App Tests', () => {
  test('GET / should return welcome message', async () => {
    const response = await request(app)
      .get('/')
      .expect(200);
    
    expect(response.body.message).toBe('Hello from Docker CI/CD Lab!');
    expect(response.body.version).toBe('1.0.0');
  });

  test('GET /health should return healthy status', async () => {
    const response = await request(app)
      .get('/health')
      .expect(200);
    
    expect(response.body.status).toBe('healthy');
  });
});
Subtask 1.3: Create Dockerfile
Step 1: Create Dockerfile for the application

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
Subtask 1.4: Configure CircleCI Pipeline
Step 1: Create CircleCI configuration directory and file

mkdir -p .circleci
nano .circleci/config.yml
Step 2: Add basic CircleCI configuration

version: 2.1

# Define reusable executors
executors:
  docker-executor:
    docker:
      - image: cimg/node:18.16
    working_directory: ~/project

# Define reusable commands
commands:
  install-docker:
    description: "Install Docker CLI"
    steps:
      - setup_remote_docker:
          version: 20.10.14
          docker_layer_caching: true

# Define jobs
jobs:
  test:
    executor: docker-executor
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "package-lock.json" }}
            - v1-dependencies-
      - run:
          name: Install dependencies
          command: npm ci
      - save_cache:
          paths:
            - node_modules
          key: v1-dependencies-{{ checksum "package-lock.json" }}
      - run:
          name: Run tests
          command: npm test
      - store_test_results:
          path: test-results

  build-and-push:
    executor: docker-executor
    steps:
      - checkout
      - install-docker
      - run:
          name: Build Docker image
          command: |
            docker build -t $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1 .
            docker build -t $DOCKER_USERNAME/docker-circleci-lab:latest .
      - run:
          name: Run container tests
          command: |
            # Start container for testing
            docker run -d --name test-container -p 3000:3000 $DOCKER_USERNAME/docker-circleci-lab:latest
            sleep 10
            # Test container health
            docker exec test-container curl -f http://localhost:3000/health || exit 1
            # Stop and remove test container
            docker stop test-container
            docker rm test-container
      - run:
          name: Login to Docker Hub
          command: echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
      - run:
          name: Push to Docker Hub
          command: |
            docker push $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1
            docker push $DOCKER_USERNAME/docker-circleci-lab:latest

# Define workflows
workflows:
  version: 2
  test-build-deploy:
    jobs:
      - test
      - build-and-push:
          requires:
            - test
          filters:
            branches:
              only: main
Task 2: Build and Test Docker Images in CircleCI Pipeline
Subtask 2.1: Configure Docker Hub Credentials
Step 1: Set up Docker Hub account and access token

Create a Docker Hub account at https://hub.docker.com if you don't have one
Go to Account Settings > Security > New Access Token
Create a new access token with read/write permissions
Step 2: Add environment variables to CircleCI

Go to your CircleCI project settings
Navigate to Environment Variables
Add the following variables:
DOCKER_USERNAME: Your Docker Hub username
DOCKER_PASSWORD: Your Docker Hub access token
Subtask 2.2: Enhanced Testing Configuration
Step 1: Create Jest configuration file

nano jest.config.js
Add the following content:

module.exports = {
  testEnvironment: 'node',
  collectCoverage: true,
  coverageDirectory: 'coverage',
  coverageReporters: ['text', 'lcov', 'html'],
  testMatch: ['**/tests/**/*.test.js'],
  collectCoverageFrom: [
    '*.js',
    '!jest.config.js',
    '!coverage/**'
  ]
};
Step 2: Update the CircleCI config for enhanced testing

nano .circleci/config.yml
Update the test job section:

  test:
    executor: docker-executor
    steps:
      - checkout
      - restore_cache:
          keys:
            - v1-dependencies-{{ checksum "package-lock.json" }}
            - v1-dependencies-
      - run:
          name: Install dependencies
          command: npm ci
      - save_cache:
          paths:
            - node_modules
          key: v1-dependencies-{{ checksum "package-lock.json" }}
      - run:
          name: Run linting
          command: |
            npx eslint . --ext .js --format junit --output-file test-results/eslint.xml || true
      - run:
          name: Run unit tests
          command: |
            mkdir -p test-results
            npm test -- --ci --testResultsProcessor=jest-junit
          environment:
            JEST_JUNIT_OUTPUT_DIR: test-results
            JEST_JUNIT_OUTPUT_NAME: jest.xml
      - store_test_results:
          path: test-results
      - store_artifacts:
          path: coverage
          destination: coverage-reports
Subtask 2.3: Add Multi-stage Docker Build
Step 1: Update Dockerfile for multi-stage build

nano Dockerfile
Replace with the following multi-stage Dockerfile:

# Build stage
FROM node:18-alpine AS builder

WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev dependencies)
RUN npm ci

# Copy source code
COPY . .

# Run tests in build stage
RUN npm test

# Production stage
FROM node:18-alpine AS production

WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install only production dependencies
RUN npm ci --only=production && npm cache clean --force

# Copy application code
COPY --from=builder /usr/src/app/app.js ./
COPY --from=builder /usr/src/app/package.json ./

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
Task 3: Push Docker Image to Docker Hub After Successful Tests
Subtask 3.1: Implement Conditional Push Strategy
Step 1: Update CircleCI configuration for conditional pushing

nano .circleci/config.yml
Update the build-and-push job:

  build-and-push:
    executor: docker-executor
    steps:
      - checkout
      - install-docker
      - run:
          name: Build Docker image
          command: |
            # Build with commit SHA tag
            docker build -t $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1 .
            # Build with branch name tag
            docker build -t $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_BRANCH .
            # Build latest tag only for main branch
            if [ "$CIRCLE_BRANCH" = "main" ]; then
              docker build -t $DOCKER_USERNAME/docker-circleci-lab:latest .
            fi
      - run:
          name: Test Docker image
          command: |
            # Test image can start successfully
            docker run -d --name test-app -p 3000:3000 $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1
            
            # Wait for application to start
            sleep 15
            
            # Test health endpoint
            docker exec test-app wget --spider -q http://localhost:3000/health
            
            # Test main endpoint
            docker exec test-app wget -q -O - http://localhost:3000 | grep -q "Hello from Docker CI/CD Lab"
            
            # Check container logs
            docker logs test-app
            
            # Cleanup
            docker stop test-app
            docker rm test-app
      - run:
          name: Security scan with Trivy
          command: |
            # Install Trivy
            sudo apt-get update
            sudo apt-get install wget apt-transport-https gnupg lsb-release
            wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
            echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
            sudo apt-get update
            sudo apt-get install trivy
            
            # Scan image for vulnerabilities
            trivy image --exit-code 0 --severity HIGH,CRITICAL $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1
      - run:
          name: Login to Docker Hub
          command: echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
      - run:
          name: Push images to Docker Hub
          command: |
            # Always push commit SHA tag
            docker push $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1
            
            # Push branch tag
            docker push $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_BRANCH
            
            # Push latest tag only for main branch
            if [ "$CIRCLE_BRANCH" = "main" ]; then
              docker push $DOCKER_USERNAME/docker-circleci-lab:latest
            fi
      - run:
          name: Update Docker Hub description
          command: |
            # Create description update script
            cat > update_description.sh << 'EOF'
            #!/bin/bash
            DOCKER_REPO="$DOCKER_USERNAME/docker-circleci-lab"
            
            # Get Docker Hub token
            TOKEN=$(curl -s -H "Content-Type: application/json" -X POST \
              -d '{"username": "'$DOCKER_USERNAME'", "password": "'$DOCKER_PASSWORD'"}' \
              https://hub.docker.com/v2/users/login/ | jq -r .token)
            
            # Update repository description
            curl -s -H "Authorization: JWT $TOKEN" -H "Content-Type: application/json" \
              -X PATCH -d '{"description": "Docker CI/CD Lab - Automated build from CircleCI", "full_description": "This image is automatically built and tested using CircleCI. Built from commit: '$CIRCLE_SHA1'"}' \
              https://hub.docker.com/v2/repositories/$DOCKER_REPO/
            EOF
            
            chmod +x update_description.sh
            ./update_description.sh
Subtask 3.2: Add Image Tagging Strategy
Step 1: Create a comprehensive tagging strategy

Add this to your CircleCI config under the build-and-push job:

      - run:
          name: Generate image tags
          command: |
            # Create tags file
            echo "Tags to be created:" > tags.txt
            echo "- $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1" >> tags.txt
            echo "- $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_BRANCH" >> tags.txt
            
            # Add version tag if this is a tagged release
            if [ -n "$CIRCLE_TAG" ]; then
              echo "- $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_TAG" >> tags.txt
            fi
            
            # Add latest tag for main branch
            if [ "$CIRCLE_BRANCH" = "main" ]; then
              echo "- $DOCKER_USERNAME/docker-circleci-lab:latest" >> tags.txt
            fi
            
            cat tags.txt
      - store_artifacts:
          path: tags.txt
          destination: build-info/tags.txt
Task 4: Deploy Docker Containers to Cloud Services
Subtask 4.1: Deploy to AWS ECS (Elastic Container Service)
Step 1: Create AWS deployment configuration

mkdir -p deployment/aws
nano deployment/aws/task-definition.json
Add the following ECS task definition:

{
  "family": "docker-circleci-lab",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::YOUR_ACCOUNT_ID:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "docker-circleci-lab",
      "image": "YOUR_DOCKER_USERNAME/docker-circleci-lab:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/docker-circleci-lab",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:3000/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      }
    }
  ]
}
Step 2: Create AWS deployment script

nano deployment/aws/deploy.sh
Add the following deployment script:

#!/bin/bash

set -e

# Variables
CLUSTER_NAME="docker-circleci-cluster"
SERVICE_NAME="docker-circleci-service"
TASK_DEFINITION_FILE="deployment/aws/task-definition.json"

echo "Starting AWS ECS deployment..."

# Update task definition with new image
sed -i "s|YOUR_DOCKER_USERNAME|$DOCKER_USERNAME|g" $TASK_DEFINITION_FILE
sed -i "s|YOUR_ACCOUNT_ID|$AWS_ACCOUNT_ID|g" $TASK_DEFINITION_FILE

# Register new task definition
echo "Registering new task definition..."
aws ecs register-task-definition \
  --cli-input-json file://$TASK_DEFINITION_FILE \
  --region $AWS_DEFAULT_REGION

# Get the latest task definition ARN
TASK_DEFINITION_ARN=$(aws ecs describe-task-definition \
  --task-definition docker-circleci-lab \
  --region $AWS_DEFAULT_REGION \
  --query 'taskDefinition.taskDefinitionArn' \
  --output text)

echo "New task definition ARN: $TASK_DEFINITION_ARN"

# Update service with new task definition
echo "Updating ECS service..."
aws ecs update-service \
  --cluster $CLUSTER_NAME \
  --service $SERVICE_NAME \
  --task-definition $TASK_DEFINITION_ARN \
  --region $AWS_DEFAULT_REGION

# Wait for deployment to complete
echo "Waiting for deployment to complete..."
aws ecs wait services-stable \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME \
  --region $AWS_DEFAULT_REGION

echo "Deployment completed successfully!"

# Get service status
aws ecs describe-services \
  --cluster $CLUSTER_NAME \
  --services $SERVICE_NAME \
  --region $AWS_DEFAULT_REGION \
  --query 'services[0].{Status:status,RunningCount:runningCount,DesiredCount:desiredCount}'
Step 3: Add AWS deployment job to CircleCI

Add this job to your .circleci/config.yml:

  deploy-aws:
    docker:
      - image: cimg/aws:2023.03
    steps:
      - checkout
      - run:
          name: Install dependencies
          command: |
            sudo apt-get update
            sudo apt-get install -y jq
      - run:
          name: Deploy to AWS ECS
          command: |
            chmod +x deployment/aws/deploy.sh
            ./deployment/aws/deploy.sh
      - run:
          name: Verify deployment
          command: |
            # Get service endpoint (assuming ALB is configured)
            echo "Deployment verification would go here"
            echo "Service deployed with image: $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1"
Subtask 4.2: Deploy to Google Cloud Run
Step 1: Create Google Cloud deployment configuration

mkdir -p deployment/gcp
nano deployment/gcp/service.yaml
Add the following Cloud Run service configuration:

apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: docker-circleci-lab
  annotations:
    run.googleapis.com/ingress: all
spec:
  template:
    metadata:
      annotations:
        autoscaling.knative.dev/maxScale: "10"
        run.googleapis.com/cpu-throttling: "false"
    spec:
      containerConcurrency: 100
      containers:
      - image: gcr.io/PROJECT_ID/docker-circleci-lab:latest
        ports:
        - containerPort: 3000
        env:
        - name: PORT
          value: "3000"
        resources:
          limits:
            cpu: "1"
            memory: "512Mi"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
Step 2: Create Google Cloud deployment script

nano deployment/gcp/deploy.sh
Add the following script:

#!/bin/bash

set -e

# Variables
PROJECT_ID=$GOOGLE_PROJECT_ID
SERVICE_NAME="docker-circleci-lab"
REGION="us-central1"
IMAGE_NAME="gcr.io/$PROJECT_ID/$SERVICE_NAME:$CIRCLE_SHA1"

echo "Starting Google Cloud Run deployment..."

# Authenticate with Google Cloud
echo $GOOGLE_SERVICE_KEY | base64 --decode > gcloud-service-key.json
gcloud auth activate-service-account --key-file gcloud-service-key.json
gcloud config set project $PROJECT_ID

# Tag and push image to Google Container Registry
echo "Tagging and pushing image to GCR..."
docker tag $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1 $IMAGE_NAME
docker push $IMAGE_NAME

# Update service configuration with new image
sed -i "s|PROJECT_ID|$PROJECT_ID|g" deployment/gcp/service.yaml
sed -i "s|docker-circleci-lab:latest|docker-circleci-lab:$CIRCLE_SHA1|g" deployment/gcp/service.yaml

# Deploy to Cloud Run
echo "Deploying to Cloud Run..."
gcloud run services replace deployment/gcp/service.yaml \
  --region=$REGION \
  --platform=managed

# Make service publicly accessible
gcloud run services add-iam-policy-binding $SERVICE_NAME \
  --region=$REGION \
  --member="allUsers" \
  --role="roles/run.invoker"

# Get service URL
SERVICE_URL=$(gcloud run services describe $SERVICE_NAME \
  --region=$REGION \
  --format="value(status.url)")

echo "Service deployed successfully!"
echo "Service URL: $SERVICE_URL"

# Test the deployed service
echo "Testing deployed service..."
curl -f "$SERVICE_URL/health" || exit 1
echo "Health check passed!"
Step 3: Add Google Cloud deployment job to CircleCI

Add this job to your .circleci/config.yml:

  deploy-gcp:
    docker:
      - image: google/cloud-sdk:alpine
    steps:
      - checkout
      - setup_remote_docker:
          version: 20.10.14
      - run:
          name: Install Docker
          command: |
            apk add --no-cache docker-cli
      - run:
          name: Login to Docker Hub
          command: echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
      - run:
          name: Pull image from Docker Hub
          command: docker pull $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1
      - run:
          name: Deploy to Google Cloud Run
          command: |
            chmod +x deployment/gcp/deploy.sh
            ./deployment/gcp/deploy.sh
Task 5: Automate Container Testing and Deployment
Subtask 5.1: Implement Comprehensive Testing Pipeline
Step 1: Create integration tests

mkdir -p tests/integration
nano tests/integration/container.test.js
Add comprehensive container tests:

const { exec } = require('child_process');
const util = require('util');

const execAsync = util.promisify(exec);

describe('Container Integration Tests', () => {
  const containerName = 'test-integration-container';
  const imageName = process.env.DOCKER_USERNAME + '/docker-circleci-lab:latest';

  beforeAll(async () => {
    // Pull the latest image
    await execAsync(`docker pull ${imageName}`);
    
    // Start container
    await execAsync(`docker run -d --name ${containerName} -p 3001:3000 ${imageName}`);
    
    // Wait for container to be ready
    await new Promise(resolve => setTimeout(resolve, 10000));
  });

  afterAll(async () => {
    // Clean up container
    try {
      await execAsync(`docker stop ${containerName}`);
      await execAsync(`docker rm ${containerName}`);
    } catch (error) {
      console.log('Cleanup error:', error.message);
    }
  });

  test('Container should be running', async () => {
    const { stdout } = await execAsync(`docker ps --filter name=${containerName} --format "{{.Status}}"`);
    expect(stdout).toContain('Up');
  });

  test('Application should respond to health check', async () => {
    const { stdout } = await execAsync(`curl -s http://localhost:3001/health`);
    const response = JSON.parse(stdout);
    expect(response.status).toBe('healthy');
  });

  test('Application should return correct response', async () => {
    const { stdout } = await execAsync(`curl -s http://localhost:3001/`);
    const response = JSON.parse(stdout);
    expect(response.message).toBe('Hello from Docker CI/CD Lab!');
    expect(response.version).toBe('1.0.0');
  });

  test('Container should have correct resource limits', async () => {
    const { stdout } = await execAsync(`docker stats ${containerName} --no-stream --format "{{.CPUPerc}} {{.MemUsage}}"`);
    expect(stdout).toBeDefined();
    // Basic check that stats are returned
    expect(stdout.length).toBeGreaterThan(0);
  });

  test('Container logs should not contain errors', async () => {
    const { stdout } = await execAsync(`docker logs ${containerName}`);
    expect(stdout).toContain('Server running on port 3000');
    expect(stdout).not.toContain('ERROR');
    expect(stdout).not.toContain('FATAL');
  });
});
Subtask 5.2: Create Performance and Load Testing
Step 1: Add performance testing dependencies

Update your package.json to include performance testing tools:

nano package.json
Add to devDependencies:

{
  "devDependencies": {
    "jest": "^29.5.0",
    "supertest": "^6.3.3",
    "artillery": "^2.0.0",
    "jest-junit": "^16.0.0"
  }
}
Step 2: Create load testing configuration

mkdir -p tests/performance
nano tests/performance/load-test.yml
Add Artillery load test configuration:

config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Load test"
    - duration: 60
      arrivalRate: 100
      name: "Stress test"
  defaults:
    headers:
      User-Agent: "Artillery Load Test"

scenarios:
  - name: "Health check endpoint"
    weight: 30
    flow:
      - get:
          url: "/health"
          expect:
            - statusCode: 200
            - contentType: json
            - hasProperty: "status"

  - name: "Main endpoint"
    weight: 70
    flow:
      - get:
          url: "/"
          expect:
            - statusCode: 200
            - contentType: json
            - hasProperty: "message"
Step 3: Create performance test script

nano tests/performance/run-performance-tests.sh
Add the following script:

#!/bin/bash

set -e

echo "Starting performance tests..."

# Start application container for testing
docker run -d --name perf-test-container -p 3000:3000 $DOCKER_USERNAME/docker-circleci-lab:$CIRCLE_SHA1

# Wait for container to be ready
sleep 15

# Verify container is responding
curl -f http://localhost:3000/health

# Run load tests
echo "Running load tests with Artillery..."
npx artillery run tests/performance/load-test.yml --output performance-report.json

# Generate HTML report
npx artillery report performance-report.json --output performance-report.html

# Basic performance validation
echo "Validating performance metrics..."
node -e "
const report = require('./performance-report.json');
const summary = report.aggregate;

console.log('Performance Summary:');
console.log('- Total requests:', summary.requestsCompleted);
console.log('- Failed requests:', summary.errors || 0);
console.log('- Average response time:', summary.latency.mean + 'ms');
console.log('- 95th percentile:', summary.latency.p95 + 'ms');

// Fail if error rate > 1%
const errorRate = (summary.errors || 0) / summary.requestsCompleted * 100;
