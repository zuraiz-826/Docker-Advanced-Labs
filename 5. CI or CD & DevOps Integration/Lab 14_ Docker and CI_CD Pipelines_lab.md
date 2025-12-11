Lab 14: Docker and CI/CD Pipelines
Objectives
By the end of this lab, students will be able to:

Set up and configure GitLab CI/CD pipelines for Docker-based applications
Build and deploy Docker images automatically using CI/CD workflows
Use docker-compose to orchestrate multi-service deployments in pipelines
Implement proper Docker image versioning and tagging strategies
Integrate automated vulnerability scanning into CI/CD processes
Securely manage secrets and environment variables in containerized deployments
Understand best practices for Docker in production CI/CD environments
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and images
Familiarity with Git version control system
Basic knowledge of YAML syntax
Understanding of web application deployment concepts
Access to a GitLab account (free tier is sufficient)
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment. No need to build your own VM or install software locally.

Your cloud machine includes:

Docker Engine (latest stable version)
Docker Compose
Git client
Text editors (nano, vim)
curl and wget utilities
Task 1: Set up a GitLab CI/CD Pipeline to Build and Deploy Docker Images
Subtask 1.1: Create a Sample Application
First, let's create a simple Node.js web application that we'll containerize and deploy.

Create a new directory for your project:
mkdir docker-cicd-lab
cd docker-cicd-lab
Create a simple Node.js application:
nano app.js
Add the following content:

const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
    res.json({
        message: 'Hello from Docker CI/CD Pipeline!',
        version: process.env.APP_VERSION || '1.0.0',
        environment: process.env.NODE_ENV || 'development'
    });
});

app.get('/health', (req, res) => {
    res.status(200).json({ status: 'healthy' });
});

app.listen(port, () => {
    console.log(`Server running on port ${port}`);
});
Create a package.json file:
nano package.json
Add the following content:

{
  "name": "docker-cicd-app",
  "version": "1.0.0",
  "description": "Sample app for Docker CI/CD lab",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo \"No tests specified\" && exit 0"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
Subtask 1.2: Create a Dockerfile
Create a Dockerfile to containerize the application:

nano Dockerfile
Add the following content:

# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory in container
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --only=production

# Copy application code
COPY . .

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Change ownership of app directory
RUN chown -R nodejs:nodejs /app
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
Subtask 1.3: Initialize Git Repository and Connect to GitLab
Initialize Git repository:
git init
git add .
git commit -m "Initial commit: Add Node.js app and Dockerfile"
Create a new project on GitLab:

Go to GitLab.com and sign in
Click New Project
Choose Create blank project
Name it docker-cicd-lab
Set visibility to Private
Click Create project
Connect local repository to GitLab:

git remote add origin https://gitlab.com/YOUR_USERNAME/docker-cicd-lab.git
git branch -M main
git push -u origin main
Replace YOUR_USERNAME with your actual GitLab username.

Subtask 1.4: Create GitLab CI/CD Pipeline Configuration
Create the GitLab CI/CD configuration file:

nano .gitlab-ci.yml
Add the following content:

# Define stages for the pipeline
stages:
  - build
  - test
  - security
  - deploy

# Global variables
variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA

# Services needed for Docker-in-Docker
services:
  - docker:24-dind

# Before script to login to GitLab Container Registry
before_script:
  - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY

# Build stage - Build Docker image
build_image:
  stage: build
  image: docker:24
  script:
    - echo "Building Docker image..."
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker build -t $IMAGE_NAME:latest .
    - echo "Pushing image to registry..."
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:latest
  only:
    - main
    - develop

# Test stage - Run basic tests
test_application:
  stage: test
  image: docker:24
  script:
    - echo "Running application tests..."
    - docker run --rm $IMAGE_NAME:$IMAGE_TAG npm test
    - echo "Testing container health..."
    - docker run -d --name test-container -p 3000:3000 $IMAGE_NAME:$IMAGE_TAG
    - sleep 10
    - docker exec test-container curl -f http://localhost:3000/health || exit 1
    - docker stop test-container
    - docker rm test-container
  dependencies:
    - build_image
  only:
    - main
    - develop

# Security stage - Vulnerability scanning (placeholder)
security_scan:
  stage: security
  image: docker:24
  script:
    - echo "Running security scan..."
    - echo "Checking image for vulnerabilities..."
    # This is a placeholder - we'll implement actual scanning later
    - docker run --rm $IMAGE_NAME:$IMAGE_TAG echo "Security scan completed"
  dependencies:
    - build_image
  only:
    - main

# Deploy stage - Deploy to staging
deploy_staging:
  stage: deploy
  image: docker:24
  script:
    - echo "Deploying to staging environment..."
    - echo "Image $IMAGE_NAME:$IMAGE_TAG deployed successfully"
  environment:
    name: staging
    url: http://staging.example.com
  dependencies:
    - test_application
    - security_scan
  only:
    - main
Subtask 1.5: Commit and Push Pipeline Configuration
Add and commit the pipeline configuration:
git add .gitlab-ci.yml
git commit -m "Add GitLab CI/CD pipeline configuration"
git push origin main
Verify pipeline execution:
Go to your GitLab project
Navigate to CI/CD > Pipelines
You should see your pipeline running
Click on the pipeline to view detailed logs
Task 2: Use docker-compose to Deploy Services as Part of the Pipeline
Subtask 2.1: Create docker-compose Configuration
Create a docker-compose file for multi-service deployment:

nano docker-compose.yml
Add the following content:

version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=${NODE_ENV:-production}
      - APP_VERSION=${APP_VERSION:-1.0.0}
      - DATABASE_URL=mongodb://mongo:27017/myapp
    depends_on:
      - mongo
      - redis
    networks:
      - app-network
    restart: unless-stopped

  mongo:
    image: mongo:6.0
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${MONGO_USERNAME:-admin}
      - MONGO_INITDB_ROOT_PASSWORD=${MONGO_PASSWORD:-password123}
    volumes:
      - mongo-data:/data/db
    networks:
      - app-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD:-redis123}
    volumes:
      - redis-data:/data
    networks:
      - app-network
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - app
    networks:
      - app-network
    restart: unless-stopped

