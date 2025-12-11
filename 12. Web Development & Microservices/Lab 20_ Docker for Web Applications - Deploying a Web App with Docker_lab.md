Lab 20: Docker for Web Applications - Deploying a Web App with Docker
Lab Objectives
By the end of this lab, students will be able to:

Create a Dockerfile for a Python Flask web application
Build Docker images and run containers for web applications
Configure port mapping to expose web services
Connect web applications to database containers
Use Docker Compose to manage multi-container applications
Deploy and test web applications in containerized environments
Understand the fundamentals of containerized web application deployment
Prerequisites
Before starting this lab, students should have:

Basic understanding of web applications and HTTP protocols
Familiarity with command-line interface operations
Basic knowledge of Python programming (helpful but not required)
Understanding of basic networking concepts (ports, IP addresses)
Completion of previous Docker fundamentals labs or equivalent knowledge
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker already installed. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your lab environment includes:

Ubuntu Linux machine with Docker Engine installed
Docker Compose pre-installed
Text editor (nano/vim) available
All necessary networking configurations
Task 1: Create a Simple Flask Web Application
Subtask 1.1: Set Up Project Directory Structure
First, let's create a organized directory structure for our web application project.

# Create main project directory
mkdir flask-docker-app
cd flask-docker-app

# Create subdirectories for organization
mkdir app
mkdir database
Subtask 1.2: Create the Flask Application
Create a simple Flask web application that we'll containerize.

# Create the main Flask application file
nano app/app.py
Add the following Python code to create a basic web application:

from flask import Flask, render_template, request, jsonify
import os
import sqlite3
from datetime import datetime

app = Flask(__name__)

# Database configuration
DATABASE = '/app/data/visitors.db'

def init_db():
    """Initialize the database with a visitors table"""
    os.makedirs('/app/data', exist_ok=True)
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS visitors (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            ip_address TEXT,
            visit_time TIMESTAMP,
            user_agent TEXT
        )
    ''')
    conn.commit()
    conn.close()

def log_visitor(ip_address, user_agent):
    """Log visitor information to database"""
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO visitors (ip_address, visit_time, user_agent)
        VALUES (?, ?, ?)
    ''', (ip_address, datetime.now(), user_agent))
    conn.commit()
    conn.close()

def get_visitor_count():
    """Get total number of visitors"""
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('SELECT COUNT(*) FROM visitors')
    count = cursor.fetchone()[0]
    conn.close()
    return count

@app.route('/')
def home():
    """Main page route"""
    # Log the visitor
    ip_address = request.environ.get('HTTP_X_FORWARDED_FOR', request.remote_addr)
    user_agent = request.environ.get('HTTP_USER_AGENT', 'Unknown')
    log_visitor(ip_address, user_agent)
    
    # Get visitor count
    visitor_count = get_visitor_count()
    
    return f'''
    <html>
    <head>
        <title>Flask Docker App</title>
        <style>
            body {{ font-family: Arial, sans-serif; margin: 40px; background-color: #f0f0f0; }}
            .container {{ background-color: white; padding: 30px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }}
            h1 {{ color: #333; }}
            .info {{ background-color: #e7f3ff; padding: 15px; border-radius: 5px; margin: 20px 0; }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Welcome to Flask Docker Application!</h1>
            <div class="info">
                <p><strong>Application Status:</strong> Running in Docker Container</p>
                <p><strong>Total Visitors:</strong> {visitor_count}</p>
                <p><strong>Your IP:</strong> {ip_address}</p>
            </div>
            <p>This is a simple Flask web application running inside a Docker container.</p>
            <p>The application demonstrates:</p>
            <ul>
                <li>Web application containerization</li>
                <li>Database integration with SQLite</li>
                <li>Visitor tracking functionality</li>
                <li>Port mapping and networking</li>
            </ul>
        </div>
    </body>
    </html>
    '''

