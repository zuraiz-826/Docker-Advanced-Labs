Lab 105: Docker and Cloud - Deploying Docker Containers with AWS Fargate
Lab Objectives
By the end of this lab, you will be able to:

Create and configure an Amazon ECS cluster using AWS Fargate
Build Docker images and push them to Amazon Elastic Container Registry (ECR)
Create and deploy task definitions for containerized applications on Fargate
Configure Application Load Balancer for container services
Monitor container performance and implement auto-scaling using CloudWatch
Understand the fundamentals of serverless container deployment
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and containerization concepts
Familiarity with AWS console navigation
Basic knowledge of Linux command line operations
Understanding of web applications and HTTP protocols
AWS account with appropriate permissions for ECS, ECR, and related services
Note: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own virtual machine.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-installed with:

Docker Engine
AWS CLI v2
Git
Node.js and npm
Text editors (nano, vim)
Task 1: Create an ECS Cluster on AWS using AWS Fargate
Subtask 1.1: Configure AWS CLI
First, configure your AWS CLI with the provided credentials.

# Configure AWS CLI
aws configure

# When prompted, enter:
# AWS Access Key ID: [provided by instructor]
# AWS Secret Access Key: [provided by instructor]
# Default region name: us-east-1
# Default output format: json
Subtask 1.2: Create ECS Cluster
# Create a new ECS cluster
aws ecs create-cluster \
    --cluster-name fargate-lab-cluster \
    --capacity-providers FARGATE \
    --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1

# Verify cluster creation
aws ecs describe-clusters --clusters fargate-lab-cluster
Alternative: Using AWS Console

Navigate to Amazon ECS in the AWS Console
Click Create Cluster
Select Networking only (Powered by AWS Fargate)
Enter cluster name: fargate-lab-cluster
Click Create
Task 2: Build and Push a Docker Image to AWS ECR
Subtask 2.1: Create a Sample Application
Create a simple Node.js web application for demonstration.

# Create project directory
mkdir fargate-demo-app
cd fargate-demo-app

# Create package.json
cat > package.json << 'EOF'
{
  "name": "fargate-demo-app",
  "version": "1.0.0",
  "description": "Demo app for AWS Fargate",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
EOF

# Create server.js
cat > server.js << 'EOF'
const express = require('express');
const app = express();
const port = process.env.PORT || 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from AWS Fargate!',
    timestamp: new Date().toISOString(),
    hostname: require('os').hostname()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(port, '0.0.0.0', () => {
  console.log(`Server running on port ${port}`);
});
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install --production

COPY . .

EXPOSE 3000

USER node

CMD ["npm", "start"]
EOF
Subtask 2.2: Create ECR Repository
# Create ECR repository
aws ecr create-repository \
    --repository-name fargate-demo-app \
    --region us-east-1

# Get login token and authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com
Subtask 2.3: Build and Push Docker Image
# Get your AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
echo "Account ID: $ACCOUNT_ID"

# Build Docker image
docker build -t fargate-demo-app .

# Tag image for ECR
docker tag fargate-demo-app:latest $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/fargate-demo-app:latest

# Push image to ECR
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/fargate-demo-app:latest

# Verify image in ECR
aws ecr describe-images --repository-name fargate-demo-app
Task 3: Create a Task Definition for Fargate and Run the Container
Subtask 3.1: Create IAM Role for ECS Tasks
# Create trust policy for ECS tasks
cat > ecs-task-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create execution role
aws iam create-role \
    --role-name ecsTaskExecutionRole \
    --assume-role-policy-document file://ecs-task-trust-policy.json

# Attach managed policy
aws iam attach-role-policy \
    --role-name ecsTaskExecutionRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
Subtask 3.2: Create Task Definition
# Get account ID and create task definition
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

cat > task-definition.json << EOF
{
  "family": "fargate-demo-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::$ACCOUNT_ID:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "fargate-demo-container",
      "image": "$ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/fargate-demo-app:latest",
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
          "awslogs-group": "/ecs/fargate-demo-task",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
EOF

# Create CloudWatch log group
aws logs create-log-group --log-group-name /ecs/fargate-demo-task

# Register task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json
Subtask 3.3: Create VPC and Security Groups
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)
echo "VPC ID: $VPC_ID"

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# Create subnets in different AZs
SUBNET1_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
SUBNET2_ID=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1b --query 'Subnet.SubnetId' --output text)

