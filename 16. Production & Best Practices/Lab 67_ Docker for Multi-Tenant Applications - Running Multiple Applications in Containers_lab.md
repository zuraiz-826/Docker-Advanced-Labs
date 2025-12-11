Lab 67: Docker for Multi-Tenant Applications - Running Multiple Applications in Containers
Lab Objectives
By the end of this lab, you will be able to:

Deploy multiple isolated applications using Docker containers
Configure different network modes to ensure proper application isolation
Implement resource limits for CPU and memory usage using Docker flags
Use Docker Compose to manage multi-tenant application environments
Set up inter-container communication and access control for multi-tenant scenarios
Understand best practices for running multiple applications securely in containerized environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Linux command line operations
Familiarity with Docker concepts (containers, images, basic commands)
Understanding of networking fundamentals
Basic knowledge of YAML file structure
No prior experience with multi-tenant architectures required
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine installed
Docker Compose pre-configured
All necessary networking tools
Sample application files ready for deployment
Task 1: Deploy Multiple Applications in Separate Docker Containers
Subtask 1.1: Create Directory Structure and Sample Applications
First, let's create a proper directory structure for our multi-tenant setup:

# Create main project directory
mkdir ~/multi-tenant-docker
cd ~/multi-tenant-docker

# Create directories for different tenants
mkdir tenant-a tenant-b tenant-c
mkdir shared-configs logs
Subtask 1.2: Create Sample Web Applications
Let's create three different web applications representing different tenants:

Tenant A - Simple Python Flask App:

# Create Tenant A application
cd ~/multi-tenant-docker/tenant-a

# Create app.py
cat > app.py << 'EOF'
from flask import Flask, jsonify
import os

app = Flask(__name__)

@app.route('/')
def home():
    return jsonify({
        'tenant': 'Tenant A - E-commerce Platform',
        'status': 'running',
        'port': os.environ.get('PORT', '5000'),
        'container_id': os.environ.get('HOSTNAME', 'unknown')
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'tenant': 'A'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=int(os.environ.get('PORT', 5000)))
EOF

# Create Dockerfile for Tenant A
cat > Dockerfile << 'EOF'
FROM python:3.9-slim
WORKDIR /app
RUN pip install flask
COPY app.py .
EXPOSE 5000
CMD ["python", "app.py"]
EOF
Tenant B - Node.js Application:

# Create Tenant B application
cd ~/multi-tenant-docker/tenant-b

# Create package.json
cat > package.json << 'EOF'
{
  "name": "tenant-b-app",
  "version": "1.0.0",
  "description": "Tenant B - CRM System",
  "main": "server.js",
  "dependencies": {
    "express": "^4.18.0"
  }
}
EOF

# Create server.js
cat > server.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
    res.json({
        tenant: 'Tenant B - CRM System',
        status: 'running',
        port: port,
        container_id: process.env.HOSTNAME || 'unknown',
        timestamp: new Date().toISOString()
    });
});

app.get('/health', (req, res) => {
    res.json({ status: 'healthy', tenant: 'B' });
});

app.listen(port, '0.0.0.0', () => {
    console.log(`Tenant B app listening on port ${port}`);
});
EOF

# Create Dockerfile for Tenant B
cat > Dockerfile << 'EOF'
FROM node:16-slim
WORKDIR /app
COPY package.json .
RUN npm install
COPY server.js .
EXPOSE 3000
CMD ["node", "server.js"]
EOF
Tenant C - Simple Nginx Static Site:

# Create Tenant C application
cd ~/multi-tenant-docker/tenant-c

