pipeline {

```
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

    // Jenkins Credentials ID
    SSH_CREDENTIAL_ID = 'ec2-ssh-key'
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

                echo "======================================"
                echo " Docker"
                echo "======================================"

                which docker
                docker --version

                echo ""
                echo "======================================"
                echo " Docker Buildx"
                echo "======================================"

                docker buildx version

                echo ""
                echo "======================================"
                echo " AWS CLI"
                echo "======================================"

                which aws
                aws --version

                echo ""
                echo "======================================"
                echo " Build Information"
                echo "======================================"

                echo "AWS Region   : ${AWS_REGION}"
                echo "ECR Registry : ${ECR_REGISTRY}"
                echo "ECR Image    : ${ECR_IMAGE}"
                echo "Image Tag    : ${IMAGE_TAG}"
            '''
        }
    }

    stage('ECR Login') {
        steps {
            sh '''
                export PATH="/usr/local/bin:/opt/homebrew/bin:$PATH"

                echo "Logging in to Amazon ECR..."

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

                echo "======================================"
                echo " Build & Push Multi-Platform Image"
                echo "======================================"

                echo "Image:"
                echo "${ECR_IMAGE}:${IMAGE_TAG}"

                echo ""
                echo "Platforms:"
                echo " - linux/amd64"
                echo " - linux/arm64"

                docker buildx build \
                    --platform linux/amd64,linux/arm64 \
                    --tag ${ECR_IMAGE}:${IMAGE_TAG} \
                    --push \
                    .
            '''
        }
    }

    stage('Verify ECR Image') {
        steps {
            sh '''
                export PATH="/usr/local/bin:/opt/homebrew/bin:$PATH"

                echo "======================================"
                echo " Verify ECR Multi-Platform Image"
                echo "======================================"

                docker buildx imagetools inspect \
                    ${ECR_IMAGE}:${IMAGE_TAG}
            '''
        }
    }

    stage('Prepare Docker Stack') {
        steps {
            sh '''
                echo "======================================"
                echo " Prepare Docker Stack"
                echo "======================================"

                if [ ! -f docker-stack.yml ]; then
                    echo "ERROR: docker-stack.yml not found!"
                    exit 1
                fi

                sed \
                    "s|IMAGE_PLACEHOLDER|${ECR_IMAGE}:${IMAGE_TAG}|g" \
                    docker-stack.yml \
                    > docker-stack-deploy.yml

                echo ""
                echo "Generated docker-stack-deploy.yml:"
                echo "--------------------------------------"

                cat docker-stack-deploy.yml

                echo "--------------------------------------"
            '''
        }
    }

    stage('Test SSH Connection') {
        steps {
            sshagent(credentials: ["${SSH_CREDENTIAL_ID}"]) {
                sh '''
                    echo "======================================"
                    echo " Test SSH Connection"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o ConnectTimeout=15 \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "echo 'SSH connection successful' && whoami && hostname"
                '''
            }
        }
    }

    stage('Copy Stack to Swarm Manager') {
        steps {
            sshagent(credentials: ["${SSH_CREDENTIAL_ID}"]) {
                sh '''
                    echo "======================================"
                    echo " Copy Docker Stack"
                    echo "======================================"

                    scp \
                        -o StrictHostKeyChecking=no \
                        -o ConnectTimeout=15 \
                        docker-stack-deploy.yml \
                        ${MANAGER_USER}@${MANAGER_HOST}:/home/${MANAGER_USER}/docker-stack.yml
                '''
            }
        }
    }

    stage('Move Stack to /root/projects') {
        steps {
            sshagent(credentials: ["${SSH_CREDENTIAL_ID}"]) {
                sh '''
                    echo "======================================"
                    echo " Move Docker Stack"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "
                            sudo mkdir -p /root/projects &&
                            sudo mv /home/${MANAGER_USER}/docker-stack.yml /root/projects/docker-stack.yml &&
                            sudo chmod 644 /root/projects/docker-stack.yml &&
                            sudo ls -l /root/projects/docker-stack.yml
                        "
                '''
            }
        }
    }

    stage('Deploy to Docker Swarm') {
        steps {
            sshagent(credentials: ["${SSH_CREDENTIAL_ID}"]) {
                sh '''
                    echo "======================================"
                    echo " Deploy to Docker Swarm"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "
                            cd /root/projects &&
                            sudo docker stack deploy \
                                --with-registry-auth \
                                -c docker-stack.yml \
                                usea-app
                        "
                '''
            }
        }
    }

    stage('Verify Swarm Services') {
        steps {
            sshagent(credentials: ["${SSH_CREDENTIAL_ID}"]) {
                sh '''
                    echo "======================================"
                    echo " Docker Swarm Services"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "
                            sudo docker stack services usea-app
                        "

                    echo ""
                    echo "======================================"
                    echo " Docker Service Tasks"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "
                            sudo docker service ps usea-app_web --no-trunc
                        "
                '''
            }
        }
    }
}

post {

    success {
        echo '======================================'
        echo ' CI/CD DEPLOYMENT SUCCESSFUL'
        echo '======================================'
        echo "Image    : ${ECR_IMAGE}:${IMAGE_TAG}"
        echo "Platforms: linux/amd64, linux/arm64"
        echo "Swarm    : usea-app"
    }

    failure {
        echo '======================================'
        echo ' CI/CD PIPELINE FAILED'
```
