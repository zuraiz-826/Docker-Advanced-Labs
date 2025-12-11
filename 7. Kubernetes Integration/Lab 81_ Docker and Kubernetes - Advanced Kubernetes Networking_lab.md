Lab 81: Docker and Kubernetes - Advanced Kubernetes Networking
Objectives
By the end of this lab, students will be able to:

Set up and configure advanced Kubernetes networking using CNI plugins (Calico or Flannel)
Implement network policies to secure pod-to-pod communication
Deploy and configure a service mesh using Istio for advanced traffic management
Configure Kubernetes Ingress Controllers to manage external traffic
Test and troubleshoot inter-container communication using kubectl port-forward
Understand the fundamentals of Kubernetes networking architecture
Apply security best practices for container networking
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and containerization concepts
Familiarity with Kubernetes fundamentals (pods, services, deployments)
Basic knowledge of Linux command line operations
Understanding of networking concepts (IP addresses, ports, protocols)
Experience with YAML configuration files
Basic kubectl command knowledge
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. No need to build your own VM or install additional software - everything is ready to use!

Your lab environment includes:

Ubuntu 20.04 LTS with Docker pre-installed
Kubernetes cluster (minikube) ready for configuration
kubectl command-line tool configured
All necessary networking tools and utilities
Task 1: Set up Kubernetes Networking with Calico
Subtask 1.1: Initialize Kubernetes Cluster
First, let's start our Kubernetes cluster and verify it's running properly.

# Start minikube with specific networking configuration
minikube start --driver=docker --cni=calico --memory=4096 --cpus=2

# Verify cluster status
kubectl cluster-info

# Check node status
kubectl get nodes -o wide
Subtask 1.2: Install Calico CNI Plugin
Calico provides advanced networking and security features for Kubernetes.

# Download and apply Calico manifest
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/calico.yaml

# Wait for Calico pods to be ready
kubectl wait --for=condition=ready pod -l k8s-app=calico-node -n kube-system --timeout=300s

# Verify Calico installation
kubectl get pods -n kube-system | grep calico
Subtask 1.3: Verify Network Configuration
Let's check that our networking is properly configured.

# Check available network interfaces
ip addr show

# Verify Calico is managing networking
kubectl get nodes -o yaml | grep podCIDR

# Create a test deployment to verify networking
kubectl create deployment test-nginx --image=nginx:latest --replicas=3

# Check pod IPs and distribution
kubectl get pods -o wide
Task 2: Configure Network Policies for Secure Communication
Subtask 2.1: Create Test Applications
We'll create multiple applications to demonstrate network policy enforcement.

# Create namespaces for different environments
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database

# Label namespaces for policy targeting
kubectl label namespace frontend tier=frontend
kubectl label namespace backend tier=backend
kubectl label namespace database tier=database
Create the frontend application:

# Save as frontend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-app
  namespace: frontend
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
  namespace: frontend
spec:
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
# Apply frontend application
kubectl apply -f frontend-app.yaml
Create the backend application:

# Save as backend-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend-app
  namespace: backend
  labels:
    app: backend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: backend
    spec:
      containers:
      - name: httpd
        image: httpd:2.4
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: backend-service
  namespace: backend
spec:
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
# Apply backend application
kubectl apply -f backend-app.yaml
Subtask 2.2: Test Default Communication
Before implementing network policies, let's test that pods can communicate freely.

# Get pod names and IPs
kubectl get pods -n frontend -o wide
kubectl get pods -n backend -o wide

# Test communication from frontend to backend
FRONTEND_POD=$(kubectl get pods -n frontend -l app=frontend -o jsonpath='{.items[0].metadata.name}')
BACKEND_IP=$(kubectl get service backend-service -n backend -o jsonpath='{.spec.clusterIP}')

# Test connectivity
kubectl exec -n frontend $FRONTEND_POD -- curl -m 5 http://$BACKEND_IP
Subtask 2.3: Implement Network Policies
Now let's create network policies to control traffic flow.

# Save as network-policies.yaml
# Deny all ingress traffic to backend namespace
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-backend
  namespace: backend
spec:
  podSelector: {}
  policyTypes:
  - Ingress
---
# Allow frontend to communicate with backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          tier: frontend
    ports:
    - protocol: TCP
      port: 80
---
# Deny all egress from frontend except to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress-policy
  namespace: frontend
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          tier: backend
    ports:
    - protocol: TCP
      port: 80
  - to: []
    ports:
    - protocol: UDP
      port: 53
# Apply network policies
kubectl apply -f network-policies.yaml

# Verify policies are created
kubectl get networkpolicies --all-namespaces
Subtask 2.4: Test Network Policy Enforcement
# Test that frontend can still reach backend
kubectl exec -n frontend $FRONTEND_POD -- curl -m 5 http://$BACKEND_IP

# Try to access backend from a pod in default namespace (should fail)
kubectl run test-pod --image=busybox --rm -it --restart=Never -- wget -qO- --timeout=5 http://$BACKEND_IP
Task 3: Implement Service Mesh with Istio
Subtask 3.1: Install Istio
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH

# Install Istio with demo profile
istioctl install --set values.defaultRevision=default -y

# Enable Istio injection for default namespace
kubectl label namespace default istio-injection=enabled

# Verify Istio installation
kubectl get pods -n istio-system
Subtask 3.2: Deploy Sample Application with Istio
# Save as bookinfo-app.yaml
apiVersion: v1
kind: Service
metadata:
  name: productpage
  labels:
    app: productpage
    service: productpage
spec:
  ports:
  - port: 9080
    name: http
  selector:
    app: productpage
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: bookinfo-productpage
  labels:
    account: productpage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: productpage-v1
  labels:
    app: productpage
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: productpage
      version: v1
  template:
    metadata:
      labels:
        app: productpage
        version: v1
    spec:
      serviceAccountName: bookinfo-productpage
      containers:
      - name: productpage
        image: docker.io/istio/examples-bookinfo-productpage-v1:1.17.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 9080
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
# Apply the application
kubectl apply -f bookinfo-app.yaml

# Wait for deployment to be ready
kubectl wait --for=condition=available --timeout=300s deployment/productpage-v1

# Verify Istio sidecar injection
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].name}'
Subtask 3.3: Configure Istio Gateway and Virtual Service
# Save as istio-gateway.yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
# Apply Istio configuration
kubectl apply -f istio-gateway.yaml

# Get Istio ingress gateway external IP
kubectl get svc istio-ingressgateway -n istio-system
Subtask 3.4: Test Service Mesh Features
# Enable Istio's built-in dashboard
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/kiali.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/release-1.19/samples/addons/prometheus.yaml

# Wait for Kiali to be ready
kubectl wait --for=condition=available --timeout=300s deployment/kiali -n istio-system

# Access Kiali dashboard (in a new terminal)
kubectl port-forward svc/kiali 20001:20001 -n istio-system

# Generate some traffic to see in Kiali
GATEWAY_URL=$(minikube service istio-ingressgateway -n istio-system --url | head -1)
for i in {1..10}; do curl -s "$GATEWAY_URL/productpage" > /dev/null; done
Task 4: Manage Ingress and Egress Traffic
Subtask 4.1: Install NGINX Ingress Controller
# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=300s

# Verify installation
kubectl get pods -n ingress-nginx
Subtask 4.2: Create Applications for Ingress Testing
# Save as web-apps.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app1
  template:
    metadata:
      labels:
        app: app1
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: app1-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app1-html
data:
  index.html: |
    <html>
    <body>
    <h1>Welcome to App 1</h1>
    <p>This is the first application</p>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: app1-service
spec:
  selector:
    app: app1
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app2
spec:
  replicas: 2
  selector:
    matchLabels:
      app: app2
  template:
    metadata:
      labels:
        app: app2
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: app2-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app2-html
data:
  index.html: |
    <html>
    <body>
    <h1>Welcome to App 2</h1>
    <p>This is the second application</p>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: app2-service
spec:
  selector:
    app: app2
  ports:
  - port: 80
    targetPort: 80
# Apply web applications
kubectl apply -f web-apps.yaml

# Wait for deployments to be ready
kubectl wait --for=condition=available --timeout=300s deployment/app1
kubectl wait --for=condition=available --timeout=300s deployment/app2
Subtask 4.3: Configure Ingress Rules
# Save as ingress-rules.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
  - host: app1.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
  - host: app2.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
  - http:
      paths:
      - path: /app1
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80
      - path: /app2
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
# Apply ingress rules
kubectl apply -f ingress-rules.yaml

# Verify ingress creation
kubectl get ingress
kubectl describe ingress web-ingress
Subtask 4.4: Test Ingress Functionality
# Get ingress controller external IP
INGRESS_IP=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# If using minikube, get the service URL
INGRESS_URL=$(minikube service ingress-nginx-controller -n ingress-nginx --url)

# Test path-based routing
curl $INGRESS_URL/app1
curl $INGRESS_URL/app2

# Test host-based routing (add entries to /etc/hosts if needed)
curl -H "Host: app1.local" $INGRESS_URL
curl -H "Host: app2.local" $INGRESS_URL
Subtask 4.5: Configure Egress Policies
# Save as egress-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
  - to:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    ports:
    - protocol: TCP
      port: 443
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-internal-egress
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: app1
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector:
        matchLabels:
          app: app2
    ports:
    - protocol: TCP
      port: 80
  - to: []
    ports:
    - protocol: UDP
      port: 53
# Apply egress policies
kubectl apply -f egress-policy.yaml

# Test egress restrictions
kubectl exec -it deployment/app1 -- curl -m 5 http://app2-service
kubectl exec -it deployment/app1 -- curl -m 5 http://google.com
Task 5: Test Inter-Container Communication
Subtask 5.1: Set Up Test Environment
# Create a debug pod for testing
kubectl run debug-pod --image=nicolaka/netshoot --rm -it --restart=Never -- /bin/bash