# Create index.html
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Tenant C - Analytics Dashboard</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; background-color: #f0f8ff; }
        .container { max-width: 600px; margin: 0 auto; padding: 20px; background: white; border-radius: 8px; }
        .status { color: #28a745; font-weight: bold; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Tenant C - Analytics Dashboard</h1>
        <p><strong>Status:</strong> <span class="status">Running</span></p>
        <p><strong>Service Type:</strong> Static Web Application</p>
        <p><strong>Container:</strong> <span id="hostname">Loading...</span></p>
        <p><strong>Last Updated:</strong> <span id="timestamp"></span></p>
    </div>
    <script>
        document.getElementById('hostname').textContent = window.location.hostname;
        document.getElementById('timestamp').textContent = new Date().toLocaleString();
    </script>
</body>
</html>
EOF

# Create Dockerfile for Tenant C
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
EOF
Subtask 1.3: Build Docker Images
Now let's build Docker images for all three applications:

# Build Tenant A image
cd ~/multi-tenant-docker/tenant-a
docker build -t tenant-a-app:v1.0 .

# Build Tenant B image
cd ~/multi-tenant-docker/tenant-b
docker build -t tenant-b-app:v1.0 .

# Build Tenant C image
cd ~/multi-tenant-docker/tenant-c
docker build -t tenant-c-app:v1.0 .

# Verify images are built
docker images | grep tenant
Subtask 1.4: Deploy Applications in Separate Containers
Deploy each application in its own container:

# Deploy Tenant A (Python Flask)
docker run -d \
  --name tenant-a-container \
  -p 8081:5000 \
  --restart unless-stopped \
  tenant-a-app:v1.0

# Deploy Tenant B (Node.js)
docker run -d \
  --name tenant-b-container \
  -p 8082:3000 \
  --restart unless-stopped \
  tenant-b-app:v1.0

# Deploy Tenant C (Nginx)
docker run -d \
  --name tenant-c-container \
  -p 8083:80 \
  --restart unless-stopped \
  tenant-c-app:v1.0

# Verify all containers are running
docker ps
Subtask 1.5: Test Application Deployment
Test each application to ensure they're working correctly:

# Test Tenant A
curl http://localhost:8081
curl http://localhost:8081/health

# Test Tenant B
curl http://localhost:8082
curl http://localhost:8082/health

# Test Tenant C
curl http://localhost:8083

# Check container logs
docker logs tenant-a-container
docker logs tenant-b-container
docker logs tenant-c-container
Task 2: Use Different Network Modes to Ensure Isolation
Subtask 2.1: Create Custom Networks for Tenant Isolation
Stop existing containers and create isolated networks:

# Stop all running containers
docker stop tenant-a-container tenant-b-container tenant-c-container
docker rm tenant-a-container tenant-b-container tenant-c-container

# Create separate networks for each tenant
docker network create --driver bridge tenant-a-network
docker network create --driver bridge tenant-b-network
docker network create --driver bridge tenant-c-network

# Create a shared network for inter-tenant communication (if needed)
docker network create --driver bridge shared-services-network

# List all networks
docker network ls
Subtask 2.2: Deploy Applications with Network Isolation
Redeploy applications using isolated networks:

# Deploy Tenant A with isolated network
docker run -d \
  --name tenant-a-container \
  --network tenant-a-network \
  -p 8081:5000 \
  --restart unless-stopped \
  tenant-a-app:v1.0

# Deploy Tenant B with isolated network
docker run -d \
  --name tenant-b-container \
  --network tenant-b-network \
  -p 8082:3000 \
  --restart unless-stopped \
  tenant-b-app:v1.0

# Deploy Tenant C with isolated network
docker run -d \
  --name tenant-c-container \
  --network tenant-c-network \
  -p 8083:80 \
  --restart unless-stopped \
  tenant-c-app:v1.0
Subtask 2.3: Test Network Isolation
Verify that containers cannot communicate with each other across different networks:

# Test network isolation by trying to ping between containers
# This should fail because they're on different networks

# Get container IP addresses
docker inspect tenant-a-container | grep IPAddress
docker inspect tenant-b-container | grep IPAddress
docker inspect tenant-c-container | grep IPAddress

# Try to ping from Tenant A to Tenant B (should fail)
docker exec tenant-a-container ping -c 3 tenant-b-container || echo "Network isolation working - ping failed as expected"

# Verify applications still work externally
curl http://localhost:8081
curl http://localhost:8082
curl http://localhost:8083
Subtask 2.4: Create Shared Database Container
Create a shared database that multiple tenants can access through the shared network:

# Create a PostgreSQL database container
docker run -d \
  --name shared-database \
  --network shared-services-network \
  -e POSTGRES_DB=multitenantdb \
  -e POSTGRES_USER=dbadmin \
  -e POSTGRES_PASSWORD=securepassword123 \
  -p 5432:5432 \
  postgres:13

# Connect Tenant A to shared services network (multi-network container)
docker network connect shared-services-network tenant-a-container

# Verify Tenant A can reach the database
docker exec tenant-a-container ping -c 3 shared-database
Task 3: Implement Resource Limits for CPU and Memory Usage
Subtask 3.1: Stop Existing Containers and Redeploy with Resource Limits
# Stop all containers
docker stop tenant-a-container tenant-b-container tenant-c-container shared-database
docker rm tenant-a-container tenant-b-container tenant-c-container shared-database
Subtask 3.2: Deploy with CPU and Memory Limits
Redeploy applications with specific resource constraints:

# Deploy Tenant A with resource limits (512MB RAM, 0.5 CPU cores)
docker run -d \
  --name tenant-a-container \
  --network tenant-a-network \
  -p 8081:5000 \
  --memory="512m" \
  --cpus="0.5" \
  --memory-swap="512m" \
  --restart unless-stopped \
  tenant-a-app:v1.0

# Deploy Tenant B with resource limits (256MB RAM, 0.3 CPU cores)
docker run -d \
  --name tenant-b-container \
  --network tenant-b-network \
  -p 8082:3000 \
  --memory="256m" \
  --cpus="0.3" \
  --memory-swap="256m" \
  --restart unless-stopped \
  tenant-b-app:v1.0

# Deploy Tenant C with resource limits (128MB RAM, 0.2 CPU cores)
docker run -d \
  --name tenant-c-container \
  --network tenant-c-network \
  -p 8083:80 \
  --memory="128m" \
  --cpus="0.2" \
  --memory-swap="128m" \
  --restart unless-stopped \
  tenant-c-app:v1.0

# Deploy shared database with higher limits (1GB RAM, 1 CPU core)
docker run -d \
  --name shared-database \
  --network shared-services-network \
  -e POSTGRES_DB=multitenantdb \
  -e POSTGRES_USER=dbadmin \
  -e POSTGRES_PASSWORD=securepassword123 \
  -p 5432:5432 \
  --memory="1g" \
  --cpus="1.0" \
  --memory-swap="1g" \
  postgres:13
Subtask 3.3: Monitor Resource Usage
Monitor the resource usage of containers:

# Check resource usage statistics
docker stats --no-stream

# Get detailed resource information for specific containers
docker inspect tenant-a-container | grep -A 10 "Memory"
docker inspect tenant-a-container | grep -A 5 "CpuShares"

# Create a monitoring script
cat > ~/multi-tenant-docker/monitor-resources.sh << 'EOF'
#!/bin/bash
echo "=== Container Resource Monitoring ==="
echo "Timestamp: $(date)"
echo ""
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}"
echo ""
echo "=== Container Limits ==="
for container in tenant-a-container tenant-b-container tenant-c-container shared-database; do
    echo "Container: $container"
    docker inspect $container | grep -A 3 '"Memory":'
    echo ""
