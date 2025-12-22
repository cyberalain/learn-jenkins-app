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
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # Check if build/index.html exists
                    if [ -f "build/index.html" ]; then
                        echo "✓ build/index.html exists"
                    else
                        echo "✗ build/index.html does not exist"
                        exit 1  # Fail the stage if file doesn't exist
                    fi
                    
                    # Run tests
                    npm test
                '''
            }
        }
    }

    post {
        always {
            junit 'test-results/junint.xml'
        }
    }
}