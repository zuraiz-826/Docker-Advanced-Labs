Lab 110: Docker in Production - Deploying Docker Containers in High Availability Setup
Lab Objectives
By the end of this lab, you will be able to:

Set up a Docker Swarm cluster with multiple nodes for high availability
Deploy multi-container applications across multiple nodes
Configure load balancing to distribute traffic effectively
Test and verify failover capabilities when nodes become unavailable
Implement automated scaling to handle varying traffic loads
Understand production-ready Docker deployment strategies
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with Linux command line operations
Knowledge of basic networking concepts
Understanding of YAML file structure
Experience with text editors like nano or vim
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines for this lab. Simply click Start Lab to access your pre-configured environment. No need to build your own VM or install Docker - everything is ready to use.

Your lab environment includes:

3 Ubuntu 20.04 LTS virtual machines
Docker Engine pre-installed on all nodes
Network connectivity between all machines
Root access to all systems
Task 1: Set up Docker Swarm with Multiple Nodes
Subtask 1.1: Initialize Docker Swarm Manager
First, we'll set up the Docker Swarm cluster by initializing the manager node.

Connect to the first machine (Manager Node):

# Check Docker installation
docker --version

# Check system information
hostname
ip addr show
Initialize Docker Swarm:

# Initialize swarm mode (replace with your actual IP)
docker swarm init --advertise-addr $(hostname -I | awk '{print $1}')
Verify swarm initialization:

# Check swarm status
docker info | grep Swarm

# List swarm nodes
docker node ls
Subtask 1.2: Join Worker Nodes to the Swarm
Get the join token from manager node:

# Display worker join token
docker swarm join-token worker
Connect to the second machine (Worker Node 1) and join the swarm:

# Use the join command from previous step (example format)
docker swarm join --token SWMTKN-1-xxxxx manager-ip:2377
Connect to the third machine (Worker Node 2) and join the swarm:

# Use the same join command
docker swarm join --token SWMTKN-1-xxxxx manager-ip:2377
Verify all nodes joined (run on manager):

# Check all nodes in swarm
docker node ls

# Get detailed node information
docker node inspect $(docker node ls -q) --pretty
Subtask 1.3: Configure Node Labels
Add labels to nodes for better organization:

# Label nodes based on their roles
docker node update --label-add role=frontend worker1-hostname
docker node update --label-add role=backend worker2-hostname
docker node update --label-add role=database manager-hostname
Verify node labels:

# Check node labels
docker node ls --format "table {{.Hostname}}\t{{.Status}}\t{{.Availability}}"
docker node inspect worker1-hostname --format '{{.Spec.Labels}}'
Task 2: Deploy a Multi-Container Application Across Nodes
Subtask 2.1: Create Application Stack Definition
Create a directory for the application:

mkdir -p ~/docker-ha-app
cd ~/docker-ha-app
Create a Docker Compose stack file:

nano docker-stack.yml
Add the following stack configuration:

version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    deploy:
      replicas: 3
      placement:
        constraints:
          - node.labels.role == frontend
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
    networks:
      - app-network

  api:
    image: httpd:alpine
    deploy:
      replicas: 2
      placement:
        constraints:
          - node.labels.role == backend
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    networks:
      - app-network

  database:
    image: postgres:13-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: securepassword
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.labels.role == database
      restart_policy:
        condition: on-failure
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-network

configs:
  nginx_config:
    external: true

volumes:
  db-data:
    driver: local

networks:
  app-network:
    driver: overlay
    attachable: true
Subtask 2.2: Create Nginx Configuration
Create Nginx configuration file:

nano nginx.conf
Add load balancing configuration:

events {
    worker_connections 1024;
}

