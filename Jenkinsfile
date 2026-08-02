pipeline {

    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        AWS_REGION = 'ap-south-1'
        ACCOUNT_ID = '463605761106'
        REPOSITORY = 'frontend'

        IMAGE_TAG = "${BUILD_NUMBER}"

        ECR_REGISTRY = "${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        ECR_URI = "${ECR_REGISTRY}/${REPOSITORY}"

        EKS_CLUSTER = 'demo-cluster-1'

        DEPLOYMENT_NAME = 'frontend'
        CONTAINER_NAME = 'frontend'

        K8S_NAMESPACE = 'development'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }


        stage('Verify Tools') {
            steps {
                sh '''
                    echo "Checking required tools..."

                    aws --version
                    docker --version
                    kubectl version --client

                    echo "Tools verified."
                '''
            }
        }


        stage('Verify AWS Identity') {
            steps {
                sh '''
                    aws sts get-caller-identity
                '''
            }
        }


        stage('Login to Amazon ECR') {
            steps {
                sh '''
                    aws ecr get-login-password \
                    --region ${AWS_REGION} | \
                    docker login \
                    --username AWS \
                    --password-stdin ${ECR_REGISTRY}
                '''
            }
        }


        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build \
                    -t ${REPOSITORY}:${IMAGE_TAG} .
                '''
            }
        }


        stage('Tag Docker Image') {
            steps {
                sh '''
                    docker tag \
                    ${REPOSITORY}:${IMAGE_TAG} \
                    ${ECR_URI}:${IMAGE_TAG}

                    docker tag \
                    ${REPOSITORY}:${IMAGE_TAG} \
                    ${ECR_URI}:latest
                '''
            }
        }


        stage('Push Docker Image') {
            steps {
                sh '''
                    docker push ${ECR_URI}:${IMAGE_TAG}

                    docker push ${ECR_URI}:latest
                '''
            }
        }


        stage('Configure kubectl') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                    --region ${AWS_REGION} \
                    --name ${EKS_CLUSTER}

                    kubectl config current-context
                '''
            }
        }


        stage('Verify EKS Connection') {
            steps {
                sh '''
                    kubectl get nodes
                '''
            }
        }


        stage('Deploy to EKS') {
            steps {
                sh '''
                    kubectl set image \
                    deployment/${DEPLOYMENT_NAME} \
                    ${CONTAINER_NAME}=${ECR_URI}:${IMAGE_TAG} \
                    -n ${K8S_NAMESPACE}
                '''
            }
        }


        stage('Verify Rollout') {
            steps {
                sh '''
                    kubectl rollout status \
                    deployment/${DEPLOYMENT_NAME} \
                    -n ${K8S_NAMESPACE} \
                    --timeout=300s
                '''
            }
        }


        stage('Verify Deployment') {
            steps {
                sh '''
                    echo "===== PODS ====="
                    kubectl get pods \
                    -n ${K8S_NAMESPACE} \
                    -o wide

                    echo "===== SERVICES ====="
                    kubectl get svc \
                    -n ${K8S_NAMESPACE}

                    echo "===== INGRESS ====="
                    kubectl get ingress \
                    -n ${K8S_NAMESPACE}

                    echo "===== DEPLOYMENTS ====="
                    kubectl get deployment \
                    -n ${K8S_NAMESPACE}

                    echo "===== DEPLOYED IMAGE ====="
                    kubectl get deployment ${DEPLOYMENT_NAME} \
                    -n ${K8S_NAMESPACE} \
                    -o jsonpath='{.spec.template.spec.containers[0].image}'

                    echo ""
                '''
            }
        }

    }


    post {

        success {
            echo 'Deployment Successful'
        }


        failure {
            echo 'Pipeline failed - attempting rollback'

            sh '''
                if command -v kubectl >/dev/null 2>&1; then
                    kubectl rollout undo \
                    deployment/${DEPLOYMENT_NAME} \
                    -n ${K8S_NAMESPACE} || true
                fi
            '''
        }


        always {
            sh '''
                docker image prune -af || true
            '''

            cleanWs()
        }
    }
}