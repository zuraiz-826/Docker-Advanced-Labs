Lab 42: Docker with Cloud Functions - Integrating Docker with Google Cloud Functions
Objectives
By the end of this lab, you will be able to:

Build a Docker image containing a Python function for cloud deployment
Push Docker images to Google Container Registry (GCR)
Deploy containerized functions to Google Cloud Functions
Trigger cloud functions using HTTP requests
Monitor function execution using Google Cloud Logging
Understand the integration between Docker containers and serverless computing
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts and commands
Familiarity with Python programming
Basic knowledge of HTTP requests and REST APIs
Understanding of cloud computing fundamentals
A Google Cloud Platform account with billing enabled
Note: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-installed with:

Docker Engine
Google Cloud SDK (gcloud CLI)
Python 3.8+
Text editors (nano, vim)
curl for HTTP testing
Task 1: Build a Docker Image with a Python Function
Subtask 1.1: Create the Project Directory Structure
First, let's create a organized directory structure for our cloud function project.

# Create the main project directory
mkdir ~/cloud-function-docker
cd ~/cloud-function-docker

# Create subdirectories for organization
mkdir src
mkdir config
Subtask 1.2: Create the Python Function
Create a simple Python function that will serve as our cloud function.

# Create the main Python file
nano src/main.py
Add the following Python code:

import json
import os
from datetime import datetime

def hello_docker(request):
    """
    HTTP Cloud Function that demonstrates Docker integration.
    Args:
        request (flask.Request): The request object.
    Returns:
        The response text, or any set of values that can be turned into a
        Response object using `make_response`.
    """
    
    # Handle both GET and POST requests
    if request.method == 'POST':
        request_json = request.get_json(silent=True)
        if request_json and 'name' in request_json:
            name = request_json['name']
        else:
            name = 'World'
    else:
        name = request.args.get('name', 'World')
    
    # Create response data
    response_data = {
        'message': f'Hello {name} from Docker Cloud Function!',
        'timestamp': datetime.now().isoformat(),
        'container_info': {
            'hostname': os.environ.get('HOSTNAME', 'unknown'),
            'function_name': os.environ.get('FUNCTION_NAME', 'hello_docker'),
            'function_version': os.environ.get('FUNCTION_VERSION', '1.0')
        }
    }
    
    return json.dumps(response_data, indent=2)

# For local testing
if __name__ == '__main__':
    from flask import Flask, request
    app = Flask(__name__)
    
    @app.route('/', methods=['GET', 'POST'])
    def main():
        return hello_docker(request)
    
    app.run(host='0.0.0.0', port=8080, debug=True)
Subtask 1.3: Create Requirements File
Create a requirements.txt file to specify Python dependencies.

nano requirements.txt
Add the following content:

functions-framework==3.4.0
flask==2.3.3
Subtask 1.4: Create the Dockerfile
Now create a Dockerfile that will containerize our Python function.

nano Dockerfile
Add the following Dockerfile content:

# Use the official Python runtime as the base image
FROM python:3.9-slim

# Set environment variables
ENV PYTHONUNBUFFERED=1
ENV FUNCTION_SOURCE=src/main.py
ENV FUNCTION_TARGET=hello_docker

# Set the working directory in the container
WORKDIR /app

# Copy requirements first for better caching
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy the source code
COPY src/ ./src/

# Expose the port that the function will run on
EXPOSE 8080

# Use the Functions Framework to run the function
CMD exec functions-framework --target=${FUNCTION_TARGET} --source=${FUNCTION_SOURCE} --port=8080 --host=0.0.0.0
Subtask 1.5: Build the Docker Image
Build the Docker image locally to test it before pushing to the registry.

# Build the Docker image
docker build -t cloud-function-demo:latest .

# Verify the image was created
docker images | grep cloud-function-demo
Subtask 1.6: Test the Docker Image Locally
Test the containerized function locally before deploying to the cloud.

# Run the container locally
docker run -d -p 8080:8080 --name test-function cloud-function-demo:latest

# Wait a moment for the container to start
sleep 5

# Test with a GET request
curl "http://localhost:8080?name=Docker"

# Test with a POST request
curl -X POST -H "Content-Type: application/json" \
     -d '{"name":"Container"}' \
     http://localhost:8080

# Stop and remove the test container
docker stop test-function
docker rm test-function
Task 2: Push the Image to Google Container Registry
Subtask 2.1: Authenticate with Google Cloud
First, authenticate your environment with Google Cloud Platform.

# Initialize gcloud configuration
gcloud init

