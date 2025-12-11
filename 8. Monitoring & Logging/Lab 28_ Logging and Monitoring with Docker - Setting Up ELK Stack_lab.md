Lab 28: Logging and Monitoring with Docker - Setting Up ELK Stack
Lab Objectives
By the end of this lab, you will be able to:

• Set up and configure the complete ELK stack (Elasticsearch, Logstash, and Kibana) using Docker Compose • Configure Logstash to receive and parse logs from Docker containers • Create visualizations and dashboards in Kibana to monitor log data • Integrate multiple Docker containers to send logs to the ELK stack • Manage the entire logging infrastructure using Docker Compose • Understand best practices for centralized logging in containerized environments

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker containers and Docker Compose • Familiarity with YAML configuration files • Basic knowledge of Linux command line operations • Understanding of logging concepts and log formats • At least 8GB of RAM available on your system (ELK stack is resource-intensive)

Lab Environment
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install Docker. Your machine comes with Docker and Docker Compose pre-installed.

Task 1: Set Up the ELK Stack with Docker Compose
Subtask 1.1: Create Project Directory Structure
First, let's create a proper directory structure for our ELK stack project.

# Create main project directory
mkdir elk-stack-lab
cd elk-stack-lab

# Create subdirectories for configuration files
mkdir -p config/elasticsearch
mkdir -p config/logstash
mkdir -p config/kibana
mkdir -p logs
mkdir -p data/elasticsearch
Subtask 1.2: Configure Elasticsearch
Create the Elasticsearch configuration file:

nano config/elasticsearch/elasticsearch.yml
Add the following configuration:

cluster.name: "docker-cluster"
network.host: 0.0.0.0
discovery.type: single-node
xpack.security.enabled: false
xpack.monitoring.collection.enabled: true
Subtask 1.3: Configure Kibana
Create the Kibana configuration file:

nano config/kibana/kibana.yml
Add the following configuration:

server.name: kibana
server.host: 0.0.0.0
elasticsearch.hosts: ["http://elasticsearch:9200"]
monitoring.ui.container.elasticsearch.enabled: true
Subtask 1.4: Configure Logstash
Create the Logstash pipeline configuration:

nano config/logstash/pipeline.conf
Add the following configuration:

input {
  beats {
    port => 5044
  }
  
  gelf {
    port => 12201
  }
  
  tcp {
    port => 5000
    codec => json_lines
  }
}

filter {
  if [docker] {
    if [docker][container][name] {
      mutate {
        add_field => { "container_name" => "%{[docker][container][name]}" }
      }
    }
  }
  
  # Parse common log formats
  if [message] =~ /^\d{4}-\d{2}-\d{2}/ {
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:log_level} %{GREEDYDATA:log_message}" }
    }
    
    date {
      match => [ "timestamp", "yyyy-MM-dd HH:mm:ss.SSS" ]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "docker-logs-%{+YYYY.MM.dd}"
  }
  
  stdout {
    codec => rubydebug
  }
}
Create Logstash settings file:

nano config/logstash/logstash.yml
Add the following configuration:

http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: ["http://elasticsearch:9200"]
Subtask 1.5: Create Docker Compose File
Create the main Docker Compose file for the ELK stack:

nano docker-compose.yml
Add the following configuration:

version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms1g -Xmx1g"
      - xpack.security.enabled=false
      - xpack.security.enrollment.enabled=false
    volumes:
      - ./config/elasticsearch/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
      - ./data/elasticsearch:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"
      - "9300:9300"
    networks:
      - elk-network
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash
    volumes:
      - ./config/logstash/pipeline.conf:/usr/share/logstash/pipeline/logstash.conf:ro
      - ./config/logstash/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
    ports:
      - "5044:5044"
      - "5000:5000/tcp"
      - "5000:5000/udp"
      - "12201:12201/udp"
    networks:
      - elk-network
    depends_on:
      elasticsearch:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9600 || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana
    volumes:
      - ./config/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml:ro
    ports:
      - "5601:5601"
    networks:
      - elk-network
    depends_on:
      logstash:
        condition: service_healthy
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5601/api/status || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

