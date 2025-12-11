Lab 60: Docker for Continuous Deployment - Blue-Green Deployment with Docker
Objectives
By the end of this lab, you will be able to:

Understand the concept and benefits of blue-green deployment strategy
Create and manage two separate Docker environments (blue and green)
Implement traffic routing between blue and green containers using NGINX
Deploy applications using blue-green deployment methodology
Test failover strategies and simulate deployment failures
Automate the blue-green deployment process using shell scripts
Monitor and validate deployments in both environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and Docker Compose
Familiarity with Linux command line operations
Basic knowledge of web applications and HTTP requests
Understanding of load balancing concepts
Basic shell scripting knowledge
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Docker and Docker Compose already installed. Simply click Start Lab to access your environment. No need to build your own VM or install Docker manually.

Your cloud machine includes:

Ubuntu 20.04 LTS
Docker Engine (latest stable version)
Docker Compose
NGINX
curl and other networking tools
Text editors (nano, vim)
Lab Environment Setup
Task 1: Create Project Structure and Sample Application
Subtask 1.1: Create Project Directory Structure
First, let's create a well-organized directory structure for our blue-green deployment project.

# Create main project directory
mkdir ~/blue-green-deployment
cd ~/blue-green-deployment

# Create subdirectories for different components
mkdir -p app/{blue,green} nginx scripts

# Create directory structure
tree
Subtask 1.2: Create Sample Web Application
We'll create a simple Node.js application that will help us distinguish between blue and green deployments.

Create the application files:

# Create package.json for our Node.js app
cat > app/package.json << 'EOF'
{
  "name": "blue-green-app",
  "version": "1.0.0",
  "description": "Sample app for blue-green deployment",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF
Create the blue version of the application:

# Create blue version server
cat > app/blue/server.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    version: 'blue',
    environment: 'BLUE Environment',
    timestamp: new Date().toISOString(),
    message: 'Welcome to Blue Environment - Version 1.0',
    color: '#0066CC'
  });
});

app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    environment: 'blue',
    uptime: process.uptime()
  });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`Blue app listening at http://0.0.0.0:${port}`);
});
EOF
Create the green version of the application:

# Create green version server
cat > app/green/server.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    version: 'green',
    environment: 'GREEN Environment',
    timestamp: new Date().toISOString(),
    message: 'Welcome to Green Environment - Version 2.0',
    color: '#00CC66'
  });
});

app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    environment: 'green',
    uptime: process.uptime()
  });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`Green app listening at http://0.0.0.0:${port}`);
});
EOF
Subtask 1.3: Create Dockerfile for Applications
Create a Dockerfile that can be used for both blue and green environments:

# Create Dockerfile
cat > app/Dockerfile << 'EOF'
FROM node:16-alpine

WORKDIR /app

# Copy package.json
COPY package.json .

# Install dependencies
RUN npm install

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
EOF
Task 2: Create Docker Compose Configurations for Blue and Green Environments
Subtask 2.1: Create Blue Environment Configuration
Create Docker Compose configuration for the blue environment:

# Create blue environment docker-compose file
cat > docker-compose.blue.yml << 'EOF'
version: '3.8'

services:
  blue-app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: blue-app
    volumes:
      - ./app/blue:/app
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=production
      - APP_VERSION=blue
    networks:
      - blue-green-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  blue-green-network:
    driver: bridge
    name: blue-green-network
EOF
Subtask 2.2: Create Green Environment Configuration
Create Docker Compose configuration for the green environment:

# Create green environment docker-compose file
cat > docker-compose.green.yml << 'EOF'
version: '3.8'

services:
  green-app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: green-app
    volumes:
      - ./app/green:/app
    ports:
      - "3002:3000"
    environment:
      - NODE_ENV=production
      - APP_VERSION=green
    networks:
      - blue-green-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  blue-green-network:
    external: true
EOF
Subtask 2.3: Create Load Balancer Configuration
Create NGINX configuration for traffic routing:

# Create NGINX configuration directory
mkdir -p nginx/conf

