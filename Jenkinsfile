pipeline {
    agent any

    stages {
        stage('Build and Test') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.39.0-jammy'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh '''
                    echo "🔧 Build and Test Stage"
                    node --version
                    npm --version

                    npm ci
                    npm run build

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

        stage('E2E Tests') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.57.0-noble'
                    args '-v /var/run/docker.sock:/var/run/docker.sock'
                }
            }
            steps {
                sh '''
                    echo "🚀 E2E Testing Stage"

                    if [ ! -d "build" ]; then
                        echo "✗ build folder missing"
                        exit 1
                    fi

                    npm install -g serve
                    serve -s build -l 3000 &
                    SERVE_PID=$!

                    sleep 5

                    npx playwright test
                    TEST_EXIT_CODE=$?

                    kill $SERVE_PID 2>/dev/null || true
                    exit $TEST_EXIT_CODE
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
