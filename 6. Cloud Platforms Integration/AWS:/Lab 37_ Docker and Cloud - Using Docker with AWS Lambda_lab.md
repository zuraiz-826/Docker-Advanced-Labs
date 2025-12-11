Lab 37: Docker and Cloud - Using Docker with AWS Lambda
Lab Objectives
By the end of this lab, you will be able to:

Create a Docker container for a Python-based Lambda function
Package and push Docker containers to Amazon Elastic Container Registry (ECR)
Deploy containerized Lambda functions using AWS CLI
Configure API Gateway to trigger Lambda functions
Monitor and analyze function execution using CloudWatch logs
Understand the benefits of using Docker containers with AWS Lambda
Implement best practices for containerized serverless applications
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts and commands
Familiarity with Python programming language
Basic knowledge of AWS services (Lambda, ECR, API Gateway, CloudWatch)
Understanding of REST APIs and HTTP methods
Experience with command-line interfaces
AWS account with appropriate permissions for Lambda, ECR, API Gateway, and CloudWatch
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to access your environment. No need to build your own VM or install additional software.

Your cloud machine includes:

Docker Engine
AWS CLI v2
Python 3.9+
Text editors (nano, vim)
All required dependencies
Task 1: Create a Docker Container for a Python-based Lambda Function
Subtask 1.1: Create the Project Directory Structure
First, let's create a organized directory structure for our Lambda function project.

# Create the main project directory
mkdir lambda-docker-project
cd lambda-docker-project

# Create subdirectories for organization
mkdir src
mkdir tests
Subtask 1.2: Create the Lambda Function Code
Create a simple Python Lambda function that will process HTTP requests.

# Create the main Lambda function file
nano src/lambda_function.py
Add the following Python code:

import json
import datetime
import os

def lambda_handler(event, context):
    """
    AWS Lambda function handler that processes incoming events
    """
    
    # Get current timestamp
    current_time = datetime.datetime.now().isoformat()
    
    # Extract information from the event
    http_method = event.get('httpMethod', 'Unknown')
    path = event.get('path', '/')
    
    # Get query parameters if they exist
    query_params = event.get('queryStringParameters') or {}
    
    # Create response data
    response_data = {
        'message': 'Hello from Dockerized Lambda!',
        'timestamp': current_time,
        'method': http_method,
        'path': path,
        'query_parameters': query_params,
        'container_info': {
            'python_version': os.sys.version,
            'environment': 'Docker Container'
        }
    }
    
    # Return proper API Gateway response format
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
        },
        'body': json.dumps(response_data, indent=2)
    }
Subtask 1.3: Create Requirements File
Create a requirements file for Python dependencies:

# Create requirements.txt file
nano requirements.txt
Add the following content:

# No additional requirements needed for this basic example
# The AWS Lambda Python runtime includes standard libraries
Subtask 1.4: Create the Dockerfile
Now create a Dockerfile that will package our Lambda function:

# Create Dockerfile in the project root
nano Dockerfile
Add the following Dockerfile content:

# Use the official AWS Lambda Python runtime as base image
FROM public.ecr.aws/lambda/python:3.9

# Copy requirements and install dependencies
COPY requirements.txt ${LAMBDA_TASK_ROOT}
RUN pip install -r requirements.txt

# Copy function code to the Lambda task root directory
COPY src/lambda_function.py ${LAMBDA_TASK_ROOT}

# Set the CMD to your handler
CMD ["lambda_function.lambda_handler"]
Subtask 1.5: Build the Docker Image
Build the Docker image for your Lambda function:

# Build the Docker image with a descriptive tag
docker build -t my-lambda-function:latest .

# Verify the image was created successfully
docker images | grep my-lambda-function
Subtask 1.6: Test the Container Locally
Test your containerized Lambda function locally using the Lambda Runtime Interface Emulator:

# Run the container locally on port 9000
docker run -p 9000:8080 my-lambda-function:latest
Open a new terminal and test the function:

# Test the Lambda function with a sample event
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" \
  -d '{
    "httpMethod": "GET",
    "path": "/test",
    "queryStringParameters": {
      "name": "Docker",
      "version": "1.0"
    }
  }'
Stop the container by pressing Ctrl+C in the first terminal.

Task 2: Package and Push the Docker Container to Amazon ECR
Subtask 2.1: Configure AWS CLI
Configure your AWS CLI with appropriate credentials:

# Configure AWS CLI (you'll be prompted for credentials)
aws configure

# Verify your AWS identity
aws sts get-caller-identity
Subtask 2.2: Create an ECR Repository
Create a new ECR repository to store your Docker image:

# Create ECR repository
aws ecr create-repository \
  --repository-name my-lambda-function \
  --region us-east-1

