Lab 84: Docker for Monitoring - Monitoring Docker with Datadog
Lab Objectives
By the end of this lab, you will be able to:

Set up and configure the Datadog Agent to monitor Docker containers
Install and configure Datadog integration for Docker on a Linux host machine
Monitor key container metrics including CPU, memory, disk, and network usage
Create custom Datadog dashboards to visualize container performance metrics
Configure alerting rules for high resource usage and container failure scenarios
Understand best practices for container monitoring in production environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, volumes)
Familiarity with Linux command line operations
Basic knowledge of system monitoring concepts
Understanding of YAML configuration files
A Datadog account (free trial available at datadog.com)
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker pre-installed
Docker Compose pre-installed
Internet connectivity for downloading Datadog Agent
Sample applications ready for monitoring
Task 1: Set up Datadog Agent to Monitor Docker Containers
Subtask 1.1: Obtain Datadog API Key
First, you need to get your Datadog API key to authenticate the agent.

Log into your Datadog account at https://app.datadoghq.com
Navigate to Organization Settings:
Click on your profile icon in the bottom left
Select "Organization Settings"
Get your API Key:
Click on "API Keys" in the left sidebar
Copy your existing API key or create a new one
Save this key securely - you'll need it shortly
Subtask 1.2: Install Datadog Agent Container
The easiest way to monitor Docker containers is to run the Datadog Agent as a container itself.

Create a directory for Datadog configuration:
mkdir -p ~/datadog-lab
cd ~/datadog-lab
Set your API key as an environment variable:
export DD_API_KEY="your-api-key-here"
Replace your-api-key-here with the actual API key you obtained from Datadog.

Run the Datadog Agent container:
docker run -d --name datadog-agent \
  --cgroupns host \
  --pid host \
  -e DD_API_KEY=$DD_API_KEY \
  -e DD_SITE="datadoghq.com" \
  -e DD_CONTAINER_EXCLUDE="name:datadog-agent" \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /proc/:/host/proc/:ro \
  -v /opt/datadog-agent/run:/opt/datadog-agent/run:rw \
  -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
  datadog/agent:latest
Verify the agent is running:
docker ps | grep datadog-agent
You should see the Datadog Agent container running.

Check agent logs to ensure successful connection:
docker logs datadog-agent | tail -20
Look for messages indicating successful connection to Datadog servers.

Subtask 1.3: Verify Agent Status
Check agent status from within the container:
docker exec -it datadog-agent agent status
This command will show you the agent's health and what integrations are active.

Verify in Datadog UI:
Go to your Datadog dashboard
Navigate to "Infrastructure" → "Host Map"
You should see your host appearing in the map
Task 2: Install Datadog Integration for Docker on Host Machine
Subtask 2.1: Configure Docker Integration
The Docker integration is automatically enabled when you run the Datadog Agent with Docker socket access, but let's verify and customize the configuration.

Create a custom configuration directory:
mkdir -p ~/datadog-lab/conf.d/docker.d
Create Docker integration configuration file:
cat > ~/datadog-lab/conf.d/docker.d/conf.yaml << 'EOF'
init_config:

instances:
  - url: "unix://var/run/docker.sock"
    collect_events: true
    collect_container_size: true
    collect_volume_count: true
    collect_images_stats: true
    collect_disk_stats: true
    collect_exit_codes: true
    tags:
      - "environment:lab"
      - "team:monitoring"
EOF
Restart the Datadog Agent with custom configuration:
docker stop datadog-agent
docker rm datadog-agent

docker run -d --name datadog-agent \
  --cgroupns host \
  --pid host \
  -e DD_API_KEY=$DD_API_KEY \
  -e DD_SITE="datadoghq.com" \
  -e DD_CONTAINER_EXCLUDE="name:datadog-agent" \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /proc/:/host/proc/:ro \
  -v /opt/datadog-agent/run:/opt/datadog-agent/run:rw \
  -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
  -v ~/datadog-lab/conf.d:/etc/datadog-agent/conf.d:ro \
  datadog/agent:latest
Subtask 2.2: Deploy Sample Applications for Monitoring
Let's create some sample applications to monitor.

