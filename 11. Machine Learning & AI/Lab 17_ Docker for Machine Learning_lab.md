Lab 17: Docker for Machine Learning
Objectives
By the end of this lab, students will be able to:

Create Dockerfiles optimized for machine learning environments
Build and run containerized machine learning models using TensorFlow
Set up Jupyter Notebooks inside Docker containers for interactive ML development
Implement persistent storage for trained models using Docker volumes
Design and deploy multi-service ML applications using Docker Compose
Understand best practices for containerizing machine learning workflows
Prerequisites
Before starting this lab, students should have:

Basic understanding of machine learning concepts
Familiarity with Python programming
Basic knowledge of Docker commands and concepts
Understanding of command-line interface operations
Basic knowledge of Jupyter Notebooks
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install Docker manually.

Your cloud machine includes:

Ubuntu 20.04 LTS
Docker Engine (latest version)
Docker Compose
Python 3.8+
Git
Text editors (nano, vim)
Task 1: Create a Dockerfile for Machine Learning Environment
Subtask 1.1: Create Project Directory Structure
First, let's create a organized directory structure for our machine learning project.

# Create main project directory
mkdir ml-docker-lab
cd ml-docker-lab

# Create subdirectories for organization
mkdir -p {notebooks,models,data,src}

# Create necessary files
touch Dockerfile requirements.txt docker-compose.yml
Subtask 1.2: Create Requirements File
Create a requirements file that includes all necessary Python packages for machine learning.

nano requirements.txt
Add the following content to requirements.txt:

tensorflow==2.13.0
numpy==1.24.3
pandas==2.0.3
matplotlib==3.7.2
seaborn==0.12.2
scikit-learn==1.3.0
jupyter==1.0.0
jupyterlab==4.0.5
flask==2.3.3
requests==2.31.0
pillow==10.0.0
Subtask 1.3: Create the Dockerfile
Now, create a Dockerfile optimized for machine learning workloads.

nano Dockerfile
Add the following content to Dockerfile:

# Use official Python runtime as base image
FROM python:3.9-slim

# Set maintainer information
LABEL maintainer="ML Docker Lab"
LABEL description="Machine Learning Environment with TensorFlow"

# Set environment variables
ENV PYTHONUNBUFFERED=1
ENV PYTHONDONTWRITEBYTECODE=1
ENV JUPYTER_ENABLE_LAB=yes

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    curl \
    software-properties-common \
    git \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements file
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Create directories for ML workflow
RUN mkdir -p /app/notebooks /app/models /app/data /app/src

# Copy application files
COPY . .

# Create a non-root user for security
RUN useradd -m -u 1000 mluser && \
    chown -R mluser:mluser /app
USER mluser

# Expose ports for Jupyter and Flask
EXPOSE 8888 5000

# Default command to start Jupyter Lab
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--port=8888", "--no-browser", "--allow-root", "--NotebookApp.token=''", "--NotebookApp.password=''"]
Subtask 1.4: Build the Docker Image
Build your machine learning Docker image.

# Build the Docker image
docker build -t ml-tensorflow:latest .

# Verify the image was created
docker images | grep ml-tensorflow
Task 2: Build and Run a Containerized Machine Learning Model
Subtask 2.1: Create a Simple ML Model Script
Create a Python script that demonstrates a basic machine learning workflow.

nano src/simple_ml_model.py
Add the following content:

#!/usr/bin/env python3
"""
Simple Machine Learning Model for Docker Demo
"""

import numpy as np
import pandas as pd
import tensorflow as tf
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
import matplotlib.pyplot as plt
import os
import pickle

def create_sample_data():
    """Create sample dataset for demonstration"""
    np.random.seed(42)
    
    # Generate synthetic data
    n_samples = 1000
    X = np.random.randn(n_samples, 4)
    
    # Create target variable with some relationship to features
    y = (X[:, 0] * 2 + X[:, 1] * 1.5 - X[:, 2] * 0.5 + X[:, 3] * 0.8 + 
         np.random.randn(n_samples) * 0.1)
    
    # Convert to binary classification
    y = (y > np.median(y)).astype(int)
    
    return X, y