# Authenticate Docker with Google Container Registry
gcloud auth configure-docker
Subtask 2.2: Set Project Variables
Set up environment variables for your Google Cloud project.

# Replace 'your-project-id' with your actual GCP project ID
export PROJECT_ID="your-project-id"
export FUNCTION_NAME="hello-docker-function"
export REGION="us-central1"

# Verify your project ID
echo "Project ID: $PROJECT_ID"
gcloud config set project $PROJECT_ID
Subtask 2.3: Tag the Image for Google Container Registry
Tag your Docker image with the appropriate GCR format.

# Tag the image for Google Container Registry
docker tag cloud-function-demo:latest gcr.io/$PROJECT_ID/cloud-function-demo:latest

# Verify the tag was applied
docker images | grep gcr.io
Subtask 2.4: Push the Image to GCR
Push the tagged image to Google Container Registry.

# Push the image to Google Container Registry
docker push gcr.io/$PROJECT_ID/cloud-function-demo:latest

# Verify the push was successful
gcloud container images list --repository=gcr.io/$PROJECT_ID
Task 3: Deploy the Function to Google Cloud Functions
Subtask 3.1: Enable Required APIs
Enable the necessary Google Cloud APIs for Cloud Functions.

# Enable Cloud Functions API
gcloud services enable cloudfunctions.googleapis.com

# Enable Cloud Build API (required for container deployments)
gcloud services enable cloudbuild.googleapis.com

# Enable Container Registry API
gcloud services enable containerregistry.googleapis.com

# Verify APIs are enabled
gcloud services list --enabled | grep -E "(cloudfunctions|cloudbuild|containerregistry)"
Subtask 3.2: Deploy the Containerized Function
Deploy the function using the Docker image from Google Container Registry.

# Deploy the function using the container image
gcloud functions deploy $FUNCTION_NAME \
    --source=gcr.io/$PROJECT_ID/cloud-function-demo:latest \
    --trigger-http \
    --runtime=python39 \
    --region=$REGION \
    --allow-unauthenticated \
    --memory=256MB \
    --timeout=60s

# Wait for deployment to complete (this may take a few minutes)
echo "Deployment initiated. This may take 2-3 minutes..."
Subtask 3.3: Verify the Deployment
Check that the function was deployed successfully.

# List deployed functions
gcloud functions list --region=$REGION

# Get detailed information about the function
gcloud functions describe $FUNCTION_NAME --region=$REGION

# Get the function's trigger URL
export FUNCTION_URL=$(gcloud functions describe $FUNCTION_NAME --region=$REGION --format="value(httpsTrigger.url)")
echo "Function URL: $FUNCTION_URL"
Task 4: Trigger the Function Using HTTP Requests
Subtask 4.1: Test with GET Requests
Test the deployed function using various GET request patterns.

# Basic GET request
curl "$FUNCTION_URL"

# GET request with query parameter
curl "$FUNCTION_URL?name=CloudFunction"

# GET request with multiple parameters
curl "$FUNCTION_URL?name=GoogleCloud&test=true"
Subtask 4.2: Test with POST Requests
Test the function with POST requests containing JSON data.

# POST request with JSON data
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"name":"Docker Container"}' \
     "$FUNCTION_URL"

# POST request with more complex JSON
curl -X POST \
     -H "Content-Type: application/json" \
     -d '{"name":"Advanced Test","environment":"production","version":"1.0"}' \
     "$FUNCTION_URL"
Subtask 4.3: Performance Testing
Perform basic load testing to see how the function scales.

# Create a simple load test script
cat > load_test.sh << 'EOF'
#!/bin/bash
FUNCTION_URL=$1
REQUESTS=${2:-10}

echo "Sending $REQUESTS requests to $FUNCTION_URL"
for i in $(seq 1 $REQUESTS); do
    echo "Request $i:"
    curl -s "$FUNCTION_URL?name=LoadTest$i" | head -1
    sleep 0.5
done
EOF

# Make the script executable
chmod +x load_test.sh

# Run the load test
./load_test.sh "$FUNCTION_URL" 5
Task 5: Monitor the Function with Google Cloud Logging
Subtask 5.1: View Function Logs
Access and examine the function execution logs.

# View recent logs for the function
gcloud functions logs read $FUNCTION_NAME --region=$REGION --limit=20

# View logs in real-time (run this in a separate terminal if needed)
gcloud functions logs tail $FUNCTION_NAME --region=$REGION
Subtask 5.2: Generate Log Entries
Create some activity to generate log entries for monitoring.

# Generate multiple requests to create log entries
for i in {1..5}; do
    echo "Generating request $i"
    curl -s "$FUNCTION_URL?name=LogTest$i" > /dev/null
    sleep 2
