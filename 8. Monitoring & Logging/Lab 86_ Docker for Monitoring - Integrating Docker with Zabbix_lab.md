Lab 86: Docker for Monitoring - Integrating Docker with Zabbix
Lab Objectives
By the end of this lab, students will be able to:

• Install and configure Zabbix Server and Agent using Docker containers • Set up monitoring for Docker containers' health and performance metrics • Configure Zabbix triggers and actions based on container performance thresholds • Integrate Zabbix with Docker's API to collect real-time container metrics • Create custom Zabbix dashboards for visualizing Docker container statistics • Understand the fundamentals of container monitoring in production environments

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Docker concepts (containers, images, volumes) • Familiarity with Linux command line operations • Basic knowledge of networking concepts (ports, IP addresses) • Understanding of monitoring concepts (metrics, alerts, dashboards) • No prior Zabbix experience required - this lab covers everything from basics

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your dedicated environment. No need to build your own VM or install Docker - everything is ready to use.

Your lab environment includes: • Ubuntu 20.04 LTS with Docker and Docker Compose pre-installed • 4GB RAM and 20GB storage • All necessary ports configured for Zabbix communication • Internet access for downloading required images

Task 1: Install Zabbix Server and Agent in Docker Containers
Subtask 1.1: Create Project Directory Structure
First, let's organize our lab environment by creating a dedicated directory structure.

# Create main project directory
mkdir -p ~/zabbix-docker-lab
cd ~/zabbix-docker-lab

# Create subdirectories for configuration files
mkdir -p config/zabbix-server
mkdir -p config/zabbix-agent
mkdir -p data/mysql
mkdir -p data/zabbix
Subtask 1.2: Create Docker Compose Configuration
Create a comprehensive Docker Compose file that will deploy our entire Zabbix monitoring stack.

# Create the main docker-compose.yml file
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # MySQL Database for Zabbix
  mysql-server:
    image: mysql:8.0
    container_name: zabbix-mysql
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_password
      MYSQL_ROOT_PASSWORD: root_password
    command:
      - mysqld
      - --character-set-server=utf8
      - --collation-server=utf8_bin
      - --default-authentication-plugin=mysql_native_password
    volumes:
      - ./data/mysql:/var/lib/mysql
    networks:
      - zabbix-network

  # Zabbix Server
  zabbix-server:
    image: zabbix/zabbix-server-mysql:alpine-6.4-latest
    container_name: zabbix-server
    restart: unless-stopped
    environment:
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_password
      MYSQL_ROOT_PASSWORD: root_password
    ports:
      - "10051:10051"
    volumes:
      - ./data/zabbix:/var/lib/zabbix
    depends_on:
      - mysql-server
    networks:
      - zabbix-network

  # Zabbix Web Interface
  zabbix-web:
    image: zabbix/zabbix-web-apache-mysql:alpine-6.4-latest
    container_name: zabbix-web
    restart: unless-stopped
    environment:
      ZBX_SERVER_HOST: zabbix-server
      DB_SERVER_HOST: mysql-server
      MYSQL_DATABASE: zabbix
      MYSQL_USER: zabbix
      MYSQL_PASSWORD: zabbix_password
      PHP_TZ: America/New_York
    ports:
      - "8080:8080"
    depends_on:
      - mysql-server
      - zabbix-server
    networks:
      - zabbix-network

  # Zabbix Agent for monitoring the Docker host
  zabbix-agent:
    image: zabbix/zabbix-agent:alpine-6.4-latest
    container_name: zabbix-agent
    restart: unless-stopped
    environment:
      ZBX_HOSTNAME: docker-host
      ZBX_SERVER_HOST: zabbix-server
      ZBX_SERVER_PORT: 10051
    ports:
      - "10050:10050"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /dev:/host/dev:ro
    privileged: true
    pid: host
    networks:
      - zabbix-network

networks:
  zabbix-network:
    driver: bridge
EOF
Subtask 1.3: Deploy the Zabbix Stack
Now let's deploy our complete Zabbix monitoring infrastructure.

# Start all services
docker-compose up -d

# Verify all containers are running
docker-compose ps

# Check container logs to ensure proper startup
docker-compose logs zabbix-server | tail -20
docker-compose logs zabbix-web | tail -20
Subtask 1.4: Verify Installation
Wait for all services to initialize (this may take 2-3 minutes), then verify the installation.