# Create NGINX configuration template
cat > nginx/conf/nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server blue-app:3000 weight=100;
        server green-app:3000 weight=0 backup;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Health check timeout
            proxy_connect_timeout 5s;
            proxy_send_timeout 5s;
            proxy_read_timeout 5s;
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF
Create alternative NGINX configurations for different routing scenarios:

# Create blue-active configuration
cat > nginx/conf/nginx-blue.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server blue-app:3000 weight=100;
        server green-app:3000 weight=0 backup;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
EOF

# Create green-active configuration
cat > nginx/conf/nginx-green.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server green-app:3000 weight=100;
        server blue-app:3000 weight=0 backup;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
EOF
Subtask 2.4: Create Load Balancer Docker Compose Configuration
Create Docker Compose configuration for NGINX load balancer:

# Create load balancer docker-compose file
cat > docker-compose.lb.yml << 'EOF'
version: '3.8'

services:
  nginx-lb:
    image: nginx:alpine
    container_name: nginx-lb
    ports:
      - "80:80"
    volumes:
      - ./nginx/conf/nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - blue-green-network
    restart: unless-stopped
    depends_on:
      - blue-app
      - green-app
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  blue-app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: blue-app
    volumes:
      - ./app/blue:/app
    environment:
      - NODE_ENV=production
      - APP_VERSION=blue
    networks:
      - blue-green-network
    restart: unless-stopped

  green-app:
    build:
      context: ./app
      dockerfile: Dockerfile
    container_name: green-app
    volumes:
      - ./app/green:/app
    environment:
      - NODE_ENV=production
      - APP_VERSION=green
    networks:
      - blue-green-network
    restart: unless-stopped

networks:
  blue-green-network:
    driver: bridge
EOF
Task 3: Deploy and Switch Between Blue and Green Environments
Subtask 3.1: Build and Deploy Blue Environment
Let's start by deploying the blue environment:

# Create the network first
docker network create blue-green-network

# Build and start blue environment
docker-compose -f docker-compose.blue.yml up -d --build

# Verify blue deployment
docker-compose -f docker-compose.blue.yml ps

# Test blue environment directly
curl http://localhost:3001
Subtask 3.2: Build and Deploy Green Environment
Deploy the green environment alongside the blue:

# Build and start green environment
docker-compose -f docker-compose.green.yml up -d --build

# Verify green deployment
docker-compose -f docker-compose.green.yml ps

# Test green environment directly
curl http://localhost:3002

# Check both environments are running
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
Subtask 3.3: Deploy Load Balancer
Deploy the NGINX load balancer to route traffic:

# Start the complete stack with load balancer
docker-compose -f docker-compose.lb.yml up -d --build

# Verify all services are running
docker-compose -f docker-compose.lb.yml ps

# Test load balancer (should route to blue by default)
curl http://localhost
Subtask 3.4: Create Deployment Scripts
Create automation scripts for easier management:

# Create deployment script
cat > scripts/deploy.sh << 'EOF'
#!/bin/bash

set -e

ENVIRONMENT=${1:-blue}
ACTION=${2:-deploy}

echo "=== Blue-Green Deployment Script ==="
echo "Environment: $ENVIRONMENT"
echo "Action: $ACTION"
echo "=================================="

case $ACTION in
    "deploy")
        echo "Deploying $ENVIRONMENT environment..."
        docker-compose -f docker-compose.$ENVIRONMENT.yml up -d --build
        echo "Waiting for $ENVIRONMENT environment to be healthy..."
        sleep 10
        
        # Health check
        if [ "$ENVIRONMENT" = "blue" ]; then
            PORT=3001
        else
            PORT=3002
        fi
        
        for i in {1..30}; do
            if curl -f http://localhost:$PORT/health > /dev/null 2>&1; then
                echo "$ENVIRONMENT environment is healthy!"
                break
            fi
            echo "Waiting for $ENVIRONMENT environment... ($i/30)"
            sleep 2
        done
        ;;
    "stop")
        echo "Stopping $ENVIRONMENT environment..."
        docker-compose -f docker-compose.$ENVIRONMENT.yml down
        ;;
    "status")
        echo "Status of $ENVIRONMENT environment:"
        docker-compose -f docker-compose.$ENVIRONMENT.yml ps
        ;;
    *)
        echo "Usage: $0 {blue|green} {deploy|stop|status}"
        exit 1
        ;;
