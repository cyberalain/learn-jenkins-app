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
                    echo "Build stage running in Playwright container"
                    node --version
                    npm --version

                    npm install
                    npm run build

                    ls -la
                '''
            }
        }

        stage('Test') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo "Test stage running in Node container"

                    node --version
                    npm --version

                    test -f build/index.html
                    npm test
                '''
            }
        }
    }

    post {
        always {
            junit 'test-results/**/*.xml'
        }
    }
}

