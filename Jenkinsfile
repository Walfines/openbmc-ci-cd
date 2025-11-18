pipeline {
    agent any
    
    environment {
        BMC_IP = '192.168.1.100'
        BMC_USER = 'root'
        BMC_PASSWORD = '0penBmc'
        QEMU_PID = 'qemu.pid'
        TEST_TIMEOUT = '30m'
    }
    
    stages {
        stage('Setup Environment') {
            steps {
                echo "Setting up test environment..."
                sh '''
                    mkdir -p images test-results logs
                    # Check if QEMU is installed
                    which qemu-system-arm || {
                        echo "QEMU not found, installing..."
                        sudo apt-get update && sudo apt-get install -y qemu-system-arm
                    }
                    # Check if OpenBMC image exists, if not download
                    if [ ! -f images/openbmc.qcow2 ]; then
                        echo "Downloading OpenBMC image..."
                        wget -O images/openbmc.qcow2 https://example.com/openbmc-test-image.qcow2
                    fi
                '''
            }
        }
        
        stage('Start QEMU with OpenBMC') {
            steps {
                echo "Starting QEMU with OpenBMC..."
                sh '''
                    cd images
                    # Start QEMU with proper OpenBMC configuration
                    qemu-system-arm -machine virt -nographic \
                        -cpu cortex-a15 \
                        -m 512M \
                        -drive file=openbmc.qcow2,format=qcow2,if=sd \
                        -netdev user,id=net0,hostfwd=tcp::2222-:22,hostfwd=tcp::8080-:80 \
                        -device virtio-net-device,netdev=net0 \
                        -serial mon:stdio &
                    echo $! > ../${QEMU_PID}
                    
                    # Wait for BMC to boot
                    echo "Waiting for OpenBMC to boot..."
                    sleep 60
                '''
            }
        }
        
        stage('Wait for BMC Ready') {
            steps {
                echo "Waiting for BMC services to be ready..."
                sh '''
                    # Wait for BMC web interface
                    timeout 300 bash -c '
                        until curl -f -s -o /dev/null http://localhost:8080; do
                            echo "Waiting for BMC web interface..."
                            sleep 10
                        done
                    '
                    
                    # Wait for SSH access
                    timeout 300 bash -c "
                        until sshpass -p '${BMC_PASSWORD}' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 ${BMC_USER}@localhost -p 2222 'echo BMC is ready'; do
                            echo 'Waiting for BMC SSH...'
                            sleep 10
                        done
                    "
                '''
            }
        }
        
        stage('BMC Basic Health Check') {
            steps {
                echo "Running BMC basic health checks..."
                sh '''
                    timeout ${TEST_TIMEOUT} sshpass -p '${BMC_PASSWORD}' ssh -o StrictHostKeyChecking=no -p 2222 ${BMC_USER}@localhost << 'EOF' > test-results/health-check.log
                        echo "=== BMC Version ==="
                        cat /etc/os-release
                        echo ""
                        
                        echo "=== System Uptime ==="
                        uptime
                        echo ""
                        
                        echo "=== Memory Usage ==="
                        free -m
                        echo ""
                        
                        echo "=== Disk Usage ==="
                        df -h
                        echo ""
                        
                        echo "=== Running Services ==="
                        systemctl list-units --state=running | grep -E "(phosphor|openbmc|bmc)"
                        echo ""
                        
                        echo "=== Network Configuration ==="
                        ip addr show
                        echo ""
EOF

                    # Check if health check passed
                    if grep -q "BMC" test-results/health-check.log && grep -q "running" test-results/health-check.log; then
                        echo "BMC Health Check: PASSED" > test-results/health-check-result.txt
                    else
                        echo "BMC Health Check: FAILED" > test-results/health-check-result.txt
                        exit 1
                    fi
                '''
            }
            post {
                always {
                    archiveArtifacts 'test-results/health-check.log, test-results/health-check-result.txt'
                }
            }
        }
        
        stage('Run REST API Tests') {
            steps {
                echo "Testing REST API endpoints..."
                sh '''
                    # Create Python test script for REST API
                    cat > test_rest_api.py << 'EOF'
import requests
import json
import sys
from requests.auth import HTTPBasicAuth

BMC_URL = "http://localhost:8080"
USERNAME = "root"
PASSWORD = "0penBmc"

def test_api_endpoint(endpoint, method="GET", data=None):
    url = f"{BMC_URL}{endpoint}"
    auth = HTTPBasicAuth(USERNAME, PASSWORD)
    
    try:
        if method == "GET":
            response = requests.get(url, auth=auth, timeout=10)
        elif method == "POST":
            response = requests.post(url, auth=auth, json=data, timeout=10)
        
        print(f"Testing {endpoint}: {response.status_code}")
        return response.status_code == 200 or response.status_code == 201
    except Exception as e:
        print(f"Error testing {endpoint}: {e}")
        return False

# Test basic API endpoints
endpoints_to_test = [
    "/xyz/openbmc_project/",
    "/xyz/openbmc_project/enumerate",
    "/xyz/openbmc_project/logging",
    "/xyz/openbmc_project/network",
]

results = {}
for endpoint in endpoints_to_test:
    results[endpoint] = test_api_endpoint(endpoint)

# Write results to JUnit XML
with open("test-results/rest-api-tests.xml", "w") as f:
    f.write('<?xml version="1.0" encoding="UTF-8"?>\\n')
    f.write('<testsuite name="OpenBMC REST API Tests">\\n')
    
    for endpoint, passed in results.items():
        f.write(f'  <testcase name="API {endpoint}" classname="REST-API">\\n')
        if not passed:
            f.write('    <failure message="API endpoint not accessible"/>\\n')
        f.write('  </testcase>\\n')
    
    f.write('</testsuite>\\n')

# Overall result
if all(results.values()):
    print("All API tests passed!")
    sys.exit(0)
else:
    print("Some API tests failed!")
    sys.exit(1)
EOF

                    # Run REST API tests
                    python3 test_rest_api.py
                '''
            }
            post {
                always {
                    junit 'test-results/rest-api-tests.xml'
                }
            }
        }
        
        stage('Run WebUI Functional Tests') {
            steps {
                echo "Running WebUI functional tests..."
                sh '''
                    # Install required packages for WebUI testing
                    pip3 install selenium requests webdriver-manager
                    
                    # Create WebUI test script
                    cat > test_webui.py << 'EOF'
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
import unittest
import xmlrunner
import time
import sys

class OpenBMCWebUITest(unittest.TestCase):
    
    def setUp(self):
        chrome_options = Options()
        chrome_options.add_argument("--headless")
        chrome_options.add_argument("--no-sandbox")
        chrome_options.add_argument("--disable-dev-shm-usage")
        
        self.driver = webdriver.Chrome(options=chrome_options)
        self.driver.get("http://localhost:8080")
        self.wait = WebDriverWait(self.driver, 15)
    
    def test_login_page(self):
        """Test that login page loads correctly"""
        print("Testing login page...")
        self.assertIn("OpenBMC", self.driver.title)
        username_field = self.wait.until(EC.presence_of_element_located((By.ID, "username")))
        password_field = self.driver.find_element(By.ID, "password")
        
        self.assertTrue(username_field.is_displayed())
        self.assertTrue(password_field.is_displayed())
        print("Login page elements found")
    
    def test_successful_login(self):
        """Test successful login"""
        print("Testing login functionality...")
        username_field = self.wait.until(EC.presence_of_element_located((By.ID, "username")))
        password_field = self.driver.find_element(By.ID, "password")
        login_button = self.driver.find_element(By.XPATH, "//button[contains(text(), 'Login')]")
        
        username_field.send_keys("root")
        password_field.send_keys("0penBmc")
        login_button.click()
        
        # Wait for dashboard to load
        time.sleep(5)
        
        # Check if we're redirected to dashboard
        self.assertIn("Dashboard", self.driver.page_source)
        print("Login successful")
    
    def tearDown(self):
        self.driver.quit()

if __name__ == "__main__":
    unittest.main(
        testRunner=xmlrunner.XMLTestRunner(output='test-results'),
        failfast=False, buffer=False, catchbreak=False
    )
EOF

                    # Run WebUI tests
                    python3 test_webui.py
                '''
            }
            post {
                always {
                    junit 'test-results/*.xml'
                    archiveArtifacts 'logs/*.png'
                }
            }
        }
        
        stage('Run Stress Tests') {
            steps {
                echo "Running stress tests..."
                sh '''
                    # Create stress test script
                    cat > stress_test.py << 'EOF'
import requests
import threading
import time
import json
from requests.auth import HTTPBasicAuth

BMC_URL = "http://localhost:8080"
AUTH = HTTPBasicAuth("root", "0penBmc")

def api_stress_worker(worker_id, results):
    """Worker function for API stress testing"""
    successful_requests = 0
    failed_requests = 0
    
    for i in range(50):  # 50 requests per worker
        try:
            response = requests.get(
                f"{BMC_URL}/xyz/openbmc_project/enumerate",
                auth=AUTH,
                timeout=5
            )
            if response.status_code == 200:
                successful_requests += 1
            else:
                failed_requests += 1
        except:
            failed_requests += 1
        
        time.sleep(0.1)  # Small delay between requests
    
    results[worker_id] = {
        "successful": successful_requests,
        "failed": failed_requests
    }

# Run stress test with multiple concurrent workers
threads = []
results = {}

print("Starting stress test with 10 concurrent workers...")
for i in range(10):
    thread = threading.Thread(target=api_stress_worker, args=(i, results))
    threads.append(thread)
    thread.start()

# Wait for all threads to complete
for thread in threads:
    thread.join()

# Calculate totals
total_successful = sum(r["successful"] for r in results.values())
total_failed = sum(r["failed"] for r in results.values())
success_rate = (total_successful / (total_successful + total_failed)) * 100

print(f"Stress test completed: {total_successful} successful, {total_failed} failed")
print(f"Success rate: {success_rate:.2f}%")

# Save results
stress_results = {
    "test_type": "api_stress",
    "concurrent_workers": 10,
    "requests_per_worker": 50,
    "total_requests": total_successful + total_failed,
    "successful_requests": total_successful,
    "failed_requests": total_failed,
    "success_rate": success_rate,
    "status": "completed" if success_rate > 90 else "degraded"
}

with open("test-results/stress-test.json", "w") as f:
    json.dump(stress_results, f, indent=2)

# Exit with error if success rate is too low
if success_rate < 90:
    print("Stress test FAILED: Success rate below 90%")
    exit(1)
else:
    print("Stress test PASSED")
    exit(0)
EOF

                    # Run stress test
                    timeout 300 python3 stress_test.py
                '''
            }
            post {
                always {
                    archiveArtifacts 'test-results/stress-test.json'
                }
            }
        }
        
        stage('System Shutdown') {
            steps {
                echo "Shutting down QEMU instance..."
                sh '''
                    # Gracefully shutdown QEMU
                    if [ -f "${QEMU_PID}" ]; then
                        QEMU_PID_VALUE=$(cat ${QEMU_PID})
                        kill -TERM ${QEMU_PID_VALUE} 2>/dev/null || true
                        sleep 5
                        kill -KILL ${QEMU_PID_VALUE} 2>/dev/null || true
                        rm -f ${QEMU_PID}
                    fi
                '''
            }
        }
    }
    
    post {
        always {
            echo "Cleaning up and archiving artifacts..."
            archiveArtifacts 'test-results/*, logs/*, images/openbmc.qcow2'
        }
        success {
            echo "OpenBMC testing completed SUCCESSFULLY!"
            emailext (
                subject: "SUCCESS: OpenBMC Test Pipeline ${BUILD_NUMBER}",
                body: "All tests passed successfully. Build URL: ${BUILD_URL}",
                to: "team@example.com"
            )
        }
        failure {
            echo "OpenBMC testing FAILED!"
            emailext (
                subject: "FAILED: OpenBMC Test Pipeline ${BUILD_NUMBER}",
                body: "Some tests failed. Please check: ${BUILD_URL}",
                to: "team@example.com"
            )
        }
    }
}
