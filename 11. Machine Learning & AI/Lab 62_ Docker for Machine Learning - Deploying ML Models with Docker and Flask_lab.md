Lab 62: Docker for Machine Learning - Deploying ML Models with Docker and Flask
Objectives
By the end of this lab, you will be able to:

Containerize a trained machine learning model using Docker
Create a Flask REST API to serve machine learning predictions
Deploy ML models in Docker containers for production use
Integrate database storage for model results using PostgreSQL
Scale ML applications using Docker Compose
Test ML APIs using REST clients
Understand best practices for ML model deployment
Prerequisites
Before starting this lab, you should have:

Basic understanding of Python programming
Familiarity with machine learning concepts
Basic knowledge of REST APIs
Understanding of Docker fundamentals
Experience with command-line interfaces
Technical Requirements:

Python 3.8 or higher
Docker and Docker Compose
Basic understanding of Flask framework
Familiarity with scikit-learn library
Lab Environment Setup
Al Nafi Cloud Machine Access: This lab uses Al Nafi's pre-configured Linux-based cloud machines. Simply click Start Lab to access your environment. No need to build your own VM - everything is ready to use!

Your cloud machine comes pre-installed with:

Docker and Docker Compose
Python 3.8+
Git
Text editors (nano, vim)
All necessary development tools
Task 1: Prepare the Machine Learning Model
Step 1.1: Create Project Directory Structure
First, let's create a well-organized project structure for our ML deployment.

# Create main project directory
mkdir ml-docker-deployment
cd ml-docker-deployment

# Create subdirectories
mkdir app
mkdir model
mkdir data
mkdir docker-compose
Step 1.2: Create a Simple Machine Learning Model
Create a simple trained model that we can deploy. We'll use a basic iris classification model.

# Create the model training script
nano model/train_model.py
Add the following content:

import pickle
import pandas as pd
from sklearn.datasets import load_iris
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
import joblib

def train_iris_model():
    """Train a simple iris classification model"""
    
    # Load the iris dataset
    iris = load_iris()
    X, y = iris.data, iris.target
    
    # Split the data
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    # Train the model
    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)
    
    # Test the model
    y_pred = model.predict(X_test)
    accuracy = accuracy_score(y_test, y_pred)
    print(f"Model accuracy: {accuracy:.2f}")
    
    # Save the model
    joblib.dump(model, 'iris_model.pkl')
    
    # Save feature names and target names for reference
    model_info = {
        'feature_names': iris.feature_names,
        'target_names': iris.target_names.tolist(),
        'accuracy': accuracy
    }
    
    with open('model_info.pkl', 'wb') as f:
        pickle.dump(model_info, f)
    
    print("Model saved successfully!")
    return model, model_info

if __name__ == "__main__":
    train_iris_model()
Step 1.3: Train and Save the Model
# Navigate to model directory
cd model

# Install required packages
pip3 install scikit-learn pandas joblib

# Train the model
python3 train_model.py

# Verify model files are created
ls -la *.pkl

# Return to main directory
cd ..
Task 2: Create Flask API Application
Step 2.1: Create Flask Application Structure
# Create Flask app file
nano app/app.py
Add the following Flask application code:

from flask import Flask, request, jsonify
import joblib
import pickle
import numpy as np
import os
import logging
from datetime import datetime

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# Global variables for model and info
model = None
model_info = None

def load_model():
    """Load the trained model and model info"""
    global model, model_info
    
    try:
        # Load the model
        model = joblib.load('/app/model/iris_model.pkl')
        
        # Load model info
        with open('/app/model/model_info.pkl', 'rb') as f:
            model_info = pickle.load(f)
        
        logger.info("Model loaded successfully")
        logger.info(f"Model accuracy: {model_info['accuracy']:.2f}")
        
    except Exception as e:
        logger.error(f"Error loading model: {str(e)}")
        raise

@app.route('/health', methods=['GET'])
def health_check():
    """Health check endpoint"""
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.now().isoformat(),
        'model_loaded': model is not None
    })

