Lab 27: Docker Health Checks - Implementing Health Checks in Containers
Objectives
By the end of this lab, you will be able to:

Understand the importance of Docker health checks for container reliability
Implement health checks in Dockerfiles using the HEALTHCHECK directive
Configure containers to respond appropriately when health checks fail
Test and validate health check functionality in various scenarios
Use Docker commands to monitor and inspect container health status
Implement automatic restart policies for unhealthy containers
Troubleshoot common health check issues and optimize container reliability
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with Linux command line operations
Knowledge of basic networking concepts
Understanding of web applications and HTTP status codes
Experience with text editors (nano, vim, or similar)
Note: Al Nafi provides ready-to-use Linux-based cloud machines with Docker pre-installed. Simply click "Start Lab" to begin - no need to build your own VM or install Docker manually.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with:

Docker Engine (latest stable version)
Docker Compose
Text editors (nano, vim)
curl and wget utilities
All necessary networking tools
Task 1: Add a Health Check to a Dockerfile Using the HEALTHCHECK Directive
Subtask 1.1: Create a Simple Web Application
First, let's create a basic web application that we can monitor with health checks.

Create a new directory for our project:
mkdir ~/docker-healthcheck-lab
cd ~/docker-healthcheck-lab
Create a simple Python web application:
nano app.py
Add the following Python code:
from flask import Flask, jsonify
import os
import time
import random

app = Flask(__name__)

# Global variable to simulate application state
app_healthy = True
start_time = time.time()

@app.route('/')
def home():
    return jsonify({
        "message": "Hello from Docker Health Check Lab!",
        "status": "running",
        "uptime": int(time.time() - start_time)
    })

@app.route('/health')
def health_check():
    global app_healthy
    
    # Simulate random failures for testing
    if random.random() < 0.1:  # 10% chance of failure
        app_healthy = False
    
    if app_healthy:
        return jsonify({
            "status": "healthy",
            "timestamp": int(time.time()),
            "uptime": int(time.time() - start_time)
        }), 200
    else:
        return jsonify({
            "status": "unhealthy",
            "timestamp": int(time.time()),
            "error": "Application is experiencing issues"
        }), 500

@app.route('/make-unhealthy')
def make_unhealthy():
    global app_healthy
    app_healthy = False
    return jsonify({"message": "Application marked as unhealthy"})

@app.route('/make-healthy')
def make_healthy():
    global app_healthy
    app_healthy = True
    return jsonify({"message": "Application marked as healthy"})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
Create a requirements file for Python dependencies:
nano requirements.txt
Add the following content:
Flask==2.3.3
Subtask 1.2: Create a Dockerfile with Health Check
Create a Dockerfile with health check configuration:
nano Dockerfile
Add the following Dockerfile content:
# Use Python 3.9 slim image as base
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy requirements and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .

# Expose port 5000
EXPOSE 5000

# Add health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Install curl for health check
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Run the application
CMD ["python", "app.py"]
Subtask 1.3: Build and Test the Basic Health Check
Build the Docker image:
docker build -t healthcheck-app:v1 .
Run the container to test the health check:
docker run -d --name healthcheck-test -p 5000:5000 healthcheck-app:v1
Wait a few moments and check the container status:
docker ps
Test the application endpoints:
# Test main endpoint
curl http://localhost:5000/

# Test health endpoint
curl http://localhost:5000/health
Task 2: Configure a Container to Stop if the Health Check Fails
Subtask 2.1: Create an Advanced Dockerfile with Failure Handling
Create an improved Dockerfile with better health check configuration:
nano Dockerfile.advanced
Add the following content:
FROM python:3.9-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && \
    apt-get install -y curl procps && \
    rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .
COPY healthcheck.sh .

# Make health check script executable
RUN chmod +x healthcheck.sh

# Expose port
EXPOSE 5000

# Configure health check with stricter parameters
HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=2 \
    CMD ./healthcheck.sh

# Run application
CMD ["python", "app.py"]
Create a custom health check script:
nano healthcheck.sh
Add the following health check script:
#!/bin/bash

# Health check script for Docker container
set -e

# Check if the application is responding
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:5000/health)

# Check if HTTP response is 200 (OK)
if [ "$HTTP_CODE" -eq 200 ]; then
    echo "Health check passed - HTTP $HTTP_CODE"
    exit 0
else
    echo "Health check failed - HTTP $HTTP_CODE"
    exit 1
