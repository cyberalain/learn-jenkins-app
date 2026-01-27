pipeline {
    agent any

    environment {
        NETLIFY_SITE_ID = '3bf421b8-2d38-42b5-b9e8-d197ab62d91c'
        NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    }

    stages {
        stage('Build') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'
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

        stage('Tests') {
            parallel {
                stage('Unit tests') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.49.1-noble'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E - Local') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.49.1-noble'
                            reuseNode true
                        }
                    }

                    steps {
                        sh '''
                            npm install serve
                            # Start the local server
                            node_modules/.bin/serve -s build -l 3000 &
                            SERVER_PID=\$!
                            sleep 10
                            
                            # Run tests with local URL
                            CI_ENVIRONMENT_URL=http://localhost:3000 npx playwright test --reporter=html
                            
                            # Kill the server after tests
                            kill \$SERVER_PID
                        '''
                    }

                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Local E2E', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to staging. Site ID: \$NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    
                    # Deploy to staging and capture the URL
                    DEPLOY_OUTPUT=\$(node_modules/.bin/netlify deploy --dir=build --json)
                    echo "\$DEPLOY_OUTPUT" > deploy-output.json
                    
                    # Parse the JSON to get the deploy URL
                    DEPLOY_URL=\$(echo "\$DEPLOY_OUTPUT" | grep -o '"deploy_url":"[^"]*"' | cut -d'"' -f4)
                    
                    if [ -z "\$DEPLOY_URL" ]; then
                        echo "ERROR: Failed to get deploy URL from Netlify"
                        cat deploy-output.json
                        exit 1
                    fi
                    
                    echo "Staging URL: \$DEPLOY_URL"
                    
                    # Set CI_ENVIRONMENT_URL for Playwright tests
                    export CI_ENVIRONMENT_URL=\$DEPLOY_URL
                    
                    # Run tests against the staging URL
                    npx playwright test --reporter=html || true
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Staging E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    input message: 'Do you wish to deploy to production?', ok: 'Yes, I am sure!'
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'
                    reuseNode true
                }
            }

            steps {
                sh '''
                    node --version
                    npm install netlify-cli
                    node_modules/.bin/netlify --version
                    echo "Deploying to production. Site ID: \$NETLIFY_SITE_ID"
                    node_modules/.bin/netlify status
                    node_modules/.bin/netlify deploy --dir=build --prod
                    
                    # Get the production URL (Netlify uses the site name by default)
                    # Replace with your actual production URL
                    PROD_URL="https://incandescent-cucurucho-8b9065.netlify.app"
                    
                    echo "Production URL: \$PROD_URL"
                    
                    # Set CI_ENVIRONMENT_URL for Playwright tests
                    export CI_ENVIRONMENT_URL=\$PROD_URL
                    
                    # Run tests against production
                    npx playwright test --reporter=html
                '''
            }

            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
}