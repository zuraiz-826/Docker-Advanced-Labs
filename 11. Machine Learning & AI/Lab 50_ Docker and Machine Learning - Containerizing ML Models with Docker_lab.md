Start Lab
Connect
Delete Lab

Video
Lab Material
Lab 50: Docker and Machine Learning - Containerizing ML Models with Docker
Objectives
By the end of this lab, students will be able to:

Create a Dockerfile for containerizing machine learning models
Build and run Docker containers containing ML models
Expose machine learning models as REST APIs using Flask
Connect ML containers to PostgreSQL databases for data retrieval
Deploy containerized ML models using Kubernetes
Understand the benefits of containerizing ML workflows for production deployment
Prerequisites
Before starting this lab, students should have:

Basic understanding of machine learning concepts
Familiarity with Python programming
Basic knowledge of Docker containers and commands
Understanding of REST APIs and HTTP methods
Basic knowledge of SQL and databases
Familiarity with command-line interface
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment. No need to build your own VM or install software locally.

Your cloud machine includes:

Docker Engine
Python 3.8+
pip package manager
kubectl (Kubernetes CLI)
PostgreSQL client tools
Text editors (nano, vim)
Task 1: Create a Dockerfile for a Machine Learning Model
Subtask 1.1: Prepare the Machine Learning Model
First, let's create a simple machine learning model using Scikit-learn that predicts house prices.

Create a project directory and navigate to it:
mkdir ml-docker-lab
cd ml-docker-lab
Create the main application file:
nano app.py
Add the following Python code for a simple linear regression model:
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LinearRegression
from sklearn.metrics import mean_squared_error
import joblib
import os

class HousePricePredictor:
    def __init__(self):
        self.model = LinearRegression()
        self.is_trained = False
    
    def generate_sample_data(self, n_samples=1000):
        """Generate sample house data for training"""
        np.random.seed(42)
        
        # Generate features: size, bedrooms, age
        size = np.random.normal(2000, 500, n_samples)
        bedrooms = np.random.randint(1, 6, n_samples)
        age = np.random.randint(0, 50, n_samples)
        
        # Generate price based on features with some noise
        price = (size * 100 + bedrooms * 5000 - age * 1000 + 
                np.random.normal(0, 10000, n_samples))
        
        # Ensure positive prices
        price = np.maximum(price, 50000)
        
        data = pd.DataFrame({
            'size': size,
            'bedrooms': bedrooms,
            'age': age,
            'price': price
        })
        
        return data
    
    def train_model(self):
        """Train the model with sample data"""
        print("Generating training data...")
        data = self.generate_sample_data()
        
        X = data[['size', 'bedrooms', 'age']]
        y = data['price']
        
        X_train, X_test, y_train, y_test = train_test_split(
            X, y, test_size=0.2, random_state=42
        )
        
        print("Training model...")
        self.model.fit(X_train, y_train)
        
        # Evaluate model
        y_pred = self.model.predict(X_test)
        mse = mean_squared_error(y_test, y_pred)
        print(f"Model trained successfully. MSE: {mse:.2f}")
        
        self.is_trained = True
        
        # Save the model
        joblib.dump(self.model, 'house_price_model.pkl')
        print("Model saved as house_price_model.pkl")
    
    def load_model(self):
        """Load a pre-trained model"""
        if os.path.exists('house_price_model.pkl'):
            self.model = joblib.load('house_price_model.pkl')
            self.is_trained = True
            print("Model loaded successfully")
        else:
            print("No saved model found. Training new model...")
            self.train_model()
    
    def predict(self, size, bedrooms, age):
        """Make a prediction"""
        if not self.is_trained:
            self.load_model()
        
        features = np.array([[size, bedrooms, age]])
        prediction = self.model.predict(features)[0]
        return max(prediction, 0)  # Ensure non-negative price

if __name__ == "__main__":
    predictor = HousePricePredictor()
    predictor.train_model()
Create the Flask API wrapper:
nano flask_app.py
Add the Flask application code:
from flask import Flask, request, jsonify
import psycopg2
import os
from app import HousePricePredictor
import json

app = Flask(__name__)

# Initialize the ML model
predictor = HousePricePredictor()

# Database configuration
DB_CONFIG = {
    'host': os.getenv('DB_HOST', 'localhost'),
    'database': os.getenv('DB_NAME', 'mldata'),
    'user': os.getenv('DB_USER', 'postgres'),
    'password': os.getenv('DB_PASSWORD', 'password'),
    'port': os.getenv('DB_PORT', '5432')
}

def get_db_connection():
    """Create database connection"""
    try:
        conn = psycopg2.connect(**DB_CONFIG)
        return conn
    except Exception as e:
        print(f"Database connection error: {e}")
        return None

