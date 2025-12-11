Lab 88: Docker in Production - Running Containers on DigitalOcean
Lab Objectives
By the end of this lab, students will be able to:

Create and configure a DigitalOcean droplet with Docker installed
Deploy containerized web applications in a production environment
Configure load balancing for Docker containers using DigitalOcean's infrastructure
Implement monitoring solutions for containerized applications
Scale Docker applications and manage them using Docker commands
Apply production-ready practices for container deployment
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (containers, images, Dockerfiles)
Familiarity with Linux command line operations
Basic knowledge of web applications and HTTP protocols
Understanding of networking concepts (ports, load balancing)
A DigitalOcean account (free tier available for learning)
Note: Al Nafi provides ready-to-use Linux-based cloud machines. Simply click Start Lab to begin - no need to build your own VM.

Lab Environment Setup
This lab uses Al Nafi's cloud infrastructure which provides:

Pre-configured Linux environment
Docker installation capabilities
Network access to DigitalOcean services
All necessary development tools
Task 1: Create a DigitalOcean Droplet and Install Docker
Subtask 1.1: Create a DigitalOcean Droplet
Step 1: Log into your DigitalOcean account

Navigate to the DigitalOcean control panel
Click on Create and select Droplets
Step 2: Configure the droplet settings

Image: Select Ubuntu 22.04 LTS
Plan: Choose Basic plan with 1GB RAM, 1 vCPU
Region: Select a region closest to your location
Authentication: Add your SSH key or create a password
Hostname: Name your droplet docker-production-lab
Step 3: Create the droplet

Click Create Droplet
Wait for the droplet to be provisioned (usually 1-2 minutes)
Note the public IP address assigned to your droplet
Subtask 1.2: Connect to Your Droplet
Step 1: Connect via SSH from your Al Nafi lab environment

# Replace YOUR_DROPLET_IP with your actual droplet IP
ssh root@YOUR_DROPLET_IP
Step 2: Update the system packages

# Update package index
sudo apt update

# Upgrade installed packages
sudo apt upgrade -y
Subtask 1.3: Install Docker
Step 1: Install required packages

# Install packages to allow apt to use a repository over HTTPS
sudo apt install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
Step 2: Add Docker's official GPG key

# Create directory for keyrings
sudo mkdir -p /etc/apt/keyrings

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
Step 3: Set up the Docker repository

# Add Docker repository
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
Step 4: Install Docker Engine

# Update package index
sudo apt update

# Install Docker Engine, containerd, and Docker Compose
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
Step 5: Verify Docker installation

# Check Docker version
sudo docker --version

# Run hello-world container to verify installation
sudo docker run hello-world
Step 6: Configure Docker for non-root user (optional but recommended)

# Add current user to docker group
sudo usermod -aG docker $USER

# Apply group changes (or logout and login again)
newgrp docker

# Test Docker without sudo
docker run hello-world
Task 2: Deploy a Docker Container for a Basic Web Application
Subtask 2.1: Create a Simple Web Application
Step 1: Create a project directory

# Create and navigate to project directory
mkdir ~/docker-web-app
cd ~/docker-web-app
Step 2: Create a simple HTML file

