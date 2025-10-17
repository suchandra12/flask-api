pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker-s-creds')
        IMAGE_NAME = "suchandra12/flask-api"
        TASK_DEF_FILE = "task-definition.json"
        CLUSTER_NAME = "flask-ecs-cluster"
        SERVICE_NAME = "flask-ecs-service"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/suchandra12/flask-api.git'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Login to Docker Hub') {
            steps {
                sh """
                echo ${DOCKERHUB_CREDENTIALS_PSW} | \
                docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }

        stage('Register ECS Task Definition') {
            steps {
                withAWS(credentials: 'aws-credentials', region: 'us-east-1') {
                    sh "aws ecs register-task-definition --cli-input-json file://${TASK_DEF_FILE}"
                }
            }
        }

        stage('Deploy to ECS') {
            steps {
                withAWS(credentials: 'aws-credentials', region: 'us-east-1') {
                    sh """
                    aws ecs update-service \
                      --cluster ${CLUSTER_NAME} \
                      --service ${SERVICE_NAME} \
                      --force-new-deployment
                    """
                }
            }
        }
    }

    post {
        always {
            sh "docker logout"
        }
    }
}
