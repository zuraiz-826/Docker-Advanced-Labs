Lab 70: Docker for Deployment - Deploying Docker Containers with Kubernetes
Lab Objectives
By the end of this lab, you will be able to:

• Set up and configure a local Kubernetes cluster using Minikube • Build and containerize applications using Docker • Push Docker images to Docker Hub container registry • Create and manage Kubernetes deployments for Docker containers • Expose applications as Kubernetes services • Scale deployments and manage container replicas • Understand the relationship between Docker containers and Kubernetes orchestration

Prerequisites
Before starting this lab, you should have:

• Basic understanding of containerization concepts • Familiarity with command-line interface operations • Basic knowledge of Docker commands • Understanding of YAML file structure • A Docker Hub account (free registration at hub.docker.com)

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment - no need to build your own VM or install software locally.

Your cloud machine includes: • Docker Engine (latest stable version) • Minikube for local Kubernetes cluster • kubectl command-line tool • Git for version control • Text editors (nano, vim)

Task 1: Create a Kubernetes Cluster using Minikube
Subtask 1.1: Verify Installation and Start Minikube
First, let's verify that all required tools are installed and start our local Kubernetes cluster.

Step 1: Check Docker installation

docker --version
Step 2: Check Minikube installation

minikube version
Step 3: Check kubectl installation

kubectl version --client
Step 4: Start Minikube cluster

minikube start --driver=docker
This command will: • Download the Kubernetes cluster image • Start a single-node Kubernetes cluster • Configure kubectl to communicate with the cluster

Step 5: Verify cluster status

minikube status
Step 6: Check cluster information

kubectl cluster-info
Subtask 1.2: Configure Docker Environment
Step 1: Configure your shell to use Minikube's Docker daemon

eval $(minikube docker-env)
Step 2: Verify Docker configuration

docker ps
You should see Kubernetes system containers running.

Task 2: Build a Docker Image and Push to Container Registry
Subtask 2.1: Create a Sample Application
Step 1: Create a project directory

mkdir ~/kubernetes-docker-lab
cd ~/kubernetes-docker-lab
Step 2: Create a simple Node.js application

cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Kubernetes!',
    hostname: require('os').hostname(),
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`App running on port ${port}`);
});
EOF
Step 3: Create package.json file

cat > package.json << 'EOF'
{
  "name": "kubernetes-docker-app",
  "version": "1.0.0",
  "description": "Sample app for Kubernetes deployment",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF
Subtask 2.2: Create Dockerfile
Step 1: Create Dockerfile

cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
EOF
Subtask 2.3: Build and Test Docker Image Locally
Step 1: Build the Docker image

docker build -t kubernetes-docker-app:v1.0 .
Step 2: Test the image locally

docker run -d -p 8080:3000 --name test-app kubernetes-docker-app:v1.0
Step 3: Test the application

curl http://localhost:8080
Step 4: Stop and remove test container

docker stop test-app
docker rm test-app
Subtask 2.4: Push Image to Docker Hub
Step 1: Log in to Docker Hub

docker login
Enter your Docker Hub username and password when prompted.

Step 2: Tag the image with your Docker Hub username

docker tag kubernetes-docker-app:v1.0 YOUR_DOCKERHUB_USERNAME/kubernetes-docker-app:v1.0
Replace YOUR_DOCKERHUB_USERNAME with your actual Docker Hub username.

Step 3: Push the image to Docker Hub

docker push YOUR_DOCKERHUB_USERNAME/kubernetes-docker-app:v1.0
Task 3: Create a Kubernetes Deployment
Subtask 3.1: Create Deployment YAML File
Step 1: Create deployment configuration

cat > deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubernetes-docker-app
  labels:
    app: kubernetes-docker-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubernetes-docker-app
  template:
    metadata:
      labels:
        app: kubernetes-docker-app
    spec:
      containers:
      - name: kubernetes-docker-app
        image: YOUR_DOCKERHUB_USERNAME/kubernetes-docker-app:v1.0
        ports:
        - containerPort: 3000
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
Step 2: Update the image name in deployment.yaml

sed -i 's/YOUR_DOCKERHUB_USERNAME/your-actual-username/g' deployment.yaml
Replace your-actual-username with your Docker Hub username.

Subtask 3.2: Deploy to Kubernetes
Step 1: Apply the deployment

kubectl apply -f deployment.yaml
Step 2: Check deployment status

kubectl get deployments
Step 3: Check pods status

kubectl get pods
Step 4: View detailed pod information

kubectl describe pods
Task 4: Expose the Deployment as a Service
Subtask 4.1: Create Service YAML File
Step 1: Create service configuration

cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-docker-app-service
  labels:
    app: kubernetes-docker-app
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 3000
    nodePort: 30080
  selector:
    app: kubernetes-docker-app
EOF
Subtask 4.2: Deploy the Service
Step 1: Apply the service configuration

kubectl apply -f service.yaml
Step 2: Check service status

kubectl get services
Step 3: Get service details

kubectl describe service kubernetes-docker-app-service
Subtask 4.3: Test the Service
Step 1: Get Minikube IP address

minikube ip
Step 2: Test the service using curl

curl http://$(minikube ip):30080
Step 3: Open service in browser (alternative method)

minikube service kubernetes-docker-app-service --url
Task 5: Scale the Deployment and Manage Container Replicas
Subtask 5.1: Scale Up the Deployment
Step 1: Scale deployment to 5 replicas

kubectl scale deployment kubernetes-docker-app --replicas=5
Step 2: Watch the scaling process

kubectl get pods -w
Press Ctrl+C to stop watching after pods are running.

Step 3: Verify scaling

kubectl get deployment kubernetes-docker-app
Subtask 5.2: Test Load Distribution
Step 1: Make multiple requests to see different hostnames

for i in {1..10}; do
  curl -s http://$(minikube ip):30080 | grep hostname
  sleep 1
done
Subtask 5.3: Scale Down the Deployment
Step 1: Scale down to 2 replicas

kubectl scale deployment kubernetes-docker-app --replicas=2
Step 2: Verify scaling down

kubectl get pods
Subtask 5.4: Update Deployment with Rolling Update
Step 1: Create a new version of the application

cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Kubernetes - Version 2.0!',
    hostname: require('os').hostname(),
    timestamp: new Date().toISOString(),
    version: '2.0'
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy', version: '2.0' });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`App v2.0 running on port ${port}`);
});
EOF
Step 2: Build and push new version

