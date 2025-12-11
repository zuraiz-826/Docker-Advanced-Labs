Lab 21: Docker and CI/CD - Automating Builds with Jenkins
Objectives
By the end of this lab, you will be able to:

Install and configure Jenkins on a Linux machine
Set up Docker integration with Jenkins for automated builds
Create and configure Jenkins pipelines for Docker image building and testing
Integrate SonarQube for static code analysis in CI/CD pipelines
Implement automated pipeline triggers based on code changes
Monitor and analyze build results in Jenkins dashboard
Prerequisites
Before starting this lab, you should have:

Basic understanding of Linux command line operations
Familiarity with Docker concepts and basic commands
Understanding of version control systems (Git)
Basic knowledge of software development lifecycle
Familiarity with YAML syntax for pipeline configuration
Lab Environment Setup
Ready-to-Use Cloud Machines: Al Nafi provides pre-configured Linux-based cloud machines for this lab. Simply click Start Lab to access your environment - no need to build your own VM or install an operating system.

Your lab environment includes:

Ubuntu 20.04 LTS with Docker pre-installed
Internet connectivity for downloading packages
Sufficient resources for running Jenkins and SonarQube
Git client pre-installed
Task 1: Install Jenkins on Linux Machine
Subtask 1.1: Update System and Install Java
Jenkins requires Java to run. Let's start by updating the system and installing OpenJDK.

# Update package repository
sudo apt update

# Install OpenJDK 11 (required for Jenkins)
sudo apt install -y openjdk-11-jdk

# Verify Java installation
java -version
Subtask 1.2: Add Jenkins Repository and Install Jenkins
# Add Jenkins repository key
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null

# Add Jenkins repository to sources list
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null

# Update package repository
sudo apt update

# Install Jenkins
sudo apt install -y jenkins
Subtask 1.3: Start and Enable Jenkins Service
# Start Jenkins service
sudo systemctl start jenkins

# Enable Jenkins to start on boot
sudo systemctl enable jenkins

# Check Jenkins service status
sudo systemctl status jenkins
Subtask 1.4: Configure Jenkins Initial Setup
# Get the initial admin password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
Note: Copy this password as you'll need it for the web setup.

Open your web browser and navigate to http://localhost:8080 (or your server's IP address with port 8080).

Enter the initial admin password you copied
Click Install suggested plugins
Create your first admin user with the following details:
Username: admin
Password: admin123
Full name: Jenkins Administrator
Email: admin@example.com
Keep the default Jenkins URL
Click Start using Jenkins
Task 2: Configure Jenkins to Use Docker for Building Images
Subtask 2.1: Install Docker Plugin in Jenkins
In Jenkins dashboard, go to Manage Jenkins → Manage Plugins
Click on Available tab
Search for Docker and install the following plugins:
Docker Pipeline
Docker plugin
Docker Commons Plugin
Click Install without restart
Check Restart Jenkins when installation is complete
Subtask 2.2: Add Jenkins User to Docker Group
# Add jenkins user to docker group
sudo usermod -aG docker jenkins

# Restart Jenkins service to apply changes
sudo systemctl restart jenkins

# Verify docker access for jenkins user
sudo -u jenkins docker ps
Subtask 2.3: Configure Docker in Jenkins Global Tool Configuration
Go to Manage Jenkins → Global Tool Configuration
Scroll down to Docker section
Click Add Docker
Configure as follows:
Name: docker
Installation root: /usr/bin/docker
Check Install automatically and select Download from docker.com
Click Save
Task 3: Create a Jenkins Pipeline for Docker Image Building and Testing
Subtask 3.1: Create Sample Application
First, let's create a simple Node.js application for our pipeline.

# Create project directory
mkdir -p /home/ubuntu/sample-app
cd /home/ubuntu/sample-app

# Create package.json
cat > package.json << 'EOF'
{
  "name": "sample-node-app",
  "version": "1.0.0",
  "description": "Sample Node.js application for Jenkins CI/CD",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo 'Running tests...' && exit 0"
  },
  "dependencies": {
    "express": "^4.18.0"
  }
}
EOF

# Create main application file
cat > app.js << 'EOF'
const express = require('express');
const app = express();
const port = 3000;