esac
EOF

# Make script executable
chmod +x scripts/deploy.sh
Create a traffic switching script:

# Create traffic switching script
cat > scripts/switch-traffic.sh << 'EOF'
#!/bin/bash

set -e

TARGET_ENV=${1:-blue}

echo "=== Traffic Switching Script ==="
echo "Switching traffic to: $TARGET_ENV"
echo "==============================="

if [ "$TARGET_ENV" != "blue" ] && [ "$TARGET_ENV" != "green" ]; then
    echo "Error: Environment must be 'blue' or 'green'"
    exit 1
fi

# Copy the appropriate nginx configuration
cp nginx/conf/nginx-$TARGET_ENV.conf nginx/conf/nginx.conf

# Reload nginx configuration
docker exec nginx-lb nginx -s reload

echo "Traffic switched to $TARGET_ENV environment"

# Verify the switch
echo "Testing new configuration..."
sleep 2
RESPONSE=$(curl -s http://localhost | jq -r '.environment' 2>/dev/null || echo "Could not parse response")
echo "Current active environment: $RESPONSE"
EOF

# Make script executable
chmod +x scripts/switch-traffic.sh
Task 4: Implement Traffic Routing Between Blue and Green Containers
Subtask 4.1: Test Initial Traffic Routing
Test the current traffic routing setup:

# Test multiple requests to see current routing
echo "Testing current traffic routing:"
for i in {1..5}; do
    echo "Request $i:"
    curl -s http://localhost | jq '.environment, .version'
    echo "---"
done
Subtask 4.2: Switch Traffic to Green Environment
Now let's switch traffic from blue to green:

# Switch traffic to green
./scripts/switch-traffic.sh green

# Test the switch
echo "Testing after switching to green:"
for i in {1..5}; do
    echo "Request $i:"
    curl -s http://localhost | jq '.environment, .version'
    echo "---"
done
Subtask 4.3: Implement Gradual Traffic Shifting
Create a script for gradual traffic shifting (canary deployment):

# Create canary deployment script
cat > scripts/canary-deploy.sh << 'EOF'
#!/bin/bash

set -e

TARGET_ENV=${1:-green}
PERCENTAGE=${2:-10}

echo "=== Canary Deployment Script ==="
echo "Target Environment: $TARGET_ENV"
echo "Traffic Percentage: $PERCENTAGE%"
echo "==============================="

# Calculate weights
if [ "$TARGET_ENV" = "green" ]; then
    GREEN_WEIGHT=$PERCENTAGE
    BLUE_WEIGHT=$((100 - PERCENTAGE))
else
    BLUE_WEIGHT=$PERCENTAGE
    GREEN_WEIGHT=$((100 - PERCENTAGE))
fi

# Create canary nginx configuration
cat > nginx/conf/nginx.conf << EOF
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server blue-app:3000 weight=$BLUE_WEIGHT;
        server green-app:3000 weight=$GREEN_WEIGHT;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://backend;
            proxy_set_header Host \$host;
            proxy_set_header X-Real-IP \$remote_addr;
            proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto \$scheme;
        }

        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
EOF

# Reload nginx
docker exec nginx-lb nginx -s reload

echo "Canary deployment configured: Blue($BLUE_WEIGHT%) Green($GREEN_WEIGHT%)"
EOF

# Make script executable
chmod +x scripts/canary-deploy.sh
Subtask 4.4: Test Canary Deployment
Test the canary deployment with different traffic percentages:

# Start with 20% traffic to green
./scripts/canary-deploy.sh green 20

