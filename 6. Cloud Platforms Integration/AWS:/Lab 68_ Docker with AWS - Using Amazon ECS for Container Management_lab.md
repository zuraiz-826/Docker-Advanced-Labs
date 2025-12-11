Lab 68: Docker with AWS - Using Amazon ECS for Container Management
Lab Objectives
By the end of this lab, you will be able to:

• Set up and configure an Amazon ECS cluster for container orchestration • Create and manage Docker images using Amazon Elastic Container Registry (ECR) • Deploy containerized applications to ECS using task definitions and services • Monitor ECS services and containers using AWS CloudWatch • Implement auto-scaling policies for ECS services to handle varying workloads • Understand the fundamentals of container management in AWS cloud environment

Prerequisites
Before starting this lab, you should have:

• Basic understanding of Docker containers and containerization concepts • Familiarity with AWS console navigation and basic AWS services • Knowledge of command-line interface operations • Understanding of web applications and HTTP protocols • Basic knowledge of JSON configuration files

Required Tools: • AWS CLI (pre-installed on your cloud machine) • Docker (pre-installed on your cloud machine) • Git (pre-installed on your cloud machine) • Text editor (nano/vim available on your cloud machine)

Lab Environment Setup
Good News! Al Nafi provides you with a ready-to-use Linux-based cloud machine with all necessary tools pre-installed. Simply click Start Lab to begin - no need to build your own virtual machine or install software.

Your cloud machine includes: • Ubuntu Linux environment • Docker Engine • AWS CLI • Git and text editors • All required dependencies

Task 1: Set up an ECS Cluster on AWS
Subtask 1.1: Configure AWS CLI
First, we need to configure the AWS CLI with your credentials.

Open your terminal in the cloud machine
Configure AWS CLI with your credentials:
aws configure
When prompted, enter: • AWS Access Key ID: [Your Access Key] • AWS Secret Access Key: [Your Secret Key] • Default region name: us-east-1 • Default output format: json

Verify your configuration:
aws sts get-caller-identity
Subtask 1.2: Create an ECS Cluster
Create a new ECS cluster using the AWS CLI:
aws ecs create-cluster --cluster-name my-docker-cluster
Verify the cluster creation:
aws ecs describe-clusters --clusters my-docker-cluster
List all available clusters:
aws ecs list-clusters
Expected Output: You should see your cluster in "ACTIVE" status with the name "my-docker-cluster".

Subtask 1.3: Create a VPC and Security Group (if needed)
Create a VPC for your ECS cluster:
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=ecs-vpc}]'
Note the VPC ID from the output and create a subnet:
# Replace vpc-xxxxxxxxx with your actual VPC ID
aws ec2 create-subnet --vpc-id vpc-xxxxxxxxx --cidr-block 10.0.1.0/24 --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=ecs-subnet}]'
Create a security group for ECS:
# Replace vpc-xxxxxxxxx with your actual VPC ID
aws ec2 create-security-group --group-name ecs-security-group --description "Security group for ECS" --vpc-id vpc-xxxxxxxxx
Task 2: Push a Docker Image to Amazon ECR
Subtask 2.1: Create an ECR Repository
Create a new ECR repository:
aws ecr create-repository --repository-name my-web-app
Get the login token for ECR:
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $(aws sts get-caller-identity --query Account --output text).dkr.ecr.us-east-1.amazonaws.com
Subtask 2.2: Create a Sample Web Application
Create a directory for your application:
mkdir ~/my-web-app
cd ~/my-web-app
Create a simple HTML file:
cat > index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>My ECS Web App</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; }
        .container { background-color: #f0f0f0; padding: 20px; border-radius: 10px; display: inline-block; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Welcome to My ECS Application!</h1>
        <p>This application is running on Amazon ECS</p>
        <p>Container ID: <span id="hostname"></span></p>
    </div>
    <script>
        document.getElementById('hostname').textContent = window.location.hostname;
    </script>
</body>
</html>
EOF
Create a Dockerfile:
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
EOF
Subtask 2.3: Build and Push Docker Image
Build the Docker image:
docker build -t my-web-app .
Tag the image for ECR:
# Replace ACCOUNT_ID with your actual AWS account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
docker tag my-web-app:latest $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/my-web-app:latest
Push the image to ECR:
docker push $ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/my-web-app:latest
Verify the image in ECR:
aws ecr describe-images --repository-name my-web-app
Task 3: Deploy the Image to ECS using Task Definitions and Services
Subtask 3.1: Create an ECS Task Definition
Create a task definition JSON file:
cat > task-definition.json << 'EOF'
{
    "family": "my-web-app-task",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "256",
    "memory": "512",
    "executionRoleArn": "arn:aws:iam::ACCOUNT_ID:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "name": "my-web-app-container",
            "image": "ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/my-web-app:latest",
            "portMappings": [
                {
                    "containerPort": 80,
                    "protocol": "tcp"
                }
            ],
            "essential": true,
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/my-web-app",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
    ]
}
EOF
Replace ACCOUNT_ID in the task definition:
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
sed -i "s/ACCOUNT_ID/$ACCOUNT_ID/g" task-definition.json
Create the ECS task execution role (if it doesn't exist):
cat > trust-policy.json << 'EOF'
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

aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
Create CloudWatch log group:
aws logs create-log-group --log-group-name /ecs/my-web-app
Register the task definition:
aws ecs register-task-definition --cli-input-json file://task-definition.json
Subtask 3.2: Create an ECS Service
Get your default VPC and subnet information:
# Get default VPC ID
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=is-default,Values=true" --query 'Vpcs[0].VpcId' --output text)
echo "Default VPC ID: $VPC_ID"

# Get subnet ID
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[0].SubnetId' --output text)
echo "Subnet ID: $SUBNET_ID"

# Get default security group
SECURITY_GROUP_ID=$(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=default" --query 'SecurityGroups[0].GroupId' --output text)
echo "Security Group ID: $SECURITY_GROUP_ID"
Update security group to allow HTTP traffic:
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
Create the ECS service:
aws ecs create-service \
    --cluster my-docker-cluster \
    --service-name my-web-app-service \
    --task-definition my-web-app-task \
    --desired-count 2 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SECURITY_GROUP_ID],assignPublicIp=ENABLED}"
