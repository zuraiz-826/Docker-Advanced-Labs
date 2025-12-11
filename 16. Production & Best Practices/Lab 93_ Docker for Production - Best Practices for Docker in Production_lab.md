Lab 93: Docker for Production - Best Practices for Docker in Production
Objectives
By the end of this lab, students will be able to:

Implement lightweight base images using Alpine Linux for production containers
Configure Docker health checks and automatic container restart policies
Set appropriate resource limits for containers to ensure system stability
Deploy logging and monitoring solutions using ELK stack and Prometheus
Secure Docker containers using Docker Bench for Security and Content Trust
Apply production-ready Docker best practices in real-world scenarios
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (containers, images, Dockerfiles)
Familiarity with Linux command line operations
Basic knowledge of YAML configuration files
Understanding of networking concepts
Experience with text editors (nano, vim, or similar)
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine pre-installed
Docker Compose pre-installed
All necessary tools and dependencies ready to use
Task 1: Use Lightweight Base Images (Alpine Linux)
Subtask 1.1: Understanding Base Image Sizes
First, let's examine the difference between standard and Alpine-based images.

Pull and compare different base images:
# Pull standard Ubuntu image
docker pull ubuntu:20.04

# Pull Alpine Linux image
docker pull alpine:3.18

# Pull Node.js standard image
docker pull node:18

# Pull Node.js Alpine image
docker pull node:18-alpine

# Compare image sizes
docker images | grep -E "(ubuntu|alpine|node)"
Analyze the size differences:
# Get detailed size information
docker image ls --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | grep -E "(ubuntu|alpine|node)"
Subtask 1.2: Creating Production-Ready Dockerfile with Alpine
Create a sample Node.js application:
# Create project directory
mkdir production-app
cd production-app

# Create package.json
cat > package.json << 'EOF'
{
  "name": "production-app",
  "version": "1.0.0",
  "description": "Production Docker demo",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

# Create server.js
cat > server.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Production Docker App',
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString()
  });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`Server running on port ${port}`);
});
EOF
Create optimized Dockerfile using Alpine:
cat > Dockerfile << 'EOF'
# Use Alpine-based Node.js image
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy package files first (for better caching)
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application code
COPY server.js ./

# Change ownership to non-root user
RUN chown -R nodejs:nodejs /app

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Start application
CMD ["npm", "start"]
EOF
Build and test the Alpine-based image:
# Build the image
docker build -t production-app:alpine .

# Run the container
docker run -d --name prod-app -p 3000:3000 production-app:alpine

# Test the application
curl http://localhost:3000
curl http://localhost:3000/health

# Check container status
docker ps
Task 2: Implement Docker Health Checks and Automatic Container Restarts
Subtask 2.1: Advanced Health Check Configuration
Create a comprehensive health check script:
# Create health check directory
mkdir health-checks
cd health-checks

# Create advanced health check script
cat > health-check.sh << 'EOF'
#!/bin/sh

# Check if the application is responding
if wget --no-verbose --tries=1 --spider --timeout=5 http://localhost:3000/health 2>/dev/null; then
    echo "Health check passed"
    exit 0
else
    echo "Health check failed"
    exit 1
fi
EOF

chmod +x health-check.sh
Create Dockerfile with advanced health checks:
cat > Dockerfile.healthcheck << 'EOF'
FROM node:18-alpine

WORKDIR /app

# Install wget for health checks
RUN apk add --no-cache wget

# Create non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application and health check script
COPY server.js ./
COPY health-check.sh ./

# Make health check script executable
RUN chmod +x health-check.sh

# Change ownership
RUN chown -R nodejs:nodejs /app

USER nodejs

EXPOSE 3000

# Advanced health check with custom script
HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=5 \
  CMD ./health-check.sh

CMD ["npm", "start"]
EOF

# Build the image
docker build -f Dockerfile.healthcheck -t production-app:healthcheck .
Subtask 2.2: Configure Automatic Restart Policies
Test different restart policies:
# Stop any running containers
docker stop prod-app 2>/dev/null || true
docker rm prod-app 2>/dev/null || true

# Run with restart policy: always
docker run -d --name prod-app-always \
  --restart=always \
  -p 3001:3000 \
  production-app:healthcheck

# Run with restart policy: unless-stopped
docker run -d --name prod-app-unless-stopped \
  --restart=unless-stopped \
  -p 3002:3000 \
  production-app:healthcheck

# Run with restart policy: on-failure with max retries
docker run -d --name prod-app-on-failure \
  --restart=on-failure:3 \
  -p 3003:3000 \
  production-app:healthcheck
Test restart functionality:
# Check container status
docker ps

# Simulate container failure
docker kill prod-app-always

# Wait and check if it restarted
sleep 5
docker ps

