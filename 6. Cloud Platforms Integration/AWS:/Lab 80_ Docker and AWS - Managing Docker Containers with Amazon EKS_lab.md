Lab 80: Docker and AWS - Managing Docker Containers with Amazon EKS
Lab Objectives
By the end of this lab, you will be able to:

Set up and configure Amazon Elastic Kubernetes Service (EKS) cluster
Configure AWS CLI and kubectl for EKS management
Build Docker images and push them to Amazon Elastic Container Registry (ECR)
Deploy containerized applications to EKS using Kubernetes manifests
Expose applications externally using AWS Load Balancer Controller
Scale applications based on traffic demands
Monitor application performance using Amazon CloudWatch
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and containerization concepts
Familiarity with Kubernetes fundamentals (pods, services, deployments)
Basic knowledge of AWS services and cloud computing concepts
Understanding of command-line interface operations
Basic knowledge of YAML configuration files
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes:

AWS CLI pre-installed and configured
Docker Engine ready to use
kubectl command-line tool
eksctl utility for EKS management
Text editors (nano, vim)
All necessary permissions and IAM roles configured
Task 1: Set up Amazon EKS and Configure AWS CLI
Subtask 1.1: Verify AWS CLI Configuration
First, let's verify that AWS CLI is properly configured and check our current region settings.

# Check AWS CLI version
aws --version

# Verify AWS configuration
aws sts get-caller-identity

# Check current region
aws configure get region
Subtask 1.2: Set Environment Variables
Set up environment variables for consistent configuration throughout the lab.

# Set cluster name and region
export CLUSTER_NAME=my-eks-cluster
export AWS_REGION=us-west-2
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Verify environment variables
echo "Cluster Name: $CLUSTER_NAME"
echo "AWS Region: $AWS_REGION"
echo "Account ID: $ACCOUNT_ID"
Subtask 1.3: Create EKS Cluster
Create an EKS cluster using eksctl, which simplifies the cluster creation process.

# Create EKS cluster with managed node group
eksctl create cluster \
  --name $CLUSTER_NAME \
  --region $AWS_REGION \
  --version 1.28 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed
Note: Cluster creation takes approximately 15-20 minutes. The command will automatically configure kubectl context.

Subtask 1.4: Verify Cluster Setup
Once the cluster is created, verify the setup and check cluster status.

# Check cluster status
aws eks describe-cluster --name $CLUSTER_NAME --region $AWS_REGION

# Verify kubectl configuration
kubectl config current-context

# Check cluster nodes
kubectl get nodes

# Check cluster information
kubectl cluster-info
Task 2: Build Docker Image and Push to AWS ECR
Subtask 2.1: Create Sample Application
Create a simple Node.js web application for demonstration purposes.

# Create project directory
mkdir ~/my-app && cd ~/my-app

# Create package.json
cat > package.json << 'EOF'
{
  "name": "my-app",
  "version": "1.0.0",
  "description": "Simple Node.js app for EKS demo",
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
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from EKS!',
    timestamp: new Date().toISOString(),
    hostname: require('os').hostname()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`App running on port ${port}`);
});
EOF
Subtask 2.2: Create Dockerfile
Create a Dockerfile to containerize the application.

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs

# Start application
CMD ["npm", "start"]
EOF
Subtask 2.3: Create ECR Repository
Create an Amazon ECR repository to store your Docker images.

# Create ECR repository
aws ecr create-repository \
  --repository-name my-app \
  --region $AWS_REGION

# Get ECR login token and login to Docker
aws ecr get-login-password --region $AWS_REGION | \
docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
Subtask 2.4: Build and Push Docker Image
Build the Docker image and push it to ECR.

# Set ECR repository URI
export ECR_REPO_URI=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/my-app

# Build Docker image
docker build -t my-app .

# Tag image for ECR
docker tag my-app:latest $ECR_REPO_URI:latest
docker tag my-app:latest $ECR_REPO_URI:v1.0

# Push images to ECR
docker push $ECR_REPO_URI:latest
docker push $ECR_REPO_URI:v1.0

