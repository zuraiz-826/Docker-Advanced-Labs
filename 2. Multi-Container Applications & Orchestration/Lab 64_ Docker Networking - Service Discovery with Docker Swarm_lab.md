Lab 64: Docker Networking - Service Discovery with Docker Swarm
Lab Objectives
By the end of this lab, you will be able to:

Create and manage a Docker Swarm cluster for container orchestration
Deploy services with dynamic port allocation in Docker Swarm
Configure and implement service discovery mechanisms for inter-service communication
Understand and utilize DNS resolution within Docker Swarm networks
Test service discovery by establishing API communication between containers
Implement fault tolerance and automatic recovery mechanisms for failed services
Monitor and troubleshoot service discovery issues in a distributed environment
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with Linux command line operations
Knowledge of networking concepts (IP addresses, ports, DNS)
Understanding of REST APIs and HTTP requests
Basic knowledge of YAML configuration files
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. No need to build your own virtual machine or install Docker - everything is ready to use.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine pre-installed
Docker Compose pre-configured
Network utilities (curl, dig, nslookup) available
Text editors (nano, vim) for configuration files
Lab Tasks
Task 1: Create a Docker Swarm Cluster and Deploy Services with Dynamic Ports
Subtask 1.1: Initialize Docker Swarm
First, let's initialize a Docker Swarm cluster on your machine.

# Initialize Docker Swarm mode
docker swarm init

# Verify swarm status
docker info | grep -A 10 "Swarm:"
Subtask 1.2: Create a Custom Network for Services
Create an overlay network that will enable service discovery across the swarm.

# Create an overlay network for service communication
docker network create --driver overlay --attachable swarm-network

# List networks to verify creation
docker network ls
Subtask 1.3: Create Application Files
Create a simple web application and API service for testing service discovery.

Create the web application (app.py):

# Create directory for our application
mkdir -p ~/swarm-lab/web-app
cd ~/swarm-lab/web-app

# Create the Python web application
cat > app.py << 'EOF'
from flask import Flask, jsonify, request
import requests
import os
import socket

app = Flask(__name__)

@app.route('/')
def home():
    hostname = socket.gethostname()
    return jsonify({
        'service': 'web-app',
        'hostname': hostname,
        'message': 'Web application is running',
        'port': os.environ.get('PORT', '5000')
    })

@app.route('/call-api')
def call_api():
    try:
        # Use service name for discovery - Docker Swarm will resolve this
        api_url = 'http://api-service:8080/data'
        response = requests.get(api_url, timeout=5)
        return jsonify({
            'web_service': socket.gethostname(),
            'api_response': response.json(),
            'status': 'success'
        })
    except Exception as e:
        return jsonify({
            'web_service': socket.gethostname(),
            'error': str(e),
            'status': 'failed'
        }), 500

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 5000))
    app.run(host='0.0.0.0', port=port)
EOF

# Create requirements file
cat > requirements.txt << 'EOF'
Flask==2.3.3
requests==2.31.0
EOF

# Create Dockerfile for web application
cat > Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
Create the API service (api.py):

# Create directory for API service
mkdir -p ~/swarm-lab/api-service
cd ~/swarm-lab/api-service

# Create the API service
cat > api.py << 'EOF'
from flask import Flask, jsonify
import socket
import os
import time
import random

app = Flask(__name__)

@app.route('/data')
def get_data():
    hostname = socket.gethostname()
    return jsonify({
        'service': 'api-service',
        'hostname': hostname,
        'data': {
            'timestamp': time.time(),
            'random_number': random.randint(1, 1000),
            'message': 'Data from API service'
        },
        'port': os.environ.get('PORT', '8080')
    })

@app.route('/health')
def health_check():
    return jsonify({
        'service': 'api-service',
        'hostname': socket.gethostname(),
        'status': 'healthy',
        'timestamp': time.time()
    })

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
EOF

# Create requirements file
cat > requirements.txt << 'EOF'
Flask==2.3.3
EOF

# Create Dockerfile for API service
cat > Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY api.py .

EXPOSE 8080

CMD ["python", "api.py"]
EOF
Subtask 1.4: Build Docker Images
Build the Docker images for both services.

