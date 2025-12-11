Lab 32: Docker for Big Data - Running Apache Hadoop in Docker
Objectives
By the end of this lab, students will be able to:

Set up and configure Docker containers for Apache Hadoop components (HDFS and YARN)
Deploy a multi-container Hadoop environment using Docker Compose
Interact with the Hadoop Distributed File System (HDFS) through containerized interfaces
Submit and monitor MapReduce jobs to a YARN cluster running in Docker containers
Understand the fundamentals of big data processing using the Hadoop ecosystem in containerized environments
Troubleshoot common issues when running Hadoop in Docker containers
Prerequisites
Before starting this lab, students should have:

Basic understanding of Linux command line operations
Familiarity with Docker concepts (containers, images, volumes)
Basic knowledge of distributed systems concepts
Understanding of file systems and data processing workflows
Access to a terminal or command prompt
Technical Requirements:

Docker Engine (version 20.10 or later)
Docker Compose (version 2.0 or later)
At least 4GB of available RAM
10GB of free disk space
Lab Environment Setup
Good News! Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools already installed. Simply click Start Lab to access your ready-to-use environment. No need to build your own virtual machine or install Docker manually.

Your cloud machine includes:

Ubuntu 22.04 LTS
Docker Engine (latest stable version)
Docker Compose
Text editors (nano, vim)
Network utilities
Task 1: Set up Docker Containers for Hadoop (HDFS and YARN)
Subtask 1.1: Create Project Directory Structure
First, let's create a organized directory structure for our Hadoop Docker setup.

# Create main project directory
mkdir ~/hadoop-docker-lab
cd ~/hadoop-docker-lab

# Create subdirectories for configuration and data
mkdir -p config/hadoop
mkdir -p data/namenode
mkdir -p data/datanode
mkdir -p data/logs
Subtask 1.2: Create Hadoop Configuration Files
Create the core Hadoop configuration files that will be mounted into our containers.

Create core-site.xml:

cat > config/hadoop/core-site.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://namenode:9000</value>
    </property>
    <property>
        <name>hadoop.http.staticuser.user</name>
        <value>root</value>
    </property>
</configuration>
EOF
Create hdfs-site.xml:

cat > config/hadoop/hdfs-site.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/hadoop/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/hadoop/dfs/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.permissions.enabled</name>
        <value>false</value>
    </property>
    <property>
        <name>dfs.namenode.http-address</name>
        <value>namenode:9870</value>
    </property>
</configuration>
EOF
Create mapred-site.xml:

cat > config/hadoop/mapred-site.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
EOF
Create yarn-site.xml:

cat > config/hadoop/yarn-site.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.resourcemanager.hostname</name>
        <value>resourcemanager</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
EOF
Subtask 1.3: Verify Configuration Files
Let's verify that all configuration files were created correctly:

# List all configuration files
ls -la config/hadoop/

