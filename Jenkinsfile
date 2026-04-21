pipeline {
  agent any

  options {
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '7'))
  }

  parameters {
    booleanParam(name: 'RUN_TERRAFORM_PLAN', defaultValue: false, description: 'Run terraform plan (optional)')
    choice(name: 'ENV', choices: ['nonprod'], description: 'Deployment environment')
  }
