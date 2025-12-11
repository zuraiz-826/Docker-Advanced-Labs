Lab 43: Docker Security Best Practices - Protecting Docker Containers
Objectives
By the end of this lab, you will be able to:

Understand fundamental Docker security concepts and vulnerabilities
Configure non-root users for containers to minimize privilege escalation risks
Implement read-only filesystems to prevent unauthorized modifications
Restrict container capabilities using capability dropping techniques
Apply container security policies using AppArmor or SELinux
Enable Docker Content Trust for image signing and verification
Implement a comprehensive Docker security strategy for production environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Linux command line operations
Familiarity with Docker fundamentals (containers, images, Dockerfile)
Knowledge of basic security concepts (users, permissions, access control)
Understanding of Linux file systems and permissions
Basic familiarity with text editors like nano or vim
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker already installed. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine installed
All necessary security tools pre-installed
Root and sudo access for configuration tasks
Sample applications for testing security implementations
Task 1: Set Up a Non-Root User for Containers
Understanding the Security Risk
Running containers as root poses significant security risks. If an attacker compromises your container, they gain root privileges, potentially accessing the host system.

Subtask 1.1: Create a Secure Dockerfile with Non-Root User
First, let's create a directory for our security lab and build a sample application:

mkdir ~/docker-security-lab
cd ~/docker-security-lab
Create a simple Python application:

cat > app.py << 'EOF'
#!/usr/bin/env python3
import os
import getpass
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello():
    return f"""
    <h1>Docker Security Demo</h1>
    <p>Current User: {getpass.getuser()}</p>
    <p>User ID: {os.getuid()}</p>
    <p>Group ID: {os.getgid()}</p>
    <p>Working Directory: {os.getcwd()}</p>
    """

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
Create a requirements file:

cat > requirements.txt << 'EOF'
Flask==2.3.3
EOF
Subtask 1.2: Create an Insecure Dockerfile (Root User)
First, let's create a Dockerfile that runs as root to demonstrate the security issue:

cat > Dockerfile.insecure << 'EOF'
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
Build and test the insecure image:

docker build -f Dockerfile.insecure -t insecure-app .
docker run -d --name insecure-container -p 5001:5000 insecure-app
Check the user running inside the container:

docker exec insecure-container whoami
docker exec insecure-container id
Subtask 1.3: Create a Secure Dockerfile with Non-Root User
Now, create a secure version with a non-root user:

cat > Dockerfile.secure << 'EOF'
FROM python:3.9-slim

# Create a non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Create app directory and set ownership
WORKDIR /app
RUN chown appuser:appuser /app

# Install dependencies as root
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application files and set ownership
COPY app.py .
RUN chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

EXPOSE 5000

CMD ["python", "app.py"]
EOF
Build and test the secure image:

docker build -f Dockerfile.secure -t secure-app .
docker run -d --name secure-container -p 5002:5000 secure-app
Verify the non-root user:

docker exec secure-container whoami
docker exec secure-container id
Subtask 1.4: Compare Security Implications
Test the security difference by trying to access sensitive files:

# Try to access /etc/passwd in insecure container (should work)
docker exec insecure-container cat /etc/passwd

# Try to access /etc/passwd in secure container (should work but with limited privileges)
docker exec secure-container cat /etc/passwd

# Try to write to system directories
docker exec insecure-container touch /etc/test-file
docker exec secure-container touch /etc/test-file
Clean up the test containers:

docker stop insecure-container secure-container
docker rm insecure-container secure-container
Task 2: Use Docker's Read-Only Filesystem for Containers
Understanding Read-Only Filesystems
Read-only filesystems prevent malicious code from modifying system files or installing persistent backdoors.

Subtask 2.1: Create an Application That Needs Temporary Storage
Create a new application that demonstrates read-only filesystem usage:

cat > logging-app.py << 'EOF'
#!/usr/bin/env python3
import os
import time
import tempfile
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return """
    <h1>Logging Application</h1>
    <p><a href="/write-temp">Write to /tmp</a></p>
    <p><a href="/write-app">Write to /app</a></p>
    <p><a href="/logs">View Logs</a></p>
    """

@app.route('/write-temp')
def write_temp():
    try:
        with open('/tmp/temp-log.txt', 'a') as f:
            f.write(f"Temp log entry at {time.ctime()}\n")
        return "Successfully wrote to /tmp"
    except Exception as e:
        return f"Error writing to /tmp: {str(e)}"

@app.route('/write-app')
def write_app():
    try:
        with open('/app/app-log.txt', 'a') as f:
            f.write(f"App log entry at {time.ctime()}\n")
        return "Successfully wrote to /app"
    except Exception as e:
        return f"Error writing to /app: {str(e)}"

@app.route('/logs')
def view_logs():
    logs = ""
    try:
        if os.path.exists('/tmp/temp-log.txt'):
            with open('/tmp/temp-log.txt', 'r') as f:
                logs += "<h3>Temp Logs:</h3><pre>" + f.read() + "</pre>"
    except:
        pass
    
    try:
        if os.path.exists('/app/app-log.txt'):
            with open('/app/app-log.txt', 'r') as f:
                logs += "<h3>App Logs:</h3><pre>" + f.read() + "</pre>"
    except:
        pass
    
    return logs if logs else "No logs found"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
Subtask 2.2: Create Dockerfile for Read-Only Testing
cat > Dockerfile.readonly << 'EOF'
FROM python:3.9-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app
RUN chown appuser:appuser /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY logging-app.py .
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 5000

CMD ["python", "logging-app.py"]
EOF
Build the image:

docker build -f Dockerfile.readonly -t readonly-test .
Subtask 2.3: Test Normal Container (Read-Write)
Run the container normally:

docker run -d --name normal-container -p 5003:5000 readonly-test
Test writing capabilities:

curl http://localhost:5003/write-temp
curl http://localhost:5003/write-app
curl http://localhost:5003/logs
Subtask 2.4: Test Read-Only Container with Temporary Volume
Stop the normal container and run with read-only filesystem:

docker stop normal-container
docker rm normal-container

# Run with read-only filesystem and tmpfs for /tmp
docker run -d --name readonly-container \
  --read-only \
  --tmpfs /tmp \
  -p 5003:5000 \
  readonly-test
Test the read-only behavior:

# This should work (writing to /tmp which is mounted as tmpfs)
curl http://localhost:5003/write-temp

# This should fail (trying to write to read-only /app)
curl http://localhost:5003/write-app

# View logs
curl http://localhost:5003/logs
Subtask 2.5: Advanced Read-Only Configuration
Create a more sophisticated read-only setup:

docker stop readonly-container
docker rm readonly-container

# Create a volume for persistent logs
docker volume create app-logs

# Run with read-only root, tmpfs for temp files, and volume for logs
docker run -d --name advanced-readonly \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --mount source=app-logs,target=/app/logs \
  -p 5003:5000 \
  readonly-test
Task 3: Limit Container Capabilities Using --cap-drop
Understanding Linux Capabilities
Linux capabilities divide root privileges into distinct units. Containers inherit many capabilities by default, which can be security risks.

Subtask 3.1: Examine Default Container Capabilities
First, let's see what capabilities a container has by default:

# Run a container and check its capabilities
docker run --rm -it ubuntu:20.04 bash -c "
apt-get update -qq && apt-get install -y libcap2-bin -qq
capsh --print
"
Subtask 3.2: Create a Capability Testing Application
Create a Python script to test various capabilities:

cat > capability-test.py << 'EOF'
#!/usr/bin/env python3
import os
import socket
import subprocess
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return """
    <h1>Capability Testing</h1>
    <p><a href="/test-network">Test Network Capabilities</a></p>
    <p><a href="/test-process">Test Process Capabilities</a></p>
    <p><a href="/test-filesystem">Test Filesystem Capabilities</a></p>
    <p><a href="/capabilities">View Current Capabilities</a></p>
    """

@app.route('/capabilities')
def show_capabilities():
    try:
        result = subprocess.run(['capsh', '--print'], 
                              capture_output=True, text=True)
        return f"<pre>{result.stdout}</pre>"
    except Exception as e:
        return f"Error checking capabilities: {str(e)}"

@app.route('/test-network')
def test_network():
    try:
        # Try to bind to a privileged port (requires CAP_NET_BIND_SERVICE)
        sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        sock.bind(('0.0.0.0', 80))
        sock.close()
        return "SUCCESS: Can bind to privileged port 80"
    except Exception as e:
        return f"FAILED: Cannot bind to privileged port: {str(e)}"

@app.route('/test-process')
def test_process():
    try:
        # Try to change process priority (requires CAP_SYS_NICE)
        os.nice(-1)
        return "SUCCESS: Can change process priority"
    except Exception as e:
        return f"FAILED: Cannot change process priority: {str(e)}"

@app.route('/test-filesystem')
def test_filesystem():
    try:
        # Try to change file ownership (requires CAP_CHOWN)
        with open('/tmp/test-file', 'w') as f:
            f.write('test')
        os.chown('/tmp/test-file', 1000, 1000)
        return "SUCCESS: Can change file ownership"
    except Exception as e:
        return f"FAILED: Cannot change file ownership: {str(e)}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
Subtask 3.3: Create Dockerfile for Capability Testing
cat > Dockerfile.capabilities << 'EOF'
FROM python:3.9-slim

# Install required packages
RUN apt-get update && apt-get install -y \
    libcap2-bin \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app
RUN chown appuser:appuser /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY capability-test.py .
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 5000

CMD ["python", "capability-test.py"]
EOF
Build the capability testing image:

docker build -f Dockerfile.capabilities -t capability-test .
Subtask 3.4: Test Default Capabilities
Run the container with default capabilities:

docker run -d --name default-caps -p 5004:5000 capability-test
Test the capabilities:

curl http://localhost:5004/capabilities
curl http://localhost:5004/test-network
curl http://localhost:5004/test-process
curl http://localhost:5004/test-filesystem
Subtask 3.5: Drop All Capabilities
Stop the container and run with all capabilities dropped:

docker stop default-caps
docker rm default-caps

# Drop all capabilities
docker run -d --name no-caps \
  --cap-drop=ALL \
  -p 5004:5000 \
  capability-test
Test with no capabilities:

curl http://localhost:5004/capabilities
curl http://localhost:5004/test-network
curl http://localhost:5004/test-process
curl http://localhost:5004/test-filesystem
Subtask 3.6: Selective Capability Management
Run with specific capabilities dropped:

docker stop no-caps
docker rm no-caps

# Drop specific dangerous capabilities
docker run -d --name selective-caps \
  --cap-drop=NET_ADMIN \
  --cap-drop=SYS_ADMIN \
  --cap-drop=DAC_OVERRIDE \
  --cap-drop=SETUID \
  --cap-drop=SETGID \
  -p 5004:5000 \
  capability-test
Test selective capability dropping:

curl http://localhost:5004/capabilities
curl http://localhost:5004/test-network
Task 4: Implement Container Security Policies with AppArmor
Understanding AppArmor
AppArmor is a Linux security module that provides mandatory access control by restricting programs' capabilities.

Subtask 4.1: Check AppArmor Status
First, verify AppArmor is available and running:

# Check if AppArmor is installed and running
sudo systemctl status apparmor

# Check AppArmor status
sudo apparmor_status

# List current profiles
sudo aa-status
Subtask 4.2: Create a Custom AppArmor Profile
Create a restrictive AppArmor profile for our Docker container:

sudo mkdir -p /etc/apparmor.d/docker

cat << 'EOF' | sudo tee /etc/apparmor.d/docker-secure-app
#include <tunables/global>

profile docker-secure-app flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  
  # Allow network access
  network inet tcp,
  network inet udp,
  
  # Allow reading from specific directories
  /app/** r,
  /usr/local/lib/python3.9/** r,
  /usr/lib/python3.9/** r,
  /lib/x86_64-linux-gnu/** r,
  /usr/lib/x86_64-linux-gnu/** r,
  
  # Allow execution of Python
  /usr/local/bin/python3.9 ix,
  /usr/bin/python3.9 ix,
  
  # Allow temporary file access
  /tmp/** rw,
  
  # Deny access to sensitive system files
  deny /etc/passwd r,
  deny /etc/shadow r,
  deny /proc/sys/** w,
  deny /sys/** w,
  
  # Allow basic system access
  /proc/*/stat r,
  /proc/*/status r,
  /proc/meminfo r,
  /proc/cpuinfo r,
  
  # Allow library access
  /lib/** r,
  /usr/lib/** r,
  /usr/local/lib/** r,
}
EOF
Subtask 4.3: Load and Test the AppArmor Profile
Load the AppArmor profile:

# Parse and load the profile
sudo apparmor_parser -r /etc/apparmor.d/docker-secure-app

# Verify the profile is loaded
sudo aa-status | grep docker-secure-app
Subtask 4.4: Create a Test Application for AppArmor
Create an application that tries to access restricted resources:

cat > apparmor-test.py << 'EOF'
#!/usr/bin/env python3
import os
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return """
    <h1>AppArmor Security Test</h1>
    <p><a href="/read-passwd">Try to read /etc/passwd</a></p>
    <p><a href="/read-shadow">Try to read /etc/shadow</a></p>
    <p><a href="/write-proc">Try to write to /proc</a></p>
    <p><a href="/list-root">Try to list root directory</a></p>
    """

@app.route('/read-passwd')
def read_passwd():
    try:
        with open('/etc/passwd', 'r') as f:
            content = f.read()
        return f"<pre>{content}</pre>"
    except Exception as e:
        return f"BLOCKED: {str(e)}"

@app.route('/read-shadow')
def read_shadow():
    try:
        with open('/etc/shadow', 'r') as f:
            content = f.read()
        return f"<pre>{content}</pre>"
    except Exception as e:
        return f"BLOCKED: {str(e)}"

@app.route('/write-proc')
def write_proc():
    try:
        with open('/proc/sys/kernel/hostname', 'w') as f:
            f.write('hacked')
        return "SUCCESS: Modified system hostname"
    except Exception as e:
        return f"BLOCKED: {str(e)}"

@app.route('/list-root')
def list_root():
    try:
        files = os.listdir('/')
        return f"<pre>{chr(10).join(files)}</pre>"
    except Exception as e:
        return f"BLOCKED: {str(e)}"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
Subtask 4.5: Build and Test with AppArmor
Create a Dockerfile for AppArmor testing:

cat > Dockerfile.apparmor << 'EOF'
FROM python:3.9-slim

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app
RUN chown appuser:appuser /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY apparmor-test.py .
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 5000

CMD ["python", "apparmor-test.py"]
EOF
Build and run with AppArmor profile:

docker build -f Dockerfile.apparmor -t apparmor-test .

# Run without AppArmor (for comparison)
docker run -d --name no-apparmor -p 5005:5000 apparmor-test

# Run with AppArmor profile
docker run -d --name with-apparmor \
  --security-opt apparmor=docker-secure-app \
  -p 5006:5000 \
  apparmor-test
Subtask 4.6: Compare AppArmor Protection
Test both containers:

echo "Testing container WITHOUT AppArmor:"
curl http://localhost:5005/read-passwd
echo -e "\n"

echo "Testing container WITH AppArmor:"
curl http://localhost:5006/read-passwd
echo -e "\n"

echo "Testing shadow file access:"
curl http://localhost:5005/read-shadow
curl http://localhost:5006/read-shadow
Task 5: Enable Docker Content Trust for Signing Images
Understanding Docker Content Trust
Docker Content Trust provides cryptographic signing of Docker images, ensuring image integrity and authenticity.

Subtask 5.1: Set Up Docker Content Trust Environment
Enable Docker Content Trust:

# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Create a directory for trust data
mkdir -p ~/.docker/trust

# Set up trust server (using Docker Hub's Notary server)
export DOCKER_CONTENT_TRUST_SERVER=https://notary.docker.io
Subtask 5.2: Create a Signing Key
Generate keys for signing:

# Create a root key (you'll be prompted for passphrases)
docker trust key generate mykey

# List the generated keys
docker trust key load mykey.pub --name mykey
Subtask 5.3: Create and Sign a Custom Image
Create a simple application for signing:

cat > signed-app.py << 'EOF'
#!/usr/bin/env python3
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return """
    <h1>Signed Docker Image</h1>
    <p>This image has been cryptographically signed!</p>
    <p>Image integrity verified through Docker Content Trust</p>
    """

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
Create a Dockerfile for the signed image:

cat > Dockerfile.signed << 'EOF'
FROM python:3.9-slim

RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app
RUN chown appuser:appuser /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY signed-app.py .
RUN chown -R appuser:appuser /app

USER appuser

EXPOSE 5000

CMD ["python", "signed-app.py"]
EOF
Subtask 5.4: Build and Push Signed Image
Note: For this lab, we'll demonstrate the signing process. In a real environment, you would push to a registry like Docker Hub.

# Build the image
docker build -f Dockerfile.signed -t localhost:5000/signed-app:v1.0 .

# For demonstration, let's set up a local registry
docker run -d -p 5000:5000 --name registry registry:2

# Push the image (this will be signed due to DOCKER_CONTENT_TRUST=1)
# Note: You would need proper registry credentials for actual signing
docker tag localhost:5000/signed-app:v1.0 localhost:5000/signed-app:latest
Subtask 5.5: Verify Image Signatures
Create a script to demonstrate signature verification:

cat > verify-signature.sh << 'EOF'
#!/bin/bash

echo "=== Docker Content Trust Verification Demo ==="

# Show trust information for an image
echo "1. Checking trust information:"
docker trust inspect --pretty alpine:latest 2>/dev/null || echo "No trust data available for alpine:latest"

# Show the difference between trusted and untrusted pulls
echo -e "\n2. Demonstrating trust enforcement:"

# Disable trust temporarily
export DOCKER_CONTENT_TRUST=0
echo "With DOCKER_CONTENT_TRUST=0 (disabled):"
docker pull alpine:3.14 >/dev/null 2>&1 && echo "✓ Pull succeeded without trust verification"

# Enable trust
export DOCKER_CONTENT_TRUST=1
echo "With DOCKER_CONTENT_TRUST=1 (enabled):"
docker pull alpine:3.14 >/dev/null 2>&1 && echo "✓ Pull succeeded with trust verification" || echo "✗ Pull failed - no trust data"

echo -e "\n3. Trust policy enforcement prevents unsigned images from running"
EOF

chmod +x verify-signature.sh
./verify-signature.sh
Subtask 5.6: Create a Content Trust Policy
Create a comprehensive content trust policy:

cat > content-trust-policy.json << 'EOF'
{
  "default": [
    {
      "type": "signedBy",
      "keyType": "x509",
      "keyPath": "./mykey.pub"
    }
  ],
  "repositories": {
    "library/alpine": [
      {
        "type": "signedBy",
        "keyType": "x509",
        "keyPath": "./docker-official.pub"
      }
    ]
  }
}
EOF
Comprehensive Security Implementation
Subtask 6.1: Create a Fully Secured Container
Now let's combine all security practices into one comprehensive example:

cat > secure-production-app.py << 'EOF'
#!/usr/bin/env python3
import os
import logging
from flask import Flask, request, jsonify
from datetime import datetime

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        "message": "Secure Production Application",
        "user": os.getenv('USER', 'unknown'),
        "uid": os.getuid(),
        "gid": os.getgid(),
        "timestamp": datetime.now().isoformat(),
        "security_features": [
            "Non-root user",
            "Read-only filesystem",
            "Dropped capabilities",
            "AppArmor protection",
            "Signed image"
        ]
    })

@app.route('/health')
def health():
    return jsonify({"status": "healthy", "timestamp": datetime.now().isoformat()})

@app.route('/security-test')
def security_test():
    tests = {}
    
    # Test file system write permissions
    try:
        with open('/tmp/test-write', 'w') as f:
            f.write('test')
        tests['tmp_write'] = 'allowed'
        os.remove('/tmp/test-write')
    except:
        tests['tmp_write'] = 'denied'
    
    try:
        with open('/app/test-write', 'w') as f:
            f.write('test')
        tests['app_write'] = 'allowed'
        os.remove('/app/test-write')
    except:
        tests['app_write'] = 'denied'
    
    # Test sensitive file access
    try:
        with open('/etc/passwd', 'r') as f:
            f.read(10)
        tests['passwd_read'] = 'allowed'
    except:
        tests['passwd_read'] = 'denied'
    
    return jsonify(tests)

if __name__ == '__main__':
    logger.info("Starting secure application")
    app.run(host='0.0.0.0', port=5000)
EOF
Subtask 6.2: Create the Ultimate Secure Dockerfile
cat > Dockerfile.production << 'EOF'
FROM python:3.9-slim

# Install security updates
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    libcap2-bin && \
    rm -rf /var/lib/apt/lists/*

# Create non-root user with specific UID/GID
RUN groupadd -r -g 1001 appuser && \
    useradd -r -u 1001 -g appuser -d /app -s /sbin/nologin appuser

# Set up application directory
WORKDIR /app
RUN chown appuser:appuser /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt && \
    pip cache purge

# Copy application files
COPY secure-production-app.py .
RUN chown -R appuser:appuser /app && \
    chmod 755 /app && \
    chmod 644 /app/*

# Remove unnecessary packages and clean up
RUN apt-get autoremove -y && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# Switch to non-root user
USER appuser

# Set security-focused environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1
ENV PATH="/app:$PATH"

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Use exec form for better signal handling
CMD ["python", "secure-production-app.py"]
EOF
Subtask 6.3: Deploy with All Security Features
Build the production-ready secure image:

docker build -f Dockerfile.production -t secure-production-app .
Run with all security features enabled:

# Stop any existing containers
docker stop $(docker ps -q) 2>/dev/null || true

# Run the fully secured container
docker run -d --name production-secure \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=50m \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt apparmor=docker-secure-app \
  --security-opt no-new-privileges:true \
  --user 1001:1001 \
  --memory=256m \
  --cpus=0.5 \
  --restart=unless-stopped \
  -p 5007:5000 \
  secure-production-app
Subtask 6.4: Verify All Security Measures
Test the comprehensive security implementation:

echo "=== Comprehensive Security Test ==="

echo "1. Application Status:"
curl -s http://localhost:5007/ | python3 -m json.tool

echo -e "\n2. Security Tests:"
curl -s http://localhost:5007/security-test | python3 -m json.tool

echo -e "\n3. Container Security Information:"
docker exec production-secure id
docker exec production-secure capsh --print | head -5

echo -e "\n4. File System Tests:"
docker exec production-secure ls -la /
docker exec production-secure touch /test-file 2>&1 || echo "✓ Root filesystem is read-only"
docker exec production