volumes:
  mongo-data:
  redis-data:

networks:
  app-network:
    driver: bridge
Subtask 2.2: Create Nginx Configuration
Create an Nginx configuration for load balancing:

nano nginx.conf
Add the following content:

events {
    worker_connections 1024;
}

http {
    upstream app_servers {
        server app:3000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://app_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            proxy_pass http://app_servers/health;
        }
    }
}
Subtask 2.3: Create Environment File Template
Create a template for environment variables:

nano .env.example
Add the following content:

# Application Configuration
NODE_ENV=production
APP_VERSION=1.0.0

# Database Configuration
MONGO_USERNAME=admin
MONGO_PASSWORD=secure_password_here

# Redis Configuration
REDIS_PASSWORD=redis_secure_password
Subtask 2.4: Update GitLab CI/CD Pipeline for docker-compose
Update the .gitlab-ci.yml file to include docker-compose deployment:

nano .gitlab-ci.yml
Replace the existing content with:

stages:
  - build
  - test
  - security
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  COMPOSE_PROJECT_NAME: docker-cicd-lab

services:
  - docker:24-dind

before_script:
  - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  - apk add --no-cache docker-compose

build_image:
  stage: build
  image: docker:24
  script:
    - echo "Building Docker image..."
    - docker build -t $IMAGE_NAME:$IMAGE_TAG .
    - docker build -t $IMAGE_NAME:latest .
    - docker push $IMAGE_NAME:$IMAGE_TAG
    - docker push $IMAGE_NAME:latest
  only:
    - main
    - develop

test_application:
  stage: test
  image: docker:24
  script:
    - echo "Running docker-compose tests..."
    - export APP_VERSION=$CI_COMMIT_SHORT_SHA
    - docker-compose -f docker-compose.yml up -d app mongo redis
    - sleep 30
    - docker-compose exec -T app npm test
    - docker-compose exec -T app curl -f http://localhost:3000/health || exit 1
    - docker-compose down -v
  dependencies:
    - build_image
  only:
    - main
    - develop

security_scan:
  stage: security
  image: docker:24
  script:
    - echo "Running security scan on all services..."
    - docker run --rm $IMAGE_NAME:$IMAGE_TAG echo "App security scan completed"
    - docker run --rm mongo:6.0 echo "MongoDB security check completed"
    - docker run --rm redis:7-alpine echo "Redis security check completed"
  dependencies:
    - build_image
  only:
    - main

