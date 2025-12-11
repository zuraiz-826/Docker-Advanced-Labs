Lab 2: Understanding Docker Images
Lab Objectives
By the end of this lab, students will be able to:

Understand the concept of Docker images and their role in containerization
Navigate and search Docker Hub for available images
Pull Docker images from Docker Hub to their local machine
List, inspect, and manage Docker images on their system
Remove unused Docker images to manage storage space
Use specific image tags when running containers
Examine the layered architecture of Docker images
Apply best practices for Docker image management
Prerequisites
Before starting this lab, students should have:

Basic understanding of command-line interface (CLI) operations
Completion of Lab 1 or equivalent Docker installation knowledge
Familiarity with basic Linux commands
Understanding of what containers are conceptually
Lab Environment Setup
Good News: Al Nafi provides ready-to-use Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to access your environment - no need to build your own virtual machine or install Docker manually.

Your cloud machine will include:

Ubuntu Linux operating system
Docker Engine pre-installed and configured
Terminal access with sudo privileges
Internet connectivity for downloading images
Task 1: Exploring Docker Hub
Subtask 1.1: Understanding Docker Hub
Docker Hub is the world's largest repository of container images. Think of it as an app store for Docker images where developers share pre-built applications and operating systems.

Step 1: Open your web browser and navigate to Docker Hub

https://hub.docker.com
Step 2: Explore the interface without logging in

Notice the search bar at the top
Observe the featured repositories
Look at the categories of available images
Subtask 1.2: Searching for Images via Web Interface
Step 1: Search for popular images using the web interface

Search for ubuntu in the search bar
Click on the official Ubuntu repository
Examine the repository details:
Description and documentation
Available tags (versions)
Pull command
Number of downloads
Step 2: Explore other popular images

Search for nginx (web server)
Search for mysql (database)
Search for python (programming language)
Subtask 1.3: Searching for Images via Command Line
Step 1: Open your terminal in the cloud machine

Step 2: Search for images using Docker CLI

docker search ubuntu
Step 3: Understand the output columns

NAME: Repository name
DESCRIPTION: Brief description of the image
STARS: Community rating (like GitHub stars)
OFFICIAL: Whether it's an official image
AUTOMATED: Whether it's automatically built
Step 4: Search for other images

docker search nginx
docker search --limit 5 python
Task 2: Pulling Docker Images
Subtask 2.1: Understanding Image Tags
Docker images use tags to specify versions. The latest tag refers to the most recent version, but it's better to use specific version tags for production environments.

Subtask 2.2: Pulling the Ubuntu Image
Step 1: Pull the latest Ubuntu image

docker pull ubuntu
Step 2: Observe the download process

Notice the different layers being downloaded
Each layer represents a filesystem change
Layers are cached for efficiency
Step 3: Pull a specific Ubuntu version

docker pull ubuntu:20.04
Step 4: Pull another specific version

docker pull ubuntu:22.04
Subtask 2.3: Pulling Other Popular Images
Step 1: Pull an Nginx web server image

docker pull nginx:alpine
Step 2: Pull a Python runtime image

docker pull python:3.9-slim
Step 3: Understanding why we chose specific tags

alpine: Smaller, security-focused Linux distribution
slim: Reduced size version with fewer packages
3.9: Specific Python version for consistency
Task 3: Managing Docker Images
Subtask 3.1: Listing Docker Images
Step 1: List all images on your system

docker images
Step 2: Understand the output columns

REPOSITORY: Image name
TAG: Version or variant
IMAGE ID: Unique identifier (first 12 characters)
CREATED: When the image was built
SIZE: Disk space used
Step 3: List images with additional formatting

docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"
Subtask 3.2: Filtering and Sorting Images
Step 1: Filter images by repository name

docker images ubuntu
Step 2: Show only image IDs

docker images -q
Step 3: Show all images including intermediate layers

docker images -a
Subtask 3.3: Removing Docker Images
Step 1: Remove a specific image by name and tag

docker rmi ubuntu:20.04
Step 2: Remove an image by IMAGE ID

# First, get the IMAGE ID
docker images python:3.9-slim

# Then remove using the ID (use first few characters)
docker rmi [IMAGE_ID]
Step 3: Force remove an image (if containers are using it)

docker rmi -f nginx:alpine
Step 4: Remove multiple images at once

docker rmi ubuntu:22.04 python:3.9-slim
Step 5: Remove all unused images

docker image prune
Step 6: Remove all images (use with caution)

docker rmi $(docker images -q)
Task 4: Working with Image Tags
Subtask 4.1: Understanding Tag Importance
Tags help you:

Specify exact versions for reproducibility
Choose optimized variants (alpine, slim)
Avoid unexpected changes from latest tag updates
Subtask 4.2: Running Containers with Specific Tags
Step 1: Pull multiple versions of the same image

docker pull ubuntu:18.04
docker pull ubuntu:20.04
docker pull ubuntu:22.04
Step 2: Run containers with different Ubuntu versions

# Run Ubuntu 18.04
docker run -it ubuntu:18.04 /bin/bash
Step 3: Check the Ubuntu version inside the container

