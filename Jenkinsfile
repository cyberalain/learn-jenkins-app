pipeline {
    agent any

    stages {

        stage('Build') {
            steps {
                sh '''
                    echo "Build stage"
                    node --version
                    npm --version

                    npm ci
                    npm run build
                '''
            }
        }

        stage('Test') {
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

        stage('E2E') {
            steps {
                sh '''
                    echo "E2E stage"
                    npx playwright test
                '''
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: 'test-results/**/*.xml'
        }
    }