pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-southeast-2'
        AWS_ACCOUNT_ID = '075810104493'
        ECR_REPO = 'petclinic'
        IMAGE_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        IMAGE_TAG = "${BUILD_NUMBER}"
        NAMESPACE = 'petclinic'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Unit Test') {
            steps {
                sh './mvnw test'
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    ./mvnw spring-boot:build-image \
                      -Dspring-boot.build-image.imageName=${IMAGE_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    mkdir -p "$WORKSPACE/trivy-tmp"

                    TMPDIR="$WORKSPACE/trivy-tmp" \
                    trivy image \
                      --scanners vuln \
                      --exit-code 1 \
                      --severity HIGH,CRITICAL \
                      "${IMAGE_URI}:${IMAGE_TAG}"
                '''
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    aws ecr get-login-password --region ${AWS_REGION} | \
                    docker login \
                      --username AWS \
                      --password-stdin \
                      ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                    docker push ${IMAGE_URI}:${IMAGE_TAG}
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig \
                      --region ${AWS_REGION} \
                      --name devops-project2

                    kubectl apply -f k8s/petclinic.yaml

                    kubectl -n ${NAMESPACE} set image deployment/petclinic \
                      petclinic=${IMAGE_URI}:${IMAGE_TAG}

                    kubectl -n ${NAMESPACE} rollout status \
                      deployment/petclinic \
                      --timeout=5m
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    kubectl get deployment petclinic -n ${NAMESPACE}
                    kubectl get pods -n ${NAMESPACE} -o wide
                    kubectl get svc petclinic -n ${NAMESPACE}
                '''
            }
        }
    }

    post {
        success {
            echo 'PetClinic CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'PetClinic CI/CD pipeline failed.'
        }
    }
}

