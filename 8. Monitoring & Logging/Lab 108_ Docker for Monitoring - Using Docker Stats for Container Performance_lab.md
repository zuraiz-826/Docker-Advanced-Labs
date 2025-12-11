Lab 108: Docker for Monitoring - Using Docker Stats for Container Performance
Lab Objectives
By the end of this lab, you will be able to:

Monitor Docker container resource usage using the docker stats command
Track multiple containers simultaneously and interpret performance metrics
Identify performance bottlenecks and resource constraints in containerized applications
Create automated monitoring scripts for continuous container performance tracking
Integrate Docker stats with external monitoring tools like Prometheus for enhanced observability
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, commands)
Familiarity with Linux command line interface
Basic knowledge of system resource concepts (CPU, memory, network, disk I/O)
Understanding of shell scripting fundamentals
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker already installed. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine installed
Pre-configured user with Docker permissions
Sample applications for monitoring exercises
Network access for downloading container images
Task 1: Run a Docker Container and Monitor Resource Usage with Docker Stats
Subtask 1.1: Understanding Docker Stats Command
The docker stats command provides real-time resource usage statistics for running containers, including:

CPU %: Percentage of CPU usage
MEM USAGE / LIMIT: Current memory usage and memory limit
MEM %: Percentage of memory usage
NET I/O: Network input/output bytes
BLOCK I/O: Disk read/write operations
PIDS: Number of processes running in the container
Subtask 1.2: Start a Basic Container
First, let's run a simple container to monitor:

# Pull and run an nginx web server container
docker run -d --name web-server -p 8080:80 nginx:latest

# Verify the container is running
docker ps
Subtask 1.3: Monitor Single Container with Docker Stats
Now let's monitor the container's resource usage:

# Monitor the web-server container in real-time
docker stats web-server

# To see stats once and exit (non-interactive mode)
docker stats --no-stream web-server
Expected Output Example:

CONTAINER ID   NAME         CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O   PIDS
a1b2c3d4e5f6   web-server   0.00%     3.5MiB / 1.944GiB     0.18%     1.09kB / 0B   0B / 0B     3
Subtask 1.4: Generate Load and Observe Changes
Let's create some load on our web server to see how stats change:

# Install curl if not available (in a new terminal)
sudo apt update && sudo apt install -y curl

# Generate some HTTP requests to create load
for i in {1..100}; do
    curl -s http://localhost:8080 > /dev/null
    sleep 0.1
done
While running the load test, observe the changes in CPU and network I/O in your docker stats output.

Task 2: Use Docker Stats to Monitor Multiple Containers at Once
Subtask 2.1: Deploy Multiple Containers
Let's create several containers to monitor simultaneously:

# Run a Redis database container
docker run -d --name redis-db redis:alpine

# Run a MySQL database container
docker run -d --name mysql-db -e MYSQL_ROOT_PASSWORD=password mysql:8.0

# Run a Python application container
docker run -d --name python-app python:3.9-slim sleep 3600

# Verify all containers are running
docker ps
Subtask 2.2: Monitor All Containers Simultaneously
# Monitor all running containers
docker stats

# Monitor specific containers by name
docker stats web-server redis-db mysql-db python-app

# Monitor with custom format (showing only specific columns)
docker stats --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"
Subtask 2.3: Understanding Multi-Container Output
The output will show all containers with their respective resource usage:

CONTAINER ID   NAME         CPU %     MEM USAGE / LIMIT     MEM %     NET I/O       BLOCK I/O     PIDS
a1b2c3d4e5f6   web-server   0.01%     3.5MiB / 1.944GiB     0.18%     1.45kB / 0B   0B / 0B       3
b2c3d4e5f6g7   redis-db     0.15%     7.2MiB / 1.944GiB     0.36%     656B / 0B     0B / 8.19kB   4
c3d4e5f6g7h8   mysql-db     0.25%     401MiB / 1.944GiB     20.1%     1.09kB / 0B   0B / 45.1kB   38
d4e5f6g7h8i9   python-app   0.00%     8.5MiB / 1.944GiB     0.43%     656B / 0B     0B / 0B       1
Task 3: Identify Performance Bottlenecks in Containers
Subtask 3.1: Create a CPU-Intensive Container
Let's create a container that will consume significant CPU resources:

# Run a container that performs CPU-intensive operations
docker run -d --name cpu-stress --cpus="1.0" alpine sh -c "while true; do echo 'CPU stress test' | sha256sum; done"
Subtask 3.2: Create a Memory-Intensive Container
# Run a container with memory constraints and stress
docker run -d --name memory-stress --memory="512m" alpine sh -c "
# Allocate memory gradually
for i in \$(seq 1 100); do
    dd if=/dev/zero of=/tmp/memory_\$i bs=1M count=5 2>/dev/null
    sleep 1
done
sleep 3600
"
Subtask 3.3: Monitor and Identify Bottlenecks
# Monitor all containers including the stress test containers
docker stats --no-stream

# Monitor continuously to see resource usage patterns
docker stats cpu-stress memory-stress web-server
Subtask 3.4: Analyzing Performance Bottlenecks
Key indicators to watch for:

CPU Bottleneck: CPU % consistently at or near 100%
Memory Bottleneck: MEM % approaching the limit, potential OOM kills
I/O Bottleneck: High BLOCK I/O values indicating disk stress
Network Bottleneck: High NET I/O values
Example Analysis Script:

# Create a script to analyze container performance
cat > analyze_performance.sh << 'EOF'
#!/bin/bash

echo "=== Container Performance Analysis ==="
echo "Timestamp: $(date)"
echo

# Get stats in parseable format
docker stats --no-stream --format "{{.Container}},{{.CPUPerc}},{{.MemPerc}},{{.NetIO}},{{.BlockIO}}" > /tmp/docker_stats.csv

# Analyze CPU usage
echo "=== CPU Usage Analysis ==="
while IFS=',' read -r container cpu mem net block; do
    cpu_num=$(echo $cpu | sed 's/%//')
    if (( $(echo "$cpu_num > 80" | bc -l) )); then
        echo "WARNING: $container has high CPU usage: $cpu"
    fi
done < /tmp/docker_stats.csv

echo

# Analyze Memory usage
echo "=== Memory Usage Analysis ==="
while IFS=',' read -r container cpu mem net block; do
    mem_num=$(echo $mem | sed 's/%//')
    if (( $(echo "$mem_num > 80" | bc -l) )); then
        echo "WARNING: $container has high memory usage: $mem"
    fi
done < /tmp/docker_stats.csv

rm /tmp/docker_stats.csv
EOF

chmod +x analyze_performance.sh
./analyze_performance.sh
Task 4: Automate Resource Monitoring with a Custom Script
Subtask 4.1: Create a Comprehensive Monitoring Script
# Create an advanced monitoring script
cat > docker_monitor.sh << 'EOF'
#!/bin/bash

# Docker Container Monitoring Script
# Usage: ./docker_monitor.sh [interval_seconds] [log_file]

INTERVAL=${1:-5}  # Default 5 seconds
LOGFILE=${2:-"docker_monitoring.log"}
ALERT_CPU_THRESHOLD=80
ALERT_MEM_THRESHOLD=80

# Colors for output
RED='\033[0;31m'
YELLOW='\033[1;33m'
GREEN='\033[0;32m'
NC='\033[0m' # No Color

# Function to log messages
log_message() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') - $1" | tee -a "$LOGFILE"
}

# Function to send alerts
send_alert() {
    local container=$1
    local metric=$2
    local value=$3
    local threshold=$4
    
    echo -e "${RED}ALERT: Container $container has high $metric usage: $value% (threshold: $threshold%)${NC}"
    log_message "ALERT: Container $container has high $metric usage: $value% (threshold: $threshold%)"
}

# Function to check container health
check_container_health() {
    local stats_line=$1
    local container=$(echo $stats_line | awk '{print $2}')
    local cpu=$(echo $stats_line | awk '{print $3}' | sed 's/%//')
    local mem=$(echo $stats_line | awk '{print $7}' | sed 's/%//')
    
    # Check CPU threshold
    if (( $(echo "$cpu > $ALERT_CPU_THRESHOLD" | bc -l) )); then
        send_alert "$container" "CPU" "$cpu" "$ALERT_CPU_THRESHOLD"
    fi
    
    # Check Memory threshold
    if (( $(echo "$mem > $ALERT_MEM_THRESHOLD" | bc -l) )); then
        send_alert "$container" "Memory" "$mem" "$ALERT_MEM_THRESHOLD"
    fi
}

# Main monitoring loop
echo -e "${GREEN}Starting Docker Container Monitoring...${NC}"
echo "Monitoring interval: ${INTERVAL} seconds"
echo "Log file: ${LOGFILE}"
echo "CPU Alert Threshold: ${ALERT_CPU_THRESHOLD}%"
echo "Memory Alert Threshold: ${ALERT_MEM_THRESHOLD}%"
echo "Press Ctrl+C to stop monitoring"
echo

