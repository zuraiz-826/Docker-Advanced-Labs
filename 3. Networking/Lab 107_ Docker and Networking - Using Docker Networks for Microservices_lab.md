Lab 107: Docker and Networking - Using Docker Networks for Microservices
Objectives
By the end of this lab, you will be able to:

Create and manage custom Docker networks
Deploy multiple microservices using Docker Compose
Configure inter-container communication using custom networks
Set up environment variables for container communication
Test and troubleshoot network connectivity between containers
Understand the fundamentals of microservices networking in Docker
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with command-line interface
Basic knowledge of web applications (frontend and backend concepts)
Understanding of YAML file format
Basic networking concepts (IP addresses, ports)
Lab Environment
Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click Start Lab to access your pre-configured environment. No need to build your own VM or install Docker - everything is ready to go!

Your lab environment includes:

Ubuntu Linux with Docker pre-installed
Docker Compose pre-installed
Text editor (nano/vim)
All necessary networking tools
Task 1: Create a Custom Docker Network
1.1 Understanding Docker Networks
Docker networks allow containers to communicate with each other securely and efficiently. By default, Docker creates a bridge network, but custom networks provide better isolation and naming resolution.

1.2 Create a Custom Network
First, let's examine existing networks and then create our own:

# List existing Docker networks
docker network ls
You should see output similar to:

NETWORK ID     NAME      DRIVER    SCOPE
xxxxxxxxxxxx   bridge    bridge    local
xxxxxxxxxxxx   host      host      local
xxxxxxxxxxxx   none      null      local
Now, create a custom network for our microservices:

# Create a custom bridge network named 'microservices-net'
docker network create --driver bridge microservices-net
Verify the network was created:

# List networks again to confirm creation
docker network ls
You should now see your new network:

NETWORK ID     NAME                DRIVER    SCOPE
xxxxxxxxxxxx   bridge              bridge    local
xxxxxxxxxxxx   host                host      local
xxxxxxxxxxxx   microservices-net   bridge    local
xxxxxxxxxxxx   none                null      local
1.3 Inspect the Custom Network
Examine the network configuration:

# Inspect the custom network details
docker network inspect microservices-net
This command shows detailed information about the network including:

Network ID and name
Driver type
Subnet and gateway information
Connected containers (currently none)
Task 2: Deploy Multiple Microservices Using Docker Compose
2.1 Create Project Directory Structure
Set up the project directory:

# Create project directory
mkdir microservices-lab
cd microservices-lab

# Create subdirectories for our services
mkdir frontend backend
2.2 Create Backend API Service
Create a simple Node.js backend API:

# Navigate to backend directory
cd backend

# Create package.json
cat > package.json << 'EOF'
{
  "name": "backend-api",
  "version": "1.0.0",
  "description": "Simple backend API for microservices lab",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5"
  }
}
EOF

# Create the backend server
cat > server.js << 'EOF'
const express = require('express');
const cors = require('cors');

const app = express();
const PORT = process.env.PORT || 3001;

// Enable CORS for all routes
app.use(cors());
app.use(express.json());

// Sample data
const users = [
  { id: 1, name: 'Alice Johnson', email: 'alice@example.com' },
  { id: 2, name: 'Bob Smith', email: 'bob@example.com' },
  { id: 3, name: 'Charlie Brown', email: 'charlie@example.com' }
];

// Routes
app.get('/health', (req, res) => {
  res.json({ status: 'Backend API is running!', timestamp: new Date().toISOString() });
});

app.get('/api/users', (req, res) => {
  res.json({ users: users });
});

app.get('/api/users/:id', (req, res) => {
  const userId = parseInt(req.params.id);
  const user = users.find(u => u.id === userId);
  
  if (user) {
    res.json({ user: user });
  } else {
    res.status(404).json({ error: 'User not found' });
  }
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`Backend API server running on port ${PORT}`);
  console.log(`Environment: ${process.env.NODE_ENV || 'development'}`);
});
EOF

# Create Dockerfile for backend
cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy application code
COPY . .

# Expose port
EXPOSE 3001

