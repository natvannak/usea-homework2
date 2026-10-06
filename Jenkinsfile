pipeline {

    agent any

    environment {
        AWS_REGION = 'us-east-1'
        AWS_ACCOUNT_ID = '464604123652'
        ECR_REPO = 'usea-homework2'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_IMAGE = "${ECR_REGISTRY}/${ECR_REPO}"

        IMAGE_TAG = "${BUILD_NUMBER}"

        MANAGER_HOST = '184.73.20.152'
        MANAGER_USER = 'ubuntu'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh '''
                    docker build \
                      -t ${ECR_IMAGE}:${IMAGE_TAG} .
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password \
                      --region ${AWS_REGION} \
                    | docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Push') {
            steps {
                sh '''
                    docker push ${ECR_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy') {
            steps {

                sh '''
                    sed \
                      "s|IMAGE_PLACEHOLDER|${ECR_IMAGE}:${IMAGE_TAG}|g" \
                      docker-stack.yml > docker-stack-deploy.yml
                '''

                sh '''
                    scp \
                      -o StrictHostKeyChecking=no \
                      docker-stack-deploy.yml \
                      ${MANAGER_USER}@${MANAGER_HOST}:/home/${MANAGER_USER}/docker-stack.yml
                '''

                sh '''
                    ssh \
                      -o StrictHostKeyChecking=no \
                      ${MANAGER_USER}@${MANAGER_HOST} \
                      "docker stack deploy \
                       --with-registry-auth \
                       -c /home/${MANAGER_USER}/docker-stack.yml \
                       usea-app"
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD deployment completed successfully.'
        }

        failure {
            echo 'CI/CD pipeline failed.'
        }
    }
}