Create a Docker Compose file for sample applications:
cat > ~/datadog-lab/docker-compose.yml << 'EOF'
version: '3.8'

services:
  web-app:
    image: nginx:alpine
    container_name: sample-web-app
    ports:
      - "8080:80"
    labels:
      - "com.datadoghq.ad.check_names=nginx"
      - "com.datadoghq.ad.init_configs=[{}]"
      - "com.datadoghq.ad.instances=[{\"nginx_status_url\":\"http://%%host%%:%%port%%/nginx_status\"}]"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    restart: unless-stopped

  database:
    image: postgres:13-alpine
    container_name: sample-database
    environment:
      POSTGRES_DB: sampledb
      POSTGRES_USER: sampleuser
      POSTGRES_PASSWORD: samplepass
    ports:
      - "5432:5432"
    labels:
      - "com.datadoghq.ad.check_names=postgres"
      - "com.datadoghq.ad.init_configs=[{}]"
      - "com.datadoghq.ad.instances=[{\"host\":\"%%host%%\",\"port\":5432,\"username\":\"sampleuser\",\"password\":\"samplepass\"}]"
    restart: unless-stopped

  redis-cache:
    image: redis:6-alpine
    container_name: sample-redis
    ports:
      - "6379:6379"
    labels:
      - "com.datadoghq.ad.check_names=redisdb"
      - "com.datadoghq.ad.init_configs=[{}]"
      - "com.datadoghq.ad.instances=[{\"host\":\"%%host%%\",\"port\":6379}]"
    restart: unless-stopped

  load-generator:
    image: busybox
    container_name: cpu-load-generator
    command: sh -c "while true; do dd if=/dev/zero of=/dev/null bs=1M count=100; sleep 10; done"
    restart: unless-stopped
EOF
Create nginx configuration for status endpoint:
cat > ~/datadog-lab/nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    server {
        listen 80;
        
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        
        location /nginx_status {
            stub_status on;
            access_log off;
            allow all;
        }
    }
}
EOF
Start the sample applications:
cd ~/datadog-lab
docker-compose up -d
Verify all containers are running:
docker ps
You should see all sample applications running along with the Datadog Agent.

Task 3: Monitor Key Metrics of Running Containers
Subtask 3.1: Verify Metrics Collection
Check if Docker metrics are being collected:
docker exec -it datadog-agent agent status | grep -A 10 "docker"
Generate some load to create interesting metrics:
# Generate web traffic
for i in {1..100}; do
  curl -s http://localhost:8080 > /dev/null
  sleep 1
done &

# Generate database activity
docker exec -it sample-database psql -U sampleuser -d sampledb -c "CREATE TABLE IF NOT EXISTS test_table (id SERIAL PRIMARY KEY, data TEXT);"

for i in {1..50}; do
  docker exec -it sample-database psql -U sampleuser -d sampledb -c "INSERT INTO test_table (data) VALUES ('test data $i');"
  sleep 2
done &
Subtask 3.2: Explore Metrics in Datadog
Access Datadog Metrics Explorer:

Go to your Datadog dashboard
Navigate to "Metrics" → "Explorer"
Key Docker metrics to explore:

CPU Metrics:

docker.cpu.usage
docker.cpu.user
docker.cpu.system
Memory Metrics:

docker.mem.usage
docker.mem.limit
docker.mem.rss
Network Metrics:

docker.net.bytes_rcvd
docker.net.bytes_sent
Disk Metrics:

docker.io.read_bytes
docker.io.write_bytes
Create a simple graph:
In Metrics Explorer, search for docker.cpu.usage
Group by container_name
Set the time range to "Last 1 Hour"
Click "Graph" to visualize
Subtask 3.3: Use Datadog Agent Commands for Local Monitoring
Check what integrations are running:
docker exec -it datadog-agent agent status
List all available metrics:
docker exec -it datadog-agent agent check docker
View real-time metrics:
docker exec -it datadog-agent agent check docker -v
Task 4: Create Custom Datadog Dashboards
Subtask 4.1: Create a Container Overview Dashboard
Navigate to Dashboards in Datadog:

Go to "Dashboards" → "New Dashboard"
Choose "New Dashboard"
Name it "Docker Container Monitoring Lab"
Add CPU Usage Widget:

Click "Add Widget" → "Timeseries"
Metric: docker.cpu.usage
Group by: container_name
Title: "Container CPU Usage"
Click "Save"
Add Memory Usage Widget:

Click "Add Widget" → "Timeseries"
Metric: docker.mem.usage
Group by: container_name
Title: "Container Memory Usage"
Click "Save"
Add Network I/O Widget:

Click "Add Widget" → "Timeseries"
Add two metrics:
docker.net.bytes_rcvd
docker.net.bytes_sent
Group by: container_name
Title: "Container Network I/O"
Click "Save"
Subtask 4.2: Create Container Status Widget
Add Container Count Widget:

Click "Add Widget" → "Query Value"
Metric: docker.containers.running
Aggregation: sum
Title: "Running Containers"
Click "Save"
Add Top List Widget:

Click "Add Widget" → "Top List"
Metric: docker.cpu.usage
Group by: container_name
Title: "Top CPU Consuming Containers"
Limit: 10
Click "Save"
Subtask 4.3: Create Application-Specific Widgets
Add Nginx-specific Widget:

Click "Add Widget" → "Timeseries"
Metric: nginx.net.connections
Filter by: container_name:sample-web-app
Title: "Nginx Connections"
Click "Save"
Add PostgreSQL-specific Widget:

Click "Add Widget" → "Timeseries"
Metric: postgresql.connections
Filter by: container_name:sample-database
Title: "PostgreSQL Connections"
Click "Save"
Save your dashboard by clicking "Save" in the top right corner.

Task 5: Set up Alerting for High Resource Usage
Subtask 5.1: Create CPU Usage Alert
Navigate to Monitors:

Go to "Monitors" → "New Monitor"
Select "Metric"
Configure the CPU Alert:

Detection Method: Threshold
Metric: docker.cpu.usage
From: container_name:*
Alert Threshold: > 80 (80% CPU usage)
Warning Threshold: > 60 (60% CPU usage)
Evaluation Window: 5 minutes
Set Alert Message:

**High CPU Usage Detected**

Container {{container_name.name}} is using {{value}}% CPU.

**Troubleshooting Steps:**
1. Check container logs: `docker logs {{container_name.name}}`
2. Inspect running processes: `docker exec {{container_name.name}} top`
3. Consider scaling or resource limits

@your-email@example.com
Configure Recipients:

Add your email address
Set notification preferences
Name the Monitor: "Docker Container High CPU Usage"

Save the Monitor

Subtask 5.2: Create Memory Usage Alert
Create another Monitor:

Go to "Monitors" → "New Monitor"
Select "Metric"
Configure Memory Alert:

Metric: docker.mem.usage
From: container_name:*
Alert Threshold: > 500000000 (500MB)
Warning Threshold: > 300000000 (300MB)
Set Alert Message:

**High Memory Usage Detected**

Container {{container_name.name}} is using {{value}} bytes of memory.

**Actions to Take:**
1. Check memory usage: `docker stats {{container_name.name}}`
2. Review application logs for memory leaks
3. Consider increasing memory limits or optimizing application

@your-email@example.com
Save as: "Docker Container High Memory Usage"
Subtask 5.3: Create Container Failure Alert
Create a Service Check Monitor:

Go to "Monitors" → "New Monitor"
Select "Service Check"
Configure Container Health Alert:

Service Check: docker.service_up
Host: *
Alert Threshold: Select "CRITICAL"
Warning Threshold: Select "WARNING"
Set Alert Message:

**Container Service Down**

Docker service check failed on host {{host.name}}.

**Immediate Actions:**
1. Check Docker daemon: `sudo systemctl status docker`
2. Restart Docker if needed: `sudo systemctl restart docker`
3. Check container status: `docker ps -a`
4. Review Docker logs: `sudo journalctl -u docker`

@your-email@example.com
Save as: "Docker Service Health Check"
Subtask 5.4: Test Alert Functionality
Trigger a CPU alert by creating high load:
docker run -d --name cpu-stress --rm alpine sh -c "while true; do :; done"
Wait 5-10 minutes and check if you receive an alert notification.

Stop the stress container:

docker stop cpu-stress
Verify alert recovery - you should receive a recovery notification.
Task 6: Advanced Monitoring Configuration
Subtask 6.1: Configure Log Collection
Enable log collection in Datadog Agent:
docker stop datadog-agent
docker rm datadog-agent

docker run -d --name datadog-agent \
  --cgroupns host \
  --pid host \
  -e DD_API_KEY=$DD_API_KEY \
  -e DD_SITE="datadoghq.com" \
  -e DD_LOGS_ENABLED=true \
  -e DD_CONTAINER_EXCLUDE="name:datadog-agent" \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  -v /proc/:/host/proc/:ro \
  -v /opt/datadog-agent/run:/opt/datadog-agent/run:rw \
  -v /sys/fs/cgroup/:/host/sys/fs/cgroup:ro \
  -v /var/lib/docker/containers:/var/lib/docker/containers:ro \
  -v ~/datadog-lab/conf.d:/etc/datadog-agent/conf.d:ro \
  datadog/agent:latest
Add log collection labels to your containers:
# Stop existing containers
docker-compose down

# Update docker-compose.yml to include log labels
cat > ~/datadog-lab/docker-compose.yml << 'EOF'
version: '3.8'

services:
  web-app:
    image: nginx:alpine
    container_name: sample-web-app
    ports:
      - "8080:80"
    labels:
      - "com.datadoghq.ad.check_names=nginx"
      - "com.datadoghq.ad.init_configs=[{}]"
      - "com.datadoghq.ad.instances=[{\"nginx_status_url\":\"http://%%host%%:%%port%%/nginx_status\"}]"
      - "com.datadoghq.ad.logs=[{\"source\":\"nginx\",\"service\":\"web-app\"}]"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    restart: unless-stopped

  database:
    image: postgres:13-alpine
    container_name: sample-database
    environment:
      POSTGRES_DB: sampledb
      POSTGRES_USER: sampleuser
      POSTGRES_PASSWORD: samplepass
    ports:
      - "5432:5432"
    labels:
      - "com.datadoghq.ad.check_names=postgres"
      - "com.datadoghq.ad.init_configs=[{}]"
      - "com.datadoghq.ad.instances=[{\"host\":\"%%host%%\",\"port\":5432,\"username\":\"sampleuser\",\"password\":\"samplepass\"}]"
      - "com.datadoghq.ad.logs=[{\"source\":\"postgresql\",\"service\":\"database\"}]"
    restart: unless-stopped

  redis-cache:
    image: redis:6-alpine
    container_name: sample-redis
    ports:
      - "6379:6379"
    labels:
      - "com.datadoghq.ad.check_names=redisdb"
      - "com.datadoghq.ad.init_configs=[{}]"
      - "com.datadoghq.ad.instances=[{\"host\":\"%%host%%\",\"port\":6379}]"
      - "com.datadoghq.ad.logs=[{\"source\":\"redis\",\"service\":\"cache\"}]"
    restart: unless-stopped

  load-generator:
    image: busybox
    container_name: cpu-load-generator
    command: sh -c "while true; do echo 'Load test running...'; dd if=/dev/zero of=/dev/null bs=1M count=100; sleep 10; done"
    labels:
      - "com.datadoghq.ad.logs=[{\"source\":\"busybox\",\"service\":\"load-generator\"}]"
    restart: unless-stopped
EOF
Restart containers with log collection:
docker-compose up -d
Subtask 6.2: Verify Complete Monitoring Setup
Check agent status:
docker exec -it datadog-agent agent status
Verify all integrations are working:
docker exec -it datadog-agent agent check docker
docker exec -it datadog-agent agent check nginx
docker exec -it datadog-agent agent check postgres
docker exec -it datadog-agent agent check redisdb
Check log collection:
Go to Datadog → "Logs"
You should see logs from your containers appearing
Troubleshooting Common Issues
Issue 1: Agent Not Connecting to Datadog
Symptoms: Agent logs show connection errors

Solutions:

Verify API key is correct
Check internet connectivity
Ensure DD_SITE is set correctly for your region
# Check agent logs
docker logs datadog-agent | grep -i error

# Verify API key
docker exec -it datadog-agent agent config | grep api_key
Issue 2: Docker Metrics Not Appearing
Symptoms: No Docker metrics in Datadog dashboard

Solutions:

Ensure Docker socket is mounted correctly
Check agent has permission to access Docker socket
Verify Docker integration is enabled
# Check Docker socket access
docker exec -it datadog-agent ls -la /var/run/docker.sock

# Test Docker integration
docker exec -it datadog-agent agent check docker -v
Issue 3: High Resource Usage by Datadog Agent
Symptoms: Datadog Agent consuming too much CPU/memory

Solutions:

Exclude unnecessary containers from monitoring
Adjust collection intervals
Limit metrics collection
# Add exclusion rules
docker run -d --name datadog-agent \
  -e DD_CONTAINER_EXCLUDE="name:datadog-agent name:some-other-container" \
  # ... other parameters
Lab Validation
Validation Checklist
Verify you have completed all tasks by checking:

 Datadog Agent is running and connected
 Docker integration is active and collecting metrics
 Sample applications are running and being monitored
 Custom dashboard shows container metrics
 CPU and memory alerts are configured
 Container failure alert is set up
 Log collection is working (optional)
 All monitors are in "OK" state
Validation Commands
Run these commands to verify your setup:

# Check all containers are running
docker ps

# Verify Datadog Agent status
docker exec -it datadog-agent agent status | head -20

# Check integration status
docker exec -it datadog-agent agent check docker | head -10

# List configured monitors (requires Datadog CLI or API)
curl -X GET "https://api.datadoghq.com/api/v1/monitor" \
  -H "DD-API-KEY: $DD_API_KEY" \
  -H "DD-APPLICATION-KEY: $DD_APP_KEY" | jq '.[] | .name'
Cleanup Instructions
When you're finished with the lab, clean up resources:

# Stop and remove all containers
cd ~/datadog-lab
docker-compose down

# Remove Datadog Agent
docker stop datadog-agent
docker rm datadog-agent

# Remove any additional test containers
docker stop cpu-stress 2>/dev/null || true

# Clean up volumes (optional)
docker volume prune -f

# Remove lab directory
rm -rf ~/datadog-lab
Conclusion
Congratulations! You have successfully completed Lab 84: Docker for Monitoring with Datadog. In this comprehensive lab, you have:

Key Accomplishments:

Successfully deployed and configured the Datadog Agent to monitor Docker containers in a production-like environment
Integrated Docker monitoring with Datadog's powerful observability platform, enabling comprehensive container visibility
Monitored critical metrics including CPU usage, memory consumption, disk I/O, and network traffic across multiple containerized applications
Created custom dashboards that provide real-time visualization of container performance and health metrics
Implemented proactive alerting for high resource usage scenarios and container failures, enabling rapid incident response
Configured advanced features like log collection and application-specific monitoring for Nginx, PostgreSQL, and Redis
Why This Matters:

Container monitoring is crucial in modern DevOps and cloud-native environments because:

Operational Visibility: Provides deep insights into container performance and resource utilization
Proactive Issue Detection: Enables early identification of performance bottlenecks and resource constraints
Scalability Planning: Helps make informed decisions about resource allocation and scaling strategies
Compliance and SLA Management: Ensures applications meet performance requirements and service level agreements
Cost Optimization: Identifies over-provisioned resources and optimization opportunities
Real-World Applications:

The skills you've developed in this lab directly apply to:

Production Container Orchestration: Monitoring Kubernetes clusters, Docker Swarm, or other container platforms
Microservices Architecture: Tracking performance across distributed application components
DevOps Pipeline Integration: Incorporating monitoring into CI/CD workflows for continuous observability
Incident Response: Using metrics and alerts to quickly diagnose and resolve production issues
Capacity Planning: Making data-driven decisions about infrastructure scaling and resource allocation
Next Steps:

To further enhance your container monitoring expertise, consider:

Exploring Datadog's APM (Application Performance Monitoring) features
Learning about distributed tracing for microservices
Implementing custom metrics and business KPIs
Integrating with container orchestration platforms like Kubernetes
Exploring infrastructure as code approaches for monitoring configuration
This lab has provided you with practical, hands-on experience that directly translates to real-world container monitoring scenarios, making you better prepared for Docker Certified Associate (DCA) certification and professional DevOps roles.
