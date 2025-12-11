Lab 57: Advanced Docker Compose - Handling Dependencies in Multi-Service Applications
Objectives
By the end of this lab, you will be able to:

Define multi-service dependencies in docker-compose.yml files
Configure service startup order and dependency links between containers
Implement wait-for-it scripts to ensure containers start in the correct sequence
Test failure scenarios and implement proper service restart mechanisms
Use environment variables for effective configuration management in multi-service applications
Troubleshoot common dependency issues in Docker Compose environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and images
Familiarity with Docker Compose fundamentals
Knowledge of YAML syntax
Basic command-line interface skills
Understanding of web applications and databases
Lab Environment
Al Nafi provides you with a ready-to-use Linux-based cloud machine. Simply click Start Lab to access your pre-configured environment. No need to build your own VM or install Docker - everything is ready for you to begin immediately.

Lab Overview
In this lab, you will create a multi-service application consisting of:

A web application (Node.js)
A database (PostgreSQL)
A cache layer (Redis)
A reverse proxy (Nginx)
You'll learn how to manage dependencies between these services to ensure they start in the correct order and handle failures gracefully.

Task 1: Define Multi-Service Dependencies in docker-compose.yml
Step 1.1: Create the Project Directory Structure
First, let's create a well-organized project structure for our multi-service application.

mkdir advanced-compose-lab
cd advanced-compose-lab
mkdir -p app nginx wait-for-it
Step 1.2: Create the Basic docker-compose.yml File
Create the main Docker Compose file that defines all our services and their dependencies:

nano docker-compose.yml
Add the following content:

version: '3.8'

services:
  # Database service - foundational service with no dependencies
  postgres:
    image: postgres:15-alpine
    container_name: postgres_db
    environment:
      POSTGRES_DB: ${DB_NAME:-myapp}
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-password123}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Cache service - independent service
  redis:
    image: redis:7-alpine
    container_name: redis_cache
    ports:
      - "6379:6379"
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3

  # Web application - depends on both database and cache
  webapp:
    build: ./app
    container_name: web_app
    environment:
      DB_HOST: postgres
      DB_NAME: ${DB_NAME:-myapp}
      DB_USER: ${DB_USER:-postgres}
      DB_PASSWORD: ${DB_PASSWORD:-password123}
      REDIS_HOST: redis
      NODE_ENV: production
    ports:
      - "3000:3000"
    networks:
      - app-network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  # Reverse proxy - depends on web application
  nginx:
    image: nginx:alpine
    container_name: nginx_proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    networks:
      - app-network
    depends_on:
      webapp:
        condition: service_healthy
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
Step 1.3: Create Environment Variables File
Create a .env file to manage configuration:

nano .env
Add the following content:

# Database Configuration
DB_NAME=myapp
DB_USER=postgres
DB_PASSWORD=securepassword123

# Application Configuration
NODE_ENV=production
APP_PORT=3000

# Redis Configuration
REDIS_PORT=6379
Task 2: Configure Service Startup Order and Dependency Links
Step 2.1: Create Database Initialization Script
Create an SQL script to initialize the database:

nano init.sql
Add the following content:

-- Create users table
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create posts table
CREATE TABLE IF NOT EXISTS posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    content TEXT,
    user_id INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO users (username, email) VALUES 
    ('john_doe', 'john@example.com'),
    ('jane_smith', 'jane@example.com')
ON CONFLICT (username) DO NOTHING;

INSERT INTO posts (title, content, user_id) VALUES 
    ('First Post', 'This is the first post content', 1),
    ('Second Post', 'This is the second post content', 2)
ON CONFLICT DO NOTHING;
Step 2.2: Create the Web Application
Create the Node.js application directory and files:

cd app
nano package.json
Add the following content:

{
  "name": "multi-service-app",
  "version": "1.0.0",
  "description": "Multi-service application with dependencies",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0",
    "redis": "^4.6.7",
    "cors": "^2.8.5"
  }
}
Create the main application file:

nano server.js
Add the following content:

const express = require('express');
const { Client } = require('pg');
const redis = require('redis');
const cors = require('cors');

const app = express();
const PORT = process.env.APP_PORT || 3000;

// Middleware
app.use(cors());
app.use(express.json());

// Database configuration
const dbConfig = {
  host: process.env.DB_HOST || 'localhost',
  database: process.env.DB_NAME || 'myapp',
  user: process.env.DB_USER || 'postgres',
  password: process.env.DB_PASSWORD || 'password123',
  port: 5432,
};

