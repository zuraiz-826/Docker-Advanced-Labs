Lab 58: Docker and AWS ECS - ECS with Fargate
Lab Objectives
By the end of this lab, you will be able to:

• Set up Amazon ECS with Fargate for serverless container deployments • Build and push Docker images to Amazon Elastic Container Registry (ECR) • Create and configure Fargate task definitions • Launch containers using ECS Fargate • Configure auto-scaling policies for Fargate tasks • Monitor ECS tasks and containers using Amazon CloudWatch • Understand the benefits of serverless container orchestration

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker containers and containerization concepts • Familiarity with AWS services and the AWS Management Console • Basic knowledge of Linux command line operations • Understanding of web applications and HTTP protocols • AWS CLI installed and configured with appropriate permissions

Required AWS Permissions: • ECS Full Access • ECR Full Access • CloudWatch Full Access • IAM permissions for creating roles and policies

Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with all necessary tools pre-installed. Simply click Start Lab to begin - no need to build your own virtual machine or install additional software.

Your lab environment includes: • AWS CLI pre-configured • Docker Engine installed and running • Git for version control • Text editors (nano, vim) • All necessary development tools

Task 1: Set up AWS ECS with Fargate
Subtask 1.1: Create an ECS Cluster
First, we'll create an ECS cluster that will use Fargate as the launch type.

Access the AWS Management Console

# Verify AWS CLI configuration
aws sts get-caller-identity
Create ECS Cluster using AWS CLI

# Create a new ECS cluster
aws ecs create-cluster \
  --cluster-name fargate-lab-cluster \
  --capacity-providers FARGATE \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1 \
  --region us-east-1
Verify cluster creation

# List all clusters
aws ecs list-clusters

# Describe the specific cluster
aws ecs describe-clusters --clusters fargate-lab-cluster
Subtask 1.2: Create IAM Roles for ECS Tasks
ECS tasks running on Fargate require specific IAM roles for proper execution.

Create Task Execution Role

# Create trust policy document
cat > task-execution-assume-role-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the IAM role
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document file://task-execution-assume-role-policy.json

# Attach the managed policy
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
Create Task Role (for application permissions)

# Create task role
aws iam create-role \
  --role-name ecsTaskRole \
  --assume-role-policy-document file://task-execution-assume-role-policy.json

# Create custom policy for CloudWatch access
cat > cloudwatch-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Create and attach the policy
aws iam create-policy \
  --policy-name CloudWatchLogsPolicy \
  --policy-document file://cloudwatch-policy.json

# Get account ID for policy ARN
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

aws iam attach-role-policy \
  --role-name ecsTaskRole \
  --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/CloudWatchLogsPolicy
Task 2: Build and Push a Docker Image to ECR
Subtask 2.1: Create an ECR Repository
Create ECR repository

# Create a new ECR repository
aws ecr create-repository \
  --repository-name fargate-web-app \
  --region us-east-1
Get repository URI

# Get the repository URI
REPO_URI=$(aws ecr describe-repositories \
  --repository-names fargate-web-app \
  --query 'repositories[0].repositoryUri' \
  --output text)

echo "Repository URI: $REPO_URI"
Subtask 2.2: Create a Sample Web Application
Create application directory and files
# Create project directory
mkdir fargate-web-app
cd fargate-web-app

# Create a simple Node.js application
cat > app.js << EOF
const express = require('express');
const app = express();
const port = 3000;

// Health check endpoint
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime()
  });
});

// Main application endpoint
app.get('/', (req, res) => {
  res.json({
    message: 'Hello from ECS Fargate!',
    container_id: require('os').hostname(),
    timestamp: new Date().toISOString()
  });
});

// Metrics endpoint for monitoring
app.get('/metrics', (req, res) => {
  res.json({
    memory_usage: process.memoryUsage(),
    uptime: process.uptime(),
    cpu_usage: process.cpuUsage()
  });
});

