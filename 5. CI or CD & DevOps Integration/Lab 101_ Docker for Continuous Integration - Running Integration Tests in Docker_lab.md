Lab 101: Docker for Continuous Integration - Running Integration Tests in Docker
Lab Objectives
By the end of this lab, you will be able to:

Create Docker containers with environments suitable for running integration tests
Write and execute integration tests for multi-container applications
Set up CI/CD pipelines using Jenkins to run tests in Docker containers
Monitor and analyze test results from containerized environments
Automate integration testing processes with scheduled pipeline executions
Understand best practices for Docker-based continuous integration workflows
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with command-line interface operations
Basic knowledge of web applications and databases
Understanding of testing concepts (unit tests vs integration tests)
Basic familiarity with CI/CD concepts
Note: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install Docker locally.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-installed with:

Docker Engine
Docker Compose
Git
Node.js and npm
Jenkins
Text editors (nano, vim)
Task 1: Create Docker Environment for Integration Tests
Subtask 1.1: Set up the Project Structure
First, let's create a directory structure for our integration testing project.

# Create the main project directory
mkdir docker-ci-lab
cd docker-ci-lab

# Create subdirectories for our application components
mkdir -p app tests docker-configs jenkins

# Create the basic file structure
touch app/package.json app/server.js app/Dockerfile
touch tests/integration.test.js tests/package.json tests/Dockerfile
touch docker-compose.yml docker-compose.test.yml
touch jenkins/Jenkinsfile
Subtask 1.2: Create a Simple Web Application
Let's create a basic Node.js web application that connects to a database.

Create the application's package.json:

