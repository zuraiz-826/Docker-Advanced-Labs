Lab 94: Docker and Kubernetes - Configuring Auto-Scaling for Kubernetes
Objectives
By the end of this lab, students will be able to:

Set up a Kubernetes cluster using Minikube
Deploy a Dockerized application on Kubernetes
Configure Horizontal Pod Autoscaling (HPA) based on CPU usage
Simulate load to test auto-scaling functionality
Monitor and adjust scaling parameters for optimal performance
Understand the principles of container orchestration and auto-scaling
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and containerization concepts
Familiarity with command-line interface operations
Basic knowledge of YAML file structure
Understanding of CPU and memory resource concepts
Access to a Linux-based system with internet connectivity
Note: Al Nafi provides ready-to-use Linux-based cloud machines. Simply click Start Lab to begin - no need to build your own VM or install additional software.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with:

Docker Engine
kubectl (Kubernetes command-line tool)
Minikube
curl and other essential utilities
Task 1: Set up a Kubernetes Cluster with Minikube
Subtask 1.1: Start Minikube Cluster
First, let's start our local Kubernetes cluster using Minikube.

# Start Minikube with specific resource allocation
minikube start --cpus=2 --memory=4096 --driver=docker

# Verify the cluster is running
minikube status
Subtask 1.2: Enable Metrics Server
The Metrics Server is essential for auto-scaling as it provides resource usage data.

# Enable metrics-server addon
minikube addons enable metrics-server

# Verify metrics-server is running
kubectl get pods -n kube-system | grep metrics-server
Subtask 1.3: Verify Cluster Configuration
# Check cluster information
kubectl cluster-info

# View all nodes in the cluster
kubectl get nodes

# Check if metrics are available (may take a few minutes)
kubectl top nodes
Task 2: Deploy a Dockerized Application on Kubernetes
Subtask 2.1: Create Application Deployment
We'll deploy a simple PHP application that can generate CPU load for testing auto-scaling.

Create a deployment file:

# Create deployment configuration
cat > php-apache-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  labels:
    app: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
            memory: 128Mi
          requests:
            cpu: 200m
            memory: 64Mi
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    app: php-apache
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: php-apache
  type: ClusterIP
EOF
Subtask 2.2: Deploy the Application
# Apply the deployment
kubectl apply -f php-apache-deployment.yaml

# Verify deployment status
kubectl get deployments

# Check if pods are running
kubectl get pods

# Verify service is created
kubectl get services
Subtask 2.3: Test Application Connectivity
# Get service details
kubectl describe service php-apache

# Test the application (from within cluster)
kubectl run -i --tty load-generator --rm --image=busybox --restart=Never -- /bin/sh
Inside the busybox container, test connectivity:

# Test connection to the service
wget -q -O- http://php-apache.default.svc.cluster.local

# Exit the container
exit
Task 3: Enable Horizontal Pod Autoscaling Based on CPU Usage
Subtask 3.1: Create Horizontal Pod Autoscaler
# Create HPA with CPU threshold of 50%
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

# Verify HPA creation
kubectl get hpa
Subtask 3.2: Create HPA Using YAML Configuration
For more control, create an HPA configuration file:

cat > php-apache-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
EOF
Subtask 3.3: Apply Advanced HPA Configuration
# Delete the simple HPA first
kubectl delete hpa php-apache

# Apply the advanced HPA configuration
kubectl apply -f php-apache-hpa.yaml

# Verify the new HPA
kubectl get hpa php-apache-hpa
kubectl describe hpa php-apache-hpa
Task 4: Test Auto-Scaling by Simulating Load
Subtask 4.1: Monitor Current State
Open a new terminal window to monitor the scaling process:

# Watch HPA status in real-time
watch -n 2 kubectl get hpa php-apache-hpa

# In another terminal, watch pods
watch -n 2 kubectl get pods
Subtask 4.2: Generate Load
Create a load generator to increase CPU usage:

# Create load generator deployment
cat > load-generator.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: load-generator
spec:
  replicas: 1
  selector:
    matchLabels:
      app: load-generator
  template:
    metadata:
      labels:
        app: load-generator
    spec:
      containers:
      - name: load-generator
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          while true; do
            wget -q -O- http://php-apache.default.svc.cluster.local
            sleep 0.01
          done
EOF
# Deploy load generator
kubectl apply -f load-generator.yaml

# Verify load generator is running
kubectl get pods | grep load-generator
Subtask 4.3: Increase Load Intensity
# Scale load generator to create more traffic
kubectl scale deployment load-generator --replicas=5

# Monitor CPU usage
kubectl top pods
Subtask 4.4: Observe Auto-Scaling Behavior
Monitor the scaling process for 5-10 minutes:

# Check HPA status
kubectl get hpa php-apache-hpa

# View detailed HPA information
kubectl describe hpa php-apache-hpa

# Check current number of pods
kubectl get pods | grep php-apache
Task 5: Monitor and Adjust Scaling Settings
Subtask 5.1: Analyze Scaling Metrics
# Get detailed metrics
kubectl top pods

# Check HPA events
kubectl describe hpa php-apache-hpa | grep Events -A 10

# View deployment scaling events
kubectl describe deployment php-apache | grep Events -A 10
Subtask 5.2: Test Scale-Down Behavior
# Stop load generation
kubectl scale deployment load-generator --replicas=0

# Monitor scale-down process
watch -n 5 kubectl get pods
Subtask 5.3: Adjust Scaling Parameters
Create an optimized HPA configuration:

cat > optimized-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache-optimized
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 200
        periodSeconds: 30
EOF
Subtask 5.4: Apply Optimized Configuration
# Remove existing HPA
kubectl delete hpa php-apache-hpa

# Apply optimized HPA
kubectl apply -f optimized-hpa.yaml

# Verify new configuration
kubectl get hpa php-apache-optimized
kubectl describe hpa php-apache-optimized
Subtask 5.5: Performance Testing and Validation
# Create comprehensive load test
cat > comprehensive-load-test.yaml << 'EOF'
apiVersion: batch/v1
kind: Job
metadata:
  name: load-test
spec:
  parallelism: 3
  completions: 3
  template:
    spec:
      containers:
      - name: load-test
        image: busybox
        command:
        - /bin/sh
        - -c
        - |
          echo "Starting load test..."
          for i in $(seq 1 1000); do
            wget -q -O- http://php-apache.default.svc.cluster.local
            if [ $((i % 100)) -eq 0 ]; then
              echo "Completed $i requests"
            fi
          done
          echo "Load test completed"
      restartPolicy: Never
EOF
# Run comprehensive load test
kubectl apply -f comprehensive-load-test.yaml

# Monitor test progress
kubectl get jobs
kubectl logs job/load-test
Troubleshooting Common Issues
Issue 1: Metrics Server Not Working
# Check metrics-server status
kubectl get pods -n kube-system | grep metrics-server

# Restart metrics-server if needed
kubectl rollout restart deployment/metrics-server -n kube-system
Issue 2: HPA Shows Unknown CPU Usage
# Wait for metrics to be available (can take 2-3 minutes)
kubectl top pods

# Check if resource requests are defined
kubectl describe deployment php-apache | grep -A 10 "Requests"
Issue 3: Pods Not Scaling
# Check HPA conditions
kubectl describe hpa

# Verify resource limits and requests
kubectl describe pod <pod-name>
Lab Validation
Verification Checklist
Cluster Setup: Minikube cluster is running with metrics-server enabled
Application Deployment: PHP application is deployed and accessible
HPA Configuration: Horizontal Pod Autoscaler is properly configured
Load Testing: Successfully generated load and observed scaling
Monitoring: Able to monitor and adjust scaling parameters
Final Validation Commands
# Check all components
echo "=== Cluster Status ==="
kubectl get nodes

echo "=== Deployments ==="
kubectl get deployments

echo "=== HPA Status ==="
kubectl get hpa

echo "=== Current Pods ==="
kubectl get pods

echo "=== Resource Usage ==="
kubectl top pods
Cleanup
# Clean up resources
kubectl delete deployment php-apache
kubectl delete deployment load-generator
kubectl delete service php-apache
kubectl delete hpa php-apache-optimized
kubectl delete job load-test

# Stop Minikube
minikube stop
Conclusion
In this lab, you have successfully:

Established a Kubernetes Environment: Set up a local Kubernetes cluster using Minikube with proper monitoring capabilities
Deployed Containerized Applications: Successfully deployed a Dockerized application with appropriate resource constraints
Implemented Auto-Scaling: Configured Horizontal Pod Autoscaling based on CPU utilization metrics
Conducted Load Testing: Simulated realistic load conditions to trigger and validate auto-scaling behavior
Optimized Scaling Parameters: Fine-tuned scaling policies for better performance and resource utilization
Why This Matters: Auto-scaling is crucial for modern cloud-native applications as it ensures optimal resource utilization, maintains application performance under varying loads, and reduces operational costs. Understanding these concepts prepares you for real-world container orchestration scenarios and is essential for the Docker Certified Associate (DCA) certification.

The skills learned in this lab directly apply to production environments where applications must handle dynamic workloads efficiently while maintaining cost-effectiveness and performance standards.