fi
Subtask 2.2: Build and Test the Advanced Health Check
Build the advanced image:
docker build -f Dockerfile.advanced -t healthcheck-app:advanced .
Stop the previous container:
docker stop healthcheck-test
docker rm healthcheck-test
Run the new container with restart policy:
docker run -d --name healthcheck-advanced --restart=on-failure:3 -p 5001:5000 healthcheck-app:advanced
Task 3: Test the Health Check by Running Containers with Unhealthy States
Subtask 3.1: Simulate Application Failures
First, verify the container is healthy:
docker ps
curl http://localhost:5001/health
Make the application unhealthy:
curl http://localhost:5001/make-unhealthy
Check the health status immediately:
curl http://localhost:5001/health
Monitor the container status over time:
# Check every 10 seconds for 2 minutes
for i in {1..12}; do
    echo "Check $i:"
    docker ps --format "table {{.Names}}\t{{.Status}}"
    sleep 10
done
Subtask 3.2: Create a Container That Fails Health Checks
Create a deliberately failing application:
nano failing-app.py
Add the following code:
from flask import Flask, jsonify
import time

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({"message": "This app will fail health checks"})

@app.route('/health')
def health_check():
    # Always return unhealthy status
    return jsonify({
        "status": "unhealthy",
        "error": "Deliberately failing for testing"
    }), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
Create a Dockerfile for the failing app:
nano Dockerfile.failing
Add the following content:
FROM python:3.9-slim

WORKDIR /app

RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY failing-app.py app.py

EXPOSE 5000

# Health check that will always fail
HEALTHCHECK --interval=10s --timeout=3s --start-period=5s --retries=2 \
    CMD curl -f http://localhost:5000/health || exit 1

CMD ["python", "app.py"]
Build and run the failing container:
docker build -f Dockerfile.failing -t failing-app .
docker run -d --name failing-container -p 5002:5000 failing-app
Monitor the failing container:
# Watch the container status change
watch -n 5 'docker ps --format "table {{.Names}}\t{{.Status}}"'
Task 4: Use docker inspect to View the Health Status of Running Containers
Subtask 4.1: Inspect Container Health Details
Get detailed health information for the healthy container:
docker inspect healthcheck-advanced --format='{{json .State.Health}}' | python3 -m json.tool
Get health information for the failing container:
docker inspect failing-container --format='{{json .State.Health}}' | python3 -m json.tool
Create a script to monitor health status:
nano monitor-health.sh
Add the following monitoring script:
#!/bin/bash

echo "Docker Container Health Monitor"
echo "==============================="

containers=("healthcheck-advanced" "failing-container")

for container in "${containers[@]}"; do
    if docker ps -q -f name="$container" > /dev/null 2>&1; then
        echo ""
        echo "Container: $container"
        echo "Status: $(docker inspect $container --format='{{.State.Status}}')"
        echo "Health Status: $(docker inspect $container --format='{{.State.Health.Status}}')"
        echo "Failed Health Checks: $(docker inspect $container --format='{{len .State.Health.Log}}')"
        
        # Show last health check result
        echo "Last Health Check:"
        docker inspect $container --format='{{range .State.Health.Log}}{{.Start}} - {{.ExitCode}} - {{.Output}}{{end}}' | tail -1
    else
        echo "Container $container is not running"
    fi
done
Make the script executable and run it:
chmod +x monitor-health.sh
./monitor-health.sh
Subtask 4.2: Create a Comprehensive Health Monitoring Dashboard
Create an advanced monitoring script:
nano health-dashboard.sh
Add the following comprehensive monitoring script:
#!/bin/bash

# Function to get container health info
get_health_info() {
    local container_name=$1
    
    if docker ps -q -f name="$container_name" > /dev/null 2>&1; then
        local status=$(docker inspect $container_name --format='{{.State.Status}}')
        local health_status=$(docker inspect $container_name --format='{{.State.Health.Status}}')
        local failing_streak=$(docker inspect $container_name --format='{{.State.Health.FailingStreak}}')
        
        echo "Container: $container_name"
        echo "  Status: $status"
        echo "  Health: $health_status"
        echo "  Failing Streak: $failing_streak"
        
        # Show recent health check logs
        echo "  Recent Health Checks:"
        docker inspect $container_name --format='{{range .State.Health.Log}}    {{.Start}} | Exit: {{.ExitCode}} | {{.Output}}{{end}}' | tail -3
        echo ""
    else
        echo "Container: $container_name - NOT RUNNING"
        echo ""
    fi
}

