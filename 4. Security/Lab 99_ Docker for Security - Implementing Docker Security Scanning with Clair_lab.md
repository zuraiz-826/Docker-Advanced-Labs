Lab 99: Docker for Security - Implementing Docker Security Scanning with Clair
Lab Objectives
By the end of this lab, students will be able to:

Install and configure Clair vulnerability scanner for Docker images
Perform comprehensive vulnerability scans on Docker images using Clair
Set up automated scanning pipelines for Docker registries
Analyze security scan results and implement remediation strategies
Integrate Clair into CI/CD pipelines for continuous security monitoring
Understand Docker security best practices and vulnerability management
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and images
Familiarity with Linux command line operations
Knowledge of YAML configuration files
Understanding of basic networking concepts
Experience with Docker registries (Docker Hub, local registries)
Basic knowledge of CI/CD concepts
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your dedicated environment. No need to build your own VM or install additional software - everything is ready to use!

Your cloud machine includes:

Ubuntu 20.04 LTS
Docker Engine pre-installed
Docker Compose pre-installed
Git and curl utilities
All necessary network configurations
Task 1: Install Clair and Set Up Scanning Environment
Subtask 1.1: Verify Docker Installation and Prepare Environment
First, let's verify that Docker is properly installed and prepare our working directory.

# Check Docker version
docker --version

# Check Docker Compose version
docker-compose --version

# Create working directory for Clair setup
mkdir -p ~/clair-lab
cd ~/clair-lab

# Create directory structure
mkdir -p {config,data,logs}
Subtask 1.2: Download and Configure Clair
We'll use Docker Compose to set up Clair with PostgreSQL database.

# Create docker-compose.yml for Clair setup
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:13
    container_name: clair-postgres
    environment:
      POSTGRES_DB: clair
      POSTGRES_USER: clair
      POSTGRES_PASSWORD: clairpassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - clair-network
    restart: unless-stopped

  clair:
    image: quay.io/coreos/clair:v2.1.8
    container_name: clair-scanner
    depends_on:
      - postgres
    ports:
      - "6060:6060"
      - "6061:6061"
    volumes:
      - ./config:/config
      - ./logs:/var/log/clair
    command: ["-config", "/config/config.yaml"]
    networks:
      - clair-network
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  clair-network:
    driver: bridge
EOF
Subtask 1.3: Create Clair Configuration File
# Create Clair configuration file
cat > config/config.yaml << 'EOF'
clair:
  database:
    type: pgsql
    options:
      source: host=postgres port=5432 user=clair password=clairpassword dbname=clair sslmode=disable
      cachesize: 16384
  api:
    healthport: 6061
    port: 6060
    grpcport: 6062
  updater:
    interval: 2h
    enabledupdaters:
      - debian
      - ubuntu
      - rhel
      - oracle
      - alpine
  notifier:
    attempts: 3
    renotifyinterval: 2h
    http:
      endpoint: http://localhost:3000/notify
      timeout: 900
EOF
Subtask 1.4: Start Clair Services
# Start Clair and PostgreSQL services
docker-compose up -d

# Wait for services to initialize (about 2-3 minutes)
echo "Waiting for Clair to initialize..."
sleep 180

# Check if services are running
docker-compose ps

# Check Clair health
curl -s http://localhost:6061/health | jq '.' || echo "Clair is starting up..."
Subtask 1.5: Install Clair Client Tools
# Download clair-scanner client
wget https://github.com/arminc/clair-scanner/releases/download/v12/clair-scanner_linux_amd64
chmod +x clair-scanner_linux_amd64
sudo mv clair-scanner_linux_amd64 /usr/local/bin/clair-scanner

# Verify installation
clair-scanner --version
Task 2: Scan a Docker Image for Vulnerabilities Using Clair
Subtask 2.1: Prepare Test Images
Let's create and pull some Docker images to scan for vulnerabilities.

# Pull various images with different vulnerability levels
docker pull ubuntu:18.04
docker pull nginx:1.19
docker pull node:14-alpine
docker pull python:3.8-slim

# Create a custom vulnerable image for testing
cat > Dockerfile.vulnerable << 'EOF'
FROM ubuntu:18.04

