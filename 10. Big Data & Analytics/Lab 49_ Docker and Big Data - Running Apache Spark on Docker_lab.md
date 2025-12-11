Lab 49: Docker and Big Data - Running Apache Spark on Docker
Lab Objectives
By the end of this lab, you will be able to:

Pull and run Apache Spark Docker images for distributed data processing
Set up a Spark cluster with master and worker nodes using Docker
Submit and execute Spark jobs on a containerized cluster
Configure persistent storage for Spark data and applications
Use Docker Compose to manage and orchestrate a complete Spark cluster
Understand the fundamentals of running big data workloads in containers
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, volumes)
Familiarity with command-line interface operations
Basic knowledge of distributed computing concepts
Understanding of YAML file structure for Docker Compose
No prior Apache Spark experience required - we'll cover the basics
Lab Environment Setup
Al Nafi Cloud Machine Access: This lab uses Al Nafi's pre-configured Linux-based cloud machines. Simply click Start Lab to access your environment. No need to build your own VM or install Docker - everything is ready to use.

Your cloud machine includes:

Docker Engine (latest version)
Docker Compose
Text editors (nano, vim)
All necessary networking configurations
Task 1: Pull and Run Apache Spark Docker Image
Subtask 1.1: Explore Available Spark Images
First, let's understand what Spark Docker images are available and pull the official one.

# Search for official Apache Spark images
docker search apache/spark

# Pull the official Apache Spark image
docker pull apache/spark:latest

# Verify the image was downloaded
docker images | grep spark
Subtask 1.2: Run a Basic Spark Container
Let's start with a simple Spark container to understand the basics.

# Run Spark in interactive mode to explore
docker run -it --name spark-test apache/spark:latest /bin/bash

# Inside the container, check Spark installation
ls -la /opt/spark/
echo $SPARK_HOME

# Exit the container
exit

# Remove the test container
docker rm spark-test
Subtask 1.3: Run Spark Shell
Now let's run the Spark shell to test basic functionality.

# Run Spark shell in a container
docker run -it --rm \
  --name spark-shell \
  -p 4040:4040 \
  apache/spark:latest \
  /opt/spark/bin/spark-shell

# Note: The Spark UI will be available at http://localhost:4040
Inside the Spark shell, try these basic commands:

// Create a simple RDD (Resilient Distributed Dataset)
val data = Array(1, 2, 3, 4, 5)
val distData = sc.parallelize(data)

// Perform a simple operation
val result = distData.map(x => x * 2).collect()
println(result.mkString(", "))

// Exit Spark shell
:quit
Task 2: Set Up Spark with Master and Worker Nodes
Subtask 2.1: Create a Docker Network
First, create a dedicated network for our Spark cluster.

# Create a custom bridge network for Spark cluster
docker network create spark-network

# Verify network creation
docker network ls | grep spark
Subtask 2.2: Start Spark Master Node
# Start Spark Master container
docker run -d \
  --name spark-master \
  --network spark-network \
  -p 8080:8080 \
  -p 7077:7077 \
  -p 4040:4040 \
  apache/spark:latest \
  /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master

# Check if master is running
docker ps | grep spark-master

# View master logs
docker logs spark-master
The Spark Master UI will be available at http://localhost:8080

Subtask 2.3: Start Spark Worker Nodes
# Start first worker node
docker run -d \
  --name spark-worker-1 \
  --network spark-network \
  -p 8081:8081 \
  apache/spark:latest \
  /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077

# Start second worker node
docker run -d \
  --name spark-worker-2 \
  --network spark-network \
  -p 8082:8081 \
  apache/spark:latest \
  /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077

# Verify all containers are running
docker ps | grep spark
Subtask 2.4: Verify Cluster Setup
# Check cluster status through master logs
docker logs spark-master | tail -10

# Check worker logs
docker logs spark-worker-1 | tail -5
docker logs spark-worker-2 | tail -5
Visit http://localhost:8080 in your browser to see the Spark Master UI with connected workers.

Task 3: Submit Spark Jobs to the Cluster
Subtask 3.1: Prepare Sample Data
Create a directory structure for our Spark applications and data.

# Create directories for Spark applications
mkdir -p ~/spark-lab/apps
mkdir -p ~/spark-lab/data

# Create sample data file
cat > ~/spark-lab/data/sample.txt << EOF
Hello World
Apache Spark
Docker Container
Big Data Processing
Distributed Computing
Cluster Management
Data Analytics
Machine Learning
EOF
Subtask 3.2: Create a Simple Spark Application
Create a Python Spark application for word counting.

