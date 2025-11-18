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
        stage('Install Build Dependencies') {
            steps {
                echo "Installing build dependencies..."
                sh '''
                    apt-get update
                    apt-get install -y \
                        git build-essential libssl-dev libncurses5-dev \
                        python3 python3-pip python3-requests sshpass curl \
                        gcc g++ make file wget cpio unzip rsync bc
                '''
            }
        }
        
        stage('Clone OpenBMC Source') {
            steps {
                echo "Cloning OpenBMC source code..."
                sh '''
                    git clone --depth 1 https://github.com/openbmc/openbmc.git
                    cd openbmc
                    ls -la
                '''
            }
        }
        
        stage('Setup Build Environment') {
            steps {
                echo "Setting up build environment..."
                sh '''
                    cd openbmc
                    # Setup for romulus machine
                    . setup meta-ibm
                    echo "Build environment configured for romulus"
                '''
            }
        }
        
        stage('Build OpenBMC Image') {
            steps {
                echo "Building OpenBMC image for romulus..."
                sh '''
                    cd openbmc
                    echo "Starting build process... (this may take 30-60 minutes)"
                    timeout 7200 bitbake obmc-phosphor-image
                    
                    echo "Build completed, checking for images..."
                    find tmp/deploy/images -name "*.mtd" -o -name "*.qcow2" | head -10
                '''
            }
        }
        
        stage('Prepare BMC Image') {
            steps {
                echo "Preparing BMC image for testing..."
                sh '''
                    mkdir -p images test-results
                    cd openbmc
                    
                    # Find the built image
                    BMC_IMAGE=$(find tmp/deploy/images -name "*.static.mtd" | head -1)
                    if [ -n "$BMC_IMAGE" ]; then
                        echo "Found BMC image: $BMC_IMAGE"
                        cp "$BMC_IMAGE" ../images/openbmc.mtd
                        ls -lh ../images/openbmc.mtd
                    else
                        echo "No BMC image found, using fallback method"
                        # Fallback - download pre-built image
                        wget -O ../images/openbmc.mtd "https://example.com/fallback-image.mtd" || \
                        echo "Fallback failed - cannot continue"
                        exit 1
                    fi
                '''
            }
        }
        
        stage('Start QEMU with Built BMC') {
            steps {
                echo "Starting QEMU with built BMC..."
                sh '''
                    cd images
                    
                    qemu-system-arm -m 256 -M romulus-bmc -nographic \
                        -drive file=openbmc.mtd,format=raw,if=mtd \
                        -net nic \
                        -net user,hostfwd=:0.0.0.0:2222-:22,hostfwd=:0.0.0.0:2443-:443,hostfwd=udp:0.0.0.0:2623-:623,hostname=qemu &
                    
                    echo $! > ../qemu.pid
                    echo "QEMU started with PID: $(cat ../qemu.pid)"
                    
                    echo "Waiting for BMC to boot (120 seconds)..."
                    sleep 120
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
        
        stage('Quick BMC Tests') {
            steps {
                echo "Running quick BMC tests..."
                sh '''
                    mkdir -p test-results
                    
                    # Basic connectivity test
                    sshpass -p '0penBmc' ssh -o StrictHostKeyChecking=no -p 2222 root@localhost '
                        echo "=== Basic System Info ==="
                        cat /etc/os-release 2>/dev/null || echo "No os-release"
                        echo "Uptime: $(uptime)"
                        echo "Hostname: $(hostname)"
                    ' > test-results/basic-test.log
                    
                    echo "Quick tests completed"
                '''
            }
        }
        
        stage('Cleanup') {
            steps {
                echo "Cleaning up..."
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
            echo "Build and Test Pipeline completed"
            archiveArtifacts 'test-results/*, images/*.mtd, qemu.pid'
        }
        success {
            echo "OpenBMC build and test completed successfully!"
        }
        failure {
            echo "Build or test failed"
        }
    }
}
