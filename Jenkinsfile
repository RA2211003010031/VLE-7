pipeline {
    agent any
    tools {
        maven 'Maven 3.8.1'
    }
    environment {
        DOCKER_IMAGE = "wiiwake3101/spring-petclinic:${BUILD_NUMBER}"
        DEPLOY_COLOR = "blue"
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
                sh 'docker build -t $DOCKER_IMAGE -t wiiwake3101/spring-petclinic:latest .'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-creds', url: 'https://index.docker.io/v1/']) {
                    sh 'docker push $DOCKER_IMAGE'
                    sh 'docker push wiiwake3101/spring-petclinic:latest'
                }
            }
        }
        stage('Deploy to Kubernetes') {
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
                AWS_DEFAULT_REGION = 'ap-south-1'
            }
            steps {
                withKubeConfig([credentialsId: 'kubeconfig-creds']) {
                    script {
                        if (env.DEPLOY_COLOR == "blue") {
                            sh 'kubectl apply -f k8s/petclinic-blue-deployment.yml'
                            sh 'kubectl apply -f k8s/petclinic-blue-service.yml'
                        } else {
                            sh 'kubectl apply -f k8s/petclinic-green-deployment.yml'
                            sh 'kubectl apply -f k8s/petclinic-green-service.yml'
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            sh 'docker compose down -v || true'
        }
    }
}