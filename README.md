# Flask CI/CD Pipeline

## Project
Student Registration System using Flask and MongoDB.

## Technologies
- Python
- Flask
- MongoDB
- Docker
- Jenkins
- AWS ECR
- AWS EC2

## Pipeline
1. Checkout
2. Install
3. Test
4. Build Docker image
5. Push to ECR
6. Deploy to EC2
7. Verify /health
8. Send email

## AWS Setup
- ECR repository
- EC2 instance
- IAM role
- Security group

## Jenkins Secrets
- AWS credentials
- EC2 SSH key
- MongoDB credentials
- SMTP credentials

## Deployment
Jenkins builds a Docker image using the Git commit SHA,
pushes it to ECR and deploys it to EC2.

## Health Check
The /health endpoint checks MongoDB connectivity.

## Security
No real secrets, AWS keys, MongoDB URI or PEM files
are committed to GitHub.