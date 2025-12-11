Lab 41: Docker Swarm - Deploying a Multi-Service Application
Lab Objectives
By the end of this lab, you will be able to:

• Initialize a Docker Swarm cluster and add worker nodes • Create a docker-compose.yml file for deploying multi-service applications • Deploy applications using docker stack deploy command • Scale services dynamically in Docker Swarm mode • Monitor application health and troubleshoot common deployment issues • Understand the fundamentals of container orchestration with Docker Swarm

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker containers and images • Familiarity with YAML file format • Basic Linux command line knowledge • Understanding of networking concepts (ports, IP addresses) • Knowledge of web applications and databases

Required Tools: • Docker Engine (version 20.10 or later) • Docker Compose (version 2.0 or later) • Text editor (nano, vim, or any preferred editor)

Lab Environment Setup
Al Nafi Cloud Machines: This lab uses Al Nafi's pre-configured Linux-based cloud machines. Simply click Start Lab to access your environment. No need to build your own VM or install Docker - everything is ready to use!

Your lab environment includes: • 3 Ubuntu Linux machines with Docker pre-installed • Machine 1: Swarm Manager Node • Machine 2: Swarm Worker Node 1 • Machine 3: Swarm Worker Node 2

Task 1: Initialize Docker Swarm and Add Nodes
Step 1.1: Verify Docker Installation
First, let's verify that Docker is properly installed on all machines.

On Machine 1 (Manager Node):

# Check Docker version
docker --version

# Check Docker service status
sudo systemctl status docker

# Verify Docker is running
docker info
Step 1.2: Initialize Docker Swarm
On Machine 1 (Manager Node):

# Initialize Docker Swarm
docker swarm init --advertise-addr $(hostname -I | awk '{print $1}')

# The output will show a command to join worker nodes
# Save this join command for the next step
Expected Output:

Swarm initialized: current node (xyz123) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-xxxxx <MANAGER-IP>:2377
Step 1.3: Add Worker Nodes to the Swarm
On Machine 2 (Worker Node 1):

# Use the join command from the previous step
# Replace <JOIN-TOKEN> and <MANAGER-IP> with actual values
docker swarm join --token <JOIN-TOKEN> <MANAGER-IP>:2377
On Machine 3 (Worker Node 2):

# Use the same join command
docker swarm join --token <JOIN-TOKEN> <MANAGER-IP>:2377
Step 1.4: Verify Swarm Cluster
On Machine 1 (Manager Node):

# List all nodes in the swarm
docker node ls

# Check swarm status
docker info | grep -A 10 "Swarm:"
Expected Output:

ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS
abc123 *                      machine1   Ready     Active         Leader
def456                        machine2   Ready     Active         
ghi789                        machine3   Ready     Active         
Task 2: Create a Docker Compose File for Multi-Service Application
Step 2.1: Create Project Directory
On Machine 1 (Manager Node):

# Create project directory
mkdir ~/swarm-app
cd ~/swarm-app

# Create necessary subdirectories
mkdir -p web database
Step 2.2: Create Web Application
Create a simple web application that connects to a database.

# Create a simple Python web app
cat > web/app.py << 'EOF'
from flask import Flask, jsonify
import redis
import os
import socket

app = Flask(__name__)
redis_client = redis.Redis(host=os.getenv('REDIS_HOST', 'redis'), port=6379, decode_responses=True)

@app.route('/')
def hello():
    try:
        # Increment visit counter
        visits = redis_client.incr('visits')
        hostname = socket.gethostname()
        return jsonify({
            'message': 'Hello from Docker Swarm!',
            'hostname': hostname,
            'visits': visits,
            'status': 'success'
        })
    except Exception as e:
        return jsonify({
            'message': 'Hello from Docker Swarm!',
            'hostname': socket.gethostname(),
            'error': str(e),
            'status': 'error'
        })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
Step 2.3: Create Dockerfile for Web Application
# Create Dockerfile for the web application
cat > web/Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

# Install required packages
RUN pip install flask redis

# Copy application code
COPY app.py .

# Expose port
EXPOSE 5000

# Run the application
CMD ["python", "app.py"]
EOF
Step 2.4: Create Docker Compose File
# Create docker-compose.yml for the multi-service application
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    build: ./web
    ports:
      - "80:5000"
    environment:
      - REDIS_HOST=redis
    networks:
      - app-network
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s
      placement:
        constraints:
          - node.role == worker
    depends_on:
      - redis

  redis:
    image: redis:7-alpine
    networks:
      - app-network
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == manager
    volumes:
      - redis-data:/data

  nginx:
    image: nginx:alpine
    ports:
      - "8080:80"
    networks:
      - app-network
    deploy:
      replicas: 2
      restart_policy:
        condition: on-failure
    configs:
      - source: nginx-config
        target: /etc/nginx/nginx.conf

networks:
  app-network:
    driver: overlay
    attachable: true

volumes:
  redis-data:

configs:
  nginx-config:
    external: true