networks:
  elk-network:
    driver: bridge

volumes:
  elasticsearch-data:
    driver: local
Subtask 1.6: Start the ELK Stack
Launch the ELK stack using Docker Compose:

# Start the ELK stack in detached mode
docker-compose up -d

# Monitor the startup process
docker-compose logs -f
Wait for all services to be healthy. This may take 2-3 minutes.

Subtask 1.7: Verify ELK Stack Installation
Check if all services are running:

# Check container status
docker-compose ps

# Test Elasticsearch
curl -X GET "localhost:9200/_cluster/health?pretty"

# Test Logstash
curl -X GET "localhost:9600"

# Access Kibana (should show Kibana interface)
echo "Open your browser and navigate to: http://localhost:5601"
Task 2: Configure Logstash to Receive Docker Container Logs
Subtask 2.1: Create a Sample Application with Logging
Create a simple web application that generates logs:

mkdir sample-app
cd sample-app
Create a simple Python Flask application:

nano app.py
Add the following code:

from flask import Flask, jsonify
import logging
import json
import time
import random
from datetime import datetime

app = Flask(__name__)

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(message)s',
    handlers=[
        logging.StreamHandler()
    ]
)

logger = logging.getLogger(__name__)

@app.route('/')
def home():
    logger.info("Home page accessed")
    return jsonify({
        "message": "Welcome to Sample App",
        "timestamp": datetime.now().isoformat(),
        "status": "success"
    })

@app.route('/api/data')
def get_data():
    # Simulate different log levels
    log_level = random.choice(['info', 'warning', 'error'])
    
    if log_level == 'info':
        logger.info("Data endpoint accessed successfully")
    elif log_level == 'warning':
        logger.warning("Data endpoint accessed with potential issues")
    else:
        logger.error("Data endpoint encountered an error")
    
    return jsonify({
        "data": [1, 2, 3, 4, 5],
        "timestamp": datetime.now().isoformat(),
        "log_level": log_level
    })

@app.route('/health')
def health():
    logger.info("Health check performed")
    return jsonify({"status": "healthy", "timestamp": datetime.now().isoformat()})

if __name__ == '__main__':
    logger.info("Starting Sample Application")
    app.run(host='0.0.0.0', port=5000, debug=True)
Create a Dockerfile for the sample application:

nano Dockerfile
Add the following content:

FROM python:3.9-slim

WORKDIR /app

RUN pip install flask

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
Subtask 2.2: Configure Docker Logging Driver
Create a Docker Compose file for the sample application with logging configuration:

nano docker-compose-app.yml
Add the following configuration:

version: '3.8'

services:
  sample-app:
    build: .
    container_name: sample-app
    ports:
      - "8080:5000"
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
        tag: "sample-app"
    networks:
      - elk-stack-lab_elk-network
    depends_on:
      - logstash-forwarder

  nginx-app:
    image: nginx:alpine
    container_name: nginx-sample
    ports:
      - "8081:80"
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
        tag: "nginx-app"
    networks:
      - elk-stack-lab_elk-network
    depends_on:
      - logstash-forwarder

  logstash-forwarder:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash-forwarder
    command: >
      bash -c "
        echo 'input { 
          gelf { port => 12201 } 
        } 
        output { 
          tcp { 
            host => \"logstash\" 
            port => 5000 
            codec => json_lines 
          } 
        }' > /usr/share/logstash/pipeline/logstash.conf &&
        /usr/local/bin/docker-entrypoint
      "
    ports:
      - "12201:12201/udp"
    networks:
      - elk-stack-lab_elk-network

networks:
  elk-stack-lab_elk-network:
    external: true
Subtask 2.3: Start Sample Applications
Build and start the sample applications:

# Build and start the sample applications
docker-compose -f docker-compose-app.yml up -d

# Check if applications are running
docker-compose -f docker-compose-app.yml ps
Subtask 2.4: Generate Log Data
Generate some log data by accessing the applications:

# Generate logs from the sample Flask app
for i in {1..10}; do
  curl http://localhost:8080/
  curl http://localhost:8080/api/data
  curl http://localhost:8080/health
  sleep 2
