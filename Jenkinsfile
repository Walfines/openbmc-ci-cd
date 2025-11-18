pipeline {
    agent any
    
    environment {
        BMC_USER = 'root'
        BMC_PASSWORD = '0penBmc'
        BMC_IP = 'localhost'
        SSH_PORT = '2222'
        HTTPS_PORT = '2443'
        QEMU_PID = 'qemu.pid'
        BMC_IMAGE_PATH = '/home/ubuntu/Desktop/romulus/obmc-phosphor-image-romulus-20250902012112.static.mtd'
    }
    
    stages {
        stage('Verify BMC Image') {
            steps {
                echo "Checking BMC image..."
                sh '''
                    echo "BMC Image path: /home/ubuntu/Desktop/romulus/obmc-phosphor-image-romulus-20250902012112.static.mtd"
                    if [ -f "/home/ubuntu/Desktop/romulus/obmc-phosphor-image-romulus-20250902012112.static.mtd" ]; then
                        echo "Real BMC image found!"
                        ls -lh "/home/ubuntu/Desktop/romulus/obmc-phosphor-image-romulus-20250902012112.static.mtd"
                        echo "Image size: $(du -h "/home/ubuntu/Desktop/romulus/obmc-phosphor-image-romulus-20250902012112.static.mtd" | cut -f1)"
                    else
                        echo "BMC image not found"
                        ls -la /home/ubuntu/Desktop/romulus/ || echo "Directory not found"
                        exit 1
                    fi
                '''
            }
        }
        
        stage('Start QEMU with Real BMC') {
            steps {
                echo "Starting QEMU with Romulus BMC..."
                sh '''
                    qemu-system-arm -m 256 -M romulus-bmc -nographic -drive file=/home/ubuntu/Desktop/romulus/obmc-phosphor-image-romulus-20250902012112.static.mtd,format=raw,if=mtd -net nic -net user,hostfwd=:0.0.0.0:2222-:22,hostfwd=:0.0.0.0:2443-:443,hostfwd=udp:0.0.0.0:2623-:623,hostname=qemu &
                    
                    echo $! > qemu.pid
                    echo "QEMU started with PID: $(cat qemu.pid)"
                    echo "Ports: SSH: 2222, HTTPS: 2443"
                    
                    echo "Waiting for BMC to boot (90 seconds)..."
                    sleep 90
                '''
            }
        }
        
        stage('Wait for BMC Ready') {
            steps {
                echo "Waiting for BMC services..."
                sh '''
                    echo "Waiting for BMC SSH service..."
                    MAX_RETRIES=30
                    for i in $(seq 1 ${MAX_RETRIES}); do
                        if sshpass -p '0penBmc' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 -p 2222 root@localhost 'echo BMC is alive' 2>/dev/null; then
                            echo "BMC is ready and responsive!"
                            break
                        fi
                        echo "Attempt ${i}/${MAX_RETRIES}: BMC not ready yet..."
                        if [ ${i} -eq ${MAX_RETRIES} ]; then
                            echo "BMC failed to start within expected time"
                            exit 1
                        fi
                        sleep 10
                    done
                '''
            }
        }
        
        stage('Real BMC Health Tests') {
            steps {
                echo "Running real BMC health tests..."
                sh '''
                    mkdir -p test-results
                    
                    echo "Testing BMC System Information"
                    sshpass -p '0penBmc' ssh -o StrictHostKeyChecking=no -p 2222 root@localhost '
                        echo "BMC Version"
                        cat /etc/os-release 2>/dev/null || echo "No os-release file"
                        echo ""
                        echo "System Uptime"
                        uptime
                        echo ""
                        echo "Memory Usage"
                        free -m || cat /proc/meminfo | head -5
                        echo ""
                        echo "Storage"
                        df -h 2>/dev/null || echo "df not available"
                        echo ""
                        echo "Running Processes"
                        ps aux | head -10
                    ' > test-results/bmc-system-info.log
                    
                    echo "Testing BMC Services"
                    sshpass -p '0penBmc' ssh -o StrictHostKeyChecking=no -p 2222 root@localhost '
                        echo "BMC Services Status"
                        systemctl list-units --state=running 2>/dev/null | grep -E "(phosphor|openbmc|redfish|web|ssh)" | head -20
                        echo ""
                        echo "Network Interfaces"
                        ip addr show 2>/dev/null || ifconfig 2>/dev/null || echo "Network tools not available"
                    ' > test-results/bmc-services.log
                    
                    echo "Checking BMC Logs"
                    sshpass -p '0penBmc' ssh -o StrictHostKeyChecking=no -p 2222 root@localhost '
                        echo "Recent Journal Logs"
                        journalctl --no-pager -n 30 2>/dev/null || dmesg | tail -30 2>/dev/null || echo "Logs not available"
                    ' > test-results/bmc-logs.log
                    
                    echo "Health tests completed"
                '''
            }
        }
        
        stage('REST API Tests') {
            steps {
                echo "Testing BMC REST API..."
                sh '''
                    cat > test_bmc_rest.py << "ENDFILE"
import requests
import json
import sys
import urllib3
from requests.auth import HTTPBasicAuth

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

BMC_URL = "https://localhost:2443"
USERNAME = "root"
PASSWORD = "0penBmc"

def test_rest_endpoint(endpoint):
    try:
        response = requests.get(
            f"{BMC_URL}{endpoint}",
            auth=HTTPBasicAuth(USERNAME, PASSWORD),
            verify=False,
            timeout=15
        )
        print(f"Testing {endpoint}: Status {response.status_code}")
        if response.status_code == 200:
            try:
                data = response.json()
                return True, f"Success: {len(data)} keys in response"
            except:
                return True, "Success: Response received"
        return False, f"Failed: HTTP {response.status_code}"
    except Exception as e:
        return False, f"Error: {str(e)}"

endpoints = [
    "/redfish/v1/",
    "/redfish/v1/Managers/",
    "/redfish/v1/Systems/",
    "/redfish/v1/Chassis/",
    "/xyz/openbmc_project/",
    "/org/open_power/",
]

print("Starting BMC REST API Tests...")
results = {}

for endpoint in endpoints:
    success, message = test_rest_endpoint(endpoint)
    results[endpoint] = {"success": success, "message": message}
    print(f"{endpoint}: {message}")

with open("test-results/rest-api-detailed.json", "w") as f:
    json.dump(results, f, indent=2)

with open("test-results/bmc-rest-api-tests.xml", "w") as f:
    f.write('<?xml version="1.0" encoding="UTF-8"?>\\n')
    f.write('<testsuite name="BMC REST API Tests" tests="{}">\\n'.format(len(endpoints)))
    
    success_count = 0
    for endpoint, result in results.items():
        f.write('  <testcase name="REST {}" classname="BMC-API">\\n'.format(endpoint))
        if not result["success"]:
            f.write('    <failure message="{}"/>\\n'.format(result["message"]))
        else:
            success_count += 1
        f.write('  </testcase>\\n')
    
    f.write('</testsuite>\\n')

success_rate = (success_count / len(endpoints)) * 100
print(f"REST API Test Results: {success_count}/{len(endpoints)} passed ({success_rate:.1f}%)")

if success_rate >= 50:
    print("REST API tests: ACCEPTABLE")
    sys.exit(0)
else:
    print("REST API tests: UNACCEPTABLE")
    sys.exit(1)
ENDFILE

                    python3 test_bmc_rest.py
                '''
            }
            post {
                always {
                    junit 'test-results/bmc-rest-api-tests.xml'
                    archiveArtifacts 'test-results/rest-api-detailed.json'
                }
            }
        }
        
        stage('BMC Functional Tests') {
            steps {
                echo "Running functional tests..."
                sh '''
                    sshpass -p '0penBmc' ssh -o StrictHostKeyChecking=no -p 2222 root@localhost '
                        echo "BMC Specific Commands"
                        echo "Available BMC commands:"
                        which busctl 2>/dev/null && echo "busctl: available" || echo "busctl: not available"
                        which obmcutil 2>/dev/null && echo "obmcutil: available" || echo "obmcutil: not available"
                        
                        if which busctl >/dev/null 2>&1; then
                            echo "Busctl Services"
                            busctl list --no-pager | grep -i openbmc | head -10
                        fi
                        
                        echo "Firmware Information"
                        cat /etc/version 2>/dev/null || echo "No version file"
                        
                        echo "Hostname"
                        hostname
                        
                        echo "Functional tests completed"
                    ' > test-results/bmc-functional-tests.log
                    
                    echo "Functional tests completed"
                '''
            }
            post {
                always {
                    archiveArtifacts 'test-results/bmc-*.log'
                }
            }
        }
        
        stage('Generate Test Report') {
            steps {
                echo "Generating test report..."
                sh '''
                    cat > test-results/test-summary.md << "ENDFILE"
# BMC Test Report
## Test Results
- BMC Image: /home/ubuntu/Desktop/romulus/obmc-phosphor-image-romulus-20250902012112.static.mtd
- SSH Port: 2222
- HTTPS Port: 2443
- Test Time: $(date)

## System Information
$(cat test-results/bmc-system-info.log | head -20)

## Services Status
$(cat test-results/bmc-services.log | head -15)

## Test Artifacts
- System Info: bmc-system-info.log
- Services: bmc-services.log
- Logs: bmc-logs.log
- Functional Tests: bmc-functional-tests.log
- REST API: rest-api-detailed.json
ENDFILE

                    echo "Test report generated"
                '''
            }
            post {
                always {
                    archiveArtifacts 'test-results/test-summary.md'
                }
            }
        }
        
        stage('Cleanup') {
            steps {
                echo "Cleaning up QEMU..."
                sh '''
                    if [ -f "qemu.pid" ]; then
                        echo "Stopping QEMU process $(cat qemu.pid)"
                        kill -TERM $(cat qemu.pid) 2>/dev/null || true
                        sleep 5
                        kill -KILL $(cat qemu.pid) 2>/dev/null || true
                        rm -f qemu.pid
                        echo "QEMU stopped"
                    fi
                    pkill -f qemu-system || true
                '''
            }
        }
    }
    
    post {
        always {
            echo "BMC Testing Pipeline completed"
            archiveArtifacts 'test-results/*, qemu.pid'
        }
        success {
            echo "REAL BMC TESTS COMPLETED SUCCESSFULLY!"
        }
        failure {
            echo "BMC tests failed - check logs for details"
        }
    }
}