# Check the content of one configuration file
cat config/hadoop/core-site.xml
Task 2: Start a Multi-Container Hadoop Environment with Docker Compose
Subtask 2.1: Create Docker Compose File
Create a comprehensive Docker Compose file that defines all Hadoop services:

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  namenode:
    image: apache/hadoop:3.3.6
    container_name: hadoop-namenode
    hostname: namenode
    ports:
      - "9870:9870"  # NameNode Web UI
      - "9000:9000"  # HDFS
    volumes:
      - ./config/hadoop:/opt/hadoop/etc/hadoop
      - ./data/namenode:/hadoop/dfs/name
      - ./data/logs:/opt/hadoop/logs
    environment:
      - CLUSTER_NAME=hadoop-cluster
    command: >
      bash -c "
        hdfs namenode -format -force &&
        hdfs namenode
      "
    networks:
      - hadoop-network

  datanode:
    image: apache/hadoop:3.3.6
    container_name: hadoop-datanode
    hostname: datanode
    ports:
      - "9864:9864"  # DataNode Web UI
    volumes:
      - ./config/hadoop:/opt/hadoop/etc/hadoop
      - ./data/datanode:/hadoop/dfs/data
      - ./data/logs:/opt/hadoop/logs
    environment:
      - CLUSTER_NAME=hadoop-cluster
    command: hdfs datanode
    depends_on:
      - namenode
    networks:
      - hadoop-network

  resourcemanager:
    image: apache/hadoop:3.3.6
    container_name: hadoop-resourcemanager
    hostname: resourcemanager
    ports:
      - "8088:8088"  # ResourceManager Web UI
    volumes:
      - ./config/hadoop:/opt/hadoop/etc/hadoop
      - ./data/logs:/opt/hadoop/logs
    environment:
      - CLUSTER_NAME=hadoop-cluster
    command: yarn resourcemanager
    depends_on:
      - namenode
    networks:
      - hadoop-network

  nodemanager:
    image: apache/hadoop:3.3.6
    container_name: hadoop-nodemanager
    hostname: nodemanager
    ports:
      - "8042:8042"  # NodeManager Web UI
    volumes:
      - ./config/hadoop:/opt/hadoop/etc/hadoop
      - ./data/logs:/opt/hadoop/logs
    environment:
      - CLUSTER_NAME=hadoop-cluster
    command: yarn nodemanager
    depends_on:
      - resourcemanager
    networks:
      - hadoop-network

  historyserver:
    image: apache/hadoop:3.3.6
    container_name: hadoop-historyserver
    hostname: historyserver
    ports:
      - "19888:19888"  # History Server Web UI
    volumes:
      - ./config/hadoop:/opt/hadoop/etc/hadoop
      - ./data/logs:/opt/hadoop/logs
    environment:
      - CLUSTER_NAME=hadoop-cluster
    command: mapred historyserver
    depends_on:
      - namenode
    networks:
      - hadoop-network

networks:
  hadoop-network:
    driver: bridge

volumes:
  namenode-data:
  datanode-data:
  logs-data:
EOF
Subtask 2.2: Start the Hadoop Cluster
Now let's start our multi-container Hadoop environment:

# Start all services in detached mode
docker-compose up -d

# Check the status of all containers
docker-compose ps

# View logs from all services
docker-compose logs
Subtask 2.3: Verify Cluster Status
Wait for all services to start (this may take 2-3 minutes), then verify the cluster is running:

# Check individual container status
docker ps

# Check namenode logs specifically
docker-compose logs namenode

# Check if HDFS is accessible
docker exec hadoop-namenode hdfs dfsadmin -report
Task 3: Interact with the Hadoop Distributed File System (HDFS)
Subtask 3.1: Access HDFS Command Line Interface
Let's interact with HDFS using the command line tools:

# Access the namenode container
docker exec -it hadoop-namenode bash

# Inside the container, check HDFS status
hdfs dfsadmin -report

# List root directory contents
hdfs dfs -ls /

# Create a directory in HDFS
hdfs dfs -mkdir /user
hdfs dfs -mkdir /user/input
hdfs dfs -mkdir /user/output
Subtask 3.2: Upload and Manage Files in HDFS
Create sample data and upload it to HDFS:

# Still inside the namenode container, create a sample text file
echo "Hello Hadoop World" > /tmp/sample.txt
echo "This is a test file for HDFS" >> /tmp/sample.txt
echo "Docker makes Hadoop deployment easy" >> /tmp/sample.txt

# Upload the file to HDFS
hdfs dfs -put /tmp/sample.txt /user/input/

# List files in HDFS
hdfs dfs -ls /user/input/

# View file content from HDFS
hdfs dfs -cat /user/input/sample.txt

# Check file details
hdfs dfs -stat /user/input/sample.txt
Subtask 3.3: Explore HDFS Web Interface
Exit the container and access the HDFS web interface:

# Exit the container
exit

# The NameNode Web UI should be accessible at:
echo "Access NameNode Web UI at: http://localhost:9870"
echo "Access DataNode Web UI at: http://localhost:9864"
Note: Open your web browser and navigate to http://localhost:9870 to explore the HDFS web interface. You can browse the file system, check cluster health, and view detailed information about your Hadoop cluster.

