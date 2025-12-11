Lab 75: Docker for Continuous Integration - Running Unit Tests in Docker Containers
Lab Objectives
By the end of this lab, you will be able to:

• Create Docker containers with testing environments for Python and Node.js applications • Write and execute unit tests for simple web applications • Set up continuous integration pipelines using Docker containers • Automate testing processes using Jenkins and GitLab CI • Analyze test results and implement improvements in your CI/CD workflow • Understand best practices for containerized testing in CI environments

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker concepts (containers, images, Dockerfile) • Familiarity with command-line interface operations • Basic knowledge of Python or Node.js programming • Understanding of version control systems (Git) • Basic knowledge of continuous integration concepts

Note: Al Nafi provides ready-to-use Linux-based cloud machines with all necessary tools pre-installed. Simply click "Start Lab" to begin - no need to build your own virtual machine.

Lab Environment Setup
Your cloud machine comes pre-configured with: • Docker Engine (latest stable version) • Git version control system • Python 3.9+ with pip package manager • Node.js 18+ with npm package manager • Jenkins (for CI automation) • Text editors (nano, vim)

Task 1: Create Docker Containers with Testing Environments
Subtask 1.1: Set Up Python Testing Environment
First, let's create a directory structure for our Python application and tests.

# Create project directory
mkdir -p ~/docker-ci-lab/python-app
cd ~/docker-ci-lab/python-app

# Create application structure
mkdir -p src tests
Create a simple Flask application:

# Create the main application file
cat > src/app.py << 'EOF'
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/')
def hello():
    return jsonify({"message": "Hello, World!"})

@app.route('/health')
def health():
    return jsonify({"status": "healthy"})

@app.route('/add/<int:a>/<int:b>')
def add_numbers(a, b):
    return jsonify({"result": a + b})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
EOF
Create requirements file for Python dependencies:

cat > requirements.txt << 'EOF'
Flask==2.3.3
pytest==7.4.2
pytest-flask==1.2.0
coverage==7.3.2
EOF
Create a Dockerfile for the Python testing environment:

cat > Dockerfile << 'EOF'
# Use Python 3.9 slim image as base
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy requirements first for better caching
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Set environment variables
ENV PYTHONPATH=/app/src
ENV FLASK_APP=src/app.py

# Expose port 5000
EXPOSE 5000

# Default command to run tests
CMD ["python", "-m", "pytest", "tests/", "-v", "--tb=short"]
EOF
Subtask 1.2: Write Unit Tests for Python Application
Create comprehensive unit tests:

cat > tests/test_app.py << 'EOF'
import pytest
import json
from src.app import app

@pytest.fixture
def client():
    """Create a test client for the Flask application."""
    app.config['TESTING'] = True
    with app.test_client() as client:
        yield client

def test_hello_endpoint(client):
    """Test the hello endpoint returns correct message."""
    response = client.get('/')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['message'] == 'Hello, World!'

def test_health_endpoint(client):
    """Test the health check endpoint."""
    response = client.get('/health')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['status'] == 'healthy'

def test_add_numbers_endpoint(client):
    """Test the add numbers endpoint with various inputs."""
    # Test positive numbers
    response = client.get('/add/5/3')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['result'] == 8
    
    # Test negative numbers
    response = client.get('/add/-2/7')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['result'] == 5
    
    # Test zero
    response = client.get('/add/0/10')
    assert response.status_code == 200
    data = json.loads(response.data)
    assert data['result'] == 10

def test_invalid_endpoint(client):
    """Test that invalid endpoints return 404."""
    response = client.get('/nonexistent')
    assert response.status_code == 404
EOF
Subtask 1.3: Set Up Node.js Testing Environment
Create a Node.js project structure:

# Create Node.js project directory
mkdir -p ~/docker-ci-lab/nodejs-app
cd ~/docker-ci-lab/nodejs-app

# Create application structure
mkdir -p src tests
Create a simple Express.js application:

cat > src/app.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

// Middleware
app.use(express.json());

// Routes
app.get('/', (req, res) => {
    res.json({ message: 'Hello, World!' });
});

app.get('/health', (req, res) => {
    res.json({ status: 'healthy' });
});

app.get('/multiply/:a/:b', (req, res) => {
    const a = parseInt(req.params.a);
    const b = parseInt(req.params.b);
    
    if (isNaN(a) || isNaN(b)) {
        return res.status(400).json({ error: 'Invalid numbers provided' });
    }
    
    res.json({ result: a * b });
});