# Test traffic distribution
echo "Testing canary deployment (20% green, 80% blue):"
for i in {1..20}; do
    RESPONSE=$(curl -s http://localhost | jq -r '.version')
    echo -n "$RESPONSE "
done
echo ""

# Increase to 50% traffic to green
./scripts/canary-deploy.sh green 50

echo "Testing canary deployment (50% green, 50% blue):"
for i in {1..20}; do
    RESPONSE=$(curl -s http://localhost | jq -r '.version')
    echo -n "$RESPONSE "
done
echo ""
Task 5: Test Failover Strategy by Simulating Deployment Failure
Subtask 5.1: Create Health Check Monitoring Script
Create a script to monitor the health of both environments:

# Create health monitoring script
cat > scripts/health-monitor.sh << 'EOF'
#!/bin/bash

BLUE_URL="http://localhost:3001/health"
GREEN_URL="http://localhost:3002/health"
LB_URL="http://localhost/health"

echo "=== Health Monitoring ==="
echo "Timestamp: $(date)"
echo "========================"

# Check blue environment
echo -n "Blue Environment: "
if curl -f -s $BLUE_URL > /dev/null 2>&1; then
    echo "✓ HEALTHY"
    BLUE_STATUS=$(curl -s $BLUE_URL | jq -r '.status')
    echo "  Status: $BLUE_STATUS"
else
    echo "✗ UNHEALTHY"
fi

# Check green environment
echo -n "Green Environment: "
if curl -f -s $GREEN_URL > /dev/null 2>&1; then
    echo "✓ HEALTHY"
    GREEN_STATUS=$(curl -s $GREEN_URL | jq -r '.status')
    echo "  Status: $GREEN_STATUS"
else
    echo "✗ UNHEALTHY"
fi

# Check load balancer
echo -n "Load Balancer: "
if curl -f -s $LB_URL > /dev/null 2>&1; then
    echo "✓ HEALTHY"
else
    echo "✗ UNHEALTHY"
fi

echo "========================"
EOF

# Make script executable
chmod +x scripts/health-monitor.sh

# Test health monitoring
./scripts/health-monitor.sh
Subtask 5.2: Simulate Application Failure
Let's simulate a failure in the green environment:

# First, switch all traffic to green
./scripts/switch-traffic.sh green

# Verify traffic is going to green
echo "Current traffic routing:"
curl -s http://localhost | jq '.environment, .version'

# Simulate green environment failure by stopping it
echo "Simulating green environment failure..."
docker stop green-app

# Check health status
./scripts/health-monitor.sh

# Test if load balancer fails over to blue
echo "Testing failover to blue environment:"
for i in {1..5}; do
    echo "Request $i:"
    curl -s http://localhost | jq '.environment, .version' || echo "Request failed"
    sleep 1
done
Subtask 5.3: Create Automatic Failover Script
Create a script that automatically handles failover:

# Create automatic failover script
cat > scripts/auto-failover.sh << 'EOF'
#!/bin/bash

set -e

CURRENT_ENV=${1:-green}
BACKUP_ENV=${2:-blue}

echo "=== Automatic Failover Script ==="
echo "Current Environment: $CURRENT_ENV"
echo "Backup Environment: $BACKUP_ENV"
echo "================================"

# Function to check environment health
check_health() {
    local env=$1
    local port
    
    if [ "$env" = "blue" ]; then
        port=3001
    else
        port=3002
    fi
    
    if curl -f -s http://localhost:$port/health > /dev/null 2>&1; then
        return 0
    else
        return 1
    fi
}

# Check current environment health
if check_health $CURRENT_ENV; then
    echo "$CURRENT_ENV environment is healthy"
else
    echo "$CURRENT_ENV environment is unhealthy - initiating failover"
    
    # Check if backup environment is healthy
    if check_health $BACKUP_ENV; then
        echo "$BACKUP_ENV environment is healthy - switching traffic"
        ./scripts/switch-traffic.sh $BACKUP_ENV
        echo "Failover completed successfully"
    else
        echo "ERROR: Both environments are unhealthy!"
        exit 1
    fi
fi
EOF

# Make script executable
chmod +x scripts/auto-failover.sh
Subtask 5.4: Test Automatic Failover
Test the automatic failover functionality:

# Restart green environment
docker start green-app
sleep 5

# Switch traffic back to green
./scripts/switch-traffic.sh green

# Test automatic failover when green fails
echo "Testing automatic failover..."
docker stop green-app
sleep 2

# Run failover script
./scripts/auto-failover.sh green blue

# Verify failover worked
echo "Verifying failover:"
curl -s http://localhost | jq '.environment, .version'
Subtask 5.5: Simulate Network Issues
Create a script to simulate network issues:

# Create network simulation script
cat > scripts/simulate-issues.sh << 'EOF'
#!/bin/bash

ISSUE_TYPE=${1:-latency}
ENVIRONMENT=${2:-green}

echo "=== Network Issue Simulation ==="
echo "Issue Type: $ISSUE_TYPE"
echo "Target Environment: $ENVIRONMENT"
echo "==============================="

case $ISSUE_TYPE in
    "latency")
        echo "Simulating high latency in $ENVIRONMENT environment..."
        # Add network delay using tc (traffic control)
        docker exec ${ENVIRONMENT}-app sh -c "apk add --no-cache iproute2 && tc qdisc add dev eth0 root netem delay 2000ms" 2>/dev/null || echo "Latency simulation requires additional setup"
        ;;
    "packet-loss")
        echo "Simulating packet loss in $ENVIRONMENT environment..."
        docker exec ${ENVIRONMENT}-app sh -c "apk add --no-cache iproute2 && tc qdisc add dev eth0 root netem loss 50%" 2>/dev/null || echo "Packet loss simulation requires additional setup"
        ;;
    "stop")
        echo "Stopping $ENVIRONMENT environment..."
        docker stop ${ENVIRONMENT}-app
        ;;
    "restart")
        echo "Restarting $ENVIRONMENT environment..."
        docker restart ${ENVIRONMENT}-app
        ;;
    "reset")
        echo "Resetting network conditions for $ENVIRONMENT environment..."
        docker exec ${ENVIRONMENT}-app sh -c "tc qdisc del dev eth0 root" 2>/dev/null || echo "No network rules to reset"
        ;;
    *)
        echo "Usage: $0 {latency|packet-loss|stop|restart|reset} {blue|green}"
        exit 1
        ;;
