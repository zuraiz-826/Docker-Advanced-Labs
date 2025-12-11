Lab 25: Docker Compose for Multi-Container Web App
Objectives
By the end of this lab, you will be able to:

• Create and configure a docker-compose.yml file for multi-container applications • Deploy a complete web application stack with database and cache services • Configure network settings and persistent volumes using Docker Compose • Use docker-compose commands to manage multi-container applications • Scale services dynamically and monitor application logs • Troubleshoot common Docker Compose issues • Understand service orchestration principles in containerized environments

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker containers and images • Familiarity with YAML file format • Basic knowledge of web applications and databases • Understanding of command-line interface operations • Completion of previous Docker fundamentals labs

Required Knowledge Level: Beginner to Intermediate Docker concepts

Lab Environment Setup
Good News! Al Nafi provides you with ready-to-use Linux-based cloud machines. Simply click Start Lab and you'll have access to a fully configured environment with Docker and Docker Compose pre-installed. No need to build your own virtual machine or install software locally.

Your lab environment includes: • Ubuntu Linux with Docker Engine installed • Docker Compose v2.x pre-configured • Text editors (nano, vim) available • All necessary networking and permissions configured

Lab Overview
In this lab, you'll build a complete web application stack consisting of: • Web Application: A Python Flask application • Database: PostgreSQL for data persistence • Cache: Redis for session management and caching • Reverse Proxy: Nginx for load balancing

This represents a real-world application architecture commonly used in production environments.

Task 1: Create the Application Structure
Step 1.1: Set Up the Project Directory
First, create a organized directory structure for your multi-container application:

mkdir ~/docker-compose-lab
cd ~/docker-compose-lab
mkdir app nginx
Step 1.2: Create the Flask Web Application
Create the main application file:

nano app/app.py
Add the following Python Flask application code:

from flask import Flask, request, jsonify, session
import psycopg2
import redis
import os
import json
from datetime import datetime

app = Flask(__name__)
app.secret_key = 'your-secret-key-change-in-production'

# Database configuration
DB_HOST = os.environ.get('DB_HOST', 'db')
DB_NAME = os.environ.get('DB_NAME', 'webapp')
DB_USER = os.environ.get('DB_USER', 'postgres')
DB_PASS = os.environ.get('DB_PASS', 'password')

# Redis configuration
REDIS_HOST = os.environ.get('REDIS_HOST', 'redis')
REDIS_PORT = int(os.environ.get('REDIS_PORT', 6379))

# Initialize Redis connection
try:
    r = redis.Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=True)
except Exception as e:
    print(f"Redis connection error: {e}")

def get_db_connection():
    try:
        conn = psycopg2.connect(
            host=DB_HOST,
            database=DB_NAME,
            user=DB_USER,
            password=DB_PASS
        )
        return conn
    except Exception as e:
        print(f"Database connection error: {e}")
        return None

@app.route('/')
def home():
    # Increment page views in Redis
    try:
        views = r.incr('page_views')
    except:
        views = 'N/A'
    
    return jsonify({
        'message': 'Welcome to Multi-Container Web App!',
        'page_views': views,
        'timestamp': datetime.now().isoformat()
    })

@app.route('/users', methods=['GET', 'POST'])
def users():
    conn = get_db_connection()
    if not conn:
        return jsonify({'error': 'Database connection failed'}), 500
    
    if request.method == 'POST':
        data = request.get_json()
        name = data.get('name')
        email = data.get('email')
        
        try:
            cur = conn.cursor()
            cur.execute(
                "INSERT INTO users (name, email, created_at) VALUES (%s, %s, %s)",
                (name, email, datetime.now())
            )
            conn.commit()
            cur.close()
            conn.close()
            
            # Cache the user data
            user_data = {'name': name, 'email': email}
            r.setex(f'user:{email}', 3600, json.dumps(user_data))
            
            return jsonify({'message': 'User created successfully'}), 201
        except Exception as e:
            conn.close()
            return jsonify({'error': str(e)}), 400
    
    else:  # GET request
        try:
            cur = conn.cursor()
            cur.execute("SELECT name, email, created_at FROM users ORDER BY created_at DESC")
            users = cur.fetchall()
            cur.close()
            conn.close()
            
            user_list = []
            for user in users:
                user_list.append({
                    'name': user[0],
                    'email': user[1],
                    'created_at': user[2].isoformat()
                })
            
            return jsonify({'users': user_list})
        except Exception as e:
            conn.close()
            return jsonify({'error': str(e)}), 500

