Lab 48: Advanced Docker Networking - Custom Network Types
Objectives
By the end of this lab, you will be able to:

Create and manage custom overlay networks in Docker Swarm
Deploy services to custom networks and understand network isolation
Experiment with different Docker network modes (host, none, bridge)
Configure inter-container communication across multiple Docker networks
Implement network aliases for efficient multi-container communication
Troubleshoot common Docker networking issues
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with Docker CLI commands
Knowledge of basic networking concepts (IP addresses, ports, DNS)
Understanding of Docker Compose fundamentals
Basic Linux command line skills
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to access your environment. No need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine installed
Docker Compose pre-configured
All necessary networking tools available
Lab Tasks
Task 1: Create a Custom Overlay Network in Docker Swarm
Subtask 1.1: Initialize Docker Swarm
First, we need to initialize Docker Swarm mode to enable overlay networking capabilities.

# Initialize Docker Swarm
docker swarm init

# Verify swarm status
docker node ls
Expected Output:

ID                            HOSTNAME   STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
abc123def456                  lab-vm     Ready     Active         Leader           20.10.x
Subtask 1.2: Create Custom Overlay Network
Create a custom overlay network that will span across swarm nodes.

# Create a custom overlay network
docker network create --driver overlay --attachable custom-overlay-net

# List all networks to verify creation
docker network ls

# Inspect the custom network details
docker network inspect custom-overlay-net
Key Concepts:

Overlay Network: Enables communication between containers across multiple Docker hosts
Attachable Flag: Allows standalone containers to connect to the overlay network
Subtask 1.3: Create Additional Custom Networks
Create multiple networks to demonstrate network isolation and communication.

# Create a bridge network for local communication
docker network create --driver bridge frontend-net

# Create another overlay network for backend services
docker network create --driver overlay --attachable backend-net

# Verify all custom networks
docker network ls | grep -E "(custom-overlay-net|frontend-net|backend-net)"
Task 2: Deploy Services to the Custom Network
Subtask 2.1: Create a Multi-Service Application
Create a simple web application stack using custom networks.

# Create a directory for our application
mkdir ~/docker-networking-lab
cd ~/docker-networking-lab

# Create a simple HTML file for our web server
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Docker Networking Lab</title>
</head>
<body>
    <h1>Welcome to Advanced Docker Networking</h1>
    <p>This page is served from a container on a custom network!</p>
    <p>Container ID: <span id="container-id"></span></p>
    <script>
        document.getElementById('container-id').textContent = window.location.hostname;
    </script>
</body>
</html>
EOF
Subtask 2.2: Deploy Web Service to Custom Overlay Network
# Create a web service using nginx on the custom overlay network
docker service create \
  --name web-service \
  --network custom-overlay-net \
  --publish 8080:80 \
  --mount type=bind,source=$(pwd)/index.html,target=/usr/share/nginx/html/index.html \
  nginx:alpine

# Verify service deployment
docker service ls

# Check service details
docker service inspect web-service
Subtask 2.3: Deploy Database Service to Backend Network
# Create a Redis database service on the backend network
docker service create \
  --name redis-service \
  --network backend-net \
  redis:alpine

# Create a simple API service that connects to Redis
docker service create \
  --name api-service \
  --network backend-net \
  --network custom-overlay-net \
  --publish 3000:3000 \
  --env REDIS_HOST=redis-service \
  node:alpine sh -c "
    npm init -y && 
    npm install express redis && 
    node -e \"
      const express = require('express');
      const redis = require('redis');
      const app = express();
      const client = redis.createClient({host: 'redis-service'});
      
      app.get('/', (req, res) => {
        client.incr('visits', (err, visits) => {
          res.json({message: 'API Service Running', visits: visits || 0});
        });
      });
      
      app.listen(3000, () => console.log('API running on port 3000'));
    \"
  "
Subtask 2.4: Verify Service Communication
# Test web service
curl http://localhost:8080

# Test API service
curl http://localhost:3000