deploy_staging:
  stage: deploy
  image: docker:24
  script:
    - echo "Deploying full stack to staging..."
    - export NODE_ENV=staging
    - export APP_VERSION=$CI_COMMIT_SHORT_SHA
    - export MONGO_USERNAME=$STAGING_MONGO_USERNAME
    - export MONGO_PASSWORD=$STAGING_MONGO_PASSWORD
    - export REDIS_PASSWORD=$STAGING_REDIS_PASSWORD
    - docker-compose up -d
    - echo "Waiting for services to be ready..."
    - sleep 30
    - docker-compose ps
    - echo "Full stack deployed successfully"
  environment:
    name: staging
    url: http://staging.example.com
  dependencies:
    - test_application
    - security_scan
  only:
    - main
Task 3: Implement Docker Image Versioning and Tagging Strategies
Subtask 3.1: Create Advanced Tagging Strategy
Create a script for advanced image tagging:

nano scripts/tag-image.sh
Add the following content:

#!/bin/bash

# Advanced Docker image tagging script
set -e

# Variables
IMAGE_NAME=${1:-$CI_REGISTRY_IMAGE}
COMMIT_SHA=${2:-$CI_COMMIT_SHORT_SHA}
BRANCH_NAME=${3:-$CI_COMMIT_REF_NAME}
BUILD_NUMBER=${4:-$CI_PIPELINE_ID}

# Function to create semantic version tag
create_semantic_tag() {
    local version_file="VERSION"
    if [ -f "$version_file" ]; then
        VERSION=$(cat $version_file)
    else
        VERSION="1.0.0"
    fi
    
    # Increment patch version for main branch
    if [ "$BRANCH_NAME" = "main" ]; then
        MAJOR=$(echo $VERSION | cut -d. -f1)
        MINOR=$(echo $VERSION | cut -d. -f2)
        PATCH=$(echo $VERSION | cut -d. -f3)
        PATCH=$((PATCH + 1))
        NEW_VERSION="$MAJOR.$MINOR.$PATCH"
        echo $NEW_VERSION > $version_file
        echo $NEW_VERSION
    else
        echo "$VERSION-$BRANCH_NAME"
    fi
}

# Create tags
SEMANTIC_TAG=$(create_semantic_tag)
TIMESTAMP=$(date +%Y%m%d-%H%M%S)

echo "Creating multiple tags for image: $IMAGE_NAME"

# Tag with commit SHA
docker tag $IMAGE_NAME:latest $IMAGE_NAME:$COMMIT_SHA
echo "Tagged with commit SHA: $COMMIT_SHA"

# Tag with semantic version
docker tag $IMAGE_NAME:latest $IMAGE_NAME:$SEMANTIC_TAG
echo "Tagged with semantic version: $SEMANTIC_TAG"

# Tag with branch name
docker tag $IMAGE_NAME:latest $IMAGE_NAME:$BRANCH_NAME
echo "Tagged with branch name: $BRANCH_NAME"

# Tag with timestamp
docker tag $IMAGE_NAME:latest $IMAGE_NAME:$TIMESTAMP
echo "Tagged with timestamp: $TIMESTAMP"

# Tag with build number
docker tag $IMAGE_NAME:latest $IMAGE_NAME:build-$BUILD_NUMBER
echo "Tagged with build number: build-$BUILD_NUMBER"

# Push all tags
echo "Pushing all tags..."
docker push $IMAGE_NAME:$COMMIT_SHA
docker push $IMAGE_NAME:$SEMANTIC_TAG
docker push $IMAGE_NAME:$BRANCH_NAME
docker push $IMAGE_NAME:$TIMESTAMP
docker push $IMAGE_NAME:build-$BUILD_NUMBER

echo "All tags pushed successfully!"
Subtask 3.2: Create VERSION File
Create a version file for semantic versioning:

echo "1.0.0" > VERSION
Subtask 3.3: Update Pipeline with Advanced Tagging
Create a new version of the pipeline with advanced tagging:

nano .gitlab-ci.yml
Update the build stage:

stages:
  - build
  - test
  - security
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  IMAGE_TAG: $CI_COMMIT_SHORT_SHA
  COMPOSE_PROJECT_NAME: docker-cicd-lab

services:
  - docker:24-dind