# Start the application
CMD ["npm", "start"]
EOF
2.3 Create Frontend Service
Navigate back and create the frontend:

# Go back to project root
cd ../frontend

# Create a simple HTML frontend
cat > index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Microservices Lab - Frontend</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            max-width: 800px;
            margin: 0 auto;
            padding: 20px;
            background-color: #f5f5f5;
        }
        .container {
            background-color: white;
            padding: 30px;
            border-radius: 10px;
            box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        h1 {
            color: #333;
            text-align: center;
        }
        button {
            background-color: #007bff;
            color: white;
            border: none;
            padding: 10px 20px;
            margin: 10px;
            border-radius: 5px;
            cursor: pointer;
        }
        button:hover {
            background-color: #0056b3;
        }
        .result {
            margin-top: 20px;
            padding: 15px;
            background-color: #f8f9fa;
            border-radius: 5px;
            border-left: 4px solid #007bff;
        }
        .error {
            border-left-color: #dc3545;
            background-color: #f8d7da;
        }
        .user-card {
            background-color: #e9ecef;
            margin: 10px 0;
            padding: 15px;
            border-radius: 5px;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Microservices Lab Frontend</h1>
        <p>This frontend communicates with a backend API running in a separate container.</p>
        
        <div>
            <button onclick="checkBackendHealth()">Check Backend Health</button>
            <button onclick="fetchUsers()">Fetch All Users</button>
            <button onclick="fetchUser(1)">Fetch User 1</button>
        </div>
        
        <div id="result"></div>
    </div>

    <script>
        const BACKEND_URL = window.location.protocol + '//' + window.location.hostname + ':3001';
        
        function displayResult(data, isError = false) {
            const resultDiv = document.getElementById('result');
            resultDiv.className = isError ? 'result error' : 'result';
            
            if (typeof data === 'object') {
                resultDiv.innerHTML = '<pre>' + JSON.stringify(data, null, 2) + '</pre>';
            } else {
                resultDiv.innerHTML = data;
            }
        }
        
        async function checkBackendHealth() {
            try {
                const response = await fetch(`${BACKEND_URL}/health`);
                const data = await response.json();
                displayResult(data);
            } catch (error) {
                displayResult(`Error connecting to backend: ${error.message}`, true);
            }
        }
        
        async function fetchUsers() {
            try {
                const response = await fetch(`${BACKEND_URL}/api/users`);
                const data = await response.json();
                
                let html = '<h3>All Users:</h3>';
                data.users.forEach(user => {
                    html += `<div class="user-card">
                        <strong>${user.name}</strong><br>
                        Email: ${user.email}<br>
                        ID: ${user.id}
                    </div>`;
                });
                
                document.getElementById('result').innerHTML = html;
            } catch (error) {
                displayResult(`Error fetching users: ${error.message}`, true);
            }
        }
        
        async function fetchUser(id) {
            try {
                const response = await fetch(`${BACKEND_URL}/api/users/${id}`);
                const data = await response.json();
                
                if (response.ok) {
                    let html = `<h3>User Details:</h3>
                        <div class="user-card">
                            <strong>${data.user.name}</strong><br>
                            Email: ${data.user.email}<br>
                            ID: ${data.user.id}
                        </div>`;
                    document.getElementById('result').innerHTML = html;
                } else {
                    displayResult(data, true);
                }
            } catch (error) {
                displayResult(`Error fetching user: ${error.message}`, true);
            }
        }
    </script>
</body>
</html>
EOF

# Create Dockerfile for frontend
cat > Dockerfile << 'EOF'
FROM nginx:alpine

# Copy HTML file to nginx html directory
COPY index.html /usr/share/nginx/html/

# Copy custom nginx configuration
COPY nginx.conf /etc/nginx/nginx.conf

# Expose port 80
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
EOF

# Create nginx configuration
cat > nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server {
        listen 80;
        server_name localhost;

        location / {
            root   /usr/share/nginx/html;
            index  index.html;
        }

        # Enable CORS for API calls
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            expires 1y;
            add_header Cache-Control "public, immutable";
        }
    }
}
EOF
2.4 Create Docker Compose Configuration
Navigate back to the project root and create the Docker Compose file:

# Go back to project root
cd ..

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  backend:
    build: ./backend
    container_name: backend-api
    ports:
      - "3001:3001"
    environment:
      - NODE_ENV=production
      - PORT=3001
    networks:
      - microservices-net
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  frontend:
    build: ./frontend
    container_name: frontend-web
    ports:
      - "8080:80"
    depends_on:
      - backend
    networks:
      - microservices-net

networks:
  microservices-net:
    external: true
EOF
Task 3: Use the Custom Network for Inter-Container Communication
3.1 Build and Deploy the Services
Build and start the services using Docker Compose:

# Build and start all services
docker-compose up --build -d
The -d flag runs containers in detached mode (in the background).

3.2 Verify Container Deployment
Check that both containers are running:

# Check running containers
docker-compose ps
You should see both services running: ``` Name Command State Ports
backend-api docker-entrypoint.sh npm ... Up 0.0.0.0:3001->3001/tcp frontend-web /docker-entrypoint.sh ngin ... Up 0.0.0.0:8080->80/tcp


### 3.3 Test Inter-Container Communication

Test communication between containers using container names:

```bash
# Test backend health from within the frontend container
docker exec frontend-web wget -qO- http://backend:3001/health

# Test API endpoint
docker exec frontend-web wget -qO- http://backend:3001/api/users
Note: Notice how we use backend as the hostname instead of an IP address. Docker's built-in DNS resolution allows containers on the same network to communicate using container names.

Task 4: Set Up Environment Variables for Container Communication
4.1 Update Docker Compose with Environment Variables
Let's enhance our configuration with more environment variables:

# Create an environment file
cat > .env << 'EOF'
# Backend Configuration
BACKEND_PORT=3001
NODE_ENV=production

# Frontend Configuration
FRONTEND_PORT=8080

# Network Configuration
NETWORK_NAME=microservices-net
EOF

# Update docker-compose.yml with environment variables
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  backend:
    build: ./backend
    container_name: backend-api
    ports:
      - "${BACKEND_PORT:-3001}:3001"
    environment:
      - NODE_ENV=${NODE_ENV:-development}
      - PORT=3001
      - SERVICE_NAME=backend-api
      - FRONTEND_URL=http://frontend:80
    networks:
      - microservices-net
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  frontend:
    build: ./frontend
    container_name: frontend-web
    ports:
      - "${FRONTEND_PORT:-8080}:80"
    environment:
      - BACKEND_API_URL=http://backend:3001
      - SERVICE_NAME=frontend-web
    depends_on:
      - backend
    networks:
      - microservices-net

networks:
  microservices-net:
    external: true
EOF
4.2 Update Backend to Use Environment Variables
Enhance the backend server to display environment information:

# Update backend server.js
cat > backend/server.js << 'EOF'
const express = require('express');
const cors = require('cors');

const app = express();
const PORT = process.env.PORT || 3001;
const SERVICE_NAME = process.env.SERVICE_NAME || 'backend-api';
const NODE_ENV = process.env.NODE_ENV || 'development';

// Enable CORS for all routes
app.use(cors());
app.use(express.json());

// Sample data
const users = [
  { id: 1, name: 'Alice Johnson', email: 'alice@example.com' },
  { id: 2, name: 'Bob Smith', email: 'bob@example.com' },
  { id: 3, name: 'Charlie Brown', email: 'charlie@example.com' }
];

// Routes
app.get('/health', (req, res) => {
  res.json({ 
    status: 'Backend API is running!', 
    service: SERVICE_NAME,
    environment: NODE_ENV,
    timestamp: new Date().toISOString(),
    port: PORT
  });
});

app.get('/api/users', (req, res) => {
  res.json({ 
    users: users,
    service: SERVICE_NAME,
    timestamp: new Date().toISOString()
  });
});

app.get('/api/users/:id', (req, res) => {
  const userId = parseInt(req.params.id);
  const user = users.find(u => u.id === userId);
  
  if (user) {
    res.json({ 
      user: user,
      service: SERVICE_NAME,
      timestamp: new Date().toISOString()
    });
  } else {
    res.status(404).json({ 
      error: 'User not found',
      service: SERVICE_NAME,
      timestamp: new Date().toISOString()
    });
  }
});