done

# Generate logs from Nginx
for i in {1..5}; do
  curl http://localhost:8081/
  sleep 1
done
Task 3: Use Kibana to Visualize Log Data
Subtask 3.1: Access Kibana Dashboard
Open your web browser and navigate to Kibana:

echo "Open your browser and go to: http://localhost:5601"
Subtask 3.2: Create Index Pattern
In Kibana, click on Management in the left sidebar
Click on Stack Management
Under Kibana, click on Index Patterns
Click Create index pattern
Enter docker-logs-* as the index pattern
Click Next step
Select @timestamp as the time field
Click Create index pattern
Subtask 3.3: Explore Logs in Discover
Click on Discover in the left sidebar
Select the docker-logs-* index pattern
Set the time range to Last 15 minutes
Explore the log entries and available fields
Subtask 3.4: Create Basic Visualizations
Create a simple visualization:

Click on Visualize Library in the left sidebar
Click Create visualization
Select Vertical bar chart
Choose the docker-logs-* index pattern
Configure the chart:
Y-axis: Count
X-axis: Date Histogram on @timestamp
Click Save and name it "Log Count Over Time"
Subtask 3.5: Create a Dashboard
Click on Dashboard in the left sidebar
Click Create dashboard
Click Add an existing
Select the "Log Count Over Time" visualization
Click Save and name the dashboard "Docker Logs Dashboard"
Task 4: Integrate Additional Docker Containers
Subtask 4.1: Create a Multi-Service Application
Create a more complex application setup:

cd ..
mkdir multi-service-app
cd multi-service-app
Create a Docker Compose file for multiple services:

nano docker-compose-multi.yml
Add the following configuration:

version: '3.8'

services:
  web-frontend:
    image: nginx:alpine
    container_name: web-frontend
    ports:
      - "8082:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
        tag: "web-frontend"
    networks:
      - elk-stack-lab_elk-network
    depends_on:
      - api-backend

  api-backend:
    image: python:3.9-slim
    container_name: api-backend
    command: >
      bash -c "
        pip install flask requests &&
        python -c \"
from flask import Flask, jsonify
import logging
import time
from datetime import datetime

app = Flask(__name__)
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@app.route('/api/users')
def users():
    logger.info('Users API called')
    return jsonify({'users': ['Alice', 'Bob', 'Charlie']})

@app.route('/api/orders')
def orders():
    logger.info('Orders API called')
    return jsonify({'orders': [{'id': 1, 'item': 'laptop'}, {'id': 2, 'item': 'mouse'}]})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
        \"
      "
    ports:
      - "8083:5000"
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
        tag: "api-backend"
    networks:
      - elk-stack-lab_elk-network

  database:
    image: postgres:13-alpine
    container_name: database
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
        tag: "database"
    networks:
      - elk-stack-lab_elk-network

  redis-cache:
    image: redis:alpine
    container_name: redis-cache
    logging:
      driver: gelf
      options:
        gelf-address: "udp://localhost:12201"
        tag: "redis-cache"
    networks:
      - elk-stack-lab_elk-network

networks:
  elk-stack-lab_elk-network:
    external: true
Create a simple Nginx configuration:

nano nginx.conf
Add the following configuration:

events {
    worker_connections 1024;
}

