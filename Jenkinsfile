pipeline {

agent any

environment {

    // =====================================================
    // AWS / ECR Configuration
    // =====================================================
    AWS_REGION     = 'us-east-1'
    AWS_ACCOUNT_ID = '464604123652'
    ECR_REPO       = 'usea-homework2'

    ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
    ECR_IMAGE    = "${ECR_REGISTRY}/${ECR_REPO}"

    // Jenkins Build Number
    IMAGE_TAG = "${BUILD_NUMBER}"

    // =====================================================
    // Docker Swarm Manager
    // =====================================================
    MANAGER_HOST = '184.73.20.152'
    MANAGER_USER = 'ubuntu'

    // =====================================================
    // Jenkins macOS PATH
    // =====================================================
    PATH = '/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin'
}


stages {

    // =====================================================
    // 1. CHECKOUT
    // =====================================================
    stage('Checkout') {
        steps {
            echo '======================================'
            echo 'CHECKOUT SOURCE CODE'
            echo '======================================'

            checkout scm
        }
    }


    // =====================================================
    // 2. CHECK DOCKER / GIT / AWS
    // =====================================================
    stage('Check Tools') {
        steps {
            sh '''
                set -e

                echo "======================================"
                echo "CHECKING REQUIRED TOOLS"
                echo "======================================"

                echo ""
                echo "===== Git ====="
                which git
                git --version

                echo ""
                echo "===== Docker ====="
                which docker
                docker --version

                echo ""
                echo "===== Docker Buildx ====="
                docker buildx version

                echo ""
                echo "===== AWS CLI ====="
                which aws
                aws --version

                echo ""
                echo "======================================"
                echo "ALL TOOLS ARE AVAILABLE"
                echo "======================================"
            '''
        }
    }


    // =====================================================
    // 3. CHECK AWS IDENTITY
    // =====================================================
    stage('Check AWS Identity') {
        steps {
            sh '''
                set -e

                echo "======================================"
                echo "AWS IDENTITY"
                echo "======================================"

                aws sts get-caller-identity \
                    --region ${AWS_REGION}
            '''
        }
    }


    // =====================================================
    // 4. CHECK ECR REPOSITORY
    // =====================================================
    stage('Check ECR Repository') {
        steps {
            sh '''
                set -e

                echo "======================================"
                echo "CHECK ECR REPOSITORY"
                echo "======================================"

                aws ecr describe-repositories \
                    --repository-names ${ECR_REPO} \
                    --region ${AWS_REGION}

                echo ""
                echo "ECR repository exists:"
                echo "${ECR_IMAGE}"
            '''
        }
    }


    // =====================================================
    // 5. ECR LOGIN
    // =====================================================
    stage('ECR Login') {
        steps {
            sh '''
                set -e

                echo "======================================"
                echo "LOGIN TO AMAZON ECR"
                echo "======================================"

                aws ecr get-login-password \
                    --region ${AWS_REGION} \
                | docker login \
                    --username AWS \
                    --password-stdin ${ECR_REGISTRY}

                echo ""
                echo "ECR login successful."
            '''
        }
    }


    // =====================================================
    // 6. CHECK DOCKER BUILDx
    // =====================================================
    stage('Check Buildx') {
        steps {
            sh '''
                set -e

                echo "======================================"
                echo "DOCKER BUILDX"
                echo "======================================"

                docker buildx ls

                echo ""
                echo "Inspecting current builder..."

                docker buildx inspect --bootstrap
            '''
        }
    }


    // =====================================================
    // 7. BUILD & PUSH MULTI-PLATFORM IMAGE
    // =====================================================
    stage('Build & Push Multi-Platform') {
        steps {
            sh '''
                set -e

                echo "======================================"
                echo "BUILD & PUSH DOCKER IMAGE"
                echo "======================================"

                echo "Image:"
                echo "${ECR_IMAGE}:${IMAGE_TAG}"

                echo ""
                echo "Platforms:"
                echo "  - linux/amd64"
                echo "  - linux/arm64"

                echo ""

                docker buildx build \
                    --platform linux/amd64,linux/arm64 \
                    --tag ${ECR_IMAGE}:${IMAGE_TAG} \
                    --push \
                    .

                echo ""
                echo "======================================"
                echo "IMAGE PUSH SUCCESSFUL"
                echo "======================================"
            '''
        }
    }


    // =====================================================
    // 8. VERIFY ECR IMAGE
    // =====================================================
    stage('Verify ECR Image') {
        steps {
            sh '''
                set -e

                echo "======================================"
                echo "VERIFY ECR IMAGE"
                echo "======================================"

                docker buildx imagetools inspect \
                    ${ECR_IMAGE}:${IMAGE_TAG}

                echo ""
                echo "ECR image verified:"
                echo "${ECR_IMAGE}:${IMAGE_TAG}"
            '''
        }
    }


    // =====================================================
    // 9. PREPARE DOCKER STACK
    // =====================================================
    stage('Prepare Docker Stack') {
        steps {
            sh '''
                set -e

                echo "======================================"
                echo "PREPARE DOCKER STACK"
                echo "======================================"

                if [ ! -f docker-stack.yml ]; then
                    echo "ERROR: docker-stack.yml not found."
                    exit 1
                fi

                if ! grep -q "IMAGE_PLACEHOLDER" docker-stack.yml; then
                    echo "ERROR: IMAGE_PLACEHOLDER not found in docker-stack.yml."
                    echo "Please use:"
                    echo "image: IMAGE_PLACEHOLDER"
                    exit 1
                fi

                sed \
                    "s|IMAGE_PLACEHOLDER|${ECR_IMAGE}:${IMAGE_TAG}|g" \
                    docker-stack.yml \
                    > docker-stack-deploy.yml

                echo ""
                echo "===== Generated docker-stack-deploy.yml ====="

                cat docker-stack-deploy.yml

                echo ""
                echo "======================================"
                echo "DOCKER STACK READY"
                echo "======================================"
            '''
        }
    }


    // =====================================================
    // 10. TEST SSH
    // =====================================================
    stage('Test SSH') {
        steps {

            sshagent(['ec2-ssh-key']) {

                sh '''
                    set -e

                    echo "======================================"
                    echo "TEST SSH CONNECTION"
                    echo "======================================"

                    echo ""
                    echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK"

                    echo ""
                    echo "Checking SSH key..."

                    ssh-add -l

                    echo ""
                    echo "Connecting to:"
                    echo "${MANAGER_USER}@${MANAGER_HOST}"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        -o ConnectTimeout=10 \
                        -o BatchMode=yes \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "echo SSH_SUCCESS && whoami && hostname"

                    echo ""
                    echo "======================================"
                    echo "SSH TEST PASSED"
                    echo "======================================"
                '''
            }
        }
    }


    // =====================================================
    // 11. PREPARE SWARM MANAGER DIRECTORY
    // =====================================================
    stage('Prepare Swarm Manager') {
        steps {

            sshagent(['ec2-ssh-key']) {

                sh '''
                    set -e

                    echo "======================================"
                    echo "PREPARE SWARM MANAGER"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "mkdir -p /home/${MANAGER_USER}/projects"

                    echo ""
                    echo "Manager directory ready."
                '''
            }
        }
    }


    // =====================================================
    // 12. COPY DOCKER STACK
    // =====================================================
    stage('Copy Stack to Swarm Manager') {
        steps {

            sshagent(['ec2-ssh-key']) {

                sh '''
                    set -e

                    echo "======================================"
                    echo "COPY DOCKER STACK TO SWARM MANAGER"
                    echo "======================================"

                    scp \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        docker-stack-deploy.yml \
                        ${MANAGER_USER}@${MANAGER_HOST}:/home/${MANAGER_USER}/projects/docker-stack.yml

                    echo ""
                    echo "Docker stack copied successfully."
                '''
            }
        }
    }


    // =====================================================
    // 13. VERIFY STACK FILE ON MANAGER
    // =====================================================
    stage('Verify Stack on Manager') {
        steps {

            sshagent(['ec2-ssh-key']) {

                sh '''
                    set -e

                    echo "======================================"
                    echo "VERIFY STACK FILE ON MANAGER"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "cat /home/${MANAGER_USER}/projects/docker-stack.yml"

                    echo ""
                    echo "Stack file verified on manager."
                '''
            }
        }
    }


    // =====================================================
    // 14. CHECK SWARM
    // =====================================================
    stage('Check Docker Swarm') {
        steps {

            sshagent(['ec2-ssh-key']) {

                sh '''
                    set -e

                    echo "======================================"
                    echo "CHECK DOCKER SWARM"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "docker info --format '{{.Swarm.LocalNodeState}}'"

                    echo ""
                    echo "Swarm status checked."
                '''
            }
        }
    }


    // =====================================================
    // 15. DEPLOY STACK
    // =====================================================
    stage('Deploy Stack') {
        steps {

            sshagent(['ec2-ssh-key']) {

                sh '''
                    set -e

                    echo "======================================"
                    echo "DEPLOY DOCKER SWARM STACK"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "docker stack deploy \
                            --with-registry-auth \
                            -c /home/${MANAGER_USER}/projects/docker-stack.yml \
                            usea-app"

                    echo ""
                    echo "======================================"
                    echo "STACK DEPLOYED"
                    echo "======================================"
                '''
            }
        }
    }


    // =====================================================
    // 16. VERIFY SWARM SERVICES
    // =====================================================
    stage('Verify Swarm Services') {
        steps {

            sshagent(['ec2-ssh-key']) {

                sh '''
                    set -e

                    echo "======================================"
                    echo "DOCKER SWARM SERVICES"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "docker service ls"

                    echo ""
                    echo "======================================"
                    echo "USEA APP WEB SERVICE"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "docker service ps usea-app_web --no-trunc"
                '''
            }
        }
    }


    // =====================================================
    // 17. CHECK SERVICE STATUS
    // =====================================================
    stage('Check Service Status') {
        steps {

            sshagent(['ec2-ssh-key']) {

                sh '''
                    set -e

                    echo "======================================"
                    echo "CHECK SERVICE STATUS"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "docker service inspect \
                            usea-app_web \
                            --format '{{.Spec.TaskTemplate.ContainerSpec.Image}}'"

                    echo ""
                    echo "Expected image:"
                    echo "${ECR_IMAGE}:${IMAGE_TAG}"
                '''
            }
        }
    }


    // =====================================================
    // 18. SHOW FINAL STATUS
    // =====================================================
    stage('Final Verification') {
        steps {

            sshagent(['ec2-ssh-key']) {

                sh '''
                    set -e

                    echo "======================================"
                    echo "FINAL DEPLOYMENT VERIFICATION"
                    echo "======================================"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        ${MANAGER_USER}@${MANAGER_HOST} \
                        "docker service ls"

                    echo ""
                    echo "======================================"
                    echo "IMAGE"
                    echo "======================================"

                    echo "${ECR_IMAGE}:${IMAGE_TAG}"

                    echo ""
                    echo "======================================"
                    echo "PLATFORMS"
                    echo "======================================"

                    echo "linux/amd64"
                    echo "linux/arm64"

                    echo ""
                    echo "======================================"
                    echo "DEPLOYMENT COMPLETE"
                    echo "======================================"
                '''
            }
        }
    }
}


// =========================================================
// POST ACTIONS
// =========================================================
post {

    success {

        echo '======================================'
        echo 'CI/CD DEPLOYMENT SUCCESSFUL'
        echo '======================================'

        echo "Image: ${ECR_IMAGE}:${IMAGE_TAG}"
        echo "Platforms: linux/amd64, linux/arm64"
        echo "Swarm Manager: ${MANAGER_USER}@${MANAGER_HOST}"
        echo "Stack: usea-app"
        echo "Service: usea-app_web"
    }

    failure {

        echo '======================================'
        echo 'CI/CD PIPELINE FAILED'
        echo '======================================'

        echo 'Please check the failed stage and Jenkins console log.'
    }

    always {

        sh '''
            rm -f docker-stack-deploy.yml || true
        '''
    }
}
}
