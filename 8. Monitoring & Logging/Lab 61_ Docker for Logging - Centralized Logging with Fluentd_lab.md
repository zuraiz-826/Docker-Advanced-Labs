Lab 61: Docker for Logging - Centralized Logging with Fluentd
Objectives
By the end of this lab, you will be able to:

Install and configure Fluentd as a logging agent for Docker containers
Set up Fluentd to collect logs from multiple Docker containers
Store collected logs in Elasticsearch for centralized log management
Visualize logs using Kibana for better log analysis
Configure alerting based on log data and container performance metrics
Understand the EFK (Elasticsearch, Fluentd, Kibana) stack architecture
Implement log parsing and filtering using Fluentd configuration
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and Docker Compose
Familiarity with Linux command line operations
Basic knowledge of YAML configuration files
Understanding of logging concepts and log levels
Basic networking concepts (ports, IP addresses)
Note: Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click "Start Lab" to begin - no need to build your own VM or install Docker manually.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with:

Ubuntu 20.04 LTS
Docker Engine (latest stable version)
Docker Compose
Basic development tools
Internet connectivity for downloading required images
Task 1: Install and Configure Fluentd as a Logging Agent
Subtask 1.1: Create Project Directory Structure
First, let's create a organized directory structure for our logging project.

# Create main project directory
mkdir -p ~/docker-logging-lab
cd ~/docker-logging-lab

# Create subdirectories for different components
mkdir -p fluentd/conf
mkdir -p elasticsearch/data
mkdir -p kibana/config
mkdir -p sample-apps
mkdir -p alerts

# Set proper permissions for Elasticsearch data directory
sudo chown -R 1000:1000 elasticsearch/data
Subtask 1.2: Create Fluentd Configuration
Create the main Fluentd configuration file that will collect Docker container logs.

# Create Fluentd configuration file
cat > fluentd/conf/fluent.conf << 'EOF'
<source>
  @type forward
  port 24224
  bind 0.0.0.0
</source>