# Check restart count
docker inspect prod-app-always | grep -A 5 "RestartCount"
Task 3: Set Resource Limits for Containers
Subtask 3.1: Configure Memory and CPU Limits
Create containers with resource limits:
# Stop previous containers
docker stop $(docker ps -q) 2>/dev/null || true
docker rm $(docker ps -aq) 2>/dev/null || true

# Run container with memory limit (256MB)
docker run -d --name app-memory-limit \
  --memory=256m \
  --memory-swap=256m \
  -p 3000:3000 \
  production-app:alpine

# Run container with CPU limit (0.5 CPU cores)
docker run -d --name app-cpu-limit \
  --cpus=0.5 \
  -p 3001:3000 \
  production-app:alpine

# Run container with both memory and CPU limits
docker run -d --name app-resource-limits \
  --memory=512m \
  --memory-swap=512m \
  --cpus=1.0 \
  --restart=unless-stopped \
  -p 3002:3000 \
  production-app:alpine
Monitor resource usage:
# Check resource usage
docker stats --no-stream

# Get detailed resource information
docker inspect app-resource-limits | grep -A 10 "HostConfig"
Subtask 3.2: Create Docker Compose with Resource Limits
Create production Docker Compose file:
cd ~/production-app

cat > docker-compose.prod.yml << 'EOF'
version: '3.8'

services:
  web:
    build: .
    ports:
      - "3000:3000"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
        reservations:
          memory: 256M
          cpus: '0.5'
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    environment:
      - NODE_ENV=production
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.5'
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
EOF
Create Nginx configuration:
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream app {
        server web:3000;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        
        location /health {
            proxy_pass http://app/health;
            access_log off;
        }
    }
}
EOF
Deploy with resource limits:
# Deploy the stack
docker-compose -f docker-compose.prod.yml up -d

# Check resource usage
docker stats --no-stream

# Test the application through Nginx
curl http://localhost/
curl http://localhost/health
Task 4: Use Logging and Monitoring Tools (ELK Stack and Prometheus)
Subtask 4.1: Set Up ELK Stack for Logging
Create ELK stack configuration:
# Create monitoring directory
mkdir ~/monitoring
cd ~/monitoring

# Create Elasticsearch configuration
mkdir elasticsearch
cat > elasticsearch/elasticsearch.yml << 'EOF'
cluster.name: "docker-cluster"
network.host: 0.0.0.0
discovery.type: single-node
xpack.security.enabled: false
xpack.monitoring.collection.enabled: true
EOF

# Create Logstash configuration
mkdir logstash
cat > logstash/logstash.conf << 'EOF'
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][service] == "docker" {
    json {
      source => "message"
    }
    
    date {
      match => [ "timestamp", "ISO8601" ]
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "docker-logs-%{+YYYY.MM.dd}"
  }
}
EOF

# Create Filebeat configuration
mkdir filebeat
cat > filebeat/filebeat.yml << 'EOF'
filebeat.inputs:
- type: container
  paths:
    - '/var/lib/docker/containers/*/*.log'
  fields:
    service: docker
  fields_under_root: true

output.logstash:
  hosts: ["logstash:5044"]

logging.level: info
EOF
Create ELK Docker Compose file:
cat > docker-compose.elk.yml << 'EOF'
version: '3.8'

services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.8.0
    container_name: elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch_data:/usr/share/elasticsearch/data
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'

  logstash:
    image: docker.elastic.co/logstash/logstash:8.8.0
    container_name: logstash
    volumes:
      - ./logstash/logstash.conf:/usr/share/logstash/pipeline/logstash.conf:ro
    ports:
      - "5044:5044"
    depends_on:
      - elasticsearch
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'

  kibana:
    image: docker.elastic.co/kibana/kibana:8.8.0
    container_name: kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.8.0
    container_name: filebeat
    user: root
    volumes:
      - ./filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - logstash
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.25'

volumes:
  elasticsearch_data:
EOF
Deploy ELK stack:
# Start ELK stack
docker-compose -f docker-compose.elk.yml up -d

# Wait for services to start
sleep 60

# Check if Elasticsearch is running
curl http://localhost:9200

# Check Kibana (may take a few minutes to start)
echo "Kibana will be available at: http://localhost:5601"
Subtask 4.2: Set Up Prometheus and Grafana for Monitoring
Create Prometheus configuration:
mkdir prometheus
cat > prometheus/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

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

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
EOF

# Create alert rules
cat > prometheus/alert_rules.yml << 'EOF'
groups:
- name: docker_alerts
  rules:
  - alert: ContainerDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Container {{ $labels.instance }} is down"
      description: "Container {{ $labels.instance }} has been down for more than 1 minute."

  - alert: HighMemoryUsage
    expr: (container_memory_usage_bytes / container_spec_memory_limit_bytes) * 100 > 80
    for: 2m
    labels:
      severity: warning
    annotations:
      summary: "High memory usage on {{ $labels.name }}"
      description: "Container {{ $labels.name }} is using {{ $value }}% of its memory limit."
