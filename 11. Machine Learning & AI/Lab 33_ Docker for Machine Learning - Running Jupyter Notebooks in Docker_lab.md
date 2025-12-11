Lab 33: Docker for Machine Learning - Running Jupyter Notebooks in Docker
Lab Objectives
By the end of this lab, you will be able to:

Understand how to containerize machine learning environments using Docker
Pull and run Jupyter Notebook Docker images from Docker Hub
Access Jupyter Notebooks through a web browser from a containerized environment
Install and configure machine learning libraries within Docker containers
Build and train simple machine learning models in a containerized Jupyter environment
Implement data persistence using Docker volumes for model storage
Apply best practices for Docker-based machine learning workflows
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, volumes)
Familiarity with command-line interface operations
Basic knowledge of Python programming
Understanding of machine learning fundamentals
Access to a web browser for Jupyter Notebook interface
Note: Al Nafi provides ready-to-use Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to begin - no need to build your own virtual machine or install Docker manually.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with:

Ubuntu Linux operating system
Docker Engine installed and running
Internet connectivity for pulling Docker images
All necessary permissions configured
Task 1: Pull Jupyter Notebook Image from Docker Hub
Subtask 1.1: Verify Docker Installation
First, let's confirm that Docker is properly installed and running on your system.

# Check Docker version
docker --version

# Verify Docker service is running
docker info
Subtask 1.2: Pull the Jupyter Base Notebook Image
Now we'll download the official Jupyter Notebook image from Docker Hub.

# Pull the Jupyter base notebook image
docker pull jupyter/base-notebook

# Verify the image was downloaded successfully
docker images | grep jupyter
Expected Output: You should see the jupyter/base-notebook image listed with its size and creation date.

Subtask 1.3: Explore Image Details
Let's examine the details of the downloaded image to understand its configuration.

# Inspect the Jupyter image
docker inspect jupyter/base-notebook

# Check image layers and history
docker history jupyter/base-notebook
Task 2: Run Jupyter Notebook Container and Access via Browser
Subtask 2.1: Create a Working Directory
First, create a directory on the host system to store our notebooks and data.

# Create a directory for our ML projects
mkdir -p ~/ml-docker-lab
cd ~/ml-docker-lab

# Create subdirectories for organization
mkdir notebooks data models
Subtask 2.2: Run the Jupyter Container
Launch the Jupyter Notebook container with proper port mapping and volume mounting.

# Run Jupyter container with volume mounting
docker run -d \
  --name jupyter-ml-lab \
  -p 8888:8888 \
  -v ~/ml-docker-lab:/home/jovyan/work \
  -e JUPYTER_ENABLE_LAB=yes \
  jupyter/base-notebook
Command Explanation:

-d: Run container in detached mode (background)
--name jupyter-ml-lab: Assign a name to the container
-p 8888:8888: Map host port 8888 to container port 8888
-v ~/ml-docker-lab:/home/jovyan/work: Mount host directory to container
-e JUPYTER_ENABLE_LAB=yes: Enable JupyterLab interface
Subtask 2.3: Retrieve Access Token
Get the access token needed to connect to the Jupyter Notebook.

# Get the Jupyter access token
docker logs jupyter-ml-lab 2>&1 | grep -E "token=" | tail -1
Note: Copy the complete URL with the token - you'll need this to access Jupyter in your browser.

Subtask 2.4: Access Jupyter in Browser
Open your web browser
Navigate to http://localhost:8888 or use the complete URL with token from the previous step
You should see the JupyterLab interface with your mounted work directory
Task 3: Install Machine Learning Libraries Inside Container
Subtask 3.1: Access Container Terminal
We'll install additional libraries by accessing the container's terminal.

# Execute bash shell inside the running container
docker exec -it jupyter-ml-lab bash
Subtask 3.2: Install TensorFlow and Scikit-learn
Inside the container terminal, install the required machine learning libraries.

# Update pip to latest version
pip install --upgrade pip

# Install TensorFlow
pip install tensorflow==2.13.0

# Install Scikit-learn and additional ML libraries
pip install scikit-learn pandas numpy matplotlib seaborn

# Install additional useful libraries
pip install plotly jupyter-widgets

# Verify installations
pip list | grep -E "(tensorflow|scikit-learn|pandas|numpy)"
Subtask 3.3: Exit Container Terminal
# Exit the container terminal
exit
Subtask 3.4: Restart Jupyter to Load New Libraries
# Restart the container to ensure all libraries are properly loaded
docker restart jupyter-ml-lab

# Wait a few seconds, then get the new access token
sleep 5
docker logs jupyter-ml-lab 2>&1 | grep -E "token=" | tail -1
Task 4: Build and Train a Simple Model in Jupyter
Subtask 4.1: Create a New Notebook
Access Jupyter in your browser using the new token
Navigate to the work folder
Click New → Python 3 to create a new notebook
Rename the notebook to ml-docker-demo.ipynb
Subtask 4.2: Import Required Libraries
In the first cell of your notebook, add and execute the following code:

# Import necessary libraries
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.datasets import make_classification
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score, classification_report, confusion_matrix
import joblib
import os

