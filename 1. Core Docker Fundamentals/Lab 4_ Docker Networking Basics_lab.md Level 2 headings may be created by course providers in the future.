.

🐳 Lab 4: Docker Networking Basics
🎯 Objectives
By the end of this lab, you will be able to:

🔹 Understand Docker's default bridge network and how it works
🔹 Create and manage custom Docker networks
🔹 Connect multiple containers to the same network
🔹 Use Docker networking commands to inspect and manage networks
🔹 Configure container communication using internal networking
🔹 Troubleshoot basic Docker networking issues

📋 Prerequisites
Before starting this lab, you should have:

✅ Basic understanding of Docker containers and images
✅ Familiarity with command-line interface (CLI)
✅ Completed previous Docker labs or equivalent knowledge
✅ Understanding of basic networking concepts (IP addresses, ports)

🛠️ Lab Environment Setup
☁️ Al Nafi Cloud Machines
This lab uses Al Nafi's pre-configured Linux-based cloud machines. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your cloud machine comes with:
🐳 Docker Engine pre-installed and configured
🌐 All necessary networking tools
🔓 Root/sudo access for Docker commands

📝 Task 1: Understanding Docker's Default Bridge Network
🔍 Subtask 1.1: Explore the Default Bridge Network
Docker automatically creates a default bridge network when installed. Let's examine this network.

Step 1: List all Docker networks

bash
docker network ls
Expected Output:

text
NETWORK ID     NAME      DRIVER    SCOPE
abcd1234efgh   bridge    bridge    local
ijkl5678mnop   host      host      local
qrst9012uvwx   none      null      local
Step 2: Inspect the default bridge network

bash
docker network inspect bridge
This command shows detailed information about:
📍 Subnet – The IP range assigned to containers
📍 Gateway – The gateway IP address
📍 Connected containers – Currently running containers on this network

🚀 Subtask 1.2: Run a Container on the Default Network
Step 1: Run a simple container on the default bridge network

bash
docker run -d --name web-server-1 nginx:alpine
Step 2: Verify the container is running

bash
docker ps
Step 3: Inspect the bridge network again

bash
docker network inspect bridge
Look for the Containers section to see your web-server-1 container with its assigned IP address.

🔗 Task 2: Running Multiple Containers on the Same Network
📦 Subtask 2.1: Create Multiple Containers on Default Network
Step 1: Run a second nginx container

bash
docker run -d --name web-server-2 nginx:alpine
Step 2: Run a third container with Alpine Linux

bash
docker run -d --name alpine-client alpine:latest sleep 3600
Step 3: Verify all containers are running

bash
docker ps
Step 4: Check the bridge network

bash
docker network inspect bridge
📡 Subtask 2.2: Test Container Communication
Step 1: Get the IP address of web-server-1

bash
docker inspect web-server-1 | grep IPAddress
Step 2: Test connectivity from alpine-client to web-server-1

bash
docker exec alpine-client ping -c 3 <IP_ADDRESS_OF_WEB_SERVER_1>
Replace <IP_ADDRESS_OF_WEB_SERVER_1> with the actual IP address

Step 3: Test HTTP connectivity

bash
docker exec alpine-client wget -qO- http://<IP_ADDRESS_OF_WEB_SERVER_1>
✅ You should see the default nginx welcome page HTML content.

🏗️ Task 3: Creating and Managing Custom Networks
🔧 Subtask 3.1: Create a Custom Bridge Network
Step 1: Create a custom network named app-network

bash
docker network create app-network
Step 2: List networks to confirm creation

bash
docker network ls
Step 3: Inspect the custom network

bash
docker network inspect app-network
🎛️ Subtask 3.2: Create Additional Network Types
Step 1: Create a custom network with specific subnet

bash
docker network create --driver bridge --subnet=192.168.100.0/24 custom-subnet-network
Step 2: Verify the network creation

bash
docker network inspect custom-subnet-network
📌 Notice the custom subnet configuration in the IPAM section.

🚦 Task 4: Using --network Flag to Specify Networks
📍 Subtask 4.1: Run Containers on Custom Network
Step 1: Run a container on app-network

bash
docker run -d --name app-server-1 --network app-network nginx:alpine
Step 2: Run a second container on the same network

bash
docker run -d --name app-server-2 --network app-network nginx:alpine
Step 3: Run a client container on the custom network

bash
docker run -d --name app-client --network app-network alpine:latest sleep 3600
🔄 Subtask 4.2: Test Custom Network Communication
Step 1: Test connectivity using container names (DNS resolution)

bash
docker exec app-client ping -c 3 app-server-1
Step 2: Test HTTP connectivity using container names

