pipeline {
    agent any
    
    triggers {
        githubPush()
    }
    
    tools {
        maven 'Maven 3.8.1'
    }
    environment {
        DOCKER_IMAGE = "adarsh05122002/spring-petclinic:${BUILD_NUMBER}"
        DEPLOY_COLOR = "blue"
    }
    stages {
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
                sh 'docker build -t $DOCKER_IMAGE -t adarsh05122002/spring-petclinic:latest .'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                withDockerRegistry([credentialsId: 'docker-hub-creds', url: 'https://index.docker.io/v1/']) {
                    sh 'docker push $DOCKER_IMAGE'
                    sh 'docker push adarsh05122002/spring-petclinic:latest'
                }
            }
        }
        stage('Deploy to Kubernetes') {
            when {
                expression {
                    // Only run if kubeconfig credentials exist
                    try {
                        return credentials('kubeconfig-creds') != null
                    } catch (Exception e) {
                        echo "Kubeconfig credentials not found, skipping Kubernetes deployment"
                        return false
                    }
                }
            }
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
                AWS_DEFAULT_REGION = 'ap-south-1'
            }
            steps {
                script {
                    try {
                        withKubeConfig([credentialsId: 'kubeconfig-creds']) {
                            if (env.DEPLOY_COLOR == "blue") {
                                sh 'kubectl apply -f k8s/petclinic-blue-deployment.yml'
                                sh 'kubectl apply -f k8s/petclinic-service.yml'
                            } else {
                                sh 'kubectl apply -f k8s/petclinic-green-deployment.yml'
                                sh 'kubectl apply -f k8s/petclinic-service.yml'
                            }
                        }
                    } catch (Exception e) {
                        echo "Kubernetes deployment failed: ${e.getMessage()}"
                        echo "Continuing without Kubernetes deployment..."
                    }
                }
            }
        }
        stage('Switch Traffic') {
            when {
                expression {
                    // Only run if previous Kubernetes deployment was successful
                    return currentBuild.result != 'FAILURE'
                }
            }
            steps {
                script {
                    try {
                        withKubeConfig([credentialsId: 'kubeconfig-creds']) {
                            if (env.DEPLOY_COLOR == "blue") {
                                sh "kubectl patch service petclinic-service -p '{\"spec\": {\"selector\": {\"app\": \"petclinic\", \"color\": \"blue\"}}}'"
                            } else {
                                sh "kubectl patch service petclinic-service -p '{\"spec\": {\"selector\": {\"app\": \"petclinic\", \"color\": \"green\"}}}'"
                            }
                        }
                    } catch (Exception e) {
                        echo "Traffic switching failed: ${e.getMessage()}"
                        echo "Kubernetes deployment may not be available"
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
