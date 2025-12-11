Lab 103: Docker for Cloud - Running Docker Containers on Microsoft Azure
Objectives
By the end of this lab, you will be able to:

Set up Azure CLI and authenticate with your Azure account
Create and configure Azure Container Registry (ACR) for storing Docker images
Build a Docker image and push it to Azure Container Registry
Deploy Docker containers using Azure Container Instances (ACI)
Configure networking to expose containers to the internet
Monitor container performance and resource usage through Azure Monitor
Understand the fundamentals of containerized applications in cloud environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with command-line interface operations
Basic knowledge of web applications and HTTP protocols
An active Microsoft Azure account (free tier is sufficient)
Understanding of basic networking concepts
Note: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own virtual machine.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-installed with:

Docker Engine
Azure CLI
Git
Text editors (nano, vim)
All necessary development tools
Task 1: Create Azure Account and Set Up Azure CLI
Subtask 1.1: Verify Azure CLI Installation
First, let's verify that Azure CLI is properly installed on your cloud machine.

# Check Azure CLI version
az --version

# If Azure CLI is not installed, install it
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
Subtask 1.2: Login to Azure Account
Authenticate with your Azure account using the Azure CLI.

# Login to Azure (this will open a browser window)
az login

# If you're using a headless environment, use device code authentication
az login --use-device-code
Follow the authentication prompts in your browser or use the device code as instructed.

Subtask 1.3: Set Default Subscription
If you have multiple Azure subscriptions, set the default one for this lab.

# List available subscriptions
az account list --output table

# Set default subscription (replace with your subscription ID)
az account set --subscription "your-subscription-id"

# Verify current subscription
az account show --output table
Subtask 1.4: Create Resource Group
Create a resource group to organize all Azure resources for this lab.

# Create resource group
az group create \
    --name docker-lab-rg \
    --location eastus

# Verify resource group creation
az group show --name docker-lab-rg --output table
Task 2: Build Docker Image and Push to Azure Container Registry
Subtask 2.1: Create Azure Container Registry
Set up Azure Container Registry to store your Docker images.

# Create Azure Container Registry
az acr create \
    --resource-group docker-lab-rg \
    --name dockerlabacr$(date +%s) \
    --sku Basic \
    --admin-enabled true

# Store ACR name in variable for later use
ACR_NAME=$(az acr list --resource-group docker-lab-rg --query "[0].name" --output tsv)
echo "ACR Name: $ACR_NAME"
Subtask 2.2: Create Sample Web Application
Create a simple Node.js web application for containerization.

# Create project directory
mkdir docker-web-app
cd docker-web-app

# Create package.json
cat > package.json << 'EOF'
{
  "name": "docker-web-app",
  "version": "1.0.0",
  "description": "Simple web app for Docker Azure lab",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
EOF

# Create server.js
cat > server.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.send(`
    <html>
      <head><title>Docker Azure Lab</title></head>
      <body style="font-family: Arial, sans-serif; text-align: center; padding: 50px;">
        <h1 style="color: #0078d4;">Hello from Azure Container Instance!</h1>
        <p>This application is running in a Docker container on Microsoft Azure.</p>
        <p>Container ID: ${process.env.HOSTNAME || 'unknown'}</p>
        <p>Current time: ${new Date().toISOString()}</p>
      </body>
    </html>
  `);
});

app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

app.listen(port, () => {
  console.log(`Server running on port ${port}`);
});
EOF
Subtask 2.3: Create Dockerfile
Create a Dockerfile to containerize the web application.

# Create Dockerfile
cat > Dockerfile << 'EOF'
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory in container
WORKDIR /usr/src/app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install --only=production

# Copy application code
COPY . .

# Expose port 3000
EXPOSE 3000

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs

# Start the application
CMD ["npm", "start"]
EOF
Subtask 2.4: Build and Test Docker Image Locally
Build the Docker image and test it locally before pushing to Azure.

# Build Docker image
docker build -t docker-web-app:v1.0 .

# Run container locally to test
docker run -d -p 8080:3000 --name test-container docker-web-app:v1.0

# Test the application
curl http://localhost:8080

# Check container logs
docker logs test-container

# Stop and remove test container
docker stop test-container
docker rm test-container
Subtask 2.5: Login to Azure Container Registry
Authenticate Docker with your Azure Container Registry.

# Login to ACR
az acr login --name $ACR_NAME

