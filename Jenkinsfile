pipeline {
    agent any

    environment {
        MERCURY_DOCKER_IMAGE = "vinit2620/node-api"
        MERCURY_IMAGE_TAG = "${BUILD_NUMBER}"

        MERCURY_AWS_REGION = "ap-south-1"
        MERCURY_EKS_CLUSTER = "production-eks"

        MERCURY_HELM_RELEASE = "node-api"
        MERCURY_NAMESPACE = "production"
    }

    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '10'))
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

        stage('Run Tests') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Application') {
            steps {
                sh 'npm run build'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    docker build \
                    -t ${MERCURY_DOCKER_IMAGE}:${MERCURY_IMAGE_TAG} .
                """
            }
        }

        stage('Security Scan') {
            steps {
                sh """
                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    ${MERCURY_DOCKER_IMAGE}:${MERCURY_IMAGE_TAG}
                """
            }
        }

        stage('Push Docker Image') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh """
                        echo \$DOCKER_PASS | docker login \
                        -u \$DOCKER_USER \
                        --password-stdin

                        docker push \
                        ${MERCURY_DOCKER_IMAGE}:${MERCURY_IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy To EKS') {
            steps {

                withCredentials([
                    file(
                        credentialsId: 'eks-kubeconfig',
                        variable: 'KUBECONFIG'
                    )
                ]) {

                    sh """
                        helm upgrade \
                        --install \
                        ${MERCURY_HELM_RELEASE} \
                        ./helm/node-api \
                        --namespace ${MERCURY_NAMESPACE} \
                        --create-namespace \
                        --set image.repository=${MERCURY_DOCKER_IMAGE} \
                        --set image.tag=${MERCURY_IMAGE_TAG}
                    """
                }
            }
        }

    }

    post {

        success {
            echo 'Deployment completed successfully.'
        }

        failure {
            echo 'Pipeline failed.'
        }

        always {
            cleanWs()
        }
    }
}