Lab 59: Docker for High Availability - Setting up Multi-Node Clusters
Lab Objectives
By the end of this lab, you will be able to:

• Set up a multi-node Docker Swarm cluster across multiple machines • Deploy highly available web applications using Docker Swarm mode • Test and verify failover capabilities when nodes become unavailable • Configure health checks and automatic container restart policies • Implement automatic load balancing for even traffic distribution • Understand the fundamentals of container orchestration for high availability

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker containers and images • Familiarity with Linux command line operations • Knowledge of basic networking concepts • Understanding of web applications and HTTP protocols • Experience with text editors like nano or vim

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own virtual machines or install Docker manually.

Your lab environment includes: • 3 Ubuntu 20.04 LTS machines with Docker pre-installed • Machine 1: manager-node (192.168.1.10) • Machine 2: worker-node-1 (192.168.1.11) • Machine 3: worker-node-2 (192.168.1.12) • All necessary networking configured between machines

Task 1: Set up a Multi-Node Docker Swarm Cluster
Subtask 1.1: Initialize the Swarm Manager Node
First, we'll set up the primary manager node that will coordinate the entire cluster.

Connect to the manager node:
# You should already be connected to manager-node
# Verify Docker is running
sudo systemctl status docker
Initialize Docker Swarm on the manager node:
# Initialize swarm with the manager node's IP address
sudo docker swarm init --advertise-addr 192.168.1.10

# The output will show a join token - save this for later steps
Verify swarm initialization:
# Check swarm status
sudo docker node ls

# You should see one node listed as the Leader
Subtask 1.2: Join Worker Nodes to the Swarm
Now we'll add the worker nodes to our swarm cluster.

Get the worker join token from manager node:
# On manager-node, get the join token for workers
sudo docker swarm join-token worker
Connect to worker-node-1 and join the swarm:
# Open a new terminal or SSH session to worker-node-1
# Use the join command from the previous step (example below)
sudo docker swarm join --token SWMTKN-1-xxxxx 192.168.1.10:2377
Connect to worker-node-2 and join the swarm:
# Open another terminal or SSH session to worker-node-2
# Use the same join command
sudo docker swarm join --token SWMTKN-1-xxxxx 192.168.1.10:2377
Verify all nodes joined successfully:
# Back on manager-node, check all nodes
sudo docker node ls

# You should see 3 nodes: 1 manager and 2 workers
Subtask 1.3: Configure Node Labels and Roles
Let's add labels to our nodes for better organization and deployment control.

Add labels to worker nodes:
# On manager-node, add labels to identify node roles
sudo docker node update --label-add role=frontend worker-node-1
sudo docker node update --label-add role=backend worker-node-2

# Verify labels were added
sudo docker node inspect worker-node-1 --format '{{.Spec.Labels}}'
sudo docker node inspect worker-node-2 --format '{{.Spec.Labels}}'
Task 2: Deploy a Highly Available Web Application with Swarm Mode
Subtask 2.1: Create a Docker Compose File for the Application
We'll deploy a simple web application with multiple replicas for high availability.

