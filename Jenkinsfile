
pipeline {
    agent any

    environment {
        IMAGE_NAME = 'soham045/campus-equipment-hub'
        K8S_DEPLOYMENT = 'campus-equipment-hub'
        K8S_CONTAINER = 'campus-equipment-hub'
        KUBECONFIG = '/var/lib/jenkins/.kube/config'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                      -t ${IMAGE_NAME}:latest .
                '''
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DH_USER',
                        passwordVariable: 'DH_PASSWORD'
                    )
                ]) {
                    sh '''
                        set -e

                        echo "$DH_PASSWORD" | docker login \
                          -u "$DH_USER" --password-stdin

                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker push ${IMAGE_NAME}:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Minikube') {
            steps {
                sh '''
                    set -e

                    kubectl get nodes

                    kubectl apply -f k8s/

                    kubectl set image \
                      deployment/${K8S_DEPLOYMENT} \
                      ${K8S_CONTAINER}=${IMAGE_NAME}:${BUILD_NUMBER}

                    kubectl rollout status \
                      deployment/${K8S_DEPLOYMENT} \
                      --timeout=180s
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    set -e

                    echo "Checking Kubernetes pods..."
                    kubectl get pods

                    echo "Checking deployments..."
                    kubectl get deployments

                    echo "Checking services..."
                    kubectl get services

                    echo "Checking application health..."
                    kubectl run campus-hub-health-check \
                      --image=curlimages/curl:latest \
                      --restart=Never \
                      --rm -i \
                      --command -- \
                      curl --fail --silent --show-error \
                      http://campus-equipment-hub-service:3000/health
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully!'
            echo "Docker image: ${IMAGE_NAME}:${BUILD_NUMBER}"
        }

        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}
