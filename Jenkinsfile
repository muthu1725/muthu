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
    AWS_ACCOUNT_ID= "102461617910"

    REPO_NAME     = "week2-sample-app"
    ECR_REPO      = "102461617910.dkr.ecr.eu-west-1.amazonaws.com/ecs-rds-test-sample-app"

    ECS_CLUSTER   = "ecs-rds-test-cluster"
    ECS_SERVICE   = "ecs-rds-test-service"

    IMAGE_TAG     = "${BUILD_NUMBER}"
    IMAGE_URI     = "${ECR_REPO}:${IMAGE_TAG}"

    SONARQUBE_SERVER  = "sonarqube"
    SONAR_PROJECT_KEY = "week2-sample-app"

    SNYK_TOKEN = "a0d93a0b-9393-471c-8e8a-9280d565abf5"
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
          python3 --version
          pip3 install -r requirements.txt
        '''
      }
    }

    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          sh """
            sonar-scanner \
              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
              -Dsonar.sources=. \
              -Dsonar.host.url=$SONAR_HOST_URL \
              -Dsonar.login=$SONAR_AUTH_TOKEN
          """
        }
      }
    }

    stage('Quality Gate') {
      steps {
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
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

    /* ✅✅ NEW STAGES FOR TRIVY ✅✅ */

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
    success {
      echo "Pipeline SUCCESS – Image deployed to ECS"
    }
    failure {
      echo "Pipeline FAILED – Check Sonar/Snyk/Trivy results"
    }
    always {
      cleanWs()
    }
  }
}