esac
EOF

# Make script executable
chmod +x scripts/simulate-issues.sh
Task 6: Automate the Deployment Process with CI/CD Tools
Subtask 6.1: Create Complete Deployment Pipeline Script
Create a comprehensive deployment pipeline script:

# Create CI/CD pipeline script
cat > scripts/cicd-pipeline.sh << 'EOF'
#!/bin/bash

set -e

# Configuration
CURRENT_ENV_FILE=".current-env"
DEFAULT_ENV="blue"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Functions
log_info() {
    echo -e "${GREEN}[INFO]${NC} $1"
}

log_warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

log_error() {
    echo -e "${RED}[ERROR]${NC} $1"
}

get_current_env() {
    if [ -f "$CURRENT_ENV_FILE" ]; then
        cat $CURRENT_ENV_FILE
    else
        echo $DEFAULT_ENV
    fi
}

get_target_env() {
    local current=$(get_current_env)
    if [ "$current" = "blue" ]; then
        echo "green"
    else
        echo "blue"
    fi
}

health_check() {
    local env=$1
    local port
    
    if [ "$env" = "blue" ]; then
        port=3001
    else
        port=3002
    fi
    
    log_info "Performing health check for $env environment..."
    
    for i in {1..30}; do
        if curl -f -s http://localhost:$port/health > /dev/null 2>&1; then
            log_info "$env environment is healthy"
            return 0
        fi
        log_warn "Health check attempt $i/30 failed, retrying..."
        sleep 2
    done
    
    log_error "$env environment failed health check"
    return 1
}

deploy_environment() {
    local env=$1
    
    log_info "Deploying $env environment..."
    
    # Build and deploy
    docker-compose -f docker-compose.$env.yml up -d --build
    
    # Wait for deployment
    sleep 10
    
    # Health check
    if health_check $env; then
        log_info "$env environment deployed successfully"
        return 0
    else
        log_error "$env environment deployment failed"
        return 1
    fi
}

