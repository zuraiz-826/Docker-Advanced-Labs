Lab 100: Docker for Automation - Using Docker with Terraform for Infrastructure Provisioning
Lab Objectives
By the end of this lab, students will be able to:

Install and configure Terraform with the Docker provider
Write Terraform configuration files to define and manage Docker containers
Provision Docker containers using Terraform automation
Implement container scaling strategies with Terraform
Integrate Docker provisioning into CI/CD pipelines using Terraform
Understand Infrastructure as Code (IaC) principles for container management
Troubleshoot common Docker and Terraform integration issues
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts (containers, images, registries)
Familiarity with command-line interface operations
Basic knowledge of YAML/JSON file formats
Understanding of version control systems (Git)
No prior Terraform experience required - this lab covers the basics
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your dedicated environment. No need to build or configure your own virtual machine.

Your cloud machine includes:

Ubuntu 22.04 LTS
Docker Engine pre-installed
Git version control
Text editors (nano, vim)
Internet connectivity for downloading tools
Task 1: Set up Terraform and Configure the Docker Provider
Subtask 1.1: Install Terraform
First, let's install Terraform on your cloud machine.

Update the system packages:
sudo apt update && sudo apt upgrade -y
Install required dependencies:
sudo apt install -y gnupg software-properties-common curl
Add HashiCorp GPG key:
wget -O- https://apt.releases.hashicorp.com/gpg | \
gpg --dearmor | \
sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
Add HashiCorp repository:
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
sudo tee /etc/apt/sources.list.d/hashicorp.list
Update package list and install Terraform:
sudo apt update
sudo apt install terraform
Verify Terraform installation:
terraform version
You should see output similar to:

Terraform v1.6.0
Subtask 1.2: Verify Docker Installation
Check Docker status:
sudo systemctl status docker
Verify Docker is working:
sudo docker run hello-world
Add your user to the docker group (to avoid using sudo):
sudo usermod -aG docker $USER
newgrp docker
Test Docker without sudo:
docker ps
Subtask 1.3: Create Project Directory Structure
Create a project directory:
mkdir ~/terraform-docker-lab
cd ~/terraform-docker-lab
Create subdirectories for organization:
mkdir -p {basic-container,multi-container,scaling,cicd}
Verify directory structure:
tree ~/terraform-docker-lab
Task 2: Write Terraform Configuration Files to Define Docker Containers
Subtask 2.1: Create Basic Docker Container Configuration
Navigate to the basic container directory:
cd ~/terraform-docker-lab/basic-container
Create the main Terraform configuration file:
nano main.tf
Add the following configuration:
# Configure the Docker Provider
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.1"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

# Pull the nginx image
resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = false
}

# Create a container
resource "docker_container" "nginx_server" {
  image = docker_image.nginx.image_id
  name  = "terraform-nginx"
  
  ports {
    internal = 80
    external = 8080
  }
  
  # Environment variables
  env = [
    "NGINX_HOST=localhost",
    "NGINX_PORT=80"
  ]
  
  # Restart policy
  restart = "unless-stopped"
  
  # Labels for organization
  labels {
    label = "environment"
    value = "development"
  }
  
  labels {
    label = "managed-by"
    value = "terraform"
  }
}
Save and exit (Ctrl+X, then Y, then Enter)
Subtask 2.2: Create Variables File
Create a variables file:
nano variables.tf
Add variable definitions:
variable "container_name" {
  description = "Name of the Docker container"
  type        = string
  default     = "terraform-nginx"
}

variable "external_port" {
  description = "External port for the container"
  type        = number
  default     = 8080
}

variable "image_name" {
  description = "Docker image name"
  type        = string
  default     = "nginx:latest"
}

variable "environment" {
  description = "Environment label"
  type        = string
  default     = "development"
}
Save and exit
Subtask 2.3: Create Outputs File
Create an outputs file:
nano outputs.tf
Add output definitions:
output "container_id" {
  description = "ID of the Docker container"
  value       = docker_container.nginx_server.id
}

