Lab 34: Docker and Security - Using Docker Security Scanners
Lab Objectives
By the end of this lab, you will be able to:

• Install and configure Docker security scanning tools including Docker Scout and Trivy • Pull Docker images from Docker Hub and perform comprehensive security scans • Analyze and interpret vulnerability scan results to identify security risks • Implement vulnerability mitigation strategies by upgrading to more secure base images • Automate Docker image security scanning in CI/CD pipelines using GitHub Actions • Apply Docker security best practices to reduce attack surface

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker concepts (containers, images, Dockerfile) • Familiarity with Linux command line operations • Basic knowledge of software vulnerabilities and security concepts • Understanding of CI/CD pipeline concepts • GitHub account (for automation section)

Note: Al Nafi provides pre-configured Linux-based cloud machines with Docker already installed. Simply click Start Lab to begin - no need to build your own VM or install Docker manually.

Lab Environment Setup
Your Al Nafi cloud machine comes with: • Ubuntu 22.04 LTS • Docker Engine (latest stable version) • Git and curl pre-installed • Internet connectivity for pulling images and tools

Task 1: Install and Configure Docker Security Scanning Tools
Subtask 1.1: Verify Docker Installation
First, let's verify that Docker is properly installed and running on your system.

# Check Docker version
docker --version

# Check Docker service status
sudo systemctl status docker

# Test Docker with hello-world
docker run hello-world
Subtask 1.2: Install Docker Scout
Docker Scout is Docker's official vulnerability scanning tool that provides detailed security insights.

# Docker Scout is built into Docker Desktop and Docker CLI (v4.17+)
# Enable Docker Scout if not already enabled
docker scout version

# If Docker Scout is not available, update Docker CLI
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
Subtask 1.3: Install Trivy Security Scanner
Trivy is a comprehensive open-source vulnerability scanner that works excellently with Docker images.

# Install Trivy using the installation script
sudo apt-get update
sudo apt-get install wget apt-transport-https gnupg lsb-release -y

# Add Trivy repository
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list

# Update package list and install Trivy
sudo apt-get update
sudo apt-get install trivy -y

# Verify Trivy installation
trivy version
Subtask 1.4: Install Grype (Alternative Scanner)
Grype is another excellent open-source vulnerability scanner from Anchore.

# Install Grype using curl
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin

# Verify Grype installation
grype version
Task 2: Pull Docker Images and Run Security Scans
Subtask 2.1: Pull Test Images
Let's pull several images with different security profiles for testing.

# Pull a recent Ubuntu image
docker pull ubuntu:22.04

# Pull an older Ubuntu image (likely to have more vulnerabilities)
docker pull ubuntu:18.04

# Pull a Node.js image
docker pull node:16

# Pull an Alpine-based image (typically more secure)
docker pull node:16-alpine

# Pull a deliberately vulnerable image for testing
docker pull vulnerables/web-dvwa

# List all pulled images
docker images
Subtask 2.2: Scan Images with Docker Scout
# Scan Ubuntu 22.04 image
docker scout cves ubuntu:22.04

# Scan with detailed output
docker scout cves --format sarif --output ubuntu-22.04-scan.sarif ubuntu:22.04

# Compare two images
docker scout compare ubuntu:18.04 --to ubuntu:22.04

# Get recommendations for the image
docker scout recommendations ubuntu:18.04
Subtask 2.3: Scan Images with Trivy
# Basic vulnerability scan
trivy image ubuntu:22.04

# Scan with specific severity levels
trivy image --severity HIGH,CRITICAL ubuntu:18.04

# Generate JSON report
trivy image --format json --output ubuntu-18.04-trivy.json ubuntu:18.04

# Scan for secrets and misconfigurations
trivy image --scanners vuln,secret,config node:16

# Scan the vulnerable DVWA image
trivy image --severity HIGH,CRITICAL vulnerables/web-dvwa
Subtask 2.4: Scan Images with Grype
# Basic scan with Grype
grype ubuntu:18.04

# Scan with specific output format
grype -o json ubuntu:18.04 > ubuntu-18.04-grype.json

# Scan with severity filtering
grype --fail-on high ubuntu:18.04

# Compare Alpine vs regular Node image
grype node:16
grype node:16-alpine
Task 3: Inspect and Analyze Scan Results
Subtask 3.1: Understanding Vulnerability Reports
Create a script to analyze the scan results systematically:

# Create analysis script
cat > analyze_scans.sh << 'EOF'
#!/bin/bash

