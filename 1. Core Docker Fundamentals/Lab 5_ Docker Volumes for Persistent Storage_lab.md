Lab 5: Docker Volumes for Persistent Storage
Lab Objectives
By the end of this lab, you will be able to:

Understand the fundamental difference between container storage and persistent volumes
Create and manage Docker volumes using command-line tools
Mount volumes to containers for persistent data storage
Inspect volume properties and metadata
Remove volumes safely from your Docker environment
Demonstrate how data persists across container lifecycle events
Apply volume management best practices for real-world applications
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with Linux command-line interface
Knowledge of basic file system operations
Completion of previous Docker labs or equivalent experience
Understanding of container lifecycle (create, start, stop, remove)
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker already installed. Simply click Start Lab to access your environment - no need to build your own virtual machine or install Docker manually.

Your lab environment includes:

Ubuntu Linux with Docker Engine installed
Command-line access with sudo privileges
All necessary tools and utilities pre-installed
Task 1: Understanding Container Storage vs Persistent Volumes
Subtask 1.1: Explore Container Storage Behavior
First, let's understand how container storage works by default and why we need persistent volumes.

Step 1: Create a simple container with temporary data

# Run a container and create some data inside it
docker run -it --name temp-container ubuntu:20.04 bash
Step 2: Inside the container, create a test file

# Inside the container shell
echo "This is temporary data" > /tmp/test-file.txt
cat /tmp/test-file.txt
ls -la /tmp/
exit
Step 3: Remove the container and observe data loss

# Remove the container
docker rm temp-container

# Try to run the same container again
docker run -it --name temp-container2 ubuntu:20.04 bash
Step 4: Check if the data still exists

# Inside the new container
ls -la /tmp/
cat /tmp/test-file.txt  # This will fail - file doesn't exist
exit
Step 5: Clean up

docker rm temp-container2
Subtask 1.2: Understanding the Problem
The data created in the previous steps disappeared because:

Container Storage: Data stored inside a container's filesystem is ephemeral
Container Lifecycle: When a container is removed, all data inside it is lost
No Persistence: There's no way to recover the data once the container is deleted
This is where Docker Volumes come to the rescue!

Task 2: Creating Docker Volumes
Subtask 2.1: Create Your First Docker Volume
Step 1: List existing volumes

# Check current volumes on the system
docker volume ls
Step 2: Create a new volume

# Create a volume named 'my-persistent-data'
docker volume create my-persistent-data
Step 3: Verify volume creation

# List volumes again to see the new volume
docker volume ls
Step 4: Get detailed information about the volume

# Inspect the volume properties
docker volume inspect my-persistent-data
Subtask 2.2: Understanding Volume Properties
The docker volume inspect command shows important information:

Driver: Usually 'local' for standard volumes
Mountpoint: Physical location on the host system
Name: The volume identifier
Options: Configuration settings
Scope: Availability scope (local or global)
Subtask 2.3: Create Multiple Volumes
Step 1: Create additional volumes for different purposes

# Create a volume for database data
docker volume create database-data

# Create a volume for application logs
docker volume create app-logs

# Create a volume for configuration files
docker volume create app-config
Step 2: List all volumes

docker volume ls
Task 3: Using Volumes with Containers
Subtask 3.1: Mount a Volume to a Container
Step 1: Run a container with a mounted volume

# Run Ubuntu container with volume mounted to /data directory
docker run -it --name persistent-container -v my-persistent-data:/data ubuntu:20.04 bash
Step 2: Create data in the mounted volume

# Inside the container
cd /data
echo "This data will persist!" > persistent-file.txt
echo "Container ID: $(hostname)" >> persistent-file.txt
date >> persistent-file.txt
ls -la
cat persistent-file.txt
exit
Step 3: Remove the container

docker rm persistent-container
Step 4: Create a new container with the same volume

# Mount the same volume to a new container
docker run -it --name new-persistent-container -v my-persistent-data:/data ubuntu:20.04 bash
Step 5: Verify data persistence

# Inside the new container
cd /data
ls -la
cat persistent-file.txt
# The data is still there!
exit
Subtask 3.2: Working with Multiple Volumes
Step 1: Run a container with multiple volumes

# Run container with multiple volume mounts
docker run -it --name multi-volume-container \
  -v database-data:/var/lib/database \
  -v app-logs:/var/log/app \
  -v app-config:/etc/app \
  ubuntu:20.04 bash
Step 2: Create data in different volumes

# Inside the container
# Create database data
mkdir -p /var/lib/database
echo "user_data=sample" > /var/lib/database/users.db

# Create log data
mkdir -p /var/log/app
echo "$(date): Application started" > /var/log/app/app.log

# Create configuration data
mkdir -p /etc/app
echo "debug=true" > /etc/app/config.ini
echo "port=8080" >> /etc/app/config.ini

# Verify all data
ls -la /var/lib/database/
ls -la /var/log/app/
ls -la /etc/app/
exit
Step 3: Clean up the container

docker rm multi-volume-container
Subtask 3.3: Using Volumes with Real Applications
Step 1: Run a web server with persistent storage