before_script:
  - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
  - apk add --no-cache docker-compose

build_and_tag:
  stage: build
  image: docker:24
  script:
    - echo "Building Docker image with advanced tagging..."
    - mkdir -p scripts
    - chmod +x scripts/tag-image.sh
    
    # Build base image
    - docker build -t $IMAGE_NAME:latest .
    
    # Create and push multiple tags
    - ./scripts/tag-image.sh $IMAGE_NAME $CI_COMMIT_SHORT_SHA $CI_COMMIT_REF_NAME $CI_PIPELINE_ID
    
    # Update VERSION file if on main branch
    - |
      if [ "$CI_COMMIT_REF_NAME" = "main" ]; then
        git config --global user.email "ci@gitlab.com"
        git config --global user.name "GitLab CI"
        git add VERSION
        git commit -m "Bump version [skip ci]" || echo "No version changes"
        git push https://gitlab-ci-token:${CI_PUSH_TOKEN}@gitlab.com/$CI_PROJECT_PATH.git HEAD:$CI_COMMIT_REF_NAME || echo "Could not push version update"
      fi
  artifacts:
    paths:
      - VERSION
    expire_in: 1 week
  only:
    - main
    - develop
    - /^release\/.*$/

# Rest of the pipeline remains the same...
test_application:
  stage: test
  image: docker:24
  script:
    - echo "Testing with tagged image..."
    - export APP_VERSION=$(cat VERSION)
    - docker-compose -f docker-compose.yml up -d app mongo redis
    - sleep 30
    - docker-compose exec -T app npm test
    - docker-compose exec -T app curl -f http://localhost:3000/health || exit 1
    - docker-compose down -v
  dependencies:
    - build_and_tag
  only:
    - main
    - develop
    - /^release\/.*$/
Task 4: Automate Vulnerability Scanning During CI/CD Pipeline
Subtask 4.1: Install and Configure Trivy Scanner
Create a comprehensive security scanning stage:

nano .gitlab-ci.yml
Update the security stage:

# Add this to the existing .gitlab-ci.yml file

security_comprehensive_scan:
  stage: security
  image: docker:24
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
    # Install Trivy
    - apk add --no-cache curl
    - curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
  script:
    - echo "Running comprehensive security scans..."
    
    # Scan the application image
    - echo "Scanning application image for vulnerabilities..."
    - trivy image --exit-code 0 --severity HIGH,CRITICAL --format table $IMAGE_NAME:$IMAGE_TAG
    - trivy image --exit-code 1 --severity CRITICAL --format json --output app-scan-results.json $IMAGE_NAME:$IMAGE_TAG || echo "Critical vulnerabilities found in app image"
    
    # Scan base images used in docker-compose
    - echo "Scanning MongoDB image..."
    - trivy image --exit-code 0 --severity HIGH,CRITICAL --format table mongo:6.0
    
    - echo "Scanning Redis image..."
    - trivy image --exit-code 0 --severity HIGH,CRITICAL --format table redis:7-alpine
    
    - echo "Scanning Nginx image..."
    - trivy image --exit-code 0 --severity HIGH,CRITICAL --format table nginx:alpine
    
    # Scan Dockerfile for misconfigurations
    - echo "Scanning Dockerfile for security issues..."
    - trivy config --exit-code 0 --severity HIGH,CRITICAL .
    
    # Generate security report
    - echo "Generating security report..."
    - |
      cat > security-report.md << EOF
      # Security Scan Report
      
      ## Application Image Scan
      - Image: $IMAGE_NAME:$IMAGE_TAG
      - Scan Date: $(date)
      - Status: $([ -f app-scan-results.json ] && echo "Completed with findings" || echo "Clean")
      
      ## Base Images Status
      - MongoDB 6.0: Scanned
      - Redis 7 Alpine: Scanned  
      - Nginx Alpine: Scanned
      
      ## Dockerfile Security Check
      - Configuration scan completed
      
      For detailed results, check the pipeline logs.
      EOF
    
    - cat security-report.md
    
  artifacts:
    paths:
      - app-scan-results.json
      - security-report.md
    expire_in: 1 week
    reports:
      # GitLab can parse this for security dashboard
      container_scanning: app-scan-results.json
  dependencies:
    - build_and_tag
  allow_failure: true  # Don't fail pipeline on security issues, but report them
  only:
    - main
    - develop