// Redis configuration
const redisClient = redis.createClient({
  host: process.env.REDIS_HOST || 'localhost',
  port: process.env.REDIS_PORT || 6379,
});

// Initialize connections
let pgClient;
let isDbConnected = false;
let isRedisConnected = false;

async function initializeConnections() {
  try {
    // Initialize PostgreSQL connection
    pgClient = new Client(dbConfig);
    await pgClient.connect();
    isDbConnected = true;
    console.log('Connected to PostgreSQL database');

    // Initialize Redis connection
    await redisClient.connect();
    isRedisConnected = true;
    console.log('Connected to Redis cache');

  } catch (error) {
    console.error('Error initializing connections:', error);
    process.exit(1);
  }
}

// Health check endpoint
app.get('/health', (req, res) => {
  const health = {
    status: 'OK',
    timestamp: new Date().toISOString(),
    services: {
      database: isDbConnected ? 'connected' : 'disconnected',
      cache: isRedisConnected ? 'connected' : 'disconnected'
    }
  };
  
  if (isDbConnected && isRedisConnected) {
    res.status(200).json(health);
  } else {
    res.status(503).json(health);
  }
});

// Get all users
app.get('/api/users', async (req, res) => {
  try {
    // Check cache first
    const cacheKey = 'users:all';
    const cachedUsers = await redisClient.get(cacheKey);
    
    if (cachedUsers) {
      console.log('Returning cached users');
      return res.json(JSON.parse(cachedUsers));
    }

    // Query database
    const result = await pgClient.query('SELECT * FROM users ORDER BY created_at DESC');
    const users = result.rows;

    // Cache the result for 5 minutes
    await redisClient.setEx(cacheKey, 300, JSON.stringify(users));
    
    res.json(users);
  } catch (error) {
    console.error('Error fetching users:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Get all posts with user information
app.get('/api/posts', async (req, res) => {
  try {
    const cacheKey = 'posts:all';
    const cachedPosts = await redisClient.get(cacheKey);
    
    if (cachedPosts) {
      console.log('Returning cached posts');
      return res.json(JSON.parse(cachedPosts));
    }

    const query = `
      SELECT p.id, p.title, p.content, p.created_at, u.username, u.email
      FROM posts p
      JOIN users u ON p.user_id = u.id
      ORDER BY p.created_at DESC
    `;
    
    const result = await pgClient.query(query);
    const posts = result.rows;

    // Cache the result for 5 minutes
    await redisClient.setEx(cacheKey, 300, JSON.stringify(posts));
    
    res.json(posts);
  } catch (error) {
    console.error('Error fetching posts:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Create a new user
app.post('/api/users', async (req, res) => {
  try {
    const { username, email } = req.body;
    
    if (!username || !email) {
      return res.status(400).json({ error: 'Username and email are required' });
    }

    const query = 'INSERT INTO users (username, email) VALUES ($1, $2) RETURNING *';
    const result = await pgClient.query(query, [username, email]);
    
    // Invalidate cache
    await redisClient.del('users:all');
    
    res.status(201).json(result.rows[0]);
  } catch (error) {
    console.error('Error creating user:', error);
    if (error.code === '23505') {
      res.status(409).json({ error: 'Username or email already exists' });
    } else {
      res.status(500).json({ error: 'Internal server error' });
    }
  }
});

// Root endpoint
app.get('/', (req, res) => {
  res.json({
    message: 'Multi-Service Application API',
    version: '1.0.0',
    endpoints: {
      health: '/health',
      users: '/api/users',
      posts: '/api/posts'
    }
  });
});

// Error handling middleware
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: 'Something went wrong!' });
});

// Graceful shutdown
process.on('SIGTERM', async () => {
  console.log('Received SIGTERM, shutting down gracefully');
  
  if (pgClient) {
    await pgClient.end();
  }
  
  if (redisClient) {
    await redisClient.quit();
  }
  
  process.exit(0);
});

// Start server
async function startServer() {
  await initializeConnections();
  
  app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server running on port ${PORT}`);
    console.log(`Health check available at http://localhost:${PORT}/health`);
  });
}

startServer().catch(console.error);
Create the Dockerfile for the web application:

nano Dockerfile
Add the following content:

FROM node:18-alpine

# Install curl for health checks
RUN apk add --no-cache curl

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Change ownership of the app directory
RUN chown -R nodejs:nodejs /app
USER nodejs

EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["npm", "start"]
Return to the main directory:

cd ..
Step 2.3: Create Nginx Configuration
Create the Nginx configuration:

nano nginx/nginx.conf
Add the following content:

events {
    worker_connections 1024;
}

http {
    upstream webapp {
        server webapp:3000;
    }

    server {
        listen 80;
        server_name localhost;

        # Health check endpoint
        location /nginx-health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }

        # Proxy all requests to the web application
        location / {
            proxy_pass http://webapp;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeout settings
            proxy_connect_timeout 30s;
            proxy_send_timeout 30s;
            proxy_read_timeout 30s;
        }

        # Static content caching
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            proxy_pass http://webapp;
            proxy_cache_valid 200 1h;
            add_header Cache-Control "public, immutable";
        }
    }
}
Task 3: Set Up Wait-for-it Scripts to Ensure Containers Start in the Correct Order
Step 3.1: Download and Configure Wait-for-it Script
Download the wait-for-it script:

cd wait-for-it
curl -o wait-for-it.sh https://raw.githubusercontent.com/vishnubob/wait-for-it/master/wait-for-it.sh
chmod +x wait-for-it.sh
cd ..
Step 3.2: Create Enhanced docker-compose.yml with Wait Scripts
Update the docker-compose.yml file to include wait-for-it scripts:

nano docker-compose.yml
Replace the existing content with this enhanced version:

version: '3.8'

services:
  # Database service - foundational service
  postgres:
    image: postgres:15-alpine
    container_name: postgres_db
    environment:
      POSTGRES_DB: ${DB_NAME:-myapp}
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-password123}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-postgres}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # Cache service
  redis:
    image: redis:7-alpine
    container_name: redis_cache
    ports:
      - "6379:6379"
    networks:
      - app-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
      start_period: 10s

  # Web application with wait-for-it
  webapp:
    build: ./app
    container_name: web_app
    environment:
      DB_HOST: postgres
      DB_NAME: ${DB_NAME:-myapp}
      DB_USER: ${DB_USER:-postgres}
      DB_PASSWORD: ${DB_PASSWORD:-password123}
      REDIS_HOST: redis
      NODE_ENV: production
    ports:
      - "3000:3000"
    volumes:
      - ./wait-for-it:/wait-for-it
    networks:
      - app-network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    command: ["/wait-for-it/wait-for-it.sh", "postgres:5432", "--", "/wait-for-it/wait-for-it.sh", "redis:6379", "--", "npm", "start"]
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Reverse proxy
  nginx:
    image: nginx:alpine
    container_name: nginx_proxy
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./wait-for-it:/wait-for-it
    networks:
      - app-network
    depends_on:
      webapp:
        condition: service_healthy
    restart: unless-stopped
    command: ["/wait-for-it/wait-for-it.sh", "webapp:3000", "--", "nginx", "-g", "daemon off;"]

volumes:
  postgres_data:

networks:
  app-network:
    driver: bridge
Step 3.3: Create a Custom Wait Script for Complex Dependencies
Create a more sophisticated wait script for complex scenarios:

nano wait-for-services.sh
Add the following content:

#!/bin/bash

# wait-for-services.sh - Advanced service dependency checker

set -e

# Configuration
MAX_ATTEMPTS=30
SLEEP_INTERVAL=2

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Logging function
log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}"
}

warn() {
    echo -e "${YELLOW}[$(date +'%Y-%m-%d %H:%M:%S')] WARNING: $1${NC}"
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERROR: $1${NC}"
}

# Function to check if a service is ready
check_service() {
    local service_name=$1
    local host=$2
    local port=$3
    local endpoint=${4:-""}
    
    log "Checking $service_name at $host:$port"
    
    for i in $(seq 1 $MAX_ATTEMPTS); do
        if nc -z "$host" "$port" 2>/dev/null; then
            if [ -n "$endpoint" ]; then
                # Additional HTTP health check
                if curl -f "http://$host:$port$endpoint" >/dev/null 2>&1; then
                    log "$service_name is ready (attempt $i/$MAX_ATTEMPTS)"
                    return 0
                else
                    warn "$service_name port is open but health check failed (attempt $i/$MAX_ATTEMPTS)"
                fi
            else
                log "$service_name is ready (attempt $i/$MAX_ATTEMPTS)"
                return 0
            fi
        else
            warn "$service_name is not ready (attempt $i/$MAX_ATTEMPTS)"
        fi
        
        if [ $i -lt $MAX_ATTEMPTS ]; then
            sleep $SLEEP_INTERVAL
        fi
    done
    
    error "$service_name failed to become ready after $MAX_ATTEMPTS attempts"
    return 1
}

