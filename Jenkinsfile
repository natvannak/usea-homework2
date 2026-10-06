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

      // Use Jenkins Build Number as image tag
      IMAGE_TAG = "${BUILD_NUMBER}"

      // =====================================================
      // Docker Swarm Manager
      // =====================================================
      MANAGER_HOST = '184.73.20.152'
      MANAGER_USER = 'ubuntu'

      // =====================================================
      // macOS Jenkins PATH
      // Docker = /usr/local/bin/docker
      // Git    = /opt/homebrew/bin/git
      // AWS CLI = /opt/homebrew/bin/aws
      // =====================================================
      PATH = '/usr/local/bin:/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin'
  }

  stages {

      // =====================================================
      // 1. Checkout
      // =====================================================
      stage('Checkout') {
          steps {
              checkout scm
          }
      }

      // =====================================================
      // 2. Check Tools
      // =====================================================
      stage('Check Docker & AWS') {
          steps {
              sh '''
                  set -e

                  echo "======================================"
                  echo "Checking Docker"
                  echo "======================================"

                  which docker
                  docker --version

                  echo ""
                  echo "======================================"
                  echo "Checking Docker Buildx"
                  echo "======================================"

                  docker buildx version

                  echo ""
                  echo "======================================"
                  echo "Checking Git"
                  echo "======================================"

                  which git
                  git --version

                  echo ""
                  echo "======================================"
                  echo "Checking AWS CLI"
                  echo "======================================"

                  which aws
                  aws --version
              '''
          }
      }

      // =====================================================
      // 3. ECR Login
      // =====================================================
      stage('ECR Login') {
          steps {
              sh '''
                  set -e

                  echo "Logging in to Amazon ECR..."

                  aws ecr get-login-password \
                      --region ${AWS_REGION} \
                  | docker login \
                      --username AWS \
                      --password-stdin ${ECR_REGISTRY}

                  echo "ECR login successful."
              '''
          }
      }

      // =====================================================
      // 4. Build & Push Multi-Platform Image
      // =====================================================
      stage('Build & Push Multi-Platform') {
          steps {
              sh '''
                  set -e

                  echo "======================================"
                  echo "Building Docker Image"
                  echo "======================================"

                  echo "Image:"
                  echo "${ECR_IMAGE}:${IMAGE_TAG}"

                  echo ""
                  echo "Platforms:"
                  echo "linux/amd64"
                  echo "linux/arm64"

                  docker buildx build \
                      --platform linux/amd64,linux/arm64 \
                      -t ${ECR_IMAGE}:${IMAGE_TAG} \
                      --push \
                      .

                  echo ""
                  echo "Docker image pushed successfully."
              '''
          }
      }

      // =====================================================
      // 5. Verify ECR Image
      // =====================================================
      stage('Verify ECR Image') {
          steps {
              sh '''
                  set -e

                  echo "======================================"
                  echo "Verifying ECR Image"
                  echo "======================================"

                  docker buildx imagetools inspect \
                      ${ECR_IMAGE}:${IMAGE_TAG}
              '''
          }
      }

      // =====================================================
      // 6. Prepare Docker Stack
      // =====================================================
      stage('Prepare Docker Stack') {
          steps {
              sh '''
                  set -e

                  echo "======================================"
                  echo "Preparing Docker Stack"
                  echo "======================================"

                  if [ ! -f docker-stack.yml ]; then
                      echo "ERROR: docker-stack.yml not found."
                      exit 1
                  fi

                  sed \
                      "s|IMAGE_PLACEHOLDER|${ECR_IMAGE}:${IMAGE_TAG}|g" \
                      docker-stack.yml \
                      > docker-stack-deploy.yml

                  echo ""
                  echo "===== Generated Docker Stack ====="

                  cat docker-stack-deploy.yml
              '''
          }
      }

      // =====================================================
      // 7. Test SSH
      // =====================================================
      stage('Test SSH') {
        steps {
        sshagent(['ec2-ssh-key']) {
        sh '''
        set -e

                    echo "======================================"
                    echo "TEST SSH CONNECTION"
                    echo "======================================"

                    echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK"

                    echo ""
                    echo "Checking SSH key..."
                    ssh-add -l

                    echo ""
                    echo "Testing connection to:"
                    echo "ubuntu@184.73.20.152"

                    ssh \
                        -o StrictHostKeyChecking=no \
                        -o UserKnownHostsFile=/dev/null \
                        -o ConnectTimeout=10 \
                        -o BatchMode=yes \
                        ubuntu@184.73.20.152 \
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
      // 8. Copy Stack to Swarm Manager
      // =====================================================
      stage('Copy Stack to Swarm Manager') {
          steps {
              sshagent(['ec2-ssh-key']) {
                  sh '''
                      set -e

                      echo "======================================"
                      echo "Copying Docker Stack"
                      echo "======================================"

                      ssh \
                          -o StrictHostKeyChecking=no \
                          ${MANAGER_USER}@${MANAGER_HOST} \
                          "mkdir -p /home/${MANAGER_USER}/projects"

                      scp \
                          -o StrictHostKeyChecking=no \
                          docker-stack-deploy.yml \
                          ${MANAGER_USER}@${MANAGER_HOST}:/home/${MANAGER_USER}/projects/docker-stack.yml

                      echo ""
                      echo "Docker stack copied successfully."
                  '''
              }
          }
      }

      // =====================================================
      // 9. Deploy Stack
      // =====================================================
      stage('Deploy Stack') {
          steps {
              sshagent(['ec2-ssh-key']) {
                  sh '''
                      set -e

                      echo "======================================"
                      echo "Deploying Docker Swarm Stack"
                      echo "======================================"

                      ssh \
                          -o StrictHostKeyChecking=no \
                          ${MANAGER_USER}@${MANAGER_HOST} \
                          "docker stack deploy \
                              -c /home/${MANAGER_USER}/projects/docker-stack.yml \
                              usea-app"

                      echo ""
                      echo "Docker Swarm stack deployed successfully."
                  '''
              }
          }
      }

      // =====================================================
      // 10. Verify Swarm Services
      // =====================================================
      stage('Verify Swarm Services') {
          steps {
              sshagent(['ec2-ssh-key']) {
                  sh '''
                      set -e

                      echo "======================================"
                      echo "Docker Swarm Services"
                      echo "======================================"

                      ssh \
                          -o StrictHostKeyChecking=no \
                          ${MANAGER_USER}@${MANAGER_HOST} \
                          "docker service ls"

                      echo ""
                      echo "======================================"
                      echo "USEA APP Service"
                      echo "======================================"

                      ssh \
                          -o StrictHostKeyChecking=no \
                          ${MANAGER_USER}@${MANAGER_HOST} \
                          "docker service ps usea-app_web --no-trunc"
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
          echo 'CI/CD Deployment Successful'
          echo '======================================'

          echo "Image: ${ECR_IMAGE}:${IMAGE_TAG}"
          echo "Platforms: linux/amd64, linux/arm64"
          echo "Swarm Manager: ${MANAGER_USER}@${MANAGER_HOST}"
          echo "Stack: usea-app"
      }

      failure {
          echo '======================================'
          echo 'CI/CD Pipeline Failed'
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
