Lab 13: Securing Docker Containers
Objectives
By the end of this lab, students will be able to:

Implement Docker security best practices for containerized applications
Configure containers to run as non-root users using the USER directive
Set up Docker Content Trust and sign images for enhanced security
Use security scanning tools to identify vulnerabilities in container images
Limit container privileges using security options and capability dropping
Implement read-only file systems for containers to enhance security posture
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with Linux command line operations
Knowledge of basic security concepts
Understanding of user permissions and file systems in Linux
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to begin - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine installed
Docker Compose pre-configured
All necessary security scanning tools
Sample applications for testing
Task 1: Run Containers as Non-Root Users
Understanding the Security Risk
Running containers as root poses significant security risks. If an attacker compromises your container, they gain root access to the host system. This task demonstrates how to create and run containers with non-privileged users.

Subtask 1.1: Create a Dockerfile with Non-Root User
First, let's create a simple web application that runs as a non-root user.

Create a new directory for your secure application:
mkdir secure-app
cd secure-app
Create a simple Python web application:
cat > app.py << 'EOF'
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return f"Hello from secure container! Running as user: {os.getuid()}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF
Create a requirements file:
cat > requirements.txt << 'EOF'
Flask==2.3.3
EOF
Create a secure Dockerfile:
cat > Dockerfile << 'EOF'
FROM python:3.9-slim

# Create a non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Set working directory
WORKDIR /app

# Copy requirements and install dependencies as root
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application files
COPY app.py .

# Change ownership of the app directory to appuser
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Run the application
CMD ["python", "app.py"]
EOF
Subtask 1.2: Build and Test the Secure Image
Build the Docker image:
docker build -t secure-app:v1 .
Run the container and verify it's running as non-root:
docker run -d -p 8080:8080 --name secure-app-container secure-app:v1
Check the running processes inside the container:
docker exec secure-app-container ps aux
Test the application:
curl http://localhost:8080
Verify the user ID (should not be 0 for root):
docker exec secure-app-container id
Clean up:
docker stop secure-app-container
docker rm secure-app-container
Task 2: Implement Docker Content Trust
Understanding Docker Content Trust
Docker Content Trust provides cryptographic signing of Docker images, ensuring image integrity and authenticity. This prevents tampering and ensures you're running trusted images.

Subtask 2.1: Enable Docker Content Trust
Enable Docker Content Trust globally:
export DOCKER_CONTENT_TRUST=1
Verify the environment variable is set:
echo $DOCKER_CONTENT_TRUST
Subtask 2.2: Generate Signing Keys
Create a directory for trust data:
mkdir -p ~/.docker/trust
Initialize trust for your repository (replace 'yourusername' with your actual username):
docker trust key generate mykey
Add the key to your repository:
docker trust signer add --key mykey.pub mykey secure-app
Subtask 2.3: Sign and Push Images
Tag your image for signing:
docker tag secure-app:v1 localhost:5000/secure-app:signed
Start a local registry for testing:
docker run -d -p 5000:5000 --name registry registry:2
Push the signed image:
docker push localhost:5000/secure-app:signed
Verify the signature:
docker trust inspect localhost:5000/secure-app:signed
Subtask 2.4: Test Content Trust Verification
Try to pull an unsigned image (this should fail):
docker pull hello-world
Disable content trust temporarily and pull:
export DOCKER_CONTENT_TRUST=0
docker pull hello-world
export DOCKER_CONTENT_TRUST=1
Task 3: Security Scanning with Docker Scout
Understanding Security Scanning
Security scanning identifies known vulnerabilities in your container images by checking against vulnerability databases. Docker Scout is Docker's built-in security scanning tool.

Subtask 3.1: Enable Docker Scout
Check if Docker Scout is available:
docker scout version
If not available, update Docker to the latest version or install Docker Scout:
curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --
Subtask 3.2: Scan Images for Vulnerabilities
Scan your secure application image:
docker scout cves secure-app:v1
Get a detailed vulnerability report:
docker scout cves --format sarif --output secure-app-scan.json secure-app:v1
Scan a base image to compare:
docker scout cves python:3.9-slim
Subtask 3.3: Compare Image Security
Compare your image with a different base image:
docker scout compare secure-app:v1 --to python:3.9-alpine
Get recommendations for improving security:
docker scout recommendations secure-app:v1
Subtask 3.4: Create a More Secure Dockerfile
Based on scanning results, create an improved Dockerfile:

cat > Dockerfile.secure << 'EOF'
# Use a more secure base image
FROM python:3.9-alpine

