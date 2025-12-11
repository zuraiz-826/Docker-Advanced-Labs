Lab 15: Docker and Kubernetes - Introduction to Kubernetes
Lab Objectives
By the end of this lab, students will be able to:

Understand the fundamental concepts of Kubernetes and its role in container orchestration
Install and configure a local Kubernetes cluster using minikube
Deploy multi-container applications using Kubernetes pods
Create and manage Kubernetes services to expose applications
Use kubectl command-line tool to interact with Kubernetes clusters
Understand how Kubernetes manages containerized workloads across nodes
Implement basic load balancing and service discovery in Kubernetes
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and images
Familiarity with Linux command line operations
Basic knowledge of YAML file structure
Understanding of networking concepts (ports, IP addresses)
Completed previous Docker labs in this series
Required Knowledge Areas:
Docker fundamentals
Container lifecycle management
Basic networking concepts
Command-line interface usage
Lab Environment Setup
Good News! Al Nafi provides you with ready-to-use Linux-based cloud machines. Simply click Start Lab to access your pre-configured environment. No need to build your own virtual machine or worry about system compatibility issues.

Your lab environment includes:

Ubuntu 20.04 LTS or later
Docker pre-installed and configured
Internet connectivity for downloading Kubernetes components
Sufficient resources (2 CPU cores, 4GB RAM minimum)
Task 1: Install and Configure Kubernetes Cluster using Minikube
Subtask 1.1: Install Minikube
Minikube is a tool that creates a single-node Kubernetes cluster on your local machine, perfect for learning and development.

Step 1: Update your system packages

sudo apt update && sudo apt upgrade -y
Step 2: Install required dependencies

sudo apt install -y curl wget apt-transport-https
Step 3: Download and install minikube

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
Step 4: Verify minikube installation

minikube version
Subtask 1.2: Install kubectl
kubectl is the command-line tool for interacting with Kubernetes clusters.

Step 1: Download kubectl

curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
Step 2: Make kubectl executable and move to PATH

chmod +x kubectl
sudo mv kubectl /usr/local/bin/
Step 3: Verify kubectl installation

kubectl version --client
Subtask 1.3: Start Minikube Cluster
Step 1: Start minikube with Docker driver

minikube start --driver=docker
Step 2: Verify cluster status

minikube status
Step 3: Check cluster information

kubectl cluster-info
Step 4: View cluster nodes

kubectl get nodes
You should see output similar to:

NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.28.3
Task 2: Deploy a Simple Multi-Container Application using Kubernetes Pods
Subtask 2.1: Create a Simple Web Application Pod
Step 1: Create a directory for your Kubernetes manifests

mkdir ~/k8s-lab && cd ~/k8s-lab
Step 2: Create a simple nginx pod manifest

cat > nginx-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
EOF
Step 3: Deploy the pod

kubectl apply -f nginx-pod.yaml
Step 4: Verify pod creation

kubectl get pods
Step 5: Get detailed pod information

kubectl describe pod nginx-pod
Subtask 2.2: Create a Multi-Container Pod
Step 1: Create a multi-container pod with nginx and redis

cat > multi-container-pod.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
  labels:
    app: multi-app
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
  - name: redis
    image: redis:latest
    ports:
    - containerPort: 6379
EOF
Step 2: Deploy the multi-container pod

kubectl apply -f multi-container-pod.yaml
Step 3: Check both pods are running

kubectl get pods
Step 4: View logs from specific container

kubectl logs multi-container-pod -c nginx
kubectl logs multi-container-pod -c redis
Subtask 2.3: Create a Deployment for Better Management
Step 1: Create a deployment manifest

cat > nginx-deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
EOF
Step 2: Deploy the application

kubectl apply -f nginx-deployment.yaml
Step 3: Verify deployment

kubectl get deployments
kubectl get pods -l app=nginx
Task 3: Expose a Service and Access it Externally using LoadBalancer
Subtask 3.1: Create a ClusterIP Service
Step 1: Create a service manifest for internal access

