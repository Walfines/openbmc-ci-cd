pipeline {
    agent any
    
    stages {
        stage('Test File Access') {
            steps {
                echo "🔍 Testing file access from host..."
                sh '''
                    echo "=== Checking host files ==="
                    ls -la /home/ubuntu/Desktop/romulus/ 2>/dev/null || echo "Cannot access host files directly"
                    
                    echo "=== Trying to copy ==="
                    mkdir -p test-images
                    cp /home/ubuntu/Desktop/romulus/obmc-phosphor-image-romulus-20250902012112.static.mtd test-images/ 2>/dev/null && \
                    echo "✅ Successfully copied BMC image!" || \
                    echo "❌ Cannot copy - need Docker volume"
                    
                    ls -la test-images/ 2>/dev/null || echo "No images copied"
                '''
            }
        }
    }
}