# Check if all containers are healthy
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

# Test Zabbix server connectivity
docker exec zabbix-server zabbix_server -R config_cache_reload

# Verify database connection
docker exec zabbix-mysql mysql -u zabbix -pzabbix_password -e "SHOW DATABASES;"
Task 2: Configure Zabbix to Monitor Docker Containers' Health
Subtask 2.1: Access Zabbix Web Interface
First, let's access the Zabbix web interface and complete the initial setup.

# Get your lab machine's IP address
ip addr show | grep "inet " | grep -v 127.0.0.1

# The Zabbix web interface will be available at:
# http://YOUR_LAB_IP:8080
Web Interface Setup Steps:

Open your web browser and navigate to http://YOUR_LAB_IP:8080
Default login credentials:
Username: Admin
Password: zabbix
Click Sign in to access the Zabbix dashboard
Subtask 2.2: Create Docker Monitoring Template
Let's create a custom template for monitoring Docker containers.

# Create a script to gather Docker metrics
cat > config/docker-stats.sh << 'EOF'
#!/bin/bash

# Docker container monitoring script for Zabbix
case $1 in
    "container.count")
        docker ps -q | wc -l
        ;;
    "container.running")
        docker ps --filter "status=running" -q | wc -l
        ;;
    "container.stopped")
        docker ps --filter "status=exited" -q | wc -l
        ;;
    "container.cpu")
        if [ -n "$2" ]; then
            docker stats --no-stream --format "{{.CPUPerc}}" $2 | sed 's/%//'
        fi
        ;;
    "container.memory")
        if [ -n "$2" ]; then
            docker stats --no-stream --format "{{.MemUsage}}" $2 | cut -d'/' -f1 | sed 's/[^0-9.]//g'
        fi
        ;;
    "container.status")
        if [ -n "$2" ]; then
            docker inspect --format='{{.State.Status}}' $2 2>/dev/null || echo "not_found"
        fi
        ;;
    *)
        echo "Usage: $0 {container.count|container.running|container.stopped|container.cpu|container.memory|container.status} [container_name]"
        exit 1
        ;;
esac
EOF

# Make the script executable
chmod +x config/docker-stats.sh

# Copy the script to the Zabbix agent container
docker cp config/docker-stats.sh zabbix-agent:/usr/local/bin/docker-stats.sh
docker exec zabbix-agent chmod +x /usr/local/bin/docker-stats.sh
Subtask 2.3: Configure Zabbix Agent for Docker Monitoring
Create a custom Zabbix agent configuration to enable Docker monitoring.

# Create custom Zabbix agent configuration
cat > config/zabbix-agent/docker-monitoring.conf << 'EOF'
# Docker monitoring user parameters
UserParameter=docker.container.count,/usr/local/bin/docker-stats.sh container.count
UserParameter=docker.container.running,/usr/local/bin/docker-stats.sh container.running
UserParameter=docker.container.stopped,/usr/local/bin/docker-stats.sh container.stopped
UserParameter=docker.container.cpu[*],/usr/local/bin/docker-stats.sh container.cpu $1
UserParameter=docker.container.memory[*],/usr/local/bin/docker-stats.sh container.memory $1
UserParameter=docker.container.status[*],/usr/local/bin/docker-stats.sh container.status $1
UserParameter=docker.version,docker --version | cut -d' ' -f3 | sed 's/,//'
UserParameter=docker.info.containers,docker info --format '{{.Containers}}'
UserParameter=docker.info.images,docker info --format '{{.Images}}'
EOF

# Copy configuration to the agent container
docker cp config/zabbix-agent/docker-monitoring.conf zabbix-agent:/etc/zabbix/zabbix_agentd.d/

# Restart the Zabbix agent to load new configuration
docker restart zabbix-agent

# Wait for agent to restart
sleep 10

# Test the custom parameters
docker exec zabbix-agent zabbix_agentd -t docker.container.count
docker exec zabbix-agent zabbix_agentd -t docker.container.running
Subtask 2.4: Add Host in Zabbix Web Interface
Now let's configure the Zabbix server to monitor our Docker host through the web interface.

Web Interface Configuration Steps:

In Zabbix web interface, go to Configuration → Hosts
Click Create host
Fill in the host details:
Host name: docker-host
Visible name: Docker Host Monitor
Groups: Select Linux servers (click Select button)
Interfaces:
Type: Agent
IP address: zabbix-agent (container name)
Port: 10050
Go to Templates tab
Click Select next to Link new templates
Choose Linux by Zabbix agent template
Click Add then Add again to create the host
Task 3: Set Up Triggers and Actions Based on Container Performance
Subtask 3.1: Create Docker Container Items
Let's create monitoring items for Docker containers through the web interface.

Web Interface Steps:

Go to Configuration → Hosts
Click on Items next to docker-host
Click Create item and add the following items one by one:
Item 1: Total Container Count

Name: Docker Total Containers
Type: Zabbix agent
Key: docker.container.count
Type of information: Numeric (unsigned)
Update interval: 30s
Click Add
Item 2: Running Containers

Name: Docker Running Containers
Type: Zabbix agent
Key: docker.container.running
Type of information: Numeric (unsigned)
Update interval: 30s
Click Add
Item 3: Stopped Containers

Name: Docker Stopped Containers
Type: Zabbix agent
Key: docker.container.stopped
Type of information: Numeric (unsigned)
Update interval: 30s
Click Add
Subtask 3.2: Create Performance Triggers
Now let's create triggers that will alert us when container performance issues occur.

Web Interface Steps:

Go to Configuration → Hosts
Click on Triggers next to docker-host
Click Create trigger
Trigger 1: High Number of Stopped Containers

Name: Too many stopped Docker containers
Severity: Warning
Expression: Click Add and configure:
Item: Docker Stopped Containers
Function: last()
Operator: >
Value: 3
Description: More than 3 containers are in stopped state
Click Add
Trigger 2: No Running Containers

Name: No Docker containers running
Severity: High
Expression: Click Add and configure:
Item: Docker Running Containers
Function: last()
Operator: =
Value: 0
Description: All Docker containers have stopped
Click Add
Subtask 3.3: Create Test Containers
Let's create some test containers to verify our monitoring setup.

# Create test containers to monitor
docker run -d --name test-web-1 nginx:alpine
docker run -d --name test-web-2 nginx:alpine
docker run -d --name test-app-1 httpd:alpine

# Create a container that will stop (for testing stopped container trigger)
docker run --name test-temp alpine echo "This container will stop"

# Verify containers are created
docker ps -a

# Check our monitoring items are working
docker exec zabbix-agent zabbix_agentd -t docker.container.count
docker exec zabbix-agent zabbix_agentd -t docker.container.running
docker exec zabbix-agent zabbix_agentd -t docker.container.stopped
Subtask 3.4: Configure Actions for Triggers
Let's set up automated actions when triggers fire.

Web Interface Steps:

Go to Configuration → Actions
Make sure Trigger actions is selected
Click Create action
Action Configuration:

Name: Docker Container Alert
Conditions: Click Add
Type: Trigger
Operator: equals
Value: Select both triggers we created
Operations tab: Click Add
Operation type: Send message
Send to users: Admin
Send only to: Zabbix
Subject: Docker Alert: {TRIGGER.NAME}
Message:
Problem: {TRIGGER.NAME}
Host: {HOST.NAME}
Severity: {TRIGGER.SEVERITY}
Time: {EVENT.DATE} {EVENT.TIME}

Docker container issue detected on {HOST.NAME}
Current status: {ITEM.LASTVALUE}
Click Add to save the operation
Click Add to save the action
Task 4: Integrate Zabbix with Docker's API to Gather Container Metrics
Subtask 4.1: Create Docker API Monitoring Script
Let's create a more advanced script that uses Docker's API to gather detailed container metrics.

# Create advanced Docker API monitoring script
cat > config/docker-api-monitor.sh << 'EOF'
#!/bin/bash

# Docker API monitoring script using docker commands
# This script provides detailed container metrics

get_container_list() {
    docker ps --format "{{.Names}}" | tr '\n' ','
}

get_container_cpu_usage() {
    local container=$1
    if [ -n "$container" ]; then
        docker stats --no-stream --format "{{.CPUPerc}}" "$container" 2>/dev/null | sed 's/%//' || echo "0"
    fi
}