@app.route('/health')
def health_check():
    """Health check endpoint"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.now().isoformat(),
        'visitors': get_visitor_count()
    })

@app.route('/visitors')
def visitors():
    """Display recent visitors"""
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('SELECT * FROM visitors ORDER BY visit_time DESC LIMIT 10')
    recent_visitors = cursor.fetchall()
    conn.close()
    
    html = '<html><head><title>Recent Visitors</title></head><body>'
    html += '<h1>Recent Visitors</h1>'
    html += '<table border="1"><tr><th>ID</th><th>IP Address</th><th>Visit Time</th><th>User Agent</th></tr>'
    
    for visitor in recent_visitors:
        html += f'<tr><td>{visitor[0]}</td><td>{visitor[1]}</td><td>{visitor[2]}</td><td>{visitor[3][:50]}...</td></tr>'
    
    html += '</table><br><a href="/">Back to Home</a></body></html>'
    return html

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000, debug=True)
Subtask 1.3: Create Requirements File
Create a requirements file to specify Python dependencies:

nano app/requirements.txt
Add the following content:

Flask==2.3.3
Werkzeug==2.3.7
Task 2: Create Dockerfile for the Flask Application
Subtask 2.1: Create the Dockerfile
Now let's create a Dockerfile to containerize our Flask application:

nano Dockerfile
Add the following Dockerfile content:

# Use official Python runtime as base image
FROM python:3.9-slim

# Set working directory in container
WORKDIR /app

# Set environment variables
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements file
COPY app/requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ .

# Create directory for database
RUN mkdir -p /app/data

# Expose port 5000
EXPOSE 5000

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Create non-root user for security
RUN adduser --disabled-password --gecos '' appuser && \
    chown -R appuser:appuser /app
USER appuser

# Command to run the application
CMD ["python", "app.py"]
Subtask 2.2: Create .dockerignore File
Create a .dockerignore file to exclude unnecessary files:

nano .dockerignore
Add the following content:

__pycache__
*.pyc
*.pyo
*.pyd
.Python
env/
venv/
.venv/
pip-log.txt
pip-delete-this-directory.txt
.tox
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
*.log
.git
.mypy_cache
.pytest_cache
.hypothesis
.DS_Store
Task 3: Build and Run the Docker Container
Subtask 3.1: Build the Docker Image
Build the Docker image for our Flask application:

# Build the Docker image with a tag
docker build -t flask-web-app:v1.0 .

# Verify the image was created
docker images | grep flask-web-app
Subtask 3.2: Run the Container with Port Mapping
Run the container and map ports to make the web application accessible:

# Run the container with port mapping
docker run -d \
  --name flask-app-container \
  -p 8080:5000 \
  -v flask-app-data:/app/data \
  flask-web-app:v1.0

# Verify the container is running
docker ps
Subtask 3.3: Test the Web Application
Test the web application to ensure it's working correctly:

# Test the main page
curl http://localhost:8080

# Test the health check endpoint
curl http://localhost:8080/health

# Check container logs
docker logs flask-app-container
Task 4: Add Database Container with Docker Compose
Subtask 4.1: Create Docker Compose Configuration
Create a Docker Compose file to manage multiple containers:

nano docker-compose.yml
Add the following Docker Compose configuration:

version: '3.8'

services:
  web:
    build: .
    container_name: flask-web-app
    ports:
      - "8080:5000"
    volumes:
      - app-data:/app/data
    environment:
      - FLASK_ENV=production
      - DATABASE_URL=sqlite:///app/data/visitors.db
    depends_on:
      - redis
    networks:
      - app-network
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    container_name: flask-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - app-network
    restart: unless-stopped
    command: redis-server --appendonly yes

  nginx:
    image: nginx:alpine
    container_name: flask-nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web
    networks:
      - app-network
    restart: unless-stopped

volumes:
  app-data:
    driver: local
  redis-data:
    driver: local

networks:
  app-network:
    driver: bridge
Subtask 4.2: Create Nginx Configuration
Create an Nginx configuration file for load balancing:

nano nginx.conf
Add the following Nginx configuration:

events {
    worker_connections 1024;
}

http {
    upstream flask_app {
        server web:5000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://flask_app;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /health {
            proxy_pass http://flask_app/health;
            access_log off;
        }
    }
}
Subtask 4.3: Enhanced Flask Application with Redis
Update the Flask application to use Redis for session management:

nano app/app_enhanced.py
Add the following enhanced application code:

from flask import Flask, render_template, request, jsonify, session
import os
import sqlite3
import redis
from datetime import datetime
import json

app = Flask(__name__)
app.secret_key = 'your-secret-key-change-in-production'

# Database configuration
DATABASE = '/app/data/visitors.db'

# Redis configuration
try:
    redis_client = redis.Redis(host='redis', port=6379, db=0, decode_responses=True)
    redis_client.ping()
    REDIS_AVAILABLE = True
except:
    REDIS_AVAILABLE = False

def init_db():
    """Initialize the database with a visitors table"""
    os.makedirs('/app/data', exist_ok=True)
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS visitors (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            ip_address TEXT,
            visit_time TIMESTAMP,
            user_agent TEXT,
            session_id TEXT
        )
    ''')
    conn.commit()
    conn.close()

def log_visitor(ip_address, user_agent, session_id):
    """Log visitor information to database"""
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('''
        INSERT INTO visitors (ip_address, visit_time, user_agent, session_id)
        VALUES (?, ?, ?, ?)
    ''', (ip_address, datetime.now(), user_agent, session_id))
    conn.commit()
    conn.close()

def get_visitor_count():
    """Get total number of visitors"""
    conn = sqlite3.connect(DATABASE)
    cursor = conn.cursor()
    cursor.execute('SELECT COUNT(*) FROM visitors')
    count = cursor.fetchone()[0]
    conn.close()
    return count

def increment_page_views():
    """Increment page views counter in Redis"""
    if REDIS_AVAILABLE:
        try:
            return redis_client.incr('page_views')
        except:
            return 0
    return 0

@app.route('/')
def home():
    """Main page route"""
    # Generate session ID if not exists
    if 'session_id' not in session:
        session['session_id'] = os.urandom(16).hex()
    
    # Log the visitor
    ip_address = request.environ.get('HTTP_X_FORWARDED_FOR', request.remote_addr)
    user_agent = request.environ.get('HTTP_USER_AGENT', 'Unknown')
    log_visitor(ip_address, user_agent, session['session_id'])
    
    # Get statistics
    visitor_count = get_visitor_count()
    page_views = increment_page_views()
    
    return f'''
    <html>
    <head>
        <title>Enhanced Flask Docker App</title>
        <style>
            body {{ font-family: Arial, sans-serif; margin: 40px; background-color: #f0f0f0; }}
            .container {{ background-color: white; padding: 30px; border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }}
            h1 {{ color: #333; }}
            .info {{ background-color: #e7f3ff; padding: 15px; border-radius: 5px; margin: 20px 0; }}
            .status {{ background-color: #d4edda; padding: 10px; border-radius: 5px; margin: 10px 0; }}
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Enhanced Flask Docker Application!</h1>
            <div class="info">
                <p><strong>Application Status:</strong> Running in Docker Container</p>
                <p><strong>Total Visitors:</strong> {visitor_count}</p>
                <p><strong>Page Views:</strong> {page_views}</p>
                <p><strong>Your IP:</strong> {ip_address}</p>
                <p><strong>Session ID:</strong> {session['session_id'][:8]}...</p>
            </div>
            <div class="status">
                <p><strong>Redis Status:</strong> {'Connected' if REDIS_AVAILABLE else 'Not Available'}</p>
                <p><strong>Database:</strong> SQLite Connected</p>
            </div>
            <p>This enhanced Flask application demonstrates:</p>
            <ul>
                <li>Multi-container deployment with Docker Compose</li>
                <li>Redis integration for session management</li>
                <li>Nginx reverse proxy configuration</li>
                <li>Persistent data storage</li>
                <li>Container networking</li>
            </ul>
            <p><a href="/visitors">View Recent Visitors</a> | <a href="/stats">View Statistics</a></p>
        </div>
    </body>
    </html>
    '''

@app.route('/health')
def health_check():
    """Health check endpoint"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.now().isoformat(),
        'visitors': get_visitor_count(),
        'redis_available': REDIS_AVAILABLE,
        'page_views': increment_page_views() if REDIS_AVAILABLE else 0
    })

