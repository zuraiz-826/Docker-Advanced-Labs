Lab 82: Docker and Security - Using Docker Bench for Security Audits
Lab Objectives
By the end of this lab, students will be able to:

• Install and configure Docker Bench for Security on a Docker host system • Execute comprehensive security audits on Docker containers and infrastructure • Analyze and interpret security audit results to identify vulnerabilities • Implement Docker security best practices including user namespaces and seccomp profiles • Apply security recommendations to harden Docker environments • Verify security improvements through re-auditing processes • Understand the importance of continuous security monitoring in containerized environments

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Linux command line operations • Fundamental knowledge of Docker containers and Docker commands • Familiarity with system administration concepts • Understanding of basic security principles • Access to a Linux-based system with sudo privileges

Note: Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click "Start Lab" to begin - no need to build your own virtual machine.

Lab Environment Setup
This lab uses the following technologies: • Docker Engine: Container runtime platform • Docker Bench for Security: Open-source security auditing tool • Ubuntu Linux: Operating system platform • Git: Version control system for downloading tools

All tools used in this lab are open-source and freely available.

Task 1: Install Docker Bench for Security on a Docker Host
Subtask 1.1: Verify Docker Installation
First, let's ensure Docker is properly installed and running on your system.

# Check Docker version
docker --version

# Check Docker service status
sudo systemctl status docker

# Test Docker installation
docker run hello-world
If Docker is not installed, install it using the following commands:

# Update package index
sudo apt update

# Install required packages
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Set up stable repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io

# Add current user to docker group
sudo usermod -aG docker $USER

# Restart session or run
newgrp docker
Subtask 1.2: Download Docker Bench for Security
Docker Bench for Security is available as a GitHub repository. Let's download and prepare it:

# Create a directory for security tools
mkdir -p ~/docker-security
cd ~/docker-security

# Clone Docker Bench for Security repository
git clone https://github.com/docker/docker-bench-security.git

# Navigate to the downloaded directory
cd docker-bench-security

# Make the script executable
chmod +x docker-bench-security.sh

# List the contents to verify download
ls -la
Subtask 1.3: Understand Docker Bench Components
Let's examine what Docker Bench for Security includes:

# View the main script
head -20 docker-bench-security.sh

# Check available test categories
ls tests/

# View documentation
cat README.md | head -30
The tool tests various security aspects including: • Host Configuration: System-level security settings • Docker Daemon Configuration: Docker service security • Docker Daemon Configuration Files: File permissions and ownership • Container Images and Build Files: Image security practices • Container Runtime: Running container security • Docker Security Operations: Operational security measures

Task 2: Run the Audit and Analyze the Results
Subtask 2.1: Execute Initial Security Audit
Now let's run our first comprehensive security audit:

# Run Docker Bench for Security with detailed output
sudo ./docker-bench-security.sh

# Run with specific output format (optional)
sudo ./docker-bench-security.sh -l /tmp/docker-bench-results.log

# Run only specific tests (example: host configuration)
sudo ./docker-bench-security.sh -c host_configuration
Subtask 2.2: Create Test Containers for Comprehensive Analysis
To get meaningful results, let's create some containers to audit:

# Create a simple web server container
docker run -d --name test-nginx -p 8080:80 nginx:latest

# Create a container with some security issues (for demonstration)
docker run -d --name insecure-container --privileged -v /:/host ubuntu:latest sleep 3600

# Create a database container
docker run -d --name test-mysql -e MYSQL_ROOT_PASSWORD=password123 mysql:8.0

# List running containers
docker ps
Subtask 2.3: Run Comprehensive Audit with Containers
# Run full audit with containers running
sudo ./docker-bench-security.sh > audit-results.txt 2>&1

# Display results with color coding
sudo ./docker-bench-security.sh | tee audit-output.log
Subtask 2.4: Analyze Audit Results
Let's examine the audit results systematically:

# View the complete results
cat audit-results.txt

# Filter for WARN (warning) messages
grep "WARN" audit-results.txt

# Filter for FAIL (failed) checks
grep "FAIL" audit-results.txt