echo "=== Docker Security Scan Analysis ==="
echo "Date: $(date)"
echo

# Function to analyze Trivy JSON output
analyze_trivy_json() {
    local file=$1
    echo "Analyzing Trivy scan results from: $file"
    
    if [ -f "$file" ]; then
        echo "Total vulnerabilities found:"
        jq '.Results[].Vulnerabilities | length' "$file" 2>/dev/null || echo "No vulnerabilities section found"
        
        echo "Vulnerabilities by severity:"
        jq -r '.Results[].Vulnerabilities[]?.Severity' "$file" 2>/dev/null | sort | uniq -c || echo "Unable to parse severity data"
        
        echo "Top 5 critical vulnerabilities:"
        jq -r '.Results[].Vulnerabilities[] | select(.Severity=="CRITICAL") | .VulnerabilityID + ": " + .Title' "$file" 2>/dev/null | head -5 || echo "No critical vulnerabilities found"
    else
        echo "File $file not found"
    fi
    echo
}

# Analyze the Ubuntu 18.04 scan
analyze_trivy_json "ubuntu-18.04-trivy.json"

EOF

chmod +x analyze_scans.sh
./analyze_scans.sh
Subtask 3.2: Create Vulnerability Summary Report
# Create a comprehensive report script
cat > vulnerability_report.sh << 'EOF'
#!/bin/bash

echo "=== Comprehensive Vulnerability Report ==="
echo "Generated on: $(date)"
echo "=========================================="
echo

# Function to scan and report
scan_and_report() {
    local image=$1
    echo "Scanning image: $image"
    echo "----------------------------------------"
    
    # Get image info
    echo "Image details:"
    docker inspect "$image" --format='Size: {{.Size}} bytes' 2>/dev/null
    docker inspect "$image" --format='Created: {{.Created}}' 2>/dev/null
    echo
    
    # Quick Trivy scan
    echo "Vulnerability summary (Trivy):"
    trivy image --quiet --format table "$image" | head -20
    echo
    
    # Count vulnerabilities by severity
    echo "Vulnerability count by severity:"
    trivy image --format json "$image" 2>/dev/null | jq -r '.Results[]?.Vulnerabilities[]?.Severity' | sort | uniq -c
    echo
    echo "=========================================="
    echo
}

# Scan multiple images
scan_and_report "ubuntu:18.04"
scan_and_report "ubuntu:22.04"
scan_and_report "node:16"
scan_and_report "node:16-alpine"

EOF

chmod +x vulnerability_report.sh
./vulnerability_report.sh > vulnerability_report.txt

# Display the report
cat vulnerability_report.txt
Subtask 3.3: Identify Critical Security Issues
# Create a script to identify critical issues
cat > critical_issues.sh << 'EOF'
#!/bin/bash

echo "=== Critical Security Issues Analysis ==="
echo

