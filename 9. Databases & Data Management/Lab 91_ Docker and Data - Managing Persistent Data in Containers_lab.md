Lab 91: Docker and Data - Managing Persistent Data in Containers
Lab Objectives
By the end of this lab, you will be able to:

Understand the concept of persistent data in Docker containers
Create and manage Docker volumes for data persistence
Mount volumes to containers and store data effectively
Ensure data persists across container restarts and removals
Implement backup and restore strategies for Docker volumes
Deploy database containers with persistent storage in production environments
Apply best practices for data management in containerized applications
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images)
Familiarity with Linux command line operations
Knowledge of basic database operations (helpful but not required)
Understanding of file system concepts
Technical Requirements:

Docker Engine installed and running
Sufficient disk space for creating volumes and containers
Network connectivity for pulling Docker images
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with Docker already installed and configured. Simply click Start Lab to access your environment. No need to build your own virtual machine or install Docker manually.

Your lab environment includes:

Ubuntu Linux with Docker Engine
Pre-pulled common Docker images
All necessary tools and utilities
Root access for system-level operations
Lab Tasks Overview
This lab is divided into five main tasks, each broken down into manageable subtasks:

Create Docker Volumes - Learn volume creation and inspection
Mount Volumes and Store Data - Practice volume mounting and data storage
Data Persistence Testing - Verify data survives container lifecycle
Backup and Restore Operations - Implement data protection strategies
Production Database Deployment - Apply concepts in real-world scenarios
Task 1: Create a Docker Volume
Understanding Docker Volumes
Docker volumes are the preferred mechanism for persisting data generated and used by Docker containers. Unlike bind mounts, volumes are completely managed by Docker and offer several advantages:

Isolation: Volumes are stored in a part of the host filesystem managed by Docker
Portability: Easier to back up or migrate than bind mounts
Performance: Better performance on Docker Desktop
Security: Can be managed using Docker CLI commands
Subtask 1.1: Create Your First Docker Volume
Let's start by creating a simple Docker volume:

# Create a new Docker volume named 'mydata'
docker volume create mydata
Subtask 1.2: Inspect the Created Volume
Examine the volume details to understand its properties:

# List all Docker volumes
docker volume ls

# Get detailed information about the specific volume
docker volume inspect mydata
The output will show important information including:

Driver: The volume driver used (usually 'local')
Mountpoint: Where the volume is stored on the host system
Name: The volume name
Scope: The scope of the volume (usually 'local')
Subtask 1.3: Create Multiple Volumes with Different Purposes
Create additional volumes for different use cases:

# Create a volume for database data
docker volume create db_data

# Create a volume for application logs
docker volume create app_logs

# Create a volume for configuration files
docker volume create app_config

# Verify all volumes are created
docker volume ls
Subtask 1.4: Understanding Volume Storage Location
Explore where Docker stores volume data on the host system:

# Find the Docker root directory
docker info | grep "Docker Root Dir"

# List the volumes directory (requires root access)
sudo ls -la /var/lib/docker/volumes/
Key Concept: Docker manages the storage location automatically, but understanding where data is stored helps with troubleshooting and system administration.

Task 2: Mount the Volume to a Container and Store Data
Understanding Volume Mounting
Volume mounting connects a Docker volume to a specific directory inside a container. This allows the container to read from and write to persistent storage.

Subtask 2.1: Mount Volume to a Simple Container
Start with a basic example using an Ubuntu container:

# Run an Ubuntu container with the volume mounted
docker run -it --name data-container \
  --mount source=mydata,target=/app/data \
  ubuntu:latest bash
Command Explanation:

--mount source=mydata,target=/app/data: Mounts the 'mydata' volume to '/app/data' inside the container
--name data-container: Assigns a name to the container for easy reference
Subtask 2.2: Create and Store Data in the Mounted Volume
Inside the running container, create some test data:

# Create a directory structure
mkdir -p /app/data/documents
mkdir -p /app/data/images

