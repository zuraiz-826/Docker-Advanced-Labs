Lab 90: Docker for Security - Using Docker Content Trust
Objectives
By the end of this lab, students will be able to:

Enable Docker Content Trust to secure image distribution
Sign Docker images using docker trust commands
Verify image signatures before pulling and running containers
Implement Docker Content Trust in CI/CD pipeline workflows
Analyze the security benefits of signed images in container environments
Understand the cryptographic foundation of Docker Content Trust
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (images, containers, registries)
Familiarity with command-line interface operations
Knowledge of public key cryptography concepts
Understanding of CI/CD pipeline fundamentals
Experience with Docker Hub or container registries
Lab Environment Setup
Al Nafi Cloud Machines: This lab uses Al Nafi's pre-configured Linux-based cloud machines. Simply click Start Lab to access your environment - no VM setup required!

Your cloud machine includes:

Docker Engine (latest stable version)
Docker Compose
Git
Text editors (nano, vim)
All necessary dependencies pre-installed
Task 1: Enable Docker Content Trust in Docker Settings
Subtask 1.1: Understanding Docker Content Trust
Docker Content Trust (DCT) provides cryptographic signing of Docker images, ensuring image integrity and publisher authenticity. It uses The Update Framework (TUF) and Notary for secure image distribution.

Subtask 1.2: Check Current Docker Configuration
First, let's examine the current Docker configuration and Content Trust status.

# Check Docker version and configuration
docker version

# Check current Content Trust status
echo $DOCKER_CONTENT_TRUST

# View Docker daemon configuration
docker info | grep -i trust
Subtask 1.3: Enable Docker Content Trust Globally
Enable Docker Content Trust for all Docker operations in your session.

# Enable Docker Content Trust for current session
export DOCKER_CONTENT_TRUST=1

# Verify Content Trust is enabled
echo $DOCKER_CONTENT_TRUST

# Make the setting persistent across sessions
echo 'export DOCKER_CONTENT_TRUST=1' >> ~/.bashrc
source ~/.bashrc
Subtask 1.4: Configure Docker Content Trust Environment
Set up additional environment variables for Content Trust operations.

# Set Content Trust server (using Docker's default Notary server)
export DOCKER_CONTENT_TRUST_SERVER=https://notary.docker.io

# Create directory for trust metadata
mkdir -p ~/.docker/trust

# Verify environment variables
env | grep DOCKER_CONTENT_TRUST
Task 2: Sign a Docker Image Using Docker Trust
Subtask 2.1: Create a Sample Application
Create a simple application to containerize and sign.

# Create project directory
mkdir ~/docker-trust-lab
cd ~/docker-trust-lab

# Create a simple Python web application
cat > app.py << 'EOF'
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return f"<h1>Secure Docker App</h1><p>Container ID: {os.uname().nodename}</p>"

@app.route('/health')
def health():
    return {"status": "healthy", "signed": "true"}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF

# Create requirements file
cat > requirements.txt << 'EOF'
Flask==2.3.3
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
Subtask 2.2: Build and Tag the Docker Image
Build the Docker image with proper tagging for signing.

# Build the Docker image
docker build -t secure-app:v1.0 .

# Tag the image for Docker Hub (replace 'yourusername' with your Docker Hub username)
# For this lab, we'll use a generic username
docker tag secure-app:v1.0 labuser/secure-app:v1.0

# List images to verify
docker images | grep secure-app
Subtask 2.3: Initialize Docker Trust Repository
Initialize the trust repository for your image.

# Initialize trust for the repository
# This will prompt for passphrases - use strong passwords and remember them
docker trust key generate labuser

# Initialize the repository
docker trust signer add --key labuser.pub labuser labuser/secure-app

# View trust information
docker trust inspect labuser/secure-app:v1.0 --pretty
Subtask 2.4: Sign and Push the Image
Sign the image and push it to the registry.

# Login to Docker Hub (you'll need a Docker Hub account)
docker login

# Push the signed image (this automatically signs it when Content Trust is enabled)
docker push labuser/secure-app:v1.0

# Verify the image was signed
docker trust inspect labuser/secure-app:v1.0 --pretty
Task 3: Verify Image Signatures Before Pulling Them
Subtask 3.1: Clean Local Images
Remove local images to test signature verification during pull operations.

