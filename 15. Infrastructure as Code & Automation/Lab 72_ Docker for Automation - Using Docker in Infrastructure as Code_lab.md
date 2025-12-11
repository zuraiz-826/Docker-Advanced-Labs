Lab 72: Docker for Automation - Using Docker in Infrastructure as Code
Lab Objectives
By the end of this lab, students will be able to:

Implement Docker containers as part of automated infrastructure setup
Use docker-compose for managing multi-container applications in IaC environments
Integrate Docker with Terraform and Ansible for infrastructure provisioning and orchestration
Deploy containers automatically using CI/CD pipelines with GitHub Actions
Automate scaling and updates for containerized services using Infrastructure as Code principles
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with Linux command line operations
Basic knowledge of YAML syntax
Understanding of version control with Git
Conceptual knowledge of Infrastructure as Code principles
Note: Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click "Start Lab" to access your pre-configured environment - no need to build your own VM.

Lab Environment Setup
Your cloud machine comes pre-installed with:

Docker Engine
Docker Compose
Terraform
Ansible
Git
Text editors (nano, vim)
Task 1: Create Docker Containers as Part of Automated Infrastructure Setup
Subtask 1.1: Create a Multi-Tier Application Structure
First, let's create a directory structure for our Infrastructure as Code project:

mkdir -p ~/docker-iac-lab
cd ~/docker-iac-lab
mkdir -p {web-app,database,infrastructure,scripts}
Subtask 1.2: Build a Web Application Container
Create a simple web application that will be part of our infrastructure:

cd ~/docker-iac-lab/web-app
Create the application file:

cat > app.py << 'EOF'
from flask import Flask, jsonify
import os
import socket

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({
        'message': 'Hello from Docker IaC Lab!',
        'hostname': socket.gethostname(),
        'environment': os.environ.get('ENVIRONMENT', 'development')
    })

@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
EOF
Create the requirements file:

cat > requirements.txt << 'EOF'
Flask==2.3.3
Werkzeug==2.3.7
EOF
Create the Dockerfile:

cat > Dockerfile << 'EOF'
FROM python:3.9-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
EOF
Subtask 1.3: Create Infrastructure Configuration Files
Create a configuration file for our infrastructure:

cd ~/docker-iac-lab/infrastructure
cat > config.yml << 'EOF'
application:
  name: "docker-iac-demo"
  version: "1.0.0"
  environment: "production"
  
containers:
  web:
    image: "docker-iac-web"
    replicas: 2
    port: 5000
  
  database:
    image: "postgres:13"
    replicas: 1
    port: 5432
    
network:
  name: "iac-network"
  driver: "bridge"

volumes:
  database:
    name: "postgres-data"
EOF
Task 2: Use Docker-Compose for Managing Containers in IaC Environment
Subtask 2.1: Create Docker Compose Configuration
Navigate to the main project directory and create a comprehensive docker-compose file:

cd ~/docker-iac-lab
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    build: ./web-app
    ports:
      - "5000:5000"
    environment:
      - ENVIRONMENT=production
      - DATABASE_URL=postgresql://postgres:password@db:5432/iacdb
    depends_on:
      - db
    networks:
      - iac-network
    restart: unless-stopped
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 512M
        reservations:
          memory: 256M

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=iacdb
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - iac-network
    restart: unless-stopped
    ports:
      - "5432:5432"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web
    networks:
      - iac-network
    restart: unless-stopped

networks:
  iac-network:
    driver: bridge

volumes:
  postgres-data:
    driver: local
EOF
Subtask 2.2: Create Nginx Load Balancer Configuration
Create an Nginx configuration for load balancing:

cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream web_servers {
        server web:5000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://web_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /health {
            proxy_pass http://web_servers/health;
        }
    }
}
EOF
Subtask 2.3: Deploy the Stack
Build and deploy the complete stack:

# Build the web application image
docker-compose build

# Start the entire stack
docker-compose up -d

# Verify all services are running
docker-compose ps
Test the deployment:

# Test the web application
curl http://localhost/

# Test the health endpoint
curl http://localhost/health