<source>
  @type tail
  path /var/log/containers/*.log
  pos_file /var/log/fluentd-containers.log.pos
  tag kubernetes.*
  format json
  read_from_head true
</source>

# Filter to parse Docker container logs
<filter docker.**>
  @type parser
  key_name log
  reserve_data true
  <parse>
    @type json
  </parse>
</filter>

# Add container metadata
<filter docker.**>
  @type record_transformer
  <record>
    hostname ${hostname}
    timestamp ${time}
  </record>
</filter>

# Output to Elasticsearch
<match **>
  @type elasticsearch
  host elasticsearch
  port 9200
  logstash_format true
  logstash_prefix docker-logs
  logstash_dateformat %Y.%m.%d
  include_tag_key true
  type_name access_log
  tag_key @log_name
  flush_interval 1s
</match>
EOF
Subtask 1.3: Create Custom Fluentd Dockerfile
Create a custom Fluentd image with the Elasticsearch plugin.

# Create Fluentd Dockerfile
cat > fluentd/Dockerfile << 'EOF'
FROM fluent/fluentd:v1.16-debian-1

# Use root account to use apk
USER root

# Install plugins
RUN buildDeps="sudo make gcc g++ libc-dev" \
 && apt-get update \
 && apt-get install -y --no-install-recommends $buildDeps \
 && sudo gem install fluent-plugin-elasticsearch \
 && sudo gem sources --clear-all \
 && SUDO_FORCE_REMOVE=yes \
    apt-get purge -y --auto-remove \
                  -o APT::AutoRemove::RecommendsImportant=false \
                  $buildDeps \
 && rm -rf /var/lib/apt/lists/* \
 && rm -rf /tmp/* /var/tmp/* /usr/lib/ruby/gems/*/cache/*.gem

# Copy configuration files
COPY conf/fluent.conf /fluentd/etc/
COPY entrypoint.sh /bin/

USER fluent
EOF
Subtask 1.4: Create Fluentd Entrypoint Script
# Create entrypoint script for Fluentd
cat > fluentd/entrypoint.sh << 'EOF'
#!/bin/sh

#source vars if file exists
DEFAULT=/etc/default/fluentd

if [ -r $DEFAULT ]; then
    set -o allexport
    . $DEFAULT
    set +o allexport
fi

# If the user has supplied only arguments append them to `fluentd` command
if [ "${1#-}" != "$1" ]; then
    set -- fluentd "$@"
fi

# If user does not supply config file or plugins, use the default
if [ "$1" = "fluentd" ]; then
    if ! echo $@ | grep ' \-c' ; then
       set -- "$@" -c /fluentd/etc/fluent.conf
    fi
    
    if ! echo $@ | grep ' \-p' ; then
       set -- "$@" -p /fluentd/plugins
    fi
fi

exec "$@"
EOF

# Make the script executable
chmod +x fluentd/entrypoint.sh
Task 2: Set Up Fluentd to Collect Logs from Docker Containers
Subtask 2.1: Create Docker Compose Configuration
Create a comprehensive Docker Compose file that includes the EFK stack and sample applications.

# Create Docker Compose file for the entire logging stack
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # Elasticsearch for log storage
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:7.17.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - ./elasticsearch/data:/usr/share/elasticsearch/data
    networks:
      - logging-network

  # Kibana for log visualization
  kibana:
    image: docker.elastic.co/kibana/kibana:7.17.0
    container_name: kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch
    networks:
      - logging-network

  # Fluentd for log collection
  fluentd:
    build: ./fluentd
    container_name: fluentd
    ports:
      - "24224:24224"
      - "24224:24224/udp"
    volumes:
      - /var/log:/var/log
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - elasticsearch
    networks:
      - logging-network

  # Sample web application 1
  web-app-1:
    image: nginx:alpine
    container_name: web-app-1
    ports:
      - "8081:80"
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.web-app-1
    depends_on:
      - fluentd
    networks:
      - logging-network

  # Sample web application 2
  web-app-2:
    image: httpd:alpine
    container_name: web-app-2
    ports:
      - "8082:80"
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.web-app-2
    depends_on:
      - fluentd
    networks:
      - logging-network

  # Sample Python application that generates logs
  log-generator:
    build: ./sample-apps/log-generator
    container_name: log-generator
    logging:
      driver: fluentd
      options:
        fluentd-address: localhost:24224
        tag: docker.log-generator
    depends_on:
      - fluentd
    networks:
      - logging-network

networks:
  logging-network:
    driver: bridge

volumes:
  elasticsearch-data:
    driver: local
EOF
Subtask 2.2: Create Sample Log-Generating Application
Create a Python application that generates various types of logs for testing.

# Create log generator application directory
mkdir -p sample-apps/log-generator

# Create Python log generator script
cat > sample-apps/log-generator/app.py << 'EOF'
import logging
import time
import random
import json
from datetime import datetime

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger('log-generator')

def generate_log_entry():
    """Generate different types of log entries"""
    log_types = ['INFO', 'WARNING', 'ERROR', 'DEBUG']
    actions = ['user_login', 'user_logout', 'data_processing', 'api_call', 'database_query']
    users = ['alice', 'bob', 'charlie', 'diana', 'eve']
    
    log_data = {
        'timestamp': datetime.now().isoformat(),
        'user': random.choice(users),
        'action': random.choice(actions),
        'response_time': random.randint(10, 1000),
        'status_code': random.choice([200, 201, 400, 401, 404, 500]),
        'ip_address': f"192.168.1.{random.randint(1, 254)}",
        'user_agent': 'LogGenerator/1.0'
    }
    
    log_level = random.choice(log_types)
    
    if log_level == 'INFO':
        logger.info(json.dumps(log_data))
    elif log_level == 'WARNING':
        log_data['message'] = 'High response time detected'
        logger.warning(json.dumps(log_data))
    elif log_level == 'ERROR':
        log_data['error'] = 'Database connection failed'
        logger.error(json.dumps(log_data))
    else:
        logger.debug(json.dumps(log_data))

def main():
    """Main function to continuously generate logs"""
    logger.info("Log generator started")
    
    while True:
        try:
            generate_log_entry()
            # Random sleep between 1-5 seconds
            time.sleep(random.randint(1, 5))
        except KeyboardInterrupt:
            logger.info("Log generator stopped")
            break
        except Exception as e:
            logger.error(f"Error generating log: {str(e)}")
            time.sleep(5)

if __name__ == "__main__":
    main()
EOF

# Create Dockerfile for log generator
cat > sample-apps/log-generator/Dockerfile << 'EOF'
FROM python:3.9-alpine

WORKDIR /app

COPY app.py .

CMD ["python", "app.py"]
EOF
Subtask 2.3: Build and Start the Logging Stack
Now let's build and start all the services.

# Build the custom Fluentd image
docker-compose build fluentd

# Build the log generator application
docker-compose build log-generator

# Start all services
docker-compose up -d

# Check if all services are running
docker-compose ps
Subtask 2.4: Verify Log Collection
Let's verify that Fluentd is collecting logs properly.

# Check Fluentd logs
docker-compose logs fluentd

# Check if Elasticsearch is receiving data
sleep 30  # Wait for logs to be indexed
curl -X GET "localhost:9200/_cat/indices?v"

# Check for Docker logs in Elasticsearch
curl -X GET "localhost:9200/docker-logs-*/_search?pretty" | head -50
Task 3: Store Logs in Elasticsearch
Subtask 3.1: Configure Elasticsearch Index Templates
Create index templates for better log management and performance.