@app.route('/stats')
def stats():
    """Statistics page"""
    visitor_count = get_visitor_count()
    page_views = redis_client.get('page_views') if REDIS_AVAILABLE else 0
    
    return jsonify({
        'total_visitors': visitor_count,
        'total_page_views': int(page_views) if page_views else 0,
        'redis_status': 'connected' if REDIS_AVAILABLE else 'disconnected',
        'timestamp': datetime.now().isoformat()
    })

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000, debug=False)
Subtask 4.4: Update Requirements for Enhanced App
Update the requirements file to include Redis support:

nano app/requirements.txt
Update with the following content:

Flask==2.3.3
Werkzeug==2.3.7
redis==5.0.1
Task 5: Deploy with Docker Compose
Subtask 5.1: Stop Single Container
First, stop the single container we ran earlier:

# Stop and remove the single container
docker stop flask-app-container
docker rm flask-app-container
Subtask 5.2: Update Dockerfile for Enhanced App
Update the Dockerfile to use the enhanced application:

nano Dockerfile
Update the CMD line at the end:

# Command to run the enhanced application
CMD ["python", "app_enhanced.py"]
Subtask 5.3: Deploy Multi-Container Application
Deploy the complete application stack using Docker Compose:

# Build and start all services
docker-compose up -d --build