# Filter for PASS (passed) checks
grep "PASS" audit-results.txt

# Count different result types
echo "Passed checks: $(grep -c "PASS" audit-results.txt)"
echo "Failed checks: $(grep -c "FAIL" audit-results.txt)"
echo "Warning checks: $(grep -c "WARN" audit-results.txt)"
Subtask 2.5: Generate Detailed Report
Create a structured analysis of the results:

# Create a summary report
cat > security-analysis.md << 'EOF'
# Docker Security Audit Report

## Executive Summary
Date: $(date)
Audit Tool: Docker Bench for Security

## Results Summary
- Total Checks: $(grep -E "(PASS|FAIL|WARN)" audit-results.txt | wc -l)
- Passed: $(grep -c "PASS" audit-results.txt)
- Failed: $(grep -c "FAIL" audit-results.txt)
- Warnings: $(grep -c "WARN" audit-results.txt)

## Critical Issues (FAIL)
$(grep "FAIL" audit-results.txt)

## Warnings (WARN)
$(grep "WARN" audit-results.txt)
EOF

# Process the template
eval "echo \"$(cat security-analysis.md)\"" > final-security-report.md

# View the report
cat final-security-report.md
Task 3: Address Security Recommendations
Subtask 3.1: Fix Host Configuration Issues
Based on common audit findings, let's address host-level security issues:

# Create separate partition for Docker (if not already done)
# Note: This is typically done during system setup
sudo mkdir -p /var/lib/docker

# Set proper permissions on Docker socket
sudo chmod 660 /var/run/docker.sock
sudo chown root:docker /var/run/docker.sock

# Configure Docker daemon with security options
sudo mkdir -p /etc/docker

# Create daemon configuration file
sudo tee /etc/docker/daemon.json << 'EOF'
{
  "icc": false,
  "userns-remap": "default",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true
}
EOF
Subtask 3.2: Configure Docker Daemon Security
# Restart Docker daemon to apply configuration
sudo systemctl restart docker

# Verify daemon configuration
docker info | grep -A 10 "Security Options"

# Check if user namespace remapping is active
docker info | grep "userns"

# Verify logging configuration
docker info | grep -A 5 "Logging Driver"
Subtask 3.3: Secure Container Images
Let's create and use more secure container images:

# Remove insecure containers
docker stop insecure-container test-mysql test-nginx
docker rm insecure-container test-mysql test-nginx

# Create a Dockerfile with security best practices
cat > Dockerfile.secure << 'EOF'
FROM ubuntu:20.04

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