# Run nginx with a volume for web content
docker run -d --name web-server \
  -v my-persistent-data:/usr/share/nginx/html \
  -p 8080:80 \
  nginx:alpine
Step 2: Add content to the volume

# Run a temporary container to add content
docker run --rm -v my-persistent-data:/data ubuntu:20.04 \
  bash -c "echo '<h1>Hello from Persistent Volume!</h1>' > /data/index.html"
Step 3: Test the web server

# Check if the web server is running
curl http://localhost:8080
Step 4: Restart the web server container

# Stop and remove the container
docker stop web-server
docker rm web-server

# Start a new web server container with the same volume
docker run -d --name web-server-new \
  -v my-persistent-data:/usr/share/nginx/html \
  -p 8080:80 \
  nginx:alpine
Step 5: Verify content persistence

# The content should still be there
curl http://localhost:8080
Step 6: Clean up

docker stop web-server-new
docker rm web-server-new
Task 4: Inspecting and Managing Volumes
Subtask 4.1: Volume Inspection Commands
Step 1: Inspect volume details

# Get detailed information about a specific volume
docker volume inspect my-persistent-data
Step 2: Check volume usage

# List all volumes with their drivers
docker volume ls

# Get volume information in different formats
docker volume inspect my-persistent-data --format '{{.Mountpoint}}'
docker volume inspect my-persistent-data --format '{{.Driver}}'
Step 3: Explore volume contents from host

# Find the volume mountpoint
VOLUME_PATH=$(docker volume inspect my-persistent-data --format '{{.Mountpoint}}')
echo "Volume is mounted at: $VOLUME_PATH"

# List contents (requires sudo on most systems)
sudo ls -la $VOLUME_PATH
sudo cat $VOLUME_PATH/persistent-file.txt
Subtask 4.2: Volume Cleanup and Removal
Step 1: Remove unused volumes

# Remove volumes that are not being used by any container
docker volume prune
Step 2: Remove specific volumes

# Remove a specific volume (only works if no containers are using it)
docker volume rm app-logs
Step 3: Force remove volumes

# If you need to remove a volume that might be in use
# First, stop and remove any containers using it
docker ps -a

# Then remove the volume
docker volume rm app-config
Step 4: Remove multiple volumes

# Remove multiple volumes at once
docker volume rm database-data
Subtask 4.3: Volume Backup and Restore
Step 1: Create a backup of volume data

# Create a backup using a temporary container
docker run --rm -v my-persistent-data:/data -v $(pwd):/backup ubuntu:20.04 \
  tar czf /backup/volume-backup.tar.gz -C /data .
Step 2: Verify backup creation

ls -la volume-backup.tar.gz
Step 3: Create a new volume and restore data

# Create a new volume
docker volume create restored-volume

# Restore data to the new volume
docker run --rm -v restored-volume:/data -v $(pwd):/backup ubuntu:20.04 \
  bash -c "cd /data && tar xzf /backup/volume-backup.tar.gz"
Step 4: Verify restored data

# Check the restored data
docker run --rm -v restored-volume:/data ubuntu:20.04 \
  bash -c "ls -la /data && cat /data/persistent-file.txt"
Task 5: Demonstrating Data Persistence Across Container Restarts
Subtask 5.1: Database Persistence Example
Step 1: Run a MySQL database with persistent storage

# Create a volume for MySQL data
docker volume create mysql-data

# Run MySQL container with persistent volume
docker run -d --name mysql-db \
  -e MYSQL_ROOT_PASSWORD=mypassword \
  -e MYSQL_DATABASE=testdb \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0
Step 2: Wait for MySQL to initialize and connect

# Wait a moment for MySQL to start
sleep 30

# Connect to MySQL and create some data
docker exec -it mysql-db mysql -uroot -pmypassword testdb
Step 3: Create a table and insert data

-- Inside MySQL shell
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

INSERT INTO users (name, email) VALUES 
('John Doe', 'john@example.com'),
('Jane Smith', 'jane@example.com');

SELECT * FROM users;
exit
Step 4: Stop and remove the MySQL container

docker stop mysql-db
docker rm mysql-db
Step 5: Start a new MySQL container with the same volume

# Start new MySQL container with the same data volume
docker run -d --name mysql-db-new \
  -e MYSQL_ROOT_PASSWORD=mypassword \
  -e MYSQL_DATABASE=testdb \
  -v mysql-data:/var/lib/mysql \
  mysql:8.0
Step 6: Verify data persistence

# Wait for MySQL to start
sleep 30

# Connect and check if data is still there
docker exec -it mysql-db-new mysql -uroot -pmypassword testdb
-- Inside MySQL shell
SELECT * FROM users;
-- Your data should still be there!
exit
Subtask 5.2: Application Log Persistence
Step 1: Create a logging application

# Create a volume for logs
docker volume create app-logs-demo

# Run a container that generates logs
docker run -d --name log-generator \
  -v app-logs-demo:/var/log/app \
  ubuntu:20.04 \
  bash -c "while true; do echo \$(date): Log entry >> /var/log/app/application.log; sleep 5; done"
Step 2: Monitor log generation