// Start server only if this file is run directly
if (require.main === module) {
    app.listen(port, () => {
        console.log(`Server running on port ${port}`);
    });
}

module.exports = app;
EOF
Create package.json for Node.js dependencies:

cat > package.json << 'EOF'
{
  "name": "nodejs-ci-app",
  "version": "1.0.0",
  "description": "Node.js application for CI testing with Docker",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "test": "jest --verbose",
    "test:coverage": "jest --coverage"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^6.3.3"
  },
  "jest": {
    "testEnvironment": "node",
    "collectCoverageFrom": [
      "src/**/*.js"
    ]
  }
}
EOF
Create Dockerfile for Node.js testing environment:

cat > Dockerfile << 'EOF'
# Use Node.js 18 Alpine image as base
FROM node:18-alpine

# Set working directory
WORKDIR /app

# Copy package files first for better caching
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production && npm ci --only=development

# Copy application code
COPY . .

# Expose port 3000
EXPOSE 3000

# Default command to run tests
CMD ["npm", "test"]
EOF
Subtask 1.4: Write Unit Tests for Node.js Application
Create comprehensive unit tests for the Node.js application:

cat > tests/app.test.js << 'EOF'
const request = require('supertest');
const app = require('../src/app');

describe('Express App Tests', () => {
    
    describe('GET /', () => {
        it('should return hello message', async () => {
            const response = await request(app)
                .get('/')
                .expect(200);
            
            expect(response.body).toEqual({
                message: 'Hello, World!'
            });
        });
    });
    
    describe('GET /health', () => {
        it('should return health status', async () => {
            const response = await request(app)
                .get('/health')
                .expect(200);
            
            expect(response.body).toEqual({
                status: 'healthy'
            });
        });
    });
    
    describe('GET /multiply/:a/:b', () => {
        it('should multiply two positive numbers', async () => {
            const response = await request(app)
                .get('/multiply/4/5')
                .expect(200);
            
            expect(response.body).toEqual({
                result: 20
            });
        });
        
        it('should multiply negative numbers', async () => {
            const response = await request(app)
                .get('/multiply/-3/4')
                .expect(200);
            
            expect(response.body).toEqual({
                result: -12
            });
        });
        
        it('should handle zero multiplication', async () => {
            const response = await request(app)
                .get('/multiply/0/100')
                .expect(200);
            
            expect(response.body).toEqual({
                result: 0
            });
        });
        
        it('should return error for invalid numbers', async () => {
            const response = await request(app)
                .get('/multiply/abc/def')
                .expect(400);
            
            expect(response.body).toEqual({
                error: 'Invalid numbers provided'
            });
        });
    });
    
    describe('GET /nonexistent', () => {
        it('should return 404 for invalid routes', async () => {
            await request(app)
                .get('/nonexistent')
                .expect(404);
        });
    });
});
EOF
Task 2: Build and Test Docker Images
Subtask 2.1: Build and Test Python Application
Build the Python Docker image:

cd ~/docker-ci-lab/python-app

# Build the Docker image
docker build -t python-ci-app:latest .

# Verify the image was created
docker images | grep python-ci-app
Run tests in the Docker container:

# Run tests using the default CMD
docker run --rm python-ci-app:latest

# Run tests with coverage report
docker run --rm python-ci-app:latest python -m pytest tests/ -v --cov=src --cov-report=term-missing
Test the application interactively:

# Run the application in detached mode
docker run -d --name python-test-app -p 5000:5000 python-ci-app:latest python src/app.py

# Test the endpoints
curl http://localhost:5000/
curl http://localhost:5000/health
curl http://localhost:5000/add/10/20

# Clean up
docker stop python-test-app
docker rm python-test-app
Subtask 2.2: Build and Test Node.js Application
Build the Node.js Docker image:

cd ~/docker-ci-lab/nodejs-app

# Build the Docker image
docker build -t nodejs-ci-app:latest .

# Verify the image was created
docker images | grep nodejs-ci-app
Run tests in the Docker container:

# Run tests using the default CMD
docker run --rm nodejs-ci-app:latest

# Run tests with coverage report
docker run --rm nodejs-ci-app:latest npm run test:coverage
Test the application interactively:

# Run the application in detached mode
docker run -d --name nodejs-test-app -p 3000:3000 nodejs-ci-app:latest npm start

# Test the endpoints
curl http://localhost:3000/
curl http://localhost:3000/health
curl http://localhost:3000/multiply/6/7

# Clean up
docker stop nodejs-test-app
docker rm nodejs-test-app
Task 3: Set Up CI Pipeline with Docker Compose
Subtask 3.1: Create Docker Compose for Multi-Service Testing
Create a Docker Compose file to orchestrate both applications:

cd ~/docker-ci-lab

cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  python-tests:
    build: 
      context: ./python-app
      dockerfile: Dockerfile
    container_name: python-ci-tests
    volumes:
      - ./python-app:/app
      - ./test-results:/test-results
    environment:
      - PYTHONPATH=/app/src
    command: >
      sh -c "python -m pytest tests/ -v --junitxml=/test-results/python-results.xml --cov=src --cov-report=xml:/test-results/python-coverage.xml"
    networks:
      - ci-network

  nodejs-tests:
    build:
      context: ./nodejs-app
      dockerfile: Dockerfile
    container_name: nodejs-ci-tests
    volumes:
      - ./nodejs-app:/app
      - ./test-results:/test-results
    command: >
      sh -c "npm test -- --testResultsProcessor=jest-junit --coverageReporters=cobertura --coverageDirectory=/test-results"
    environment:
      - JEST_JUNIT_OUTPUT_DIR=/test-results
      - JEST_JUNIT_OUTPUT_NAME=nodejs-results.xml
    networks:
      - ci-network

  python-app:
    build: 
      context: ./python-app
      dockerfile: Dockerfile
    container_name: python-ci-app
    ports:
      - "5000:5000"
    command: python src/app.py
    networks:
      - ci-network
    depends_on:
      - python-tests

  nodejs-app:
    build:
      context: ./nodejs-app
      dockerfile: Dockerfile
    container_name: nodejs-ci-app
    ports:
      - "3000:3000"
    command: npm start
    networks:
      - ci-network
    depends_on:
      - nodejs-tests

networks:
  ci-network:
    driver: bridge

volumes:
  test-results:
EOF
Create test results directory:

mkdir -p ~/docker-ci-lab/test-results
Subtask 3.2: Update Node.js Configuration for CI
Update the Node.js package.json to include jest-junit for XML test reports:

cd ~/docker-ci-lab/nodejs-app

# Update package.json to include jest-junit
cat > package.json << 'EOF'
{
  "name": "nodejs-ci-app",
  "version": "1.0.0",
  "description": "Node.js application for CI testing with Docker",
  "main": "src/app.js",
  "scripts": {
    "start": "node src/app.js",
    "test": "jest --verbose",
    "test:coverage": "jest --coverage",
    "test:ci": "jest --ci --coverage --testResultsProcessor=jest-junit"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "jest": "^29.7.0",
    "supertest": "^6.3.3",
    "jest-junit": "^16.0.0"
  },
  "jest": {
    "testEnvironment": "node",
    "collectCoverageFrom": [
      "src/**/*.js"
    ]
  }
}
EOF
Task 4: Automate Testing with Jenkins
Subtask 4.1: Set Up Jenkins Pipeline
Create a Jenkins pipeline script:

cd ~/docker-ci-lab

cat > Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    environment {
        DOCKER_COMPOSE_FILE = 'docker-compose.yml'
        TEST_RESULTS_DIR = 'test-results'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                // In a real scenario, this would checkout from Git
                sh 'echo "Source code checked out"'
            }
        }
        
        stage('Build Images') {
            steps {
                echo 'Building Docker images...'
                sh 'docker-compose build'
            }
        }
        
        stage('Run Tests') {
            parallel {
                stage('Python Tests') {
                    steps {
                        echo 'Running Python tests...'
                        sh 'docker-compose run --rm python-tests'
                    }
                }
                stage('Node.js Tests') {
                    steps {
                        echo 'Running Node.js tests...'
                        sh 'docker-compose run --rm nodejs-tests'
                    }
                }
            }
        }
        
        stage('Collect Test Results') {
            steps {
                echo 'Collecting test results...'
                sh 'ls -la test-results/ || echo "No test results found"'
            }
        }
        
        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                echo 'Deploying to staging environment...'
                sh 'docker-compose up -d python-app nodejs-app'
                
                // Health checks
                sh '''
                    echo "Waiting for applications to start..."
                    sleep 10
                    
                    echo "Testing Python app health..."
                    curl -f http://localhost:5000/health || exit 1
                    
                    echo "Testing Node.js app health..."
                    curl -f http://localhost:3000/health || exit 1
                '''
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up...'
            sh 'docker-compose down || true'
            sh 'docker system prune -f || true'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
        }
    }
}
EOF
Subtask 4.2: Create Jenkins Job Configuration
Create a simple shell script to simulate Jenkins job execution:

cat > run-jenkins-pipeline.sh << 'EOF'
#!/bin/bash

echo "=== Docker CI Pipeline Execution ==="
echo "Starting at: $(date)"

# Set environment variables
export DOCKER_COMPOSE_FILE="docker-compose.yml"
export TEST_RESULTS_DIR="test-results"

# Stage 1: Checkout
echo ""
echo "Stage 1: Checkout"
echo "=================="
echo "Source code checked out successfully"

# Stage 2: Build Images
echo ""
echo "Stage 2: Build Images"
echo "===================="
docker-compose build

if [ $? -ne 0 ]; then
    echo "ERROR: Failed to build Docker images"
    exit 1
fi

# Stage 3: Run Tests
echo ""
echo "Stage 3: Run Tests"
echo "=================="

echo "Running Python tests..."
docker-compose run --rm python-tests
PYTHON_TEST_RESULT=$?

echo "Running Node.js tests..."
docker-compose run --rm nodejs-tests
NODEJS_TEST_RESULT=$?

# Stage 4: Collect Test Results
echo ""
echo "Stage 4: Collect Test Results"
echo "============================="
echo "Test results directory contents:"
ls -la test-results/ 2>/dev/null || echo "No test results directory found"

# Stage 5: Deploy to Staging (if tests passed)
if [ $PYTHON_TEST_RESULT -eq 0 ] && [ $NODEJS_TEST_RESULT -eq 0 ]; then
    echo ""
    echo "Stage 5: Deploy to Staging"
    echo "=========================="
    echo "All tests passed. Deploying applications..."
    
    docker-compose up -d python-app nodejs-app
    
    echo "Waiting for applications to start..."
    sleep 15
    
    echo "Running health checks..."
    
    # Test Python app
    if curl -f http://localhost:5000/health > /dev/null 2>&1; then
        echo "✓ Python app health check passed"
    else
        echo "✗ Python app health check failed"
    fi
    
    # Test Node.js app
    if curl -f http://localhost:3000/health > /dev/null 2>&1; then
        echo "✓ Node.js app health check passed"
    else
        echo "✗ Node.js app health check failed"
    fi
    
    echo ""
    echo "Applications are running:"
    echo "Python app: http://localhost:5000"
    echo "Node.js app: http://localhost:3000"
    
else
    echo ""
    echo "Tests failed. Skipping deployment."
    echo "Python tests result: $PYTHON_TEST_RESULT"
    echo "Node.js tests result: $NODEJS_TEST_RESULT"
fi

echo ""
echo "=== Pipeline Completed at: $(date) ==="
EOF

chmod +x run-jenkins-pipeline.sh
Task 5: Set Up GitLab CI Pipeline
Subtask 5.1: Create GitLab CI Configuration
Create a GitLab CI/CD pipeline configuration:

cat > .gitlab-ci.yml << 'EOF'
# GitLab CI/CD Pipeline for Docker-based Testing

stages:
  - build
  - test
  - deploy

variables:
  DOCKER_DRIVER: overlay2
  DOCKER_TLS_CERTDIR: "/certs"

services:
  - docker:20.10.16-dind

before_script:
  - docker info
  - docker-compose --version

