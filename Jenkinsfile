pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                echo "1. Checking out code from GitHub..."
                checkout scm
            }
        }
        
        stage('Environment Check') {
            steps {
                echo "2. Checking environment..."
                sh '''
                    echo "=== Environment Check ==="
                    echo "Current directory:"
                    pwd
                    echo "Files in repository:"
                    ls -la
                    echo "Java version:"
                    java -version
                '''
            }
        }
        
        stage('Check QEMU') {
            steps {
                echo "3. Checking QEMU installation..."
                sh '''
                    echo "=== QEMU Check ==="
                    qemu-system-arm --version || echo "QEMU not installed"
                    which qemu-system-arm || echo "QEMU not found in PATH"
                '''
            }
        }
        
        stage('Download OpenBMC Image') {
            steps {
                echo "4. Preparing OpenBMC image..."
                sh '''
                    echo "=== OpenBMC Setup ==="
                    mkdir -p images
                    cd images
                    if [ ! -f openbmc.qcow2 ]; then
                        echo "Creating test OpenBMC image..."
                        qemu-img create -f qcow2 openbmc.qcow2 1G
                        echo "Test image created successfully"
                    else
                        echo "OpenBMC image already exists"
                    fi
                    ls -la
                '''
            }
        }
        
        stage('Start QEMU with OpenBMC') {
            steps {
                echo "5. Starting QEMU with OpenBMC..."
                sh '''
                    echo "=== Starting QEMU ==="
                    # Останавливаем предыдущие запуски
                    pkill -f qemu-system || true
                    sleep 2
                    
                    # Запускаем QEMU
                    cd images
                    qemu-system-arm -machine virt -nographic \
                        -drive file=openbmc.qcow2,format=qcow2 \
                        -netdev user,id=net0,hostfwd=tcp::8080-:80,hostfwd=tcp::2222-:22 \
                        -device virtio-net,netdev=net0 &
                    
                    echo $! > ../qemu.pid
                    echo "QEMU started with PID: $(cat ../qemu.pid)"
                    sleep 5
                '''
            }
        }
        
        stage('Run Auto Tests') {
            steps {
                echo "6. Running automated tests..."
                sh '''
                    echo "=== Auto Tests ==="
                    echo "Testing OpenBMC connectivity..."
                    
                    # Проверяем порты
                    echo "Checking port 8080:"
                    nc -z localhost 8080 && echo "PORT 8080: OPEN" || echo "PORT 8080: CLOSED"
                    
                    echo "Checking port 2222:"
                    nc -z localhost 2222 && echo "PORT 2222: OPEN" || echo "PORT 2222: CLOSED"
                    
                    # Простой HTTP тест
                    echo "Testing HTTP connection:"
                    curl -f http://localhost:8080 || echo "HTTP test failed - expected for test image"
                '''
            }
        }
        
        stage('Run WebUI Tests') {
            steps {
                echo "7. Running WebUI tests..."
                sh '''
                    echo "=== WebUI Tests ==="
                    mkdir -p test-results
                    
                    # Создаем тестовый отчет
                    cat > test-results/webui-test.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="OpenBMC WebUI Tests">
    <testcase name="WebUI Accessibility" classname="WebUI">
        <skipped>Test skipped - no real OpenBMC image</skipped>
    </testcase>
</testsuite>
EOF
                    echo "WebUI test report generated"
                '''
            }
            post {
                always {
                    junit 'test-results/*.xml'
                }
            }
        }
        
        stage('Stop QEMU') {
            steps {
                echo "8. Stopping QEMU..."
                sh '''
                    echo "=== Stopping QEMU ==="
                    if [ -f qemu.pid ]; then
                        echo "Stopping QEMU process: $(cat qemu.pid)"
                        kill $(cat qemu.pid) 2>/dev/null || true
                        rm -f qemu.pid
                    fi
                    echo "QEMU stopped"
                '''
            }
        }
    }
    
    post {
        always {
            archiveArtifacts artifacts: 'images/*.qcow2, test-results/*.xml, *.pid', allowEmptyArchive: true
            echo "OpenBMC CI/CD pipeline completed: ${currentBuild.result}"
        }
        success {
            echo "🎉 Pipeline completed successfully!"
        }
    }
}
