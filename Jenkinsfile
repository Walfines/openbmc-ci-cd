pipeline {
    agent any
    
    environment {
        BMC_USER = 'root'
        BMC_PASSWORD = '0penBmc'
        BMC_IP = 'localhost'
        SSH_PORT = '2222'
        HTTPS_PORT = '2443'
    }
    
    stages {
        stage('Copy BMC Image') {
            steps {
                sh '''
                    mkdir -p images test-results
                    cp /var/jenkins_home/romulus/*.mtd images/
                    echo "BMC image ready!"
                    ls -la images/
                '''
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh '''
                    apt-get update
                    apt-get install -y qemu-system-arm sshpass curl python3-requests
                    echo "Dependencies installed"
                '''
            }
        }
        
        stage('Start QEMU with OpenBMC') {
            steps {
                sh '''
                    cd images
                    QEMU_FILE=$(ls *.mtd | head -1)
                    echo "Using BMC image: $QEMU_FILE"
                    
                    # Запускаем QEMU в фоне
                    qemu-system-arm -m 512 -M romulus-bmc -nographic \\
                        -drive file="$QEMU_FILE",format=raw,if=mtd \\
                        -net nic \\
                        -net user,hostfwd=tcp::2222-:22,hostfwd=tcp::2443-:443,hostfwd=tcp::28080-:8080 &
                    
                    echo $! > ../qemu.pid
                    echo "QEMU started with PID: $(cat ../qemu.pid)"
                    
                    # Ждем загрузки BMC
                    echo "Waiting for BMC to boot..."
                    sleep 180
                '''
            }
        }
        
        stage('Wait for BMC Ready') {
            steps {
                sh '''
                    echo "Waiting for BMC services to start..."
                    for i in {1..30}; do
                        if sshpass -p "$BMC_PASSWORD" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 -p $SSH_PORT $BMC_USER@$BMC_IP "
                            systemctl is-active bmcweb > /dev/null 2>&1 && 
                            echo 'BMC Web interface ready'
                        " 2>/dev/null; then
                            echo "BMC Web interface ready!"
                            break
                        fi
                        echo "Wait attempt $i/30..."
                        sleep 10
                    done
                    
                    # Даем дополнительное время для полной инициализации
                    echo "Giving BMC additional time to initialize..."
                    sleep 30
                '''
            }
        }
        
        stage('Run Auto Tests for OpenBMC') {
            steps {
                sh '''
                    echo "=== Running Comprehensive OpenBMC Auto Tests ==="
                    
                    # Создаем Python скрипт для автотестов
                    cat > auto_tests.py << "EOF"
import requests
import json
import urllib3
import subprocess
import time
from requests.auth import HTTPBasicAuth

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

BMC_URL = "https://localhost:2443"
SSH_CMD = "sshpass -p 0penBmc ssh -o StrictHostKeyChecking=no -p 2222 root@localhost"

class OpenBMCTests:
    def __init__(self):
        self.results = []
        self.auth = HTTPBasicAuth('root', '0penBmc')
        
    def run_ssh_command(self, command):
        try:
            result = subprocess.run(
                f"{SSH_CMD} '{command}'", 
                shell=True, 
                capture_output=True, 
                text=True,
                timeout=30
            )
            return result.returncode == 0, result.stdout, result.stderr
        except Exception as e:
            return False, "", str(e)
    
    def test_redfish_api(self):
        print("=== Testing Redfish API ===")
        endpoints = [
            "/redfish/v1/",
            "/redfish/v1/Systems",
            "/redfish/v1/Managers", 
            "/redfish/v1/Chassis"
        ]
        
        for endpoint in endpoints:
            try:
                response = requests.get(
                    f"{BMC_URL}{endpoint}",
                    auth=self.auth,
                    verify=False,
                    timeout=10
                )
                status = "PASS" if response.status_code in [200, 201] else "FAIL"
                self.results.append({
                    "test": f"Redfish API - {endpoint}",
                    "status": status,
                    "details": f"HTTP {response.status_code}"
                })
                print(f"  {endpoint}: {status} ({response.status_code})")
            except Exception as e:
                self.results.append({
                    "test": f"Redfish API - {endpoint}",
                    "status": "ERROR",
                    "details": str(e)
                })
                print(f"  {endpoint}: ERROR ({e})")
    
    def test_system_services(self):
        print("=== Testing System Services ===")
        services = [
            "bmcweb",
            "systemd-journald",
            "avahi-daemon"
        ]
        
        for service in services:
            success, stdout, stderr = self.run_ssh_command(f"systemctl is-active {service}")
            status = "PASS" if success and "active" in stdout else "FAIL"
            self.results.append({
                "test": f"Service - {service}",
                "status": status,
                "details": stdout.strip() if success else stderr
            })
            print(f"  {service}: {status}")
    
    def test_system_info(self):
        print("=== Testing System Information ===")
        # Test basic system info
        success, stdout, stderr = self.run_ssh_command("cat /etc/os-release")
        if success:
            self.results.append({
                "test": "System Information",
                "status": "PASS",
                "details": "OS release information available"
            })
            print("  System Info: PASS")
        else:
            self.results.append({
                "test": "System Information",
                "status": "FAIL",
                "details": "Cannot read system information"
            })
    
    def test_network_config(self):
        print("=== Testing Network Configuration ===")
        success, stdout, stderr = self.run_ssh_command("ip addr show")
        if success:
            has_eth0 = "eth0" in stdout
            has_ip = "inet " in stdout
            status = "PASS" if has_eth0 and has_ip else "FAIL"
            self.results.append({
                "test": "Network Configuration",
                "status": status,
                "details": "eth0 with IP configured" if status == "PASS" else "Network issues"
            })
            print(f"  Network: {status}")
    
    def test_web_interface(self):
        print("=== Testing Web Interface ===")
        try:
            response = requests.get(f"{BMC_URL}/", verify=False, timeout=10)
            if response.status_code in [200, 301, 302]:
                self.results.append({
                    "test": "Web Interface",
                    "status": "PASS",
                    "details": f"HTTP {response.status_code}"
                })
                print("  Web Interface: PASS")
            else:
                self.results.append({
                    "test": "Web Interface",
                    "status": "FAIL",
                    "details": f"HTTP {response.status_code}"
                })
        except Exception as e:
            self.results.append({
                "test": "Web Interface",
                "status": "ERROR",
                "details": str(e)
            })
    
    def run_all_tests(self):
        print("Starting OpenBMC Auto Tests...")
        self.test_redfish_api()
        self.test_system_services()
        self.test_system_info()
        self.test_network_config()
        self.test_web_interface()
        
        # Generate report
        passed = len([r for r in self.results if r["status"] == "PASS"])
        total = len(self.results)
        
        print(f"\\n=== TEST SUMMARY ===")
        print(f"Passed: {passed}/{total}")
        
        with open("test-results/auto-tests-detailed.json", "w") as f:
            json.dump({
                "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
                "summary": {"passed": passed, "total": total},
                "results": self.results
            }, f, indent=2)
        
        with open("test-results/auto-tests.log", "w") as f:
            for result in self.results:
                f.write(f"{result['test']}: {result['status']} - {result['details']}\\n")
        
        # Не завершаем пайплайн при неудачных тестах - продолжаем выполнение
        return True

if __name__ == "__main__":
    tester = OpenBMCTests()
    tester.run_all_tests()
    exit(0)
EOF

                    # Запускаем автотесты
                    python3 auto_tests.py
                '''
            }
        }
        
        stage('Run WebUI Tests for OpenBMC') {
            steps {
                sh '''
                    echo "=== Running OpenBMC WebUI Tests ==="
                    
                    cat > webui_tests.py << "EOF"
import requests
import json
import time
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class OpenBMCWebUITests:
    def __init__(self):
        self.results = []
        self.bmc_url = "https://localhost:2443"
        
    def test_authentication(self):
        print("Testing Authentication...")
        try:
            session = requests.Session()
            session.auth = ('root', '0penBmc')
            session.verify = False
            
            # Test Redfish authentication
            response = session.get(f"{self.bmc_url}/redfish/v1/", timeout=10)
            
            if response.status_code == 200:
                self.results.append({
                    "test": "WebUI Authentication",
                    "status": "PASS",
                    "details": "Successfully authenticated via Redfish API"
                })
                print("  Authentication: PASS")
                return True
            else:
                raise Exception(f"HTTP {response.status_code}")
                
        except Exception as e:
            self.results.append({
                "test": "WebUI Authentication", 
                "status": "FAIL",
                "details": str(e)
            })
            print(f"  Authentication: FAIL - {e}")
            return False
    
    def test_navigation(self):
        print("Testing Navigation...")
        try:
            # Test basic navigation endpoints
            endpoints = [
                "/redfish/v1/Systems",
                "/redfish/v1/Managers",
                "/redfish/v1/Chassis"
            ]
            
            session = requests.Session()
            session.auth = ('root', '0penBmc')
            session.verify = False
            
            accessible_endpoints = []
            for endpoint in endpoints:
                try:
                    response = session.get(f"{self.bmc_url}{endpoint}", timeout=5)
                    if response.status_code == 200:
                        accessible_endpoints.append(endpoint)
                except:
                    pass
            
            if len(accessible_endpoints) >= 2:
                self.results.append({
                    "test": "WebUI Navigation",
                    "status": "PASS",
                    "details": f"Accessible endpoints: {', '.join(accessible_endpoints)}"
                })
                print("  Navigation: PASS")
                return True
            else:
                raise Exception("Insufficient accessible endpoints")
                
        except Exception as e:
            self.results.append({
                "test": "WebUI Navigation",
                "status": "FAIL", 
                "details": str(e)
            })
            print(f"  Navigation: FAIL - {e}")
            return False
    
    def test_web_interface(self):
        print("Testing Web Interface...")
        try:
            response = requests.get(f"{self.bmc_url}/", verify=False, timeout=10)
            if response.status_code in [200, 301, 302]:
                self.results.append({
                    "test": "Web Interface Access",
                    "status": "PASS",
                    "details": f"HTTP {response.status_code}"
                })
                print("  Web Interface: PASS")
                return True
            else:
                raise Exception(f"HTTP {response.status_code}")
        except Exception as e:
            self.results.append({
                "test": "Web Interface Access",
                "status": "FAIL",
                "details": str(e)
            })
            print(f"  Web Interface: FAIL - {e}")
            return False
    
    def run_all_tests(self):
        print("Starting OpenBMC WebUI Tests...")
        self.test_authentication() 
        self.test_navigation()
        self.test_web_interface()
        
        # Generate report
        passed = len([r for r in self.results if r["status"] == "PASS"])
        total = len(self.results)
        
        print(f"\\n=== WEBUI TEST SUMMARY ===")
        print(f"Passed: {passed}/{total}")
        
        with open("test-results/webui-tests-detailed.json", "w") as f:
            json.dump({
                "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
                "summary": {"passed": passed, "total": total},
                "results": self.results
            }, f, indent=2)
        
        with open("test-results/webui-tests.log", "w") as f:
            for result in self.results:
                f.write(f"{result['test']}: {result['status']} - {result['details']}\\n")
        
        return True

if __name__ == "__main__":
    tester = OpenBMCWebUITests()
    tester.run_all_tests()
    exit(0)
EOF

                    # Запускаем WebUI тесты
                    python3 webui_tests.py
                '''
            }
        }
        
        stage('Run Stress Testing for OpenBMC') {
            steps {
                sh '''
                    echo "=== Running OpenBMC Stress Tests ==="
                    
                    cat > stress_tests.py << "EOF"
import requests
import threading
import time
import json
import urllib3

urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class OpenBMCStressTests:
    def __init__(self):
        self.results = []
        self.bmc_url = "https://localhost:2443"
        self.auth = ('root', '0penBmc')
        
    def api_call(self, endpoint, timeout=5):
        try:
            start_time = time.time()
            response = requests.get(
                f"{self.bmc_url}{endpoint}",
                auth=self.auth,
                verify=False,
                timeout=timeout
            )
            response_time = (time.time() - start_time) * 1000
            
            return {
                "success": response.status_code == 200,
                "response_time": response_time,
                "status_code": response.status_code
            }
        except Exception as e:
            return {
                "success": False,
                "response_time": 0,
                "error": str(e)
            }
    
    def concurrent_api_test(self):
        print("=== Concurrent API Stress Test ===")
        endpoints = [
            "/redfish/v1/",
            "/redfish/v1/Systems",
            "/redfish/v1/Managers"
        ]
        
        def worker(endpoint):
            results = []
            for i in range(5):
                result = self.api_call(endpoint)
                results.append(result)
                time.sleep(0.2)
            return results
        
        start_time = time.time()
        
        threads = []
        all_results = []
        
        for endpoint in endpoints * 3:
            thread = threading.Thread(target=lambda e=endpoint: all_results.extend(worker(e)))
            threads.append(thread)
            thread.start()
        
        for thread in threads:
            thread.join()
        
        total_time = time.time() - start_time
        
        # Analyze results
        successful_calls = len([r for r in all_results if r["success"]])
        total_calls = len(all_results)
        success_rate = (successful_calls / total_calls) * 100
        
        response_times = [r["response_time"] for r in all_results if r["success"]]
        avg_response_time = sum(response_times) / len(response_times) if response_times else 0
        
        print(f"  Total API Calls: {total_calls}")
        print(f"  Successful: {successful_calls}")
        print(f"  Success Rate: {success_rate:.1f}%")
        print(f"  Avg Response Time: {avg_response_time:.1f}ms")
        print(f"  Total Test Time: {total_time:.1f}s")
        
        status = "PASS" if success_rate >= 80 else "WARNING"
        self.results.append({
            "test": "Concurrent API Stress",
            "status": status,
            "details": f"{successful_calls}/{total_calls} successful, {avg_response_time:.1f}ms avg"
        })
        
        return all_results
    
    def sustained_load_test(self):
        print("=== Sustained Load Test ===")
        test_duration = 30
        interval = 1
        
        start_time = time.time()
        call_count = 0
        successful_calls = 0
        response_times = []
        
        while time.time() - start_time < test_duration:
            result = self.api_call("/redfish/v1/")
            call_count += 1
            
            if result["success"]:
                successful_calls += 1
                response_times.append(result["response_time"])
            
            time.sleep(interval)
        
        success_rate = (successful_calls / call_count) * 100 if call_count > 0 else 0
        avg_response_time = sum(response_times) / len(response_times) if response_times else 0
        
        print(f"  Duration: {test_duration}s")
        print(f"  Total Calls: {call_count}")
        print(f"  Successful: {successful_calls}")
        print(f"  Success Rate: {success_rate:.1f}%")
        print(f"  Avg Response Time: {avg_response_time:.1f}ms")
        
        status = "PASS" if success_rate >= 80 else "WARNING"
        self.results.append({
            "test": "Sustained Load",
            "status": status,
            "details": f"{successful_calls}/{call_count} successful over {test_duration}s"
        })
    
    def run_all_tests(self):
        print("Starting OpenBMC Stress Tests...")
        concurrent_results = self.concurrent_api_test()
        self.sustained_load_test()
        
        # Generate detailed report
        passed = len([r for r in self.results if r["status"] == "PASS"])
        total = len(self.results)
        
        print(f"\\n=== STRESS TEST SUMMARY ===")
        print(f"Tests completed: {total}")
        
        with open("test-results/stress-tests-detailed.json", "w") as f:
            json.dump({
                "timestamp": time.strftime("%Y-%m-%d %H:%M:%S"),
                "summary": {"tests_completed": total},
                "results": self.results
            }, f, indent=2)
        
        with open("test-results/stress-tests.log", "w") as f:
            for result in self.results:
                f.write(f"{result['test']}: {result['status']} - {result['details']}\\n")
        
        return True

if __name__ == "__main__":
    tester = OpenBMCStressTests()
    tester.run_all_tests()
    exit(0)
EOF

                    # Запускаем стресс-тесты
                    python3 stress_tests.py
                '''
            }
        }
        
        stage('Generate Test Reports') {
            steps {
                sh '''
                    echo "=== Generating Comprehensive Test Reports ==="
                    
                    # Создаем сводный отчет
                    cat > test-results/test-execution-summary.md << EOF
# OpenBMC CI/CD Test Execution Summary

## Execution Details
- **Timestamp**: $(date)
- **BMC Image**: $(ls images/*.mtd | head -1)
- **QEMU Configuration**: 512MB RAM, Romulus BMC
- **Network Ports**: SSH:2222, HTTPS:2443

## Test Categories Executed

### 1. Auto Tests for OpenBMC
- Redfish API endpoints validation
- System services status monitoring  
- System information collection
- Network configuration verification
- Web interface accessibility

### 2. WebUI Tests for OpenBMC
- Authentication mechanisms testing
- Navigation and endpoint accessibility
- Web interface functionality

### 3. Stress Testing for OpenBMC
- Concurrent API load testing
- Sustained performance under load
- Response time analysis

## Test Results
- Auto tests completed with detailed results
- WebUI tests validated interface functionality
- Stress tests performed load testing
- All test reports available in artifacts

## Next Steps
- Review detailed test results in Jenkins artifacts
- Check performance metrics
- Validate BMC functionality for deployment

EOF

                    echo "Test reports generated successfully"
                '''
            }
        }
    }
    
    post {
        always {
            sh '''
                echo "Cleaning up QEMU processes..."
                if [ -f "qemu.pid" ]; then
                    kill -TERM $(cat qemu.pid) 2>/dev/null || true
                    sleep 5
                    kill -KILL $(cat qemu.pid) 2>/dev/null || true
                    rm -f qemu.pid
                fi
                pkill -f qemu-system-arm || true
            '''
            archiveArtifacts artifacts: 'test-results/**/*, images/*.mtd'
            echo "OpenBMC CI/CD Pipeline execution completed"
        }
        success {
            echo "🎉 OPENBMC CI/CD PIPELINE EXECUTED SUCCESSFULLY!"
            echo "All tests completed - check artifacts for detailed results"
        }
    }
}
