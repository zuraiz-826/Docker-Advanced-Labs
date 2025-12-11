Lab 85: Docker and Big Data - Running Apache Flink with Docker
Objectives
By the end of this lab, you will be able to:

Deploy Apache Flink containers using Docker Compose for distributed stream processing
Configure and manage Flink job manager and task manager components
Submit and execute stream processing jobs on a Flink cluster
Scale Flink clusters dynamically using Docker container orchestration
Monitor Flink cluster performance and analyze key metrics through the web dashboard
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, Docker Compose)
Familiarity with command-line interface operations
Basic knowledge of distributed computing concepts
Understanding of stream processing fundamentals
No prior Apache Flink experience required - this lab will guide you through the basics
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker and Docker Compose already installed. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine
Docker Compose v2.x
4 CPU cores and 8GB RAM
Network access for downloading Flink images
Pre-configured firewall rules for Flink web interfaces
Task 1: Deploy Apache Flink Containers Using Docker Compose
Subtask 1.1: Create Project Directory Structure
First, let's create a dedicated directory for our Flink cluster setup:

mkdir ~/flink-docker-lab
cd ~/flink-docker-lab
Create subdirectories for organizing our lab files:

mkdir -p jobs logs data config
Subtask 1.2: Create Docker Compose Configuration
Create a comprehensive Docker Compose file that defines our Flink cluster architecture:

nano docker-compose.yml
Add the following configuration:

version: '3.8'

services:
  jobmanager:
    image: flink:1.18-scala_2.12
    container_name: flink-jobmanager
    ports:
      - "8081:8081"
    command: jobmanager
    environment:
      - FLINK_PROPERTIES=jobmanager.rpc.address:jobmanager
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
    volumes:
      - ./jobs:/opt/flink/jobs
      - ./logs:/opt/flink/log
      - ./data:/opt/flink/data
    networks:
      - flink-network
    restart: unless-stopped

  taskmanager1:
    image: flink:1.18-scala_2.12
    container_name: flink-taskmanager1
    depends_on:
      - jobmanager
    command: taskmanager
    environment:
      - FLINK_PROPERTIES=jobmanager.rpc.address:jobmanager
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
      - TASK_MANAGER_NUMBER_OF_TASK_SLOTS=2
    volumes:
      - ./jobs:/opt/flink/jobs
      - ./logs:/opt/flink/log
      - ./data:/opt/flink/data
    networks:
      - flink-network
    restart: unless-stopped

  taskmanager2:
    image: flink:1.18-scala_2.12
    container_name: flink-taskmanager2
    depends_on:
      - jobmanager
    command: taskmanager
    environment:
      - FLINK_PROPERTIES=jobmanager.rpc.address:jobmanager
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
      - TASK_MANAGER_NUMBER_OF_TASK_SLOTS=2
    volumes:
      - ./jobs:/opt/flink/jobs
      - ./logs:/opt/flink/log
      - ./data:/opt/flink/data
    networks:
      - flink-network
    restart: unless-stopped

networks:
  flink-network:
    driver: bridge
Save and exit the file (Ctrl+X, then Y, then Enter).

Subtask 1.3: Deploy the Flink Cluster
Start the Flink cluster using Docker Compose:

docker-compose up -d
Verify that all containers are running:

docker-compose ps
You should see output similar to:

NAME                  COMMAND             SERVICE         STATUS          PORTS
flink-jobmanager      "/docker-entrypoint…"   jobmanager      running         6123/tcp, 0.0.0.0:8081->8081/tcp
flink-taskmanager1    "/docker-entrypoint…"   taskmanager1    running         6121-6125/tcp
flink-taskmanager2    "/docker-entrypoint…"   taskmanager2    running         6121-6125/tcp
Check the logs to ensure proper startup:

docker-compose logs jobmanager
Task 2: Set Up Flink Job Manager and Task Manager
Subtask 2.1: Verify Job Manager Configuration
Access the Flink Web Dashboard by opening your browser and navigating to:

http://localhost:8081
Note: If you're using a cloud machine, replace localhost with your machine's public IP address.

In the dashboard, you should see:

Job Manager: 1 instance running
Task Managers: 2 instances connected
Available Task Slots: 4 total (2 per task manager)
Subtask 2.2: Inspect Cluster Configuration
Check the detailed configuration of your job manager:

docker exec flink-jobmanager cat /opt/flink/conf/flink-conf.yaml
Examine task manager configuration:

docker exec flink-taskmanager1 cat /opt/flink/conf/flink-conf.yaml
Subtask 2.3: Verify Network Connectivity
Test communication between job manager and task managers:

docker exec flink-jobmanager ping -c 3 flink-taskmanager1
docker exec flink-jobmanager ping -c 3 flink-taskmanager2
Check the cluster status using Flink CLI:

docker exec flink-jobmanager flink list
Task 3: Submit a Stream Processing Job to the Flink Cluster
Subtask 3.1: Create a Sample Data Generator
First, let's create a simple data generator that will simulate streaming data:

nano data/sample-data.txt
Add sample data:

user1,login,2024-01-15T10:00:00
user2,purchase,2024-01-15T10:01:00
user3,logout,2024-01-15T10:02:00
user1,purchase,2024-01-15T10:03:00
user4,login,2024-01-15T10:04:00
user2,logout,2024-01-15T10:05:00
user5,login,2024-01-15T10:06:00
user3,purchase,2024-01-15T10:07:00
user1,logout,2024-01-15T10:08:00
user4,purchase,2024-01-15T10:09:00
Subtask 3.2: Submit a Built-in Example Job
Flink comes with several example jobs. Let's submit the WordCount example:

docker exec flink-jobmanager flink run /opt/flink/examples/streaming/WordCount.jar
Check the job status in the web dashboard or via CLI:

docker exec flink-jobmanager flink list
Subtask 3.3: Submit a Socket-based Streaming Job
Start a socket text stream example that reads from a socket:

docker exec -d flink-jobmanager flink run /opt/flink/examples/streaming/SocketWindowWordCount.jar --hostname localhost --port 9999
In another terminal, create a socket server to send data:

docker exec -it flink-jobmanager nc -l 9999
Type some text and press Enter. You can type phrases like:

hello world
apache flink streaming
real time processing
big data analytics
Subtask 3.4: Monitor Job Execution
View the running jobs:

docker exec flink-jobmanager flink list -r
Check job details in the web dashboard:

Navigate to Jobs tab
Click on your running job
Explore the Overview, Timeline, and Subtasks sections
Task 4: Scale the Flink Cluster and Adjust Resources
Subtask 4.1: Add Additional Task Managers
Create an updated Docker Compose configuration to add more task managers:

nano docker-compose-scaled.yml
Add the following content:

version: '3.8'

services:
  jobmanager:
    image: flink:1.18-scala_2.12
    container_name: flink-jobmanager
    ports:
      - "8081:8081"
    command: jobmanager
    environment:
      - FLINK_PROPERTIES=jobmanager.rpc.address:jobmanager
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
    volumes:
      - ./jobs:/opt/flink/jobs
      - ./logs:/opt/flink/log
      - ./data:/opt/flink/data
    networks:
      - flink-network
    restart: unless-stopped

  taskmanager1:
    image: flink:1.18-scala_2.12
    container_name: flink-taskmanager1
    depends_on:
      - jobmanager
    command: taskmanager
    environment:
      - FLINK_PROPERTIES=jobmanager.rpc.address:jobmanager
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
      - TASK_MANAGER_NUMBER_OF_TASK_SLOTS=4
    volumes:
      - ./jobs:/opt/flink/jobs
      - ./logs:/opt/flink/log
      - ./data:/opt/flink/data
    networks:
      - flink-network
    restart: unless-stopped

  taskmanager2:
    image: flink:1.18-scala_2.12
    container_name: flink-taskmanager2
    depends_on:
      - jobmanager
    command: taskmanager
    environment:
      - FLINK_PROPERTIES=jobmanager.rpc.address:jobmanager
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
      - TASK_MANAGER_NUMBER_OF_TASK_SLOTS=4
    volumes:
      - ./jobs:/opt/flink/jobs
      - ./logs:/opt/flink/log
      - ./data:/opt/flink/data
    networks:
      - flink-network
    restart: unless-stopped

  taskmanager3:
    image: flink:1.18-scala_2.12
    container_name: flink-taskmanager3
    depends_on:
      - jobmanager
    command: taskmanager
    environment:
      - FLINK_PROPERTIES=jobmanager.rpc.address:jobmanager
      - JOB_MANAGER_RPC_ADDRESS=jobmanager
      - TASK_MANAGER_NUMBER_OF_TASK_SLOTS=4
    volumes:
      - ./jobs:/opt/flink/jobs
      - ./logs:/opt/flink/log
      - ./data:/opt/flink/data
    networks:
      - flink-network
    restart: unless-stopped