output "container_name" {
  description = "Name of the Docker container"
  value       = docker_container.nginx_server.name
}

output "container_ip" {
  description = "IP address of the Docker container"
  value       = docker_container.nginx_server.network_data[0].ip_address
}

output "external_port" {
  description = "External port of the container"
  value       = docker_container.nginx_server.ports[0].external
}

output "access_url" {
  description = "URL to access the application"
  value       = "http://localhost:${docker_container.nginx_server.ports[0].external}"
}
Save and exit
Subtask 2.4: Update Main Configuration to Use Variables
Edit the main.tf file:
nano main.tf
Update the configuration to use variables:
# Configure the Docker Provider
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.1"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

# Pull the nginx image
resource "docker_image" "nginx" {
  name         = var.image_name
  keep_locally = false
}

# Create a container
resource "docker_container" "nginx_server" {
  image = docker_image.nginx.image_id
  name  = var.container_name
  
  ports {
    internal = 80
    external = var.external_port
  }
  
  # Environment variables
  env = [
    "NGINX_HOST=localhost",
    "NGINX_PORT=80"
  ]
  
  # Restart policy
  restart = "unless-stopped"
  
  # Labels for organization
  labels {
    label = "environment"
    value = var.environment
  }
  
  labels {
    label = "managed-by"
    value = "terraform"
  }
}
Save and exit
Task 3: Use Terraform to Provision Docker Containers
Subtask 3.1: Initialize Terraform
Initialize the Terraform working directory:
terraform init
You should see output indicating successful initialization and provider download.

Subtask 3.2: Validate Configuration
Validate the Terraform configuration:
terraform validate
Format the configuration files:
terraform fmt
Subtask 3.3: Plan the Deployment
Create an execution plan:
terraform plan
Review the plan output to understand what resources will be created.

Subtask 3.4: Apply the Configuration
Apply the Terraform configuration:
terraform apply
Type yes when prompted to confirm the deployment.

Verify the container is running:

docker ps
Test the deployed application:
curl http://localhost:8080
You should see the nginx welcome page HTML.

Subtask 3.5: View Terraform State
Show the current state:
terraform show
List resources in state:
terraform state list
Task 4: Automate Scaling of Docker Containers with Terraform
Subtask 4.1: Create Multi-Container Configuration
Navigate to the scaling directory:
cd ~/terraform-docker-lab/scaling
Create a scalable configuration:
nano main.tf
Add the multi-container configuration:
# Configure the Docker Provider
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.1"
    }
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

# Pull the nginx image
resource "docker_image" "nginx" {
  name         = "nginx:latest"
  keep_locally = false
}

# Create multiple nginx containers
resource "docker_container" "nginx_cluster" {
  count = var.container_count
  image = docker_image.nginx.image_id
  name  = "${var.container_prefix}-${count.index + 1}"
  
  ports {
    internal = 80
    external = var.base_port + count.index
  }
  
  # Environment variables
  env = [
    "NGINX_HOST=localhost",
    "NGINX_PORT=80",
    "INSTANCE_ID=${count.index + 1}"
  ]
  
  # Restart policy
  restart = "unless-stopped"
  
  # Labels
  labels {
    label = "environment"
    value = var.environment
  }
  
  labels {
    label = "managed-by"
    value = "terraform"
  }
  
  labels {
    label = "instance"
    value = tostring(count.index + 1)
  }
}

# Create a simple load balancer using nginx
resource "docker_image" "nginx_lb" {
  name         = "nginx:latest"
  keep_locally = false
}

# Create nginx config for load balancing
resource "local_file" "nginx_config" {
  content = templatefile("${path.module}/nginx.conf.tpl", {
    backend_servers = [
      for i in range(var.container_count) : {
        name = "${var.container_prefix}-${i + 1}"
        port = var.base_port + i
      }
    ]
  })
  filename = "${path.module}/nginx.conf"
}

