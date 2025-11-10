pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '222892837737'
        REGION = 'ap-south-1'
        ECR_REPO_NAME = 'flask-app-repo'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-id', passwordVariable: 'AWS_SECRET', usernameVariable: 'AWS_KEY')]) {
                    sh '''
                      aws configure set aws_access_key_id $AWS_KEY
                      aws configure set aws_secret_access_key $AWS_SECRET
                      aws configure set default.region ${REGION}
                      aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                  aws ecr create-repository --repository-name ${ECR_REPO_NAME} --region ${REGION} || true
                  docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                  docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy (SSH)') {
            steps {
                // assumes you added an SSH credential with id 'ec2-ssh' and remote user ubuntu
                sshagent(['ec2-ssh']) {
                    sh '''
                      ssh -o StrictHostKeyChecking=no ubuntu@<EC2_PUBLIC_IP> "
                        docker pull ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG} &&
                        docker stop flask-container || true &&
                        docker rm flask-container || true &&
                        docker run -d -p 5000:5000 --name flask-container ${AWS_ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}
                      "
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "Build and deploy succeeded: ${env.BUILD_NUMBER}"
        }
        failure {
            echo "Build failed."
        }
    }
}
