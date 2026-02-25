pipeline {
  agent any

  environment {
    NETLIFY_SITE_ID    = '3bf421b8-2d38-42b5-b9e8-d197ab62d91c'
    NETLIFY_AUTH_TOKEN = credentials('netlify-token')
    REACT_APP_VERSION  = "1.0.${BUILD_ID}"
    AWS_S3_BUCKET      = 'learn-jenkins-20260224'
    AWS_DEFAULT_REGION = 'us-east-1'
  }

  options { timestamps() }

  stages {

    stage('Build') {
      agent {
        docker {
          image 'my-playwright'
          reuseNode true
        }
      }
      steps {
        sh '''
          set -eu
          node --version
          npm --version
          npm ci
          npm run build
          ls -la build
        '''
      }
    }

    stage('AWS Upload to S3') {
      agent {
        docker {
          image 'amazon/aws-cli'
          args "--entrypoint=''"
          reuseNode true
        }
      }
      steps {
        withCredentials([
          usernamePassword(
            credentialsId: 'my-aws',
            usernameVariable: 'AWS_ACCESS_KEY_ID',
            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
          )
        ]) {
          sh '''
            set -eu
            aws --version

            echo "Checking AWS identity..."
            aws sts get-caller-identity

            echo "Checking bucket exists: $AWS_S3_BUCKET"
            aws s3 ls "s3://$AWS_S3_BUCKET" >/dev/null

            echo "Uploading build/ to S3..."
            aws s3 sync build "s3://$AWS_S3_BUCKET/" --delete

            echo "Done. Listing objects:"
            aws s3 ls "s3://$AWS_S3_BUCKET/"
          '''
        }
      }
    }

    stage('Tests') {
      parallel {

        stage('Unit tests') {
          agent {
            docker {
              image 'my-playwright'
              reuseNode true
            }
          }
          steps {
            sh '''
              set -eu
              CI=true npm test
            '''
          }
          post {
            always { junit 'jest-results/junit.xml' }
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
              set -eu

              npx --yes serve -s build -l 3000 >/tmp/serve.log 2>&1 &
              SERVER_PID=$!

              node -e "
                const http=require('http');
                const start=Date.now();
                (function ping(){
                  http.get('http://127.0.0.1:3000', r => process.exit(r.statusCode===200?0:1))
                    .on('error', () => {
                      if (Date.now()-start > 60000) process.exit(1);
                      setTimeout(ping, 1000);
                    });
                })();
              " || (echo 'Server did not start' && cat /tmp/serve.log && kill $SERVER_PID || true && exit 1)

              npx playwright test --reporter=html
              kill $SERVER_PID || true
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
          set -eu
          netlify --version
          netlify status

          BASE_URL=$(netlify deploy --dir=build --json | node -p "JSON.parse(require('fs').readFileSync(0,'utf8')).deploy_url")
          echo "Staging URL: $BASE_URL"

          npx playwright test --reporter=html || true
        '''
      }
      post {
        always {
          publishHTML([
            allowMissing: true,
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
          set -eu
          netlify --version
          netlify status

          netlify deploy --dir=build --prod

          export BASE_URL="https://incandescent-cucurucho-8b9065.netlify.app"
          echo "Prod URL: $BASE_URL"

          npx playwright test --reporter=html || true
        '''
      }
      post {
        always {
          publishHTML([
            allowMissing: true,
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