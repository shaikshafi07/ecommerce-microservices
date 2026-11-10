
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

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
            }
        }

        stage('Prepare Temporary JWT Test Keys') {
            steps {
                sh '''
                    set -eu

                    mkdir -p keys

                    openssl genpkey -algorithm RSA \
                      -pkeyopt rsa_keygen_bits:2048 \
                      -out keys/jwt-private.pem

                    openssl pkey \
                      -in keys/jwt-private.pem \
                      -pubout \
                      -out keys/jwt-public.pem
                '''
            }
        }

        stage('Maven Build and Test') {
            steps {
                sh '''
                    set -eu
                    mvn -B clean verify
                '''
            }
        }

        stage('Remove Temporary Test Keys') {
            steps {
                sh '''
                    rm -f keys/jwt-private.pem keys/jwt-public.pem
                    rmdir keys 2>/dev/null || true
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -eu

                    docker build \
                      -f user-service/Dockerfile \
                      -t $ECR_REGISTRY/user-service:$IMAGE_TAG .

                    docker build \
                      -f product-service/Dockerfile \
                      -t $ECR_REGISTRY/product-service:$IMAGE_TAG .

                    docker build \
                      -f order-service/Dockerfile \
                      -t $ECR_REGISTRY/order-service:$IMAGE_TAG .
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    set -eu

                    aws sts get-caller-identity

                    aws ecr get-login-password \
                      --region $AWS_REGION |
                    docker login \
                      --username AWS \
                      --password-stdin $ECR_REGISTRY
                '''
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh '''
                    set -eu

                    docker push $ECR_REGISTRY/user-service:$IMAGE_TAG
                    docker push $ECR_REGISTRY/product-service:$IMAGE_TAG
                    docker push $ECR_REGISTRY/order-service:$IMAGE_TAG
                '''
            }
        }
    }

    post {
        always {
            sh '''
                docker logout $ECR_REGISTRY 2>/dev/null || true
                rm -f keys/jwt-private.pem keys/jwt-public.pem
                rmdir keys 2>/dev/null || true
            '''
        }

        success {
            echo 'SUCCESS: All three microservices built and pushed to ECR.'
        }

        failure {
            echo 'FAILURE: Check the failed pipeline stage and console output.'
        }
    }
}