# Check logs
docker-compose logs web
Task 3: Integrate Docker with Terraform for Infrastructure Provisioning
Subtask 3.1: Create Terraform Configuration
Create Terraform configuration files for Docker infrastructure:

cd ~/docker-iac-lab/infrastructure
Create the main Terraform configuration:

cat > main.tf << 'EOF'
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

# Create a custom network
resource "docker_network" "iac_network" {
  name = "terraform-iac-network"
  driver = "bridge"
}

# Create volume for database
resource "docker_volume" "postgres_data" {
  name = "terraform-postgres-data"
}

# Database container
resource "docker_container" "database" {
  name  = "terraform-postgres"
  image = "postgres:13"
  
  env = [
    "POSTGRES_DB=terraformdb",
    "POSTGRES_USER=postgres",
    "POSTGRES_PASSWORD=terraform123"
  ]
  
  volumes {
    volume_name    = docker_volume.postgres_data.name
    container_path = "/var/lib/postgresql/data"
  }
  
  networks_advanced {
    name = docker_network.iac_network.name
  }
  
  ports {
    internal = 5432
    external = 5433
  }
  
  restart = "unless-stopped"
}

# Web application container
resource "docker_container" "web_app" {
  count = 2
  name  = "terraform-web-${count.index + 1}"
  image = "docker-iac-web:latest"
  
  env = [
    "ENVIRONMENT=terraform-managed",
    "DATABASE_URL=postgresql://postgres:terraform123@terraform-postgres:5432/terraformdb"
  ]
  
  networks_advanced {
    name = docker_network.iac_network.name
  }
  
  ports {
    internal = 5000
    external = 5001 + count.index
  }
  
  depends_on = [docker_container.database]
  restart = "unless-stopped"
}
EOF
Create variables file:

cat > variables.tf << 'EOF'
variable "app_name" {
  description = "Name of the application"
  type        = string
  default     = "docker-iac-terraform"
}

variable "environment" {
  description = "Environment name"
  type        = string
  default     = "development"
}

variable "web_replicas" {
  description = "Number of web application replicas"
  type        = number
  default     = 2
}
EOF
Create outputs file:

cat > outputs.tf << 'EOF'
output "web_app_urls" {
  description = "URLs of the web applications"
  value = [
    for container in docker_container.web_app :
    "http://localhost:${container.ports[0].external}"
  ]
}

output "database_port" {
  description = "Database external port"
  value = docker_container.database.ports[0].external
}

output "network_name" {
  description = "Docker network name"
  value = docker_network.iac_network.name
}
EOF
Subtask 3.2: Deploy Infrastructure with Terraform
First, build the web application image:

cd ~/docker-iac-lab/web-app
docker build -t docker-iac-web:latest .
Now deploy with Terraform:

cd ~/docker-iac-lab/infrastructure

# Initialize Terraform
terraform init

# Plan the deployment
terraform plan

# Apply the configuration
terraform apply -auto-approve

# Check the outputs
terraform output
Test the Terraform-managed containers:

# Test web applications
curl http://localhost:5001/
curl http://localhost:5002/

# List Terraform-managed resources
terraform state list
Task 4: Integrate Docker with Ansible for Configuration Management
Subtask 4.1: Create Ansible Playbook
Create Ansible configuration for Docker container management:

cd ~/docker-iac-lab
mkdir ansible-config
cd ansible-config
Create the Ansible inventory:

cat > inventory.ini << 'EOF'
[local]
localhost ansible_connection=local

[local:vars]
ansible_python_interpreter=/usr/bin/python3
EOF
Create the main playbook:

cat > docker-playbook.yml << 'EOF'
---
- name: Deploy Docker Infrastructure with Ansible
  hosts: local
  become: yes
  vars:
    app_name: "ansible-docker-iac"
    environment: "ansible-managed"
    web_replicas: 3
    
  tasks:
    - name: Ensure Docker is running
      service:
        name: docker
        state: started
        enabled: yes

    - name: Create custom Docker network
      docker_network:
        name: "{{ app_name }}-network"
        driver: bridge
        state: present

    - name: Create Docker volume for database
      docker_volume:
        name: "{{ app_name }}-postgres-data"
        state: present

    - name: Deploy PostgreSQL database
      docker_container:
        name: "{{ app_name }}-postgres"
        image: postgres:13
        state: started
        restart_policy: unless-stopped
        env:
          POSTGRES_DB: "ansibledb"
          POSTGRES_USER: "postgres"
          POSTGRES_PASSWORD: "ansible123"
        volumes:
          - "{{ app_name }}-postgres-data:/var/lib/postgresql/data"
        networks:
          - name: "{{ app_name }}-network"
        published_ports:
          - "5434:5432"

    - name: Deploy web application containers
      docker_container:
        name: "{{ app_name }}-web-{{ item }}"
        image: "docker-iac-web:latest"
        state: started
        restart_policy: unless-stopped
        env:
          ENVIRONMENT: "{{ environment }}"
          DATABASE_URL: "postgresql://postgres:ansible123@{{ app_name }}-postgres:5432/ansibledb"
        networks:
          - name: "{{ app_name }}-network"
        published_ports:
          - "{{ 6000 + item }}:5000"
      loop: "{{ range(1, web_replicas + 1) | list }}"

    - name: Deploy Nginx load balancer
      docker_container:
        name: "{{ app_name }}-nginx"
        image: nginx:alpine
        state: started
        restart_policy: unless-stopped
        networks:
          - name: "{{ app_name }}-network"
        published_ports:
          - "8080:80"
        volumes:
          - "{{ playbook_dir }}/nginx-ansible.conf:/etc/nginx/nginx.conf:ro"
EOF
Create Nginx configuration for Ansible deployment:

cat > nginx-ansible.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    upstream ansible_web_servers {
        server ansible-docker-iac-web-1:5000;
        server ansible-docker-iac-web-2:5000;
        server ansible-docker-iac-web-3:5000;
    }

    server {
        listen 80;
        server_name localhost;

        location / {
            proxy_pass http://ansible_web_servers;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /health {
            proxy_pass http://ansible_web_servers/health;
        }
    }
}
EOF
Subtask 4.2: Deploy with Ansible
Install Ansible Docker collection:

ansible-galaxy collection install community.docker
Run the Ansible playbook:

# Check syntax
ansible-playbook -i inventory.ini docker-playbook.yml --syntax-check

# Run the playbook
ansible-playbook -i inventory.ini docker-playbook.yml

# Test the deployment
curl http://localhost:8080/
curl http://localhost:8080/health
Task 5: Deploy Containers Automatically with CI/CD Pipelines
Subtask 5.1: Create GitHub Actions Workflow
Create a CI/CD pipeline configuration:

cd ~/docker-iac-lab
mkdir -p .github/workflows
cat > .github/workflows/docker-iac-pipeline.yml << 'EOF'
name: Docker IaC Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: docker.io
  IMAGE_NAME: docker-iac-web

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build Docker image
      uses: docker/build-push-action@v5
      with:
        context: ./web-app
        push: false
        tags: ${{ env.IMAGE_NAME }}:test
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: Test Docker image
      run: |
        docker run -d --name test-container -p 5000:5000 ${{ env.IMAGE_NAME }}:test
        sleep 10
        curl -f http://localhost:5000/health || exit 1
        docker stop test-container
        docker rm test-container

  deploy-staging:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build and push staging image
      uses: docker/build-push-action@v5
      with:
        context: ./web-app
        push: false
        tags: ${{ env.IMAGE_NAME }}:staging
        cache-from: type=gha

    - name: Deploy to staging with docker-compose
      run: |
        export ENVIRONMENT=staging
        docker-compose -f docker-compose.staging.yml up -d
        
    - name: Run staging tests
      run: |
        sleep 30
        curl -f http://localhost/health || exit 1

  deploy-production:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Build production image
      uses: docker/build-push-action@v5
      with:
        context: ./web-app
        push: false
        tags: ${{ env.IMAGE_NAME }}:latest
        cache-from: type=gha

    - name: Deploy with Terraform
      run: |
        cd infrastructure
        terraform init
        terraform plan -out=tfplan
        terraform apply tfplan

    - name: Run production health checks
      run: |
        sleep 30
        curl -f http://localhost:5001/health || exit 1
        curl -f http://localhost:5002/health || exit 1