Subtask 4.2: Create Custom Security Policy
Create a security policy file:

nano .trivyignore
Add content to ignore certain vulnerabilities if needed:

# Ignore specific CVEs that are false positives or accepted risks
# CVE-2023-12345
# CVE-2023-67890

# Ignore low severity issues in development
# GHSA-*
Subtask 4.3: Add Security Scanning to docker-compose Services
Create a script to scan all services in docker-compose:

nano scripts/scan-compose-services.sh
Add the following content:

#!/bin/bash

# Script to scan all docker-compose services
set -e

echo "Scanning all docker-compose services..."

# Extract service images from docker-compose.yml
SERVICES=$(docker-compose config --services)

for service in $SERVICES; do
    echo "Scanning service: $service"
    
    # Get the image for this service
    IMAGE=$(docker-compose config | grep -A 10 "^  $service:" | grep "image:" | awk '{print $2}' | head -1)
    
    if [ ! -z "$IMAGE" ]; then
        echo "Scanning image: $IMAGE"
        trivy image --exit-code 0 --severity HIGH,CRITICAL --format table $IMAGE
        echo "---"
    else
        echo "Service $service uses build context, skipping image scan"
    fi
done

echo "All service scans completed!"
Make the script executable:

chmod +x scripts/scan-compose-services.sh
Task 5: Use GitLab CI/CD Variables to Inject Secrets and Environment Variables
Subtask 5.1: Configure GitLab CI/CD Variables
Navigate to your GitLab project
Go to Settings > CI/CD
Expand the Variables section
Add the following variables:
Production Variables:

Key: PROD_MONGO_USERNAME, Value: prod_admin, Protected: ✓, Masked: ✓
Key: PROD_MONGO_PASSWORD, Value: super_secure_prod_password, Protected: ✓, Masked: ✓
Key: PROD_REDIS_PASSWORD, Value: redis_prod_secure_pass, Protected: ✓, Masked: ✓
Key: PROD_DATABASE_URL, Value: mongodb://prod_admin:super_secure_prod_password@mongo:27017/proddb, Protected: ✓
Staging Variables:

Key: STAGING_MONGO_USERNAME, Value: staging_admin, Masked: ✓
Key: STAGING_MONGO_PASSWORD, Value: staging_secure_password, Masked: ✓
Key: STAGING_REDIS_PASSWORD, Value: redis_staging_pass, Masked: ✓
General Variables:

Key: CI_PUSH_TOKEN, Value: your_gitlab_token_here, Masked: ✓
Key: NOTIFICATION_WEBHOOK, Value: https://hooks.slack.com/your/webhook/url
Subtask 5.2: Create Environment-Specific docker-compose Files
Create staging-specific compose file:

nano docker-compose.staging.yml
Add the following content:

version: '3.8'

services:
  app:
    environment:
      - NODE_ENV=staging
      - APP_VERSION=${APP_VERSION}
      - DATABASE_URL=mongodb://${STAGING_MONGO_USERNAME}:${STAGING_MONGO_PASSWORD}@mongo:27017/stagingdb
      - REDIS_URL=redis://:${STAGING_REDIS_PASSWORD}@redis:6379
    ports:
      - "3001:3000"  # Different port for staging

  mongo:
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${STAGING_MONGO_USERNAME}
      - MONGO_INITDB_ROOT_PASSWORD=${STAGING_MONGO_PASSWORD}
      - MONGO_INITDB_DATABASE=stagingdb

  redis:
    command: redis-server --requirepass ${STAGING_REDIS_PASSWORD}
Create production-specific compose file:

nano docker-compose.production.yml
Add the following content:

version: '3.8'

services:
  app:
    environment:
      - NODE_ENV=production
      - APP_VERSION=${APP_VERSION}
      - DATABASE_URL=${PROD_DATABASE_URL}
      - REDIS_URL=redis://:${PROD_REDIS_PASSWORD}@redis:6379
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

  mongo:
    environment:
      - MONGO_INITDB_ROOT_USERNAME=${PROD_MONGO_USERNAME}
      - MONGO_INITDB_ROOT_PASSWORD=${PROD_MONGO_PASSWORD}
      - MONGO_INITDB_DATABASE=proddb
    volumes:
      - prod-mongo-data:/data/db

  redis:
    command: redis-server --requirepass ${PROD_REDIS_PASSWORD} --appendonly yes
    volumes:
      - prod-redis-data:/data

