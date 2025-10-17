pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('docker-s-creds')
        AWS_CREDS = credentials('aws-credentials')
        IMAGE_NAME = "suchandra12/flask-api"
        TASK_DEF_FILE = "task-definition.json"
        CLUSTER_NAME = "flask-ecs-cluster"
        SERVICE_NAME = "flask-ecs-service"
        SUBNETS = "subnet-xxxxxxxx,subnet-yyyyyyyy"       // replace with your subnets
        SECURITY_GROUPS = "sg-xxxxxxxx"                  // replace with your security group
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
                sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}:latest"
            }
        }

        stage('Ensure ECS Cluster Exists') {
            steps {
                script {
                    def clusterExists = sh(
                        script: "aws ecs describe-clusters --clusters ${CLUSTER_NAME} --query 'clusters[0].status' --output text",
                        returnStatus: true
                    ) == 0

                    if (!clusterExists) {
                        echo "Cluster not found. Creating ECS cluster ${CLUSTER_NAME}..."
                        sh "aws ecs create-cluster --cluster-name ${CLUSTER_NAME}"
                    } else {
                        echo "Cluster ${CLUSTER_NAME} already exists."
                    }
                }
            }
        }

        stage('Register ECS Task Definition') {
            steps {
                withAWS(credentials: 'aws-credentials', region: 'us-east-1') {
                    sh "aws ecs register-task-definition --cli-input-json file://${TASK_DEF_FILE}"
                }
            }
        }

        stage('Ensure ECS Service Exists') {
            steps {
                script {
                    def serviceExists = sh(
                        script: "aws ecs describe-services --cluster ${CLUSTER_NAME} --services ${SERVICE_NAME} --query 'services[0].status' --output text",
                        returnStatus: true
                    ) == 0

                    if (!serviceExists) {
                        echo "Service not found. Creating ECS service ${SERVICE_NAME}..."
                        sh """
                        aws ecs create-service \
                          --cluster ${CLUSTER_NAME} \
                          --service-name ${SERVICE_NAME} \
                          --task-definition $(jq -r '.family' ${TASK_DEF_FILE}) \
                          --desired-count 1 \
                          --launch-type FARGATE \
                          --network-configuration "awsvpcConfiguration={subnets=[${SUBNETS}],securityGroups=[${SECURITY_GROUPS}],assignPublicIp=ENABLED}"
                        """
                    } else {
                        echo "Service ${SERVICE_NAME} exists. Updating deployment..."
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
    }

    post {
        always {
            sh "docker logout"
        }
    }
}