get_container_memory_usage() {
    local container=$1
    if [ -n "$container" ]; then
        # Get memory usage in MB
        docker stats --no-stream --format "{{.MemUsage}}" "$container" 2>/dev/null | \
        cut -d'/' -f1 | sed 's/[^0-9.]//g' || echo "0"
    fi
}

get_container_memory_limit() {
    local container=$1
    if [ -n "$container" ]; then
        # Get memory limit in MB
        docker stats --no-stream --format "{{.MemUsage}}" "$container" 2>/dev/null | \
        cut -d'/' -f2 | sed 's/[^0-9.]//g' || echo "0"
    fi
}

get_container_network_io() {
    local container=$1
    local direction=$2  # "rx" or "tx"
    if [ -n "$container" ]; then
        if [ "$direction" = "rx" ]; then
            docker stats --no-stream --format "{{.NetIO}}" "$container" 2>/dev/null | \
            cut -d'/' -f1 | sed 's/[^0-9.]//g' || echo "0"
        else
            docker stats --no-stream --format "{{.NetIO}}" "$container" 2>/dev/null | \
            cut -d'/' -f2 | sed 's/[^0-9.]//g' || echo "0"
        fi
    fi
}

get_container_uptime() {
    local container=$1
    if [ -n "$container" ]; then
        docker inspect --format='{{.State.StartedAt}}' "$container" 2>/dev/null | \
        xargs -I {} date -d {} +%s 2>/dev/null || echo "0"
    fi
}

# Main script logic
case $1 in
    "api.container.list")
        get_container_list
        ;;
    "api.container.cpu")
        get_container_cpu_usage "$2"
        ;;
    "api.container.memory.usage")
        get_container_memory_usage "$2"
        ;;
    "api.container.memory.limit")
        get_container_memory_limit "$2"
        ;;
    "api.container.network.rx")
        get_container_network_io "$2" "rx"
        ;;
    "api.container.network.tx")
        get_container_network_io "$2" "tx"
        ;;
    "api.container.uptime")
        get_container_uptime "$2"
        ;;
    "api.docker.version")
        docker version --format '{{.Server.Version}}' 2>/dev/null || echo "unknown"
        ;;
    *)
        echo "Usage: $0 {api.container.list|api.container.cpu|api.container.memory.usage|api.container.memory.limit|api.container.network.rx|api.container.network.tx|api.container.uptime|api.docker.version} [container_name]"
        exit 1
        ;;
esac
EOF

# Make the script executable
chmod +x config/docker-api-monitor.sh

# Copy to Zabbix agent container
docker cp config/docker-api-monitor.sh zabbix-agent:/usr/local/bin/docker-api-monitor.sh
docker exec zabbix-agent chmod +x /usr/local/bin/docker-api-monitor.sh
Subtask 4.2: Add API Monitoring Parameters
Create additional Zabbix agent parameters for API-based monitoring.

# Create API monitoring configuration
cat > config/zabbix-agent/docker-api-monitoring.conf << 'EOF'
# Docker API monitoring user parameters
UserParameter=docker.api.container.list,/usr/local/bin/docker-api-monitor.sh api.container.list
UserParameter=docker.api.container.cpu[*],/usr/local/bin/docker-api-monitor.sh api.container.cpu $1
UserParameter=docker.api.container.memory.usage[*],/usr/local/bin/docker-api-monitor.sh api.container.memory.usage $1
UserParameter=docker.api.container.memory.limit[*],/usr/local/bin/docker-api-monitor.sh api.container.memory.limit $1
UserParameter=docker.api.container.network.rx[*],/usr/local/bin/docker-api-monitor.sh api.container.network.rx $1
UserParameter=docker.api.container.network.tx[*],/usr/local/bin/docker-api-monitor.sh api.container.network.tx $1
UserParameter=docker.api.container.uptime[*],/usr/local/bin/docker-api-monitor.sh api.container.uptime $1
UserParameter=docker.api.version,/usr/local/bin/docker-api-monitor.sh api.docker.version
EOF

# Copy configuration to agent
docker cp config/zabbix-agent/docker-api-monitoring.conf zabbix-agent:/etc/zabbix/zabbix_agentd.d/

# Restart agent to load new configuration
docker restart zabbix-agent
sleep 10