app.get('/', (req, res) => {
  res.json({
    message: 'Hello from Docker CI/CD Pipeline!',
    version: '1.0.0',
    timestamp: new Date().toISOString()
  });
});

app.get('/health', (req, res) => {
  res.status(200).json({ status: 'healthy' });
});

app.listen(port, () => {
  console.log(`App running on port ${port}`);
});

module.exports = app;
EOF

# Create Dockerfile
cat > Dockerfile << 'EOF'
FROM node:16-alpine

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .

EXPOSE 3000

CMD ["npm", "start"]
EOF

# Create .dockerignore
cat > .dockerignore << 'EOF'
node_modules
npm-debug.log
.git
.gitignore
README.md
Dockerfile
.dockerignore
EOF
Subtask 3.2: Initialize Git Repository
# Initialize git repository
git init

# Create .gitignore
cat > .gitignore << 'EOF'
node_modules/
npm-debug.log*
.env
EOF

# Add files to git
git add .
git commit -m "Initial commit: Sample Node.js application"
Subtask 3.3: Create Jenkins Pipeline Job
In Jenkins dashboard, click New Item
Enter item name: docker-pipeline-demo
Select Pipeline and click OK
In the configuration page, scroll down to Pipeline section
Select Pipeline script and enter the following:
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'sample-node-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    // For this demo, we'll copy files from local directory
                    sh '''
                        rm -rf workspace-temp
                        mkdir -p workspace-temp
                        cp -r /home/ubuntu/sample-app/* workspace-temp/
                        ls -la workspace-temp/
                    '''
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    dir('workspace-temp') {
                        sh '''
                            echo "Building Docker image..."
                            docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                            docker build -t ${DOCKER_IMAGE}:latest .
                        '''
                    }
                }
            }
        }
        
        stage('Test Docker Image') {
            steps {
                script {
                    sh '''
                        echo "Testing Docker image..."
                        # Run container in detached mode
                        docker run -d --name test-container-${BUILD_NUMBER} -p 3001:3000 ${DOCKER_IMAGE}:${DOCKER_TAG}
                        
                        # Wait for container to start
                        sleep 10
                        
                        # Test health endpoint
                        curl -f http://localhost:3001/health || exit 1
                        
                        # Test main endpoint
                        curl -f http://localhost:3001/ || exit 1
                        
                        echo "Tests passed!"
                    '''
                }
            }
            post {
                always {
                    sh '''
                        # Clean up test container
                        docker stop test-container-${BUILD_NUMBER} || true
                        docker rm test-container-${BUILD_NUMBER} || true
                    '''
                }
            }
        }
        
        stage('Push to Registry') {
            steps {
                script {
                    sh '''
                        echo "Image built successfully: ${DOCKER_IMAGE}:${DOCKER_TAG}"
                        docker images | grep ${DOCKER_IMAGE}
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                # Clean up old images (keep last 5 builds)
                docker images ${DOCKER_IMAGE} --format "table {{.Repository}}:{{.Tag}}" | tail -n +2 | head -n -5 | xargs -r docker rmi || true
            '''
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
Click Save
Subtask 3.4: Run the Pipeline
Click Build Now to trigger the pipeline
Monitor the build progress in the Build History
Click on the build number to view detailed logs
Verify that all stages complete successfully
Task 4: Integrate SonarQube for Static Code Analysis
Subtask 4.1: Install and Configure SonarQube
# Create SonarQube directory
sudo mkdir -p /opt/sonarqube
cd /opt/sonarqube

# Download SonarQube Community Edition
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-9.9.0.65466.zip

# Install unzip if not available
sudo apt install -y unzip

# Extract SonarQube
sudo unzip sonarqube-9.9.0.65466.zip
sudo mv sonarqube-9.9.0.65466 sonarqube

# Create sonarqube user
sudo useradd -r -s /bin/false sonarqube

# Set ownership
sudo chown -R sonarqube:sonarqube /opt/sonarqube

# Configure SonarQube service
sudo tee /etc/systemd/system/sonarqube.service > /dev/null << 'EOF'
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonarqube
Group=sonarqube
Restart=always
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
EOF

# Start SonarQube service
sudo systemctl daemon-reload
sudo systemctl start sonarqube
sudo systemctl enable sonarqube

# Check service status
sudo systemctl status sonarqube
Subtask 4.2: Configure SonarQube Initial Setup
Wait for SonarQube to start (it may take a few minutes), then:

Open browser and navigate to http://localhost:9000
Login with default credentials:
Username: admin
Password: admin
Change the password when prompted to: admin123
Click Create new project
Choose Manually
Configure project:
Project key: sample-node-app
Display name: Sample Node App
Click Set Up
Choose Use global setting for token
Generate token and copy it (save it as you'll need it later)
Subtask 4.3: Install SonarQube Plugin in Jenkins
Go to Manage Jenkins → Manage Plugins
Search for SonarQube Scanner plugin
Install the plugin and restart Jenkins
Subtask 4.4: Configure SonarQube in Jenkins
Go to Manage Jenkins → Configure System
Scroll to SonarQube servers section
Click Add SonarQube
Configure:
Name: SonarQube
Server URL: http://localhost:9000
Server authentication token: Click Add → Jenkins
Kind: Secret text
Secret: (paste the SonarQube token you generated)
ID: sonarqube-token
Description: SonarQube Authentication Token
Click Save
Subtask 4.5: Configure SonarQube Scanner Tool
Go to Manage Jenkins → Global Tool Configuration
Scroll to SonarQube Scanner section
Click Add SonarQube Scanner
Configure:
Name: SonarQube Scanner
Check Install automatically
Version: Select latest version
Click Save
Subtask 4.6: Create SonarQube Configuration File
# Create sonar-project.properties in the sample app directory
cd /home/ubuntu/sample-app

cat > sonar-project.properties << 'EOF'
sonar.projectKey=sample-node-app
sonar.projectName=Sample Node App
sonar.projectVersion=1.0
sonar.sources=.
sonar.exclusions=node_modules/**,coverage/**
sonar.javascript.lcov.reportPaths=coverage/lcov.info
EOF

# Add to git
git add sonar-project.properties
git commit -m "Add SonarQube configuration"
Subtask 4.7: Update Jenkins Pipeline with SonarQube Integration
Update your Jenkins pipeline with the following enhanced version:

pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'sample-node-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
        SONARQUBE_SERVER = 'SonarQube'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    sh '''
                        rm -rf workspace-temp
                        mkdir -p workspace-temp
                        cp -r /home/ubuntu/sample-app/* workspace-temp/
                        ls -la workspace-temp/
                    '''
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    dir('workspace-temp') {
                        withSonarQubeEnv('SonarQube') {
                            sh '''
                                sonar-scanner \
                                -Dsonar.projectKey=sample-node-app \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://localhost:9000 \
                                -Dsonar.exclusions=node_modules/**
                            '''
                        }
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    dir('workspace-temp') {
                        sh '''
                            echo "Building Docker image..."
                            docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                            docker build -t ${DOCKER_IMAGE}:latest .
                        '''
                    }
                }
            }
        }
        
        stage('Test Docker Image') {
            steps {
                script {
                    sh '''
                        echo "Testing Docker image..."
                        docker run -d --name test-container-${BUILD_NUMBER} -p 3001:3000 ${DOCKER_IMAGE}:${DOCKER_TAG}
                        
                        sleep 10
                        
                        curl -f http://localhost:3001/health || exit 1
                        curl -f http://localhost:3001/ || exit 1
                        
                        echo "Tests passed!"
                    '''
                }
            }
            post {
                always {
                    sh '''
                        docker stop test-container-${BUILD_NUMBER} || true
                        docker rm test-container-${BUILD_NUMBER} || true
                    '''
                }
            }
        }
        
        stage('Deploy') {
            steps {
                script {
                    sh '''
                        echo "Deploying application..."
                        # Stop existing container if running
                        docker stop sample-app-prod || true
                        docker rm sample-app-prod || true
                        
                        # Run new container
                        docker run -d --name sample-app-prod -p 3000:3000 ${DOCKER_IMAGE}:${DOCKER_TAG}
                        
                        echo "Application deployed successfully!"
                        echo "Access the application at: http://localhost:3000"
                    '''
                }
            }
        }
    }
    
    post {
        always {
            sh '''
                docker images ${DOCKER_IMAGE} --format "table {{.Repository}}:{{.Tag}}" | tail -n +2 | head -n -5 | xargs -r docker rmi || true
            '''
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
Task 5: Trigger Pipeline on Code Changes and View Build Results
Subtask 5.1: Configure Webhook for Automatic Triggers
Since we're working in a local environment, we'll simulate automatic triggers using Jenkins polling.

Go to your pipeline job configuration
In Build Triggers section, check Poll SCM
Set schedule to: H/2 * * * * (polls every 2 minutes)
Click Save
Subtask 5.2: Set Up Git Repository Monitoring
For a more realistic setup, let's create a local Git repository that Jenkins can monitor:

# Create a bare repository to simulate remote repo
sudo mkdir -p /opt/git-repos
sudo git init --bare /opt/git-repos/sample-app.git
sudo chown -R jenkins:jenkins /opt/git-repos

# Add remote to our existing repository
cd /home/ubuntu/sample-app
git remote add origin /opt/git-repos/sample-app.git
git push -u origin master
Subtask 5.3: Update Pipeline to Use Git Repository
Go to your pipeline job configuration
In Pipeline section, change from Pipeline script to Pipeline script from SCM
Configure:
SCM: Git
Repository URL: /opt/git-repos/sample-app.git
Branch: */master
Script Path: Jenkinsfile
Subtask 5.4: Create Jenkinsfile in Repository
cd /home/ubuntu/sample-app

# Create Jenkinsfile with the complete pipeline
cat > Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    environment {
        DOCKER_IMAGE = 'sample-node-app'
        DOCKER_TAG = "${BUILD_NUMBER}"
        SONARQUBE_SERVER = 'SonarQube'
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                        -Dsonar.projectKey=sample-node-app \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://localhost:9000 \
                        -Dsonar.exclusions=node_modules/**
                    '''
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t ${DOCKER_IMAGE}:${DOCKER_TAG} .
                    docker build -t ${DOCKER_IMAGE}:latest .
                '''
            }
        }
        
        stage('Test Docker Image') {
            steps {
                sh '''
                    echo "Testing Docker image..."
                    docker run -d --name test-container-${BUILD_NUMBER} -p 3001:3000 ${DOCKER_IMAGE}:${DOCKER_TAG}
                    
                    sleep 10
                    
                    curl -f http://localhost:3001/health || exit 1
                    curl -f http://localhost:3001/ || exit 1
                    
                    echo "Tests passed!"
                '''
            }
            post {
                always {
                    sh '''
                        docker stop test-container-${BUILD_NUMBER} || true
                        docker rm test-container-${BUILD_NUMBER} || true
                    '''
                }
            }
        }
        
        stage('Deploy') {
            steps {
                sh '''
                    echo "Deploying application..."
                    docker stop sample-app-prod || true
                    docker rm sample-app-prod || true
                    
                    docker run -d --name sample-app-prod -p 3000:3000 ${DOCKER_IMAGE}:${DOCKER_TAG}
                    
                    echo "Application deployed successfully!"
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                docker images ${DOCKER_IMAGE} --format "table {{.Repository}}:{{.Tag}}" | tail -n +2 | head -n -5 | xargs -r docker rmi || true
            '''
        }
        success {
            echo 'Pipeline completed successfully!'
            emailext (
                subject: "Build Success: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "The build completed successfully. Check console output at ${env.BUILD_URL}",
                to: "admin@example.com"
            )
        }
        failure {
            echo 'Pipeline failed!'
            emailext (
                subject: "Build Failed: ${env.JOB_NAME} - ${env.BUILD_NUMBER}",
                body: "The build failed. Check console output at ${env.BUILD_URL}",
                to: "admin@example.com"
            )
        }
    }
}
EOF

# Commit and push Jenkinsfile
git add Jenkinsfile
git commit -m "Add Jenkinsfile for CI/CD pipeline"
git push origin master
Subtask 5.5: Test Automatic Pipeline Triggers
Make a change to the application:
cd /home/ubuntu/sample-app

# Update the application version
sed -i 's/"version": "1.0.0"/"version": "1.1.0"/' package.json
sed -i 's/version: '\''1.0.0'\''/version: '\''1.1.0'\''/' app.js

# Commit and push changes
git add .
git commit -m "Update application version to 1.1.0"
git push origin master
Wait for Jenkins to detect the change (up to 2 minutes)
Observe the automatic pipeline execution
Subtask 5.6: Monitor Build Results and Metrics
View Build History and Trends
In Jenkins dashboard, click on your pipeline job
Observe the Build History section showing recent builds
Click on Trend to see build success/failure trends
Click on individual build numbers to view:
Console output
Build artifacts
Test results
SonarQube analysis results
Access SonarQube Dashboard
Open http://localhost:9000 in your browser
Login with your SonarQube credentials
Click on your project sample-node-app
Review:
Code quality metrics
Security vulnerabilities
Code coverage
Technical debt
Code smells and bugs
Verify Application Deployment
Open http://localhost:3000 to access the deployed application
Test the endpoints:
Main endpoint: http://localhost:3000/
Health check: http://localhost:3000/health
# Test the deployed application
curl http://localhost:3000/
curl http://localhost:3000/health

# Check running containers
docker ps | grep sample-app
Troubleshooting Common Issues
Jenkins Service Issues
# If Jenkins fails to start
sudo systemctl status jenkins
sudo journalctl -u jenkins

# Check Jenkins logs
sudo tail -f /var/log/jenkins/jenkins.log
Docker Permission Issues
# If Jenkins can't access Docker
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins

# Test Docker access
sudo -u jenkins docker ps
SonarQube Connection Issues
# Check SonarQube service
sudo systemctl status sonarqube

# Check SonarQube logs
sudo tail -f /opt/sonarqube/sonarqube/logs/sonar.log
Pipeline Failures
Check console output for specific error messages
Verify all required plugins are installed
Ensure proper credentials are configured
Check file permissions and paths
Best Practices Implemented
Security Best Practices
Credential Management: Using Jenkins credential store for sensitive information
User Permissions: Proper user separation between Jenkins and system users
Container Security: Running containers with non-root users where possible
CI/CD Best Practices
Pipeline as Code: Using Jenkinsfile for version-controlled pipeline definitions
Automated Testing: Including automated tests in the pipeline
Quality Gates: Implementing SonarQube quality gates to prevent poor code deployment
Artifact Management: Proper cleanup of old Docker images
Notification System: Email notifications for build status
Docker Best Practices
Multi-stage Builds: Using appropriate base images
Layer Optimization: Minimizing Docker image layers
Security Scanning: Including security analysis in the pipeline
Resource Management: Proper container lifecycle management
Conclusion
Congratulations! You have successfully completed Lab 21: Docker and CI/CD - Automating Builds with Jenkins.

What You Accomplished
In this comprehensive lab, you have:

Installed and Configured Jenkins: Set up a complete Jenkins environment on Linux with proper security configurations and plugin management.

Integrated Docker with Jenkins: Configured Jenkins to build, test, and deploy Docker containers automatically, demonstrating the power of containerized CI/CD workflows.

Created Advanced CI/CD Pipelines: Built sophisticated Jenkins pipelines that include multiple stages for code checkout, building, testing, and deployment with proper error handling and cleanup.

Implemented Code Quality Analysis: Integrated SonarQube for static code analysis, ensuring code quality standards are maintained throughout the development lifecycle.

Automated Pipeline Triggers: Set up automatic pipeline execution based on code changes, demonstrating true continuous integration practices.

Monitored and Analyzed Results: Learned to interpret build results, quality metrics, and deployment status through Jenkins and SonarQube dashboards.

Why This Matters
The skills you've developed in this lab are crucial for modern software development because:

DevOps Integration: You now understand how to bridge development and operations through automated CI/CD pipelines
Quality Assurance: You can implement automated quality checks that prevent defective code from reaching production
Efficiency: Automated builds and deployments reduce manual effort and human error
Scalability: These practices enable teams to deploy code more frequently and reliably
Industry Relevance: Jenkins and Docker are industry-standard tools used by organizations worldwide
Next Steps
To further enhance your CI/CD skills, consider:

Exploring advanced Jenkins features like Blue Ocean UI and parallel pipeline execution
Learning about container orchestration with Kubernetes
Implementing more sophisticated testing strategies including integration and performance testing
Exploring other CI/CD tools like GitLab CI, GitHub Actions, or Azure DevOps
Studying infrastructure as code with tools like Terraform or Ansible
This lab has provided you with a solid foundation in modern CI/CD practices that will serve you well in your journey toward Docker Certified Associate (DCA) certification and professional software development career.