Verify the service is running:
aws ecs describe-services --cluster my-docker-cluster --services my-web-app-service
List running tasks:
aws ecs list-tasks --cluster my-docker-cluster --service-name my-web-app-service
Task 4: Monitor ECS Services with AWS CloudWatch
Subtask 4.1: View CloudWatch Logs
Check if logs are being generated:
aws logs describe-log-streams --log-group-name /ecs/my-web-app
View recent log events:
aws logs filter-log-events --log-group-name /ecs/my-web-app --start-time $(date -d '10 minutes ago' +%s)000
Subtask 4.2: Set up CloudWatch Metrics
View ECS cluster metrics:
aws cloudwatch get-metric-statistics \
    --namespace AWS/ECS \
    --metric-name CPUUtilization \
    --dimensions Name=ClusterName,Value=my-docker-cluster \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average
View service-level metrics:
aws cloudwatch get-metric-statistics \
    --namespace AWS/ECS \
    --metric-name CPUUtilization \
    --dimensions Name=ServiceName,Value=my-web-app-service Name=ClusterName,Value=my-docker-cluster \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average
Subtask 4.3: Create CloudWatch Alarms
Create a CPU utilization alarm:
aws cloudwatch put-metric-alarm \
    --alarm-name "ECS-HighCPU" \
    --alarm-description "Alarm when ECS CPU exceeds 70%" \
    --metric-name CPUUtilization \
    --namespace AWS/ECS \
    --statistic Average \
    --period 300 \
    --threshold 70 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=ServiceName,Value=my-web-app-service Name=ClusterName,Value=my-docker-cluster \
    --evaluation-periods 2
List your alarms:
aws cloudwatch describe-alarms --alarm-names "ECS-HighCPU"
Task 5: Scale ECS Services using Auto-scaling Policies
Subtask 5.1: Create Application Auto Scaling Target
Register the ECS service as a scalable target:
aws application-autoscaling register-scalable-target \
    --service-namespace ecs \
    --resource-id service/my-docker-cluster/my-web-app-service \
    --scalable-dimension ecs:service:DesiredCount \
    --min-capacity 1 \
    --max-capacity 10
Verify the scalable target:
aws application-autoscaling describe-scalable-targets --service-namespace ecs
Subtask 5.2: Create Scaling Policies
Create a scale-up policy:
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --resource-id service/my-docker-cluster/my-web-app-service \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-name my-web-app-scale-up \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
        "TargetValue": 50.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
        },
        "ScaleOutCooldown": 300,
        "ScaleInCooldown": 300
    }'
Create a scale-down policy for memory utilization:
aws application-autoscaling put-scaling-policy \
    --service-namespace ecs \
    --resource-id service/my-docker-cluster/my-web-app-service \
    --scalable-dimension ecs:service:DesiredCount \
    --policy-name my-web-app-memory-scale \
    --policy-type TargetTrackingScaling \
    --target-tracking-scaling-policy-configuration '{
        "TargetValue": 60.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ECSServiceAverageMemoryUtilization"
        },
        "ScaleOutCooldown": 300,
        "ScaleInCooldown": 300
    }'
