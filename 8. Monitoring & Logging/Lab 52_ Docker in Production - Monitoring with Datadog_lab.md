Lab 52: Docker in Production - Monitoring with Datadog
Objectives
By the end of this lab, you will be able to:

Set up Datadog monitoring for Docker containers in a production environment
Configure Docker integration with Datadog Agent
Monitor key container metrics including CPU, memory, and network usage
Create custom alerts based on container performance thresholds
Build comprehensive dashboards for visualizing container metrics
Configure automatic log collection and reporting from Docker containers
Implement best practices for container monitoring in production environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, volumes)
Familiarity with Linux command line operations
Basic knowledge of system monitoring concepts
Understanding of YAML configuration files
A Datadog account (free trial available at datadog.com)
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. No need to build your own VM or install Docker - everything is ready to use!

Your lab environment includes:

Ubuntu 20.04 LTS with Docker pre-installed
Docker Compose ready for use
Network access for Datadog integration
Sample applications for monitoring
Lab Tasks
Task 1: Set up Datadog and Configure Docker Integration
Subtask 1.1: Obtain Datadog API Key
Access your Datadog account

Navigate to https://app.datadoghq.com
Log in with your credentials or create a free trial account
Generate API Key

Go to Organization Settings > API Keys
Click New Key
Name it docker-lab-key
Copy the generated API key and save it securely
Subtask 1.2: Install Datadog Agent
Create Datadog Agent configuration directory
sudo mkdir -p /opt/datadog-agent/conf.d
sudo mkdir -p /opt/datadog-agent/logs
Create Docker Compose file for Datadog Agent
cat > docker-compose-datadog.yml << 'EOF'
version: '3.8'

services:
  datadog-agent:
    image: gcr.io/datadoghq/agent:7
    container_name: datadog-agent
    environment:
      - DD_API_KEY=${DD_API_KEY}
      - DD_SITE=datadoghq.com
      - DD_HOSTNAME=docker-lab-host
      - DD_TAGS=env:lab,team:devops
      - DD_APM_ENABLED=true
      - DD_LOGS_ENABLED=true
      - DD_LOGS_CONFIG_CONTAINER_COLLECT_ALL=true
      - DD_CONTAINER_EXCLUDE_LOGS="name:datadog-agent"
      - DD_DOCKER_LABELS_AS_TAGS='{"my.custom.label.team":"team","my.custom.label.version":"version"}'
      - DD_COLLECT_KUBERNETES_EVENTS=false
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /proc/:/host/proc/:ro
      - /opt/datadog-agent/conf.d:/conf.d:ro
      - /sys/fs/cgroup/:/host/sys/fs/cgroup:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /opt/datadog-agent/logs:/opt/datadog-agent/logs:rw
    ports:
      - "8125:8125/udp"
      - "8126:8126"
    restart: unless-stopped
    privileged: true
    pid: host
EOF
Set your Datadog API key as environment variable
export DD_API_KEY="your-datadog-api-key-here"
echo "export DD_API_KEY=\"your-datadog-api-key-here\"" >> ~/.bashrc
Start Datadog Agent
docker-compose -f docker-compose-datadog.yml up -d
Verify Datadog Agent is running
docker logs datadog-agent
docker ps | grep datadog
Subtask 1.3: Configure Docker Integration
Create Docker integration configuration
sudo mkdir -p /opt/datadog-agent/conf.d/docker.d
Create Docker configuration file
sudo cat > /opt/datadog-agent/conf.d/docker.d/conf.yaml << 'EOF'
init_config:

instances:
  - url: "unix://var/run/docker.sock"
    collect_events: true
    collect_container_size: true
    collect_volume_count: true
    collect_images_stats: true
    collect_image_size: true
    collect_disk_stats: true
    collect_exit_codes: true
    tags:
      - "docker_integration:enabled"
      - "environment:lab"