resource "docker_container" "load_balancer" {
  image = docker_image.nginx_lb.image_id
  name  = "nginx-load-balancer"
  
  ports {
    internal = 80
    external = var.lb_port
  }
  
  # Mount the nginx config
  volumes {
    host_path      = abspath("${path.module}/nginx.conf")
    container_path = "/etc/nginx/nginx.conf"
    read_only      = true
  }
  
  # Restart policy
  restart = "unless-stopped"
  
  # Labels
  labels {
    label = "environment"
    value = var.environment
  }
  
  labels {
    label = "managed-by"
    value = "terraform"
  }
  
  labels {
    label = "role"
    value = "load-balancer"
  }
  
  depends_on = [docker_container.nginx_cluster]
}
Save and exit
Subtask 4.2: Create Variables for Scaling
Create variables file:
nano variables.tf
Add scaling variables:
variable "container_count" {
  description = "Number of containers to create"
  type        = number
  default     = 3
}

variable "container_prefix" {
  description = "Prefix for container names"
  type        = string
  default     = "web-server"
}

variable "base_port" {
  description = "Base port for containers"
  type        = number
  default     = 8080
}

variable "lb_port" {
  description = "Load balancer port"
  type        = number
  default     = 80
}

variable "environment" {
  description = "Environment label"
  type        = string
  default     = "production"
}
Save and exit
Subtask 4.3: Create Load Balancer Configuration Template
Create nginx configuration template:
nano nginx.conf.tpl
Add the template content:
events {
    worker_connections 1024;
}

http {
    upstream backend {
        %{ for server in backend_servers ~}
        server host.docker.internal:${server.port};
        %{ endfor ~}
    }
    
    server {
        listen 80;
        
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
        
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
}
Save and exit
Subtask 4.4: Create Outputs for Scaling
Create outputs file:
nano outputs.tf
Add outputs for all containers:
output "container_ids" {
  description = "IDs of all Docker containers"
  value       = docker_container.nginx_cluster[*].id
}

output "container_names" {
  description = "Names of all Docker containers"
  value       = docker_container.nginx_cluster[*].name
}

output "container_ports" {
  description = "External ports of all containers"
  value       = docker_container.nginx_cluster[*].ports[0].external
}

output "load_balancer_url" {
  description = "URL to access the load balancer"
  value       = "http://localhost:${docker_container.load_balancer.ports[0].external}"
}

output "individual_urls" {
  description = "URLs to access individual containers"
  value = [
    for container in docker_container.nginx_cluster :
    "http://localhost:${container.ports[0].external}"
  ]
}
Save and exit
Subtask 4.5: Deploy the Scaled Infrastructure
Initialize Terraform:
terraform init
Validate the configuration:
terraform validate
Plan the deployment:
terraform plan
Apply the configuration:
terraform apply
Verify all containers are running:
docker ps
Test individual containers:
curl http://localhost:8080
curl http://localhost:8081
curl http://localhost:8082
Subtask 4.6: Scale Up the Infrastructure
Create a terraform.tfvars file to override defaults:
nano terraform.tfvars
Add scaling configuration:
container_count = 5
environment = "production-scaled"
Save and exit

Apply the changes:

terraform plan
terraform apply
Verify the new containers:
docker ps | grep web-server
Task 5: Integrate Docker Provisioning with CI/CD Pipelines Using Terraform
Subtask 5.1: Create CI/CD Pipeline Structure
Navigate to the CI/CD directory:
cd ~/terraform-docker-lab/cicd
Create directory structure for CI/CD:
mkdir -p {scripts,configs,pipelines}
Subtask 5.2: Create Terraform Configuration for CI/CD
Create main configuration:
nano main.tf
Add CI/CD-focused configuration:
# Configure the Docker Provider
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.0.1"
    }
  }
  
  # Backend configuration for state management
  backend "local" {
    path = "terraform.tfstate"
  }
}