# Enable auto-assign public IP
aws ec2 modify-subnet-attribute --subnet-id $SUBNET1_ID --map-public-ip-on-launch
aws ec2 modify-subnet-attribute --subnet-id $SUBNET2_ID --map-public-ip-on-launch

# Create route table and add route to IGW
ROUTE_TABLE_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $ROUTE_TABLE_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# Associate subnets with route table
aws ec2 associate-route-table --subnet-id $SUBNET1_ID --route-table-id $ROUTE_TABLE_ID
aws ec2 associate-route-table --subnet-id $SUBNET2_ID --route-table-id $ROUTE_TABLE_ID

# Create security group
SG_ID=$(aws ec2 create-security-group \
    --group-name fargate-demo-sg \
    --description "Security group for Fargate demo" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

# Add inbound rules
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 3000 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

echo "Subnet 1 ID: $SUBNET1_ID"
echo "Subnet 2 ID: $SUBNET2_ID"
echo "Security Group ID: $SG_ID"
Subtask 3.4: Run the Container
# Run task on Fargate
aws ecs run-task \
    --cluster fargate-lab-cluster \
    --task-definition fargate-demo-task \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[$SUBNET1_ID,$SUBNET2_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}"

# List running tasks
aws ecs list-tasks --cluster fargate-lab-cluster

# Get task details
TASK_ARN=$(aws ecs list-tasks --cluster fargate-lab-cluster --query 'taskArns[0]' --output text)
aws ecs describe-tasks --cluster fargate-lab-cluster --tasks $TASK_ARN
Task 4: Set up Load Balancing for the Containerized Service using an Application Load Balancer
Subtask 4.1: Create Application Load Balancer
# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
    --name fargate-demo-alb \
    --subnets $SUBNET1_ID $SUBNET2_ID \
    --security-groups $SG_ID \
    --query 'LoadBalancers[0].LoadBalancerArn' --output text)

echo "ALB ARN: $ALB_ARN"

# Get ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $ALB_ARN \
    --query 'LoadBalancers[0].DNSName' --output text)

echo "ALB DNS: $ALB_DNS"
Subtask 4.2: Create Target Group
# Create target group
TG_ARN=$(aws elbv2 create-target-group \
    --name fargate-demo-tg \
    --protocol HTTP \
    --port 3000 \
    --vpc-id $VPC_ID \
    --target-type ip \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3 \
    --query 'TargetGroups[0].TargetGroupArn' --output text)

echo "Target Group ARN: $TG_ARN"
Subtask 4.3: Create Listener
# Create listener
aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$TG_ARN
Subtask 4.4: Create ECS Service
# Create ECS service with load balancer
aws ecs create-service \
    --cluster fargate-lab-cluster \
    --service-name fargate-demo-service \
    --task-definition fargate-demo-task \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[$SUBNET1_ID,$SUBNET2_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}" \
    --load-balancers targetGroupArn=$TG_ARN,containerName=fargate-demo-container,containerPort=3000

# Wait for service to stabilize
echo "Waiting for service to become stable..."
aws ecs wait services-stable --cluster fargate-lab-cluster --services fargate-demo-service

# Check service status
aws ecs describe-services --cluster fargate-lab-cluster --services fargate-demo-service
Subtask 4.5: Test Load Balancer
# Test the application through load balancer
echo "Testing application at: http://$ALB_DNS"
curl -s http://$ALB_DNS | jq .

# Test multiple times to see load balancing
for i in {1..5}; do
  echo "Request $i:"
  curl -s http://$ALB_DNS | jq .hostname
  sleep 1
