Lab 10: Debugging Docker Containers
Objectives
By the end of this lab, you will be able to:

Inspect and analyze Docker container logs using docker logs
Execute commands inside running containers with docker exec
Troubleshoot networking issues by examining container network configurations
Attach to running containers using docker attach
Identify and resolve common Docker container issues using built-in debugging tools
Apply systematic debugging approaches to containerized applications
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with Linux command line operations
Knowledge of basic networking concepts
Understanding of log file analysis
Previous experience with Docker basic commands (docker run, docker ps, docker stop)
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker already installed. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine installed
Pre-configured user with Docker permissions
Sample applications for debugging exercises
Network tools for troubleshooting
Task 1: Inspect Container Logs with docker logs
Subtask 1.1: Create a Container with Logging Issues
First, let's create a container that generates various types of logs to practice debugging.

Create a simple web application with intentional issues:
# Create a directory for our test application
mkdir ~/debug-lab
cd ~/debug-lab

# Create a simple Python web application with logging
cat > app.py << 'EOF'
import time
import sys
import logging
from http.server import HTTPServer, BaseHTTPRequestHandler

# Configure logging
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')
logger = logging.getLogger(__name__)

class SimpleHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        logger.info(f"Received GET request for {self.path}")
        
        if self.path == '/':
            self.send_response(200)
            self.send_header('Content-type', 'text/html')
            self.end_headers()
            self.wfile.write(b'<h1>Debug Lab Application</h1>')
        elif self.path == '/error':
            logger.error("Intentional error endpoint accessed")
            self.send_response(500)
            self.end_headers()
            self.wfile.write(b'Internal Server Error')
        elif self.path == '/slow':
            logger.warning("Slow endpoint accessed - simulating delay")
            time.sleep(5)
            self.send_response(200)
            self.end_headers()
            self.wfile.write(b'Slow response completed')
        else:
            logger.warning(f"404 - Path not found: {self.path}")
            self.send_response(404)
            self.end_headers()
            self.wfile.write(b'Not Found')

if __name__ == '__main__':
    server = HTTPServer(('0.0.0.0', 8080), SimpleHandler)
    logger.info("Starting server on port 8080")
    try:
        server.serve_forever()
    except KeyboardInterrupt:
        logger.info("Server stopped")
        server.server_close()
EOF
Create a Dockerfile for the application:
cat > Dockerfile << 'EOF'
FROM python:3.9-slim
WORKDIR /app
COPY app.py .
EXPOSE 8080
CMD ["python", "app.py"]
EOF
Build and run the container:
# Build the image
docker build -t debug-app .

# Run the container in detached mode
docker run -d --name debug-container -p 8080:8080 debug-app
Subtask 1.2: Basic Log Inspection
View real-time logs:
# View logs in real-time (similar to tail -f)
docker logs -f debug-container
In a new terminal, generate some log entries:
# Generate different types of requests to create logs
curl http://localhost:8080/
curl http://localhost:8080/error
curl http://localhost:8080/nonexistent
curl http://localhost:8080/slow &
View logs with timestamps:
# Stop the real-time log viewing (Ctrl+C) and view logs with timestamps
docker logs -t debug-container
View recent logs only:
# View only the last 10 log entries
docker logs --tail 10 debug-container

# View logs from the last 5 minutes
docker logs --since 5m debug-container
Subtask 1.3: Advanced Log Analysis
Filter logs by time range:
# View logs from a specific time (adjust the time to your current time)
docker logs --since "2024-01-01T10:00:00" --until "2024-01-01T11:00:00" debug-container

# View logs from the last hour
docker logs --since 1h debug-container
Search for specific log patterns:
# Use grep to filter logs for errors
docker logs debug-container 2>&1 | grep -i error

# Search for specific HTTP status codes
docker logs debug-container 2>&1 | grep "404\|500"
Task 2: Use docker exec to Run Commands Inside Containers
Subtask 2.1: Interactive Container Access
Access the container with an interactive bash shell:
# Execute bash inside the running container
docker exec -it debug-container bash
Once inside the container, explore the environment:
# Check the current working directory
pwd

# List files in the application directory
ls -la

# Check running processes
ps aux

# Check network configuration
ip addr show

# Check environment variables
env | grep -E "(PATH|PYTHON|HOME)"

# Exit the container shell
exit
Subtask 2.2: Running Specific Commands
Execute single commands without entering interactive mode:
# Check the Python version in the container
docker exec debug-container python --version

# View the application file
docker exec debug-container cat app.py