@app.route('/health', methods=['GET'])
def health_check():
    """Health check endpoint"""
    return jsonify({
        'status': 'healthy',
        'model_loaded': predictor.is_trained
    })

@app.route('/predict', methods=['POST'])
def predict_price():
    """Predict house price"""
    try:
        data = request.get_json()
        
        # Validate input
        required_fields = ['size', 'bedrooms', 'age']
        for field in required_fields:
            if field not in data:
                return jsonify({'error': f'Missing field: {field}'}), 400
        
        # Make prediction
        prediction = predictor.predict(
            data['size'], 
            data['bedrooms'], 
            data['age']
        )
        
        # Store prediction in database if available
        store_prediction(data, prediction)
        
        return jsonify({
            'prediction': round(prediction, 2),
            'input': data
        })
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/predictions', methods=['GET'])
def get_predictions():
    """Get stored predictions from database"""
    try:
        conn = get_db_connection()
        if not conn:
            return jsonify({'error': 'Database connection failed'}), 500
        
        cursor = conn.cursor()
        cursor.execute("""
            SELECT id, size, bedrooms, age, predicted_price, created_at 
            FROM predictions 
            ORDER BY created_at DESC 
            LIMIT 10
        """)
        
        results = cursor.fetchall()
        predictions = []
        
        for row in results:
            predictions.append({
                'id': row[0],
                'size': row[1],
                'bedrooms': row[2],
                'age': row[3],
                'predicted_price': float(row[4]),
                'created_at': row[5].isoformat()
            })
        
        cursor.close()
        conn.close()
        
        return jsonify({'predictions': predictions})
    
    except Exception as e:
        return jsonify({'error': str(e)}), 500

def store_prediction(input_data, prediction):
    """Store prediction in database"""
    try:
        conn = get_db_connection()
        if not conn:
            return
        
        cursor = conn.cursor()
        cursor.execute("""
            INSERT INTO predictions (size, bedrooms, age, predicted_price)
            VALUES (%s, %s, %s, %s)
        """, (input_data['size'], input_data['bedrooms'], input_data['age'], prediction))
        
        conn.commit()
        cursor.close()
        conn.close()
        
    except Exception as e:
        print(f"Error storing prediction: {e}")

@app.route('/train', methods=['POST'])
def retrain_model():
    """Retrain the model"""
    try:
        predictor.train_model()
        return jsonify({'message': 'Model retrained successfully'})
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    # Load or train model on startup
    predictor.load_model()
    app.run(host='0.0.0.0', port=5000, debug=False)
Subtask 1.2: Create Requirements File
Create a requirements file for Python dependencies:

nano requirements.txt
Add the following dependencies:

Flask==2.3.3
scikit-learn==1.3.0
pandas==2.0.3
numpy==1.24.3
joblib==1.3.2
psycopg2-binary==2.9.7
gunicorn==21.2.0
Subtask 1.3: Create the Dockerfile
Now create the Dockerfile to containerize the ML application:

nano Dockerfile
Add the following Dockerfile content:

# Use official Python runtime as base image
FROM python:3.9-slim

# Set working directory in container
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements first for better caching
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Create directory for model files
RUN mkdir -p /app/models

# Expose port 5000
EXPOSE 5000

# Create non-root user for security
RUN useradd -m -u 1000 mluser && chown -R mluser:mluser /app
USER mluser

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:5000/health || exit 1

# Run the application
CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "flask_app:app"]
Task 2: Build the Docker Image and Run the Model Container
Subtask 2.1: Build the Docker Image
Build the Docker image:
docker build -t ml-house-predictor:v1.0 .
Verify the image was created:
docker images | grep ml-house-predictor
Subtask 2.2: Run the Container
Run the container in detached mode:
docker run -d \
  --name ml-predictor \
  -p 5000:5000 \
  ml-house-predictor:v1.0
Check if the container is running:
docker ps
View container logs:
docker logs ml-predictor
Subtask 2.3: Test the ML API
Test the health endpoint:
curl http://localhost:5000/health
Test the prediction endpoint:
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "size": 2500,
    "bedrooms": 3,
    "age": 10
  }'
Task 3: Expose the Model as a REST API using Flask
The Flask API is already implemented in the previous steps. Let's enhance it with additional endpoints and documentation.

Subtask 3.1: Add API Documentation Endpoint
Stop the current container:
docker stop ml-predictor
docker rm ml-predictor
Update the Flask application to include documentation:
nano flask_app.py
Add the following route at the end of the file (before the if __name__ == '__main__': block):
@app.route('/', methods=['GET'])
def api_documentation():
    """API documentation"""
    docs = {
        'title': 'House Price Prediction API',
        'version': '1.0',
        'endpoints': {
            'GET /': 'API documentation',
            'GET /health': 'Health check',
            'POST /predict': 'Predict house price (requires: size, bedrooms, age)',
            'GET /predictions': 'Get recent predictions from database',
            'POST /train': 'Retrain the model'
        },
        'example_request': {
            'url': '/predict',
            'method': 'POST',
            'body': {
                'size': 2500,
                'bedrooms': 3,
                'age': 10
            }
        }
    }
    return jsonify(docs)