// Environment info endpoint
app.get('/api/env', (req, res) => {
  res.json({
    service: SERVICE_NAME,
    environment: NODE_ENV,
    port: PORT,
    frontend_url: process.env.FRONTEND_URL || 'not set',
    timestamp: new Date().toISOString()
  });
});

app.listen(PORT, '0.0.0.0', () => {
  console.log(`${SERVICE_NAME} running on port ${PORT}`);
  console.log(`Environment: ${NODE_ENV}`);
  console.log(`Frontend URL: ${process.env.FRONTEND_URL || 'not set'}`);
});
EOF
4.3 Restart Services with New Configuration
# Stop current services
docker-compose down

# Rebuild and start with new configuration
docker-compose up --build -d
4.4 Test Environment Variables
Test the new environment endpoint:

# Test environment information
curl http://localhost:3001/api/env
You should see output showing the environment variables:

{
  "service": "backend-api",
  "environment": "production",
  "port": "3001",
  "frontend_url": "http://frontend:80",
  "timestamp": "2024-01-15T10:30:45.123Z"
}
Task 5: Test Connectivity and Troubleshoot Using Docker Network Inspect
5.1 Inspect the Network with Connected Containers
Now that our containers are running, let's inspect the network:

# Inspect the network to see connected containers
docker network inspect microservices-net
Look for the Containers section in the output. You should see both containers listed with their IP addresses:

"Containers": {
    "container-id-1": {
        "Name": "backend-api",
        "EndpointID": "...",
        "MacAddress": "...",
        "IPv4Address": "172.18.0.2/16",
        "IPv6Address": ""
    },
    "container-id-2": {
        "Name": "frontend-web",
        "EndpointID": "...",
        "MacAddress": "...",
        "IPv4Address": "172.18.0.3/16",
        "IPv6Address": ""
    }
}
5.2 Test Network Connectivity
Test various aspects of network connectivity:

# Test ping between containers using container names
docker exec frontend-web ping -c 3 backend

# Test ping using IP addresses (use the IPs from network inspect)
docker exec frontend-web ping -c 3 172.18.0.2

# Test HTTP connectivity
docker exec frontend-web wget -qO- http://backend:3001/health

# Test from backend to frontend
docker exec backend-api wget -qO- http://frontend:80
5.3 Test DNS Resolution
Docker provides automatic DNS resolution for container names:

# Test DNS resolution
docker exec frontend-web nslookup backend
docker exec backend-api nslookup frontend
5.4 Monitor Network Traffic
You can monitor network activity:

# View container logs to see network activity
docker-compose logs backend
docker-compose logs frontend

# Follow logs in real-time
docker-compose logs -f backend
Press Ctrl+C to stop following logs.

5.5 Test the Complete Application
Test the full application workflow:

# Test backend directly
curl http://localhost:3001/health
curl http://localhost:3001/api/users

# Test frontend (open in browser or use curl)
curl http://localhost:8080
For browser testing: Open your web browser and navigate to:

Frontend: http://localhost:8080
Backend API: http://localhost:3001/health
5.6 Advanced Network Troubleshooting
If you encounter connectivity issues, use these troubleshooting commands:

# Check if containers are on the correct network
docker inspect backend-api | grep NetworkMode
docker inspect frontend-web | grep NetworkMode

# List all networks and their containers
docker network ls
for network in $(docker network ls --format "{{.Name}}"); do
    echo "=== Network: $network ==="
    docker network inspect $network --format "{{range .Containers}}{{.Name}} {{end}}"
done

# Check container network settings
docker exec backend-api ip addr show
docker exec frontend-web ip addr show

# Test port connectivity
docker exec frontend-web nc -zv backend 3001
5.7 Performance Testing
Test the performance of inter-container communication:

# Time multiple requests
time for i in {1..10}; do
    docker exec frontend-web wget -qO- http://backend:3001/health > /dev/null
done

