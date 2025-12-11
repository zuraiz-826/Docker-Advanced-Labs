Lab 98: Docker and Cloud Functions - Docker on AWS Lambda
Objectives
By the end of this lab, you will be able to:

• Understand the integration between Docker containers and AWS Lambda serverless functions • Build a Docker image specifically designed for AWS Lambda runtime • Push Docker images to Amazon Elastic Container Registry (ECR) • Create AWS Lambda functions using container images • Configure API Gateway to trigger containerized Lambda functions • Monitor Lambda function execution using AWS CloudWatch • Implement serverless containerized applications in production environments

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker concepts (containers, images, Dockerfile) • Familiarity with AWS services and AWS CLI • Knowledge of Python programming language • Understanding of REST APIs and HTTP methods • Experience with command-line interface operations

Required Tools and Accounts
• AWS Account with appropriate permissions for Lambda, ECR, API Gateway, and CloudWatch • AWS CLI installed and configured • Docker installed and running • Text editor or IDE for code editing

Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. Your machine comes with:

• Docker pre-installed and configured • AWS CLI pre-installed • Python 3.9 runtime environment • All necessary development tools • Internet connectivity for AWS services

No need to build your own VM or install additional software!

Task 1: Build a Docker Image for Lambda Function
Subtask 1.1: Create the Project Directory Structure
First, let's create a organized directory structure for our Lambda function project.

# Create project directory
mkdir lambda-docker-demo
cd lambda-docker-demo

# Create subdirectories
mkdir src
mkdir scripts
Subtask 1.2: Create the Lambda Function Code
Create a simple Python Lambda function that will be containerized.

# Create the main Lambda function file
cat > src/app.py << 'EOF'
import json
import os
from datetime import datetime

def lambda_handler(event, context):
    """
    AWS Lambda function handler for processing HTTP requests
    """
    
    # Extract information from the event
    http_method = event.get('httpMethod', 'UNKNOWN')
    path = event.get('path', '/')
    query_params = event.get('queryStringParameters') or {}
    
    # Get environment variables
    function_name = os.environ.get('AWS_LAMBDA_FUNCTION_NAME', 'unknown')
    function_version = os.environ.get('AWS_LAMBDA_FUNCTION_VERSION', 'unknown')
    
    # Create response data
    response_data = {
        'message': 'Hello from Dockerized Lambda!',
        'timestamp': datetime.utcnow().isoformat(),
        'function_info': {
            'name': function_name,
            'version': function_version
        },
        'request_info': {
            'method': http_method,
            'path': path,
            'query_parameters': query_params
        },
        'container_info': {
            'runtime': 'Docker Container',
            'python_version': '3.9'
        }
    }
    
    # Return successful response
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE',
            'Access-Control-Allow-Headers': 'Content-Type'
        },
        'body': json.dumps(response_data, indent=2)
    }
EOF
Subtask 1.3: Create Requirements File
Create a requirements file for Python dependencies.

# Create requirements.txt file
cat > requirements.txt << 'EOF'
# AWS Lambda Runtime Interface Client
# This is required for custom container images
boto3==1.26.137
botocore==1.29.137
EOF
Subtask 1.4: Create the Dockerfile
Create a Dockerfile optimized for AWS Lambda container runtime.

# Create Dockerfile for Lambda
cat > Dockerfile << 'EOF'
# Use the official AWS Lambda Python 3.9 base image
FROM public.ecr.aws/lambda/python:3.9

# Set working directory
WORKDIR ${LAMBDA_TASK_ROOT}

# Copy requirements file
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy function code
COPY src/app.py .

# Set the CMD to your handler
CMD ["app.lambda_handler"]
EOF
Subtask 1.5: Build the Docker Image
Now let's build the Docker image for our Lambda function.

# Build the Docker image
docker build -t lambda-docker-demo:latest .

# Verify the image was created
docker images | grep lambda-docker-demo
Subtask 1.6: Test the Image Locally (Optional)
Test the Docker image locally using the Lambda Runtime Interface Emulator.

# Run the container locally for testing
docker run -p 9000:8080 lambda-docker-demo:latest &

# Wait a moment for the container to start
sleep 5

# Test the function locally
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" \
  -d '{
    "httpMethod": "GET",
    "path": "/test",
    "queryStringParameters": {
      "name": "Docker",
      "version": "1.0"
    }
  }'

# Stop the local container
docker stop $(docker ps -q --filter ancestor=lambda-docker-demo:latest)
Task 2: Push the Image to Amazon ECR
Subtask 2.1: Create ECR Repository
Create an Amazon ECR repository to store our Docker image.

# Set variables
REGION="us-east-1"
REPO_NAME="lambda-docker-demo"

