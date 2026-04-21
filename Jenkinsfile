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

    // ✅ Recommended: store in Jenkins Credentials (Secret Text) with ID: snyk-token
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
        script {
          def scannerHome = tool 'Sonar'   // ✅ MUST match Jenkins Tools name (your screenshot shows Name=Sonar)
          withSonarQubeEnv("${SONARQUBE_SERVER}") {
            withEnv(["PATH+SONAR=${scannerHome}/bin"]) {
              sh """
                sonar-scanner \
                  -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