# Check service logs
docker service logs web-service
docker service logs api-service
Task 3: Experiment with Host and None Network Modes
Subtask 3.1: Understanding Network Modes
Docker supports several network modes:

bridge: Default network mode for containers
host: Container shares the host's network stack
none: Container has no network access
overlay: Multi-host networking for swarm services
Subtask 3.2: Deploy Container with Host Network Mode
# Run a container with host networking
docker run -d \
  --name host-network-container \
  --network host \
  nginx:alpine

# Verify the container is using host networking
docker inspect host-network-container | grep -A 10 "NetworkMode"

# Check if nginx is accessible directly on host port 80
# Note: This might conflict with existing services
curl http://localhost:80 || echo "Port 80 might be in use"

# Clean up
docker stop host-network-container
docker rm host-network-container
Subtask 3.3: Deploy Container with None Network Mode
# Run a container with no network access
docker run -d \
  --name no-network-container \
  --network none \
  alpine:latest sleep 3600

# Verify no network interfaces (except loopback)
docker exec no-network-container ip addr show

# Try to ping (should fail)
docker exec no-network-container ping -c 3 8.8.8.8 || echo "No network access as expected"

# Clean up
docker stop no-network-container
docker rm no-network-container
Subtask 3.4: Compare Network Modes
Create a comparison script to demonstrate differences:

# Create a network comparison script
cat > network-comparison.sh << 'EOF'
#!/bin/bash

echo "=== Network Mode Comparison ==="

# Bridge mode (default)
echo "1. Bridge Mode:"
docker run --rm --network bridge alpine ip route | head -3

# Host mode
echo "2. Host Mode:"
docker run --rm --network host alpine ip route | head -3

# None mode
echo "3. None Mode:"
docker run --rm --network none alpine ip addr show | grep -E "(lo|eth)"

echo "Comparison complete!"
EOF

chmod +x network-comparison.sh
./network-comparison.sh
Task 4: Set Up Inter-Container Communication Across Docker Networks
Subtask 4.1: Create Multi-Network Architecture
# Create three separate networks for different tiers
docker network create --driver bridge web-tier
docker network create --driver bridge app-tier  
docker network create --driver bridge db-tier

# Verify networks
docker network ls | grep -E "(web-tier|app-tier|db-tier)"
Subtask 4.2: Deploy Multi-Tier Application
# Deploy database container (only on db-tier)
docker run -d \
  --name database \
  --network db-tier \
  -e POSTGRES_DB=testdb \
  -e POSTGRES_USER=testuser \
  -e POSTGRES_PASSWORD=testpass \
  postgres:13-alpine

# Deploy application container (connected to both app-tier and db-tier)
docker run -d \
  --name application \
  --network app-tier \
  -e DATABASE_URL=postgresql://testuser:testpass@database:5432/testdb \
  node:alpine sh -c "
    npm init -y &&
    npm install express pg &&
    node -e \"
      const express = require('express');
      const { Client } = require('pg');
      const app = express();
      
      app.get('/health', (req, res) => {
        res.json({status: 'Application healthy', network: 'app-tier'});
      });
      
      app.get('/db-test', async (req, res) => {
        try {
          const client = new Client(process.env.DATABASE_URL);
          await client.connect();
          const result = await client.query('SELECT NOW()');
          await client.end();
          res.json({database: 'connected', time: result.rows[0].now});
        } catch (err) {
          res.status(500).json({error: 'Database connection failed'});
        }
      });
      
      app.listen(3000, () => console.log('App running on port 3000'));
    \"
  "

# Connect application to db-tier network
docker network connect db-tier application

# Deploy web server (connected to both web-tier and app-tier)
docker run -d \
  --name webserver \
  --network web-tier \
  -p 8081:80 \
  nginx:alpine

# Connect webserver to app-tier
docker network connect app-tier webserver
Subtask 4.3: Configure Nginx Proxy
# Create nginx configuration for proxying to application
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream app_backend {
        server application:3000;
    }
    
    server {
        listen 80;
        
        location / {
            proxy_pass http://app_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            proxy_pass http://app_backend/health;
        }
        
        location /db-test {
            proxy_pass http://app_backend/db-test;
        }
    }
}
EOF