# Create a Python Spark application
cat > ~/spark-lab/apps/wordcount.py << 'EOF'
from pyspark.sql import SparkSession
import sys

def main():
    # Create Spark session
    spark = SparkSession.builder \
        .appName("WordCount") \
        .getOrCreate()
    
    # Read input file
    input_file = sys.argv[1] if len(sys.argv) > 1 else "/data/sample.txt"
    
    # Read text file and count words
    text_file = spark.read.text(input_file)
    
    # Split lines into words and count
    from pyspark.sql.functions import split, explode, lower, col
    
    words = text_file.select(
        explode(split(lower(col("value")), " ")).alias("word")
    ).filter(col("word") != "")
    
    word_counts = words.groupBy("word").count().orderBy("count", ascending=False)
    
    # Show results
    print("Word Count Results:")
    word_counts.show()
    
    # Stop Spark session
    spark.stop()

if __name__ == "__main__":
    main()
EOF
Subtask 3.3: Submit Job to Cluster
# Copy data and application to master container
docker cp ~/spark-lab/data/sample.txt spark-master:/data/
docker cp ~/spark-lab/apps/wordcount.py spark-master:/apps/

# Submit the Spark job to the cluster
docker exec spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --executor-memory 1g \
  --total-executor-cores 2 \
  /apps/wordcount.py /data/sample.txt
Subtask 3.4: Monitor Job Execution
# Check application logs
docker logs spark-master | grep -A 10 -B 10 "WordCount"

# View running applications in Spark UI
echo "Visit http://localhost:8080 to see running applications"
echo "Visit http://localhost:4040 to see application details (when running)"
Task 4: Configure Persistent Storage for Spark Data
Subtask 4.1: Create Docker Volumes
# Create volumes for persistent storage
docker volume create spark-data
docker volume create spark-logs
docker volume create spark-apps

# List created volumes
docker volume ls | grep spark
Subtask 4.2: Restart Cluster with Persistent Storage
First, stop the existing cluster:

# Stop and remove existing containers
docker stop spark-master spark-worker-1 spark-worker-2
docker rm spark-master spark-worker-1 spark-worker-2
Start the cluster with persistent volumes:

# Start master with persistent storage
docker run -d \
  --name spark-master \
  --network spark-network \
  -p 8080:8080 \
  -p 7077:7077 \
  -p 4040:4040 \
  -v spark-data:/data \
  -v spark-logs:/opt/spark/logs \
  -v spark-apps:/apps \
  apache/spark:latest \
  /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master

# Start workers with persistent storage
docker run -d \
  --name spark-worker-1 \
  --network spark-network \
  -p 8081:8081 \
  -v spark-data:/data \
  -v spark-logs:/opt/spark/logs \
  apache/spark:latest \
  /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077

docker run -d \
  --name spark-worker-2 \
  --network spark-network \
  -p 8082:8081 \
  -v spark-data:/data \
  -v spark-logs:/opt/spark/logs \
  apache/spark:latest \
  /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
Subtask 4.3: Test Persistent Storage
# Copy data to persistent volume
docker cp ~/spark-lab/data/sample.txt spark-master:/data/
docker cp ~/spark-lab/apps/wordcount.py spark-master:/apps/

# Create a larger dataset for testing
cat > ~/spark-lab/data/large_dataset.txt << 'EOF'
Big Data Analytics with Apache Spark
Docker containers provide isolation
Distributed computing across multiple nodes
Machine learning algorithms process data
Real-time streaming data processing
Batch processing for historical data
Cluster resource management and scheduling
Fault tolerance and data recovery
Scalable data processing pipelines
Cloud-native big data solutions
EOF

# Copy large dataset
docker cp ~/spark-lab/data/large_dataset.txt spark-master:/data/

# Run job with larger dataset
docker exec spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --executor-memory 1g \
  --total-executor-cores 2 \
  /apps/wordcount.py /data/large_dataset.txt
Task 5: Use Docker Compose to Manage the Spark Cluster
Subtask 5.1: Create Docker Compose Configuration
First, stop the existing containers:

# Stop existing containers
docker stop spark-master spark-worker-1 spark-worker-2
docker rm spark-master spark-worker-1 spark-worker-2
Create a Docker Compose file:

# Create Docker Compose file
cat > ~/spark-lab/docker-compose.yml << 'EOF'
version: '3.8'

