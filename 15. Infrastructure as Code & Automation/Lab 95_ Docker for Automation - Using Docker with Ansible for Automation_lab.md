Lab 95: Docker for Automation - Using Docker with Ansible for Automation
Lab Objectives
By the end of this lab, students will be able to:

• Set up and configure Ansible for Docker container management • Create and execute Ansible playbooks for automated Docker deployments • Implement container configuration management using Ansible • Scale Docker services across multiple hosts using Ansible automation • Integrate Ansible with CI/CD pipelines for continuous deployment workflows • Understand best practices for infrastructure as code with Docker and Ansible

Prerequisites
Before starting this lab, students should have:

• Basic understanding of Linux command line operations • Familiarity with Docker concepts (containers, images, volumes) • Basic knowledge of YAML syntax • Understanding of SSH key-based authentication • Familiarity with text editors like nano or vim

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install software manually.

Your lab environment includes: • Ubuntu 20.04 LTS with Docker pre-installed • Ansible 2.9+ pre-installed • SSH access configured between machines • Sample application files ready for deployment

Task 1: Set up Ansible and Docker Integration
Subtask 1.1: Verify Ansible and Docker Installation
First, let's verify that both Ansible and Docker are properly installed and running.

# Check Ansible version
ansible --version

# Check Docker version
docker --version

# Check Docker service status
sudo systemctl status docker

# Verify Docker is running by listing containers
docker ps
Subtask 1.2: Configure Ansible Inventory
Create an inventory file to define the hosts that Ansible will manage.

# Create a directory for our Ansible project
mkdir ~/ansible-docker-lab
cd ~/ansible-docker-lab

# Create the inventory file
nano inventory.ini
Add the following content to the inventory file:

[docker_hosts]
localhost ansible_connection=local

[docker_hosts:vars]
ansible_user=ubuntu
ansible_become=yes
Subtask 1.3: Test Ansible Connectivity
Verify that Ansible can connect to the target hosts.

# Test connectivity to all hosts
ansible all -i inventory.ini -m ping

# Test with elevated privileges
ansible all -i inventory.ini -m setup -a "filter=ansible_distribution*"
Subtask 1.4: Install Docker Python Module
Ansible requires the Docker Python module to manage Docker containers.

# Install Docker Python module
pip3 install docker

# Verify installation
python3 -c "import docker; print('Docker module installed successfully')"
Task 2: Create Ansible Playbooks for Deploying Docker Containers
Subtask 2.1: Create a Basic Container Deployment Playbook
Create your first Ansible playbook to deploy a simple web application container.

# Create the playbook file
nano deploy-nginx.yml
Add the following content:

---
- name: Deploy Nginx Container with Ansible
  hosts: docker_hosts
  become: yes
  tasks:
    - name: Pull Nginx Docker image
      docker_image:
        name: nginx
        tag: latest
        source: pull

    - name: Create Nginx container
      docker_container:
        name: nginx-web
        image: nginx:latest
        state: started
        restart_policy: always
        ports:
          - "8080:80"
        volumes:
          - /tmp/nginx-content:/usr/share/nginx/html:ro

    - name: Create custom HTML content
      copy:
        content: |
          <html>
          <head><title>Ansible + Docker Lab</title></head>
          <body>
            <h1>Welcome to Ansible + Docker Automation!</h1>
            <p>This container was deployed using Ansible automation.</p>
            <p>Deployment time: {{ ansible_date_time.iso8601 }}</p>
          </body>
          </html>
        dest: /tmp/nginx-content/index.html
        mode: '0644'

    - name: Verify container is running
      docker_container_info:
        name: nginx-web
      register: container_info

    - name: Display container status
      debug:
        msg: "Container {{ container_info.container.Name }} is {{ container_info.container.State.Status }}"
Subtask 2.2: Execute the Deployment Playbook
Run the playbook to deploy your first container.

# Execute the playbook
ansible-playbook -i inventory.ini deploy-nginx.yml

# Verify the container is running
docker ps

# Test the web application
curl http://localhost:8080
Subtask 2.3: Create a Multi-Container Application Playbook
Create a more complex playbook that deploys a multi-container application.

# Create a new playbook for a WordPress application
nano deploy-wordpress.yml
Add the following content:

---
- name: Deploy WordPress with MySQL using Ansible
  hosts: docker_hosts
  become: yes
  vars:
    mysql_root_password: "secure_password_123"
    wordpress_db_password: "wordpress_pass_456"
    
  tasks:
    - name: Create Docker network for WordPress
      docker_network:
        name: wordpress-network
        driver: bridge

    - name: Deploy MySQL container
      docker_container:
        name: wordpress-mysql
        image: mysql:5.7
        state: started
        restart_policy: always
        networks:
          - name: wordpress-network
        env:
          MYSQL_ROOT_PASSWORD: "{{ mysql_root_password }}"
          MYSQL_DATABASE: wordpress
          MYSQL_USER: wordpress
          MYSQL_PASSWORD: "{{ wordpress_db_password }}"
        volumes:
          - mysql-data:/var/lib/mysql

    - name: Wait for MySQL to be ready
      wait_for:
        port: 3306
        host: "{{ ansible_default_ipv4.address }}"
        delay: 10
        timeout: 60

    - name: Deploy WordPress container
      docker_container:
        name: wordpress-app
        image: wordpress:latest
        state: started
        restart_policy: always
        ports:
          - "8081:80"
        networks:
          - name: wordpress-network
        env:
          WORDPRESS_DB_HOST: wordpress-mysql:3306
          WORDPRESS_DB_USER: wordpress
          WORDPRESS_DB_PASSWORD: "{{ wordpress_db_password }}"
          WORDPRESS_DB_NAME: wordpress

    - name: Create Docker volume for MySQL data
      docker_volume:
        name: mysql-data
Subtask 2.4: Execute the Multi-Container Deployment
# Deploy the WordPress application
ansible-playbook -i inventory.ini deploy-wordpress.yml

# Verify both containers are running
docker ps

# Check the network
docker network ls
docker network inspect wordpress-network

# Test WordPress application
curl -I http://localhost:8081
Task 3: Automate Container Configuration Management with Ansible
Subtask 3.1: Create Configuration Templates
Create Ansible templates for dynamic container configuration.

# Create templates directory
mkdir templates

# Create Nginx configuration template
nano templates/nginx.conf.j2
Add the following content:

server {
    listen 80;
    server_name {{ server_name | default('localhost') }};
    
    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }
    
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
    
    # Custom headers
    add_header X-Deployed-By "Ansible-Docker-Automation";
    add_header X-Environment "{{ environment | default('development') }}";
}
Subtask 3.2: Create Advanced Configuration Playbook
# Create advanced configuration playbook
nano configure-containers.yml
Add the following content:

---
- name: Advanced Container Configuration Management
  hosts: docker_hosts
  become: yes
  vars:
    environment: "production"
    server_name: "webapp.local"
    containers:
      - name: web-app-1
        image: nginx:latest
        port: 8082
      - name: web-app-2
        image: nginx:latest
        port: 8083
      - name: web-app-3
        image: nginx:latest
        port: 8084

  tasks:
    - name: Create configuration directory
      file:
        path: /tmp/nginx-configs
        state: directory
        mode: '0755'

    - name: Generate Nginx configuration files
      template:
        src: templates/nginx.conf.j2
        dest: "/tmp/nginx-configs/{{ item.name }}.conf"
        mode: '0644'
      loop: "{{ containers }}"

    - name: Deploy multiple configured containers
      docker_container:
        name: "{{ item.name }}"
        image: "{{ item.image }}"
        state: started
        restart_policy: always
        ports:
          - "{{ item.port }}:80"
        volumes:
          - "/tmp/nginx-configs/{{ item.name }}.conf:/etc/nginx/conf.d/default.conf:ro"
          - "/tmp/nginx-content:/usr/share/nginx/html:ro"
        labels:
          environment: "{{ environment }}"
          managed_by: "ansible"
          deployment_date: "{{ ansible_date_time.epoch }}"
      loop: "{{ containers }}"

    - name: Wait for containers to be healthy
      uri:
        url: "http://localhost:{{ item.port }}/health"
        method: GET
        status_code: 200
      loop: "{{ containers }}"
      retries: 5
      delay: 3

    - name: Display container information
      docker_container_info:
        name: "{{ item.name }}"
      loop: "{{ containers }}"
      register: container_details

    - name: Show deployment summary
      debug:
        msg: |
          Container: {{ item.container.Name }}
          Status: {{ item.container.State.Status }}
          Port: {{ item.container.NetworkSettings.Ports['80/tcp'][0].HostPort }}
          Environment: {{ item.container.Config.Labels.environment }}
      loop: "{{ container_details.results }}"
Subtask 3.3: Execute Configuration Management
# Run the configuration management playbook
ansible-playbook -i inventory.ini configure-containers.yml

# Verify all containers are running
docker ps

# Test health endpoints
for port in 8082 8083 8084; do
  echo "Testing port $port:"
  curl http://localhost:$port/health
