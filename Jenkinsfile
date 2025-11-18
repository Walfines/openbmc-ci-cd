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
                sh """
                    echo "BMC Image path: ${BMC_IMAGE_PATH}"
                    if [ -f "${BMC_IMAGE_PATH}" ]; then
                        echo "✅ Real BMC image found!"
                        ls -lh "${BMC_IMAGE_PATH}"
                        echo "Image size: $(du -h "${BMC_IMAGE_PATH}" | cut -f1)"
                    else
                        echo "❌ BMC image not found at ${BMC_IMAGE_PATH}"
                        echo "Available files in /home/ubuntu/Desktop/romulus/:"
                        ls -la /home/ubuntu/Desktop/romulus/ || echo "Directory not found"
                        exit 1
                    fi
                """
            }
        }
        
        stage('Start QEMU with Real BMC') {
            steps {
                echo "Starting QEMU with Romulus BMC..."
                sh """
                    # Используем ВАШУ реальную команду запуска
                    qemu-system-arm \\
                        -m 256 \\
                        -M romulus-bmc \\
                        -nographic \\
                        -drive file=${BMC_IMAGE_PATH},format=raw,if=mtd \\
                        -net nic \\
                        -net user,hostfwd=:0.0.0.0:${SSH_PORT}-:22,hostfwd=:0.0.0.0:${HTTPS_PORT}-:443,hostfwd=udp:0.0.0.0:2623-:623,hostname=qemu &
                    
                    echo \$! > ${QEMU_PID}
                    echo "✅ QEMU started with PID: \$(cat ${QEMU_PID})"
                    echo "📡 Ports:"
                    echo "  - SSH: ${SSH_PORT} → 22"
                    echo "  - HTTPS: ${HTTPS_PORT} → 443"
                    
                    # Ждем загрузки BMC (обычно 1-2 минуты)
                    echo "⏳ Waiting for BMC to boot (90 seconds)..."
                    sleep 90
                """
            }
        }
        
        stage('Wait for BMC Ready') {
            steps {
                echo "Waiting for BMC services..."
                sh """
                    # Ждем пока BMC станет доступен через SSH
                    echo "⏳ Waiting for BMC SSH service..."
                    MAX_RETRIES=30
                    for i in \$(seq 1 \${MAX_RETRIES}); do
                        if sshpass -p '${BMC_PASSWORD}' ssh -o StrictHostKeyChecking=no -o ConnectTimeout=10 -p ${SSH_PORT} ${BMC_USER}@${BMC_IP} 'echo BMC is alive' 2>/dev/null; then
                            echo "✅ BMC is ready and responsive!"
                            break
                        fi
                        echo "Attempt \$i/\${MAX_RETRIES}: BMC not ready yet..."
                        if [ \$i -eq \${MAX_RETRIES} ]; then
                            echo "❌ BMC failed to start within expected time"
                            exit 1
                        fi
                        sleep 10
                    done
                """
            }
        }
        
        stage('Real BMC Health Tests') {
            steps {
                echo "Running real BMC health tests..."
                sh """
                    mkdir -p test-results
                    
                    # Тест 1: Базовая информация о системе
                    echo "=== Testing BMC System Information ==="
                    sshpass -p '${BMC_PASSWORD}' ssh -o StrictHostKeyChecking=no -p ${SSH_PORT} ${BMC_USER}@${BMC_IP} '
                        echo "=== BMC Version ==="
                        cat /etc/os-release 2>/dev/null || echo "No os-release file"
                        echo ""
                        echo "=== System Uptime ==="
                        uptime
                        echo ""
                        echo "=== Memory Usage ==="
                        free -m || cat /proc/meminfo | head -5
                        echo ""
                        echo "=== Storage ==="
                        df -h 2>/dev/null || echo "df not available"
                        echo ""
                        echo "=== Running Processes ==="
                        ps aux | head -10
                    ' > test-results/bmc-system-info.log
                    
                    # Тест 2: Проверка сервисов BMC
                    echo "=== Testing BMC Services ==="
                    sshpass -p '${BMC_PASSWORD}' ssh -o StrictHostKeyChecking=no -p ${SSH_PORT} ${BMC_USER}@${BMC_IP} '
                        echo "=== BMC Services Status ==="
                        systemctl list-units --state=running 2>/dev/null | grep -E "(phosphor|openbmc|redfish|web|ssh)" | head -20
                        echo ""
                        echo "=== Network Interfaces ==="
                        ip addr show 2>/dev/null || ifconfig 2>/dev/null || echo "Network tools not available"
                    ' > test-results/bmc-services.log
                    
                    # Тест 3: Проверка журналов
                    echo "=== Checking BMC Logs ==="
                    sshpass -p '${BMC_PASSWORD}' ssh -o StrictHostKeyChecking=no -p ${SSH_PORT} ${BMC_USER}@${BMC_IP} '
                        echo "=== Recent Journal Logs ==="
                        journalctl --no-pager -n 30 2>/dev/null || dmesg | tail -30 2>/dev/null || echo "Logs not available"
                    ' > test-results/bmc-logs.log
                    
                    echo "✅ Health tests completed"
                """
            }
        }
        
        stage('REST API Tests') {
            steps {
                echo "Testing BMC REST API..."
                sh """
                    # Тестируем REST API через HTTPS
                    echo "=== Testing REST API Endpoints ==="
                    
                    # Создаем Python скрипт для тестирования API
                    cat > test_bmc_rest.py << 'EOF'
import requests
import json
import sys
import urllib3
from requests.auth import HTTPBasicAuth

# Отключаем предупреждения о SSL (для самоподписанных сертификатов)
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

BMC_URL = "https://localhost:2443"
USERNAME = "root"
PASSWORD = "0penBmc"

def test_rest_endpoint(endpoint):
    try:
        response = requests.get(
            f"{BMC_URL}{endpoint}",
            auth=HTTPBasicAuth(USERNAME, PASSWORD),
            verify=False,  # Игнорируем SSL ошибки
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

# Тестируем основные REST endpoints
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

# Сохраняем детальные результаты
with open("test-results/rest-api-detailed.json", "w") as f:
    json.dump(results, f, indent=2)

# Создаем JUnit отчет
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
print(f"\\\\n📊 REST API Test Results: {success_count}/{len(endpoints)} passed ({success_rate:.1f}%)")

if success_rate >= 50:
    print("✅ REST API tests: ACCEPTABLE")
    sys.exit(0)
else:
    print("❌ REST API tests: UNACCEPTABLE")
    sys.exit(1)
EOF

                    # Запускаем тесты REST API
                    python3 test_bmc_rest.py
                """
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
                sh """
                    # Дополнительные функциональные тесты
                    echo "=== Running Functional Tests ==="
                    
                    sshpass -p '${BMC_PASSWORD}' ssh -o StrictHostKeyChecking=no -p ${SSH_PORT} ${BMC_USER}@${BMC_IP} '
                        # Тест команд BMC
                        echo "=== BMC Specific Commands ==="
                        
                        # Проверка доступных команд
                        echo "Available BMC commands:"
                        which busctl 2>/dev/null && echo "busctl: available" || echo "busctl: not available"
                        which obmcutil 2>/dev/null && echo "obmcutil: available" || echo "obmcutil: not available"
                        
                        # Проверка состояния через busctl (если доступно)
                        if which busctl >/dev/null 2>&1; then
                            echo "=== Busctl Services ==="
                            busctl list --no-pager | grep -i openbmc | head -10
                        fi
                        
                        # Проверка версии firmware
                        echo "=== Firmware Information ==="
                        cat /etc/version 2>/dev/null || echo "No version file"
                        
                        # Проверка хоста
                        echo "=== Hostname ==="
                        hostname
                        
                        echo "=== Functional tests completed ==="
                    ' > test-results/bmc-functional-tests.log
                    
                    echo "✅ Functional tests completed"
                """
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
                sh """
                    # Создаем сводный отчет
                    cat > test-results/test-summary.md << EOF
# BMC Test Report
## Test Results
- BMC Image: ${BMC_IMAGE_PATH}
- SSH Port: ${SSH_PORT}
- HTTPS Port: ${HTTPS_PORT}
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
EOF

                    echo "✅ Test report generated"
                """
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
                sh """
                    if [ -f "${QEMU_PID}" ]; then
                        echo "Stopping QEMU process \$(cat ${QEMU_PID})"
                        kill -TERM \$(cat ${QEMU_PID}) 2>/dev/null || true
                        sleep 5
                        kill -KILL \$(cat ${QEMU_PID}) 2>/dev/null || true
                        rm -f ${QEMU_PID}
                        echo "✅ QEMU stopped"
                    fi
                    pkill -f qemu-system || true
                """
            }
        }
    }
    
    post {
        always {
            echo "BMC Testing Pipeline completed"
            archiveArtifacts 'test-results/*, ${QEMU_PID}'
        }
        success {
            echo "🎉 REAL BMC TESTS COMPLETED SUCCESSFULLY!"
        }
        failure {
            echo "💥 BMC tests failed - check logs for details"
        }
    }
}
