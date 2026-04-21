pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '7'))
  }

  parameters {
    booleanParam(name: 'RUN_TERRAFORM_PLAN', defaultValue: false, description: 'Run terraform plan (optional)')
    choice(name: 'ENV', choices: ['nonprod'], description: 'Deployment environment')
  }

  environment {
    AWS_REGION     = "eu-west-1"
    AWS_ACCOUNT_ID = "102461617910"

    REPO_NAME   = "week2-sample-app"
    ECR_REPO    = "102461617910.dkr.ecr.eu-west-1.amazonaws.com/ecs-rds-test-sample-app"
    IMAGE_TAG   = "${BUILD_NUMBER}"
    IMAGE_URI   = "${ECR_REPO}:${IMAGE_TAG}"

    ECS_CLUSTER = "ecs-rds-test-cluster"
    ECS_SERVICE = "ecs-rds-test-service"

    SONARQUBE_SERVER  = "sonar"
    SONAR_PROJECT_KEY = "week2-sample-app"
    SONAR_URL         = "http://10.8.54.92:9000"

    SONAR_TOKEN_CRED_ID = "squ_3833ea7189fe32152909d806de1880d49ac571f4"
    SNYK_TOKEN_CRED_ID  = "a0d93a0b-9393-471c-8e8a-9280d565abf5"
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
              sh """
                set -e
                sonar-scanner \
                  -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                  -Dsonar.sources=.
              """
            }
          }
        }
      }
    }

    stage('Quality Gate (Poll)') {
      steps {
        withCredentials([string(credentialsId: "${SONAR_TOKEN_CRED_ID}", variable: 'SONAR_TOKEN')]) {
          timeout(time: 20, unit: 'MINUTES') {
            sh """
              set -e
              while true; do
                RESP=\$(curl -s -u "\${SONAR_TOKEN}:" \
                  "${SONAR_URL}/api/qualitygates/project_status?projectKey=${SONAR_PROJECT_KEY}")

                STATUS=\$(echo "\$RESP" | python3 -c 'import sys,json; print(json.load(sys.stdin)["projectStatus"]["status"])')

                if [ "\$STATUS" = "OK" ]; then
                  echo "✅ Quality Gate PASSED"
                  break
                elif [ "\$STATUS" = "ERROR" ]; then
                  echo "❌ Quality Gate FAILED"
                  echo "\$RESP"
                  exit 1
                fi

                echo "⏳ Quality Gate still \$STATUS, waiting..."
                sleep 15
              done
            """
          }
        }
      }
    }

    stage('Snyk Dependency Scan') {
      steps {
        withCredentials([string(credentialsId: "${SNYK_TOKEN_CRED_ID}", variable: 'SNYK_TOKEN')]) {
          sh '''
            set -e
            snyk auth $SNYK_TOKEN
            snyk test --severity-threshold=high
          '''
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          set -e
          docker build -t ${IMAGE_URI} .
        '''
      }
    }

    stage('Trivy Image Scan') {
      steps {
        sh '''
          set -e
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
        '''
      }
    }
  }

  post {
    success {
      echo "✅ Pipeline SUCCESS - Image deployed to ECS"
    }
    failure {
      echo "❌ Pipeline FAILED - Check Sonar / Snyk / Trivy logs"
    }
    always {
      cleanWs(notFailBuild: true)
    }
  }
}
