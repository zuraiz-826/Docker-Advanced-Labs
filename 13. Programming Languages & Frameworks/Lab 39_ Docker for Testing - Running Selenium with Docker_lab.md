Lab 39: Docker for Testing - Running Selenium with Docker
Objectives
By the end of this lab, you will be able to:

Understand the benefits of using Docker for automated testing environments
Pull and run Selenium Docker images with Chrome browser support
Configure and connect to Selenium containers from Python test scripts
Execute automated browser tests using Selenium WebDriver
Manage Docker containers and images for testing workflows
Implement best practices for containerized testing environments
Prerequisites
Before starting this lab, you should have:

Basic understanding of Docker concepts (containers, images, ports)
Familiarity with command-line interface operations
Basic knowledge of Python programming
Understanding of web browser automation concepts
Access to a terminal or command prompt
Required Software:

Docker Engine (pre-installed on your cloud machine)
Python 3.x with pip package manager
Text editor or IDE for writing Python scripts
Ready-to-Use Cloud Machines
Al Nafi provides pre-configured Linux-based cloud machines with all necessary tools installed. Simply click Start Lab to access your environment. No need to build your own virtual machine or install Docker manually - everything is ready to use!

Your cloud machine includes:

Docker Engine (latest stable version)
Python 3.x with pip
Text editors (nano, vim)
All necessary network configurations
Lab Tasks
Task 1: Pull the Selenium Docker Image
Subtask 1.1: Verify Docker Installation
First, let's confirm that Docker is running properly on your system.

docker --version
docker info
You should see Docker version information and system details. If Docker is not running, start it with:

sudo systemctl start docker
Subtask 1.2: Pull the Selenium Standalone Chrome Image
Download the official Selenium Docker image with Chrome browser support:

docker pull selenium/standalone-chrome:latest
This command downloads the latest Selenium image that includes:

Selenium WebDriver server
Google Chrome browser
ChromeDriver
VNC server for remote viewing
Subtask 1.3: Verify Image Download
Check that the image was successfully downloaded:

docker images | grep selenium
You should see output similar to:

selenium/standalone-chrome   latest    abc123def456   2 days ago   1.2GB
Task 2: Run the Selenium Container with Chrome Browser
Subtask 2.1: Start the Selenium Container
Run the Selenium container with proper port mapping and configuration:

docker run -d \
  --name selenium-chrome \
  --shm-size=2g \
  -p 4444:4444 \
  -p 7900:7900 \
  selenium/standalone-chrome:latest
Command Explanation:

-d: Run container in detached mode (background)
--name selenium-chrome: Assign a friendly name to the container
--shm-size=2g: Allocate 2GB shared memory (prevents Chrome crashes)
-p 4444:4444: Map Selenium WebDriver port
-p 7900:7900: Map VNC port for browser viewing
Subtask 2.2: Verify Container Status
Check that the container is running:

docker ps
You should see your selenium-chrome container in the list with status "Up".

Subtask 2.3: Check Selenium Hub Status
Verify that Selenium is ready to accept connections:

curl http://localhost:4444/wd/hub/status
You should receive a JSON response indicating the Selenium hub is ready.

Task 3: Connect to Selenium Container from Python Test Script
Subtask 3.1: Install Required Python Packages
Install the Selenium Python library:

pip3 install selenium requests
Subtask 3.2: Create a Basic Test Script
Create a new Python file for your test script:

nano selenium_test.py
Add the following code to create a basic Selenium test:

#!/usr/bin/env python3

from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
import time
import sys

def test_google_search():
    """Test Google search functionality"""
    
    # Configure Chrome options for Docker environment
    chrome_options = Options()
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-dev-shm-usage")
    chrome_options.add_argument("--disable-gpu")
    
    # Connect to remote Selenium server
    try:
        driver = webdriver.Remote(
            command_executor='http://localhost:4444/wd/hub',
            options=chrome_options
        )
        
        print("✓ Successfully connected to Selenium container")
        
        # Navigate to Google
        driver.get("https://www.google.com")
        print("✓ Navigated to Google homepage")
        
        # Wait for search box to be present
        search_box = WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.NAME, "q"))
        )
        
        # Perform search
        search_term = "Docker Selenium testing"
        search_box.send_keys(search_term)
        search_box.submit()
        
        print(f"✓ Searched for: {search_term}")
        
        # Wait for results to load
        WebDriverWait(driver, 10).until(
            EC.presence_of_element_located((By.ID, "search"))
        )
        
        # Get page title
        page_title = driver.title
        print(f"✓ Page title: {page_title}")
        
        # Verify search results
        if search_term.lower() in page_title.lower():
            print("✓ Test PASSED: Search results displayed correctly")
            return True
        else:
            print("✗ Test FAILED: Search results not as expected")
            return False
            
    except Exception as e:
        print(f"✗ Test FAILED with error: {str(e)}")
        return False
        
    finally:
        # Clean up
        if 'driver' in locals():
            driver.quit()
            print("✓ Browser session closed")