# In the debug pod, test various networking tools
nslookup kubernetes.default.svc.cluster.local
dig app1-service.default.svc.cluster.local
ping app2-service.default.svc.cluster.local
Subtask 5.2: Use kubectl port-forward for Testing
# Port-forward to app1 service
kubectl port-forward svc/app1-service 8080:80 &

# Test local access
curl http://localhost:8080

# Port-forward to a specific pod
APP1_POD=$(kubectl get pods -l app=app1 -o jsonpath='{.items[0].metadata.name}')
kubectl port-forward pod/$APP1_POD 8081:80 &

# Test pod-specific access
curl http://localhost:8081

# Clean up port-forwards
pkill -f "kubectl port-forward"
Subtask 5.3: Advanced Communication Testing
# Create a comprehensive test script
cat << 'EOF' > network-test.sh
#!/bin/bash

echo "=== Kubernetes Network Connectivity Test ==="

# Test DNS resolution
echo "1. Testing DNS resolution..."
kubectl exec deployment/app1 -- nslookup kubernetes.default.svc.cluster.local

# Test service-to-service communication
echo "2. Testing service-to-service communication..."
kubectl exec deployment/app1 -- curl -s http://app2-service/

# Test pod-to-pod communication
echo "3. Testing pod-to-pod communication..."
APP2_POD_IP=$(kubectl get pod -l app=app2 -o jsonpath='{.items[0].status.podIP}')
kubectl exec deployment/app1 -- curl -s http://$APP2_POD_IP/

# Test external connectivity (if allowed by network policies)
echo "4. Testing external connectivity..."
kubectl exec deployment/app1 -- curl -s -m 5 http://httpbin.org/ip || echo "External access blocked by network policy"

# Test ingress connectivity
echo "5. Testing ingress connectivity..."
INGRESS_URL=$(minikube service ingress-nginx-controller -n ingress-nginx --url)
curl -s $INGRESS_URL/app1 | grep -o "<title>.*</title>"

echo "=== Network test completed ==="
EOF

chmod +x network-test.sh
./network-test.sh
Subtask 5.4: Monitor Network Traffic
# Install and use tcpdump for network monitoring
kubectl exec -it deployment/app1 -- apt-get update && apt-get install -y tcpdump

# Monitor traffic on the pod (in background)
kubectl exec deployment/app1 -- tcpdump -i any -n port 80 &

# Generate traffic to monitor
kubectl exec deployment/app2 -- curl http://app1-service/

# Use Kubernetes network debugging tools
kubectl run tmp-shell --rm -i --tty --image nicolaka/netshoot -- /bin/bash
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Calico pods not starting

# Check system requirements
kubectl describe nodes
kubectl get events --sort-by=.metadata.creationTimestamp

# Restart minikube with more resources
minikube delete
minikube start --driver=docker --cni=calico --memory=6144 --cpus=4
Issue 2: Network policies not working

# Verify CNI supports network policies
kubectl get pods -n kube-system | grep calico

# Check policy syntax
kubectl describe networkpolicy <policy-name>

# Test with policy disabled
kubectl delete networkpolicy <policy-name>
Issue 3: Istio installation fails

# Check cluster resources
kubectl top nodes
kubectl get pods --all-namespaces

# Reinstall with minimal profile
istioctl install --set values.defaultRevision=default --set values.pilot.resources.requests.memory=128Mi
Issue 4: Ingress not accessible

# Check ingress controller status
kubectl get pods -n ingress-nginx
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller

# Verify service endpoints
kubectl get endpoints
Conclusion
In this comprehensive lab, you have successfully:

Configured Advanced Networking: Set up Kubernetes with Calico CNI plugin, providing robust networking capabilities for container orchestration
Implemented Security Policies: Created and tested network policies to control traffic flow between pods and namespaces, enhancing cluster security
Deployed Service Mesh: Installed and configured Istio to provide advanced traffic management, security, and observability features
Managed External Traffic: Set up NGINX Ingress Controller to handle external traffic routing and load balancing
Tested Communication: Used various tools and techniques to test and troubleshoot inter-container communication
Why This Matters: Advanced Kubernetes networking is crucial for production environments where security, scalability, and reliability are paramount. The skills you've learned enable you to:

Design secure, multi-tier applications with proper network isolation
Implement zero-trust networking principles in containerized environments
Manage complex traffic routing and load balancing scenarios
Monitor and troubleshoot network issues in distributed systems
Prepare for real-world container orchestration challenges
These networking concepts form the foundation for building robust, secure, and scalable containerized applications in enterprise environments. The hands-on experience with tools like Calico, Istio, and Ingress Controllers prepares you for advanced Kubernetes deployments and supports your journey toward Docker Certified Associate (DCA) certification.

Next Steps: Consider exploring advanced topics such as multi-cluster networking, service mesh security policies, and automated network policy generation to further enhance your Kubernetes networking expertise.