EOF
Create monitoring Docker Compose file:
cat > docker-compose.monitoring.yml << 'EOF'
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
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - prometheus
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.25'

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
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.25'

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:rw
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.25'

volumes:
  prometheus_data:
  grafana_data:
EOF
Deploy monitoring stack:
# Start monitoring stack
docker-compose -f docker-compose.monitoring.yml up -d

# Wait for services to start
sleep 30

# Check Prometheus
curl http://localhost:9090

# Access Grafana at http://localhost:3001 (admin/admin123)
echo "Grafana available at: http://localhost:3001"
echo "Username: admin"
echo "Password: admin123"
Task 5: Secure Docker Containers using Docker Bench and Content Trust
Subtask 5.1: Run Docker Bench for Security
Install and run Docker Bench for Security:
# Create security directory
mkdir ~/security
cd ~/security

# Download Docker Bench for Security
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security

# Run Docker Bench Security scan
sudo ./docker-bench-security.sh

# Save results to file
sudo ./docker-bench-security.sh > docker-bench-results.txt 2>&1

# Review critical findings
grep -A 5 -B 5 "WARN\|FAIL" docker-bench-results.txt | head -50
Implement security recommendations:
# Create secure Docker daemon configuration
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json << 'EOF'
{
  "icc": false,
  "userns-remap": "default",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "live-restore": true,
  "userland-proxy": false,
  "no-new-privileges": true
}
EOF

# Restart Docker daemon
sudo systemctl restart docker

# Verify configuration
docker info | grep -A 10 "Security Options"
Subtask 5.2: Implement Content Trust
Enable Docker Content Trust:
# Enable Content Trust
export DOCKER_CONTENT_TRUST=1

# Generate delegation keys
docker trust key generate mykey

# Create a repository for signed images
docker tag production-app:alpine localhost:5000/production-app:signed

# Note: For full content trust, you would need a Docker registry with notary
# For this lab, we'll demonstrate the concepts
Create security-hardened Dockerfile:
cd ~/production-app

cat > Dockerfile.secure << 'EOF'
# Use specific version with digest for security
FROM node:18-alpine@sha256:c7620fdecfefb96813da62519897808775230386f4c8482e972e37b8b18cb460