# Create index template for Docker logs
curl -X PUT "localhost:9200/_index_template/docker-logs-template" -H 'Content-Type: application/json' -d'
{
  "index_patterns": ["docker-logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "index.refresh_interval": "5s"
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "container_name": {
          "type": "keyword"
        },
        "container_id": {
          "type": "keyword"
        },
        "source": {
          "type": "keyword"
        },
        "log": {
          "type": "text",
          "analyzer": "standard"
        },
        "tag": {
          "type": "keyword"
        },
        "hostname": {
          "type": "keyword"
        }
      }
    }
  }
}
'
Subtask 3.2: Create Index Lifecycle Management Policy
Set up ILM policy to manage log retention and storage optimization.

# Create ILM policy for log rotation
curl -X PUT "localhost:9200/_ilm/policy/docker-logs-policy" -H 'Content-Type: application/json' -d'
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "1GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "2d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "delete": {
        "min_age": "7d"
      }
    }
  }
}
'
Subtask 3.3: Verify Data Storage
Check that logs are being properly stored and indexed.

# Check Elasticsearch cluster health
curl -X GET "localhost:9200/_cluster/health?pretty"

# List all indices
curl -X GET "localhost:9200/_cat/indices?v&s=index"

# Get document count for Docker logs
curl -X GET "localhost:9200/docker-logs-*/_count?pretty"

# Search for recent logs
curl -X GET "localhost:9200/docker-logs-*/_search?pretty&size=5&sort=@timestamp:desc"
Task 4: Visualize Logs Using Kibana
Subtask 4.1: Access Kibana and Create Index Pattern
# Wait for Kibana to be fully ready
echo "Waiting for Kibana to start..."
sleep 60

# Check if Kibana is accessible
curl -I http://localhost:5601

echo "Kibana should now be accessible at http://localhost:5601"
echo "Please open your browser and navigate to http://localhost:5601"
Manual Steps in Kibana UI:

Open your browser and go to http://localhost:5601
Click on "Stack Management" in the left sidebar
Under "Kibana", click on "Index Patterns"
Click "Create index pattern"
Enter docker-logs-* as the index pattern
Click "Next step"
Select @timestamp as the time field
Click "Create index pattern"
Subtask 4.2: Create Kibana Dashboards
Create a script to import pre-configured dashboards.

# Create Kibana dashboard configuration
mkdir -p kibana/dashboards

cat > kibana/dashboards/docker-logs-dashboard.json << 'EOF'
{
  "version": "7.17.0",
  "objects": [
    {
      "id": "docker-logs-overview",
      "type": "dashboard",
      "attributes": {
        "title": "Docker Logs Overview",
        "hits": 0,
        "description": "Overview of Docker container logs",
        "panelsJSON": "[{\"version\":\"7.17.0\",\"gridData\":{\"x\":0,\"y\":0,\"w\":24,\"h\":15,\"i\":\"1\"},\"panelIndex\":\"1\",\"embeddableConfig\":{},\"panelRefName\":\"panel_1\"}]",
        "timeRestore": false,
        "timeTo": "now",
        "timeFrom": "now-24h",
        "refreshInterval": {
          "pause": false,
          "value": 10000
        },
        "kibanaSavedObjectMeta": {
          "searchSourceJSON": "{\"query\":{\"match_all\":{}},\"filter\":[]}"
        }
      }
    }
  ]
}
EOF