# Verify images in ECR
aws ecr list-images --repository-name my-app --region $AWS_REGION
Task 3: Deploy Docker Image to EKS
Subtask 3.1: Create Kubernetes Namespace
Create a dedicated namespace for your application.

# Create namespace
kubectl create namespace my-app

# Set default namespace context
kubectl config set-context --current --namespace=my-app

# Verify namespace
kubectl get namespaces
Subtask 3.2: Create Deployment Manifest
Create a Kubernetes deployment manifest for your application.

# Create deployment.yaml
cat > deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-deployment
  namespace: my-app
  labels:
    app: my-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app
        image: $ECR_REPO_URI:v1.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
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
Subtask 3.3: Create Service Manifest
Create a Kubernetes service to expose your application within the cluster.

# Create service.yaml
cat > service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
  namespace: my-app
  labels:
    app: my-app
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
EOF
Subtask 3.4: Deploy Application to EKS
Deploy your application using the created manifests.

# Apply deployment
kubectl apply -f deployment.yaml

# Apply service
kubectl apply -f service.yaml

# Check deployment status
kubectl get deployments

# Check pods
kubectl get pods

# Check service
kubectl get services

# View pod logs
kubectl logs -l app=my-app --tail=50
Task 4: Expose Application with AWS Load Balancer
Subtask 4.1: Install AWS Load Balancer Controller
Install the AWS Load Balancer Controller to manage Application Load Balancers.

# Download IAM policy for AWS Load Balancer Controller
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.6.0/docs/install/iam_policy.json

# Create IAM policy
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# Create IAM service account
eksctl create iamserviceaccount \
  --cluster=$CLUSTER_NAME \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
Subtask 4.2: Install Load Balancer Controller using Helm
# Add eks-charts repository
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Install AWS Load Balancer Controller
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

# Verify installation
kubectl get deployment -n kube-system aws-load-balancer-controller
Subtask 4.3: Create Ingress Resource
Create an Ingress resource to expose your application externally.

# Create ingress.yaml
cat > ingress.yaml << 'EOF'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  namespace: my-app
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
EOF

# Apply ingress
kubectl apply -f ingress.yaml

# Check ingress status
kubectl get ingress

# Get load balancer URL (may take a few minutes)
kubectl get ingress my-app-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
Subtask 4.4: Test External Access
Test your application through the load balancer.

# Get the load balancer hostname
export LB_HOSTNAME=$(kubectl get ingress my-app-ingress -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

# Wait for load balancer to be ready (may take 3-5 minutes)
echo "Load Balancer URL: http://$LB_HOSTNAME"

# Test the application
curl http://$LB_HOSTNAME

# Test health endpoint
curl http://$LB_HOSTNAME/health
Task 5: Scale Application and Monitor with CloudWatch
Subtask 5.1: Create Horizontal Pod Autoscaler
Set up automatic scaling based on CPU utilization.

# Create HPA
kubectl autoscale deployment my-app-deployment --cpu-percent=70 --min=2 --max=10

# Check HPA status
kubectl get hpa

# View HPA details
kubectl describe hpa my-app-deployment
Subtask 5.2: Install Metrics Server
Install metrics server for resource monitoring.

# Install metrics server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verify metrics server
kubectl get deployment metrics-server -n kube-system

# Check node metrics (wait a few minutes after installation)
kubectl top nodes

# Check pod metrics
kubectl top pods
Subtask 5.3: Generate Load for Testing Autoscaling
Create load to test the autoscaling functionality.

# Create a load generator pod
kubectl run load-generator --image=busybox --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://my-app-service/; done"

# Monitor HPA in real-time (run in separate terminal)
kubectl get hpa -w

# Check pod scaling
kubectl get pods -w

# After testing, delete load generator
kubectl delete pod load-generator
Subtask 5.4: Enable CloudWatch Container Insights
Enable CloudWatch monitoring for your EKS cluster.

# Install CloudWatch agent
curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/$CLUSTER_NAME/;s/{{region_name}}/$AWS_REGION/" | kubectl apply -f -

# Verify CloudWatch agent installation
kubectl get daemonset cloudwatch-agent -n amazon-cloudwatch

# Check logs
kubectl logs -n amazon-cloudwatch -l name=cloudwatch-agent --tail=50
Subtask 5.5: Create CloudWatch Dashboard
Create a custom dashboard to monitor your application.

# Create dashboard configuration
cat > dashboard.json << EOF
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          [ "ContainerInsights", "pod_cpu_utilization", "ClusterName", "$CLUSTER_NAME", "Namespace", "my-app" ],
          [ ".", "pod_memory_utilization", ".", ".", ".", "." ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "$AWS_REGION",
        "title": "Pod Resource Utilization"
      }
    }
  ]
}
EOF

