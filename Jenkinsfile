
pipeline {
  agent any

  tools {
    maven 'mymaven'
    jdk 'jdk17'
    nodejs 'node16'
  }

  environment {
    // Optional when using withSonarQubeEnv, but kept for reference
    SCANNER_HOME = tool 'mysonar'

    // Cache Maven deps to speed up builds
    MAVEN_OPTS   = "-Dmaven.repo.local=/var/lib/jenkins/.m2/repository"

    AWS_REGION   = 'us-east-1'
    ECR_REGISTRY = '503015902555.dkr.ecr.us-east-1.amazonaws.com'
    ECR_REPO_APP = '503015902555.dkr.ecr.us-east-1.amazonaws.com/ourproject/app'
    ECR_REPO_DB  = '503015902555.dkr.ecr.us-east-1.amazonaws.com/ourproject/db'
  }

  stages {

    stage('Clean') {
      steps {
        cleanWs()
      }
    }

    stage('Code') {
      steps {
        // If the job is "Pipeline from SCM", you can remove this and rely on Declarative: Checkout SCM
        git url: 'https://github.com/Vaasanth3/docker-webapp.git', branch: 'master'
        // If default branch is 'main', change branch accordingly.
      }
    }

    stage('CQA') {
      steps {
        dir(env.WORKSPACE) {
          withSonarQubeEnv('mysonar') {
            sh '''
              mvn clean verify sonar:sonar \
                -Dsonar.projectKey=docker-webapp \
                -Dsonar.projectName="docker-webapp"
            '''
          }
        }
      }
    }

    stage('Build') {
      steps {
        dir(env.WORKSPACE) {
          sh 'echo "Working directory:" && pwd'
          sh 'ls -la'
          sh 'mvn clean package'
          // Copy artifacts to Docker-app context if your Dockerfile expects them there
          sh 'cp -r target Docker-app'
        }
      }
    }

    stage('Docker Build') {
      steps {
        dir(env.WORKSPACE) {
          // Optional pre-checks to avoid silent failures
          sh '''
            test -d Docker-app || (echo "Docker-app directory not found" && exit 1)
            test -d Docker-db  || (echo "Docker-db directory not found" && exit 1)
          '''
          sh "docker build -t ourproject/app Docker-app"
          sh "docker build -t ourproject/db Docker-db"
          sh "docker tag ourproject/app $ECR_REPO_APP"
          sh "docker tag ourproject/db $ECR_REPO_DB"
        }
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'AWS-ECR-CREDS', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''
            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
            aws configure set default.region $AWS_REGION

            # Login to the ECR REGISTRY (not the repo URL)
            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

            # Push tagged images
            docker push $ECR_REPO_APP
            docker push $ECR_REPO_DB
          '''
        }
      }
    }

    stage('TrivyScan') {
      steps {
        dir(env.WORKSPACE) {
