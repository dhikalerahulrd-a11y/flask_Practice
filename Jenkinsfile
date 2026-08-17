pipeline {

    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPOSITORY = 'flask-practice'
        ECR_REGISTRY = '916080963016.dkr.ecr.ap-south-1.amazonaws.com'
        IMAGE_NAME = "${ECR_REGISTRY}/${ECR_REPOSITORY}"
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
                    python3 -m pip install --upgrade pip
                    pip3 install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    pytest
                '''
            }
        }

        stage('Build') {
            steps {
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()

                    sh """
                        docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    """
                }
            }
        }

        stage('Push to ECR') {
            steps {
                withCredentials([
                    [$class: 'AmazonWebServicesCredentialsBinding',
                     credentialsId: 'aws-credentials']
                ]) {

                    sh """
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY}

                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    """
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-ssh-key']) {

                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@YOUR_EC2_IP '
                            aws ecr get-login-password --region ${AWS_REGION} |
                            docker login --username AWS --password-stdin ${ECR_REGISTRY}

                            docker pull ${IMAGE_NAME}:${IMAGE_TAG}

                            docker stop flask-app || true
                            docker rm flask-app || true

                            docker run -d \
                              --name flask-app \
                              --restart unless-stopped \
                              -p 5000:5000 \
                              ${IMAGE_NAME}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    sleep 10

                    curl --fail http://15.206.165.57:5000/health
                '''
            }
        }
    }

    post {

        success {
            emailext(
                subject: "SUCCESS: Flask CI/CD Pipeline - ${env.BUILD_NUMBER}",
                body: """
Pipeline completed successfully.

Commit SHA: ${env.GIT_COMMIT}
Image Tag: ${env.IMAGE_TAG}

Application deployed successfully to EC2.

Health check passed.
""",
                to: 'dhikalerahul.rd@gmail.com'
            )
        }

        failure {
            emailext(
                subject: "FAILED: Flask CI/CD Pipeline - ${env.BUILD_NUMBER}",
                body: """
Pipeline FAILED.

Build Number: ${env.BUILD_NUMBER}
Commit SHA: ${env.GIT_COMMIT}

Please check the Jenkins console output to identify the failed stage.
""",
                to: 'dhikalerahul.rd@gmail.com'
            )
        }
    }
}