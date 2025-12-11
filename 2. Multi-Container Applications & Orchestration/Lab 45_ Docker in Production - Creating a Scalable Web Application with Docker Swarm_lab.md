Lab 45: Docker in Production - Creating a Scalable Web Application with Docker Swarm
Objectives
By the end of this lab, you will be able to:

Build and deploy a multi-container web application using Docker Compose
Initialize and manage a Docker Swarm cluster for production deployment
Scale web application services dynamically based on demand
Configure Nginx as a reverse proxy to manage incoming traffic
Test and validate fault tolerance by simulating node failures
Understand production-ready Docker deployment strategies
Implement load balancing and service discovery in Docker Swarm
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with command-line interface (CLI) operations
Knowledge of web applications and HTTP protocols
Understanding of basic networking concepts
Experience with text editors (nano, vim, or similar)
Technical Requirements:

Docker Engine version 20.10 or higher
Docker Compose version 2.0 or higher
Basic knowledge of YAML file structure
Understanding of Linux file permissions
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker already installed. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 22.04 LTS with Docker Engine pre-installed
Docker Compose pre-configured
All necessary networking tools
Text editors and utilities ready to use
Task 1: Create a Multi-Container Web Application Using Docker Compose
Subtask 1.1: Set Up the Project Structure
First, let's create a organized directory structure for our web application.

# Create the main project directory
mkdir ~/docker-swarm-lab
cd ~/docker-swarm-lab

# Create subdirectories for different components
mkdir -p web-app nginx database
Subtask 1.2: Create a Simple Web Application
We'll create a Python Flask web application that connects to a Redis database.

# Navigate to the web-app directory
cd ~/docker-swarm-lab/web-app

# Create the Flask application file
cat > app.py << 'EOF'
from flask import Flask, request, jsonify
import redis
import os
import socket
import json
from datetime import datetime

app = Flask(__name__)

# Connect to Redis
redis_host = os.environ.get('REDIS_HOST', 'redis')
redis_port = int(os.environ.get('REDIS_PORT', 6379))
r = redis.Redis(host=redis_host, port=redis_port, decode_responses=True)

@app.route('/')
def home():
    hostname = socket.gethostname()
    try:
        # Increment visit counter
        visits = r.incr('visits')
        
        # Store visitor info
        visitor_info = {
            'timestamp': datetime.now().isoformat(),
            'hostname': hostname,
            'remote_addr': request.remote_addr
        }
        
        return jsonify({
            'message': 'Welcome to Docker Swarm Lab!',
            'container_hostname': hostname,
            'total_visits': visits,
            'visitor_info': visitor_info,
            'status': 'healthy'
        })
    except Exception as e:
        return jsonify({
            'message': 'Welcome to Docker Swarm Lab!',
            'container_hostname': hostname,
            'error': str(e),
            'status': 'redis_unavailable'
        }), 500

@app.route('/health')
def health():
    try:
        r.ping()
        return jsonify({'status': 'healthy', 'redis': 'connected'})
    except:
        return jsonify({'status': 'unhealthy', 'redis': 'disconnected'}), 500