# Copy configuration to webserver container
docker cp nginx.conf webserver:/etc/nginx/nginx.conf

# Reload nginx configuration
docker exec webserver nginx -s reload
Subtask 4.4: Test Inter-Network Communication
# Test application health
curl http://localhost:8081/health

# Test database connectivity through the application
curl http://localhost:8081/db-test

# Verify network connectivity between containers
docker exec application ping -c 3 database
docker exec webserver ping -c 3 application

# Show network topology
echo "=== Network Topology ==="
docker network inspect web-tier --format '{{.Name}}: {{range .Containers}}{{.Name}} {{end}}'
docker network inspect app-tier --format '{{.Name}}: {{range .Containers}}{{.Name}} {{end}}'
docker network inspect db-tier --format '{{.Name}}: {{range .Containers}}{{.Name}} {{end}}'
Task 5: Use Network Aliases to Manage Multi-Container Communication
Subtask 5.1: Understanding Network Aliases
Network aliases allow containers to be reached by multiple names within a network, providing flexibility and load balancing capabilities.

Subtask 5.2: Create Services with Network Aliases
# Create a new network for alias demonstration
docker network create --driver bridge alias-demo-net

# Deploy multiple instances of the same service with different aliases
docker run -d \
  --name web1 \
  --network alias-demo-net \
  --network-alias webserver \
  --network-alias frontend \
  nginx:alpine

docker run -d \
  --name web2 \
  --network alias-demo-net \
  --network-alias webserver \
  --network-alias frontend \
  nginx:alpine

docker run -d \
  --name web3 \
  --network alias-demo-net \
  --network-alias webserver \
  --network-alias frontend \
  nginx:alpine
