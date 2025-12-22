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
                    echo "Build stage"
                    node --version
                    npm --version

                    npm ci
                    npm run build

                    ls -la
                '''
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "Test stage"

                    if [ -f "build/index.html" ]; then
                        echo "✓ build/index.html exists"
                    else
                        echo "✗ build/index.html does not exist"
                        exit 1
                    fi

                    npm test
                '''
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: 'test-results/**/*.xml'
        }
    }
}