def build_model(input_shape):
    """Build a simple neural network model"""
    model = tf.keras.Sequential([
        tf.keras.layers.Dense(64, activation='relu', input_shape=(input_shape,)),
        tf.keras.layers.Dropout(0.2),
        tf.keras.layers.Dense(32, activation='relu'),
        tf.keras.layers.Dropout(0.2),
        tf.keras.layers.Dense(1, activation='sigmoid')
    ])
    
    model.compile(
        optimizer='adam',
        loss='binary_crossentropy',
        metrics=['accuracy']
    )
    
    return model

def train_model():
    """Main training function"""
    print("Creating sample data...")
    X, y = create_sample_data()
    
    # Split the data
    X_train, X_test, y_train, y_test = train_test_split(
        X, y, test_size=0.2, random_state=42
    )
    
    # Scale the features
    scaler = StandardScaler()
    X_train_scaled = scaler.fit_transform(X_train)
    X_test_scaled = scaler.transform(X_test)
    
    print("Building model...")
    model = build_model(X_train.shape[1])
    
    print("Training model...")
    history = model.fit(
        X_train_scaled, y_train,
        epochs=50,
        batch_size=32,
        validation_split=0.2,
        verbose=1
    )
    
    # Evaluate model
    test_loss, test_accuracy = model.evaluate(X_test_scaled, y_test, verbose=0)
    print(f"Test Accuracy: {test_accuracy:.4f}")
    
    # Save model and scaler
    os.makedirs('/app/models', exist_ok=True)
    model.save('/app/models/simple_model.h5')
    
    with open('/app/models/scaler.pkl', 'wb') as f:
        pickle.dump(scaler, f)
    
    print("Model saved successfully!")
    
    return model, history, test_accuracy

if __name__ == "__main__":
    train_model()
Subtask 2.2: Run the Containerized Model
Run your container and execute the machine learning model.

# Run the container with volume mounting for model persistence
docker run -it --rm \
    -v $(pwd)/models:/app/models \
    -v $(pwd)/data:/app/data \
    ml-tensorflow:latest \
    python src/simple_ml_model.py
Subtask 2.3: Verify Model Training
Check if the model was saved successfully.

# List the contents of the models directory
ls -la models/

# You should see:
# simple_model.h5
# scaler.pkl
Task 3: Integrate Jupyter Notebooks Inside the Container
Subtask 3.1: Create a Sample Jupyter Notebook
Create a Jupyter notebook for interactive ML development.

mkdir -p notebooks
nano notebooks/ml_exploration.ipynb
Add the following content to create a basic notebook structure:

{
 "cells": [
  {
   "cell_type": "markdown",
   "metadata": {},
   "source": [
    "# Machine Learning Exploration in Docker\n",
    "\n",
    "This notebook demonstrates machine learning workflows inside a Docker container."
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "import tensorflow as tf\n",
    "import numpy as np\n",
    "import pandas as pd\n",
    "import matplotlib.pyplot as plt\n",
    "import seaborn as sns\n",
    "\n",
    "print(f\"TensorFlow version: {tf.__version__}\")\n",
    "print(f\"NumPy version: {np.__version__}\")\n",
    "print(f\"Pandas version: {pd.__version__}\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Load and explore the Iris dataset\n",
    "from sklearn.datasets import load_iris\n",
    "from sklearn.model_selection import train_test_split\n",
    "from sklearn.preprocessing import StandardScaler\n",
    "\n",
    "# Load dataset\n",
    "iris = load_iris()\n",
    "X, y = iris.data, iris.target\n",
    "\n",
    "# Create DataFrame for easier exploration\n",
    "df = pd.DataFrame(X, columns=iris.feature_names)\n",
    "df['target'] = y\n",
    "df['species'] = df['target'].map({0: 'setosa', 1: 'versicolor', 2: 'virginica'})\n",
    "\n",
    "print(\"Dataset shape:\", df.shape)\n",
    "df.head()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Visualize the data\n",
    "plt.figure(figsize=(12, 8))\n",
    "sns.pairplot(df, hue='species', markers=['o', 's', 'D'])\n",
    "plt.suptitle('Iris Dataset Pairplot', y=1.02)\n",
    "plt.show()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Build and train a neural network\n",
    "from tensorflow.keras.utils import to_categorical\n",
    "\n",
    "# Prepare data\n",
    "X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)\n",
    "\n",
    "# Scale features\n",
    "scaler = StandardScaler()\n",
    "X_train_scaled = scaler.fit_transform(X_train)\n",
    "X_test_scaled = scaler.transform(X_test)\n",
    "\n",
    "# Convert labels to categorical\n",
    "y_train_cat = to_categorical(y_train)\n",
    "y_test_cat = to_categorical(y_test)\n",
    "\n",
    "# Build model\n",
    "model = tf.keras.Sequential([\n",
    "    tf.keras.layers.Dense(64, activation='relu', input_shape=(4,)),\n",
    "    tf.keras.layers.Dropout(0.2),\n",
    "    tf.keras.layers.Dense(32, activation='relu'),\n",
    "    tf.keras.layers.Dense(3, activation='softmax')\n",
    "])\n",
    "\n",
    "model.compile(optimizer='adam', loss='categorical_crossentropy', metrics=['accuracy'])\n",
    "model.summary()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Train the model\n",
    "history = model.fit(\n",
    "    X_train_scaled, y_train_cat,\n",
    "    epochs=100,\n",
    "    batch_size=16,\n",
    "    validation_split=0.2,\n",
    "    verbose=1\n",
    ")\n",
    "\n",
    "# Evaluate on test set\n",
    "test_loss, test_accuracy = model.evaluate(X_test_scaled, y_test_cat, verbose=0)\n",
    "print(f\"Test Accuracy: {test_accuracy:.4f}\")"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Plot training history\n",
    "plt.figure(figsize=(12, 4))\n",
    "\n",
    "plt.subplot(1, 2, 1)\n",
    "plt.plot(history.history['loss'], label='Training Loss')\n",
    "plt.plot(history.history['val_loss'], label='Validation Loss')\n",
    "plt.title('Model Loss')\n",
    "plt.xlabel('Epoch')\n",
    "plt.ylabel('Loss')\n",
    "plt.legend()\n",
    "\n",
    "plt.subplot(1, 2, 2)\n",
    "plt.plot(history.history['accuracy'], label='Training Accuracy')\n",
    "plt.plot(history.history['val_accuracy'], label='Validation Accuracy')\n",
    "plt.title('Model Accuracy')\n",
    "plt.xlabel('Epoch')\n",
    "plt.ylabel('Accuracy')\n",
    "plt.legend()\n",
    "\n",
    "plt.tight_layout()\n",
    "plt.show()"
   ]
  },
  {
   "cell_type": "code",
   "execution_count": null,
   "metadata": {},
   "outputs": [],
   "source": [
    "# Save the trained model\n",
    "import os\n",
    "os.makedirs('/app/models', exist_ok=True)\n",
    "model.save('/app/models/iris_model.h5')\n",
    "print(\"Model saved to /app/models/iris_model.h5\")"
   ]
  }
 ],
 "metadata": {
  "kernelspec": {
   "display_name": "Python 3",
   "language": "python",
   "name": "python3"
  },
  "language_info": {
   "codemirror_mode": {
    "name": "ipython",
    "version": 3
   },
   "file_extension": ".py",
   "mimetype": "text/x-python",
   "name": "python",
   "nbconvert_exporter": "python",
   "pygments_lexer": "ipython3",
   "version": "3.9.0"
  }
 },
 "nbformat": 4,
 "nbformat_minor": 4
}
Subtask 3.2: Run Jupyter Lab in Container
Start the container with Jupyter Lab for interactive development.

# Run container with Jupyter Lab
docker run -it --rm \
    -p 8888:8888 \
    -v $(pwd)/notebooks:/app/notebooks \
    -v $(pwd)/models:/app/models \
    -v $(pwd)/data:/app/data \
    ml-tensorflow:latest
Subtask 3.3: Access Jupyter Lab
After running the container, you should see output similar to:

[I 2024-01-01 12:00:00.000 ServerApp] Jupyter Server 2.7.0 is running at:
[I 2024-01-01 12:00:00.000 ServerApp] http://0.0.0.0:8888/lab
Open your web browser and navigate to:

http://localhost:8888/lab
You can now access Jupyter Lab running inside your Docker container and work with the notebook interactively.

