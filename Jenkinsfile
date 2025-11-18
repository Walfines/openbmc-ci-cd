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
                    
                    qemu-system-arm -m 256 -M romulus-bmc -nographic \\
                        -drive file="$QEMU_FILE",format=raw,if=mtd \\
                        -net nic \\
                        -net user,hostfwd=tcp::2222-:22,hostfwd=tcp::2443-:443 &
                    
                    echo $! > ../qemu.pid
                    echo "QEMU started with PID: $(cat ../qemu.pid)"
                    sleep 120
                '''
            }
        }
        
        stage('Wait for BMC Ready') {
            steps {
                sh '''
                    for i in {1..30}; do
                        if sshpass -p "$BMC_PASSWORD" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 -p $SSH_PORT $BMC_USER@$BMC_IP "echo BMC ready" 2>/dev/null; then
                            echo "BMC ready!"
                            break
                        fi
                        echo "Wait attempt $i/30..."
                        sleep 10
                    done
                '''
            }
        }
        
        stage('Run Auto Tests') {
            steps {
                sh '''
                    sshpass -p "$BMC_PASSWORD" ssh -o StrictHostKeyChecking=no -p $SSH_PORT $BMC_USER@$BMC_IP '
                        echo "=== System Info ==="
                        cat /etc/os-release 2>/dev/null
                        echo "=== Services ==="
                        systemctl list-units --state=running 2>/dev/null | head -10
                    ' > test-results/auto-tests.log
                '''
            }
        }
        
        stage('Run WebUI Tests') {
            steps {
                sh '''
                    cat > webui_test.py << "EOF"
import requests
import urllib3
urllib3.disable_warnings()

BMC_URL = "https://localhost:2443"
endpoints = ["/redfish/v1/", "/"]

print("=== WebUI Tests ===")
for endpoint in endpoints:
    try:
        r = requests.get(f"{BMC_URL}{endpoint}", auth=("root", "0penBmc"), verify=False, timeout=10)
        print(f"{endpoint}: HTTP {r.status_code}")
    except Exception as e:
        print(f"{endpoint}: ERROR {e}")
EOF
                    python3 webui_test.py > test-results/webui-tests.log
                '''
            }
        }
        
        stage('Run Stress Tests') {
            steps {
                sh '''
                    cat > stress_test.py << "EOF"
import requests
import threading
import urllib3
urllib3.disable_warnings()

def worker():
    for i in range(10):
        try:
            requests.get("https://localhost:2443/", auth=("root", "0penBmc"), verify=False, timeout=5)
        except:
            pass

threads = []
for i in range(5):
    t = threading.Thread(target=worker)
    threads.append(t)
    t.start()

for t in threads:
    t.join()
print("Stress test completed")
EOF
                    python3 stress_test.py > test-results/stress-tests.log
                '''
            }
        }
    }
    
    post {
        always {
            sh 'pkill -f qemu-system-arm || true'
            archiveArtifacts artifacts: 'test-results/**'
            echo "Pipeline completed"
        }
    }
}