# Test the new API parameters
echo "Testing Docker API monitoring parameters:"
docker exec zabbix-agent zabbix_agentd -t docker.api.container.list
docker exec zabbix-agent zabbix_agentd -t docker.api.version
docker exec zabbix-agent zabbix_agentd -t "docker.api.container.cpu[test-web-1]"
Subtask 4.3: Create Container-Specific Monitoring Items
Now let's add specific monitoring items for individual containers.

Web Interface Steps:

Go to Configuration → Hosts
Click on Items next to docker-host
Create the following items:
Item 1: Container CPU Usage (test-web-1)

Name: Container CPU Usage - test-web-1
Type: Zabbix agent
Key: docker.api.container.cpu[test-web-1]
Type of information: Numeric (float)
Units: %
Update interval: 30s
Click Add
Item 2: Container Memory Usage (test-web-1)

Name: Container Memory Usage - test-web-1
Type: Zabbix agent
Key: docker.api.container.memory.usage[test-web-1]
Type of information: Numeric (float)
Units: MB
Update interval: 30s
Click Add
Item 3: Docker Version

Name: Docker Version
Type: Zabbix agent
Key: docker.api.version
Type of information: Text
Update interval: 3600s
Click Add
Subtask 4.4: Create Container Performance Triggers
Add triggers for container-specific performance monitoring.

Web Interface Steps:

Go to Configuration → Hosts
Click on Triggers next to docker-host
Create these triggers:
Trigger 1: High Container CPU Usage

Name: High CPU usage on container test-web-1
Severity: Warning
Expression:
Item: Container CPU Usage - test-web-1
Function: avg(5m)
Operator: >
Value: 80
Description: Container test-web-1 CPU usage is above 80% for 5 minutes
Click Add
Trigger 2: High Container Memory Usage

Name: High memory usage on container test-web-1
Severity: Warning
Expression:
Item: Container Memory Usage - test-web-1
Function: last()
Operator: >
Value: 500
Description: Container test-web-1 memory usage is above 500MB
Click Add
Task 5: Create Custom Zabbix Dashboards to Visualize Docker Container Stats
Subtask 5.1: Create Docker Overview Dashboard
Let's create a comprehensive dashboard for Docker monitoring.

Web Interface Steps:

Go to Monitoring → Dashboards
Click Create dashboard
Dashboard name: Docker Container Overview
Click Add to create the dashboard
Subtask 5.2: Add Container Count Widget
Now let's add widgets to visualize our Docker metrics.

Web Interface Steps:

Click Edit dashboard on your new dashboard
Click Add widget
Type: Graph (classic)
Name: Docker Container Status
Graph: Click Select
Click Create graph
Name: Container Count Graph
Items: Click Add
Host: docker-host
Item: Docker Total Containers
Click Add
Items: Click Add again
Host: docker-host
Item: Docker Running Containers
Click Add
Items: Click Add again
Host: docker-host
Item: Docker Stopped Containers
Click Add
Click Add to save the graph
Click Add to add the widget
Subtask 5.3: Add Container Performance Widget
Add a widget to show container performance metrics.

Web Interface Steps:

Click Add widget (while still editing the dashboard)
Type: Graph (classic)
Name: Container Performance
Graph: Click Select
Click Create graph
Name: Container CPU and Memory
Items: Click Add
Host: docker-host
Item: Container CPU Usage - test-web-1
Y axis side: Left
Click Add
Items: Click Add again
Host: docker-host
Item: Container Memory Usage - test-web-1
Y axis side: Right
Click Add
Click Add to save the graph
Click Add to add the widget
Subtask 5.4: Add System Information Widget
Add a widget showing Docker system information.

Web Interface Steps:

Click Add widget
Type: Plain text
Name: Docker System Info
Items: Click Add
Host: docker-host
Item: Docker Version
Click Add
Click Add to add the widget
Subtask 5.5: Add Problems Widget
Add a widget to show current Docker-related problems.

Web Interface Steps:

Click Add widget
Type: Problems
Name: Docker Problems
Host groups: Select Linux servers
Hosts: Select docker-host
Problem: Leave empty (to show all problems)
Severity: Select Not classified, Information, Warning, Average, High, Disaster
Click Add to add the widget
Subtask 5.6: Save and View Dashboard
Complete the dashboard setup.

Web Interface Steps:

Click Save changes to save the dashboard
Go to Monitoring → Dashboards
Click on Docker Container Overview to view your dashboard
You should now see all widgets displaying Docker metrics in real-time
Subtask 5.7: Generate Load for Testing
Let's create some load on our containers to see the monitoring in action.