Subtask 5.3: Test Auto Scaling
Manually scale the service to test:
aws ecs update-service \
    --cluster my-docker-cluster \
    --service my-web-app-service \
    --desired-count 3
Monitor the scaling activity:
aws application-autoscaling describe-scaling-activities --service-namespace ecs
Check the current service status:
aws ecs describe-services --cluster my-docker-cluster --services my-web-app-service --query 'services[0].{DesiredCount:desiredCount,RunningCount:runningCount,PendingCount:pendingCount}'
Subtask 5.4: View Running Tasks and Get Public IPs
Get task ARNs:
TASK_ARNS=$(aws ecs list-tasks --cluster my-docker-cluster --service-name my-web-app-service --query 'taskArns' --output text)
echo "Task ARNs: $TASK_ARNS"
Get public IP addresses of running tasks:
for task_arn in $TASK_ARNS; do
    echo "Getting details for task: $task_arn"
    aws ecs describe-tasks --cluster my-docker-cluster --tasks $task_arn --query 'tasks[0].attachments[0].details[?name==`networkInterfaceId`].value' --output text | while read eni_id; do
        if [ ! -z "$eni_id" ]; then
            public_ip=$(aws ec2 describe-network-interfaces --network-interface-ids $eni_id --query 'NetworkInterfaces[0].Association.PublicIp' --output text)
            echo "Task: $task_arn - Public IP: $public_ip"
            echo "Access your app at: http://$public_ip"
        fi
    done
done
Troubleshooting Tips
Common Issues and Solutions
Issue 1: Task fails to start • Check CloudWatch logs for error messages • Verify ECR image exists and is accessible • Ensure task execution role has proper permissions

Issue 2: Service not accessible • Verify security group allows inbound traffic on port 80 • Check if tasks have public IP addresses assigned • Ensure subnet has internet gateway attached

Issue 3: Auto-scaling not working • Verify CloudWatch metrics are being generated • Check scaling policy configuration • Ensure sufficient time has passed for cooldown periods

Debugging Commands:

# Check service events
aws ecs describe-services --cluster my-docker-cluster --services my-web-app-service --query 'services[0].events'

# Check task definition
aws ecs describe-task-definition --task-definition my-web-app-task

# View detailed task information
aws ecs describe-tasks --cluster my-docker-cluster --tasks TASK_ARN
Lab Cleanup
To avoid unnecessary charges, clean up the resources:

# Delete the service
aws ecs update-service --cluster my-docker-cluster --service my-web-app-service --desired-count 0
aws ecs delete-service --cluster my-docker-cluster --service my-web-app-service

# Delete the cluster
aws ecs delete-cluster --cluster my-docker-cluster

# Delete ECR repository
aws ecr delete-repository --repository-name my-web-app --force

# Delete CloudWatch log group
aws logs delete-log-group --log-group-name /ecs/my-web-app

# Delete CloudWatch alarm
aws cloudwatch delete-alarms --alarm-names "ECS-HighCPU"

# Deregister scalable target
aws application-autoscaling deregister-scalable-target --service-namespace ecs --resource-id service/my-docker-cluster/my-web-app-service --scalable-dimension ecs:service:DesiredCount
Conclusion
Congratulations! You have successfully completed Lab 68 on Docker with AWS ECS. In this comprehensive lab, you have accomplished the following:

Key Achievements:

• ECS Cluster Management: You created and configured an Amazon ECS cluster, learning how to set up container orchestration infrastructure in AWS • Container Registry Operations: You successfully pushed a Docker image to Amazon ECR, understanding how to manage container images in AWS • Service Deployment: You deployed containerized applications using ECS task definitions and services, gaining hands-on experience with container deployment strategies • Monitoring and Observability: You implemented CloudWatch monitoring for your ECS services, learning how to track performance metrics and set up alerts • Auto-scaling Implementation: You configured auto-scaling policies for your ECS services, understanding how to handle varying workloads automatically

Why This Matters:

This lab provided you with practical experience in modern container orchestration using AWS managed services. ECS is widely used in enterprise environments for running containerized applications at scale. The skills you've learned are directly applicable to:

• Production Deployments: Managing real-world containerized applications in cloud environments • DevOps Practices: Implementing continuous deployment and infrastructure as code • Cost Optimization: Using auto-scaling to optimize resource usage and costs • Monitoring and Reliability: Ensuring application health and performance through proper monitoring

Next Steps:

Consider exploring advanced ECS features such as: • ECS with Application Load Balancers for better traffic distribution • ECS Service Discovery for microservices communication • ECS with AWS CodePipeline for CI/CD automation • EKS (Elastic Kubernetes Service) as an alternative container orchestration platform

You now have a solid foundation in AWS container management that will serve you well in your cloud and DevOps journey!