# Verify all containers are running
docker-compose ps

# Check logs for all services
docker-compose logs
Subtask 5.4: Test the Multi-Container Application
Test the enhanced application with multiple services:

# Test through Nginx (port 80)
curl http://localhost/

# Test direct Flask app (port 8080)
curl http://localhost:8080/

# Test health endpoint
curl http://localhost/health

# Test statistics endpoint
curl http://localhost/stats

# Check Redis connectivity
docker-compose exec redis redis-cli ping
Task 6: Application Testing and Monitoring
Subtask 6.1: Load Testing
Create a simple load test to verify application performance:

# Create a simple load test script
nano load_test.sh
Add the following script:

#!/bin/bash

echo "Starting load test..."
echo "Testing main page..."

for i in {1..10}; do
    echo "Request $i:"
    curl -s -w "Time: %{time_total}s, Status: %{http_code}\n" http://localhost/ > /dev/null
    sleep 1
done

echo "Testing health endpoint..."
for i in {1..5}; do
    echo "Health check $i:"
    curl -s http://localhost/health | jq '.status'
    sleep 1
done

echo "Load test completed!"
Make the script executable and run it:

chmod +x load_test.sh
./load_test.sh
Subtask 6.2: Monitor Container Resources
Monitor container resource usage:

# Check container resource usage
docker stats

# Check specific container logs
docker-compose logs web
docker-compose logs redis
docker-compose logs nginx

# Check container health
docker-compose exec web curl http://localhost:5000/health
Subtask 6.3: Database Verification
Verify that data is being stored correctly:

# Access the web container
docker-compose exec web /bin/bash

# Inside the container, check the database
python3 -c "
import sqlite3
conn = sqlite3.connect('/app/data/visitors.db')
cursor = conn.cursor()
cursor.execute('SELECT COUNT(*) FROM visitors')
print('Total visitors:', cursor.fetchone()[0])
cursor.execute('SELECT * FROM visitors ORDER BY visit_time DESC LIMIT 5')
print('Recent visitors:')
for row in cursor.fetchall():
    print(row)
conn.close()
"

# Exit the container
exit
Task 7: Scaling and Management
Subtask 7.1: Scale the Web Application
Scale the web application to multiple instances:

# Scale the web service to 3 instances
docker-compose up -d --scale web=3

# Verify scaling
docker-compose ps

# Test load balancing
for i in {1..10}; do
    curl -s http://localhost/ | grep "Session ID" | head -1
    sleep 1
done
Subtask 7.2: Update Application Configuration
Create an environment configuration file:

nano .env
Add the following environment variables:

FLASK_ENV=production
FLASK_DEBUG=False
SECRET_KEY=your-production-secret-key-here
REDIS_URL=redis://redis:6379/0
DATABASE_PATH=/app/data/visitors.db
Update docker-compose.yml to use environment file:

nano docker-compose.yml
Add the env_file directive to the web service:

  web:
    build: .
    container_name: flask-web-app
    ports:
      - "8080:5000"
    volumes:
      - app-data:/app/data
    env_file:
      - .env
    depends_on:
      - redis
    networks:
      - app-network
    restart: unless-stopped
Subtask 7.3: Backup and Restore Data
Create backup and restore procedures:

# Create backup script
nano backup.sh
Add the following backup script:

#!/bin/bash

BACKUP_DIR="./backups"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

# Create backup directory
mkdir -p $BACKUP_DIR

# Backup application data
docker run --rm \
  -v flask-docker-app_app-data:/data \
  -v $(pwd)/$BACKUP_DIR:/backup \
  alpine tar czf /backup/app-data-$TIMESTAMP.tar.gz -C /data .

# Backup Redis data
docker run --rm \
  -v flask-docker-app_redis-data:/data \
  -v $(pwd)/$BACKUP_DIR:/backup \
  alpine tar czf /backup/redis-data-$TIMESTAMP.tar.gz -C /data .

echo "Backup completed: $TIMESTAMP"
ls -la $BACKUP_DIR/
Make executable and run:

chmod +x backup.sh
./backup.sh
Troubleshooting Common Issues
Issue 1: Container Won't Start
Problem: Container fails to start or exits immediately.

Solution:

# Check container logs
docker-compose logs web

# Check if ports are already in use
netstat -tulpn | grep :8080

# Rebuild containers
docker-compose down
docker-compose up --build -d
Issue 2: Database Connection Issues
Problem: Application can't connect to database.

Solution:

# Check if data directory exists and has correct permissions
docker-compose exec web ls -la /app/data/

# Recreate the database
docker-compose exec web python3 -c "
from app_enhanced import init_db
init_db()
print('Database initialized')
"
Issue 3: Redis Connection Problems
Problem: Redis service is not accessible.

Solution:

# Check Redis container status
docker-compose ps redis

# Test Redis connectivity
docker-compose exec redis redis-cli ping

# Check network connectivity
docker-compose exec web ping redis
Issue 4: Port Conflicts
Problem: Ports are already in use.

Solution:

# Find processes using the ports
sudo lsof -i :80
sudo lsof -i :8080

# Change ports in docker-compose.yml
# For example, change "80:80" to "8081:80"
Performance Optimization Tips
Tip 1: Optimize Docker Images
# Use multi-stage builds for smaller images
nano Dockerfile.optimized
Add optimized Dockerfile:

# Multi-stage build for optimization
FROM python:3.9-slim as builder

WORKDIR /app
COPY app/requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.9-slim

WORKDIR /app
COPY --from=builder /root/.local /root/.local
COPY app/ .

ENV PATH=/root/.local/bin:$PATH
EXPOSE 5000
CMD ["python", "app_enhanced.py"]
Tip 2: Configure Resource Limits
Update docker-compose.yml with resource limits:

  web:
    build: .
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
Conclusion
Congratulations! You have successfully completed Lab 20: Docker for Web Applications. In this comprehensive lab, you have accomplished the following:

Key Achievements
Created a Containerized Web Application: You built a complete Flask web application from scratch and successfully containerized it using Docker, demonstrating the fundamental principles of application containerization.

Mastered Docker Fundamentals: You learned to create effective Dockerfiles, build optimized images, and run containers with proper port mapping and volume management.

Implemented Multi-Container Architecture: You deployed a sophisticated multi-container application using Docker Compose, integrating a web application, Redis cache, and Nginx reverse proxy.

Configured Container Networking: You established secure communication between containers using Docker networks, enabling services to interact seamlessly.

Implemented Data Persistence: You configured persistent storage for both application data and Redis cache, ensuring data survives container restarts.

Applied Production Best Practices: You implemented health checks, resource limits, environment configuration, and backup procedures suitable for production environments.

Why This Matters
The skills you've developed in this lab are crucial for modern software development and deployment:

Industry Relevance: Container technologies like Docker are standard in modern DevOps practices, used by companies of all sizes for application deployment and scaling.

Scalability: The multi-container architecture you've implemented can be easily scaled horizontally to handle increased traffic and load.

Portability: Your containerized application can run consistently across different environments, from development laptops to production servers.

Efficiency: Container-based deployment reduces resource overhead compared to traditional virtual machines while providing isolation and security.

Next Steps
To further enhance your Docker and containerization skills, consider:

Explore Kubernetes: Learn container orchestration at scale with Kubernetes
Implement CI/CD Pipelines: Integrate Docker builds into automated deployment pipelines
Study Security Best Practices: Learn about container security, image scanning, and secure deployment practices
Practice with Different Applications: Containerize various types of applications (databases, microservices, etc.)
Certification Preparation
This lab directly supports your preparation for the Docker Certified Associate (DCA) certification by covering:

Container lifecycle management
Docker Compose and multi-container applications
Networking and storage configuration
Security and best practices
You now have hands-on experience with real-world Docker scenarios that are commonly tested in the DCA certification exam.

The containerization skills you've mastered today form the foundation for modern application deployment, microservices architecture, and cloud-native development practices that are essential in today's technology landscape.
