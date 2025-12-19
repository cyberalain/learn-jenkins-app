pipeline {
    agent any

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    ls -la
                    node --version
                    npm --version
                    npm ci
                    npm run build
                    ls -la
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Test'
                sh 'test -f "build/index.html" && echo "build/index.html exists" || echo "build/index.html does not exist"'
                sh 'ls -la build/index.html 2>/dev/null || echo "Cannot list build/index.html"'
            }
        }
    }
}