EOF
Subtask 5.2: Create Staging Docker Compose Configuration
cat > docker-compose.staging.yml << 'EOF'
version: '3.8'

services:
  web:
    build: ./web-app
    ports:
      - "5000:5000"
    environment:
      - ENVIRONMENT=staging
      - DATABASE_URL=postgresql://postgres:staging123@db:5432/stagingdb
    depends_on:
      - db
    networks:
      - staging-network

  db:
    image: postgres:13
    environment:
      - POSTGRES_DB=stagingdb
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=staging123
    volumes:
      - staging-postgres-data:/var/lib/postgresql/data
    networks:
      - staging-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - web
    networks:
      - staging-network

networks:
  staging-network:
    driver: bridge

volumes:
  staging-postgres-data:
    driver: local
EOF
Subtask 5.3: Create Deployment Scripts
Create automated deployment scripts:

cd ~/docker-iac-lab/scripts
cat > deploy.sh << 'EOF'
#!/bin/bash

set -e

ENVIRONMENT=${1:-development}
ACTION=${2:-deploy}

echo "Starting $ACTION for $ENVIRONMENT environment..."

case $ENVIRONMENT in
  "development")
    echo "Deploying to development with docker-compose..."
    docker-compose up -d
    ;;
  "staging")
    echo "Deploying to staging with docker-compose..."
    docker-compose -f docker-compose.staging.yml up -d
    ;;
  "production")
    echo "Deploying to production with Terraform..."
    cd ../infrastructure
    terraform init
    terraform apply -auto-approve
    ;;
  *)
    echo "Unknown environment: $ENVIRONMENT"
    exit 1
    ;;
esac

echo "Deployment completed successfully!"
EOF
cat > health-check.sh << 'EOF'
#!/bin/bash

set -e

ENVIRONMENT=${1:-development}
MAX_RETRIES=30
RETRY_INTERVAL=5

echo "Running health checks for $ENVIRONMENT environment..."

case $ENVIRONMENT in
  "development")
    URLS=("http://localhost/health")
    ;;
  "staging")
    URLS=("http://localhost/health")
    ;;
  "production")
    URLS=("http://localhost:5001/health" "http://localhost:5002/health")
    ;;
esac

for url in "${URLS[@]}"; do
  echo "Checking $url..."
  
  for i in $(seq 1 $MAX_RETRIES); do
    if curl -f -s "$url" > /dev/null; then
      echo "✓ $url is healthy"
      break
    else
      if [ $i -eq $MAX_RETRIES ]; then
        echo "✗ $url failed health check after $MAX_RETRIES attempts"
        exit 1
      fi
      echo "Attempt $i/$MAX_RETRIES failed, retrying in ${RETRY_INTERVAL}s..."
      sleep $RETRY_INTERVAL
    fi
  done
done

echo "All health checks passed!"
EOF
Make scripts executable:

chmod +x deploy.sh health-check.sh
Task 6: Automate Scaling and Updates for Containerized Services
Subtask 6.1: Create Scaling Scripts
Create automated scaling solutions:

cd ~/docker-iac-lab/scripts
cat > scale.sh << 'EOF'
#!/bin/bash

set -e

SERVICE=${1:-web}
REPLICAS=${2:-3}
ENVIRONMENT=${3:-development}

echo "Scaling $SERVICE to $REPLICAS replicas in $ENVIRONMENT environment..."

case $ENVIRONMENT in
  "development")
    echo "Scaling with docker-compose..."
    docker-compose up -d --scale $SERVICE=$REPLICAS
    ;;
  "production")
    echo "Scaling with Terraform..."
    cd ../infrastructure
    terraform apply -var="web_replicas=$REPLICAS" -auto-approve
    ;;
  *)
    echo "Manual scaling for $ENVIRONMENT..."
    for i in $(seq 1 $REPLICAS); do
      container_name="${ENVIRONMENT}-${SERVICE}-${i}"
      if ! docker ps -q -f name="$container_name" | grep -q .; then
        echo "Starting $container_name..."
        docker run -d \
          --name "$container_name" \
          --network "${ENVIRONMENT}-network" \
          -p "$((5000 + i)):5000" \
          docker-iac-web:latest
      fi
    done
    ;;
