Lab 26: Docker Swarm Mode - Setting up Docker Swarm
Lab Objectives
By the end of this lab, students will be able to:

Understand the fundamentals of Docker Swarm mode and container orchestration
Initialize a Docker Swarm cluster on a manager node
Join worker nodes to an existing Docker Swarm cluster
Deploy and manage services across multiple nodes in a swarm
Scale services up and down based on requirements
Monitor swarm nodes and services effectively
Troubleshoot common Docker Swarm issues
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and images
Familiarity with Linux command line operations
Knowledge of basic networking concepts
Understanding of YAML file structure
Completed previous Docker labs covering container basics
Required Knowledge Areas
Docker container lifecycle management
Basic Linux system administration
Network port concepts
Service discovery fundamentals
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own virtual machines or install Docker manually.

Environment Details
Operating System: Ubuntu 20.04 LTS
Docker Version: Latest stable version pre-installed
Network Configuration: Multi-node setup with proper connectivity
Access Method: SSH access to multiple nodes
Task 1: Initialize Docker Swarm on Manager Node
Subtask 1.1: Verify Docker Installation and Node Status
First, let's verify that Docker is properly installed and check the current swarm status.

# Check Docker version
docker --version

# Check current swarm status
docker info | grep -i swarm
Expected Output: You should see Docker version information and "Swarm: inactive" indicating the node is not part of a swarm yet.

Subtask 1.2: Initialize Docker Swarm
Initialize the Docker Swarm on the first node, which will become the manager node.

# Initialize Docker Swarm (replace with your actual IP address)
docker swarm init --advertise-addr $(hostname -I | awk '{print $1}')

# Alternative method if you know the specific IP
# docker swarm init --advertise-addr 192.168.1.100
Key Concepts:

Manager Node: Controls the swarm and makes scheduling decisions
Advertise Address: The IP address other nodes will use to connect to this manager
Subtask 1.3: Verify Swarm Initialization
Check that the swarm has been successfully initialized.

# Check swarm status
docker info | grep -i swarm

# List nodes in the swarm
docker node ls

# Get detailed information about the current node
docker node inspect self --pretty
Expected Results:

Swarm status should show "active"
Node list should show one manager node
Current node should have "Leader" status
Task 2: Join Additional Nodes to the Swarm
Subtask 2.1: Retrieve Join Tokens
From the manager node, get the join tokens for both worker and manager nodes.

# Get worker join token
docker swarm join-token worker

# Get manager join token (for adding additional managers)
docker swarm join-token manager

# Save tokens to variables for easy access
WORKER_TOKEN=$(docker swarm join-token worker -q)
MANAGER_TOKEN=$(docker swarm join-token manager -q)

echo "Worker Token: $WORKER_TOKEN"
echo "Manager Token: $MANAGER_TOKEN"
Subtask 2.2: Join Worker Nodes to the Swarm
Connect to your second node (worker node) and join it to the swarm.

# On the worker node, execute the join command
# Replace <MANAGER-IP> with the actual IP address of your manager node
# Replace <TOKEN> with the actual worker token from previous step

docker swarm join --token <WORKER_TOKEN> <MANAGER-IP>:2377

# Example:
# docker swarm join --token SWMTKN-1-49nj1cmql0jkz5s954yi3oex3nedyz0fb0xx14ie39trti4wxv-8vxv8rssmk743ojnwacrr2e7c 192.168.1.100:2377
Subtask 2.3: Verify Node Addition
Return to the manager node and verify that the worker node has joined successfully.

# List all nodes in the swarm
docker node ls

# Get detailed information about all nodes
docker node inspect $(docker node ls -q) --pretty
Expected Results:

You should see both manager and worker nodes listed
Worker node should show "Ready" status
Manager node should maintain "Leader" status
Subtask 2.4: Add Additional Worker Nodes (Optional)
If you have access to a third node, repeat the join process to create a more robust swarm.

# On additional worker nodes
docker swarm join --token <WORKER_TOKEN> <MANAGER-IP>:2377

# Verify from manager node
docker node ls
Task 3: Deploy Services in Swarm Mode
Subtask 3.1: Create Your First Swarm Service
Deploy a simple web service across the swarm cluster.

# Create a simple nginx service
docker service create \
  --name web-service \
  --publish 8080:80 \
  --replicas 2 \
  nginx:latest

