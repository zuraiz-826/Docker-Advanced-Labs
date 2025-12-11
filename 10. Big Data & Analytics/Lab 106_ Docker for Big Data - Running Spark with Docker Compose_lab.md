Lab 106: Docker for Big Data - Running Spark with Docker Compose
Objectives
By the end of this lab, you will be able to:

Understand how to containerize Apache Spark for big data processing
Create and configure Docker Compose files for multi-container Spark clusters
Deploy a distributed Spark cluster with master and worker nodes
Execute Spark jobs on containerized clusters
Monitor Spark job execution through Docker container logs
Scale Spark clusters dynamically by adding worker containers
Apply Docker best practices for big data workloads
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, volumes)
Familiarity with command-line interface operations
Basic knowledge of distributed computing concepts
Understanding of YAML file format
No prior Apache Spark experience required (we'll cover the basics)
Lab Environment Setup
Good News: Al Nafi provides you with ready-to-use Linux-based cloud machines. Simply click Start Lab and you'll have access to a fully configured environment with Docker and Docker Compose pre-installed. No need to build your own virtual machine or install software.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker Engine
Docker Compose v2.x
4 CPU cores and 8GB RAM
Pre-configured network settings
Internet access for downloading Docker images
Task 1: Understanding the Spark Architecture and Setting Up the Environment
Subtask 1.1: Verify Docker Installation
First, let's confirm that Docker and Docker Compose are properly installed and running.

# Check Docker version
docker --version

# Check Docker Compose version
docker compose version

# Verify Docker daemon is running
docker info
Subtask 1.2: Create Project Directory Structure
Create a organized directory structure for our Spark cluster project.

# Create main project directory
mkdir spark-docker-lab
cd spark-docker-lab

# Create subdirectories for different components
mkdir -p data/input data/output scripts logs

# Create sample data directory
mkdir -p sample-data
Subtask 1.3: Prepare Sample Data
Create sample text files that we'll use for our Spark word count job.

# Create sample text file 1
cat > sample-data/sample1.txt << 'EOF'
Apache Spark is a unified analytics engine for large-scale data processing.
Spark provides high-level APIs in Java, Scala, Python and R.
Spark runs on Hadoop, Apache Mesos, Kubernetes, standalone, or in the cloud.
It can access diverse data sources including HDFS, Alluxio, Apache Cassandra, Apache HBase, Apache Hive, and hundreds of other data sources.
EOF

# Create sample text file 2
cat > sample-data/sample2.txt << 'EOF'
Docker is a platform designed to help developers build, share, and run modern applications.
Docker containers are lightweight, portable, and provide consistent environments.
Docker Compose is a tool for defining and running multi-container Docker applications.
Container orchestration helps manage complex distributed applications.
EOF

# Create sample text file 3
cat > sample-data/sample3.txt << 'EOF'
Big data processing requires distributed computing frameworks.
Apache Spark excels at in-memory data processing and analytics.
Docker containers provide isolation and scalability for big data workloads.
Combining Spark with Docker enables flexible and scalable data processing pipelines.
EOF

# Verify files were created
ls -la sample-data/
Task 2: Create Docker Compose Configuration for Spark Cluster
Subtask 2.1: Create the Main Docker Compose File
Create a comprehensive Docker Compose file that defines our Spark cluster architecture.

# Create the docker-compose.yml file
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  spark-master:
    image: bitnami/spark:3.4
    container_name: spark-master
    hostname: spark-master
    environment:
      - SPARK_MODE=master
      - SPARK_RPC_AUTHENTICATION_ENABLED=no
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
      - SPARK_MASTER_HOST=spark-master
      - SPARK_MASTER_PORT_NUMBER=7077
      - SPARK_MASTER_WEBUI_PORT=8080
    ports:
      - "8080:8080"  # Spark Master Web UI
      - "7077:7077"  # Spark Master Port
    volumes:
      - ./sample-data:/opt/bitnami/spark/data
      - ./scripts:/opt/bitnami/spark/scripts
      - ./logs:/opt/bitnami/spark/logs
    networks:
      - spark-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 30s
      timeout: 10s
      retries: 3

  spark-worker-1:
    image: bitnami/spark:3.4
    container_name: spark-worker-1
    hostname: spark-worker-1
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_MEMORY=2g
      - SPARK_WORKER_CORES=2
      - SPARK_RPC_AUTHENTICATION_ENABLED=no
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
    volumes:
      - ./sample-data:/opt/bitnami/spark/data
      - ./scripts:/opt/bitnami/spark/scripts
      - ./logs:/opt/bitnami/spark/logs
    networks:
      - spark-network
    depends_on:
      - spark-master
    healthcheck:
      test: ["CMD", "pgrep", "-f", "org.apache.spark.deploy.worker.Worker"]
      interval: 30s
      timeout: 10s
      retries: 3

  spark-worker-2:
    image: bitnami/spark:3.4
    container_name: spark-worker-2
    hostname: spark-worker-2
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_MEMORY=2g
      - SPARK_WORKER_CORES=2
      - SPARK_RPC_AUTHENTICATION_ENABLED=no
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
    volumes:
      - ./sample-data:/opt/bitnami/spark/data
      - ./scripts:/opt/bitnami/spark/scripts
      - ./logs:/opt/bitnami/spark/logs
    networks:
      - spark-network
    depends_on:
      - spark-master
    healthcheck:
      test: ["CMD", "pgrep", "-f", "org.apache.spark.deploy.worker.Worker"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  spark-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

volumes:
  spark-logs:
    driver: local
EOF
Subtask 2.2: Understand the Configuration
Let's break down the key components of our Docker Compose configuration:

Master Node Configuration:

Image: Uses bitnami/spark:3.4 which is a production-ready Spark image
Ports: Exposes port 8080 for Web UI and 7077 for Spark communication
Environment: Configures Spark in master mode with security disabled for simplicity
Volumes: Mounts local directories for data, scripts, and logs
Worker Node Configuration:

Memory: Each worker allocated 2GB RAM
Cores: Each worker gets 2 CPU cores
Master URL: Points to the master node for cluster coordination
Dependencies: Workers wait for master to be ready
Subtask 2.3: Validate the Configuration
Check that our Docker Compose file is syntactically correct.

# Validate the docker-compose.yml syntax
docker compose config

# Check if the configuration is valid and view the resolved configuration
docker compose config --services
Task 3: Launch the Spark Cluster
Subtask 3.1: Start the Spark Cluster
Launch all containers defined in our Docker Compose file.

# Pull the required Docker images (this may take a few minutes)
docker compose pull

# Start the Spark cluster in detached mode
docker compose up -d

# Verify all containers are running
docker compose ps
You should see output similar to:

NAME            IMAGE               COMMAND                  SERVICE         CREATED         STATUS                   PORTS
spark-master    bitnami/spark:3.4   "/opt/bitnami/script…"   spark-master    2 minutes ago   Up 2 minutes (healthy)   0.0.0.0:7077->7077/tcp, 0.0.0.0:8080->8080/tcp
spark-worker-1  bitnami/spark:3.4   "/opt/bitnami/script…"   spark-worker-1  2 minutes ago   Up 2 minutes (healthy)   8080/tcp
spark-worker-2  bitnami/spark:3.4   "/opt/bitnami/script…"   spark-worker-2  2 minutes ago   Up 2 minutes (healthy)   8080/tcp
Subtask 3.2: Verify Cluster Health
Check the health and connectivity of our Spark cluster.

# Check container logs to ensure proper startup
docker compose logs spark-master --tail=20

# Check worker logs
docker compose logs spark-worker-1 --tail=10
docker compose logs spark-worker-2 --tail=10

# Test network connectivity between containers
docker exec spark-master ping -c 3 spark-worker-1
docker exec spark-master ping -c 3 spark-worker-2
Subtask 3.3: Access Spark Web UI
The Spark Master Web UI provides valuable insights into cluster status and job execution.

# Get the IP address of your lab machine
hostname -I

# The Spark Web UI should be accessible at:
# http://YOUR_LAB_IP:8080
echo "Access Spark Web UI at: http://$(hostname -I | awk '{print $1}'):8080"
Note: If you're using Al Nafi's cloud environment, the Web UI will be accessible through the provided lab interface or by using the public IP address shown in your lab dashboard.

Task 4: Create and Run Spark Jobs
Subtask 4.1: Create a Word Count Spark Job
Create a Python script that performs word counting using PySpark.

# Create a Python script for word count
cat > scripts/word_count.py << 'EOF'
from pyspark.sql import SparkSession
from pyspark.sql.functions import explode, split, lower, regexp_replace, col
import sys
import time

def main():
    # Create Spark session
    spark = SparkSession.builder \
        .appName("DockerSparkWordCount") \
        .config("spark.sql.adaptive.enabled", "true") \
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
        .getOrCreate()
    
    # Set log level to reduce verbose output
    spark.sparkContext.setLogLevel("WARN")
    
    print("=" * 50)
    print("Starting Spark Word Count Job")
    print("=" * 50)
    
    try:
        # Read text files from the data directory
        input_path = "/opt/bitnami/spark/data/*.txt"
        print(f"Reading files from: {input_path}")
        
        # Read all text files
        df = spark.read.text(input_path)
        print(f"Number of lines read: {df.count()}")
        
        # Split lines into words and clean them
        words_df = df.select(
            explode(split(col("value"), "\\s+")).alias("word")
        ).select(
            regexp_replace(lower(col("word")), "[^a-zA-Z0-9]", "").alias("clean_word")
        ).filter(
            col("clean_word") != ""
        )
        
        # Count word occurrences
        word_counts = words_df.groupBy("clean_word").count().orderBy(col("count").desc())
        
        print("\nTop 20 most frequent words:")
        print("-" * 30)
        word_counts.show(20, truncate=False)
        
        # Save results (optional)
        output_path = "/opt/bitnami/spark/data/output/word_count_results"
        print(f"\nSaving results to: {output_path}")
        word_counts.coalesce(1).write.mode("overwrite").option("header", "true").csv(output_path)
        
        # Show some statistics
        total_words = words_df.count()
        unique_words = word_counts.count()
        
        print(f"\nJob Statistics:")
        print(f"Total words processed: {total_words}")
        print(f"Unique words found: {unique_words}")
        print(f"Most common word: {word_counts.first()['clean_word']} (appears {word_counts.first()['count']} times)")
        
    except Exception as e:
        print(f"Error during job execution: {str(e)}")
        return 1
    
    finally:
        # Stop Spark session
        spark.stop()
        print("\nSpark Word Count Job Completed Successfully!")
        print("=" * 50)
    
    return 0

if __name__ == "__main__":
    exit_code = main()
    sys.exit(exit_code)
EOF
Subtask 4.2: Create a Data Analysis Spark Job
Create another Spark job that performs more complex data analysis.

# Create a data analysis script
cat > scripts/data_analysis.py << 'EOF'
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
import sys

def main():
    # Create Spark session with optimized configuration
    spark = SparkSession.builder \
        .appName("DockerSparkDataAnalysis") \
        .config("spark.sql.adaptive.enabled", "true") \
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
        .config("spark.serializer", "org.apache.spark.serializer.KryoSerializer") \
        .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    print("=" * 60)
    print("Starting Spark Data Analysis Job")
    print("=" * 60)
    
    try:
        # Read text files
        input_path = "/opt/bitnami/spark/data/*.txt"
        df = spark.read.text(input_path)
        
        # Add filename to track source
        df_with_source = df.withColumn("filename", input_file_name())
        
        # Extract filename only (remove path)
        df_with_source = df_with_source.withColumn(
            "source_file", 
            regexp_extract(col("filename"), r"([^/]+)\.txt$", 1)
        )
        
        # Analyze text characteristics
        analysis_df = df_with_source.select(
            col("source_file"),
            col("value").alias("line"),
            length(col("value")).alias("line_length"),
            size(split(col("value"), "\\s+")).alias("word_count_per_line")
        )
        
        print("File Analysis Results:")
        print("-" * 40)
        
        # Statistics by file
        file_stats = analysis_df.groupBy("source_file").agg(
            count("line").alias("total_lines"),
            sum("word_count_per_line").alias("total_words"),
            avg("line_length").alias("avg_line_length"),
            max("line_length").alias("max_line_length"),
            avg("word_count_per_line").alias("avg_words_per_line")
        ).orderBy("source_file")
        
        file_stats.show(truncate=False)
        
        # Overall statistics
        overall_stats = analysis_df.agg(
            count("line").alias("total_lines"),
            sum("word_count_per_line").alias("total_words"),
            avg("line_length").alias("avg_line_length"),
            countDistinct("source_file").alias("total_files")
        ).collect()[0]
        
        print("\nOverall Dataset Statistics:")
        print("-" * 40)
        print(f"Total files processed: {overall_stats['total_files']}")
        print(f"Total lines: {overall_stats['total_lines']}")
        print(f"Total words: {overall_stats['total_words']}")
        print(f"Average line length: {overall_stats['avg_line_length']:.2f} characters")
        
        # Find lines with specific keywords
        keywords = ["spark", "docker", "data", "processing"]
        print(f"\nKeyword Analysis:")
        print("-" * 40)
        
        for keyword in keywords:
            keyword_count = analysis_df.filter(
                lower(col("line")).contains(keyword.lower())
            ).count()
            print(f"Lines containing '{keyword}': {keyword_count}")
        
        # Save detailed analysis
        output_path = "/opt/bitnami/spark/data/output/data_analysis_results"
        print(f"\nSaving detailed analysis to: {output_path}")
        file_stats.coalesce(1).write.mode("overwrite").option("header", "true").csv(output_path)
        
    except Exception as e:
        print(f"Error during analysis: {str(e)}")
        return 1
    
    finally:
        spark.stop()
        print("\nData Analysis Job Completed Successfully!")
        print("=" * 60)
    
    return 0

if __name__ == "__main__":
    exit_code = main()
    sys.exit(exit_code)
EOF
Subtask 4.3: Execute the Spark Jobs
Now let's run our Spark jobs on the distributed cluster.

# Run the word count job
echo "Executing Word Count Job..."
docker exec spark-master spark-submit \
    --master spark://spark-master:7077 \
    --deploy-mode client \
    --driver-memory 1g \
    --executor-memory 1g \
    --executor-cores 1 \
    --total-executor-cores 4 \
    /opt/bitnami/spark/scripts/word_count.py

# Wait a moment between jobs
sleep 5

# Run the data analysis job
echo "Executing Data Analysis Job..."
docker exec spark-master spark-submit \
    --master spark://spark-master:7077 \
    --deploy-mode client \
    --driver-memory 1g \
    --executor-memory 1g \
    --executor-cores 1 \
    --total-executor-cores 4 \
    /opt/bitnami/spark/scripts/data_analysis.py
Subtask 4.4: Verify Job Results
Check the output and results of our Spark jobs.

# Check if output directories were created
ls -la sample-data/output/

# View word count results
echo "Word Count Results:"
echo "==================="
if [ -d "sample-data/output/word_count_results" ]; then
    find sample-data/output/word_count_results -name "*.csv" -exec cat {} \; | head -20
fi

# View data analysis results
echo -e "\nData Analysis Results:"
echo "======================"
if [ -d "sample-data/output/data_analysis_results" ]; then
    find sample-data/output/data_analysis_results -name "*.csv" -exec cat {} \;
fi
Task 5: Monitor Spark Job Progress and Performance
Subtask 5.1: Monitor Through Container Logs
Learn how to monitor Spark job execution through Docker container logs.

# Create a monitoring script
cat > scripts/monitor_cluster.sh << 'EOF'
#!/bin/bash

echo "Spark Cluster Monitoring Dashboard"
echo "=================================="
echo

# Function to show container status
show_container_status() {
    echo "Container Status:"
    echo "-----------------"
    docker compose ps
    echo
}

# Function to show resource usage
show_resource_usage() {
    echo "Resource Usage:"
    echo "---------------"
    docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
    echo
}

# Function to show recent logs
show_recent_logs() {
    echo "Recent Master Logs:"
    echo "-------------------"
    docker compose logs spark-master --tail=10
    echo
    
    echo "Recent Worker Logs:"
    echo "-------------------"
    docker compose logs spark-worker-1 --tail=5
    docker compose logs spark-worker-2 --tail=5
    echo
}

# Function to check cluster health
check_cluster_health() {
    echo "Cluster Health Check:"
    echo "--------------------"
    
    # Check if master is responding
    if docker exec spark-master curl -s http://localhost:8080 > /dev/null; then
        echo "✓ Master Web UI is accessible"
    else
        echo "✗ Master Web UI is not accessible"
    fi
    
    # Check worker connectivity
    if docker exec spark-master nc -z spark-worker-1 8080; then
        echo "✓ Worker 1 is reachable"
    else
        echo "✗ Worker 1 is not reachable"
    fi
    
    if docker exec spark-master nc -z spark-worker-2 8080; then
        echo "✓ Worker 2 is reachable"
    else
        echo "✗ Worker 2 is not reachable"
    fi
    echo
}

# Main monitoring loop
case "${1:-status}" in
    "status")
        show_container_status
        ;;
    "resources")
        show_resource_usage
        ;;
    "logs")
        show_recent_logs
        ;;
    "health")
        check_cluster_health
        ;;
    "all")
        show_container_status
        show_resource_usage
        check_cluster_health
        show_recent_logs
        ;;
    *)
        echo "Usage: $0 {status|resources|logs|health|all}"
        echo "  status    - Show container status"
        echo "  resources - Show resource usage"
        echo "  logs      - Show recent logs"
        echo "  health    - Check cluster health"
        echo "  all       - Show all monitoring information"
        ;;