done
EOF

chmod +x ~/multi-tenant-docker/monitor-resources.sh
~/multi-tenant-docker/monitor-resources.sh
Subtask 3.4: Test Resource Limits
Create a simple load test to verify resource limits are enforced:

# Create a simple load test script
cat > ~/multi-tenant-docker/load-test.sh << 'EOF'
#!/bin/bash
echo "Starting load test on all tenant applications..."

# Test Tenant A
echo "Testing Tenant A (Python Flask)..."
for i in {1..10}; do
    curl -s http://localhost:8081 > /dev/null &
done

# Test Tenant B
echo "Testing Tenant B (Node.js)..."
for i in {1..10}; do
    curl -s http://localhost:8082 > /dev/null &
done

# Test Tenant C
echo "Testing Tenant C (Nginx)..."
for i in {1..10}; do
    curl -s http://localhost:8083 > /dev/null &
done

wait
echo "Load test completed. Check resource usage:"
docker stats --no-stream
EOF

chmod +x ~/multi-tenant-docker/load-test.sh
~/multi-tenant-docker/load-test.sh
Task 4: Use Docker Compose to Manage Multi-Tenant Applications
Subtask 4.1: Create Docker Compose Configuration
Create a comprehensive Docker Compose file to manage all services:

cd ~/multi-tenant-docker

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # Tenant A - Python Flask Application
  tenant-a:
    build: ./tenant-a
    container_name: tenant-a-compose
    networks:
      - tenant-a-network
      - shared-services-network
    ports:
      - "8091:5000"
    environment:
      - PORT=5000
      - TENANT_ID=A
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    volumes:
      - ./logs:/app/logs

  # Tenant B - Node.js Application
  tenant-b:
    build: ./tenant-b
    container_name: tenant-b-compose
    networks:
      - tenant-b-network
    ports:
      - "8092:3000"
    environment:
      - PORT=3000
      - TENANT_ID=B
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.3'
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    volumes:
      - ./logs:/app/logs

  # Tenant C - Nginx Static Site
  tenant-c:
    build: ./tenant-c
    container_name: tenant-c-compose
    networks:
      - tenant-c-network
    ports:
      - "8093:80"
    environment:
      - TENANT_ID=C
    deploy:
      resources:
        limits:
          memory: 128M
          cpus: '0.2'
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Shared Database
  shared-database:
    image: postgres:13
    container_name: shared-db-compose
    networks:
      - shared-services-network
    ports:
      - "5433:5432"
    environment:
      - POSTGRES_DB=multitenantdb
      - POSTGRES_USER=dbadmin
      - POSTGRES_PASSWORD=securepassword123
    deploy:
      resources:
        limits:
          memory: 1G
          cpus: '1.0'
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./shared-configs/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U dbadmin -d multitenantdb"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Reverse Proxy (Nginx)
  reverse-proxy:
    image: nginx:alpine
    container_name: reverse-proxy-compose
    networks:
      - tenant-a-network
      - tenant-b-network
      - tenant-c-network
    ports:
      - "80:80"
    volumes:
      - ./shared-configs/nginx.conf:/etc/nginx/nginx.conf
    depends_on:
      - tenant-a
      - tenant-b
      - tenant-c
    restart: unless-stopped

