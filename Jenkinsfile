pipeline {

    agent any

    environment {
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '464604123652'
        ECR_REPO       = 'usea-homework2'

        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_IMAGE    = "${ECR_REGISTRY}/${ECR_REPO}"

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

        stage('Check Docker & AWS') {
            steps {
                sh '''
                    export PATH="/usr/local/bin:/opt/homebrew/bin:$PATH"

                    echo "===== Docker ====="
                    which docker
                    docker --version

                    echo "===== Docker Buildx ====="
                    docker buildx version

                    echo "===== AWS CLI ====="
                    which aws
                    aws --version
                '''
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    export PATH="/usr/local/bin:/opt/homebrew/bin:$PATH"

                    aws ecr get-login-password \
                      --region ${AWS_REGION} \
                    | docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}
                '''
            }
        }

        stage('Build & Push Multi-Platform') {
            steps {
                sh '''
                    export PATH="/usr/local/bin:/opt/homebrew/bin:$PATH"

                    echo "Building image:"
                    echo "${ECR_IMAGE}:${IMAGE_TAG}"

                    echo "Platforms:"
                    echo "linux/amd64"
                    echo "linux/arm64"

                    docker buildx build \
                      --platform linux/amd64,linux/arm64 \
                      -t ${ECR_IMAGE}:${IMAGE_TAG} \
                      --push \
                      .
                '''
            }
        }

        stage('Verify ECR Image') {
            steps {
                sh '''
                    export PATH="/usr/local/bin:/opt/homebrew/bin:$PATH"

                    echo "===== Image Manifest ====="

                    docker buildx imagetools inspect \
                      ${ECR_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Prepare Docker Stack') {
            steps {
                sh '''
                    sed \
                      "s|IMAGE_PLACEHOLDER|${ECR_IMAGE}:${IMAGE_TAG}|g" \
                      docker-stack.yml \
                      > docker-stack-deploy.yml

                    echo "===== Docker Stack ====="
                    cat docker-stack-deploy.yml
                '''
            }
        }
        stage('Test SSH') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no \
                            ubuntu@184.73.20.152 \
                            "whoami && hostname"
                    '''
                }
            }
        }
        stage('Copy Stack to Swarm Manager') {
            steps {
                sh '''
                    scp \
                      -o StrictHostKeyChecking=no \
                      docker-stack-deploy.yml \
                      ${MANAGER_USER}@${MANAGER_HOST}:/home/${MANAGER_USER}/projects/docker-stack.yml
                '''
            }
        }

        stage('Deploy Stack') {
            steps {
                sshagent(['ec2-ssh-key']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no \
                            docker-stack-deploy.yml \
                            ubuntu@184.73.20.152:/home/ubuntu/projects/docker-stack.yml
                    '''
                }
            }
        }

        stage('Verify Swarm Services') {
            steps {
                sh '''
                    ssh \
                      -o StrictHostKeyChecking=no \
                      ${MANAGER_USER}@${MANAGER_HOST} \
                      "docker service ls"
                '''

                sh '''
                    ssh \
                      -o StrictHostKeyChecking=no \
                      ${MANAGER_USER}@${MANAGER_HOST} \
                      "docker service ps usea-app_web --no-trunc"
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD deployment completed successfully.'
            echo "Image: ${ECR_IMAGE}:${IMAGE_TAG}"
            echo 'Platforms: linux/amd64, linux/arm64'
        }

        failure {
            echo 'CI/CD pipeline failed.'
        }

        always {
            sh '''
                rm -f docker-stack-deploy.yml || true
            '''
        }
    }
}
