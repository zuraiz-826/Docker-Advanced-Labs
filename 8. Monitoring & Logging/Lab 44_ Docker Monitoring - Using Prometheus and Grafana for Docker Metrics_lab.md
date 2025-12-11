Lab 44: Docker Monitoring - Using Prometheus and Grafana for Docker Metrics
Lab Objectives
By the end of this lab, you will be able to:

Set up and configure Prometheus to collect Docker container metrics
Deploy and configure Grafana for data visualization
Implement cAdvisor for comprehensive container performance monitoring
Create custom dashboards in Grafana to display CPU, memory, and network usage metrics
Configure alerting rules for container resource usage thresholds
Understand the complete monitoring stack architecture for Docker environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and Docker Compose
Familiarity with Linux command line operations
Basic knowledge of YAML configuration files
Understanding of basic networking concepts
No prior experience with Prometheus or Grafana required
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to begin - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine installed
Docker Compose pre-configured
All necessary ports available for the monitoring stack
Internet connectivity for downloading container images
Task 1: Set up Prometheus Container to Collect Metrics
Subtask 1.1: Create Project Directory Structure
First, let's create a organized directory structure for our monitoring stack:

# Create main project directory
mkdir docker-monitoring-lab
cd docker-monitoring-lab

# Create subdirectories for configuration files
mkdir prometheus grafana alertmanager
Subtask 1.2: Configure Prometheus
Create the Prometheus configuration file that defines what metrics to collect:

# Create Prometheus configuration file
cat > prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
EOF
Subtask 1.3: Create Alert Rules
Define alerting rules for container resource monitoring:

# Create alert rules file
cat > prometheus/alert_rules.yml << 'EOF'
groups:
  - name: container_alerts
    rules:
      - alert: HighCPUUsage
        expr: rate(container_cpu_usage_seconds_total[5m]) * 100 > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage detected"
          description: "Container {{ $labels.name }} CPU usage is above 80%"

      - alert: HighMemoryUsage
        expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100 > 90
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High memory usage detected"
          description: "Container {{ $labels.name }} memory usage is above 90%"

      - alert: ContainerDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Container is down"
          description: "Container {{ $labels.instance }} has been down for more than 1 minute"
EOF
Task 2: Set up Grafana to Visualize Prometheus Data
Subtask 2.1: Configure Grafana Data Sources
Create Grafana provisioning configuration to automatically add Prometheus as a data source:

# Create Grafana provisioning directories
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards

# Create datasource configuration
cat > grafana/provisioning/datasources/prometheus.yml << 'EOF'
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
EOF
Subtask 2.2: Configure Dashboard Provisioning
Set up automatic dashboard loading:

# Create dashboard provisioning configuration
cat > grafana/provisioning/dashboards/dashboard.yml << 'EOF'
apiVersion: 1

providers:
  - name: 'default'
    orgId: 1
    folder: ''
    type: file
    disableDeletion: false
    updateIntervalSeconds: 10
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
EOF
Task 3: Use cAdvisor Docker Image for Container Performance Monitoring
Subtask 3.1: Create Docker Compose Configuration
Create a comprehensive Docker Compose file that includes all monitoring components:

# Create Docker Compose file
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'
      - '--web.enable-admin-api'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    networks:
      - monitoring

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
    devices:
      - /dev/kmsg
    networks:
      - monitoring

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
    networks:
      - monitoring

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager:/etc/alertmanager
    networks:
      - monitoring

  # Sample application to monitor
  nginx-app:
    image: nginx:latest
    container_name: nginx-sample
    ports:
      - "8081:80"
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring:
    driver: bridge
EOF
Subtask 3.2: Configure Alertmanager
Create Alertmanager configuration for handling alerts:

# Create Alertmanager configuration
cat > alertmanager/alertmanager.yml << 'EOF'
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alertmanager@example.com'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'

receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://127.0.0.1:5001/'
        send_resolved: true

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
EOF
Subtask 3.3: Deploy the Monitoring Stack
Start all monitoring services:

# Deploy the complete monitoring stack
docker-compose up -d

