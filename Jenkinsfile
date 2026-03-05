pipeline {
    agent any

    environment {
        // AWS and ECR Configurations
        AWS_REGION     = "ap-south-1"
        // Replace with your actual AWS Account ID from the image
        ECR_REGISTRY   = "226608812300.dkr.ecr.ap-south-1.amazonaws.com" 
        ECR_REPO       = "jenkins-repo"
        IMAGE_TAG      = "${BUILD_NUMBER}"
        
        // Deployment Target
        APP_EC2_HOST   = "13.233.196.148  " 
        CONTAINER_NAME = "jenkins-repo"
        APP_PORT       = "80"
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Building the image locally on Jenkins agent
                    sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
                    sh "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REPO}:latest"
                }
            }
        }

        stage('Login to ECR') {
            steps {
                // Authenticating Jenkins with Amazon ECR
                sh """
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin ${ECR_REGISTRY}
                """
            }
        }

        stage('Tag & Push to ECR') {
            steps {
                sh """
                docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                docker tag ${ECR_REPO}:latest ${ECR_REGISTRY}/${ECR_REPO}:latest
                
                docker push ${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}
                docker push ${ECR_REGISTRY}/${ECR_REPO}:latest
                """
            }
        }

        stage('Deploy to App EC2') {
            steps {
                // Ensure 'app-ec2-ssh' is configured in Jenkins Credentials
                sshagent(credentials: ['app-ec2-ssh']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@${APP_EC2_HOST} "
                        # Login to ECR on the remote EC2 instance
                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin ${ECR_REGISTRY} && \
                        
                        # Pull the latest image
                        docker pull ${ECR_REGISTRY}/${ECR_REPO}:latest && \
                        
                        # Stop and remove existing container if it exists
                        docker rm -f ${CONTAINER_NAME} || true && \
                        
                        # Run the new container
                        docker run -d --name ${CONTAINER_NAME} -p ${APP_PORT}:80 ${ECR_REGISTRY}/${ECR_REPO}:latest
                    "
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Deployment completed successfully!"
        }
        failure {
            echo "Pipeline failed. Please check the console logs."
        }
    }
}