# Build stage
build:
  stage: build
  script:
    - echo "Building Docker images..."
    - docker-compose build
    - docker images
  artifacts:
    reports:
      junit: test-results/*.xml
    paths:
      - test-results/
    expire_in: 1 week

# Test stage - Python
test:python:
  stage: test
  script:
    - echo "Running Python tests..."
    - docker-compose run --rm python-tests
    - echo "Python tests completed"
  artifacts:
    reports:
      junit: test-results/python-results.xml
      coverage_report:
        coverage_format: cobertura
        path: test-results/python-coverage.xml
    paths:
      - test-results/
    expire_in: 1 week
  dependencies:
    - build

# Test stage - Node.js
test:nodejs:
  stage: test
  script:
    - echo "Running Node.js tests..."
    - docker-compose run --rm nodejs-tests
    - echo "Node.js tests completed"
  artifacts:
    reports:
      junit: test-results/nodejs-results.xml
    paths:
      - test-results/
    expire_in: 1 week
  dependencies:
    - build

# Deploy stage
deploy:staging:
  stage: deploy
  script:
    - echo "Deploying to staging environment..."
    - docker-compose up -d python-app nodejs-app
    - sleep 15
    - echo "Running health checks..."
    - curl -f http://localhost:5000/health
    - curl -f http://localhost:3000/health
    - echo "Deployment successful"
  environment:
    name: staging
    url: http://localhost:5000
  dependencies:
    - test:python
    - test:nodejs
  only:
    - main
    - master

# Cleanup
after_script:
  - docker-compose down || true
  - docker system prune -f || true
EOF
Subtask 5.2: Create GitLab CI Simulation Script
Create a script to simulate GitLab CI execution locally:

cat > run-gitlab-ci.sh << 'EOF'
#!/bin/bash

echo "=== GitLab CI Pipeline Simulation ==="
echo "Starting at: $(date)"

# Simulate GitLab CI environment variables
export CI=true
export GITLAB_CI=true
export CI_PIPELINE_ID="12345"
export CI_JOB_ID="67890"

# Stage: Build
echo ""
echo "🔨 Stage: Build"
echo "==============="
echo "Building Docker images..."
docker-compose build

if [ $? -ne 0 ]; then
    echo "❌ Build stage failed"
    exit 1
fi

echo "✅ Build stage completed successfully"

# Stage: Test - Python
echo ""
echo "🧪 Stage: Test - Python"
echo "======================="
echo "Running Python tests..."
docker-compose run --rm python-tests
PYTHON_EXIT_CODE=$?

if [ $PYTHON_EXIT_CODE -eq 0 ]; then
    echo "✅ Python tests passed"
else
    echo "❌ Python tests failed"
fi

# Stage: Test - Node.js
echo ""
echo "🧪 Stage: Test - Node.js"
echo "======================="
echo "Running Node.js tests..."
docker-compose run --rm nodejs-tests
NODEJS_EXIT_CODE=$?

if [ $NODEJS_EXIT_CODE -eq 0 ]; then
    echo "✅ Node.js tests passed"
else
    echo "❌ Node.js tests failed"
fi

# Stage: Deploy (only if all tests pass)
if [ $PYTHON_EXIT_CODE -eq 0 ] && [ $NODEJS_EXIT_CODE -eq 0 ]; then
    echo ""
    echo "🚀 Stage: Deploy"
    echo "==============="
    echo "All tests passed. Deploying to staging..."
    
    docker-compose up -d python-app nodejs-app
    
    echo "Waiting for applications to start..."
    sleep 15
    
    echo "Running health checks..."
    
    # Health check for Python app
    if curl -f http://localhost:5000/health > /dev/null 2>&1; then
        echo "✅ Python app health check passed"
        PYTHON_HEALTH=0
    else
        echo "❌ Python app health check failed"
        PYTHON_HEALTH=1
    fi
    
    # Health check for Node.js app
    if curl -f http://localhost:3000/health > /dev/null 2>&1; then
        echo "✅ Node.js app health check passed"
        NODEJS_HEALTH=0
    else
        echo "❌ Node.js app health check failed"
        NODEJS_HEALTH=1
    fi
    
    if [ $PYTHON_HEALTH -eq 0 ] && [ $NODEJS_HEALTH -eq 0 ]; then
        echo "✅ Deployment successful"
        echo ""
        echo "🌐 Applications are now running:"
        echo "   Python app: http://localhost:5000"
        echo "   Node.js app: http://localhost:3000"
    else
        echo "❌ Deployment health checks failed"
    fi
    
else
    echo ""
    echo "❌ Tests failed. Skipping deployment stage."
    echo "   Python tests: $([ $PYTHON_EXIT_CODE -eq 0 ] && echo 'PASSED' || echo 'FAILED')"
    echo "   Node.js tests: $([ $NODEJS_EXIT_CODE -eq 0 ] && echo 'PASSED' || echo 'FAILED')"
fi

# Cleanup
echo ""
echo "🧹 Cleanup"
echo "========="
echo "Cleaning up resources..."
# Note: Keeping containers running for demonstration
# docker-compose down || true
# docker system prune -f || true

echo ""
echo "=== GitLab CI Pipeline Completed at: $(date) ==="

# Show final status
if [ $PYTHON_EXIT_CODE -eq 0 ] && [ $NODEJS_EXIT_CODE -eq 0 ]; then
    echo "🎉 Pipeline Status: SUCCESS"
    exit 0
else
    echo "💥 Pipeline Status: FAILED"
    exit 1
fi
EOF

chmod +x run-gitlab-ci.sh
Task 6: Analyze Test Results and Implement Improvements
Subtask 6.1: Run Complete CI Pipeline
Execute the complete CI pipeline:

cd ~/docker-ci-lab

# Run the Jenkins-style pipeline
echo "Running Jenkins-style pipeline..."
./run-jenkins-pipeline.sh

echo ""
echo "Waiting 5 seconds before running GitLab CI simulation..."
sleep 5

# Clean up before running GitLab CI
docker-compose down

# Run the GitLab CI simulation
echo "Running GitLab CI simulation..."
./run-gitlab-ci.sh
Subtask 6.2: Analyze Test Results
Create a script to analyze and display test results:

cat > analyze-results.sh << 'EOF'
#!/bin/bash

echo "=== Test Results Analysis ==="
echo "Generated at: $(date)"
echo ""

# Check if test results directory exists
if [ ! -d "test-results" ]; then
    echo "❌ No test results directory found"
    echo "Please run the CI pipeline first"
    exit 1
fi

echo "📊 Test Results Summary"
echo "======================"

# Analyze Python test results
if [ -f "test-results/python-results.xml" ]; then
    echo "✅ Python test results found"
    PYTHON_TESTS=$(grep -o 'tests="[0-9]*"' test-results/python-results.xml | cut -d'"' -f2)
    PYTHON_FAILURES=$(grep -o 'failures="[0-9]*"' test-results/python-results.xml | cut -d'"' -f2)
    PYTHON_ERRORS=$(grep -o 'errors="[0-9]*"' test-results/python-results.xml | cut -d'"' -f2)
    
    echo "   Total tests: ${PYTHON_TESTS:-0}"
    echo "   Failures: ${PYTHON_FAILURES:-0}"
    echo "   Errors: ${PYTHON_ERRORS:-0}"
    echo "   Success rate: $(echo "scale=2; (${PYTHON_TESTS:-0} - ${PYTHON_FAILURES:-0} - ${PYTHON_ERRORS:-0}) * 100 / ${PYTHON_TESTS:-1}" | bc)%"
else
    echo "❌ Python test results not found"
fi

echo ""

# Analyze Node.js test results
if [ -f "test-results/nodejs-results.xml" ]; then
    echo "✅ Node.js test results found"
    NODEJS_TESTS=$(grep -o 'tests="[0-9]*"' test-results/nodejs-results.xml | cut -d'"' -f2)
    NODEJS_FAILURES=$(grep -o 'failures="[0-9]*"' test-results/nodejs-results.xml | cut -d'"' -f2)
    NODEJS_ERRORS=$(grep -o 'errors="[0-9]*"' test-results/nodejs-results.xml | cut -d'"' -f2)
    
    echo "   Total tests: ${NODEJS_TESTS:-0}"
    echo "   Failures: ${NODEJS_FAILURES:-0}"
    echo "   Errors: ${NODEJS_ERRORS:-0}"
    echo "   Success rate: $(echo "scale=2; (${NODEJS_TESTS:-0} - ${NODEJS_FAILURES:-0} - ${NODEJS_ERRORS:-0}) * 100 / ${NODEJS_TESTS:-1}" | bc)%"
else
    echo "❌ Node.js test results not found"
fi

echo ""
echo "📁 Available Result Files"
echo "========================"
ls -la test-results/ 2>/dev/null || echo "No files found"

echo ""
echo "🐳 Docker Images Status"
echo "======================"
docker images | grep -E "(python-ci-app|nodejs-ci-app)"

echo ""
echo "📦 Running Containers"
echo "===================="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

echo ""
echo "🔍 Quick Application Tests"
echo "========================="

# Test Python app if running
if docker ps | grep -q python-ci-app; then
    echo
