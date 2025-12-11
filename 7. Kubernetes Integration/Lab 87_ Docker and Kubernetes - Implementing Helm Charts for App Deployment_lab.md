Lab 87: Docker and Kubernetes - Implementing Helm Charts for App Deployment
Lab Objectives
By the end of this lab, students will be able to:

Install and configure Helm on a Kubernetes cluster
Deploy Dockerized applications using existing Helm charts
Create and customize Helm charts for specific applications
Perform application upgrades and rollbacks using Helm
Monitor and manage Helm deployments using Kubernetes commands
Understand the benefits of using Helm for Kubernetes application management
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and containerization concepts
Fundamental knowledge of Kubernetes architecture and components
Familiarity with YAML syntax and configuration files
Basic command-line interface experience
Understanding of application deployment concepts
Required Knowledge Areas
Docker container basics
Kubernetes pods, services, and deployments
Command-line operations in Linux
Basic networking concepts
YAML file structure and syntax
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker and Kubernetes already installed. Simply click Start Lab to access your environment - no need to build your own virtual machine or install software.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker installed
Kubernetes cluster (minikube) pre-configured
kubectl command-line tool ready to use
Internet access for downloading Helm and charts
Task 1: Set up Helm on a Kubernetes cluster
Subtask 1.1: Verify Kubernetes Cluster Status
First, let's ensure your Kubernetes cluster is running properly.

# Check if minikube is running
minikube status

# If minikube is not running, start it
minikube start

# Verify kubectl can connect to the cluster
kubectl cluster-info

# Check available nodes
kubectl get nodes
Subtask 1.2: Install Helm
Helm is the package manager for Kubernetes that simplifies application deployment and management.

# Download and install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify Helm installation
helm version

# Add the official Helm stable repository
helm repo add stable https://charts.helm.sh/stable

# Add the bitnami repository (commonly used)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update repository information
helm repo update

# List available repositories
helm repo list
Subtask 1.3: Verify Helm Setup
# Check Helm can communicate with Kubernetes
helm list

# Search for available charts
helm search repo nginx

# Get information about a specific chart
helm show chart bitnami/nginx
Task 2: Deploy a Dockerized application using an existing Helm chart
Subtask 2.1: Deploy NGINX Web Server
We'll deploy NGINX as our first application using an existing Helm chart.

# Create a namespace for our applications
kubectl create namespace helm-demo

# Deploy NGINX using Helm
helm install my-nginx bitnami/nginx --namespace helm-demo

# Check the deployment status
helm status my-nginx --namespace helm-demo

# Verify the deployment in Kubernetes
kubectl get all -n helm-demo
Subtask 2.2: Access the Deployed Application
# Get the service details
kubectl get svc -n helm-demo

# Port forward to access NGINX locally
kubectl port-forward -n helm-demo svc/my-nginx 8080:80 &

# Test the application (open another terminal or run in background)
curl http://localhost:8080
Subtask 2.3: Explore Helm Release Information
# List all Helm releases
helm list --namespace helm-demo

# Get detailed information about the release
helm get all my-nginx --namespace helm-demo

# View the values used in the deployment
helm get values my-nginx --namespace helm-demo
Task 3: Customize a Helm chart for your application
Subtask 3.1: Create a Custom Values File
Let's customize the NGINX deployment with our own configuration.

# Create a custom values file
cat > custom-nginx-values.yaml << EOF
replicaCount: 2

image:
  tag: "1.21"

service:
  type: NodePort
  port: 80
  nodePort: 30080

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

ingress:
  enabled: false
EOF
Subtask 3.2: Deploy with Custom Values
# Deploy a new release with custom values
helm install my-custom-nginx bitnami/nginx \
  --namespace helm-demo \
  --values custom-nginx-values.yaml

# Verify the custom deployment
kubectl get pods -n helm-demo
kubectl get svc -n helm-demo
Subtask 3.3: Create Your Own Helm Chart
Now let's create a completely custom Helm chart for a simple web application.

# Create a new Helm chart
helm create my-webapp

