pipeline {
    // Define where the pipeline should run. In this case, on an EC2 agent labeled 'ec2-agent'.
    agent { label 'ec2-agent' }

    // Define environment variables to be used across the pipeline.
    environment {
        // Full ECR repository URL where the Docker image will be pushed.
        ECR_REPO = 'your-ecr-repo-url' 
        // Name of the Docker image.
        IMAGE_NAME = 'app-image' 
        // Tag for the Docker image, using the branch name and the build ID to make it unique.
        TAG = "${env.BRANCH_NAME}-${env.BUILD_ID}" 
        // SSH key credentials stored in Jenkins to access EC2 instances.
        SSH_KEY = credentials('ec2-ssh-key') 
        // Full image name with the ECR repository and tag.
        DOCKER_IMAGE = "${ECR_REPO}:${TAG}" 
    }

    // The stages of the pipeline where different tasks are executed.
    stages {
        // Stage to checkout the code from the Git repository.
        stage('Checkout') {
            steps {
                // Checkout the specified branch from the GitHub repository.
                git branch: "${env.BRANCH_NAME}", url: 'https://github.com/your-org/your-repo.git'
            }
        }

        // Stage to build the Docker image from the source code.
        stage('Build Docker Image') {
            steps {
                script {
                    // Build the Docker image using the Dockerfile in the repository.
                    docker.build("${env.ECR_REPO}:${env.TAG}")
                }
            }
        }

        // Stage to scan the Docker image for security vulnerabilities.
        stage('Scan Docker Image') {
            steps {
                script {
                    // Run the Trivy tool to scan the image. If vulnerabilities are found, exit with a code of 1.
                    sh "trivy image --exit-code 1 ${DOCKER_IMAGE} || echo 'Scan failed'"
                }
            }
        }

        // Stage to login to AWS ECR using AWS CLI and Docker.
        stage('Login to ECR') {
            steps {
                script {
                    // Authenticate Docker to the AWS ECR registry by retrieving the login password and piping it to Docker login.
                    sh "aws ecr get-login-password --region ${env.AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REPO}"
                }
            }
        }

        // Stage to push the Docker image to ECR.
        stage('Push to ECR') {
            steps {
                script {
                    // Push the Docker image to ECR using the tag created earlier.
                    sh "docker push ${env.ECR_REPO}:${env.TAG}"
                }
            }
        }

        // Stage to deploy the Docker image to the appropriate EC2 environment.
        stage('Deploy to Environment') {
            steps {
                script {
                    // Define a variable to hold the EC2 instance IP address for the environment.
                    def targetHost = ''

                    // Determine the target EC2 instance based on the branch name.
                    if (env.BRANCH_NAME == 'dev') {
                        // If the branch is 'dev', set the target EC2 IP to the development environment.
                        targetHost = '<DEV-EC2-IP>'
                    } else if (env.BRANCH_NAME == 'staging') {
                        // If the branch is 'staging', set the target EC2 IP to the staging environment.
                        targetHost = '<STAGING-EC2-IP>'
                    } else if (env.BRANCH_NAME == 'main') {
                        // If the branch is 'main', set the target EC2 IP to the production environment.
                        targetHost = '<PROD-EC2-IP>'
                    }

                    // Deploy the Docker image to the selected EC2 instance.
                    // This block uses SSH to connect to the EC2 instance and run the necessary Docker commands.
                    sh """
                    ssh -i ${SSH_KEY} ec2-user@${targetHost} << EOF
                    // Pull the Docker image from ECR on the EC2 instance.
                    docker pull ${ECR_REPO}:${TAG}
                    // Stop and remove any existing container with the same name to ensure we can run the new one.
                    docker stop ${IMAGE_NAME} || true
                    docker rm ${IMAGE_NAME} || true
                    // Run the new Docker container on the EC2 instance with the pulled image.
                    docker run -d --name ${IMAGE_NAME} -p 80:80 ${ECR_REPO}:${TAG}
                    EOF
                    """
                }
            }
        }
    }

    // Post actions to send notifications after the pipeline execution.
    post {
        // Action on successful build.
        success {
            // Send an email notification when the Docker image is successfully pushed to ECR.
            emailext(
                // Email subject when the build is successful.
                subject: "Jenkins Job - Docker Image Pushed to ECR Successfully",
                // Email body providing information about the successful image push.
                body: "Hello,\n\nThe Docker image '${env.IMAGE_NAME}:${env.TAG}' has been successfully pushed to ECR.\n\nBest regards,\nJenkins",
                // Define the recipients of the email, using Jenkins' developers list as a dynamic recipient provider.
                recipientProviders: [[$class: 'DevelopersRecipientProvider']],
                // Fixed email recipient.
                to: "m.ehtasham.azhar@gmail.com"
            )
        }

        // Action on build failure.
        failure {
            // Send an email notification when the build fails.
            emailext(
                // Email subject when the build fails.
                subject: "Build Failed: ${env.JOB_NAME} #${env.BUILD_ID}",
                // Email body providing information that the build failed.
                body: "The build has failed.\n\nCheck Jenkins for details.",
                // Fixed email recipient in case of failure.
                to: "m.ehtasham.azhar@gmail.com"
            )
        }
    }
}