services:
  spark-master:
    image: apache/spark:latest
    container_name: spark-master
    hostname: spark-master
    ports:
      - "8080:8080"
      - "7077:7077"
      - "4040:4040"
    volumes:
      - spark-data:/data
      - spark-logs:/opt/spark/logs
      - spark-apps:/apps
    environment:
      - SPARK_MODE=master
      - SPARK_MASTER_HOST=spark-master
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
    networks:
      - spark-network

  spark-worker-1:
    image: apache/spark:latest
    container_name: spark-worker-1
    hostname: spark-worker-1
    ports:
      - "8081:8081"
    volumes:
      - spark-data:/data
      - spark-logs:/opt/spark/logs
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_MEMORY=1g
      - SPARK_WORKER_CORES=1
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    depends_on:
      - spark-master
    networks:
      - spark-network

  spark-worker-2:
    image: apache/spark:latest
    container_name: spark-worker-2
    hostname: spark-worker-2
    ports:
      - "8082:8081"
    volumes:
      - spark-data:/data
      - spark-logs:/opt/spark/logs
    environment:
      - SPARK_MODE=worker
      - SPARK_MASTER_URL=spark://spark-master:7077
      - SPARK_WORKER_MEMORY=1g
      - SPARK_WORKER_CORES=1
    command: /opt/spark/bin/spark-class org.apache.spark.deploy.worker.Worker spark://spark-master:7077
    depends_on:
      - spark-master
    networks:
      - spark-network

volumes:
  spark-data:
    external: true
  spark-logs:
    external: true
  spark-apps:
    external: true

networks:
  spark-network:
    external: true
EOF
Subtask 5.2: Launch Cluster with Docker Compose
# Navigate to the lab directory
cd ~/spark-lab

# Start the Spark cluster
docker-compose up -d

# Check cluster status
docker-compose ps

# View logs from all services
docker-compose logs --tail=10
Subtask 5.3: Scale Workers with Docker Compose
# Scale worker nodes to 3 instances
docker-compose up -d --scale spark-worker-1=2 --scale spark-worker-2=2

# Check running containers
docker-compose ps

# View cluster in Spark UI
echo "Visit http://localhost:8080 to see all workers"
Subtask 5.4: Submit Jobs Using Docker Compose
# Copy applications and data
docker cp ~/spark-lab/apps/wordcount.py spark-master:/apps/
docker cp ~/spark-lab/data/large_dataset.txt spark-master:/data/

# Submit job through Docker Compose
docker-compose exec spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --executor-memory 1g \
  --total-executor-cores 4 \
  /apps/wordcount.py /data/large_dataset.txt
Subtask 5.5: Create Advanced Spark Application
Create a more complex Spark application for data analysis:

# Create advanced analytics application
cat > ~/spark-lab/apps/data_analytics.py << 'EOF'
from pyspark.sql import SparkSession
from pyspark.sql.functions import *
from pyspark.sql.types import *
import sys

def main():
    # Create Spark session with more configuration
    spark = SparkSession.builder \
        .appName("DataAnalytics") \
        .config("spark.sql.adaptive.enabled", "true") \
        .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
        .getOrCreate()
    
    # Create sample sales data
    sales_data = [
        ("2023-01-01", "Electronics", "Laptop", 1200.00, 2),
        ("2023-01-02", "Electronics", "Phone", 800.00, 3),
        ("2023-01-03", "Clothing", "Shirt", 50.00, 5),
        ("2023-01-04", "Electronics", "Tablet", 400.00, 1),
        ("2023-01-05", "Clothing", "Pants", 80.00, 2),
        ("2023-01-06", "Electronics", "Laptop", 1200.00, 1),
        ("2023-01-07", "Books", "Novel", 15.00, 10),
        ("2023-01-08", "Books", "Textbook", 120.00, 3),
        ("2023-01-09", "Clothing", "Jacket", 150.00, 1),
        ("2023-01-10", "Electronics", "Phone", 800.00, 2)
    ]
    
    # Define schema
    schema = StructType([
        StructField("date", StringType(), True),
        StructField("category", StringType(), True),
        StructField("product", StringType(), True),
        StructField("price", DoubleType(), True),
        StructField("quantity", IntegerType(), True)
    ])
    
    # Create DataFrame
    df = spark.createDataFrame(sales_data, schema)
    
    # Convert date string to date type
    df = df.withColumn("date", to_date(col("date"), "yyyy-MM-dd"))
    df = df.withColumn("total_sales", col("price") * col("quantity"))
    
    print("=== Sales Data Analysis ===")
    df.show()
    
    # Analysis 1: Total sales by category
    print("\n=== Total Sales by Category ===")
    category_sales = df.groupBy("category") \
        .agg(sum("total_sales").alias("total_revenue"),
             sum("quantity").alias("total_quantity")) \
        .orderBy(desc("total_revenue"))
    category_sales.show()
    
    # Analysis 2: Top selling products
    print("\n=== Top Selling Products ===")
    product_sales = df.groupBy("product") \
        .agg(sum("total_sales").alias("total_revenue"),
             sum("quantity").alias("total_quantity")) \
        .orderBy(desc("total_revenue"))
    product_sales.show()
    
    # Analysis 3: Daily sales trend
    print("\n=== Daily Sales Trend ===")
    daily_sales = df.groupBy("date") \
        .agg(sum("total_sales").alias("daily_revenue")) \
        .orderBy("date")
    daily_sales.show()
    
    # Save results to persistent storage
    category_sales.coalesce(1).write.mode("overwrite").csv("/data/category_sales")
    product_sales.coalesce(1).write.mode("overwrite").csv("/data/product_sales")
    
    print("\n=== Analysis Complete - Results saved to /data/ ===")
    
    spark.stop()

