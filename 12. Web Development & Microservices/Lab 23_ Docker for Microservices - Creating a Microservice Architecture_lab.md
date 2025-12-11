Lab 23: Docker for Microservices - Creating a Microservice Architecture
Lab Objectives
By the end of this lab, students will be able to:

Understand the fundamentals of microservices architecture
Build multiple containerized microservices using Docker
Implement inter-service communication using Docker networking
Deploy and orchestrate microservices using Docker Compose
Scale microservices horizontally and monitor their performance
Integrate an API Gateway to manage service routing
Apply best practices for microservices deployment and management
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker containers and images
Familiarity with command-line interface (CLI)
Basic knowledge of REST APIs and HTTP protocols
Understanding of JSON data format
Basic programming concepts (variables, functions, APIs)
Completed previous Docker labs or equivalent experience
Required Knowledge Areas:
Docker fundamentals (containers, images, Dockerfile)
Basic networking concepts
Understanding of web services and APIs
Command-line navigation skills
Lab Environment Setup
Good News! Al Nafi provides ready-to-use Linux-based cloud machines for this lab. Simply click Start Lab and you'll have access to a fully configured environment with all necessary tools pre-installed.

Pre-installed Tools:
Docker Engine (latest stable version)
Docker Compose
Node.js runtime
Python 3.x
Text editors (nano, vim)
curl and wget utilities
Task 1: Understanding Microservices Architecture
Subtask 1.1: Introduction to Microservices
Microservices are a software architecture pattern where applications are built as a collection of small, independent services that communicate over well-defined APIs. Each service is:

Independently deployable
Loosely coupled
Organized around business capabilities
Owned by small teams
Subtask 1.2: Benefits of Microservices with Docker
Docker containers provide the perfect platform for microservices because they offer:

Isolation: Each service runs in its own container
Portability: Services can run anywhere Docker is supported
Scalability: Individual services can be scaled independently
Consistency: Same environment from development to production
Task 2: Building Multiple Microservices
Subtask 2.1: Create Project Structure
First, let's create a well-organized project structure for our microservices:

mkdir microservices-lab
cd microservices-lab

# Create directories for each service
mkdir user-service
mkdir order-service
mkdir product-service
mkdir api-gateway
mkdir docker-compose-files

# Create a shared network configuration
mkdir shared-configs
Subtask 2.2: Build User Service
The User Service will handle user registration, authentication, and profile management.

Create User Service Files:
cd user-service
Create the main application file:

nano app.js
Add the following Node.js code:

const express = require('express');
const app = express();
const PORT = process.env.PORT || 3001;

// Middleware
app.use(express.json());

// In-memory user storage (for demo purposes)
let users = [
    { id: 1, name: 'John Doe', email: 'john@example.com', role: 'customer' },
    { id: 2, name: 'Jane Smith', email: 'jane@example.com', role: 'admin' }
];

// Health check endpoint
app.get('/health', (req, res) => {
    res.json({ 
        service: 'user-service', 
        status: 'healthy', 
        timestamp: new Date().toISOString() 
    });
});

// Get all users
app.get('/users', (req, res) => {
    res.json({
        success: true,
        data: users,
        count: users.length
    });
});

// Get user by ID
app.get('/users/:id', (req, res) => {
    const userId = parseInt(req.params.id);
    const user = users.find(u => u.id === userId);
    
    if (!user) {
        return res.status(404).json({
            success: false,
            message: 'User not found'
        });
    }
    
    res.json({
        success: true,
        data: user
    });
});

// Create new user
app.post('/users', (req, res) => {
    const { name, email, role } = req.body;
    
    if (!name || !email) {
        return res.status(400).json({
            success: false,
            message: 'Name and email are required'
        });
    }
    
    const newUser = {
        id: users.length + 1,
        name,
        email,
        role: role || 'customer'
    };
    
    users.push(newUser);
    
    res.status(201).json({
        success: true,
        data: newUser,
        message: 'User created successfully'
    });
});

// Start server
app.listen(PORT, '0.0.0.0', () => {
    console.log(`User Service running on port ${PORT}`);
    console.log(`Health check: http://localhost:${PORT}/health`);
});
Create package.json:

nano package.json
{
    "name": "user-service",
    "version": "1.0.0",
    "description": "User management microservice",
    "main": "app.js",
    "scripts": {
        "start": "node app.js",
        "dev": "nodemon app.js"
    },
    "dependencies": {
        "express": "^4.18.2"
    },
    "keywords": ["microservice", "user", "docker"],
    "author": "Lab Student",
    "license": "MIT"
}
Create Dockerfile:

nano Dockerfile
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory in container
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --only=production

# Copy application code
COPY . .

# Expose port
EXPOSE 3001

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3001/health || exit 1

# Start the application
CMD ["npm", "start"]
Subtask 2.3: Build Product Service
Navigate to product service directory:

cd ../product-service
Create the main application file:

nano app.js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3002;

// Middleware
app.use(express.json());

// In-memory product storage
let products = [
    { id: 1, name: 'Laptop', price: 999.99, category: 'Electronics', stock: 50 },
    { id: 2, name: 'Smartphone', price: 699.99, category: 'Electronics', stock: 100 },
    { id: 3, name: 'Book', price: 19.99, category: 'Education', stock: 200 },
    { id: 4, name: 'Headphones', price: 149.99, category: 'Electronics', stock: 75 }
];

// Health check endpoint
app.get('/health', (req, res) => {
    res.json({ 
        service: 'product-service', 
        status: 'healthy', 
        timestamp: new Date().toISOString() 
    });
});

// Get all products
app.get('/products', (req, res) => {
    const { category, minPrice, maxPrice } = req.query;
    let filteredProducts = [...products];
    
    if (category) {
        filteredProducts = filteredProducts.filter(p => 
            p.category.toLowerCase() === category.toLowerCase()
        );
    }
    
    if (minPrice) {
        filteredProducts = filteredProducts.filter(p => p.price >= parseFloat(minPrice));
    }
    
    if (maxPrice) {
        filteredProducts = filteredProducts.filter(p => p.price <= parseFloat(maxPrice));
    }
    
    res.json({
        success: true,
        data: filteredProducts,
        count: filteredProducts.length
    });
});

// Get product by ID
app.get('/products/:id', (req, res) => {
    const productId = parseInt(req.params.id);
    const product = products.find(p => p.id === productId);
    
    if (!product) {
        return res.status(404).json({
            success: false,
            message: 'Product not found'
        });
    }
    
    res.json({
        success: true,
        data: product
    });
});

// Create new product
app.post('/products', (req, res) => {
    const { name, price, category, stock } = req.body;
    
    if (!name || !price || !category) {
        return res.status(400).json({
            success: false,
            message: 'Name, price, and category are required'
        });
    }
    
    const newProduct = {
        id: products.length + 1,
        name,
        price: parseFloat(price),
        category,
        stock: parseInt(stock) || 0
    };
    
    products.push(newProduct);
    
    res.status(201).json({
        success: true,
        data: newProduct,
        message: 'Product created successfully'
    });
});

// Update product stock
app.patch('/products/:id/stock', (req, res) => {
    const productId = parseInt(req.params.id);
    const { quantity } = req.body;
    
    const product = products.find(p => p.id === productId);
    
    if (!product) {
        return res.status(404).json({
            success: false,
            message: 'Product not found'
        });
    }
    
    if (quantity !== undefined) {
        product.stock = parseInt(quantity);
    }
    
    res.json({
        success: true,
        data: product,
        message: 'Stock updated successfully'
    });
});

// Start server
app.listen(PORT, '0.0.0.0', () => {
    console.log(`Product Service running on port ${PORT}`);
    console.log(`Health check: http://localhost:${PORT}/health`);
});
Create package.json:

nano package.json
{
    "name": "product-service",
    "version": "1.0.0",
    "description": "Product management microservice",
    "main": "app.js",
    "scripts": {
        "start": "node app.js",
        "dev": "nodemon app.js"
    },
    "dependencies": {
        "express": "^4.18.2"
    },
    "keywords": ["microservice", "product", "docker"],
    "author": "Lab Student",
    "license": "MIT"
}
Create Dockerfile:

nano Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --only=production

COPY . .

EXPOSE 3002

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3002/health || exit 1

CMD ["npm", "start"]
Subtask 2.4: Build Order Service
Navigate to order service directory:

cd ../order-service
Create the main application file:

nano app.js
const express = require('express');
const axios = require('axios');
const app = express();
const PORT = process.env.PORT || 3003;

// Service URLs (will be resolved by Docker networking)
const USER_SERVICE_URL = process.env.USER_SERVICE_URL || 'http://user-service:3001';
const PRODUCT_SERVICE_URL = process.env.PRODUCT_SERVICE_URL || 'http://product-service:3002';

// Middleware
app.use(express.json());

// In-memory order storage
let orders = [
    {
        id: 1,
        userId: 1,
        products: [
            { productId: 1, quantity: 1, price: 999.99 }
        ],
        total: 999.99,
        status: 'completed',
        createdAt: new Date('2024-01-15').toISOString()
    }
];

