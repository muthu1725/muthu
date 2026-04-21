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

    REPO_NAME     = "week2-sample-app"
    ECR_REPO      = "102461617910.dkr.ecr.eu-west-1.amazonaws.com/ecs-rds-test-sample-app"

    ECS_CLUSTER   = "ecs-rds-test-cluster"
    ECS_SERVICE   = "ecs-rds-test-service"

    IMAGE_TAG     = "${BUILD_NUMBER}"
    IMAGE_URI     = "${ECR_REPO}:${IMAGE_TAG}"

    SONARQUBE_SERVER  = "sonar"
    SONAR_PROJECT_KEY = "week2-sample-app"

    // ✅ SonarQube URL (reachable from Jenkins EC2)
    SONAR_URL = "http://10.8.54.92:9000"

    // ✅ Recommended: store SONAR token in Jenkins Credentials (Secret text) with ID: sonar-token
    SONAR_TOKEN = credentials('sonar-token')

    // ⚠️ Recommended to move to Jenkins Credentials (Secret text) with ID: snyk-token
    SNYK_TOKEN = "a0d93a0b-9393-471c-8e8a-9280d565abf5"
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
    }

    stage('Build (Python)') {
      steps {
        sh '''
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
                sonar-scanner \
                  -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                  -Dsonar.sources=.
              """
            }
          }
        }
      }
    }

    stage('Install jq (if missing)') {
      steps {
        sh '''
          if ! command -v jq >/dev/null 2>&1; then
            echo "jq not found, installing..."
            sudo yum install -y jq || sudo apt-get update && sudo apt-get install -y jq
          else
            echo "jq already installed"
          fi
        '''
      }
    }

    // ✅ ✅ Quality Gate using Polling (No Webhook dependency)
    stage('Quality Gate (Poll)') {
      steps {
        timeout(time: 20, unit: 'MINUTES') {
          sh """
            while true; do
              STATUS=\$(curl -s -u ${SONAR_TOKEN}: \\
                ${SONAR_URL}/api/qualitygates/project_status?projectKey=${SONAR_PROJECT_KEY} \\
                | jq -r '.projectStatus.status')

              if [ "\$STATUS" = "OK" ]; then
                echo "Quality Gate PASSED"
                break
              elif [ "\$STATUS" = "ERROR" ]; then
                echo "Quality Gate FAILED"
                exit 1
              fi

              echo "Quality Gate still \$STATUS, waiting..."
              sleep 15
            done
          """
        }
      }
    }

    stage('Snyk Dependency Scan') {
      steps {
        sh '''
          snyk auth $SNYK_TOKEN
          snyk test --severity-threshold=high
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          docker version
          docker build -t ${IMAGE_URI} .
        '''
      }
    }

    stage('Install Trivy') {
      steps {
        sh '''
          if ! command -v trivy &> /dev/null
          then
            echo "Installing Trivy..."
            sudo rpm -ivh https://github.com/aquasecurity/trivy/releases/download/v0.50.1/trivy_0.50.1_Linux-64bit.rpm
          else
            echo "Trivy already installed"
          fi
        '''
      }
    }

    stage('Trivy Image Scan (Fail if HIGH/CRITICAL)') {
      steps {
        sh '''
          trivy version
          trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_URI}
        '''
      }
    }

    stage('Login to ECR & Push Image') {
      steps {
        sh '''
          aws ecr get-login-password --region ${AWS_REGION} \
            | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

          docker push ${IMAGE_URI}
        '''
      }
    }

    stage('Deploy to ECS (Non-Prod)') {
      steps {
        sh '''
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
    success { echo "Pipeline SUCCESS - Image deployed to ECS" }
    failure { echo "Pipeline FAILED - Check Sonar/Snyk/Trivy results" }
    always  { cleanWs() }
  }
}
