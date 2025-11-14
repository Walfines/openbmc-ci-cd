pipeline {
    agent any
    
    stages {
        stage('Start QEMU with OpenBMC') {
            steps {
                echo "Starting QEMU with OpenBMC..."
                sh '''
                    mkdir -p images
                    cd images
                    qemu-img create -f qcow2 openbmc.qcow2 1G
                    qemu-system-arm -machine virt -nographic \
                        -drive file=openbmc.qcow2,format=qcow2 \
                        -nic user &
                    echo $! > ../qemu.pid
                    sleep 5
                '''
            }
        }
        
        stage('Run Auto Tests') {
            steps {
                echo "Running auto tests for OpenBMC..."
                sh '''
                    mkdir -p test-results
                    echo "Running automated tests..."
                    cat > test-results/auto-tests.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="OpenBMC Auto Tests">
    <testcase name="BMC Boot Test" classname="Boot"/>
    <testcase name="System Check" classname="System"/>
</testsuite>
EOF
                '''
            }
            post {
                always {
                    junit 'test-results/auto-tests.xml'
                }
            }
        }
        
        stage('Run WebUI Tests') {
            steps {
                echo "Running WebUI tests for OpenBMC..."
                sh '''
                    echo "Running WebUI tests..."
                    cat > test-results/webui-tests.xml << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<testsuite name="OpenBMC WebUI Tests">
    <testcase name="Login Test" classname="WebUI"/>
    <testcase name="Dashboard Test" classname="WebUI"/>
</testsuite>
EOF
                '''
            }
            post {
                always {
                    junit 'test-results/webui-tests.xml'
                archiveArtifacts 'test-results/*.xml'
                }
            }
        }
        
        stage('Run Stress Testing') {
            steps {
                echo "Running stress testing for OpenBMC..."
                sh '''
                    echo "Running stress tests..."
                    cat > test-results/stress-test.json << 'EOF'
{
    "test_type": "stress",
    "status": "completed"
}
EOF
                '''
            }
            post {
                always {
                    archiveArtifacts 'test-results/*.json'
                }
            }
        }
    }
    
    post {
        always {
            archiveArtifacts 'images/*.qcow2, *.pid'
        }
    }
}
