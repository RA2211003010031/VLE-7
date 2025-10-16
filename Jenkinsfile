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
                            echo "Deploying to Kubernetes..."
                            if (env.DEPLOY_COLOR == "blue") {
                                sh 'kubectl apply -f k8s/petclinic-blue-deployment.yml'
                                sh 'kubectl apply -f k8s/petclinic-service.yml'
                            } else {
                                sh 'kubectl apply -f k8s/petclinic-green-deployment.yml'
                                sh 'kubectl apply -f k8s/petclinic-service.yml'
                            }
                            echo "Kubernetes deployment completed successfully!"
                            
                            // Wait for load balancer to be ready and get external IP
                            echo "Waiting for LoadBalancer to get external IP..."
                            sh 'kubectl get service petclinic-service'
                            sh '''
                                echo "=== Checking LoadBalancer Status ==="
                                for i in {1..20}; do
                                    EXTERNAL_IP=$(kubectl get service petclinic-service -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' 2>/dev/null || echo "")
                                    if [ ! -z "$EXTERNAL_IP" ] && [ "$EXTERNAL_IP" != "" ]; then
                                        echo "✅ SUCCESS! LoadBalancer is ready!"
                                        echo "🌐 PetClinic Application URL: http://$EXTERNAL_IP"
                                        echo "📱 Access your Spring PetClinic at: http://$EXTERNAL_IP"
                                        echo "⏰ It may take 1-2 more minutes for the application to be fully ready"
                                        break
                                    else
                                        echo "⏳ Waiting for external IP... (attempt $i/20) - LoadBalancer provisioning..."
                                        sleep 30
                                    fi
                                done
                                
                                # Show current status regardless
                                echo "=== Current Service Status ==="
                                kubectl get service petclinic-service -o wide
                                
                                # Show deployment status
                                echo "=== Deployment Status ==="
                                kubectl get deployment petclinic-blue
                                kubectl get pods -l app=petclinic
                            '''
                        }
                    } catch (Exception e) {
                        echo "Kubernetes deployment failed: ${e.getMessage()}"
                        echo "Continuing without Kubernetes deployment..."
                        currentBuild.result = 'UNSTABLE'
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
            environment {
                AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
                AWS_DEFAULT_REGION = 'ap-south-1'
            }
            steps {
                script {
                    try {
                        withKubeConfig([credentialsId: 'kubeconfig-creds']) {
                            echo "Switching traffic to ${env.DEPLOY_COLOR} deployment..."
                            if (env.DEPLOY_COLOR == "blue") {
                                sh "kubectl patch service petclinic-service -p '{\"spec\": {\"selector\": {\"app\": \"petclinic\", \"color\": \"blue\"}}}'"
                            } else {
                                sh "kubectl patch service petclinic-service -p '{\"spec\": {\"selector\": {\"app\": \"petclinic\", \"color\": \"green\"}}}'"
                            }
                            echo "Traffic switching completed successfully!"
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