esac
EOF

# Make the script executable
chmod +x scripts/monitor_cluster.sh

# Run the monitoring script
./scripts/monitor_cluster.sh all
Subtask 5.2: Real-time Log Monitoring
Set up real-time monitoring of Spark job execution.

# Monitor logs in real-time (run this in a separate terminal if needed)
echo "Starting real-time log monitoring..."
echo "Press Ctrl+C to stop monitoring"

# Follow logs from all containers
docker compose logs -f --tail=20
Subtask 5.3: Performance Metrics Collection
Create a script to collect and display performance metrics.

# Create performance monitoring script
cat > scripts/performance_monitor.py << 'EOF'
#!/usr/bin/env python3

import subprocess
import json
import time
import sys
from datetime import datetime

def run_command(command):
    """Execute shell command and return output"""
    try:
        result = subprocess.run(command, shell=True, capture_output=True, text=True)
        return result.stdout.strip()
    except Exception as e:
        return f"Error: {str(e)}"

def get_container_stats():
    """Get Docker container statistics"""
    cmd = "docker stats --no-stream --format 'table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}'"
    return run_command(cmd)

def get_spark_cluster_info():
    """Get Spark cluster information from master"""
    try:
        # Try to get cluster info from Spark master
        cmd = "docker exec spark-master curl -s http://localhost:8080/json/"
        result = run_command(cmd)
        if result and not result.startswith("Error"):
            data = json.loads(result)
            return {
                'status': data.get('status', 'Unknown'),
                'workers': len(data.get('workers', [])),
                'cores': sum(w.get('cores', 0) for w in data.get('workers', [])),
                'memory': sum(w.get('memory', 0) for w in data.get('workers', [])),
                'active_apps': len(data.get('activeapps', []))
            }
    except:
        pass
    
    return {
        'status': 'Unknown',
        'workers': 'N/A',
        'cores': 'N/A',
        'memory': 'N/A',
        'active_apps': 'N/A'
    }

