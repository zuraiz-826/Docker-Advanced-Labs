Lab 24: Docker and Databases - Running a MySQL Container
Objectives
By the end of this lab, you will be able to:

Pull and run a MySQL Docker image from Docker Hub
Create and configure a MySQL container with custom network settings
Initialize a MySQL database with a custom schema
Implement data persistence using Docker volumes
Backup and restore MySQL data effectively
Create and connect a MySQL client container to interact with the database
Understand best practices for running databases in Docker containers
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, volumes)
Familiarity with command-line interface
Basic knowledge of SQL and database concepts
Understanding of networking fundamentals
Note: Al Nafi provides ready-to-use Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to begin - no need to build your own virtual machine.

Lab Environment Setup
Your cloud machine comes pre-configured with:

Docker Engine (latest stable version)
Docker Compose
Basic networking tools
Text editors (nano, vim)
Task 1: Pull and Run a MySQL Docker Image
Subtask 1.1: Pull the MySQL Image
First, let's pull the official MySQL image from Docker Hub.

# Pull the latest MySQL image
docker pull mysql:8.0

# Verify the image has been downloaded
docker images | grep mysql
Subtask 1.2: Run Your First MySQL Container
Let's create a basic MySQL container to understand the fundamentals.

# Run a MySQL container with basic configuration
docker run --name mysql-basic \
  -e MYSQL_ROOT_PASSWORD=mypassword123 \
  -d mysql:8.0

# Check if the container is running
docker ps
Subtask 1.3: Test the Basic Connection
# Check container logs to ensure MySQL is ready
docker logs mysql-basic

# Wait for the message "ready for connections" before proceeding
Task 2: Create a Docker Container for MySQL with Custom Network
Subtask 2.1: Create a Custom Docker Network
# Create a custom bridge network for our database setup
docker network create mysql-network

# Verify the network was created
docker network ls
Subtask 2.2: Stop and Remove the Basic Container
# Stop the basic container
docker stop mysql-basic

# Remove the basic container
docker rm mysql-basic
Subtask 2.3: Create MySQL Container with Custom Network
# Create a MySQL container connected to our custom network
docker run --name mysql-server \
  --network mysql-network \
  -e MYSQL_ROOT_PASSWORD=SecurePass123 \
  -e MYSQL_DATABASE=company_db \
  -e MYSQL_USER=app_user \
  -e MYSQL_PASSWORD=AppPass456 \
  -p 3306:3306 \
  -d mysql:8.0

# Verify the container is running on the custom network
docker network inspect mysql-network
Subtask 2.4: Verify MySQL Server Status
# Check container status
docker ps

# Monitor MySQL startup logs
docker logs -f mysql-server
Wait until you see the message indicating MySQL is ready for connections before proceeding.

Task 3: Initialize the Database with Custom Schema
Subtask 3.1: Create SQL Schema File
Create a directory for our database files and schema:

# Create a directory for our database files
mkdir -p ~/mysql-lab/sql-scripts

# Create the schema file
cat > ~/mysql-lab/sql-scripts/init-schema.sql << 'EOF'
-- Company Database Schema
USE company_db;