Create the application directory:
# On manager-node, create a directory for our application
mkdir ~/ha-webapp
cd ~/ha-webapp
Create a simple web application:
# Create a simple HTML file
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>High Availability Web App</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; }
        .container { max-width: 600px; margin: 0 auto; }
        .hostname { color: #007acc; font-weight: bold; }
    </style>
</head>
<body>
    <div class="container">
        <h1>High Availability Web Application</h1>
        <p>This application is running in Docker Swarm mode</p>
        <p>Server hostname: <span class="hostname" id="hostname"></span></p>
        <p>Current time: <span id="time"></span></p>
    </div>
    <script>
        document.getElementById('hostname').textContent = window.location.hostname;
        document.getElementById('time').textContent = new Date().toLocaleString();
        setInterval(() => {
            document.getElementById('time').textContent = new Date().toLocaleString();
        }, 1000);
    </script>
</body>
</html>
EOF
Create a Dockerfile:
# Create Dockerfile for our web application
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/ || exit 1
EOF
Create a Docker Compose file for Swarm deployment:
# Create docker-compose.yml for swarm stack
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    build: .
    ports:
      - "80:80"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      placement:
        max_replicas_per_node: 2
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - webnet

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
    networks:
      - webnet

networks:
  webnet:
    driver: overlay
EOF
Subtask 2.2: Build and Deploy the Application Stack
Build the application image:
# Build the Docker image
sudo docker build -t ha-webapp:latest .

# Verify the image was created
sudo docker images | grep ha-webapp
Deploy the stack to the swarm:
# Deploy the stack using docker stack deploy
sudo docker stack deploy -c docker-compose.yml webapp-stack

# Verify the stack was deployed
sudo docker stack ls
Check service status:
# List all services in the stack
sudo docker stack services webapp-stack

# Get detailed information about the web service
sudo docker service ls
sudo docker service ps webapp-stack_web
Subtask 2.3: Verify Load Balancing
Test the web application:
# Test the application from manager node
curl http://192.168.1.10

# Test multiple times to see load balancing
for i in {1..10}; do
    echo "Request $i:"
    curl -s http://192.168.1.10 | grep -o "Server hostname.*"
    sleep 1
done
Access the visualizer:
# The visualizer should be accessible at:
echo "Access the visualizer at: http://192.168.1.10:8080"
Task 3: Test Failover by Shutting Down One of the Nodes
Subtask 3.1: Monitor Current Service Distribution
Check current service distribution:
# See which nodes are running which containers
sudo docker service ps webapp-stack_web

# Check node status
sudo docker node ls
Monitor services in real-time:
# Open a new terminal and run this command to monitor services
watch -n 2 'sudo docker service ps webapp-stack_web'
Subtask 3.2: Simulate Node Failure
Shut down worker-node-1:
# Connect to worker-node-1 and shut it down
# On worker-node-1 terminal:
sudo systemctl stop docker
# Or simulate complete node failure:
sudo shutdown -h now
Observe failover behavior:
# On manager-node, watch the service redistribution
sudo docker service ps webapp-stack_web

# Check node status
sudo docker node ls

# The failed node should show as "Down"
Test application availability during failover:
# Test that the application is still accessible
for i in {1..20}; do
    echo "Request $i at $(date):"
    curl -s http://192.168.1.10 || echo "Request failed"
    sleep 2
done
Subtask 3.3: Verify Automatic Recovery
Restart the failed node:
# If you shut down worker-node-1, restart it
# The node should automatically rejoin the swarm
Monitor service rebalancing:
# Watch as services potentially rebalance
sudo docker service ps webapp-stack_web

# Check that all nodes are back online
sudo docker node ls
Task 4: Configure Health Checks and Automatic Container Restarts
Subtask 4.1: Enhance Health Check Configuration
Create an enhanced health check script:
# Create a more comprehensive health check
cat > healthcheck.sh << 'EOF'
#!/bin/bash
# Enhanced health check script

# Check if nginx is running
if ! pgrep nginx > /dev/null; then
    echo "Nginx process not found"
    exit 1
fi

# Check if the web page is accessible
if ! curl -f -s http://localhost/ > /dev/null; then
    echo "Web page not accessible"
    exit 1
fi

# Check system resources
MEMORY_USAGE=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100}')
if [ "$MEMORY_USAGE" -gt 90 ]; then
    echo "Memory usage too high: ${MEMORY_USAGE}%"
    exit 1
fi

echo "Health check passed"
exit 0
EOF

chmod +x healthcheck.sh
Update the Dockerfile with enhanced health check:
# Create an updated Dockerfile
cat > Dockerfile << 'EOF'
FROM nginx:alpine

# Install curl for health checks
RUN apk add --no-cache curl

# Copy application files
COPY index.html /usr/share/nginx/html/
COPY healthcheck.sh /usr/local/bin/

EXPOSE 80

# Enhanced health check
HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
    CMD /usr/local/bin/healthcheck.sh

CMD ["nginx", "-g", "daemon off;"]
EOF
Subtask 4.2: Update Service with Enhanced Restart Policies
Create an updated compose file with better restart policies:
# Create enhanced docker-compose.yml
cat > docker-compose-enhanced.yml << 'EOF'
version: '3.8'