esac

echo "Scaling completed!"
EOF
Subtask 6.2: Create Update and Rollback Scripts
cat > update.sh << 'EOF'
#!/bin/bash

set -e

IMAGE_TAG=${1:-latest}
ENVIRONMENT=${2:-development}
STRATEGY=${3:-rolling}

echo "Updating containers to $IMAGE_TAG in $ENVIRONMENT environment using $STRATEGY strategy..."

case $STRATEGY in
  "rolling")
    echo "Performing rolling update..."
    case $ENVIRONMENT in
      "development")
        docker-compose pull
        docker-compose up -d --no-deps web
        ;;
      "production")
        # Update containers one by one
        containers=$(docker ps --format "table {{.Names}}" | grep "terraform-web" | tail -n +2)
        for container in $containers; do
          echo "Updating $container..."
          docker stop "$container"
          docker rm "$container"
          # Terraform will recreate the container
          cd ../infrastructure
          terraform apply -auto-approve
          sleep 10
        done
        ;;
    esac
    ;;
  "blue-green")
    echo "Performing blue-green deployment..."
    # Create new version alongside existing
    docker-compose -f docker-compose.blue-green.yml up -d
    echo "New version deployed. Manual verification required before switching traffic."
    ;;
esac

echo "Update completed!"
EOF
cat > rollback.sh << 'EOF'
#!/bin/bash

set -e

PREVIOUS_TAG=${1:-previous}
ENVIRONMENT=${2:-development}

echo "Rolling back to $PREVIOUS_TAG in $ENVIRONMENT environment..."

case $ENVIRONMENT in
  "development")
    # Update docker-compose to use previous tag
    sed -i "s/docker-iac-web:latest/docker-iac-web:$PREVIOUS_TAG/g" ../docker-compose.yml
    docker-compose up -d --no-deps web
    ;;
  "production")
    echo "Rolling back Terraform deployment..."
    cd ../infrastructure
    terraform apply -auto-approve
    ;;
esac

echo "Rollback completed!"
EOF
Subtask 6.3: Create Monitoring and Auto-scaling Script
cat > monitor-and-scale.sh << 'EOF'
#!/bin/bash

set -e

ENVIRONMENT=${1:-development}
CPU_THRESHOLD=${2:-80}
MEMORY_THRESHOLD=${3:-80}
MIN_REPLICAS=${4:-2}
MAX_REPLICAS=${5:-10}

echo "Starting monitoring and auto-scaling for $ENVIRONMENT environment..."
echo "CPU Threshold: $CPU_THRESHOLD%, Memory Threshold: $MEMORY_THRESHOLD%"
echo "Replica range: $MIN_REPLICAS - $MAX_REPLICAS"

while true; do
  echo "$(date): Checking container metrics..."
  
  # Get current replica count
  current_replicas=$(docker ps --filter "name=web" --format "table {{.Names}}" | tail -n +2 | wc -l)
  
  # Get average CPU and memory usage
  avg_cpu=$(docker stats --no-stream --format "table {{.CPUPerc}}" | tail -n +2 | sed 's/%//g' | awk '{sum+=$1} END {print sum/NR}')
  avg_memory=$(docker stats --no-stream --format "table {{.MemPerc}}" | tail -n +2 | sed 's/%//g' | awk '{sum+=$1} END {print sum/NR}')
  
  echo "Current replicas: $current_replicas, Avg CPU: ${avg_cpu}%, Avg Memory: ${avg_memory}%"
  
  # Scale up if needed
  if (( $(echo "$avg_cpu > $CPU_THRESHOLD" | bc -l) )) || (( $(echo "$avg_memory > $MEMORY_THRESHOLD" | bc -l) )); then
    if [ $current_replicas -lt $MAX_REPLICAS ]; then
      new_replicas=$((current_replicas + 1))
      echo "High resource usage detected. Scaling up to $new_replicas replicas..."
      ./scale.sh web $new_replicas $ENVIRONMENT
    else
      echo "Already at maximum replicas ($MAX_REPLICAS)"
    fi
  # Scale down if needed
  elif (( $(echo "$avg_cpu < 30" | bc -l) )) && (( $(echo "$avg_memory < 30" | bc -l) )); then
    if [ $current_replicas -gt $MIN_REPLICAS ]; then
      new_replicas=$((current_replicas - 1))
      echo "Low resource usage detected. Scaling down to $new_replicas replicas..."
      ./scale.sh web $new_replicas $ENVIRONMENT
    else
      echo "Already at minimum replicas ($MIN_REPLICAS)"
    fi
  else
    echo "Resource usage within normal range"
  fi
  
  sleep 60