# Alternative single-line command
docker service create --name web-service --publish 8080:80 --replicas 2 nginx:latest
Service Parameters Explained:

--name: Assigns a name to the service
--publish: Maps host port to container port
--replicas: Specifies number of service instances
Subtask 3.2: Verify Service Deployment
Check that the service has been created and is running properly.

# List all services
docker service ls

# Get detailed service information
docker service inspect web-service --pretty

# List service tasks (individual containers)
docker service ps web-service
Subtask 3.3: Test Service Accessibility
Verify that the service is accessible from any node in the swarm.

# Test from manager node
curl http://localhost:8080

# Test from worker node (if accessible)
curl http://<WORKER-NODE-IP>:8080

# Check which nodes are running the service
docker service ps web-service
Subtask 3.4: Deploy a More Complex Service
Create a service with additional configuration options.

# Deploy a service with resource constraints and update policy
docker service create \
  --name api-service \
  --publish 3000:3000 \
  --replicas 3 \
  --limit-cpu 0.5 \
  --limit-memory 512M \
  --update-delay 10s \
  --update-parallelism 1 \
  --restart-condition on-failure \
  --restart-max-attempts 3 \
  node:16-alpine sh -c "while true; do echo 'API Service Running'; sleep 30; done"
Task 4: Scale Services Using Docker Service Scale
Subtask 4.1: Scale Up Services
Increase the number of replicas for existing services.

# Scale the web service to 5 replicas
docker service scale web-service=5

# Verify scaling operation
docker service ls
docker service ps web-service

# Scale multiple services simultaneously
docker service scale web-service=4 api-service=6
Subtask 4.2: Monitor Scaling Process
Watch the scaling process in real-time.

# Watch service status updates
watch docker service ls

# Monitor specific service tasks
watch docker service ps web-service

# Check resource utilization across nodes
docker node ls
Subtask 4.3: Scale Down Services
Reduce the number of replicas to optimize resource usage.

# Scale down the web service
docker service scale web-service=2

# Scale down the api service
docker service scale api-service=2

# Verify the scaling down process
docker service ps web-service
docker service ps api-service
Subtask 4.4: Implement Auto-Scaling Considerations
While Docker Swarm doesn't have built-in auto-scaling, understand the concepts.

# Create a service with resource reservations
docker service create \
  --name resource-aware-service \
  --replicas 2 \
  --reserve-cpu 0.25 \
  --reserve-memory 256M \
  --limit-cpu 1.0 \
  --limit-memory 1G \
  nginx:latest

# Monitor resource usage
docker service inspect resource-aware-service --pretty
Task 5: Monitor and Manage Services and Nodes
Subtask 5.1: Monitor Swarm Health
Implement comprehensive monitoring of your swarm cluster.

# Get overall swarm information
docker system info

# Check node availability and status
docker node ls

# Inspect individual nodes
docker node inspect <NODE-ID> --pretty

# Check service health
docker service ls
Subtask 5.2: Monitor Service Logs
Access and monitor logs from services running across the swarm.

# View logs from a specific service
docker service logs web-service

# Follow logs in real-time
docker service logs -f web-service

# View logs with timestamps
docker service logs -t web-service

# Limit log output
docker service logs --tail 50 web-service
Subtask 5.3: Node Management Operations
Learn how to manage nodes within the swarm.

# Drain a node (move services away from it)
docker node update --availability drain <NODE-ID>

# Make a drained node active again
docker node update --availability active <NODE-ID>

# Pause a node (stop scheduling new tasks)
docker node update --availability pause <NODE-ID>

# Add labels to nodes for service placement
docker node update --label-add environment=production <NODE-ID>
docker node update --label-add datacenter=east <NODE-ID>
Subtask 5.4: Service Update and Rollback
Perform rolling updates and rollbacks on services.

# Update service image
docker service update --image nginx:1.21 web-service

# Monitor update progress
docker service ps web-service

# Rollback to previous version if needed
docker service rollback web-service

# Update service with specific parameters
docker service update \
  --replicas 3 \
  --update-delay 30s \
  --update-parallelism 1 \
  web-service
Subtask 5.5: Advanced Monitoring Commands
Implement advanced monitoring and troubleshooting techniques.

