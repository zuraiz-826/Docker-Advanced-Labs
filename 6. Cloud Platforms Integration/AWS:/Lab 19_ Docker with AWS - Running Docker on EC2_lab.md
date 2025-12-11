Lab 19: Docker with AWS - Running Docker on EC2
Lab Objectives
By the end of this lab, you will be able to:

Launch and configure an EC2 instance with Docker pre-installed
Use docker-machine to manage remote Docker hosts on AWS
Push Docker images to Amazon Elastic Container Registry (ECR)
Deploy Docker containers on EC2 using Amazon ECS (Elastic Container Service)
Monitor containerized applications using AWS CloudWatch
Understand the integration between Docker and AWS services
Implement container orchestration best practices in cloud environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, Dockerfile)
Familiarity with Linux command line operations
Basic knowledge of AWS services and concepts
Understanding of cloud computing fundamentals
Experience with terminal/command prompt usage
Technical Requirements:

AWS account with appropriate permissions
Basic understanding of networking concepts
Familiarity with JSON configuration files
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install additional software.

Your lab environment includes:

Pre-configured AWS CLI
Docker and docker-machine installed
Necessary IAM permissions configured
Access to AWS services (EC2, ECR, ECS, CloudWatch)
Task 1: Launch an EC2 Instance with Docker Pre-installed
Subtask 1.1: Create a Security Group for Docker
First, we'll create a security group that allows the necessary ports for Docker communication.

Create the security group:
# Create security group for Docker
aws ec2 create-security-group \
    --group-name docker-sg \
    --description "Security group for Docker on EC2"
Add inbound rules for Docker:
# Get your security group ID
SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
    --group-names docker-sg \
    --query 'SecurityGroups[0].GroupId' \
    --output text)

# Allow SSH access
aws ec2 authorize-security-group-ingress \
    --group-id $SECURITY_GROUP_ID \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0

# Allow Docker daemon port
aws ec2 authorize-security-group-ingress \
    --group-id $SECURITY_GROUP_ID \
    --protocol tcp \
    --port 2376 \
    --cidr 0.0.0.0/0

# Allow HTTP traffic
aws ec2 authorize-security-group-ingress \
    --group-id $SECURITY_GROUP_ID \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

# Allow HTTPS traffic
aws ec2 authorize-security-group-ingress \
    --group-id $SECURITY_GROUP_ID \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0
Subtask 1.2: Launch EC2 Instance with Docker
Create a user data script for Docker installation:
# Create user data script
cat > docker-install.sh << 'EOF'
#!/bin/bash
yum update -y
yum install -y docker
systemctl start docker
systemctl enable docker
usermod -a -G docker ec2-user

# Install docker-compose
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose

# Create a simple test container
docker run -d --name test-nginx -p 80:80 nginx:latest
EOF
Launch the EC2 instance:
# Launch EC2 instance with Docker
aws ec2 run-instances \
    --image-id ami-0c02fb55956c7d316 \
    --count 1 \
    --instance-type t2.micro \
    --key-name your-key-pair \
    --security-group-ids $SECURITY_GROUP_ID \
    --user-data file://docker-install.sh \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Docker-Lab-Instance}]'
Get the instance information:
# Get instance ID and public IP
INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=Docker-Lab-Instance" "Name=instance-state-name,Values=running" \
    --query 'Reservations[0].Instances[0].InstanceId' \
    --output text)

PUBLIC_IP=$(aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query 'Reservations[0].Instances[0].PublicIpAddress' \
    --output text)

echo "Instance ID: $INSTANCE_ID"
echo "Public IP: $PUBLIC_IP"
Subtask 1.3: Verify Docker Installation
Connect to the instance and verify Docker:
# SSH into the instance (replace with your key file)
ssh -i your-key-pair.pem ec2-user@$PUBLIC_IP

# Once connected, verify Docker installation
sudo docker --version
sudo docker ps
sudo docker images
Test the running container:
# Check if nginx is running
curl http://localhost

# View container logs
sudo docker logs test-nginx
Task 2: Use docker-machine to Manage Remote Docker Hosts
Subtask 2.1: Install and Configure docker-machine
Install docker-machine on your local environment:
# Download docker-machine
curl -L https://github.com/docker/machine/releases/latest/download/docker-machine-$(uname -s)-$(uname -m) > docker-machine

