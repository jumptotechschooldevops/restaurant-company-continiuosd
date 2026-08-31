pipeline {
    agent any

    environment {
        // Homebrew tools on your Mac
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin"

        // AWS / EKS
        AWS_REGION = "us-east-2"
        EKS_CLUSTER_NAME = "restaurant-company"

        // ECR
        AWS_ACCOUNT_ID = "288673275952"
        ECR_REPOSITORY = "restaurant-company"

        // Image produced by CI
        IMAGE = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}:latest"
    }

    stages {

        // =====================================================
        // 1. VERIFY TOOLS
        // =====================================================
        stage('1. Verify Tools') {
            steps {
                sh '''
                    echo "===== AWS CLI ====="
                    which aws
                    aws --version

                    echo "===== KUBECTL ====="
                    which kubectl
                    kubectl version --client
                '''
            }
        }

        // =====================================================
        // 2. VERIFY AWS CREDENTIALS
        // =====================================================
        stage('2. Verify AWS') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "===== AWS IDENTITY ====="

                        aws sts get-caller-identity \
                          --region ${AWS_REGION}
                    '''
                }
            }
        }

        // =====================================================
        // 3. CONNECT TO EKS
        // =====================================================
        stage('3. Connect to EKS') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "===== CONNECTING TO EKS ====="

                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER_NAME}

                        echo "===== CURRENT CONTEXT ====="
                        kubectl config current-context

                        echo "===== EKS NODES ====="
                        kubectl get nodes
                    '''
                }
            }
        }

        // =====================================================
        // 4. DEPLOY MANIFESTS
        // =====================================================
        stage('4. Deploy to EKS') {
            steps {
                sh '''
                    echo "===== APPLY DEPLOYMENT ====="
                    kubectl apply -f deployment.yaml

                    echo "===== APPLY SERVICE ====="
                    kubectl apply -f service.yaml
                '''
            }
        }

        // =====================================================
        // 5. UPDATE IMAGE
        // =====================================================
        stage('5. Update Image') {
            steps {
                sh '''
                    echo "===== IMAGE ====="
                    echo "${IMAGE}"

                    kubectl set image \
                      deployment/restaurant-company \
                      restaurant-company=${IMAGE}
                '''
            }
        }

        // =====================================================
        // 6. WAIT FOR DEPLOYMENT
        // =====================================================
        stage('6. Wait for Rollout') {
            steps {
                sh '''
                    echo "===== WAITING FOR ROLLOUT ====="

                    kubectl rollout status \
                      deployment/restaurant-company \
                      --timeout=300s
                '''
            }
        }

        // =====================================================
        // 7. VERIFY
        // =====================================================
        stage('7. Verify Deployment') {
            steps {
                sh '''
                    echo "===== PODS ====="
                    kubectl get pods -o wide

                    echo "===== DEPLOYMENT ====="
                    kubectl get deployment restaurant-company

                    echo "===== SERVICE ====="
                    kubectl get service restaurant-company

                    echo "===== LOAD BALANCER ====="
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
RESTAURANT COMPANY CD PASSED
==========================================
AWS Authentication: PASSED
EKS Connection:      PASSED
Deployment:          PASSED
Service:             PASSED
==========================================
'''
        }

        failure {
            echo '''
==========================================
RESTAURANT COMPANY CD FAILED
==========================================
Check the failed stage above.
==========================================
'''
        }
    }
}
