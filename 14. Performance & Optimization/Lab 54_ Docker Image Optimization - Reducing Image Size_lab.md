Lab 54: Docker Image Optimization - Reducing Image Size
Lab Objectives
By the end of this lab, you will be able to:

Understand the importance of Docker image optimization for performance and storage efficiency
Implement multi-stage builds to create smaller, more efficient Docker images
Utilize minimal base images like Alpine Linux to reduce image footprint
Optimize Dockerfile structure to minimize layers and improve build performance
Remove unnecessary build dependencies from production images
Use .dockerignore files to exclude unnecessary files from build context
Compare image sizes before and after optimization techniques
Apply best practices for creating production-ready Docker images
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, Dockerfiles)
Familiarity with Linux command line operations
Basic knowledge of application development (we'll use a simple Node.js app as an example)
Understanding of file systems and directory structures
Note: Al Nafi provides pre-configured Linux-based cloud machines with Docker already installed. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Lab Environment Setup
Your Al Nafi cloud machine comes with:

Docker Engine (latest stable version)
Docker Compose
Basic development tools
Text editors (nano, vim)
Git for version control
Task 1: Create a Sample Application and Unoptimized Dockerfile
Subtask 1.1: Set Up the Working Directory
First, let's create a workspace for our optimization exercises.

# Create and navigate to the lab directory
mkdir ~/docker-optimization-lab
cd ~/docker-optimization-lab

# Create application directory
mkdir sample-app
cd sample-app
Subtask 1.2: Create a Simple Node.js Application
We'll create a basic Node.js web application to demonstrate optimization techniques.

# Create package.json file
cat > package.json << 'EOF'
{
  "name": "docker-optimization-demo",
  "version": "1.0.0",
  "description": "Sample app for Docker optimization",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
EOF
# Create the main application file
cat > server.js << 'EOF'
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

app.get('/', (req, res) => {
    res.json({
        message: 'Hello from optimized Docker container!',
        timestamp: new Date().toISOString(),
        environment: process.env.NODE_ENV || 'development'
    });
});

app.get('/health', (req, res) => {
    res.status(200).json({ status: 'healthy' });
});

app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on port ${PORT}`);
});
EOF
Subtask 1.3: Create an Unoptimized Dockerfile
Let's start with a typical, unoptimized Dockerfile to establish a baseline.

# Create unoptimized Dockerfile
cat > Dockerfile.unoptimized << 'EOF'
FROM node:18

# Set working directory
WORKDIR /app

# Copy all files
COPY . .

# Install all dependencies (including dev dependencies)
RUN npm install

# Install additional tools that might not be needed in production
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    vim \
    git \
    build-essential \
    python3 \
    && rm -rf /var/lib/apt/lists/*

# Expose port
EXPOSE 3000

# Start the application
CMD ["npm", "start"]
EOF
Subtask 1.4: Build and Analyze the Unoptimized Image
# Build the unoptimized image
docker build -f Dockerfile.unoptimized -t sample-app:unoptimized .

# Check the image size
docker images sample-app:unoptimized

# Get detailed size information
docker history sample-app:unoptimized
Expected Output: You should see an image size of approximately 1.2-1.5 GB due to the full Node.js base image and additional packages.

Task 2: Implement Multi-Stage Builds
Subtask 2.1: Understanding Multi-Stage Builds
Multi-stage builds allow you to use multiple FROM statements in your Dockerfile. Each FROM instruction can use a different base, and you can selectively copy artifacts from one stage to another.

Subtask 2.2: Create a Multi-Stage Dockerfile
# Create multi-stage Dockerfile
cat > Dockerfile.multistage << 'EOF'
# Build stage
FROM node:18 AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev dependencies for building)
RUN npm ci --only=production

# Production stage
FROM node:18-slim AS production

WORKDIR /app

# Create non-root user for security
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Copy only production dependencies from builder stage
COPY --from=builder /app/node_modules ./node_modules

# Copy application code
COPY server.js ./

# Change ownership to non-root user
RUN chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Start the application
CMD ["node", "server.js"]
EOF
Subtask 2.3: Build and Compare Multi-Stage Image
# Build the multi-stage image
docker build -f Dockerfile.multistage -t sample-app:multistage .

# Compare image sizes
echo "=== Image Size Comparison ==="
docker images | grep sample-app
Task 3: Use Minimal Base Images (Alpine)
Subtask 3.1: Understanding Alpine Linux
Alpine Linux is a security-oriented, lightweight Linux distribution based on musl libc and busybox. It's significantly smaller than traditional distributions.

Subtask 3.2: Create Alpine-Based Dockerfile
# Create Alpine-based Dockerfile
cat > Dockerfile.alpine << 'EOF'
# Build stage with Alpine
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && npm cache clean --force

# Production stage with Alpine
FROM node:18-alpine AS production

# Install curl for health checks (Alpine uses apk package manager)
RUN apk add --no-cache curl

WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S appuser && \
    adduser -S appuser -u 1001

# Copy production dependencies from builder
COPY --from=builder /app/node_modules ./node_modules

# Copy application code
COPY server.js ./

# Change ownership to non-root user
RUN chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["node", "server.js"]
EOF
Subtask 3.3: Build and Test Alpine Image
# Build Alpine-based image
docker build -f Dockerfile.alpine -t sample-app:alpine .

# Test the Alpine image
docker run -d --name test-alpine -p 3001:3000 sample-app:alpine

# Test the application
curl http://localhost:3001
curl http://localhost:3001/health

# Stop and remove test container
docker stop test-alpine
docker rm test-alpine

# Compare all image sizes so far
echo "=== Complete Image Size Comparison ==="
docker images | grep sample-app
Task 4: Minimize Dockerfile Layers
Subtask 4.1: Understanding Layer Optimization
Each instruction in a Dockerfile creates a new layer. Combining commands reduces the number of layers and can reduce image size.

Subtask 4.2: Create Layer-Optimized Dockerfile
# Create layer-optimized Dockerfile
cat > Dockerfile.optimized << 'EOF'
# Multi-stage build with Alpine and optimized layers
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files first (for better caching)
COPY package*.json ./

# Install dependencies and clean up in single layer
RUN npm ci --only=production && \
    npm cache clean --force

# Production stage
FROM node:18-alpine AS production

# Install runtime dependencies and create user in single layer
RUN apk add --no-cache curl && \
    addgroup -g 1001 -S appuser && \
    adduser -S appuser -u 1001

WORKDIR /app

# Copy dependencies and application code
COPY --from=builder /app/node_modules ./node_modules
COPY server.js ./

# Set ownership and switch user in single layer
RUN chown -R appuser:appuser /app
USER appuser

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
EOF
Subtask 4.3: Build Layer-Optimized Image
# Build layer-optimized image
docker build -f Dockerfile.optimized -t sample-app:optimized .

# Analyze layers in the optimized image
docker history sample-app:optimized

# Compare with previous versions
echo "=== Layer Count Comparison ==="
echo "Unoptimized layers:"
docker history sample-app:unoptimized --format "table {{.CreatedBy}}" | wc -l

echo "Optimized layers:"
docker history sample-app:optimized --format "table {{.CreatedBy}}" | wc -l
Task 5: Remove Unnecessary Build Dependencies
Subtask 5.1: Create Production-Ready Dockerfile
# Create production-ready Dockerfile with minimal dependencies
cat > Dockerfile.production << 'EOF'
# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ALL dependencies (including dev) for building
RUN npm ci

# If we had build steps, they would go here
# RUN npm run build

# Production stage - minimal runtime image
FROM node:18-alpine AS production

# Install only essential runtime packages
RUN apk add --no-cache \
    curl \
    tini && \
    addgroup -g 1001 -S appuser && \
    adduser -S appuser -u 1001

WORKDIR /app

# Copy only production node_modules (rebuilt for production)
COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application code
COPY server.js ./

# Set proper ownership
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

EXPOSE 3000

# Use tini as init system for proper signal handling
ENTRYPOINT ["/sbin/tini", "--"]

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "server.js"]
EOF
Subtask 5.2: Build and Test Production Image
# Build production image
docker build -f Dockerfile.production -t sample-app:production .

# Test the production image
docker run -d --name test-production -p 3002:3000 sample-app:production

# Verify it works
curl http://localhost:3002
curl http://localhost:3002/health

# Check what's running inside the container
docker exec test-production ps aux

# Clean up
docker stop test-production
docker rm test-production
Task 6: Use .dockerignore to Exclude Unnecessary Files
Subtask 6.1: Create Test Files and Directories
# Create some files that shouldn't be in the Docker image
mkdir -p docs tests logs temp
echo "This is documentation" > docs/README.md
echo "This is a test file" > tests/test.js
echo "This is a log file" > logs/app.log
echo "This is temporary" > temp/temp.txt

# Create some hidden files
echo "node_modules/" > .gitignore
echo "*.log" >> .gitignore
touch .env.local
touch .DS_Store

# Create a large dummy file to demonstrate exclusion
dd if=/dev/zero of=large-file.bin bs=1M count=10

# List all files to see what we have
ls -la
du -sh *
Subtask 6.2: Create .dockerignore File
# Create comprehensive .dockerignore file
cat > .dockerignore << 'EOF'
# Version control
.git
.gitignore

# Dependencies (will be installed in container)
node_modules
npm-debug.log*

# Environment files
.env
.env.local
.env.*.local

# IDE and editor files
.vscode
.idea
*.swp
*.swo
*~

# OS generated files
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# Documentation and tests (not needed in production)
docs/
tests/
*.md
README*

# Log files
logs/
*.log

# Temporary files
temp/
tmp/
*.tmp

# Large files that aren't needed
*.bin
*.iso
*.tar
*.zip

# Docker files (except the one being used)
Dockerfile*
!Dockerfile.production
docker-compose*.yml

# Build artifacts
dist/
build/
coverage/
EOF
Subtask 6.3: Build Image with .dockerignore
# Build image with .dockerignore in place
docker build -f Dockerfile.production -t sample-app:final .

# Compare the build context size by temporarily renaming .dockerignore
mv .dockerignore .dockerignore.backup

echo "=== Building without .dockerignore ==="
docker build -f Dockerfile.production -t sample-app:no-ignore . 2>&1 | grep "Sending build context"

# Restore .dockerignore
mv .dockerignore.backup .dockerignore

echo "=== Building with .dockerignore ==="
docker build -f Dockerfile.production -t sample-app:with-ignore . 2>&1 | grep "Sending build context"
Subtask 6.4: Verify File Exclusion
# Check what files are actually in the final image
docker run --rm sample-app:final ls -la /app

# Compare with a container that includes everything
docker run --rm sample-app:no-ignore ls -la /app

# Check the size difference
echo "=== Final Size Comparison ==="
docker images | grep sample-app
Task 7: Final Optimization and Best Practices
Subtask 7.1: Create Ultimate Optimized Dockerfile
# Create the ultimate optimized Dockerfile combining all techniques
cat > Dockerfile.ultimate << 'EOF'
# syntax=docker/dockerfile:1.4
# Use BuildKit for advanced features

# Build stage
FROM node:18-alpine AS builder

WORKDIR /app

# Copy package files first for better layer caching
COPY package*.json ./

# Install dependencies with cache mount for faster builds
RUN --mount=type=cache,target=/root/.npm \
    npm ci --only=production && \
    npm cache clean --force

# Production stage - distroless for maximum security and minimal size
FROM gcr.io/distroless/nodejs18-debian11 AS production

WORKDIR /app

# Copy production dependencies
COPY --from=builder /app/node_modules ./node_modules

# Copy application code
COPY server.js ./

# Expose port
EXPOSE 3000

# Use non-root user (distroless default)
USER nonroot:nonroot

# Start application
CMD ["server.js"]
EOF
Subtask 7.2: Build Ultimate Optimized Image
# Enable BuildKit for advanced features
export DOCKER_BUILDKIT=1

# Build ultimate optimized image
docker build -f Dockerfile.ultimate -t sample-app:ultimate .

# If distroless fails (not available), create alternative minimal version
cat > Dockerfile.minimal << 'EOF'
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM alpine:3.18 AS production
RUN apk add --no-cache nodejs && \
    addgroup -g 1001 -S appuser && \
    adduser -S appuser -u 1001
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY server.js ./
RUN chown -R appuser:appuser /app
USER appuser
EXPOSE 3000
CMD ["node", "server.js"]
EOF

# Build minimal version as backup
docker build -f Dockerfile.minimal -t sample-app:minimal .
Task 8: Performance Testing and Comparison
Subtask 8.1: Create Comprehensive Comparison
# Create a script to compare all images
cat > compare_images.sh << 'EOF'
#!/bin/bash

echo "=== Docker Image Optimization Results ==="
echo "========================================"
echo

echo "Image Sizes:"
echo "------------"
docker images | grep sample-app | sort -k7 -h

echo
echo "Detailed Size Analysis:"
echo "----------------------"

for tag in unoptimized multistage alpine optimized production final minimal; do
    if docker images sample-app:$tag &>/dev/null; then
        size=$(docker images sample-app:$tag --format "{{.Size}}")
        echo "sample-app:$tag - $size"
    fi
done

echo
echo "Layer Analysis (Top 3 optimized images):"
echo "----------------------------------------"

for tag in production final minimal; do
    if docker images sample-app:$tag &>/dev/null; then
        echo "Layers in sample-app:$tag:"
        docker history sample-app:$tag --format "table {{.Size}}\t{{.CreatedBy}}" | head -10
        echo
    fi
done
EOF

chmod +x compare_images.sh
./compare_images.sh
Subtask 8.2: Test Application Performance
# Test startup time for different images
echo "=== Startup Time Comparison ==="

for tag in unoptimized alpine production minimal; do
    if docker images sample-app:$tag &>/dev/null; then
        echo "Testing sample-app:$tag startup time..."
        time docker run --rm -d --name test-$tag -p 3000:3000 sample-app:$tag
        sleep 2
        curl -s http://localhost:3000 > /dev/null && echo "✓ App started successfully"
        docker stop test-$tag > /dev/null
        echo "---"
    fi
done
Task 9: Security Scanning and Vulnerability Assessment
Subtask 9.1: Scan Images for Vulnerabilities
# Install Docker Scout (if not available, we'll use alternative methods)
# Note: This might not be available in all environments

echo "=== Security Analysis ==="

# Check for common security issues
for tag in unoptimized alpine production minimal; do
    if docker images sample-app:$tag &>/dev/null; then
        echo "Analyzing sample-app:$tag..."
        
        # Check if running as root
        user=$(docker run --rm sample-app:$tag whoami 2>/dev/null || echo "unknown")
        echo "  Running as user: $user"
        
        # Check for package managers in final image
        has_apt=$(docker run --rm sample-app:$tag which apt 2>/dev/null && echo "yes" || echo "no")
        has_apk=$(docker run --rm sample-app:$tag which apk 2>/dev/null && echo "yes" || echo "no")
        echo "  Has apt: $has_apt, Has apk: $has_apk"
        
        echo "---"
    fi
done
Task 10: Create Production Deployment Configuration
Subtask 10.1: Create Docker Compose for Production
# Create production docker-compose file
cat > docker-compose.prod.yml << 'EOF'
version: '3.8'

services:
  app:
    image: sample-app:minimal
    container_name: optimized-app
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.5'
        reservations:
          memory: 64M
          cpus: '0.25'
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100m

networks:
  default:
    driver: bridge
EOF
Subtask 10.2: Test Production Deployment
# Deploy using docker-compose
docker-compose -f docker-compose.prod.yml up -d

# Check the deployment
docker-compose -f docker-compose.prod.yml ps

# Test the application
curl http://localhost:3000
curl http://localhost:3000/health

# Check resource usage
docker stats optimized-app --no-stream

# Clean up
docker-compose -f docker-compose.prod.yml down
Troubleshooting Common Issues
Issue 1: Build Context Too Large
Problem: Docker build is slow due to large build context.

Solution:

# Check build context size
echo "Build context contents:"
tar -czh . | wc -c

# Improve .dockerignore
echo "node_modules" >> .dockerignore
echo "*.log" >> .dockerignore
echo ".git" >> .dockerignore
Issue 2: Multi-stage Build Failures
Problem: COPY --from=builder fails.

Solution:

# Ensure the builder stage name matches
# Verify files exist in builder stage
docker build --target builder -t debug-builder .
docker run --rm debug-builder ls -la /app
Issue 3: Alpine Package Issues
Problem: Packages not found in Alpine.

Solution:

# Search for Alpine packages
docker run --rm alpine:3.18 apk search curl

# Use Alpine package names
RUN apk add --no-cache curl-dev
Issue 4: Permission Issues
Problem: Application can't write files.

Solution:

# Create writable directories
RUN mkdir -p /app/logs && chown appuser:appuser /app/logs

# Or use tmpfs for temporary files
tmpfs:
  - /app/tmp:noexec,nosuid,size=100m
Lab Summary and Key Takeaways
What You Accomplished
In this lab, you successfully:

Created baseline measurements by building an unoptimized Docker image and measuring its size and performance
Implemented multi-stage builds to separate build dependencies from runtime requirements
Utilized Alpine Linux as a minimal base image to significantly reduce image size
Optimized Dockerfile layers by combining commands and reducing the total number of layers
Removed unnecessary build dependencies from production images
Used .dockerignore to exclude unnecessary files from the build context
Applied security best practices including non-root users and minimal attack surface
Performed comprehensive testing to validate optimizations didn't break functionality
Key Optimization Results
Your optimization journey likely achieved:

Size reduction: From ~1.5GB to ~50-100MB (90%+ reduction)
Layer reduction: From 15+ layers to 8-10 layers
Security improvement: Non-root user, minimal packages
Build speed: Faster builds due to better caching and smaller context
Startup time: Faster container startup due to smaller image size
Best Practices Learned
Use multi-stage builds for applications requiring build tools
Choose minimal base images like Alpine or distroless
Optimize layer caching by copying package files before source code
Combine RUN commands to reduce layers
Use .dockerignore to exclude unnecessary files
Run as non-root user for security
Include health checks for production readiness
Clean package caches after installation
Real-World Impact
These optimization techniques provide significant benefits in production:

Reduced storage costs in container registries
Faster deployment times due to smaller image pulls
Improved security posture with minimal attack surface
Better resource utilization in container orchestration platforms
Enhanced developer productivity with faster build times
Next Steps
To continue your Docker optimization journey:

Explore distroless images for even smaller production images
Learn about BuildKit features for advanced optimizations
Implement automated security scanning in CI/CD pipelines
Study container orchestration optimization techniques
Practice with different application stacks (Python, Java, Go)
This lab has equipped you with essential skills for creating production-ready, optimized Docker images that are secure, efficient, and maintainable.