# Build web application image
cd ~/swarm-lab/web-app
docker build -t web-app:latest .

# Build API service image
cd ~/swarm-lab/api-service
docker build -t api-service:latest .

# Verify images are created
docker images | grep -E "(web-app|api-service)"
Subtask 1.5: Deploy Services with Dynamic Ports
Deploy both services to the Docker Swarm with dynamic port allocation.

# Deploy API service with 2 replicas
docker service create \
  --name api-service \
  --replicas 2 \
  --network swarm-network \
  --publish published=8080,target=8080,mode=ingress \
  api-service:latest

# Deploy web application with 2 replicas
docker service create \
  --name web-service \
  --replicas 2 \
  --network swarm-network \
  --publish published=5000,target=5000,mode=ingress \
  web-app:latest

# Verify services are running
docker service ls
Task 2: Set Up Service Discovery for Services to Communicate
Subtask 2.1: Verify Service Discovery Configuration
Check that services can discover each other using service names.

# List all services and their details
docker service ps api-service
docker service ps web-service

# Inspect the network to see connected services
docker network inspect swarm-network
Subtask 2.2: Test Basic Service Discovery
Test if services can resolve each other's names within the swarm network.

# Get a container ID from the web service
WEB_CONTAINER=$(docker ps --filter "name=web-service" --format "{{.ID}}" | head -1)

# Test DNS resolution from web service to API service
docker exec $WEB_CONTAINER nslookup api-service

# Test connectivity
docker exec $WEB_CONTAINER ping -c 3 api-service
Subtask 2.3: Create a Service Discovery Test Script
Create a script to test service discovery functionality.

# Create test directory
mkdir -p ~/swarm-lab/tests
cd ~/swarm-lab/tests

# Create service discovery test script
cat > test_discovery.sh << 'EOF'
#!/bin/bash

echo "=== Docker Swarm Service Discovery Test ==="
echo

# Test 1: Check if services are running
echo "1. Checking service status..."
docker service ls
echo

# Test 2: Test DNS resolution
echo "2. Testing DNS resolution..."
WEB_CONTAINER=$(docker ps --filter "name=web-service" --format "{{.ID}}" | head -1)
if [ ! -z "$WEB_CONTAINER" ]; then
    echo "Resolving api-service from web-service container:"
    docker exec $WEB_CONTAINER nslookup api-service
else
    echo "No web-service container found"
fi
echo

# Test 3: Test HTTP connectivity
echo "3. Testing HTTP connectivity..."
API_CONTAINER=$(docker ps --filter "name=api-service" --format "{{.ID}}" | head -1)
if [ ! -z "$API_CONTAINER" ]; then
    echo "Testing API service health endpoint:"
    docker exec $API_CONTAINER curl -s http://localhost:8080/health | python3 -m json.tool
else
    echo "No api-service container found"
fi
echo

# Test 4: Test service-to-service communication
echo "4. Testing service-to-service communication..."
if [ ! -z "$WEB_CONTAINER" ]; then
    echo "Web service calling API service:"
    docker exec $WEB_CONTAINER curl -s http://api-service:8080/data | python3 -m json.tool
else
    echo "Cannot test - no web container available"
fi

echo
echo "=== Test Complete ==="
EOF

# Make script executable
chmod +x test_discovery.sh

# Run the test
./test_discovery.sh
Task 3: Explore DNS Resolution in Docker Swarm
Subtask 3.1: Understand Docker Swarm DNS
Docker Swarm provides built-in DNS resolution for service discovery. Let's explore how it works.

# Check Docker daemon DNS configuration
docker system info | grep -A 5 "DNS"

# Inspect swarm network DNS settings
docker network inspect swarm-network | grep -A 10 "IPAM"
Subtask 3.2: Test DNS Resolution from Different Perspectives
Test DNS resolution from various containers and services.

# Create a debug container for network testing
docker service create \
  --name debug-service \
  --network swarm-network \
  --replicas 1 \
  alpine:latest sleep 3600

# Wait for service to start
sleep 10

# Get debug container ID
DEBUG_CONTAINER=$(docker ps --filter "name=debug-service" --format "{{.ID}}" | head -1)