if __name__ == "__main__":
    main()
EOF

# Copy and run the advanced application
docker cp ~/spark-lab/apps/data_analytics.py spark-master:/apps/

# Submit the advanced analytics job
docker-compose exec spark-master /opt/spark/bin/spark-submit \
  --master spark://spark-master:7077 \
  --deploy-mode client \
  --executor-memory 1g \
  --total-executor-cores 4 \
  /apps/data_analytics.py
Troubleshooting Common Issues
Issue 1: Container Connection Problems
# Check network connectivity
docker network inspect spark-network

# Verify containers are on the same network
docker inspect spark-master | grep NetworkMode
docker inspect spark-worker-1 | grep NetworkMode
Issue 2: Port Conflicts
# Check if ports are already in use
netstat -tulpn | grep :8080
netstat -tulpn | grep :7077

# Use different ports if needed
docker run -p 8090:8080 -p 7078:7077 ...
Issue 3: Memory Issues
# Check container resource usage
docker stats

# Adjust memory settings in Docker Compose
# Add to service configuration:
# mem_limit: 2g
# memswap_limit: 2g
Issue 4: Volume Mount Problems
# Check volume status
docker volume inspect spark-data

# Verify volume mounts
docker inspect spark-master | grep -A 10 Mounts
Cleanup
When you're finished with the lab:

# Stop and remove all containers
docker-compose down

# Remove volumes (optional - this will delete all data)
docker volume rm spark-data spark-logs spark-apps

# Remove network
docker network rm spark-network

# Remove images (optional)
docker rmi apache/spark:latest
Conclusion
Congratulations! You have successfully completed Lab 49: Docker and Big Data - Running Apache Spark on Docker.

What You Accomplished
In this lab, you have:

Mastered Spark Containerization: Successfully pulled and ran Apache Spark Docker images, understanding how big data frameworks can be containerized for easier deployment and management.

Built a Distributed Cluster: Set up a complete Spark cluster with master and worker nodes using Docker containers, demonstrating how to create scalable distributed computing environments.

Executed Real Data Processing: Submitted and ran actual Spark jobs including word counting and sales data analytics, gaining hands-on experience with distributed data processing.

Implemented Persistent Storage: Configured Docker volumes to ensure data persistence across container restarts, a critical requirement for production big data systems.

Orchestrated with Docker Compose: Used Docker Compose to manage complex multi-container Spark deployments, making cluster management more efficient and reproducible.

Why This Matters
This lab demonstrates several important concepts in modern data engineering:

Containerized Big Data: Running Spark in containers provides consistency across development, testing, and production environments
Scalable Architecture: The ability to easily scale worker nodes shows how containerization supports elastic computing resources
Infrastructure as Code: Using Docker Compose files makes your Spark cluster configuration version-controlled and reproducible
Cloud-Ready Deployments: These skills directly translate to running Spark on Kubernetes and cloud container services
Real-World Applications
The skills you've learned apply to:

Data Engineering Pipelines: Processing large datasets in production environments
Machine Learning Workflows: Running ML training jobs on distributed Spark clusters
Real-time Analytics: Setting up streaming data processing systems
Cloud Migration: Moving on-premises big data workloads to containerized cloud platforms
Next Steps
To continue building on this foundation:

Explore Spark Streaming for real-time data processing
Integrate with data sources like Kafka, Hadoop, or cloud storage
Learn about Spark on Kubernetes for production deployments
Study performance tuning and optimization techniques for Spark clusters
You now have practical experience with containerized big data processing that's directly applicable to modern data engineering roles and Docker Certified Associate certification objectives.
