Lab 69: Docker for Continuous Testing - Running Selenium Tests in Docker
Lab Objectives
By the end of this lab, you will be able to:

Pull and run Selenium container images with Chrome browser support
Configure Selenium WebDriver to work with containerized browsers
Create and execute automated web tests using Python and pytest
Integrate Selenium Docker containers with continuous integration pipelines
Set up Jenkins to automatically run Selenium tests in Docker containers
Understand the benefits of containerized testing environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts and commands
Familiarity with Python programming language
Basic knowledge of web browsers and HTML elements
Understanding of software testing concepts
Access to command line interface (CLI)
Note: Al Nafi provides ready-to-use Linux-based cloud machines with Docker pre-installed. Simply click Start Lab to begin - no need to build your own VM or install Docker manually.

Lab Environment Setup
Your Al Nafi cloud machine comes pre-configured with:

Docker Engine installed and running
Python 3.x with pip package manager
Git for version control
Text editor (nano/vim) for file editing
Task 1: Pull and Run Selenium Container Image
Subtask 1.1: Pull the Selenium Chrome Container
First, let's pull the official Selenium standalone Chrome image from Docker Hub.

# Pull the latest Selenium standalone Chrome image
docker pull selenium/standalone-chrome

# Verify the image was downloaded successfully
docker images | grep selenium
Subtask 1.2: Run Selenium Container with Chrome Browser
Now let's run the Selenium container with proper port mapping and configuration.

# Run Selenium container in detached mode
docker run -d \
  --name selenium-chrome \
  -p 4444:4444 \
  -p 7900:7900 \
  --shm-size=2g \
  selenium/standalone-chrome

# Check if the container is running
docker ps
Key Configuration Options Explained:

-d: Run container in detached mode (background)
--name selenium-chrome: Assign a friendly name to the container
-p 4444:4444: Map Selenium WebDriver port
-p 7900:7900: Map VNC port for browser viewing (optional)
--shm-size=2g: Increase shared memory for Chrome stability
Subtask 1.3: Verify Selenium Hub is Running
Check if the Selenium Grid hub is accessible and ready to accept connections.

# Test Selenium hub status using curl
curl -s http://localhost:4444/wd/hub/status | python3 -m json.tool

# Alternative: Check container logs
docker logs selenium-chrome
You should see JSON output indicating the Selenium hub is ready and Chrome browser is available.

Task 2: Set Up Python Testing Environment
Subtask 2.1: Create Project Directory Structure
Let's create a proper project structure for our Selenium tests.

# Create project directory
mkdir selenium-docker-lab
cd selenium-docker-lab

# Create subdirectories
mkdir tests
mkdir reports
mkdir screenshots

# Create project files
touch requirements.txt
touch tests/__init__.py
touch tests/test_web_automation.py
touch conftest.py
Subtask 2.2: Install Required Python Packages
Create a requirements file with necessary dependencies.

# Create requirements.txt file
cat > requirements.txt << EOF
selenium==4.15.2
pytest==7.4.3
pytest-html==4.1.1
webdriver-manager==4.0.1
requests==2.31.0
EOF

# Install the packages
pip3 install -r requirements.txt
Subtask 2.3: Configure Pytest Settings
Create a pytest configuration file to customize test execution.

# Create conftest.py for pytest configuration
cat > conftest.py << 'EOF'
import pytest
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities

@pytest.fixture(scope="session")
def driver():
    """Create a Selenium WebDriver instance for testing."""
    
    # Configure Chrome options for containerized environment
    chrome_options = Options()
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-dev-shm-usage")
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--window-size=1920,1080")
    
    # Set up desired capabilities
    capabilities = DesiredCapabilities.CHROME
    capabilities['acceptSslCerts'] = True
    capabilities['acceptInsecureCerts'] = True
    
    # Connect to remote Selenium hub running in Docker
    driver = webdriver.Remote(
        command_executor='http://localhost:4444/wd/hub',
        desired_capabilities=capabilities,
        options=chrome_options
    )
    
    # Set implicit wait time
    driver.implicitly_wait(10)
    
    yield driver
    
    # Cleanup: quit the driver after tests
    driver.quit()

@pytest.fixture(scope="function")
def screenshot_on_failure(request, driver):
    """Take screenshot on test failure."""
    yield
    if request.node.rep_call.failed:
        screenshot_name = f"screenshots/failed_{request.node.name}.png"
        driver.save_screenshot(screenshot_name)
        print(f"Screenshot saved: {screenshot_name}")

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Hook to capture test results for screenshot functionality."""
    outcome = yield
    rep = outcome.get_result()
    setattr(item, "rep_" + rep.when, rep)
EOF
Task 3: Create Selenium Test Cases
Subtask 3.1: Write Basic Web Automation Tests
Create comprehensive test cases that demonstrate various Selenium capabilities.

# Create the main test file
cat > tests/test_web_automation.py << 'EOF'
import pytest
import time
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.common.keys import Keys

class TestWebAutomation:
    """Test suite for web automation using Selenium in Docker."""
    
    def test_google_search(self, driver):
        """Test Google search functionality."""
        print("Starting Google search test...")
        
        # Navigate to Google
        driver.get("https://www.google.com")
        
        # Verify page title
        assert "Google" in driver.title
        
        # Find search box and enter search term
        search_box = WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.NAME, "q"))
        )
        
        search_term = "Docker Selenium automation"
        search_box.clear()
        search_box.send_keys(search_term)
        search_box.send_keys(Keys.RETURN)
        
        # Wait for results to load
        WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.ID, "search"))
        )
        
        # Verify search results contain our term
        page_source = driver.page_source.lower()
        assert "docker" in page_source
        assert "selenium" in page_source
        
        print("Google search test completed successfully!")
    
    def test_form_interaction(self, driver):
        """Test form filling and submission."""
        print("Starting form interaction test...")
        
        # Navigate to a test form page
        driver.get("https://httpbin.org/forms/post")
        
        # Fill out the form
        customer_name = driver.find_element(By.NAME, "custname")
        customer_name.clear()
        customer_name.send_keys("Docker Test User")
        
        customer_tel = driver.find_element(By.NAME, "custtel")
        customer_tel.clear()
        customer_tel.send_keys("555-1234")
        
        customer_email = driver.find_element(By.NAME, "custemail")
        customer_email.clear()
        customer_email.send_keys("test@docker-selenium.com")
        
        # Select pizza size
        pizza_size = driver.find_element(By.XPATH, "//input[@value='large']")
        pizza_size.click()
        
        # Add toppings
        topping_bacon = driver.find_element(By.NAME, "topping")
        topping_bacon.click()
        
        # Submit the form
        submit_button = driver.find_element(By.XPATH, "//input[@type='submit']")
        submit_button.click()
        
        # Verify form submission
        WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.TAG_NAME, "pre"))
        )
        
        page_content = driver.page_source
        assert "Docker Test User" in page_content
        assert "test@docker-selenium.com" in page_content
        
        print("Form interaction test completed successfully!")
    
    def test_navigation_and_elements(self, driver):
        """Test page navigation and element interactions."""
        print("Starting navigation and elements test...")
        
        # Navigate to example.com
        driver.get("https://example.com")
        
        # Verify page elements
        heading = driver.find_element(By.TAG_NAME, "h1")
        assert heading.text == "Example Domain"
        
        # Check for links
        more_info_link = driver.find_element(By.LINK_TEXT, "More information...")
        assert more_info_link.is_displayed()
        
        # Get page dimensions
        window_size = driver.get_window_size()
        assert window_size['width'] > 0
        assert window_size['height'] > 0
        
        # Take a screenshot for verification
        driver.save_screenshot("screenshots/example_page.png")
        
        print("Navigation and elements test completed successfully!")
    
    def test_javascript_execution(self, driver):
        """Test JavaScript execution capabilities."""
        print("Starting JavaScript execution test...")
        
        driver.get("https://example.com")
        
        # Execute JavaScript to get page title
        page_title = driver.execute_script("return document.title;")
        assert "Example Domain" in page_title
        
        # Execute JavaScript to scroll page
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
        
        # Execute JavaScript to modify page content
        driver.execute_script(
            "document.body.style.backgroundColor = 'lightblue';"
        )
        
        # Verify the change took effect
        bg_color = driver.execute_script(
            "return window.getComputedStyle(document.body).backgroundColor;"
        )
        
        # Take screenshot to verify visual change
        driver.save_screenshot("screenshots/modified_page.png")
        
        print("JavaScript execution test completed successfully!")

@pytest.mark.usefixtures("screenshot_on_failure")
class TestErrorHandling:
    """Test suite for error handling scenarios."""
    
    def test_element_not_found_handling(self, driver):
        """Test handling of missing elements."""
        driver.get("https://example.com")
        
        # This should handle the case gracefully
        try:
            non_existent = driver.find_element(By.ID, "non-existent-element")
            assert False, "Should not find non-existent element"
        except Exception as e:
            print(f"Correctly handled missing element: {type(e).__name__}")
            assert True
    
    def test_timeout_handling(self, driver):
        """Test timeout handling for slow-loading elements."""
        driver.get("https://httpbin.org/delay/2")
        
        # This tests our implicit wait configuration
        start_time = time.time()
        try:
            element = WebDriverWait(driver, 5).until(
                EC.presence_of_element_located((By.TAG_NAME, "body"))
            )
            end_time = time.time()
            load_time = end_time - start_time
            print(f"Page loaded in {load_time:.2f} seconds")
            assert load_time >= 2  # Should take at least 2 seconds due to delay
        except Exception as e:
            print(f"Timeout handled correctly: {type(e).__name__}")
EOF
Subtask 3.2: Run the Test Suite
Execute the test suite and generate reports.

# Run tests with verbose output and HTML report
pytest tests/ -v --html=reports/selenium_test_report.html --self-contained-html

# Run specific test class
pytest tests/test_web_automation.py::TestWebAutomation::test_google_search -v

# Run tests with custom markers (if any)
pytest tests/ -v -m "not slow"
Task 4: Advanced Docker Configuration for Testing
Subtask 4.1: Create Docker Compose for Selenium Grid
Set up a more robust testing environment using Docker Compose.

# Create docker-compose.yml for Selenium Grid
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  selenium-hub:
    image: selenium/hub:4.15.0
    container_name: selenium-hub
    ports:
      - "4442:4442"
      - "4443:4443"
      - "4444:4444"
    environment:
      - GRID_MAX_SESSION=16
      - GRID_BROWSER_TIMEOUT=300
      - GRID_TIMEOUT=300

  chrome-node:
    image: selenium/node-chrome:4.15.0
    container_name: chrome-node
    shm_size: 2gb
    depends_on:
      - selenium-hub
    environment:
      - HUB_HOST=selenium-hub
      - HUB_PORT=4444
      - NODE_MAX_INSTANCES=4
      - NODE_MAX_SESSION=4
    ports:
      - "7900:7900"
    volumes:
      - ./screenshots:/home/seluser/screenshots

  firefox-node:
    image: selenium/node-firefox:4.15.0
    container_name: firefox-node
    shm_size: 2gb
    depends_on:
      - selenium-hub
    environment:
      - HUB_HOST=selenium-hub
      - HUB_PORT=4444
      - NODE_MAX_INSTANCES=2
      - NODE_MAX_SESSION=2
    ports:
      - "7901:7900"

  test-runner:
    build: .
    container_name: test-runner
    depends_on:
      - selenium-hub
      - chrome-node
    volumes:
      - ./tests:/app/tests
      - ./reports:/app/reports
      - ./screenshots:/app/screenshots
    environment:
      - SELENIUM_HUB_URL=http://selenium-hub:4444/wd/hub
    command: ["pytest", "tests/", "-v", "--html=reports/test_report.html"]
EOF
Subtask 4.2: Create Dockerfile for Test Runner
Create a Dockerfile for containerizing the test execution environment.

# Create Dockerfile for test runner
cat > Dockerfile << 'EOF'
FROM python:3.11-slim

# Set working directory
WORKDIR /app

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    wget \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements and install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy test files
COPY tests/ ./tests/
COPY conftest.py .

# Create directories for reports and screenshots
RUN mkdir -p reports screenshots

# Set environment variables
ENV PYTHONPATH=/app
ENV SELENIUM_HUB_URL=http://selenium-hub:4444/wd/hub

# Default command
CMD ["pytest", "tests/", "-v"]
EOF
Subtask 4.3: Update Test Configuration for Grid
Update the conftest.py to work with Selenium Grid.

# Create updated conftest.py for Grid setup
cat > conftest_grid.py << 'EOF'
import pytest
import os
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.firefox.options import Options as FirefoxOptions
from selenium.webdriver.common.desired_capabilities import DesiredCapabilities

@pytest.fixture(scope="session", params=["chrome", "firefox"])
def driver(request):
    """Create a Selenium WebDriver instance for cross-browser testing."""
    
    browser = request.param
    selenium_hub_url = os.getenv('SELENIUM_HUB_URL', 'http://localhost:4444/wd/hub')
    
    if browser == "chrome":
        chrome_options = Options()
        chrome_options.add_argument("--no-sandbox")
        chrome_options.add_argument("--disable-dev-shm-usage")
        chrome_options.add_argument("--disable-gpu")
        chrome_options.add_argument("--window-size=1920,1080")
        
        capabilities = DesiredCapabilities.CHROME
        capabilities['acceptSslCerts'] = True
        capabilities['acceptInsecureCerts'] = True
        
        driver = webdriver.Remote(
            command_executor=selenium_hub_url,
            desired_capabilities=capabilities,
            options=chrome_options
        )
    
    elif browser == "firefox":
        firefox_options = FirefoxOptions()
        firefox_options.add_argument("--width=1920")
        firefox_options.add_argument("--height=1080")
        
        capabilities = DesiredCapabilities.FIREFOX
        capabilities['acceptSslCerts'] = True
        capabilities['acceptInsecureCerts'] = True
        
        driver = webdriver.Remote(
            command_executor=selenium_hub_url,
            desired_capabilities=capabilities,
            options=firefox_options
        )
    
    driver.implicitly_wait(10)
    yield driver
    driver.quit()

@pytest.fixture(autouse=True)
def test_info(request):
    """Print test information."""
    print(f"\n--- Running test: {request.node.name} ---")
    yield
    print(f"--- Completed test: {request.node.name} ---")
EOF
Task 5: Jenkins Integration for Continuous Testing
Subtask 5.1: Install Jenkins in Docker
Set up Jenkins to run alongside our Selenium infrastructure.

# Create Jenkins directory for persistent data
mkdir jenkins_home
chmod 777 jenkins_home

# Run Jenkins in Docker
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v $(pwd)/jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v $(which docker):/usr/bin/docker \
  jenkins/jenkins:lts

# Get initial admin password
echo "Waiting for Jenkins to start..."
sleep 30
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
Subtask 5.2: Create Jenkins Pipeline Script
Create a Jenkinsfile for automated Selenium testing.

# Create Jenkinsfile for CI/CD pipeline
cat > Jenkinsfile << 'EOF'
pipeline {
    agent any
    
    environment {
        DOCKER_COMPOSE_FILE = 'docker-compose.yml'
        TEST_REPORTS_DIR = 'reports'
        SCREENSHOTS_DIR = 'screenshots'
    }
    
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }
        
        stage('Setup Test Environment') {
            steps {
                echo 'Setting up Selenium Grid...'
                script {
                    // Clean up any existing containers
                    sh 'docker-compose -f ${DOCKER_COMPOSE_FILE} down || true'
                    
                    // Start Selenium Grid
                    sh 'docker-compose -f ${DOCKER_COMPOSE_FILE} up -d selenium-hub chrome-node'
                    
                    // Wait for services to be ready
                    sh 'sleep 30'
                    
                    // Verify Selenium Hub is running
                    sh 'curl -f http://localhost:4444/wd/hub/status || exit 1'
                }
            }
        }
        
        stage('Run Selenium Tests') {
            steps {
                echo 'Running Selenium tests...'
                script {
                    try {
                        // Run tests using docker-compose
                        sh '''
                            docker-compose -f ${DOCKER_COMPOSE_FILE} run --rm test-runner \
                            pytest tests/ -v \
                            --html=reports/jenkins_test_report.html \
                            --self-contained-html \
                            --junitxml=reports/junit_results.xml
                        '''
                    } catch (Exception e) {
                        echo "Tests failed: ${e.getMessage()}"
                        currentBuild.result = 'UNSTABLE'
                    }
                }
            }
        }
        
        stage('Collect Test Results') {
            steps {
                echo 'Collecting test results and artifacts...'
                
                // Archive test reports
                archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true
                archiveArtifacts artifacts: 'screenshots/**/*', allowEmptyArchive: true
                
                // Publish JUnit test results
                publishTestResults testResultsPattern: 'reports/junit_results.xml'
                
                // Publish HTML reports
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'reports',
                    reportFiles: 'jenkins_test_report.html',
                    reportName: 'Selenium Test Report'
                ])
            }
        }
    }
    
    post {
        always {
            echo 'Cleaning up test environment...'
            script {
                // Stop and remove containers
                sh 'docker-compose -f ${DOCKER_COMPOSE_FILE} down || true'
                
                // Clean up dangling images
                sh 'docker image prune -f || true'
            }
        }
        
        success {
            echo 'All tests passed successfully!'
            // You can add notifications here (email, Slack, etc.)
        }
        
        failure {
            echo 'Tests failed! Check the reports for details.'
            // You can add failure notifications here
        }
        
        unstable {
            echo 'Some tests failed, but build completed.'
            // You can add unstable build notifications here
        }
    }
}
EOF
Subtask 5.3: Create Jenkins Job Configuration
Create a shell script to help configure Jenkins jobs.

# Create Jenkins job configuration script
cat > setup_jenkins_job.sh << 'EOF'
#!/bin/bash

echo "=== Jenkins Job Setup Instructions ==="
echo ""
echo "1. Open Jenkins at http://localhost:8080"
echo "2. Use the initial admin password shown above"
echo "3. Install suggested plugins"
echo "4. Create admin user"
echo "5. Create a new Pipeline job:"
echo "   - Click 'New Item'"
echo "   - Enter name: 'Selenium-Docker-Tests'"
echo "   - Select 'Pipeline'"
echo "   - Click 'OK'"
echo ""
echo "6. In the job configuration:"
echo "   - Under 'Pipeline', select 'Pipeline script from SCM'"
echo "   - SCM: Git"
echo "   - Repository URL: (your git repository URL)"
echo "   - Script Path: Jenkinsfile"
echo ""
echo "7. Required Jenkins Plugins:"
echo "   - Docker Pipeline"
echo "   - HTML Publisher"
echo "   - JUnit"
echo "   - Pipeline"
echo ""
echo "8. Save and run the job!"
echo ""
echo "=== Manual Jenkins Job Creation ==="
echo "If you prefer to create the job manually:"
echo ""

# Create a basic Jenkins job XML
cat > jenkins_job_config.xml << 'JOBEOF'
<?xml version='1.1' encoding='UTF-8'?>
<flow-definition plugin="workflow-job@2.40">
  <actions/>
  <description>Automated Selenium tests running in Docker containers</description>
  <keepDependencies>false</keepDependencies>
  <properties>
    <org.jenkinsci.plugins.workflow.job.properties.PipelineTriggersJobProperty>
      <triggers>
        <hudson.triggers.SCMTrigger>
          <spec>H/5 * * * *</spec>
          <ignorePostCommitHooks>false</ignorePostCommitHooks>
        </hudson.triggers.SCMTrigger>
      </triggers>
    </org.jenkinsci.plugins.workflow.job.properties.PipelineTriggersJobProperty>
  </properties>
  <definition class="org.jenkinsci.plugins.workflow.cps.CpsFlowDefinition" plugin="workflow-cps@2.92">
    <script>
pipeline {
    agent any
    
    stages {
        stage('Setup') {
            steps {
                echo 'Setting up Selenium Grid...'
                sh 'docker-compose up -d selenium-hub chrome-node'
                sh 'sleep 30'
            }
        }
        
        stage('Test') {
            steps {
                echo 'Running tests...'
                sh 'pytest tests/ -v --html=reports/test_report.html'
            }
        }
        
        stage('Reports') {
            steps {
                archiveArtifacts artifacts: 'reports/**/*'
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'reports',
                    reportFiles: 'test_report.html',
                    reportName: 'Test Report'
                ])
            }
        }
    }
    
    post {
        always {
            sh 'docker-compose down'
        }
    }
}
    </script>
    <sandbox>true</sandbox>
  </definition>
  <triggers/>
  <disabled>false</disabled>
</flow-definition>
JOBEOF

echo "Job configuration XML created: jenkins_job_config.xml"
echo "You can import this into Jenkins if needed."
EOF

# Make the script executable
chmod +x setup_jenkins_job.sh
Task 6: Running and Monitoring Tests
Subtask 6.1: Execute Complete Test Suite
Run the full test suite using Docker Compose.

# Start the complete testing infrastructure
docker-compose up -d

# Wait for services to be ready
echo "Waiting for Selenium Grid to be ready..."
sleep 30

# Check Selenium Grid status
curl -s http://localhost:4444/wd/hub/status | python3 -m json.tool

# Run tests against the Grid
pytest tests/ -v --html=reports/grid_test_report.html --self-contained-html

# View running containers
docker ps
Subtask 6.2: Monitor Test Execution
Set up monitoring and logging for test execution.

# Create monitoring script
cat > monitor_tests.sh << 'EOF'
#!/bin/bash

echo "=== Selenium Docker Test Monitoring ==="
echo ""

# Check container status
echo "1. Container Status:"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
echo ""

# Check Selenium Hub status
echo "2. Selenium Hub Status:"
if curl -s http://localhost:4444/wd/hub/status > /dev/null; then
    echo "✓ Selenium Hub is running"
    curl -s http://localhost:4444/wd/hub/status | python3 -c "
import sys, json
data = json.load(sys.stdin)
print(f'Ready: {data[\"value\"][\"ready\"]}')
print(f'Message: {data[\"value\"][\"message\"]}')
"
else
    echo "✗ Selenium Hub is not accessible"
fi
echo ""

# Check available browsers
echo "3. Available Browsers:"
curl -s http://localhost:4444/wd/hub/status | python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    nodes = data['value']['nodes']
    for node in nodes:
        for slot in node['slots']:
            stereotype = slot['stereotype']
            print(f'Browser: {stereotype[\"browserName\"]} {stereotype.get(\"browserVersion\", \"latest\")}')
except:
    print('Unable to retrieve browser information')
"
echo ""

# Check resource usage
echo "4. Resource Usage:"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
echo ""

# Check logs for errors
echo "5. Recent Container Logs:"
echo "--- Selenium Hub ---"
docker logs --tail 5 selenium-hub 2>/dev/null || echo "No selenium-hub container found"
echo ""
echo "--- Chrome Node ---"
docker logs --tail 5 chrome-node 2>/dev/null || echo "No chrome-node container found"
echo ""

# Check test reports
echo "6. Test Reports:"
if [ -d "reports" ]; then
    ls -la reports/
else
    echo "No reports directory found"
fi
echo ""

# Check screenshots
echo "7. Screenshots:"
if [ -d "screenshots" ]; then
    ls -la screenshots/
else
    echo "No screenshots directory found"
fi
EOF

# Make monitoring script executable
chmod +x monitor_tests.sh

# Run monitoring
./monitor_tests.sh
Subtask 6.3: Create Test Cleanup Script
Create a script to clean up test environments.

# Create cleanup script
cat > cleanup_tests.sh << 'EOF'
#!/bin/bash

echo "=== Cleaning up Selenium Docker Test Environment ==="

# Stop and remove containers
echo "Stopping containers..."
docker-compose down

# Remove individual containers if they exist
docker stop selenium-chrome jenkins 2>/dev/null || true
docker rm selenium-chrome jenkins 2>/dev/null || true

# Clean up dangling images
echo "Cleaning up Docker images..."
docker image prune -f

# Clean up volumes (optional - uncomment if needed)
# docker volume prune -f

# Clean up networks
docker network prune -f

# Show remaining Docker resources
echo ""
echo "Remaining Docker resources:"
echo "Containers:"
docker ps -a
echo ""
echo "Images:"
docker images
echo ""
echo "Volumes:"
docker volume ls
echo ""

echo "Cleanup completed!"
EOF

# Make cleanup script executable
chmod +x cleanup_tests.sh
Troubleshooting Common Issues
Issue 1: Container Memory Problems
If you encounter memory-related issues:

# Increase shared memory for Chrome
docker run -d --name selenium-chrome --shm-size=2g selenium/standalone-chrome

# Check container memory usage
docker stats selenium-chrome
Issue 2: Port Conflicts
If ports are already in use:

# Check what's using port 4444
sudo netstat -tulpn | grep 4444

# Use different ports
docker run -d -p 4445:4444 selenium/standalone-chrome
Issue 3: Test Timeouts
If tests are timing out:

# Increase timeout in conftest.py
# driver.implicitly_wait(20)  # Increase from 10 to 20 seconds

# Check Selenium hub logs
docker logs selenium-hub
Issue 4: Browser Crashes
If browsers are crashing:

# Increase shared memory
docker run -d --shm-size=2g selenium/standalone-chrome

# Check Chrome node logs
docker logs chrome-node
Lab Validation and Testing
Validation Checklist
Run through this checklist to ensure everything is working:

# Create validation script
cat > validate_lab.sh << 'EOF'
#!/bin/bash

echo "=== Lab 69 Validation Checklist ==="
echo ""

# Test 1: Docker containers running
echo "1. Checking Docker containers..."
if docker ps | grep -q selenium; then
    echo "✓ Selenium containers are running"
else
    echo "✗ Selenium containers not found"
fi

# Test 2: Selenium Hub accessible
echo "2
