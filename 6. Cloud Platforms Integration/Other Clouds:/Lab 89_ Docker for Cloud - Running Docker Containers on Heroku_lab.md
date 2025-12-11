Lab 89: Docker for Cloud - Running Docker Containers on Heroku
Lab Objectives
By the end of this lab, students will be able to:

Set up a Heroku account and install the Heroku CLI
Create a Dockerfile for a web application
Build and deploy Docker containers to Heroku
Configure environment variables and application settings in Heroku
Monitor application performance and scale resources using the Heroku dashboard
Understand the fundamentals of cloud-based container deployment
Prerequisites
Before starting this lab, students should have:

Basic understanding of Docker concepts and commands
Familiarity with command-line interface operations
Basic knowledge of web applications and HTTP protocols
Understanding of environment variables and configuration management
A valid email address for creating a Heroku account
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to access your environment - no need to build your own VM or install Docker locally.

Your cloud machine includes:

Ubuntu Linux with Docker Engine
Git version control system
Text editors (nano, vim)
Network connectivity for external services
Task 1: Set up Heroku Account and Install Heroku CLI
Subtask 1.1: Create Heroku Account
Open a web browser in your cloud machine
Navigate to https://signup.heroku.com/
Fill out the registration form with the following information:
First Name: Your first name
Last Name: Your last name
Email Address: Your valid email address
Company: Student (or leave blank)
Role: Student
Country: United States
Development Language: Other
Click Create Free Account
Check your email and verify your account by clicking the verification link
Set up your password when prompted
Subtask 1.2: Install Heroku CLI
Open terminal in your cloud machine
Download and install the Heroku CLI using the following commands:
# Update package list
sudo apt update

# Install curl if not already installed
sudo apt install curl -y

# Download and install Heroku CLI
curl https://cli-assets.heroku.com/install.sh | sh
Verify the installation:
heroku --version
Expected output should show the Heroku CLI version number.

Subtask 1.3: Login to Heroku CLI
Login to your Heroku account through the CLI:
heroku login
Press any key when prompted to open the browser
Click Log In in the browser window that opens
Return to the terminal - you should see a confirmation message
Task 2: Create a Dockerfile for a Web Application
Subtask 2.1: Create Project Directory and Files
Create a new directory for your project:
mkdir heroku-docker-app
cd heroku-docker-app
Create a simple Node.js web application. First, create the package.json file:
nano package.json
Add the following content to package.json:
{
  "name": "heroku-docker-app",
  "version": "1.0.0",
  "description": "Simple web app for Heroku Docker deployment",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "engines": {
    "node": "18.x"
  }
}
Save and exit (Ctrl+X, then Y, then Enter)
Subtask 2.2: Create the Web Application
Create the main application file:
nano server.js
Add the following Node.js code:
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// Middleware to parse JSON
app.use(express.json());

// Basic route
app.get('/', (req, res) => {
    res.json({
        message: 'Hello from Docker on Heroku!',
        timestamp: new Date().toISOString(),
        environment: process.env.NODE_ENV || 'development'
    });
});

// Health check endpoint
app.get('/health', (req, res) => {
    res.status(200).json({
        status: 'healthy',
        uptime: process.uptime(),
        memory: process.memoryUsage()
    });
});

// API endpoint with environment variable
app.get('/api/info', (req, res) => {
    res.json({
        app_name: process.env.APP_NAME || 'Heroku Docker App',
        version: '1.0.0',
        author: process.env.AUTHOR_NAME || 'Student Developer'
    });
});

app.listen(PORT, '0.0.0.0', () => {
    console.log(`Server is running on port ${PORT}`);
    console.log(`Environment: ${process.env.NODE_ENV || 'development'}`);
});
Save and exit the file
Subtask 2.3: Create the Dockerfile
Create a Dockerfile in the project directory:
nano Dockerfile
Add the following Docker configuration:
# Use official Node.js runtime as base image
FROM node:18-alpine

# Set working directory in container
WORKDIR /usr/src/app

# Copy package.json and package-lock.json (if available)
COPY package*.json ./

# Install application dependencies
RUN npm install --only=production

# Copy application source code
COPY . .

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001

# Change ownership of app directory
RUN chown -R nodejs:nodejs /usr/src/app
USER nodejs

# Expose port (Heroku will assign the actual port)
EXPOSE 3000