@app.route('/info')
def info():
    hostname = socket.gethostname()
    return jsonify({
        'container_hostname': hostname,
        'app_version': '1.0.0',
        'python_version': os.sys.version,
        'environment': os.environ.get('ENVIRONMENT', 'development')
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
EOF
Subtask 1.3: Create Requirements File and Dockerfile
# Create requirements.txt for Python dependencies
cat > requirements.txt << 'EOF'
Flask==2.3.3
redis==5.0.1
gunicorn==21.2.0
EOF

# Create Dockerfile for the web application
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

WORKDIR /app

# Copy requirements first for better caching
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .

# Create non-root user for security
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

EXPOSE 5000

# Use gunicorn for production
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
EOF
Subtask 1.4: Configure Nginx Reverse Proxy
# Navigate to nginx directory
cd ~/docker-swarm-lab/nginx

# Create Nginx configuration
cat > nginx.conf << 'EOF'
upstream web_backend {
    server web-app:5000;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://web_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Health check and load balancing
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
    }

    location /nginx-health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
EOF

# Create Dockerfile for Nginx
cat > Dockerfile << 'EOF'
FROM nginx:1.25-alpine

# Copy custom configuration
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Create non-root user
RUN addgroup -g 1000 -S nginx-user && \
    adduser -S -D -H -u 1000 -h /var/cache/nginx -s /sbin/nologin -G nginx-user -g nginx-user nginx-user

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
EOF
Subtask 1.5: Create Docker Compose Configuration
# Navigate back to the main project directory
cd ~/docker-swarm-lab

# Create docker-compose.yml file
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web-app:
    build: ./web-app
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - ENVIRONMENT=production
    depends_on:
      - redis
    networks:
      - app-network
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M

  nginx:
    build: ./nginx
    ports:
      - "80:80"
    depends_on:
      - web-app
    networks:
      - app-network
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.25'
          memory: 128M

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - app-network
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.25'
          memory: 128M

volumes:
  redis-data:

networks:
  app-network:
    driver: overlay
    attachable: true
EOF
Subtask 1.6: Test the Application Locally
# Build and test the application using Docker Compose
docker-compose build

# Start the application in detached mode
docker-compose up -d

# Check if all services are running
docker-compose ps

# Test the application
curl http://localhost/
curl http://localhost/health
curl http://localhost/info

# View logs to ensure everything is working
docker-compose logs web-app
docker-compose logs nginx

# Stop the local test
docker-compose down
Task 2: Deploy the Application in Swarm Mode
Subtask 2.1: Initialize Docker Swarm
# Initialize Docker Swarm on the current node
docker swarm init

# Display swarm information
docker info | grep -A 10 "Swarm:"

# List swarm nodes
docker node ls
Subtask 2.2: Create a Docker Stack File
Docker Stack is the production equivalent of Docker Compose for Swarm mode.

# Create a stack file (similar to compose but with swarm-specific features)
cat > docker-stack.yml << 'EOF'
version: '3.8'

services:
  web-app:
    build: ./web-app
    image: swarm-lab-web:latest
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - ENVIRONMENT=production
    networks:
      - app-network
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M
      placement:
        max_replicas_per_node: 2

  nginx:
    build: ./nginx
    image: swarm-lab-nginx:latest
    ports:
      - "80:80"
    networks:
      - app-network
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s
      resources:
        limits:
          cpus: '0.25'
          memory: 128M
      placement:
        constraints:
          - node.role == manager

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis-data:/data
    networks:
      - app-network
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.25'
          memory: 128M
      placement:
        constraints:
          - node.role == manager

volumes:
  redis-data:

networks:
  app-network:
    driver: overlay
    attachable: true
EOF
Subtask 2.3: Build Images and Deploy Stack
# Build the images first
docker build -t swarm-lab-web:latest ./web-app
docker build -t swarm-lab-nginx:latest ./nginx

# Deploy the stack to the swarm
docker stack deploy -c docker-stack.yml webapp-stack

# Check the stack deployment
docker stack ls

# List services in the stack
docker stack services webapp-stack

# Check service details
docker service ls
Subtask 2.4: Verify Deployment
# Check service status and replicas
docker service ps webapp-stack_web-app
docker service ps webapp-stack_nginx
docker service ps webapp-stack_redis

# Test the deployed application
curl http://localhost/
curl http://localhost/health
curl http://localhost/info

# Check service logs
docker service logs webapp-stack_web-app
docker service logs webapp-stack_nginx
Task 3: Scale the Web Application Services
Subtask 3.1: Scale Services Manually
# Scale the web application to 5 replicas
docker service scale webapp-stack_web-app=5

# Scale nginx to 3 replicas for better load distribution
docker service scale webapp-stack_nginx=3

# Check the scaling progress
docker service ps webapp-stack_web-app
docker service ps webapp-stack_nginx

# Verify all replicas are running
docker service ls
Subtask 3.2: Test Load Distribution
# Create a script to test load distribution
cat > test-load.sh << 'EOF'
#!/bin/bash

echo "Testing load distribution across replicas..."
echo "Making 20 requests to see different container hostnames:"
echo

for i in {1..20}; do
    echo "Request $i:"
    curl -s http://localhost/ | grep -o '"container_hostname":"[^"]*"' | cut -d'"' -f4
    sleep 0.5
done

echo
echo "Testing completed. You should see requests distributed across different containers."
EOF

# Make the script executable and run it
chmod +x test-load.sh
./test-load.sh
Subtask 3.3: Monitor Resource Usage
# Check resource usage of services
docker stats --no-stream

# Get detailed information about service resource allocation
docker service inspect webapp-stack_web-app --format '{{.Spec.TaskTemplate.Resources}}'

# Check node resource usage
docker node ls
docker system df
Subtask 3.4: Scale Down Services
# Scale down web-app service to 2 replicas
docker service scale webapp-stack_web-app=2

# Scale down nginx to 1 replica
docker service scale webapp-stack_nginx=1

# Verify the scaling down
docker service ls
docker service ps webapp-stack_web-app
Task 4: Set Up Advanced Reverse Proxy Configuration
Subtask 4.1: Update Nginx Configuration for Better Load Balancing
# Create an enhanced Nginx configuration
cd ~/docker-swarm-lab/nginx

# Backup the original configuration
cp nginx.conf nginx.conf.backup

# Create enhanced configuration with load balancing
cat > nginx.conf << 'EOF'
upstream web_backend {
    # Load balancing method
    least_conn;
    
    # Backend servers (Docker Swarm will handle service discovery)
    server web-app:5000 max_fails=3 fail_timeout=30s;
    
    # Health check configuration
    keepalive 32;
}

# Rate limiting
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;
    server_name localhost;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Logging
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # Main application
    location / {
        # Apply rate limiting
        limit_req zone=api_limit burst=20 nodelay;
        
        proxy_pass http://web_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts and buffering
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
        
        # Connection reuse
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    # Health check endpoint
    location /nginx-health {
        access_log off;
        return 200 "nginx-healthy\n";
        add_header Content-Type text/plain;
    }

    # Application health check proxy
    location /app-health {
        proxy_pass http://web_backend/health;
        proxy_set_header Host $host;
        access_log off;
    }

    # Static content caching (if needed)
    location ~* \.(jpg|jpeg|png|gif|ico|css|js)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
EOF
Subtask 4.2: Rebuild and Update Nginx Service
# Rebuild the nginx image with new configuration
docker build -t swarm-lab-nginx:v2 ./nginx

# Update the service with the new image
docker service update --image swarm-lab-nginx:v2 webapp-stack_nginx

# Check the update progress
docker service ps webapp-stack_nginx

# Test the updated configuration
curl http://localhost/nginx-health
curl http://localhost/app-health
Subtask 4.3: Add SSL/TLS Support (Optional Enhancement)
# Create self-signed certificates for testing
mkdir -p ~/docker-swarm-lab/nginx/ssl

# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout ~/docker-swarm-lab/nginx/ssl/nginx.key \
    -out ~/docker-swarm-lab/nginx/ssl/nginx.crt \
    -subj "/C=US/ST=State/L=City/O=Organization/CN=localhost"

# Update Nginx configuration to include HTTPS
cat > ~/docker-swarm-lab/nginx/nginx-ssl.conf << 'EOF'
upstream web_backend {
    least_conn;
    server web-app:5000 max_fails=3 fail_timeout=30s;
    keepalive 32;
}

# Redirect HTTP to HTTPS
server {
    listen 80;
    server_name localhost;
    return 301 https://$server_name$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name localhost;

    # SSL configuration
    ssl_certificate /etc/nginx/ssl/nginx.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;

    location / {
        proxy_pass http://web_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        proxy_connect_timeout 5s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;
    }

    location /nginx-health {
        access_log off;
        return 200 "nginx-ssl-healthy\n";
        add_header Content-Type text/plain;
    }
}
EOF
Task 5: Test Fault Tolerance by Simulating Node Failure
Subtask 5.1: Create Monitoring Script
# Create a monitoring script to observe service behavior
cat > monitor-services.sh << 'EOF'
#!/bin/bash

echo "Starting service monitoring..."
echo "Press Ctrl+C to stop monitoring"
echo

while true; do
    clear
    echo "=== Docker Swarm Service Status ==="
    echo "Timestamp: $(date)"
    echo
    
    echo "--- Service List ---"
    docker service ls
    echo
    
    echo "--- Web App Service Details ---"
    docker service ps webapp-stack_web-app --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}\t{{.Error}}"
    echo
    
    echo "--- Nginx Service Details ---"
    docker service ps webapp-stack_nginx --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}\t{{.Error}}"
    echo
    
    echo "--- Application Health Check ---"
    curl -s http://localhost/health | head -1 || echo "Health check failed"
    echo
    
    sleep 5
done
EOF

chmod +x monitor-services.sh
Subtask 5.2: Simulate Container Failure
# Start monitoring in background (optional)
# ./monitor-services.sh &
# MONITOR_PID=$!

# Get list of running containers for web-app service
docker service ps webapp-stack_web-app

# Kill a specific container to simulate failure
CONTAINER_ID=$(docker ps --filter "label=com.docker.swarm.service.name=webapp-stack_web-app" --format "{{.ID}}" | head -1)

echo "Killing container: $CONTAINER_ID"
docker kill $CONTAINER_ID

# Watch Docker Swarm automatically restart the container
echo "Watching service recovery..."
sleep 2
docker service ps webapp-stack_web-app

# Test application availability during recovery
echo "Testing application availability:"
for i in {1..10}; do
    echo "Test $i: $(curl -s http://localhost/ | grep -o '"status":"[^"]*"' || echo 'Request failed')"
    sleep 1
done
Subtask 5.3: Simulate High Load and Stress Testing
# Create a load testing script
cat > load-test.sh << 'EOF'
#!/bin/bash

echo "Starting load test..."
echo "This will make 100 concurrent requests to test fault tolerance"

# Function to make requests
make_requests() {
    local thread_id=$1
    local requests_per_thread=$2
    
    for ((i=1; i<=requests_per_thread; i++)); do
        response=$(curl -s -w "%{http_code}" http://localhost/ -o /dev/null)
        if [ "$response" != "200" ]; then
            echo "Thread $thread_id: Request $i failed with code $response"
        else
            echo "Thread $thread_id: Request $i successful"
        fi
        sleep 0.1
    done
}

# Start multiple background processes
for thread in {1..5}; do
    make_requests $thread 20 &
done

# Wait for all background jobs to complete
wait

echo "Load test completed"
EOF

chmod +x load-test.sh

# Run the load test
./load-test.sh
Subtask 5.4: Test Service Recovery and Scaling
# Scale down to minimum replicas
echo "Scaling down to test minimum service availability..."
docker service scale webapp-stack_web-app=1

# Test application with minimal resources
curl http://localhost/
curl http://localhost/health

# Simulate failure of the single replica
CONTAINER_ID=$(docker ps --filter "label=com.docker.swarm.service.name=webapp-stack_web-app" --format "{{.ID}}")
echo "Killing the only web-app container: $CONTAINER_ID"
docker kill $CONTAINER_ID

# Monitor recovery time
echo "Monitoring recovery..."
start_time=$(date +%s)

while true; do
    if curl -s http://localhost/health > /dev/null 2>&1; then
        end_time=$(date +%s)
        recovery_time=$((end_time - start_time))
        echo "Service recovered in $recovery_time seconds"
        break
    fi
    echo "Service still recovering..."
    sleep 1
done

# Scale back up
echo "Scaling back to normal operations..."
docker service scale webapp-stack_web-app=3
Subtask 5.5: Test Network Partition Simulation
# Create a script to simulate network issues
cat > network-test.sh << 'EOF'
#!/bin/bash

echo "Testing network resilience..."

# Get container IDs
WEB_CONTAINERS=$(docker ps --filter "label=com.docker.swarm.service.name=webapp-stack_web-app" --format "{{.ID}}")
REDIS_CONTAINER=$(docker ps --filter "label=com.docker.swarm.service.name=webapp-stack_redis" --format "{{.ID}}")

echo "Web containers: $WEB_CONTAINERS"
echo "Redis container: $REDIS_CONTAINER"

# Test application behavior when Redis is unavailable
echo "Stopping Redis to simulate database failure..."
docker pause $REDIS_CONTAINER

# Test application response
echo "Testing application response without Redis:"
for i in {1..5}; do
    echo "Test $i:"
    curl -s http://localhost/ | grep -o '"status":"[^"]*"' || echo "Request failed"
    sleep 1
done

# Restore Redis
echo "Restoring Redis..."
docker unpause $REDIS_CONTAINER

# Wait for Redis to be ready
sleep 3

# Test recovery
echo "Testing application recovery:"
for i in {1..5}; do
    echo "Recovery test $i:"
    curl -s http://localhost/ | grep -o '"status":"[^"]*"' || echo "Request failed"
    sleep 1
done

echo "Network resilience test completed"
EOF

chmod +x network-test.sh
./network-test.sh
Troubleshooting Common Issues
Issue 1: Services Not Starting
# Check service logs
docker service logs webapp-stack_web-app
docker service logs webapp-stack_nginx
docker service logs webapp-stack_redis

# Check service constraints and placement
docker service inspect webapp-stack_web-app --format '{{.Spec.TaskTemplate.Placement}}'

# Check node availability
docker node ls
Issue 2: Load Balancing Not Working
# Verify network connectivity
docker network ls
docker network inspect webapp-stack_app-network

# Test service discovery
docker exec -it $(docker ps --filter "label=com.docker.swarm.service.name=webapp-stack_nginx" --format "{{.ID}}" | head -1) nslookup web-app

# Check Nginx configuration
docker exec -it $(docker ps --filter "label=com.docker.swarm.service.name=webapp-stack_nginx" --format "{{.ID}}" | head -1) nginx -t
Issue 3: Performance Issues
# Check resource usage
docker stats --no-stream

# Check service resource limits
docker service inspect webapp-stack_web-app --format '{{.Spec.TaskTemplate.Resources}}'

# Monitor system resources
top
df -h
free -h
Cleanup and Resource Management
# Remove the stack
docker stack rm webapp-stack

# Wait for cleanup to complete
sleep 30

# Remove unused images
docker image prune -f

# Remove unused volumes
docker volume prune -f

# Remove unused networks
docker network prune -f

# Leave swarm mode (if needed)
# docker swarm leave --force

# Verify cleanup
docker ps
docker service ls
docker stack ls
Conclusion
Congratulations! You have successfully completed Lab 45: Docker in Production - Creating a Scalable Web Application with Docker Swarm.

What You Accomplished
In this comprehensive lab, you have:

Built a Production-Ready Multi-Container Application: Created a complete web application stack with Flask, Redis, and Nginx, demonstrating real-world application architecture patterns.

Mastered Docker Swarm Orchestration: Successfully initialized a Docker Swarm cluster and deployed applications using Docker Stack, understanding the differences between development and production deployment strategies.

Implemented Scalable Service Architecture: Learned how to scale services dynamically, configure resource limits, and manage service placement across cluster nodes.

Configured Advanced Load Balancing: Set up Nginx as a reverse proxy with sophisticated load balancing, health checks, and security headers, preparing you for production traffic management.

Validated Fault Tolerance and High Availability: Tested your application's resilience by simulating various failure scenarios, including container crashes, network partitions, and resource constraints.

Why This Matters
The skills you've developed in this lab are crucial for modern DevOps and cloud engineering roles because:

Production Readiness: You now understand how to deploy applications that can handle real-world traffic and failures
Scalability: You can design systems that grow with demand and maintain performance under load
Reliability: You've learned to build fault-tolerant systems that recover automatically from failures
Industry Standards: Docker Swarm and containerization are fundamental technologies in modern infrastructure
Next Steps
To further enhance your Docker and container orchestration skills:

Explore Kubernetes as an alternative orchestration platform
Implement CI/CD pipelines for automated deployments
Study advanced networking concepts like service mesh
Learn about monitoring and observability tools for containerized applications
Practice with multi-node Swarm clusters in cloud environments
Certification Preparation
This lab directly supports your preparation for the Docker Certified Associate (DCA) certification by covering:

Docker Swarm cluster management
Service scaling and load balancing
Production deployment strategies
Fault tolerance and high availability
Security best practices in containerized environments
You now have hands-on experience with production-grade Docker deployments that will serve you well in both certification exams and real-world scenarios.