services:
  web:
    build: .
    ports:
      - "80:80"
    deploy:
      replicas: 4
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        monitor: 60s
        max_failure_ratio: 0.3
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 5
        window: 120s
      placement:
        max_replicas_per_node: 2
        preferences:
          - spread: node.labels.role
    healthcheck:
      test: ["/usr/local/bin/healthcheck.sh"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 10s
    networks:
      - webnet
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    networks:
      - webnet

networks:
  webnet:
    driver: overlay
    attachable: true
EOF
Update the deployed stack:
# Remove the old stack
sudo docker stack rm webapp-stack

# Wait for cleanup
sleep 30

# Rebuild the image with enhancements
sudo docker build -t ha-webapp:enhanced .

# Deploy the enhanced stack
sudo docker stack deploy -c docker-compose-enhanced.yml webapp-stack-enhanced
Subtask 4.3: Test Health Check and Restart Functionality
Monitor health check status:
# Check service health
sudo docker service ps webapp-stack-enhanced_web

# Inspect a specific container's health
CONTAINER_ID=$(sudo docker ps --format "table {{.ID}}\t{{.Image}}" | grep ha-webapp | head -1 | awk '{print $1}')
sudo docker inspect $CONTAINER_ID | grep -A 10 "Health"
Simulate container failure:
# Find a running container
sudo docker ps | grep ha-webapp

# Kill nginx process inside a container to trigger health check failure
CONTAINER_ID=$(sudo docker ps --format "table {{.ID}}\t{{.Image}}" | grep ha-webapp | head -1 | awk '{print $1}')
sudo docker exec $CONTAINER_ID pkill nginx

# Watch the container get restarted
watch -n 2 'sudo docker service ps webapp-stack-enhanced_web'
Task 5: Set up Automatic Load Balancing to Distribute Traffic Evenly
Subtask 5.1: Configure Advanced Load Balancing
Create a load balancer configuration:
# Create HAProxy configuration for advanced load balancing
mkdir ~/load-balancer
cd ~/load-balancer

cat > haproxy.cfg << 'EOF'
global
    daemon
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms
    option httplog
    option dontlognull
    option redispatch
    retries 3

frontend web_frontend
    bind *:8081
    default_backend web_servers
    
    # Health check endpoint
    monitor-uri /health

backend web_servers
    balance roundrobin
    option httpchk GET /
    
    # Add all swarm nodes as potential backends
    server web1 192.168.1.10:80 check inter 2000ms rise 2 fall 3
    server web2 192.168.1.11:80 check inter 2000ms rise 2 fall 3
    server web3 192.168.1.12:80 check inter 2000ms rise 2 fall 3

stats enable
stats uri /stats
stats refresh 30s
stats admin if TRUE
EOF
Create HAProxy Docker service:
# Create Dockerfile for HAProxy
cat > Dockerfile << 'EOF'
FROM haproxy:alpine
COPY haproxy.cfg /usr/local/etc/haproxy/haproxy.cfg
EXPOSE 8081 8404
EOF

# Build HAProxy image
sudo docker build -t custom-haproxy .
Subtask 5.2: Deploy Load Balancer as a Service
Add load balancer to the stack:
# Update compose file to include load balancer
cat > docker-compose-final.yml << 'EOF'
version: '3.8'

services:
  web:
    build: 
      context: ~/ha-webapp
      dockerfile: Dockerfile
    ports:
      - "80:80"
    deploy:
      replicas: 4
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
        monitor: 60s
        max_failure_ratio: 0.3
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 5
        window: 120s
      placement:
        max_replicas_per_node: 2
    healthcheck:
      test: ["/usr/local/bin/healthcheck.sh"]
      interval: 15s
      timeout: 5s
      retries: 3
      start_period: 10s
    networks:
      - webnet

  loadbalancer:
    build: 
      context: ~/load-balancer
      dockerfile: Dockerfile
    ports:
      - "8081:8081"
      - "8404:8404"
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    networks:
      - webnet
    depends_on:
      - web

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints: [node.role == manager]
      restart_policy:
        condition: on-failure
    networks:
      - webnet

networks:
  webnet:
    driver: overlay
    attachable: true
EOF
Deploy the complete stack:
# Remove previous stack
sudo docker stack rm webapp-stack-enhanced

# Wait for cleanup
sleep 30

# Deploy final stack with load balancer
sudo docker stack deploy -c docker-compose-final.yml webapp-final
Subtask 5.3: Test Load Balancing and Monitor Traffic Distribution
Test load balancing functionality:
# Test the load balancer
for i in {1..20}; do
    echo "Request $i:"
    curl -s http://192.168.1.10:8081 | grep -o "Server hostname.*" || echo "Request failed"
    sleep 1
done
Monitor load balancer statistics:
# Access HAProxy stats page
echo "HAProxy statistics available at: http://192.168.1.10:8404/stats"

# Test load balancer health endpoint
curl http://192.168.1.10:8081/health
Perform comprehensive load testing:
# Install Apache Bench for load testing
sudo apt-get update
sudo apt-get install -y apache2-utils

# Perform load test
ab -n 1000 -c 10 http://192.168.1.10:8081/

# Monitor during load test
watch -n 1 'sudo docker service ps webapp-final_web'
Subtask 5.4: Test Complete High Availability Scenario
Simulate multiple failure scenarios:
# Create a comprehensive test script
cat > ha-test.sh << 'EOF'
#!/bin/bash

echo "Starting High Availability Test..."

# Function to test application availability
test_availability() {
    local test_name=$1
    echo "Testing: $test_name"
    
    for i in {1..10}; do
        response=$(curl -s -o /dev/null -w "%{http_code}" http://192.168.1.10:8081)
        if [ "$response" = "200" ]; then
            echo "  Request $i: SUCCESS"
        else
            echo "  Request $i: FAILED (HTTP $response)"
        fi
        sleep 1
    done
}

# Test 1: Normal operation
test_availability "Normal Operation"

# Test 2: Simulate container failure
echo "Simulating container failure..."
CONTAINER_ID=$(sudo docker ps --format "table {{.ID}}\t{{.Image}}" | grep ha-webapp | head -1 | awk '{print $1}')
sudo docker kill $CONTAINER_ID
sleep 5
test_availability "After Container Failure"

# Test 3: Scale down and up
echo "Testing scaling..."
sudo docker service scale webapp-final_web=2
sleep 10
test_availability "After Scaling Down"

sudo docker service scale webapp-final_web=4
sleep 10
test_availability "After Scaling Up"

echo "High Availability Test Complete!"
EOF

chmod +x ha-test.sh
./ha-test.sh
Monitor cluster health:
# Create monitoring script
cat > monitor-cluster.sh << 'EOF'
#!/bin/bash

while true; do
    clear
    echo "=== Docker Swarm Cluster Status ==="
    echo "Timestamp: $(date)"
    echo
    
    echo "=== Nodes ==="
    sudo docker node ls
    echo
    
    echo "=== Services ==="
    sudo docker service ls
    echo
    
    echo "=== Service Tasks ==="
    sudo docker service ps webapp-final_web --no-trunc
    echo
    
    sleep 5
done
EOF

chmod +x monitor-cluster.sh
# Run in background: ./monitor-cluster.sh &
Troubleshooting Common Issues
Issue 1: Nodes Not Joining Swarm
Problem: Worker nodes fail to join the swarm cluster.

Solution:

# Check firewall settings
sudo ufw status

# Ensure required ports are open (2377, 7946, 4789)
sudo ufw allow 2377
sudo ufw allow 7946
sudo ufw allow 4789

# Regenerate join token if needed
sudo docker swarm join-token worker
Issue 2: Services Not Starting
Problem: Services remain in pending state.

Solution:

# Check service logs
sudo docker service logs webapp-final_web

# Check node resources
sudo docker node ls
sudo docker system df

# Inspect service constraints
sudo docker service inspect webapp-final_web
Issue 3: Load Balancer Not Working
Problem: Load balancer not distributing traffic evenly.

Solution:

# Check HAProxy logs
sudo docker service logs webapp-final_loadbalancer

# Verify network connectivity
sudo docker network ls
sudo docker network inspect webapp-final_webnet

# Test individual backend servers
curl http://192.168.1.10:80
curl http://192.168.1.11:80
curl http://192.168.1.12:80
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Remove all stacks
sudo docker stack rm webapp-final

# Leave swarm (on worker nodes first)
sudo docker swarm leave

# Leave swarm on manager (force)
sudo docker swarm leave --force

# Remove unused images and containers
sudo docker system prune -a
Conclusion
Congratulations! You have successfully completed Lab 59: Docker for High Availability - Setting up Multi-Node Clusters.

What You Accomplished
In this lab, you have:

• Built a Production-Ready Cluster: Set up a multi-node Docker Swarm cluster with proper node roles and labels • Deployed Highly Available Applications: Created and deployed web applications with multiple replicas across different nodes • Implemented Failover Testing: Verified that your applications continue running even when nodes fail • Configured Advanced Health Monitoring: Set up comprehensive health checks and automatic restart policies • Established Load Balancing: Implemented both Docker Swarm's built-in load balancing and external load balancing with HAProxy

Why This Matters
High availability is crucial in modern application deployment because:

• Business Continuity: Applications remain accessible even during hardware failures or maintenance • Improved User Experience: Users experience minimal downtime and consistent performance • Scalability: Applications can handle increased load by distributing traffic across multiple instances • Cost Efficiency: Automated failover and scaling reduce the need for manual intervention • Professional Readiness: These skills are essential for Docker Certified Associate certification and real-world container orchestration

Next Steps
To further develop your Docker and container orchestration skills:

• Explore Kubernetes as an alternative orchestration platform • Learn about persistent storage in containerized environments • Study advanced networking concepts in container clusters • Practice with monitoring and logging solutions for containerized applications • Investigate CI/CD pipelines for containerized applications

This lab has provided you with hands-on experience in building resilient, scalable applications using Docker Swarm - skills that are directly applicable to production environments and valuable for your career in DevOps and cloud computing.