# Install security updates
RUN apk update && apk upgrade && apk add --no-cache \
    && rm -rf /var/cache/apk/*

# Create non-root user
RUN addgroup -g 1001 -S appuser && \
    adduser -u 1001 -S appuser -G appuser

# Set working directory
WORKDIR /app

# Copy and install requirements
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copy application
COPY app.py .

# Set ownership
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Use non-root port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8080/ || exit 1

CMD ["python", "app.py"]
EOF
Build and scan the improved image:
docker build -f Dockerfile.secure -t secure-app:v2 .
docker scout cves secure-app:v2
Task 4: Limit Container Privileges
Understanding Container Privileges
By default, containers run with many Linux capabilities that may not be necessary for your application. Limiting these privileges reduces the attack surface.

Subtask 4.1: Drop Unnecessary Capabilities
First, let's see what capabilities a container normally has:
docker run --rm alpine:latest sh -c "apk add --no-cache libcap && capsh --print"
Run a container with dropped capabilities:
docker run -d --name limited-container \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  -p 8081:8080 \
  secure-app:v2
Verify the container is running with limited capabilities:
docker exec limited-container sh -c "apk add --no-cache libcap && capsh --print"
Subtask 4.2: Use Security Options
Run a container with additional security options:
docker run -d --name secure-container \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges:true \
  --security-opt=apparmor:docker-default \
  -p 8082:8080 \
  secure-app:v2
Verify the security options are applied:
docker inspect secure-container | grep -A 10 "SecurityOpt"
Subtask 4.3: Test Privilege Limitations
Try to perform privileged operations in the limited container:
# This should fail
docker exec limited-container mount -t tmpfs tmpfs /tmp/test
Compare with a regular container:
docker run --rm --privileged alpine:latest mount -t tmpfs tmpfs /tmp/test
Clean up containers:
docker stop limited-container secure-container
docker rm limited-container secure-container
Task 5: Use Read-Only File Systems
Understanding Read-Only File Systems
Read-only file systems prevent malicious code from writing to the container's file system, significantly improving security by limiting the impact of potential compromises.

Subtask 5.1: Create Application with Temporary Storage
Create an improved application that uses temporary storage:
cat > app_readonly.py << 'EOF'
from flask import Flask, request, jsonify
import os
import tempfile
import json

app = Flask(__name__)

@app.route('/')
def hello():
    return f"Hello from read-only container! Running as user: {os.getuid()}"

@app.route('/write-test', methods=['POST'])
def write_test():
    try:
        # This will work - writing to /tmp
        with tempfile.NamedTemporaryFile(mode='w', delete=False, dir='/tmp') as f:
            f.write("Temporary data")
            temp_file = f.name
        
        return jsonify({
            "status": "success",
            "message": f"Successfully wrote to {temp_file}",
            "writable_dirs": ["/tmp"]
        })
    except Exception as e:
        return jsonify({
            "status": "error",
            "message": str(e)
        })

@app.route('/illegal-write', methods=['POST'])
def illegal_write():
    try:
        # This will fail in read-only mode
        with open('/app/illegal_file.txt', 'w') as f:
            f.write("This should not work")
        return jsonify({"status": "success", "message": "File written"})
    except Exception as e:
        return jsonify({"status": "error", "message": str(e)})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
EOF
Subtask 5.2: Create Dockerfile for Read-Only Application
Create a Dockerfile optimized for read-only operation:
cat > Dockerfile.readonly << 'EOF'
FROM python:3.9-alpine

# Install dependencies and create user
RUN apk update && apk upgrade && \
    addgroup -g 1001 -S appuser && \
    adduser -u 1001 -S appuser -G appuser

# Set working directory
WORKDIR /app

# Copy and install requirements
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Copy application
COPY app_readonly.py .

# Create necessary directories and set permissions
RUN mkdir -p /tmp/app-data && \
    chown -R appuser:appuser /app /tmp/app-data

# Switch to non-root user
USER appuser

EXPOSE 8080

CMD ["python", "app_readonly.py"]
EOF
Build the read-only optimized image:
docker build -f Dockerfile.readonly -t readonly-app:v1 .
Subtask 5.3: Run Container with Read-Only File System
Run the container with read-only file system:
docker run -d --name readonly-container \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges:true \
  -p 8083:8080 \
  readonly-app:v1
Subtask 5.4: Test Read-Only Restrictions
Test that the application works normally:
curl http://localhost:8083/
Test writing to allowed temporary directory:
curl -X POST http://localhost:8083/write-test
Test writing to read-only file system (should fail):
curl -X POST http://localhost:8083/illegal-write
Try to create files directly in the container:
# This should fail
docker exec readonly-container touch /app/test-file.txt

# This should work
docker exec readonly-container touch /tmp/test-file.txt
Subtask 5.5: Monitor File System Access
Check what directories are writable:
docker exec readonly-container df -h
Verify mount options:
docker exec readonly-container mount | grep -E "(tmpfs|ro)"
Clean up:
docker stop readonly-container
docker rm readonly-container
Task 6: Comprehensive Security Implementation
Subtask 6.1: Create a Fully Secured Container
Combine all security practices into one comprehensive example:

cat > docker-compose.secure.yml << 'EOF'
version: '3.8'

services:
  secure-web-app:
    build:
      context: .
      dockerfile: Dockerfile.readonly
    ports:
      - "8084:8080"
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=100m
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-default
    user: "1001:1001"
    environment:
      - PYTHONUNBUFFERED=1
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
EOF
Subtask 6.2: Deploy and Test Secure Application
Deploy the secure application:
docker-compose -f docker-compose.secure.yml up -d
Verify the application is running securely:
docker-compose -f docker-compose.secure.yml ps
Test the application functionality:
curl http://localhost:8084/
curl -X POST http://localhost:8084/write-test
curl -X POST http://localhost:8084/illegal-write
Check security settings:
docker inspect $(docker-compose -f docker-compose.secure.yml ps -q) | grep -A 20 "SecurityOpt"
Subtask 6.3: Security Audit
Perform a final security scan:
docker scout cves readonly-app:v1
Check for running processes:
docker exec $(docker-compose -f docker-compose.secure.yml ps -q) ps aux
Verify file system permissions:
docker exec $(docker-compose -f docker-compose.secure.yml ps -q) ls -la /
Test privilege escalation (should fail):
docker exec $(docker-compose -f docker-compose.secure.yml ps -q) su -
Troubleshooting Common Issues
Issue 1: Docker Content Trust Errors
Problem: Content trust signing fails Solution:

# Reset trust data
rm -rf ~/.docker/trust
# Reinitialize
docker trust key generate mykey
Issue 2: Permission Denied in Read-Only Containers
Problem: Application fails to start due to write permissions Solution:

# Ensure proper tmpfs mounts
--tmpfs /tmp:rw,noexec,nosuid,size=100m
--tmpfs /var/tmp:rw,noexec,nosuid,size=50m
Issue 3: Security Scanning Tool Not Found
Problem: Docker Scout not available Solution:

# Install Docker Scout CLI
curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --
# Or use alternative tools
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image secure-app:v1
Issue 4: Capability Issues
Problem: Application needs specific capabilities Solution:

# Identify required capabilities
strace -e trace=capget,capset docker run --rm your-image
# Add only necessary capabilities
--cap-add=NET_ADMIN  # Only if needed
Lab Cleanup
Clean up all resources created during this lab:

# Stop and remove containers
docker-compose -f docker-compose.secure.yml down
docker stop $(docker ps -aq) 2>/dev/null || true
docker rm $(docker ps -aq) 2>/dev/null || true

# Remove images
docker rmi secure-app:v1 secure-app:v2 readonly-app:v1 2>/dev/null || true

# Remove local registry
docker stop registry && docker rm registry 2>/dev/null || true

# Clean up files
rm -rf secure-app/
rm -f *.json *.pub mykey

# Reset Docker Content Trust
unset DOCKER_CONTENT_TRUST
Conclusion
In this comprehensive lab, you have successfully implemented multiple layers of Docker container security:

Key Accomplishments:

Non-Root User Implementation: You learned to create and run containers with non-privileged users, significantly reducing the attack surface if a container is compromised.

Docker Content Trust: You implemented cryptographic signing and verification of container images, ensuring image integrity and authenticity in your deployment pipeline.

Security Scanning: You used Docker Scout to identify vulnerabilities in container images and learned to create more secure base images based on scan results.

Privilege Limitation: You successfully dropped unnecessary Linux capabilities and applied security options to limit what containers can do on the host system.

Read-Only File Systems: You implemented read-only containers with proper temporary storage, preventing malicious code from persisting changes to the container file system.

Why This Matters:

Container security is critical in production environments because:

Defense in Depth: Multiple security layers provide better protection than relying on a single security measure
Compliance Requirements: Many industries require specific security controls for containerized applications
Risk Mitigation: Proper security practices significantly reduce the impact of potential security breaches
Trust and Reliability: Signed and scanned images ensure you're running trusted, vulnerability-free code
Real-World Applications:

These security practices are essential for:

Production deployments in cloud environments
CI/CD pipelines requiring secure image handling
Compliance with security frameworks like NIST, CIS, or SOC 2
Multi-tenant environments where container isolation is critical
By mastering these Docker security techniques, you're now equipped to deploy containers safely in production environments and contribute to your organization's security posture. These skills are highly valued in the industry and directly applicable to Docker Certified Associate (DCA) certification requirements.