# Get ACR login server
ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --query loginServer --output tsv)
echo "ACR Login Server: $ACR_LOGIN_SERVER"
Subtask 2.6: Tag and Push Image to ACR
Tag your Docker image and push it to Azure Container Registry.

# Tag image for ACR
docker tag docker-web-app:v1.0 $ACR_LOGIN_SERVER/docker-web-app:v1.0

# Push image to ACR
docker push $ACR_LOGIN_SERVER/docker-web-app:v1.0

# Verify image in ACR
az acr repository list --name $ACR_NAME --output table
az acr repository show-tags --name $ACR_NAME --repository docker-web-app --output table
Task 3: Deploy Docker Container to Azure Container Instances
Subtask 3.1: Get ACR Credentials
Retrieve credentials needed for ACI to pull images from ACR.

# Get ACR credentials
ACR_USERNAME=$(az acr credential show --name $ACR_NAME --query username --output tsv)
ACR_PASSWORD=$(az acr credential show --name $ACR_NAME --query passwords[0].value --output tsv)

echo "ACR Username: $ACR_USERNAME"
echo "ACR Password: [HIDDEN]"
Subtask 3.2: Deploy Container to ACI
Create and deploy your container using Azure Container Instances.

# Deploy container to ACI
az container create \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --image $ACR_LOGIN_SERVER/docker-web-app:v1.0 \
    --registry-login-server $ACR_LOGIN_SERVER \
    --registry-username $ACR_USERNAME \
    --registry-password $ACR_PASSWORD \
    --dns-name-label docker-web-app-$(date +%s) \
    --ports 3000 \
    --cpu 1 \
    --memory 1 \
    --restart-policy Always

# Check deployment status
az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query "{Status:instanceView.state, FQDN:ipAddress.fqdn, IP:ipAddress.ip}" \
    --output table
Subtask 3.3: Verify Container Deployment
Check that your container is running successfully.

# Get container details
az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --output table

# Check container logs
az container logs \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci

# Get container events
az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query instanceView.events \
    --output table
Task 4: Expose Container to Internet Using Azure Networking
Subtask 4.1: Get Container Public URL
Retrieve the public URL for your deployed container.

# Get container FQDN
CONTAINER_FQDN=$(az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query ipAddress.fqdn \
    --output tsv)

echo "Container URL: http://$CONTAINER_FQDN:3000"

# Get container IP address
CONTAINER_IP=$(az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query ipAddress.ip \
    --output tsv)

echo "Container IP: $CONTAINER_IP"
Subtask 4.2: Test Internet Connectivity
Test that your container is accessible from the internet.

# Test HTTP connectivity
curl -i http://$CONTAINER_FQDN:3000

# Test health endpoint
curl -i http://$CONTAINER_FQDN:3000/health

# Test with different HTTP methods
curl -X GET http://$CONTAINER_FQDN:3000/health
Subtask 4.3: Configure Custom Domain (Optional)
If you have a custom domain, you can configure DNS to point to your container.

# Display IP for DNS configuration
echo "Configure your DNS A record to point to: $CONTAINER_IP"
echo "Example DNS configuration:"
echo "Type: A"
echo "Name: docker-lab (or your preferred subdomain)"
echo "Value: $CONTAINER_IP"
echo "TTL: 300"
Task 5: Monitor Container Resource Usage Through Azure Monitor
Subtask 5.1: Enable Container Insights
Set up monitoring for your container instance.

# Get container resource usage
az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query "containers[0].instanceView.currentState" \
    --output table

# Check resource limits and requests
az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query "containers[0].resources" \
    --output json
Subtask 5.2: Monitor Container Metrics
View real-time metrics for your container.

# Get container metrics (CPU and Memory)
az monitor metrics list \
    --resource "/subscriptions/$(az account show --query id --output tsv)/resourceGroups/docker-lab-rg/providers/Microsoft.ContainerInstance/containerGroups/docker-web-app-aci" \
    --metric "CpuUsage,MemoryUsage" \
    --interval PT1M \
    --output table

# Get container group details
az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query "{Name:name, Status:instanceView.state, CPU:containers[0].resources.requests.cpu, Memory:containers[0].resources.requests.memoryInGb, RestartCount:containers[0].instanceView.restartCount}" \
    --output table
Subtask 5.3: Set Up Log Analytics (Optional)
Create a Log Analytics workspace for advanced monitoring.