# Define environment variable
ENV NODE_ENV=production

# Command to run the application
CMD ["npm", "start"]
Save and exit the file
Subtask 2.4: Create Additional Configuration Files
Create a .dockerignore file to exclude unnecessary files:
nano .dockerignore
Add the following content:
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.nyc_output
coverage
.nyc_output
Save and exit

Create a heroku.yml file for Heroku container deployment:

nano heroku.yml
Add the following configuration:
build:
  docker:
    web: Dockerfile
run:
  web: npm start
Save and exit
Task 3: Push Docker Image to Heroku and Deploy
Subtask 3.1: Initialize Git Repository
Initialize a Git repository in your project directory:
git init
Create a .gitignore file:
nano .gitignore
Add the following content:
node_modules/
npm-debug.log*
.env
.DS_Store
*.log
Save and exit

Add all files to Git:

git add .
git commit -m "Initial commit: Docker web app for Heroku"
Subtask 3.2: Create Heroku Application
Create a new Heroku application:
heroku create your-docker-app-name-$(date +%s)
Note: Replace with a unique name or let Heroku generate one automatically. The $(date +%s) adds a timestamp to ensure uniqueness.

Set the stack to container for Docker deployment:
heroku stack:set container
Subtask 3.3: Deploy to Heroku
Add the Heroku remote repository:
heroku git:remote -a $(heroku apps --json | grep -o '"name":"[^"]*' | head -1 | cut -d'"' -f4)
Deploy your application to Heroku:
git push heroku main
Note: If you're using an older Git version, you might need to use master instead of main.

Wait for the build process to complete. You should see output similar to:
=== Building web (Dockerfile)
Sending build context to Docker daemon
Step 1/12 : FROM node:18-alpine
...
=== Pushing web (Dockerfile)
...
=== Releasing...
=== Deployed to Heroku
Subtask 3.4: Verify Deployment
Open your application in the browser:
heroku open
Alternatively, get the application URL:
heroku apps:info
Test the application endpoints using curl:
# Get the app URL
APP_URL=$(heroku apps:info --json | grep -o '"web_url":"[^"]*' | cut -d'"' -f4)

# Test main endpoint
curl $APP_URL

# Test health endpoint
curl ${APP_URL}health

# Test API endpoint
curl ${APP_URL}api/info
Task 4: Set up Environment Variables and Configurations
Subtask 4.1: Configure Environment Variables
Set environment variables for your application:
# Set application name
heroku config:set APP_NAME="My Heroku Docker App"

# Set author name
heroku config:set AUTHOR_NAME="Student Developer"

# Set Node environment
heroku config:set NODE_ENV="production"

# Set custom configuration
heroku config:set DEBUG_MODE="false"
View all configured environment variables:
heroku config
View specific environment variable:
heroku config:get APP_NAME
Subtask 4.2: Update Application to Use New Variables
Update your server.js file to use the new environment variables:
nano server.js
Add a new endpoint that displays all environment information:
// Add this new endpoint before the app.listen() line
app.get('/api/env', (req, res) => {
    res.json({
        app_name: process.env.APP_NAME || 'Default App Name',
        author: process.env.AUTHOR_NAME || 'Unknown Author',
        node_env: process.env.NODE_ENV || 'development',
        debug_mode: process.env.DEBUG_MODE || 'true',
        port: process.env.PORT || 3000,
        heroku_app_name: process.env.HEROKU_APP_NAME || 'Not set'
    });
});
Save and exit
Subtask 4.3: Redeploy with New Configuration
Commit your changes:
git add .
git commit -m "Add environment variables endpoint"
Deploy the updated application:
git push heroku main
Test the new endpoint:
APP_URL=$(heroku apps:info --json | grep -o '"web_url":"[^"]*' | cut -d'"' -f4)
curl ${APP_URL}api/env
Task 5: Monitor and Scale Application Using Heroku Dashboard
Subtask 5.1: Access Heroku Dashboard
Open the Heroku dashboard in your browser:
heroku dashboard
Alternatively, open your specific app dashboard:
heroku dashboard --app $(heroku apps --json | grep -o '"name":"[^"]*' | head -1 | cut -d'"' -f4)
Subtask 5.2: Monitor Application Metrics
View application logs in real-time:
heroku logs --tail
View recent logs:
heroku logs --num=100
Check application status:
heroku ps
View application information:
heroku apps:info
Subtask 5.3: Performance Testing and Monitoring
Generate some traffic to your application for monitoring:
# Get app URL
APP_URL=$(heroku apps:info --json | grep -o '"web_url":"[^"]*' | cut -d'"' -f4)

# Create a simple load test script
cat > load_test.sh << 'EOF'
#!/bin/bash
APP_URL=$1
echo "Starting load test for $APP_URL"
for i in {1..50}; do
    curl -s $APP_URL > /dev/null &
    curl -s ${APP_URL}health > /dev/null &
    curl -s ${APP_URL}api/info > /dev/null &
    curl -s ${APP_URL}api/env > /dev/null &
    if [ $((i % 10)) -eq 0 ]; then
        echo "Completed $i requests"
    fi
done
wait
echo "Load test completed"
EOF

chmod +x load_test.sh
Run the load test:
./load_test.sh $APP_URL
Monitor the logs during the test:
heroku logs --tail
Subtask 5.4: Scale the Application
Check current dyno formation:
heroku ps
Scale up the application (note: this may incur charges on Heroku):
# Scale to 2 dynos (this will move you to paid tier)
# heroku ps:scale web=2

# For this lab, we'll demonstrate the command without executing
echo "To scale to 2 dynos, you would run: heroku ps:scale web=2"
echo "This would incur charges, so we're not executing it in this lab"
View dyno types and pricing:
heroku ps:type
Subtask 5.5: Application Health Monitoring
Set up a simple monitoring script:
cat > monitor.sh << 'EOF'
#!/bin/bash
APP_URL=$1
echo "Monitoring application health..."
while true; do
    RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" ${APP_URL}health)
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    if [ "$RESPONSE" = "200" ]; then
        echo "[$TIMESTAMP] ✓ Application is healthy (HTTP $RESPONSE)"
    else
        echo "[$TIMESTAMP] ✗ Application issue detected (HTTP $RESPONSE)"
    fi
    sleep 30
done
EOF

chmod +x monitor.sh
Run the monitor for a few cycles (press Ctrl+C to stop):
timeout 120 ./monitor.sh $APP_URL
Troubleshooting Common Issues
Issue 1: Build Failures
If your Docker build fails:

Check the build logs:
heroku logs --tail
Verify your Dockerfile syntax:
docker build -t test-build .
Issue 2: Application Won't Start
If your application doesn't start:

Check that your application listens on the correct port:
const PORT = process.env.PORT || 3000;
app.listen(PORT, '0.0.0.0', callback);
Verify your package.json start script is correct
Issue 3: Environment Variables Not Working
If environment variables aren't being read:

Verify they're set:
heroku config
Check your application code references them correctly:
process.env.VARIABLE_NAME
Lab Conclusion
Congratulations! You have successfully completed Lab 89: Docker for Cloud - Running Docker Containers on Heroku.

What You Accomplished
In this lab, you have:

Set up Heroku Infrastructure: Created a Heroku account and installed the CLI tools necessary for cloud deployment
Containerized a Web Application: Built a complete Docker container with a Node.js web application, including proper security practices and optimization
Deployed to Production: Successfully pushed and deployed your Docker container to Heroku's cloud platform
Configured Cloud Environment: Set up environment variables and application configurations for production deployment
Implemented Monitoring: Used Heroku's dashboard and CLI tools to monitor application performance and health
Why This Matters
This lab demonstrates critical skills for modern cloud development:

Container Orchestration: Understanding how to deploy containers in cloud environments is essential for scalable applications
DevOps Practices: You've experienced the complete deployment pipeline from development to production
Cloud Platform Management: Heroku represents Platform-as-a-Service (PaaS) solutions that abstract infrastructure complexity
Production Monitoring: Real-world applications require continuous monitoring and the ability to scale based on demand
Environment Management: Proper configuration management is crucial for maintaining applications across different environments
Next Steps
To continue building on this knowledge:

Explore Heroku add-ons for databases, monitoring, and logging
Learn about CI/CD pipelines for automated deployments
Study container orchestration platforms like Kubernetes
Practice with other cloud providers like AWS, Google Cloud, or Azure
Implement more complex applications with databases and external services
This hands-on experience with Docker and Heroku provides a solid foundation for cloud-native application development and prepares you for the Docker Certified Associate (DCA) certification and real-world cloud deployment scenarios.