done
Task 5: Monitor and Scale Services using CloudWatch
Subtask 5.1: Create CloudWatch Dashboard
# Create CloudWatch dashboard
cat > dashboard-body.json << EOF
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
          [ "AWS/ECS", "CPUUtilization", "ServiceName", "fargate-demo-service", "ClusterName", "fargate-lab-cluster" ],
          [ ".", "MemoryUtilization", ".", ".", ".", "." ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "ECS Service Metrics"
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
          [ "AWS/ApplicationELB", "RequestCount", "LoadBalancer", "${ALB_ARN##*/}" ],
          [ ".", "TargetResponseTime", ".", "." ]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Load Balancer Metrics"
      }
    }
  ]
}
EOF

# Create dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "Fargate-Demo-Dashboard" \
    --dashboard-body file://dashboard-body.json
Subtask 5.2: Set up Auto Scaling
# Register scalable target
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --resource-id service/fargate-lab-cluster/fargate-demo-service \
    --scalable-dimension ecs:service:DesiredCount \
    --min-capacity 1 \
    --max-capacity 10

# Create scaling policy
POLICY_ARN=$(aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --resource-id service/fargate-lab-cluster/fargate-demo-service \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-name fargate-demo-scaling-policy \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
        "TargetValue": 70.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
        },
        "ScaleOutCooldown": 300,
        "ScaleInCooldown": 300
    }' \
    --query 'PolicyARN' --output text)

echo "Scaling Policy ARN: $POLICY_ARN"
Subtask 5.3: Create CloudWatch Alarms
# Create CPU utilization alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "Fargate-Demo-High-CPU" \
    --alarm-description "Alarm when CPU exceeds 80%" \
    --metric-name CPUUtilization \
    --namespace AWS/ECS \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --dimensions Name=ServiceName,Value=fargate-demo-service Name=ClusterName,Value=fargate-lab-cluster

# Create memory utilization alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "Fargate-Demo-High-Memory" \
    --alarm-description "Alarm when Memory exceeds 80%" \
    --metric-name MemoryUtilization \
    --namespace AWS/ECS \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --dimensions Name=ServiceName,Value=fargate-demo-service Name=ClusterName,Value=fargate-lab-cluster
Subtask 5.4: Generate Load for Testing
# Create a simple load testing script
cat > load_test.sh << 'EOF'
#!/bin/bash
ALB_DNS=$1
if [ -z "$ALB_DNS" ]; then
    echo "Usage: $0 <ALB_DNS_NAME>"
    exit 1
fi

echo "Starting load test against $ALB_DNS"
echo "Press Ctrl+C to stop"

while true; do
    for i in {1..10}; do
        curl -s http://$ALB_DNS > /dev/null &
    done
    sleep 1
done
EOF

chmod +x load_test.sh

# Run load test (in background)
echo "Starting load test..."
./load_test.sh $ALB_DNS &
LOAD_TEST_PID=$!

echo "Load test running with PID: $LOAD_TEST_PID"
echo "Monitor scaling in AWS Console or run: aws ecs describe-services --cluster fargate-lab-cluster --services fargate-demo-service"
echo "To stop load test: kill $LOAD_TEST_PID"
Subtask 5.5: Monitor Scaling Activity
# Monitor service scaling
watch -n 30 'aws ecs describe-services --cluster fargate-lab-cluster --services fargate-demo-service --query "services[0].{DesiredCount:desiredCount,RunningCount:runningCount,PendingCount:pendingCount}"'

# View scaling activities
aws application-autoscaling describe-scaling-activities \
    --service-namespace ecs \
    --resource-id service/fargate-lab-cluster/fargate-demo-service
Verification and Testing
Test Application Functionality
# Test application endpoints
echo "Testing main endpoint:"
curl -s http://$ALB_DNS | jq .

echo "Testing health endpoint:"
curl -s http://$ALB_DNS/health | jq .