networks:
  flink-network:
    driver: bridge
Subtask 4.2: Scale Up the Cluster
Stop the current cluster and start the scaled version:

docker-compose down
docker-compose -f docker-compose-scaled.yml up -d
Verify the scaled cluster:

docker-compose -f docker-compose-scaled.yml ps
Check the web dashboard to confirm you now have:

3 Task Managers
12 Available Task Slots (4 per task manager)
Subtask 4.3: Dynamic Scaling with Docker Compose
You can also scale services dynamically:

docker-compose -f docker-compose-scaled.yml up -d --scale taskmanager1=2
Note: This creates additional instances of the same service, which may require additional configuration for proper load distribution.

Subtask 4.4: Adjust Resource Allocation
Create a custom configuration file for resource tuning:

nano config/flink-conf-custom.yaml
Add resource configurations:

# Job Manager Memory Configuration
jobmanager.memory.process.size: 1024m
jobmanager.memory.flink.size: 768m

# Task Manager Memory Configuration
taskmanager.memory.process.size: 2048m
taskmanager.memory.flink.size: 1536m
taskmanager.memory.managed.fraction: 0.4

# Network Configuration
taskmanager.network.memory.fraction: 0.1
taskmanager.network.memory.min: 64mb
taskmanager.network.memory.max: 1gb

# Parallelism Configuration
parallelism.default: 4
Task 5: Monitor the Flink Cluster's Performance and Metrics
Subtask 5.1: Access Built-in Monitoring Dashboard
Navigate to the Flink Web Dashboard and explore different monitoring sections:

Overview: Cluster summary and resource utilization
Jobs: Running and completed job details
Task Managers: Individual task manager metrics
Job Manager: Job manager configuration and logs
Subtask 5.2: Monitor Resource Utilization
Check CPU and memory usage of containers:

docker stats flink-jobmanager flink-taskmanager1 flink-taskmanager2 flink-taskmanager3
Monitor container logs for performance insights:

docker-compose -f docker-compose-scaled.yml logs --tail=50 jobmanager
docker-compose -f docker-compose-scaled.yml logs --tail=50 taskmanager1
Subtask 5.3: Create a Performance Monitoring Script
Create a script to collect cluster metrics:

nano monitor-cluster.sh
Add the following content:

#!/bin/bash

echo "=== Flink Cluster Monitoring ==="
echo "Timestamp: $(date)"
echo ""

echo "=== Container Status ==="
docker-compose -f docker-compose-scaled.yml ps
echo ""

echo "=== Resource Usage ==="
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
echo ""

echo "=== Running Jobs ==="
docker exec flink-jobmanager flink list -r
echo ""

echo "=== Task Manager Count ==="
echo "Active Task Managers: $(docker ps --filter "name=flink-taskmanager" --format "{{.Names}}" | wc -l)"
echo ""

echo "=== Disk Usage ==="
du -sh logs/ data/ jobs/
echo ""
Make the script executable and run it:

chmod +x monitor-cluster.sh
./monitor-cluster.sh
Subtask 5.4: Set Up Log Analysis
Create a log analysis script to identify performance patterns:

nano analyze-logs.sh
Add the following content:

#!/bin/bash

echo "=== Flink Log Analysis ==="
echo ""

echo "=== Job Manager Errors ==="
docker logs flink-jobmanager 2>&1 | grep -i error | tail -10
echo ""

