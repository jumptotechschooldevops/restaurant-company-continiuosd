pipeline {
    agent any

    environment {
        PATH = "/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin"

        AWS_REGION = "us-east-2"
        EKS_CLUSTER_NAME = "restaurant-company"

        AWS_ACCOUNT_ID = "288673275952"
        ECR_REPOSITORY = "restaurant-company"

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
        // 2. AWS + EKS + DEPLOYMENT
        // Keep AWS credentials available for ALL kubectl commands
        // =====================================================
        stage('2. Deploy to AWS EKS') {

            steps {

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    sh '''
                        set -e

                        echo "=========================================="
                        echo "AWS IDENTITY"
                        echo "=========================================="

                        aws sts get-caller-identity


                        echo ""
                        echo "=========================================="
                        echo "CONNECTING TO EKS"
                        echo "=========================================="

                        aws eks update-kubeconfig \
                          --region ${AWS_REGION} \
                          --name ${EKS_CLUSTER_NAME}


                        echo ""
                        echo "=========================================="
                        echo "CURRENT KUBERNETES CONTEXT"
                        echo "=========================================="

                        kubectl config current-context


                        echo ""
                        echo "=========================================="
                        echo "EKS NODES"
                        echo "=========================================="

                        kubectl get nodes


                        echo ""
                        echo "=========================================="
                        echo "APPLYING DEPLOYMENT"
                        echo "=========================================="

                        kubectl apply -f deployment.yaml


                        echo ""
                        echo "=========================================="
                        echo "APPLYING SERVICE"
                        echo "=========================================="

                        kubectl apply -f service.yaml


                        echo ""
                        echo "=========================================="
                        echo "UPDATING APPLICATION IMAGE"
                        echo "=========================================="

                        echo "Deploying:"
                        echo "${IMAGE}"

                        kubectl set image \
                          deployment/restaurant-company \
                          restaurant-company=${IMAGE}


                        echo ""
                        echo "=========================================="
                        echo "WAITING FOR ROLLOUT"
                        echo "=========================================="

                        kubectl rollout status \
                          deployment/restaurant-company \
                          --timeout=300s


                        echo ""
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

Check the failed command above.

==========================================
'''
        }
    }
}
