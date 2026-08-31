pipeline {
    agent any

    environment {
        AWS_REGION     = 'us-east-2'
        EKS_CLUSTER    = 'restaurant-company'
        ECR_REPOSITORY = 'restaurant-company'

        // CHANGE THIS
        AWS_ACCOUNT_ID = 'YOUR_AWS_ACCOUNT_ID'

        // CHANGE THIS to the tag already in ECR
        IMAGE_TAG      = 'YOUR_IMAGE_TAG'

        IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:${IMAGE_TAG}"
    }

    stages {

        // =====================================================
        // 1. CHECKOUT
        // We only need deployment.yaml and service.yaml
        // =====================================================
        stage('Checkout') {
            steps {
                checkout scm
            }
        }


        // =====================================================
        // 2. CONNECT TO EKS
        // =====================================================
        stage('Connect to EKS') {
            steps {
                sh '''
                    echo "Connecting Jenkins to EKS..."

                    aws eks update-kubeconfig \
                      --region "$AWS_REGION" \
                      --name "$EKS_CLUSTER"

                    echo "Checking nodes..."
                    kubectl get nodes
                '''
            }
        }


        // =====================================================
        // 3. DEPLOY KUBERNETES FILES
        // =====================================================
        stage('Deploy to EKS') {
            steps {
                sh '''
                    echo "Applying Kubernetes manifests..."

                    kubectl apply -f k8s/deployment.yaml
                    kubectl apply -f k8s/service.yaml
                '''
            }
        }


        // =====================================================
        // 4. DEPLOY IMAGE ALREADY IN ECR
        // =====================================================
        stage('Update Image') {
            steps {
                sh '''
                    echo "Deploying existing ECR image:"
                    echo "$IMAGE_URI"

                    kubectl set image \
                      deployment/restaurant-company \
                      restaurant-company="$IMAGE_URI"
                '''
            }
        }


        // =====================================================
        // 5. VERIFY DEPLOYMENT
        // =====================================================
        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "Waiting for rollout..."

                    kubectl rollout status \
                      deployment/restaurant-company \
                      --timeout=180s

                    echo "===== DEPLOYMENT ====="
                    kubectl get deployment restaurant-company

                    echo "===== PODS ====="
                    kubectl get pods \
                      -l app=restaurant-company \
                      -o wide

                    echo "===== SERVICE ====="
                    kubectl get svc restaurant-company
                '''
            }
        }
    }

    post {
        success {
            echo "======================================"
            echo "DEPLOYMENT SUCCESSFUL"
            echo "======================================"
            echo "Image: ${IMAGE_URI}"
            echo "Cluster: ${EKS_CLUSTER}"
        }

        failure {
            echo "DEPLOYMENT FAILED"

            sh '''
                kubectl get pods -o wide || true
                kubectl get events --sort-by=.lastTimestamp | tail -30 || true
            '''
        }
    }
}