http {
    upstream backend {
        server api:80;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            index index.html;
        }

        location /api/ {
            proxy_pass http://backend/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
Create the config in Docker Swarm:

# Create nginx config in swarm
docker config create nginx_config nginx.conf

# Verify config creation
docker config ls
Subtask 2.3: Deploy the Application Stack
Deploy the stack:

# Deploy the application stack
docker stack deploy -c docker-stack.yml ha-app
Verify stack deployment:

# List deployed stacks
docker stack ls

# List services in the stack
docker stack services ha-app

# Check service details
docker service ls
Monitor service deployment:

# Watch service status
watch docker service ls

# Check specific service details
docker service ps ha-app_web
docker service ps ha-app_api
docker service ps ha-app_database
Task 3: Set up Load Balancing to Distribute Traffic
Subtask 3.1: Configure External Load Balancer
Install HAProxy on the manager node:

# Update package list
apt update

# Install HAProxy
apt install -y haproxy
Create HAProxy configuration:

# Backup original config
cp /etc/haproxy/haproxy.cfg /etc/haproxy/haproxy.cfg.backup

# Create new configuration
nano /etc/haproxy/haproxy.cfg
Add HAProxy configuration:

global
    daemon
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog

frontend web_frontend
    bind *:8080
    default_backend web_servers

backend web_servers
    balance roundrobin
    option httpchk GET /
    server web1 node1-ip:80 check
    server web2 node2-ip:80 check
    server web3 node3-ip:80 check

stats enable
stats uri /stats
stats refresh 30s
Start and enable HAProxy:

# Start HAProxy service
systemctl start haproxy
systemctl enable haproxy

# Check HAProxy status
systemctl status haproxy
Subtask 3.2: Test Load Balancing
Create a simple test page for each service:

# Create custom index pages for testing
docker service update --config-rm nginx_config ha-app_web

# Create test script
cat > test-lb.sh << 'EOF'
#!/bin/bash
echo "Testing load balancing..."
for i in {1..10}; do
    echo "Request $i:"
    curl -s http://localhost:8080 | grep -o "Server: [^<]*" || echo "Response received"
    sleep 1
done
EOF

chmod +x test-lb.sh
./test-lb.sh
Monitor load balancing statistics:

# Access HAProxy stats (in browser or curl)
curl http://localhost:8080/stats
Task 4: Test Failover by Stopping a Node
Subtask 4.1: Monitor Current Service Distribution
Check current service placement:

# Show where services are running
docker service ps ha-app_web --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
docker service ps ha-app_api --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
Create monitoring script:

cat > monitor-services.sh << 'EOF'
#!/bin/bash
while true; do
    clear
    echo "=== Service Status ==="
    docker service ls
    echo ""
    echo "=== Node Status ==="
    docker node ls
    echo ""
    echo "=== Web Service Distribution ==="
    docker service ps ha-app_web --format "table {{.Name}}\t{{.Node}}\t{{.CurrentState}}"
    sleep 5
done
EOF

chmod +x monitor-services.sh
Subtask 4.2: Simulate Node Failure
Start monitoring in one terminal:

# Run monitoring script
./monitor-services.sh
In another terminal, simulate node failure:

# Drain a worker node (simulate maintenance)
docker node update --availability drain worker1-hostname

# Check node status
docker node ls
Observe service redistribution:

# Watch services move to available nodes
docker service ps ha-app_web

# Check service logs
docker service logs ha-app_web
Subtask 4.3: Test Service Continuity
Test application availability during failover:

# Create continuous testing script
cat > test-availability.sh << 'EOF'
#!/bin/bash
echo "Testing service availability during failover..."
success=0
total=0

for i in {1..60}; do
    total=$((total + 1))
    if curl -s -o /dev/null -w "%{http_code}" http://localhost:8080 | grep -q "200"; then
        success=$((success + 1))
        echo "Request $i: SUCCESS"
    else
        echo "Request $i: FAILED"
    fi
    sleep 2
done

echo "Availability: $success/$total ($(echo "scale=2; $success*100/$total" | bc)%)"
EOF

chmod +x test-availability.sh
./test-availability.sh
Restore the drained node:

# Restore node availability
docker node update --availability active worker1-hostname

# Verify node restoration
docker node ls
Task 5: Automate Scaling to Handle High Traffic
Subtask 5.1: Configure Service Scaling
Scale services manually first:

# Scale web service up
docker service scale ha-app_web=5

# Scale API service
docker service scale ha-app_api=4

# Check scaling progress
docker service ls
docker service ps ha-app_web
Create scaling script:

cat > auto-scale.sh << 'EOF'
#!/bin/bash

# Function to get current load (simplified example)
get_load() {
    # In production, this would check actual metrics
    # For demo, we'll use random values
    echo $((RANDOM % 100))
}

# Function to scale services based on load
scale_services() {
    load=$(get_load)
    echo "Current load: $load%"
    
    if [ $load -gt 80 ]; then
        echo "High load detected, scaling up..."
        docker service scale ha-app_web=6 ha-app_api=4
    elif [ $load -lt 30 ]; then
        echo "Low load detected, scaling down..."
        docker service scale ha-app_web=2 ha-app_api=2
    else
        echo "Normal load, maintaining current scale"
    fi
}

# Main monitoring loop
while true; do
    scale_services
    echo "Waiting 30 seconds before next check..."
    sleep 30
done
EOF

chmod +x auto-scale.sh
Subtask 5.2: Implement Resource Constraints
Update stack with resource limits:

# Create updated stack file with resource constraints
cat > docker-stack-resources.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 128M
        reservations:
          cpus: '0.25'
          memory: 64M
      placement:
        constraints:
          - node.labels.role == frontend
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    networks:
      - app-network

  api:
    image: httpd:alpine
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M
      placement:
        constraints:
          - node.labels.role == backend
      restart_policy:
        condition: on-failure
    networks:
      - app-network

  database:
    image: postgres:13-alpine
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: appuser
      POSTGRES_PASSWORD: securepassword
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
      placement:
        constraints:
          - node.labels.role == database
    volumes:
      - db-data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  db-data:
    driver: local

networks:
  app-network:
    driver: overlay
    attachable: true
EOF
Update the stack with resource constraints:

# Update the existing stack
docker stack deploy -c docker-stack-resources.yml ha-app

# Monitor the update
docker service ls
Subtask 5.3: Create Load Testing and Monitoring
Install load testing tools:

# Install Apache Bench for load testing
apt install -y apache2-utils

# Install monitoring tools
apt install -y htop iotop
Create load testing script:

cat > load-test.sh << 'EOF'
#!/bin/bash

echo "Starting load test..."
echo "Testing with 100 concurrent connections, 1000 total requests"

# Run load test
ab -n 1000 -c 100 http://localhost:8080/

echo "Load test completed. Check service scaling..."
docker service ls
EOF

chmod +x load-test.sh
Monitor system resources during load test:

# Create resource monitoring script
cat > monitor-resources.sh << 'EOF'
#!/bin/bash

while true; do
    clear
    echo "=== System Resources ==="
    echo "CPU Usage:"
    top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1
    
    echo "Memory Usage:"
    free -h
    
    echo "=== Docker Services ==="
    docker service ls
    
    echo "=== Container Stats ==="
    docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}"
    
    sleep 5
done
EOF

chmod +x monitor-resources.sh
Troubleshooting Tips
Common Issues and Solutions
Swarm initialization fails:

# Check firewall settings
ufw status

# Open required ports
ufw allow 2377/tcp
ufw allow 7946/tcp
ufw allow 7946/udp
ufw allow 4789/udp
Services not starting:

# Check service logs
docker service logs ha-app_web

# Inspect service configuration
docker service inspect ha-app_web --pretty
Node communication issues:

# Test connectivity between nodes
ping node-ip

# Check Docker daemon status
systemctl status docker
Load balancer not working:

# Check HAProxy logs
tail -f /var/log/haproxy.log

# Test backend connectivity
curl -I http://node-ip:80
Lab Verification
Verification Checklist
Verify Swarm Cluster:

# Should show 3 nodes (1 manager, 2 workers)
docker node ls
Verify Service Deployment:

# Should show all services running
docker service ls
docker stack services ha-app
Verify Load Balancing:

# Test multiple requests
for i in {1..5}; do curl -s http://localhost:8080; done
Verify Failover:

# Drain a node and verify services move
docker node update --availability drain worker1-hostname
docker service ps ha-app_web
Verify Scaling:

# Scale service and verify
docker service scale ha-app_web=4
docker service ls
Conclusion
Congratulations! You have successfully completed Lab 110: Docker in Production - Deploying Docker Containers in High Availability Setup.

What You Accomplished
In this lab, you have:

Set up a production-ready Docker Swarm cluster with multiple nodes, providing the foundation for high availability deployments
Deployed a multi-container application across multiple nodes with proper service distribution and placement constraints
Configured load balancing using both Docker Swarm's built-in routing mesh and external HAProxy load balancer
Tested failover scenarios by simulating node failures and observing automatic service redistribution
Implemented automated scaling strategies to handle varying traffic loads with resource constraints
Why This Matters
High availability Docker deployments are crucial for production environments because they:

Ensure service continuity even when individual nodes fail
Distribute load effectively across multiple servers to prevent bottlenecks
Provide automatic recovery mechanisms that reduce downtime
Enable horizontal scaling to handle traffic spikes
Offer production-grade reliability for business-critical applications
Next Steps
To further enhance your Docker production skills, consider:

Implementing monitoring and alerting with Prometheus and Grafana
Setting up centralized logging with ELK stack
Exploring Kubernetes as an alternative orchestration platform
Learning about Docker security best practices
Implementing CI/CD pipelines for automated deployments
This lab has provided you with hands-on experience in deploying and managing Docker containers in a high availability production environment, preparing you for real-world scenarios and the Docker Certified Associate (DCA) certification.