# Get the repository URI (save this for later use)
aws ecr describe-repositories \
  --repository-names my-lambda-function \
  --region us-east-1 \
  --query 'repositories[0].repositoryUri' \
  --output text
Subtask 2.3: Authenticate Docker with ECR
Get authentication token and login to ECR:

# Get login token and authenticate Docker with ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com
Subtask 2.4: Tag and Push the Image
Tag your local image and push it to ECR:

# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Tag the image for ECR
docker tag my-lambda-function:latest \
  $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/my-lambda-function:latest

# Push the image to ECR
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/my-lambda-function:latest

# Verify the image was pushed successfully
aws ecr list-images --repository-name my-lambda-function --region us-east-1
Task 3: Deploy the Container as a Lambda Function
Subtask 3.1: Create IAM Role for Lambda
Create an IAM role that Lambda can assume:

# Create trust policy document
cat > lambda-trust-policy.json << EOF
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

# Create the IAM role
aws iam create-role \
  --role-name lambda-docker-execution-role \
  --assume-role-policy-document file://lambda-trust-policy.json

# Attach basic execution policy
aws iam attach-role-policy \
  --role-name lambda-docker-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
Subtask 3.2: Create the Lambda Function
Deploy your containerized Lambda function:

# Get the role ARN
ROLE_ARN=$(aws iam get-role \
  --role-name lambda-docker-execution-role \
  --query 'Role.Arn' \
  --output text)

# Get your account ID and construct image URI
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
IMAGE_URI="$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/my-lambda-function:latest"

# Create the Lambda function
aws lambda create-function \
  --function-name my-docker-lambda \
  --package-type Image \
  --code ImageUri=$IMAGE_URI \
  --role $ROLE_ARN \
  --timeout 30 \
  --memory-size 256 \
  --region us-east-1
Subtask 3.3: Test the Lambda Function
Test your deployed Lambda function:

# Create a test event file
cat > test-event.json << EOF
{
  "httpMethod": "GET",
  "path": "/hello",
  "queryStringParameters": {
    "message": "Hello from AWS Lambda",
    "source": "Docker Container"
  }
}
EOF

# Invoke the Lambda function
aws lambda invoke \
  --function-name my-docker-lambda \
  --payload file://test-event.json \
  --region us-east-1 \
  response.json

# View the response
cat response.json | python3 -m json.tool
Task 4: Trigger the Lambda Function via AWS API Gateway
Subtask 4.1: Create API Gateway REST API
Create a new REST API in API Gateway:

# Create the REST API
API_ID=$(aws apigateway create-rest-api \
  --name my-docker-lambda-api \
  --description "API for Docker Lambda function" \
  --region us-east-1 \
  --query 'id' \
  --output text)

echo "API ID: $API_ID"

# Get the root resource ID
ROOT_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --region us-east-1 \
  --query 'items[0].id' \
  --output text)

echo "Root Resource ID: $ROOT_RESOURCE_ID"
Subtask 4.2: Create API Resource and Method
Create a resource and method for your API:

# Create a new resource
RESOURCE_ID=$(aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_RESOURCE_ID \
  --path-part hello \
  --region us-east-1 \
  --query 'id' \
  --output text)

echo "Resource ID: $RESOURCE_ID"

# Create GET method
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET \
  --authorization-type NONE \
  --region us-east-1
Subtask 4.3: Configure Lambda Integration
Connect the API Gateway method to your Lambda function:

# Get Lambda function ARN
LAMBDA_ARN=$(aws lambda get-function \
  --function-name my-docker-lambda \
  --region us-east-1 \
  --query 'Configuration.FunctionArn' \
  --output text)

# Create integration
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri "arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/$LAMBDA_ARN/invocations" \
  --region us-east-1

# Grant API Gateway permission to invoke Lambda
aws lambda add-permission \
  --function-name my-docker-lambda \
  --statement-id api-gateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:$(aws sts get-caller-identity --query Account --output text):$API_ID/*/*" \
  --region us-east-1
Subtask 4.4: Deploy the API
Deploy your API to make it accessible:

# Create deployment
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod \
  --region us-east-1

# Get the API endpoint URL
API_URL="https://$API_ID.execute-api.us-east-1.amazonaws.com/prod/hello"
echo "API Endpoint: $API_URL"
Subtask 4.5: Test the API Gateway Integration
Test your API endpoint:

# Test the API endpoint
curl -X GET "$API_URL?name=Docker&environment=AWS"

# Test with different parameters
curl -X GET "$API_URL?message=Success&test=true"
Task 5: Monitor and Log Function Execution with CloudWatch
Subtask 5.1: View CloudWatch Logs
Access and analyze your Lambda function logs:

# List log groups for your Lambda function
aws logs describe-log-groups \
  --log-group-name-prefix "/aws/lambda/my-docker-lambda" \
  --region us-east-1

# Get the latest log stream
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name "/aws/lambda/my-docker-lambda" \
  --order-by LastEventTime \
  --descending \
  --max-items 1 \
  --region us-east-1 \
  --query 'logStreams[0].logStreamName' \
  --output text)

echo "Latest Log Stream: $LOG_STREAM"

# View recent log events
aws logs get-log-events \
  --log-group-name "/aws/lambda/my-docker-lambda" \
  --log-stream-name "$LOG_STREAM" \
  --region us-east-1 \
  --query 'events[*].[timestamp,message]' \
  --output table
Subtask 5.2: Generate Test Traffic and Monitor
Generate some test traffic to create logs:

# Make multiple API calls to generate logs
for i in {1..5}; do
  echo "Making request $i..."
  curl -s "$API_URL?request=$i&timestamp=$(date +%s)" | python3 -m json.tool
  sleep 2
done
Subtask 5.3: View Lambda Metrics
Check Lambda function metrics in CloudWatch:

# Get function invocation metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Invocations \
  --dimensions Name=FunctionName,Value=my-docker-lambda \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum \
  --region us-east-1

# Get function duration metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=my-docker-lambda \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum \
  --region us-east-1
Subtask 5.4: Create Custom Log Queries
Use CloudWatch Insights to analyze logs:

# Create a log insights query
cat > log-query.txt << 'EOF'
fields @timestamp, @message
| filter @message like /Hello from Dockerized Lambda/
| sort @timestamp desc
| limit 10
EOF

# Run the query (Note: This requires the query to be run via AWS Console or SDK)
echo "Log Insights Query created. You can run this in the AWS Console:"
cat log-query.txt
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Docker build fails

Solution: Ensure Dockerfile syntax is correct and base image is accessible
Check: Verify internet connectivity and Docker daemon is running
Issue 2: ECR authentication fails

Solution: Ensure AWS CLI is configured with proper permissions
Check: Verify IAM user has ECR permissions
Issue 3: Lambda function creation fails

Solution: Check IAM role permissions and image URI format
Check: Ensure ECR image exists and is accessible
Issue 4: API Gateway returns 502 error

Solution: Verify Lambda integration configuration and permissions
Check: Ensure Lambda function returns proper response format
Issue 5: CloudWatch logs not appearing

Solution: Check IAM role has CloudWatch Logs permissions
Check: Verify function is actually being invoked
Lab Cleanup
To avoid ongoing charges, clean up the resources created in this lab:

# Delete API Gateway
aws apigateway delete-rest-api --rest-api-id $API_ID --region us-east-1

# Delete Lambda function
aws lambda delete-function --function-name my-docker-lambda --region us-east-1

# Delete ECR repository
aws ecr delete-repository --repository-name my-lambda-function --force --region us-east-1

# Delete IAM role
aws iam detach-role-policy --role-name lambda-docker-execution-role --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name lambda-docker-execution-role

# Clean up local files
cd ..
rm -rf lambda-docker-project
Conclusion
Congratulations! You have successfully completed Lab 37: Docker and Cloud - Using Docker with AWS Lambda. In this comprehensive lab, you have accomplished the following:

Key Achievements:

Created a containerized Lambda function using Docker and Python, demonstrating how to package serverless applications in containers
Mastered ECR integration by pushing Docker images to Amazon Elastic Container Registry for secure storage and deployment
Deployed container-based Lambda functions using AWS CLI, showcasing modern serverless deployment practices
Configured API Gateway integration to create RESTful endpoints that trigger your containerized Lambda functions
Implemented comprehensive monitoring using CloudWatch for logging, metrics, and observability
Why This Matters: This lab demonstrates the powerful combination of containerization and serverless computing. Docker containers with AWS Lambda provide several advantages:

Consistency: Your application runs the same way locally and in production
Flexibility: Support for larger deployment packages and custom runtime environments
Portability: Easy migration between different cloud providers or on-premises environments
Development Efficiency: Simplified local testing and debugging capabilities
Real-World Applications: The skills you've learned are directly applicable to:

Microservices Architecture: Building scalable, containerized microservices
DevOps Pipelines: Implementing CI/CD workflows with containerized deployments
Multi-Cloud Strategies: Creating portable applications that can run across different cloud platforms
Enterprise Applications: Developing robust, monitored serverless solutions for production environments
Next Steps: Consider exploring advanced topics such as:

Container image optimization for faster cold starts
Multi-stage Docker builds for smaller images
Advanced CloudWatch monitoring and alerting
Integration with other AWS services like DynamoDB or S3
Implementing authentication and authorization with API Gateway
You now have the foundational knowledge to build, deploy, and monitor containerized serverless applications using industry-standard tools and practices. This combination of Docker and AWS Lambda represents a modern approach to cloud-native application development that is widely adopted in enterprise environments.