# Remove local images
docker rmi labuser/secure-app:v1.0 secure-app:v1.0

# Verify images are removed
docker images | grep secure-app
Subtask 3.2: Pull and Verify Signed Images
Pull the signed image with Content Trust enabled.

# Pull the signed image (Content Trust will verify signature automatically)
docker pull labuser/secure-app:v1.0

# Inspect the trust information
docker trust inspect labuser/secure-app:v1.0 --pretty

# View detailed signature information
docker trust inspect labuser/secure-app:v1.0
Subtask 3.3: Test Unsigned Image Rejection
Demonstrate how Content Trust rejects unsigned images.

# Try to pull an unsigned image (this should fail with Content Trust enabled)
docker pull hello-world

# Temporarily disable Content Trust to pull unsigned image
export DOCKER_CONTENT_TRUST=0
docker pull hello-world

# Re-enable Content Trust
export DOCKER_CONTENT_TRUST=1

# Try to run the unsigned image (this should fail)
docker run hello-world
Subtask 3.4: Verify Image Integrity
Create a script to verify image signatures programmatically.

# Create image verification script
cat > verify-image.sh << 'EOF'
#!/bin/bash

IMAGE_NAME=$1

if [ -z "$IMAGE_NAME" ]; then
    echo "Usage: $0 <image-name>"
    exit 1
fi

echo "Verifying image: $IMAGE_NAME"

# Check if Content Trust is enabled
if [ "$DOCKER_CONTENT_TRUST" != "1" ]; then
    echo "Warning: Docker Content Trust is not enabled"
    exit 1
fi

# Inspect trust information
echo "Trust information:"
docker trust inspect "$IMAGE_NAME" --pretty

# Check if image has valid signatures
SIGNATURES=$(docker trust inspect "$IMAGE_NAME" 2>/dev/null | jq -r '.[] | select(.SignedTags != null) | .SignedTags | length')

if [ "$SIGNATURES" -gt 0 ]; then
    echo "✓ Image is properly signed with $SIGNATURES signature(s)"
    exit 0
else
    echo "✗ Image is not signed or signatures are invalid"
    exit 1
fi
EOF

# Make script executable
chmod +x verify-image.sh

# Test the verification script
./verify-image.sh labuser/secure-app:v1.0
Task 4: Use Docker Content Trust in a CI/CD Pipeline
Subtask 4.1: Create CI/CD Pipeline Configuration
Create a GitHub Actions workflow that implements Docker Content Trust.

# Create GitHub Actions directory structure
mkdir -p .github/workflows

# Create CI/CD pipeline with Content Trust
cat > .github/workflows/secure-build.yml << 'EOF'
name: Secure Docker Build and Deploy

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

env:
  DOCKER_CONTENT_TRUST: 1
  REGISTRY: docker.io
  IMAGE_NAME: labuser/secure-app

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Run security scan
      run: |
        # Install security scanning tools
        curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin
        
        # Scan Dockerfile for security issues
        docker run --rm -i hadolint/hadolint < Dockerfile

  build-and-sign:
    needs: security-scan
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Login to Docker Hub
      uses: docker/login-action@v2
      with:
        username: ${{ secrets.DOCKER_USERNAME }}
        password: ${{ secrets.DOCKER_PASSWORD }}
    
    - name: Enable Docker Content Trust
      run: |
        export DOCKER_CONTENT_TRUST=1
        echo "DOCKER_CONTENT_TRUST=1" >> $GITHUB_ENV
    
    - name: Import signing keys
      run: |
        echo "${{ secrets.DOCKER_TRUST_PRIVATE_KEY }}" | base64 -d > private_key.pem
        docker trust key load private_key.pem
        rm private_key.pem
    
    - name: Build and push signed image
      run: |
        docker build -t ${{ env.IMAGE_NAME }}:${{ github.sha }} .
        docker push ${{ env.IMAGE_NAME }}:${{ github.sha }}
    
    - name: Verify image signature
      run: |
        docker trust inspect ${{ env.IMAGE_NAME }}:${{ github.sha }} --pretty

  deploy:
    needs: build-and-sign
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - name: Deploy to production
      run: |
        export DOCKER_CONTENT_TRUST=1
        # Deployment commands would go here
        echo "Deploying signed image: ${{ env.IMAGE_NAME }}:${{ github.sha }}"