# Create a monitoring script
cat > monitor_swarm.sh << 'EOF'
#!/bin/bash
echo "=== Docker Swarm Status ==="
docker node ls
echo ""
echo "=== Service Status ==="
docker service ls
echo ""
echo "=== Service Tasks ==="
docker service ps $(docker service ls -q) --no-trunc
echo ""
echo "=== System Resource Usage ==="
docker system df
EOF

chmod +x monitor_swarm.sh
./monitor_swarm.sh
Subtask 5.6: Troubleshooting Common Issues
Learn to identify and resolve common swarm issues.

# Check for failed tasks
docker service ps web-service --filter "desired-state=running"

# Inspect failed tasks
docker service ps web-service --no-trunc

# Remove failed services
docker service rm <SERVICE-NAME>

# Force remove stuck services
docker service rm --force <SERVICE-NAME>

# Check network connectivity between nodes
docker network ls
docker network inspect ingress
Advanced Configuration and Best Practices
Service Placement Constraints
# Deploy service only on manager nodes
docker service create \
  --name manager-only-service \
  --constraint 'node.role==manager' \
  --replicas 1 \
  nginx:latest

# Deploy service based on node labels
docker service create \
  --name production-service \
  --constraint 'node.labels.environment==production' \
  --replicas 2 \
  nginx:latest
Network Configuration
# Create custom overlay network
docker network create --driver overlay custom-network

# Deploy service on custom network
docker service create \
  --name networked-service \
  --network custom-network \
  --replicas 2 \
  nginx:latest

# List networks
docker network ls
Secret Management
# Create a secret
echo "mysecretpassword" | docker secret create db_password -

# Use secret in service
docker service create \
  --name secure-service \
  --secret db_password \
  --replicas 1 \
  alpine:latest sleep 3600

# List secrets
docker secret ls
Troubleshooting Guide
Common Issues and Solutions
Issue 1: Node Cannot Join Swarm

# Check firewall settings - ensure ports 2377, 7946, 4789 are open
sudo ufw allow 2377
sudo ufw allow 7946
sudo ufw allow 4789

# Verify network connectivity
ping <MANAGER-IP>
telnet <MANAGER-IP> 2377
Issue 2: Service Not Starting

# Check service logs
docker service logs <SERVICE-NAME>

# Inspect service configuration
docker service inspect <SERVICE-NAME>

# Check node resources
docker node ls
docker system df
Issue 3: Port Already in Use

# Check what's using the port
sudo netstat -tulpn | grep :8080

# Use different port or stop conflicting service
docker service update --publish-rm 8080:80 --publish-add 8081:80 web-service
Lab Cleanup
Remove Services and Leave Swarm
# Remove all services
docker service rm $(docker service ls -q)

# On worker nodes, leave the swarm
docker swarm leave

# On manager node, force leave (this destroys the swarm)
docker swarm leave --force

# Verify cleanup
docker node ls
docker service ls
Conclusion
In this comprehensive lab, you have successfully:

Initialized a Docker Swarm cluster and understood the role of manager and worker nodes
Joined multiple nodes to create a distributed container orchestration platform
Deployed services across the swarm with proper load distribution
Scaled services up and down based on demand requirements
Monitored and managed both services and nodes effectively
Implemented advanced features like service constraints, custom networks, and secrets
Troubleshot common issues that arise in swarm environments
Why This Matters
Docker Swarm mode provides a native clustering and orchestration solution that enables you to:

High Availability: Services continue running even if individual nodes fail
Load Distribution: Automatically distribute containers across available nodes
Service Discovery: Built-in DNS-based service discovery for inter-service communication
Rolling Updates: Update services with zero downtime
Scalability: Easily scale services up or down based on demand
Next Steps
To further enhance your Docker Swarm skills:

Explore Docker Stack deployments using docker-compose.yml files
Implement monitoring solutions like Prometheus and Grafana
Study advanced networking configurations and service mesh concepts
Practice disaster recovery scenarios and backup strategies
Learn about integration with CI/CD pipelines for automated deployments
Real-World Applications
The skills learned in this lab are directly applicable to:

Production container deployments in enterprise environments
Microservices architecture implementation and management
DevOps practices for continuous deployment and scaling
Cloud-native application development and operations
Docker Certified Associate (DCA) certification preparation
This lab provides the foundation for managing containerized applications at scale using Docker's native orchestration capabilities.
