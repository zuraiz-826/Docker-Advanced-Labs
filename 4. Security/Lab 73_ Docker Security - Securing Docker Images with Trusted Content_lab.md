Lab 73: Docker Security - Securing Docker Images with Trusted Content
Objectives
By the end of this lab, students will be able to:

Understand and implement Docker Content Trust for image signing and verification
Use image scanning tools to identify vulnerabilities in Docker images
Apply security best practices when creating Dockerfiles
Implement the least privilege principle for container access
Set up automated vulnerability scanning in CI/CD pipelines
Secure Docker environments using open-source tools and techniques
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (containers, images, Dockerfiles)
Familiarity with Linux command line operations
Basic knowledge of YAML syntax
Understanding of CI/CD pipeline concepts
Access to a terminal or command prompt
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker and all necessary tools pre-installed. Simply click Start Lab to begin - no need to build your own VM or install software.

Your cloud machine includes:

Docker Engine (latest stable version)
Docker Compose
Git
Text editors (nano, vim)
All security scanning tools pre-configured
Task 1: Implementing Docker Content Trust
Docker Content Trust (DCT) provides cryptographic signing of Docker images, ensuring image integrity and publisher authenticity.

Subtask 1.1: Enable Docker Content Trust
First, let's enable Docker Content Trust on our system:

# Enable Docker Content Trust globally
export DOCKER_CONTENT_TRUST=1

# Verify the setting
echo $DOCKER_CONTENT_TRUST
Subtask 1.2: Create a Test Application
Let's create a simple application to demonstrate secure image practices:

# Create a project directory
mkdir secure-docker-lab
cd secure-docker-lab

# Create a simple Node.js application
cat > app.js << 'EOF'
const http = require('http');
const port = 3000;

const server = http.createServer((req, res) => {
  res.writeHead(200, {'Content-Type': 'text/plain'});
  res.end('Secure Docker Application Running!\n');
});

server.listen(port, () => {
  console.log(`Server running at http://localhost:${port}/`);
});
EOF

# Create package.json
cat > package.json << 'EOF'
{
  "name": "secure-docker-app",
  "version": "1.0.0",
  "description": "A secure Docker application example",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {}
}
EOF
Subtask 1.3: Create a Secure Dockerfile
Create a Dockerfile following security best practices:

cat > Dockerfile << 'EOF'
# Use minimal base image with specific version
FROM node:18-alpine3.17

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

# Set working directory
WORKDIR /app

# Copy package files first for better caching
COPY package*.json ./

# Install dependencies (if any)
RUN npm ci --only=production && npm cache clean --force

# Copy application code
COPY --chown=nextjs:nodejs app.js .

# Switch to non-root user
USER nextjs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/ || exit 1

# Start application
CMD ["node", "app.js"]
EOF
Subtask 1.4: Generate Signing Keys
Generate keys for Docker Content Trust:

# Create a directory for keys
mkdir -p ~/.docker/trust/private

# Generate a root key (you'll be prompted for passphrases)
docker trust key generate mykey

# List the generated keys
ls ~/.docker/trust/private/
Subtask 1.5: Build and Sign the Image
Build the image with Content Trust enabled:

# Build the image (this will automatically sign it with DCT enabled)
docker build -t localhost:5000/secure-app:v1.0 .

# Note: Since we're using a local registry, we'll set up a local registry first
docker run -d -p 5000:5000 --name registry registry:2

# Tag and push the signed image
docker tag localhost:5000/secure-app:v1.0 localhost:5000/secure-app:signed
docker push localhost:5000/secure-app:signed
Subtask 1.6: Verify Image Signatures
Verify the signed image:

# Pull and verify the signed image
docker pull localhost:5000/secure-app:signed

# Inspect trust information
docker trust inspect localhost:5000/secure-app:signed

# View signing information
docker trust view localhost:5000/secure-app:signed
Task 2: Implementing Image Scanning with Open-Source Tools
Subtask 2.1: Install and Configure Trivy Scanner
Trivy is an open-source vulnerability scanner for containers:

# Install Trivy (already pre-installed in your cloud machine)
# For reference, installation command would be:
# curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Verify Trivy installation
trivy --version

# Update vulnerability database
trivy image --download-db-only
Subtask 2.2: Scan Images for Vulnerabilities
Perform comprehensive vulnerability scanning:

# Scan our built image
trivy image localhost:5000/secure-app:v1.0

# Scan with specific severity levels
trivy image --severity HIGH,CRITICAL localhost:5000/secure-app:v1.0

# Generate JSON report
trivy image --format json --output scan-report.json localhost:5000/secure-app:v1.0

# Generate HTML report
trivy image --format template --template '@contrib/html.tpl' --output scan-report.html localhost:5000/secure-app:v1.0

# View the JSON report
cat scan-report.json | jq '.Results[0].Vulnerabilities | length'
Subtask 2.3: Scan Base Images
Compare vulnerability levels of different base images:

# Scan different Node.js base images
echo "Scanning node:18 (full image)..."
trivy image --severity HIGH,CRITICAL node:18

echo "Scanning node:18-slim..."
trivy image --severity HIGH,CRITICAL node:18-slim

echo "Scanning node:18-alpine..."
trivy image --severity HIGH,CRITICAL node:18-alpine

# Create a comparison report
cat > compare-base-images.sh << 'EOF'
#!/bin/bash
echo "=== Base Image Vulnerability Comparison ===" > base-image-comparison.txt
echo "" >> base-image-comparison.txt

for image in "node:18" "node:18-slim" "node:18-alpine"; do
    echo "Scanning $image..." >> base-image-comparison.txt
    trivy image --severity HIGH,CRITICAL --format table $image >> base-image-comparison.txt
    echo "" >> base-image-comparison.txt
done
EOF

chmod +x compare-base-images.sh
./compare-base-images.sh
Task 3: Applying Security Best Practices in Dockerfiles
Subtask 3.1: Create a Security-Hardened Dockerfile
Create an improved Dockerfile with advanced security practices:

cat > Dockerfile.secure << 'EOF'
# Use specific version with minimal attack surface
FROM node:18.17.1-alpine3.18