# Test DNS resolution from debug container
echo "Testing DNS resolution from debug container:"
docker exec $DEBUG_CONTAINER nslookup api-service
docker exec $DEBUG_CONTAINER nslookup web-service

# Test with dig for more detailed DNS information
docker exec $DEBUG_CONTAINER apk add --no-cache bind-tools
docker exec $DEBUG_CONTAINER dig api-service
Subtask 3.3: Examine Service Discovery Logs
Check logs to understand how service discovery works.

# Check service logs for DNS-related information
docker service logs api-service --tail 20
docker service logs web-service --tail 20

# Check system logs for DNS resolution
sudo journalctl -u docker.service --since "10 minutes ago" | grep -i dns
Task 4: Test Service Discovery with API Calls Between Containers
Subtask 4.1: Test Direct API Communication
Test communication between services using their service names.

# Test API service directly
echo "Testing API service health endpoint:"
curl -s http://localhost:8080/health | python3 -m json.tool

echo -e "\nTesting API service data endpoint:"
curl -s http://localhost:8080/data | python3 -m json.tool
Subtask 4.2: Test Service-to-Service Communication
Test the web service calling the API service through service discovery.

# Test web service home endpoint
echo "Testing web service home endpoint:"
curl -s http://localhost:5000/ | python3 -m json.tool

echo -e "\nTesting web service calling API service:"
curl -s http://localhost:5000/call-api | python3 -m json.tool
Subtask 4.3: Create Load Testing Script
Create a script to test service discovery under load.

# Create load test script
cd ~/swarm-lab/tests

cat > load_test.sh << 'EOF'
#!/bin/bash

echo "=== Load Testing Service Discovery ==="
echo "Running 20 requests to test service discovery..."
echo