Task 4: Save and Load Trained Models from Docker Volumes
Subtask 4.1: Create Model Management Script
Create a script to demonstrate saving and loading models with Docker volumes.

nano src/model_manager.py
Add the following content:

#!/usr/bin/env python3
"""
Model Management Script for Docker Volumes
"""

import tensorflow as tf
import numpy as np
import pickle
import os
import json
from datetime import datetime

class ModelManager:
    def __init__(self, models_dir="/app/models"):
        self.models_dir = models_dir
        os.makedirs(models_dir, exist_ok=True)
    
    def save_model_with_metadata(self, model, model_name, metadata=None):
        """Save model with metadata to Docker volume"""
        model_path = os.path.join(self.models_dir, f"{model_name}.h5")
        metadata_path = os.path.join(self.models_dir, f"{model_name}_metadata.json")
        
        # Save the model
        model.save(model_path)
        
        # Create metadata
        if metadata is None:
            metadata = {}
        
        metadata.update({
            "model_name": model_name,
            "saved_at": datetime.now().isoformat(),
            "model_path": model_path,
            "tensorflow_version": tf.__version__
        })
        
        # Save metadata
        with open(metadata_path, 'w') as f:
            json.dump(metadata, f, indent=2)
        
        print(f"Model saved: {model_path}")
        print(f"Metadata saved: {metadata_path}")
    
    def load_model_with_metadata(self, model_name):
        """Load model with metadata from Docker volume"""
        model_path = os.path.join(self.models_dir, f"{model_name}.h5")
        metadata_path = os.path.join(self.models_dir, f"{model_name}_metadata.json")
        
        if not os.path.exists(model_path):
            raise FileNotFoundError(f"Model not found: {model_path}")
        
        # Load model
        model = tf.keras.models.load_model(model_path)
        
        # Load metadata
        metadata = {}
        if os.path.exists(metadata_path):
            with open(metadata_path, 'r') as f:
                metadata = json.load(f)
        
        print(f"Model loaded: {model_path}")
        print(f"Metadata: {metadata}")
        
        return model, metadata
    
    def list_saved_models(self):
        """List all saved models in the volume"""
        models = []
        for file in os.listdir(self.models_dir):
            if file.endswith('.h5'):
                model_name = file.replace('.h5', '')
                models.append(model_name)
        
        return models
    
    def save_preprocessing_objects(self, objects_dict, name):
        """Save preprocessing objects like scalers"""
        file_path = os.path.join(self.models_dir, f"{name}_preprocessing.pkl")
        
        with open(file_path, 'wb') as f:
            pickle.dump(objects_dict, f)
        
        print(f"Preprocessing objects saved: {file_path}")
    
    def load_preprocessing_objects(self, name):
        """Load preprocessing objects"""
        file_path = os.path.join(self.models_dir, f"{name}_preprocessing.pkl")
        
        if not os.path.exists(file_path):
            raise FileNotFoundError(f"Preprocessing objects not found: {file_path}")
        
        with open(file_path, 'rb') as f:
            objects = pickle.load(f)
        
        print(f"Preprocessing objects loaded: {file_path}")
        return objects