# Test concurrent requests
docker exec frontend-web ab -n 100 -c 10 http://backend:3001/health
Note: The ab (Apache Bench) command might not be available in the nginx container. If needed, you can install it or use alternative testing methods.

Additional Network Management Commands
Useful Docker Network Commands
# Create different types of networks
docker network create --driver bridge my-bridge-network
docker network create --driver host my-host-network

# Connect/disconnect containers to networks
docker network connect microservices-net container-name
docker network disconnect microservices-net container-name

# Remove unused networks
docker network prune

# Remove specific network (containers must be stopped first)
docker network rm microservices-net
Container Communication Patterns
# Test different communication patterns

# 1. Container name resolution
docker exec frontend-web wget -qO- http://backend:3001/health

# 2. Service name resolution (in Docker Compose)
docker exec frontend-web wget -qO- http://backend:3001/api/users

# 3. IP address communication
docker exec frontend-web wget -qO- http://172.18.0.2:3001/health

# 4. External network access
docker exec backend-api wget -qO- http://google.com
Troubleshooting Common Issues
Issue 1: Containers Cannot Communicate
Symptoms: Connection refused or timeout errors

Solutions:

# Check if containers are on the same network
docker network inspect microservices-net

# Verify container names and ports
docker-compose ps

# Check if services are listening on correct ports
docker exec backend-api netstat -tlnp
Issue 2: DNS Resolution Not Working
Symptoms: "Name or service not known" errors

Solutions:

# Restart Docker daemon
sudo systemctl restart docker

# Recreate containers
docker-compose down
docker-compose up -d

# Check Docker DNS settings
docker exec frontend-web cat /etc/resolv.conf
Issue 3: Port Conflicts
Symptoms: "Port already in use" errors

Solutions:

# Check what's using the port
sudo netstat -tlnp | grep :3001
sudo lsof -i :3001

# Use different ports in docker-compose.yml
# Change "3001:3001" to "3002:3001"
Issue 4: Network Already Exists
Symptoms: "Network already exists" error

Solutions:

# Remove existing network (stop containers first)
docker-compose down
docker network rm microservices-net
docker network create microservices-net
docker-compose up -d
Cleanup
When you're finished with the lab, clean up resources:

# Stop and remove containers
docker-compose down

# Remove custom network
docker network rm microservices-net

# Remove images (optional)
docker rmi microservices-lab_backend microservices-lab_frontend

# Remove project directory
cd ..
rm -rf microservices-lab
Conclusion
Congratulations! You have successfully completed Lab 107: Docker and Networking - Using Docker Networks for Microservices.

What You Accomplished
In this lab, you:

Created a custom Docker network - You learned how to create isolated networks for better container communication and security
Deployed multiple microservices - You built and deployed a frontend web application and backend API service using Docker Compose
Configured inter-container communication - You established secure communication between containers using Docker's built-in DNS resolution
Implemented environment variables - You configured services using environment variables for flexible deployment configurations
Mastered network troubleshooting - You learned essential commands and techniques for diagnosing and resolving network connectivity issues
Why This Matters
Understanding Docker networking is crucial for:

Microservices Architecture: Modern applications are built as collections of small, independent services that need to communicate efficiently
Security: Custom networks provide isolation and controlled communication between services
Scalability: Proper networking enables horizontal scaling and load distribution
DevOps Practices: Container networking is fundamental to CI/CD pipelines and production deployments
Cloud-Native Development: These skills directly apply to Kubernetes and other orchestration platforms
Key Concepts Learned
Docker Networks: Bridge, host, and custom network types
Service Discovery: How containers find and communicate with each other
DNS Resolution: Automatic hostname resolution in Docker networks
Environment Configuration: Using environment variables for service configuration
Network Troubleshooting: Essential debugging techniques for container networking
Next Steps
To further develop your Docker networking skills:

Explore Docker Swarm mode networking
Learn about Kubernetes networking concepts
Study load balancing with Docker networks
Practice with multi-host networking scenarios
Investigate network security best practices
This lab provides a solid foundation for the Docker Certified Associate (DCA) certification and prepares you for real-world microservices deployment scenarios.