http {
    upstream backend {
        server api-backend:5000;
    }

    server {
        listen 80;
        
        location / {
            return 200 'Frontend Service Running';
            add_header Content-Type text/plain;
        }
        
        location /api/ {
            proxy_pass http://backend/api/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
Subtask 4.2: Start Multi-Service Application
Start the multi-service application:

# Start all services
docker-compose -f docker-compose-multi.yml up -d

# Check service status
docker-compose -f docker-compose-multi.yml ps
Subtask 4.3: Generate Diverse Log Data
Generate logs from different services:

# Test frontend
curl http://localhost:8082/

# Test API endpoints
curl http://localhost:8082/api/users
curl http://localhost:8082/api/orders

# Direct API calls
curl http://localhost:8083/api/users
curl http://localhost:8083/api/orders

# Generate continuous traffic
for i in {1..20}; do
  curl -s http://localhost:8082/ > /dev/null
  curl -s http://localhost:8082/api/users > /dev/null
  curl -s http://localhost:8082/api/orders > /dev/null
  sleep 1
done
Task 5: Advanced ELK Stack Management with Docker Compose
Subtask 5.1: Create Production-Ready Configuration
Go back to the main ELK directory and create an enhanced configuration:

cd ../../elk-stack-lab
Create an enhanced Docker Compose file:

nano docker-compose-production.yml
Add the following configuration:

version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    container_name: elasticsearch-prod
    environment:
      - cluster.name=production-cluster
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms2g -Xmx2g"
      - xpack.security.enabled=false
      - xpack.monitoring.collection.enabled=true
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
      - ./config/elasticsearch/elasticsearch.yml:/usr/share/elasticsearch/config/elasticsearch.yml:ro
    ports:
      - "9200:9200"
    networks:
      - elk-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  logstash:
    image: docker.elastic.co/logstash/logstash:8.11.0
    container_name: logstash-prod
    volumes:
      - ./config/logstash/pipeline.conf:/usr/share/logstash/pipeline/logstash.conf:ro
      - ./config/logstash/logstash.yml:/usr/share/logstash/config/logstash.yml:ro
      - ./logs:/var/log/logstash
    ports:
      - "5044:5044"
      - "5000:5000"
      - "12201:12201/udp"
    networks:
      - elk-network
    depends_on:
      elasticsearch:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9600 || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  kibana:
    image: docker.elastic.co/kibana/kibana:8.11.0
    container_name: kibana-prod
    volumes:
      - ./config/kibana/kibana.yml:/usr/share/kibana/config/kibana.yml:ro
    ports:
      - "5601:5601"
    networks:
      - elk-network
    depends_on:
      logstash:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:5601/api/status || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.11.0
    container_name: filebeat
    user: root
    volumes:
      - ./config/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - elk-network
    depends_on:
      logstash:
        condition: service_healthy
    restart: unless-stopped

networks:
  elk-network:
    driver: bridge

volumes:
  elasticsearch-data:
    driver: local
Subtask 5.2: Configure Filebeat
Create Filebeat configuration:

mkdir -p config/filebeat
nano config/filebeat/filebeat.yml
Add the following configuration:

filebeat.inputs:
- type: container
  paths:
    - '/var/lib/docker/containers/*/*.log'
  processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644
Subtask 5.3: Create Management Scripts
Create a script to manage the ELK stack:

nano manage-elk.sh
Add the following script:

#!/bin/bash

COMPOSE_FILE="docker-compose-production.yml"

case "$1" in
    start)
        echo "Starting ELK Stack..."
        docker-compose -f $COMPOSE_FILE up -d
        echo "Waiting for services to be ready..."
        sleep 30
        echo "ELK Stack started successfully!"
        ;;
    stop)
        echo "Stopping ELK Stack..."
        docker-compose -f $COMPOSE_FILE down
        echo "ELK Stack stopped!"
        ;;
    restart)
        echo "Restarting ELK Stack..."
        docker-compose -f $COMPOSE_FILE restart
        echo "ELK Stack restarted!"
        ;;
    status)
        echo "ELK Stack Status:"
        docker-compose -f $COMPOSE_FILE ps
        ;;
    logs)
        if [ -z "$2" ]; then
            docker-compose -f $COMPOSE_FILE logs -f
        else
            docker-compose -f $COMPOSE_FILE logs -f $2
        fi
        ;;
    cleanup)
        echo "Cleaning up ELK Stack..."
        docker-compose -f $COMPOSE_FILE down -v
        docker system prune -f
        echo "Cleanup completed!"
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status|logs [service]|cleanup}"
        exit 1
        ;;
esac
Make the script executable:

chmod +x manage-elk.sh
Subtask 5.4: Test Production Configuration
Test the production setup:

# Start production ELK stack
./manage-elk.sh start

# Check status
./manage-elk.sh status