echo "=== Task Manager Performance ==="
for tm in flink-taskmanager1 flink-taskmanager2 flink-taskmanager3; do
    echo "--- $tm ---"
    docker logs $tm 2>&1 | grep -i "memory\|cpu\|performance" | tail -5
    echo ""
done

echo "=== Network Issues ==="
docker logs flink-jobmanager 2>&1 | grep -i "connection\|network\|timeout" | tail -5
echo ""
Make it executable and run:

chmod +x analyze-logs.sh
./analyze-logs.sh
Subtask 5.5: Performance Testing
Submit a more intensive job to test cluster performance:

docker exec flink-jobmanager flink run -p 8 /opt/flink/examples/streaming/WordCount.jar
Monitor the job execution:

watch -n 2 'docker exec flink-jobmanager flink list -r'
Troubleshooting Common Issues
Issue 1: Containers Not Starting
If containers fail to start, check:

docker-compose logs
docker system df
docker system prune -f
Issue 2: Task Managers Not Connecting
Verify network connectivity:

docker network ls
docker network inspect flink-docker-lab_flink-network
Issue 3: Out of Memory Errors
Increase container memory limits in docker-compose.yml:

services:
  jobmanager:
    # ... other configuration
    deploy:
      resources:
        limits:
          memory: 2G
        reservations:
          memory: 1G
Issue 4: Port Conflicts
If port 8081 is already in use:

netstat -tulpn | grep 8081
# Change the port mapping in docker-compose.yml
ports:
  - "8082:8081"  # Use port 8082 instead
Lab Cleanup
When you're finished with the lab, clean up resources:

# Stop all containers
docker-compose -f docker-compose-scaled.yml down

# Remove volumes (optional)
docker-compose -f docker-compose-scaled.yml down -v

# Clean up unused Docker resources
docker system prune -f

# Remove lab directory (optional)
cd ~
rm -rf flink-docker-lab
Conclusion
Congratulations! You have successfully completed Lab 85: Docker and Big Data - Running Apache Flink with Docker.

What You Accomplished
In this lab, you:

Deployed a Multi-Container Flink Cluster: You used Docker Compose to orchestrate a complete Apache Flink cluster with one job manager and multiple task managers, demonstrating how containerization simplifies big data infrastructure deployment.

Configured Distributed Stream Processing: You set up job manager and task manager components with proper networking and resource allocation, learning how Flink's architecture enables scalable real-time data processing.

Executed Stream Processing Jobs: You submitted and monitored various streaming jobs, including built-in examples and socket-based streaming applications, gaining hands-on experience with Flink's job execution model.

Implemented Dynamic Scaling: You learned how to scale Flink clusters horizontally by adding task managers and adjusting resource allocation, demonstrating the flexibility of containerized big data solutions.

Established Comprehensive Monitoring: You implemented monitoring solutions using Flink's web dashboard, Docker stats, and custom scripts to track cluster performance and identify potential issues.

Why This Matters
This lab demonstrates critical skills for modern big data engineering:

Container Orchestration: Docker Compose skills are essential for managing complex distributed systems in development and production environments.

Stream Processing Expertise: Apache Flink is a leading technology for real-time data processing, used by companies like Netflix, Uber, and LinkedIn for mission-critical applications.

Scalability Understanding: Learning to scale distributed systems dynamically is crucial for handling varying workloads in production environments.

Monitoring and Operations: Effective monitoring is essential for maintaining reliable big data systems and ensuring optimal performance.

Infrastructure as Code: Using configuration files to define and manage infrastructure promotes consistency, version control, and reproducibility.

Next Steps
To further develop your skills:

Explore Flink's advanced features like event time processing, watermarks, and complex event processing
Integrate Flink with other big data tools like Apache Kafka, Elasticsearch, or Apache Cassandra
Learn about Flink's state management and fault tolerance mechanisms
Practice deploying Flink clusters on Kubernetes for production-scale environments
Study Flink SQL for declarative stream processing applications
This foundation in containerized stream processing will serve you well in modern data engineering roles and big data system architecture.
