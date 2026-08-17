pipeline {

    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '916080963016'

        ECR_REPO = 'flask-practice'
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

        IMAGE_TAG = "${GIT_COMMIT}"

        EC2_HOST = '15.206.165.57'
        CONTAINER_NAME = 'flask-practice'
        APP_PORT = '5000'
    }

    stages {

        stage('Checkout') {
            steps {
                script {
                    env.FAILED_STAGE = 'Checkout'
                }

                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                script {
                    env.FAILED_STAGE = 'Install Dependencies'
                }

                sh '''
                    set -e

                    echo "======================================"
                    echo "Installing Python Dependencies"
                    echo "======================================"

                    python3 -m pip install \
                        --break-system-packages \
                        -r requirements.txt

                    echo "Dependencies installed successfully."
                '''
            }
        }

        stage('Run Tests') {
            steps {
                script {
                    env.FAILED_STAGE = 'Run Tests'
                }

                withCredentials([
                    string(
                        credentialsId: 'MONGO_URI_RI',
                        variable: 'MONGO_URI'
                    )
                ]) {

                    sh '''
                        set -e

                        echo "======================================"
                        echo "Checking MongoDB Credential"
                        echo "======================================"

                        if [ -z "$MONGO_URI" ]; then
                            echo "ERROR: MongoDB credential is missing."
                            exit 1
                        fi

                        echo "MongoDB credential is available."

                        echo "======================================"
                        echo "Running Pytest"
                        echo "======================================"

                        python3 -m pytest -v

                        echo "======================================"
                        echo "All tests passed successfully."
                        echo "======================================"
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    env.FAILED_STAGE = 'Build Docker Image'
                }

                sh '''
                    set -e

                    echo "======================================"
                    echo "Building Docker Image"
                    echo "======================================"

                    echo "Image Tag: $IMAGE_TAG"

                    docker build \
                        -t "$ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG" \
                        .

                    echo "Docker image built successfully."

                    echo "======================================"
                    echo "Docker Image Details"
                    echo "======================================"

                    docker images | grep flask-practice || true
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                script {
                    env.FAILED_STAGE = 'Push Image to ECR'
                }

                withCredentials([
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'rd-aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    sh '''
                        set -e

                        echo "======================================"
                        echo "Checking AWS Credentials"
                        echo "======================================"

                        aws sts get-caller-identity

                        echo "======================================"
                        echo "Logging in to Amazon ECR"
                        echo "======================================"

                        aws ecr get-login-password \
                            --region "$AWS_REGION" |
                        docker login \
                            --username AWS \
                            --password-stdin "$ECR_REGISTRY"

                        echo "ECR login successful."

                        echo "======================================"
                        echo "Pushing Docker Image"
                        echo "======================================"

                        docker push \
                            "$ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG"

                        echo "======================================"
                        echo "Docker Image Pushed Successfully"
                        echo "======================================"

                        echo "Image:"
                        echo "$ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG"
                    '''
                }
            }
        }

        stage('Test EC2 SSH') {
            steps {
                script {
                    env.FAILED_STAGE = 'Test EC2 SSH'
                }

                sshagent(['rd-ec2-ssh-key']) {

                    sh '''
                        set -e

                        echo "======================================"
                        echo "Testing SSH Connection to EC2"
                        echo "======================================"

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ec2-user@$EC2_HOST \
                            "echo 'SSH connection successful' && hostname"

                        echo "SSH connection test completed successfully."
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    env.FAILED_STAGE = 'Deploy to EC2'
                }

                withCredentials([
                    string(
                        credentialsId: 'MONGO_URI_RI',
                        variable: 'MONGO_URI'
                    ),
                    string(
                        credentialsId: 'flask-secret-key',
                        variable: 'SECRET_KEY'
                    ),
                    [
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: 'rd-aws-credentials',
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]
                ]) {

                    sshagent(['rd-ec2-ssh-key']) {

                        sh '''
                            set -e

                            echo "======================================"
                            echo "Starting EC2 Deployment"
                            echo "======================================"

                            ssh \
                                -o StrictHostKeyChecking=no \
                                ec2-user@$EC2_HOST \
                                bash -s \
                                -- \
                                "$AWS_REGION" \
                                "$ECR_REGISTRY" \
                                "$ECR_REPO" \
                                "$IMAGE_TAG" \
                                "$MONGO_URI" \
                                "$SECRET_KEY" <<'REMOTE_SCRIPT'

set -e

AWS_REGION="$1"
ECR_REGISTRY="$2"
ECR_REPO="$3"
IMAGE_TAG="$4"
MONGO_URI="$5"
SECRET_KEY="$6"

echo "======================================"
echo "Connected to EC2"
echo "======================================"

echo "======================================"
echo "Logging in to Amazon ECR"
echo "======================================"

aws ecr get-login-password \
    --region "$AWS_REGION" |
docker login \
    --username AWS \
    --password-stdin "$ECR_REGISTRY"

echo "ECR login successful."

echo "======================================"
echo "Pulling New Docker Image"
echo "======================================"

docker pull \
    "$ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG"

echo "Docker image pulled successfully."

echo "======================================"
echo "Stopping Existing Container"
echo "======================================"

docker stop flask-practice || true

echo "======================================"
echo "Removing Existing Container"
echo "======================================"

docker rm flask-practice || true

echo "======================================"
echo "Starting New Container"
echo "======================================"

docker run -d \
    --name flask-practice \
    -p 5000:5000 \
    -e MONGO_URI="$MONGO_URI" \
    -e SECRET_KEY="$SECRET_KEY" \
    "$ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG"

echo "======================================"
echo "Container Started"
echo "======================================"

docker ps \
    --filter name=flask-practice

echo "======================================"
echo "Deployment Completed"
echo "======================================"

REMOTE_SCRIPT

                            echo "======================================"
                            echo "EC2 Deployment Finished"
                            echo "======================================"
                        '''
                    }
                }
            }
        }

        stage('Verify Application') {
            steps {
                script {
                    env.FAILED_STAGE = 'Verify Application'
                }

                sshagent(['rd-ec2-ssh-key']) {

                    sh '''
                        set -e

                        echo "======================================"
                        echo "Verifying Docker Container"
                        echo "======================================"

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ec2-user@$EC2_HOST \
                            "docker ps --filter name=$CONTAINER_NAME"

                        echo "======================================"
                        echo "Checking Flask Health Endpoint"
                        echo "======================================"

                        ssh \
                            -o StrictHostKeyChecking=no \
                            ec2-user@$EC2_HOST \
                            "curl -f http://localhost:$APP_PORT/health"

                        echo ""

                        echo "======================================"
                        echo "Application Verification Successful"
                        echo "======================================"
                    '''
                }
            }
        }
    }

    post {

        success {

            echo '''
==========================================
CI/CD PIPELINE COMPLETED SUCCESSFULLY!
==========================================
Application built, pushed to ECR,
deployed to EC2 and verified using /health.
==========================================
'''

            emailext(
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                to: "dhikalerahul.rd@gmail.com",
                mimeType: 'text/plain',
                body: """
Hello,

The Jenkins CI/CD pipeline completed successfully.

==============================
BUILD DETAILS
==============================
Job Name     : ${env.JOB_NAME}
Build Number : ${env.BUILD_NUMBER}
Build Status : SUCCESS
Branch       : ${env.GIT_BRANCH}
Commit SHA   : ${env.GIT_COMMIT}

==============================
DOCKER / ECR
==============================
ECR Repository:
${env.ECR_REGISTRY}/${env.ECR_REPO}

Docker Image:
${env.ECR_REGISTRY}/${env.ECR_REPO}:${env.IMAGE_TAG}

==============================
EC2 DEPLOYMENT
==============================
EC2 Host     : ${env.EC2_HOST}
Container    : ${env.CONTAINER_NAME}
Port         : ${env.APP_PORT}
Health Check : PASSED
Endpoint     : /health

==============================
DEPLOYMENT RESULT
==============================
Docker image built successfully.
Docker image pushed to Amazon ECR.
Docker container deployed to EC2.
Application health check passed.

==============================
PIPELINE RUN
==============================
Build URL:
${env.BUILD_URL}

Regards,
Jenkins
"""
            )
        }

        failure {

            echo '''
==========================================
CI/CD PIPELINE FAILED!
==========================================
Check the failed stage and console log.
==========================================
'''

            emailext(
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                to: "dhikalerahul.rd@gmail.com",
                mimeType: 'text/plain',
                body: """
Hello,

The Jenkins CI/CD pipeline has FAILED.

==============================
BUILD DETAILS
==============================
Job Name     : ${env.JOB_NAME}
Build Number : ${env.BUILD_NUMBER}
Build Status : FAILURE
Branch       : ${env.GIT_BRANCH}
Commit SHA   : ${env.GIT_COMMIT}

==============================
FAILURE INFORMATION
==============================
Stage in Progress When Failure Occurred:
${env.FAILED_STAGE ?: 'Unknown'}

The pipeline stopped because this stage failed.

==============================
DOCKER / ECR
==============================
ECR Repository:
${env.ECR_REGISTRY}/${env.ECR_REPO}

Docker Image:
${env.ECR_REGISTRY}/${env.ECR_REPO}:${env.IMAGE_TAG}

==============================
PIPELINE LOG
==============================
Build URL:
${env.BUILD_URL}

Console Log:
${env.BUILD_URL}console

Please investigate the failed stage using the Jenkins
console output.

Regards,
Jenkins
"""
            )
        }
    }
}