# Create CloudWatch dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "EKS-MyApp-Dashboard" \
  --dashboard-body file://dashboard.json

echo "Dashboard created: https://$AWS_REGION.console.aws.amazon.com/cloudwatch/home?region=$AWS_REGION#dashboards:name=EKS-MyApp-Dashboard"
Verification and Testing
Test Application Functionality
# Test application endpoints
echo "Testing main endpoint:"
curl http://$LB_HOSTNAME

echo "Testing health endpoint:"
curl http://$LB_HOSTNAME/health

# Check application logs
kubectl logs -l app=my-app --tail=20

# Verify scaling configuration
kubectl get hpa
kubectl get pods
Monitor Resource Usage
# Check resource usage
kubectl top pods
kubectl top nodes

# View deployment status
kubectl get deployments
kubectl describe deployment my-app-deployment

# Check service endpoints
kubectl get endpoints
Troubleshooting Tips
Common Issues and Solutions
Issue: Pods stuck in Pending state

# Check node resources
kubectl describe nodes
kubectl get pods -o wide

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp
Issue: Load balancer not accessible

# Check ingress status
kubectl describe ingress my-app-ingress

# Verify security groups
aws ec2 describe-security-groups --filters "Name=group-name,Values=*$CLUSTER_NAME*"
Issue: HPA not scaling

# Check metrics server
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml

# Verify resource requests are set
kubectl describe deployment my-app-deployment
Cleanup Resources
When you're finished with the lab, clean up resources to avoid charges.

# Delete application resources
kubectl delete -f ingress.yaml
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
kubectl delete namespace my-app

# Delete HPA
kubectl delete hpa my-app-deployment

# Delete ECR repository
aws ecr delete-repository --repository-name my-app --force --region $AWS_REGION

# Delete EKS cluster (this will take several minutes)
eksctl delete cluster --name $CLUSTER_NAME --region $AWS_REGION

# Delete CloudWatch dashboard
aws cloudwatch delete-dashboards --dashboard-names EKS-MyApp-Dashboard
Conclusion
Congratulations! You have successfully completed Lab 80: Docker and AWS - Managing Docker Containers with Amazon EKS.

What You Accomplished
In this lab, you have:

Set up Amazon EKS: Created a managed Kubernetes cluster with worker nodes
Containerized an Application: Built a Docker image and pushed it to Amazon ECR
Deployed to Kubernetes: Used Kubernetes manifests to deploy your application to EKS
Exposed Externally: Configured AWS Load Balancer Controller and Ingress for external access
Implemented Autoscaling: Set up Horizontal Pod Autoscaler for automatic scaling
Enabled Monitoring: Configured CloudWatch Container Insights for comprehensive monitoring
Why This Matters
This lab demonstrates enterprise-grade container orchestration using AWS managed services. The skills you've learned are essential for:

Production Deployments: Managing containerized applications at scale
High Availability: Ensuring applications remain available during traffic spikes
Cost Optimization: Automatically scaling resources based on demand
Operational Excellence: Monitoring and maintaining containerized workloads
Cloud-Native Architecture: Building resilient, scalable applications in the cloud
Next Steps
To further enhance your container orchestration skills, consider:

Exploring advanced Kubernetes features like StatefulSets and DaemonSets
Implementing CI/CD pipelines with AWS CodePipeline and EKS
Learning about service mesh technologies like AWS App Mesh
Studying Kubernetes security best practices and Pod Security Standards
Investigating multi-cluster management with Amazon EKS Anywhere
This hands-on experience with EKS provides a solid foundation for managing containerized applications in production environments and prepares you for advanced cloud-native development practices.