# Import dashboard (this requires Kibana to be fully loaded)
sleep 30
curl -X POST "localhost:5601/api/saved_objects/_import" -H "kbn-xsrf: true" -H "Content-Type: application/json" --form file=@kibana/dashboards/docker-logs-dashboard.json
Subtask 4.3: Create Visualizations
Manual Steps to Create Visualizations in Kibana:

Go to Kibana at http://localhost:5601
Click on "Visualize Library" in the left sidebar
Click "Create visualization"
Select "Vertical Bar Chart"
Choose the docker-logs-* index pattern
Configure the visualization:
Y-axis: Count
X-axis: Date Histogram on @timestamp
Split series: Terms on container_name.keyword
Save the visualization as "Container Logs Over Time"
Subtask 4.4: Generate Test Traffic
Generate some test traffic to see logs in action.

# Generate traffic to web applications
echo "Generating test traffic..."

for i in {1..20}; do
    curl -s http://localhost:8081 > /dev/null
    curl -s http://localhost:8082 > /dev/null
    sleep 2
done

echo "Test traffic generated. Check Kibana for new logs."
Task 5: Set Up Alerting Based on Log Data
Subtask 5.1: Install and Configure ElastAlert
Create an alerting system using ElastAlert for monitoring log patterns.

# Create ElastAlert configuration directory
mkdir -p alerts/elastalert

# Create ElastAlert configuration file
cat > alerts/elastalert/config.yaml << 'EOF'
rules_folder: /opt/elastalert/rules
run_every:
  minutes: 1
buffer_time:
  minutes: 15
es_host: elasticsearch
es_port: 9200
writeback_index: elastalert_status
writeback_alias: elastalert_alerts
alert_time_limit:
  days: 2
EOF

# Create error detection rule
cat > alerts/elastalert/error_detection.yaml << 'EOF'
name: Docker Container Error Detection
type: frequency
index: docker-logs-*
num_events: 5
timeframe:
  minutes: 5

filter:
- terms:
    log: ["ERROR", "error", "Error", "CRITICAL", "critical"]

alert:
- "email"
- "debug"

email:
- "admin@example.com"

alert_text: |
  Error detected in Docker containers!
  
  Container: {0}
  Error Count: {1}
  Time Range: {2}
  
  Recent Errors:
  {3}

alert_text_args:
  - container_name
  - num_matches
  - timeframe
  - log

include:
  - container_name
  - "@timestamp"
  - log
  - source
EOF

# Create high log volume rule
cat > alerts/elastalert/high_volume.yaml << 'EOF'
name: High Log Volume Alert
type: frequency
index: docker-logs-*
num_events: 100
timeframe:
  minutes: 5

filter:
- range:
    "@timestamp":
      gte: "now-5m"

alert:
- "debug"

alert_text: |
  High log volume detected!
  
  Log count in last 5 minutes: {0}
  This may indicate an issue with one or more containers.

alert_text_args:
  - num_matches

include:
  - container_name
  - "@timestamp"
  - num_matches
EOF
Subtask 5.2: Add ElastAlert to Docker Compose
Update the Docker Compose file to include ElastAlert.

# Add ElastAlert service to docker-compose.yml
cat >> docker-compose.yml << 'EOF'

  # ElastAlert for monitoring and alerting
  elastalert:
    image: jertel/elastalert2:latest
    container_name: elastalert
    volumes:
      - ./alerts/elastalert:/opt/elastalert/rules
      - ./alerts/elastalert/config.yaml:/opt/elastalert/config.yaml
    depends_on:
      - elasticsearch
    networks:
      - logging-network
    command: ["elastalert", "--config", "/opt/elastalert/config.yaml", "--verbose"]
EOF

# Restart services to include ElastAlert
docker-compose up -d elastalert
Subtask 5.3: Create Custom Alert Script
Create a custom monitoring script for additional alerting capabilities.

# Create custom monitoring script
cat > alerts/monitor_containers.py << 'EOF'
#!/usr/bin/env python3
import requests
import json
import time
import smtplib
from email.mime.text import MIMEText
from datetime import datetime, timedelta

