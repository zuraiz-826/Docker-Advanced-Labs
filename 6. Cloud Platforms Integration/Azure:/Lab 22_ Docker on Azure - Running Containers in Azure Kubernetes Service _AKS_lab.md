Lab 22: Docker on Azure - Running Containers in Azure Kubernetes Service (AKS)
Lab Objectives
By the end of this lab, you will be able to:

Set up and configure an Azure Kubernetes Service (AKS) cluster
Build and containerize applications using Docker
Push Docker images to Azure Container Registry (ACR)
Deploy containerized applications to AKS using kubectl
Expose applications using Azure Load Balancer
Scale applications horizontally in AKS
Monitor application performance using Azure Monitor
Understand the fundamentals of container orchestration with Kubernetes
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and containerization concepts
Familiarity with command-line interface (CLI) operations
Basic knowledge of web applications and HTTP protocols
Understanding of cloud computing fundamentals
No prior Kubernetes experience required - this lab will guide you through the basics
Required Tools
The following tools will be available on your Al Nafi cloud machine:

Azure CLI - Command-line interface for Azure services
Docker - Container platform for building and running applications
kubectl - Kubernetes command-line tool
Git - Version control system
Text editor (nano/vim) - For editing configuration files
Lab Environment Setup
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment. No need to build or configure your own virtual machine.

Your cloud machine includes:

Ubuntu 20.04 LTS operating system
Docker Engine pre-installed
Azure CLI pre-installed
kubectl pre-installed
All necessary development tools
Task 1: Set up Azure Account and Create AKS Cluster
Subtask 1.1: Login to Azure CLI
First, we need to authenticate with Azure using the Azure CLI.

# Login to Azure (this will open a browser window)
az login

# Verify your subscription
az account show
If you don't have an Azure account, you can create a free account at https://azure.microsoft.com/free/

Subtask 1.2: Create Resource Group
A resource group is a container that holds related resources for an Azure solution.

# Set variables for consistent naming
RESOURCE_GROUP="aks-lab-rg"
LOCATION="eastus"
AKS_CLUSTER_NAME="aks-lab-cluster"
ACR_NAME="akslabregistry$(date +%s)"

# Create resource group
az group create --name $RESOURCE_GROUP --location $LOCATION

# Verify resource group creation
az group show --name $RESOURCE_GROUP
Subtask 1.3: Create Azure Container Registry (ACR)
Azure Container Registry will store our Docker images securely.

# Create Azure Container Registry
az acr create \
    --resource-group $RESOURCE_GROUP \
    --name $ACR_NAME \
    --sku Basic \
    --admin-enabled true

# Get ACR login server
ACR_LOGIN_SERVER=$(az acr show --name $ACR_NAME --resource-group $RESOURCE_GROUP --query "loginServer" --output tsv)
echo "ACR Login Server: $ACR_LOGIN_SERVER"
Subtask 1.4: Create AKS Cluster
Now we'll create the Azure Kubernetes Service cluster.

# Create AKS cluster with ACR integration
az aks create \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME \
    --node-count 2 \
    --node-vm-size Standard_B2s \
    --attach-acr $ACR_NAME \
    --generate-ssh-keys \
    --enable-managed-identity

# This command takes 5-10 minutes to complete
echo "AKS cluster creation in progress..."
Subtask 1.5: Configure kubectl
Configure kubectl to connect to your AKS cluster.

# Get AKS credentials
az aks get-credentials --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME

# Verify connection to cluster
kubectl get nodes

# Check cluster information
kubectl cluster-info
Task 2: Build Docker Image and Push to Azure Container Registry
Subtask 2.1: Create Sample Application
We'll create a simple Node.js web application to containerize.

# Create application directory
mkdir ~/aks-demo-app
cd ~/aks-demo-app

