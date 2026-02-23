pipeline {
    agent any

    environment {
        DOCKER_USERNAME = credentials('docker-hub-username')
        DOCKER_PASSWORD = credentials('docker-hub-password')
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out code from GitHub..."
                checkout scm
            }
        }

        stage('Docker Login') {
            steps {
                echo "Logging in to Docker Hub..."
                sh 'echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin'
            }
        }

        stage('Build Backend Image') {
            steps {
                echo "Building backend image..."
                sh """
                    docker build -t $DOCKER_USERNAME/mean-backend:$IMAGE_TAG -t $DOCKER_USERNAME/mean-backend:latest ./backend
                """
            }
        }

        stage('Build Frontend Image') {
            steps {
                echo "Building frontend image..."
                sh """
                    docker build -t $DOCKER_USERNAME/mean-frontend:$IMAGE_TAG -t $DOCKER_USERNAME/mean-frontend:latest ./frontend
                """
            }
        }

        stage('Push Images') {
            steps {
                echo "Pushing images to Docker Hub..."
                sh """
                    docker push $DOCKER_USERNAME/mean-backend:$IMAGE_TAG
                    docker push $DOCKER_USERNAME/mean-backend:latest
                    docker push $DOCKER_USERNAME/mean-frontend:$IMAGE_TAG
                    docker push $DOCKER_USERNAME/mean-frontend:latest
                """
            }
        }

        stage('Deploy') {
            steps {
                echo "Deploying with Docker Compose..."
                sh """
                    DOCKER_USERNAME=$DOCKER_USERNAME IMAGE_TAG=latest docker compose pull
                    DOCKER_USERNAME=$DOCKER_USERNAME IMAGE_TAG=latest docker compose up -d --remove-orphans
                """
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo "Build #${env.BUILD_NUMBER} deployed successfully!"
        }
        failure {
            echo "Build #${env.BUILD_NUMBER} failed!"
        }
    }
}