// Health check endpoint
app.get('/health', (req, res) => {
    res.json({ 
        service: 'order-service', 
        status: 'healthy', 
        timestamp: new Date().toISOString() 
    });
});

// Get all orders
app.get('/orders', (req, res) => {
    res.json({
        success: true,
        data: orders,
        count: orders.length
    });
});

// Get orders by user ID
app.get('/orders/user/:userId', (req, res) => {
    const userId = parseInt(req.params.userId);
    const userOrders = orders.filter(o => o.userId === userId);
    
    res.json({
        success: true,
        data: userOrders,
        count: userOrders.length
    });
});

// Get order by ID
app.get('/orders/:id', (req, res) => {
    const orderId = parseInt(req.params.id);
    const order = orders.find(o => o.id === orderId);
    
    if (!order) {
        return res.status(404).json({
            success: false,
            message: 'Order not found'
        });
    }
    
    res.json({
        success: true,
        data: order
    });
});

// Create new order
app.post('/orders', async (req, res) => {
    try {
        const { userId, products } = req.body;
        
        if (!userId || !products || !Array.isArray(products) || products.length === 0) {
            return res.status(400).json({
                success: false,
                message: 'User ID and products array are required'
            });
        }
        
        // Verify user exists
        try {
            const userResponse = await axios.get(`${USER_SERVICE_URL}/users/${userId}`);
            if (!userResponse.data.success) {
                return res.status(404).json({
                    success: false,
                    message: 'User not found'
                });
            }
        } catch (error) {
            console.log('User service unavailable, proceeding without verification');
        }
        
        // Calculate total and verify products
        let total = 0;
        const orderProducts = [];
        
        for (const item of products) {
            try {
                const productResponse = await axios.get(`${PRODUCT_SERVICE_URL}/products/${item.productId}`);
                if (productResponse.data.success) {
                    const product = productResponse.data.data;
                    const itemTotal = product.price * item.quantity;
                    total += itemTotal;
                    
                    orderProducts.push({
                        productId: item.productId,
                        quantity: item.quantity,
                        price: product.price,
                        name: product.name
                    });
                }
            } catch (error) {
                console.log(`Product ${item.productId} service unavailable, using provided data`);
                orderProducts.push({
                    productId: item.productId,
                    quantity: item.quantity,
                    price: item.price || 0
                });
                total += (item.price || 0) * item.quantity;
            }
        }
        
        const newOrder = {
            id: orders.length + 1,
            userId,
            products: orderProducts,
            total: Math.round(total * 100) / 100, // Round to 2 decimal places
            status: 'pending',
            createdAt: new Date().toISOString()
        };
        
        orders.push(newOrder);
        
        res.status(201).json({
            success: true,
            data: newOrder,
            message: 'Order created successfully'
        });
        
    } catch (error) {
        console.error('Error creating order:', error.message);
        res.status(500).json({
            success: false,
            message: 'Internal server error'
        });
    }
});

// Update order status
app.patch('/orders/:id/status', (req, res) => {
    const orderId = parseInt(req.params.id);
    const { status } = req.body;
    
    const order = orders.find(o => o.id === orderId);
    
    if (!order) {
        return res.status(404).json({
            success: false,
            message: 'Order not found'
        });
    }
    
    const validStatuses = ['pending', 'processing', 'shipped', 'delivered', 'cancelled'];
    if (!validStatuses.includes(status)) {
        return res.status(400).json({
            success: false,
            message: 'Invalid status. Valid statuses: ' + validStatuses.join(', ')
        });
    }
    
    order.status = status;
    order.updatedAt = new Date().toISOString();
    
    res.json({
        success: true,
        data: order,
        message: 'Order status updated successfully'
    });
});

// Start server
app.listen(PORT, '0.0.0.0', () => {
    console.log(`Order Service running on port ${PORT}`);
    console.log(`Health check: http://localhost:${PORT}/health`);
    console.log(`User Service URL: ${USER_SERVICE_URL}`);
    console.log(`Product Service URL: ${PRODUCT_SERVICE_URL}`);
});
Create package.json:

nano package.json
{
    "name": "order-service",
    "version": "1.0.0",
    "description": "Order management microservice",
    "main": "app.js",
    "scripts": {
        "start": "node app.js",
        "dev": "nodemon app.js"
    },
    "dependencies": {
        "express": "^4.18.2",
        "axios": "^1.6.0"
    },
    "keywords": ["microservice", "order", "docker"],
    "author": "Lab Student",
    "license": "MIT"
}
Create Dockerfile:

nano Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --only=production

COPY . .

EXPOSE 3003

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:3003/health || exit 1

CMD ["npm", "start"]
Task 3: Setting Up Docker Networking and Communication
Subtask 3.1: Understanding Docker Networks
Docker networks allow containers to communicate with each other securely. We'll create a custom network for our microservices.

Subtask 3.2: Create Docker Network
cd ..
docker network create microservices-network
Verify the network was created:

docker network ls
Subtask 3.3: Build All Service Images
Build each service image:

# Build User Service
cd user-service
docker build -t user-service:latest .

# Build Product Service
cd ../product-service
docker build -t product-service:latest .

# Build Order Service
cd ../order-service
docker build -t order-service:latest .

cd ..
Verify images were built:

docker images | grep -E "(user-service|product-service|order-service)"
Task 4: Deploy Microservices Using Docker Compose
Subtask 4.1: Create Docker Compose Configuration
Navigate to the docker-compose-files directory:

cd docker-compose-files
Create the main docker-compose.yml file:

nano docker-compose.yml
version: '3.8'

services:
  # User Service
  user-service:
    build:
      context: ../user-service
      dockerfile: Dockerfile
    container_name: user-service
    ports:
      - "3001:3001"
    networks:
      - microservices-network
    environment:
      - NODE_ENV=production
      - PORT=3001
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    labels:
      - "service.name=user-service"
      - "service.version=1.0.0"

  # Product Service
  product-service:
    build:
      context: ../product-service
      dockerfile: Dockerfile
    container_name: product-service
    ports:
      - "3002:3002"
    networks:
      - microservices-network
    environment:
      - NODE_ENV=production
      - PORT=3002
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3002/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    labels:
      - "service.name=product-service"
      - "service.version=1.0.0"

  # Order Service
  order-service:
    build:
      context: ../order-service
      dockerfile: Dockerfile
    container_name: order-service
    ports:
      - "3003:3003"
    networks:
      - microservices-network
    environment:
      - NODE_ENV=production
      - PORT=3003
      - USER_SERVICE_URL=http://user-service:3001
      - PRODUCT_SERVICE_URL=http://product-service:3002
    depends_on:
      - user-service
      - product-service
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3003/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    labels:
      - "service.name=order-service"
      - "service.version=1.0.0"

networks:
  microservices-network:
    driver: bridge
    name: microservices-network
Subtask 4.2: Deploy the Microservices
Start all services using Docker Compose:

docker-compose up -d
Check the status of all services:

docker-compose ps
View logs from all services:

docker-compose logs -f
To view logs from a specific service:

docker-compose logs -f user-service
Subtask 4.3: Test Service Communication
Test each service individually:

# Test User Service
curl -X GET http://localhost:3001/health
curl -X GET http://localhost:3001/users

# Test Product Service
curl -X GET http://localhost:3002/health
curl -X GET http://localhost:3002/products

# Test Order Service
curl -X GET http://localhost:3003/health
curl -X GET http://localhost:3003/orders
Test inter-service communication by creating an order:

curl -X POST http://localhost:3003/orders \
  -H "Content-Type: application/json" \
  -d '{
    "userId": 1,
    "products": [
      {
        "productId": 1,
        "quantity": 2
      },
      {
        "productId": 2,
        "quantity": 1
      }
    ]
  }'
Task 5: Scale Microservices and Monitor Performance
Subtask 5.1: Scale Services Horizontally
Scale the product service to handle more load:

docker-compose up -d --scale product-service=3
Check the scaled services:

docker-compose ps
Subtask 5.2: Create Load Balancer Configuration
Create a simple load balancer using nginx:

cd ..
mkdir nginx-lb
cd nginx-lb
Create nginx configuration:

nano nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream product-service {
        server product-service:3002;
    }
    
    upstream user-service {
        server user-service:3001;
    }
    
    upstream order-service {
        server order-service:3003;
    }

    server {
        listen 80;
        
        location /users {
            proxy_pass http://user-service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /products {
            proxy_pass http://product-service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /orders {
            proxy_pass http://order-service;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            return 200 'Load Balancer is healthy\n';
            add_header Content-Type text/plain;
        }
    }
}
Create Dockerfile for nginx:

nano Dockerfile
FROM nginx:alpine
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
Subtask 5.3: Monitor Service Performance
Create a monitoring script:

cd ..
nano monitor-services.sh
#!/bin/bash

echo "=== Microservices Health Check ==="
echo "Timestamp: $(date)"
echo "=================================="

services=("user-service:3001" "product-service:3002" "order-service:3003")