bash
docker exec app-client wget -qO- http://app-server-1
💡 Important Note: On custom networks, containers can communicate using container names as hostnames – this is NOT available on the default bridge network!

🛡️ Subtask 4.3: Compare Network Isolation
Step 1: Try to ping a container on the default network from custom network

bash
docker exec app-client ping -c 3 web-server-1
❌ This should fail, demonstrating network isolation between different Docker networks.

⚡ Task 5: Advanced Container Communication and Network Management
🔗 Subtask 5.1: Connect Containers to Multiple Networks
Step 1: Connect an existing container to an additional network

bash
docker network connect app-network alpine-client
Step 2: Verify the connection

bash
docker network inspect app-network
Step 3: Test connectivity from the multi-network container

bash
docker exec alpine-client ping -c 3 app-server-1
🚪 Subtask 5.2: Port Mapping and External Access
Step 1: Run a container with port mapping

bash
docker run -d --name web-public --network app-network -p 8080:80 nginx:alpine
Step 2: Test external access (from host)

bash
curl http://localhost:8080
Step 3: Test internal network access

bash
docker exec app-client wget -qO- http://web-public
🧹 Subtask 5.3: Network Cleanup and Management
Step 1: Disconnect a container from a network

bash
docker network disconnect app-network alpine-client
Step 2: Remove containers

bash
docker stop web-server-1 web-server-2 alpine-client app-server-1 app-server-2 app-client web-public
docker rm web-server-1 web-server-2 alpine-client app-server-1 app-server-2 app-client web-public
Step 3: Remove custom networks

bash
docker network rm app-network custom-subnet-network
Step 4: Verify cleanup

bash
docker network ls
docker ps -a
🚨 Troubleshooting Common Issues
⚠️ Issue 1: Container Cannot Communicate
Symptoms: Ping or HTTP requests fail between containers
Solutions:
🔸 Verify containers are on the same network: docker network inspect <network_name>
🔸 Check container names and IP addresses
🔸 Ensure containers are running: docker ps

⚠️ Issue 2: Network Already Exists Error
Symptoms: Error when creating network with existing name
Solutions:
🔸 List existing networks: docker network ls
🔸 Use a different network name
🔸 Remove existing network if not needed: docker network rm <network_name>

⚠️ Issue 3: Cannot Remove Network
Symptoms: Error when trying to remove network
Solutions:
🔸 Check for connected containers: docker network inspect <network_name>
🔸 Stop and remove connected containers first
🔸 Disconnect containers: docker network disconnect <network_name> <container_name>

📖 Key Networking Commands Reference
🌐 Network Management
bash
docker network ls                          # 📋 List all networks
docker network create <network_name>       # 🆕 Create new network
docker network rm <network_name>          # 🗑️ Remove network
docker network inspect <network_name>     # 🔍 Inspect network details
📦 Container Network Operations
bash
docker run --network <network_name>       # 🚀 Run container on specific network
docker network connect <network> <container>    # 🔗 Connect container to network
docker network disconnect <network> <container> # 🔌 Disconnect container from network
🔍 Container Inspection
bash
docker inspect <container_name>           # 📊 Get container details including IP
docker exec <container> <command>         # ⚡ Execute command in container
💡 Best Practices for Docker Networking
✅ Use Custom Networks – Always create custom networks for multi-container applications instead of relying on the default bridge
✅ Meaningful Names – Use descriptive names for networks and containers
✅ Network Segmentation – Separate different application tiers using different networks
✅ DNS Resolution – Leverage container name-based DNS resolution on custom networks
✅ Port Mapping – Only expose ports that need external access
✅ Cleanup – Regularly remove unused networks and containers

🎉 Conclusion
In this lab, you have successfully learned the fundamentals of Docker networking! You explored Docker's default bridge network, created custom networks, and demonstrated container communication both within and across networks.

🏆 Key Accomplishments:
🔹 Understood how Docker's default bridge network operates
🔹 Created and managed custom Docker networks
🔹 Connected multiple containers to the same network
🔹 Used container names for DNS resolution on custom networks
🔹 Implemented network isolation and security
🔹 Managed port mapping for external access

🌟 Why This Matters:
Docker networking is crucial for building scalable, secure containerized applications. Understanding these concepts enables you to:

Design proper network architectures for microservices

Implement security through network segmentation

Troubleshoot connectivity issues in containerized environments

These skills are essential for the Docker Certified Associate (DCA) certification and real-world container orchestration scenarios. In production environments, proper networking configuration ensures your applications can communicate securely and efficiently while maintaining isolation between different services and environments.

Happy Networking! 🌐🐳