EOF
Subtask 4.2: Create Docker Compose with Content Trust
Create a Docker Compose configuration that enforces signed images.

# Create Docker Compose file with Content Trust
cat > docker-compose.secure.yml << 'EOF'
version: '3.8'

services:
  secure-app:
    image: labuser/secure-app:v1.0
    ports:
      - "5000:5000"
    environment:
      - DOCKER_CONTENT_TRUST=1
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped
    
  nginx:
    image: nginx:1.21-alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - secure-app
    restart: unless-stopped

networks:
  default:
    driver: bridge
EOF

# Create nginx configuration
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream app {
        server secure-app:5000;
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
Subtask 4.3: Create Deployment Script with Trust Verification
Create an automated deployment script that verifies signatures.

# Create secure deployment script
cat > deploy-secure.sh << 'EOF'
#!/bin/bash

set -e

# Configuration
IMAGE_NAME="labuser/secure-app:v1.0"
COMPOSE_FILE="docker-compose.secure.yml"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

echo -e "${YELLOW}Starting secure deployment process...${NC}"

# Enable Content Trust
export DOCKER_CONTENT_TRUST=1

# Function to verify image signature
verify_image() {
    local image=$1
    echo -e "${YELLOW}Verifying signature for: $image${NC}"
    
    if docker trust inspect "$image" --pretty > /dev/null 2>&1; then
        echo -e "${GREEN}✓ Image signature verified successfully${NC}"
        return 0
    else
        echo -e "${RED}✗ Image signature verification failed${NC}"
        return 1
    fi
}

# Function to deploy application
deploy_app() {
    echo -e "${YELLOW}Deploying application...${NC}"
    
    # Stop existing containers
    docker-compose -f "$COMPOSE_FILE" down
    
    # Pull latest signed images
    docker-compose -f "$COMPOSE_FILE" pull
    
    # Start services
    docker-compose -f "$COMPOSE_FILE" up -d
    
    # Wait for health check
    echo -e "${YELLOW}Waiting for application to be healthy...${NC}"
    sleep 10
    
    # Check application health
    if curl -f http://localhost:5000/health > /dev/null 2>&1; then
        echo -e "${GREEN}✓ Application deployed successfully and is healthy${NC}"
        return 0
    else
        echo -e "${RED}✗ Application deployment failed or unhealthy${NC}"
        return 1
    fi
}

# Main deployment process
main() {
    # Verify image signature
    if verify_image "$IMAGE_NAME"; then
        # Deploy application
        if deploy_app; then
            echo -e "${GREEN}Secure deployment completed successfully!${NC}"
            exit 0
        else
            echo -e "${RED}Deployment failed!${NC}"
            exit 1
        fi
    else
        echo -e "${RED}Image verification failed. Deployment aborted for security reasons.${NC}"
        exit 1
    fi
}

# Run main function
main
EOF

# Make deployment script executable
chmod +x deploy-secure.sh

# Test the deployment script
./deploy-secure.sh
Task 5: Analyze the Impact of Signed Images on Container Security
Subtask 5.1: Create Security Analysis Script
Develop a comprehensive script to analyze the security benefits of signed images.

# Create security analysis script
cat > security-analysis.sh << 'EOF'
#!/bin/bash

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

echo -e "${BLUE}=== Docker Content Trust Security Analysis ===${NC}"

# Function to analyze image trust status
analyze_image_trust() {
    local image=$1
    echo -e "\n${YELLOW}Analyzing: $image${NC}"
    
    # Check if image exists locally
    if docker images --format "table {{.Repository}}:{{.Tag}}" | grep -q "$image"; then
        echo -e "${GREEN}✓ Image exists locally${NC}"
    else
        echo -e "${RED}✗ Image not found locally${NC}"
        return 1
    fi
    
    # Check trust information
    if docker trust inspect "$image" --pretty > /dev/null 2>&1; then
        echo -e "${GREEN}✓ Image has trust information${NC}"
        
        # Get signature details
        SIGNERS=$(docker trust inspect "$image" --pretty | grep "Signer:" | wc -l)
        echo -e "${BLUE}  - Number of signers: $SIGNERS${NC}"
        
        # Get signature timestamp
        TIMESTAMP=$(docker trust inspect "$image" | jq -r '.[0].SignedTags[0].SignedTag.Signed.Expires' 2>/dev/null)
        if [ "$TIMESTAMP" != "null" ] && [ -n "$TIMESTAMP" ]; then
            echo -e "${BLUE}  - Signature expires: $TIMESTAMP${NC}"
        fi
        
        return 0
    else
        echo -e "${RED}✗ No trust information available${NC}"
        return 1
    fi
}

# Function to demonstrate security benefits
demonstrate_security_benefits() {
    echo -e "\n${BLUE}=== Security Benefits Analysis ===${NC}"
    
    echo -e "\n${YELLOW}1. Image Integrity Protection:${NC}"
    echo -e "   - Cryptographic signatures ensure image hasn't been tampered with"
    echo -e "   - SHA256 hashes verify content integrity"
    echo -e "   - Prevents man-in-the-middle attacks during image distribution"
    
    echo -e "\n${YELLOW}2. Publisher Authentication:${NC}"
    echo -e "   - Digital signatures verify image publisher identity"
    echo -e "   - Prevents impersonation attacks"
    echo -e "   - Establishes chain of trust from publisher to consumer"
    
    echo -e "\n${YELLOW}3. Supply Chain Security:${NC}"
    echo -e "   - Ensures images come from trusted sources"
    echo -e "   - Prevents injection of malicious images"
    echo -e "   - Enables audit trail of image provenance"
    
    echo -e "\n${YELLOW}4. Compliance and Governance:${NC}"
    echo -e "   - Enforces organizational security policies"
    echo -e "   - Provides non-repudiation of image publishing"
    echo -e "   - Supports regulatory compliance requirements"
}

# Function to show trust vs non-trust comparison
compare_trust_scenarios() {
    echo -e "\n${BLUE}=== Trust vs Non-Trust Comparison ===${NC}"
    
    echo -e "\n${YELLOW}With Docker Content Trust ENABLED:${NC}"
    echo -e "${GREEN}✓ Only signed images can be pulled and run${NC}"
    echo -e "${GREEN}✓ Image integrity is cryptographically verified${NC}"
    echo -e "${GREEN}✓ Publisher identity is authenticated${NC}"
    echo -e "${GREEN}✓ Tampered images are automatically rejected${NC}"
    
    echo -e "\n${YELLOW}With Docker Content Trust DISABLED:${NC}"
    echo -e "${RED}✗ Any image can be pulled and run${NC}"
    echo -e "${RED}✗ No integrity verification${NC}"
    echo -e "${RED}✗ No publisher authentication${NC}"
    echo -e "${RED}✗ Vulnerable to supply chain attacks${NC}"
}

# Function to calculate security metrics
calculate_security_metrics() {
    echo -e "\n${BLUE}=== Security Metrics ===${NC}"
    
    # Count total images
    TOTAL_IMAGES=$(docker images --format "{{.Repository}}:{{.Tag}}" | wc -l)
    
    # Count signed images (this is a simplified check)
    SIGNED_IMAGES=0
    for image in $(docker images --format "{{.Repository}}:{{.Tag}}"); do
        if docker trust inspect "$image" > /dev/null 2>&1; then
            ((SIGNED_IMAGES++))
        fi
    done
    
    # Calculate percentage
    if [ $TOTAL_IMAGES -gt 0 ]; then
        SIGNED_PERCENTAGE=$((SIGNED_IMAGES * 100 / TOTAL_IMAGES))
    else
        SIGNED_PERCENTAGE=0
    fi
    
    echo -e "${BLUE}Total images: $TOTAL_IMAGES${NC}"
    echo -e "${BLUE}Signed images: $SIGNED_IMAGES${NC}"
    echo -e "${BLUE}Signed percentage: $SIGNED_PERCENTAGE%${NC}"
    
    if [ $SIGNED_PERCENTAGE -ge 80 ]; then
        echo -e "${GREEN}✓ Good security posture (≥80% signed)${NC}"
    elif [ $SIGNED_PERCENTAGE -ge 50 ]; then
        echo -e "${YELLOW}⚠ Moderate security posture (50-79% signed)${NC}"
    else
        echo -e "${RED}✗ Poor security posture (<50% signed)${NC}"
    fi
}

# Main analysis function
main() {
    # Analyze specific images
    analyze_image_trust "labuser/secure-app:v1.0"
    
    # Show security benefits
    demonstrate_security_benefits
    
    # Compare scenarios
    compare_trust_scenarios
    
    # Calculate metrics
    calculate_security_metrics
    
    echo -e "\n${GREEN}Security analysis completed!${NC}"
}

# Run main function
main
EOF

# Make script executable
chmod +x security-analysis.sh

# Run security analysis
./security-analysis.sh
Subtask 5.2: Create Security Monitoring Dashboard
Create a simple monitoring script to track Content Trust usage.

# Create monitoring script
cat > trust-monitor.sh << 'EOF'
#!/bin/bash

# Configuration
LOG_FILE="/tmp/docker-trust-monitor.log"
REPORT_FILE="/tmp/trust-report.html"

# Function to log events
log_event() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Function to monitor Docker operations
monitor_docker_operations() {
    echo "Starting Docker Content Trust monitoring..."
    log_event "Monitoring started"
    
    # Monitor for 60 seconds (adjust as needed)
    timeout 60s docker events --filter type=image --format "{{.Time}} {{.Action}} {{.Actor.Attributes.name}}" | while read line; do
        log_event "Docker event: $line"
    done
}

# Function to generate HTML report
generate_report() {
    cat > "$REPORT_FILE" << 'HTML_EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Docker Content Trust Security Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #2196F3; color: white; padding: 20px; }
        .section { margin: 20px 0; padding: 15px; border: 1px solid #ddd; }
        .success { background-color: #d4edda; border-color: #c3e6cb; }
        .warning { background-color: #fff3cd; border-color: #ffeaa7; }
        .danger { background-color: #f8d7da; border-color: #f5c6cb; }
        table { width: 100%; border-collapse: collapse; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Docker Content Trust Security Report</h1>
        <p>Generated on: $(date)</p>
    </div>
    
    <div class="section success">
        <h2>Content Trust Status</h2>
        <p><strong>Status:</strong> $([ "$DOCKER_CONTENT_TRUST" = "1" ] && echo "ENABLED" || echo "DISABLED")</p>
        <p><strong>Server:</strong> ${DOCKER_CONTENT_TRUST_SERVER:-"Default"}</p>
    </div>
    
    <div class="section">
        <h2>Image Security Summary</h2>
        <table>
            <tr>
                <th>Metric</th>
                <th>Value</th>
                <th>Status</th>
            </tr>
            <tr>
                <td>Total Images</td>
                <td>$(docker images | wc -l)</td>
                <td>-</td>
            </tr>
            <tr>
                <td>Content Trust Enabled</td>
                <td>$([ "$DOCKER_CONTENT_TRUST" = "1" ] && echo "Yes" || echo "No")</td>
                <td>$([ "$DOCKER_CONTENT_TRUST" = "1" ] && echo "✓" || echo "✗")</td>
            </tr>
        </table>
    </div>
    
    <div class="section">
        <h2>Security Recommendations</h2>
        <ul>
            <li>Always enable Docker Content Trust in production environments</li>
            <li>Regularly rotate signing keys</li>
            <li>Implement automated signature verification in CI/CD pipelines</li>
            <li>Monitor and audit image signatures</li>
            <li>Train development teams on secure image practices</li>
        </ul>
    </div>
    
    <div class="section">
        <h2>Recent Activity Log</h2>
        <pre>$(tail -20 "$LOG_FILE" 2>/dev/null || echo "No recent activity")</pre>
    </div>
</body>
</html>
HTML_EOF

    echo "Report generated: $REPORT_FILE"
}

# Main monitoring function
main() {
    # Initialize log
    log_event "Trust monitoring initialized"
    
    # Monitor operations
    monitor_docker_operations &
    MONITOR_PID=$!
    
    # Generate report
    generate_report
    
    # Clean up
    kill $MONITOR_PID 2>/dev/null
    log_event "Monitoring completed"
    
    echo "Monitoring completed. Check $REPORT_FILE for detailed report."
}

# Run monitoring
main
EOF

# Make script executable
chmod +x trust-monitor.sh

# Run monitoring (this will run for 60 seconds)
./trust-monitor.sh
Subtask 5.3: Performance Impact Analysis
Analyze the performance impact of enabling Docker Content Trust.

# Create performance analysis script
cat > performance-analysis.sh << 'EOF'
#!/bin/bash

# Test image for performance comparison
TEST_IMAGE="alpine:latest"

echo "=== Docker Content Trust Performance Analysis ==="

# Function to measure operation time
measure_time() {
    local operation=$1
    local command=$2
    
    echo "Measuring: $operation"
    
    # Run command and measure time
    start_time=$(date +%s.%N)
    eval "$command" > /dev/null 2>&1
    end_time=$(date +%s.%N)
    
    # Calculate duration
    duration=$(echo "$end_time - $start_time" | bc -l)
    printf "  Duration: %.3f seconds\n" "$duration"
    
    return 0
}

# Test without Content Trust
echo -e "\n--- Testing WITHOUT Content Trust ---"
export DOCKER_CONTENT_TRUST=0

# Clean up
docker rmi "$TEST_IMAGE" 2>/dev/null

# Measure pull time
measure_time "Image pull (no trust)" "docker pull $TEST_IMAGE"

# Clean up
docker rmi "$TEST_IMAGE" 2>/dev/null

# Test with Content Trust
echo -e "\n--- Testing WITH Content Trust ---"
export DOCKER_CONTENT_TRUST=1

# Measure pull time (this might fail for unsigned images)
measure_time "Image pull (with trust)" "docker pull $TEST_IMAGE" || echo "  Note: Pull failed due to unsigned image (expected behavior)"

# Test with signed image
echo -e "\n--- Testing with Signed Image ---"
measure_time "Signed image pull" "docker pull labuser/secure-app:v1.0"

# Summary
echo -e "\n=== Performance Impact Summary ==="
echo "1. Content Trust adds cryptographic verification overhead"
echo "2. Initial setup requires key generation and repository initialization"
echo "3. Image pulls include signature verification step"
echo "4. Network overhead for downloading trust metadata"
echo "5. Overall impact is minimal for most use cases"
echo "6. Security benefits outweigh performance costs"

# Reset Content Trust
export DOCKER_CONTENT_TRUST=1
EOF

# Make script executable
chmod +x performance-analysis.sh

# Run performance analysis
./performance-analysis.sh
Troubleshooting Common Issues
Issue 1: Trust Key Generation Fails
# If key generation fails, check permissions
ls -la ~/.docker/trust/

# Reset trust directory if needed
rm -rf ~/.docker/trust/
mkdir -p ~/.docker/trust/
chmod 700 ~/.docker/trust/
Issue 2: Image Push Fails with Trust Enabled
# Check if you're logged into Docker Hub
docker login

# Verify Content Trust environment
env | grep DOCKER_CONTENT_TRUST

# Check repository initialization
docker trust inspect --pretty <your-image>
Issue 3: Signature Verification Fails
# Check trust server connectivity
curl -I https://notary.docker.io

# Verify image exists and is signed
docker trust inspect <image-name> --pretty

# Check local trust metadata
ls -la ~/.docker/trust/
Conclusion
In this comprehensive lab, you have successfully:

Implemented Docker Content Trust by enabling cryptographic signing and verification of Docker images, providing a robust security layer for container distribution.

Mastered Image Signing by learning to sign Docker images using docker trust commands, establishing publisher authenticity and image integrity.

Verified Image Signatures by implementing automated signature verification processes that reject unsigned or tampered images.

Integrated Security into CI/CD by creating pipeline configurations that enforce Content Trust throughout the development and deployment lifecycle.

Analyzed Security Impact by understanding how signed images protect against supply chain attacks, ensure compliance, and maintain organizational security policies.

Key Security Benefits Achieved:

Image Integrity: Cryptographic verification prevents tampering
Publisher Authentication: Digital signatures verify image sources
Supply Chain Security: Protection against malicious image injection
Compliance Support: Audit trails and governance enforcement
Real-World Applications:

Enterprise container deployments requiring security compliance
Multi-team environments needing publisher verification
Production systems with strict security requirements
Regulated industries requiring image provenance tracking
Docker Content Trust provides essential security capabilities for modern container environments, ensuring that only trusted, verified images are deployed in production systems. The skills learned in this lab are crucial for maintaining secure container infrastructures and meeting enterprise security standards.