for service in "${services[@]}"; do
    name=$(echo $service | cut -d':' -f1)
    port=$(echo $service | cut -d':' -f2)
    
    echo -n "Checking $name... "
    
    if curl -s -f "http://localhost:$port/health" > /dev/null; then
        echo "✓ Healthy"
        # Get response time
        response_time=$(curl -s -w "%{time_total}" -o /dev/null "http://localhost:$port/health")
        echo "  Response time: ${response_time}s"
    else
        echo "✗ Unhealthy"
    fi
    echo
done

echo "=== Container Resource Usage ==="
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"

echo -e "\n=== Service Logs (Last 5 lines) ==="
for service in user-service product-service order-service; do
    echo "--- $service ---"
    docker logs --tail 5 $service 2>/dev/null || echo "Service not running"
    echo
done
Make the script executable:

chmod +x monitor-services.sh
Run the monitoring script:

./monitor-services.sh
Subtask 5.4: Performance Testing
Create a simple load testing script:

nano load-test.sh
#!/bin/bash

echo "Starting load test..."
echo "Testing User Service..."

# Test user service with multiple concurrent requests
for i in {1..10}; do
    curl -s "http://localhost:3001/users" > /dev/null &
done

echo "Testing Product Service..."

# Test product service with multiple concurrent requests
for i in {1..10}; do
    curl -s "http://localhost:3002/products" > /dev/null &
done

echo "Testing Order Service..."

# Test order creation
for i in {1..5}; do
    curl -s -X POST "http://localhost:3003/orders" \
        -H "Content-Type: application/json" \
        -d "{\"userId\": $i, \"products\": [{\"productId\": 1, \"quantity\": 1}]}" > /dev/null &
done

wait
echo "Load test completed!"

# Check service health after load test
./monitor-services.sh
Make executable and run:

chmod +x load-test.sh
./load-test.sh
Task 6: Integrate API Gateway
Subtask 6.1: Build API Gateway Service
Navigate to the api-gateway directory:

cd api-gateway
Create the API Gateway application:

nano app.js
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');
const rateLimit = require('express-rate-limit');
const app = express();
const PORT = process.env.PORT || 3000;

// Rate limiting
const limiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 minutes
    max: 100, // limit each IP to 100 requests per windowMs
    message: {
        error: 'Too many requests from this IP, please try again later.'
    }
});

// Apply rate limiting to all requests
app.use(limiter);

// Middleware for logging
app.use((req, res, next) => {
    console.log(`${new Date().toISOString()} - ${req.method} ${req.path} - IP: ${req.ip}`);
    next();
});

// Health check for API Gateway
app.get('/health', (req, res) => {
    res.json({
        service: 'api-gateway',
        status: 'healthy',
        timestamp: new Date().toISOString(),
        uptime: process.uptime()
    });
});

// API Gateway info
app.get('/', (req, res) => {
    res.json({
        message: 'Microservices API Gateway',
        version: '1.0.0',
        services: {
            users: '/api/users',
            products: '/api/products',
            orders: '/api/orders'
        },
        documentation: {
            health: '/health',
            users: '/api/users - User management service',
            products: '/api/products - Product catalog service',
            orders: '/api/orders - Order processing service'
        }
    });
});

// Service URLs
const services = {
    user: process.env.USER_SERVICE_URL || 'http://user-service:3001',
    product: process.env.PRODUCT_SERVICE_URL || 'http://product-service:3002',
    order: process.env.ORDER_SERVICE_URL || 'http://order-service:3003'
};

// Proxy configuration options
const proxyOptions = {
    changeOrigin: true,
    timeout: 10000,
    proxyTimeout: 10000,
    onError: (err, req, res) => {
        console.error('Proxy error:', err.message);
        res.status(503).json({
            error: 'Service temporarily unavailable',
            message: 'The requested service is currently unavailable. Please try again later.',
            timestamp: new Date().toISOString()
        });
    },
    onProxyReq: (proxyReq, req, res) => {
        console.log(`Proxying ${req.method} ${req.path} to ${proxyReq.path}`);
    }
};

// User Service Proxy
app.use('/api/users', createProxyMiddleware({
    target: services.user,
    pathRewrite: {
        '^/api/users': '/users'
    },
    ...proxyOptions
}));

// Product Service Proxy
app.use('/api/products', createProxyMiddleware({
    target: services.product,
    pathRewrite: {
        '^/api/products': '/products'
    },
    ...proxyOptions
}));

// Order Service Proxy
app.use('/api/orders', createProxyMiddleware({
    target: services.order,
    pathRewrite