cat > nginx-service-clusterip.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-clusterip
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
EOF
Step 2: Apply the service

kubectl apply -f nginx-service-clusterip.yaml
Step 3: View the service

kubectl get services
Subtask 3.2: Create a NodePort Service
Step 1: Create a NodePort service for external access

cat > nginx-service-nodeport.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-nodeport
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
  type: NodePort
EOF
Step 2: Apply the NodePort service

kubectl apply -f nginx-service-nodeport.yaml
Step 3: Get service details

kubectl get services
Subtask 3.3: Access the Application Externally
Step 1: Get minikube IP address

minikube ip
Step 2: Access the application using NodePort

curl http://$(minikube ip):30080
Step 3: Use minikube service command for easy access

minikube service nginx-service-nodeport --url
Step 4: Open in browser (if GUI available)

minikube service nginx-service-nodeport
Subtask 3.4: Create a LoadBalancer Service (Minikube Tunnel)
Step 1: Create LoadBalancer service

cat > nginx-service-loadbalancer.yaml << EOF
apiVersion: v1
kind: Service
metadata:
  name: nginx-service-loadbalancer
spec:
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
EOF
Step 2: Apply the LoadBalancer service

kubectl apply -f nginx-service-loadbalancer.yaml
Step 3: Start minikube tunnel (in a new terminal)

minikube tunnel
Step 4: Check external IP assignment

kubectl get services nginx-service-loadbalancer
Step 5: Access via LoadBalancer IP

curl http://<EXTERNAL-IP>
Task 4: Understand Kubernetes Workload Management
Subtask 4.1: Explore Pod Lifecycle
Step 1: Watch pods in real-time

kubectl get pods -w
Step 2: Delete a pod and observe recreation

kubectl delete pod <pod-name>
kubectl get pods
Step 3: Scale the deployment

kubectl scale deployment nginx-deployment --replicas=5
kubectl get pods
Step 4: Scale down

kubectl scale deployment nginx-deployment --replicas=2
kubectl get pods
Subtask 4.2: Understand Resource Management
Step 1: Check resource usage

kubectl top nodes
kubectl top pods
Step 2: Describe deployment for resource information

kubectl describe deployment nginx-deployment
Step 3: View events

kubectl get events --sort-by=.metadata.creationTimestamp
Subtask 4.3: Rolling Updates
Step 1: Update the nginx image

kubectl set image deployment/nginx-deployment nginx=nginx:1.21
Step 2: Watch the rolling update

kubectl rollout status deployment/nginx-deployment
Step 3: Check rollout history

kubectl rollout history deployment/nginx-deployment
Step 4: Rollback if needed

kubectl rollout undo deployment/nginx-deployment
Task 5: Use kubectl to Interact with Kubernetes Clusters and Resources
Subtask 5.1: Basic kubectl Commands
Step 1: List all resources in default namespace

kubectl get all
Step 2: Get detailed information about resources

kubectl describe deployment nginx-deployment
kubectl describe service nginx-service-nodeport
Step 3: View resource definitions in YAML

kubectl get deployment nginx-deployment -o yaml
kubectl get service nginx-service-nodeport -o yaml
Step 4: Get resources with labels

kubectl get pods -l app=nginx
kubectl get all -l app=nginx
Subtask 5.2: Advanced kubectl Operations
Step 1: Execute commands in pods

kubectl exec -it nginx-pod -- /bin/bash
# Inside the pod:
# nginx -v
# exit
Step 2: Port forwarding for local access

kubectl port-forward pod/nginx-pod 8080:80
# Test in another terminal: curl http://localhost:8080
Step 3: Copy files to/from pods

echo "Hello Kubernetes" > test.txt
kubectl cp test.txt nginx-pod:/tmp/test.txt
kubectl exec nginx-pod -- cat /tmp/test.txt
Step 4: View logs with different options