# Function to check database readiness
check_database() {
    local host=$1
    local port=$2
    local user=$3
    local database=$4
    
    log "Checking database connectivity"
    
    for i in $(seq 1 $MAX_ATTEMPTS); do
        if PGPASSWORD="$DB_PASSWORD" psql -h "$host" -p "$port" -U "$user" -d "$database" -c "SELECT 1;" >/dev/null 2>&1; then
            log "Database is ready and accepting connections (attempt $i/$MAX_ATTEMPTS)"
            return 0
        else
            warn "Database is not ready (attempt $i/$MAX_ATTEMPTS)"
        fi
        
        if [ $i -lt $MAX_ATTEMPTS ]; then
            sleep $SLEEP_INTERVAL
        fi
    done
    
    error "Database failed to become ready after $MAX_ATTEMPTS attempts"
    return 1
}

# Function to check Redis readiness
check_redis() {
    local host=$1
    local port=$2
    
    log "Checking Redis connectivity"
    
    for i in $(seq 1 $MAX_ATTEMPTS); do
        if redis-cli -h "$host" -p "$port" ping | grep -q "PONG"; then
            log "Redis is ready (attempt $i/$MAX_ATTEMPTS)"
            return 0
        else
            warn "Redis is not ready (attempt $i/$MAX_ATTEMPTS)"
        fi
        
        if [ $i -lt $MAX_ATTEMPTS ]; then
            sleep $SLEEP_INTERVAL
        fi
    done
    
    error "Redis failed to become ready after $MAX_ATTEMPTS attempts"
    return 1
}

# Main execution
main() {
    log "Starting service dependency checks..."
    
    # Check all required services based on arguments
    case "$1" in
        "webapp")
            check_service "PostgreSQL" "postgres" "5432"
            check_database "postgres" "5432" "$DB_USER" "$DB_NAME"
            check_service "Redis" "redis" "6379"
            check_redis "redis" "6379"
            ;;
        "nginx")
            check_service "Web Application" "webapp" "3000" "/health"
            ;;
        *)
            error "Unknown service type: $1"
            echo "Usage: $0 {webapp|nginx}"
            exit 1
            ;;
    esac
    
    log "All dependency checks passed! Starting $1..."
    
    # Execute the original command
    shift
    exec "$@"
}

# Install required tools if not present
if ! command -v nc &> /dev/null; then
    log "Installing netcat..."
    apk add --no-cache netcat-openbsd
fi

if ! command -v curl &> /dev/null; then
    log "Installing curl..."
    apk add --no-cache curl
fi

# Run main function with all arguments
main "$@"
Make the script executable:

chmod +x wait-for-services.sh
Task 4: Test Failure Scenarios and Ensure Proper Service Restarts
Step 4.1: Create a Testing Script
Create a comprehensive testing script:

nano test-dependencies.sh
Add the following content:

#!/bin/bash

# test-dependencies.sh - Comprehensive dependency testing script

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')] $1${NC}"
}

warn() {
    echo -e "${YELLOW}[$(date +'%Y-%m-%d %H:%M:%S')] WARNING: $1${NC}"
}

error() {
    echo -e "${RED}[$(date +'%Y-%m-%d %H:%M:%S')] ERROR: $1${NC}"
}

info() {
    echo -e "${BLUE}[$(date +'%Y-%m-%d %H:%M:%S')] INFO: $1${NC}"
}

# Function to wait for service to be ready
wait_for_service() {
    local service_name=$1
    local url=$2
    local max_attempts=30
    
    log "Waiting for $service_name to be ready..."
    
    for i in $(seq 1 $max_attempts); do
        if curl -f "$url" >/dev/null 2>&1; then
            log "$service_name is ready!"
            return 0
        fi
        sleep 2
    done
    
    error "$service_name failed to become ready"
    return 1
}

# Function to test API endpoints
test_api_endpoints() {
    log "Testing API endpoints..."
    
    # Test root endpoint
    info "Testing root endpoint..."
    curl -s http://localhost/
    echo
    
    # Test health endpoint
    info "Testing health endpoint..."
    curl -s http://localhost/health | jq .
    echo
    
    # Test users endpoint
    info "Testing users endpoint..."
    curl -s http://localhost/api/users | jq .
    echo
    
    # Test posts endpoint
    info "Testing posts endpoint..."
    curl -s http://localhost/api/posts | jq .
    echo
    
    # Test creating a new user
    info "Testing user creation..."
    curl -s -X POST http://localhost/api/users \
        -H "Content-Type: application/json" \
        -d '{"username":"testuser","email":"test@example.com"}' | jq .
    echo
}