def demo_model_persistence():
    """Demonstrate model persistence with Docker volumes"""
    print("=== Model Persistence Demo ===")
    
    # Initialize model manager
    manager = ModelManager()
    
    # Create a simple model
    model = tf.keras.Sequential([
        tf.keras.layers.Dense(64, activation='relu', input_shape=(10,)),
        tf.keras.layers.Dense(32, activation='relu'),
        tf.keras.layers.Dense(1, activation='sigmoid')
    ])
    
    model.compile(optimizer='adam', loss='binary_crossentropy', metrics=['accuracy'])
    
    # Create dummy data for demonstration
    X_dummy = np.random.randn(100, 10)
    y_dummy = np.random.randint(0, 2, 100)
    
    # Train briefly
    print("Training demo model...")
    model.fit(X_dummy, y_dummy, epochs=5, verbose=0)
    
    # Save model with metadata
    metadata = {
        "description": "Demo model for Docker volume persistence",
        "input_shape": [10],
        "output_shape": [1],
        "task_type": "binary_classification"
    }
    
    manager.save_model_with_metadata(model, "demo_model", metadata)
    
    # Save preprocessing objects
    from sklearn.preprocessing import StandardScaler
    scaler = StandardScaler()
    scaler.fit(X_dummy)
    
    preprocessing_objects = {
        "scaler": scaler,
        "feature_names": [f"feature_{i}" for i in range(10)]
    }
    
    manager.save_preprocessing_objects(preprocessing_objects, "demo_model")
    
    # List saved models
    print("\nSaved models:")
    for model_name in manager.list_saved_models():
        print(f"  - {model_name}")
    
    # Load model back
    print("\nLoading model...")
    loaded_model, loaded_metadata = manager.load_model_with_metadata("demo_model")
    loaded_preprocessing = manager.load_preprocessing_objects("demo_model")
    
    # Test loaded model
    print("\nTesting loaded model...")
    test_input = np.random.randn(1, 10)
    prediction = loaded_model.predict(test_input, verbose=0)
    print(f"Prediction: {prediction[0][0]:.4f}")
    
    print("\n=== Demo Complete ===")

if __name__ == "__main__":
    demo_model_persistence()
Subtask 4.2: Test Model Persistence
Run the model management script to test saving and loading models.

# Run the model persistence demo
docker run -it --rm \
    -v $(pwd)/models:/app/models \
    ml-tensorflow:latest \
    python src/model_manager.py
Subtask 4.3: Verify Persistent Storage
Check that models persist across container restarts.

# List files in the models directory
ls -la models/

# Start a new container and list models
docker run -it --rm \
    -v $(pwd)/models:/app/models \
    ml-tensorflow:latest \
    python -c "
from src.model_manager import ModelManager
manager = ModelManager()
print('Available models:', manager.list_saved_models())
"
Task 5: Create Docker Compose Setup for Multi-Service ML Application
Subtask 5.1: Create Flask API for Model Serving
Create a Flask application to serve the machine learning model.

nano src/ml_api.py
Add the following content:

#!/usr/bin/env python3
"""
Flask API for serving machine learning models
"""

from flask import Flask, request, jsonify
import tensorflow as tf
import numpy as np
import pickle
import os
import json
from datetime import datetime

app = Flask(__name__)

# Global variables for model and preprocessing
model = None
scaler = None
model_metadata = None

def load_model_and_preprocessing():
    """Load model and preprocessing objects at startup"""
    global model, scaler, model_metadata
    
    models_dir = "/app/models"
    
    try:
        # Load the iris model if it exists
        model_path = os.path.join(models_dir, "iris_model.h5")
        if os.path.exists(model_path):
            model = tf.keras.models.load_model(model_path)
            print(f"Model loaded from {model_path}")
        
        # Load preprocessing objects
        preprocessing_path = os.path.join(models_dir, "iris_preprocessing.pkl")
        if os.path.exists(preprocessing_path):
            with open(preprocessing_path, 'rb') as f:
                preprocessing_objects = pickle.load(f)
                scaler = preprocessing_objects.get('scaler')
            print(f"Preprocessing objects loaded from {preprocessing_path}")
        
        # Load metadata
        metadata_path = os.path.join(models_dir, "iris_model_metadata.json")
        if os.path.exists(metadata_path):
            with open(metadata_path, 'r') as f:
                model_metadata = json.load(f)
            print(f"Metadata loaded from {metadata_path}")
    
    except Exception as e:
        print(f"Error loading model: {e}")

@app.route('/health', methods=['GET'])
def health_check():
    """Health check endpoint"""
    return jsonify({
        "status": "healthy",
        "timestamp": datetime.now().isoformat(),
        "model_loaded": model is not None,
        "scaler_loaded": scaler is not None
    })

@app.route('/model/info', methods=['GET'])
def model_info():
    """Get model information"""
    if model is None:
        return jsonify({"error": "Model not loaded"}), 400
    
    info = {
        "model_loaded": True,
        "input_shape": model.input_shape,
        "output_shape": model.output_shape,
        "metadata": model_metadata
    }
    
    return jsonify(info)