cat /etc/os-release
exit
Step 4: Run Ubuntu 22.04 and compare

docker run -it ubuntu:22.04 /bin/bash
cat /etc/os-release
exit
Subtask 4.3: Creating Custom Tags
Step 1: Tag an existing image with a custom name

docker tag ubuntu:22.04 my-ubuntu:production
Step 2: Verify the new tag was created

docker images | grep my-ubuntu
Step 3: Both tags point to the same image (same IMAGE ID)

docker images ubuntu:22.04
docker images my-ubuntu:production
Task 5: Inspecting Docker Image Layers
Subtask 5.1: Understanding Docker Image Layers
Docker images are built in layers, like a stack of transparent sheets. Each layer represents a change to the filesystem. This layered approach enables:

Efficiency: Shared layers between images
Caching: Faster builds and pulls
Storage optimization: Reduced disk usage
Subtask 5.2: Inspecting Image Details
Step 1: Get detailed information about an image

docker inspect ubuntu:22.04
Step 2: Extract specific information using formatting

# Get just the architecture
docker inspect --format='{{.Architecture}}' ubuntu:22.04

# Get the creation date
docker inspect --format='{{.Created}}' ubuntu:22.04

# Get the size
docker inspect --format='{{.Size}}' ubuntu:22.04
Subtask 5.3: Viewing Image History and Layers
Step 1: View the build history of an image

docker history ubuntu:22.04
Step 2: Understand the history output

IMAGE: Layer ID
CREATED: When the layer was created
CREATED BY: Command that created the layer
SIZE: Size added by this layer
COMMENT: Additional information
Step 3: View history without truncation

docker history --no-trunc ubuntu:22.04
Step 4: Compare histories of different images

docker history nginx:alpine
docker history python:3.9-slim
Subtask 5.4: Analyzing Layer Efficiency
Step 1: Pull a larger image to see more layers

docker pull node:16
Step 2: Compare the layer structure

docker history node:16
docker history ubuntu:22.04
Step 3: Notice how Node.js image builds upon base layers

Base operating system layers
Package manager updates
Node.js installation layers
Configuration layers
Task 6: Best Practices and Cleanup
Subtask 6.1: Image Management Best Practices
Key Principles:

Use specific tags instead of latest for production
Regularly clean up unused images
Choose minimal base images (alpine, slim) when possible
Understand the layers you're adding to images
Subtask 6.2: System Cleanup
Step 1: Check current disk usage

docker system df
Step 2: Clean up unused images

docker image prune
Step 3: Clean up everything unused (images, containers, networks)

docker system prune
Step 4: Aggressive cleanup (remove all unused images, not just dangling ones)

docker system prune -a
Subtask 6.3: Monitoring Image Usage
Step 1: List images sorted by size

docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | sort -k3 -h
Step 2: Find large images consuming space

docker images --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}" | head -10
Troubleshooting Common Issues
Issue 1: Permission Denied
Problem: Getting permission denied when running Docker commands Solution:

sudo docker [command]
# Or add user to docker group (requires logout/login)
sudo usermod -aG docker $USER
Issue 2: Image Pull Fails
Problem: Network issues or repository not found Solution:

# Check internet connectivity
ping docker.io

# Verify image name spelling
docker search [image-name]
Issue 3: Cannot Remove Image
Problem: Image is being used by a container Solution:

# List containers using the image
docker ps -a

# Remove containers first, then image
docker rm [container-id]
docker rmi [image-name]
Issue 4: Disk Space Issues
Problem: Running out of disk space Solution:

# Check Docker disk usage
docker system df

# Clean up aggressively
docker system prune -a --volumes
Lab Verification
To verify you've completed the lab successfully, run these commands:

# Should show multiple images
docker images

# Should show image details
docker inspect ubuntu:latest

# Should show layer history
docker history ubuntu:latest

# Should show system usage
docker system df
Conclusion
Congratulations! You have successfully completed Lab 2: Understanding Docker Images. In this lab, you have accomplished the following:

Key Achievements:

Explored Docker Hub: You learned how to search for and evaluate Docker images both through the web interface and command line
Mastered Image Management: You can now pull, list, tag, and remove Docker images efficiently
Understood Image Layers: You gained insight into Docker's layered architecture and how it optimizes storage and performance
Applied Best Practices: You learned how to use specific tags, manage disk space, and maintain a clean Docker environment
Why This Matters: Docker images are the foundation of containerization. Understanding how to work with images effectively is crucial because:

Consistency: Using specific image tags ensures your applications run the same way across different environments
Efficiency: Understanding layers helps you optimize image sizes and build times
Security: Knowing how to manage and update images helps maintain secure deployments
Resource Management: Proper image cleanup prevents disk space issues in production environments
Next Steps: With this foundation in Docker images, you're now ready to:

Create your own custom Docker images using Dockerfiles
Build multi-stage images for optimized production deployments
Implement image scanning and security practices
Work with private image registries
The skills you've learned in this lab are directly applicable to the Docker Certified Associate (DCA) certification and real-world container deployments. You now have the knowledge to make informed decisions about image selection, management, and optimization in your containerization journey
