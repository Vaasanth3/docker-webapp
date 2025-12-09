
pipeline {
  agent any

  tools {
    maven 'mymaven'   // Ensure this exists in Global Tool Configuration
    jdk   'jdk17'     // Ensure this exists or remove if system JDK is used
    // nodejs 'node16' // Comment out if you don't use Node in this pipeline
  }

  environment {
    // Optional; not needed when using withSonarQubeEnv, but harmless
    SCANNER_HOME = tool 'mysonar' // Ensure a SonarScanner tool is named 'mysonar' if you keep this

    // Cache Maven deps to speed builds
    MAVEN_OPTS   = "-Dmaven.repo.local=/var/lib/jenkins/.m2/repository"

    AWS_REGION   = 'us-east-1'
    ECR_REGISTRY = '503015902555.dkr.ecr.us-east-1.amazonaws.com'
    ECR_REPO_APP = '503015902555.dkr.ecr.us-east-1.amazonaws.com/ourproject/app'
    ECR_REPO_DB  = '503015902555.dkr.ecr.us-east-1.amazonaws.com/ourproject/db'
  }

  stages {
    stage('Clean') {
      steps { cleanWs() }
    }

    stage('Code') {
      steps {
        // If job is "Pipeline from SCM", you can remove this and rely on Jenkins' automatic checkout
        git url: 'https://github.com/Vaasanth3/docker-webapp.git', branch: 'master'
      }
    }

    stage('CQA') {
      steps {
        dir(env.WORKSPACE) {
          // Ensure a SonarQube server named 'mysonar' exists under Manage Jenkins -> Configure System
          withSonarQubeEnv('mysonar') {
            sh '''
              set -euxo pipefail
              mvn -B -ntp clean verify sonar:sonar \
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
          sh '''
            set -euxo pipefail
            mvn -B -ntp clean package
            # Copy build artifacts to Docker-app context if required by your Dockerfile
            cp -r target Docker-app || { echo "Docker-app not found or copy failed"; exit 1; }
          '''
        }
      }
    }

    stage('Docker Build') {
      steps {
        dir(env.WORKSPACE) {
          sh '''
            set -euxo pipefail
            test -d Docker-app || { echo "Docker-app directory not found"; exit 1; }
            test -d Docker-db  || { echo "Docker-db directory not found"; exit 1; }

            docker build -t ourproject/app Docker-app
            docker build -t ourproject/db Docker-db
            docker tag ourproject/app $ECR_REPO_APP
            docker tag ourproject/db $ECR_REPO_DB
          '''
        }
      }
    }

    stage('Push to ECR') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'AWS-ECR-CREDS', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''
            set -euxo pipefail
            aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
            aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
            aws configure set default.region $AWS_REGION

            # Login to ECR REGISTRY (not repo path)
            aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY

            docker push $ECR_REPO_APP
            docker push $ECR_REPO_DB
          '''
        }
      }
    }

    stage('TrivyScan') {
      steps {
        dir(env.WORKSPACE) {
          // Skip gracefully if trivy is not installed
          script {
            def trivyInstalled = sh(script: 'command -v trivy >/dev/null 2>&1', returnStatus: true) == 0
            if (trivyInstalled) {
              sh 'trivy fs . > trivyfs.txt'
              sh "trivy image $ECR_REPO_APP || true"
              sh "trivy image $ECR_REPO_DB  || true"
            } else {
              echo 'Trivy not installed on this agent—skipping scan.'
            }
          }
        }
      }
    }

    stage('Deploy to container') {
      steps {
        dir(env.WORKSPACE) {
          sh '''
            set -euxo pipefail
            docker-compose up -d
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'trivyfs.txt', onlyIfSuccessful: false
      cleanWs()
    }
  }
}