# Create package.json file
cat > package.json << EOF
{
  "name": "aks-demo-app",
  "version": "1.0.0",
  "description": "Demo app for AKS lab",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
EOF

# Create server.js file
cat > server.js << EOF
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
    res.send(\`
        <h1>Hello from AKS Demo App!</h1>
        <p>This application is running in Azure Kubernetes Service</p>
        <p>Hostname: \${require('os').hostname()}</p>
        <p>Timestamp: \${new Date().toISOString()}</p>
    \`);
});

app.get('/health', (req, res) => {
    res.status(200).json({ status: 'healthy', timestamp: new Date().toISOString() });
});

app.listen(port, () => {
    console.log(\`Server running on port \${port}\`);
});
EOF
Subtask 2.2: Create Dockerfile
Create a Dockerfile to containerize the application.

# Create Dockerfile
cat > Dockerfile << EOF
# Use official Node.js runtime as base image
FROM node:16-alpine

# Set working directory in container
WORKDIR /app

# Copy package.json and package-lock.json
COPY package*.json ./

# Install dependencies
RUN npm install --production

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

# Create .dockerignore file
cat > .dockerignore << EOF
node_modules
npm-debug.log
.git
.gitignore
README.md
Dockerfile
.dockerignore
EOF
Subtask 2.3: Build and Test Docker Image Locally
# Build Docker image
docker build -t aks-demo-app:v1 .

# List Docker images
docker images

# Test the image locally
docker run -d -p 8080:3000 --name test-app aks-demo-app:v1

# Test the application
curl http://localhost:8080

# Check container logs
docker logs test-app

# Stop and remove test container
docker stop test-app
docker rm test-app
Subtask 2.4: Push Image to Azure Container Registry
# Login to ACR
az acr login --name $ACR_NAME

# Tag image for ACR
docker tag aks-demo-app:v1 $ACR_LOGIN_SERVER/aks-demo-app:v1

# Push image to ACR
docker push $ACR_LOGIN_SERVER/aks-demo-app:v1

# Verify image in ACR
az acr repository list --name $ACR_NAME --output table
az acr repository show-tags --name $ACR_NAME --repository aks-demo-app --output table
Task 3: Deploy Image to AKS using kubectl
Subtask 3.1: Create Kubernetes Deployment Manifest
# Create Kubernetes manifests directory
mkdir ~/k8s-manifests
cd ~/k8s-manifests

# Create deployment.yaml
cat > deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-demo-app
  labels:
    app: aks-demo-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: aks-demo-app
  template:
    metadata:
      labels:
        app: aks-demo-app
    spec:
      containers:
      - name: aks-demo-app
        image: ${ACR_LOGIN_SERVER}/aks-demo-app:v1
        ports:
        - containerPort: 3000
        env:
        - name: PORT
          value: "3000"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
EOF
Subtask 3.2: Deploy Application to AKS
# Apply deployment
kubectl apply -f deployment.yaml

# Check deployment status
kubectl get deployments

# Check pods
kubectl get pods

# Get detailed pod information
kubectl get pods -o wide

# Check pod logs
kubectl logs -l app=aks-demo-app

# Describe deployment for troubleshooting
kubectl describe deployment aks-demo-app
Task 4: Expose Application via Azure Load Balancer
Subtask 4.1: Create Service Manifest
# Create service.yaml
cat > service.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: aks-demo-app-service
  labels:
    app: aks-demo-app
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
    protocol: TCP
  selector:
    app: aks-demo-app
EOF
Subtask 4.2: Deploy Service
# Apply service
kubectl apply -f service.yaml

# Check service status
kubectl get services

# Wait for external IP (this may take a few minutes)
echo "Waiting for external IP assignment..."
kubectl get service aks-demo-app-service --watch
Note: Press Ctrl+C when you see the EXTERNAL-IP column shows an actual IP address instead of <pending>.

Subtask 4.3: Test External Access
# Get external IP
EXTERNAL_IP=$(kubectl get service aks-demo-app-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "External IP: $EXTERNAL_IP"

# Test application access
curl http://$EXTERNAL_IP

# Test health endpoint
curl http://$EXTERNAL_IP/health

# Open in browser (if available)
echo "Application URL: http://$EXTERNAL_IP"
Task 5: Scale Application and Monitor using Azure Monitor
Subtask 5.1: Scale Application Horizontally
# Scale deployment to 5 replicas
kubectl scale deployment aks-demo-app --replicas=5

# Check scaling progress
kubectl get pods -w

# Verify all pods are running
kubectl get pods -l app=aks-demo-app

# Check deployment status
kubectl get deployment aks-demo-app
Subtask 5.2: Test Load Distribution
# Test multiple requests to see different hostnames
for i in {1..10}; do
    echo "Request $i:"
    curl -s http://$EXTERNAL_IP | grep "Hostname"
    echo ""
done
Subtask 5.3: Configure Horizontal Pod Autoscaler
# Create HPA (Horizontal Pod Autoscaler)
kubectl autoscale deployment aks-demo-app --cpu-percent=70 --min=3 --max=10

# Check HPA status
kubectl get hpa

# Describe HPA for details
kubectl describe hpa aks-demo-app
Subtask 5.4: Enable Azure Monitor for Containers
# Enable monitoring add-on for AKS
az aks enable-addons \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME \
    --addons monitoring

# Verify monitoring is enabled
az aks show --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query "addonProfiles.omsagent.enabled"
Subtask 5.5: View Monitoring Data
# Get cluster resource ID for Azure portal
CLUSTER_RESOURCE_ID=$(az aks show --resource-group $RESOURCE_GROUP --name $AKS_CLUSTER_NAME --query "id" --output tsv)
echo "Cluster Resource ID: $CLUSTER_RESOURCE_ID"

# Check cluster metrics using kubectl
kubectl top nodes
kubectl top pods

# View cluster events
kubectl get events --sort-by=.metadata.creationTimestamp
Advanced Operations and Troubleshooting
Rolling Updates
# Update application image (simulate new version)
kubectl set image deployment/aks-demo-app aks-demo-app=$ACR_LOGIN_SERVER/aks-demo-app:v1

# Check rollout status
kubectl rollout status deployment/aks-demo-app

# View rollout history
kubectl rollout history deployment/aks-demo-app
Troubleshooting Commands
# Check cluster health
kubectl get componentstatuses

# View all resources
kubectl get all

# Check node status
kubectl describe nodes

# Check pod logs for specific pod
kubectl logs <pod-name>

# Execute commands in pod
kubectl exec -it <pod-name> -- /bin/sh

# Port forward for local testing
kubectl port-forward service/aks-demo-app-service 8080:80
Cleanup Resources
# Delete Kubernetes resources
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
kubectl delete hpa aks-demo-app

# Delete Azure resources (optional - only if you want to clean up completely)
# az group delete --name $RESOURCE_GROUP --yes --no-wait
Key Concepts Summary
Docker Concepts Covered
Containerization: Packaging applications with their dependencies
Docker Images: Read-only templates used to create containers
Dockerfile: Text file containing instructions to build Docker images
Container Registry: Service for storing and distributing container images
Kubernetes Concepts Covered
Pods: Smallest deployable units in Kubernetes
Deployments: Manage replica sets and provide declarative updates
Services: Expose applications running on pods
Load Balancer: Distributes traffic across multiple pods
Horizontal Pod Autoscaler: Automatically scales pods based on metrics
Azure Concepts Covered
Azure Kubernetes Service (AKS): Managed Kubernetes service
Azure Container Registry (ACR): Private container registry
Azure Load Balancer: Layer 4 load balancing service
Azure Monitor: Comprehensive monitoring solution
Common Issues and Solutions
Issue 1: ACR Authentication Problems
Problem: Cannot push images to ACR Solution:

az acr login --name $ACR_NAME
# Or use admin credentials
az acr credential show --name $ACR_NAME
Issue 2: Pods Stuck in Pending State
Problem: Pods not starting Solution:

kubectl describe pod <pod-name>
kubectl get events
# Check resource constraints and node capacity
Issue 3: External IP Not Assigned
Problem: LoadBalancer service shows <pending> for external IP Solution:

# Wait longer (can take 5-10 minutes)
# Check Azure portal for load balancer creation
# Verify AKS cluster has proper permissions
Conclusion
Congratulations! You have successfully completed Lab 22: Docker on Azure - Running Containers in Azure Kubernetes Service (AKS).

What You Accomplished
Set up Azure Infrastructure: Created an AKS cluster and Azure Container Registry
Containerized an Application: Built a Docker image for a Node.js web application
Deployed to Kubernetes: Used kubectl to deploy your application to AKS
Exposed Your Application: Made your application accessible via Azure Load Balancer
Implemented Scaling: Configured horizontal pod autoscaling for high availability
Enabled Monitoring: Set up Azure Monitor for comprehensive observability
Why This Matters
This lab demonstrates real-world container orchestration skills that are essential in modern DevOps and cloud engineering roles. You've learned how to:

Modernize Applications: Transform traditional applications into cloud-native, scalable services
Implement High Availability: Use Kubernetes features to ensure your applications remain available
Optimize Resource Usage: Leverage autoscaling to handle varying workloads efficiently
Monitor Production Systems: Use Azure Monitor to maintain visibility into application performance
Next Steps
To continue your container orchestration journey:

Explore Advanced Kubernetes Features: Learn about ConfigMaps, Secrets, and Persistent Volumes
Implement CI/CD Pipelines: Automate your deployment process using Azure DevOps or GitHub Actions
Study Security Best Practices: Learn about pod security policies and network policies
Practice with Different Workloads: Deploy databases, microservices, and batch jobs
Certification Relevance
This lab directly supports your preparation for the Docker Certified Associate (DCA) certification by covering:

Container orchestration with Kubernetes
Image management and registry operations
Application deployment and scaling
Monitoring and troubleshooting containerized applications
The skills you've developed are also valuable for other cloud certifications including Azure certifications (AZ-104, AZ-204) and Kubernetes certifications (CKA, CKAD).

Keep practicing these concepts and exploring the rich ecosystem of cloud-native technologies!