# Function to find critical issues
find_critical_issues() {
    local image=$1
    echo "Critical issues in $image:"
    echo "-------------------------"
    
    # Find CRITICAL severity vulnerabilities
    trivy image --severity CRITICAL --format json "$image" 2>/dev/null | \
    jq -r '.Results[]?.Vulnerabilities[]? | select(.Severity=="CRITICAL") | 
    "CVE: " + .VulnerabilityID + " | Package: " + .PkgName + " | Fixed Version: " + (.FixedVersion // "Not available")'
    
    echo
}

# Check critical issues in different images
find_critical_issues "ubuntu:18.04"
find_critical_issues "vulnerables/web-dvwa"

EOF

chmod +x critical_issues.sh
./critical_issues.sh
Task 4: Mitigate Vulnerabilities by Upgrading Base Images
Subtask 4.1: Compare Base Image Versions
# Create comparison script
cat > compare_images.sh << 'EOF'
#!/bin/bash

echo "=== Base Image Security Comparison ==="
echo

compare_vulnerability_count() {
    local image1=$1
    local image2=$2
    
    echo "Comparing $image1 vs $image2"
    echo "----------------------------------------"
    
    # Get vulnerability counts
    count1=$(trivy image --format json "$image1" 2>/dev/null | jq '.Results[]?.Vulnerabilities | length' | head -1)
    count2=$(trivy image --format json "$image2" 2>/dev/null | jq '.Results[]?.Vulnerabilities | length' | head -1)
    
    echo "$image1: $count1 vulnerabilities"
    echo "$image2: $count2 vulnerabilities"
    
    if [ "$count1" -gt "$count2" ]; then
        echo "✅ $image2 is more secure (fewer vulnerabilities)"
    elif [ "$count2" -gt "$count1" ]; then
        echo "✅ $image1 is more secure (fewer vulnerabilities)"
    else
        echo "Both images have similar vulnerability counts"
    fi
    echo
}

# Compare different base images
compare_vulnerability_count "ubuntu:18.04" "ubuntu:22.04"
compare_vulnerability_count "node:16" "node:16-alpine"

EOF

chmod +x compare_images.sh
./compare_images.sh
Subtask 4.2: Create Secure Dockerfile Examples
# Create a directory for secure Dockerfile examples
mkdir -p secure-dockerfiles
cd secure-dockerfiles

# Create an insecure Dockerfile
cat > Dockerfile.insecure << 'EOF'
# Insecure Dockerfile example
FROM ubuntu:18.04

# Running as root (insecure)
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    nodejs \
    npm

# No cleanup (larger attack surface)
# No specific user (running as root)

COPY app.js /app/
WORKDIR /app
CMD ["node", "app.js"]
EOF

# Create a secure Dockerfile
cat > Dockerfile.secure << 'EOF'
# Secure Dockerfile example
FROM ubuntu:22.04

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Install only necessary packages and clean up
RUN apt-get update && apt-get install -y --no-install-recommends \
    nodejs \
    npm \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy application files
COPY --chown=appuser:appuser app.js /app/

# Switch to non-root user
USER appuser

# Use specific command
CMD ["node", "app.js"]
EOF

# Create a sample application file
cat > app.js << 'EOF'
const http = require('http');

const server = http.createServer((req, res) => {
    res.writeHead(200, {'Content-Type': 'text/plain'});
    res.end('Hello from secure container!\n');
});

server.listen(3000, () => {
    console.log('Server running on port 3000');
});
EOF

# Build both images
docker build -f Dockerfile.insecure -t myapp:insecure .
docker build -f Dockerfile.secure -t myapp:secure .

cd ..
Subtask 4.3: Scan and Compare Custom Images
# Scan the custom images we just built
echo "=== Scanning Custom Images ==="
echo

echo "Scanning insecure image:"
trivy image --severity HIGH,CRITICAL myapp:insecure

echo
echo "Scanning secure image:"
trivy image --severity HIGH,CRITICAL myapp:secure

# Generate comparison report
echo
echo "=== Security Comparison Report ==="
trivy image --format json myapp:insecure > insecure-scan.json
trivy image --format json myapp:secure > secure-scan.json

# Count vulnerabilities
insecure_count=$(jq '.Results[]?.Vulnerabilities | length' insecure-scan.json 2>/dev/null | head -1)
secure_count=$(jq '.Results[]?.Vulnerabilities | length' secure-scan.json 2>/dev/null | head -1)

echo "Insecure image vulnerabilities: $insecure_count"
echo "Secure image vulnerabilities: $secure_count"

if [ "$insecure_count" -gt "$secure_count" ]; then
    echo "✅ Security improvement achieved!"
    echo "Vulnerability reduction: $((insecure_count - secure_count)) fewer vulnerabilities"
fi
Subtask 4.4: Create Multi-stage Build for Enhanced Security
# Create advanced secure Dockerfile with multi-stage build
cd secure-dockerfiles

cat > Dockerfile.multistage << 'EOF'
# Multi-stage build for enhanced security
# Stage 1: Build stage
FROM node:16 AS builder

WORKDIR /build
COPY package*.json ./
RUN npm ci --only=production

# Stage 2: Runtime stage with minimal base image
FROM node:16-alpine

# Install security updates
RUN apk update && apk upgrade && apk add --no-cache dumb-init

# Create non-root user
RUN addgroup -g 1001 -S nodejs && adduser -S nodejs -u 1001

# Set working directory
WORKDIR /app

# Copy only necessary files from builder stage
COPY --from=builder --chown=nodejs:nodejs /build/node_modules ./node_modules
COPY --chown=nodejs:nodejs app.js ./

# Switch to non-root user
USER nodejs

# Use dumb-init for proper signal handling
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "app.js"]
EOF

# Create package.json for the multi-stage build
cat > package.json << 'EOF'
{
  "name": "secure-app",
  "version": "1.0.0",
  "description": "Secure Node.js application",
  "main": "app.js",
  "dependencies": {},
  "scripts": {
    "start": "node app.js"
  }
}
EOF