# Clear screen and show header
clear
echo "=========================================="
echo "    Docker Health Check Dashboard"
echo "=========================================="
echo "Timestamp: $(date)"
echo ""

# Monitor all containers with health checks
for container in $(docker ps --format '{{.Names}}'); do
    # Check if container has health check configured
    if docker inspect $container --format='{{.Config.Healthcheck}}' | grep -q 'Test'; then
        get_health_info $container
    fi
done

echo "=========================================="
Run the dashboard:
chmod +x health-dashboard.sh
./health-dashboard.sh
Task 5: Demonstrate Automatic Restart of Unhealthy Containers Using the --restart Policy
Subtask 5.1: Configure Different Restart Policies
Create containers with different restart policies:
# Container with no restart policy
docker run -d --name no-restart-policy -p 5003:5000 healthcheck-app:advanced

# Container with always restart policy
docker run -d --name always-restart --restart=always -p 5004:5000 healthcheck-app:advanced

# Container with restart on failure with limit
docker run -d --name restart-on-failure --restart=on-failure:5 -p 5005:5000 healthcheck-app:advanced

# Container with restart unless stopped
docker run -d --name restart-unless-stopped --restart=unless-stopped -p 5006:5000 healthcheck-app:advanced
Create a test script to simulate failures:
nano test-restart-policies.sh
Add the following test script:
#!/bin/bash

echo "Testing Docker Restart Policies with Health Checks"
echo "=================================================="

containers=("no-restart-policy" "always-restart" "restart-on-failure" "restart-unless-stopped")
ports=(5003 5004 5005 5006)

# Function to make container unhealthy
make_unhealthy() {
    local port=$1
    local container=$2
    echo "Making $container unhealthy..."
    curl -s http://localhost:$port/make-unhealthy > /dev/null
}

# Function to check container status
check_status() {
    local container=$1
    local status=$(docker inspect $container --format='{{.State.Status}}' 2>/dev/null || echo "not found")
    local health=$(docker inspect $container --format='{{.State.Health.Status}}' 2>/dev/null || echo "no health check")
    local restart_count=$(docker inspect $container --format='{{.RestartCount}}' 2>/dev/null || echo "0")
    
    echo "$container: Status=$status, Health=$health, Restarts=$restart_count"
}

# Make all containers unhealthy
echo "Step 1: Making all containers unhealthy..."
for i in "${!containers[@]}"; do
    make_unhealthy ${ports[$i]} ${containers[$i]}
done

echo ""
echo "Step 2: Monitoring container behavior over time..."

# Monitor for 3 minutes
for minute in {1..3}; do
    echo ""
    echo "Minute $minute:"
    echo "----------"
    for container in "${containers[@]}"; do
        check_status $container
    done
    sleep 60
done

echo ""
echo "Test completed!"
Run the restart policy test:
chmod +x test-restart-policies.sh
./test-restart-policies.sh
Subtask 5.2: Analyze Restart Behavior
Create a detailed analysis script:
nano analyze-restarts.sh
Add the following analysis script:
#!/bin/bash

echo "Docker Restart Policy Analysis"
echo "=============================="

containers=("no-restart-policy" "always-restart" "restart-on-failure" "restart-unless-stopped")

for container in "${containers[@]}"; do
    echo ""
    echo "Container: $container"
    echo "Restart Policy: $(docker inspect $container --format='{{.HostConfig.RestartPolicy.Name}}:{{.HostConfig.RestartPolicy.MaximumRetryCount}}')"
    echo "Current Status: $(docker inspect $container --format='{{.State.Status}}')"
    echo "Health Status: $(docker inspect $container --format='{{.State.Health.Status}}')"
    echo "Restart Count: $(docker inspect $container --format='{{.RestartCount}}')"
    echo "Started At: $(docker inspect $container --format='{{.State.StartedAt}}')"
    
    # Show restart history if available
    echo "Exit Code: $(docker inspect $container --format='{{.State.ExitCode}}')"
    echo "----------------------------------------"
done

echo ""
echo "Summary of Restart Policies:"
echo "- no-restart-policy: Container stops when unhealthy, no restart"
echo "- always-restart: Container restarts regardless of exit code"
echo "- restart-on-failure: Container restarts only on non-zero exit codes"
echo "- restart-unless-stopped: Container restarts unless manually stopped"
Run the analysis:
chmod +x analyze-restarts.sh
./analyze-restarts.sh
Subtask 5.3: Create a Production-Ready Health Check Configuration
Create a production-ready Dockerfile:
nano Dockerfile.production
Add the following production configuration:
FROM python:3.9-slim

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app