def test_form_interaction():
    """Test form interaction on a demo website"""
    
    chrome_options = Options()
    chrome_options.add_argument("--no-sandbox")
    chrome_options.add_argument("--disable-dev-shm-usage")
    chrome_options.add_argument("--disable-gpu")
    
    try:
        driver = webdriver.Remote(
            command_executor='http://localhost:4444/wd/hub',
            options=chrome_options
        )
        
        print("\n--- Form Interaction Test ---")
        
        # Navigate to a demo form page
        driver.get("https://httpbin.org/forms/post")
        print("✓ Navigated to demo form page")
        
        # Fill out form fields
        customer_name = driver.find_element(By.NAME, "custname")
        customer_name.send_keys("John Doe")
        
        customer_tel = driver.find_element(By.NAME, "custtel")
        customer_tel.send_keys("555-1234")
        
        customer_email = driver.find_element(By.NAME, "custemail")
        customer_email.send_keys("john.doe@example.com")
        
        print("✓ Filled out form fields")
        
        # Submit form
        submit_button = driver.find_element(By.CSS_SELECTOR, "input[type='submit']")
        submit_button.click()
        
        print("✓ Form submitted successfully")
        
        # Wait for response page
        time.sleep(2)
        
        # Check if we got a response
        if "httpbin.org" in driver.current_url:
            print("✓ Form submission test PASSED")
            return True
        else:
            print("✗ Form submission test FAILED")
            return False
            
    except Exception as e:
        print(f"✗ Form test FAILED with error: {str(e)}")
        return False
        
    finally:
        if 'driver' in locals():
            driver.quit()
            print("✓ Browser session closed")

if __name__ == "__main__":
    print("Starting Selenium Docker Tests...")
    print("=" * 50)
    
    # Run tests
    test1_result = test_google_search()
    test2_result = test_form_interaction()
    
    # Summary
    print("\n" + "=" * 50)
    print("TEST SUMMARY:")
    print(f"Google Search Test: {'PASSED' if test1_result else 'FAILED'}")
    print(f"Form Interaction Test: {'PASSED' if test2_result else 'FAILED'}")
    
    if test1_result and test2_result:
        print("✓ All tests PASSED!")
        sys.exit(0)
    else:
        print("✗ Some tests FAILED!")
        sys.exit(1)
Save the file by pressing Ctrl+X, then Y, then Enter.

Subtask 3.3: Make the Script Executable
Make your test script executable:

chmod +x selenium_test.py
Task 4: Run Automated Browser Tests and Capture Results
Subtask 4.1: Execute the Test Script
Run your automated tests:

python3 selenium_test.py
The script will:

Connect to the Selenium container
Open Google and perform a search
Navigate to a demo form and fill it out
Display test results and status
Subtask 4.2: Monitor Container Logs
In a separate terminal, monitor the Selenium container logs to see browser activity:

docker logs -f selenium-chrome
You'll see logs showing:

Selenium server startup
WebDriver session creation
Browser navigation events
Test execution details
Subtask 4.3: View Browser Activity via VNC (Optional)
If you want to see the browser in action, you can connect to the VNC server:

Open a web browser on your local machine
Navigate to: http://your-cloud-machine-ip:7900
Use password: secret
You'll see the Chrome browser running your tests in real-time
Subtask 4.4: Create a Test Results Report
Create a script to generate a simple test report:

nano generate_report.py
Add the following code:

#!/usr/bin/env python3

import subprocess
import datetime
import json