# Install some packages with known vulnerabilities
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    openssl=1.1.1-1ubuntu2.1~18.04.13 \
    && rm -rf /var/lib/apt/lists/*

# Add a simple application
COPY app.py /app/
WORKDIR /app
CMD ["python3", "app.py"]
EOF

# Create a simple Python app
cat > app.py << 'EOF'
#!/usr/bin/env python3
print("Hello from vulnerable container!")
EOF

# Build the vulnerable image
docker build -f Dockerfile.vulnerable -t vulnerable-app:latest .
Subtask 2.2: Perform Basic Vulnerability Scans
# Scan Ubuntu 18.04 image
echo "Scanning Ubuntu 18.04 image..."
clair-scanner --clair=http://localhost:6060 --ip=$(hostname -I | awk '{print $1}') ubuntu:18.04

# Scan nginx image
echo "Scanning nginx image..."
clair-scanner --clair=http://localhost:6060 --ip=$(hostname -I | awk '{print $1}') nginx:1.19

# Scan our custom vulnerable image
echo "Scanning custom vulnerable image..."
clair-scanner --clair=http://localhost:6060 --ip=$(hostname -I | awk '{print $1}') vulnerable-app:latest
Subtask 2.3: Generate Detailed Scan Reports
# Create reports directory
mkdir -p ~/clair-lab/reports

# Scan with detailed JSON output
clair-scanner \
  --clair=http://localhost:6060 \
  --ip=$(hostname -I | awk '{print $1}') \
  --report=reports/ubuntu-scan-report.json \
  --log=reports/ubuntu-scan.log \
  --whitelist=whitelist.yaml \
  ubuntu:18.04

# Scan nginx with threshold settings
clair-scanner \
  --clair=http://localhost:6060 \
  --ip=$(hostname -I | awk '{print $1}') \
  --report=reports/nginx-scan-report.json \
  --threshold=Medium \
  nginx:1.19

# Create a whitelist file for testing
cat > whitelist.yaml << 'EOF'
generalwhitelist:
  - CVE-2019-1234  # Example CVE to whitelist
  - CVE-2020-5678  # Another example
EOF
Subtask 2.4: Analyze Scan Results
# View scan results
echo "=== Ubuntu Scan Results ==="
if [ -f reports/ubuntu-scan-report.json ]; then
    cat reports/ubuntu-scan-report.json | jq '.vulnerabilities[] | select(.severity=="High") | {name: .featurename, version: .featureversion, vulnerability: .vulnerability, severity: .severity}'
fi

# Count vulnerabilities by severity
echo "=== Vulnerability Summary ==="
if [ -f reports/nginx-scan-report.json ]; then
    echo "Critical: $(cat reports/nginx-scan-report.json | jq '.vulnerabilities[] | select(.severity=="Critical")' | jq -s length)"
    echo "High: $(cat reports/nginx-scan-report.json | jq '.vulnerabilities[] | select(.severity=="High")' | jq -s length)"
    echo "Medium: $(cat reports/nginx-scan-report.json | jq '.vulnerabilities[] | select(.severity=="Medium")' | jq -s length)"
    echo "Low: $(cat reports/nginx-scan-report.json | jq '.vulnerabilities[] | select(.severity=="Low")' | jq -s length)"
fi
Task 3: Set Up Pipeline to Automatically Scan Docker Images
Subtask 3.1: Create Local Docker Registry
# Start a local Docker registry
docker run -d -p 5000:5000 --name local-registry registry:2

# Tag and push images to local registry
docker tag ubuntu:18.04 localhost:5000/ubuntu:18.04
docker tag nginx:1.19 localhost:5000/nginx:1.19
docker tag vulnerable-app:latest localhost:5000/vulnerable-app:latest

docker push localhost:5000/ubuntu:18.04
docker push localhost:5000/nginx:1.19
docker push localhost:5000/vulnerable-app:latest
Subtask 3.2: Create Automated Scanning Script
# Create automated scanning script
cat > scan-pipeline.sh << 'EOF'
#!/bin/bash

# Configuration
CLAIR_URL="http://localhost:6060"
REGISTRY_URL="localhost:5000"
REPORTS_DIR="./reports"
HOST_IP=$(hostname -I | awk '{print $1}')

# Create reports directory
mkdir -p $REPORTS_DIR

# Function to scan image
scan_image() {
    local image=$1
    local report_name=$(echo $image | sed 's/[\/:]/-/g')
    
    echo "Scanning image: $image"
    
    clair-scanner \
        --clair=$CLAIR_URL \
        --ip=$HOST_IP \
        --report="$REPORTS_DIR/${report_name}-report.json" \
        --log="$REPORTS_DIR/${report_name}.log" \
        --threshold=Medium \
        $image
    
    # Check scan result
    if [ $? -eq 0 ]; then
        echo "✓ Scan completed successfully for $image"
    else
        echo "✗ Scan failed or vulnerabilities found for $image"
    fi
    
    echo "----------------------------------------"
}

# Function to get images from registry
get_registry_images() {
    echo "Fetching images from registry..."
    curl -s http://$REGISTRY_URL/v2/_catalog | jq -r '.repositories[]' | while read repo; do
        curl -s http://$REGISTRY_URL/v2/$repo/tags/list | jq -r '.tags[]' | while read tag; do
            echo "$REGISTRY_URL/$repo:$tag"
        done
    done
}

# Main scanning pipeline
echo "Starting automated Docker image scanning pipeline..."
echo "======================================================="

# Scan images from local registry
get_registry_images | while read image; do
    scan_image $image
done

echo "Scanning pipeline completed!"
EOF

# Make script executable
chmod +x scan-pipeline.sh
Subtask 3.3: Create Registry Webhook for Automatic Scanning
# Create webhook listener script
cat > webhook-listener.py << 'EOF'
#!/usr/bin/env python3

import json
import subprocess
import sys
from http.server import HTTPServer, BaseHTTPRequestHandler
import threading
import time

class WebhookHandler(BaseHTTPRequestHandler):
    def do_POST(self):
        content_length = int(self.headers['Content-Length'])
        post_data = self.rfile.read(content_length)
        
        try:
            webhook_data = json.loads(post_data.decode('utf-8'))
            
            # Extract image information
            if 'repository' in webhook_data and 'tag' in webhook_data:
                image_name = f"localhost:5000/{webhook_data['repository']}:{webhook_data['tag']}"
                print(f"Received webhook for image: {image_name}")
                
                # Trigger scan in background
                threading.Thread(target=self.scan_image, args=(image_name,)).start()
                
                self.send_response(200)
                self.send_header('Content-type', 'application/json')
                self.end_headers()
                self.wfile.write(json.dumps({"status": "accepted"}).encode())
            else:
                self.send_response(400)
                self.end_headers()
                
        except Exception as e:
            print(f"Error processing webhook: {e}")
            self.send_response(500)
            self.end_headers()
    
    def scan_image(self, image_name):
        print(f"Starting scan for {image_name}")
        try:
            result = subprocess.run([
                'clair-scanner',
                '--clair=http://localhost:6060',
                f'--ip={subprocess.check_output(["hostname", "-I"]).decode().strip().split()[0]}',
                f'--report=reports/{image_name.replace("/", "-").replace(":", "-")}-webhook-report.json',
                image_name
            ], capture_output=True, text=True)
            
            if result.returncode == 0:
                print(f"✓ Webhook scan completed for {image_name}")
            else:
                print(f"✗ Webhook scan found issues for {image_name}")
                
        except Exception as e:
            print(f"Error scanning {image_name}: {e}")

if __name__ == '__main__':
    server = HTTPServer(('localhost', 8080), WebhookHandler)
    print("Webhook listener started on http://localhost:8080")
    server.serve_forever()
EOF

# Make webhook script executable
chmod +x webhook-listener.py
Subtask 3.4: Test Automated Pipeline
# Run the scanning pipeline
echo "Testing automated scanning pipeline..."
./scan-pipeline.sh

# Start webhook listener in background (for demonstration)
python3 webhook-listener.py &
WEBHOOK_PID=$!

# Simulate webhook trigger
sleep 5
curl -X POST http://localhost:8080 \
  -H "Content-Type: application/json" \
  -d '{"repository": "nginx", "tag": "1.19"}'

# Stop webhook listener
sleep 10
kill $WEBHOOK_PID 2>/dev/null || true
Task 4: Analyze Security Scan Results and Apply Patches
Subtask 4.1: Create Vulnerability Analysis Script
# Create vulnerability analysis script
cat > analyze-vulnerabilities.py << 'EOF'
#!/usr/bin/env python3

import json
import os
import sys
from collections import defaultdict

def analyze_report(report_file):
    """Analyze a Clair scan report"""
    if not os.path.exists(report_file):
        print(f"Report file {report_file} not found")
        return
    
    with open(report_file, 'r') as f:
        data = json.load(f)
    
    vulnerabilities = data.get('vulnerabilities', [])
    
    # Group by severity
    severity_count = defaultdict(int)
    severity_details = defaultdict(list)
    
    for vuln in vulnerabilities:
        severity = vuln.get('severity', 'Unknown')
        severity_count[severity] += 1
        severity_details[severity].append({
            'name': vuln.get('vulnerability', 'Unknown'),
            'feature': vuln.get('featurename', 'Unknown'),
            'version': vuln.get('featureversion', 'Unknown'),
            'description': vuln.get('description', 'No description'),
            'link': vuln.get('link', 'No link')
        })
    
    return severity_count, severity_details

def generate_remediation_report(report_files):
    """Generate remediation recommendations"""
    print("=== VULNERABILITY ANALYSIS AND REMEDIATION REPORT ===")
    print("=" * 60)
    
    all_vulnerabilities = defaultdict(int)
    critical_issues = []
    
    for report_file in report_files:
        if os.path.exists(report_file):
            image_name = os.path.basename(report_file).replace('-report.json', '')
            print(f"\n📊 Analysis for: {image_name}")
            print("-" * 40)
            
            severity_count, severity_details = analyze_report(report_file)
            
            # Display summary
            for severity in ['Critical', 'High', 'Medium', 'Low']:
                count = severity_count.get(severity, 0)
                all_vulnerabilities[severity] += count
                if count > 0:
                    emoji = "🔴" if severity == "Critical" else "🟠" if severity == "High" else "🟡" if severity == "Medium" else "🟢"
                    print(f"{emoji} {severity}: {count}")
                    
                    # Show details for Critical and High
                    if severity in ['Critical', 'High'] and count > 0:
                        print(f"   Top {min(3, count)} {severity} vulnerabilities:")
                        for i, vuln in enumerate(severity_details[severity][:3]):
                            print(f"   {i+1}. {vuln['name']} in {vuln['feature']} {vuln['version']}")
                            critical_issues.append({
                                'image': image_name,
                                'severity': severity,
                                'vulnerability': vuln
                            })
    
    # Overall summary
    print(f"\n🎯 OVERALL SUMMARY")
    print("=" * 30)
    total_vulns = sum(all_vulnerabilities.values())
    print(f"Total vulnerabilities found: {total_vulns}")
    for severity in ['Critical', 'High', 'Medium', 'Low']:
        count = all_vulnerabilities.get(severity, 0)
        if count > 0:
            print(f"{severity}: {count}")
    
    # Remediation recommendations
    print(f"\n💡 REMEDIATION RECOMMENDATIONS")
    print("=" * 35)
    
    if all_vulnerabilities.get('Critical', 0) > 0:
        print("🚨 IMMEDIATE ACTION REQUIRED:")
        print("   - Critical vulnerabilities found")
        print("   - Update base images immediately")
        print("   - Consider using minimal/distroless images")
    
    if all_vulnerabilities.get('High', 0) > 0:
        print("⚠️  HIGH PRIORITY:")
        print("   - Update affected packages")
        print("   - Review security patches")
        print("   - Implement security scanning in CI/CD")
    
    print("\n📋 GENERAL RECOMMENDATIONS:")
    print("   - Use specific image tags instead of 'latest'")
    print("   - Regularly update base images")
    print("   - Use multi-stage builds to reduce attack surface")
    print("   - Implement image signing and verification")
    print("   - Set up automated vulnerability monitoring")

if __name__ == "__main__":
    # Find all report files
    reports_dir = "reports"
    report_files = []
    
    if os.path.exists(reports_dir):
        for file in os.listdir(reports_dir):
            if file.endswith('-report.json'):
                report_files.append(os.path.join(reports_dir, file))
    
    if report_files:
        generate_remediation_report(report_files)
    else:
        print("No scan reports found. Please run scans first.")
EOF

# Make analysis script executable
chmod +x analyze-vulnerabilities.py
Subtask 4.2: Run Vulnerability Analysis
# Run vulnerability analysis
python3 analyze-vulnerabilities.py

# Create detailed remediation plan
cat > remediation-plan.md << 'EOF'
# Docker Security Remediation Plan

## Immediate Actions (Critical/High Vulnerabilities)

### 1. Update Base Images
- Replace `ubuntu:18.04` with `ubuntu:20.04` or `ubuntu:22.04`
- Use specific version tags instead of `latest`
- Consider using minimal images like Alpine or distroless

### 2. Package Updates
- Update system packages in Dockerfiles
- Use `apt-get update && apt-get upgrade` for Ubuntu/Debian
- Pin specific package versions when possible

### 3. Security Scanning Integration
- Add Clair scanning to CI/CD pipeline
- Set vulnerability thresholds for build failures
- Implement automated alerts for new vulnerabilities

## Medium-Term Actions

### 1. Image Optimization
- Use multi-stage builds to reduce image size
- Remove unnecessary packages and files
- Use .dockerignore to exclude sensitive files

### 2. Runtime Security
- Run containers as non-root users
- Use read-only filesystems where possible
- Implement resource limits and security contexts

### 3. Monitoring and Compliance
- Set up continuous monitoring for new vulnerabilities
- Implement image signing and verification
- Regular security audits and compliance checks

## Best Practices

1. **Image Selection**: Choose official, well-maintained base images
2. **Regular Updates**: Establish a schedule for updating images
3. **Minimal Principle**: Include only necessary components
4. **Security Scanning**: Integrate into development workflow
5. **Documentation**: Maintain security documentation and procedures
EOF

echo "Remediation plan created: remediation-plan.md"
Subtask 4.3: Create Secure Docker Images
# Create improved, more secure Dockerfile
cat > Dockerfile.secure << 'EOF'
# Use specific, recent version
FROM ubuntu:22.04

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Update packages and install only necessary ones
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
        python3 \
        python3-pip && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# Set up application directory
WORKDIR /app
COPY app.py /app/

# Change ownership to non-root user
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Use specific command
CMD ["python3", "app.py"]
EOF

# Build secure image
docker build -f Dockerfile.secure -t secure-app:latest .

# Scan the secure image
echo "Scanning secure image..."
clair-scanner \
  --clair=http://localhost:6060 \
  --ip=$(hostname -I | awk '{print $1}') \
  --report=reports/secure-app-report.json \
  secure-app:latest

# Compare results
echo "Comparing vulnerability counts..."
echo "Original vulnerable image:"
if [ -f reports/vulnerable-app-latest-report.json ]; then
    python3 -c "
import json
with open('reports/vulnerable-app-latest-report.json') as f:
    data = json.load(f)
    vulns = data.get('vulnerabilities', [])
    print(f'Total vulnerabilities: {len(vulns)}')
"
fi

echo "Secure image:"
if [ -f reports/secure-app-report.json ]; then
    python3 -c "
import json
with open('reports/secure-app-report.json') as f:
    data = json.load(f)
    vulns = data.get('vulnerabilities', [])
    print(f'Total vulnerabilities: {len(vulns)}')
"
fi
Task 5: Integrate Clair into CI/CD Pipeline
Subtask 5.1: Create GitHub Actions Workflow
# Create GitHub Actions workflow directory
mkdir -p .github/workflows

# Create CI/CD workflow with Clair integration
cat > .github/workflows/docker-security-scan.yml << 'EOF'
name: Docker Security Scan with Clair

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  security-scan:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_DB: clair
          POSTGRES_USER: clair
          POSTGRES_PASSWORD: clairpassword
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      clair:
        image: quay.io/coreos/clair:v2.1.8
        env:
          CLAIR_CONF: /config/config.yaml
        ports:
          - 6060:6060
          - 6061:6061
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v2
    
    - name: Build Docker image
      run: |
        docker build -t test-image:${{ github.sha }} .
    
    - name: Install Clair Scanner
      run: |
        wget https://github.com/arminc/clair-scanner/releases/download/v12/clair-scanner_linux_amd64
        chmod +x clair-scanner_linux_amd64
        sudo mv clair-scanner_linux_amd64 /usr/local/bin/clair-scanner
    
    - name: Wait for Clair to be ready
      run: |
        timeout 300 bash -c 'until curl -s http://localhost:6061/health; do sleep 5; done'
    
    - name: Scan image for vulnerabilities
      run: |
        clair-scanner \
          --clair=http://localhost:6060 \
          --ip=$(hostname -I | awk '{print $1}') \
          --report=security-report.json \
          --threshold=High \
          test-image:${{ github.sha }}
    
    - name: Upload security report
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: security-report
        path: security-report.json
    
    - name: Comment PR with scan results
      if: github.event_name == 'pull_request'
      uses: actions/github-script@v6
      with:
        script: |
          const fs = require('fs');
          if (fs.existsSync('security-report.json')) {
            const report = JSON.parse(fs.readFileSync('security-report.json', 'utf8'));
            const vulns = report.vulnerabilities || [];
            const critical = vulns.filter(v => v.severity === 'Critical').length;
            const high = vulns.filter(v => v.severity === 'High').length;
            const medium = vulns.filter(v => v.severity === 'Medium').length;
            
            const comment = `## 🔍 Security Scan Results
            
            | Severity | Count |
            |----------|-------|
            | Critical | ${critical} |
            | High     | ${high} |
            | Medium   | ${medium} |
            
            ${critical > 0 ? '🚨 **Critical vulnerabilities found! Please address before merging.**' : ''}
            ${high > 0 ? '⚠️ **High severity vulnerabilities found.**' : ''}
            `;
            
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: comment
            });
          }
EOF
Subtask 5.2: Create Jenkins Pipeline
# Create Jenkinsfile for Jenkins CI/CD integration
cat > Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = "myapp:${BUILD_NUMBER}"
        CLAIR_URL = "http://clair:6060"
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    docker.build(env.DOCKER_IMAGE)
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                script {
                    // Start Clair services
                    sh '''
                        docker-compose -f docker-compose.clair.yml up -d
                        sleep 120  # Wait for Clair to initialize
                    '''
                    
                    // Run security scan
                    sh '''
                        clair-scanner \
                            --clair=${CLAIR_URL} \
                            --ip=$(hostname -I | awk '{print $1}') \
                            --report=security-report-${BUILD_NUMBER}.json \
                            --threshold=High \
                            ${DOCKER_IMAGE}
                    '''
                }
            }
            post {
                always {
                    // Archive security report
                    archiveArtifacts artifacts: 'security-report-*.json', fingerprint: true
                    
                    // Publish security report
                    publishHTML([
                        allowMissing: false,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'security-report-*.json',
                        reportName: 'Security Scan Report'
                    ])
                }
                cleanup {
                    sh 'docker-compose -f docker-compose.clair.yml down'
                }
            }
        }
        
        stage('Deploy') {
            when {
                allOf {
                    branch 'main'
                    expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
                }
            }
            steps {
                echo 'Deploying to production...'
                // Add deployment steps here
            }
        }
    }
    
    post {
        failure {
            emailext (
                subject: "Security Scan Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "Security vulnerabilities found in build ${env.BUILD_NUMBER}. Please check the scan report.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
EOF

# Create Docker Compose file for Jenkins Clair setup
cat > docker-compose.clair.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_DB: clair
      POSTGRES_USER: clair
      POSTGRES_PASSWORD: clairpassword
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - clair-network

  clair:
    image: quay.io/coreos/clair:v2.1.8
    depends_on:
      - postgres
    ports:
      - "6060:6060"
      - "6061:6061"
    volumes:
      - ./config:/config
    command: ["-config", "/config/config.yaml"]
    networks:
      - clair-network

volumes:
  postgres_data:

networks:
  clair-network:
    driver: bridge
EOF
Subtask 5.3: Create GitLab CI/CD Pipeline
# Create GitLab CI/CD configuration
cat > .gitlab-ci.yml << 'EOF'
stages:
  - build
  - security-scan
  - deploy

variables:
  DOCKER_IMAGE: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
  CLAIR_URL: "http://clair:6060"

services:
  - name: postgres:13
    alias: postgres
    variables:
      POSTGRES_DB: clair
      POSTGRES_USER: clair
      POSTGRES_PASSWORD: clairpassword
  
  - name: quay.io/coreos/clair:v2.1.8
    alias: clair
    variables:
      CLAIR_CONF: /config/config.yaml

build:
  stage: build
  image: docker:latest
  services:
