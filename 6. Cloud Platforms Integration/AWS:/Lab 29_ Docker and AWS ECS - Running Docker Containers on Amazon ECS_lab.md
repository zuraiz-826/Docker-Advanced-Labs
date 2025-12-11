Lab 29: Docker and AWS ECS - Running Docker Containers on Amazon ECS
Objectives
By the end of this lab, you will be able to:

Create and configure an Amazon ECS cluster
Define and deploy task definitions for containerized applications
Run Docker containers in ECS using the AWS Management Console
Configure an Application Load Balancer for ECS services
Scale ECS services and manage container tasks effectively
Monitor ECS services using Amazon CloudWatch
Understand the fundamentals of container orchestration in AWS
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker containers and containerization concepts
Familiarity with AWS services and the AWS Management Console
Knowledge of basic networking concepts (load balancers, security groups)
Understanding of JSON format for configuration files
Basic command-line interface experience
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment. No need to build or configure your own virtual machine - everything is ready to use!

Your lab environment includes:

AWS CLI pre-installed and configured
Docker installed and running
All necessary permissions configured
Sample application files ready for deployment
Task 1: Create an ECS Cluster and Configure Task Definitions
Subtask 1.1: Create an ECS Cluster
Access the AWS Management Console

Open your web browser and navigate to the AWS Management Console
Search for "ECS" in the services search bar
Click on "Elastic Container Service"
Create a New Cluster

Click on "Clusters" in the left navigation panel
Click the "Create Cluster" button
Choose "EC2 Linux + Networking" as the cluster template
Click "Next step"
Configure Cluster Settings

Cluster name: my-ecs-cluster
EC2 instance type: t3.micro (free tier eligible)
Number of instances: 2
Key pair: Select an existing key pair or create a new one
VPC: Use the default VPC
Subnets: Select at least two subnets in different availability zones
Security group: Create a new security group or use the default
Click "Create"
Verify Cluster Creation

Wait for the cluster creation to complete (this may take 5-10 minutes)
Verify that your cluster shows as "ACTIVE" status
Note the cluster ARN for future reference
Subtask 1.2: Create a Task Definition
Navigate to Task Definitions

In the ECS console, click on "Task Definitions" in the left panel
Click "Create new Task Definition"
Select Launch Type

Choose "EC2" as the launch type compatibility
Click "Next step"
Configure Task Definition

Task Definition Name: nginx-task-definition
Task Role: Leave blank for now
Network Mode: bridge
Task execution role: ecsTaskExecutionRole
Task memory (MiB): 512
Task CPU (unit): 256
Add Container Definition

Click "Add container"
Container name: nginx-container
Image: nginx:latest
Memory Limits (MiB): Soft limit 256
Port mappings:
Host port: 0 (dynamic port mapping)
Container port: 80
Protocol: tcp
Click "Add"
Create Task Definition

Review all settings
Click "Create"
Verify the task definition is created successfully
Task 2: Run a Docker Container in ECS Using AWS Management Console
Subtask 2.1: Create an ECS Service
Navigate to Your Cluster

Go back to "Clusters" and click on my-ecs-cluster
Click on the "Services" tab
Click "Create"
Configure Service

Launch type: EC2
Task Definition: Select nginx-task-definition:1
Platform version: LATEST
Cluster: my-ecs-cluster
Service name: nginx-service
Service type: REPLICA
Number of tasks: 2
Configure Deployments

Minimum healthy percent: 50
Maximum percent: 200
Leave other settings as default
Click "Next step"
Configure Network

Load balancer type: Skip for now (we'll add this in Task 3)
Service discovery: Uncheck "Enable service discovery"
Click "Next step"
Set Auto Scaling

Service Auto Scaling: Do not adjust the service's desired count
Click "Next step"
Review and Create

Review all configurations
Click "Create Service"
Click "View Service" to monitor the deployment
Subtask 2.2: Verify Container Deployment
Check Service Status

Monitor the service until it shows "RUNNING" status
Verify that 2 tasks are running successfully
View Running Tasks

Click on the "Tasks" tab within your service
Click on one of the running tasks to view details
Note the public IP address and dynamic port assigned
Test the Application

Copy the public IP address from the task details
Note the port number from the port mapping
Open a new browser tab and navigate to http://[PUBLIC_IP]:[PORT]
You should see the default Nginx welcome page
Task 3: Configure a Load Balancer for Your ECS Service
Subtask 3.1: Create an Application Load Balancer
Navigate to EC2 Console

Open a new tab and go to the EC2 console
Click on "Load Balancers" in the left navigation panel
Click "Create Load Balancer"
Select Load Balancer Type

Choose "Application Load Balancer"
Click "Create"
Configure Load Balancer

Name: ecs-nginx-alb
Scheme: Internet-facing
IP address type: IPv4
VPC: Select the same VPC as your ECS cluster
Availability Zones: Select at least 2 AZs with public subnets
Security groups: Create a new security group or select existing one
Allow HTTP (port 80) from anywhere (0.0.0.0/0)
Configure Target Group

Target group name: ecs-nginx-targets
Target type: Instance
Protocol: HTTP
Port: 80
VPC: Same as load balancer
Health check path: /
Click "Next"
Register Targets

Don't register targets manually (ECS will handle this)
Click "Next"
Review and click "Create"
Subtask 3.2: Update ECS Service with Load Balancer
Update the Service

Go back to your ECS cluster
Select your nginx-service
Click "Update"
Configure Load Balancer Integration

Load balancer type: Application Load Balancer
Load balancer name: Select ecs-nginx-alb
Container to load balance: nginx-container:80:0
Target group name: ecs-nginx-targets
Target group protocol: HTTP
Target group port: 80
Path pattern: /*
Health check grace period: 30 seconds
Complete the Update

Click "Next step" through the remaining screens
Click "Update Service"
Wait for the service to stabilize
Subtask 3.3: Test Load Balancer
Get Load Balancer DNS Name

Go to EC2 console > Load Balancers
Select your ecs-nginx-alb
Copy the DNS name from the description
Test the Application

Open a browser and navigate to the load balancer DNS name
You should see the Nginx welcome page
Refresh the page multiple times to verify load balancing
Task 4: Scale the ECS Service and Manage Container Tasks
Subtask 4.1: Manual Scaling
Scale Up the Service

Navigate to your ECS service
Click "Update"
Change "Number of tasks" from 2 to 4
Click through the update process
Monitor the deployment of new tasks
Verify Scaling

Check that 4 tasks are now running
Verify all tasks are healthy in the target group
Test the load balancer to ensure traffic distribution
Subtask 4.2: Configure Auto Scaling
Set Up Service Auto Scaling

In your ECS service, click "Update"
Navigate to "Set Auto Scaling"
Service Auto Scaling: Adjust the service's desired count
Minimum number of tasks: 2
Desired number of tasks: 4
Maximum number of tasks: 8
Create Scaling Policies

Scale-out policy:

Policy name: scale-out-policy
Metric type: ECSServiceAverageCPUUtilization
Target value: 70
Scale-out cooldown: 300 seconds
Scale-in policy:

Policy name: scale-in-policy
Metric type: ECSServiceAverageCPUUtilization
Target value: 70
Scale-in cooldown: 300 seconds
Apply Auto Scaling Configuration

Review the settings
Click "Update Service"
Subtask 4.3: Test Auto Scaling
Generate Load (Optional)
Use a load testing tool or script to generate traffic
Monitor CPU utilization in CloudWatch
Observe auto scaling behavior
# Simple load generation script (run from your lab machine)
for i in {1..1000}; do
  curl -s http://[YOUR-LOAD-BALANCER-DNS] > /dev/null
  sleep 0.1
done
Task 5: Monitor ECS Services with Amazon CloudWatch
Subtask 5.1: Access CloudWatch Metrics
Navigate to CloudWatch

Open the CloudWatch console
Click on "Metrics" in the left navigation panel
Click "All metrics"
Find ECS Metrics

Click on "ECS" namespace
Select "ClusterName, ServiceName"
Choose your cluster and service metrics
Key Metrics to Monitor

CPUUtilization: Monitor CPU usage across tasks
MemoryUtilization: Track memory consumption
RunningTaskCount: Number of running tasks
PendingTaskCount: Tasks waiting to be scheduled
Subtask 5.2: Create CloudWatch Dashboard
Create a New Dashboard

In CloudWatch, click "Dashboards"
Click "Create dashboard"
Dashboard name: ECS-Monitoring-Dashboard
Add Widgets

Click "Add widget"
Choose "Line" chart
Select ECS metrics for your service:
CPU Utilization
Memory Utilization
Running Task Count
Configure the time range and refresh interval
Customize Dashboard

Add multiple widgets for different metrics
Arrange widgets for optimal viewing
Save the dashboard
Subtask 5.3: Set Up CloudWatch Alarms
Create CPU Utilization Alarm

Go to CloudWatch > Alarms
Click "Create alarm"
Select ECS CPU utilization metric
Threshold: Greater than 80%
Period: 5 minutes
Datapoints to alarm: 2 out of 3
Alarm name: ECS-High-CPU-Utilization
Create Task Count Alarm

Create another alarm for running task count
Threshold: Less than 2
Alarm name: ECS-Low-Task-Count
Configure Notifications (Optional)

Set up SNS topics for alarm notifications
Configure email or SMS alerts
Subtask 5.4: View Container Insights
Enable Container Insights

In the ECS console, go to your cluster
Click on "Metrics" tab
Enable Container Insights if not already enabled
Explore Container Insights

View cluster-level metrics
Drill down to service and task-level metrics
Analyze performance patterns and trends
Troubleshooting Tips
Common Issues and Solutions
Tasks Not Starting

Check task definition for correct image name
Verify sufficient resources (CPU/memory) on cluster instances
Review CloudWatch logs for error messages
Load Balancer Health Checks Failing

Ensure security groups allow traffic on the correct ports
Verify the health check path is accessible
Check that containers are listening on the expected port
Auto Scaling Not Working

Verify CloudWatch metrics are being published
Check scaling policies and thresholds
Ensure proper IAM permissions for auto scaling
Service Update Stuck

Check for resource constraints
Verify task definition is valid
Review deployment configuration settings
Useful Commands for Debugging
# Check ECS agent logs on cluster instances
sudo docker logs ecs-agent

# View running containers on an instance
docker ps

# Check container logs
docker logs [container-id]

# Describe ECS service
aws ecs describe-services --cluster my-ecs-cluster --services nginx-service

# List running tasks
aws ecs list-tasks --cluster my-ecs-cluster --service-name nginx-service
Cleanup Instructions
To avoid ongoing charges, clean up the resources created in this lab:

Delete ECS Service

Scale service down to 0 tasks
Delete the service
Delete Load Balancer

Delete the Application Load Balancer
Delete the target group
Delete ECS Cluster

Ensure no services are running
Delete the cluster (this will terminate EC2 instances)
Delete CloudWatch Resources

Delete custom dashboards
Delete alarms
Clean up log groups if needed
Conclusion
Congratulations! You have successfully completed Lab 29 on Docker and AWS ECS. In this comprehensive lab, you have:

Created and configured an Amazon ECS cluster with EC2 instances
Defined and deployed task definitions for containerized applications
Successfully ran Docker containers in ECS using the AWS Management Console
Configured an Application Load Balancer to distribute traffic across container instances
Implemented both manual and automatic scaling for your ECS services
Set up comprehensive monitoring using Amazon CloudWatch with custom dashboards and alarms
Key Takeaways
This lab demonstrates the power of container orchestration in the cloud. Amazon ECS provides a managed service that eliminates the complexity of running your own container orchestration infrastructure. You've learned how to:

Leverage AWS managed services for container deployment
Implement high availability through load balancing and multi-AZ deployment
Use auto scaling to handle varying workloads efficiently
Monitor and troubleshoot containerized applications in production
Real-World Applications
The skills you've developed in this lab are directly applicable to:

Microservices Architecture: Deploy and manage multiple containerized services
DevOps Practices: Implement continuous deployment pipelines with ECS
Cost Optimization: Use auto scaling to optimize resource usage and costs
High Availability: Design resilient applications that can handle failures gracefully
Next Steps
To further enhance your container orchestration skills, consider:

Exploring Amazon EKS (Elastic Kubernetes Service) for Kubernetes-based orchestration
Implementing CI/CD pipelines with AWS CodePipeline and ECS
Learning about service mesh technologies like AWS App Mesh
Studying advanced ECS features like Fargate for serverless containers
This foundational knowledge of ECS will serve you well as you continue your journey in cloud computing and containerization technologies. The principles and practices you've learned here are essential for modern application deployment and management in the cloud.
