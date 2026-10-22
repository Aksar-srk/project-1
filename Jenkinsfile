pipeline {
    agent any

    tools {
        nodejs 'NodeJS-26'
    }

    environment {
        AWS_REGION  = 'ap-south-2'
        ECR_REGISTRY = '060119699323.dkr.ecr.ap-south-2.amazonaws.com'
        IMAGE_REPO  = "${ECR_REGISTRY}/aksarsr/ecr"
        EKS_CLUSTER = 'jenkins-eks'
        K8S_NAMESPACE = 'aksar'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Trivy Filesystem Scan') {
            steps {
                sh 'trivy fs --exit-code 1 --severity HIGH,CRITICAL .'
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

        stage('Build Application') {
            steps {
                sh 'npm run build'
            }
        }

        stage('ECR Login') {
            steps {
                sh '''
                    aws ecr get-login-password --region "$AWS_REGION" |
                    docker login --username AWS --password-stdin "$ECR_REGISTRY"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build --platform linux/amd64 \
                      -t "$IMAGE_REPO:$BUILD_NUMBER" \
                      -t "$IMAGE_REPO:latest" .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image --ignore-unfixed \
                      --exit-code 1 --severity CRITICAL \
                      "$IMAGE_REPO:$BUILD_NUMBER"
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    docker push "$IMAGE_REPO:$BUILD_NUMBER"
                    docker push "$IMAGE_REPO:latest"
                '''
            }
        }

        stage('Configure EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region "$AWS_REGION" \
                      --name "$EKS_CLUSTER"

                    kubectl create namespace "$K8S_NAMESPACE" \
                      --dry-run=client -o yaml | kubectl apply -f -
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    sed -i '/image: /s|image: .*|image: '"$IMAGE_REPO:$BUILD_NUMBER"'|' kubernetes/deploy-svc.yaml  

                    kubectl apply -n "$K8S_NAMESPACE" \
                      -f kubernetes/deploy-svc.yaml

                    kubectl apply -n "$K8S_NAMESPACE" \
                      -f kubernetes/ingress.yaml

                    if ! kubectl rollout status \
                      deployment/tic-tac-toe \
                      -n "$K8S_NAMESPACE" \
                      --timeout=180s; then

                        kubectl rollout undo deployment/tic-tac-toe \
                          -n "$K8S_NAMESPACE" || true
                        exit 1
                    fi
                '''
            }
        }
    }

    post {
        success {
            echo 'Build and deployment succeeded!'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs.'
        }
        always {
            echo "Build ${env.BUILD_NUMBER} finished."
        }
    }
}