switch_traffic() {
    local target_env=$1
    
    log_info "Switching traffic to $target_env environment..."
    
    # Update nginx configuration
    cp nginx/conf/nginx-$target_env.conf nginx/conf/nginx.conf
    
    # Reload nginx
    docker exec nginx-lb nginx -s reload
    
    # Update current environment file
    echo $target_env > $CURRENT_ENV_FILE
    
    log_info "Traffic switched to $target_env environment"
}

rollback() {
    local current_env=$(get_current_env)
    local rollback_env
    
    if [ "$current_env" = "blue" ]; then
        rollback_env="green"
    else
        rollback_env="blue"
    fi
    
    log_warn "Initiating rollback to $rollback_env environment..."
    
    if health_check $rollback_env; then
        switch_traffic $rollback_env
        log_info "Rollback completed successfully"
    else
        log_error "Rollback failed - $rollback_env environment is also unhealthy"
        exit 1
    fi
}

# Main deployment pipeline
main() {
    local action=${1:-deploy}
    
    log_info "=== Blue-Green CI/CD Pipeline ==="
    log_info "Action: $action"
    log_info "Current Environment: $(get_current_env)"
    log_info "Target Environment: $(get_target_env)"
    log_info "================================="
    
    case $action in
        "deploy")
            local current_env=$(get_current_env)
            local target_env=$(get_target_env)
            
            # Deploy to target environment
            if deploy_environment $target_env; then
                # Switch traffic
                switch_traffic $target_env
                
                # Verify deployment
                sleep 5
                if health_check $target_env; then
                    log_info "Deployment completed successfully"
                    
                    # Optional: Stop old environment after successful deployment
                    log_info "Stopping old $current_env environment..."
                    docker-compose -f docker-compose.$current_env.yml down
                else
                    log_error "Post-deployment health check failed"
                    rollback
                fi
            else
                log_error "Deployment failed"
                exit 1
            fi
            ;;
        "rollback")
            rollback
            ;;
        "status")
            log_info "Current Environment: $(get_current_env)"
            ./scripts/health-monitor.sh
            ;;
        "canary")
            local percentage=${2:-10}
            local target_env=$(get_target_env)
            
            if deploy_environment $target_env; then
                log_info "Starting canary deployment with $percentage% traffic"
                ./scripts/canary-deploy.sh $target_env $percentage
            fi
            ;;
        *)
            echo "Usage: $0 {deploy|rollback|status|canary [percentage]}"
            exit 1
            ;;
    esac
}

# Run main function
main "$@"
EOF

# Make script executable
chmod +x scripts/cicd-pipeline.sh
Subtask 6.2: Create Environment Configuration
Create environment-specific configuration files:

# Create environment configuration
cat > .env.blue << 'EOF'
APP_VERSION=blue
APP_COLOR=#0066CC
APP_MESSAGE=Welcome to Blue Environment - Version 1.0
ENVIRONMENT=blue
NODE_ENV=production
EOF

cat > .env.green << 'EOF'
APP_VERSION=green
APP_COLOR=#00CC66
APP_MESSAGE=Welcome to Green Environment - Version 2.0
ENVIRONMENT=green
NODE_ENV=production
EOF

# Create deployment configuration
cat > deployment-config.yml << 'EOF'
deployment:
  strategy: blue-green
  health_check:
    timeout: 30
    retries: 3
    interval: 2
  rollback:
    enabled: true
    automatic: true
  environments:
    blue:
      port: 3001
      weight: 100
    green:
      port: 3002
      weight: 0
EOF
Subtask 6.3: Test Complete CI/CD Pipeline
Test the complete CI/CD pipeline:

# Initialize the pipeline (start with blue environment)
echo "blue" > .current-env

# Deploy initial blue environment
./scripts/cicd-pipeline.sh deploy

# Check status
./scripts/cicd-pipeline.sh status

# Test canary deployment
./scripts/cicd-pipeline.sh canary 25

# Test full deployment
./scripts/cicd-pipeline.