log_message "Docker monitoring started"

while true; do
    echo -e "${YELLOW}=== $(date) ===${NC}"
    
    # Get container stats
    stats_output=$(docker stats --no-stream 2>/dev/null)
    
    if [ $? -eq 0 ]; then
        echo "$stats_output"
        
        # Process each container's stats (skip header)
        echo "$stats_output" | tail -n +2 | while read -r line; do
            check_container_health "$line"
        done
        
        # Log summary
        container_count=$(echo "$stats_output" | tail -n +2 | wc -l)
        log_message "Monitored $container_count containers"
    else
        echo -e "${RED}Error: Unable to retrieve Docker stats${NC}"
        log_message "ERROR: Unable to retrieve Docker stats"
    fi
    
    echo
    sleep "$INTERVAL"
done
EOF

chmod +x docker_monitor.sh
Subtask 4.2: Run the Monitoring Script
# Start monitoring with default settings (5-second interval)
./docker_monitor.sh

# Or specify custom interval and log file
# ./docker_monitor.sh 10 custom_monitor.log
Subtask 4.3: Create a Resource Usage Report Script
# Create a script to generate resource usage reports
cat > generate_report.sh << 'EOF'
#!/bin/bash

# Docker Resource Usage Report Generator
REPORT_FILE="docker_resource_report_$(date +%Y%m%d_%H%M%S).html"