# Install security updates
RUN apk update && apk upgrade && \
    apk add --no-cache dumb-init && \
    rm -rf /var/cache/apk/*

# Create non-root user with specific UID/GID
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

# Set secure working directory
WORKDIR /app

# Copy package files with proper ownership
COPY --chown=appuser:appgroup package*.json ./

# Install dependencies and clean up
RUN npm ci --only=production && \
    npm cache clean --force && \
    rm -rf /tmp/* /var/tmp/*

# Copy application code
COPY --chown=appuser:appgroup app.js .

# Remove unnecessary packages and files
RUN apk del --purge && \
    rm -rf /var/cache/apk/* /tmp/* /var/tmp/*

# Switch to non-root user
USER appuser

# Set security-focused environment variables
ENV NODE_ENV=production
ENV NPM_CONFIG_LOGLEVEL=warn

# Expose port (non-privileged)
EXPOSE 3000

# Add health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/ || exit 1

# Use dumb-init to handle signals properly
ENTRYPOINT ["dumb-init", "--"]

# Start application
CMD ["node", "app.js"]
EOF
Subtask 3.2: Build and Test the Secure Image
# Build the secure image
docker build -f Dockerfile.secure -t secure-app:hardened .

# Run security scan on the hardened image
trivy image --severity HIGH,CRITICAL secure-app:hardened

# Compare with the original image
echo "=== Security Comparison ===" > security-comparison.txt
echo "Original Image Vulnerabilities:" >> security-comparison.txt
trivy image --severity HIGH,CRITICAL --format table localhost:5000/secure-app:v1.0 >> security-comparison.txt
echo "" >> security-comparison.txt
echo "Hardened Image Vulnerabilities:" >> security-comparison.txt
trivy image --severity HIGH,CRITICAL --format table secure-app:hardened >> security-comparison.txt

# View the comparison
cat security-comparison.txt
Subtask 3.3: Implement Multi-Stage Build
Create a multi-stage Dockerfile for even better security:

cat > Dockerfile.multistage << 'EOF'
# Build stage
FROM node:18.17.1-alpine3.18 AS builder

WORKDIR /build

# Copy package files
COPY package*.json ./

# Install all dependencies (including dev dependencies)
RUN npm ci

# Copy source code
COPY app.js .

# Production stage
FROM node:18.17.1-alpine3.18 AS production

# Install security updates and dumb-init
RUN apk update && apk upgrade && \
    apk add --no-cache dumb-init && \
    rm -rf /var/cache/apk/*

# Create non-root user
RUN addgroup -g 1001 -S appgroup && \
    adduser -S appuser -u 1001 -G appgroup

WORKDIR /app

# Copy only production files from builder stage
COPY --from=builder --chown=appuser:appgroup /build/package*.json ./
COPY --from=builder --chown=appuser:appgroup /build/app.js .

# Install only production dependencies
RUN npm ci --only=production && \
    npm cache clean --force && \
    rm -rf /tmp/* /var/tmp/*

# Switch to non-root user
USER appuser

# Security settings
ENV NODE_ENV=production
ENV NPM_CONFIG_LOGLEVEL=warn

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:3000/ || exit 1

ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "app.js"]
EOF

# Build multi-stage image
docker build -f Dockerfile.multistage -t secure-app:multistage .

# Compare image sizes
echo "=== Image Size Comparison ==="
docker images | grep secure-app
Task 4: Implementing Least Privilege Principle
Subtask 4.1: Configure Container Security Context
Create a Docker Compose file with security constraints:

cat > docker-compose.secure.yml << 'EOF'
version: '3.8'

services:
  secure-app:
    build:
      context: .
      dockerfile: Dockerfile.multistage
    ports:
      - "3000:3000"
    user: "1001:1001"
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100m
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    networks:
      - app-network
    restart: unless-stopped
    environment:
      - NODE_ENV=production
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  app-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
EOF
Subtask 4.2: Test Container Security
# Run the secure container
docker-compose -f docker-compose.secure.yml up -d

# Verify the container is running with restricted privileges
docker exec $(docker-compose -f docker-compose.secure.yml ps -q secure-app) id

# Test that the container cannot escalate privileges
docker exec $(docker-compose -f docker-compose.secure.yml ps -q secure-app) whoami

# Verify read-only filesystem
docker exec $(docker-compose -f docker-compose.secure.yml ps -q secure-app) touch /test-file 2>&1 || echo "Read-only filesystem working correctly"

# Check container capabilities
docker exec $(docker-compose -f docker-compose.secure.yml ps -q secure-app) cat /proc/self/status | grep Cap

# Clean up
docker-compose -f docker-compose.secure.yml down
Subtask 4.3: Implement Resource Limits
Create a resource-constrained deployment:

cat > docker-compose.limits.yml << 'EOF'
version: '3.8'

services:
  secure-app:
    build:
      context: .
      dockerfile: Dockerfile.multistage
    ports:
      - "3000:3000"
    user: "1001:1001"
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=50m
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 128M
        reservations:
          cpus: '0.25'
          memory: 64M
    ulimits:
      nproc: 65535
      nofile:
        soft: 20000
        hard: 40000
    networks:
      - app-network
    restart: unless-stopped
    environment:
      - NODE_ENV=production

networks:
  app-network:
    driver: bridge
EOF

# Test resource limits
docker-compose -f docker-compose.limits.yml up -d

# Monitor resource usage
docker stats $(docker-compose -f docker-compose.limits.yml ps -q)
Task 5: Setting Up Automated Vulnerability Scanning in CI/CD Pipeline
Subtask 5.1: Create GitHub Actions Workflow
Create a CI/CD pipeline with automated security scanning:

# Create GitHub Actions directory structure
mkdir -p .github/workflows

cat > .github/workflows/security-scan.yml << 'EOF'
name: Docker Security Scan

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
      
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
      
    - name: Build Docker image
      run: |
        docker build -f Dockerfile.multistage -t security-test:${{ github.sha }} .
        
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'security-test:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
        
    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'
        
    - name: Run Trivy for high/critical vulnerabilities
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'security-test:${{ github.sha }}'
        format: 'table'
        severity: 'HIGH,CRITICAL'
        exit-code: '1'
        
    - name: Generate security report
      if: always()
      run: |
        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          -v $(pwd):/workspace aquasec/trivy:latest \
          image --format json --output /workspace/security-report.json \
          security-test:${{ github.sha }}
          
    - name: Upload security report
      if: always()
      uses: actions/upload-artifact@v3
      with:
        name: security-report
        path: security-report.json
EOF
Subtask 5.2: Create Local CI/CD Simulation
Since we're in a lab environment, let's simulate the CI/CD process locally:

# Create a local CI/CD simulation script
cat > local-cicd.sh << 'EOF'
#!/bin/bash

echo "=== Starting Local CI/CD Security Pipeline ==="

# Step 1: Build the image
echo "Step 1: Building Docker image..."
docker build -f Dockerfile.multistage -t security-test:latest .

# Step 2: Run vulnerability scan
echo "Step 2: Running vulnerability scan..."
trivy image --format json --output security-report.json security-test:latest

# Step 3: Check for high/critical vulnerabilities
echo "Step 3: Checking for high/critical vulnerabilities..."
HIGH_CRITICAL_COUNT=$(trivy image --format json security-test:latest | jq '[.Results[]?.Vulnerabilities[]? | select(.Severity == "HIGH" or .Severity == "CRITICAL")] | length')

echo "Found $HIGH_CRITICAL_COUNT high/critical vulnerabilities"

# Step 4: Generate reports
echo "Step 4: Generating security reports..."
trivy image --format table --output security-table.txt security-test:latest
trivy image --format template --template '@contrib/html.tpl' --output security-report.html security-test:latest

# Step 5: Security gate
if [ "$HIGH_CRITICAL_COUNT" -gt 0 ]; then
    echo "❌ Security gate failed: High/Critical vulnerabilities found"
    echo "Please review and fix vulnerabilities before deployment"
    exit 1
else
    echo "✅ Security gate passed: No high/critical vulnerabilities found"
fi

echo "=== CI/CD Security Pipeline Complete ==="
EOF

chmod +x local-cicd.sh

# Run the local CI/CD pipeline
./local-cicd.sh
Subtask 5.3: Create Security Policy Configuration
Create a security policy file for automated scanning:

cat > .trivyignore << 'EOF'
# Ignore specific CVEs that are false positives or accepted risks
# CVE-2023-12345

# Ignore low severity issues in development
# (Remove this in production)
EOF

cat > trivy-config.yaml << 'EOF'
format: json
severity:
  - HIGH
  - CRITICAL
vulnerability:
  type:
    - os
    - library
ignore-unfixed: true
exit-code: 1
timeout: 10m
EOF

# Test with configuration
trivy image --config trivy-config.yaml security-test:latest
Subtask 5.4: Set Up Continuous Monitoring
Create a monitoring script for ongoing security assessment:

cat > security-monitor.sh << 'EOF'
#!/bin/bash

IMAGES=("security-test:latest" "secure-app:hardened" "secure-app:multistage")
REPORT_DIR="security-reports"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $REPORT_DIR

echo "=== Docker Security Monitoring Report - $DATE ===" > $REPORT_DIR/monitoring-report-$DATE.txt

for image in "${IMAGES[@]}"; do
    echo "Scanning $image..." >> $REPORT_DIR/monitoring-report-$DATE.txt
    
    # Check if image exists
    if docker image inspect $image > /dev/null 2>&1; then
        trivy image --format table $image >> $REPORT_DIR/monitoring-report-$DATE.txt
        echo "" >> $REPORT_DIR/monitoring-report-$DATE.txt
    else
        echo "Image $image not found" >> $REPORT_DIR/monitoring-report-$DATE.txt
    fi
done

echo "Security monitoring complete. Report saved to $REPORT_DIR/monitoring-report-$DATE.txt"
EOF

chmod +x security-monitor.sh

# Run security monitoring
./security-monitor.sh

# View the latest report
ls -la security-reports/
Troubleshooting Common Issues
Issue 1: Docker Content Trust Key Generation Problems
# If key generation fails, reset and try again
rm -rf ~/.docker/trust/
docker trust key generate mykey --dir ~/.docker/trust/private/
Issue 2: Trivy Database Update Issues
# Clear Trivy cache and update database
trivy image --clear-cache
trivy image --download-db-only
Issue 3: Container Permission Issues
# Check container user and permissions
docker exec <container_id> id
docker exec <container_id> ls -la /app

# Fix ownership if needed (in Dockerfile)
# COPY --chown=appuser:appgroup . .
Issue 4: Network Connectivity in Secure Containers
# Test network connectivity
docker exec <container_id> wget --spider http://localhost:3000/

# Check if health check is working
docker inspect <container_id> | jq '.[0].State.Health'
Verification and Testing
Final Security Assessment
Run a comprehensive security assessment:

cat > final-assessment.sh << 'EOF'
#!/bin/bash

echo "=== Final Docker Security Assessment ==="

# 1. Check Docker Content Trust status
echo "1. Docker Content Trust Status:"
echo "DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST"

# 2. Scan all built images
echo "2. Vulnerability Scan Results:"
for image in $(docker images --format "{{.Repository}}:{{.Tag}}" | grep -E "(secure-app|security-test)"); do
    echo "Scanning $image..."
    trivy image --severity HIGH,CRITICAL --format table $image
done

# 3. Check running containers security
echo "3. Running Container Security:"
for container in $(docker ps --format "{{.Names}}"); do
    echo "Container: $container"
    docker exec $container id 2>/dev/null || echo "Cannot access container"
done

# 4. Generate final report
echo "4. Generating final security report..."
trivy image --format json --output final-security-assessment.json security-test:latest

echo "=== Assessment Complete ==="
EOF

chmod +x final-assessment.sh
./final-assessment.sh
Conclusion
In this comprehensive lab, you have successfully implemented multiple layers of Docker security:

Key Accomplishments
Docker Content Trust Implementation: You learned how to enable DCT, generate signing keys, and verify image signatures to ensure image integrity and authenticity.

Vulnerability Scanning: You implemented Trivy scanner to identify and assess security vulnerabilities in Docker images, comparing different base images and generating detailed reports.

Secure Dockerfile Practices: You created security-hardened Dockerfiles using minimal base images, non-root users, multi-stage builds, and proper file permissions.

Least Privilege Implementation: You configured containers with restricted capabilities, read-only filesystems, resource limits, and proper security contexts.

Automated CI/CD Security: You set up automated vulnerability scanning in CI/CD pipelines with security gates and continuous monitoring.

Why This Matters
Docker security is critical in modern containerized environments because:

Attack Surface Reduction: Minimal base images and proper configuration reduce potential vulnerabilities
Supply Chain Security: Content trust ensures you're running verified, unmodified images
Compliance Requirements: Many industries require vulnerability scanning and security controls
Operational Security: Automated scanning catches vulnerabilities before they reach production
Defense in Depth: Multiple security layers provide comprehensive protection
Next Steps
To further enhance your Docker security knowledge:

Explore advanced security tools like Falco for runtime security monitoring
Learn about Kubernetes security contexts and Pod Security Standards
Implement image signing with Notary and Harbor registry
Study container runtime security with tools like gVisor or Kata Containers
Practice with security benchmarks like CIS Docker Benchmark
Best Practices Summary
Always use specific image tags, never latest in production
Regularly update base images and scan for vulnerabilities
Implement least privilege principle for all containers
Use multi-stage builds to minimize final image size
Enable Docker Content Trust for production environments
Automate security scanning in your CI/CD pipelines
Monitor containers continuously for security threats
Keep security tools and databases updated
This lab has provided you with practical, hands-on experience in securing Docker environments using industry-standard open-source tools and best practices that are essential for the Docker Certified Associate (DCA) certification and real-world container security implementations.
