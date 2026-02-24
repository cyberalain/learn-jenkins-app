pipeline {
  agent any

  environment {
    NETLIFY_SITE_ID     = '3bf421b8-2d38-42b5-b9e8-d197ab62d91c'
    NETLIFY_AUTH_TOKEN  = credentials('netlify-token')
    REACT_APP_VERSION   = "1.0.${BUILD_ID}"

    // npm stability settings
    NPM_CONFIG_REGISTRY = 'https://registry.npmjs.org/'
    NPM_CONFIG_FETCH_RETRIES = '5'
    NPM_CONFIG_FETCH_RETRY_FACTOR = '2'
    NPM_CONFIG_FETCH_RETRY_MINTIMEOUT = '20000'
    NPM_CONFIG_FETCH_RETRY_MAXTIMEOUT = '120000'
  }

  options {
    timestamps()
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
        sh 'aws --version'
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
          set -euxo pipefail

          node --version
          npm --version

          # quick network sanity checks (won't fail build if blocked, just prints)
          echo "DNS test:"
          cat /etc/resolv.conf || true
          echo "Try reaching npm registry:"
          wget -qSO- https://registry.npmjs.org/ >/dev/null || true

          npm config set registry "$NPM_CONFIG_REGISTRY"

          # Retry npm ci (handles flaky networks)
          i=1
          until [ $i -gt 3 ]; do
            echo "npm ci attempt $i/3..."
            npm ci && break
            i=$((i+1))
            echo "npm ci failed. Waiting before retry..."
            sleep 15
          done

          npm run build
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
              set -euxo pipefail
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
              set -euxo pipefail
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
          set -euxo pipefail
          netlify --version
          echo "Deploying to staging. Site ID: $NETLIFY_SITE_ID"
          netlify status

          CI_ENVIRONMENT_URL=$(netlify deploy --dir=build --json | node -p "JSON.parse(require('fs').readFileSync(0,'utf8')).deploy_url")
          echo "Staging deploy URL: $CI_ENVIRONMENT_URL"

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
          set -euxo pipefail
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
