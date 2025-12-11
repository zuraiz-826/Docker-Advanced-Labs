Lab 102: Docker for Big Data - Running Apache Kafka with Docker Compose
Lab Objectives
By the end of this lab, you will be able to:

Deploy Apache Kafka using Docker Compose for distributed messaging
Configure Zookeeper and Kafka services in a containerized environment
Create and manage Kafka producers and consumers
Monitor Kafka broker performance using container metrics
Scale Kafka brokers horizontally using Docker Compose
Understand the fundamentals of distributed messaging systems
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and Docker Compose
Familiarity with command-line interface (CLI)
Basic knowledge of distributed systems concepts
Understanding of messaging patterns (producer-consumer model)
Note: Al Nafi provides ready-to-use Linux-based cloud machines with Docker and Docker Compose pre-installed. Simply click "Start Lab" to begin - no need to build your own VM or install additional software.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with:

Docker Engine (latest stable version)
Docker Compose (latest stable version)
Text editors (nano, vim)
Network utilities for testing
Task 1: Set up Docker Compose File for Apache Kafka and Zookeeper Services
Subtask 1.1: Create Project Directory Structure
First, let's create a dedicated directory for our Kafka project:

mkdir kafka-docker-lab
cd kafka-docker-lab
mkdir config logs data
Subtask 1.2: Create Docker Compose Configuration
Create the main Docker Compose file that will define our Kafka ecosystem:

nano docker-compose.yml
Add the following configuration to the file:

version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - ./data/zookeeper:/var/lib/zookeeper/data
      - ./logs/zookeeper:/var/lib/zookeeper/log
    networks:
      - kafka-network

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka
    container_name: kafka
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'true'
    volumes:
      - ./data/kafka:/var/lib/kafka/data
      - ./logs/kafka:/var/log/kafka
    networks:
      - kafka-network

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
    networks:
      - kafka-network

networks:
  kafka-network:
    driver: bridge
Save and exit the file (Ctrl+X, then Y, then Enter in nano).

Subtask 1.3: Understand the Configuration
Let's break down the key components:

Zookeeper: Manages Kafka cluster metadata and coordination
Kafka: The main message broker service
Kafka-UI: Web interface for monitoring and managing Kafka
Networks: Isolated network for service communication
Volumes: Persistent storage for data and logs
Task 2: Start Kafka and Zookeeper Containers Using Docker Compose
Subtask 2.1: Validate Docker Compose File
Before starting services, validate the configuration:

docker-compose config
This command should display the parsed configuration without errors.

Subtask 2.2: Start the Services
Launch all services in detached mode:

docker-compose up -d
Subtask 2.3: Verify Service Status
Check that all containers are running:

docker-compose ps
You should see output similar to:

    Name                  Command               State                       Ports
------------------------------------------------------------------------------------------------
kafka         /etc/confluent/docker/run        Up      0.0.0.0:9092->9092/tcp, 0.0.0.0:9101->9101/tcp
kafka-ui      java -jar kafka-ui-api.jar       Up      0.0.0.0:8080->8080/tcp
zookeeper     /etc/confluent/docker/run        Up      0.0.0.0:2181->2181/tcp
Subtask 2.4: Check Service Logs
Monitor the startup process by checking logs:

# Check Zookeeper logs
docker-compose logs zookeeper

# Check Kafka logs
docker-compose logs kafka

# Follow logs in real-time
docker-compose logs -f kafka
Wait until you see messages indicating Kafka is ready to accept connections.

Task 3: Create a Producer and Consumer that Sends and Receives Messages
Subtask 3.1: Access Kafka Container
Open a new terminal session and access the Kafka container:

docker exec -it kafka bash
Subtask 3.2: Create a Kafka Topic
Inside the Kafka container, create a topic for testing:

kafka-topics --create \
  --topic test-messages \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 1
Verify the topic was created:

kafka-topics --list --bootstrap-server localhost:9092
Subtask 3.3: Create a Simple Producer Script
Exit the container and create a producer script on the host:

exit
nano producer.py
Add the following Python script:

#!/usr/bin/env python3

import json
import time
from kafka import KafkaProducer
from datetime import datetime
import random