Subtask 3.2: Rebuild and Test Enhanced API
Rebuild the Docker image:
docker build -t ml-house-predictor:v1.1 .
Run the updated container:
docker run -d \
  --name ml-predictor-v2 \
  -p 5000:5000 \
  ml-house-predictor:v1.1
Test the documentation endpoint:
curl http://localhost:5000/
Test multiple predictions:
# Prediction 1
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"size": 1800, "bedrooms": 2, "age": 5}'

# Prediction 2
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"size": 3000, "bedrooms": 4, "age": 15}'
Task 4: Connect the Model Container to a PostgreSQL Container
Subtask 4.1: Create PostgreSQL Container
Create a Docker network for container communication:
docker network create ml-network
Run PostgreSQL container:
docker run -d \
  --name postgres-ml \
  --network ml-network \
  -e POSTGRES_DB=mldata \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  postgres:13
Wait for PostgreSQL to start (about 10 seconds):
sleep 10
Subtask 4.2: Initialize Database Schema
Create the database schema:
docker exec -i postgres-ml psql -U postgres -d mldata << EOF
CREATE TABLE IF NOT EXISTS predictions (
    id SERIAL PRIMARY KEY,
    size REAL NOT NULL,
    bedrooms INTEGER NOT NULL,
    age INTEGER NOT NULL,
    predicted_price REAL NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert some sample data
INSERT INTO predictions (size, bedrooms, age, predicted_price) VALUES
(2000, 3, 5, 245000.50),
(1500, 2, 10, 185000.75),
(2800, 4, 2, 320000.25);
EOF
Verify the table was created:
docker exec -it postgres-ml psql -U postgres -d mldata -c "SELECT * FROM predictions;"
Subtask 4.3: Connect ML Container to Database
Stop the current ML container:
docker stop ml-predictor-v2
docker rm ml-predictor-v2
Run the ML container with database connection:
docker run -d \
  --name ml-predictor-db \
  --network ml-network \
  -p 5000:5000 \
  -e DB_HOST=postgres-ml \
  -e DB_NAME=mldata \
  -e DB_USER=postgres \
  -e DB_PASSWORD=password \
  -e DB_PORT=5432 \
  ml-house-predictor:v1.1
Test the database connection:
curl http://localhost:5000/predictions
Make a prediction to test database storage:
curl -X POST http://localhost:5000/predict \
  -H "Content-Type: application/json" \
  -d '{"size": 2200, "bedrooms": 3, "age": 8}'
Verify the prediction was stored:
curl http://localhost:5000/predictions
Task 5: Integrate the Docker Container into a Kubernetes Deployment
Subtask 5.1: Create Kubernetes Configuration Files
Create a directory for Kubernetes manifests:
mkdir k8s-manifests
cd k8s-manifests
Create PostgreSQL deployment and service:
nano postgres-deployment.yaml
Add the following content:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:13
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          value: "mldata"
        - name: POSTGRES_USER
          value: "postgres"
        - name: POSTGRES_PASSWORD
          value: "password"
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
  type: ClusterIP
Create ML application deployment and service:
nano ml-deployment.yaml
Add the following content:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: ml-predictor-deployment
  labels:
    app: ml-predictor
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ml-predictor
  template:
    metadata:
      labels:
        app: ml-predictor
    spec:
      containers:
      - name: ml-predictor
        image: ml-house-predictor:v1.1
        ports:
        - containerPort: 5000
        env:
        - name: DB_HOST
          value: "postgres-service"
        - name: DB_NAME
          value: "mldata"
        - name: DB_USER
          value: "postgres"
        - name: DB_PASSWORD
          value: "password"
        - name: DB_PORT
          value: "5432"
        livenessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 5000
          initialDelaySeconds: 5
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: ml-predictor-service
spec:
  selector:
    app: ml-predictor
  ports:
  - port: 80
    targetPort: 5000
  type: LoadBalancer
Create a ConfigMap for database initialization:
nano db-init-configmap.yaml
Add the following content:

apiVersion: v1
kind: ConfigMap
metadata:
  name: db-init-script
data:
  init.sql: |
    CREATE TABLE IF NOT EXISTS predictions (
        id SERIAL PRIMARY KEY,
        size REAL NOT NULL,
        bedrooms INTEGER NOT NULL,
        age INTEGER NOT NULL,
        predicted_price REAL NOT NULL,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
    );
    
    INSERT INTO predictions (size, bedrooms, age, predicted_price) VALUES
    (2000, 3, 5, 245000.50),
    (1500, 2, 10, 185000.75),
    (2800, 4, 2, 320000.25)
    ON CONFLICT DO NOTHING;
Subtask 5.2: Deploy to Kubernetes
Note: For this lab, we'll use Docker Desktop's built-in Kubernetes or minikube. In a production environment, you would use a managed Kubernetes service.

Start minikube (if not already running):
minikube start
Load the Docker image into minikube:
minikube image load ml-house-predictor:v1.1
Apply the Kubernetes manifests:
kubectl apply -f db-init-configmap.yaml
kubectl apply -f postgres-deployment.yaml
kubectl apply -f ml-deployment.yaml
Check the deployment status:
kubectl get deployments
kubectl get pods
kubectl get services
Subtask 5.3: Initialize Database in Kubernetes
Wait for PostgreSQL pod to be ready:
kubectl wait --for=condition=ready pod -l app=postgres --timeout=60s
Initialize the database:
kubectl exec -i deployment/postgres-deployment -- psql -U postgres -d mldata << EOF
CREATE TABLE IF NOT EXISTS predictions (
    id SERIAL PRIMARY KEY,
    size REAL NOT NULL,
    bedrooms INTEGER NOT NULL,
    age INTEGER NOT NULL,
    predicted_price REAL NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO predictions (size, bedrooms, age, predicted_price) VALUES
(2000, 3, 5, 245000.50),
(1500, 2, 10, 185000.75),
(2800, 4, 2, 320000.25);
EOF
Subtask 5.4: Test the Kubernetes Deployment
Get the service URL:
minikube service ml-predictor-service --url
Test the API (replace URL with the output from previous command):
# Get the service URL
SERVICE_URL=$(minikube service ml-predictor-service --url)

# Test health endpoint
curl $SERVICE_URL/health

# Test prediction
curl -X POST $SERVICE_URL/predict \
  -H "Content-Type: application/json" \
  -d '{"size": 2400, "bedrooms": 3, "age": 7}'

# Get predictions from database
curl $SERVICE_URL/predictions
Check pod logs:
kubectl logs -l app=ml-predictor
Scale the deployment:
kubectl scale deployment ml-predictor-deployment --replicas=3
kubectl get pods
Subtask 5.5: Monitor the Deployment
Check resource usage:
kubectl top pods
View deployment details:
kubectl describe deployment ml-predictor-deployment
Check service endpoints:
kubectl get endpoints
Troubleshooting Tips
Common Issues and Solutions
Container fails to start:

Check logs: docker logs <container-name>
Verify port availability: netstat -tulpn | grep :5000
Database connection fails:

Ensure containers are on the same network
Check environment variables
Verify PostgreSQL is ready: docker exec postgres-ml pg_isready
Kubernetes pods not starting:

Check pod status: kubectl describe pod <pod-name>
Verify image availability: kubectl get events
API returns errors:

Check application logs
Verify JSON format in requests
Ensure all required fields are provided
Model predictions seem incorrect:

The model uses synthetic data for demonstration
In production, use real training data
Consider model validation and testing
Cleanup
To clean up resources after completing the lab:

Stop and remove Docker containers:
docker stop ml-predictor-db postgres-ml
docker rm ml-predictor-db postgres-ml
docker network rm ml-network
Clean up Kubernetes resources:
kubectl delete -f ml-deployment.yaml
kubectl delete -f postgres-deployment.yaml
kubectl delete -f db-init-configmap.yaml
Stop minikube (if used):
minikube stop
Conclusion
In this comprehensive lab, you have successfully:

Containerized a machine learning model using Docker, making it portable and easy to deploy across different environments
Built and managed Docker images for ML applications with proper dependency management and security practices
Created a REST API using Flask to expose your ML model as a web service, enabling integration with other applications
Integrated database connectivity by connecting your ML container to a PostgreSQL database for data persistence and retrieval
Deployed to Kubernetes for production-ready orchestration with scaling, health checks, and service discovery
Why This Matters:

Containerizing ML models is crucial in modern MLOps practices because it:

Ensures consistency across development, testing, and production environments
Enables scalability through container orchestration platforms like Kubernetes
Facilitates continuous deployment and integration workflows
Provides isolation and security for ML workloads
Simplifies dependency management and reduces "it works on my machine" issues
This lab demonstrates industry best practices for deploying machine learning models in production environments, preparing you for real-world ML engineering challenges and supporting your journey toward Docker Certified Associate (DCA) certification.

The skills learned here are directly applicable to modern cloud-native ML platforms and are essential for anyone working in machine learning operations, DevOps, or cloud engineering roles.
