
pipeline {
    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '415081250301'
        ECR_REGISTRY = '415081250301.dkr.ecr.us-east-1.amazonaws.com'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo 'GitHub source code checked out successfully'
            }
        }

        stage('Maven Build and Test') {
            steps {
                sh '''
                    mvn -B clean verify
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build -f user-service/Dockerfile \
                      -t $ECR_REGISTRY/user-service:$IMAGE_TAG .

                    docker build -f product-service/Dockerfile \
                      -t $ECR_REGISTRY/product-service:$IMAGE_TAG .

                    docker build -f order-service/Dockerfile \
                      -t $ECR_REGISTRY/order-service:$IMAGE_TAG .
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password --region $AWS_REGION |
                    docker login --username AWS --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    docker push $ECR_REGISTRY/user-service:$IMAGE_TAG
                    docker push $ECR_REGISTRY/product-service:$IMAGE_TAG
                    docker push $ECR_REGISTRY/order-service:$IMAGE_TAG
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout $ECR_REGISTRY || true'
        }

        success {
            echo 'All three microservices were successfully pushed to ECR.'
        }

        failure {
            echo 'Pipeline failed. Check the failed stage in Jenkins console output.'
        }
    }
}

