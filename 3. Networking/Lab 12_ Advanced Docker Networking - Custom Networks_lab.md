Lab 12: Advanced Docker Networking - Custom Networks
Objectives
By the end of this lab, you will be able to:

Create and manage custom Docker bridge networks
Launch containers within custom networks and verify inter-container connectivity
Use Docker network inspection commands to explore network configurations
Understand host networking concepts and their practical applications
Troubleshoot Docker networking issues using network disconnect commands
Apply advanced Docker networking concepts for real-world container deployments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with Linux command line operations
Knowledge of basic networking concepts (IP addresses, subnets, ports)
Completed previous Docker labs or equivalent experience
Understanding of container lifecycle management
Lab Environment Setup
Al Nafi Cloud Machines: This lab uses Al Nafi's pre-configured Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to access your ready-to-use environment. No need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine installed
Root access for network configuration
All necessary networking tools pre-installed
Task 1: Create Custom Bridge Networks
Subtask 1.1: Understanding Docker Network Types
Before creating custom networks, let's explore the default Docker networks:

# List all existing Docker networks
docker network ls
You should see output similar to:

NETWORK ID     NAME      DRIVER    SCOPE
xxxxxxxxxxxx   bridge    bridge    local
xxxxxxxxxxxx   host      host      local
xxxxxxxxxxxx   none      null      local
Subtask 1.2: Create Your First Custom Bridge Network
Create a custom bridge network for web applications:

# Create a custom bridge network named 'webapp-network'
docker network create --driver bridge webapp-network
Verify the network creation:

# List networks to confirm creation
docker network ls
Subtask 1.3: Create a Network with Custom Configuration
Create a more advanced custom network with specific subnet and gateway:

# Create a custom network with specific IP range
docker network create \
  --driver bridge \
  --subnet=172.20.0.0/16 \
  --ip-range=172.20.240.0/20 \
  --gateway=172.20.0.1 \
  backend-network
Subtask 1.4: Create Multiple Networks for Different Purposes
Create additional networks to simulate a multi-tier application:

# Create frontend network
docker network create frontend-network

# Create database network
docker network create database-network
Verify all networks are created:

docker network ls
Task 2: Launch Containers Within Custom Networks
Subtask 2.1: Deploy Containers in Custom Networks
Launch containers in your custom networks:

# Run a web server container in the webapp-network
docker run -d --name web-server --network webapp-network nginx:alpine

# Run another container in the same network
docker run -d --name web-client --network webapp-network alpine:latest sleep 3600
Subtask 2.2: Deploy Multi-Container Application
Create a more complex setup with multiple containers:

# Run a database container in the backend network
docker run -d --name mysql-db \
  --network backend-network \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=webapp \
  mysql:8.0

# Run an application server connected to both networks
docker run -d --name app-server \
  --network backend-network \
  nginx:alpine

# Connect the app-server to frontend network as well
docker network connect frontend-network app-server
Subtask 2.3: Verify Container Network Assignments
Check which containers are running and their network assignments:

# List running containers
docker ps

# Check container network details
docker inspect web-server | grep -A 10 "Networks"
Task 3: Verify Connectivity Between Containers
Subtask 3.1: Test Basic Container-to-Container Communication
Test connectivity between containers in the same network:

# Access the web-client container
docker exec -it web-client sh

# Inside the container, test connectivity to web-server
ping web-server

# Test HTTP connectivity
wget -qO- http://web-server

# Exit the container
exit
Subtask 3.2: Test Cross-Network Communication
Verify that containers in different networks cannot communicate by default:

# Run a container in the frontend network
docker run -d --name frontend-app --network frontend-network alpine:latest sleep 3600

# Try to ping from frontend to backend (should fail)
docker exec frontend-app ping mysql-db
Subtask 3.3: Test Multi-Network Container Communication
Test the app-server that's connected to multiple networks:

# From app-server, test connectivity to backend
docker exec app-server ping mysql-db

# Run a test container in frontend network
docker run --rm --network frontend-network alpine:latest ping -c 3 app-server
Task 4: Use Docker Network Inspect
Subtask 4.1: Inspect Custom Network Configuration
Examine the detailed configuration of your custom networks:

# Inspect the webapp-network
docker network inspect webapp-network
Subtask 4.2: Analyze Network Settings in Detail
Look for specific information in the network inspection:

# Get specific network information using format filters
docker network inspect webapp-network --format='{{.IPAM.Config}}'

# Check which containers are connected
docker network inspect webapp-network --format='{{range .Containers}}{{.Name}} {{.IPv4Address}}{{end}}'
Subtask 4.3: Compare Different Network Types
Inspect and compare different network configurations:

# Inspect the default bridge network
docker network inspect bridge

# Inspect your custom backend network
docker network inspect backend-network

# Compare the subnet configurations
echo "Default bridge network:"
docker network inspect bridge --format='{{.IPAM.Config}}'
echo "Custom backend network:"
docker network inspect backend-network --format='{{.IPAM.Config}}'
Task 5: Learn About Host Networking
Subtask 5.1: Understanding Host Network Mode
Create a container using host networking:

# Run a container with host networking
docker run -d --name host-nginx --network host nginx:alpine
Subtask 5.2: Compare Host vs Bridge Networking
Test the differences between host and bridge networking:

# Check the host-nginx container network settings
docker inspect host-nginx | grep -A 5 "NetworkMode"

# Run a simple web server on host network
docker run --rm --network host -p 8080:80 nginx:alpine &

# Test accessibility (should be accessible on host IP)
curl localhost:8080

# Stop the background container
docker stop $(docker ps -q --filter ancestor=nginx:alpine)
Subtask 5.3: Host Network Use Cases
Create a monitoring container that needs host network access:

# Run a container that monitors host network interfaces
docker run --rm --network host alpine:latest ip addr show

# Compare with bridge network container
docker run --rm --network bridge alpine:latest ip addr show
Task 6: Troubleshoot Networking Issues
Subtask 6.1: Identify Network Connectivity Problems
Create a scenario with network issues:

# Create an isolated container
docker run -d --name isolated-app --network none alpine:latest sleep 3600

# Try to access it (should fail)
docker exec isolated-app ping 8.8.8.8
Subtask 6.2: Use Docker Network Disconnect
Practice disconnecting containers from networks:

# Disconnect app-server from frontend network
docker network disconnect frontend-network app-server

# Verify disconnection
docker network inspect frontend-network --format='{{range .Containers}}{{.Name}}{{end}}'

# Test connectivity (should fail now)
docker run --rm --network frontend-network alpine:latest ping -c 2 app-server
Subtask 6.3: Reconnect and Verify
Reconnect the container and verify connectivity:

# Reconnect app-server to frontend network
docker network connect frontend-network app-server

# Verify reconnection
docker network inspect frontend-network --format='{{range .Containers}}{{.Name}} {{.IPv4Address}}{{end}}'

# Test connectivity (should work now)
docker run --rm --network frontend-network alpine:latest ping -c 2 app-server
Subtask 6.4: Advanced Troubleshooting
Use advanced troubleshooting techniques:

# Check container network namespace
docker exec web-server ip route

# Check network interface details
docker exec web-server ip addr show

# Test DNS resolution
docker exec web-server nslookup web-client

# Check iptables rules (on host)
sudo iptables -t nat -L DOCKER
Task 7: Clean Up and Network Management
Subtask 7.1: Remove Containers
Clean up all containers created during the lab:

# Stop all running containers
docker stop $(docker ps -q)

# Remove all containers
docker rm $(docker ps -aq)
Subtask 7.2: Remove Custom Networks
Remove the custom networks (only works when no containers are using them):

# Remove custom networks
docker network rm webapp-network
docker network rm backend-network
docker network rm frontend-network
docker network rm database-network

# Verify removal
docker network ls
Subtask 7.3: Network Pruning
Use Docker's built-in cleanup commands:

# Remove all unused networks
docker network prune -f

# Verify only default networks remain
docker network ls
Practical Examples and Use Cases
Example 1: Microservices Architecture
Here's how you might set up networking for a microservices application:

# Create networks for different tiers
docker network create --driver bridge microservices-frontend
docker network create --driver bridge microservices-backend
docker network create --driver bridge microservices-database

# Deploy database
docker run -d --name postgres-db \
  --network microservices-database \
  -e POSTGRES_PASSWORD=dbpass \
  postgres:13

# Deploy API service (connected to backend and database)
docker run -d --name api-service \
  --network microservices-backend \
  nginx:alpine

docker network connect microservices-database api-service

# Deploy frontend (connected to frontend and backend)
docker run -d --name frontend-app \
  --network microservices-frontend \
  nginx:alpine

docker network connect microservices-backend frontend-app
Example 2: Development Environment Isolation
Create isolated development environments:

# Create development environment networks
docker network create dev-env-1
docker network create dev-env-2

# Deploy isolated development stacks
docker run -d --name dev1-web --network dev-env-1 nginx:alpine
docker run -d --name dev1-db --network dev-env-1 mysql:8.0 -e MYSQL_ROOT_PASSWORD=dev1pass

docker run -d --name dev2-web --network dev-env-2 nginx:alpine
docker run -d --name dev2-db --network dev-env-2 mysql:8.0 -e MYSQL_ROOT_PASSWORD=dev2pass
Key Concepts Summary
Custom Bridge Networks
Provide better isolation than the default bridge network
Enable automatic DNS resolution between containers
Allow custom IP addressing schemes
Network Inspection
docker network inspect provides detailed network configuration
Shows connected containers and their IP addresses
Reveals IPAM (IP Address Management) settings
Host Networking
Containers share the host's network stack
Better performance but less isolation
Useful for network monitoring and high-performance applications
Network Troubleshooting
docker network disconnect/connect for dynamic network management
Container network namespaces provide isolation
DNS resolution works automatically within custom networks
Common Issues and Solutions
Issue 1: Container Cannot Resolve Other Container Names
Solution: Ensure both containers are on the same custom network (not default bridge)

Issue 2: Port Conflicts with Host Networking
Solution: Check for port conflicts on the host system before using host networking

Issue 3: Cannot Remove Network
Solution: Ensure no containers are connected to the network before removal

Conclusion
In this lab, you have successfully mastered advanced Docker networking concepts including:

Custom Network Creation: You learned to create bridge networks with custom configurations, enabling better container isolation and communication control.

Inter-Container Communication: You practiced deploying containers within custom networks and verified connectivity between them using both ping and application-level communication.

Network Inspection: You gained expertise in using docker network inspect to analyze network configurations, IP assignments, and container connections.

Host Networking: You explored host networking mode and understood its use cases for high-performance applications and system monitoring.

Network Troubleshooting: You developed skills in diagnosing and resolving network connectivity issues using disconnect and reconnect operations.

These advanced networking skills are essential for:

Production Deployments: Properly isolating different application tiers
Microservices Architecture: Enabling secure communication between services
Development Environments: Creating isolated testing environments
Security: Implementing network-level access controls
The knowledge gained in this lab directly applies to the Docker Certified Associate (DCA) certification and real-world container orchestration scenarios. You now have the foundation to design and implement sophisticated Docker networking solutions for complex applications.