# Check service health
aws elbv2 describe-target-health --target-group-arn $TG_ARN
Verify Monitoring Setup
# List CloudWatch alarms
aws cloudwatch describe-alarms --alarm-names "Fargate-Demo-High-CPU" "Fargate-Demo-High-Memory"

# Check dashboard
echo "Dashboard URL: https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:name=Fargate-Demo-Dashboard"
Troubleshooting Tips
Common Issues and Solutions
Issue: Task fails to start

Solution: Check CloudWatch logs for container errors
aws logs describe-log-streams --log-group-name /ecs/fargate-demo-task
Issue: Load balancer health checks failing

Solution: Verify security group allows traffic on port 3000 and health check endpoint is accessible
Issue: Auto scaling not working

Solution: Ensure CloudWatch metrics are being published and scaling policies are correctly configured
Issue: Cannot access application

Solution: Check if subnets have internet gateway route and security groups allow inbound traffic
Cleanup Commands
# Stop load test
kill $LOAD_TEST_PID 2>/dev/null

# Delete ECS service
aws ecs update-service --cluster fargate-lab-cluster --service fargate-demo-service --desired-count 0
aws ecs delete-service --cluster fargate-lab-cluster --service fargate-demo-service

# Delete load balancer components
aws elbv2 delete-listener --listener-arn $(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query 'Listeners[0].ListenerArn' --output text)
aws elbv2 delete-target-group --target-group-arn $TG_ARN
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN

# Delete ECS cluster
aws ecs delete-cluster --cluster fargate-lab-cluster

# Delete ECR repository
aws ecr delete-repository --repository-name fargate-demo-app --force

# Delete CloudWatch resources
aws cloudwatch delete-dashboards --dashboard-names Fargate-Demo-Dashboard
aws cloudwatch delete-alarms --alarm-names "Fargate-Demo-High-CPU" "Fargate-Demo-High-Memory"
aws logs delete-log-group --log-group-name /ecs/fargate-demo-task

# Delete networking resources
aws ec2 delete-security-group --group-id $SG_ID
aws ec2 delete-subnet --subnet-id $SUBNET1_ID
aws ec2 delete-subnet --subnet-id $SUBNET2_ID
aws ec2 detach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-route-table --route-table-id $ROUTE_TABLE_ID
aws ec2 delete-vpc --vpc-id $VPC_ID

# Delete IAM role
aws iam detach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
aws iam delete-role --role-name ecsTaskExecutionRole
Conclusion
Congratulations! You have successfully completed Lab 105: Docker and Cloud - Deploying Docker Containers with AWS Fargate.

What You Accomplished
In this lab, you have:

Created an ECS Cluster: Set up a serverless container orchestration environment using AWS Fargate
Built and Deployed Container Images: Created a containerized Node.js application and pushed it to Amazon ECR
Configured Container Services: Created task definitions and deployed containers with proper networking and security
Implemented Load Balancing: Set up an Application Load Balancer to distribute traffic across multiple container instances
Established Monitoring and Auto-scaling: Configured CloudWatch monitoring, alarms, and automatic scaling policies
Why This Matters
Serverless Container Deployment: AWS Fargate eliminates the need to manage underlying infrastructure, allowing you to focus on your applications rather than server management.

Scalability and Reliability: The combination of ECS, Fargate, and Application Load Balancer provides automatic scaling and high availability for your containerized applications.

Production-Ready Architecture: The skills you've learned represent industry best practices for deploying containerized applications in the cloud, making your applications more resilient and cost-effective.

Monitoring and Observability: CloudWatch integration provides essential insights into application performance and enables proactive scaling and alerting.

Next Steps
Explore advanced ECS features like service discovery and task placement strategies
Implement CI/CD pipelines for automated container deployments
Learn about container security best practices and AWS security services
Investigate multi-region deployments and disaster recovery strategies
This lab has provided you with fundamental skills for deploying and managing containerized applications in AWS, preparing you for the Docker Certified Associate (DCA) certification and real-world cloud container deployments.