done
Task 4: Use Ansible to Scale Docker Services Across Multiple Hosts
Subtask 4.1: Prepare Multi-Host Inventory
Create an inventory file for multiple hosts simulation.

# Create multi-host inventory
nano multi-host-inventory.ini
Add the following content:

[web_servers]
web1 ansible_host=localhost ansible_port=22 ansible_connection=local
web2 ansible_host=localhost ansible_port=22 ansible_connection=local

[database_servers]
db1 ansible_host=localhost ansible_port=22 ansible_connection=local

[all:vars]
ansible_user=ubuntu
ansible_become=yes
Subtask 4.2: Create Load Balancer Configuration
# Create HAProxy configuration template
nano templates/haproxy.cfg.j2
Add the following content:

global
    daemon
    maxconn 4096

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend web_frontend
    bind *:80
    default_backend web_servers

backend web_servers
    balance roundrobin
{% for host in groups['web_servers'] %}
    server {{ host }} {{ hostvars[host]['ansible_default_ipv4']['address'] }}:8080 check
{% endfor %}

listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 30s
Subtask 4.3: Create Scaling Playbook
# Create scaling playbook
nano scale-services.yml
Add the following content:

---
- name: Scale Docker Services Across Multiple Hosts
  hosts: all
  become: yes
  vars:
    app_replicas: 2
    
  tasks:
    - name: Install Docker on all hosts
      apt:
        name: docker.io
        state: present
        update_cache: yes

    - name: Start Docker service
      systemd:
        name: docker
        state: started
        enabled: yes

    - name: Add user to docker group
      user:
        name: "{{ ansible_user }}"
        groups: docker
        append: yes

- name: Deploy Web Applications
  hosts: web_servers
  become: yes
  tasks:
    - name: Pull application image
      docker_image:
        name: nginx
        tag: latest
        source: pull

    - name: Create web content directory
      file:
        path: /opt/webapp
        state: directory
        mode: '0755'

    - name: Create custom index page
      copy:
        content: |
          <html>
          <head><title>Scaled Web App</title></head>
          <body>
            <h1>Web Server: {{ inventory_hostname }}</h1>
            <p>This is instance {{ ansible_default_ipv4.address }}</p>
            <p>Deployed at: {{ ansible_date_time.iso8601 }}</p>
          </body>
          </html>
        dest: /opt/webapp/index.html

    - name: Deploy web application containers
      docker_container:
        name: "webapp-{{ item }}"
        image: nginx:latest
        state: started
        restart_policy: always
        ports:
          - "{{ 8080 + item }}:80"
        volumes:
          - "/opt/webapp:/usr/share/nginx/html:ro"
        labels:
          service: "webapp"
          instance: "{{ item }}"
          host: "{{ inventory_hostname }}"
      loop: "{{ range(app_replicas) | list }}"

- name: Deploy Load Balancer
  hosts: database_servers
  become: yes
  tasks:
    - name: Install HAProxy
      apt:
        name: haproxy
        state: present

    - name: Generate HAProxy configuration
      template:
        src: templates/haproxy.cfg.j2
        dest: /etc/haproxy/haproxy.cfg
        backup: yes
      notify: restart haproxy

    - name: Start HAProxy service
      systemd:
        name: haproxy
        state: started
        enabled: yes

  handlers:
    - name: restart haproxy
      systemd:
        name: haproxy
        state: restarted
Subtask 4.4: Execute Scaling Deployment
# Deploy scaled services
ansible-playbook -i multi-host-inventory.ini scale-services.yml

# Verify deployment
docker ps

# Test load balancing (if HAProxy is configured)
for i in {1..5}; do
  echo "Request $i:"
  curl -s http://localhost:8080 | grep "Web Server"
done
Task 5: Integrate Ansible with CI/CD Pipelines for Automated Deployments
Subtask 5.1: Create CI/CD Pipeline Configuration
Create a simple CI/CD pipeline configuration using GitHub Actions format.

# Create .github/workflows directory
mkdir -p .github/workflows

# Create CI/CD pipeline file
nano .github/workflows/docker-deploy.yml
Add the following content:

name: Docker Deployment Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  ANSIBLE_HOST_KEY_CHECKING: False

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    
    - name: Set up Python
      uses: actions/setup-python@v2
      with:
        python-version: '3.8'
    
    - name: Install dependencies
      run: |
        pip install ansible docker
        
    - name: Lint Ansible playbooks
      run: |
        ansible-lint deploy-nginx.yml || true
        
    - name: Validate YAML syntax
      run: |
        ansible-playbook --syntax-check deploy-nginx.yml

  deploy-staging:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/develop'
    steps:
    - uses: actions/checkout@v2
    
    - name: Deploy to staging
      run: |
        ansible-playbook -i inventory.ini deploy-nginx.yml -e "environment=staging"

  deploy-production:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
    - uses: actions/checkout@v2
    
    - name: Deploy to production
      run: |
        ansible-playbook -i inventory.ini deploy-nginx.yml -e "environment=production"
Subtask 5.2: Create Deployment Script
Create a deployment script that can be used in various CI/CD systems.

# Create deployment script
nano deploy.sh
Add the following content:

#!/bin/bash

# Docker + Ansible Deployment Script
# Usage: ./deploy.sh [environment] [service]

set -e

ENVIRONMENT=${1:-development}
SERVICE=${2:-all}
INVENTORY_FILE="inventory.ini"
PLAYBOOK_DIR="."

echo "=== Docker + Ansible Deployment Script ==="
echo "Environment: $ENVIRONMENT"
echo "Service: $SERVICE"
echo "=========================================="

# Function to check prerequisites
check_prerequisites() {
    echo "Checking prerequisites..."
    
    if ! command -v ansible &> /dev/null; then
        echo "Error: Ansible is not installed"
        exit 1
    fi
    
    if ! command -v docker &> /dev/null; then
        echo "Error: Docker is not installed"
        exit 1
    fi
    
    if [ ! -f "$INVENTORY_FILE" ]; then
        echo "Error: Inventory file not found: $INVENTORY_FILE"
        exit 1
    fi
    
    echo "Prerequisites check passed!"
}

# Function to deploy specific service
deploy_service() {
    local service=$1
    local env=$2
    
    case $service in
        "nginx")
            echo "Deploying Nginx service..."
            ansible-playbook -i $INVENTORY_FILE deploy-nginx.yml -e "environment=$env"
            ;;
        "wordpress")
            echo "Deploying WordPress service..."
            ansible-playbook -i $INVENTORY_FILE deploy-wordpress.yml -e "environment=$env"
            ;;
        "scaled")
            echo "Deploying scaled services..."
            ansible-playbook -i multi-host-inventory.ini scale-services.yml -e "environment=$env"
            ;;
        "all")
            echo "Deploying all services..."
            deploy_service "nginx" "$env"
            deploy_service "wordpress" "$env"
            ;;
        *)
            echo "Unknown service: $service"
            echo "Available services: nginx, wordpress, scaled, all"
            exit 1
            ;;
    esac
}

# Function to run health checks
health_check() {
    echo "Running health checks..."
    
    # Check Nginx
    if curl -f http://localhost:8080 > /dev/null 2>&1; then
        echo "✓ Nginx service is healthy"
    else
        echo "✗ Nginx service is not responding"
    fi
    
    # Check WordPress
    if curl -f http://localhost:8081 > /dev/null 2>&1; then
        echo "✓ WordPress service is healthy"
    else
        echo "✗ WordPress service is not responding"
    fi
    
    # List running containers
    echo "Running containers:"
    docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
}

# Function to cleanup old containers
cleanup() {
    echo "Cleaning up old containers..."
    
    # Remove stopped containers
    docker container prune -f
    
    # Remove unused images
    docker image prune -f
    
    echo "Cleanup completed!"
}

# Main execution
main() {
    check_prerequisites
    
    if [ "$SERVICE" == "cleanup" ]; then
        cleanup
        exit 0
    fi
    
    deploy_service "$SERVICE" "$ENVIRONMENT"
    
    echo "Waiting for services to start..."
    sleep 10
    
    health_check
    
    echo "Deployment completed successfully!"
}

# Execute main function
main
Make the script executable:

chmod +x deploy.sh
Subtask 5.3: Create Monitoring and Rollback Playbook
# Create monitoring playbook
nano monitor-rollback.yml
Add the following content:

---
- name: Monitor and Rollback Docker Services
  hosts: docker_hosts
  become: yes
  vars:
    health_check_timeout: 30
    rollback_enabled: true
    
  tasks:
    - name: Get current container information
      docker_container_info:
        name: "{{ item }}"
      loop:
        - nginx-web
        - wordpress-app
        - wordpress-mysql
      register: current_containers
      ignore_errors: yes

    - name: Check container health
      uri:
        url: "http://localhost:{{ item.port }}"
        method: GET
        timeout: 5
      loop:
        - { name: "nginx", port: 8080 }
        - { name: "wordpress", port: 8081 }
      register: health_checks
      ignore_errors: yes

    - name: Identify unhealthy services
      set_fact:
        unhealthy_services: "{{ health_checks.results | selectattr('failed', 'equalto', true) | map(attribute='item.name') | list }}"

    - name: Display health status
      debug:
        msg: |
          Service Health Status:
          {% for check in health_checks.results %}
          - {{ check.item.name }}: {{ 'HEALTHY' if not check.failed else 'UNHEALTHY' }}
          {% endfor %}

    - name: Rollback unhealthy services
      block:
        - name: Stop unhealthy containers
          docker_container:
            name: "{{ item }}-web"
            state: stopped
          loop: "{{ unhealthy_services }}"
          when: rollback_enabled and unhealthy_services | length > 0

        - name: Deploy previous version
          docker_container:
            name: "{{ item }}-web-backup"
            image: "{{ item }}:previous"
            state: started
            restart_policy: always
            ports:
              - "{{ 8080 if item == 'nginx' else 8081 }}:80"
          loop: "{{ unhealthy_services }}"
          when: rollback_enabled and unhealthy_services | length > 0

      when: unhealthy_services | length > 0

    - name: Send notification
      debug:
        msg: |
          Deployment Status Report:
          - Total services checked: {{ health_checks.results | length }}
          - Healthy services: {{ health_checks.results | selectattr('failed', 'equalto', false) | list | length }}
          - Unhealthy services: {{ unhealthy_services | length }}
          {% if unhealthy_services | length > 0 %}
          - Rollback performed: {{ rollback_enabled }}
          {% endif %}
Subtask 5.4: Test CI/CD Integration
# Test the deployment script
./deploy.sh development nginx

# Test monitoring and rollback
ansible-playbook -i inventory.ini monitor-rollback.yml

# Simulate a complete CI/CD pipeline
echo "=== Simulating CI/CD Pipeline ==="

# Step 1: Syntax check
echo "1. Syntax validation..."
ansible-playbook --syntax-check deploy-nginx.yml

# Step 2: Deploy to staging
echo "2. Deploying to staging..."
./deploy.sh staging nginx

# Step 3: Run tests
echo "3. Running health checks..."
curl -f http://localhost:8080 && echo "✓ Health check passed"

# Step 4: Deploy to production
echo "4. Deploying to production..."
./deploy.sh production nginx

# Step 5: Final verification
echo "5. Final verification..."
ansible-playbook -i inventory.ini monitor-rollback.yml

echo "CI/CD pipeline simulation completed!"
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Docker permission denied

# Solution: Add user to docker group
sudo usermod -aG docker $USER
newgrp docker
Issue 2: Ansible cannot connect to Docker

# Solution: Install Docker Python module
pip3 install docker
Issue 3: Container port conflicts

# Solution: Check and stop conflicting containers
docker ps
docker stop <container_name>
Issue 4: Playbook syntax errors

# Solution: Validate YAML syntax
ansible-playbook --syntax-check playbook.yml
Issue 5: Container health check failures

# Solution: Check container logs
docker logs <container_name>
Lab Cleanup
To clean up the lab environment:

# Stop all containers
docker stop $(docker ps -aq)

# Remove all containers
docker rm $(docker ps -aq)

# Remove custom networks
docker network rm wordpress-network

# Remove volumes
docker volume rm mysql-data

# Remove images (optional)
docker rmi nginx wordpress mysql:5.7

# Clean up files
rm -rf ~/ansible-docker-lab
Conclusion
In this comprehensive lab, you have successfully:

• Integrated Ansible with Docker to create powerful automation workflows for container management • Created sophisticated playbooks that can deploy, configure, and manage Docker containers across multiple environments • Implemented configuration management using Ansible templates and variables for dynamic container configuration • Scaled services across multiple hosts using Ansible's inventory and group management capabilities • Built CI/CD pipeline integration with automated testing, deployment, and rollback capabilities

Why This Matters:

This lab demonstrates real-world DevOps practices that are essential in modern software development and deployment. The combination of Ansible and Docker provides:

Infrastructure as Code: All configurations are version-controlled and repeatable
Scalability: Easy horizontal scaling across multiple hosts
Consistency: Identical deployments across different environments
Automation: Reduced manual intervention and human error
Reliability: Built-in health checks and rollback mechanisms
These skills are directly applicable to Docker Certified Associate (DCA) certification objectives and are highly valued in the industry for roles involving containerization, automation, and DevOps practices.

The techniques learned in this lab form the foundation for enterprise-level container orchestration and can be extended to work with Kubernetes, Docker Swarm, and other container orchestration platforms.
