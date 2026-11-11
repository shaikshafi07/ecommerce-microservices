
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '415081250301'
        ECR_REGISTRY = '415081250301.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_TAG = "${BUILD_NUMBER}"

        APP_DIR = '/home/ubuntu/microservices-project/ecommerce-microservices'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Configuration Validation') {
            steps {
                sh '''
                    set -eu

                    git diff --check

                    test -f user-service/Dockerfile
                    test -f product-service/Dockerfile
                    test -f order-service/Dockerfile
                    test -f docker-compose.yml
                    test -f nginx/nginx.conf

                    docker version
                    docker compose version
                '''
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
                sh 'mvn -B clean verify'
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    set -eu

                   	 docker build -f user-service/Dockerfile \
            		-t ${ECR_REGISTRY}/user-service:${BUILD_NUMBER} user-service

			docker build -f product-service/Dockerfile \
 			 -t ${ECR_REGISTRY}/product-service:${BUILD_NUMBER} product-service

			docker build -f order-service/Dockerfile \
  			-t ${ECR_REGISTRY}/order-service:${BUILD_NUMBER} order-service               
		'''
            }
        }

        stage('ECR Authentication') {
            steps {
                sh '''
                    set -eu

                    aws sts get-caller-identity

                    aws ecr describe-repositories \
                      --region "$AWS_REGION" \
                      --repository-names \
                      user-service product-service order-service

                    aws ecr get-login-password \
                      --region "$AWS_REGION" |
                    docker login \
                      --username AWS \
                      --password-stdin "$ECR_REGISTRY"
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

       stage('Deploy to EC2') {
          steps {
          sh '''
            set -eu

            DEPLOY_ENV_FILE="/tmp/ecommerce-deploy-${BUILD_NUMBER}.env"

            trap 'rm -f "$DEPLOY_ENV_FILE"' EXIT

            printf 'ECR_REGISTRY=%s\\nIMAGE_TAG=%s\\n' \
              "$ECR_REGISTRY" "$IMAGE_TAG" > "$DEPLOY_ENV_FILE"

            chmod 644 "$DEPLOY_ENV_FILE"

            # Authenticate ubuntu user with ECR
            aws ecr get-login-password \
              --region "$AWS_REGION" |
            sudo -n -u ubuntu docker login \
              --username AWS \
              --password-stdin "$ECR_REGISTRY"

            # Pull images from ECR
            sudo -n -u ubuntu docker compose \
              --env-file "$APP_DIR/.env" \
              --env-file "$DEPLOY_ENV_FILE" \
              -f "$APP_DIR/docker-compose.yml" \
              --project-directory "$APP_DIR" \
              --profile apps pull \
              user-service product-service order-service

            # Deploy the images
            sudo -n -u ubuntu docker compose \
              --env-file "$APP_DIR/.env" \
              --env-file "$DEPLOY_ENV_FILE" \
              -f "$APP_DIR/docker-compose.yml" \
              --project-directory "$APP_DIR" \
              --profile apps up -d --no-build \
              user-service product-service order-service nginx

            # Verify service health
            for service in user-service product-service order-service; do
                container_id=$(sudo -n -u ubuntu docker compose \
                  --env-file "$APP_DIR/.env" \
                  --env-file "$DEPLOY_ENV_FILE" \
                  -f "$APP_DIR/docker-compose.yml" \
                  --project-directory "$APP_DIR" \
                  --profile apps ps -q "$service")

                test -n "$container_id"

                healthy=false

                for attempt in $(seq 1 60); do
                    status=$(sudo -n -u ubuntu docker inspect \
                      --format '{{.State.Health.Status}}' \
                      "$container_id")

                    if [ "$status" = "healthy" ]; then
                        healthy=true
                        break
                    fi

                    if [ "$status" = "unhealthy" ]; then
                        break
                    fi

                    sleep 5
                done

                if [ "$healthy" != true ]; then
                    echo "$service health verification failed"
                    exit 1
                fi

                echo "$service is healthy"
                   done
             '''
           }
       }

       post {
          always {
            sh '''
                docker logout "$ECR_REGISTRY" 2>/dev/null || true
                sudo -n -u ubuntu docker logout "$ECR_REGISTRY" \
                  2>/dev/null || true

                rm -f keys/jwt-private.pem keys/jwt-public.pem
            '''
        }

        success {
            echo "SUCCESS: Images ${IMAGE_TAG} pushed and deployed."
        }

        failure {
            echo 'FAILURE: Review the failed pipeline stage and console log.'
        }
    }
}