# Create some test files
echo "This is my first persistent file" > /app/data/documents/readme.txt
echo "Application configuration data" > /app/data/config.json
echo "Log entry: $(date)" > /app/data/app.log

# Create a larger file to simulate real data
dd if=/dev/zero of=/app/data/images/testfile.img bs=1M count=10

# List the created files
ls -la /app/data/
ls -la /app/data/documents/
Subtask 2.3: Verify Data from Host System
Open a new terminal (keep the container running) and inspect the volume from the host:

# Inspect the volume to see the mountpoint
docker volume inspect mydata

# View the data from the host system (requires root access)
sudo ls -la /var/lib/docker/volumes/mydata/_data/
sudo cat /var/lib/docker/volumes/mydata/_data/documents/readme.txt
Subtask 2.4: Exit Container and Verify Data Persistence
Exit the container and check if data remains:

# Exit the container (from inside the container)
exit

# Check container status
docker ps -a

# Start a new container with the same volume
docker run --rm -it \
  --mount source=mydata,target=/data \
  ubuntu:latest \
  ls -la /data/
Important Note: The data persists even though we're using a different mount point (/data instead of /app/data).

Task 3: Persist Data Across Container Restarts
Understanding Container Lifecycle and Data Persistence
This task demonstrates how Docker volumes maintain data integrity through various container lifecycle events.

Subtask 3.1: Create a Web Application with Persistent Data
Create a simple web application that stores data:

# Create a directory for our web application
mkdir -p ~/docker-lab/webapp
cd ~/docker-lab/webapp

# Create a simple HTML file
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Persistent Data Demo</title>
</head>
<body>
    <h1>Docker Volume Persistence Test</h1>
    <p>This page is served from a persistent volume.</p>
    <p>Last updated: <span id="timestamp"></span></p>
    <script>
        document.getElementById('timestamp').textContent = new Date().toLocaleString();
    </script>
</body>
</html>
EOF
Subtask 3.2: Run Web Server with Volume Mount
Deploy the web application using nginx with a mounted volume:

# Create a volume for web content
docker volume create web_content

# Copy our HTML file to the volume using a temporary container
docker run --rm -v web_content:/data -v $(pwd):/source ubuntu:latest \
  cp /source/index.html /data/

# Run nginx web server with the volume mounted
docker run -d --name web-server \
  -p 8080:80 \
  --mount source=web_content,target=/usr/share/nginx/html \
  nginx:latest
Subtask 3.3: Test the Web Application
Verify the web application is working:

# Check if the container is running
docker ps

# Test the web application
curl http://localhost:8080

# Or use wget if curl is not available
wget -qO- http://localhost:8080
Subtask 3.4: Modify Data and Test Persistence
Update the web content and test persistence through container restarts:

# Update the HTML file with new content
cat > updated.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Updated Persistent Data Demo</title>
    <style>
        body { font-family: Arial, sans-serif; background-color: #f0f0f0; }
        .container { max-width: 800px; margin: 0 auto; padding: 20px; }
        .update-info { background-color: #d4edda; padding: 10px; border-radius: 5px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Updated Docker Volume Persistence Test</h1>
        <div class="update-info">
            <p><strong>Update:</strong> This content was modified while the container was running!</p>
            <p>Last updated: <span id="timestamp"></span></p>
        </div>
        <p>This demonstrates that data in Docker volumes persists across container lifecycle events.</p>
    </div>
    <script>
        document.getElementById('timestamp').textContent = new Date().toLocaleString();
    </script>
</body>
</html>
EOF

# Update the content in the volume
docker run --rm -v web_content:/data -v $(pwd):/source ubuntu:latest \
  cp /source/updated.html /data/index.html

# Test the updated content
curl http://localhost:8080
Subtask 3.5: Test Persistence Through Container Restart
Restart the container and verify data persistence:

# Stop the web server container
docker stop web-server

# Remove the container
docker rm web-server

# Start a new container with the same volume
docker run -d --name web-server-v2 \
  -p 8080:80 \
  --mount source=web_content,target=/usr/share/nginx/html \
  nginx:latest

# Verify the data persisted
curl http://localhost:8080

# Check that it's a completely new container
docker ps
Key Learning: The data in the volume persists even when containers are stopped, removed, and recreated.

Task 4: Back Up and Restore Data from Docker Volumes
Understanding Volume Backup Strategies
Backing up Docker volumes is crucial for data protection and disaster recovery. Docker provides several methods for backing up volume data.

Subtask 4.1: Create Test Data for Backup
First, let's create comprehensive test data:

# Create a volume for backup testing
docker volume create backup_test

# Create a container to populate the volume with test data
docker run --rm -it \
  --mount source=backup_test,target=/data \
  ubuntu:latest bash -c "
    mkdir -p /data/documents /data/configs /data/logs
    echo 'Important document content' > /data/documents/important.txt
    echo 'Database configuration' > /data/configs/db.conf
    echo 'Application settings' > /data/configs/app.conf
    for i in {1..5}; do
      echo 'Log entry \$i: \$(date)' >> /data/logs/app.log
    done
    echo 'Backup test data created successfully'
    ls -la /data/
    find /data -type f -exec wc -l {} +
  "
Subtask 4.2: Create Volume Backup
Create a backup of the volume data:

# Create a backup directory on the host
mkdir -p ~/docker-backups

# Create a backup using tar
docker run --rm \
  --mount source=backup_test,target=/data \
  -v ~/docker-backups:/backup \
  ubuntu:latest \
  tar czf /backup/backup_test_$(date +%Y%m%d_%H%M%S).tar.gz -C /data .

# Verify the backup was created
ls -la ~/docker-backups/
Subtask 4.3: Alternative Backup Method Using Docker CP
Demonstrate another backup approach:

# Create a temporary container to access the volume
docker create --name temp_backup \
  --mount source=backup_test,target=/data \
  ubuntu:latest

# Copy data from the volume to host
docker cp temp_backup:/data ~/docker-backups/volume_copy

# Clean up the temporary container
docker rm temp_backup

# Verify the copied data
ls -la ~/docker-backups/volume_copy/
Subtask 4.4: Simulate Data Loss and Restore
Simulate data loss and perform a restore operation:

# Remove the original volume to simulate data loss
docker volume rm backup_test

# Verify the volume is gone
docker volume ls | grep backup_test || echo "Volume successfully removed"

# Create a new volume for restoration
docker volume create backup_test_restored

# Restore data from backup
docker run --rm \
  --mount source=backup_test_restored,target=/data \
  -v ~/docker-backups:/backup \
  ubuntu:latest \
  tar xzf /backup/backup_test_*.tar.gz -C /data

# Verify the restored data
docker run --rm \
  --mount source=backup_test_restored,target=/data \
  ubuntu:latest \
  bash -c "
    echo 'Restored data verification:'
    ls -la /data/
    echo '--- Document content ---'
    cat /data/documents/important.txt
    echo '--- Configuration files ---'
    ls -la /data/configs/
    echo '--- Log entries ---'
    wc -l /data/logs/app.log
  "
Subtask 4.5: Automated Backup Script
Create a reusable backup script:

# Create a backup script
cat > ~/docker-backups/backup_volume.sh << 'EOF'
#!/bin/bash

# Docker Volume Backup Script
# Usage: ./backup_volume.sh <volume_name> [backup_directory]

VOLUME_NAME=$1
BACKUP_DIR=${2:-~/docker-backups}
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/${VOLUME_NAME}_${TIMESTAMP}.tar.gz"

# Validate input
if [ -z "$VOLUME_NAME" ]; then
    echo "Usage: $0 <volume_name> [backup_directory]"
    exit 1
fi

# Check if volume exists
if ! docker volume inspect "$VOLUME_NAME" > /dev/null 2>&1; then
    echo "Error: Volume '$VOLUME_NAME' does not exist"
    exit 1
fi

# Create backup directory if it doesn't exist
mkdir -p "$BACKUP_DIR"

# Create backup
echo "Creating backup of volume '$VOLUME_NAME'..."
docker run --rm \
    --mount source="$VOLUME_NAME",target=/data \
    -v "$BACKUP_DIR":/backup \
    ubuntu:latest \
    tar czf "/backup/$(basename "$BACKUP_FILE")" -C /data .

if [ $? -eq 0 ]; then
    echo "Backup created successfully: $BACKUP_FILE"
    echo "Backup size: $(du -h "$BACKUP_FILE" | cut -f1)"
else
    echo "Backup failed!"
    exit 1
fi
EOF

# Make the script executable
chmod +x ~/docker-backups/backup_volume.sh

# Test the backup script
~/docker-backups/backup_volume.sh backup_test_restored
Task 5: Use Docker Volumes in a Production Environment for Database Containers
Understanding Production Database Requirements
Production database deployments require careful consideration of data persistence, performance, backup strategies, and security. This task demonstrates best practices for deploying databases with Docker volumes.

Subtask 5.1: Deploy MySQL Database with Persistent Storage
Set up a production-ready MySQL database:

# Create dedicated volumes for MySQL
docker volume create mysql_data
docker volume create mysql_config
docker volume create mysql_logs

# Create a custom MySQL configuration
mkdir -p ~/docker-lab/mysql-config
cat > ~/docker-lab/mysql-config/my.cnf << 'EOF'
[mysqld]
# Basic settings
user = mysql
pid-file = /var/run/mysqld/mysqld.pid
socket = /var/run/mysqld/mysqld.sock
port = 3306
basedir = /usr
datadir = /var/lib/mysql
tmpdir = /tmp

# Performance settings
innodb_buffer_pool_size = 256M
innodb_log_file_size = 64M
innodb_flush_log_at_trx_commit = 1
innodb_flush_method = O_DIRECT

# Logging
log-error = /var/log/mysql/error.log
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2

# Security settings
bind-address = 0.0.0.0
skip-name-resolve

[client]
port = 3306
socket = /var/run/mysqld/mysqld.sock
EOF

# Deploy MySQL with persistent volumes
docker run -d \
  --name mysql-prod \
  --restart unless-stopped \
  -e MYSQL_ROOT_PASSWORD=SecureRootPass123! \
  -e MYSQL_DATABASE=production_db \
  -e MYSQL_USER=app_user \
  -e MYSQL_PASSWORD=AppUserPass123! \
  -p 3306:3306 \
  --mount source=mysql_data,target=/var/lib/mysql \
  --mount source=mysql_logs,target=/var/log/mysql \
  -v ~/docker-lab/mysql-config/my.cnf:/etc/mysql/conf.d/custom.cnf:ro \
  mysql:8.0
Subtask 5.2: Verify Database Deployment and Create Test Data
Test the database deployment and create sample data:

# Wait for MySQL to start up
echo "Waiting for MySQL to start..."
sleep 30

# Check if MySQL is running
docker ps | grep mysql-prod

# Connect to MySQL and create test data
docker exec -it mysql-prod mysql -u app_user -pAppUserPass123! production_db << 'EOF'
-- Create a sample table
CREATE TABLE users (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO users (username, email) VALUES
('john_doe', 'john@example.com'),
('jane_smith', 'jane@example.com'),
('bob_wilson', 'bob@example.com'),
('alice_brown', 'alice@example.com'),
('charlie_davis', 'charlie@example.com');

-- Create an orders table
CREATE TABLE orders (
    id INT AUTO_INCREMENT PRIMARY KEY,
    user_id INT,
    product_name VARCHAR(100),
    quantity INT,
    price DECIMAL(10,2),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Insert sample orders
INSERT INTO orders (user_id, product_name, quantity, price) VALUES
(1, 'Laptop Computer', 1, 999.99),
(2, 'Wireless Mouse', 2, 29.99),
(3, 'Keyboard', 1, 79.99),
(1, 'Monitor', 1, 299.99),
(4, 'USB Cable', 3, 12.99);

-- Verify data
SELECT COUNT(*) as user_count FROM users;
SELECT COUNT(*) as order_count FROM orders;
SELECT u.username, COUNT(o.id) as order_count, SUM(o.price * o.quantity) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.username;
EOF
Subtask 5.3: Deploy PostgreSQL with Advanced Volume Configuration
Set up a PostgreSQL database with multiple volumes:

# Create volumes for PostgreSQL
docker volume create postgres_data
docker volume create postgres_backup
docker volume create postgres_logs

# Create PostgreSQL initialization script
mkdir -p ~/docker-lab/postgres-init
cat > ~/docker-lab/postgres-init/init.sql << 'EOF'
-- Create application database
CREATE DATABASE app_production;

-- Create application user
CREATE USER app_user WITH ENCRYPTED PASSWORD 'AppUserSecurePass123!';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE app_production TO app_user;

-- Connect to the application database
\c app_production;

-- Create application schema
CREATE SCHEMA IF NOT EXISTS app_schema;

-- Grant schema privileges
GRANT ALL ON SCHEMA app_schema TO app_user;

-- Create sample tables
CREATE TABLE app_schema.products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    category VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE app_schema.inventory (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES app_schema.products(id),
    quantity INTEGER NOT NULL DEFAULT 0,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO app_schema.products (name, description, price, category) VALUES
('Gaming Laptop', 'High-performance laptop for gaming', 1299.99, 'Electronics'),
('Office Chair', 'Ergonomic office chair', 299.99, 'Furniture'),
('Coffee Maker', 'Automatic drip coffee maker', 89.99, 'Appliances'),
('Bluetooth Headphones', 'Wireless noise-canceling headphones', 199.99, 'Electronics'),
('Standing Desk', 'Adjustable height standing desk', 449.99, 'Furniture');

INSERT INTO app_schema.inventory (product_id, quantity) VALUES
(1, 15), (2, 8), (3, 25), (4, 12), (5, 6);
EOF

# Deploy PostgreSQL
docker run -d \
  --name postgres-prod \
  --restart unless-stopped \
  -e POSTGRES_DB=postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=PostgresRootPass123! \
  -p 5432:5432 \
  --mount source=postgres_data,target=/var/lib/postgresql/data \
  --mount source=postgres_backup,target=/backup \
  -v ~/docker-lab/postgres-init:/docker-entrypoint-initdb.d:ro \
  postgres:15
Subtask 5.4: Implement Database Backup Strategy
Create automated backup solutions for both databases:

# Create backup scripts directory
mkdir -p ~/docker-lab/backup-scripts

# MySQL backup script
cat > ~/docker-lab/backup-scripts/mysql_backup.sh << 'EOF'
#!/bin/bash

# MySQL Backup Script for Production
BACKUP_DIR="/backup/mysql"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
CONTAINER_NAME="mysql-prod"
DATABASE="production_db"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Create database backup
echo "Creating MySQL backup..."
docker exec "$CONTAINER_NAME" mysqldump \
  -u root -pSecureRootPass123! \
  --single-transaction \
  --routines \
  --triggers \
  "$DATABASE" > "$BACKUP_DIR/mysql_backup_$TIMESTAMP.sql"

# Compress the backup
gzip "$BACKUP_DIR/mysql_backup_$TIMESTAMP.sql"

# Create volume backup
echo "Creating volume backup..."
docker run --rm \
  --mount source=mysql_data,target=/data \
  -v ~/docker-backups:/backup \
  ubuntu:latest \
  tar czf "/backup/mysql_volume_$TIMESTAMP.tar.gz" -C /data .

echo "MySQL backup completed: mysql_backup_$TIMESTAMP.sql.gz"
echo "Volume backup completed: mysql_volume_$TIMESTAMP.tar.gz"

# Clean up old backups (keep last 7 days)
find "$BACKUP_DIR" -name "mysql_backup_*.sql.gz" -mtime +7 -delete
find ~/docker-backups -name "mysql_volume_*.tar.gz" -mtime +7 -delete
EOF

# PostgreSQL backup script
cat > ~/docker-lab/backup-scripts/postgres_backup.sh << 'EOF'
#!/bin/bash

# PostgreSQL Backup Script for Production
BACKUP_DIR="/backup/postgres"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
CONTAINER_NAME="postgres-prod"

# Create backup directory
mkdir -p "$BACKUP_DIR"

# Create database backup
echo "Creating PostgreSQL backup..."
docker exec "$CONTAINER_NAME" pg_dumpall \
  -U postgres > "$BACKUP_DIR/postgres_backup_$TIMESTAMP.sql"

# Compress the backup
gzip "$BACKUP_DIR/postgres_backup_$TIMESTAMP.sql"

# Create volume backup
echo "Creating volume backup..."
docker run --rm \
  --mount source=postgres_data,target=/data \
  -v ~/docker-backups:/backup \
  ubuntu:latest \
  tar czf "/backup/postgres_volume_$TIMESTAMP.tar.gz" -C /data .

echo "PostgreSQL backup completed: postgres_backup_$TIMESTAMP.sql.gz"
echo "Volume backup completed: postgres_volume_$TIMESTAMP.tar.gz"

# Clean up old backups (keep last 7 days)
find "$BACKUP_DIR" -name "postgres_backup_*.sql.gz" -mtime +7 -delete
find ~/docker-backups -name "postgres_volume_*.tar.gz" -mtime +7 -delete
EOF

# Make scripts executable
chmod +x ~/docker-lab/backup-scripts/*.sh

# Create backup directories
mkdir -p /backup/mysql /backup/postgres

# Test the backup scripts
echo "Testing MySQL backup..."
~/docker-lab/backup-scripts/mysql_backup.sh

echo "Waiting for PostgreSQL to be ready..."
sleep 30

echo "Testing PostgreSQL backup..."
~/docker-lab/backup-scripts/postgres_backup.sh
Subtask 5.5: Monitor Database Performance and Volume Usage
Implement monitoring for the production databases:

# Create monitoring script
cat > ~/docker-lab/backup-scripts/db_monitor.sh << 'EOF'
#!/bin/bash

echo "=== Docker Volume Usage Report ==="
echo "Generated: $(date)"
echo

# Check volume usage
echo "--- Volume Usage ---"
docker system df -v

echo
echo "--- Database Container Status ---"
docker ps --filter "name=mysql-prod" --filter "name=postgres-prod" \
  --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

echo
echo "--- MySQL Database Status ---"
if docker ps --filter "name=mysql-prod" --format "{{.Names}}" | grep -q mysql-prod; then
    docker exec mysql-prod mysql -u root -pSecureRootPass123! -e "
        SELECT 
            table_schema as 'Database',
            ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) as 'Size (MB)'
        FROM information_schema.tables 
        WHERE table_schema = 'production_db'
        GROUP BY table_schema;
    "
    
    echo "--- MySQL Connection Count ---"
    docker exec mysql-prod mysql -u root -pSecureRootPass123! -e "
        SHOW STATUS LIKE 'Threads_connected';
        SHOW STATUS LIKE 'Max_used_connections';
    "
fi

echo
echo "--- PostgreSQL Database Status ---"
if docker ps --filter "name=postgres-prod" --format "{{.Names}}" | grep -q postgres-prod; then
    docker exec postgres-prod psql -U postgres -c "
        SELECT 
            datname as database,
            pg_size_pretty(pg_database_size(datname)) as size
        FROM pg_database 
        WHERE datname = 'app_production';
    "
    
    echo "--- PostgreSQL Connection Count ---"
    docker exec postgres-prod psql -U postgres -c "
        SELECT count(*) as active_connections 
        FROM pg_stat_activity 
        WHERE state = 'active';
    "
fi

echo
echo "--- Volume Backup Status ---"
ls -lh ~/docker-backups/ | grep -E "(mysql|postgres)"
ls -lh /backup/mysql/ 2>/dev/null | tail -5
ls -lh /backup/postgres/ 2>/dev/null | tail -5
EOF

chmod +x ~/docker-lab/backup-scripts/db_monitor.sh

# Run the monitoring script
~/docker-lab/backup-scripts/db_monitor.sh
Subtask 5.6: Test Data Persistence in Production Environment
Verify that data persists through various scenarios:

# Test 1: Container restart
echo "=== Testing Container Restart ==="
docker restart mysql-prod postgres-prod

# Wait for databases to restart
sleep 30

# Verify data integrity
echo "Verifying MySQL data after restart..."
docker exec mysql-prod mysql -u app_user -pAppUserPass123! production_db -e "
    SELECT COUNT(*) as user_count FROM users;
    SELECT COUNT(*) as order_count FROM orders;
"

echo "Verifying PostgreSQL data after restart..."
docker exec postgres-prod psql -U app_user -d app_production -c "
    SELECT COUNT(*) as product_count FROM app_schema.products;
    SELECT COUNT(*) as inventory_count FROM app_schema.inventory;
"

# Test 2: Container recreation
echo
echo "=== Testing Container Recreation ==="

# Stop and remove MySQL container
docker stop mysql-prod
docker rm mysql-prod

# Recreate MySQL container with same volumes
docker run -d \
  --name mysql-prod-new \
  --restart unless-stopped \
  -e MYSQL_ROOT_PASSWORD=SecureRootPass123! \
  -e MYSQL_DATABASE=production_db \
  -e MYSQL_USER=app_user \
  -e MYSQL_PASSWORD=AppUserPass123! \
  -p 3306:3306 \
  --mount source=mysql_data,target=/var/lib/mysql \
  --mount source=mysql_logs,target=/var/log/mysql \
  -v ~/docker-lab/mysql-config/my.cnf:/etc/mysql/conf.d/custom.cnf:ro \
  mysql:8.0

# Wait for MySQL to start
sleep 30

# Verify data persisted
echo "Verifying MySQL data after container recreation..."
docker exec mysql-prod-new mysql -u app_user -pAppUserPass123! production_db -e "
    SELECT u.username, COUNT(o.id) as order_count 
    FROM users u 
    LEFT JOIN orders o ON u.id = o.user_id 
    GROUP BY u.username;
"

echo
echo "=== Production Database Deployment Complete ==="
echo "Both MySQL and PostgreSQL are running with persistent storage"
echo "Data has been verified to persist through container lifecycle events"
echo "Backup scripts are in place for data protection"
echo "Monitoring tools are available for ongoing maintenance"
Troubleshooting Common Issues
Volume-Related Issues
Issue: Volume not mounting correctly

# Check if volume exists
docker volume ls | grep volume_name

# Inspect volume details
docker volume inspect volume_name

# Check container mount points
docker inspect container_name | grep -A 10 "Mounts"
Issue: Permission denied when accessing volume data

# Check volume permissions from inside container
docker exec container_name ls -la /mount/point

# Fix permissions if needed
docker exec container_name chown -R user:group /mount/point
Issue: Volume backup fails

# Check available disk space
df -h

# Verify backup directory permissions
ls -la ~/docker-backups/

# Test backup command manually
docker run --rm --mount source=volume_name,target=/data ubuntu:latest ls -la /data
Database-Related Issues
Issue: Database container won't start

# Check container logs
docker logs container_name

# Check if port is already in use
netstat -tlnp | grep :3306
netstat -tlnp | grep :5432

# Verify environment variables
docker inspect container_name | grep -A 20 "Env"
Issue: Cannot connect to database

# Test network connectivity
docker exec container_name netstat -tlnp

# Check database process
docker exec container_name ps aux | grep mysql
docker exec container_name ps aux | grep postgres

# Test connection from inside container
docker exec -it mysql_container mysql -u root -p
docker exec -it postgres_container psql -U postgres
Lab Summary and Key Takeaways
What You Accomplished
In