docker build -t YOUR_DOCKERHUB_USERNAME/kubernetes-docker-app:v2.0 .
docker push YOUR_DOCKERHUB_USERNAME/kubernetes-docker-app:v2.0
Step 3: Update deployment with new image

kubectl set image deployment/kubernetes-docker-app kubernetes-docker-app=YOUR_DOCKERHUB_USERNAME/kubernetes-docker-app:v2.0
Step 4: Watch rolling update

kubectl rollout status deployment/kubernetes-docker-app
Step 5: Test updated application

curl http://$(minikube ip):30080
Subtask 5.5: Monitor and Manage Resources
Step 1: View deployment history

kubectl rollout history deployment/kubernetes-docker-app
Step 2: Check resource usage

kubectl top pods
Step 3: View logs from pods

kubectl logs -l app=kubernetes-docker-app --tail=50
Step 4: Get detailed cluster information

kubectl get all
Troubleshooting Common Issues
Issue 1: Minikube Won't Start
Solution:

minikube delete
minikube start --driver=docker --force
Issue 2: Image Pull Errors
Solution: Verify image name and ensure it's pushed to Docker Hub

docker pull YOUR_DOCKERHUB_USERNAME/kubernetes-docker-app:v1.0
Issue 3: Service Not Accessible
Solution: Check service and pod status

kubectl get svc,pods
kubectl describe service kubernetes-docker-app-service
Issue 4: Pods Stuck in Pending State
Solution: Check cluster resources

kubectl describe pods
minikube status
Lab Cleanup
Step 1: Delete Kubernetes resources

kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
Step 2: Stop Minikube

minikube stop
Step 3: Clean up Docker images (optional)

docker rmi kubernetes-docker-app:v1.0
docker rmi kubernetes-docker-app:v2.0
Conclusion
Congratulations! You have successfully completed Lab 70: Docker for Deployment with Kubernetes. In this comprehensive lab, you have:

• Set up a local Kubernetes environment using Minikube, providing you with hands-on experience in cluster management • Built and containerized a Node.js application using Docker, demonstrating the containerization process from source code to deployable image • Published your container image to Docker Hub, learning about container registries and image distribution • Created Kubernetes deployments with proper resource management, health checks, and configuration best practices • Exposed applications as services using NodePort, understanding how Kubernetes networking enables external access • Implemented scaling strategies by managing replica counts and performing rolling updates without downtime

Why This Matters:

This lab demonstrates the powerful combination of Docker and Kubernetes for modern application deployment. Docker provides consistent containerization across environments, while Kubernetes offers robust orchestration, scaling, and management capabilities. This knowledge is essential for:

• DevOps Engineers implementing CI/CD pipelines • Software Developers deploying applications in cloud-native environments
• System Administrators managing containerized infrastructure • Anyone pursuing Docker Certified Associate (DCA) certification

The skills you've learned here form the foundation of modern container orchestration and are directly applicable to production environments using managed Kubernetes services like Google GKE, Amazon EKS, or Azure AKS.

You now have practical experience with the complete container deployment lifecycle, from development through production-ready orchestration, positioning you well for advanced Kubernetes topics and real-world container management scenarios.