# Create ECR repository
aws ecr create-repository \
    --repository-name $REPO_NAME \
    --region $REGION \
    --image-scanning-configuration scanOnPush=true

# Get the repository URI
REPO_URI=$(aws ecr describe-repositories \
    --repository-names $REPO_NAME \
    --region $REGION \
    --query 'repositories[0].repositoryUri' \
    --output text)

echo "Repository URI: $REPO_URI"
Subtask 2.2: Authenticate Docker with ECR
Authenticate your Docker client with Amazon ECR.

# Get ECR login token and authenticate Docker
aws ecr get-login-password --region $REGION | \
    docker login --username AWS --password-stdin $REPO_URI
Subtask 2.3: Tag and Push the Image
Tag the Docker image and push it to ECR.

# Tag the image for ECR
docker tag lambda-docker-demo:latest $REPO_URI:latest

# Push the image to ECR
docker push $REPO_URI:latest

# Verify the image was pushed
aws ecr list-images --repository-name $REPO_NAME --region $REGION
Task 3: Create AWS Lambda Function Using Docker Container Image
Subtask 3.1: Create IAM Role for Lambda
Create an IAM role that Lambda can assume to execute the function.

# Create trust policy document
cat > lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create IAM role
aws iam create-role \
    --role-name lambda-docker-execution-role \
    --assume-role-policy-document file://lambda-trust-policy.json

# Attach basic execution policy
aws iam attach-role-policy \
    --role-name lambda-docker-execution-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Get the role ARN
ROLE_ARN=$(aws iam get-role \
    --role-name lambda-docker-execution-role \
    --query 'Role.Arn' \
    --output text)

echo "Role ARN: $ROLE_ARN"
Subtask 3.2: Create Lambda Function
Create the Lambda function using the container image.

# Set function name
FUNCTION_NAME="docker-lambda-demo"

# Wait for role to be available (IAM eventual consistency)
sleep 10

# Create Lambda function with container image
aws lambda create-function \
    --function-name $FUNCTION_NAME \
    --role $ROLE_ARN \
    --code ImageUri=$REPO_URI:latest \
    --package-type Image \
    --timeout 30 \
    --memory-size 256 \
    --region $REGION

# Wait for function to be active
echo "Waiting for function to become active..."
aws lambda wait function-active --function-name $FUNCTION_NAME --region $REGION
Subtask 3.3: Test the Lambda Function
Test the Lambda function to ensure it's working correctly.

# Create test event
cat > test-event.json << 'EOF'
{
  "httpMethod": "GET",
  "path": "/hello",
  "queryStringParameters": {
    "name": "DockerLambda",
    "environment": "test"
  }
}
EOF

# Invoke the Lambda function
aws lambda invoke \
    --function-name $FUNCTION_NAME \
    --payload file://test-event.json \
    --region $REGION \
    response.json

# Display the response
echo "Lambda Response:"
cat response.json | python3 -m json.tool
Task 4: Set Up API Gateway to Trigger Lambda Function
Subtask 4.1: Create REST API
Create a REST API in API Gateway.

# Create REST API
API_ID=$(aws apigateway create-rest-api \
    --name "docker-lambda-api" \
    --description "API Gateway for Docker Lambda Demo" \
    --region $REGION \
    --query 'id' \
    --output text)

echo "API ID: $API_ID"

# Get the root resource ID
ROOT_RESOURCE_ID=$(aws apigateway get-resources \
    --rest-api-id $API_ID \
    --region $REGION \
    --query 'items[0].id' \
    --output text)

echo "Root Resource ID: $ROOT_RESOURCE_ID"
Subtask 4.2: Create API Resource and Method
Create a resource and method for the API.

# Create a resource
RESOURCE_ID=$(aws apigateway create-resource \
    --rest-api-id $API_ID \
    --parent-id $ROOT_RESOURCE_ID \
    --path-part "docker-demo" \
    --region $REGION \
    --query 'id' \
    --output text)

echo "Resource ID: $RESOURCE_ID"

# Create GET method
aws apigateway put-method \
    --rest-api-id $API_ID \
    --resource-id $RESOURCE_ID \
    --http-method GET \
    --authorization-type NONE \
    --region $REGION

# Create POST method
aws apigateway put-method \
    --rest-api-id $API_ID \
    --resource-id $RESOURCE_ID \
    --http-method POST \
    --authorization-type NONE \
    --region $REGION
Subtask 4.3: Configure Lambda Integration
Configure the API Gateway to integrate with the Lambda function.

# Get AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)

# Set Lambda function ARN
LAMBDA_ARN="arn:aws:lambda:$REGION:$ACCOUNT_ID:function:$FUNCTION_NAME"