# Create index.html
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Docker Production Lab</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            text-align: center;
            padding: 50px;
        }
        .container {
            max-width: 800px;
            margin: 0 auto;
            background: rgba(255,255,255,0.1);
            padding: 40px;
            border-radius: 10px;
        }
        h1 {
            font-size: 3em;
            margin-bottom: 20px;
        }
        .info {
            font-size: 1.2em;
            margin: 20px 0;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>🐳 Docker Production Lab</h1>
        <div class="info">
            <p>Congratulations! Your containerized web application is running in production.</p>
            <p>Server: <span id="hostname">Loading...</span></p>
            <p>Timestamp: <span id="timestamp"></span></p>
        </div>
    </div>
    
    <script>
        // Display current timestamp
        document.getElementById('timestamp').textContent = new Date().toLocaleString();
        
        // Try to get hostname (will work when we add backend)
        fetch('/api/hostname')
            .then(response => response.text())
            .then(data => document.getElementById('hostname').textContent = data)
            .catch(() => document.getElementById('hostname').textContent = 'Web Server');
    </script>
</body>
</html>
EOF
Step 3: Create a simple Node.js backend

# Create package.json
cat > package.json << 'EOF'
{
  "name": "docker-production-app",
  "version": "1.0.0",
  "description": "A simple web app for Docker production lab",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF
Step 4: Create the server application

# Create server.js
cat > server.js << 'EOF'
const express = require('express');
const path = require('path');
const os = require('os');

const app = express();
const PORT = process.env.PORT || 3000;

// Serve static files
app.use(express.static('.'));

// API endpoint to get hostname
app.get('/api/hostname', (req, res) => {
    res.send(os.hostname());
});

// Health check endpoint
app.get('/health', (req, res) => {
    res.json({
        status: 'healthy',
        timestamp: new Date().toISOString(),
        hostname: os.hostname()
    });
});

// Serve main page
app.get('/', (req, res) => {
    res.sendFile(path.join(__dirname, 'index.html'));
});

app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on port ${PORT}`);
    console.log(`Hostname: ${os.hostname()}`);
});
EOF
Subtask 2.2: Create a Dockerfile
Step 1: Create the Dockerfile

# Create Dockerfile
cat > Dockerfile << 'EOF'
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory in container
WORKDIR /app

# Copy package.json and package-lock.json (if available)
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy application code
COPY . .

# Expose port 3000
EXPOSE 3000

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Change ownership of app directory
RUN chown -R nodejs:nodejs /app
USER nodejs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3000/health || exit 1

# Start the application
CMD ["npm", "start"]
EOF
Subtask 2.3: Build and Deploy the Container
Step 1: Build the Docker image

# Build the image with a tag
docker build -t docker-web-app:v1.0 .

# Verify the image was created
docker images
Step 2: Run the container locally for testing

# Run container on port 3000
docker run -d --name web-app-test -p 3000:3000 docker-web-app:v1.0

# Check if container is running
docker ps

# Test the application
curl http://localhost:3000
Step 3: Test the health endpoint

# Test health check
curl http://localhost:3000/health

# Check container logs
docker logs web-app-test
Step 4: Stop the test container

# Stop and remove test container
docker stop web-app-test
docker rm web-app-test
Subtask 2.4: Deploy for Production
Step 1: Run the container in production mode

# Run container with restart policy and proper naming
docker run -d \
    --name production-web-app \
    --restart unless-stopped \
    -p 80:3000 \
    docker-web-app:v1.0

# Verify deployment
docker ps
Step 2: Test the production deployment

# Test from the droplet
curl http://localhost

# Test from external (replace YOUR_DROPLET_IP)
curl http://YOUR_DROPLET_IP
Task 3: Configure DigitalOcean's Load Balancer for Containerized Apps
Subtask 3.1: Prepare Multiple Container Instances
Step 1: Create multiple instances of the application

# Stop the single instance
docker stop production-web-app
docker rm production-web-app

# Run multiple instances on different ports
docker run -d --name web-app-1 --restart unless-stopped -p 3001:3000 docker-web-app:v1.0
docker run -d --name web-app-2 --restart unless-stopped -p 3002:3000 docker-web-app:v1.0
docker run -d --name web-app-3 --restart unless-stopped -p 3003:3000 docker-web-app:v1.0

# Verify all instances are running
docker ps
Step 2: Test individual instances

# Test each instance
curl http://localhost:3001/api/hostname
curl http://localhost:3002/api/hostname
curl http://localhost:3003/api/hostname
Subtask 3.2: Install and Configure Nginx as Load Balancer
Step 1: Install Nginx

# Install Nginx
sudo apt update
sudo apt install -y nginx

# Start and enable Nginx
sudo systemctl start nginx
sudo systemctl enable nginx
Step 2: Configure Nginx for load balancing

# Create Nginx configuration for load balancing
sudo tee /etc/nginx/sites-available/docker-lb << 'EOF'
upstream docker_backend {
    server localhost:3001;
    server localhost:3002;
    server localhost:3003;
}

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://docker_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Health check
        proxy_connect_timeout 5s;
        proxy_send_timeout 5s;
        proxy_read_timeout 5s;
    }

    location /health {
        proxy_pass http://docker_backend/health;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
EOF
Step 3: Enable the configuration

# Remove default site
sudo rm -f /etc/nginx/sites-enabled/default

# Enable our load balancer configuration
sudo ln -s /etc/nginx/sites-available/docker-lb /etc/nginx/sites-enabled/

# Test Nginx configuration
sudo nginx -t

# Reload Nginx
sudo systemctl reload nginx
Step 4: Test load balancing

# Test multiple requests to see load balancing in action
for i in {1..6}; do
    echo "Request $i:"
    curl -s http://localhost/api/hostname
    echo ""
    sleep 1
done
Subtask 3.3: Configure DigitalOcean Load Balancer (Alternative Method)
Step 1: Create additional droplets (if using DO Load Balancer)

# Note: This step would be done through DigitalOcean control panel
# For this lab, we'll document the process:

# 1. Go to DigitalOcean control panel
# 2. Navigate to Networking > Load Balancers
# 3. Click "Create Load Balancer"
# 4. Configure:
#    - Name: docker-app-lb
#    - Region: Same as your droplets
#    - VPC: Default
#    - Type: External (Public)
# 5. Add forwarding rules:
#    - HTTP port 80 -> HTTP port 80
# 6. Add health checks:
#    - Protocol: HTTP
#    - Port: 80
#    - Path: /health
# 7. Add droplets to the load balancer
# 8. Create load balancer
Task 4: Set Up Monitoring for Docker Containers on DigitalOcean
Subtask 4.1: Install Docker Monitoring Tools
Step 1: Create a monitoring directory

# Create monitoring directory
mkdir ~/docker-monitoring
cd ~/docker-monitoring
Step 2: Create a Docker Compose file for monitoring stack

# Create docker-compose.yml for monitoring
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    privileged: true
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
EOF
Step 3: Create Prometheus configuration

# Create prometheus.yml
cat > prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'docker-web-app'
    static_configs:
      - targets: ['host.docker.internal:3001', 'host.docker.internal:3002', 'host.docker.internal:3003']
    metrics_path: '/health'
EOF
Subtask 4.2: Deploy Monitoring Stack
Step 1: Start the monitoring services

# Start all monitoring services
docker compose up -d

# Check if all services are running
docker compose ps
Step 2: Verify monitoring services

# Check Prometheus
curl http://localhost:9090

# Check Grafana
curl http://localhost:3000

# Check Node Exporter
curl http://localhost:9100/metrics

# Check cAdvisor
curl http://localhost:8080/metrics
Subtask 4.3: Configure Grafana Dashboard
Step 1: Access Grafana web interface

# Grafana is available at http://YOUR_DROPLET_IP:3000
# Default credentials: admin / admin123
echo "Access Grafana at: http://$(curl -s ifconfig.me):3000"
echo "Username: admin"
echo "Password: admin123"
Step 2: Add Prometheus as data source (via web interface)

Navigate to Configuration > Data Sources
Click "Add data source"
Select Prometheus
Set URL to: http://prometheus:9090
Click "Save & Test"
Step 3: Import Docker monitoring dashboard

Go to Dashboards > Import
Use dashboard ID: 193 (Docker monitoring dashboard)
Select Prometheus as data source
Click Import
Subtask 4.4: Create Custom Monitoring Script
Step 1: Create a monitoring script

# Create monitoring script
cat > monitor-containers.sh << 'EOF'
#!/bin/bash

# Docker Container Monitoring Script
echo "=== Docker Container Monitoring ==="
echo "Timestamp: $(date)"
echo ""

echo "=== Running Containers ==="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
echo ""

echo "=== Container Resource Usage ==="
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
echo ""

echo "=== Container Health Status ==="
for container in $(docker ps --format "{{.Names}}"); do
    health=$(docker inspect --format='{{.State.Health.Status}}' $container 2>/dev/null || echo "no-healthcheck")
    echo "$container: $health"
done
echo ""

echo "=== Disk Usage ==="
df -h /var/lib/docker
echo ""

echo "=== System Load ==="
uptime
echo ""
EOF

# Make script executable
chmod +x monitor-containers.sh
Step 2: Run the monitoring script

# Run monitoring script
./monitor-containers.sh
Step 3: Set up automated monitoring (optional)

# Add to crontab for regular monitoring
echo "*/5 * * * * /root/docker-monitoring/monitor-containers.sh >> /var/log/docker-monitor.log 2>&1" | crontab -

# View crontab
crontab -l
Task 5: Scale the Application and Manage it Through Docker Commands
Subtask 5.1: Implement Container Scaling
Step 1: Create a scaling script

# Create scaling script
cat > scale-app.sh << 'EOF'
#!/bin/bash

# Docker Application Scaling Script

SCALE_UP() {
    local count=$1
    echo "Scaling up to $count instances..."
    
    # Stop existing instances
    docker stop $(docker ps -q --filter "name=web-app-") 2>/dev/null || true
    docker rm $(docker ps -aq --filter "name=web-app-") 2>/dev/null || true
    
    # Start new instances
    for i in $(seq 1 $count); do
        port=$((3000 + i))
        docker run -d \
            --name "web-app-$i" \
            --restart unless-stopped \
            -p "$port:3000" \
            docker-web-app:v1.0
        echo "Started web-app-$i on port $port"
    done
    
    # Update Nginx configuration
    update_nginx_config $count
}

SCALE_DOWN() {
    local count=$1
    echo "Scaling down to $count instances..."
    
    # Get current instance count
    current_count=$(docker ps --filter "name=web-app-" --format "{{.Names}}" | wc -l)
    
    # Stop excess instances
    for i in $(seq $((count + 1)) $current_count); do
        docker stop "web-app-$i" 2>/dev/null || true
        docker rm "web-app-$i" 2>/dev/null || true
        echo "Stopped web-app-$i"
    done
    
    # Update Nginx configuration
    update_nginx_config $count
}

update_nginx_config() {
    local count=$1
    echo "Updating Nginx configuration for $count instances..."
    
    # Generate upstream configuration
    upstream_config="upstream docker_backend {\n"
    for i in $(seq 1 $count); do
        port=$((3000 + i))
        upstream_config="$upstream_config    server localhost:$port;\n"
    done
    upstream_config="$upstream_config}"
    
    # Create new Nginx config
    sudo tee /etc/nginx/sites-available/docker-lb << EOF
$upstream_config

server {
    listen 80;
    server_name _;

    location / {
        proxy_pass http://docker_backend;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        
        proxy_connect_timeout 5s;
        proxy_send_timeout 5s;
        proxy_read_timeout 5s;
    }

    location /health {
        proxy_pass http://docker_backend/health;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
    }
}
EOF
    
    # Reload Nginx
    sudo nginx -t && sudo systemctl reload nginx
    echo "Nginx configuration updated and reloaded"
}

# Main script logic
case "$1" in
    up)
        SCALE_UP $2
        ;;
    down)
        SCALE_DOWN $2
        ;;
    status)
        echo "Current running instances:"
        docker ps --filter "name=web-app-" --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
        ;;
    *)
        echo "Usage: $0 {up|down|status} [number_of_instances]"
        echo "Examples:"
        echo "  $0 up 5      # Scale up to 5 instances"
        echo "  $0 down 2    # Scale down to 2 instances"
        echo "  $0 status    # Show current status"
        exit 1
        ;;
esac
EOF

# Make script executable
chmod +x scale-app.sh
Subtask 5.2: Test Application Scaling
Step 1: Scale up the application

# Scale up to 5 instances
./scale-app.sh up 5

# Check status
./scale-app.sh status

# Verify all instances are running
docker ps --filter "name=web-app-"
Step 2: Test load distribution

# Test load balancing across all instances
echo "Testing load distribution:"
for i in {1..10}; do
    echo "Request $i: $(curl -s http://localhost/api/hostname)"
done
Step 3: Scale down the application

# Scale down to 2 instances
./scale-app.sh down 2

# Check status
./scale-app.sh status
Subtask 5.3: Implement Rolling Updates
Step 1: Create a new version of the application

# Create version 2 of the application
cd ~/docker-web-app

# Update the HTML with version info
sed -i 's/Docker Production Lab/Docker Production Lab v2.0/g' index.html
sed -i 's/version": "1.0.0"/version": "2.0.0"/g' package.json

# Build new version
docker build -t docker-web-app:v2.0 .
Step 2: Create rolling update script

# Create rolling update script
cat > rolling-update.sh << 'EOF'
#!/bin/bash

# Rolling Update Script
OLD_VERSION=$1
NEW_VERSION=$2

if [ -z "$OLD_VERSION" ] || [ -z "$NEW_VERSION" ]; then
    echo "Usage: $0 <old_version> <new_version>"
    echo "Example: $0 v1.0 v2.0"
    exit 1
fi

echo "Starting rolling update from $OLD_VERSION to $NEW_VERSION..."

# Get list of running containers
containers=$(docker ps --filter "name=web-app-" --format "{{.Names}}" | sort)

for container in $containers; do
    echo "Updating $container..."
    
    # Get port mapping
    port=$(docker port $container | cut -d':' -f2)
    
    # Start new container with temporary name
    temp_name="${container}-new"
    docker run -d \
        --name "$temp_name" \
        --restart unless-stopped \
        -p "$port:3000" \
        "docker-web-app:$NEW_VERSION"
    
    # Wait for new container to be healthy
    echo "Waiting for $temp_name to be ready..."
    sleep 5
    
    # Health check
    if curl -f "http://localhost:$port/health" > /dev/null 2>&1; then
        echo "$temp_name is healthy"
        
        # Stop old container
        docker stop "$container"
        docker rm "$container"
        
        # Rename new container
        docker rename "$temp_name" "$container"
        
        echo "$container updated successfully"
    else
        echo "Health check failed for $temp_name, rolling back..."
        docker stop "$temp_name"
        docker rm "$temp_name"
        echo "Rollback completed for $container"
    fi
    
    # Wait between updates
    sleep 2
done

echo "Rolling update completed!"
EOF

# Make script executable
chmod +x rolling-update.sh
Step 3: Perform rolling update

# Perform rolling update
./rolling-update.sh v1.0 v2.0

# Verify update
curl http://localhost/
Subtask 5.4: Container Management Commands
Step 1: Create management script with common Docker commands

# Create container management script
cat > manage-containers.sh << 'EOF'
#!/bin/bash

# Container Management Script

show_help() {
    echo "Container Management Commands:"
    echo ""
    echo "Basic Operations:"
    echo "  $0 list                    # List all containers"
    echo "  $0 logs <container_name>   # Show container logs"
    echo "  $0 exec <container_name>   # Execute shell in container"
    echo "  $0 stats                   # Show resource usage"
    echo ""
    echo "Health & Monitoring:"
    echo "  $0 health                  # Check health of all containers"
    echo "  $0 inspect <container>     # Detailed container info"
    echo "  $0 top <container>         # Show processes in container"
    echo ""
    echo "Maintenance:"
    echo "  $0 cleanup                 # Remove stopped containers"
    echo "  $0 prune                   # Remove unused Docker objects"
    echo "  $0 backup                  # Backup container data"
    echo ""
}

list_containers() {
    echo "=== All Containers ==="
    docker ps -a --format "table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}"
}

show_logs() {
    local container=$1
    if [ -z "$container" ]; then
        echo "Please specify container name"
        return 1
    fi
    echo "=== Logs for $container ==="
    docker logs --tail 50 -f "$container"
}

exec_shell() {
    local container=$1
    if [ -z "$container" ]; then
        echo "Please specify container name"
        return 1
    fi
    echo "Opening shell in $container..."
    docker exec -it "$container" /bin/sh
}

show_stats() {
    echo "=== Container Resource Usage ==="
    docker stats --no-stream
}

check_health() {
    echo "=== Container Health Status ==="
    for container in $(docker ps --format "{{.Names}}"); do
        health=$(docker inspect --format='{{.State.Health.Status}}' "$container" 2>/dev/null || echo "no-healthcheck")
        status=$(docker inspect --format='{{.State.Status}}' "$container")
        echo "$container: $status ($health)"
    done
}

inspect_container() {
    local container=$1
    if [ -z "$container" ]; then
        echo "Please specify container name"
        return 1
    fi
    echo "=== Detailed info for $container ==="
    docker inspect "$container" | jq '.[0] | {Name, State, Config: {Image, Env, ExposedPorts}, NetworkSettings: {IPAddress, Ports}}'
}

show_top() {
    local container=$1
    if [ -z "$container" ]; then
        echo "Please specify container name"
        return 1
    fi
    echo "=== Processes in $container ==="
    docker top "$container"
}

cleanup_containers() {
    echo "=== Cleaning up stopped containers ==="
    docker container prune -f
    echo "Cleanup completed"
}

prune_system() {
    echo "=== Pruning unused Docker objects ==="
    docker system prune -f
    echo "System prune completed"
}

backup_data() {
    echo "=== Creating backup ==="
    backup_dir="/tmp/docker-backup-$(date +%Y%m