cat > "$REPORT_FILE" << 'HTML'
<!DOCTYPE html>
<html>
<head>
    <title>Docker Resource Usage Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
        .high-usage { background-color: #ffcccc; }
        .medium-usage { background-color: #ffffcc; }
        .low-usage { background-color: #ccffcc; }
    </style>
</head>
<body>
    <h1>Docker Container Resource Usage Report</h1>
    <p>Generated on: $(date)</p>
    
    <h2>Current Container Statistics</h2>
    <table>
        <tr>
            <th>Container Name</th>
            <th>CPU Usage (%)</th>
            <th>Memory Usage</th>
            <th>Memory Percentage (%)</th>
            <th>Network I/O</th>
            <th>Block I/O</th>
            <th>PIDs</th>
        </tr>
HTML

# Get current stats and format for HTML
docker stats --no-stream --format "{{.Container}},{{.CPUPerc}},{{.MemUsage}},{{.MemPerc}},{{.NetIO}},{{.BlockIO}},{{.PIDs}}" | while IFS=',' read -r container cpu mem_usage mem_perc net_io block_io pids; do
    
    # Determine CSS class based on CPU usage
    cpu_num=$(echo $cpu | sed 's/%//')
    if (( $(echo "$cpu_num > 80" | bc -l) )); then
        css_class="high-usage"
    elif (( $(echo "$cpu_num > 50" | bc -l) )); then
        css_class="medium-usage"
    else
        css_class="low-usage"
    fi
    
    cat >> "$REPORT_FILE" << HTML
        <tr class="$css_class">
            <td>$container</td>
            <td>$cpu</td>
            <td>$mem_usage</td>
            <td>$mem_perc</td>
            <td>$net_io</td>
            <td>$block_io</td>
            <td>$pids</td>
        </tr>
HTML
done

cat >> "$REPORT_FILE" << 'HTML'
    </table>
    
    <h2>Resource Usage Guidelines</h2>
    <ul>
        <li><span style="background-color: #ccffcc; padding: 2px;">Green</span>: Low resource usage (&lt; 50% CPU)</li>
        <li><span style="background-color: #ffffcc; padding: 2px;">Yellow</span>: Medium resource usage (50-80% CPU)</li>
        <li><span style="background-color: #ffcccc; padding: 2px;">Red</span>: High resource usage (&gt; 80% CPU)</li>
    </ul>
</body>
</html>
HTML

echo "Report generated: $REPORT_FILE"
EOF

chmod +x generate_report.sh
./generate_report.sh
Task 5: Integrate Docker Stats with Other Monitoring Tools like Prometheus
Subtask 5.1: Set Up Prometheus for Docker Monitoring
First, let's create a Prometheus configuration:

# Create Prometheus configuration directory
mkdir -p prometheus_config

# Create Prometheus configuration file
cat > prometheus_config/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'docker-containers'
    static_configs:
      - targets: ['localhost:8080']
    scrape_interval: 5s
    metrics_path: /metrics

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
EOF
Subtask 5.2: Deploy cAdvisor for Container Metrics
cAdvisor (Container Advisor) provides container users an understanding of the resource usage and performance characteristics of their running containers.

# Run cAdvisor to collect container metrics
docker run -d \
  --name=cadvisor \
  --restart=always \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  gcr.io/cadvisor/cadvisor:latest

# Verify cAdvisor is running
curl http://localhost:8080/metrics | head -20
Subtask 5.3: Deploy Prometheus
# Run Prometheus with our configuration
docker run -d \
  --name=prometheus \
  --restart=always \
  --publish=9090:9090 \
  --volume=$(pwd)/prometheus_config:/etc/prometheus \
  prom/prometheus:latest \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/prometheus \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.console.templates=/etc/prometheus/consoles \
  --web.enable-lifecycle
Subtask 5.4: Create a Docker Stats Exporter Script
# Create a script that exports Docker stats to Prometheus format
cat > docker_stats_exporter.py << 'EOF'
#!/usr/bin/env python3

import docker
import time
import json
from http.server import HTTPServer, BaseHTTPRequestHandler
import threading

class DockerStatsExporter:
    def __init__(self):
        self.client = docker.from_env()
        self.stats_cache = {}
        
    def collect_stats(self):
        """Collect stats from all running containers"""
        try:
            containers = self.client.containers.list()
            stats = {}
            
            for container in containers:
                try:
                    # Get stats (non-blocking)
                    container_stats = container.stats(stream=False)
                    stats[container.name] = self.parse_stats(container_stats)
                except Exception as e:
                    print(f"Error collecting stats for {container.name}: {e}")
                    
            self.stats_cache = stats
        except Exception as e:
            print(f"Error collecting container stats: {e}")
    
    def parse_stats(self, stats):
        """Parse Docker stats into Prometheus metrics format"""
        parsed = {}
        
        # CPU usage
        cpu_delta = stats['cpu_stats']['cpu_usage']['total_usage'] - \
                   stats['precpu_stats']['cpu_usage']['total_usage']
        system_delta = stats['cpu_stats']['system_cpu_usage'] - \
                      stats['precpu_stats']['system_cpu_usage']
        
        if system_delta > 0:
            parsed['cpu_usage_percent'] = (cpu_delta / system_delta) * 100.0
        else:
            parsed['cpu_usage_percent'] = 0.0
            
        # Memory usage
        parsed['memory_usage_bytes'] = stats['memory_stats']['usage']
        parsed['memory_limit_bytes'] = stats['memory_stats']['limit']
        parsed['memory_usage_percent'] = (parsed['memory_usage_bytes'] / parsed['memory_limit_bytes']) * 100.0
        
        # Network I/O
        if 'networks' in stats:
            rx_bytes = sum(net['rx_bytes'] for net in stats['networks'].values())
            tx_bytes = sum(net['tx_bytes'] for net in stats['networks'].values())
            parsed['network_rx_bytes'] = rx_bytes
            parsed['network_tx_bytes'] = tx_bytes
        
        return parsed
    
    def generate_prometheus_metrics(self):
        """Generate Prometheus metrics format"""
        metrics = []
        
        for container_name, stats in self.stats_cache.items():
            labels = f'container="{container_name}"'
            
            metrics.append(f'docker_cpu_usage_percent{{{labels}}} {stats.get("cpu_usage_percent", 0)}')
            metrics.append(f'docker_memory_usage_bytes{{{labels}}} {stats.get("memory_usage_bytes", 0)}')
            metrics.append(f'docker_memory_limit_bytes{{{labels}}} {stats.get("memory_limit_bytes", 0)}')
            metrics.append(f'docker_memory_usage_percent{{{labels}}} {stats.get("memory_usage_percent", 0)}')
            metrics.append(f'docker_network_rx_bytes{{{labels}}} {stats.get("network_rx_bytes", 0)}')
            metrics.append(f'docker_network_tx_bytes{{{labels}}} {stats.get("network_tx_bytes", 0)}')
        
        return '\n'.join(metrics)

class MetricsHandler(BaseHTTPRequestHandler):
    def __init__(self, exporter, *args, **kwargs):
        self.exporter = exporter
        super().__init__(*args, **kwargs)
    
    def do_GET(self):
        if self.path == '/metrics':
            self.send_response(200)
            self.send_header('Content-type', 'text/plain')
            self.end_headers()
            
            metrics = self.exporter.generate_prometheus_metrics()
            self.wfile.write(metrics.encode())
        else:
            self.send_response(404)
            self.end_headers()

def stats_collector_thread(exporter):
    """Background thread to collect stats periodically"""
    while True:
        exporter.collect_stats()
        time.sleep(5)  # Collect stats every 5 seconds

if __name__ == '__main__':
    exporter = DockerStatsExporter()
    
    # Start stats collection thread
    collector_thread = threading.Thread(target=stats_collector_thread, args=(exporter,))
    collector_thread.daemon = True
    collector_thread.start()
    
    # Start HTTP server
    handler = lambda *args, **kwargs: MetricsHandler(exporter, *args, **kwargs)
    server = HTTPServer(('localhost', 8081), handler)
    
    print("Docker Stats Exporter running on http://localhost:8081/metrics")
    server.serve_forever()
EOF

# Install required Python packages
pip3 install docker

# Run the exporter (in background)
python3 docker_stats_exporter.py &
Subtask 5.5: Verify Integration
# Check if metrics are being exported
curl http://localhost:8081/metrics

# Check Prometheus targets
curl http://localhost:9090/api/v1/targets

# Access Prometheus web interface
echo "Access Prometheus at: http://localhost:9090"
echo "Access cAdvisor at: http://localhost:8080"
Subtask 5.6: Create Grafana Dashboard (Optional)
# Run Grafana for visualization
docker run -d \
  --name=grafana \
  --restart=always \
  --publish=3000:3000 \
  --env="GF_SECURITY_ADMIN_PASSWORD=admin" \
  grafana/grafana:latest

echo "Access Grafana at: http://localhost:3000 (admin/admin)"
Troubleshooting Common Issues
Issue 1: Docker Stats Not Showing Data
Problem: docker stats command returns empty or no data

Solution:

# Check if Docker daemon is running
sudo systemctl status docker

# Restart Docker service if needed
sudo systemctl restart docker

# Verify containers are actually running
docker ps -a
Issue 2: Permission Denied Errors
Problem: Permission denied when running Docker commands

Solution:

# Add user to docker group
sudo usermod -aG docker $USER

# Log out and log back in, or run:
newgrp docker
Issue 3: High Resource Usage Not Detected
Problem: Monitoring scripts don't detect high resource usage

Solution:

# Verify threshold values in scripts
# Check if bc calculator is installed
sudo apt install -y bc

# Test threshold comparison manually
echo "85 > 80" | bc -l
Issue 4: Prometheus Not Scraping Metrics
Problem: Prometheus cannot scrape metrics from exporters

Solution:

# Check if target endpoints are accessible
curl http://localhost:8080/metrics
curl http://localhost:8081/metrics

# Verify Prometheus configuration
docker logs prometheus

# Check network connectivity between containers
docker network ls
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Stop and remove all containers
docker stop $(docker ps -aq)
docker rm $(docker ps -aq)

# Remove custom images if created
docker image prune -f

# Stop background processes
pkill -f docker_stats_exporter.py
pkill -f docker_monitor.sh

# Remove generated files
rm -f docker_monitor.sh generate_report.sh analyze_performance.sh
rm -f docker_stats_exporter.py
rm -f *.log *.html
rm -rf prometheus_config
Conclusion
In this comprehensive lab, you have successfully learned how to monitor Docker containers using various tools and techniques:

Basic Monitoring: You mastered the docker stats command to monitor individual and multiple containers in real-time, understanding key performance metrics like CPU usage, memory consumption, network I/O, and disk I/O.

Performance Analysis: You learned to identify performance bottlenecks by creating CPU and memory-intensive containers and analyzing their resource consumption patterns.

Automation: You created sophisticated monitoring scripts that can automatically track container performance, generate alerts when thresholds are exceeded, and produce detailed HTML reports.

Integration: You successfully integrated Docker monitoring with enterprise-grade tools like Prometheus and cAdvisor, creating a complete monitoring stack that can scale to production environments.

Practical Skills: You developed hands-on experience with real-world monitoring scenarios, including troubleshooting common issues and implementing best practices for container observability.

Why This Matters: Container monitoring is crucial for maintaining healthy, performant applications in production environments. The skills you've developed in this lab are directly applicable to:

DevOps Operations: Monitoring production containerized applications
Performance Optimization: Identifying and resolving resource bottlenecks
Capacity Planning: Understanding resource requirements for scaling decisions
Incident Response: Quickly identifying problematic containers during outages
Cost Optimization: Right-sizing containers based on actual resource usage
These monitoring capabilities form the foundation of effective container orchestration and are essential skills for anyone working with Docker in professional environments. The integration with Prometheus and other monitoring tools prepares you for enterprise-scale container monitoring solutions.