def main():
    print("Spark Cluster Performance Monitor")
    print("=" * 50)
    print("Press Ctrl+C to stop monitoring\n")
    
    try:
        while True:
            timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
            print(f"\n[{timestamp}] Cluster Status:")
            print("-" * 40)
            
            # Get container statistics
            stats = get_container_stats()
            print("Container Resource Usage:")
            print(stats)
            
            # Get Spark cluster info
            cluster_info = get_spark_cluster_info()
            print(f"\nSpark Cluster Information:")
            print(f"Status: {cluster_info['status']}")
            print(f"Active Workers: {cluster_info['workers']}")
            print(f"Total Cores: {cluster_info['cores']}")
            print(f"Total Memory: {cluster_info['memory']} MB")
            print(f"Active Applications: {cluster_info['active_apps']}")
            
            print("\n" + "=" * 50)
            time.sleep(10)  # Update every 10 seconds
            
    except KeyboardInterrupt:
        print("\nMonitoring stopped.")
        sys.exit(0)

if __name__ == "__main__":
    main()
EOF

# Make the script executable
chmod +x scripts/performance_monitor.py

# Run performance monitoring (for a short duration)
echo "Running performance monitor for 30 seconds..."
timeout 30 python3 scripts/performance_monitor.py || echo "Performance monitoring completed."
Task 6: Scale the Spark Cluster
Subtask 6.1: Add Additional Worker Containers
Learn how to dynamically scale the Spark cluster by adding more workers.