# Check disk usage
docker exec debug-container df -h

# Check memory usage
docker exec debug-container free -m
Debug application-specific issues:
# Check if the application is listening on the correct port
docker exec debug-container netstat -tlnp

# Test internal connectivity
docker exec debug-container curl http://localhost:8080/

# Check application logs from inside the container
docker exec debug-container ps aux | grep python
Subtask 2.3: Installing Debug Tools
Install additional debugging tools inside the container:
# Enter the container interactively
docker exec -it debug-container bash

# Update package list and install tools
apt-get update
apt-get install -y curl wget htop strace

# Test the newly installed tools
htop  # Press 'q' to quit
curl http://localhost:8080/
exit
Task 3: Troubleshoot Networking Issues
Subtask 3.1: Inspect Container Network Configuration
Examine container network details:
# Get detailed information about the container
docker inspect debug-container

# Focus on network configuration
docker inspect debug-container | grep -A 20 "NetworkSettings"
Check container IP address and network:
# Get the container's IP address
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' debug-container

# Get network name
docker inspect -f '{{range .NetworkSettings.Networks}}{{.NetworkID}}{{end}}' debug-container
Subtask 3.2: Network Connectivity Testing
Create a second container for network testing:
# Run a simple container for network testing
docker run -d --name network-test alpine sleep 3600