# Install security updates
RUN apk update && apk upgrade && \
    apk add --no-cache dumb-init && \
    rm -rf /var/cache/apk/*

# Create non-root user with specific UID/GID
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 -G nodejs

# Set working directory
WORKDIR /app

# Copy package files with proper ownership
COPY --chown=nodejs:nodejs package*.json ./

# Install dependencies and clean up
RUN npm ci --only=production && \
    npm cache clean --force && \
    rm -rf /tmp/*

# Copy application code
COPY --chown=nodejs:nodejs server.js ./

# Remove unnecessary packages and files
RUN apk del --purge && \
    rm -rf /var/cache/apk/* /tmp/* /var/tmp/*

# Switch to non-root user
USER nodejs

# Use dumb-init as PID 1
ENTRYPOINT ["dumb-init", "--"]

# Expose port
EXPOSE 3000

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --no-verbose --tries=1 --spider http://localhost:3000/health || exit 1

# Start application
CMD ["node", "server.js"]
EOF

# Build secure image
docker build -f Dockerfile.secure -t production-app:secure .
Subtask 5.3: Implement Runtime Security
Create secure container runtime configuration:
cat > docker-compose.secure.yml << 'EOF'
version: '3.8'

services:
  web:
    build:
      context: .
      dockerfile: Dockerfile.secure
    ports:
      - "3000:3000"
    restart: unless-stopped
    
    # Security configurations
    read_only: true
    tmpfs:
      - /tmp:noexec,nosuid,size=100m
    
    # Capabilities
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    
    # Security options
    security_opt:
      - no-new-privileges:true
      - apparmor:docker-default
    
    # Resource limits
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
    
    # User namespace
    user: "1001:1001"
    
    # Environment
    environment:
      - NODE_ENV=production
    
    # Logging
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    
    # Health check
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  default:
    driver: bridge
    driver_opts:
      com.docker.network.bridge.enable_icc: "false"
EOF
Deploy secure application:
# Deploy secure application
docker-compose -f docker-compose.secure.yml up -d

# Test security
docker exec -it $(docker ps -q -f "name=web") sh -c "whoami"
docker exec -it $(docker ps -q -f "name=web") sh -c "id"

# Test application functionality
curl http://localhost:3000
curl http://localhost:3000/health
Security scanning with Trivy:
# Install Trivy for vulnerability scanning
sudo apt-get update
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo "deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

# Scan the secure image
trivy image production-app:secure

# Generate detailed report
trivy image --format json --output security-report.json production-app:secure

# Check for high and critical vulnerabilities
trivy image --severity HIGH,CRITICAL production-app:secure
Verification and Testing
Comprehensive System Test
Test all components:
# Create comprehensive test script
cat > test-production-setup.sh << 'EOF'
#!/bin/bash

echo "=== Production Docker Setup Test ==="

# Test 1: Application availability
echo "Testing application availability..."
if curl -s http://localhost:3000 | grep -q "Production Docker App"; then
    echo "✓ Application is running"
else
    echo "✗ Application test failed"
fi

# Test 2: Health checks
echo "Testing health checks..."
if curl -s http://localhost:3000/health | grep -q "healthy"; then
    echo "✓ Health check is working"
else
    echo "✗ Health check failed"
fi

# Test 3: Resource limits
echo "Testing resource limits..."
MEMORY_LIMIT=$(docker inspect $(docker ps -q -f "name=web") | grep -o '"Memory":[0-9]*' | cut -d':' -f2)
if [ "$MEMORY_LIMIT" -gt 0 ]; then
    echo "✓ Memory limits are set"
else
    echo "✗ Memory limits not configured"
fi

# Test 4: Security
echo "Testing security configuration..."
USER_ID=$(docker exec $(docker ps -q -f "name=web") id -u)
if [ "$USER_ID" != "0" ]; then
    echo "✓ Running as non-root user"
else
    echo "✗ Running as root user (security risk)"
fi

# Test 5: Monitoring endpoints
echo "Testing monitoring..."
if curl -s http://localhost:9090 | grep -q "Prometheus"; then
    echo "✓ Prometheus is accessible"
else
    echo "✗ Prometheus not accessible"
fi

# Test 6: Logging
echo "Testing logging..."
if docker logs $(docker ps -q -f "name=web") | grep -q "Server running"; then
    echo "✓ Application logging is working"
else
    echo "✗ Application logging failed"
fi

echo "=== Test Complete ==="
EOF

chmod +x test-production-setup.sh
./test-production-setup.sh
Performance and load testing:
# Install Apache Bench for load testing
sudo apt-get install apache2-utils -y

# Run load test
echo "Running load test..."
ab -n 1000 -c 10 http://localhost:3000/

# Monitor during load test
docker stats --no-stream
Troubleshooting Common Issues
Issue 1: Container Memory Issues
# Check memory usage
docker stats --no-stream

# Increase memory limits if needed
docker update --memory=1g --memory-swap=1g container_name

# Check for memory leaks
docker exec container_name cat /proc/meminfo
Issue 2: Health Check Failures
# Debug health check
docker inspect container_name | grep -A 10 Health

# Check health check logs
docker logs container_name | grep health

# Test health check manually
docker exec container_name wget --spider http://localhost:3000/health
Issue 3: Security Scan Issues
# Update base images
docker pull node:18-alpine
docker build --no-cache -t production-app:secure .

# Check for security updates
docker exec container_name apk update && apk upgrade
Cleanup
# Stop all containers
docker-compose -f docker-compose.secure.yml down
docker-compose -f docker-compose.monitoring.yml down
docker-compose -f docker-compose.elk.yml down

# Remove unused resources
docker system prune -a --volumes

# Remove test directories (optional)
# rm -rf ~/production-app ~/monitoring ~/security
Conclusion
In this comprehensive lab, you have successfully implemented production-ready Docker best practices including:

Key Accomplishments:

Lightweight Images: Created optimized Docker images using Alpine Linux, reducing image size by up to 70% compared to standard base images
Health Monitoring: Implemented robust health checks and automatic restart policies to ensure application availability
Resource Management: Configured memory and CPU limits to prevent resource exhaustion and ensure fair resource allocation
Comprehensive Monitoring: Deployed ELK stack for centralized logging and Prometheus/Grafana for metrics monitoring
Security Hardening: Applied security best practices including non-root users, content trust, and vulnerability scanning
Production Benefits:

Reduced Attack Surface: Alpine-based images minimize potential vulnerabilities
Improved Reliability: Health checks and restart policies ensure high availability
Resource Efficiency: Proper resource limits prevent resource contention
Operational Visibility: Comprehensive logging and monitoring enable proactive issue resolution
Security Compliance: Security hardening meets enterprise security requirements
Next Steps:

Implement container orchestration with Kubernetes for larger deployments
Set up automated CI/CD pipelines with security scanning
Explore advanced monitoring with distributed tracing
Implement backup and disaster recovery strategies
Consider service mesh technologies for microservices architectures
These production practices form the foundation for running Docker containers reliably and securely in enterprise environments, preparing you for real-world Docker deployments and the Docker Certified Associate certification.
