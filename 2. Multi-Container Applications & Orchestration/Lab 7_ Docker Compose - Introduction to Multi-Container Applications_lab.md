Lab 7: Docker Compose - Introduction to Multi-Container Applications
Objectives
By the end of this lab, students will be able to:

Understand the purpose and benefits of Docker Compose for multi-container applications
Install and configure Docker Compose on a Linux system
Create and configure docker-compose.yml files for multi-service applications
Deploy and manage multi-container applications using Docker Compose commands
Scale services dynamically using Docker Compose
Properly stop and clean up multi-container applications
Troubleshoot common Docker Compose issues
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and images
Familiarity with Linux command line operations
Knowledge of YAML file format basics
Understanding of web applications and databases concepts
Completion of previous Docker labs (recommended)
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your dedicated environment. No need to build your own virtual machine or install Docker manually - everything is ready to use.

Your cloud machine includes:

Ubuntu 20.04 LTS or later
Docker Engine pre-installed
All necessary development tools
Internet connectivity for downloading images
What is Docker Compose?
Docker Compose is a tool for defining and running multi-container Docker applications. Instead of managing multiple containers individually, Compose allows you to define your entire application stack in a single YAML file and manage it with simple commands.

Key Benefits:

Simplified Management: Control multiple containers with single commands
Environment Consistency: Ensure same setup across development, testing, and production
Service Dependencies: Define how services depend on each other
Easy Scaling: Scale services up or down with simple commands
Task 1: Install Docker Compose
Step 1.1: Verify Docker Installation
First, let's confirm Docker is installed and running on your system.

# Check Docker version
docker --version

# Check Docker service status
sudo systemctl status docker

# Test Docker with hello-world
docker run hello-world
Step 1.2: Install Docker Compose
Docker Compose comes pre-installed with Docker Desktop, but on Linux servers, we need to install it separately.

# Download the latest stable release of Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose

# Make the binary executable
sudo chmod +x /usr/local/bin/docker-compose

# Create a symbolic link (optional, for easier access)
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
Step 1.3: Verify Docker Compose Installation
# Check Docker Compose version
docker-compose --version

# Display help information
docker-compose --help
Expected Output:

Docker Compose version v2.x.x
Task 2: Create a Docker Compose File for Multi-Service Application
We'll create a simple web application with two services: a web server (Nginx) and a database (MySQL).

Step 2.1: Create Project Directory Structure
# Create project directory
mkdir ~/docker-compose-lab
cd ~/docker-compose-lab

# Create subdirectories for organization
mkdir -p web/html
mkdir -p database/init
Step 2.2: Create a Simple Web Page
Create a basic HTML file for our web server:

# Create index.html file
cat > web/html/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Docker Compose Lab</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 50px auto;
            padding: 20px;
            background-color: #f4f4f4;
        }
        .container {
            background-color: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 0 10px rgba(0,0,0,0.1);
        }
        h1 {
            color: #333;
            text-align: center;
        }
        .info {
            background-color: #e7f3ff;
            padding: 15px;
            border-left: 4px solid #2196F3;
            margin: 20px 0;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Welcome to Docker Compose Lab!</h1>
        <div class="info">
            <h3>Multi-Container Application</h3>
            <p>This web page is served by an Nginx container, which is part of a multi-container application managed by Docker Compose.</p>
            <p><strong>Services Running:</strong></p>
            <ul>
                <li>Web Server: Nginx (Port 8080)</li>
                <li>Database: MySQL (Port 3306)</li>
            </ul>
        </div>
        <p>Congratulations! You have successfully deployed a multi-container application using Docker Compose.</p>
    </div>
</body>
</html>
EOF
Step 2.3: Create Database Initialization Script
# Create database initialization script
cat > database/init/init.sql << 'EOF'
-- Create a sample database
CREATE DATABASE IF NOT EXISTS sampleapp;

-- Use the database
USE sampleapp;

-- Create a users table
CREATE TABLE IF NOT EXISTS users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO users (username, email) VALUES 
('john_doe', 'john@example.com'),
('jane_smith', 'jane@example.com'),
('docker_user', 'docker@compose.com');

-- Create a simple view
CREATE VIEW user_count AS 
SELECT COUNT(*) as total_users FROM users;
EOF
Step 2.4: Create Docker Compose Configuration File
Now, let's create the main docker-compose.yml file:

# Create docker-compose.yml file
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  # Web Server Service
  webserver:
    image: nginx:alpine
    container_name: compose-webserver
    ports:
      - "8080:80"
    volumes:
      - ./web/html:/usr/share/nginx/html:ro
    depends_on:
      - database
    networks:
      - app-network
    restart: unless-stopped

  # Database Service
  database:
    image: mysql:8.0
    container_name: compose-database
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword123
      MYSQL_DATABASE: sampleapp
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword123
    ports:
      - "3306:3306"
    volumes:
      - ./database/init:/docker-entrypoint-initdb.d:ro
      - mysql-data:/var/lib/mysql
    networks:
      - app-network
    restart: unless-stopped

# Define named volumes
volumes:
  mysql-data:
    driver: local

# Define custom networks
networks:
  app-network:
    driver: bridge
EOF
Step 2.5: Understanding the Docker Compose File
Let's break down the key components:

Version: Specifies the Compose file format version Services: Defines the containers that make up your application Volumes: Persistent data storage Networks: Custom networks for service communication

Key Configuration Elements:

ports: Maps host ports to container ports
volumes: Mounts host directories or named volumes
environment: Sets environment variables
depends_on: Defines service dependencies
restart: Container restart policy
Task 3: Start the Multi-Container Application
Step 3.1: Launch the Application Stack
# Start all services defined in docker-compose.yml
docker-compose up -d

# The -d flag runs containers in detached mode (background)
Step 3.2: Monitor the Startup Process
# View logs from all services
docker-compose logs

# View logs from a specific service
docker-compose logs webserver
docker-compose logs database

# Follow logs in real-time
docker-compose logs -f
Step 3.3: Verify Services are Running
# List running containers
docker-compose ps

# Check container status
docker ps

# Verify network connectivity
docker network ls
Expected Output:

NAME                   COMMAND                  SERVICE             STATUS              PORTS
compose-database       "docker-entrypoint.s…"   database            running             0.0.0.0:3306->3306/tcp, 33060/tcp
compose-webserver      "/docker-entrypoint.…"   webserver           running             0.0.0.0:8080->80/tcp
Step 3.4: Test the Application
# Test web server response
curl http://localhost:8080

# Test database connectivity (from within the database container)
docker-compose exec database mysql -u appuser -papppassword123 -e "SELECT * FROM sampleapp.users;"
Access the Web Application: Open your web browser and navigate to http://localhost:8080 to see your web page.

Task 4: Scale Services Using Docker Compose
Step 4.1: Scale the Web Server Service
# Scale webserver to 3 instances
docker-compose up -d --scale webserver=3

# Note: You'll need to modify the compose file to remove container_name 
# and use port ranges for scaling to work properly
Step 4.2: Create a Scalable Version
Let's create a modified version that supports scaling:

# Create a scalable version of docker-compose.yml
cat > docker-compose-scalable.yml << 'EOF'
version: '3.8'

services:
  # Load Balancer
  loadbalancer:
    image: nginx:alpine
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - webserver
    networks:
      - app-network

  # Web Server Service (scalable)
  webserver:
    image: nginx:alpine
    volumes:
      - ./web/html:/usr/share/nginx/html:ro
    depends_on:
      - database
    networks:
      - app-network
    restart: unless-stopped

  # Database Service
  database:
    image: mysql:8.0
    container_name: compose-database
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword123
      MYSQL_DATABASE: sampleapp
      MYSQL_USER: appuser
      MYSQL_PASSWORD: apppassword123
    volumes:
      - ./database/init:/docker-entrypoint-initdb.d:ro
      - mysql-data:/var/lib/mysql
    networks:
      - app-network
    restart: unless-stopped

volumes:
  mysql-data:
    driver: local

networks:
  app-network:
    driver: bridge
EOF
Step 4.3: Create Load Balancer Configuration
# Create nginx load balancer configuration
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream webservers {
        server webserver:80;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://webservers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
}
EOF
Step 4.4: Scale with the New Configuration
# Stop current services
docker-compose down

# Start with scalable configuration
docker-compose -f docker-compose-scalable.yml up -d

# Scale webserver to 3 instances
docker-compose -f docker-compose-scalable.yml up -d --scale webserver=3

# Verify scaling
docker-compose -f docker-compose-scalable.yml ps
Task 5: Stop and Remove Services
Step 5.1: Stop Services Gracefully
# Stop all services (containers remain)
docker-compose stop

# Verify services are stopped
docker-compose ps
Step 5.2: Start Stopped Services
# Restart stopped services
docker-compose start

# Or restart specific services
docker-compose start webserver
Step 5.3: Remove Services and Resources
# Stop and remove containers, networks
docker-compose down

# Stop and remove containers, networks, and volumes
docker-compose down -v

# Stop and remove everything including images
docker-compose down -v --rmi all
Step 5.4: Clean Up System Resources
# Remove unused containers, networks, images
docker system prune -f

# Remove unused volumes
docker volume prune -f

# View disk usage
docker system df
Advanced Docker Compose Operations
Viewing Service Information
# View service configuration
docker-compose config

# View service logs with timestamps
docker-compose logs -t

# Execute commands in running containers
docker-compose exec webserver sh
docker-compose exec database mysql -u root -p
Environment-Specific Configurations
# Create environment-specific override file
cat > docker-compose.override.yml << 'EOF'
version: '3.8'

services:
  webserver:
    environment:
      - ENV=development
    ports:
      - "8081:80"
  
  database:
    environment:
      - MYSQL_ROOT_PASSWORD=devpassword123
EOF
Troubleshooting Common Issues
Issue 1: Port Already in Use
Problem: Error binding to port 8080 Solution:

# Check what's using the port
sudo netstat -tulpn | grep :8080

# Kill the process or change the port in docker-compose.yml
Issue 2: Database Connection Issues
Problem: Web server can't connect to database Solution:

# Check if database is ready
docker-compose logs database

# Test network connectivity
docker-compose exec webserver ping database
Issue 3: Volume Mount Issues
Problem: Files not appearing in container Solution:

# Check file permissions
ls -la web/html/

# Verify volume mounts
docker-compose exec webserver ls -la /usr/share/nginx/html/
Issue 4: Service Dependencies
Problem: Services starting in wrong order Solution:

# Use depends_on and health checks
# Add to docker-compose.yml:
healthcheck:
  test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
  timeout: 20s
  retries: 10
Best Practices for Docker Compose
1. File Organization
Keep docker-compose.yml in project root
Use separate directories for service-specific files
Use environment-specific override files
2. Security Considerations
# Use secrets for sensitive data
secrets:
  db_password:
    file: ./secrets/db_password.txt

# In service definition:
secrets:
  - db_password
3. Resource Management
# Add resource limits
deploy:
  resources:
    limits:
      cpus: '0.5'
      memory: 512M
4. Health Checks
# Add health checks for better reliability
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 30s
  timeout: 10s
  retries: 3
Lab Verification Checklist
Ensure you have completed the following:

 Docker Compose is installed and working
 Created a multi-service docker-compose.yml file
 Successfully started services with docker-compose up
 Verified web application is accessible on port 8080
 Database service is running and accessible
 Tested service scaling capabilities
 Successfully stopped and removed services
 Cleaned up system resources
Conclusion
Congratulations! You have successfully completed the Docker Compose lab. In this hands-on exercise, you have:

Key Accomplishments:

Installed Docker Compose and verified its functionality
Created a multi-container application with web server and database services
Learned YAML configuration for defining complex application stacks
Managed service lifecycles using Docker Compose commands
Implemented service scaling for handling increased load
Practiced proper cleanup of containers, networks, and volumes
Why This Matters: Docker Compose is essential for modern application development because it:

Simplifies Development: Developers can spin up entire application stacks with a single command
Ensures Consistency: Same environment across development, testing, and production
Improves Collaboration: Team members can easily replicate complex setups
Supports Microservices: Perfect for managing distributed applications
Enables DevOps Practices: Foundation for container orchestration and CI/CD pipelines
Next Steps:

Explore Docker Swarm for production orchestration
Learn Kubernetes for enterprise container management
Practice with more complex multi-service applications
Implement monitoring and logging for containerized applications
Study container security best practices
This lab has provided you with fundamental skills for managing multi-container applications, which is crucial for the Docker Certified Associate (DCA) certification and real-world container deployments.