# Verify all containers are running
docker-compose ps

# Check container logs if needed
docker-compose logs prometheus
docker-compose logs grafana
docker-compose logs cadvisor
Task 4: Create Dashboards in Grafana to Display Metrics
Subtask 4.1: Access Grafana Web Interface
Open your web browser and navigate to http://localhost:3000
Login with credentials:
Username: admin
Password: admin123
Subtask 4.2: Create Container Overview Dashboard
Create a comprehensive dashboard JSON configuration:

# Create dashboard configuration
cat > grafana/provisioning/dashboards/docker-monitoring.json << 'EOF'
{
  "dashboard": {
    "id": null,
    "title": "Docker Container Monitoring",
    "tags": ["docker", "monitoring"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Container CPU Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_cpu_usage_seconds_total{name!=\"\"}[5m]) * 100",
            "legendFormat": "{{name}}"
          }
        ],
        "yAxes": [
          {
            "label": "CPU Usage (%)",
            "max": 100,
            "min": 0
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 0,
          "y": 0
        }
      },
      {
        "id": 2,
        "title": "Container Memory Usage",
        "type": "graph",
        "targets": [
          {
            "expr": "container_memory_usage_bytes{name!=\"\"}",
            "legendFormat": "{{name}}"
          }
        ],
        "yAxes": [
          {
            "label": "Memory Usage (Bytes)"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 12,
          "x": 12,
          "y": 0
        }
      },
      {
        "id": 3,
        "title": "Network I/O",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(container_network_receive_bytes_total{name!=\"\"}[5m])",
            "legendFormat": "{{name}} - Received"
          },
          {
            "expr": "rate(container_network_transmit_bytes_total{name!=\"\"}[5m])",
            "legendFormat": "{{name}} - Transmitted"
          }
        ],
        "yAxes": [
          {
            "label": "Bytes/sec"
          }
        ],
        "gridPos": {
          "h": 8,
          "w": 24,
          "x": 0,
          "y": 8
        }
      }
    ],
    "time": {
      "from": "now-1h",
      "to": "now"
    },
    "refresh": "5s"
  }
}
EOF
Subtask 4.3: Create Custom Dashboard Manually
Follow these steps to create additional dashboards through the Grafana UI:

Navigate to Dashboard Creation:

Click the "+" icon in the left sidebar
Select "Dashboard"
Click "Add new panel"
Configure CPU Usage Panel:

In the Query tab, enter: rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
Set Legend format to: {{name}}
In the Panel tab, set Title to: "Container CPU Usage (%)"
Set Y-axis max to 100
Configure Memory Usage Panel:

Add another panel
Query: container_memory_usage_bytes{name!=""} / 1024 / 1024
Legend: {{name}}
Title: "Container Memory Usage (MB)"
Configure Network Traffic Panel:

Add another panel
Query 1: rate(container_network_receive_bytes_total{name!=""}[5m])
Query 2: rate(container_network_transmit_bytes_total{name!=""}[5m])
Title: "Network I/O (Bytes/sec)"
Task 5: Set up Alerting for Container Resource Usage Thresholds
Subtask 5.1: Verify Alert Rules
Check that Prometheus has loaded the alert rules:

# Check Prometheus targets
curl http://localhost:9090/api/v1/targets

# Check alert rules
curl http://localhost:9090/api/v1/rules
Subtask 5.2: Configure Grafana Alerting
Access Grafana Alerting:

Navigate to Alerting → Alert Rules in Grafana
Click "New rule"
Create High CPU Alert:

Query: rate(container_cpu_usage_seconds_total{name!=""}[5m]) * 100
Condition: IS ABOVE 80
Evaluation: Every 1m for 2m
Add labels: severity: warning
Create Memory Alert:

Query: (container_memory_usage_bytes{name!=""} / container_spec_memory_limit_bytes{name!=""}) * 100
Condition: IS ABOVE 90
Evaluation: Every 1m for 2m
Add labels: severity: critical
Subtask 5.3: Test Alerting System
Create a test container that consumes resources to trigger alerts:

# Create a CPU stress test container
docker run -d --name cpu-stress --cpus="0.5" progrium/stress --cpu 2 --timeout 300s

# Create a memory stress test container
docker run -d --name memory-stress --memory="100m" progrium/stress --vm 1 --vm-bytes 150M --timeout 300s

# Monitor the alerts in Prometheus
# Navigate to http://localhost:9090/alerts
Subtask 5.4: Verify Monitoring Data
Check that all components are collecting data properly:

# Check Prometheus metrics
curl "http://localhost:9090/api/v1/query?query=up"

# Check cAdvisor metrics
curl "http://localhost:8080/metrics" | grep container_cpu

# Verify Grafana data source
curl -u admin:admin123 "http://localhost:3000/api/datasources"
Verification and Testing
Test the Complete Monitoring Stack
Verify Service Accessibility:

Prometheus: http://localhost:9090
Grafana: http://localhost:3000
cAdvisor: http://localhost:8080
Alertmanager: http://localhost:9093
Check Data Collection:

# Verify container metrics are being collected
curl "http://localhost:9090/api/v1/query?query=container_cpu_usage_seconds_total"

# Check if targets are healthy
curl "http://localhost:9090/api/v1/targets" | grep -i health
Test Dashboard Functionality:

Login to Grafana
Navigate to the Docker Monitoring dashboard
Verify that CPU, memory, and network metrics are displaying
Check that data updates every 5 seconds
Validate Alerting:

Check Prometheus alerts page for any active alerts
Verify Alertmanager is receiving alerts
Test alert resolution when stress containers are stopped
Troubleshooting Common Issues
Issue 1: cAdvisor Not Collecting Metrics
Solution:

# Ensure cAdvisor has proper permissions
docker-compose down
docker-compose up -d cadvisor

# Check cAdvisor logs
docker-compose logs cadvisor
Issue 2: Grafana Cannot Connect to Prometheus
Solution:

# Verify network connectivity
docker network ls
docker network inspect docker-monitoring-lab_monitoring

# Test connection from Grafana container
docker exec grafana curl http://prometheus:9090/api/v1/targets
Issue 3: No Data in Dashboards
Solution:

# Check Prometheus configuration
docker exec prometheus cat /etc/prometheus/prometheus.yml

# Verify scrape targets
curl http://localhost:9090/api/v1/targets

# Restart Prometheus if needed
docker-compose restart prometheus
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Stop and remove all containers
docker-compose down

# Remove volumes (optional - this will delete all data)
docker-compose down -v

# Remove test containers
docker rm -f cpu-stress memory-stress

# Clean up project directory (optional)
cd ..
rm -rf docker-monitoring-lab
Conclusion
Congratulations! You have successfully completed the Docker Monitoring lab using Prometheus and Grafana. Here's what you accomplished:

Key Achievements:

Monitoring Infrastructure: Set up a complete monitoring stack with Prometheus, Grafana, cAdvisor, and Alertmanager
Metrics Collection: Configured automated collection of container CPU, memory, and network metrics
Data Visualization: Created comprehensive dashboards to visualize container performance in real-time
Alerting System: Implemented proactive alerting for resource usage thresholds
Best Practices: Learned industry-standard monitoring practices for containerized environments
Why This Matters:

Production Readiness: These monitoring skills are essential for managing Docker containers in production environments
Proactive Management: Early detection of resource issues prevents application downtime and performance degradation
Scalability: Understanding container metrics helps in making informed scaling decisions
Troubleshooting: Comprehensive monitoring data accelerates problem identification and resolution
Career Development: Monitoring expertise is highly valued in DevOps and Site Reliability Engineering roles
Next Steps:

Explore advanced Prometheus queries and functions
Learn about custom metrics and application-specific monitoring
Investigate monitoring in Kubernetes environments
Study advanced alerting strategies and notification channels
Practice with monitoring at scale across multiple hosts
This lab provides a solid foundation for container monitoring that you can apply to real-world Docker deployments, making you better prepared for the Docker Certified Associate (DCA) certification and professional container management responsibilities.
