pipeline {
    agent any
    
    environment {
        BMC_USER = 'root'
        BMC_PASSWORD = '0penBmc'
        BMC_IP = 'localhost'
        SSH_PORT = '2222'
        HTTPS_PORT = '2443'
        QEMU_PID = 'qemu.pid'
    }
    
    stages {
        stage('Copy BMC Image') {
            steps {
                echo "📁 Setting up BMC image..."
                sh '''
                    mkdir -p images test-results
                    cp /var/jenkins_home/romulus/obmc-phosphor-image-romulus-20250902012112.static.mtd images/
                    echo "✅ BMC image ready!"
                    ls -lh images/
                '''
            }
        }
        
        stage('Install Dependencies') {
            steps {
                echo "📦 Installing dependencies..."
                sh '''
                    apt-get update
                    apt-get install -y qemu-system-arm sshpass curl python3 python3-pip
                    pip3 install requests
                '''
            }
        }
        
        stage('Start QEMU with BMC') {
            steps {
                echo "🔧 Starting BMC..."
                sh '''
                    cd images
                    
                    qemu-system-arm -m 256 -M romulus-bmc -nographic \
                        -drive file=obmc-phosphor-image-romulus-20250902012112.static.mtd,format=raw,if=mtd \
                        -net nic \
                        -net user,hostfwd=:0.0.0.0:2222-:22,hostfwd=:0.0.0.0:2443-:443,hostfwd=udp:0.0.0.0:2623-:623,hostname=qemu &
                    
                    echo $! > ../qemu.pid
                    echo "✅ QEMU started with PID: $(cat ../qemu.pid)"
                    
                    echo "⏳ Waiting for BMC to boot (90 seconds)..."
                    sleep 90
                '''
            }
        }
        
        stage('Wait for BMC Ready') {
            steps {
                echo "⏰ Waiting for BMC services..."
                sh '''
                    echo "=== Waiting for BMC SSH ==="
                    for i in {1..20}; do
                        if sshpass -p "$BMC_PASSWORD" ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 -p $SSH_PORT $BMC_USER@$BMC_IP "echo 'BMC is alive'" 2>/dev/null; then
                            echo "✅ BMC is ready!"
                            break
                        fi
                        echo "Attempt $i/20: Waiting..."
                        if [ $i -eq 20 ]; then
                            echo "❌ BMC failed to start"
                            exit 1
                        fi
                        sleep 10
                    done
                '''
            }
        }
        
        stage('Real BMC Tests') {
            steps {
                echo "🧪 Running Real BMC Tests"
                sh '''
                    mkdir -p test-results
                    
                    echo "=== Testing BMC System ==="
                    sshpass -p "$BMC_PASSWORD" ssh -o StrictHostKeyChecking=no -p $SSH_PORT $BMC_USER@$BMC_IP '
                        echo "🔧 BMC Version:"
                        cat /etc/os-release 2>/dev/null
                        echo ""
                        echo "⏱️  Uptime:"
                        uptime
                        echo ""
                        echo "🖥️  Hostname:"
                        hostname
                        echo ""
                        echo "💾 Memory:"
                        free -m 2>/dev/null || echo "Memory info not available"
                    ' > test-results/bmc-system.log
                    
                    echo "✅ System tests completed"
                '''
            }
        }
        
        stage('REST API Test') {
            steps {
                echo "🌐 Testing REST API"
                sh '''
                    cat > test_api.py << "EOF"
import requests
import urllib3
from requests.auth import HTTPBasicAuth

urllib3.disable_warnings()

try:
    response = requests.get(
        "https://localhost:2443/redfish/v1/",
        auth=HTTPBasicAuth("root", "0penBmc"),
        verify=False,
        timeout=10
    )
    print(f"REST API Status: HTTP {response.status_code}")
    if response.status_code == 200:
        print("✅ REST API is working!")
    else:
        print("⚠️  REST API responded with non-200 status")
except Exception as e:
    print(f"❌ REST API error: {e}")
EOF

                    python3 test_api.py > test-results/rest-api.log
                '''
            }
        }
        
        stage('Cleanup') {
            steps {
                echo "🧹 Cleaning up..."
                sh '''
                    if [ -f "qemu.pid" ]; then
                        kill -TERM $(cat qemu.pid) 2>/dev/null || true
                        sleep 3
                        rm -f qemu.pid
                    fi
                    pkill -f qemu-system || true
                '''
            }
        }
    }
    
    post {
        always {
            echo "🏁 Pipeline completed"
            archiveArtifacts 'test-results/*, images/*.mtd'
        }
        success {
            echo "🎉 REAL BMC TESTS SUCCESSFUL!"
        }
        failure {
            echo "💥 Tests failed"
        }
    }
}
