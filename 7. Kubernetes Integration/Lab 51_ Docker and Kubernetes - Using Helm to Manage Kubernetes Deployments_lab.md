Lab 51: Docker and Kubernetes - Using Helm to Manage Kubernetes Deployments
Lab Objectives
By the end of this lab, you will be able to:

• Understand what Helm is and why it's essential for Kubernetes application management • Install and configure Helm CLI on a Linux system • Deploy applications to Kubernetes clusters using Helm charts • Create custom Helm charts for Dockerized web applications • Modify application configurations using Helm values • Perform application updates and rollbacks using Helm's version management features • Manage the complete lifecycle of Kubernetes applications with Helm

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker containers and containerization concepts • Familiarity with Kubernetes fundamentals including pods, services, and deployments • Knowledge of YAML file structure and syntax • Basic Linux command-line experience • Understanding of web applications and HTTP services

Technical Requirements: • Al Nafi provides ready-to-use Linux-based cloud machines with Docker and Kubernetes pre-installed • Simply click Start Lab to access your environment - no VM setup required • All necessary tools will be available in your cloud machine

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with: • Ubuntu Linux operating system • Docker Engine installed and running • Kubernetes cluster (minikube) ready for use • kubectl CLI tool configured • Internet connectivity for downloading Helm and charts

Task 1: Install Helm and Set Up the Helm CLI
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
Subtask 1.2: Download and Install Helm
Helm is the package manager for Kubernetes that simplifies application deployment and management.

# Download the latest Helm installation script
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3

# Make the script executable
chmod 700 get_helm.sh

# Run the installation script
./get_helm.sh

# Verify Helm installation
helm version

# Check Helm help to see available commands
helm help
Subtask 1.3: Initialize Helm Repository
# Add the official Helm stable charts repository
helm repo add stable https://charts.helm.sh/stable

# Add the Bitnami repository (popular for production-ready charts)
helm repo add bitnami https://charts.bitnami.com/bitnami

# Update the repository index
helm repo update

# List available repositories
helm repo list

# Search for available charts
helm search repo nginx
Task 2: Use Helm to Deploy Applications on a Kubernetes Cluster
Subtask 2.1: Deploy NGINX Web Server Using Helm
Let's deploy a simple NGINX web server to understand how Helm works.

# Search for NGINX charts
helm search repo nginx

# Install NGINX using Bitnami chart
helm install my-nginx bitnami/nginx

# Check the deployment status
helm status my-nginx

# List all Helm releases
helm list

# Check the Kubernetes resources created
kubectl get all
Subtask 2.2: Access the Deployed Application
# Get the service details
kubectl get services

# Since we're using minikube, we need to get the service URL
minikube service my-nginx --url

# In a new terminal, test the application
curl $(minikube service my-nginx --url)
Subtask 2.3: Explore Helm Chart Information
# Show the chart information
helm show chart bitnami/nginx

# Show the default values
helm show values bitnami/nginx

# Show all information about the chart
helm show all bitnami/nginx
Task 3: Create and Deploy a Helm Chart for a Dockerized Web Application
Subtask 3.1: Create a Custom Helm Chart
# Create a new directory for our project
mkdir ~/helm-lab
cd ~/helm-lab

# Create a new Helm chart
helm create my-web-app

# Explore the chart structure
ls -la my-web-app/
tree my-web-app/
Subtask 3.2: Understand the Chart Structure
# View the Chart.yaml file
cat my-web-app/Chart.yaml

# View the default values.yaml file
cat my-web-app/values.yaml

# View the deployment template
cat my-web-app/templates/deployment.yaml

# View the service template
cat my-web-app/templates/service.yaml
Subtask 3.3: Customize the Chart for Our Web Application
Let's modify the chart to deploy a simple Python web application.

# Edit the values.yaml file
nano my-web-app/values.yaml
Replace the content with:

# Default values for my-web-app
replicaCount: 2

image:
  repository: python
  pullPolicy: IfNotPresent
  tag: "3.9-slim"

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 8080
  targetPort: 8080

ingress:
  enabled: false

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

nodeSelector: {}
tolerations: []
affinity: {}

# Custom configuration for our Python app
app:
  name: "My Python Web App"
  message: "Hello from Helm!"
Subtask 3.4: Create a ConfigMap Template
# Create a ConfigMap template for our Python application
cat > my-web-app/templates/configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "my-web-app.fullname" . }}-config
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
data:
  app.py: |
    from http.server import HTTPServer, BaseHTTPRequestHandler
    import json
    import os
    
    class SimpleHandler(BaseHTTPRequestHandler):
        def do_GET(self):
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.end_headers()
            
            response = {
                'app_name': os.getenv('APP_NAME', 'Default App'),
                'message': os.getenv('APP_MESSAGE', 'Hello World!'),
                'version': '1.0.0'
            }
            
            self.wfile.write(json.dumps(response).encode())
    
    if __name__ == '__main__':
        server = HTTPServer(('0.0.0.0', 8080), SimpleHandler)
        print('Starting server on port 8080...')
        server.serve_forever()