Subtask 5.3: Create Load Balancer Configuration
# Create a simple load balancer using nginx
cat > lb-nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream web_servers {
        server webserver:80;
    }
    
    server {
        listen 80;
        
        location / {
            proxy_pass http://web_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
EOF

# Deploy load balancer
docker run -d \
  --name load-balancer \
  --network alias-demo-net \
  -p 8082:80 \
  -v $(pwd)/lb-nginx.conf:/etc/nginx/nginx.conf:ro \
  nginx:alpine
Subtask 5.4: Test Network Alias Resolution
# Create a test container to verify DNS resolution
docker run -d \
  --name dns-tester \
  --network alias-demo-net \
  alpine:latest sleep 3600

# Test DNS resolution for aliases
echo "=== Testing DNS Resolution ==="
docker exec dns-tester nslookup webserver
docker exec dns-tester nslookup frontend

# Test connectivity to aliased services
echo "=== Testing Connectivity ==="
for i in {1..6}; do
  echo "Request $i:"
  docker exec dns-tester wget -qO- http://webserver | grep -o "Welcome to nginx" || echo "Connection failed"
  sleep 1
done
Subtask 5.5: Advanced Alias Configuration with Docker Compose
Create a comprehensive example using Docker Compose:

# Create docker-compose.yml with network aliases
cat > docker-compose-aliases.yml << 'EOF'
version: '3.8'

services:
  web1:
    image: nginx:alpine
    networks:
      app-network:
        aliases:
          - webserver
          - frontend
          - www
    volumes:
      - ./web1.html:/usr/share/nginx/html/index.html

  web2:
    image: nginx:alpine
    networks:
      app-network:
        aliases:
          - webserver
          - frontend
          - www
    volumes:
      - ./web2.html:/usr/share/nginx/html/index.html

  web3:
    image: nginx:alpine
    networks:
      app-network:
        aliases:
          - webserver
          - frontend
          - www
    volumes:
      - ./web3.html:/usr/share/nginx/html/index.html

  api:
    image: node:alpine
    networks:
      app-network:
        aliases:
          - api-server
          - backend
    command: >
      sh -c "
        npm init -y &&
        npm install express &&
        node -e \"
          const express = require('express');
          const app = express();
          app.get('/', (req, res) => {
            res.json({
              message: 'API Server',
              hostname: require('os').hostname(),
              aliases: ['api-server', 'backend']
            });
          });
          app.listen(3000, () => console.log('API running'));
        \"
      "

  client:
    image: alpine:latest
    networks:
      - app-network
    command: sleep 3600
    depends_on:
      - web1
      - web2
      - web3
      - api

networks:
  app-network:
    driver: bridge
EOF

# Create different HTML files for each web server
echo "<h1>Web Server 1</h1><p>Served by container: web1</p>" > web1.html
echo "<h1>Web Server 2</h1><p>Served by container: web2</p>" > web2.html
echo "<h1>Web Server 3</h1><p>Served by container: web3</p>" > web3.html

# Deploy the stack
docker-compose -f docker-compose-aliases.yml up -d

# Test alias resolution
echo "=== Testing Compose Network Aliases ==="
docker-compose -f docker-compose-aliases.yml exec client nslookup webserver
docker-compose -f docker-compose-aliases.yml exec client nslookup api-server

# Test load balancing through aliases
echo "=== Testing Load Balancing ==="
for i in {1..6}; do
  echo "Request $i to webserver alias:"
  docker-compose -f docker-compose-aliases.yml exec client wget -qO- http://webserver | grep -o "Web Server [1-3]"
  sleep 1
done
Troubleshooting Common Issues
Issue 1: Overlay Network Not Working
Problem: Services cannot communicate across overlay network

Solution:

# Check if swarm is initialized
docker info | grep Swarm

# Verify network exists and is attachable
docker network inspect custom-overlay-net | grep Attachable

# Check firewall rules (if applicable)
sudo ufw status
Issue 2: DNS Resolution Failing
Problem: Containers cannot resolve service names

Solution:

# Check container DNS configuration
docker exec container-name cat /etc/resolv.conf

# Verify network connectivity
docker exec container-name ping -c 3 target-container

# Check if containers are on the same network
docker network inspect network-name
Issue 3: Port Conflicts
Problem: Cannot bind to host ports

Solution:

# Check what's using the port
sudo netstat -tulpn | grep :8080

# Use different port or stop conflicting service
docker stop conflicting-container
Lab Cleanup
Clean up all resources created during this lab:

# Stop and remove all containers
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)

# Remove all custom networks
docker network rm custom-overlay-net frontend-net backend-net
docker network rm web-tier app-tier db-tier alias-demo-net

# Remove Docker Compose stack
docker-compose -f docker-compose-aliases.yml down

# Leave swarm mode
docker swarm leave --force

# Remove lab files
cd ~
rm -rf docker-networking-lab
rm -f nginx.conf lb-nginx.conf network-comparison.sh
rm -f docker-compose-aliases.yml web*.html
Conclusion
In this comprehensive lab, you have successfully:

Created custom overlay networks in Docker Swarm, enabling multi-host container communication
Deployed services to custom networks, demonstrating network isolation and service discovery
Experimented with different network modes (host, none, bridge), understanding their use cases and limitations
Configured inter-container communication across multiple networks, creating a realistic multi-tier application architecture
Implemented network aliases for flexible service naming and basic load balancing
Key Takeaways
Network Isolation: Custom networks provide security and organization by isolating different application components
Service Discovery: Docker's built-in DNS enables containers to communicate using service names instead of IP addresses
Flexibility: Network aliases allow multiple names for the same service, enabling load balancing and service abstraction
Scalability: Overlay networks enable applications to span multiple Docker hosts in a swarm cluster
Real-World Applications
These networking concepts are essential for:

Microservices Architecture: Isolating and connecting different service components
Development Environments: Creating realistic network topologies for testing
Production Deployments: Implementing secure, scalable container networking
DevOps Practices: Enabling consistent networking across development, staging, and production environments
Understanding Docker networking is crucial for the Docker Certified Associate (DCA) certification and for building robust, production-ready containerized applications. The skills learned in this lab form the foundation for advanced container orchestration and cloud-native application development.