app.listen(port, '0.0.0.0', () => {
  console.log(\`Server running on port \${port}\`);
});
EOF

# Create package.json
cat > package.json << EOF
{
  "name": "fargate-web-app",
  "version": "1.0.0",
  "description": "Sample web app for ECS Fargate",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF
Subtask 2.3: Create Dockerfile
Create optimized Dockerfile
cat > Dockerfile << EOF
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory
WORKDIR /usr/src/app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install --only=production

# Copy application code
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Change ownership of the app directory
RUN chown -R nodejs:nodejs /usr/src/app
USER nodejs

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node -e "require('http').get('http://localhost:3000/health', (res) => { process.exit(res.statusCode === 200 ? 0 : 1) }).on('error', () => process.exit(1))"

# Start the application
CMD ["npm", "start"]
EOF
Subtask 2.4: Build and Push Docker Image
Build the Docker image

# Build the image
docker build -t fargate-web-app .

# Tag the image for ECR
docker tag fargate-web-app:latest $REPO_URI:latest
docker tag fargate-web-app:latest $REPO_URI:v1.0
Authenticate Docker with ECR

# Get ECR login token and authenticate
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $REPO_URI
Push image to ECR

# Push both tags
docker push $REPO_URI:latest
docker push $REPO_URI:v1.0

# Verify the push
aws ecr list-images --repository-name fargate-web-app
Task 3: Create a Fargate Task Definition and Launch the Container
Subtask 3.1: Create CloudWatch Log Group
Create log group for container logs
# Create CloudWatch log group
aws logs create-log-group \
  --log-group-name /ecs/fargate-web-app \
  --region us-east-1

# Set retention policy
aws logs put-retention-policy \
  --log-group-name /ecs/fargate-web-app \
  --retention-in-days 7
Subtask 3.2: Create Task Definition
Create task definition JSON

# Get account ID and region
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="us-east-1"

cat > task-definition.json << EOF
{
  "family": "fargate-web-app-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskRole",
  "containerDefinitions": [
    {
      "name": "web-app",
      "image": "${REPO_URI}:latest",
      "portMappings": [
        {
          "containerPort": 3000,
          "protocol": "tcp"
        }
      ],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/fargate-web-app",
          "awslogs-region": "${REGION}",
          "awslogs-stream-prefix": "ecs"
        }
      },
      "healthCheck": {
        "command": [
          "CMD-SHELL",
          "curl -f http://localhost:3000/health || exit 1"
        ],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 60
      },
      "environment": [
        {
          "name": "NODE_ENV",
          "value": "production"
        },
        {
          "name": "PORT",
          "value": "3000"
        }
      ]
    }
  ]
}
EOF
Register the task definition

# Register task definition
aws ecs register-task-definition \
  --cli-input-json file://task-definition.json

# Verify registration
aws ecs describe-task-definition \
  --task-definition fargate-web-app-task
Subtask 3.3: Create Security Group and Network Configuration
Get default VPC information

# Get default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)

echo "Default VPC ID: $VPC_ID"

# Get subnet IDs
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].SubnetId' \
  --output text)

echo "Subnet IDs: $SUBNET_IDS"
Create security group

# Create security group
SECURITY_GROUP_ID=$(aws ec2 create-security-group \
  --group-name fargate-web-app-sg \
  --description "Security group for Fargate web application" \
  --vpc-id $VPC_ID \
  --query 'GroupId' \
  --output text)

echo "Security Group ID: $SECURITY_GROUP_ID"

# Add inbound rules
aws ec2 authorize-security-group-ingress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 3000 \
  --cidr 0.0.0.0/0

# Add outbound rule for HTTPS (ECR access)
aws ec2 authorize-security-group-egress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Add outbound rule for HTTP
aws ec2 authorize-security-group-egress \
  --group-id $SECURITY_GROUP_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0
Subtask 3.4: Launch Fargate Task
Run the task

# Get first subnet ID
FIRST_SUBNET=$(echo $SUBNET_IDS | cut -d' ' -f1)

# Run the task
TASK_ARN=$(aws ecs run-task \
  --cluster fargate-lab-cluster \
  --task-definition fargate-web-app-task \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[$FIRST_SUBNET],securityGroups=[$SECURITY_GROUP_ID],assignPublicIp=ENABLED}" \
  --query 'tasks[0].taskArn' \
  --output text)

echo "Task ARN: $TASK_ARN"
Monitor task status

# Check task status
aws ecs describe-tasks \
  --cluster fargate-lab-cluster \
  --tasks $TASK_ARN \
  --query 'tasks[0].lastStatus' \
  --output text

# Get task details including public IP
aws ecs describe-tasks \
  --cluster fargate-lab-cluster \
  --tasks $TASK_ARN \
  --query 'tasks[0].attachments[0].details'
Get public IP and test the application

# Extract public IP
PUBLIC_IP=$(aws ecs describe-tasks \
  --cluster fargate-lab-cluster \
  --tasks $TASK_ARN \
  --query 'tasks[0].attachments[0].details[?name==`publicIPv4Address`].value' \
  --output text)

echo "Public IP: $PUBLIC_IP"

# Test the application (wait a few minutes for the task to be running)
sleep 60
curl http://$PUBLIC_IP:3000
curl http://$PUBLIC_IP:3000/health
Task 4: Configure Auto-scaling for Fargate Tasks
Subtask 4.1: Create ECS Service
Create service definition

cat > service-definition.json << EOF
{
  "serviceName": "fargate-web-app-service",
  "cluster": "fargate-lab-cluster",
  "taskDefinition": "fargate-web-app-task",
  "desiredCount": 2,
  "launchType": "FARGATE",
  "networkConfiguration": {
    "awsvpcConfiguration": {
      "subnets": ["$FIRST_SUBNET"],
      "securityGroups": ["$SECURITY_GROUP_ID"],
      "assignPublicIp": "ENABLED"
    }
  },
  "deploymentConfiguration": {
    "maximumPercent": 200,
    "minimumHealthyPercent": 50
  },
  "healthCheckGracePeriodSeconds": 60
}
EOF
Create the service

# Create ECS service
aws ecs create-service \
  --cli-input-json file://service-definition.json

# Wait for service to stabilize
aws ecs wait services-stable \
  --cluster fargate-lab-cluster \
  --services fargate-web-app-service
Subtask 4.2: Set up Application Auto Scaling
Register scalable target

# Register the service as a scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/fargate-lab-cluster/fargate-web-app-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 \
  --max-capacity 10 \
  --role-arn arn:aws:iam::${ACCOUNT_ID}:role/aws-service-role/ecs.application-autoscaling.amazonaws.com/AWSServiceRoleForApplicationAutoScaling_ECSService
Create scaling policies

# Scale-out policy
cat > scale-out-policy.json << EOF
{
  "PolicyName": "fargate-scale-out-policy",
  "ServiceNamespace": "ecs",
  "ResourceId": "service/fargate-lab-cluster/fargate-web-app-service",
  "ScalableDimension": "ecs:service:DesiredCount",
  "PolicyType": "TargetTrackingScaling",
  "TargetTrackingScalingPolicyConfiguration": {
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleOutCooldown": 300,
    "ScaleInCooldown": 300
  }
}
EOF

# Create the scaling policy
aws application-autoscaling put-scaling-policy \
  --cli-input-json file://scale-out-policy.json
Create memory-based scaling policy

# Memory-based scaling policy
cat > memory-scale-policy.json << EOF
{
  "PolicyName": "fargate-memory-scale-policy",
  "ServiceNamespace": "ecs",
  "ResourceId": "service/fargate-lab-cluster/fargate-web-app-service",
  "ScalableDimension": "ecs:service:DesiredCount",
  "PolicyType": "TargetTrackingScaling",
  "TargetTrackingScalingPolicyConfiguration": {
    "TargetValue": 80.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageMemoryUtilization"
    },
    "ScaleOutCooldown": 300,
    "ScaleInCooldown": 300
  }
}
EOF

# Create the memory scaling policy
aws application-autoscaling put-scaling-policy \
  --cli-input-json file://memory-scale-policy.json
Subtask 4.3: Verify Auto Scaling Configuration
Check scaling policies
# List scaling policies
aws application-autoscaling describe-scaling-policies \
  --service-namespace ecs \
  --resource-id service/fargate-lab-cluster/fargate-web-app-service

# Check scalable targets
aws application-autoscaling describe-scalable-targets \
  --service-namespace ecs \
  --resource-ids service/fargate-lab-cluster/fargate-web-app-service
Task 5: Monitor ECS Tasks using Amazon CloudWatch
Subtask 5.1: Create Custom CloudWatch Dashboard
Create dashboard configuration

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
          [ "AWS/ECS", "CPUUtilization", "ServiceName", "fargate-web-app-service", "ClusterName", "fargate-lab-cluster" ],
          [ ".", "MemoryUtilization", ".", ".", ".", "." ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "ECS Service Metrics",
        "period": 300
      }
    },
    {
      "type": "metric",
      "x": 0,
      "y": 6,
      "width": 12,
      "height": 6,
      "properties": {
        "metrics": [
          [ "AWS/ECS", "RunningTaskCount", "ServiceName", "fargate-web-app-service", "ClusterName", "fargate-lab-cluster" ],
          [ ".", "PendingTaskCount", ".", ".", ".", "." ],
          [ ".", "DesiredCount", ".", ".", ".", "." ]
        ],
        "view": "timeSeries",
        "stacked": false,
        "region": "us-east-1",
        "title": "Task Counts",
        "period": 300
      }
    }
  ]
}
EOF
Create the dashboard

# Create CloudWatch dashboard
aws cloudwatch put-dashboard \
  --dashboard-name "ECS-Fargate-Monitoring" \
  --dashboard-body file://dashboard-config.json
Subtask 5.2: Set up CloudWatch Alarms
Create CPU utilization alarm

# High CPU alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-HighCPU-fargate-web-app" \
  --alarm-description "Alarm when CPU exceeds 80%" \
  --metric-name CPUUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=ServiceName,Value=fargate-web-app-service Name=ClusterName,Value=fargate-lab-cluster
Create memory utilization alarm

# High memory alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-HighMemory-fargate-web-app" \
  --alarm-description "Alarm when Memory exceeds 85%" \
  --metric-name MemoryUtilization \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 85 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --dimensions Name=ServiceName,Value=fargate-web-app-service Name=ClusterName,Value=fargate-lab-cluster
Create task count alarm

# Low task count alarm
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-LowTaskCount-fargate-web-app" \
  --alarm-description "Alarm when running tasks is less than desired" \
  --metric-name RunningTaskCount \
  --namespace AWS/ECS \
  --statistic Average \
  --period 300 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=ServiceName,Value=fargate-web-app-service Name=ClusterName,Value=fargate-lab-cluster
Subtask 5.3: Monitor Application Logs
View container logs

# Get log streams
aws logs describe-log-streams \
  --log-group-name /ecs/fargate-web-app \
  --order-by LastEventTime \
  --descending

# Get latest log stream name
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name /ecs/fargate-web-app \
  --order-by LastEventTime \
  --descending \
  --max-items 1 \
  --query 'logStreams[0].logStreamName' \
  --output text)

# View recent logs
aws logs get-log-events \
  --log-group-name /ecs/fargate-web-app \
  --log-stream-name $LOG_STREAM \
  --start-time $(date -d '10 minutes ago' +%s)000
Create log insights queries

# Create a query to analyze application performance
cat > log-insights-query.txt << EOF
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
EOF

# Run the query (replace with actual log group)
aws logs start-query \
  --log-group-name /ecs/fargate-web-app \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string "$(cat log-insights-query.txt)"
Subtask 5.4: Test Monitoring and Scaling
Generate load to test auto-scaling

# Create a simple load testing script
cat > load-test.sh << 'EOF'
#!/bin/bash

# Get service tasks and their public IPs
TASK_ARNS=$(aws ecs list-tasks \
  --cluster fargate-lab-cluster \
  --service-name fargate-web-app-service \
  --query 'taskArns' \
  --output text)

echo "Generating load on ECS service..."

for i in {1..100}; do
  for TASK_ARN in $TASK_ARNS; do
    PUBLIC_IP=$(aws ecs describe-tasks \
      --cluster fargate-lab-cluster \
      --tasks $TASK_ARN \
      --query 'tasks[0].attachments[0].details[?name==`publicIPv4Address`].value' \
      --output text)
    
    if [ ! -z "$PUBLIC_IP" ]; then
      curl -s http://$PUBLIC_IP:3000/ > /dev/null &
      curl -s http://$PUBLIC_IP:3000/metrics > /dev/null &
    fi
  done
  sleep 1
done

wait
echo "Load test completed"
EOF

chmod +x load-test.sh
./load-test.sh
Monitor scaling events

# Check service events
aws ecs describe-services \
  --cluster fargate-lab-cluster \
  --services fargate-web-app-service \
  --query 'services[0].events[0:5]'

# Check current task count
aws ecs describe-services \
  --cluster fargate-lab-cluster \
  --services fargate-web-app-service \
  --query 'services[0].{DesiredCount:desiredCount,RunningCount:runningCount,PendingCount:pendingCount}'
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Task fails to start

# Check task definition and logs
aws ecs describe-tasks --cluster fargate-lab-cluster --tasks $TASK_ARN
aws logs get-log-events --log-group-name /ecs/fargate-web-app --log-stream-name $LOG_STREAM
Issue 2: Cannot access application

# Verify security group rules
aws ec2 describe-security-groups --group-ids $SECURITY_GROUP_ID

# Check task network configuration
aws ecs describe-tasks --cluster fargate-lab-cluster --tasks $TASK_ARN --query 'tasks[0].attachments'
Issue 3: Auto-scaling not working

# Check scaling policies
aws application-autoscaling describe-scaling-policies --service-namespace ecs

# Verify CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ECS \
  --metric-name CPUUtilization \
  --dimensions Name=ServiceName,Value=fargate-web-app-service Name=ClusterName,Value=fargate-lab-cluster \
  --start-time $(date -d '1 hour ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 300 \
  --statistics Average
Issue 4: High costs

# Check running tasks and stop unnecessary ones
aws ecs list-tasks --cluster fargate-lab-cluster
aws ecs stop-task --cluster fargate-lab-cluster --task $TASK_ARN
Lab Cleanup
To avoid ongoing charges, clean up the resources created in this lab:

# Stop all tasks
aws ecs update-service \
  --cluster fargate-lab-cluster \
  --service fargate-web-app-service \
  --desired-count 0

# Delete service
aws ecs delete-service \
  --cluster fargate-lab-cluster \
  --service fargate-web-app-service \
  --force

# Delete cluster
aws ecs delete-cluster --cluster fargate-lab-cluster

# Delete ECR repository
aws ecr delete-repository \
  --repository-name fargate-web-app \
  --force

# Delete CloudWatch resources
aws logs delete-log-group --log-group-name /ecs/fargate-web-app
aws cloudwatch delete-dashboard --dashboard-name ECS-Fargate-Monitoring

# Delete alarms
aws cloudwatch delete-alarms \
  --alarm-names ECS-HighCPU-fargate-web-app ECS-HighMemory-fargate-web-app ECS-LowTaskCount-fargate-web-app

# Delete security group
aws ec2 delete-security-group --group-id $SECURITY_GROUP_ID

# Delete IAM roles and policies
aws iam detach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam detach-role-policy --role-name ecsTaskRole --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/CloudWatchLogsPolicy
aws iam delete-role --role-name ecsTaskExecutionRole
aws iam delete-role --role-name ecsTaskRole
aws iam delete-policy --policy-