Task 4: Submit Jobs to the YARN Cluster
Subtask 4.1: Prepare Sample Data for MapReduce
Let's create more substantial sample data for our MapReduce job:

# Access the namenode container again
docker exec -it hadoop-namenode bash

# Create a larger sample file with word count data
cat > /tmp/wordcount-input.txt << 'EOF'
Apache Hadoop is a collection of open-source software utilities
that facilitates using a network of many computers to solve problems
involving massive amounts of data and computation. It provides a
software framework for distributed storage and processing of big data
using the MapReduce programming model. Hadoop was originally designed
for computer clusters built from commodity hardware. Docker containers
make it easy to deploy and manage Hadoop clusters in various environments.
The Hadoop ecosystem includes HDFS for storage and YARN for resource management.
MapReduce jobs can process large datasets efficiently across multiple nodes.
EOF

# Upload the file to HDFS
hdfs dfs -put /tmp/wordcount-input.txt /user/input/

# Verify the upload
hdfs dfs -ls /user/input/
Subtask 4.2: Run a MapReduce Word Count Job
Execute a classic MapReduce word count job:

# Still inside the namenode container
# Run the word count example
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar wordcount /user/input /user/output/wordcount

# Check if the job completed successfully
hdfs dfs -ls /user/output/wordcount/

# View the results
hdfs dfs -cat /user/output/wordcount/part-r-00000
Subtask 4.3: Monitor Jobs through YARN Web Interface
# Exit the container to access web interfaces
exit

# Access YARN ResourceManager Web UI
echo "Access YARN ResourceManager Web UI at: http://localhost:8088"
echo "Access NodeManager Web UI at: http://localhost:8042"
echo "Access History Server Web UI at: http://localhost:19888"
Open your web browser and navigate to http://localhost:8088 to explore the YARN ResourceManager interface where you can:

View running and completed applications
Monitor cluster resources
Check job history and logs
Subtask 4.4: Run Additional MapReduce Examples
Let's run a few more examples to understand different types of jobs:

# Access the namenode container
docker exec -it hadoop-namenode bash

# Run a Pi estimation job (Monte Carlo method)
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar pi 2 100

# Create numeric data for terasort example
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar teragen 1000 /user/input/teragen-input

# Run terasort on the generated data
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar terasort /user/input/teragen-input /user/output/terasort-output

# Verify the sorted output
hdfs dfs -ls /user/output/terasort-output/
Task 5: Explore Data Processing in the Hadoop Ecosystem
Subtask 5.1: Understand HDFS Block Storage
Let's explore how HDFS stores data in blocks:

# Inside the namenode container
# Check HDFS configuration
hdfs getconf -confKey dfs.blocksize

# Create a larger file to see block distribution
dd if=/dev/zero of=/tmp/largefile.txt bs=1M count=150

# Upload to HDFS
hdfs dfs -put /tmp/largefile.txt /user/input/

# Check file blocks information
hdfs fsck /user/input/largefile.txt -files -blocks -locations
Subtask 5.2: Monitor Cluster Resources
# Check cluster resource usage
yarn node -list

# View cluster metrics
yarn cluster -list-node-labels

# Check HDFS space usage
hdfs dfs -df -h

# View detailed cluster information
hdfs dfsadmin -printTopology
Subtask 5.3: Create Custom Data Processing Workflow
Let's create a simple data processing pipeline:

# Create sample log data
cat > /tmp/access.log << 'EOF'
192.168.1.1 - - [10/Oct/2023:13:55:36 +0000] "GET /index.html HTTP/1.1" 200 2326
192.168.1.2 - - [10/Oct/2023:13:55:37 +0000] "GET /about.html HTTP/1.1" 200 1024
192.168.1.1 - - [10/Oct/2023:13:55:38 +0000] "POST /login HTTP/1.1" 302 0
192.168.1.3 - - [10/Oct/2023:13:55:39 +0000] "GET /products.html HTTP/1.1" 200 4096
192.168.1.2 - - [10/Oct/2023:13:55:40 +0000] "GET /contact.html HTTP/1.1" 404 512
EOF

