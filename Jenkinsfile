pipeline {

    agent any

    options {
        skipDefaultCheckout(true)
    }

    environment {
        AWS_REGION     = 'ap-south-1'
        ECR_REPOSITORY = 'flask-practice'
        ECR_REGISTRY   = '916080963016.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_NAME     = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install') {
            steps {
                sh '''
                    set -e

                    rm -rf venv

                    python3 -m venv venv

 (Fix Python virtual environment in Jenkins pipeline)

                    ./venv/bin/pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    ./venv/bin/pytest
                '''
            }
        }

        stage('Build') {
            steps {
                script {

                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse HEAD',
                        returnStdout: true
                    ).trim()

                    echo "Image tag: ${env.IMAGE_TAG}"

                    sh """
                        set -e

                        docker build \
                            -t ${env.IMAGE_NAME}:${env.IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Push to ECR') {
            steps {

                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'rd-aws-credentials']
                ]) {

                    sh """
                        set -e

                        aws ecr get-login-password \
                            --region ${env.AWS_REGION} | \
                        docker login \
                            --username AWS \
                            --password-stdin ${env.ECR_REGISTRY}

                        docker push \
                            ${env.IMAGE_NAME}:${env.IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {

                sshagent(['rd-ec2-ssh-key']) {

                    sh """
                        set -e

                        ssh -o StrictHostKeyChecking=no \
                            ec2-user@15.206.165.57 '
                                set -e

                                aws ecr get-login-password \
                                    --region ${env.AWS_REGION} | \
                                docker login \
                                    --username AWS \
                                    --password-stdin ${env.ECR_REGISTRY}

                                docker pull \
                                    ${env.IMAGE_NAME}:${env.IMAGE_TAG}

                                docker stop flask-app || true

                                docker rm flask-app || true

                                docker run -d \
                                    --name flask-app \
                                    --restart unless-stopped \
                                    -p 5000:5000 \
                                    ${env.IMAGE_NAME}:${env.IMAGE_TAG}
                            '
                    """
                }
            }
        }

        stage('Verify') {
            steps {

                sh '''
                    set -e

                    echo "Waiting for Flask application..."
                    sleep 10

                    curl --fail \
                        http://15.206.165.57:5000/health
                '''
            }
        }
    }

    post {

        success {

            emailext(
                subject: "SUCCESS: Flask CI/CD Pipeline - Build #${BUILD_NUMBER}",

                body: """
CI/CD Pipeline completed successfully.

Build Number: ${BUILD_NUMBER}

Commit SHA:
${GIT_COMMIT}

Docker Image:
${IMAGE_NAME}:${IMAGE_TAG}

Deployment:
SUCCESS

Health Check:
PASSED

The Flask application is running successfully on EC2.
""",

                to: 'dhikalerahul.rd@gmail.com'
            )
        }

        failure {

            emailext(
                subject: "FAILED: Flask CI/CD Pipeline - Build #${BUILD_NUMBER}",

                body: """
CI/CD Pipeline FAILED.

Build Number: ${BUILD_NUMBER}

Commit SHA:
${GIT_COMMIT}

Docker Image:
${IMAGE_NAME}:${IMAGE_TAG}

Deployment:
FAILED

Please check the Jenkins console output
to identify the failed stage.
""",

                to: 'dhikalerahul.rd@gmail.com'
            )
        }
    }
}