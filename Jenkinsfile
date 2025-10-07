pipeline {
    agent any
    tools {
        maven 'Maven 3.8.1'
    }
    environment {
        DOCKER_IMAGE = "wiiwake3101/spring-petclinic:${BUILD_NUMBER}"
    }
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/WiiWake3101/spring-petclinic'
            }
        }
        stage('Start Databases') {
            steps {
                sh 'docker-compose down -v || true'
                sh 'docker-compose up -d'
                sh 'sleep 20'
            }
        }
        stage('Build with Maven') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_IMAGE .'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-creds']) {
                    sh 'docker push $DOCKER_IMAGE'
                }
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl set image deployment/sample-app-deployment sample-container=$DOCKER_IMAGE'
            }
        }
    }
    post {
        always {
            sh 'docker compose down -v || true'
        }
    }
}