# Create Log Analytics workspace
az monitor log-analytics workspace create \
    --resource-group docker-lab-rg \
    --workspace-name docker-lab-logs \
    --location eastus

# Get workspace ID
WORKSPACE_ID=$(az monitor log-analytics workspace show \
    --resource-group docker-lab-rg \
    --workspace-name docker-lab-logs \
    --query customerId \
    --output tsv)

# Get workspace key
WORKSPACE_KEY=$(az monitor log-analytics workspace get-shared-keys \
    --resource-group docker-lab-rg \
    --workspace-name docker-lab-logs \
    --query primarySharedKey \
    --output tsv)

echo "Workspace ID: $WORKSPACE_ID"
echo "Workspace Key: [HIDDEN]"
Subtask 5.4: Generate Load for Monitoring
Create some traffic to generate monitoring data.

# Create a simple load test script
cat > load_test.sh << 'EOF'
#!/bin/bash
CONTAINER_URL=$1
echo "Starting load test on $CONTAINER_URL"

for i in {1..50}; do
    echo "Request $i"
    curl -s $CONTAINER_URL > /dev/null
    curl -s $CONTAINER_URL/health > /dev/null
    sleep 1
done

echo "Load test completed"
EOF

chmod +x load_test.sh

# Run load test
./load_test.sh http://$CONTAINER_FQDN:3000

# Check container logs after load test
az container logs \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --tail 20
Troubleshooting Common Issues
Issue 1: Container Fails to Start
# Check container events for errors
az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query instanceView.events \
    --output table

# Check container logs for application errors
az container logs \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci
Issue 2: Cannot Access Container from Internet
# Verify container is running
az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query instanceView.state \
    --output tsv

# Check port configuration
az container show \
    --resource-group docker-lab-rg \
    --name docker-web-app-aci \
    --query ipAddress.ports \
    --output table
Issue 3: ACR Authentication Issues
# Re-login to ACR
az acr login --name $ACR_NAME

# Verify ACR credentials
az acr credential show --name $ACR_NAME --output table

# Test ACR connectivity
docker pull $ACR_LOGIN_SERVER/docker-web-app:v1.0
Lab Cleanup
When you're finished with the lab, clean up the Azure resources to avoid charges.

# Delete the entire resource group (this removes all resources)
az group delete --name docker-lab-rg --yes --no-wait

# Verify deletion
az group list --query "[?name=='docker-lab-rg']" --output table

# Clean up local Docker images
docker rmi docker-web-app:v1.0
docker rmi $ACR_LOGIN_SERVER/docker-web-app:v1.0
docker system prune -f
Conclusion
Congratulations! You have successfully completed Lab 103: Docker for Cloud - Running Docker Containers on Microsoft Azure.

What You Accomplished
In this lab, you have:

Set up Azure CLI and authenticated with your Azure account, establishing the foundation for cloud operations
Created Azure Container Registry and learned how to securely store and manage Docker images in the cloud
Built and containerized a web application using Docker best practices and security considerations
Deployed containers to Azure Container Instances, experiencing the simplicity of serverless container hosting
Configured networking to expose your application to the internet with automatic DNS resolution
Implemented monitoring using Azure Monitor to track container performance and resource utilization
Why This Matters
This lab demonstrates the power of containerized applications in cloud environments. By using Azure Container Instances, you've experienced:

Serverless containers: No need to manage underlying infrastructure
Rapid deployment: From code to production in minutes
Automatic scaling: Built-in capabilities for handling varying loads
Cost efficiency: Pay only for the resources you use
Enterprise security: Integration with Azure's security and compliance features
Real-World Applications
The skills you've learned apply to:

Microservices architecture: Deploying individual services as containers
CI/CD pipelines: Automated deployment of containerized applications
Development environments: Quickly spinning up isolated testing environments
Legacy application modernization: Moving traditional apps to cloud-native architectures
Multi-cloud strategies: Understanding container portability across cloud providers
Next Steps
To continue your Docker and Azure journey:

Explore Azure Kubernetes Service (AKS) for orchestrating multiple containers
Learn about Azure Container Apps for event-driven container applications
Investigate Azure DevOps for building complete CI/CD pipelines
Study container security best practices and Azure Security Center integration
Practice with multi-container applications using Docker Compose
This foundational knowledge prepares you for the Docker Certified Associate (DCA) certification and advanced cloud container orchestration scenarios.
