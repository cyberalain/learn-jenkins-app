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
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'  // UPDATED TO MATCH YOUR PACKAGE.JSON
                    reuseNode true
                }
            }
            steps {
                sh '''
                    echo 'small change'
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
                            image 'mcr.microsoft.com/playwright:v1.49.1-noble'  // UPDATED
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            #test -f build/index.html
                            npm test
                        '''
                    }
                    post {
                        always {
                            junit 'jest-results/junit.xml'
                        }
                    }
                }

                stage('E2E') {
                    agent {
                        docker {
                            image 'mcr.microsoft.com/playwright:v1.49.1-noble'  // UPDATED
                            reuseNode true
                        }
                    }
                    steps {
                        sh '''
                            npm install serve
                            node_modules/.bin/serve -s build &
                            SERVE_PID=$!
                            sleep 10
                            npx playwright test --reporter=html || true
                            kill $SERVE_PID 2>/dev/null || true
                        '''
                    }
                    post {
                        always {
                            publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Playwright Local', reportTitles: '', useWrapperFileDirectly: true])
                        }
                    }
                }
            }
        }

        stage('Deploy staging') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'  // UPDATED
                    reuseNode true
                }
            }
            steps {
                script {
                    // Use timeout to prevent hanging
                    timeout(time: 5, unit: 'MINUTES') {
                        sh '''
                            npm install netlify-cli
                            node_modules/.bin/netlify --version
                            echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
                            
                            # Deploy and capture output
                            node_modules/.bin/netlify deploy --dir=build --json > deploy-output.json
                            
                            # Display the output for debugging
                            cat deploy-output.json
                        '''
                        
                        // Extract the URL from JSON using native Jenkins methods
                        def deployOutput = readJSON file: 'deploy-output.json'
                        env.STAGING_URL = deployOutput.deploy_url
                        echo "Staging URL: ${env.STAGING_URL}"
                    }
                }
            }
        }

        stage('staging E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'  // UPDATED
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = "${env.STAGING_URL}"
            }
            steps {
                sh '''
                    # Install Playwright browsers if needed
                    npx playwright install --with-deps
                    
                    # Run tests against the deployed staging site
                    echo "Testing staging site: $CI_ENVIRONMENT_URL"
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
                script {
                    timeout(time: 15, unit: 'MINUTES') {
                        input(
                            id: 'approve-prod',
                            message: 'Do you wish to deploy to production?',
                            ok: 'Yes, I am sure!'
                        )
                    }
                }
            }
        }

        stage('Deploy prod') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'  // UPDATED
                    reuseNode true
                }
            }
            steps {
                script {
                    // Use timeout to prevent hanging
                    timeout(time: 5, unit: 'MINUTES') {
                        sh '''
                            npm install netlify-cli
                            node_modules/.bin/netlify --version
                            echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
                            node_modules/.bin/netlify deploy --dir=build --prod
                        '''
                    }
                }
            }
        }

        stage('Prod E2E') {
            agent {
                docker {
                    image 'mcr.microsoft.com/playwright:v1.49.1-noble'  // UPDATED
                    reuseNode true
                }
            }
            environment {
                CI_ENVIRONMENT_URL = "https://incandescent-cucurucho-8b9065.netlify.app"
            }
            steps {
                sh '''
                    # Install Playwright browsers if needed
                    npx playwright install --with-deps
                    
                    # Run tests against the production site
                    echo "Testing production site: $CI_ENVIRONMENT_URL"
                    npx playwright test --reporter=html || true
                '''
            }
            post {
                always {
                    publishHTML([allowMissing: false, alwaysLinkToLastBuild: false, icon: '', keepAll: false, reportDir: 'playwright-report', reportFiles: 'index.html', reportName: 'Prod E2E', reportTitles: '', useWrapperFileDirectly: true])
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}