# Install system dependencies
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl && \
    rm -rf /var/lib/apt/lists/*

# Copy and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application files
COPY app.py .
COPY production-healthcheck.sh ./healthcheck.sh

# Set proper permissions
RUN chmod +x healthcheck.sh && \
    chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

EXPOSE 5000

# Production health check configuration
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD ./healthcheck.sh

CMD ["python", "app.py"]
Create a production health check script:
nano production-healthcheck.sh
Add the following production health check:
#!/bin/bash

# Production health check script
set -e

# Configuration
HEALTH_URL="http://localhost:5000/health"
MAX_RESPONSE_TIME=5
LOG_FILE="/tmp/healthcheck.log"

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" >> "$LOG_FILE"
}

# Perform health check
start_time=$(date +%s)
HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" --max-time $MAX_RESPONSE_TIME "$HEALTH_URL")
end_time=$(date +%s)
response_time=$((end_time - start_time))

# Check results
if [ "$HTTP_CODE" -eq 200 ] && [ "$response_time" -le "$MAX_RESPONSE_TIME" ]; then
    log_message "Health check PASSED - HTTP $HTTP_CODE, Response time: ${response_time}s"
    exit 0
else
    log_message "Health check FAILED - HTTP $HTTP_CODE, Response time: ${response_time}s"
    exit 1
fi
Build and test the production container:
docker build -f Dockerfile.production -t healthcheck-app:production .
docker run -d --name production-app --restart=unless-stopped -p 5007:5000 healthcheck-app:production
Troubleshooting Common Issues
Issue 1: Health Check Always Fails
Symptoms: Container constantly restarts, health check never passes

Solutions:

Check if the application is actually listening on the specified port
Verify the health check URL is accessible
Increase the --start-period to allow more time for application startup
Check application logs: docker logs container-name
Issue 2: Health Check Takes Too Long
Symptoms: Health checks timeout frequently

Solutions:

Increase the --timeout value
Optimize the health check endpoint
Use a simpler health check (e.g., TCP check instead of HTTP)
Issue 3: Container Doesn't Restart When Unhealthy
Symptoms: Container becomes unhealthy but doesn't restart

Solutions:

Verify restart policy is configured: docker inspect container-name --format='{{.HostConfig.RestartPolicy}}'
Check if the health check is actually failing (exit code 1)
Ensure the container process exits when unhealthy
Cleanup
Clean up all containers and images created during this lab:

# Stop all containers
docker stop $(docker ps -q)

# Remove all containers
docker rm $(docker ps -aq)

# Remove images (optional)
docker rmi healthcheck-app:v1 healthcheck-app:advanced failing-app healthcheck-app:production

# Clean up files
cd ~
rm -rf docker-healthcheck-lab
Conclusion
In this comprehensive lab, you have successfully:

Implemented Docker Health Checks: You learned how to add health checks to Dockerfiles using the HEALTHCHECK directive with various configuration options including interval, timeout, start-period, and retries.

Configured Failure Handling: You created containers that respond appropriately to health check failures, including automatic stopping and restarting based on health status.

Tested Health Check Scenarios: You simulated both healthy and unhealthy application states to understand how Docker responds to different health conditions.

Monitored Container Health: You used docker inspect and created custom monitoring scripts to view detailed health status information and track health check history.

Implemented Restart Policies: You demonstrated various restart policies (no restart, always, on-failure, unless-stopped) and observed how they interact with health checks to provide automatic recovery.

Why This Matters:

Docker health checks are crucial for building resilient, production-ready containerized applications. They enable:

Automatic Recovery: Containers can automatically restart when they become unhealthy
Load Balancer Integration: Load balancers can route traffic away from unhealthy containers
Monitoring and Alerting: Health status provides valuable metrics for monitoring systems
Improved Reliability: Applications can self-heal without manual intervention
Best Practices Learned:

Use appropriate intervals and timeouts for your application's characteristics
Implement meaningful health check endpoints that verify critical application functionality
Choose restart policies that match your application's requirements
Monitor health check logs to identify patterns and optimize configuration
Use non-root users in production containers for security
This knowledge is essential for the Docker Certified Associate (DCA) certification and for building robust containerized applications in production environments.