class ContainerMonitor:
    def __init__(self, elasticsearch_url="http://localhost:9200"):
        self.es_url = elasticsearch_url
        self.alert_thresholds = {
            'error_count': 10,
            'response_time': 1000,
            'log_volume': 500
        }
    
    def query_elasticsearch(self, query):
        """Execute query against Elasticsearch"""
        try:
            response = requests.get(f"{self.es_url}/docker-logs-*/_search", 
                                  json=query, 
                                  headers={'Content-Type': 'application/json'})
            return response.json()
        except Exception as e:
            print(f"Error querying Elasticsearch: {e}")
            return None
    
    def check_error_rate(self):
        """Check for high error rates in the last 5 minutes"""
        query = {
            "query": {
                "bool": {
                    "must": [
                        {
                            "range": {
                                "@timestamp": {
                                    "gte": "now-5m"
                                }
                            }
                        },
                        {
                            "query_string": {
                                "query": "ERROR OR error OR Error OR CRITICAL"
                            }
                        }
                    ]
                }
            },
            "aggs": {
                "containers": {
                    "terms": {
                        "field": "container_name.keyword"
                    }
                }
            }
        }
        
        result = self.query_elasticsearch(query)
        if result and 'aggregations' in result:
            for bucket in result['aggregations']['containers']['buckets']:
                container = bucket['key']
                error_count = bucket['doc_count']
                
                if error_count > self.alert_thresholds['error_count']:
                    self.send_alert(f"High error rate in {container}: {error_count} errors in 5 minutes")
    
    def check_log_volume(self):
        """Check for unusual log volume"""
        query = {
            "query": {
                "range": {
                    "@timestamp": {
                        "gte": "now-5m"
                    }
                }
            },
            "aggs": {
                "containers": {
                    "terms": {
                        "field": "container_name.keyword"
                    }
                }
            }
        }
        
        result = self.query_elasticsearch(query)
        if result and 'aggregations' in result:
            for bucket in result['aggregations']['containers']['buckets']:
                container = bucket['key']
                log_count = bucket['doc_count']
                
                if log_count > self.alert_thresholds['log_volume']:
                    self.send_alert(f"High log volume in {container}: {log_count} logs in 5 minutes")
    
    def send_alert(self, message):
        """Send alert (print to console for this demo)"""
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        alert_message = f"[ALERT {timestamp}] {message}"
        print(alert_message)
        
        # In production, you would send email, Slack notification, etc.
        # Example email sending code:
        # self.send_email_alert(alert_message)
    
    def send_email_alert(self, message):
        """Send email alert (configure SMTP settings)"""
        # Configure these settings for your SMTP server
        smtp_server = "smtp.gmail.com"
        smtp_port = 587
        sender_email = "alerts@yourcompany.com"
        sender_password = "your_password"
        recipient_email = "admin@yourcompany.com"
        
        try:
            msg = MIMEText(message)
            msg['Subject'] = "Docker Container Alert"
            msg['From'] = sender_email
            msg['To'] = recipient_email
            
            server = smtplib.SMTP(smtp_server, smtp_port)
            server.starttls()
            server.login(sender_email, sender_password)
            server.send_message(msg)
            server.quit()
            
            print("Alert email sent successfully")
        except Exception as e:
            print(f"Failed to send email alert: {e}")
    
    def run_monitoring(self):
        """Main monitoring loop"""
        print("Starting container monitoring...")
        while True:
            try:
                self.check_error_rate()
                self.check_log_volume()
                time.sleep(60)  # Check every minute
            except KeyboardInterrupt:
                print("Monitoring stopped")
                break
            except Exception as e:
                print(f"Monitoring error: {e}")
                time.sleep(60)

if __name__ == "__main__":
    monitor = ContainerMonitor()
    monitor.run_monitoring()
EOF

# Make the script executable
chmod +x alerts/monitor_containers.py
Subtask 5.4: Test Alerting System
Generate some test errors to trigger alerts.

# Create a script to generate test errors
cat > generate_test_errors.py << 'EOF'
#!/usr/bin/env python3
import docker
import time

