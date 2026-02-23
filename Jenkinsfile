pipeline {
    agent any

    environment {
        DOCKER_USERNAME     = credentials('docker-hub-username')
        DOCKER_PASSWORD     = credentials('docker-hub-password')
        IMAGE_TAG           = "${env.BUILD_NUMBER}"
        BACKEND_IMAGE       = "${DOCKER_USERNAME}/mean-backend:${IMAGE_TAG}"
        FRONTEND_IMAGE      = "${DOCKER_USERNAME}/mean-frontend:${IMAGE_TAG}"
        VM_SSH_CREDENTIALS  = 'aws-vm-ssh-key'
        VM_HOST             = credentials('vm-host-ip')
        VM_USER             = 'ubuntu'
        DEPLOY_PATH         = '/home/ubuntu/mean-app'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out source code from GitHub..."
                checkout scm
            }
        }

        stage('Docker Login') {
            steps {
                echo "Logging in to Docker Hub..."
                sh '''
                    echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                '''
            }
        }

        stage('Build Backend Image') {
            steps {
                echo "Building backend Docker image: ${env.BACKEND_IMAGE}"
                dir('backend') {
                    sh '''
                        docker build -t $DOCKER_USERNAME/mean-backend:$IMAGE_TAG \
                                     -t $DOCKER_USERNAME/mean-backend:latest .
                    '''
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                echo "Building frontend Docker image: ${env.FRONTEND_IMAGE}"
                dir('frontend') {
                    sh '''
                        docker build -t $DOCKER_USERNAME/mean-frontend:$IMAGE_TAG \
                                     -t $DOCKER_USERNAME/mean-frontend:latest .
                    '''
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                echo "Pushing images to Docker Hub..."
                sh '''
                    docker push $DOCKER_USERNAME/mean-backend:$IMAGE_TAG
                    docker push $DOCKER_USERNAME/mean-backend:latest
                    docker push $DOCKER_USERNAME/mean-frontend:$IMAGE_TAG
                    docker push $DOCKER_USERNAME/mean-frontend:latest
                '''
            }
        }

        stage('Deploy to AWS VM') {
            steps {
                echo "Deploying to AWS EC2 via SSH..."
                sshagent(credentials: ["${VM_SSH_CREDENTIALS}"]) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${VM_USER}@${VM_HOST} << 'ENDSSH'
                            set -e

                            echo "==> Navigating to deploy path"
                            mkdir -p $DEPLOY_PATH && cd $DEPLOY_PATH

                            echo "==> Pulling latest docker-compose config from GitHub (optional: or use git pull)"
                            # If you have git set up on the VM:
                            # git pull origin main

                            echo "==> Logging in to Docker Hub"
                            echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin

                            echo "==> Pulling latest images"
                            DOCKER_USERNAME=$DOCKER_USERNAME IMAGE_TAG=latest docker compose pull

                            echo "==> Restarting containers"
                            DOCKER_USERNAME=$DOCKER_USERNAME IMAGE_TAG=latest docker compose up -d --remove-orphans

                            echo "==> Cleaning up old images"
                            docker image prune -f

                            echo "==> Deployment complete!"
ENDSSH
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo "Pipeline completed successfully! Build #${env.BUILD_NUMBER} is live."
        }
        failure {
            echo "Pipeline FAILED at build #${env.BUILD_NUMBER}. Check logs above."
        }
    }
}