# Upload log data to HDFS
hdfs dfs -put /tmp/access.log /user/input/

# Use grep to filter successful requests (status 200)
hadoop jar $HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-*.jar grep /user/input/access.log /user/output/successful-requests '200'

# View the results
hdfs dfs -cat /user/output/successful-requests/part-r-00000
Subtask 5.4: Cleanup and Resource Management
# Clean up output directories for reuse
hdfs dfs -rm -r /user/output/wordcount
hdfs dfs -rm -r /user/output/terasort-output
hdfs dfs -rm -r /user/output/successful-requests

# Check remaining space
hdfs dfs -df -h

# Exit the container
exit
Troubleshooting Common Issues
Issue 1: Containers Not Starting
If containers fail to start, check the following:

# Check container logs
docker-compose logs namenode
docker-compose logs datanode

# Restart specific services
docker-compose restart namenode

# Rebuild and restart all services
docker-compose down
docker-compose up -d
Issue 2: HDFS Safe Mode
If HDFS is in safe mode:

# Check safe mode status
docker exec hadoop-namenode hdfs dfsadmin -safemode get

# Force leave safe mode (use with caution)
docker exec hadoop-namenode hdfs dfsadmin -safemode leave
Issue 3: Web UI Not Accessible
If web interfaces are not accessible:

# Check if ports are properly mapped
docker port hadoop-namenode
docker port hadoop-resourcemanager

# Verify containers are running
docker ps
Issue 4: Permission Issues
If you encounter permission issues:

# Check HDFS permissions
docker exec hadoop-namenode hdfs dfs -ls -la /

# Fix permissions if needed
docker exec hadoop-namenode hdfs dfs -chmod -R 755 /user
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Stop all containers
docker-compose down

# Remove all containers and networks
docker-compose down --volumes --remove-orphans

# Optional: Remove downloaded images (if you want to free up space)
docker rmi apache/hadoop:3.3.6

# Clean up local data (optional)
rm -rf ~/hadoop-docker-lab
Conclusion
Congratulations! You have successfully completed Lab 32: Docker for Big Data - Running Apache Hadoop in Docker.

What You Accomplished:

Container Orchestration: You set up a complete multi-container Hadoop cluster using Docker Compose, demonstrating how containerization simplifies big data infrastructure deployment.

HDFS Mastery: You learned to interact with the Hadoop Distributed File System, understanding how data is stored, replicated, and managed across distributed nodes.

YARN Resource Management: You successfully submitted and monitored MapReduce jobs through YARN, gaining hands-on experience with distributed computing resource management.

Big Data Processing: You executed various MapReduce jobs including word count, Pi estimation, and data sorting, understanding different patterns of distributed data processing.

Ecosystem Integration: You explored the integration between HDFS, YARN, and MapReduce, understanding how these components work together in the Hadoop ecosystem.

Why This Matters:

Industry Relevance: Hadoop remains a cornerstone technology for big data processing in enterprise environments
Containerization Benefits: Running Hadoop in Docker containers provides consistency, portability, and easier deployment across different environments
Scalability Understanding: You've gained practical experience with distributed systems concepts that apply to modern big data architectures
DevOps Skills: This lab bridges the gap between big data technologies and modern containerization practices
Next Steps:

Explore more advanced Hadoop ecosystem tools like Apache Hive, Apache Spark, or Apache HBase
Learn about Kubernetes orchestration for Hadoop clusters
Investigate cloud-native big data solutions that build upon these foundational concepts
Practice with larger datasets and more complex MapReduce algorithms
This hands-on experience with Hadoop in Docker containers provides you with valuable skills for both the Docker Certified Associate (DCA) certification and real-world big data engineering scenarios.