done

# View the new log entries
gcloud functions logs read $FUNCTION_NAME --region=$REGION --limit=10
Subtask 5.3: Monitor Function Metrics
Check function performance metrics and statistics.

# Get function statistics
gcloud functions describe $FUNCTION_NAME --region=$REGION --format="table(
    name,
    status,
    availableMemoryMb,
    timeout,
    updateTime
)"

# View function source information
gcloud functions describe $FUNCTION_NAME --region=$REGION --format="value(sourceArchiveUrl)"
Subtask 5.4: Set Up Log Filtering
Learn to filter logs for specific information.

# Filter logs by severity
gcloud functions logs read $FUNCTION_NAME --region=$REGION --filter="severity>=ERROR"

# Filter logs by timestamp (last hour)
gcloud functions logs read $FUNCTION_NAME --region=$REGION --filter="timestamp>=\"$(date -u -d '1 hour ago' '+%Y-%m-%dT%H:%M:%SZ')\""

# Search for specific text in logs
gcloud functions logs read $FUNCTION_NAME --region=$REGION --filter="textPayload:\"Docker\""
Advanced Configuration and Troubleshooting
Environment Variables and Configuration
You can configure your function with environment variables:

# Update function with environment variables
gcloud functions deploy $FUNCTION_NAME \
    --source=gcr.io/$PROJECT_ID/cloud-function-demo:latest \
    --trigger-http \
    --runtime=python39 \
    --region=$REGION \
    --allow-unauthenticated \
    --set-env-vars="ENVIRONMENT=production,LOG_LEVEL=info"
Common Troubleshooting Commands
# Check function status
gcloud functions describe $FUNCTION_NAME --region=$REGION --format="value(status)"

# View build logs if deployment fails
gcloud builds list --limit=5

# Check IAM permissions
gcloud projects get-iam-policy $PROJECT_ID

# Test function locally with Docker
docker run -p 8080:8080 -e FUNCTION_TARGET=hello_docker gcr.io/$PROJECT_ID/cloud-function-demo:latest
Cleanup Resources
To avoid unnecessary charges, clean up the resources created in this lab:

# Delete the Cloud Function
gcloud functions delete $FUNCTION_NAME --region=$REGION --quiet

# Delete the container image from GCR
gcloud container images delete gcr.io/$PROJECT_ID/cloud-function-demo:latest --quiet

# Remove local Docker images
docker rmi cloud-function-demo:latest
docker rmi gcr.io/$PROJECT_ID/cloud-function-demo:latest

# Remove project directory
cd ~
rm -rf cloud-function-docker
Conclusion
Congratulations! You have successfully completed Lab 42: Docker with Cloud Functions. In this comprehensive lab, you have accomplished the following:

Key Achievements:

Containerized a Python Function: You built a Docker image containing a Python function designed for cloud deployment, demonstrating how traditional applications can be packaged for serverless environments.

Mastered Container Registry Operations: You successfully pushed your Docker image to Google Container Registry, learning the essential workflow for managing container images in the cloud.

Deployed Serverless Containers: You deployed your containerized function to Google Cloud Functions, bridging the gap between container technology and serverless computing.

Implemented HTTP Triggers: You tested your function with various HTTP requests (GET and POST), understanding how cloud functions respond to different input patterns.

Monitored Cloud Applications: You used Google Cloud Logging to monitor function execution, gaining insights into application performance and debugging capabilities.

Why This Matters:

This lab demonstrates the powerful combination of Docker containers and serverless computing. By containerizing your functions, you gain:

Consistency: Your function runs the same way locally and in the cloud
Flexibility: You can use any programming language or runtime supported by containers
Scalability: Cloud Functions automatically scale your containerized applications
Cost Efficiency: You only pay for actual function execution time
DevOps Integration: Container-based functions fit seamlessly into CI/CD pipelines
Real-World Applications:

The skills you've learned apply to many scenarios:

Microservices Architecture: Deploy individual services as containerized functions
API Development: Create scalable REST APIs without managing servers
Data Processing: Build event-driven data processing pipelines
Integration Services: Connect different systems through lightweight, containerized functions
This knowledge is particularly valuable for the Docker Certified Associate (DCA) certification, as it demonstrates advanced Docker usage in cloud-native environments. You now understand how Docker extends beyond traditional server deployments into the serverless computing paradigm, making you a more versatile cloud developer.

The integration of Docker with Google Cloud Functions represents the future of application deployment - combining the benefits of containerization with the simplicity and cost-effectiveness of serverless computing.