EOF
Subtask 3.5: Update the Deployment Template
# Edit the deployment template
nano my-web-app/templates/deployment.yaml
Replace the content with:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-web-app.fullname" . }}
  labels:
    {{- include "my-web-app.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "my-web-app.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "my-web-app.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          command: ["python", "/app/app.py"]
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          env:
            - name: APP_NAME
              value: "{{ .Values.app.name }}"
            - name: APP_MESSAGE
              value: "{{ .Values.app.message }}"
          volumeMounts:
            - name: app-code
              mountPath: /app
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      volumes:
        - name: app-code
          configMap:
            name: {{ include "my-web-app.fullname" . }}-config
Subtask 3.6: Deploy the Custom Chart
# Validate the chart
helm lint my-web-app

# Dry run to see what would be deployed
helm install my-web-app ./my-web-app --dry-run --debug

# Deploy the application
helm install my-web-app ./my-web-app

# Check the deployment status
helm status my-web-app

# Verify the pods are running
kubectl get pods

# Check the service
kubectl get services
Subtask 3.7: Test the Custom Application
# Get the service URL
minikube service my-web-app --url

# Test the application
curl $(minikube service my-web-app --url)

# You should see a JSON response with your custom message
Task 4: Update the Application by Changing Values in the Helm Chart
Subtask 4.1: Update Application Configuration
# Create a custom values file for updates
cat > custom-values.yaml << 'EOF'
replicaCount: 3

app:
  name: "Updated Python Web App"
  message: "Hello from Updated Helm Chart!"

resources:
  limits:
    cpu: 600m
    memory: 600Mi
  requests:
    cpu: 300m
    memory: 300Mi
EOF
Subtask 4.2: Upgrade the Application
# Upgrade the application with new values
helm upgrade my-web-app ./my-web-app -f custom-values.yaml

# Check the upgrade status
helm status my-web-app

# Verify the number of replicas increased
kubectl get pods

# Test the updated application
curl $(minikube service my-web-app --url)
Subtask 4.3: View Helm Release History
# Check the release history
helm history my-web-app

# Get detailed information about a specific revision
helm get values my-web-app --revision 1
helm get values my-web-app --revision 2
Task 5: Roll Back to Previous Versions Using Helm
Subtask 5.1: Simulate a Problematic Update
Let's create an update that causes issues to demonstrate rollback functionality.

# Create a problematic values file
cat > problematic-values.yaml << 'EOF'
replicaCount: 1

image:
  repository: nginx  # Wrong image for our Python app
  tag: "latest"

app:
  name: "Broken App"
  message: "This won't work!"
EOF
Subtask 5.2: Apply the Problematic Update
# Apply the problematic update
helm upgrade my-web-app ./my-web-app -f problematic-values.yaml

# Check the status (you'll see issues)
kubectl get pods

# The pods will likely be in error state because nginx can't run our Python code
kubectl describe pods
Subtask 5.3: Roll Back to Previous Version
# Check the release history
helm history my-web-app

# Roll back to the previous version (revision 2)
helm rollback my-web-app 2

# Verify the rollback
helm status my-web-app

# Check that pods are running correctly again
kubectl get pods

# Test the application
curl $(minikube service my-web-app --url)
Subtask 5.4: Advanced Rollback Operations
# Roll back to a specific revision
helm rollback my-web-app 1

# Check the current status
helm history my-web-app

# Test the application to confirm it's working
curl $(minikube service my-web-app --url)
Additional Helm Operations
Managing Multiple Releases
# List all releases
helm list

# List releases in all namespaces
helm list --all-namespaces

# Install another release with a different name
helm install my-nginx-2 bitnami/nginx

# List releases again
helm list
Uninstalling Releases
# Uninstall a release
helm uninstall my-nginx

# Uninstall with keeping history
helm uninstall my-nginx-2 --keep-history

# Check what's left
helm list
kubectl get all
Working with Helm Repositories
# Search for charts across all repositories
helm search repo wordpress

# Add more repositories
helm repo add jetstack https://charts.jetstack.io

# Update all repositories
helm repo update

# Remove a repository
helm repo remove stable
Troubleshooting Common Issues
Issue 1: Chart Validation Errors
# If you encounter chart validation errors
helm lint my-web-app

# Check template rendering
helm template my-web-app ./my-web-app
Issue 2: Pod Startup Issues
# Check pod logs
kubectl logs -l app.kubernetes.io/name=my-web-app

# Describe pods for more details
kubectl describe pods -l app.kubernetes.io/name=my-web-app
Issue 3: Service Access Issues
# Check service endpoints
kubectl get endpoints

# Verify service configuration
kubectl describe service my-web-app
Cleanup
# Uninstall all releases
helm uninstall my-web-app

# Verify cleanup
kubectl get all
helm list

# Stop minikube if desired
minikube stop
Conclusion
Congratulations! You have successfully completed Lab 51 on using Helm to manage Kubernetes deployments. In this comprehensive lab, you have:

Key Accomplishments:

• Installed and configured Helm CLI - You learned how to install Helm 3 and set up chart repositories for managing Kubernetes applications

• Deployed applications using existing Helm charts - You successfully deployed NGINX using pre-built charts from the Bitnami repository

• Created custom Helm charts - You built a complete Helm chart for a Python web application, including templates, values, and configuration management

• Managed application updates - You learned how to upgrade applications by modifying Helm values and applying configuration changes

• Performed version rollbacks - You mastered the ability to roll back to previous application versions when issues arise

Why This Matters:

Helm is essential for modern Kubernetes operations because it:

Simplifies complex deployments by packaging applications with their dependencies
Enables version control for your Kubernetes applications
Provides templating capabilities that make applications configurable across environments
Offers rollback functionality that ensures quick recovery from problematic deployments
Standardizes deployment processes across development teams and environments
Real-World Applications:

The skills you've learned are directly applicable to:

Managing microservices architectures in production environments
Implementing CI/CD pipelines with automated deployments
Maintaining multiple environments (development, staging, production) with consistent configurations
Collaborating with teams using standardized deployment practices
Next Steps:

To further develop your Helm expertise, consider exploring:

Helm chart testing and validation strategies
Advanced templating techniques and helper functions
Helm hooks for complex deployment scenarios
Integration with GitOps workflows
Custom chart repositories and distribution
This lab has provided you with practical, hands-on experience that directly supports Docker Certified Associate (DCA) certification objectives and prepares you for real-world Kubernetes application management scenarios.