# Create an extended docker-compose file for scaling
cat > docker-compose.scale.yml << 'EOF'
version: '3.8'

services:
  spark-worker-3:
    image: bitnami/spark:3.4
    container_name: spark-worker-3
    hostname: spark-worker-3
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_MEMORY=2g
      - SPARK_WORKER_CORES=2
      - SPARK_RPC_AUTHENTICATION_ENABLED=no
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
    volumes:
      - ./sample-data:/opt/bitnami/spark/data
      - ./scripts:/opt/bitnami/spark/scripts
      - ./logs:/opt/bitnami/spark/logs
    networks:
      - spark-network
    healthcheck:
      test: ["CMD", "pgrep", "-f", "org.apache.spark.deploy.worker.Worker"]
      interval: 30s
      timeout: 10s
      retries: 3

  spark-worker-4:
    image: bitnami/spark:3.4
    container_name: spark-worker-4
    hostname: spark-worker-4
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_MEMORY=2g
      - SPARK_WORKER_CORES=2
      - SPARK_RPC_AUTHENTICATION_ENABLED=no
      - SPARK_RPC_ENCRYPTION_ENABLED=no
      - SPARK_LOCAL_STORAGE_ENCRYPTION_ENABLED=no
      - SPARK_SSL_ENABLED=no
    volumes:
      - ./sample-data:/opt/bitnami/spark/data
      - ./scripts:/opt/bitnami/spark/scripts
      - ./logs:/opt/bitnami/spark/logs
    networks:
      - spark-network
    healthcheck:
      test: ["CMD", "pgrep", "-f", "org.apache.spark.deploy.worker.Worker"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  spark-network:
    external: true
    name: spark-docker-lab_spark-network
EOF
Subtask 6.2: Scale Up the Cluster
Add the new worker containers to the existing cluster.

# Start additional workers using the scale compose file
docker compose -f docker-compose.scale.yml up -d

# Verify all containers are running
docker compose ps
docker compose -f docker-compose.scale.yml ps

# Check the cluster now has 4 workers
echo "Checking cluster status after scaling..."
sleep 10
./scripts/monitor_cluster.sh status
Subtask 6.3: Test Scaled Cluster Performance
Run a more intensive Spark job to test the scaled cluster.

# Create a performance test script
cat > scripts/performance_test.py << 'EOF'
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
import time
import sys

def main():
    spark = SparkSession.builder \
        .appName("DockerSparkPerformanceTest") \
        .config("spark.sql.adaptive.enabled", "true") \
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
        .getOrCreate()
    
    spark.sparkContext.setLogLevel("WARN")
    
    print("=" * 60)
    print("Spark Cluster Performance Test")
    print("=" * 60)
    
    start_time = time.time()
    
    try:
        # Create a larger dataset for testing
        print("Generating test dataset...")
        
        # Create a DataFrame with more data
        data_size = 1000000  # 1 million records
        df = spark.range(data_size).toDF("id")
        
        # Add computed columns
        df = df.withColumn("squared", col("id") * col("id")) \
               .withColumn("cubed", col("id") * col("id") * col("id")) \
               .withColumn("mod_100", col("id") % 100) \
               .withColumn("is_even", col("id") % 2 == 0)
        
        print(f"Dataset created with {df.count()} records")
        
        # Perform various operations
        print("Performing aggregations...")
        
        # Group by operations
        agg_results = df.groupBy("mod_100").agg(
            count("id").alias("count"),
            avg("squared").alias("avg_squared"),
            max("cubed").alias("max_cubed"),
            sum("id").alias("sum_id")
        ).orderBy("mod_100")
        
        print(f"Aggregation completed. Results count: {agg_results.count()}")
        
        # Show sample results
        print("\nSample aggregation results:")
        agg_results.show(10)
        
        # Perform joins
        print("Performing self-join operation...")
        df_