def create_producer():
    """Create and configure Kafka producer"""
    producer = KafkaProducer(
        bootstrap_servers=['localhost:9092'],
        value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        key_serializer=lambda k: k.encode('utf-8') if k else None
    )
    return producer

def generate_sample_data():
    """Generate sample data for testing"""
    sample_data = {
        'timestamp': datetime.now().isoformat(),
        'user_id': random.randint(1000, 9999),
        'action': random.choice(['login', 'logout', 'purchase', 'view']),
        'value': random.uniform(10.0, 1000.0)
    }
    return sample_data

def main():
    producer = create_producer()
    topic = 'test-messages'
    
    print(f"Starting producer for topic: {topic}")
    print("Press Ctrl+C to stop")
    
    try:
        message_count = 0
        while True:
            # Generate and send message
            data = generate_sample_data()
            key = f"user_{data['user_id']}"
            
            future = producer.send(topic, key=key, value=data)
            result = future.get(timeout=10)
            
            message_count += 1
            print(f"Message {message_count} sent: {data}")
            
            time.sleep(2)  # Send message every 2 seconds
            
    except KeyboardInterrupt:
        print(f"\nStopping producer. Total messages sent: {message_count}")
    finally:
        producer.close()

if __name__ == "__main__":
    main()
Subtask 3.4: Create a Consumer Script
Create a consumer script:

nano consumer.py
Add the following Python script:

#!/usr/bin/env python3

import json
from kafka import KafkaConsumer
from datetime import datetime

def create_consumer():
    """Create and configure Kafka consumer"""
    consumer = KafkaConsumer(
        'test-messages',
        bootstrap_servers=['localhost:9092'],
        value_deserializer=lambda m: json.loads(m.decode('utf-8')),
        key_deserializer=lambda k: k.decode('utf-8') if k else None,
        group_id='test-consumer-group',
        auto_offset_reset='earliest',
        enable_auto_commit=True
    )
    return consumer

def main():
    consumer = create_consumer()
    
    print("Starting consumer...")
    print("Waiting for messages (Press Ctrl+C to stop)")
    
    try:
        message_count = 0
        for message in consumer:
            message_count += 1
            
            print(f"\n--- Message {message_count} ---")
            print(f"Topic: {message.topic}")
            print(f"Partition: {message.partition}")
            print(f"Offset: {message.offset}")
            print(f"Key: {message.key}")
            print(f"Value: {message.value}")
            print(f"Timestamp: {datetime.fromtimestamp(message.timestamp/1000)}")
            
    except KeyboardInterrupt:
        print(f"\nStopping consumer. Total messages processed: {message_count}")
    finally:
        consumer.close()

if __name__ == "__main__":
    main()
Subtask 3.5: Install Python Kafka Client
Install the required Python package:

pip3 install kafka-python
Subtask 3.6: Test Producer and Consumer
Open two terminal windows:

Terminal 1 - Start Consumer:

cd kafka-docker-lab
python3 consumer.py
Terminal 2 - Start Producer:

cd kafka-docker-lab
python3 producer.py
You should see messages being produced and consumed in real-time.

Subtask 3.7: Alternative Command-Line Testing
You can also test using Kafka's built-in tools:

Producer (in Kafka container):

docker exec -it kafka bash
kafka-console-producer --topic test-messages --bootstrap-server localhost:9092
Consumer (in another terminal):

docker exec -it kafka bash
kafka-console-consumer --topic test-messages --bootstrap-server localhost:9092 --from-beginning
Task 4: Monitor Kafka Broker Performance Using Metrics Exposed by the Container
Subtask 4.1: Access Kafka UI Dashboard
Open your web browser and navigate to:

http://localhost:8080
This will open the Kafka UI dashboard where you can monitor:

Topics and their configurations
Consumer groups and their lag
Broker information
Message throughput
Subtask 4.2: Monitor JMX Metrics
Create a script to collect JMX metrics:

nano monitor_kafka.py
Add the following monitoring script:

#!/usr/bin/env python3

import subprocess
import json
import time
from datetime import datetime

def get_container_stats():
    """Get Docker container statistics"""
    try:
        result = subprocess.run(
            ['docker', 'stats', 'kafka', '--no-stream', '--format', 'table {{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}\t{{.BlockIO}}'],
            capture_output=True,
            text=True
        )
        return result.stdout.strip()
    except Exception as e:
        return f"Error getting stats: {e}"