# Install only necessary packages
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    curl \
    ca-certificates && \
    rm -rf /var/lib/apt/lists/*

# Set working directory
WORKDIR /app

# Copy application files (example)
COPY --chown=appuser:appuser . /app

# Switch to non-root user
USER appuser

# Expose port
EXPOSE 8080

# Use exec form for CMD
CMD ["sleep", "3600"]
EOF

# Build secure image
docker build -f Dockerfile.secure -t secure-app:latest .
Subtask 3.4: Implement Container Runtime Security
# Run containers with security best practices
docker run -d \
  --name secure-nginx \
  --read-only \
  --tmpfs /tmp \
  --tmpfs /var/run \
  --user 1000:1000 \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  -p 8080:80 \
  nginx:latest

# Run container with resource limits
docker run -d \
  --name limited-container \
  --memory 256m \
  --cpus 0.5 \
  --pids-limit 100 \
  --user 1000:1000 \
  --read-only \
  --security-opt no-new-privileges:true \
  ubuntu:20.04 sleep 3600

# Verify container security settings
docker inspect secure-nginx | grep -A 10 "SecurityOpt"
docker inspect limited-container | grep -A 5 "Memory"
Task 4: Implement Advanced Security Features
Subtask 4.1: Configure User Namespaces
User namespaces provide isolation between container users and host users:

# Check if user namespaces are enabled
docker info | grep "userns"

# If not enabled, configure user namespace mapping
sudo tee -a /etc/subuid << 'EOF'
dockremap:165536:65536
EOF

sudo tee -a /etc/subgid << 'EOF'
dockremap:165536:65536
EOF

# Restart Docker daemon
sudo systemctl restart docker

# Verify user namespace configuration
docker info | grep -A 3 "Security Options"

# Test with a container
docker run --rm -it --name test-userns ubuntu:20.04 id
Subtask 4.2: Implement Seccomp Profiles
Seccomp (Secure Computing Mode) restricts system calls available to containers:

# Create a custom seccomp profile
mkdir -p ~/docker-security/seccomp-profiles

cat > ~/docker-security/seccomp-profiles/default-seccomp.json << 'EOF'
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
      "names": [
        "accept",
        "accept4",
        "access",
        "alarm",
        "bind",
        "brk",
        "chdir",
        "chmod",
        "chown",
        "close",
        "connect",
        "dup",
        "dup2",
        "execve",
        "exit",
        "exit_group",
        "fchdir",
        "fchmod",
        "fchown",
        "fcntl",
        "fork",
        "fstat",
        "getcwd",
        "getdents",
        "getpid",
        "getppid",
        "getuid",
        "listen",
        "lseek",
        "lstat",
        "mkdir",
        "mmap",
        "munmap",
        "open",
        "openat",
        "read",
        "readlink",
        "recv",
        "recvfrom",
        "rename",
        "rmdir",
        "send",
        "sendto",
        "socket",
        "stat",
        "unlink",
        "wait4",
        "write"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
EOF

# Run container with custom seccomp profile
docker run --rm -it \
  --security-opt seccomp=~/docker-security/seccomp-profiles/default-seccomp.json \
  --name seccomp-test \
  ubuntu:20.04 /bin/bash -c "echo 'Seccomp profile applied successfully'"

# Test with default seccomp profile
docker run --rm -it \
  --security-opt seccomp=unconfined \
  --name no-seccomp \
  ubuntu:20.04 /bin/bash -c "echo 'No seccomp restrictions'"
Subtask 4.3: Configure AppArmor Profiles
AppArmor provides additional access control:

# Check if AppArmor is available
sudo aa-status

# Create a custom AppArmor profile for Docker
sudo tee /etc/apparmor.d/docker-custom << 'EOF'
#include <tunables/global>

profile docker-custom flags=(attach_disconnected,mediate_deleted) {
  #include <abstractions/base>
  
  network,
  capability,
  file,
  umount,
  
  deny @{PROC}/* w,
  deny /sys/[^f]** wklx,
  deny /sys/f[^s]** wklx,
  deny /sys/fs/[^c]** wklx,
  deny /sys/fs/c[^g]** wklx,
  deny /sys/fs/cg[^r]** wklx,
  deny /sys/firmware/** rwklx,
  deny /sys/kernel/security/** rwklx,
}
EOF

# Load the AppArmor profile
sudo apparmor_parser -r /etc/apparmor.d/docker-custom

# Run container with AppArmor profile
docker run --rm -it \
  --security-opt apparmor=docker-custom \
  --name apparmor-test \
  ubuntu:20.04 /bin/bash -c "echo 'AppArmor profile applied'"
Subtask 4.4: Implement Content Trust
Docker Content Trust ensures image integrity:

# Enable Docker Content Trust
export DOCKER_CONTENT_TRUST=1

# Generate delegation keys (for demonstration)
docker trust key generate mykey

# Try to pull an unsigned image (should fail)
docker pull alpine:latest || echo "Content trust prevented unsigned image pull"

# Disable content trust for now
export DOCKER_CONTENT_TRUST=0

# Pull image normally
docker pull alpine:latest
Task 5: Re-run Audit to Verify Security Improvements
Subtask 5.1: Execute Post-Hardening Audit
# Navigate back to Docker Bench directory
cd ~/docker-security/docker-bench-security

# Run the audit again
sudo ./docker-bench-security.sh > post-hardening-audit.txt 2>&1

# Display results
sudo ./docker-bench-security.sh | tee post-hardening-output.log
Subtask 5.2: Compare Before and After Results
# Compare the two audit results
echo "=== BEFORE HARDENING ==="
echo "Passed: $(grep -c "PASS" audit-results.txt)"
echo "Failed: $(grep -c "FAIL" audit-results.txt)"
echo "Warnings: $(grep -c "WARN" audit-results.txt)"

echo ""
echo "=== AFTER HARDENING ==="
echo "Passed: $(grep -c "PASS" post-hardening-audit.txt)"
echo "Failed: $(grep -c "FAIL" post-hardening-audit.txt)"
echo "Warnings: $(grep -c "WARN" post-hardening-audit.txt)"

# Show improvement
before_fails=$(grep -c "FAIL" audit-results.txt)
after_fails=$(grep -c "FAIL" post-hardening-audit.txt)
improvement=$((before_fails - after_fails))

echo ""
echo "=== IMPROVEMENT ==="
echo "Reduced failures by: $improvement"
echo "Improvement percentage: $(echo "scale=2; $improvement * 100 / $before_fails" | bc)%"
Subtask 5.3: Generate Final Security Report
# Create comprehensive final report
cat > final-security-assessment.md << 'EOF'
# Docker Security Assessment - Final Report

## Overview
This report summarizes the security improvements made to the Docker environment.

## Initial Assessment
- Failed Checks: $(grep -c "FAIL" audit-results.txt)
- Warning Checks: $(grep -c "WARN" audit-results.txt)
- Passed Checks: $(grep -c "PASS" audit-results.txt)

## Post-Hardening Assessment
- Failed Checks: $(grep -c "FAIL" post-hardening-audit.txt)
- Warning Checks: $(grep -c "WARN" post-hardening-audit.txt)
- Passed Checks: $(grep -c "PASS" post-hardening-audit.txt)

## Security Improvements Implemented
1. **Docker Daemon Configuration**
   - Enabled user namespace remapping
   - Configured logging limits
   - Disabled inter-container communication
   - Enabled live restore

2. **Container Runtime Security**
   - Implemented read-only containers
   - Applied capability dropping
   - Used non-root users
   - Applied resource limits

3. **Advanced Security Features**
   - Custom seccomp profiles
   - AppArmor profiles
   - Content trust awareness

## Remaining Issues
$(grep "FAIL" post-hardening-audit.txt | head -10)

## Recommendations
1. Continue monitoring with regular security audits
2. Implement image scanning in CI/CD pipeline
3. Regular security updates for base images
4. Network segmentation for container communication
5. Implement secrets management solution

## Conclusion
The security posture has been significantly improved with $(echo "scale=0; $(grep -c "FAIL" audit-results.txt) - $(grep -c "FAIL" post-hardening-audit.txt)" | bc) fewer failed security checks.
EOF

# Process and display the final report
eval "echo \"$(cat final-security-assessment.md)\"" > processed-final-report.md
cat processed-final-report.md
Subtask 5.4: Create Monitoring Script
Let's create a script for ongoing security monitoring:

# Create monitoring script
cat > ~/docker-security/security-monitor.sh << 'EOF'
#!/bin/bash

# Docker Security Monitoring Script
REPORT_DIR="/tmp/docker-security-reports"
DATE=$(date +%Y%m%d_%H%M%S)

# Create report directory
mkdir -p $REPORT_DIR

# Run Docker Bench for Security
echo "Running Docker security audit..."
cd ~/docker-security/docker-bench-security
sudo ./docker-bench-security.sh > $REPORT_DIR/audit_$DATE.txt 2>&1

# Generate summary
FAILS=$(grep -c "FAIL" $REPORT_DIR/audit_$DATE.txt)
WARNS=$(grep -c "WARN" $REPORT_DIR/audit_$DATE.txt)
PASSES=$(grep -c "PASS" $REPORT_DIR/audit_$DATE.txt)

echo "Security Audit Complete - $DATE"
echo "Results saved to: $REPORT_DIR/audit_$DATE.txt"
echo "Summary:"
echo "  Passed: $PASSES"
echo "  Failed: $FAILS"
echo "  Warnings: $WARNS"

# Alert if critical issues found
if [ $FAILS -gt 10 ]; then
    echo "WARNING: High number of failed security checks detected!"
    echo "Critical issues:"
    grep "FAIL" $REPORT_DIR/audit_$DATE.txt | head -5
fi
EOF

# Make script executable
chmod +x ~/docker-security/security-monitor.sh

# Test the monitoring script
~/docker-security/security-monitor.sh
Troubleshooting Common Issues
Issue 1: Docker Bench Script Permission Denied
# Solution: Ensure proper permissions
chmod +x docker-bench-security.sh
sudo chown $USER:$USER docker-bench-security.sh
Issue 2: User Namespace Configuration Issues
# Check if user namespaces are supported
grep CONFIG_USER_NS /boot/config-$(uname -r)

# Verify subuid/subgid configuration
cat /etc/subuid
cat /etc/subgid

# Reset Docker daemon if needed
sudo systemctl stop docker
sudo rm -rf /var/lib/docker
sudo systemctl start docker
Issue 3: Seccomp Profile Errors
# Validate JSON syntax
python3 -m json.tool ~/docker-security/seccomp-profiles/default-seccomp.json

# Check kernel seccomp support
grep CONFIG_SECCOMP /boot/config-$(uname -r)
Issue 4: AppArmor Profile Issues
# Check AppArmor status
sudo aa-status

# Reload profile if needed
sudo apparmor_parser -r /etc/apparmor.d/docker-custom

# Check for syntax errors
sudo apparmor_parser -Q /etc/apparmor.d/docker-custom
Best Practices Summary
Container Security Best Practices
Use Official Base Images: Always start with official, minimal base images
Run as Non-Root: Create and use non-privileged users in containers
Minimize Attack Surface: Install only necessary packages and remove package managers
Use Read-Only Containers: Mount root filesystem as read-only when possible
Apply Resource Limits: Set memory, CPU, and process limits
Drop Capabilities: Remove unnecessary Linux capabilities
Use Security Profiles: Apply seccomp, AppArmor, or SELinux profiles
Docker Daemon Security
Enable User Namespaces: Isolate container users from host users
Configure Logging: Set up centralized logging with rotation
Disable Inter-Container Communication: Prevent unnecessary container networking
Use TLS: Secure Docker daemon communication
Regular Updates: Keep Docker engine updated
Monitor and Audit: Implement continuous security monitoring
Operational Security
Image Scanning: Scan images for vulnerabilities before deployment
Secrets Management: Use dedicated secrets management solutions
Network Segmentation: Implement proper network isolation
Access Control: Use RBAC and principle of least privilege
Compliance: Regular compliance checks and documentation
Incident Response: Have procedures for security incidents
Conclusion
In this comprehensive lab, you have successfully:

• Installed and configured Docker Bench for Security, a powerful open-source tool for auditing Docker environments • Performed thorough security audits on Docker containers and infrastructure, learning to identify and analyze security vulnerabilities • Implemented critical security improvements including user namespaces, seccomp profiles, and container hardening techniques • Applied Docker security best practices such as running containers as non-root users, implementing resource limits, and using read-only filesystems • Configured advanced security features including AppArmor profiles and content trust mechanisms • Verified security improvements through comparative auditing, demonstrating measurable security enhancements • Created monitoring and reporting systems for ongoing security assessment and compliance

This lab demonstrates the critical importance of proactive security measures in containerized environments. Docker Bench for Security provides an automated way to identify security gaps and ensure compliance with industry best practices. The skills learned here are essential for:

DevSecOps Implementation: Integrating security into CI/CD pipelines
Compliance Requirements: Meeting regulatory and organizational security standards
Risk Management: Reducing attack surface and potential security incidents
Operational Excellence: Maintaining secure, reliable containerized applications
The security improvements implemented in this lab significantly reduce the attack surface of your Docker environment and provide a solid foundation for secure container operations. Regular security audits using tools like Docker Bench for Security should be part of your ongoing security strategy, ensuring that your containerized applications remain secure as they evolve and scale.

Remember that security is an ongoing process, not a one-time implementation. Continue to monitor, audit, and improve your Docker security posture as new threats emerge and best practices evolve. The foundation you've built in this lab will serve as a strong starting point for maintaining secure containerized environments in production.
