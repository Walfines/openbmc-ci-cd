pipeline {
    agent any
    stages {
        stage('Hello') {
            steps {
                echo 'Hello OpenBMC!'
                sh 'qemu-system-arm --version || echo "QEMU not installed"'
            }
        }
    }
}