# Configure GET method integration
aws apigateway put-integration \
    --rest-api-id $API_ID \
    --resource-id $RESOURCE_ID \
    --http-method GET \
    --type AWS_PROXY \
    --integration-http-method POST \
    --uri "arn:aws:apigateway:$REGION:lambda:path/2015-03-31/functions/$LAMBDA_ARN/invocations" \
    --region $REGION

# Configure POST method integration
aws apigateway put-integration \
    --rest-api-id $API_ID \
    --resource-id $RESOURCE_ID \
    --http-method POST \
    --type AWS_PROXY \
    --integration-http-method POST \
    --uri "arn:aws:apigateway:$REGION:lambda:path/2015-03-31/functions/$LAMBDA_ARN/invocations" \
    --region $REGION
Subtask 4.4: Grant API Gateway Permission to Invoke Lambda
Grant API Gateway permission to invoke the Lambda function.

# Add permission for GET method
aws lambda add-permission \
    --function-name $FUNCTION_NAME \
    --statement-id "api-gateway-get-permission" \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com \
    --source-arn "arn:aws:execute-api:$REGION:$ACCOUNT_ID:$API_ID/*/GET/docker-demo" \
    --region $REGION

# Add permission for POST method
aws lambda add-permission \
    --function-name $FUNCTION_NAME \
    --statement-id "api-gateway-post-permission" \
    --action lambda:InvokeFunction \
    --principal apigateway.amazonaws.com \
    --source-arn "arn:aws:execute-api:$REGION:$ACCOUNT_ID:$API_ID/*/POST/docker-demo" \
    --region $REGION
Subtask 4.5: Deploy the API
Deploy the API to make it accessible.

# Create deployment
aws apigateway create-deployment \
    --rest-api-id $API_ID \
    --stage-name "prod" \
    --stage-description "Production stage for Docker Lambda Demo" \
    --description "Initial deployment" \
    --region $REGION

# Get the API endpoint URL
API_URL="https://$API_ID.execute-api.$REGION.amazonaws.com/prod/docker-demo"
echo "API Endpoint: $API_URL"
Subtask 4.6: Test the API Gateway
Test the deployed API Gateway endpoint.

# Test GET request
echo "Testing GET request:"
curl -X GET "$API_URL?name=APIGateway&test=true"

echo -e "\n\nTesting POST request:"
# Test POST request
curl -X POST "$API_URL" \
    -H "Content-Type: application/json" \
    -d '{"message": "Hello from API Gateway POST request"}'
Task 5: Monitor Lambda Execution in AWS CloudWatch
Subtask 5.1: View Lambda Function Logs
Access and view the Lambda function logs in CloudWatch.

# Get log group name
LOG_GROUP="/aws/lambda/$FUNCTION_NAME"

# List recent log streams
echo "Recent log streams:"
aws logs describe-log-streams \
    --log-group-name $LOG_GROUP \
    --order-by LastEventTime \
    --descending \
    --max-items 5 \
    --region $REGION \
    --query 'logStreams[*].logStreamName' \
    --output table

# Get the most recent log stream
LATEST_STREAM=$(aws logs describe-log-streams \
    --log-group-name $LOG_GROUP \
    --order-by LastEventTime \
    --descending \
    --max-items 1 \
    --region $REGION \
    --query 'logStreams[0].logStreamName' \
    --output text)

echo "Latest log stream: $LATEST_STREAM"
Subtask 5.2: Retrieve and Display Log Events
Retrieve and display the latest log events from CloudWatch.

# Get recent log events
echo "Recent log events:"
aws logs get-log-events \
    --log-group-name $LOG_GROUP \
    --log-stream-name "$LATEST_STREAM" \
    --region $REGION \
    --query 'events[*].[timestamp,message]' \
    --output table
Subtask 5.3: Create CloudWatch Dashboard
Create a CloudWatch dashboard to monitor Lambda metrics.

# Create dashboard configuration
cat > dashboard-config.json << EOF
{
  "widgets": [
    {
      "type": "metric",
      "x": 0,
      "y": 0,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/Lambda", "Invocations", "FunctionName", "$FUNCTION_NAME" ],
          [ ".", "Duration", ".", "." ],
          [ ".", "Errors", ".", "." ],
          [ ".", "Throttles", ".", "." ]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "$REGION",
        "title": "Lambda Function Metrics"
      }
    },
    {
      "type": "log",
      "x": 0,
      "y": 6,
      "width": 24,
      "height": 6,
      "properties": {
        "query": "SOURCE '$LOG_GROUP' | fields @timestamp, @message\n| sort @timestamp desc\n| limit 20",
        "region": "$REGION",
        "title": "Recent Lambda Logs"
      }
    }
  ]
}
EOF

# Create CloudWatch dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "DockerLambdaMonitoring" \
    --dashboard-body file://dashboard-config.json \
    --region $REGION

