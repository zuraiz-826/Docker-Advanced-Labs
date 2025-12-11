Lab 11: Introduction to Docker Swarm
Objectives
By the end of this lab, students will be able to:

Understand the fundamentals of Docker Swarm orchestration
Initialize a Docker Swarm cluster using docker swarm init
Add worker nodes to a Swarm cluster using docker swarm join
Deploy multi-container applications using Docker Stack
Scale services within a Swarm cluster
Monitor and inspect Swarm services and nodes
Understand the benefits of container orchestration for production environments
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and images
Familiarity with Docker CLI commands
Knowledge of YAML file structure
Understanding of basic networking concepts
Experience with Linux command line operations
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own virtual machines or install Docker manually.

Your lab environment includes:

3 Ubuntu Linux machines with Docker pre-installed
Machine 1: manager (will serve as Swarm manager)
Machine 2: worker1 (will serve as Swarm worker)
Machine 3: worker2 (will serve as Swarm worker)
All necessary networking configured between machines
Task 1: Initialize a Docker Swarm Cluster
Understanding Docker Swarm
Docker Swarm is Docker's native clustering and orchestration solution. It allows you to create and manage a cluster of Docker nodes, providing high availability, load balancing, and service discovery for your containerized applications.

Subtask 1.1: Connect to the Manager Node
Access your manager machine through the provided terminal
Verify Docker is running and check the version:
docker --version
docker info
Check the current Swarm status:
docker info | grep Swarm
You should see Swarm: inactive indicating that Swarm mode is not yet enabled.

Subtask 1.2: Initialize the Swarm Cluster
Initialize the Docker Swarm on the manager node:
docker swarm init --advertise-addr $(hostname -I | awk '{print $1}')
Note: The --advertise-addr flag specifies the IP address that other nodes will use to connect to this manager.

After successful initialization, you'll see output similar to:
Swarm initialized: current node (abc123def456) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join --token SWMTKN-1-xxxxx-xxxxx 192.168.1.100:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
Save the join command displayed in your output - you'll need it for the next task.
Subtask 1.3: Verify Swarm Initialization
Check the Swarm status:
docker info | grep Swarm
You should now see Swarm: active.

List the nodes in your Swarm:
docker node ls
You should see one node (the manager) with the status Leader.

Task 2: Add Additional Nodes to the Cluster
Subtask 2.1: Retrieve Join Tokens
On the manager node, get the worker join token:
docker swarm join-token worker
Copy the complete join command from the output.
Subtask 2.2: Join Worker Node 1
Connect to your worker1 machine
Verify Docker is running:
docker --version
Join the Swarm using the token from the manager:
docker swarm join --token SWMTKN-1-xxxxx-xxxxx MANAGER_IP:2377
Replace SWMTKN-1-xxxxx-xxxxx with your actual token and MANAGER_IP with your manager's IP address.

You should see confirmation: This node joined a swarm as a worker.
Subtask 2.3: Join Worker Node 2
Connect to your worker2 machine
Repeat the same join process:
docker swarm join --token SWMTKN-1-xxxxx-xxxxx MANAGER_IP:2377
Subtask 2.4: Verify All Nodes Joined
Return to the manager node
List all nodes in the cluster:
docker node ls
You should now see three nodes:

One manager node with Leader status
Two worker nodes with Ready status
Task 3: Deploy a Multi-Container Service Using Docker Stack
Understanding Docker Stack
Docker Stack allows you to deploy and manage multi-service applications using a Compose file format. It's the recommended way to deploy applications in Swarm mode.

Subtask 3.1: Create a Docker Compose File
On the manager node, create a directory for your stack:
mkdir ~/swarm-lab
cd ~/swarm-lab
Create a Docker Compose file for a web application with a database:
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  web:
    image: nginx:alpine
    ports:
      - "80:80"
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
      placement:
        constraints:
          - node.role == worker
    volumes:
      - web-content:/usr/share/nginx/html
    networks:
      - webnet

  redis:
    image: redis:alpine
    deploy:
      replicas: 1
      restart_policy:
        condition: on-failure
    networks:
      - webnet

  visualizer:
    image: dockersamples/visualizer:stable
    ports:
      - "8080:8080"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
    deploy:
      placement:
        constraints:
          - node.role == manager
    networks:
      - webnet

volumes:
  web-content:

networks:
  webnet:
    driver: overlay
EOF
Subtask 3.2: Deploy the Stack
Deploy the stack using Docker Stack:
docker stack deploy -c docker-compose.yml webapp
Verify the stack deployment:
docker stack ls
List the services in your stack:
docker stack services webapp
Subtask 3.3: Verify Service Deployment
Check the status of all services:
docker service ls
Get detailed information about a specific service:
docker service ps webapp_web
Check which nodes are running the services:
docker service ps webapp_web webapp_redis webapp_visualizer
Task 4: Scale Services Within the Swarm
Understanding Service Scaling
One of the key benefits of Docker Swarm is the ability to easily scale services up or down based on demand.

Subtask 4.1: Scale the Web Service
Check the current number of replicas for the web service:
docker service ls
Scale the web service to 5 replicas:
docker service scale webapp_web=5
Monitor the scaling process:
docker service ps webapp_web
Verify the new replica count:
docker service ls
Subtask 4.2: Scale Multiple Services
Scale both web and redis services simultaneously:
docker service scale webapp_web=2 webapp_redis=2
Check the updated service status:
docker service ls
docker service ps webapp_web webapp_redis
Subtask 4.3: Test Service High Availability
Identify which node is running a web service:
docker service ps webapp_web
Simulate a node failure by stopping Docker on one of the worker nodes:
# On worker1 or worker2 (whichever is running containers)
sudo systemctl stop docker
Return to the manager and observe how Swarm handles the failure:
docker service ps webapp_web
docker node ls
Restart Docker on the affected node:
# On the affected worker node
sudo systemctl start docker
Check node status recovery:
# On manager
docker node ls
Task 5: Inspect the State of Services and Nodes
Subtask 5.1: Comprehensive Service Inspection
List all services across all stacks:
docker service ls
Get detailed information about the web service:
docker service inspect webapp_web
View service logs:
docker service logs webapp_web
Check service configuration:
docker service inspect --pretty webapp_web
Subtask 5.2: Node Management and Inspection
List all nodes with detailed information:
docker node ls
Inspect a specific node:
docker node inspect worker1
View node information in a readable format:
docker node inspect --pretty worker1
Check resource usage across nodes:
docker system df
docker system events --since 10m
Subtask 5.3: Network and Volume Inspection
List Swarm networks:
docker network ls
Inspect the overlay network:
docker network inspect webapp_webnet
List volumes created by the stack:
docker volume ls
Subtask 5.4: Access the Deployed Application
Find the IP address of your manager node:
hostname -I
Open a web browser or use curl to access:

Main application: http://MANAGER_IP
Visualizer: http://MANAGER_IP:8080
Test load balancing by refreshing the page multiple times and observing the container IDs.

Advanced Operations
Rolling Updates
Update the web service with a new image:
docker service update --image nginx:latest webapp_web
Monitor the rolling update:
docker service ps webapp_web
Draining Nodes
Drain a worker node for maintenance:
docker node update --availability drain worker1
Observe how services are redistributed:
docker service ps webapp_web
docker node ls
Return the node to active status:
docker node update --availability active worker1
Cleanup
Remove the Stack
Remove the deployed stack:
docker stack rm webapp
Verify stack removal:
docker stack ls
docker service ls
Leave the Swarm (Optional)
On worker nodes, leave the Swarm:
docker swarm leave
On the manager node, remove the worker nodes:
docker node rm worker1 worker2
Leave Swarm mode on the manager:
docker swarm leave --force
Troubleshooting Tips
Common Issues and Solutions
Issue: Node fails to join the Swarm

Solution: Check network connectivity between nodes and ensure ports 2377, 7946, and 4789 are open
Issue: Services not starting

Solution: Check service logs using docker service logs SERVICE_NAME and verify image availability
Issue: Cannot access deployed application

Solution: Verify port mappings and ensure services are running on expected nodes
Issue: Swarm initialization fails

Solution: Ensure Docker daemon is running and try specifying a different advertise address
Useful Commands for Debugging
# Check Docker daemon status
sudo systemctl status docker

# View detailed service information
docker service inspect --pretty SERVICE_NAME

# Check node connectivity
docker node ls

# View system-wide information
docker system info

# Monitor real-time events
docker system events
Conclusion
In this lab, you have successfully:

Initialized a Docker Swarm cluster and understood the role of manager and worker nodes
Added multiple nodes to create a distributed container orchestration system
Deployed a multi-service application using Docker Stack with proper networking and volume management
Scaled services dynamically to handle varying workloads
Inspected and monitored the health and status of your Swarm cluster
Why This Matters
Docker Swarm provides a production-ready orchestration platform that offers:

High Availability: Automatic failover and service recovery
Load Balancing: Built-in load distribution across nodes
Service Discovery: Automatic service registration and discovery
Rolling Updates: Zero-downtime application updates
Scalability: Easy horizontal scaling of services
These skills are essential for:

DevOps Engineers managing containerized applications in production
System Administrators implementing scalable infrastructure
Developers understanding deployment and orchestration concepts
Docker Certified Associate (DCA) certification preparation
The knowledge gained in this lab provides a solid foundation for working with container orchestration platforms and prepares you for more advanced topics like Kubernetes, service mesh architectures, and cloud-native application development.