cat > app/package.json << 'EOF'
{
  "name": "docker-ci-app",
  "version": "1.0.0",
  "description": "Sample app for Docker CI testing",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0",
    "cors": "^2.8.5"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
EOF
Create the main application file server.js:

cat > app/server.js << 'EOF'
const express = require('express');
const { Client } = require('pg');
const cors = require('cors');

const app = express();
const port = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(express.json());

// Database configuration
const dbConfig = {
  host: process.env.DB_HOST || 'localhost',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || 'testdb',
  user: process.env.DB_USER || 'testuser',
  password: process.env.DB_PASSWORD || 'testpass',
};

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

// Database connection test endpoint
app.get('/db-status', async (req, res) => {
  const client = new Client(dbConfig);
  
  try {
    await client.connect();
    const result = await client.query('SELECT NOW() as current_time');
    await client.end();
    
    res.json({ 
      status: 'connected', 
      timestamp: result.rows[0].current_time 
    });
  } catch (error) {
    res.status(500).json({ 
      status: 'error', 
      message: error.message 
    });
  }
});

// Users endpoint
app.get('/users', async (req, res) => {
  const client = new Client(dbConfig);
  
  try {
    await client.connect();
    const result = await client.query('SELECT * FROM users ORDER BY id');
    await client.end();
    
    res.json(result.rows);
  } catch (error) {
    res.status(500).json({ 
      status: 'error', 
      message: error.message 
    });
  }
});

// Create user endpoint
app.post('/users', async (req, res) => {
  const { name, email } = req.body;
  const client = new Client(dbConfig);
  
  try {
    await client.connect();
    const result = await client.query(
      'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING *',
      [name, email]
    );
    await client.end();
    
    res.status(201).json(result.rows[0]);
  } catch (error) {
    res.status(500).json({ 
      status: 'error', 
      message: error.message 
    });
  }
});

app.listen(port, '0.0.0.0', () => {
  console.log(`Server running on port ${port}`);
});
EOF
Subtask 1.3: Create Dockerfile for the Application
Create the Dockerfile for our web application:

cat > app/Dockerfile << 'EOF'
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Start the application
CMD ["npm", "start"]
EOF
Task 2: Write Integration Tests for Multi-Container Application
Subtask 2.1: Create Integration Test Package
Create the test package configuration:

cat > tests/package.json << 'EOF'
{
  "name": "docker-ci-tests",
  "version": "1.0.0",
  "description": "Integration tests for Docker CI lab",
  "main": "integration.test.js",
  "scripts": {
    "test": "jest --verbose --detectOpenHandles",
    "test:integration": "jest integration.test.js --verbose"
  },
  "dependencies": {
    "axios": "^1.5.0",
    "jest": "^29.6.0",
    "pg": "^8.11.0"
  },
  "jest": {
    "testEnvironment": "node",
    "testTimeout": 30000
  }
}
EOF
Subtask 2.2: Write Comprehensive Integration Tests
Create the integration test file:

cat > tests/integration.test.js << 'EOF'
const axios = require('axios');
const { Client } = require('pg');

// Configuration
const APP_URL = process.env.APP_URL || 'http://app:3000';
const DB_CONFIG = {
  host: process.env.DB_HOST || 'postgres',
  port: process.env.DB_PORT || 5432,
  database: process.env.DB_NAME || 'testdb',
  user: process.env.DB_USER || 'testuser',
  password: process.env.DB_PASSWORD || 'testpass',
};

// Helper function to wait for services
const waitForService = async (url, maxAttempts = 30, delay = 2000) => {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      await axios.get(url);
      console.log(`Service at ${url} is ready`);
      return true;
    } catch (error) {
      console.log(`Attempt ${i + 1}: Service not ready, waiting...`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }
  throw new Error(`Service at ${url} did not become ready after ${maxAttempts} attempts`);
};

// Setup database
const setupDatabase = async () => {
  const client = new Client(DB_CONFIG);
  await client.connect();
  
  // Create users table if it doesn't exist
  await client.query(`
    CREATE TABLE IF NOT EXISTS users (
      id SERIAL PRIMARY KEY,
      name VARCHAR(100) NOT NULL,
      email VARCHAR(100) UNIQUE NOT NULL,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    )
  `);
  
  // Clear existing test data
  await client.query('DELETE FROM users WHERE email LIKE %test%');
  
  await client.end();
};

describe('Integration Tests', () => {
  beforeAll(async () => {
    console.log('Setting up integration tests...');
    
    // Wait for application to be ready
    await waitForService(`${APP_URL}/health`);
    
    // Setup database
    await setupDatabase();
    
    console.log('Integration test setup complete');
  });

  describe('Health Checks', () => {
    test('Application health endpoint should return healthy status', async () => {
      const response = await axios.get(`${APP_URL}/health`);
      
      expect(response.status).toBe(200);
      expect(response.data).toHaveProperty('status', 'healthy');
      expect(response.data).toHaveProperty('timestamp');
    });

    test('Database connection should be working', async () => {
      const response = await axios.get(`${APP_URL}/db-status`);
      
      expect(response.status).toBe(200);
      expect(response.data).toHaveProperty('status', 'connected');
      expect(response.data).toHaveProperty('timestamp');
    });
  });

  describe('User Management', () => {
    test('Should retrieve empty users list initially', async () => {
      const response = await axios.get(`${APP_URL}/users`);
      
      expect(response.status).toBe(200);
      expect(Array.isArray(response.data)).toBe(true);
    });

    test('Should create a new user successfully', async () => {
      const newUser = {
        name: 'Test User',
        email: 'test@example.com'
      };

      const response = await axios.post(`${APP_URL}/users`, newUser);
      
      expect(response.status).toBe(201);
      expect(response.data).toHaveProperty('id');
      expect(response.data).toHaveProperty('name', newUser.name);
      expect(response.data).toHaveProperty('email', newUser.email);
      expect(response.data).toHaveProperty('created_at');
    });

    test('Should retrieve the created user in users list', async () => {
      const response = await axios.get(`${APP_URL}/users`);
      
      expect(response.status).toBe(200);
      expect(Array.isArray(response.data)).toBe(true);
      expect(response.data.length).toBeGreaterThan(0);
      
      const testUser = response.data.find(user => user.email === 'test@example.com');
      expect(testUser).toBeDefined();
      expect(testUser.name).toBe('Test User');
    });

    test('Should handle duplicate email creation gracefully', async () => {
      const duplicateUser = {
        name: 'Another Test User',
        email: 'test@example.com'
      };

      try {
        await axios.post(`${APP_URL}/users`, duplicateUser);
        fail('Should have thrown an error for duplicate email');
      } catch (error) {
        expect(error.response.status).toBe(500);
        expect(error.response.data).toHaveProperty('status', 'error');
      }
    });
  });

  describe('Database Integration', () => {
    test('Database should persist data across requests', async () => {
      // Create a user
      const newUser = {
        name: 'Persistence Test User',
        email: 'persistence@test.com'
      };

      await axios.post(`${APP_URL}/users`, newUser);

      // Verify it exists
      const response = await axios.get(`${APP_URL}/users`);
      const persistedUser = response.data.find(user => user.email === 'persistence@test.com');
      
      expect(persistedUser).toBeDefined();
      expect(persistedUser.name).toBe('Persistence Test User');
    });

    test('Database connection should handle multiple concurrent requests', async () => {
      const promises = [];
      
      for (let i = 0; i < 5; i++) {
        promises.push(axios.get(`${APP_URL}/db-status`));
      }

      const responses = await Promise.all(promises);
      
      responses.forEach(response => {
        expect(response.status).toBe(200);
        expect(response.data.status).toBe('connected');
      });
    });
  });
});
EOF
Subtask 2.3: Create Test Environment Dockerfile
Create a Dockerfile for the test environment:

cat > tests/Dockerfile << 'EOF'
FROM node:18-alpine

# Install curl for health checks
RUN apk add --no-cache curl

# Set working directory
WORKDIR /tests

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy test files
COPY . .

# Default command
CMD ["npm", "test"]
EOF
Task 3: Set up Docker Compose for Multi-Container Testing
Subtask 3.1: Create Main Docker Compose File
Create the main docker-compose.yml for development:

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 10s
      timeout: 5s
      retries: 5

  app:
    build: ./app
    ports:
      - "3000:3000"
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: testdb
      DB_USER: testuser
      DB_PASSWORD: testpass
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

volumes:
  postgres_data:
EOF
Subtask 3.2: Create Test-Specific Docker Compose File
Create docker-compose.test.yml for testing:

cat > docker-compose.test.yml << 'EOF'
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser -d testdb"]
      interval: 5s
      timeout: 3s
      retries: 10

  app:
    build: ./app
    environment:
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: testdb
      DB_USER: testuser
      DB_PASSWORD: testpass
    depends_on:
      postgres:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 10

  tests:
    build: ./tests
    environment:
      APP_URL: http://app:3000
      DB_HOST: postgres
      DB_PORT: 5432
      DB_NAME: testdb
      DB_USER: testuser
      DB_PASSWORD: testpass
    depends_on:
      app:
        condition: service_healthy
      postgres:
        condition: service_healthy
    volumes:
      - ./test-results:/tests/test-results

volumes:
  test_results:
EOF
Subtask 3.3: Test the Multi-Container Setup
Let's test our setup manually before automating it:

# Build and start the test environment
docker-compose -f docker-compose.test.yml up --build --abort-on-container-exit

# Check if all services are running (in a new terminal if needed)
docker-compose -f docker-compose.test.yml ps

# View test results
docker-compose -f docker-compose.test.yml logs tests

# Clean up
docker-compose -f docker-compose.test.yml down -v
Task 4: Set up Jenkins CI Pipeline
Subtask 4.1: Start Jenkins
Start Jenkins on your Al Nafi cloud machine:

# Create Jenkins home directory
mkdir -p ~/jenkins_home

# Start Jenkins using Docker
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v ~/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  --group-add $(getent group docker | cut -d: -f3) \
  jenkins/jenkins:lts

# Wait for Jenkins to start and get the initial admin password
echo "Waiting for Jenkins to start..."
sleep 30

# Get the initial admin password
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
Subtask 4.2: Configure Jenkins
Open your browser and navigate to http://your-machine-ip:8080
Use the initial admin password from the previous step
Install suggested plugins
Create an admin user
Install additional plugins:
Docker Pipeline
Blue Ocean (optional, for better UI)
Subtask 4.3: Create Jenkinsfile
Create a comprehensive Jenkinsfile:

cat > jenkins/Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    environment {
        COMPOSE_PROJECT_NAME = "docker-ci-${BUILD_NUMBER}"
        DOCKER_BUILDKIT = "1"
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Build Images') {
            steps {
                echo 'Building Docker images...'
                script {
                    sh '''
                        docker-compose -f docker-compose.test.yml build --no-cache
                    '''
                }
            }
        }
        
        stage('Start Services') {
            steps {
                echo 'Starting application services...'
                script {
                    sh '''
                        docker-compose -f docker-compose.test.yml up -d postgres app
                        
                        # Wait for services to be healthy
                        echo "Waiting for services to be ready..."
                        timeout 120 sh -c 'until docker-compose -f docker-compose.test.yml exec -T postgres pg_isready -U testuser -d testdb; do sleep 2; done'
                        timeout 120 sh -c 'until docker-compose -f docker-compose.test.yml exec -T app curl -f http://localhost:3000/health; do sleep 2; done'
                        
                        echo "Services are ready!"
                    '''
                }
            }
        }
        
        stage('Run Integration Tests') {
            steps {
                echo 'Running integration tests...'
                script {
                    sh '''
                        # Create test results directory
                        mkdir -p test-results
                        
                        # Run tests and capture results
                        docker-compose -f docker-compose.test.yml run --rm tests npm test -- --ci --reporters=default --reporters=jest-junit --outputFile=test-results/junit.xml || true
                        
                        # Copy test results from container
                        docker-compose -f docker-compose.test.yml run --rm -v $(pwd)/test-results:/host-results tests sh -c "cp -r /tests/test-results/* /host-results/ 2>/dev/null || true"
                    '''
                }
            }
            post {
                always {
                    // Publish test results
                    publishTestResults testResultsPattern: 'test-results/*.xml'
                    
                    // Archive test artifacts
                    archiveArtifacts artifacts: 'test-results/**/*', allowEmptyArchive: true
                }
            }
        }
        
        stage('Health Check') {
            steps {
                echo 'Performing final health checks...'
                script {
                    sh '''
                        # Test application endpoints
                        docker-compose -f docker-compose.test.yml exec -T app curl -f http://localhost:3000/health
                        docker-compose -f docker-compose.test.yml exec -T app curl -f http://localhost:3000/db-status
                        docker-compose -f docker-compose.test.yml exec -T app curl -f http://localhost:3000/users
                    '''
                }
            }
        }
        
        stage('Generate Reports') {
            steps {
                echo 'Generating test reports...'
                script {
                    sh '''
                        # Generate container logs
                        mkdir -p logs
                        docker-compose -f docker-compose.test.yml logs app > logs/app.log
                        docker-compose -f docker-compose.test.yml logs postgres > logs/postgres.log
                        docker-compose -f docker-compose.test.yml logs tests > logs/tests.log
                        
                        # Generate system information
                        echo "=== Docker Images ===" > logs/system-info.log
                        docker images >> logs/system-info.log
                        echo "=== Docker Containers ===" >> logs/system-info.log
                        docker-compose -f docker-compose.test.yml ps >> logs/system-info.log
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'logs/**/*', allowEmptyArchive: true
                }
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up...'
            script {
                sh '''
                    # Stop and remove containers
                    docker-compose -f docker-compose.test.yml down -v --remove-orphans
                    
                    # Clean up dangling images
                    docker image prune -f
                '''
            }
        }
        
        success {
            echo 'Pipeline completed successfully!'
            emailext (
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "Good news! The integration tests passed successfully.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
        
        failure {
            echo 'Pipeline failed!'
            emailext (
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "The integration tests failed. Please check the logs for details.",
                to: "${env.CHANGE_AUTHOR_EMAIL}"
            )
        }
    }
}
EOF
Subtask 4.4: Create Jenkins Job
In Jenkins, click New Item
Enter name: docker-ci-integration-tests
Select Pipeline and click OK
In the configuration:
Under Pipeline, select Pipeline script from SCM
SCM: Git
Repository URL: Your repository URL (or use local path for testing)
Script Path: jenkins/Jenkinsfile
Click Save
Task 5: Monitor and Review Test Results
Subtask 5.1: Create Test Result Monitoring Script
Create a script to monitor test results:

cat > monitor-tests.sh << 'EOF'
#!/bin/bash

# Monitor test results script
echo "=== Docker CI Integration Test Monitor ==="
echo "Timestamp: $(date)"
echo

# Check if containers are running
echo "=== Container Status ==="
docker-compose -f docker-compose.test.yml ps
echo

# Check application health
echo "=== Application Health ==="
if curl -s http://localhost:3000/health > /dev/null; then
    echo "✓ Application is healthy"
    curl -s http://localhost:3000/health | jq .
else
    echo "✗ Application is not responding"
fi
echo

# Check database connectivity
echo "=== Database Status ==="
if curl -s http://localhost:3000/db-status > /dev/null; then
    echo "✓ Database is connected"
    curl -s http://localhost:3000/db-status | jq .
else
    echo "✗ Database connection failed"
fi
echo

# Show recent logs
echo "=== Recent Application Logs ==="
docker-compose -f docker-compose.test.yml logs --tail=10 app
echo

echo "=== Recent Test Logs ==="
docker-compose -f docker-compose.test.yml logs --tail=10 tests
echo

# Resource usage
echo "=== Resource Usage ==="
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.NetIO}}"
EOF

chmod +x monitor-tests.sh
Subtask 5.2: Create Test Results Dashboard
Create a simple HTML dashboard to view test results:

mkdir -p dashboard
cat > dashboard/index.html << 'EOF'
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Docker CI Test Results Dashboard</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 20px;
            background-color: #f5f5f5;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
            background-color: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        }
        .header {
            text-align: center;
            color: #333;
            border-bottom: 2px solid #007bff;
            padding-bottom: 10px;
        }
        .status-card {
            display: inline-block;
            margin: 10px;
            padding: 20px;
            border-radius: 8px;
            min-width: 200px;
            text-align: center;
        }
        .success { background-color: #d4edda; border: 1px solid #c3e6cb; }
        .failure { background-color: #f8d7da; border: 1px solid #f5c6cb; }
        .info { background-color: #d1ecf1; border: 1px solid #bee5eb; }
        .logs {
            background-color: #f8f9fa;
            border: 1px solid #dee2e6;
            border-radius: 4px;
            padding: 15px;
            margin: 10px 0;
            font-family: monospace;
            white-space: pre-wrap;
            max-height: 300px;
            overflow-y: auto;
        }
        button {
            background-color: #007bff;
            color: white;
            border: none;
            padding: 10px 20px;
            border-radius: 4px;
            cursor: pointer;
            margin: 5px;
        }
        button:hover {
            background-color: #0056b3;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>Docker CI Integration Test Dashboard</h1>
            <p>Real-time monitoring of integration test results</p>
        </div>

        <div id="status-section">
            <div class="status-card info">
                <h3>Application Status</h3>
                <div id="app-status">Checking...</div>
            </div>
            <div class="status-card info">
                <h3>Database Status</h3>
                <div id="db-status">Checking...</div>
            </div>
            <div class="status-card info">
                <h3>Last Test Run</h3>
                <div id="test-status">Checking...</div>
            </div>
        </div>

        <div>
            <button onclick="refreshStatus()">Refresh Status</button>
            <button onclick="runTests()">Run Tests</button>
            <button onclick="viewLogs()">View Logs</button>
        </div>

        <div id="logs-section">
            <h3>System Logs</h3>
            <div id="logs" class="logs">Click "View Logs" to see recent logs...</div>
        </div>
    </div>

    <script>
        async function checkAppStatus() {
            try {
                const response = await fetch('/health');
                const data = await response.json();
                document.getElementById('app-status').innerHTML = 
                    `<span style="color: green;">✓ Healthy</span><br>
                     <small>${data.timestamp}</small>`;
            } catch (error) {
                document.getElementById('app-status').innerHTML = 
                    `<span style="color: red;">✗ Unhealthy</span><br>
                     <small>${error.message}</small>`;
            }
        }

        async function checkDbStatus() {
            try {
                const response = await fetch('/db-status');
                const data = await response.json();
                document.getElementById('db-status').innerHTML = 
                    `<span style="color: green;">✓ Connected</span><br>
                     <small>${data.timestamp}</small>`;
            } catch (error) {
                document.getElementById('db-status').innerHTML = 
                    `<span style="color: red;">✗ Disconnected</span><br>
                     <small>${error.message}</small>`;
            }
        }

        function refreshStatus() {
            checkAppStatus();
            checkDbStatus();
            document.getElementById('test-status').innerHTML = 
                `<span style="color: blue;">Last checked: ${new Date().toLocaleString()}</span>`;
        }

        function runTests() {
            document.getElementById('logs').textContent = 'Running integration tests...\n';
            // In a real implementation, this would trigger the test pipeline
            setTimeout(() => {
                document.getElementById('logs').textContent += 
                    'Tests completed successfully!\n' +
                    'All integration tests passed.\n' +
                    'Database connectivity: OK\n' +
                    'API endpoints: OK\n';
            }, 3000);
        }

        function viewLogs() {
            document.getElementById('logs').textContent = 
                'Loading recent logs...\n' +
                '[2024-01-15 10:30:15] INFO: Application started\n' +
                '[2024-01-15 10:30:16] INFO: Database connection established\n' +
                '[2024-01-15 10:30:17] INFO: Health check endpoint ready\n' +
                '[2024-01-15 10:30:18] INFO: Integration tests started\n' +
                '[2024-01-15 10:30:25] INFO: All tests passed\n';
        }

        // Initialize dashboard
        refreshStatus();
        setInterval(refreshStatus, 30000); // Refresh every 30 seconds
    </script>
</body>
</html>
EOF
Task 6: Automate Integration Testing Process
Subtask 6.1: Create Automated Test Runner Script
Create a comprehensive test automation script:
