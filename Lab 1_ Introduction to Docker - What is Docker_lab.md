Lab 1: Introduction to Docker - What is Docker?
Lab Objectives
By the end of this lab, students will be able to:

Understand what Docker is and its core concepts
Differentiate between Docker containers and images
Install Docker on a Linux environment
Execute basic Docker commands using the Docker CLI
Run their first Docker container
Explain the key differences between containers and virtual machines
Navigate the Docker ecosystem with confidence
Prerequisites
Before starting this lab, students should have:

Basic familiarity with Linux command line interface
Understanding of fundamental computing concepts
No prior Docker experience required - this is a beginner-friendly lab
Lab Environment Setup
Good News! Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click Start Lab and you'll have access to a fully configured Ubuntu environment with internet connectivity. No need to build your own virtual machine or worry about system compatibility issues.

Your cloud machine will include:

Ubuntu 20.04 LTS or newer
Terminal access with sudo privileges
Internet connectivity for downloading Docker
Task 1: Install Docker on Linux Environment
Subtask 1.1: Update System Packages
First, let's ensure your system is up to date with the latest packages.

sudo apt update
sudo apt upgrade -y
What's happening here?

apt update refreshes the package list from repositories
apt upgrade -y installs available updates automatically
Subtask 1.2: Install Required Dependencies
Install packages that allow apt to use repositories over HTTPS:

sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release
Subtask 1.3: Add Docker's Official GPG Key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
Subtask 1.4: Set Up Docker Repository
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
Subtask 1.5: Install Docker Engine
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io
Subtask 1.6: Verify Docker Installation
Check if Docker is installed and running:

sudo systemctl status docker
You should see output indicating that Docker is active (running).

Subtask 1.7: Add User to Docker Group
To run Docker commands without sudo, add your user to the docker group:

sudo usermod -aG docker $USER
Important: Log out and log back in for this change to take effect, or run:

newgrp docker
Subtask 1.8: Test Docker Installation
docker --version
You should see output similar to:

Docker version 24.0.x, build xxxxxxx
Task 2: Learn About Docker Containers and Images
Subtask 2.1: Understanding Docker Images
Docker Images are like blueprints or templates. Think of them as:

A recipe for creating containers
Read-only templates containing application code, libraries, and dependencies
Stored in layers for efficiency
Subtask 2.2: Understanding Docker Containers
Docker Containers are running instances of images. Think of them as:

A running application created from an image
Isolated processes with their own file system
Lightweight and portable across different environments
Subtask 2.3: The Relationship Between Images and Containers
Analogy: If a Docker image is like a cookie cutter, then containers are the actual cookies made from that cutter. You can make many cookies (containers) from one cookie cutter (image).

Task 3: Run Your First Container
Subtask 3.1: Execute the Hello World Container
Let's run the classic first Docker command:

docker run hello-world
What happens when you run this command?

Docker looks for the hello-world image locally
If not found, it downloads the image from Docker Hub
Creates a new container from the image
Runs the container
The container displays a welcome message and exits
Subtask 3.2: Analyze the Output
You should see output similar to:

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
Congratulations! You've successfully run your first Docker container.

Task 4: Understand Container vs Virtual Machine Differences
Subtask 4.1: Key Differences Overview
Aspect	Virtual Machines	Docker Containers
Resource Usage	Heavy (GB of RAM)	Lightweight (MB of RAM)
Startup Time	Minutes	Seconds
Isolation	Complete OS isolation	Process-level isolation
Portability	Limited	Highly portable
Host OS	Runs multiple full OS	Shares host OS kernel
Subtask 4.2: Visual Understanding
Virtual Machines:

Each VM includes a full operating system
Hypervisor manages multiple VMs
Higher resource overhead
Docker Containers:

Share the host OS kernel
Docker Engine manages containers
Minimal resource overhead
Subtask 4.3: When to Use Each
Use Virtual Machines when:

You need complete isolation
Running different operating systems
Legacy applications requiring specific OS versions
Use Docker Containers when:

You want fast deployment
Microservices architecture
Development environment consistency
CI/CD pipelines
Task 5: Explore Docker CLI and Basic Commands
Subtask 5.1: List Running Containers
docker ps
This shows currently running containers. Since hello-world exited immediately, you'll likely see an empty list.

Subtask 5.2: List All Containers (Including Stopped)
docker ps -a
Now you should see the hello-world container with status "Exited".

Subtask 5.3: List Docker Images
docker images
This displays all images stored locally on your system. You should see the hello-world image.

Subtask 5.4: Get Detailed Information About Docker
docker info
This provides comprehensive information about your Docker installation.

Subtask 5.5: Run an Interactive Container
Let's run a more interactive container:

docker run -it ubuntu:latest /bin/bash
Command breakdown:

-i: Interactive mode
-t: Allocate a pseudo-TTY
ubuntu:latest: Use the latest Ubuntu image
/bin/bash: Run bash shell
You're now inside a Ubuntu container! Try some commands:

ls
pwd
cat /etc/os-release
To exit the container:

exit
Subtask 5.6: Run a Container in Background
docker run -d --name my-nginx nginx:latest
Command breakdown:

-d: Run in detached mode (background)
--name my-nginx: Give the container a custom name
nginx:latest: Use the latest Nginx web server image
Subtask 5.7: Check Running Containers Again
docker ps
You should now see the Nginx container running.

Subtask 5.8: Stop a Running Container
docker stop my-nginx
Subtask 5.9: Remove a Container
docker rm my-nginx
Subtask 5.10: Remove an Image
docker rmi hello-world
Note: You can only remove images that aren't being used by any containers.

Essential Docker Commands Summary
Here's a quick reference of the commands you've learned:

Command	Purpose
docker run <image>	Create and run a new container
docker ps	List running containers
docker ps -a	List all containers
docker images	List local images
docker stop <container>	Stop a running container
docker rm <container>	Remove a container
docker rmi <image>	Remove an image
docker info	Display system information
Troubleshooting Common Issues
Issue 1: Permission Denied
Problem: Getting permission denied errors when running Docker commands.

Solution:

sudo usermod -aG docker $USER
newgrp docker
Issue 2: Docker Daemon Not Running
Problem: Error message about Docker daemon not running.

Solution:

sudo systemctl start docker
sudo systemctl enable docker
Issue 3: Image Download Fails
Problem: Cannot download images from Docker Hub.

Solution: Check internet connectivity and try again:

ping google.com
docker run hello-world
Lab Conclusion
Congratulations! You have successfully completed your introduction to Docker. Here's what you accomplished:

Key Achievements
Installed Docker on a Linux environment using official repositories
Learned fundamental concepts of containers and images
Ran your first container using the hello-world image
Understood the differences between containers and virtual machines
Mastered basic Docker CLI commands for managing containers and images
Why This Matters
Docker has revolutionized how applications are developed, shipped, and deployed. The skills you've learned today form the foundation for:

Modern DevOps practices
Microservices architecture
Cloud-native development
Consistent development environments
Efficient application deployment
Next Steps
Now that you understand Docker basics, you're ready to:

Explore creating custom Docker images
Learn about Docker Compose for multi-container applications
Understand Docker networking and volumes
Practice with real-world application containerization
Certification Path
This lab aligns with the Docker Certified Associate (DCA) certification objectives, specifically:

Domain 1: Orchestration (25% of exam)
Domain 2: Image Creation, Management, and Registry (20% of exam)
Domain 6: Storage and Volumes (10% of exam)
Keep practicing these fundamental concepts as they are essential building blocks for advanced Docker topics and professional certification success.

Well done on completing your first Docker lab! You're now part of the containerization revolution that's transforming modern software development