EOF
Restart Datadog Agent to apply configuration
docker-compose -f docker-compose-datadog.yml restart
Task 2: Monitor Container Metrics in Datadog
Subtask 2.1: Deploy Sample Applications for Monitoring
Create a sample web application
cat > app.py << 'EOF'
from flask import Flask, jsonify
import psutil
import time
import random
import threading

app = Flask(__name__)

# Simulate some background work
def background_work():
    while True:
        # Simulate CPU usage
        for i in range(random.randint(1000, 10000)):
            _ = i ** 2
        time.sleep(random.randint(1, 5))

# Start background thread
threading.Thread(target=background_work, daemon=True).start()

@app.route('/')
def home():
    return jsonify({
        'message': 'Hello from monitored container!',
        'cpu_percent': psutil.cpu_percent(),
        'memory_percent': psutil.virtual_memory().percent
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

@app.route('/load')
def create_load():
    # Create some CPU load
    for i in range(100000):
        _ = i ** 3
    return jsonify({'message': 'Load created'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
EOF
Create Dockerfile for the application
cat > Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

RUN pip install flask psutil

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
Create Docker Compose file for sample applications
cat > docker-compose-apps.yml << 'EOF'
version: '3.8'

services:
  web-app-1:
    build: .
    container_name: web-app-1
    ports:
      - "5001:5000"
    labels:
      - "my.custom.label.team=frontend"
      - "my.custom.label.version=1.0"
      - "com.datadoghq.ad.logs=[{\"source\": \"python\", \"service\": \"web-app\"}]"
    environment:
      - APP_NAME=web-app-1
    restart: unless-stopped

  web-app-2:
    build: .
    container_name: web-app-2
    ports:
      - "5002:5000"
    labels:
      - "my.custom.label.team=backend"
      - "my.custom.label.version=1.1"
      - "com.datadoghq.ad.logs=[{\"source\": \"python\", \"service\": \"web-app\"}]"
    environment:
      - APP_NAME=web-app-2
    restart: unless-stopped

  nginx-proxy:
    image: nginx:alpine
    container_name: nginx-proxy
    ports:
      - "8080:80"
    labels:
      - "my.custom.label.team=infrastructure"
      - "my.custom.label.version=1.0"
      - "com.datadoghq.ad.logs=[{\"source\": \"nginx\", \"service\": \"nginx-proxy\"}]"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web-app-1
      - web-app-2
    restart: unless-stopped

  redis-cache:
    image: redis:alpine
    container_name: redis-cache
    ports:
      - "6379:6379"
    labels:
      - "my.custom.label.team=database"
      - "my.custom.label.version=6.2"
      - "com.datadoghq.ad.logs=[{\"source\": \"redis\", \"service\": \"redis-cache\"}]"
    restart: unless-stopped
EOF
Create Nginx configuration
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream backend {
        server web-app-1:5000;
        server web-app-2:5000;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
EOF
Build and start the applications
docker-compose -f docker-compose-apps.yml up -d --build
Verify applications are running
docker ps
curl http://localhost:5001
curl http://localhost:5002
curl http://localhost:8080
Subtask 2.2: Generate Load for Monitoring
Create a load generation script
cat > generate_load.sh << 'EOF'
#!/bin/bash

echo "Generating load on applications..."

# Function to generate HTTP requests
generate_requests() {
    local url=$1
    local name=$2
    
    for i in {1..100}; do
        curl -s "$url" > /dev/null
        curl -s "$url/load" > /dev/null
        sleep 0.1
    done
    echo "Completed requests for $name"
}

# Generate load on both applications
generate_requests "http://localhost:5001" "web-app-1" &
generate_requests "http://localhost:5002" "web-app-2" &
generate_requests "http://localhost:8080" "nginx-proxy" &

wait
echo "Load generation completed"
EOF

chmod +x generate_load.sh
Run load generation
./generate_load.sh
Subtask 2.3: Verify Metrics in Datadog
Check Datadog Agent status
docker exec datadog-agent agent status
Access Datadog Dashboard

Go to https://app.datadoghq.com
Navigate to Infrastructure > Containers
Verify your containers are visible
View Docker metrics

Go to Metrics > Explorer
Search for metrics starting with docker.
Common metrics to check:
docker.cpu.usage
docker.mem.usage
docker.net.bytes_rcvd
docker.net.bytes_sent
Task 3: Configure Alerts Based on Container Performance
Subtask 3.1: Create CPU Usage Alert
Navigate to Monitors in Datadog

Go to Monitors > New Monitor
Select Metric
Configure CPU Alert

Metric: docker.cpu.usage
From: *
Alert Threshold: 80 (80% CPU usage)
Warning Threshold: 60 (60% CPU usage)
Evaluation Window: 5 minutes
Set Alert Message

{{#is_alert}}
🚨 HIGH CPU USAGE DETECTED 🚨
Container: {{container_name.name}}
Current CPU Usage: {{value}}%
{{/is_alert}}

{{#is_warning}}
⚠️ WARNING: Elevated CPU Usage
Container: {{container_name.name}}
Current CPU Usage: {{value}}%
{{/is_warning}}

{{#is_recovery}}
✅ CPU Usage Normalized
Container: {{container_name.name}}
Current CPU Usage: {{value}}%
{{/is_recovery}}
Save the monitor with name: Docker Container - High CPU Usage
Subtask 3.2: Create Memory Usage Alert
Create new Metric Monitor

Metric: docker.mem.usage
From: *
Alert Threshold: 500000000 (500MB)
Warning Threshold: 300000000 (300MB)
Set Memory Alert Message

{{#is_alert}}
🚨 HIGH MEMORY USAGE DETECTED 🚨
Container: {{container_name.name}}
Current Memory Usage: {{value}} bytes
{{/is_alert}}

{{#is_warning}}
⚠️ WARNING: Elevated Memory Usage
Container: {{container_name.name}}
Current Memory Usage: {{value}} bytes
{{/is_warning}}
Save the monitor with name: Docker Container - High Memory Usage
Subtask 3.3: Create Network Traffic Alert
Create new Metric Monitor

Metric: docker.net.bytes_rcvd
From: *
Alert Threshold: 10000000 (10MB)
Warning Threshold: 5000000 (5MB)
Evaluation Window: 1 minute
Set Network Alert Message

{{#is_alert}}
🚨 HIGH NETWORK TRAFFIC DETECTED 🚨
Container: {{container_name.name}}
Bytes Received: {{value}} bytes/sec
{{/is_alert}}
Save the monitor with name: Docker Container - High Network Traffic
Subtask 3.4: Test Alerts
Generate high CPU load
docker exec web-app-1 python -c "
import time
import threading

def cpu_load():
    while True:
        for i in range(1000000):
            _ = i ** 2

for _ in range(4):
    threading.Thread(target=cpu_load).start()

time.sleep(300)  # Run for 5 minutes
"
Monitor alerts in Datadog
Go to Monitors > Manage Monitors
Watch for alert notifications
Task 4: Create Dashboards for Visualizing Container Metrics
Subtask 4.1: Create Main Container Dashboard
Create New Dashboard

Go to Dashboards > New Dashboard
Name: Docker Container Monitoring Lab
Description: Comprehensive monitoring dashboard for Docker containers
Add Container Overview Widget

Click Add Widget > Timeseries
Title: Container CPU Usage
Metric: docker.cpu.usage
From: *
Group by: container_name
Add Memory Usage Widget

Click Add Widget > Timeseries
Title: Container Memory Usage
Metric: docker.mem.usage
From: *
Group by: container_name
Add Network Traffic Widget

Click Add Widget > Timeseries
Title: Network Traffic (Bytes Received)
Metric: docker.net.bytes_rcvd
From: *
Group by: container_name
Subtask 4.2: Add Container Count and Status Widgets
Add Container Count Widget

Click Add Widget > Query Value
Title: Total Running Containers
Metric: docker.containers.running
Aggregation: sum
Add Container Status Widget

Click Add Widget > Table
Title: Container Status Overview
Metrics:
docker.containers.running
docker.containers.stopped
Group by: host
Subtask 4.3: Create Performance Heatmap
Add Heatmap Widget
Click Add Widget > Heatmap
Title: CPU Usage Heatmap
Metric: docker.cpu.usage
From: *
Group by: container_name
Subtask 4.4: Add Custom Metrics Dashboard
Create template variables

Click Dashboard Settings > Template Variables
Add variable: container with source tag:container_name
Update widgets to use template variable

Edit each widget
Change From to container_name:$container
Task 5: Set up Automatic Reporting of Container Logs
Subtask 5.1: Configure Log Collection
Verify log collection is enabled
docker exec datadog-agent agent status | grep -A 10 "Logs Agent"
Create custom log configuration
sudo mkdir -p /opt/datadog-agent/conf.d/docker.d
sudo cat > /opt/datadog-agent/conf.d/docker.d/logs.yaml << 'EOF'
logs:
  - type: docker
    image: "*"
    service: docker-containers
    source: docker
    sourcecategory: docker
    tags:
      - env:lab
      - team:devops
EOF
Restart Datadog Agent
docker-compose -f docker-compose-datadog.yml restart
Subtask 5.2: Configure Application-Specific Logging
Update application to generate structured logs
cat > app_with_logging.py << 'EOF'
from flask import Flask, jsonify
import psutil
import time
import random
import threading
import logging
import json
from datetime import datetime

# Configure structured logging
logging.basicConfig(
    level=logging.INFO,
    format='%(message)s'
)
logger = logging.getLogger(__name__)

app = Flask(__name__)

def log_structured(level, message, **kwargs):
    log_entry = {
        'timestamp': datetime.utcnow().isoformat(),
        'level': level,
        'message': message,
        'service': 'web-app',
        'version': '1.0',
        **kwargs
    }
    logger.info(json.dumps(log_entry))

def background_work():
    while True:
        cpu_usage = psutil.cpu_percent()
        memory_usage = psutil.virtual_memory().percent
        
        log_structured('INFO', 'Background metrics collected', 
                      cpu_percent=cpu_usage, 
                      memory_percent=memory_usage)
        
        # Simulate CPU usage
        for i in range(random.randint(1000, 10000)):
            _ = i ** 2
        time.sleep(random.randint(1, 5))

# Start background thread
threading.Thread(target=background_work, daemon=True).start()

@app.route('/')
def home():
    log_structured('INFO', 'Home endpoint accessed')
    return jsonify({
        'message': 'Hello from monitored container!',
        'cpu_percent': psutil.cpu_percent(),
        'memory_percent': psutil.virtual_memory().percent
    })

@app.route('/health')
def health():
    log_structured('INFO', 'Health check performed', status='healthy')
    return jsonify({'status': 'healthy'})

@app.route('/load')
def create_load():
    log_structured('WARNING', 'Load generation started')
    # Create some CPU load
    for i in range(100000):
        _ = i ** 3
    log_structured('INFO', 'Load generation completed')
    return jsonify({'message': 'Load created'})

@app.route('/error')
def create_error():
    log_structured('ERROR', 'Intentional error triggered', error_type='test_error')
    return jsonify({'error': 'This is a test error'}), 500

if __name__ == '__main__':
    log_structured('INFO', 'Application starting')
    app.run(host='0.0.0.0', port=5000)
EOF
Update Dockerfile
cat > Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

RUN pip install flask psutil

COPY app_with_logging.py app.py

EXPOSE 5000

CMD ["python", "app.py"]
EOF
Rebuild and restart applications
docker-compose -f docker-compose-apps.yml down
docker-compose -f docker-compose-apps.yml up -d --build
Subtask 5.3: View Logs in Datadog
Generate log entries
# Generate various types of logs
curl http://localhost:5001/
curl http://localhost:5001/health
curl http://localhost:5001/load
curl http://localhost:5001/error
Access Logs in Datadog
Go to Logs in Datadog
Filter by source:docker or service:web-app
Observe structured log entries
Subtask 5.4: Create Log-Based Alerts
Create Log Monitor

Go to Monitors > New Monitor
Select Logs
Configure Error Log Alert

Search Query: source:docker level:ERROR
Alert Threshold: > 5 errors in 5 minutes
Message:
🚨 HIGH ERROR RATE DETECTED 🚨
Service: {{service.name}}
Error Count: {{value}} errors in 5 minutes
Save monitor with name: Docker Container - High Error Rate

Subtask 5.5: Set up Log Archives and Retention
Configure Log Retention

Go to Logs > Configuration > Indexes
Create index for container logs with 30-day retention
Set up Log Parsing

Go to Logs > Configuration > Pipelines
Create pipeline for Docker logs
Add processors for JSON parsing and attribute extraction
Verification and Testing
Verify Complete Setup
Check all containers are running
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
Verify Datadog Agent connectivity
docker exec datadog-agent agent status
Test application endpoints
echo "Testing web applications..."
curl -s http://localhost:5001/health | jq .
curl -s http://localhost:5002/health | jq .
curl -s http://localhost:8080/health | jq .
Generate comprehensive test load
cat > comprehensive_test.sh << 'EOF'
#!/bin/bash

echo "Running comprehensive monitoring test..."

# Test all endpoints
endpoints=("http://localhost:5001" "http://localhost:5002" "http://localhost:8080")

for endpoint in "${endpoints[@]}"; do
    echo "Testing $endpoint"
    for i in {1..20}; do
        curl -s "$endpoint/" > /dev/null
        curl -s "$endpoint/health" > /dev/null
        curl -s "$endpoint/load" > /dev/null
        curl -s "$endpoint/error" > /dev/null
        sleep 0.5
    done
done

echo "Test completed. Check Datadog for metrics and alerts."
EOF

chmod +x comprehensive_test.sh
./comprehensive_test.sh
Troubleshooting Common Issues
Datadog Agent not reporting metrics
# Check agent logs
docker logs datadog-agent | tail -50

# Verify API key
docker exec datadog-agent agent config | grep api_key

# Restart agent
docker-compose -f docker-compose-datadog.yml restart
Containers not visible in Datadog
# Check Docker socket permissions
ls -la /var/run/docker.sock

# Verify agent has access to Docker
docker exec datadog-agent docker ps
Logs not appearing
# Check log agent status
docker exec datadog-agent agent status | grep -A 20 "Logs Agent"

# Verify log configuration
docker exec datadog-agent agent configcheck
Conclusion
Congratulations! You have successfully completed Lab 52: Docker in Production - Monitoring with Datadog.

What You Accomplished
In this lab, you have:

Set up comprehensive Docker monitoring by installing and configuring the Datadog Agent with proper Docker integration
Deployed and monitored real applications including web services, load balancers, and databases
Created intelligent alerting based on CPU, memory, and network performance thresholds
Built professional dashboards with multiple visualization types for container metrics
Implemented structured logging with automatic collection and analysis capabilities
Why This Matters
Container monitoring is crucial for production environments because:

Proactive Issue Detection: Alerts help you identify problems before they impact users
Performance Optimization: Metrics help you optimize resource allocation and scaling decisions
Troubleshooting: Logs and metrics provide essential data for debugging issues
Capacity Planning: Historical data helps predict future resource needs
Compliance: Monitoring helps meet SLA requirements and audit needs
Next Steps
To further enhance your Docker monitoring skills:

Explore custom metrics by instrumenting your applications with Datadog libraries
Set up distributed tracing to monitor request flows across containers
Implement synthetic monitoring to test application availability
Configure log-based metrics for business intelligence
Explore infrastructure as code approaches for monitoring configuration
This lab has provided you with production-ready monitoring skills that are essential for the Docker Certified Associate (DCA) certification and real-world container operations.