@app.route('/health')
def health():
    # Check database connection
    db_status = 'healthy'
    try:
        conn = get_db_connection()
        if conn:
            conn.close()
        else:
            db_status = 'unhealthy'
    except:
        db_status = 'unhealthy'
    
    # Check Redis connection
    redis_status = 'healthy'
    try:
        r.ping()
    except:
        redis_status = 'unhealthy'
    
    return jsonify({
        'status': 'running',
        'database': db_status,
        'redis': redis_status,
        'timestamp': datetime.now().isoformat()
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
Step 1.3: Create Application Requirements
Create the Python requirements file:

nano app/requirements.txt
Add the following dependencies:

Flask==2.3.3
psycopg2-binary==2.9.7
redis==4.6.0
gunicorn==21.2.0
Step 1.4: Create the Application Dockerfile
nano app/Dockerfile
Add the following Dockerfile content:

FROM python:3.11-slim

WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .

# Create non-root user
RUN useradd --create-home --shell /bin/bash app && chown -R app:app /app
USER app

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Run application
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
Task 2: Create Docker Compose Configuration
Step 2.1: Create the Main Docker Compose File
Create the main orchestration file:

nano docker-compose.yml
Add the comprehensive Docker Compose configuration:

version: '3.8'

services:
  # Web Application Service
  web:
    build: 
      context: ./app
      dockerfile: Dockerfile
    container_name: webapp
    ports:
      - "5000:5000"
    environment:
      - DB_HOST=db
      - DB_NAME=webapp
      - DB_USER=postgres
      - DB_PASS=securepassword123
      - REDIS_HOST=redis
      - REDIS_PORT=6379
      - FLASK_ENV=development
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    networks:
      - app-network
    volumes:
      - ./app:/app
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:5000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # PostgreSQL Database Service
  db:
    image: postgres:15-alpine
    container_name: postgres_db
    environment:
      - POSTGRES_DB=webapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=securepassword123
      - POSTGRES_INITDB_ARGS=--auth-host=scram-sha-256
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql:ro
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres -d webapp"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # Redis Cache Service
  redis:
    image: redis:7-alpine
    container_name: redis_cache
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf:ro
    networks:
      - app-network
    restart: unless-stopped
    command: redis-server /usr/local/etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # Nginx Reverse Proxy
  nginx:
    image: nginx:alpine
    container_name: nginx_proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - web
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

# Network Configuration
networks:
  app-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

# Volume Configuration
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
Step 2.2: Create Database Initialization Script
nano init-db.sql
Add the database schema:

-- Create users table
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO users (name, email) VALUES 
    ('John Doe', 'john@example.com'),
    ('Jane Smith', 'jane@example.com'),
    ('Bob Johnson', 'bob@example.com')
ON CONFLICT (email) DO NOTHING;

-- Create indexes for better performance
CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
CREATE INDEX IF NOT EXISTS idx_users_created_at ON users(created_at);
Step 2.3: Create Redis Configuration
nano redis.conf
Add Redis configuration:

# Redis configuration for Docker Compose lab
bind 0.0.0.0
port 6379
timeout 0
tcp-keepalive 300

# Memory management
maxmemory 256mb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10
save 60 10000

# Security
requirepass ""

# Logging
loglevel notice
logfile ""

# Performance
databases 16
Step 2.4: Create Nginx Configuration
nano nginx/nginx.conf
Add the main Nginx configuration:

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';

    access_log /var/log/nginx/access.log main;

    # Performance settings
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip compression
    gzip on;
    gzip_vary on;
    gzip_min_length 10240;
    gzip_proxied expired no-cache no-store private must-revalidate auth;
    gzip_types text/plain text/css text/xml text/javascript application/javascript application/xml+rss application/json;

    # Include server configurations
    include /etc/nginx/conf.d/*.conf;
}
Create the server configuration:

nano nginx/default.conf
Add the server block configuration:

upstream webapp {
    server web:5000;
}

server {
    listen 80;
    server_name localhost;

    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;

    # Proxy settings
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Main application
    location / {
        proxy_pass http://webapp;
        proxy_connect_timeout 30s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
    }

    # Health check endpoint
    location /health {
        proxy_pass http://webapp/health;
        access_log off;
    }

    # Static files (if any)
    location /static/ {
        alias /var/www/static/;
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Error pages
    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
Task 3: Launch and Manage the Multi-Container Application
Step 3.1: Validate the Docker Compose Configuration
Before launching, validate your configuration:

# Check if Docker Compose file is valid
docker-compose config

# Validate and view the processed configuration
docker-compose config --services
Step 3.2: Build and Start All Services
Launch the complete application stack:

# Build and start all services in detached mode
docker-compose up -d --build

# View the startup process (optional)
docker-compose up --build
Expected Output:

Creating network "docker-compose-lab_app-network" with driver "bridge"
Creating volume "docker-compose-lab_postgres_data" with default driver
Creating volume "docker-compose-lab_redis_data" with default driver
Building web
...
Creating postgres_db ... done
Creating redis_cache ... done
Creating webapp ... done
Creating nginx_proxy ... done
Step 3.3: Verify All Services Are Running
Check the status of all services:

# View running containers
docker-compose ps

# Check detailed service status
docker-compose ps --services

# View service logs
docker-compose logs
Step 3.4: Test the Application
Test each component of your application:

# Test the main application through Nginx
curl http://localhost/

# Test the health endpoint
curl http://localhost/health

# Test direct web service access
curl http://localhost:5000/

# Create a new user
curl -X POST http://localhost/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Test User", "email": "test@example.com"}'

# Retrieve all users
curl http://localhost/users
Task 4: Configure Network Settings and Volumes
Step 4.1: Inspect Network Configuration
Examine the network setup:

# List Docker networks
docker network ls

# Inspect the application network
docker network inspect docker-compose-lab_app-network

# Test inter-service connectivity
docker-compose exec web ping db
docker-compose exec web ping redis
Step 4.2: Examine Volume Configuration
Check persistent storage:

# List Docker volumes
docker volume ls

# Inspect database volume
docker volume inspect docker-compose-lab_postgres_data

# Inspect Redis volume
docker volume inspect docker-compose-lab_redis_data

# Check volume usage
docker system df
Step 4.3: Verify Data Persistence
Test data persistence by restarting services:

# Stop all services
docker-compose down

# Start services again
docker-compose up -d

# Verify data is still present
curl http://localhost/users
Task 5: Scale Services and Monitor Logs
Step 5.1: Scale the Web Service
Demonstrate horizontal scaling:

# Scale web service to 3 instances
docker-compose up -d --scale web=3

# Verify scaled services
docker-compose ps

# Check load balancing (run multiple times)
for i in {1..10}; do
  curl -s http://localhost/ | grep -o '"timestamp":"[^"]*"'
  sleep 1
done
Step 5.2: Monitor Application Logs
Monitor logs from different services:

# View logs from all services
docker-compose logs -f

# View logs from specific service
docker-compose logs -f web

# View logs with timestamps
docker-compose logs -f -t

# View last 50 lines of logs
docker-compose logs --tail=50 web
Step 5.3: Real-time Log Monitoring
Set up real-time monitoring:

# Monitor logs in real-time (open in separate terminal)
docker-compose logs -f --tail=0

# Generate some traffic to see logs
for i in {1..20}; do
  curl -s http://localhost/ > /dev/null
  curl -s http://localhost/health > /dev/null
  sleep 2
done
Task 6: Advanced Service Management and Troubleshooting
Step 6.1: Service Health Monitoring
Check service health status:

# Check health status of all services
docker-compose ps

# Inspect individual container health
docker inspect --format='{{.State.Health.Status}}' webapp
docker inspect --format='{{.State.Health.Status}}' postgres_db
docker inspect --format='{{.State.Health.Status}}' redis_cache

# View health check logs
docker inspect --format='{{range .State.Health.Log}}{{.Output}}{{end}}' webapp
Step 6.2: Resource Usage Monitoring
Monitor resource consumption:

# View resource usage statistics
docker stats

# View resource usage for specific containers
docker stats webapp postgres_db redis_cache nginx_proxy

# Check disk usage
docker system df
docker system df -v
Step 6.3: Troubleshooting Common Issues
Practice troubleshooting techniques:

# Check service dependencies
docker-compose config --services

# Restart a specific service
docker-compose restart web

# Rebuild a service
docker-compose up -d --build web

# View detailed container information
docker-compose exec web env
docker-compose exec db psql -U postgres -d webapp -c "\dt"
docker-compose exec redis redis-cli info
Step 6.4: Database Operations
Perform database operations:

# Connect to PostgreSQL
docker-compose exec db psql -U postgres -d webapp

# Run SQL commands (inside psql)
# \dt                    -- List tables
# SELECT * FROM users;   -- View users
# \q                     -- Quit

# Backup database
docker-compose exec db pg_dump -U postgres webapp > backup.sql

# View Redis data
docker-compose exec redis redis-cli
# keys *                 -- List all keys
# get page_views         -- Get page views count
# exit                   -- Exit Redis CLI
Task 7: Performance Testing and Optimization
Step 7.1: Load Testing
Test application performance:

# Install Apache Bench (if not available)
sudo apt-get update && sudo apt-get install -y apache2-utils

# Perform load testing
ab -n 1000 -c 10 http://localhost/

# Test specific endpoints
ab -n 500 -c 5 http://localhost/users
ab -n 500 -c 5 http://localhost/health
Step 7.2: Scale Testing
Test scaling capabilities:

# Scale to different numbers of web instances
docker-compose up -d --scale web=2
ab -n 1000 -c 10 http://localhost/

docker-compose up -d --scale web=4
ab -n 1000 -c 10 http://localhost/

# Monitor resource usage during scaling
docker stats --no-stream
Step 7.3: Optimize Configuration
Create an optimized production configuration:

nano docker-compose.prod.yml
Add production optimizations:

version: '3.8'

services:
  web:
    build: 
      context: ./app
      dockerfile: Dockerfile
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    environment:
      - DB_HOST=db
      - DB_NAME=webapp
      - DB_USER=postgres
      - DB_PASS=securepassword123
      - REDIS_HOST=redis
      - FLASK_ENV=production
    depends_on:
      - db
      - redis
    networks:
      - app-network
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=webapp
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=securepassword123
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-db.sql:/docker-entrypoint-initdb.d/init-db.sql:ro
    networks:
      - app-network
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '1'
          memory: 1G
        reservations:
          cpus: '0.5'
          memory: 512M

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf:ro
    networks:
      - app-network
    restart: unless-stopped
    command: redis-server /usr/local/etc/redis/redis.conf
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
        reservations:
          cpus: '0.25'
          memory: 128M

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - web
    networks:
      - app-network
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.25'
          memory: 128M

networks:
  app-network:
    driver: bridge

volumes:
  postgres_data:
  redis_data:
Task 8: Cleanup and Resource Management
Step 8.1: Graceful Shutdown
Stop services properly:

# Stop all services gracefully
docker-compose down

# Stop services and remove volumes (careful!)
docker-compose down -v

# Stop services and remove images
docker-compose down --rmi all
Step 8.2: Resource Cleanup
Clean up Docker resources:

# Remove unused containers, networks, images
docker system prune

# Remove unused volumes
docker volume prune

# Remove everything (use with caution)
docker system prune -a --volumes
Step 8.3: Backup Important Data
Before cleanup, backup important data:

# Backup database
docker-compose exec db pg_dump -U postgres webapp > webapp_backup.sql

# Backup Redis data
docker-compose exec redis redis-cli --rdb /data/dump.rdb

# Copy volume data
docker run --rm -v docker-compose-lab_postgres_data:/data -v $(pwd):/backup alpine tar czf /backup/postgres_backup.tar.gz -C /data .
Troubleshooting Guide
Common Issues and Solutions
Issue 1: Services fail to start

# Check logs for errors
docker-compose logs

# Verify port availability
netstat -tulpn | grep :5000
netstat -tulpn | grep :5432

# Check Docker daemon status
sudo systemctl status docker
Issue 2: Database connection errors

# Verify database is running
docker-compose ps db

# Check database logs
docker-compose logs db

# Test database connectivity
docker-compose exec web ping db
Issue 3: Redis connection issues

# Check Redis status
docker-compose exec redis redis-cli ping

# Verify Redis configuration
docker-compose exec redis cat /usr/local/etc/redis/redis.conf
Issue 4: Network connectivity problems

# Inspect network configuration
docker network inspect docker-compose-lab_app-network

# Test inter-service communication
docker-compose exec web nslookup db
docker-compose exec web nslookup redis
Issue 5: Volume mounting issues

# Check volume permissions
docker-compose exec web ls -la /app

# Verify volume mounts
docker inspect webapp | grep -A 10 "Mounts"
Best Practices Learned
Security Best Practices
• Use non-root users in containers • Implement proper secret management • Configure network segmentation • Enable health checks for all services • Use specific image tags instead of 'latest'

Performance Best Practices
• Implement resource limits and reservations • Use multi-stage builds for smaller images • Configure proper caching strategies • Implement connection pooling • Monitor resource usage regularly

Operational Best Practices
• Use meaningful container and service names • Implement comprehensive logging • Configure proper restart policies • Use volumes for persistent data • Implement proper backup strategies

Conclusion
Congratulations! You have successfully completed Lab 25: Docker Compose for Multi-Container Web App. In this comprehensive lab, you have accomplished the following:

Key Achievements
Multi-Container Orchestration: You created and deployed a complete web application stack consisting of a Flask web application, PostgreSQL database, Redis cache, and Nginx reverse proxy, all orchestrated using Docker Compose.

Infrastructure as Code: You learned to define complex application architectures using declarative YAML configuration, making your infrastructure reproducible and version-controlled.

Service Networking: You configured custom networks to enable secure communication between services while maintaining proper isolation from external networks.

Data Persistence: You implemented persistent storage using Docker volumes, ensuring data survives container restarts and updates.

Service Scaling: You demonstrated horizontal scaling capabilities by running multiple instances of the web service and load balancing traffic across them.

Monitoring and Troubleshooting: You gained hands-on experience with log monitoring, health checks, and troubleshooting techniques essential for production environments.

Performance Optimization: You learned to monitor resource usage, conduct load testing, and optimize configurations for production deployments.

Real-World Applications
The skills you've developed in this lab are directly applicable to:

• Microservices Architecture: Understanding how to decompose applications into manageable, scalable services • DevOps Practices: Implementing infrastructure as code and automated deployment pipelines • Production Operations: Managing complex applications with proper monitoring, scaling, and maintenance procedures • Cloud Migration: Preparing applications for containerized deployment in cloud environments • Development Workflows: Creating consistent development environments that mirror production

Next Steps
To further advance your Docker and containerization skills, consider:

• Exploring Docker Swarm for production orchestration • Learning Kubernetes for enterprise-scale container management • Implementing CI/CD pipelines with Docker • Studying container security best practices • Exploring service mesh technologies like Istio

Certification Relevance
This lab directly supports your preparation for the Docker Certified Associate (DCA) certification by covering:

• Docker Compose fundamentals and advanced features • Multi-container application deployment • Network and volume configuration • Service scaling and load balancing • Troubleshooting and monitoring techniques • Production deployment considerations

The hands-on experience gained through this lab provides practical knowledge that extends beyond certification requirements, preparing you for real-world Docker implementations in professional environments.

Well done! You now have the foundational skills to design, deploy, and manage sophisticated multi-container applications using Docker Compose, a critical capability in modern software development and operations.