# Make it executable and move to PATH
chmod +x docker-machine
sudo mv docker-machine /usr/local/bin/
Verify docker-machine installation:
docker-machine version
Subtask 2.2: Create a Docker Machine on AWS
Set up AWS credentials for docker-machine:
# Export AWS credentials (if not already configured)
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_DEFAULT_REGION=us-east-1
Create a new Docker machine:
# Create Docker machine on AWS
docker-machine create \
    --driver amazonec2 \
    --amazonec2-instance-type t2.micro \
    --amazonec2-security-group docker-sg \
    --amazonec2-region us-east-1 \
    docker-aws-machine
List and inspect the machine:
# List all Docker machines
docker-machine ls

# Get machine details
docker-machine inspect docker-aws-machine

# Get machine IP
docker-machine ip docker-aws-machine
Subtask 2.3: Connect to Remote Docker Host
Configure Docker client to connect to remote host:
# Set environment to connect to remote Docker
eval $(docker-machine env docker-aws-machine)

# Verify connection
docker info
Run containers on remote host:
# Run a simple web application
docker run -d --name remote-app -p 8080:80 nginx:alpine

# List running containers
docker ps

# Test the application
curl http://$(docker-machine ip docker-aws-machine):8080
Task 3: Push Docker Images to Amazon Elastic Container Registry (ECR)
Subtask 3.1: Create an ECR Repository
Create ECR repository:
# Create ECR repository
aws ecr create-repository \
    --repository-name docker-lab-app \
    --region us-east-1
Get repository URI:
# Get repository URI
REPO_URI=$(aws ecr describe-repositories \
    --repository-names docker-lab-app \
    --query 'repositories[0].repositoryUri' \
    --output text)

echo "Repository URI: $REPO_URI"
Subtask 3.2: Build and Tag a Custom Docker Image
Create a simple web application:
# Create application directory
mkdir docker-lab-app
cd docker-lab-app

# Create a simple HTML file
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>Docker Lab App</title>
</head>
<body>
    <h1>Hello from Docker on AWS!</h1>
    <p>This application is running in a Docker container on AWS EC2.</p>
    <p>Lab 19: Docker with AWS Integration</p>
</body>
</html>
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
Build the Docker image:
# Build the image
docker build -t docker-lab-app:latest .

# Tag for ECR
docker tag docker-lab-app:latest $REPO_URI:latest
docker tag docker-lab-app:latest $REPO_URI:v1.0
Subtask 3.3: Push Image to ECR
Authenticate Docker with ECR:
# Get ECR login token and authenticate
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $REPO_URI
Push the image:
# Push both tags
docker push $REPO_URI:latest
docker push $REPO_URI:v1.0

# Verify the push
aws ecr list-images --repository-name docker-lab-app
Task 4: Deploy Docker Containers on EC2 using ECS
Subtask 4.1: Create ECS Cluster
Create ECS cluster:
# Create ECS cluster
aws ecs create-cluster --cluster-name docker-lab-cluster