# Function to simulate database failure
test_database_failure() {
    log "Testing database failure scenario..."
    
    # Stop database
    info "Stopping PostgreSQL container..."
    docker-compose stop postgres
    
    # Wait a moment
    sleep 5
    
    # Test API response during database failure
    info "Testing API during database failure..."
    curl -s http://localhost/health | jq .
    echo
    
    # Restart database
    info "Restarting PostgreSQL container..."
    docker-compose start postgres
    
    # Wait for database to be ready
    wait_for_service "Database" "http://localhost/health"
    
    # Test API after database recovery
    info "Testing API after database recovery..."
    curl -s http://localhost/api/users | jq .
    echo
}

# Function to simulate Redis failure
test_redis_failure() {
    log "Testing Redis failure scenario..."
    
    # Stop Redis
    info "Stopping Redis container..."
    docker-compose stop redis
    
    # Wait a moment
    sleep 5
    
    # Test API response during Redis failure
    info "Testing API during Redis failure..."
    curl -s http://localhost/api/users
    echo
    
    # Restart Redis
    info "Restarting Redis container..."
    docker-compose start redis
    
    # Wait for Redis to be ready
    wait_for_service "Redis" "http://localhost/health"
    
    # Test API after Redis recovery
    info "Testing API after Redis recovery..."
    curl -s http://localhost/api/users | jq .
    echo
}

# Function to simulate webapp failure
test_webapp_failure() {
    log "Testing webapp failure scenario..."
    
    # Stop webapp
    info "Stopping webapp container..."
    docker-compose stop webapp
    
    # Wait a moment
    sleep 5
    
    # Test Nginx response during webapp failure
    info "Testing Nginx during webapp failure..."
    curl -s -w "%{http_code}" http://localhost/ || true
    echo
    
    # Restart webapp
    info "Restarting webapp container..."
    docker-compose start webapp
    
    # Wait for webapp to be ready
    wait_for_service "Web Application" "http://localhost/health"
    
    # Test API after webapp recovery
    info "Testing API after webapp recovery..."
    curl -s http://localhost/ | jq .
    echo
}

# Function to test complete restart
test_complete_restart() {
    log "Testing complete application restart..."
    
    # Stop all services
    info "Stopping all services..."
    docker-compose down
    
    # Wait a moment
    sleep 5
    
    # Start all services
    info "Starting all services..."
    docker-compose up -d
    
    # Wait for all services to be ready
    wait_for_service "Application" "http://localhost/health"
    
    # Test all endpoints
    test_api_endpoints
}

# Function to check service dependencies
check_service_dependencies() {
    log "Checking service dependencies..."
    
    # Check container status
    info "Container status:"
    docker-compose ps
    echo
    
    # Check service logs
    info "Recent logs from webapp:"
    docker-compose logs --tail=10 webapp
    echo
    
    # Check health status
    info "Health check status:"
    curl -s http://localhost/health | jq .
    echo
}

# Main test execution
main() {
    log "Starting comprehensive dependency testing..."
    
    # Ensure application is running
    if ! curl -f http://localhost/health >/dev/null 2>&1; then
        warn "Application not running, starting it..."
        docker-compose up -d
        wait_for_service "Application" "http://localhost/health"
    fi
    
    # Run all tests
    check_service_dependencies
    test_api_endpoints
    test_database_failure
    test_redis_failure
    test_webapp_failure
    test_complete_restart
    
    log "All dependency tests completed successfully!"
}

# Check if jq is installed
if ! command -v jq &> /dev/null; then
    warn "jq not found, installing..."
    sudo apt-get update && sudo apt-get install -y jq
fi

# Run main function
main "$@"
Make the script executable:

chmod +x test-dependencies.sh
Step 4.2: Create Service Monitoring Script
Create a script to monitor service health:

nano monitor-services.sh
Add the following content:

#!/bin/bash

# monitor-services.sh - Real-time service monitoring

# Colors
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m'

# Function to check service health
check_service_health() {
    local service=$1
    local url=$2
    
    if curl -f "$url" >/dev/null 2>&1; then
        echo -e "${GREEN}✓${NC} $service"
        return 0
    else
        echo -e "${RED}✗${NC} $service"
        return 1
    fi
}

# Function to display service status
display_status() {
    clear
    echo "=== Multi-Service Application Health Monitor ==="
    echo "Last updated: $(date)"
    echo "