provider "docker" {
  host = "unix:///var/run/docker.sock"
}

# Data source for current timestamp
locals {
  timestamp = formatdate("YYYY-MM-DD-hhmm", timestamp())
}

# Pull application image
resource "docker_image" "app" {
  name         = var.app_image
  keep_locally = false
  
  # Force pull on each apply for CI/CD
  force_remove = true
}

# Create application containers
resource "docker_container" "app" {
  count = var.instance_count
  image = docker_image.app.image_id
  name  = "${var.app_name}-${var.environment}-${count.index + 1}"
  
  ports {
    internal = var.app_port
    external = var.base_external_port + count.index
  }
  
  # Environment variables
  env = [
    "APP_ENV=${var.environment}",
    "INSTANCE_ID=${count.index + 1}",
    "DEPLOY_TIME=${local.timestamp}",
    "VERSION=${var.app_version}"
  ]
  
  # Health check
  healthcheck {
    test         = ["CMD", "curl", "-f", "http://localhost:${var.app_port}/health"]
    interval     = "30s"
    timeout      = "10s"
    retries      = 3
    start_period = "60s"
  }
  
  # Restart policy
  restart = "unless-stopped"
  
  # Labels for CI/CD tracking
  labels {
    label = "environment"
    value = var.environment
  }
  
  labels {
    label = "version"
    value = var.app_version
  }
  
  labels {
    label = "deployed-by"
    value = "cicd-pipeline"
  }
  
  labels {
    label = "deploy-time"
    value = local.timestamp
  }
}

# Create monitoring container
resource "docker_image" "monitoring" {
  name         = "nginx:latest"
  keep_locally = false
}

resource "docker_container" "monitoring" {
  image = docker_image.monitoring.image_id
  name  = "${var.app_name}-monitoring-${var.environment}"
  
  ports {
    internal = 80
    external = var.monitoring_port
  }
  
  # Mount monitoring config
  volumes {
    host_path      = abspath("${path.module}/configs/monitoring.html")
    container_path = "/usr/share/nginx/html/index.html"
    read_only      = true
  }
  
  labels {
    label = "role"
    value = "monitoring"
  }
  
  labels {
    label = "environment"
    value = var.environment
  }
}
Save and exit
Subtask 5.3: Create CI/CD Variables
Create variables file:
nano variables.tf
Add CI/CD variables:
variable "app_name" {
  description = "Application name"
  type        = string
  default     = "myapp"
}

variable "app_image" {
  description = "Docker image for the application"
  type        = string
  default     = "nginx:latest"
}

variable "app_version" {
  description = "Application version"
  type        = string
  default     = "1.0.0"
}

variable "environment" {
  description = "Deployment environment"
  type        = string
  default     = "staging"
  
  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment must be development, staging, or production."
  }
}

variable "instance_count" {
  description = "Number of application instances"
  type        = number
  default     = 2
  
  validation {
    condition     = var.instance_count >= 1 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}

variable "app_port" {
  description = "Internal application port"
  type        = number
  default     = 80
}

variable "base_external_port" {
  description = "Base external port for applications"
  type        = number
  default     = 9000
}

variable "monitoring_port" {
  description = "Monitoring dashboard port"
  type        = number
  default     = 9090
}
Save and exit
Subtask 5.4: Create Deployment Scripts
Create deployment script:
nano scripts/deploy.sh
Add deployment automation:
#!/bin/bash

# CI/CD Deployment Script for Docker with Terraform
set -e

# Configuration
ENVIRONMENT=${1:-staging}
VERSION=${2:-latest}
WORKSPACE_DIR=$(pwd)

echo "=== Docker + Terraform CI/CD Deployment ==="
echo "Environment: $ENVIRONMENT"
echo "Version: $VERSION"
echo "Workspace: $WORKSPACE_DIR"

# Function to check prerequisites
check_prerequisites() {
    echo "Checking prerequisites..."
    
    if ! command -v terraform &> /dev/null; then
        echo "Error: Terraform is not installed"
        exit 1
    fi
    
    if ! command -v docker &> /dev/null; then
        echo "Error: Docker is not installed"
        exit 1
    fi
    
    if ! docker info &> /dev/null; then
        echo "Error: Docker daemon is not running"
        exit 1
    fi
    
    echo "Prerequisites check passed!"
}

# Function to validate Terraform configuration
validate_terraform() {
    echo "Validating Terraform configuration..."
    terraform fmt -check=true
    terraform validate
    echo "Terraform validation passed!"
}

# Function to plan deployment
plan_deployment() {
    echo "Planning deployment..."
    terraform plan \
        -var="environment=$ENVIRONMENT" \
        -var="app_version=$VERSION" \
        -out=tfplan
    echo "Deployment plan created!"
}

# Function to apply deployment
apply_deployment() {
    echo "Applying deployment..."
    terraform apply tfplan
    echo "Deployment completed!"
}

# Function to run health checks
health_check() {
    echo "Running health checks..."
    sleep 10  # Wait for containers to start
    
    # Get container ports from Terraform output
    PORTS=$(terraform output -json container_ports | jq -r '.[]')
    
    for port in $PORTS; do
        echo "Checking health on port $port..."
        if curl -f "http://localhost:$port" &> /dev/null; then
            echo "✓ Port $port is healthy"
        else
            echo "✗ Port $port failed health check"
            exit 1
        fi
    done
    
    echo "All health checks passed!"
}

# Function to cleanup old deployments
cleanup_old() {
    echo "Cleaning up old containers..."
    # Remove containers older than 1 hour with the same app name
    docker container prune -f --filter "until=1h"
    docker image prune -f
    echo "Cleanup completed!"
}

# Main execution
main() {
    check_prerequisites
    
    # Initialize Terraform if needed
    if [ ! -d ".terraform" ]; then
        echo "Initializing Terraform..."
        terraform init
    fi
    
    validate_terraform
    plan_deployment
    apply_deployment
    health_check
    cleanup_old
    
    echo "=== Deployment Successful ==="
    echo "Access your application:"
    terraform output -json individual_urls | jq -r '.[]'
}

# Execute main function
main "$@"
Make the script executable:
chmod +x scripts/deploy.sh
Subtask 5.5: Create Rollback Script
Create rollback script:
nano scripts/rollback.sh
Add rollback functionality:
#!/bin/bash

# CI/CD Rollback Script for Docker with Terraform
set -e

ENVIRONMENT=${1:-staging}
BACKUP_STATE=${2:-terraform.tfstate.backup}

echo "=== Docker + Terraform CI/CD Rollback ==="
echo "Environment: $ENVIRONMENT"
echo "Backup State: $BACKUP_STATE"

# Function to rollback to previous state
rollback_deployment() {
    echo "Rolling back deployment..."
    
    if [ -f "$BACKUP_STATE" ]; then
        echo "Restoring from backup state: $BACKUP_STATE"
        cp "$BACKUP_STATE" terraform.tfstate
        terraform apply -auto-approve
        echo "Rollback completed!"
    else
        echo "No backup state found. Destroying current deployment..."
        terraform destroy -auto-approve
        echo "Current deployment destroyed!"
    fi
}

# Function to verify rollback
verify_rollback() {
    echo "Verifying rollback..."
    sleep 5
    
    # Check if any containers are still running
    RUNNING_CONTAINERS=$(docker ps --filter "label=deployed-by=cicd-pipeline" --format "table {{.Names}}" | tail -n +2)
    
    if [ -z "$RUNNING_CONTAINERS" ]; then
        echo "✓ All CI/CD containers have been stopped"
    else
        echo "✓ Rollback containers are running:"
        echo "$RUNNING_CONTAINERS"
    fi
}

# Main execution
main() {
    rollback_deployment
    verify_rollback
    echo "=== Rollback Completed ==="
}

# Execute main function
main "$@"
Make the script executable:
chmod +x scripts/rollback.sh
Subtask 5.6: Create Monitoring Configuration
Create monitoring config directory:
mkdir -p configs
Create monitoring dashboard:
nano configs/monitoring.html
Add monitoring HTML:
<!DOCTYPE html>
<html>
<head>
    <title>Application Monitoring Dashboard</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .container { max-width: 800px; margin: 0 auto; }
        .status { padding: 20px; margin: 10px 0; border-radius: 5px; }
        .healthy { background-color: #d4edda; border: 1px solid #c3e6cb; }
        .info { background-color: #d1ecf1; border: 1px solid #bee5eb; }
        h1 { color: #333; }
        .refresh-btn { 
            background-color: #007bff; 
            color: white; 
            padding: 10px 20px; 
            border: none; 
            border-radius: 5px; 
            cursor: pointer; 
        }
    </style>
    <script>
        function refreshStatus() {
            location.reload();
        }
        
        // Auto-refresh every 30 seconds
        setInterval(refreshStatus, 30000);
    </script>
</head>
<body>
    <div class="container">
        <h1>🐳 Docker + Terraform CI/CD Monitoring</h1>
        
        <div class="status info">
            <h3>Deployment Information</h3>
            <p><strong>Last Updated:</strong> <span id="timestamp"></span></p>
            <p><strong>Environment:</strong> Staging</p>
            <p><strong>Version:</strong> 1.0.0</p>
        </div>
        
        <div class="status healthy">
            <h3>✅ System Status: Healthy</h3>
            <p>All containers are running successfully</p>
        </div>
        
        <div class="status info">
            <h3>Quick Actions</h3>
            <button class="refresh-btn" onclick="refreshStatus()">Refresh Status</button>
        </div>
        
        <div class="status info">
            <h3>Container Links</h3>
            <p>Access your application instances:</p>
            <ul>
                <li><a href="http://localhost:9000" target="_blank">Instance 1 (Port 9000)</a></li>
                <li><a href="http://localhost:9001" target="_blank">Instance 2 (Port 9001)</a></li>
            </ul>
        </div>
    </div>
    
    <script>
        document.getElementById('timestamp').textContent = new Date().toLocaleString();
    </script>
</body>
</html>
Save and exit
Subtask 5.7: Create Environment-Specific Configurations
Create staging environment config:
nano staging.tfvars
Add staging configuration:
environment = "staging"
app_version = "1.0.0"
instance_count = 2
base_external_port = 9000
monitoring_port = 9090
Create production environment config:
nano production.tfvars
Add production configuration:
environment = "production"
app_version = "1.0.0"
instance_count = 3
base_external_port = 8000
monitoring_port = 8090
Save and exit
Subtask 5.8: Test the CI/CD Pipeline
Initialize Terraform:
terraform init
Test the deployment script:
./scripts/deploy.sh staging 1.0.0
Verify the deployment:
docker ps
curl http://localhost:9000
curl http://localhost:9001
curl http://localhost:9090
Test scaling by updating instance count:
terraform apply -var="instance_count=4" -auto-approve
Verify scaling:
docker ps | grep myapp
Test rollback:
./scripts/rollback.sh staging
Troubleshooting Common Issues
Issue 1: Docker Permission Denied
Problem: Getting permission denied errors when running Docker commands.

Solution:

sudo usermod -aG docker $USER
newgrp docker
# Or restart your session
Issue 2: Terraform Provider Download Issues
Problem: Terraform fails to download the Docker provider.

Solution:

# Clear Terraform cache
rm -rf .terraform
terraform init -upgrade
Issue 3: Port Already in Use
Problem: Container fails to start because