# Build the multi-stage image
docker build -f Dockerfile.multistage -t myapp:multistage .

# Scan the multi-stage image
echo "Scanning multi-stage image:"
trivy image --severity HIGH,CRITICAL myapp:multistage

cd ..
Task 5: Automate Docker Image Scanning in CI/CD Pipeline
Subtask 5.1: Create GitHub Actions Workflow
# Create GitHub Actions workflow directory
mkdir -p .github/workflows

# Create security scanning workflow
cat > .github/workflows/docker-security-scan.yml << 'EOF'
name: Docker Security Scan

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    # Run daily at 2 AM UTC
    - cron: '0 2 * * *'

jobs:
  security-scan:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3
    
    - name: Build Docker image
      run: |
        docker build -t test-image:${{ github.sha }} .
    
    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'test-image:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
    
    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      if: always()
      with:
        sarif_file: 'trivy-results.sarif'
    
    - name: Run Trivy scanner for critical vulnerabilities
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'test-image:${{ github.sha }}'
        format: 'table'
        severity: 'CRITICAL,HIGH'
        exit-code: '1'
    
    - name: Docker Scout scan
      if: always()
      run: |
        # Install Docker Scout
        curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --
        
        # Run Docker Scout scan
        docker scout cves test-image:${{ github.sha }}
    
    - name: Generate security report
      if: always()
      run: |
        echo "# Security Scan Report" > security-report.md
        echo "Generated on: $(date)" >> security-report.md
        echo "" >> security-report.md
        
        echo "## Trivy Scan Results" >> security-report.md
        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy:latest image --format table test-image:${{ github.sha }} >> security-report.md
    
    - name: Upload security report
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: security-report
        path: security-report.md
EOF
Subtask 5.2: Create Local CI/CD Simulation Script
# Create a local CI/CD simulation script
cat > ci-cd-security-check.sh << 'EOF'
#!/bin/bash

echo "=== CI/CD Security Pipeline Simulation ==="
echo "Starting security checks at: $(date)"
echo

# Configuration
IMAGE_NAME="myapp"
IMAGE_TAG="ci-test"
FULL_IMAGE_NAME="${IMAGE_NAME}:${IMAGE_TAG}"

# Step 1: Build the image
echo "Step 1: Building Docker image..."
if docker build -t "$FULL_IMAGE_NAME" ./secure-dockerfiles -f ./secure-dockerfiles/Dockerfile.secure; then
    echo "✅ Image built successfully"
else
    echo "❌ Image build failed"
    exit 1
fi
echo

# Step 2: Run security scans
echo "Step 2: Running security scans..."

# Trivy scan with exit code
echo "Running Trivy scan..."
if trivy image --severity HIGH,CRITICAL --exit-code 1 "$FULL_IMAGE_NAME"; then
    echo "✅ Trivy scan passed (no high/critical vulnerabilities)"
    TRIVY_RESULT="PASS"
else
    echo "⚠️  Trivy scan found high/critical vulnerabilities"
    TRIVY_RESULT="FAIL"
fi
echo

# Grype scan
echo "Running Grype scan..."
if grype --fail-on high "$FULL_IMAGE_NAME"; then
    echo "✅ Grype scan passed"
    GRYPE_RESULT="PASS"
else
    echo "⚠️  Grype scan found issues"
    GRYPE_RESULT="FAIL"
fi
echo

# Step 3: Generate reports
echo "Step 3: Generating security reports..."

# Create reports directory
mkdir -p ci-reports

# Generate JSON reports
trivy image --format json --output "ci-reports/trivy-report.json" "$FULL_IMAGE_NAME"
grype -o json "$FULL_IMAGE_NAME" > "ci-reports/grype-report.json"

# Generate summary report
cat > ci-reports/security-summary.txt << EOL
Security Scan Summary
=====================
Image: $FULL_IMAGE_NAME
Scan Date: $(date)

Results:
- Trivy Scan: $TRIVY_RESULT
- Grype Scan: $GRYPE_RESULT

Detailed reports available in:
- trivy-report.json
- grype-report.json
EOL

echo "✅ Reports generated in ci-reports/ directory"
echo

# Step 4: Decision logic
echo "Step 4: Pipeline decision..."
if [ "$TRIVY_RESULT" = "PASS" ] && [ "$GRYPE_RESULT" = "PASS" ]; then
    echo "✅ All security checks passed - Image approved for deployment"
    exit 0
else
    echo "❌ Security checks failed - Image rejected for deployment"
    echo "Please fix vulnerabilities before proceeding"
    exit 1
