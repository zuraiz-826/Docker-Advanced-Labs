Lab 96: Docker in Production - Scaling with Kubernetes
Objectives
By the end of this lab, you will be able to:

Set up a Kubernetes cluster using Minikube for local development
Deploy a Dockerized application to a Kubernetes cluster
Configure Horizontal Pod Autoscaling (HPA) based on CPU and memory usage
Simulate application load to trigger autoscaling events
Monitor scaling behavior and resource utilization in Kubernetes
Understand the fundamentals of container orchestration in production environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with command-line interface operations
Basic knowledge of YAML file structure
Understanding of web applications and HTTP requests
No prior Kubernetes experience required - we'll cover the basics
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment. No need to build your own VM or install software locally.

Your cloud machine includes:

Docker Engine
Minikube
kubectl (Kubernetes command-line tool)
curl and other networking utilities
Text editors (nano, vim)
Lab Environment Setup
Task 1: Initialize Kubernetes Cluster with Minikube
Subtask 1.1: Start Minikube Cluster
First, let's start our local Kubernetes cluster using Minikube.

# Start Minikube with specific resource allocation
minikube start --cpus=2 --memory=4096 --driver=docker

# Verify cluster is running
minikube status
Expected output should show:

minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
Subtask 1.2: Enable Required Add-ons
Enable the metrics server which is essential for autoscaling functionality.

# Enable metrics server for resource monitoring
minikube addons enable metrics-server

# Verify metrics server is running
kubectl get pods -n kube-system | grep metrics-server
Subtask 1.3: Configure kubectl Context
Ensure kubectl is configured to communicate with your Minikube cluster.

# Check current context
kubectl config current-context

# Get cluster information
kubectl cluster-info

# List all nodes in the cluster
kubectl get nodes
Task 2: Deploy a Dockerized Application to Kubernetes
Subtask 2.1: Create Application Deployment
We'll deploy a simple web application that can consume CPU resources for testing autoscaling.

Create a deployment file:

# Create deployment directory
mkdir -p ~/k8s-lab
cd ~/k8s-lab

# Create deployment YAML file
cat > web-app-deployment.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  labels:
    app: web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web-app
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        env:
        - name: PORT
          value: "80"
EOF
Subtask 2.2: Deploy the Application
Apply the deployment to your Kubernetes cluster:

# Deploy the application
kubectl apply -f web-app-deployment.yaml

# Verify deployment status
kubectl get deployments

# Check pod status
kubectl get pods -l app=web-app

# Get detailed pod information
kubectl describe pods -l app=web-app
Subtask 2.3: Create Service for Application Access
Create a service to expose your application:

# Create service YAML file
cat > web-app-service.yaml << 'EOF'
apiVersion: v1
kind: Service
metadata:
  name: web-app-service
spec:
  selector:
    app: web-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
EOF

# Apply the service
kubectl apply -f web-app-service.yaml

# Verify service creation
kubectl get services
Task 3: Configure Horizontal Pod Autoscaling (HPA)
Subtask 3.1: Create HPA Configuration
Configure autoscaling based on CPU utilization:

# Create HPA YAML file
cat > web-app-hpa.yaml << 'EOF'
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
EOF
Subtask 3.2: Apply HPA Configuration
# Apply HPA configuration
kubectl apply -f web-app-hpa.yaml

# Verify HPA creation
kubectl get hpa

# Get detailed HPA information
kubectl describe hpa web-app-hpa
Subtask 3.3: Monitor Initial HPA Status
# Watch HPA status (press Ctrl+C to stop)
kubectl get hpa web-app-hpa --watch

# In a separate terminal, check current resource usage
kubectl top pods -l app=web-app
Task 4: Test Autoscaling by Simulating Load
Subtask 4.1: Create Load Testing Pod
Deploy a pod that will generate load on our application:

# Create load generator YAML
cat > load-generator.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: load-generator
spec:
  containers:
  - name: load-generator
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do wget -q -O- http://web-app-service/; done"]
    resources:
      requests:
        cpu: 10m
        memory: 16Mi
      limits:
        cpu: 50m
        memory: 32Mi
EOF

# Deploy load generator
kubectl apply -f load-generator.yaml

# Verify load generator is running
kubectl get pods load-generator
Subtask 4.2: Create CPU Stress Test
For more intensive load testing, create a CPU stress container:

# Create CPU stress deployment
cat > cpu-stress.yaml << 'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-stress
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-stress
  template:
    metadata:
      labels:
        app: cpu-stress
    spec:
      containers:
      - name: cpu-stress
        image: progrium/stress
        command: ["stress"]
        args: ["--cpu", "2", "--timeout", "300s"]
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
EOF

# Deploy CPU stress test
kubectl apply -f cpu-stress.yaml
Subtask 4.3: Monitor Load Impact
# Monitor HPA status during load test
kubectl get hpa web-app-hpa --watch

# In another terminal, monitor pod scaling
watch kubectl get pods -l app=web-app

# Check resource usage
kubectl top pods -l app=web-app
Subtask 4.4: Generate HTTP Load
Create additional HTTP load to trigger scaling:

# Create multiple load generators
for i in {1..3}; do
cat > load-generator-$i.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: load-generator-$i
spec:
  containers:
  - name: load-generator
    image: alpine/curl
    command: ["/bin/sh"]
    args: ["-c", "while true; do curl -s http://web-app-service/ > /dev/null; sleep 0.1; done"]
EOF

kubectl apply -f load-generator-$i.yaml
done

# Verify all load generators are running
kubectl get pods | grep load-generator
Task 5: Monitor Scaling and Resource Usage
Subtask 5.1: Real-time Monitoring Setup
Set up comprehensive monitoring of the scaling behavior:

# Create monitoring script
cat > monitor-scaling.sh << 'EOF'
#!/bin/bash

echo "=== Kubernetes Autoscaling Monitor ==="
echo "Press Ctrl+C to stop monitoring"
echo

while true; do
    clear
    echo "=== Current Time: $(date) ==="
    echo
    
    echo "=== HPA Status ==="
    kubectl get hpa web-app-hpa
    echo
    
    echo "=== Pod Count and Status ==="
    kubectl get pods -l app=web-app -o wide
    echo
    
    echo "=== Resource Usage ==="
    kubectl top pods -l app=web-app 2>/dev/null || echo "Metrics not ready yet..."
    echo
    
    echo "=== Deployment Status ==="
    kubectl get deployment web-app
    echo
    
    sleep 10
done
EOF

# Make script executable
chmod +x monitor-scaling.sh

# Run monitoring script
./monitor-scaling.sh
Subtask 5.2: Analyze Scaling Events
# View HPA events
kubectl describe hpa web-app-hpa

# Check deployment events
kubectl describe deployment web-app

# View pod events
kubectl get events --sort-by=.metadata.creationTimestamp
Subtask 5.3: Test Scale-Down Behavior
# Stop load generators to test scale-down
kubectl delete pod load-generator
kubectl delete pods -l app=cpu-stress
for i in {1..3}; do kubectl delete pod load-generator-$i; done

# Monitor scale-down process
kubectl get hpa web-app-hpa --watch
Subtask 5.4: Advanced Monitoring Commands
# Get detailed resource metrics
kubectl top nodes

# View cluster resource usage
kubectl describe nodes

# Check metrics server logs if needed
kubectl logs -n kube-system -l k8s-app=metrics-server

# View HPA controller logs
kubectl logs -n kube-system -l app=horizontal-pod-autoscaler
Verification and Testing
Verify Autoscaling Functionality
# Create verification script
cat > verify-autoscaling.sh << 'EOF'
#!/bin/bash

echo "=== Autoscaling Verification ==="

# Check if HPA is configured
echo "1. Checking HPA configuration..."
if kubectl get hpa web-app-hpa &>/dev/null; then
    echo "✓ HPA is configured"
    kubectl get hpa web-app-hpa