@app.route('/predict', methods=['POST'])
def predict():
    """Make predictions using the loaded model"""
    if model is None:
        return jsonify({"error": "Model not loaded"}), 400
    
    try:
        # Get input data from request
        data = request.get_json()
        
        if 'features' not in data:
            return jsonify({"error": "Missing 'features' in request"}), 400
        
        features = np.array(data['features'])
        
        # Ensure correct shape
        if len(features.shape) == 1:
            features = features.reshape(1, -1)
        
        # Apply preprocessing if scaler is available
        if scaler is not None:
            features = scaler.transform(features)
        
        # Make prediction
        predictions = model.predict(features)
        
        # Convert to list for JSON serialization
        predictions_list = predictions.tolist()
        
        # For classification, also return predicted classes
        predicted_classes = np.argmax(predictions, axis=1).tolist()
        
        response = {
            "predictions": predictions_list,
            "predicted_classes": predicted_classes,
            "timestamp": datetime.now().isoformat()
        }
        
        return jsonify(response)
    
    except Exception as e:
        return jsonify({"error": str(e)}), 500

@app.route('/predict/iris', methods=['POST'])
def predict_iris():
    """Specific endpoint for iris classification"""
    if model is None:
        return jsonify({"error": "Model not loaded"}), 400
    
    try:
        data = request.get_json()
        
        # Expected features: sepal_length, sepal_width, petal_length, petal_width
        required_features = ['sepal_length', 'sepal_width', 'petal_length', 'petal_width']
        
        if not all(feature in data for feature in required_features):
            return jsonify({
                "error": f"Missing required features: {required_features}"
            }), 400
        
        # Extract features in correct order
        features = np.array([[
            data['sepal_length'],
            data['sepal_width'],
            data['petal_length'],
            data['petal_width']
        ]])
        
        # Apply preprocessing
        if scaler is not None:
            features = scaler.transform(features)
        
        # Make prediction
        predictions = model.predict(features)
        predicted_class = np.argmax(predictions[0])
        confidence = float(np.max(predictions[0]))
        
        # Map class to species name
        species_map = {0: 'setosa', 1: 'versicolor', 2: 'virginica'}
        predicted_species = species_map.get(predicted_class, 'unknown')
        
        response = {
            "predicted_species": predicted_species,
            "predicted_class": int(predicted_class),
            "confidence": confidence,
            "probabilities": {
                "setosa": float(predictions[0][0]),
                "versicolor": float(predictions[0][1]),
                "virginica": float(predictions[0][2])
            },
            "timestamp": datetime.now().isoformat()
        }
        
        return jsonify(response)
    
    except Exception as e:
        return jsonify({"error": str(e)}), 500

if __name__ == '__main__':
    # Load model at startup
    load_model_and_preprocessing()
    
    # Run the Flask app
    app.run(host='0.0.0.0', port=5000, debug=True)
Subtask 5.2: Create Database Setup Script
Create a script to set up a PostgreSQL database for storing predictions.

nano src/database_setup.py
Add the following content:

#!/usr/bin/env python3
"""
Database setup script for ML predictions
"""

import psycopg2
import os
import time
from datetime import datetime

def wait_for_db(host, port, database, user, password, max_retries=30):
    """Wait for database to be ready"""
    for i in range(max_retries):
        try:
            conn = psycopg2.connect(
                host=host,
                port=port,
                database=database,
                user=user,
                password=password
            )
            conn.close()
            print("Database is ready!")
            return True
        except psycopg2.OperationalError:
            print(f"Waiting for database... ({i+1}/{max_retries})")
            time.sleep(2)
    
    return False

def create_tables():
    """Create necessary tables for ML application"""
    
    # Database connection parameters
    db_params = {
        'host': os.getenv('DB_HOST', 'postgres'),
        'port': os.getenv('DB_PORT', '5432'),
        'database': os.getenv('DB_NAME', 'mldb'),
        'user': os.getenv('DB_USER', 'mluser'),
        'password': os.getenv('DB_PASSWORD', 'mlpassword')
    }
    
    # Wait for database to be ready
    if not wait_for_db(**db_params):
        print("Database not ready, exiting...")
        return False
    
    try:
        # Connect to database
        conn = psycopg2.connect(**db_params)
        cursor = conn.cursor()
        
        # Create predictions table
        cursor.execute