volumes:
  prod-mongo-data:
  prod-redis-data:
Subtask 5.3: Update Pipeline with Secret Management
Update the .gitlab-ci.yml file to use secrets properly:

nano .gitlab-ci.yml
Add the following deployment stages:

# Add these stages to your existing .gitlab-ci.yml

deploy_staging:
  stage: deploy
  image: docker:24
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
    - apk add --no-cache docker-compose
  script:
    - echo "Deploying to staging environment..."
    
    # Set environment variables from GitLab CI/CD variables
    - export APP_VERSION=$(cat VERSION)
    - export STAGING_MONGO_USERNAME=$STAGING_MONGO_USERNAME
    - export STAGING_MONGO_PASSWORD=$STAGING_MONGO_PASSWORD
    - export STAGING_REDIS_PASSWORD=$STAGING_REDIS_PASSWORD
    
    # Deploy using staging configuration
    - docker-compose -f docker-compose.yml -f docker-compose.staging.yml up -d
    
    # Wait for services to be ready
    - echo "Waiting for services to start..."
    - sleep 45
    
    # Health check
    - docker-compose exec -T app curl -f http://localhost:3000/health || exit 1
    
    # Show deployment status
    - docker-compose ps
    - echo "Staging deployment completed successfully!"
    
    # Send notification (if webhook is configured)
    - |
      if [ ! -z "$NOTIFICATION_WEBHOOK" ]; then
        curl -X POST -H 'Content-type: application/json' \
        --data '{"text":"🚀 Staging deployment successful for '"$CI_PROJECT_NAME"' version '"$(cat VERSION)"'"}' \
        $NOTIFICATION_WEBHOOK || echo "Notification failed"
      fi
  environment:
    name: staging
    url: http://staging.example.com:3001
  dependencies:
    - test_application
    - security_comprehensive_scan
  only:
    - main
  when: manual  # Require manual approval for deployment

deploy_production:
  stage: deploy
  image: docker:24
  before_script:
    - echo $CI_REGISTRY_PASSWORD | docker login -u $CI_REGISTRY_USER --password-stdin $CI_REGISTRY
    - apk add --no-cache docker-compose
  script:
    - echo "Deploying to production environment..."
    
    # Set production environment variables
    - export APP_VERSION=$(cat VERSION)
    - export PROD_MONGO_USERNAME=$PROD_MONGO_USERNAME
    - export PROD_MONGO_PASSWORD=$PROD_MONGO_PASSWORD
    - export PROD_REDIS_PASSWORD=$PROD_REDIS_PASSWORD
    - export PROD_DATABASE_URL=$PROD_DATABASE_URL
    
    # Deploy using production configuration
    - docker-compose -f docker-compose.yml -f docker-compose.production.yml up -d
    
    # Extended wait for production
    - echo "Waiting for production services to start..."
    - sleep 60
    
    # Comprehensive health checks
    - docker-compose exec -T app curl -f http://localhost:3000/health || exit 1
    - echo "Production health check passed"
    
    # Show deployment status
    - docker-compose ps
    - echo "Production deployment completed successfully!"
    
    # Send success notification
    - |
      if [ ! -z "$NOTIFICATION_WEBHOOK" ]; then
        curl -X POST -H 'Content-type: application/json' \
        --data '{"text":"🎉 Production deployment successful for '"$CI_PROJECT_NAME"' version '"$(cat VERSION)"'"}' \
        $NOTIFICATION_WEBHOOK || echo "Notification failed"
      fi
  environment:
    name: production
    url: http://production.example.com
  dependencies:
    - deploy_staging
  only:
    - main
  when: manual  # Require manual approval for production
Subtask 5.4: Create Secret Validation Script
Create a script to validate that all required secrets are available:

nano scripts/validate-secrets.sh
Add the following content:

#!/bin/bash

# Script to validate required secrets are available
set -e

echo "Validating required secrets and environment variables..."

# Function to check if variable is set and not empty
check_var() {
    local var_name=$1
    local var_value=${!var_name}
    
    if [ -z "$var_value" ]; then
        echo "❌ ERROR: $var_name is not set or empty"
