pipeline {
  agent any

  options {
    buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '7'))
    disableConcurrentBuilds()
    timestamps()
  }

  parameters {
    booleanParam(name: 'RUN_TERRAFORM_PLAN', defaultValue: false, description: 'Run terraform plan (optional)')
    choice(name: 'ENV', choices: ['nonprod'], description: 'Deployment environment')
  }

  environment {
    AWS_REGION     = 'eu-west-1'
    AWS_ACCOUNT_ID = '102461617910'

    REPO_NAME   = 'week2-sample-app'
    ECR_REPO    = '102461617910.dkr.ecr.eu-west-1.amazonaws.com/ecs-rds-test-sample-app'

    ECS_CLUSTER = 'ecs-rds-test-cluster'
    ECS_SERVICE = 'ecs-rds-test-service'

    IMAGE_TAG   = "${BUILD_NUMBER}"
    IMAGE_URI   = "${ECR_REPO}:${IMAGE_TAG}"

    SONARQUBE_SERVER  = 'sonar'
    SONAR_PROJECT_KEY = 'week2-sample-app'

    // SonarQube reachable from Jenkins EC2
    SONAR_URL = 'http://10.8.54.92:9000'

    // Jenkins credentials IDs (NOTE: these must be Jenkins Credential IDs)
    SONAR_TOKEN_CRED_ID = "sonarqube-token"
    SNYK_TOKEN_CRED_ID  = "snyk-token"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build (Python)') {
      steps {
        sh '''
          set -e
          python3 --version
          pip3 install -r requirements.txt
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        script {
          def scannerHome = tool 'Sonar'
          withSonarQubeEnv("${SONARQUBE_SERVER}") {
            withEnv(["PATH+SONAR=${scannerHome}/bin"]) {
              sh '''
                set -e
                sonar-scanner \
                  -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                  -Dsonar.sources=.
              '''
            }
          }
        }
      }
    }

    // ✅ Quality Gate PASS/FAIL controlled by Jenkinsfile (Polling)
    // FIX: use triple-single-quotes so Groovy will NOT treat $STATUS / $RESP as Groovy variables
    stage('Quality Gate (Poll)') {
      steps {
        withCredentials([string(credentialsId: "${SONAR_TOKEN_CRED_ID}", variable: 'SONAR_TOKEN')]) {
          timeout(time: 20, unit: 'MINUTES') {
            sh '''
              set -e
              while true; do
                RESP=$(curl -s -u "${SONAR_TOKEN}:" \
                  "${SONAR_URL}/api/qualitygates/project_status?projectKey=${SONAR_PROJECT_KEY}")

                STATUS=$(echo "$RESP" | python3 -c 'import sys,json; print(json.load(sys.stdin)["projectStatus"]["status"])')

                if [ "$STATUS" = "OK" ]; then
                  echo "✅ Quality Gate PASSED"
                  break
                elif [ "$STATUS" = "ERROR" ]; then
                  echo "❌ Quality Gate FAILED"
                  echo "$RESP"
                  exit 1
                fi

                echo "⏳ Quality Gate still $STATUS, waiting..."
                sleep 15
              done
            '''
          }
        }
      }
    }

    stage('Snyk Dependency Scan') {
  steps {
    withCredentials([
      string(credentialsId: "${SNYK_TOKEN_CRED_ID}", variable: 'SNYK_TOKEN')
    ]) {
      sh '''
        set -e

        echo "Checking Python & pip..."
        python3 --version
        pip3 --version

        echo "Installing Python dependencies..."
        pip3 install -r requirements.txt

        echo "Authenticating Snyk..."
        snyk auth $SNYK_TOKEN

        echo "Running Snyk scan (pip project)..."
        snyk test --package-manager=pip --severity-threshold=high
      '''
    }
  }
}
    stage('Docker Build') {
      steps {
        sh '''
          set -e
          docker version
          docker build -t ${IMAGE_URI} .
        '''
      }
    }

    stage('Trivy Image Scan (Fail if HIGH/CRITICAL)') {
      steps {
        sh '''
          set -e
          trivy version
          trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_URI}
        '''
      }
    }

    stage('Login to ECR & Push Image') {
      steps {
        sh '''
          set -e
          aws ecr get-login-password --region ${AWS_REGION} \
            | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
          docker push ${IMAGE_URI}
        '''
      }
    }

    stage('Deploy to ECS (Non-Prod)') {
      steps {
        sh '''
          set -e
          aws ecs update-service \
            --region ${AWS_REGION} \
            --cluster ${ECS_CLUSTER} \
            --service ${ECS_SERVICE} \
            --force-new-deployment
          echo "ECS deployment triggered"
        '''
      }
    }
  }

  post {
    success { echo '✅ Pipeline SUCCESS - Image deployed to ECS' }
    failure { echo '❌ Pipeline FAILED - Check Sonar/Snyk/Trivy results' }
    always  { cleanWs(notFailBuild: true) }
  }
}