def get_kafka_topics():
    """Get list of Kafka topics"""
    try:
        result = subprocess.run(
            ['docker', 'exec', 'kafka', 'kafka-topics', '--list', '--bootstrap-server', 'localhost:9092'],
            capture_output=True,
            text=True
        )
        return result.stdout.strip().split('\n')
    except Exception as e:
        return [f"Error getting topics: {e}"]

def get_topic_details(topic):
    """Get details for a specific topic"""
    try:
        result = subprocess.run(
            ['docker', 'exec', 'kafka', 'kafka-topics', '--describe', '--topic', topic, '--bootstrap-server', 'localhost:9092'],
            capture_output=True,
            text=True
        )
        return result.stdout.strip()
    except Exception as e:
        return f"Error getting topic details: {e}"

def main():
    print("Kafka Monitoring Dashboard")
    print("=" * 50)
    
    while True:
        try:
            print(f"\n[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}]")
            
            # Container statistics
            print("\n--- Container Statistics ---")
            stats = get_container_stats()
            print(stats)
            
            # Topic information
            print("\n--- Topics ---")
            topics = get_kafka_topics()
            for topic in topics:
                if topic and not topic.startswith('Error'):
                    print(f"Topic: {topic}")
            
            # Detailed info for test topic
            if 'test-messages' in topics:
                print("\n--- Test Topic Details ---")
                details = get_topic_details('test-messages')
                print(details)
            
            print("\n" + "="*50)
            time.sleep(10)  # Update every 10 seconds
            
        except KeyboardInterrupt:
            print("\nMonitoring stopped.")
            break
        except Exception as e:
            print(f"Error in monitoring: {e}")
            time.sleep(5)

if __name__ == "__main__":
    main()
Run the monitoring script:

python3 monitor_kafka.py
Subtask 4.3: Check Kafka Logs for Performance Metrics
Monitor Kafka logs for performance information:

# View recent logs
docker-compose logs --tail=50 kafka

# Monitor logs in real-time
docker-compose logs -f kafka | grep -E "(throughput|latency|performance)"
Subtask 4.4: Use Docker Stats for Resource Monitoring
Monitor resource usage of all containers:

# Real-time monitoring
docker stats

# One-time snapshot
docker stats --no-stream
Task 5: Scale Kafka Brokers Using Docker Compose and Test the Configuration
Subtask 5.1: Create Multi-Broker Configuration
Create a new Docker Compose file for scaling:

nano docker-compose-scaled.yml
Add the following configuration:

version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - ./data/zookeeper:/var/lib/zookeeper/data
      - ./logs/zookeeper:/var/lib/zookeeper/log
    networks:
      - kafka-network

  kafka1:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka1
    container_name: kafka1
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9101:9101"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka1:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 2
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 2
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: localhost
    volumes:
      - ./data/kafka1:/var/lib/kafka/data
    networks:
      - kafka-network

  kafka2:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka2
    container_name: kafka2
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
      - "9102:9102"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka2:29093,PLAINTEXT_HOST://localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 2
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 2
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9102
      KAFKA_JMX_HOSTNAME: localhost
    volumes:
      - ./data/kafka2:/var/lib/kafka/data
    networks:
      - kafka-network

  kafka3:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka3
    container_name: kafka3
    depends_on:
      - zookeeper
    ports:
      - "9094:9094"
      - "9103:9103"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka3:29094,PLAINTEXT_HOST://localhost:9094
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 2
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 2
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_JMX_PORT: 9103
      KAFKA_JMX_HOSTNAME: localhost
    volumes:
      - ./data/kafka3:/var/lib/kafka/data
    networks:
      - kafka-network

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui-scaled
    depends_on:
      - kafka1
      - kafka2
      - kafka3
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local-cluster
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka1:29092,kafka2:29093,kafka3:29094
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181
    networks:
      - kafka-network

networks:
  kafka-network:
    driver: bridge
Subtask 5.2: Stop Current Setup and Start Scaled Version
Stop the current single-broker setup:

docker-compose down
Start the scaled setup:

docker-compose -f docker-compose-scaled.yml up -d
Subtask 5.3: Verify Multi-Broker Cluster
Check that all brokers are running:

docker-compose -f docker-compose-scaled.yml ps
Subtask 5.4: Create Replicated Topic
Create a topic with replication across brokers:

docker exec -it kafka1 kafka-topics --create \
  --topic replicated-messages \
  --bootstrap-server localhost:9092 \
  --partitions 6 \
  --replication-factor 3
Verify the topic configuration:

docker exec -it kafka1 kafka-topics --describe \
  --topic replicated-messages \
  --bootstrap-server localhost:9092
Subtask 5.5: Test Multi-Broker Producer
Create a multi-broker producer script:

nano producer_scaled.py
Add the following script:

#!/usr/bin/env python3

import json
import time
from kafka import KafkaProducer
from datetime import datetime
import random

def create_producer():
    """Create producer with multiple bootstrap servers"""
    producer = KafkaProducer(
        bootstrap_servers=['localhost:9092', 'localhost:9093', 'localhost:9094'],
        value_serializer=lambda v: json.dumps(v).encode('utf-8'),
        key_serializer=lambda k: k.encode('utf-8') if k else None,
        acks='all',  # Wait for all replicas to acknowledge
        retries=3
    )
    return producer

def generate_sample_data():
    """Generate sample data for testing"""
    sample_data = {
        'timestamp': datetime.now().isoformat(),
        'user_id': random.randint(1000, 9999),
        'action': random.choice(['login', 'logout', 'purchase', 'view', 'search']),
        'value': random.uniform(10.0, 1000.0),
        'region': random.choice(['us-east', 'us-west', 'eu-central', 'asia-pacific'])
    }
    return sample_data

def main():
    producer = create_producer()
    topic = 'replicated-messages'
    
    print(f"Starting scaled producer for topic: {topic}")
    print("Sending to multiple brokers with replication")
    print("Press Ctrl+C to stop")
    
    try:
        message_count = 0
        while True:
            data = generate_sample_data()
            key = f"user_{data['user_id']}"
            
            future = producer.send(topic, key=key, value=data)
            result = future.get(timeout=10)
            
            message_count += 1
            print(f"Message {message_count} sent to partition {result.partition}: {data}")
            
            time.sleep(1)  # Send message every second
            
    except KeyboardInterrupt:
        print(f"\nStopping producer. Total messages sent: {message_count}")
    finally:
        producer.close()

if __name__ == "__main__":
    main()
Subtask 5.6: Test Fault Tolerance
Start the scaled producer:

python3 producer_scaled.py
In another terminal, simulate broker failure:

# Stop one broker
docker stop kafka2

# Check cluster status
docker exec -it kafka1 kafka-topics --describe \
  --topic replicated-messages \
  --bootstrap-server localhost:9092
The producer should continue working even with one broker down.

Restart the stopped broker:

docker start kafka2
Subtask 5.7: Monitor Cluster Performance
Create a cluster monitoring script:

nano cluster_monitor.py
Add the following script:

#!/usr/bin/env python3

import subprocess
import time
from datetime import datetime

def get_broker_list():
    """Get list of running Kafka brokers"""
    try:
        result = subprocess.run(
            ['docker', 'exec', 'kafka1', 'kafka-broker-api-versions', '--bootstrap-server', 'localhost:9092'],
            capture_output=True,
            text=True
        )
        return "Brokers accessible" if result.returncode == 0 else "Broker connection failed"
    except Exception as e:
        return f"Error: {e}"

def get_cluster_metadata():
    """Get cluster metadata"""
    try:
        result = subprocess.run(
            ['docker', 'exec', 'kafka1', 'kafka-metadata-shell', '--snapshot', '/var/lib/kafka/data/__cluster_metadata-0/00000000000000000000.log'],
            capture_output=True,
            text=True,
            timeout=10
        )
        return "Metadata accessible"
    except Exception as e:
        return "Using alternative method"

def main():
    print("Multi-Broker Kafka Cluster Monitor")
    print("=" * 50)
    
    brokers = ['kafka1', 'kafka2', 'kafka3']
    
    while True:
        try:
            print(f"\n[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}]")
            
            # Check each broker status
            print("\n--- Broker Status ---")
            for broker in brokers:
                try:
                    result = subprocess.run(
                        ['docker', 'exec', broker, 'echo', 'Broker alive'],
                        capture_output=True,
                        text=True,
                        timeout=5
                    )
                    status = "RUNNING" if result.returncode == 0 else "STOPPED"
                    print(f"{broker}: {status}")
                except:
                    print(f"{broker}: STOPPED")
            
            # Check topic replication
            print("\n--- Topic Replication Status ---")
            try:
                result = subprocess.run(
                    ['docker', 'exec', 'kafka1', 'kafka-topics', '--describe', 
                     '--topic', 'replicated-messages', '--bootstrap-server', 'localhost:9092'],
                    capture_output=True,
                    text=True
                )
                if result.returncode == 0:
                    lines = result.stdout.strip().split('\n')
                    for line in lines:
                        if 'Partition:' in line:
                            print(line.strip())
                else:
                    print("Could not retrieve topic information")
            except Exception as e:
                print(f"Error checking topics: {e}")
            
            print("\n" + "="*50)
            time.sleep(15)  # Update every 15 seconds
            
        except KeyboardInterrupt:
            print("\nCluster monitoring stopped.")
            break

if __name__ == "__main__":
    main()
Run the cluster monitor:

python3 cluster_monitor.py
Troubleshooting Common Issues
Issue 1: Containers Not Starting
Problem: Services fail to start or exit immediately.

Solution:

# Check logs for specific errors
docker-compose logs [service-name]

# Ensure ports are not in use
netstat -tulpn | grep -E "(2181|9092|9093|9094|8080)"

# Clean up and restart
docker-compose down -v
docker-compose up -d
Issue 2: Producer/Consumer Connection Issues
Problem: Cannot connect to Kafka brokers.

Solution:

# Verify Kafka is accepting connections
docker exec -it kafka1 kafka-topics --list --bootstrap-server localhost:9092

# Check network connectivity
docker network ls
docker network inspect kafka-docker-lab_kafka-network
Issue 3: Topic Creation Failures
Problem: Cannot create topics or topics not visible.

Solution:

# Wait for Kafka to fully start
docker-compose logs kafka1 | grep "started (kafka.server.KafkaServer)"

# Manually create topic with explicit settings
docker exec -it kafka1 kafka-topics --create \
  --topic test-topic \
  --bootstrap-server localhost:9092 \
  --partitions 1 \
  --replication-factor 1
Issue 4: Performance Issues
Problem: Slow message processing or high latency.

Solution:

# Increase container resources
docker-compose down
# Edit docker-compose.yml to add resource limits
docker-compose up -d

# Monitor resource usage
docker stats

# Optimize Kafka configuration for your use case
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Stop and remove containers
docker-compose -f docker-compose-scaled.yml down

# Remove volumes (optional - this will delete all data)
docker-compose -f docker-compose-scaled.yml down -v

# Remove unused Docker resources
docker system prune -f
Conclusion
Congratulations! You have successfully completed Lab 102: Docker for Big Data - Running Apache Kafka with Docker Compose.

What You Accomplished
In this lab, you have:

Deployed a Complete Kafka Ecosystem: Set up Zookeeper, Kafka brokers, and monitoring tools using Docker Compose
Mastered Container Orchestration: Learned how to configure multi-service applications with proper networking and data persistence
Implemented Messaging Patterns: Created functional producers and consumers that demonstrate real-world messaging scenarios
Monitored System Performance: Used multiple approaches to monitor Kafka broker performance and cluster health
Achieved High Availability: Scaled Kafka brokers horizontally and tested fault tolerance in a distributed environment
Why This Matters
The skills you've developed in this lab are crucial for modern data engineering and distributed systems:

Scalability: Understanding how to scale message brokers is essential for handling growing data volumes
Reliability: Learning fault tolerance patterns helps build robust production systems
Monitoring: Performance monitoring skills are critical for maintaining healthy distributed systems
Containerization: Docker and Docker Compose skills are fundamental for modern DevOps practices
Next Steps
To continue building on this foundation:

Explore Kafka Connect for integrating with external systems
Learn about Kafka Streams for real-time data processing
Study advanced Kafka configurations for production environments
Practice with other big data tools in containerized environments
Investigate Kubernetes for orchestrating Kafka at enterprise scale
This lab has provided you with practical experience in deploying and managing Apache Kafka using Docker, preparing you for real-world big data challenges and supporting your journey toward Docker Certified Associate (DCA) certification.