# Verify cluster creation
aws ecs describe-clusters --clusters docker-lab-cluster
Subtask 4.2: Create Task Definition
Create task definition JSON:
cat > task-definition.json << EOF
{
    "family": "docker-lab-task",
    "networkMode": "bridge",
    "requiresCompatibilities": ["EC2"],
    "cpu": "256",
    "memory": "512",
    "containerDefinitions": [
        {
            "name": "docker-lab-container",
            "image": "$REPO_URI:latest",
            "memory": 512,
            "essential": true,
            "portMappings": [
                {
                    "containerPort": 80,
                    "hostPort": 8080,
                    "protocol": "tcp"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/docker-lab-task",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
    ]
}
EOF
Create CloudWatch log group:
# Create log group for ECS tasks
aws logs create-log-group --log-group-name /ecs/docker-lab-task
Register task definition:
# Register the task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json
Subtask 4.3: Launch ECS Service
Create ECS service:
# Create service
aws ecs create-service \
    --cluster docker-lab-cluster \
    --service-name docker-lab-service \
    --task-definition docker-lab-task \
    --desired-count 1 \
    --launch-type EC2
Verify service deployment:
# Check service status
aws ecs describe-services \
    --cluster docker-lab-cluster \
    --services docker-lab-service

# List running tasks
aws ecs list-tasks --cluster docker-lab-cluster
Task 5: Monitor Containers using CloudWatch
Subtask 5.1: Set up CloudWatch Monitoring
Create CloudWatch dashboard:
# Create dashboard configuration
cat > dashboard-config.json << 'EOF'
{
    "widgets": [
        {
            "type": "metric",
            "properties": {
                "metrics": [
                    ["AWS/ECS", "CPUUtilization", "ServiceName", "docker-lab-service", "ClusterName", "docker-lab-cluster"],
                    ["AWS/ECS", "MemoryUtilization", "ServiceName", "docker-lab-service", "ClusterName", "docker-lab-cluster"]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-east-1",
                "title": "ECS Service Metrics"
            }
        }
    ]
}
EOF

# Create dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "Docker-Lab-Dashboard" \
    --dashboard-body file://dashboard-config.json
Subtask 5.2: Create CloudWatch Alarms
Create CPU utilization alarm:
# Create CPU alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "Docker-Lab-High-CPU" \
    --alarm-description "Alarm when CPU exceeds 70%" \
    --metric-name CPUUtilization \
    --namespace AWS/ECS \
    --statistic Average \
    --period 300 \
    --threshold 70 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=ServiceName,Value=docker-lab-service Name=ClusterName,Value=docker-lab-cluster \
    --evaluation-periods 2
Create memory utilization alarm:
# Create memory alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "Docker-Lab-High-Memory" \
    --alarm-description "Alarm when Memory exceeds 80%" \
    --metric-name MemoryUtilization \
    --namespace AWS/ECS \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=ServiceName,Value=docker-lab-service Name=ClusterName,Value=docker-lab-cluster \
    --evaluation-periods 2
Subtask 5.3: View Logs and Metrics
View container logs:
# Get log streams
aws logs describe-log-streams \
    --log-group-name /ecs/docker-lab-task

# View recent logs
aws logs tail /ecs/docker-lab-task --follow
Query CloudWatch metrics:
# Get CPU metrics
aws cloudwatch get-metric-statistics \
    --namespace AWS/ECS \
    --metric-name CPUUtilization \
    --dimensions Name=ServiceName,Value=docker-lab-service Name=ClusterName,Value=docker-lab-cluster \
    --statistics Average \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Docker daemon not accessible

# Solution: Check if Docker service is running
sudo systemctl status docker
sudo systemctl start docker
Issue 2: ECR authentication fails

# Solution: Re-authenticate with ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $REPO_URI
Issue 3: ECS task fails to start

# Solution: Check task definition and logs
aws ecs describe-tasks --cluster docker-lab-cluster --tasks TASK_ARN
aws logs tail /ecs/docker-lab-task
Issue 4: Security group blocking connections

# Solution: Verify security group rules
aws ec2 describe-security-groups --group-ids $SECURITY_GROUP_ID
Lab Cleanup
To avoid ongoing charges, clean up the resources created in this lab:

# Stop ECS service
aws ecs update-service \
    --cluster docker-lab-cluster \
    --service docker-lab-service \
    --desired-count 0

# Delete ECS service
aws ecs delete-service \
    --cluster docker-lab-cluster \
    --service docker-lab-service

# Delete ECS cluster
aws ecs delete-cluster --cluster docker-lab-cluster

# Delete ECR repository
aws ecr delete-repository \
    --repository-name docker-lab-app \
    --force

# Terminate EC2 instances
aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# Delete docker-machine
docker-machine rm docker-aws-machine

# Delete security group
aws ec2 delete-security-group --group-id $SECURITY_GROUP_ID

# Delete CloudWatch resources
aws cloudwatch delete-dashboards --dashboard-names Docker-Lab-Dashboard
aws cloudwatch delete-alarms --alarm-names Docker-Lab-High-CPU Docker-Lab-High-Memory
aws logs delete-log-group --log-group-name /ecs/docker-lab-task
Conclusion
Congratulations! You have successfully completed Lab 19: Docker with AWS. In this comprehensive lab, you have:

Launched and configured EC2 instances with Docker pre-installed, learning how to set up containerized environments in the cloud
Mastered docker-machine for managing remote Docker hosts, enabling you to control Docker environments from anywhere
Implemented ECR integration by pushing custom Docker images to Amazon's container registry service
Deployed containers using ECS, understanding how AWS orchestrates containerized applications at scale
Set up comprehensive monitoring with CloudWatch to track container performance and health
Why This Matters:

This lab demonstrates the powerful integration between Docker and AWS services, which is essential for modern cloud-native applications. The skills you've learned are directly applicable to:

Production deployments where containers need to run reliably at scale
DevOps practices that require automated container management and monitoring
Cost optimization through efficient resource utilization in cloud environments
Enterprise applications that demand robust monitoring and logging capabilities
Next Steps:

Explore advanced ECS features like auto-scaling and load balancing
Learn about AWS Fargate for serverless container deployment
Investigate container security best practices with AWS
Study multi-region container deployments for high availability
You now have hands-on experience with the core technologies that power modern containerized applications in AWS, preparing you for the Docker Certified Associate (DCA) certification and real-world container orchestration challenges.