echo "CloudWatch Dashboard created: DockerLambdaMonitoring"
Subtask 5.4: Generate Test Traffic for Monitoring
Generate some test traffic to populate the monitoring dashboard.

# Create script to generate test traffic
cat > scripts/generate-traffic.sh << 'EOF'
#!/bin/bash

API_URL=$1
REQUESTS=${2:-10}

echo "Generating $REQUESTS test requests to $API_URL"

for i in $(seq 1 $REQUESTS); do
    echo "Request $i:"
    
    # Alternate between GET and POST requests
    if [ $((i % 2)) -eq 0 ]; then
        curl -s -X GET "$API_URL?request=$i&type=monitoring" | jq -r '.message'
    else
        curl -s -X POST "$API_URL" \
            -H "Content-Type: application/json" \
            -d "{\"request\": $i, \"type\": \"monitoring\"}" | jq -r '.message'
    fi
    
    # Small delay between requests
    sleep 2
done

echo "Traffic generation completed!"
EOF

# Make script executable
chmod +x scripts/generate-traffic.sh

# Generate test traffic
./scripts/generate-traffic.sh "$API_URL" 5
Subtask 5.5: View Metrics in CloudWatch
Check the CloudWatch metrics for our Lambda function.

# Get Lambda invocation metrics
echo "Lambda Invocation Metrics (last 1 hour):"
aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name Invocations \
    --dimensions Name=FunctionName,Value=$FUNCTION_NAME \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Sum \
    --region $REGION \
    --query 'Datapoints[*].[Timestamp,Sum]' \
    --output table

# Get Lambda duration metrics
echo -e "\nLambda Duration Metrics (last 1 hour):"
aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name Duration \
    --dimensions Name=FunctionName,Value=$FUNCTION_NAME \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average,Maximum \
    --region $REGION \
    --query 'Datapoints[*].[Timestamp,Average,Maximum]' \
    --output table
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Docker build fails

# Check Docker daemon status
sudo systemctl status docker

# Restart Docker if needed
sudo systemctl restart docker
Issue 2: ECR authentication fails

# Verify AWS credentials
aws sts get-caller-identity

# Re-authenticate with ECR
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $REPO_URI
Issue 3: Lambda function timeout

# Increase timeout (max 15 minutes for Lambda)
aws lambda update-function-configuration \
    --function-name $FUNCTION_NAME \
    --timeout 60 \
    --region $REGION
Issue 4: API Gateway 502 errors

# Check Lambda function logs
aws logs tail $LOG_GROUP --follow

# Verify Lambda permissions
aws lambda get-policy --function-name $FUNCTION_NAME --region $REGION
Cleanup Resources
To avoid ongoing charges, clean up the resources created in this lab:

# Delete API Gateway
aws apigateway delete-rest-api --rest-api-id $API_ID --region $REGION

# Delete Lambda function
aws lambda delete-function --function-name $FUNCTION_NAME --region $REGION

# Delete ECR repository
aws ecr delete-repository --repository-name $REPO_NAME --force --region $REGION

# Delete IAM role
aws iam detach-role-policy --role-name lambda-docker-execution-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name lambda-docker-execution-role

# Delete CloudWatch dashboard
aws cloudwatch delete-dashboards --dashboard-names DockerLambdaMonitoring --region $REGION

# Clean up local files
cd ..
rm -rf lambda-docker-demo
Conclusion
Congratulations! You have successfully completed Lab 98: Docker and Cloud Functions - Docker on AWS Lambda. In this comprehensive lab, you have accomplished the following:

Key Achievements:

• Built a containerized Lambda function using Docker and AWS Lambda base images • Deployed container images to Amazon ECR for centralized storage and management • Created serverless functions using Docker containers instead of traditional deployment packages • Integrated API Gateway with containerized Lambda functions for HTTP endpoint access • Implemented comprehensive monitoring using AWS CloudWatch for logs and metrics • Gained hands-on experience with modern serverless container deployment patterns

Why This Matters:

This lab demonstrates the powerful combination of containerization and serverless computing. By using Docker containers with AWS Lambda, you can:

• Package complex applications with custom runtimes and dependencies • Maintain consistency between development and production environments • Leverage existing container workflows in serverless architectures • Scale automatically while maintaining the benefits of containerization • Reduce cold start times compared to traditional Lambda deployment packages

Real-World Applications:

The skills you've learned are directly applicable to:

• Microservices architectures requiring custom runtime environments • Legacy application modernization through containerized serverless functions • Multi-language applications that benefit from container packaging • Enterprise applications requiring specific system dependencies • DevOps pipelines integrating container registries with serverless deployments

You now have the foundation to build production-ready serverless applications using Docker containers, combining the best of both containerization and serverless technologies. This approach is increasingly popular in modern cloud-native architectures and is a valuable skill for cloud engineers and developers.