kubectl logs nginx-pod
kubectl logs -f nginx-pod  # Follow logs
kubectl logs --tail=10 nginx-pod  # Last 10 lines
Subtask 5.3: Working with Namespaces
Step 1: Create a new namespace

kubectl create namespace test-namespace
Step 2: List all namespaces

kubectl get namespaces
Step 3: Deploy resources in specific namespace

kubectl apply -f nginx-pod.yaml -n test-namespace
Step 4: List resources in specific namespace

kubectl get pods -n test-namespace
kubectl get all -n test-namespace
Step 5: Set default namespace context

kubectl config set-context --current --namespace=test-namespace
kubectl get pods  # Now shows test-namespace pods
Step 6: Switch back to default namespace

kubectl config set-context --current --namespace=default
Subtask 5.4: Resource Management and Cleanup
Step 1: View resource quotas and limits

kubectl describe node minikube
Step 2: Delete specific resources

kubectl delete pod nginx-pod
kubectl delete service nginx-service-clusterip
Step 3: Delete resources by label

kubectl delete pods -l app=nginx
Step 4: Delete all resources from files

kubectl delete -f nginx-deployment.yaml
kubectl delete -f nginx-service-nodeport.yaml
kubectl delete -f nginx-service-loadbalancer.yaml
Step 5: Clean up namespace

kubectl delete namespace test-namespace
Troubleshooting Common Issues
Issue 1: Minikube Won't Start
Solution:

minikube delete
minikube start --driver=docker --force
Issue 2: Pods Stuck in Pending State
Check:

kubectl describe pod <pod-name>
kubectl get events
Issue 3: Service Not Accessible
Verify:

kubectl get endpoints
kubectl describe service <service-name>
Issue 4: Image Pull Errors
Check:

kubectl describe pod <pod-name>
# Look for image pull policy and repository access
Lab Verification
Verification Checklist
Step 1: Verify cluster is running

kubectl cluster-info
kubectl get nodes
Step 2: Verify deployments are working

kubectl get deployments
kubectl get pods
kubectl get services
Step 3: Test external access

curl http://$(minikube ip):30080
Step 4: Verify kubectl functionality

kubectl version
kubectl get all
Conclusion
Congratulations! You have successfully completed the Introduction to Kubernetes lab. Here's what you accomplished:

Key Achievements:
Kubernetes Cluster Setup: You installed and configured a local Kubernetes cluster using minikube, providing you with a complete container orchestration environment.

Container Orchestration: You learned how Kubernetes manages containerized applications through pods, deployments, and services, understanding the fundamental building blocks of container orchestration.

Multi-Container Applications: You deployed both single and multi-container pods, experiencing how Kubernetes handles complex application architectures.

Service Discovery and Load Balancing: You created different types of services (ClusterIP, NodePort, LoadBalancer) to expose applications and understand how Kubernetes handles networking and external access.

kubectl Mastery: You gained hands-on experience with kubectl, the primary tool for interacting with Kubernetes clusters, learning essential commands for managing resources.

Workload Management: You explored how Kubernetes automatically manages application lifecycle, scaling, rolling updates, and self-healing capabilities.

Why This Matters:
Industry Relevance: Kubernetes is the de facto standard for container orchestration in production environments, used by organizations worldwide.
Career Advancement: These skills are essential for DevOps engineers, cloud architects, and software developers working with modern containerized applications.
Scalability Understanding: You now understand how applications can be scaled and managed across multiple nodes and environments.
Foundation for Advanced Topics: This lab provides the groundwork for more advanced Kubernetes concepts like persistent storage, security, and multi-cluster management.
Next Steps:
Explore Kubernetes persistent volumes and storage
Learn about Kubernetes security and RBAC
Study advanced networking with ingress controllers
Practice with Helm package manager
Investigate monitoring and logging solutions
You now have the fundamental knowledge to work with Kubernetes in development environments and are prepared to tackle more advanced container orchestration challenges. This foundation will serve you well as you continue your journey in modern application deployment and management.