fi

EOF

chmod +x ci-cd-security-check.sh

# Run the CI/CD simulation
./ci-cd-security-check.sh
Subtask 5.3: Create Automated Remediation Script
# Create automated remediation suggestions script
cat > auto-remediation.sh << 'EOF'
#!/bin/bash

echo "=== Automated Vulnerability Remediation ==="
echo

# Function to suggest remediation for common vulnerabilities
suggest_remediation() {
    local image=$1
    echo "Analyzing $image for remediation opportunities..."
    echo "================================================"
    
    # Get vulnerability data
    trivy image --format json "$image" > temp-scan.json
    
    # Extract package vulnerabilities
    echo "Vulnerable packages that can be updated:"
    jq -r '.Results[]?.Vulnerabilities[]? | select(.FixedVersion != null and .FixedVersion != "") | 
    "Package: " + .PkgName + " | Current: " + .InstalledVersion + " | Fixed: " + .FixedVersion' temp-scan.json | sort -u
    
    echo
    echo "Remediation suggestions:"
    echo "1. Update base image to latest version"
    echo "2. Use Alpine-based images when possible"
    echo "3. Implement multi-stage builds"
    echo "4. Remove unnecessary packages"
    echo "5. Run containers as non-root user"
    echo
    
    # Clean up
    rm -f temp-scan.json
}

# Analyze images for remediation
suggest_remediation "ubuntu:18.04"
suggest_remediation "myapp:insecure"

# Create Dockerfile improvement suggestions
echo "=== Dockerfile Security Best Practices ==="
cat << 'BEST_PRACTICES'

1. Use specific image tags instead of 'latest'
   ❌ FROM ubuntu:latest
   ✅ FROM ubuntu:22.04

2. Use official base images
   ✅ FROM node:16-alpine

3. Create and use non-root user
   ✅ RUN adduser --disabled-password --gecos '' appuser
   ✅ USER appuser