# Create CPU load on test-web-1 container
docker exec test-web-1 sh -c 'while true; do echo "Creating CPU load..."; done' &
LOAD_PID=$!

# Wait a few minutes to see the metrics change
echo "Generating load for 2 minutes to test monitoring..."
sleep 120

# Stop the load generation
kill $LOAD_PID 2>/dev/null

# Create and stop some containers to trigger alerts
docker run -d --name temp-container-1 alpine sleep 10
docker run -d --name temp-container-2 alpine sleep 10
docker run -d --name temp-container-3 alpine sleep 10
docker run -d --name temp-container-4 alpine sleep 10

# Wait for containers to stop
sleep 15

# Check the dashboard to see the changes
echo "Check your Zabbix dashboard to see the monitoring data and any triggered alerts!"
Troubleshooting Common Issues
Issue 1: Containers Not Starting
If containers fail to start, check the logs:

# Check container logs
docker-compose logs mysql-server
docker-compose logs zabbix-server
docker-compose logs zabbix-web

# Check if ports are already in use
netstat -tlnp | grep -E ':(8080|10050|10051)'

# Restart the entire stack if needed
docker-compose down
docker-compose up -d
Issue 2: Zabbix Agent Not Responding
If the Zabbix agent is not responding to custom parameters:

# Check agent configuration
docker exec zabbix-agent cat /etc/zabbix/zabbix_agentd.conf | grep -E '(Server|ServerActive)'

# Test agent connectivity
docker exec zabbix-server zabbix_get -s zabbix-agent -k agent.ping

# Check custom parameter files
docker exec zabbix-agent ls -la /etc/zabbix/zabbix_agentd.d/

# Restart agent with debug mode
docker exec zabbix-agent zabbix_agentd -f -d 4
Issue 3: Web Interface Not Accessible
If you cannot access the Zabbix web interface:

# Check if web container is running
docker ps | grep zabbix-web

# Check web container logs
docker logs zabbix-web

# Verify port mapping
docker port zabbix-web

# Test local connectivity
curl -I http://localhost:8080
Issue 4: Database Connection Issues
If there are database connectivity problems:

# Check MySQL container status
docker exec zabbix-mysql mysqladmin -u root -proot_password status

# Test database connection from Zabbix server
docker exec zabbix-server mysql -h mysql-server -u zabbix -pzabbix_password -e "SELECT VERSION();"

# Check database initialization
docker exec zabbix-mysql mysql -u zabbix -pzabbix_password -e "USE zabbix; SHOW TABLES;" | wc -l
Lab Verification and Testing
Verification Checklist
Complete these steps to verify your lab setup:

# 1. Verify all containers are running
echo "=== Container Status ==="
docker-compose ps

# 2. Test Zabbix agent custom parameters
echo "=== Testing Custom Parameters ==="
docker exec zabbix-agent zabbix_agentd -t docker.container.count
docker exec zabbix-agent zabbix_agentd -t docker.container.running
docker exec zabbix-agent zabbix_agentd -t "docker.api.container.cpu[test-web-1]"

# 3. Check web interface accessibility
echo "=== Web Interface Test ==="
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080

# 4. Verify monitoring data collection
echo "=== Monitoring Data Collection ==="
docker exec zabbix-server zabbix_get -s zabbix-agent -k docker.container.count

# 5. List all created containers
echo "=== Created Test Containers ==="
docker ps -a --filter "name=test-"

echo "=== Lab Verification Complete ==="
echo "Access your Zabbix dashboard at: http://YOUR_LAB_IP:8080"
echo "Username: Admin"
echo "Password: zabbix"
Conclusion
Congratulations! You have successfully completed Lab 86: Docker for Monitoring - Integrating Docker with Zabbix.

What You Accomplished
In this comprehensive lab, you have:

• Deployed a complete Zabbix monitoring infrastructure using Docker containers, including Zabbix Server, Web Interface, MySQL database, and Zabbix Agent • Configured Docker container health monitoring by creating custom monitoring scripts and Zabbix agent parameters to track container status, count, and performance metrics • Implemented automated alerting by setting up triggers and actions that respond to container performance issues and system problems • Integrated with Docker's API to gather detailed container metrics
