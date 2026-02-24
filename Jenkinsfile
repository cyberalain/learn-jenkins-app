pipeline {
  agent any

  environment {
    NETLIFY_SITE_ID     = '3bf421b8-2d38-42b5-b9e8-d197ab62d91c'
    NETLIFY_AUTH_TOKEN  = credentials('netlify-token')
    REACT_APP_VERSION   = "1.0.${BUILD_ID}"
  }

  stages {

    stage('AWS') {
      agent {
        docker {
          image 'amazon/aws-cli'
          args "--entrypoint=''"
          reuseNode true
        }
      }
      steps {
        sh '''
          aws --version
        '''
      }
    }

    stage('Build') {
      agent {
        docker {
          image 'node:18-alpine'
          reuseNode true
        }
      }
      steps {
        sh '''
          set -e
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
              image 'node:18-alpine'
              reuseNode true
            }
          }
          steps {
            sh '''
              set -e
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
              image 'my-playwright'
              reuseNode true
            }
          }
          steps {
            sh '''
              set -e
              serve -s build &
              sleep 10
              npx playwright test --reporter=html
            '''
          }
          post {
            always {
              publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: false,
                reportDir: 'playwright-report',
                reportFiles: 'index.html',
                reportName: 'Local E2E',
                reportTitles: '',
                useWrapperFileDirectly: true
              ])
            }
          }
        }

      }
    }

    stage('Deploy staging') {
      agent {
        docker {
          image 'my-playwright'
          reuseNode true
        }
      }

      steps {
        sh '''
          set -e
          netlify --version
          echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
          netlify status

          # Deploy and capture deploy_url WITHOUT jq (using Node to parse JSON)
          CI_ENVIRONMENT_URL=$(netlify deploy --dir=build --json | node -p "JSON.parse(require('fs').readFileSync(0,'utf8')).deploy_url")
          echo "Staging deploy URL: $CI_ENVIRONMENT_URL"

          # Run E2E against staging (do not fail pipeline if tests fail)
          npx playwright test --reporter=html || true
        '''
      }

      post {
        always {
          publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: false,
            keepAll: false,
            reportDir: 'playwright-report',
            reportFiles: 'index.html',
            reportName: 'Staging E2E',
            reportTitles: '',
            useWrapperFileDirectly: true
          ])
        }
      }
    }

    stage('Deploy prod') {
      agent {
        docker {
          image 'my-playwright'
          reuseNode true
        }
      }

      steps {
        sh '''
          set -e
          node --version
          netlify --version
          echo "Deploying to production. Site ID: $NETLIFY_SITE_ID"
          netlify status

          netlify deploy --dir=build --prod

          npx playwright test --reporter=html || true
        '''
      }

      post {
        always {
          publishHTML([
            allowMissing: false,
            alwaysLinkToLastBuild: false,
            keepAll: false,
            reportDir: 'playwright-report',
            reportFiles: 'index.html',
            reportName: 'Prod E2E',
            reportTitles: '',
            useWrapperFileDirectly: true
          ])
        }
      }
    }

  }
}
