pipeline {
    agent any
    tools {
        maven 'mymaven'
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'mysonar'
        AWS_REGION = 'us-east-1' // Change to your AWS region
        ECR_REPO_APP = '585768179486.dkr.ecr.us-east-1.amazonaws.com/ourproject/app'
        ECR_REPO_DB  = '585768179486.dkr.ecr.us-east-1.amazonaws.com/ourproject/db'
    }
    stages {
        stage("Clean") {
            steps {
                cleanWs()
            }
        }
        stage("Code") {
            steps {
                git "https://github.com/RaviVarma06/dockerwebapp.git"
            }
        }
        stage("CQA") {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh '''mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=docker-webapp \
                        -Dsonar.projectName='docker-webapp' \
                        -Dsonar.login=sqa_09f5f9a339c85ad4cd189f8c162e3abc9f95c851
                    '''
                }
            }
        }
        stage("Build") {
            steps {
                sh 'mvn clean package'
                sh 'cp -r target Docker-app'
            }
        }
        stage("Docker Build") {
            steps {
                script {
                    sh "docker build -t ourproject/app Docker-app"
                    sh "docker build -t ourproject/db Docker-db"
                    sh "docker tag ourproject/app $ECR_REPO_APP"
                    sh "docker tag ourproject/db $ECR_REPO_DB" 
                }
            }
        }
        stage("Push to ECR") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'AWS-ECR-CREDS', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID
                        aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY
                        aws configure set default.region $AWS_REGION
                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO_APP
                        docker push $ECR_REPO_APP
                        docker push $ECR_REPO_DB
                    '''
                }
            }
        }
        stage("TrivyScan") {
            steps {
                sh 'trivy fs . > trivyfs.txt'
                sh "trivy image $ECR_REPO_APP"
                sh "trivy image $ECR_REPO_DB"
            }
        }
        stage("Deploy to container") {
            steps {
                sh 'docker-compose up -d'
            }
        }
    }
}