# Explore the chart structure
ls -la my-webapp/
tree my-webapp/
Subtask 3.4: Customize the Chart Templates
# Edit the deployment template
cat > my-webapp/templates/deployment.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-webapp.fullname" . }}
  labels:
    {{- include "my-webapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-webapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
EOF
Subtask 3.5: Update Chart Values
# Edit the values file
cat > my-webapp/values.yaml << EOF
replicaCount: 1

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.21"

service:
  type: ClusterIP
  port: 80

resources:
  limits:
    cpu: 100m
    memory: 128Mi
  requests:
    cpu: 50m
    memory: 64Mi

nodeSelector: {}
tolerations: []
affinity: {}
EOF
Subtask 3.6: Deploy Your Custom Chart
# Validate the chart
helm lint my-webapp

# Deploy your custom chart
helm install my-custom-app ./my-webapp --namespace helm-demo

# Verify the deployment
kubectl get all -n helm-demo -l app.kubernetes.io/name=my-webapp
Task 4: Use Helm to upgrade and roll back deployments
Subtask 4.1: Upgrade an Existing Release
# Check current release version
helm list --namespace helm-demo

# Upgrade the NGINX release with new values
cat > upgrade-values.yaml << EOF
replicaCount: 3

image:
  tag: "1.22"

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
EOF

# Perform the upgrade
helm upgrade my-nginx bitnami/nginx \
  --namespace helm-demo \
  --values upgrade-values.yaml

# Check the upgrade status
helm status my-nginx --namespace helm-demo
Subtask 4.2: View Release History
# View the release history
helm history my-nginx --namespace helm-demo

# Get detailed information about a specific revision
helm get all my-nginx --revision 1 --namespace helm-demo
Subtask 4.3: Perform a Rollback
# Rollback to the previous version
helm rollback my-nginx 1 --namespace helm-demo

# Verify the rollback
helm history my-nginx --namespace helm-demo

# Check that pods are using the original configuration
kubectl get pods -n helm-demo -o wide
Subtask 4.4: Test Upgrade and Rollback Process
# Create a test upgrade with intentional issues
cat > problematic-values.yaml << EOF
replicaCount: 5

image:
  tag: "nonexistent-tag"

resources:
  limits:
    cpu: 2000m
    memory: 4Gi
EOF

# Attempt the upgrade
helm upgrade my-nginx bitnami/nginx \
  --namespace helm-demo \
  --values problematic-values.yaml

# Check if the upgrade caused issues
kubectl get pods -n helm-demo

# Rollback if there are problems
helm rollback my-nginx --namespace helm-demo

# Verify the rollback worked
kubectl get pods -n helm-demo
Task 5: Monitor Helm deployments using Kubernetes commands
Subtask 5.1: Monitor Deployment Status
# Watch pod status in real-time
kubectl get pods -n helm-demo -w

# In another terminal, check deployment status
kubectl get deployments -n helm-demo

# Check replica sets
kubectl get rs -n helm-demo
Subtask 5.2: View Application Logs
# Get logs from NGINX pods
kubectl logs -n helm-demo -l app.kubernetes.io/name=nginx

# Follow logs in real-time
kubectl logs -n helm-demo -l app.kubernetes.io/name=nginx -f

# Get logs from a specific pod
POD_NAME=$(kubectl get pods -n helm-demo -l app.kubernetes.io/name=nginx -o jsonpath='{.items[0].metadata.name}')
kubectl logs -n helm-demo $POD_NAME
Subtask 5.3: Monitor Resource Usage
# Check resource usage of pods
kubectl top pods -n helm-demo

# Check node resource usage
kubectl top nodes

# Describe a pod to see detailed information
kubectl describe pod -n helm-demo $POD_NAME
Subtask 5.4: Monitor Services and Endpoints
# Check service status
kubectl get svc -n helm-demo

# Check endpoints
kubectl get endpoints -n helm-demo

# Describe service details
kubectl describe svc -n helm-demo my-nginx
Subtask 5.5: Create Monitoring Dashboard Commands
# Create a script to monitor all Helm releases
cat > monitor-helm.sh << 'EOF'
#!/bin/bash

echo "=== Helm Releases ==="
helm list --all-namespaces

echo -e "\n=== Pod Status ==="
kubectl get pods -n helm-demo

echo -e "\n=== Service Status ==="
kubectl get svc -n helm-demo

echo -e "\n=== Recent Events ==="
kubectl get events -n helm-demo --sort-by='.lastTimestamp' | tail -10
EOF

chmod +x monitor-helm.sh

# Run the monitoring script
./monitor-helm.sh
Advanced Monitoring and Troubleshooting
Subtask 5.6: Troubleshooting Common Issues
# Check for failed pods
kubectl get pods -n helm-demo --field-selector=status.phase=Failed

# Check pod events for troubleshooting
kubectl get events -n helm-demo --sort-by='.lastTimestamp'

# Debug a problematic pod
kubectl describe pod -n helm-demo <pod-name>

# Check Helm release notes
helm get notes my-nginx --namespace helm-demo
Subtask 5.7: Performance Monitoring
# Monitor resource consumption over time
watch kubectl top pods -n helm-demo

# Check cluster resource availability
kubectl describe nodes

# Monitor persistent volumes if used
kubectl get pv,pvc -n helm-demo
Lab Cleanup
Before concluding the lab, let's clean up the resources we created.

# Uninstall Helm releases
helm uninstall my-nginx --namespace helm-demo
helm uninstall my-custom-nginx --namespace helm-demo
helm uninstall my-custom-app --namespace helm-demo

# Delete the namespace
kubectl delete namespace helm-demo

# Remove custom files
rm -f custom-nginx-values.yaml upgrade-values.yaml problematic-values.yaml
rm -f monitor-helm.sh
rm -rf my-webapp/

# Verify cleanup
helm list --all-namespaces
kubectl get namespaces
Troubleshooting Common Issues
Issue 1: Helm Installation Problems
Problem: Helm command not found after installation Solution:

# Check if Helm is in PATH
which helm

# If not found, add to PATH or reinstall
export PATH=$PATH:/usr/local/bin
Issue 2: Repository Access Issues
Problem: Cannot access Helm repositories Solution:

# Update repository cache
helm repo update

# Remove and re-add problematic repositories
helm repo remove bitnami
helm repo add bitnami https://charts.bitnami.com/bitnami
Issue 3: Deployment Failures
Problem: Pods stuck in Pending or CrashLoopBackOff state Solution:

# Check pod details
kubectl describe pod <pod-name> -n helm-demo

# Check resource constraints
kubectl get nodes
kubectl top nodes

# Check for image pull issues
kubectl get events -n helm-demo
Key Concepts Summary
Helm Architecture
Charts: Packages of pre-configured Kubernetes resources
Releases: Instances of charts deployed to clusters
Values: Configuration parameters for customizing deployments
Templates: Kubernetes manifest templates with placeholders
Best Practices
Always use version control for custom charts
Test charts in development environments first
Use meaningful release names and namespaces
Regularly update Helm repositories
Monitor deployments after upgrades
Keep rollback strategies ready
Conclusion
In this comprehensive lab, you have successfully:

Installed and configured Helm on a Kubernetes cluster, establishing the foundation for efficient application deployment management
Deployed applications using existing Helm charts, learning how to leverage community-maintained packages for quick deployments
Created and customized Helm charts, gaining hands-on experience in tailoring deployments to specific requirements
Performed upgrades and rollbacks, mastering essential skills for maintaining applications in production environments
Implemented monitoring strategies, ensuring you can track and troubleshoot Helm deployments effectively
Why This Matters
Helm charts revolutionize Kubernetes application deployment by:

Simplifying Complex Deployments: Converting multiple Kubernetes manifests into single, manageable packages
Enabling Reusability: Creating templates that can be used across different environments with varying configurations
Providing Version Control: Tracking deployment history and enabling easy rollbacks when issues arise
Standardizing Deployments: Ensuring consistent application deployment practices across teams and environments
Reducing Human Error: Automating deployment processes and reducing manual configuration mistakes
Real-World Applications
The skills you've developed in this lab are directly applicable to:

Production Application Deployment: Managing enterprise applications with complex dependencies
DevOps Pipeline Integration: Incorporating Helm into CI/CD workflows for automated deployments
Multi-Environment Management: Deploying the same applications across development, staging, and production environments
Microservices Architecture: Managing multiple interconnected services efficiently
Cloud-Native Development: Supporting modern application development practices
Next Steps
To further develop your Helm and Kubernetes expertise:

Explore advanced Helm features like hooks and tests
Learn about Helm chart repositories and distribution
Study integration with GitOps workflows
Practice with more complex, multi-tier applications
Investigate Helm security best practices and RBAC integration
This lab has provided you with practical, hands-on experience that directly supports Docker Certified Associate (DCA) certification objectives and prepares you for real-world container orchestration challenges in professional environments.