# Check logs being generated
docker exec log-generator tail -f /var/log/app/application.log
# Press Ctrl+C to stop monitoring
Step 3: Restart the logging container

# Stop and remove the container
docker stop log-generator
docker rm log-generator

# Start a new container with the same volume
docker run -d --name log-generator-new \
  -v app-logs-demo:/var/log/app \
  ubuntu:20.04 \
  bash -c "while true; do echo \$(date): New container log >> /var/log/app/application.log; sleep 5; done"
Step 4: Verify log persistence and continuation

# Check that old logs are preserved and new ones are being added
docker exec log-generator-new cat /var/log/app/application.log
Subtask 5.3: Configuration Persistence
Step 1: Create a configuration volume

# Create volume for configuration
docker volume create app-config-demo

# Create initial configuration
docker run --rm -v app-config-demo:/config ubuntu:20.04 \
  bash -c "echo 'server_port=8080' > /config/app.conf && echo 'debug_mode=true' >> /config/app.conf"
Step 2: Run application with configuration

# Run application that reads configuration
docker run -d --name config-app \
  -v app-config-demo:/etc/app \
  ubuntu:20.04 \
  bash -c "while true; do echo 'Reading config:'; cat /etc/app/app.conf; sleep 10; done"
Step 3: Monitor application

# Check application output
docker logs config-app
Step 4: Update configuration

# Update configuration while app is running
docker run --rm -v app-config-demo:/config ubuntu:20.04 \
  bash -c "echo 'server_port=9090' > /config/app.conf && echo 'debug_mode=false' >> /config/app.conf"
Step 5: Restart application and verify persistence

# Restart the application
docker stop config-app
docker rm config-app

docker run -d --name config-app-new \
  -v app-config-demo:/etc/app \
  ubuntu:20.04 \
  bash -c "while true; do echo 'Reading config:'; cat /etc/app/app.conf; sleep 10; done"

# Check that updated configuration persisted
docker logs config-app-new
Troubleshooting Common Issues
Issue 1: Volume Mount Permission Problems
Problem: Permission denied when accessing volume data

Solution:

# Check volume permissions
docker run --rm -v my-persistent-data:/data ubuntu:20.04 ls -la /data

# Fix permissions if needed
docker run --rm -v my-persistent-data:/data ubuntu:20.04 chmod 755 /data
Issue 2: Volume Not Found Error
Problem: Error mounting volume that doesn't exist

Solution:

# Create the volume first
docker volume create missing-volume

# Then use it with containers
docker run -v missing-volume:/data ubuntu:20.04
Issue 3: Cannot Remove Volume
Problem: Volume is in use by container

Solution:

# Find containers using the volume
docker ps -a --filter volume=my-persistent-data

# Stop and remove containers first
docker stop container-name
docker rm container-name

# Then remove the volume
docker volume rm my-persistent-data
Lab Cleanup
Step 1: Stop all running containers

# Stop all containers created in this lab
docker stop $(docker ps -q) 2>/dev/null || true
Step 2: Remove all containers

# Remove all containers
docker rm $(docker ps -aq) 2>/dev/null || true
Step 3: Remove volumes (optional)

# List all volumes
docker volume ls

# Remove specific volumes created in this lab
docker volume rm my-persistent-data mysql-data app-logs-demo app-config-demo restored-volume 2>/dev/null || true

# Or remove all unused volumes
docker volume prune -f
Step 4: Clean up backup files

# Remove backup file created during the lab
rm -f volume-backup.tar.gz
Conclusion
Congratulations! You have successfully completed Lab 5: Docker Volumes for Persistent Storage. In this comprehensive lab, you have accomplished the following:

Key Achievements
Understanding Storage Concepts: You learned the critical difference between ephemeral container storage and persistent volumes, understanding why data disappears when containers are removed and how volumes solve this problem.

Volume Management Skills: You mastered creating, inspecting, and removing Docker volumes using command-line tools, gaining practical experience with volume lifecycle management.

Practical Implementation: You successfully mounted volumes to containers, demonstrating how to persist data across container restarts and removals.

Real-World Applications: You worked with practical examples including web servers, databases, and logging applications, showing how volumes are used in production environments.

Data Persistence Verification: You proved that data stored in volumes survives container lifecycle events, ensuring business continuity and data integrity.

Why This Matters
Production Readiness: Understanding persistent storage is crucial for deploying applications in production environments where data loss is unacceptable.

Docker Certified Associate (DCA) Preparation: The skills you've learned directly align with DCA certification requirements, particularly in the area of storage and volumes.

DevOps Best Practices: Volume management is a fundamental skill for DevOps engineers working with containerized applications.

Scalability Foundation: Persistent storage knowledge is essential for building scalable, stateful applications in containerized environments.

Next Steps
With this foundation in Docker volumes, you're now prepared to:

Explore advanced volume drivers and plugins
Implement backup and disaster recovery strategies
Work with container orchestration platforms like Docker Swarm or Kubernetes
Design persistent storage solutions for complex multi-container applications
The persistent storage concepts you've mastered in this lab form the backbone of reliable containerized applications and are essential for your continued journey in container technology and DevOps practices.