EOF
Step 2.5: Create Nginx Configuration
# Create nginx configuration
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream web_servers {
        server web:5000;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://web_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF
Task 3: Deploy the Application Using Docker Stack Deploy
Step 3.1: Build and Push Images
# Build the web application image
docker build -t swarm-web:latest ./web

# Tag the image for the swarm
docker tag swarm-web:latest swarm-web:v1.0
Step 3.2: Create Docker Config for Nginx
# Create nginx config as a Docker config
docker config create nginx-config nginx.conf
Step 3.3: Deploy the Stack
# Deploy the application stack
docker stack deploy -c docker-compose.yml myapp

# Verify stack deployment
docker stack ls
Expected Output:

NAME      SERVICES   ORCHESTRATOR
myapp     3          Swarm
Step 3.4: Verify Services
# List all services in the stack
docker stack services myapp

# Get detailed information about services
docker service ls

# Check service logs
docker service logs myapp_web
Step 3.5: Test the Application
# Test the web application
curl http://localhost

# Test through nginx proxy
curl http://localhost:8080

# Check health endpoint
curl http://localhost/health
Task 4: Scale Services in Swarm Mode
Step 4.1: Scale Web Service
# Scale the web service to 5 replicas
docker service scale myapp_web=5

# Verify scaling
docker service ps myapp_web
Step 4.2: Scale Using Docker Compose
# Update docker-compose.yml to change replica count
sed -i 's/replicas: 3/replicas: 6/' docker-compose.yml

# Redeploy with updated configuration
docker stack deploy -c docker-compose.yml myapp
Step 4.3: Scale Nginx Service
# Scale nginx service
docker service scale myapp_nginx=4

# Check service distribution across nodes
docker service ps myapp_nginx
Step 4.4: Monitor Scaling Effects
# Monitor service status
watch -n 2 'docker service ls'

# Test load distribution
for i in {1..10}; do
    curl -s http://localhost | grep hostname
    sleep 1
done
Task 5: Monitor the Application and Troubleshoot Common Issues
Step 5.1: Monitor Service Health
# Check overall stack status
docker stack ps myapp

# Monitor service logs in real-time
docker service logs -f myapp_web

# Check node resource usage
docker node ls
docker system df
Step 5.2: Inspect Service Details
# Inspect web service configuration
docker service inspect myapp_web --pretty

# Check service tasks
docker service ps myapp_web --no-trunc

# Inspect network configuration
docker network ls
docker network inspect myapp_app-network
Step 5.3: Common Troubleshooting Commands
# Check failed tasks
docker service ps myapp_web --filter "desired-state=running"

# View system events
docker system events --filter type=service

# Check container logs on specific nodes
docker ps
docker logs <container-id>

# Inspect node constraints
docker node inspect <node-id> --pretty
Step 5.4: Performance Monitoring
# Monitor resource usage
docker stats

# Check service resource constraints
docker service inspect myapp_web --format '{{.Spec.TaskTemplate.Resources}}'

# Monitor network traffic
docker network inspect myapp_app-network --format '{{.Containers}}'
Step 5.5: Troubleshooting Common Issues
Issue 1: Service Not Starting

# Check service events
docker service ps myapp_web --no-trunc

# Check node availability
docker node ls

# Verify image availability
docker images
Issue 2: Network Connectivity Problems

# Test network connectivity between services
docker exec -it $(docker ps -q --filter name=myapp_web) ping redis

# Check overlay network
docker network inspect myapp_app-network
Issue 3: Load Balancing Issues

# Test service discovery
docker exec -it $(docker ps -q --filter name=myapp_web) nslookup redis

# Check service endpoints
docker service inspect myapp_web --format '{{.Endpoint}}'
Step 5.6: Update and Rollback Services
# Update web service with new image
docker service update --image swarm-web:v2.0 myapp_web

# Monitor update progress
docker service ps myapp_web

# Rollback if needed
docker service rollback myapp_web
Step 5.7: Clean Up Resources
# Remove the stack
docker stack rm myapp

# Remove unused images and volumes
docker system prune -f

# Remove configs
docker config rm nginx-config

# Leave swarm (on worker nodes)
# docker swarm leave

# Remove swarm (on manager node - only if needed)
# docker swarm leave --force
Advanced Monitoring and Maintenance
Health Checks and Auto-Recovery
# Add health check to docker-compose.yml
cat >> docker-compose.yml << 'EOF'
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
EOF
Resource Constraints
# Update service with resource limits
docker service update \
  --limit-cpu 0.5 \
  --limit-memory 512M \
  --reserve-cpu 0.25 \
  --reserve-memory 256M \
  myapp_web
Conclusion
Congratulations! You have successfully completed Lab 41: Docker Swarm - Deploying a Multi-Service Application.

What You Accomplished:

• Swarm Cluster Setup: You initialized a Docker Swarm cluster with one manager and two worker nodes, learning the fundamentals of container orchestration.

• Multi-Service Application: You created a complete multi-service application with a Python web app, Redis database, and Nginx load balancer, demonstrating real-world application architecture.

• Stack Deployment: You used docker-compose.yml and docker stack deploy to deploy complex applications across multiple nodes, understanding declarative deployment strategies.

• Service Scaling: You learned to scale services both manually and through configuration updates, experiencing how Docker Swarm handles load distribution automatically.

• Monitoring and Troubleshooting: You gained hands-on experience with monitoring tools and troubleshooting techniques essential for production environments.

Why This Matters:

Docker Swarm provides a native clustering and orchestration solution that's simpler than Kubernetes but powerful enough for many production workloads. The skills you've learned are directly applicable to:

• Production Deployments: Managing containerized applications in production environments • High Availability: Ensuring applications remain available even when individual nodes fail • Scalability: Automatically scaling applications based on demand • DevOps Practices: Implementing infrastructure as code and automated deployments

These skills are fundamental for the Docker Certified Associate (DCA) certification and are highly valued in modern DevOps and cloud engineering roles. You now have practical experience with container orchestration that you can apply to real-world projects and continue building upon as you advance in your containerization journey.