else
    echo "✗ HPA not found"
    exit 1
fi

echo

# Check current pod count
echo "2. Current pod status..."
CURRENT_PODS=$(kubectl get pods -l app=web-app --no-headers | wc -l)
echo "Current pod count: $CURRENT_PODS"

echo

# Check if metrics are available
echo "3. Checking metrics availability..."
if kubectl top pods -l app=web-app &>/dev/null; then
    echo "✓ Metrics are available"
    kubectl top pods -l app=web-app
else
    echo "⚠ Metrics not ready yet (this is normal initially)"
fi

echo

# Check service connectivity
echo "4. Testing service connectivity..."
if kubectl get service web-app-service &>/dev/null; then
    echo "✓ Service is accessible"
    kubectl get service web-app-service
else
    echo "✗ Service not found"
fi

echo
echo "=== Verification Complete ==="
EOF

chmod +x verify-autoscaling.sh
./verify-autoscaling.sh
Troubleshooting Common Issues
Issue 1: Metrics Server Not Working
# Check metrics server status
kubectl get pods -n kube-system | grep metrics-server

# If metrics server is not running, restart it
minikube addons disable metrics-server
minikube addons enable metrics-server

# Wait for metrics server to be ready
kubectl wait --for=condition=ready pod -l k8s-app=metrics-server -n kube-system --timeout=300s
Issue 2: HPA Shows Unknown Metrics
# Check if resource requests are set in deployment
kubectl describe deployment web-app | grep -A 10 "Limits\|Requests"

# Verify metrics server is collecting data
kubectl top nodes
kubectl top pods -l app=web-app
Issue 3: Pods Not Scaling
# Check HPA events for scaling decisions
kubectl describe hpa web-app-hpa

# Verify resource utilization is above threshold
kubectl top pods -l app=web-app

# Check if there are resource constraints
kubectl describe nodes
Cleanup
When you're finished with the lab, clean up resources:

# Delete all created resources
kubectl delete -f web-app-hpa.yaml
kubectl delete -f web-app-service.yaml
kubectl delete -f web-app-deployment.yaml
kubectl delete -f cpu-stress.yaml 2>/dev/null || true

# Delete any remaining load generators
kubectl delete pods -l app=load-generator 2>/dev/null || true

# Stop Minikube (optional)
minikube stop

# Remove lab files
cd ~
rm -rf k8s-lab
Conclusion
Congratulations! You have successfully completed Lab 96: Docker in Production - Scaling with Kubernetes. In this lab, you accomplished several important tasks:

What You Learned:

Set up a local Kubernetes cluster using Minikube with proper resource allocation
Deployed a Dockerized application with resource requests and limits
Configured Horizontal Pod Autoscaling (HPA) based on CPU and memory metrics
Simulated realistic application load to trigger autoscaling events
Monitored scaling behavior and resource utilization in real-time
Understood the relationship between resource usage and scaling decisions
Why This Matters: Kubernetes autoscaling is crucial for production environments because it:

Ensures High Availability: Automatically scales applications based on demand
Optimizes Resource Usage: Prevents over-provisioning and reduces costs
Improves Performance: Maintains response times during traffic spikes
Enables Resilience: Automatically recovers from failures by scaling appropriately
Real-World Applications:

E-commerce websites handling variable traffic loads
API services that need to scale during peak usage
Microservices architectures requiring dynamic resource allocation
Cost-effective cloud deployments that scale based on actual demand
Next Steps:

Explore advanced HPA configurations with custom metrics
Learn about Vertical Pod Autoscaling (VPA) for optimizing resource requests
Study cluster autoscaling for node-level scaling
Investigate monitoring solutions like Prometheus and Grafana for production environments
This foundational knowledge of Kubernetes autoscaling prepares you for managing containerized applications in production environments and is essential for the Docker Certified Associate (DCA) certification.