4. Minimize installed packages
   ✅ RUN apt-get update && apt-get install -y --no-install-recommends \
       package1 package2 \
       && rm -rf /var/lib/apt/lists/*

5. Use multi-stage builds
   ✅ FROM node:16 AS builder
   ✅ FROM node:16-alpine AS runtime

6. Set proper file permissions
   ✅ COPY --chown=appuser:appuser app.js /app/

7. Use HEALTHCHECK instruction
   ✅ HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
       CMD curl -f http://localhost:3000/health || exit 1

8. Avoid storing secrets in images
   ❌ ENV API_KEY=secret123
   ✅ Use Docker secrets or environment variables at runtime

BEST_PRACTICES

EOF

chmod +x auto-remediation.sh
./auto-remediation.sh
Subtask 5.4: Create Security Policy Configuration
# Create security policy configuration
mkdir -p security-policies

# Create Trivy policy file
cat > security-policies/trivy-policy.yaml << 'EOF'
# Trivy security policy configuration
policies:
  - name: "Critical Vulnerability Policy"
    description: "Fail build if critical vulnerabilities found"
    rules:
      - severity: "CRITICAL"
        action: "fail"
      - severity: "HIGH"
        action: "warn"
        threshold: 5
  
  - name: "Package Policy"
    description: "Check for vulnerable packages"
    rules:
      - package: "openssl"
        min_version: "1.1.1"
      - package: "curl"
        min_version: "7.68.0"

# Ignore specific CVEs (use with caution)
ignore:
  - "CVE-2023-XXXXX"  # Example: ignore specific CVE with justification
EOF

# Create OPA (Open Policy Agent) policy for advanced rules
cat > security-policies/security-policy.rego << 'EOF'
package docker.security

# Deny images with critical vulnerabilities
deny[msg] {
    input.vulnerabilities[_].severity == "CRITICAL"
    msg := "Critical vulnerabilities found - deployment denied"
}

# Warn about high severity vulnerabilities
warn[msg] {
    count(input.vulnerabilities[_].severity == "HIGH") > 5
    msg := "More than 5 high severity vulnerabilities found"
}

# Check for required security practices
deny[msg] {
    not input.user_defined
    msg := "Container must not run as root user"
}

# Check for outdated base images
warn[msg] {
    input.base_image_age_days > 90
    msg := "Base image is older than 90 days - consider updating"
}
EOF

echo "Security policies created in security-policies/ directory"
Task 6: Advanced Security Scanning and Monitoring
Subtask 6.1: Set Up Continuous Monitoring
# Create continuous monitoring script
cat > continuous-monitoring.sh << 'EOF'
#!/bin/bash

echo "=== Continuous Docker Security Monitoring ==="
echo

# Configuration
SCAN_INTERVAL=3600  # 1 hour in seconds
LOG_FILE="security-monitoring.log"
ALERT_THRESHOLD=5   # Alert if more than 5 high/critical vulnerabilities

# Function to log with timestamp
log_message() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Function to scan all local images
scan_all_images() {
    log_message "Starting security scan of all local images..."
    
    # Get list of all images
    images=$(docker images --format "{{.Repository}}:{{.Tag}}" | grep -v "<none>")
    
    for image in $images; do
        log_message "Scanning image: $image"
        
        # Run Trivy scan and count high/critical vulnerabilities
        vuln_count=$(trivy image --severity HIGH,CRITICAL --format json "$image" 2>/dev/null | \
                    jq '.Results[]?.Vulnerabilities | length' 2>/dev/null | head -1)
        
        if [ "$vuln_count" -gt "$ALERT_THRESHOLD" ]; then
            log_message "ALERT: $image has $vuln_count high/critical vulnerabilities"
            
            # Generate detailed report for problematic image
            trivy image --severity HIGH,CRITICAL "$image" > "alert-${image//[\/:]/-}-$(date +%Y%m%d-%H%M%S).txt"
        else
            log_message "OK: $image has $vuln_count high/critical vulnerabilities"
        fi
    done
    
    log_message "Security scan completed"
}

# Function to check for new vulnerabilities in existing images
check_new_vulnerabilities() {
    log_message "Checking for new vulnerabilities in existing images..."
    
    # Update Trivy database
    trivy image --download-db-only
    
    # Re-scan critical images
    critical_images=("ubuntu:22.04" "node:16" "nginx:latest")
    
    for image in "${critical_images[@]}"; do
        if docker images | grep -q "${image%:*}"; then
            log_message "Re-scanning critical image: $image"
            trivy image --severity HIGH,CRITICAL "$image" > "rescan-${image//[\/:]/-}-$(date +%Y%m%d).txt"
        fi
    done
}

# Main monitoring loop
main_loop() {
    log_message "Starting continuous monitoring (interval: ${SCAN_INTERVAL}s)"
    
    while true; do
        scan_all_images
        check_new_vulnerabilities
        
        log_message "Sleeping for $SCAN_INTERVAL seconds..."
        sleep "$SCAN_INTERVAL"
    done
}

# Check if running in daemon mode
if [ "$1" = "--daemon" ]; then
    main_loop &
    echo "Monitoring started in background (PID: $!)"
    echo "Logs: $LOG_FILE"
else
    # Run once
    scan_all_images
    check_new_vulnerabilities
fi

EOF

chmod +x continuous-monitoring.sh

# Run monitoring once to test
./continuous-monitoring.sh
Subtask 6.2: Create Security Dashboard
# Create security dashboard generator
cat > generate-dashboard.sh << 'EOF'
#!/bin/bash

echo "=== Generating Security Dashboard ==="

# Create HTML dashboard
cat > security-dashboard.html << 'HTML_START'
<!DOCTYPE html>
<html>
<head>
    <title>Docker Security Dashboard</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .header { background-color: #2c3e50; color: white; padding: 20px; border-radius: 5px; }
        .section { margin: 20px 0; padding: 15px; border: 1px solid #ddd; border-radius: 5px; }
        .critical { background-color: #e74c3c; color: white; }
        .high { background-color: #f39c12; color: white; }
        .medium { background-color: #f1c40f; }
        .low { background-color: #27ae60; color: white; }
        .safe { background-color: #2ecc71; color: white; }
        table { width: 100%; border-collapse: collapse; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <div class="header">
        <h1>Docker Security Dashboard</h1>
        <p>Generated on: $(date)</p>
    </div>
HTML_START

# Add image security summary
echo '    <div class="section">' >> security-dashboard.html
echo '        <h2>Image Security Summary</h2>' >> security-dashboard.html
echo '        <table>' >> security-dashboard.html
echo '            <tr><th>Image</th><th>Total Vulnerabilities</th><th>Critical</th><th>High</th><th>Medium</th><th>Low</th><th>Status</th></tr>' >> security-dashboard.html

# Scan images and add to dashboard
images=$(docker images --format "{{.Repository}}:{{.Tag}}" | grep -v "<none>" | head -10)

for image in $images; 