# Test connectivity between containers
docker exec network-test ping -c 3 $(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' debug-container)
Test port connectivity:
# Install network tools in the test container
docker exec network-test apk add --no-cache curl

# Test HTTP connectivity between containers
docker exec network-test curl http://$(docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' debug-container):8080/
Subtask 3.3: Troubleshoot Common Network Issues
Simulate and fix port binding issues:
# Stop the current container
docker stop debug-container

# Try to run another container on the same port (this will fail)
docker run -d --name debug-container-2 -p 8080:8080 debug-app

# Check if the port is already in use
docker ps -a
netstat -tlnp | grep 8080

# Run on a different port
docker run -d --name debug-container-alt -p 8081:8080 debug-app

# Test the new port
curl http://localhost:8081/
Examine Docker networks:
# List all Docker networks
docker network ls

# Inspect the default bridge network
docker network inspect bridge

# Create a custom network for better isolation
docker network create debug-network

# Run containers on the custom network
docker run -d --name debug-app-custom --network debug-network -p 8082:8080 debug-app
Task 4: Attach to Running Containers
Subtask 4.1: Understanding docker attach
Create a container that runs interactively:
# Stop previous containers to free up resources
docker stop debug-container debug-container-2 debug-container-alt debug-app-custom network-test

# Run a container with an interactive process
docker run -d --name interactive-container alpine sh -c "while true; do echo 'Container is running...'; sleep 5; done"
Attach to the running container:
# Attach to see the output
docker attach interactive-container

# Note: You'll see the repeating output
# Press Ctrl+C to detach (this will stop the container)
Subtask 4.2: Proper Attachment Techniques
Run a container with a proper interactive shell:
# Run a container with bash that stays open
docker run -dit --name shell-container ubuntu bash

# Attach to the container
docker attach shell-container

# You now have an interactive shell
# Try some commands:
ls -la
ps aux
echo "I'm inside the container"

# Detach without stopping the container using Ctrl+P, Ctrl+Q
# (Press Ctrl+P, then Ctrl+Q in sequence)
Compare attach vs exec:
# After detaching, check that the container is still running
docker ps

# Use exec to enter the same container (recommended approach)
docker exec -it shell-container bash

# This creates a new process, so exiting won't stop the container
exit

# Verify the container is still running
docker ps
Task 5: Identify and Fix Common Issues
Subtask 5.1: Debugging Container Startup Issues
Create a container with startup problems:
# Create a Dockerfile with issues
cat > Dockerfile.broken << 'EOF'
FROM python:3.9-slim
WORKDIR /app
COPY nonexistent-file.py .
CMD ["python", "nonexistent-file.py"]
EOF

# Try to build and run (this will fail)
docker build -f Dockerfile.broken -t broken-app .
docker run --name broken-container broken-app
Debug the startup failure:
# Check container status
docker ps -a

# Examine the logs to see what went wrong
docker logs broken-container

# Get detailed information about the failed container
docker inspect broken-container
Subtask 5.2: Debugging Resource Issues
Create a container with resource constraints:
# Run a container with limited memory
docker run -d --name memory-limited --memory=50m python:3.9-slim python -c "
import time
data = []
while True:
    data.append('x' * 1024 * 1024)  # Allocate 1MB
    time.sleep(1)
    print(f'Allocated {len(data)} MB')
"
Monitor and debug resource usage:
# Monitor container resource usage
docker stats memory-limited --no-stream

# Check container logs for memory issues
docker logs memory-limited

# Get detailed resource information
docker inspect memory-limited | grep -A 10 "Memory"
Subtask 5.3: Debugging Application Issues
Create an application with runtime errors:
cat > debug-app.py << 'EOF'
import time
import random
import sys

def problematic_function():
    # Simulate various issues
    issue = random.choice(['memory', 'exception', 'slow', 'success'])
    
    if issue == 'memory':
        print("WARNING: High memory usage detected")
        data = [0] * 1000000  # Allocate memory
        time.sleep(2)
    elif issue == 'exception':
        print("ERROR: About to raise an exception")
        raise Exception("Simulated application error")
    elif issue == 'slow':
        print("INFO: Processing slow operation")
        time.sleep(10)
    else:
        print("INFO: Operation completed successfully")

if __name__ == "__main__":
    print("Starting problematic application...")
    while True:
        try:
            problematic_function()
            time.sleep(3)
        except Exception as e:
            print(f"EXCEPTION: {e}")
            time.sleep(5)
EOF

# Create Dockerfile for the problematic app
cat > Dockerfile.debug << 'EOF'
FROM python:3.9-slim
WORKDIR /app
COPY debug-app.py .
CMD ["python", "debug-app.py"]
EOF

# Build and run
docker build -f Dockerfile.debug -t debug-problematic .
docker run -d --name problematic-app debug-problematic
Debug the application issues:
# Monitor logs in real-time
docker logs -f problematic-app &

# In another terminal, examine the container
docker exec -it problematic-app bash

# Inside the container, check processes
ps aux

# Check system resources
free -m
df -h

# Exit the container
exit

# Stop the log monitoring (Ctrl+C in the first terminal)
Subtask 5.4: Using Docker System Commands for Debugging
Check overall Docker system health:
# Get system-wide information
docker system info

# Check disk usage
docker system df

# Show detailed disk usage
docker system df -v
Clean up and optimize:
# Remove unused containers, networks, images
docker system prune

# Remove all stopped containers
docker container prune

# Remove unused images
docker image prune
Troubleshooting Common Issues
Issue 1: Container Won't Start
Symptoms: Container exits immediately after starting

Debug Steps:

# Check exit code and status
docker ps -a

# Examine logs for error messages
docker logs <container_name>

# Inspect container configuration
docker inspect <container_name>
Issue 2: Cannot Connect to Application
Symptoms: Application not accessible from host

Debug Steps:

# Check port bindings
docker port <container_name>

# Verify application is listening inside container
docker exec <container_name> netstat -tlnp

# Test internal connectivity
docker exec <container_name> curl http://localhost:<port>
Issue 3: High Resource Usage
Symptoms: Container consuming too much CPU/memory

Debug Steps:

# Monitor real-time resource usage
docker stats <container_name>

# Check processes inside container
docker exec <container_name> ps aux

# Examine system resources
docker exec <container_name> free -m
docker exec <container_name> df -h
Lab Cleanup
Before finishing the lab, clean up the created resources:

# Stop all running containers
docker stop $(docker ps -q)

# Remove all containers
docker rm $(docker ps -aq)

# Remove created images
docker rmi debug-app broken-app debug-problematic

# Remove custom network
docker network rm debug-network

# Clean up files
rm -rf ~/debug-lab
Conclusion
In this lab, you have successfully learned essential Docker debugging techniques that are crucial for maintaining containerized applications in production environments. You have mastered:

Log Analysis: Using docker logs with various options to inspect container output, filter by time, and identify issues through log patterns
Container Interaction: Leveraging docker exec to run commands inside containers, install debugging tools, and investigate runtime issues
Network Troubleshooting: Examining container network configurations, testing connectivity between containers, and resolving common networking problems
Container Attachment: Understanding when and how to use docker attach versus docker exec for different debugging scenarios
Issue Resolution: Identifying and fixing common container problems including startup failures, resource constraints, and application errors
These debugging skills are fundamental for the Docker Certified Associate (DCA) certification and essential for anyone working with containerized applications in development, testing, or production environments. The systematic approach to debugging you've learned will help you quickly identify and resolve issues, minimizing downtime and improving application reliability.

The techniques covered in this lab form the foundation of container troubleshooting and will serve you well as you advance to more complex Docker deployments and orchestration platforms like Kubernetes.