for i in {1..20}; do
    echo "Request $i:"
    response=$(curl -s http://localhost:5000/call-api)
    hostname=$(echo $response | python3 -c "import sys, json; print(json.load(sys.stdin).get('web_service', 'unknown'))")
    api_hostname=$(echo $response | python3 -c "import sys, json; print(json.load(sys.stdin).get('api_response', {}).get('hostname', 'unknown'))")
    echo "  Web: $hostname -> API: $api_hostname"
    sleep 1
done

echo
echo "=== Load Test Complete ==="
EOF

chmod +x load_test.sh
./load_test.sh
Task 5: Implement Fault Tolerance and Automatic Recovery
Subtask 5.1: Configure Health Checks
Add health checks to services for better fault tolerance.

# Update API service with health check
docker service update \
  --health-cmd "curl -f http://localhost:8080/health || exit 1" \
  --health-interval 30s \
  --health-timeout 10s \
  --health-retries 3 \
  api-service

# Update web service with health check
docker service update \
  --health-cmd "curl -f http://localhost:5000/ || exit 1" \
  --health-interval 30s \
  --health-timeout 10s \
  --health-retries 3 \
  web-service

# Check service health status
docker service ps api-service
docker service ps web-service
Subtask 5.2: Test Fault Tolerance
Simulate failures to test automatic recovery.

# Create fault tolerance test script
cat > fault_tolerance_test.sh << 'EOF'
#!/bin/bash

echo "=== Fault Tolerance Test ==="
echo

# Get initial service status
echo "Initial service status:"
docker service ls
echo

# Kill one API service container
echo "Killing one API service container..."
API_CONTAINER=$(docker ps --filter "name=api-service" --format "{{.ID}}" | head -1)
if [ ! -z "$API_CONTAINER" ]; then
    docker kill $API_CONTAINER
    echo "Killed container: $API_CONTAINER"
else
    echo "No API container found to kill"
fi

echo
echo "Waiting 30 seconds for recovery..."
sleep 30

# Check if service recovered
echo "Service status after recovery:"
docker service ls
docker service ps api-service

echo
echo "Testing service discovery after recovery:"
curl -s http://localhost:5000/call-api | python3 -m json.tool

echo
echo "=== Fault Tolerance Test Complete ==="
EOF

chmod +x fault_tolerance_test.sh
./fault_tolerance_test.sh
Subtask 5.3: Scale Services for High Availability
Scale services to improve fault tolerance.

# Scale API service to 3 replicas
docker service scale api-service=3

# Scale web service to 3 replicas
docker service scale web-service=3

# Verify scaling
docker service ls
docker service ps api-service
docker service ps web-service

# Test load distribution
echo "Testing load distribution across scaled services:"
for i in {1..10}; do
    response=$(curl -s http://localhost:5000/call-api)
    api_hostname=$(echo $response | python3 -c "import sys, json; print(json.load(sys.stdin).get('api_response', {}).get('hostname', 'unknown'))")
    echo "Request $i routed to API container: $api_hostname"
    sleep 1
done
Subtask 5.4: Monitor Service Discovery Performance
Create a monitoring script to track service discovery performance.

# Create monitoring script
cat > monitor_services.sh << 'EOF'
#!/bin/bash

echo "=== Service Discovery Monitoring ==="
echo

while true; do
    clear
    echo "=== Docker Swarm Service Status - $(date) ==="
    echo
    
    # Service status
    echo "Services:"
    docker service ls
    echo
    
    # Network status
    echo "Network connections:"
    docker network inspect swarm-network --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'
    echo
    
    # Health status
    echo "Service health:"
    for service in api-service web-service; do
        echo "$service containers:"
        docker service ps $service --format "table {{.ID}}\t{{.Name}}\t{{.CurrentState}}\t{{.DesiredState}}"
        echo
    done
    
    # Test connectivity
    echo "Connectivity test:"
    response=$(curl -s -w "Time: %{time_total}s\n" http://localhost:5000/call-api 2>/dev/null)
    if [ $? -eq 0 ]; then
        echo "✓ Service discovery working"
    else
        echo "✗ Service discovery failed"
    fi
    
    echo
    echo "Press Ctrl+C to stop monitoring..."
    sleep 10
done
EOF

chmod +x monitor_services.sh

# Run monitoring for 1 minute
timeout 60 ./monitor_services.sh
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Services cannot resolve each other

# Check if services are on the same network
docker service inspect web-service --format '{{.Spec.TaskTemplate.Networks}}'
docker service inspect api-service --format '{{.Spec.TaskTemplate.Networks}}'

# Ensure both services are attached to swarm-network
docker service update --network-add swarm-network web-service
docker service update --network-add swarm-network api-service
Issue 2: DNS resolution fails

# Check Docker daemon DNS settings
docker system info | grep DNS

# Restart Docker service if needed
sudo systemctl restart docker

# Reinitialize swarm if necessary
docker swarm leave --force
docker swarm init
Issue 3: Health checks failing

# Check health check logs
docker service logs api-service | grep health

# Update health check command
docker service update --health-cmd "curl -f http://localhost:8080/health || exit 1" api-service
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Remove services
docker service rm web-service api-service debug-service

# Remove network
docker network rm swarm-network

# Leave swarm mode
docker swarm leave --force

# Remove images
docker rmi web-app:latest api-service:latest

# Clean up files
rm -rf ~/swarm-lab
Conclusion
In this lab, you have successfully:

Created a Docker Swarm cluster and learned how to initialize swarm mode for container orchestration
Deployed services with dynamic ports using Docker Swarm's built-in load balancing and port management
Implemented service discovery that allows containers to communicate using service names instead of IP addresses
Explored DNS resolution within Docker Swarm and understood how the internal DNS server resolves service names
Tested API communication between containers using service discovery mechanisms
Implemented fault tolerance with health checks, automatic recovery, and service scaling
Why This Matters:

Service discovery is a critical component of microservices architecture. In production environments, services need to find and communicate with each other dynamically as containers are created, destroyed, and moved across different hosts. Docker Swarm's built-in service discovery eliminates the need for external service discovery tools in many scenarios, making it easier to build resilient, scalable applications.

The skills you've learned in this lab are essential for:

Building microservices architectures
Implementing container orchestration in production
Creating fault-tolerant distributed systems
Preparing for the Docker Certified Associate (DCA) certification
Understanding service discovery in Docker Swarm provides a solid foundation for working with more complex orchestration platforms like Kubernetes, where similar concepts apply but with additional complexity and features.