@app.route('/model/info', methods=['GET'])
def get_model_info():
    """Get model information"""
    if model_info is None:
        return jsonify({'error': 'Model not loaded'}), 500
    
    return jsonify({
        'feature_names': model_info['feature_names'],
        'target_names': model_info['target_names'],
        'accuracy': model_info['accuracy'],
        'model_type': 'RandomForestClassifier'
    })

@app.route('/predict', methods=['POST'])
def predict():
    """Make predictions using the loaded model"""
    try:
        # Check if model is loaded
        if model is None:
            return jsonify({'error': 'Model not loaded'}), 500
        
        # Get JSON data from request
        data = request.get_json()
        
        if not data:
            return jsonify({'error': 'No data provided'}), 400
        
        # Extract features
        if 'features' not in data:
            return jsonify({'error': 'Features not provided'}), 400
        
        features = data['features']
        
        # Validate feature count
        if len(features) != 4:
            return jsonify({
                'error': 'Expected 4 features',
                'expected_features': model_info['feature_names']
            }), 400
        
        # Convert to numpy array and reshape
        features_array = np.array(features).reshape(1, -1)
        
        # Make prediction
        prediction = model.predict(features_array)[0]
        prediction_proba = model.predict_proba(features_array)[0]
        
        # Get class name
        predicted_class = model_info['target_names'][prediction]
        
        # Create response
        response = {
            'prediction': int(prediction),
            'predicted_class': predicted_class,
            'probabilities': {
                model_info['target_names'][i]: float(prob) 
                for i, prob in enumerate(prediction_proba)
            },
            'features_used': {
                model_info['feature_names'][i]: float(features[i]) 
                for i in range(len(features))
            },
            'timestamp': datetime.now().isoformat()
        }
        
        logger.info(f"Prediction made: {predicted_class}")
        return jsonify(response)
        
    except Exception as e:
        logger.error(f"Prediction error: {str(e)}")
        return jsonify({'error': str(e)}), 500

@app.route('/predict/batch', methods=['POST'])
def predict_batch():
    """Make batch predictions"""
    try:
        if model is None:
            return jsonify({'error': 'Model not loaded'}), 500
        
        data = request.get_json()
        
        if not data or 'batch_features' not in data:
            return jsonify({'error': 'Batch features not provided'}), 400
        
        batch_features = data['batch_features']
        
        # Validate batch
        if not isinstance(batch_features, list):
            return jsonify({'error': 'Batch features must be a list'}), 400
        
        results = []
        
        for i, features in enumerate(batch_features):
            if len(features) != 4:
                results.append({
                    'index': i,
                    'error': 'Expected 4 features'
                })
                continue
            
            try:
                features_array = np.array(features).reshape(1, -1)
                prediction = model.predict(features_array)[0]
                prediction_proba = model.predict_proba(features_array)[0]
                predicted_class = model_info['target_names'][prediction]
                
                results.append({
                    'index': i,
                    'prediction': int(prediction),
                    'predicted_class': predicted_class,
                    'confidence': float(max(prediction_proba))
                })
                
            except Exception as e:
                results.append({
                    'index': i,
                    'error': str(e)
                })
        
        return jsonify({
            'results': results,
            'total_predictions': len(results),
            'timestamp': datetime.now().isoformat()
        })
        
    except Exception as e:
        logger.error(f"Batch prediction error: {str(e)}")
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    # Load model on startup
    load_model()
    
    # Run the app
    app.run(host='0.0.0.0', port=5000, debug=False)
Step 2.2: Create Requirements File
# Create requirements file for Python dependencies
nano app/requirements.txt
Add the following content:

Flask==2.3.3
scikit-learn==1.3.0
joblib==1.3.2
numpy==1.24.3
pandas==2.0.3
psycopg2-binary==2.9.7
python-dotenv==1.0.0
Task 3: Containerize the ML Application
Step 3.1: Create Dockerfile
# Create Dockerfile for the ML application
nano Dockerfile
Add the following Dockerfile content:

# Use Python 3.9 slim image as base
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first for better caching
COPY app/requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app/ .

# Create model directory
RUN mkdir -p /app/model