done
EOF
Make all scripts executable:

chmod +x scale.sh update.sh rollback.sh monitor-and-scale.sh
Subtask 6.4: Test Scaling and Updates
Test the scaling functionality:

# Test scaling up
./scale.sh web 4 development

# Verify scaling
docker-compose ps

# Test health checks after scaling
./health-check.sh development

# Test update process
./update.sh latest development rolling

# Test rollback (if needed)
# ./rollback.sh previous development
Verification and Testing
Complete System Test
Run a comprehensive test of all components:

cd ~/docker-iac-lab

# Test Docker Compose deployment
echo "Testing Docker Compose deployment..."
docker-compose up -d
sleep 30
curl -f http://localhost/health

# Test Terraform deployment
echo "Testing Terraform deployment..."
cd infrastructure
terraform apply -auto-approve
sleep 30
curl -f http://localhost:5001/health
curl -f http://localhost:5002/health

# Test Ansible deployment
echo "Testing Ansible deployment..."
cd ../ansible-config
ansible-playbook -i inventory.ini docker-playbook.yml
sleep 30
curl -f http://localhost:8080/health

# Test scaling
echo "Testing scaling..."
cd ../scripts
./scale.sh web 3 development
sleep 20
./health-check.sh development
Clean Up Resources
# Stop Docker Compose services
docker-compose down -v

# Destroy Terraform resources
cd infrastructure
terraform destroy -auto-approve

# Clean up Ansible resources
docker stop $(docker ps -q --filter "name=ansible-docker-iac")
docker rm $(docker ps -aq --filter "name=ansible-docker-iac")
docker network rm ansible-docker-iac-network
docker volume rm ansible-docker-iac-postgres-data
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Docker containers fail to start

Solution: Check Docker daemon status with sudo systemctl status docker
Verify image exists with docker images
Check logs with docker logs <container-name>
Issue 2: Terraform fails to apply

Solution: Run terraform init again
Check Docker provider version compatibility
Verify Docker socket permissions
Issue 3: Ansible playbook fails

Solution: Install required collections: ansible-galaxy collection install community.docker
Check Python Docker library: pip install docker
Verify inventory file syntax
Issue 4: Health checks fail

Solution: Wait longer for containers to start
Check container logs for application errors
Verify network connectivity between containers
Issue 5: Port conflicts

Solution: Use different external ports in configurations
Check for existing services: netstat -tlnp
Stop conflicting services before deployment
Conclusion
In this comprehensive lab, you have successfully implemented Docker in Infrastructure as Code workflows by:

Creating automated infrastructure setups with Docker containers, including multi-tier applications with web servers, databases, and load balancers
Using docker-compose for managing complex multi-container applications with proper networking, volumes, and service dependencies
Integrating Docker with Terraform for infrastructure provisioning, enabling declarative infrastructure management with version control
Integrating Docker with Ansible for configuration management, providing powerful automation capabilities for container deployment
Implementing CI/CD pipelines with GitHub Actions for automated testing, building, and deployment of containerized applications
Automating scaling and updates with custom scripts that handle rolling updates, blue-green deployments, and auto-scaling based on resource metrics
This lab demonstrates the power of combining Docker with Infrastructure as Code principles, enabling you to create reproducible, scalable, and maintainable infrastructure. These skills are essential for modern DevOps practices and are highly valued in cloud-native application development.

The techniques learned here form the foundation for enterprise-level container orchestration and can be extended to work with Kubernetes, cloud platforms, and more sophisticated CI/CD pipelines. You now have practical experience with the core concepts needed for the Docker Certified Associate (DCA) certification and real-world Docker automation scenarios.
