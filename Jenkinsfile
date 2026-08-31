pipeline {
    agent any

    environment {
        // Jenkins on macOS may not automatically see Homebrew binaries
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin"

        // AWS / EKS configuration
        AWS_REGION = "us-east-2"
        EKS_CLUSTER_NAME = "restaurant-company"

        // ECR configuration
        AWS_ACCOUNT_ID = "288673275952"
        ECR_REPOSITORY = "restaurant-company"

        // Docker image deployed to Kubernetes
        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest"
    }

    stages {

        // =====================================================
        // 1. VERIFY REQUIRED TOOLS
        // =====================================================
        stage('1. Verify Tools') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "VERIFYING JENKINS TOOLS"
                    echo "=========================================="

                    echo "AWS CLI:"
                    which aws
                    aws --version

                    echo ""

                    echo "kubectl:"
                    which kubectl
                    kubectl version --client

                    echo "=========================================="
                '''
            }
        }

        // =====================================================
        // 2. VERIFY AWS CONNECTION
        // =====================================================
        stage('2. Verify AWS') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "VERIFYING AWS CONNECTION"
                    echo "=========================================="

                    aws sts get-caller-identity

                    echo "=========================================="
                '''
            }
        }

        // =====================================================
        // 3. CONNECT JENKINS TO EKS
        // =====================================================
        stage('3. Connect to EKS') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "CONNECTING TO EKS"
                    echo "=========================================="

                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name ${EKS_CLUSTER_NAME}

                    echo ""
                    echo "Current Kubernetes context:"
                    kubectl config current-context

                    echo ""
                    echo "EKS Nodes:"
                    kubectl get nodes

                    echo "=========================================="
                '''
            }
        }

        // =====================================================
        // 4. DEPLOY KUBERNETES MANIFESTS
        // =====================================================
        stage('4. Deploy to EKS') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "DEPLOYING KUBERNETES RESOURCES"
                    echo "=========================================="

                    echo "Applying deployment.yaml..."
                    kubectl apply -f deployment.yaml

                    echo ""
                    echo "Applying service.yaml..."
                    kubectl apply -f service.yaml

                    echo "=========================================="
                '''
            }
        }

        // =====================================================
        // 5. UPDATE APPLICATION IMAGE
        // =====================================================
        stage('5. Update Image') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "UPDATING APPLICATION IMAGE"
                    echo "=========================================="

                    echo "Image:"
                    echo "${IMAGE}"

                    kubectl set image \
                      deployment/restaurant-company \
                      restaurant-company=${IMAGE}

                    echo "=========================================="
                '''
            }
        }

        // =====================================================
        // 6. WAIT FOR KUBERNETES ROLLOUT
        // =====================================================
        stage('6. Wait for Rollout') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "WAITING FOR DEPLOYMENT"
                    echo "=========================================="

                    kubectl rollout status \
                      deployment/restaurant-company \
                      --timeout=300s

                    echo "=========================================="
                '''
            }
        }

        // =====================================================
        // 7. VERIFY DEPLOYMENT
        // =====================================================
        stage('7. Verify Deployment') {
            steps {
                sh '''
                    echo "=========================================="
                    echo "PODS"
                    echo "=========================================="

                    kubectl get pods -o wide

                    echo ""
                    echo "=========================================="
                    echo "DEPLOYMENT"
                    echo "=========================================="

                    kubectl get deployment restaurant-company

                    echo ""
                    echo "=========================================="
                    echo "SERVICE"
                    echo "=========================================="

                    kubectl get service restaurant-company

                    echo ""
                    echo "=========================================="
                    echo "LOAD BALANCER"
                    echo "=========================================="

                    kubectl get service restaurant-company \
                      -o jsonpath='{.status.loadBalancer.ingress[0].hostname}' || true

                    echo ""
                '''
            }
        }
    }

    post {

        success {
            echo '''
==========================================
RESTAURANT COMPANY CD PIPELINE PASSED
==========================================
1. Jenkins tools verified
2. AWS connection verified
3. Jenkins connected to EKS
4. Kubernetes manifests applied
5. ECR image deployed
6. Kubernetes rollout completed
7. Deployment verified
==========================================
'''
        }

        failure {
            echo '''
==========================================
RESTAURANT COMPANY DEPLOYMENT FAILED
==========================================
'''

            sh '''
                if command -v kubectl >/dev/null 2>&1; then

                    echo "========== PODS =========="
                    kubectl get pods -o wide || true

                    echo ""
                    echo "========== DEPLOYMENT =========="
                    kubectl get deployment restaurant-company || true

                    echo ""
                    echo "========== SERVICE =========="
                    kubectl get service restaurant-company || true

                    echo ""
                    echo "========== RECENT EVENTS =========="
                    kubectl get events \
                      --sort-by=.lastTimestamp 2>/dev/null | tail -30 || true

                else
                    echo "kubectl is not available to Jenkins."
                fi
            '''
        }
    }
}