# Copy model files
COPY model/*.pkl /app/model/

# Create non-root user for security
RUN useradd -m -u 1000 mluser && chown -R mluser:mluser /app
USER mluser

# Expose port
EXPOSE 5000

# Health check
HEALTHCHECK --interval=30s --timeout=30s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Run the application
CMD ["python", "app.py"]
Step 3.2: Create .dockerignore File
# Create .dockerignore file
nano .dockerignore
Add the following content:

__pycache__
*.pyc
*.pyo
*.pyd
.Python
env
pip-log.txt
pip-delete-this-directory.txt
.tox
.coverage
.coverage.*
.cache
nosetests.xml
coverage.xml
*.cover
*.log
.git
.mypy_cache
.pytest_cache
.hypothesis
.DS_Store
Step 3.3: Build the Docker Image
# Build the Docker image
docker build -t ml-flask-app:latest .

# Verify the image was created
docker images | grep ml-flask-app
Step 3.4: Test the Container
# Run the container
docker run -d --name ml-app-test -p 5000:5000 ml-flask-app:latest

# Check if container is running
docker ps

# Test the health endpoint
curl http://localhost:5000/health

# Check container logs
docker logs ml-app-test

# Stop and remove test container
docker stop ml-app-test
docker rm ml-app-test
Task 4: Create Database Integration
Step 4.1: Create Database Schema
# Create database directory
mkdir database

# Create database initialization script
nano database/init.sql
Add the following SQL content:

-- Create database for ML predictions
CREATE DATABASE ml_predictions;

-- Connect to the database
\c ml_predictions;

-- Create predictions table
CREATE TABLE predictions (
    id SERIAL PRIMARY KEY,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    features JSONB NOT NULL,
    prediction INTEGER NOT NULL,
    predicted_class VARCHAR(50) NOT NULL,
    probabilities JSONB NOT NULL,
    model_version VARCHAR(20) DEFAULT '1.0',
    request_id VARCHAR(100)
);

-- Create index on timestamp for better query performance
CREATE INDEX idx_predictions_timestamp ON predictions(timestamp);

-- Create index on predicted_class
CREATE INDEX idx_predictions_class ON predictions(predicted_class);

-- Create a view for daily prediction summary
CREATE VIEW daily_predictions AS
SELECT 
    DATE(timestamp) as prediction_date,
    predicted_class,
    COUNT(*) as prediction_count,
    AVG((probabilities->>predicted_class)::float) as avg_confidence
FROM predictions 
GROUP BY DATE(timestamp), predicted_class
ORDER BY prediction_date DESC, predicted_class;

-- Insert sample data for testing
INSERT INTO predictions (features, prediction, predicted_class, probabilities, request_id) VALUES
('[5.1, 3.5, 1.4, 0.2]', 0, 'setosa', '{"setosa": 0.95, "versicolor": 0.03, "virginica": 0.02}', 'test-001'),
('[6.2, 2.9, 4.3, 1.3]', 1, 'versicolor', '{"setosa": 0.05, "versicolor": 0.85, "virginica": 0.10}', 'test-002'),
('[7.3, 2.9, 6.3, 1.8]', 2, 'virginica', '{"setosa": 0.02, "versicolor": 0.08, "virginica": 0.90}', 'test-003');
Step 4.2: Update Flask App with Database Integration
# Create updated Flask app with database integration
nano app/app_with_db.py
Add the following enhanced Flask application:

from flask import Flask, request, jsonify
import joblib
import pickle
import numpy as np
import os
import logging
import psycopg2
import json
from datetime import datetime
import uuid
from contextlib import contextmanager

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# Database configuration
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'postgres'),
    'database': os.getenv('DB_NAME', 'ml_predictions'),
    'user': os.getenv('DB_USER', 'postgres'),
    'password': os.getenv('DB_PASSWORD', 'postgres'),
    'port': os.getenv('DB_PORT', '5432')
}

# Global variables
model = None
model_info = None

@contextmanager
def get_db_connection():
    """Context manager for database connections"""
    conn = None
    try:
        conn = psycopg2.connect(**DB_CONFIG)
        yield conn
    except Exception as e:
        if conn:
            conn.rollback()
        logger.error(f"Database error: {str(e)}")
        raise
    finally:
        if conn:
            conn.close()

def load_model():
    """Load the trained model and model info"""
    global model, model_info
    
    try:
        model = joblib.load('/app/model/iris_model.pkl')
        
        with open('/app/model/model_info.pkl', 'rb') as f:
            model_info = pickle.load(f)
        
        logger.info("Model loaded successfully")
        logger.info(f"Model accuracy: {model_info['accuracy']:.2f}")
        
    except Exception as e:
        logger.error(f"Error loading model: {str(e)}")
        raise

def save_prediction_to_db(features, prediction, predicted_class, probabilities, request_id):
    """Save prediction to database"""
    try:
        with get_db_connection() as conn:
            cursor = conn.cursor()
            
            insert_query = """
                INSERT INTO predictions (features, prediction, predicted_class, probabilities, request_id)
                VALUES (%s, %s, %s, %s, %s)
                RETURNING id;
            """
            
            cursor.execute(insert_query, (
                json.dumps(features),
                prediction,
                predicted_class,
                json.dumps(probabilities),
                request_id
            ))
            
            prediction_id = cursor.fetchone()[0]
            conn.commit()
            
            logger.info(f"Prediction saved to database with ID: {prediction_id}")
            return prediction_id
            
    except Exception as e:
        logger.error(f"Error saving prediction to database: {str(e)}")
        return None

@app.route('/health', methods=['GET'])
def health_check():
    """Health check endpoint"""
    db_status = "unknown"
    
    try:
        with get_db_connection() as conn:
            cursor = conn.cursor()
            cursor.execute("SELECT 1;")
            db_status = "healthy"
    except:
        db_status = "unhealthy"
    
    return jsonify({
        'status': 'healthy',
        'timestamp': datetime.now().isoformat(),
        'model_loaded': model is not None,
        'database_status': db_status
    })

@app.route('/model/info', methods=['GET'])
def get_model_info():
    """Get model information"""
    if model_info is None:
        return jsonify({'error': 'Model not loaded'}), 500
    
    return jsonify({
        'feature_names': model_info['feature_names'],
        'target_names': model_info['target_names'],
        'accuracy': model_info['accuracy'],
        'model_type': 'RandomForestClassifier'
    })

@app.route('/predict', methods=['POST'])
def predict():
    """Make predictions and save to database"""
    try:
        if model is None:
            return jsonify({'error': 'Model not loaded'}), 500
        
        data = request.get_json()
        
        if not data or 'features' not in data:
            return jsonify({'error': 'Features not provided'}), 400
        
        features = data['features']
        
        if len(features) != 4:
            return jsonify({
                'error': 'Expected 4 features',
                'expected_features': model_info['feature_names']
            }), 400
        
        # Generate request ID
        request_id = str(uuid.uuid4())
        
        # Make prediction
        features_array = np.array(features).reshape(1, -1)
        prediction = model.predict(features_array)[0]
        prediction_proba = model.predict_proba(features_array)[0]
        predicted_class = model_info['target_names'][prediction]
        
        # Create probabilities dictionary
        probabilities = {
            model_info['target_names'][i]: float(prob) 
            for i, prob in enumerate(prediction_proba)
        }
        
        # Save to database
        db_id = save_prediction_to_db(
            features, int(prediction), predicted_class, probabilities, request_id
        )
        
        # Create response
        response = {
            'request_id': request_id,
            'database_id': db_id,
            'prediction': int(prediction),
            'predicted_class': predicted_class,
            'probabilities': probabilities,
            'features_used': {
                model_info['feature_names'][i]: float(features[i]) 
                for i in range(len(features))
            },
            'timestamp': datetime.now().isoformat()
        }
        
        logger.info(f"Prediction made: {predicted_class} (ID: {request_id})")
        return jsonify(response)
        
    except Exception as e:
        logger.error(f"Prediction error: {str(e)}")
        return jsonify({'error': str(e)}), 500

@app.route('/predictions/history', methods=['GET'])
def get_prediction_history():
    """Get prediction history from database"""
    try:
        limit = request.args.get('limit', 10, type=int)
        offset = request.args.get('offset', 0, type=int)
        
        with get_db_connection() as conn:
            cursor = conn.cursor()
            
            query = """
                SELECT id, timestamp, features, prediction, predicted_class, 
                       probabilities, request_id
                FROM predictions 
                ORDER BY timestamp DESC 
                LIMIT %s OFFSET %s;
            """
            
            cursor.execute(query, (limit, offset))
            results = cursor.fetchall()
            
            predictions = []
            for row in results:
                predictions.append({
                    'id': row[0],
                    'timestamp': row[1].isoformat(),
                    'features': json.loads(row[2]),
                    'prediction': row[3],
                    'predicted_class': row[4],
                    'probabilities': json.loads(row[5]),
                    'request_id': row[6]
                })
            
            return jsonify({
                'predictions': predictions,
                'total_returned': len(predictions),
                'limit': limit,
                'offset': offset
            })
            
    except Exception as e:
        logger.error(f"Error retrieving prediction history: {str(e)}")
        return jsonify({'error': str(e)}), 500

@app.route('/predictions/stats', methods=['GET'])
def get_prediction_stats():
    """Get prediction statistics"""
    try:
        with get_db_connection() as conn:
            cursor = conn.cursor()
            
            # Get total predictions
            cursor.execute("SELECT COUNT(*) FROM predictions;")
            total_predictions = cursor.fetchone()[0]
            
            # Get predictions by class
            cursor.execute("""
                SELECT predicted_class, COUNT(*) 
                FROM predictions 
                GROUP BY predicted_class 
                ORDER BY COUNT(*) DESC;
            """)
            class_counts = dict(cursor.fetchall())
            
            # Get recent predictions (last 24 hours)
            cursor.execute("""
                SELECT COUNT(*) 
                FROM predictions 
                WHERE timestamp > NOW() - INTERVAL '24 hours';
            """)
            recent_predictions = cursor.fetchone()[0]
            
            return jsonify({
                'total_predictions': total_predictions,
                'predictions_by_class': class_counts,
                'recent_predictions_24h': recent_predictions,
                'timestamp': datetime.now().isoformat()
            })
            
    except Exception as e:
        logger.error(f"Error retrieving prediction stats: {str(e)}")
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    # Load model on startup
    load_model()
    
    # Run the app
    app.run(host='0.0.0.0', port=5000, debug=False)
Task 5: Create Docker Compose Configuration
Step 5.1: Create Docker Compose File
# Create Docker Compose configuration
nano docker-compose.yml
Add the following Docker Compose configuration:

version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: ml-postgres
    environment:
      POSTGRES_DB: ml_predictions
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - "5432:5432"
    networks:
      - ml-network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ML Flask Application - Instance 1
  ml-app-1:
    build: .
    container_name: ml-app-1
    environment:
      DB_HOST: postgres
      DB_NAME: ml_predictions
      DB_USER: postgres
      DB_PASSWORD: postgres
      DB_PORT: 5432
    volumes:
      - ./app/app_with_db.py:/app/app.py
    ports:
      - "5001:5000"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - ml-network
    restart: unless-stopped

  # ML Flask Application - Instance 2
  ml-app-2:
    build: .
    container_name: ml-app-2
    environment:
      DB_HOST: postgres
      DB_NAME: ml_predictions
      DB_USER: postgres
      DB_PASSWORD: postgres
      DB_PORT: 5432
    volumes:
      - ./app/app_with_db.py:/app/app.py
    ports:
      - "5002:5000"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - ml-network
    restart: unless-stopped

  # Nginx Load Balancer
  nginx:
    image: nginx:alpine
    container_name: ml-nginx
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    ports:
      - "80:80"
    depends_on:
      - ml-app-1
      - ml-app-2
    networks:
      - ml-network
    restart: unless-stopped

volumes:
  postgres_data:

networks:
  ml-network:
    driver: bridge
Step 5.2: Create Nginx Configuration
# Create nginx directory
mkdir nginx

# Create nginx configuration
nano nginx/nginx.conf
Add the following Nginx configuration:

events {
    worker_connections 1024;
}

http {
    upstream ml_backend {
        server ml-app-1:5000;
        server ml-app-2:5000;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://ml_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Health check
            proxy_connect_timeout 5s;
            proxy_send_timeout 10s;
            proxy_read_timeout 10s;
        }
        
        location /health {
            proxy_pass http://ml_backend/health;
            access_log off;
        }
    }
}
Step 5.3: Deploy with Docker Compose
# Start all services
docker-compose up -d

# Check service status
docker-compose ps

# View logs
docker-compose logs -f

# Check individual service logs
docker-compose logs ml-app-1
docker-compose logs postgres
Task 6: Test the ML API
Step 6.1: Create Test Scripts
# Create test directory
mkdir tests

# Create API test script
nano tests/test_api.py
Add the following test script:

import requests
import json
import time

# API base URL
BASE_URL = "http://localhost"

def test_health_check():
    """Test health check endpoint"""
    print("Testing health check...")
    response = requests.get(f"{BASE_URL}/health")
    print(f"Status: {response.status_code}")
    print(f"Response: {response.json()}")
    print("-" * 50)

def test_model_info():
    """Test model info endpoint"""
    print("Testing model info...")
    response = requests.get(f"{BASE_URL}/model/info")
    print(f"Status: {response.status_code}")
    print(f"Response: {json.dumps(response.json(), indent=2)}")
    print("-" * 50)

def test_single_prediction():
    """Test single prediction"""
    print("Testing single prediction...")
    
    # Test data for different iris types
    test_cases = [
        {
            "name": "Setosa",
            "features": [5.1, 3.5, 1.4, 0.2]
        },
        {
            "name": "Versicolor", 
            "features": [6.2, 2.9, 4.3, 1.3]
        },
        {
            "name": "Virginica",
            "features": [7.3, 2.9, 6.3, 1.8]
        }
    ]
    
    for test_case in test_cases:
        print(f"Testing {test_case['name']}...")
        response = requests.post(
            f"{BASE_URL}/predict",
            json={"features": test_case["features"]}
        )
        print(f"Status: {response.status_code}")
        if response.status_code == 200:
            result = response.json()
            print(f"Predicted: {result['predicted_class']}")
            print(f"Confidence: {max(result['probabilities'].values()):.2f}")
        else:
            print(f"Error: {response.text}")
        print("-" * 30)
    
    print("-" * 50)

def test_batch_prediction():
    """Test batch prediction"""
    print("Testing batch prediction...")
    
    batch_data = {
        "batch_features": [
            [5.1, 3.5, 1.4, 0.2],
            [6.2, 2.9, 4.3, 1.3],
            [7.3, 2.9, 6.3, 1.8],
            [5.0, 3.0, 1.6, 0.2]
        ]
    }
    
    response = requests.post(f"{BASE_URL}/predict/batch", json=batch_data)
    print(f"Status: {response.status_code}")
    
    if response.status_code == 200:
        result = response.json()
        print(f"Total predictions: {result['total_predictions']}")
        for pred in result['results']:
            if 'error' not in pred:
                print(f"Index {pred['index']}: {pred['predicted_class']} (confidence: {pred['confidence']:.2f})")
            else:
                print(f"Index {pred['index']}: Error - {pred['error']}")
    else:
        print(f"Error: {response.text}")
    
    print("-" * 50)

def test_prediction_history():
    """Test prediction history"""
    print("Testing prediction history...")
    response = requests.get(f"{BASE_URL}/predictions/history?limit=5")
    print(f"Status: {response.status_code}")
    
    if response.status_code == 200:
        result = response.json()
        print(f"Retrieved {result['total_returned']} predictions")
        for pred in result['predictions'][:3]:  # Show first 3
            print(f"ID: {pred['id']}, Class: {pred['predicted_class']}, Time: {pred['timestamp']}")
    else:
        print(f"Error: {response.text}")
    
    print("-" * 50)

def test