def run_tests_with_report():
    """Run tests and generate a detailed report"""
    
    print("Generating Test Report...")
    print("=" * 60)
    
    # Get current timestamp
    timestamp = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    # Run the test script and capture output
    try:
        result = subprocess.run(
            ['python3', 'selenium_test.py'],
            capture_output=True,
            text=True,
            timeout=120
        )
        
        # Create report data
        report_data = {
            "timestamp": timestamp,
            "exit_code": result.returncode,
            "stdout": result.stdout,
            "stderr": result.stderr,
            "status": "PASSED" if result.returncode == 0 else "FAILED"
        }
        
        # Generate HTML report
        html_report = f"""
<!DOCTYPE html>
<html>
<head>
    <title>Selenium Docker Test Report</title>
    <style>
        body {{ font-family: Arial, sans-serif; margin: 20px; }}
        .header {{ background-color: #f0f0f0; padding: 10px; border-radius: 5px; }}
        .passed {{ color: green; font-weight: bold; }}
        .failed {{ color: red; font-weight: bold; }}
        .output {{ background-color: #f8f8f8; padding: 10px; border: 1px solid #ddd; white-space: pre-wrap; }}
    </style>
</head>
<body>
    <div class="header">
        <h1>Selenium Docker Test Report</h1>
        <p><strong>Generated:</strong> {timestamp}</p>
        <p><strong>Status:</strong> <span class="{report_data['status'].lower()}">{report_data['status']}</span></p>
    </div>
    
    <h2>Test Output</h2>
    <div class="output">{report_data['stdout']}</div>
    
    <h2>Error Output</h2>
    <div class="output">{report_data['stderr'] if report_data['stderr'] else 'No errors'}</div>
</body>
</html>
        """
        
        # Save HTML report
        with open('test_report.html', 'w') as f:
            f.write(html_report)
        
        # Save JSON report
        with open('test_report.json', 'w') as f:
            json.dump(report_data, f, indent=2)
        
        print(f"Test Status: {report_data['status']}")
        print(f"Exit Code: {report_data['exit_code']}")
        print("\nReports generated:")
        print("- test_report.html (HTML format)")
        print("- test_report.json (JSON format)")
        
        return report_data['exit_code'] == 0
        
    except subprocess.TimeoutExpired:
        print("✗ Tests timed out after 120 seconds")
        return False
    except Exception as e:
        print(f"✗ Error running tests: {str(e)}")
        return False

if __name__ == "__main__":
    success = run_tests_with_report()
    exit(0 if success else 1)
Run the report generator:

python3 generate_report.py
Task 5: Clean Up Containers and Images
Subtask 5.1: Stop the Selenium Container
Stop the running Selenium container:

docker stop selenium-chrome
Subtask 5.2: Remove the Container
Remove the stopped container:

docker rm selenium-chrome
Subtask 5.3: List and Remove Unused Images (Optional)
View all Docker images:

docker images
If you want to remove the Selenium image to free up space:

docker rmi selenium/standalone-chrome:latest
Note: Only remove images if you're sure you won't need them again, as you'll need to re-download them.

Subtask 5.4: Clean Up System Resources
Remove any dangling images and unused containers:

docker system prune -f
Subtask 5.5: Verify Cleanup
Confirm that resources have been cleaned up:

docker ps -a
docker images
You should see that the selenium-chrome container is no longer listed.

Troubleshooting Tips
Common Issues and Solutions
Issue 1: Container fails to start

Symptom: Container exits immediately after starting
Solution: Check if ports 4444 or 7900 are already in use
netstat -tulpn | grep :4444
docker run -p 4445:4444 -p 7901:7900 selenium/standalone-chrome:latest
Issue 2: Python script can't connect to Selenium

Symptom: Connection refused error
Solution: Verify container is running and ports are mapped correctly
docker ps
curl http://localhost:4444/wd/hub/status
Issue 3: Chrome crashes in container

Symptom: Browser crashes or fails to start
Solution: Increase shared memory size
docker run --shm-size=2g selenium/standalone-chrome:latest
Issue 4: Tests run slowly

Symptom: Long wait times for page loads
Solution: Add explicit waits and optimize Chrome options
chrome_options.add_argument("--disable-extensions")
chrome_options.add_argument("--disable-plugins")
Best Practices
Resource Management

Always stop and remove containers when done
Use appropriate shared memory allocation
Monitor container resource usage
Test Design

Use explicit waits instead of sleep()
Implement proper error handling
Create reusable test functions
Security

Don't expose Selenium ports to public networks
Use environment variables for sensitive data
Regularly update Docker images
Performance

Run tests in parallel when possible
Use headless mode for faster execution
Optimize Chrome browser settings
Conclusion
Congratulations! You have successfully completed Lab 39: Docker for Testing - Running Selenium with Docker.

What you accomplished:

Container Management: You learned how to pull, run, and manage Selenium Docker containers with Chrome browser support
Test Automation: You created and executed automated browser tests using Python and Selenium WebDriver
Remote Testing: You connected to containerized browsers from external test scripts
Reporting: You generated comprehensive test reports in multiple formats
Resource Cleanup: You properly cleaned up Docker resources to maintain system efficiency
Why this matters:

Consistency: Docker ensures your tests run in identical environments across different machines
Isolation: Containerized testing prevents conflicts with local browser installations
Scalability: You can easily spin up multiple browser instances for parallel testing
Portability: Your test environment can be shared and reproduced anywhere Docker runs
Efficiency: Automated cleanup and resource management keeps your system optimized
Next Steps:

Explore running multiple browser containers simultaneously
Integrate this setup with CI/CD pipelines
Learn about Selenium Grid for distributed testing
Experiment with different browser types (Firefox, Edge)
Implement more complex test scenarios and page object models
This lab has provided you with essential skills for modern automated testing workflows using containerization technology, preparing you for real-world testing scenarios and Docker Certified Associate certification objectives.