networks:
  tenant-a-network:
    driver: bridge
  tenant-b-network:
    driver: bridge
  tenant-c-network:
    driver: bridge
  shared-services-network:
    driver: bridge

volumes:
  postgres_data:
EOF
Subtask 4.2: Create Supporting Configuration Files
Create database initialization script:

# Create database initialization script
cat > ~/multi-tenant-docker/shared-configs/init.sql << 'EOF'
-- Create tenant-specific schemas
CREATE SCHEMA IF NOT EXISTS tenant_a;
CREATE SCHEMA IF NOT EXISTS tenant_b;
CREATE SCHEMA IF NOT EXISTS tenant_c;

-- Create sample tables for each tenant
CREATE TABLE IF NOT EXISTS tenant_a.users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS tenant_b.customers (
    id SERIAL PRIMARY KEY,
    company_name VARCHAR(100) NOT NULL,
    contact_email VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS tenant_c.analytics (
    id SERIAL PRIMARY KEY,
    event_name VARCHAR(100) NOT NULL,
    event_data JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO tenant_a.users (username, email) VALUES 
    ('user1', 'user1@tenanta.com'),
    ('user2', 'user2@tenanta.com');

INSERT INTO tenant_b.customers (company_name, contact_email) VALUES 
    ('Company A', 'contact@companya.com'),
    ('Company B', 'contact@companyb.com');

INSERT INTO tenant_c.analytics (event_name, event_data) VALUES 
    ('page_view', '{"page": "/dashboard", "user_id": 123}'),
    ('button_click', '{"button": "submit", "form": "contact"}');
EOF
Create Nginx reverse proxy configuration:

# Create Nginx configuration for reverse proxy
cat > ~/multi-tenant-docker/shared-configs/nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream tenant_a {
        server tenant-a-compose:5000;
    }
    
    upstream tenant_b {
        server tenant-b-compose:3000;
    }
    
    upstream tenant_c {
        server tenant-c-compose:80;
    }

    server {
        listen 80;
        server_name tenant-a.local;
        
        location / {
            proxy_pass http://tenant_a;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

    server {
        listen 80;
        server_name tenant-b.local;
        
        location / {
            proxy_pass http://tenant_b;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

    server {
        listen 80;
        server_name tenant-c.local;
        
        location / {
            proxy_pass http://tenant_c;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }

    server {
        listen 80 default_server;
        
        location /tenant-a {
            proxy_pass http://tenant_a/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /tenant-b {
            proxy_pass http://tenant_b/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /tenant-c {
            proxy_pass http://tenant_c/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location / {
            return 200 'Multi-Tenant Application Gateway\nAvailable endpoints:\n/tenant-a\n/tenant-b\n/tenant-c\n';
            add_header Content-Type text/plain;
        }
    }
}
EOF
Subtask 4.3: Deploy Using Docker Compose
Stop existing containers and deploy using Docker Compose:

# Stop and remove existing containers
docker stop $(docker ps -aq) 2>/dev/null || true
docker rm $(docker ps -aq) 2>/dev/null || true

# Deploy using Docker Compose
cd ~/multi-tenant-docker
docker-compose up -d

# Check the status of all services
docker-compose ps

# View logs from all services
docker-compose logs --tail=20
Subtask 4.4: Test Docker Compose Deployment
Test the multi-tenant application deployment:

# Test individual tenant applications
curl http://localhost:8091  # Tenant A
curl http://localhost:8092  # Tenant B
curl http://localhost:8093  # Tenant C

# Test reverse proxy
curl http://localhost/tenant-a
curl http://localhost/tenant-b
curl http://localhost/tenant-c
curl http://localhost/

# Test database connectivity
docker-compose exec shared-database psql -U dbadmin -d multitenantdb -c "\dt tenant_a.*"
docker-compose exec shared-database psql -U dbadmin -d multitenantdb -c "SELECT * FROM tenant_a.users;"

# Check health status
docker-compose exec tenant-a curl -f http://localhost:5000/health
docker-compose exec tenant-b curl -f http://localhost:3000/health
Task 5: Set Up Inter-Container Communication and Access Control
Subtask 5.1: Create API Gateway Service
Create an API gateway to manage inter-tenant communication:

# Create API gateway directory
mkdir ~/multi-tenant-docker/api-gateway
cd ~/multi-tenant-docker/api-gateway

# Create API gateway application
cat > app.py << 'EOF'
from flask import Flask, request, jsonify, abort
import requests
import jwt
import datetime
import os

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key-change-in-production'

# Tenant service mappings
TENANT_SERVICES = {
    'tenant-a': 'http://tenant-a-compose:5000',
    'tenant-b': 'http://tenant-b-compose:3000',
    'tenant-c': 'http://tenant-c-compose:80'
}

# Simple authentication
def generate_token(tenant_id):
    payload = {
        'tenant_id': tenant_id,
        'exp': datetime.datetime.utcnow() + datetime.timedelta(hours=24)
    }
    return jwt.encode(payload, app.config['SECRET_KEY'], algorithm='HS256')

def verify_token(token):
    try:
        payload = jwt.decode(token, app.config['SECRET_KEY'], algorithms=['HS256'])
        return payload['tenant_id']
    except jwt.ExpiredSignatureError:
        return None
    except jwt.InvalidTokenError:
        return None

@app.route('/auth/login', methods=['POST'])
def login():
    data = request.get_json()
    tenant_id = data.get('tenant_id')
    password = data.get('password')
    
    # Simple authentication (in production, use proper authentication)
    if tenant_id in TENANT_SERVICES and password == 'password123':
        token = generate_token(tenant_id)
        return jsonify({'token': token, 'tenant_id': tenant_id})
    
    return jsonify({'error': 'Invalid credentials'}), 401

@app.route('/api/<tenant_id>/<path:path>', methods=['GET', 'POST', 'PUT', 'DELETE'])
def proxy_request(tenant_id, path):
    # Verify authentication
    auth_header = request.headers.get('Authorization')
    if not auth_header or not auth_header.startswith('Bearer '):
        abort(401)
    
    token = auth_header.split(' ')[1]
    authenticated_tenant = verify_token(token)
    
    if not authenticated_tenant:
        abort(401)
    
    # Check if tenant is authorized to access the requested service
    if authenticated_tenant != tenant_id:
        abort(403)
    
    # Check if tenant service exists
    if tenant_id not in TENANT_SERVICES:
        abort(404)
    
    # Proxy the request
    target_url = f"{TENANT_SERVICES[tenant_id]}/{path}"
    
    try:
        if request.method == 'GET':
            response = requests.get(target_url, params=request.args)
        elif request.method == 'POST':
            response = requests.post(target_url, json=request.get_json())
        elif request.method == 'PUT':
            response = requests.put(target_url, json=request.get_json())
        elif request.method == 'DELETE':
            response = requests.delete(target_url)
        
        return jsonify(response.json()), response.status_code
    except requests.exceptions.RequestException as e:
        return jsonify({'error': 'Service unavailable'}), 503

@app.route('/health')
def health():
    return jsonify({'status': 'healthy', 'service': 'api-gateway'})

@app.route('/')
def home():
    return jsonify({
        'service': 'Multi-Tenant API Gateway',
        'endpoints': {
            'auth': '/auth/login',
            'api': '/api/<tenant_id>/<path>',
            'health': '/health'
        }
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
EOF

# Create requirements.txt
cat > requirements.txt << 'EOF'
flask==2.3.3
requests==2.31.0
PyJWT==2.8.0
EOF

# Create Dockerfile for API Gateway
cat > Dockerfile << 'EOF'
FROM python:3.9-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY app.py .
EXPOSE 8000
CMD ["python", "app.py"]
EOF
Subtask 5.2: Update Docker Compose with API Gateway
Update the Docker Compose file to include the API gateway:

cd ~/multi-tenant-docker

# Backup existing docker-compose.yml
cp docker-compose.yml docker-compose.yml.backup

# Add API Gateway service to docker-compose.yml
cat >> docker-compose.yml << 'EOF'

  # API Gateway
  api-gateway:
    build: ./api-gateway
    container_name: api-gateway-compose
    networks:
      - tenant-a-network
      - tenant-b-network
      - tenant-c-network
      - shared-services-network
    ports:
      - "8000:8000"
    environment:
      - SECRET_KEY=your-secret-key-change-in-production
    deploy:
      resources:
        limits:
          memory: 256M
          cpus: '0.3'
    restart: unless-stopped
    depends_on:
      - tenant-a
      - tenant-b
      - tenant-c
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
EOF
Subtask 5.3: Implement Database Access Control
Create a database access control service:

# Create database access control directory
mkdir ~/multi-tenant-docker/db-access-control
cd ~/multi-tenant-docker/db-access-control

# Create database access control service
cat > app.py << 'EOF'
from flask import Flask, request, jsonify, abort
import psycopg2
import jwt
import os

app = Flask(__name__)
app.config['SECRET_KEY'] = 'your-secret-key-change-in-production'

# Database connection parameters
DB_CONFIG = {
    'host': 'shared-db-compose',
    'database': 'multitenantdb',
    'user': 'dbadmin',
    'password': 'securepassword123'
}

# Tenant schema mappings
TENANT_SCHEMAS = {
    'tenant-a': 'tenant_a',
    'tenant-b': 'tenant_b',
    'tenant-c': 'tenant_c'
}

def verify_token(token):
    try:
        payload = jwt.decode(token, app.config['SECRET_KEY'], algorithms=['HS256'])
        return payload['tenant_id']
    except:
        return None

def get_db_connection():
    return psycopg2.connect(**DB_CONFIG)

@app.route('/db/<tenant_id>/query', methods=['POST'])
def execute_query(tenant_id):
    # Verify authentication
    auth_header = request.headers.get('Authorization')
    if not auth_