def generate_errors():
    """Generate test errors in containers"""
    client = docker.from_env()
    
    # Create a temporary container that generates errors
    try:
        container = client.containers.run(
            "alpine:latest",
            command="sh -c 'for i in $(seq 1 20); do echo ERROR: Test error message $i; sleep 1; done'",
            detach=True,
            remove=True,
            logging=docker.types.LogConfig(
                type="fluentd",
                config={
                    "fluentd-address": "localhost:24224",
                    "tag": "docker.error-generator"
                }
            )
        )
        
        print(f"Error generator container started: {container.id}")
        
        # Wait for container to finish
        container.wait()
        print("Error generation completed")
        
    except Exception as e:
        print(f"Error creating test container: {e}")

if __name__ == "__main__":
    generate_errors()
EOF

# Install Docker Python library and run error generator
pip3 install docker
python3 generate_test_errors.py

# Run the monitoring script in background
python3 alerts/monitor_containers.py &
MONITOR_PID=$!

echo "Monitoring started with PID: $MONITOR_PID"
echo "Check the output for any alerts generated"

# Let it run for a few minutes then stop
sleep 180
kill $MONITOR_PID
Verification and Testing
Comprehensive System Test
Let's run a comprehensive test to verify all components are working together.

# Create comprehensive test script
cat > test_logging_system.sh << 'EOF'
#!/bin/bash

echo "=== Docker Logging System Comprehensive Test ==="

# Test 1: Check all services are running
echo "1. Checking service status..."
docker-compose ps

# Test 2: Verify Elasticsearch is healthy
echo "2. Checking Elasticsearch health..."
curl -s http://localhost:9200/_cluster/health | jq '.status'

# Test 3: Check if indices exist
echo "3. Checking Elasticsearch indices..."
curl -s http://localhost:9200/_cat/indices?v

# Test 4: Verify Fluentd is receiving logs
echo "4. Checking Fluentd logs..."
docker-compose logs --tail=10 fluentd

# Test 5: Generate test traffic and verify log ingestion
echo "5. Generating test traffic..."
for i in {1..10}; do
    curl -s http://localhost:8081 > /dev/null
    curl -s http://localhost:8082 > /dev/null
done

# Wait for logs to be processed
sleep 10

# Test 6: Query recent logs
echo "6. Querying recent logs..."
curl -s "http://localhost:9200/docker-logs-*/_search?size=5&sort=@timestamp:desc" | jq '.hits.hits[]._source | {timestamp: ."@timestamp", container: .container_name, log: .log}'

# Test 7: Check Kibana accessibility
echo "7. Checking Kibana accessibility..."
curl -I http://localhost:5601

echo "=== Test Complete ==="
echo "Access Kibana at: http://localhost:5601"
echo "Elasticsearch API at: http://localhost:9200"
EOF

chmod +x test_logging_system.sh
./test_logging_system.sh
Troubleshooting Common Issues
Issue 1: Fluentd Not Receiving Logs
# Check Fluentd configuration
docker-compose exec fluentd cat /fluentd/etc/fluent.conf

# Check Fluentd logs for errors
docker-compose logs fluentd

# Verify Fluentd port is accessible
netstat -tlnp | grep 24224

# Test Fluentd connectivity
echo '{"message":"test"}' | nc localhost 24224
Issue 2: Elasticsearch Connection Issues
# Check Elasticsearch logs
docker-compose logs elasticsearch

# Verify Elasticsearch is responding
curl -X GET "localhost:9200/_cluster/health?pretty"

# Check disk space (Elasticsearch needs adequate space)
df -h

# Reset Elasticsearch data if needed
docker-compose down
sudo rm -rf elasticsearch/data/*
docker-compose up -d elasticsearch
Issue 3: Kibana Not Loading
# Check Kibana logs
docker-compose logs kibana

# Verify Kibana can connect to Elasticsearch
docker-compose exec kibana curl -X GET "elasticsearch:9200/_cluster/health"

# Restart Kibana if needed
docker-compose restart kibana
Issue 4: No Logs Appearing in Kibana
# Check if logs are in Elasticsearch
curl -X GET "localhost:9200/docker-logs-*/_count"

# Verify index pattern in Kibana matches your indices
curl -X GET "localhost:9200/_cat/indices?v" | grep docker-logs

# Check time range in Kibana (ensure it covers when logs were generated)
Performance Optimization
Optimize Fluentd Configuration
# Create optimize