-- Create employees table
CREATE TABLE employees (
    id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    department VARCHAR(50),
    salary DECIMAL(10,2),
    hire_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create departments table
CREATE TABLE departments (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    manager_id INT,
    budget DECIMAL(12,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO departments (name, budget) VALUES 
('Engineering', 500000.00),
('Marketing', 200000.00),
('Sales', 300000.00),
('HR', 150000.00);

INSERT INTO employees (first_name, last_name, email, department, salary, hire_date) VALUES 
('John', 'Doe', 'john.doe@company.com', 'Engineering', 75000.00, '2023-01-15'),
('Jane', 'Smith', 'jane.smith@company.com', 'Marketing', 65000.00, '2023-02-20'),
('Mike', 'Johnson', 'mike.johnson@company.com', 'Sales', 70000.00, '2023-03-10'),
('Sarah', 'Wilson', 'sarah.wilson@company.com', 'HR', 60000.00, '2023-04-05');

-- Create a view for employee summary
CREATE VIEW employee_summary AS
SELECT 
    CONCAT(first_name, ' ', last_name) as full_name,
    email,
    department,
    salary,
    hire_date
FROM employees
ORDER BY hire_date;
EOF
Subtask 3.2: Execute the Schema
# Copy the SQL file into the container
docker cp ~/mysql-lab/sql-scripts/init-schema.sql mysql-server:/tmp/

# Execute the schema file
docker exec -it mysql-server mysql -u root -pSecurePass123 -e "source /tmp/init-schema.sql"

# Verify the schema was created
docker exec -it mysql-server mysql -u root -pSecurePass123 -e "USE company_db; SHOW TABLES;"
Subtask 3.3: Verify Data Insertion
# Check if data was inserted correctly
docker exec -it mysql-server mysql -u root -pSecurePass123 -e "USE company_db; SELECT * FROM employees;"

# Check the view
docker exec -it mysql-server mysql -u root -pSecurePass123 -e "USE company_db; SELECT * FROM employee_summary;"
Task 4: Backup and Restore MySQL Data Using Volumes
Subtask 4.1: Create a New MySQL Container with Volume
First, let's create a new MySQL container with persistent storage:

# Stop the current container
docker stop mysql-server

# Create a directory for persistent data
mkdir -p ~/mysql-lab/mysql-data
mkdir -p ~/mysql-lab/backups

# Run MySQL with volume mounting
docker run --name mysql-persistent \
  --network mysql-network \
  -e MYSQL_ROOT_PASSWORD=SecurePass123 \
  -e MYSQL_DATABASE=company_db \
  -e MYSQL_USER=app_user \
  -e MYSQL_PASSWORD=AppPass456 \
  -v ~/mysql-lab/mysql-data:/var/lib/mysql \
  -v ~/mysql-lab/backups:/backups \
  -p 3306:3306 \
  -d mysql:8.0

# Wait for MySQL to be ready
sleep 30
Subtask 4.2: Restore Data to New Container
# Copy and execute our schema in the new container
docker cp ~/mysql-lab/sql-scripts/init-schema.sql mysql-persistent:/tmp/
docker exec -it mysql-persistent mysql -u root -pSecurePass123 -e "source /tmp/init-schema.sql"
Subtask 4.3: Create Database Backup
# Create a full database backup
docker exec mysql-persistent mysqldump -u root -pSecurePass123 --all-databases > ~/mysql-lab/backups/full-backup-$(date +%Y%m%d-%H%M%S).sql

# Create a specific database backup
docker exec mysql-persistent mysqldump -u root -pSecurePass123 company_db > ~/mysql-lab/backups/company-db-backup-$(date +%Y%m%d-%H%M%S).sql

# Verify backup files were created
ls -la ~/mysql-lab/backups/
Subtask 4.4: Test Data Persistence
# Add some test data
docker exec -it mysql-persistent mysql -u root -pSecurePass123 -e "
USE company_db; 
INSERT INTO employees (first_name, last_name, email, department, salary, hire_date) 
VALUES ('Test', 'User', 'test.user@company.com', 'Engineering', 80000.00, CURDATE());"

# Stop and remove the container
docker stop mysql-persistent
docker rm mysql-persistent

# Start a new container with the same volume
docker run --name mysql-restored \
  --network mysql-network \
  -e MYSQL_ROOT_PASSWORD=SecurePass123 \
  -v ~/mysql-lab/mysql-data:/var/lib/mysql \
  -p 3306:3306 \
  -d mysql:8.0

# Wait for startup
sleep 30

# Verify data persistence
docker exec -it mysql-restored mysql -u root -pSecurePass123 -e "USE company_db; SELECT * FROM employees WHERE first_name='Test';"
Subtask 4.5: Restore from Backup
# Create a new container for restore testing
docker run --name mysql-restore-test \
  --network mysql-network \
  -e MYSQL_ROOT_PASSWORD=SecurePass123 \
  -p 3307:3306 \
  -d mysql:8.0

# Wait for startup
sleep 30

# Find the latest backup file
BACKUP_FILE=$(ls -t ~/mysql-lab/backups/company-db-backup-*.sql | head -1)

# Restore from backup
docker exec -i mysql-restore-test mysql -u root -pSecurePass123 -e "CREATE DATABASE company_db;"
docker exec -i mysql-restore-test mysql -u root -pSecurePass123 company_db < "$BACKUP_FILE"

# Verify restore
docker exec -it mysql-restore-test mysql -u root -pSecurePass123 -e "USE company_db; SELECT COUNT(*) as employee_count FROM employees;"
Task 5: Create a MySQL Client Container and Connect to Database
Subtask 5.1: Create MySQL Client Container
# Create a MySQL client container
docker run --name mysql-client \
  --network mysql-network \
  -it --rm mysql:8.0 mysql -h mysql-restored -u root -pSecurePass123
This will open an interactive MySQL session. Try these commands:

-- Show databases
SHOW DATABASES;

-- Use our company database
USE company_db;

-- Show tables
SHOW TABLES;

-- Run a query
SELECT department, COUNT(*) as employee_count, AVG(salary) as avg_salary 
FROM employees 
GROUP BY department;

-- Exit the client
EXIT;
Subtask 5.2: Create a Persistent Client Container
# Create a client container that stays running
docker run --name mysql-admin-client \
  --network mysql-network \
  -d mysql:8.0 tail -f /dev/null

# Use the client container to run commands
docker exec -it mysql-admin-client mysql -h mysql-restored -u app_user -pAppPass456 company_db
In the MySQL session, try these operations:

-- Test user permissions
SELECT * FROM employees LIMIT 5;

-- Try to create a new employee record
INSERT INTO employees (first_name, last_name, email, department, salary, hire_date) 
VALUES ('Client', 'Test', 'client.test@company.com', 'Engineering', 85000.00, CURDATE());

-- Verify the insertion
SELECT * FROM employees WHERE first_name = 'Client';

-- Exit
EXIT;
Subtask 5.3: Create a Monitoring Script
Create a script to monitor database status:

# Create a monitoring script
cat > ~/mysql-lab/monitor-db.sh << 'EOF'
#!/bin/bash

echo "=== MySQL Database Monitoring ==="
echo "Date: $(date)"
echo

# Check container status
echo "Container Status:"
docker ps --filter name=mysql-restored --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
echo

# Check database connections
echo "Active Connections:"
docker exec mysql-restored mysql -u root -pSecurePass123 -e "SHOW PROCESSLIST;" 2>/dev/null
echo

# Check database sizes
echo "Database Sizes:"
docker exec mysql-restored mysql -u root -pSecurePass123 -e "
SELECT 
    table_schema as 'Database',
    ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) as 'Size (MB)'
FROM information_schema.tables 
GROUP BY table_schema;" 2>/dev/null
echo

# Check table row counts
echo "Table Row Counts (company_db):"
docker exec mysql-restored mysql -u root -pSecurePass123 -e "
USE company_db;
SELECT 
    TABLE_NAME as 'Table',
    TABLE_ROWS as 'Rows'
FROM information_schema.TABLES 
WHERE TABLE_SCHEMA = 'company_db';" 2>/dev/null

EOF

# Make the script executable
chmod +x ~/mysql-lab/monitor-db.sh

# Run the monitoring script
~/mysql-lab/monitor-db.sh
Troubleshooting Common Issues
Issue 1: Container Won't Start
# Check container logs
docker logs mysql-restored

# Common solutions:
# 1. Ensure port 3306 is not already in use
sudo netstat -tlnp | grep 3306

# 2. Check if another MySQL container is running
docker ps | grep mysql

# 3. Remove conflicting containers
docker rm -f $(docker ps -aq --filter name=mysql)
Issue 2: Connection Refused
# Wait for MySQL to fully initialize
docker logs -f mysql-restored

# Look for "ready for connections" message
# This can take 30-60 seconds on first startup
Issue 3: Permission Denied
# Check volume permissions
ls -la ~/mysql-lab/mysql-data/

# Fix permissions if needed
sudo chown -R 999:999 ~/mysql-lab/mysql-data/
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Stop all MySQL containers
docker stop mysql-restored mysql-admin-client mysql-restore-test 2>/dev/null

# Remove all containers
docker rm mysql-restored mysql-admin-client mysql-restore-test 2>/dev/null

# Remove the custom network
docker network rm mysql-network

# Optional: Remove MySQL images (if you want to free up space)
# docker rmi mysql:8.0

# Keep the data and backup files for future reference
echo "Lab files preserved in ~/mysql-lab/"
ls -la ~/mysql-lab/
Conclusion
Congratulations! You have successfully completed Lab 24: Docker and Databases - Running a MySQL Container.

What You Accomplished
In this lab, you have:

Mastered MySQL Container Deployment: Successfully pulled and ran MySQL Docker containers with proper configuration
Implemented Network Isolation: Created custom Docker networks to isolate database traffic and improve security
Database Schema Management: Initialized databases with custom schemas and sample data
Data Persistence: Implemented volume mounting to ensure data survives container restarts and removals
Backup and Recovery: Created comprehensive backup strategies and tested data restoration procedures
Client-Server Architecture: Set up MySQL client containers to interact with database servers
Monitoring and Maintenance: Created monitoring scripts to track database health and performance
Why This Matters
Running databases in Docker containers is a crucial skill in modern DevOps and cloud computing because:

Consistency: Ensures identical database environments across development, testing, and production
Scalability: Enables easy horizontal scaling of database instances
Isolation: Provides security and resource isolation between different applications
Portability: Allows databases to run consistently across different infrastructure platforms
Efficiency: Reduces resource overhead compared to traditional virtual machines
Next Steps
To further enhance your Docker database skills, consider:

Exploring Docker Compose for multi-container database applications
Learning about database clustering and replication in containers
Investigating container orchestration with Kubernetes
Studying database performance optimization in containerized environments
Exploring other database systems like PostgreSQL, MongoDB, or Redis in Docker
This foundation will serve you well as you progress toward Docker Certified Associate (DCA) certification and advance your career in containerized application development and deployment.
