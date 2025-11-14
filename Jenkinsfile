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
                    echo "Current directory:"
                    pwd
                    echo "Files in repository:"
                    ls -la
                    echo "Java version:"
                    java -version
                '''
            }
        }
        
        stage('Simple Test') {
            steps {
                echo "3. Running simple test..."
                sh 'echo "This is OpenBMC CI/CD test"'
            }
        }
    }
    
    post {
        always {
            echo "4. Pipeline finished with status: ${currentBuild.result}"
        }
    }
}
