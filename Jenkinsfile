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
    AWS_REGION      = "eu-west-1"
    AWS_ACCOUNT_ID  = "102461617910"

    REPO_NAME       = "week2-sample-app"
    ECR_REPO        = "102461617910.dkr.ecr.eu-west-1.amazonaws.com/ecs-rds-test-sample-app"

    ECS_CLUSTER     = "ecs-rds-test-cluster"
    ECS_SERVICE     = "ecs-rds-test-service"

    IMAGE_TAG       = "${BUILD_NUMBER}"
    IMAGE_URI       = "${ECR_REPO}:${IMAGE_TAG}"

    // Jenkins "System" -> SonarQube server name
    SONARQUBE_SERVER   = "sonar"
    SONAR_PROJECT_KEY  = "week2-sample-app"

    // SonarQube reachable from Jenkins EC2
    SONAR_URL       = "http://10.8.54.92:9000"

    // Jenkins Credentials IDs
    SONAR_TOKEN_CRED_ID = "squ_3833ea7189fe32152909d806de1880d49ac571f4"
    SNYK_TOKEN_CRED_ID  = "a0d93a0b-9393-471c-8e8a-9280d565abf5"
  }

  stages {

    stage('Checkout') {
      steps { checkout scm }
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

    // ✅ Quality Gate using Polling (No webhook dependency)
    stage('Quality Gate (Poll)') {
      steps {
        withCredentials([string(credentialsId: "${SONAR_TOKEN_CRED_ID}", variable: 'SONAR_TOKEN')]) {
          timeout(time: 20, unit: 'MINUTES') {
            sh """
              set -e
              while true; do
                RESP=\$(curl -s -u "\${SONAR_TOKEN}:" \
                  "${SONAR_URL}/api/qualitygates/project_status?projectKey=${SONAR_PROJECT_KEY}")

                STATUS=\$(echo "\$RESP" | python3 -c 'import sys, json; print(json.load(sys.stdin)["projectStatus"]["status"])')

                if [ "\$STATUS" = "OK" ]; then
                  echo "Quality Gate PASSED"
                  break
                elif [ "\$STATUS" = "ERROR" ]; then
                  echo "Quality Gate FAILED"
                  echo "\$RESP"
                  exit 1
                fi

                echo "Quality Gate still \$STATUS, waiting..."
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
          docker version
          docker build -t ${IMAGE_URI} .
        '''
      }
    }

    stage('Install Trivy') {
      steps {
        sh '''
          set -e
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
    success {
      echo "Pipeline SUCCESS - Image deployed to ECS"
    }
    failure {
      echo "Pipeline FAILED - Check Sonar/Snyk/Trivy results"
    }
    always {
      // Ensure workspace exists to avoid: hudson.FilePath is missing
      node {
        cleanWs(notFailBuild: true)
      }
    }
  }
}