# View logs
./manage-elk.sh logs elasticsearch
Subtask 5.5: Create Index Templates
Create an index template for better log management:

# Wait for Elasticsearch to be ready
sleep 60

# Create index template
curl -X PUT "localhost:9200/_index_template/docker-logs-template" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["docker-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.lifecycle.name": "docker-logs-policy",
      "index.lifecycle.rollover_alias": "docker-logs"
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "container_name": {
          "type": "keyword"
        },
        "log_level": {
          "type": "keyword"
        },
        "message": {
          "type": "text",
          "analyzer": "standard"
        }
      }
    }
  }
}'
Troubleshooting Common Issues
Issue 1: Elasticsearch Won't Start
Problem: Elasticsearch container fails to start with memory errors.

Solution:

# Increase virtual memory
sudo sysctl -w vm.max_map_count=262144

# Make it permanent
echo 'vm.max_map_count=262144' | sudo tee -a /etc/sysctl.conf
Issue 2: Logstash Not Receiving Logs
Problem: Logs are not appearing in Elasticsearch.

Solution:

# Check Logstash logs
docker-compose logs logstash

# Test Logstash input
echo '{"message": "test log", "timestamp": "'$(date -Iseconds)'"}' | nc localhost 5000

# Verify Elasticsearch indices
curl "localhost:9200/_cat/indices?v"
Issue 3: Kibana Connection Issues
Problem: Kibana cannot connect to Elasticsearch.

Solution:

# Check if Elasticsearch is accessible
curl "localhost:9200/_cluster/health"

# Restart Kibana
docker-compose restart kibana

# Check Kibana logs
docker-compose logs kibana
Issue 4: High Resource Usage
Problem: ELK stack consuming too much memory.

Solution:

# Reduce Elasticsearch heap size in docker-compose.yml
# Change: "ES_JAVA_OPTS=-Xms2g -Xmx2g"
# To: "ES_JAVA_OPTS=-Xms512m -Xmx512m"

# Monitor resource usage
docker stats
Lab Validation
Validation Checklist
Verify your lab completion by checking the following:

ELK Stack Running: All three services (Elasticsearch, Logstash, Kibana) are running

docker-compose ps
Elasticsearch Health: Cluster is healthy

curl "localhost:9200/_cluster/health?pretty"
Log Ingestion: Logs are being received and indexed

curl "localhost:9200/docker-logs-*/_search?pretty&size=5"
Kibana Access: Dashboard is accessible and showing data

Open http://localhost:5601
Navigate to Discover
Verify log entries are visible
Multiple Container Integration: Logs from different containers are visible

Check for logs from sample-app, nginx, api-backend, etc.
Conclusion
Congratulations! You have successfully completed Lab 28: Logging and Monitoring with Docker - Setting Up ELK Stack.

What You Accomplished
In this lab, you have:

• Set up a complete ELK stack using Docker Compose with Elasticsearch, Logstash, and Kibana • Configured centralized logging for Docker containers using various logging drivers • Created log parsing and filtering rules in Logstash to structure log data • Built visualizations and dashboards in Kibana to monitor application logs • Integrated multiple services to send logs to a centralized logging system • Implemented production-ready configurations with health checks and restart policies • Learned troubleshooting techniques for common ELK stack issues

Why This Matters
Centralized logging is crucial for:

• Operational Visibility: Monitor application health and performance across all services • Debugging and Troubleshooting: Quickly identify and resolve issues in distributed systems • Security Monitoring: Detect suspicious activities and security threats • Compliance: Meet regulatory requirements for log retention and auditing • Performance Optimization: Identify bottlenecks and optimize system performance

Next Steps
To further enhance your logging and monitoring skills:

• Explore advanced Logstash filters and processors • Learn about Elasticsearch index lifecycle management • Implement alerting with Elasticsearch Watcher • Study log aggregation patterns for microservices • Practice with different log formats and parsing techniques • Explore integration with metrics monitoring tools like Prometheus

This lab provides a solid foundation for implementing enterprise-grade logging solutions in containerized environments, preparing you for real-world DevOps and system administration challenges.