# Set random seed for reproducibility
np.random.seed(42)

print("All libraries imported successfully!")
print(f"Current working directory: {os.getcwd()}")
Subtask 4.3: Generate Sample Dataset
Create a synthetic dataset for our machine learning model:

# Generate a synthetic classification dataset
X, y = make_classification(
    n_samples=1000,
    n_features=20,
    n_informative=15,
    n_redundant=5,
    n_classes=2,
    random_state=42
)

# Create a DataFrame for better handling
feature_names = [f'feature_{i}' for i in range(X.shape[1])]
df = pd.DataFrame(X, columns=feature_names)
df['target'] = y

print(f"Dataset shape: {df.shape}")
print(f"Target distribution:\n{df['target'].value_counts()}")
df.head()
Subtask 4.4: Explore the Data
Add data visualization and exploration:

# Basic statistics
print("Dataset Statistics:")
print(df.describe())

# Visualize target distribution
plt.figure(figsize=(10, 4))

plt.subplot(1, 2, 1)
df['target'].value_counts().plot(kind='bar')
plt.title('Target Distribution')
plt.xlabel('Class')
plt.ylabel('Count')

plt.subplot(1, 2, 2)
# Correlation heatmap of first 10 features
correlation_matrix = df.iloc[:, :10].corr()
sns.heatmap(correlation_matrix, annot=True, cmap='coolwarm', center=0)
plt.title('Feature Correlation Heatmap')

plt.tight_layout()
plt.show()
Subtask 4.5: Train the Machine Learning Model
Split the data and train a Random Forest classifier:

# Split the data into training and testing sets
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y
)

print(f"Training set size: {X_train.shape[0]}")
print(f"Testing set size: {X_test.shape[0]}")

# Create and train the Random Forest model
rf_model = RandomForestClassifier(
    n_estimators=100,
    random_state=42,
    max_depth=10
)

# Train the model
print("Training the model...")
rf_model.fit(X_train, y_train)
print("Model training completed!")

# Make predictions
y_pred = rf_model.predict(X_test)

# Calculate accuracy
accuracy = accuracy_score(y_test, y_pred)
print(f"Model Accuracy: {accuracy:.4f}")
Subtask 4.6: Evaluate Model Performance
Add comprehensive model evaluation:

# Detailed classification report
print("Classification Report:")
print(classification_report(y_test, y_pred))

# Confusion Matrix
plt.figure(figsize=(8, 6))
cm = confusion_matrix(y_test, y_pred)
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues')
plt.title('Confusion Matrix')
plt.xlabel('Predicted')
plt.ylabel('Actual')
plt.show()

# Feature importance
feature_importance = pd.DataFrame({
    'feature': feature_names,
    'importance': rf_model.feature_importances_
}).sort_values('importance', ascending=False)

plt.figure(figsize=(10, 6))
sns.barplot(data=feature_importance.head(10), x='importance', y='feature')
plt.title('Top 10 Feature Importances')
plt.xlabel('Importance')
plt.show()

print("Top 10 Most Important Features:")
print(feature_importance.head(10))
Task 5: Save Trained Model to Volume for Persistence
Subtask 5.1: Create Model Directory
Ensure the models directory exists and save our trained model:

# Create models directory if it doesn't exist
models_dir = '/home/jovyan/work/models'
os.makedirs(models_dir, exist_ok=True)

# Save the trained model
model_path = os.path.join(models_dir, 'random_forest_model.joblib')
joblib.dump(rf_model, model_path)

print(f"Model saved to: {model_path}")

# Save model metadata
metadata = {
    'model_type': 'RandomForestClassifier',
    'accuracy': accuracy,
    'n_features': X.shape[1],
    'n_samples_train': X_train.shape[0],
    'n_samples_test': X_test.shape[0],
    'feature_names': feature_names
}

import json
metadata_path = os.path.join(models_dir, 'model_metadata.json')
with open(metadata_path, 'w') as f:
    json.dump(metadata, f, indent=2)

print(f"Model metadata saved to: {metadata_path}")
Subtask 5.2: Test Model Loading
Verify that the saved model can be loaded and used:

# Load the saved model
loaded_model = joblib.load(model_path)

# Load metadata
with open(metadata_path, 'r') as f:
    loaded_metadata = json.load(f)

print("Loaded model metadata:")
for key, value in loaded_metadata.items():
    print(f"  {key}: {value}")

# Test the loaded model
test_predictions = loaded_model.predict(X_test[:5])
print(f"\nTest predictions on first 5 samples: {test_predictions}")
print(f"Actual values: {y_test[:5]}")

# Verify accuracy matches
loaded_accuracy = accuracy_score(y_test, loaded_model.predict(X_test))
print(f"\nLoaded model accuracy: {loaded_accuracy:.4f}")
print(f"Original model accuracy: {accuracy:.4f}")
print(f"Accuracy match: {abs(loaded_accuracy - accuracy) < 1e-10}")
Subtask 5.3: Save the Notebook
Save your notebook to preserve all the work:

Click File → Save and Checkpoint in Jupyter
The notebook is automatically saved to the mounted volume
Subtask 5.4: Verify Persistence on Host System
From your terminal (outside the container), verify that files are persisted:

# Check the contents of our project directory
ls -la ~/ml-docker-lab/

# Check notebooks directory
ls -la ~/ml-docker-lab/notebooks/

# Check models directory
ls -la ~/ml-docker-lab/models/

# View model metadata
cat ~/ml-docker-lab/models/model_metadata.json
Additional Tasks and Best Practices
Task 6: Create a Custom Docker Image
Let's create a custom Docker image with our ML libraries pre-installed.

Subtask 6.1: Create a Dockerfile
# Create a Dockerfile in the project directory
cd ~/ml-docker-lab
cat > Dockerfile << 'EOF'
# Use the official Jupyter base notebook as parent image
FROM jupyter/base-notebook:latest

# Set the working directory
WORKDIR /home/jovyan

# Switch to root user to install packages
USER root

# Install system dependencies if needed
RUN apt-get update && apt-get install -y \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Switch back to jovyan user
USER jovyan

# Install Python packages
RUN pip install --no-cache-dir \
    tensorflow==2.13.0 \
    scikit-learn \
    pandas \
    numpy \
    matplotlib \
    seaborn \
    plotly \
    jupyter-widgets \
    joblib

# Enable jupyter widgets
RUN jupyter nbextension enable --py widgetsnbextension

# Set the default command
CMD ["start-notebook.sh"]
EOF
Subtask 6.2: Build Custom Image
# Build the custom Docker image
docker build -t jupyter-ml-custom:latest .

# Verify the image was created
docker images | grep jupyter-ml-custom
Subtask 6.3: Run Container from Custom Image
# Stop the current container
docker stop jupyter-ml-lab
docker rm jupyter-ml-lab

# Run container from custom image
docker run -d \
  --name jupyter-ml-custom \
  -p 8888:8888 \
  -v ~/ml-docker-lab:/home/jovyan/work \
  -e JUPYTER_ENABLE_LAB=yes \
  jupyter-ml-custom:latest

# Get the access token
docker logs jupyter-ml-custom 2>&1 | grep -E "token=" | tail -1
Troubleshooting Common Issues
Issue 1: Port Already in Use
If port 8888 is already in use:

# Check what's using port 8888
sudo netstat -tulpn | grep 8888

# Use a different port
docker run -d \
  --name jupyter-ml-lab \
  -p 8889:8888 \
  -v ~/ml-docker-lab:/home/jovyan/work \
  jupyter/base-notebook
Issue 2: Permission Issues with Volumes
If you encounter permission issues:

# Fix ownership of the mounted directory
sudo chown -R 1000:1000 ~/ml-docker-lab

# Or run container with specific user ID
docker run -d \
  --name jupyter-ml-lab \
  -p 8888:8888 \
  -v ~/ml-docker-lab:/home/jovyan/work \
  --user $(id -u):$(id -g) \
  jupyter/base-notebook
Issue 3: Container Won't Start
Check container logs for errors:

# View container logs
docker logs jupyter-ml-lab

# Check container status
docker ps -a
Lab Cleanup
When you're finished with the lab, clean up the resources:

# Stop and remove containers
docker stop jupyter-ml-lab jupyter-ml-custom
docker rm jupyter-ml-lab jupyter-ml-custom

# Remove custom image (optional)
docker rmi jupyter-ml-custom:latest

# Keep the project files for future reference
# The ~/ml-docker-lab directory contains all your work
Conclusion
Congratulations! You have successfully completed Lab 33: Docker for Machine Learning. In this comprehensive lab, you have accomplished the following:

Key Achievements:

Containerized ML Environment: You learned how to use Docker to create reproducible machine learning environments, eliminating the "it works on my machine" problem.

Jupyter Integration: You successfully pulled and ran Jupyter Notebook containers, providing a web-based interface for interactive machine learning development.

Library Management: You installed and configured essential machine learning libraries (TensorFlow, Scikit-learn) within containerized environments.

Model Development: You built, trained, and evaluated a complete machine learning model using a Random Forest classifier on synthetic data.

Data Persistence: You implemented proper data persistence using Docker volumes, ensuring your models and notebooks survive container restarts.

Best Practices: You learned Docker best practices for ML workflows, including custom image creation and proper volume mounting.

Why This Matters:

Reproducibility: Docker ensures your ML experiments can be reproduced across different environments and team members
Scalability: Containerized ML workflows can be easily scaled and deployed to cloud platforms
Collaboration: Team members can share identical development environments regardless of their local setup
Version Control: Different versions of ML libraries and models can be managed through Docker images
Production Readiness: Skills learned here directly apply to deploying ML models in production environments
Next Steps:

Explore more complex ML workflows with multiple containers
Learn about Docker Compose for multi-service ML applications
Investigate Kubernetes for orchestrating ML workloads at scale
Practice with different ML frameworks and libraries in containerized environments
This lab has provided you with foundational skills for modern MLOps practices and prepared you for the Docker Certified Associate (DCA) certification. The combination of Docker and machine learning represents a powerful approach to building robust, scalable, and maintainable ML systems.
